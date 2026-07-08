# maddy SMTP Server — Runtime-Evidenced Security Review

**Subject:** DATA message-boundary (dot-terminator) handling, and authentication-identity lifecycle across SMTP transactions
**Repository:** `github.com/foxcpp/maddy`
**Commit:** `26452dd8dd787dc455278b0fdd296f4a5432c768`
**Module / toolchain:** `github.com/foxcpp/maddy`, `go 1.13` [go.mod:L1-L3]
**Method:** every behavioral claim below was produced by **building and running maddy** and capturing the real SMTP dialogue, structured/`io_debug` logs, and on-disk queue artifacts. Code references were re-verified line-by-line against the checked-out source at the commit above. Nothing in the two answers is inferred from reading alone.

---

## 1. Scope and the Two Questions

This document answers two independent questions by observing what the *running* maddy server actually does, and then judges whether maddy **fails safe** under protocol misuse.

- **Q1 — DATA / message-boundary (dot-terminator).** A message body contains normal content, then a line with only a dot, then *more data* before the final terminator. Does maddy **stop** at the first bare-dot line, **keep consuming**, or end in an **unexpected state**? What runtime signs reveal which path executed? What actually ends up **stored/queued**, and how are the post-dot bytes answered?

- **Q2 — Authentication identity across transactions.** A client authenticates as user **A**, issues `MAIL FROM:<A>`, `RSET`, then `MAIL FROM:<B>` **without re-authenticating**. Does maddy **reject**, **tie the message to the original identity (A)**, or **blur accountability**? When delivery decisions are made, what evidence shows which identity was ultimately trusted for **headers**, **queue metadata**, and **enforcement checks**?

Both questions were exercised on **both** intake paths: the unauthenticated `smtp` endpoint on port **25** and the authenticated `submission` endpoint on port **587**.

---

## 2. Direct Answers (up front)

**Q1 — maddy STOPS at the first bare-dot line.** The stored/queued message contains only the pre-dot content, and any bytes *after* the first bare-dot line are **re-interpreted as new SMTP commands** (not absorbed into the message, not silently discarded). This message-boundary decision is **not maddy code** — it is delegated to Go's standard-library `net/textproto` dot reader, reached through the `go-smtp` protocol engine. The behavior is **identical on ports 25 and 587** (same reader). Observed leniency of that reader matters for the smuggling threat model: a **bare-`LF`** sequence (`\n.\n`) **does terminate** (lenient), while a **bare-`CR`** sequence (`\r.\r`) **does not terminate** (the transaction aborts with `unexpected EOF`).

**Q2 — maddy TIES the message to the original authenticated identity A, and does NOT re-scope identity on `RSET`.** The authenticated identity is connection-scoped and set exactly once when the session is created; neither `RSET` nor maddy's session reset clears it. A mismatched `MAIL FROM:<B>` is **accepted at the command layer** (no envelope-vs-auth comparison happens there). Downstream, identity **A** remains the trusted identity **in memory** — it appears in the acceptance log's `username` field, in the `{auth_user}` check, and in the relayed downstream SASL credentials — while the envelope **B** appears in the `Received` header and the on-disk queue metadata. On disk, the authenticated identity is **entirely absent** (the connection state is nulled before the `.meta` file is written). Default enforcement of the mismatch is **signature-withholding by the DKIM signer, not transaction rejection**; a separate domain-routing gate rejects non-local sender domains with `501 5.1.8`.

**Fails-safe verdict (detailed in §8):** For Q1, maddy's behavior is **RFC 5321 §4.5.2-compliant and fails safe** on the canonical and bare-`CR` cases; the **bare-`LF` leniency** is the one smuggling-relevant caveat, inherited from `net/textproto`. For Q2, maddy fails safe **for live enforcement and accountability in memory** (A is consistently the trusted identity and the mismatch is caught by the DKIM signer), but the **on-disk queue metadata drops the authenticated identity**, so post-hoc disk forensics see only the envelope sender.

---

## 3. Environment and Canonical Build

All commands were run as a normal user would, in the default configuration. Toolchain:

```
$ go version
go version go1.13.15 linux/amd64
```

Canonical build from the real entry point `cmd/maddy/main.go` (the `nopam`/`nosqlite3` tags — equivalently `CGO_ENABLED=0` — avoid cgo for a clean CI-style build):

```
$ go build -tags 'nopam nosqlite3' -o /tmp/maddy-bin ./cmd/maddy
$ echo exit=$?
exit=0
$ ls -la /tmp/maddy-bin | awk '{print $5}'
18859700
```

Exit `0`; binary size **18,859,700 bytes (~18 MiB)**.

Version banner. A plain `go build` does **not** VCS-stamp the version, so the banner reads:

```
$ /tmp/maddy-bin -v
maddy unknown (built from source tree)
```

Any version string injected by packaging scripts via linker flags (`-X`) would be **non-canonical**; the canonical plain build reports exactly the string above.

**Invocation interface (this revision).** `maddy` has **no** `run` subcommand. The relevant flags are `-config` (default `/etc/maddy/maddy.conf`), `-debug`, `-libexec`, `-log`, and `-v`. The server is therefore run as `/tmp/maddy-bin -debug -config <path>`. The `-debug` flag is **required** to emit the raw `io_debug` SMTP dialogue to the log (the debug writer is otherwise discarded).

**Pinned dependency versions** (from `go.mod`; not modified by this task):

| Package | Version | Relevance |
|---|---|---|
| `github.com/emersion/go-smtp` | `v0.12.1-0.20191206174923-1f576e0ec85c` [go.mod:L19] | SMTP protocol engine: DATA reader, dot handling, `RSET`, `AUTH` |
| `github.com/emersion/go-message` | `v0.10.9-0.20191116124005-65fd0119e899` [go.mod:L16] | `textproto` header read/parse used by `prepareBody` |
| `github.com/emersion/go-sasl` | `v0.0.0-20190817083125-240c8404624e` [go.mod:L18] | SASL PLAIN/LOGIN for authentication and downstream relay |
| `github.com/emersion/go-msgauth` | `v0.3.2-0.20191028231513-55b75676976c` [go.mod:L17] | DKIM signing used by `internal/modify/dkim` |
| `github.com/google/uuid` | `v1.1.1` [go.mod:L23] | Message-ID generation |
| `golang.org/x/net` | `v0.0.0-20191126235420-ef20fe5d7933` [go.mod:L31] | IDNA hostname handling for endpoints |

The DATA message boundary (Q1) is resolved inside Go's standard library (`net/textproto`, bundled with Go 1.13.15), reached through `go-smtp` — this attribution is central to Q1 and is proven in §5.

---

## 4. Ephemeral Runtime Harness

To exercise the real code paths over TCP, a minimal instance was run from the canonical binary. **All harness files live under `/tmp/maddy-scratch/` (outside the tracked repository tree) and are deleted after observation** (see §8), so the repository remains byte-for-byte unchanged apart from this document — the read-only rule this task is governed by.

**Port framing.** The user's scenario names authenticated submission on **port 587**. The shipped `maddy.conf` binds `submission` on **`tls://0.0.0.0:465`** [maddy.conf:L93]. Both traverse the *identical* `submission` endpoint code path; the harness configures `submission` on `:587` to match the user's scenario exactly. The shipped default is `:465`.

**Auth provider.** To keep the canonical cgo-free build (`nopam nosqlite3` disables the sqlite3-backed `sql` store and `maddyctl users create`), the harness uses the pure-Go `extauth` provider with a tiny external credential-check helper script — no database, no cgo.

**Two queue targets (harness detail, not a maddy behavior).** The submission path (`:587`) uses a queue whose downstream relays SASL credentials (`auth forward`) so the Q2 SASL-relay identity is observable at a sink, and so authenticated messages persist on disk (the sink returns `451` so the queue keeps `.header`/`.body`/`.meta`). The `:25` path uses a second queue **without** `auth forward`, because the credential-forwarding guard permanently fails unauthenticated mail (`530 ... Credentials forwarding is requested but the client is not authenticated`) and the queue would otherwise remove the files before they can be inspected. The **DATA message-boundary code path under test is byte-identical** on both queues.

The full scratch `maddy.conf` used:

```
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

`insecure_auth yes` maps to `endp.serv.AllowInsecureAuth` [internal/endpoint/smtp/smtp.go:L564]; `io_debug yes` maps to `ioDebug` [smtp.go:L565]. The `submission` endpoint forces authentication — `endp.authAlwaysRequired = true` [smtp.go:L590] and an auth provider is mandatory [smtp.go:L592-L593]. `sign_dkim` auto-generates an RSA-2048 key on first start.

The credential helper (`extauth` protocol: line 1 = account, line 2 = password; exit 0 = OK):

```
#!/bin/sh
read -r acct
read -r pass
if [ "$acct" = "usera" ] && [ "$pass" = "password123" ]; then exit 0; fi
if [ "$acct" = "userb" ] && [ "$pass" = "password456" ]; then exit 0; fi
exit 1
```

Run invocation and startup log (both endpoints bind; DKIM key generated; provider registered):

```
$ nohup /tmp/maddy-bin -debug -config /tmp/maddy-scratch/maddy.conf > maddy.log 2>&1 &
$ grep -E 'listening|dkim|authentication provider' maddy.log
smtp: listening on tcp://0.0.0.0:25
[debug] /tmp/maddy-scratch/maddy.conf:61: new module sign_dkim [localhost default]
[debug] submission: authentication provider: extauth local_authdb
submission: listening on tcp://0.0.0.0:587
```

Connectivity and the `SIZE` advertisement (runtime corroboration of the 32 MiB `max_message_size` default at `smtp.go:L561`, `32*1024*1024 = 33554432`):

```
banner (:25) : b'220 localhost ESMTP Service Ready\r\n'
banner (:587): b'220 localhost ESMTP Service Ready\r\n'
EHLO (:587)  : 250-PIPELINING / 250-8BITMIME / 250-ENHANCEDSTATUSCODES / 250-AUTH PLAIN / 250-SMTPUTF8 / 250 SIZE 33554432
AUTH PLAIN   : b'235 2.0.0 Authentication succeeded\r\n'
```

Each scenario below was run **at least twice** with identical input; observed variance was **zero** (only per-message IDs and timestamps differ). This determinism is stated per case.

---


## 5. Q1 — DATA / Message-Boundary (Dot-Terminator) Handling

### 5.1 Direct answer

maddy **stops reading at the first line containing only a dot.** The message that is stored/queued is exactly the content *before* that line. Any bytes that follow the first bare-dot line are **not** part of the message: they are handed back to the SMTP command loop and **answered as new SMTP commands**. On the canonical `<CR><LF>.<CR><LF>` terminator and the bare-`CR` variant, maddy behaves in an RFC-5321-compliant, fail-safe way; on the bare-`LF` variant it is **lenient** (the bare-`LF` line terminates the message), which is the one smuggling-relevant observation. **The behavior is identical on ports 25 and 587.**

### 5.2 Attribution — the boundary is decided by `net/textproto`, not by maddy

The DATA terminator is **not** resolved by any maddy-specific code. maddy's session hands a plain `io.Reader` to the delivery pipeline; that reader is constructed and driven entirely by the `go-smtp` engine and Go's standard library. The chain, re-verified at the pinned versions:

1. **go-smtp `Conn.handleData`** writes the `354` continuation, constructs the DATA reader, calls the session `Data` method, then **drains** whatever the session did not consume:
   ```
   go-smtp@v0.12.1-0.20191206174923-1f576e0ec85c/conn.go
   L498  func (c *Conn) handleData(arg string) {
   L510      c.WriteResponse(354, EnhancedCode{2, 0, 0}, "Go ahead. End your data with <CR><LF>.<CR><LF>")
   L512      defer c.reset()
   L519      r := newDataReader(c)
   L520      code, msg := toSMTPStatus(c.Session().Data(r))
   L521      io.Copy(ioutil.Discard, r)   // Make sure all the data has been consumed
   ```
2. **go-smtp `newDataReader`** wraps `c.text.DotReader()` and applies the size limit:
   ```
   go-smtp .../data.go
   L51  func newDataReader(c *Conn) io.Reader {
   L53      r: c.text.DotReader(),
   L56      if c.server.MaxMessageBytes > 0 { ... }
   ```
3. **Go stdlib `net/textproto`** — the actual boundary state machine. `DotReader` returns a `dotReader` whose `Read` terminates at the first line that is only a dot, un-stuffs leading dots (`..` → `.`), and translates the CRLF framing to `\n`:
   ```
   /usr/local/go/src/net/textproto/reader.go   (Go 1.13.15)
   L299  func (r *Reader) DotReader() io.Reader { ... }
   L318      stateDotCR            // read ".\r" at beginning of line
   L321      stateEOF              // reached ".\r\n" end marker line
   L324      for n < len(b) && d.state != stateEOF { ... }
   L347          d.state = stateDotCR      // '.' then '\r'
   L351          d.state = stateEOF        // '.' then '\n'  -> BARE-LF TERMINATES
   L356      case stateDotCR:
   L358          d.state = stateEOF        // ".\r" then '\n' -> canonical terminator
   L362-365      // else: UnreadByte, emit '\r' as data, back to stateData -> BARE-CR does NOT terminate
   L389      if err == nil && d.state == stateEOF { ... }  // EOF surfaced to caller
   ```

Because `handleData` calls `defer c.reset()` [conn.go:L512] and `io.Copy(ioutil.Discard, r)` [conn.go:L521] only drains **to the `dotReader`'s EOF** (i.e. up to the first bare-dot line), any bytes after that line remain in the connection buffer. `handleData` returns, and the top-level command loop resumes:
```
go-smtp .../conn.go
L88   func (c *Conn) handle(cmd string, arg string) { ... }   // top-level command dispatch
L102      // empty command -> 500 5.5.2 "Speak up"
L131-133  // RSET -> c.reset() + 250 "Session reset"
```
So the trailing bytes are parsed as **new commands**. This is exactly the SMTP-smuggling-relevant path, and it is observed directly below.

The **maddy side** of DATA (what stores the body) is thin and sits *above* the reader:
```
internal/endpoint/smtp/smtp.go
L283  func (s *Session) prepareBody(ctx, r) ...
L285      header, err := textproto.ReadHeader(bufr)          // read header
L291      if s.endp.submission { ... submissionPrepare ... } // 587 only
L298      body, err := buffer.BufferInMemory(bufr)           // buffer the body from the dot reader
L303      received, err := target.GenerateReceived(ctx, s.msgMeta, s.endp.serv.Domain, s.msgMeta.OriginalFrom)
L307      header.Add("Received", received)
L312  func (s *Session) Data(r io.Reader) error { ... }
L326      s.delivery.Body(ctx, ...)
L330      s.delivery.Commit(ctx)
L334      s.log.Msg("accepted", "msg_id", s.msgMeta.ID)       // acceptance signal
```
The body is held by `BufferInMemory` [internal/buffer/memory.go:L27]. Nothing in this maddy code decides the terminator — it only consumes what the `net/textproto` reader yields.

### 5.3 Per-variant runtime evidence

Five payload shapes were run on **both** ports, each **twice**. The `354` prompt on both ports is identical: `354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>`. A global finding across every stored body: the stored `.body` uses **`\n` line endings** (the `net/textproto` reader rewrites `\r\n` → `\n`); the `.header` retains CRLF folding.

#### Variant 1 — canonical CRLF bare-dot (`<CR><LF>.<CR><LF>`) — TERMINATES

Client payload (`:25`) and full dialogue:
```
>> BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nline one\r\nline two\r\n.\r\n'
<< DATA resp (expect 354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
<< POST-DATA resp: b'250 2.0.0 OK: queued\r\n'
<< NOOP probe resp: b'250 2.0.0 I have sucessfully done nothing\r\n'
<< QUIT resp: b'221 2.0.0 Goodnight and good luck\r\n'
```
Structured log (`:25`, unauthenticated — note **no** `username` field):
```
smtp: incoming message	{"msg_id":"cca37a8b","sender":"usera@localhost","src_host":"client.test","src_ip":"127.0.0.1:..."}
smtp: accepted	{"msg_id":"cca37a8b"}
smtp: 250 2.0.0 OK: queued
```
Stored `.body` byte-exact (`od -c`) — LF-normalized, 18 bytes, ends cleanly after `two\n`:
```
0000000   l   i   n   e       o   n   e  \n   l   i   n   e       t   w
0000020   o  \n
0000022
```
Stored `.header` (`:25`) — the client "from" trace is present on port 25:
```
Received: from client.test (localhost [127.0.0.1]) by localhost
 (envelope-sender <usera@localhost>) with ESMTP id cca37a8b; Wed, 08 Jul 2026 04:51:06 +0000
From: <usera@localhost>
To: <dest@localhost>
Subject: q1 test
```
On `:587` the same payload yields the same 18-byte body; the header instead carries a `Dkim-Signature:` and the `by localhost (envelope-sender <usera@localhost>)` clause **without** the client "from" trace (submission sets `DontTraceSender`). Both ports, both runs: identical (`250 2.0.0 OK: queued`).

#### Variant 2 — bare-`LF` terminator (`\n.\n`, no CR) — TERMINATES (LENIENT)

Byte-sensitive payload and result:
```
>> BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nline one\nline two\n.\n'
<< POST-DATA resp: b'250 2.0.0 OK: queued\r\n'
```
Log (`:25`):
```
smtp: incoming message	{"msg_id":"a9628cab","sender":"usera@localhost",...}
smtp: accepted	{"msg_id":"a9628cab"}
smtp: 250 2.0.0 OK: queued
```
Stored `.body` (`od -c`) — 18 bytes, identical to the canonical case:
```
0000000   l   i   n   e       o   n   e  \n   l   i   n   e       t   w
0000020   o  \n
0000022
```
**A bare `\n.\n` is accepted as end-of-data.** This is the `net/textproto` transition `stateDot` + `'\n'` → `stateEOF` [reader.go:L351]. Identical on `:25` and `:587`, both runs. This is the leniency that matters for SMTP smuggling (§7).

#### Variant 3 — bare-`CR` terminator (`\r.\r`, no LF) — DOES NOT TERMINATE (STRICT)

Byte-sensitive payload and result:
```
>> BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nline one\rline two\r.\r'
<< POST-DATA resp: b''          # client receives nothing — server is still reading DATA
<< NOOP probe resp: b''
<< QUIT resp: b''
```
When the client eventually closes the socket, the server's dot reader hits EOF mid-line and the transaction is aborted:
```
smtp: incoming message	{"msg_id":"498ffc08","sender":"usera@localhost",...}
smtp: DATA error	{"msg_id":"498ffc08","reason":"unexpected EOF"}
smtp: 554 5.0.0 Internal server error (msg ID = 498ffc08)
[debug] smtp: reset
```
Nothing is stored:
```
queue (:25)  -> (queue empty)
queue (:587) -> (queue empty)
```
A bare `\r.\r` is **not** an end-of-data marker: `stateDotCR` sees no following `\n`, un-reads the byte, emits the `'\r'` as ordinary body data, and returns to `stateData` [reader.go:L362-L365]; the reader keeps waiting for a real terminator until the connection closes → `io.ErrUnexpectedEOF`. Identical on `:25` and `:587`, both runs (empty queue all four times).

#### Variant 4 — mid-body bare dot + trailing data (the user's core example) — STOPS, trailing re-interpreted as commands

Payload: normal line, then `.<CR><LF>`, then *more lines*, then the real `<CR><LF>.<CR><LF>`. Full `:25` dialogue:
```
>> BODY PAYLOAD: b'From: <usera@localhost>\r\nTo: <dest@localhost>\r\nSubject: q1 test\r\n\r\nbefore dot line\r\n.\r\nAFTER DOT LINE ONE\r\nAFTER DOT LINE TWO\r\n.\r\n'
<< DATA resp (expect 354): b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
<< POST-DATA resp: b'250 2.0.0 OK: queued\r\n501 5.5.2 Bad command\r\n501 5.5.2 Bad command\r\n501 5.5.2 Bad command\r\n'
<< NOOP probe resp: b'250 2.0.0 I have sucessfully done nothing\r\n'
<< QUIT resp: b'221 2.0.0 Goodnight and good luck\r\n'
```
Log — exactly **one** message is accepted, then three trailing lines are answered as commands:
```
smtp: incoming message	{"msg_id":"6c802323","sender":"usera@localhost",...}
smtp: accepted	{"msg_id":"6c802323"}
smtp: 501 5.5.2 Bad command
smtp: 501 5.5.2 Bad command
smtp: 501 5.5.2 Bad command
```
Stored `.body` (`od -c`) — **only** the pre-dot line, 16 bytes:
```
0000000   b   e   f   o   r   e       d   o   t       l   i   n   e  \n
0000020
```
This is the decisive answer to the user's example. Of the three candidate outcomes:
- **(a) absorbed into the stored message** — NO; the stored body ends at `before dot line\n`.
- **(b) silently discarded** — NO; the server actively answered the trailing lines.
- **(c) re-interpreted as new SMTP commands** — **YES.** `AFTER DOT LINE ONE`, `AFTER DOT LINE TWO`, and the trailing `.` are each parsed as commands and rejected with `501 5.5.2 Bad command`.

Identical on `:25` and `:587`, both runs. On `:587` the queue persisted `3de59b61.{body,header,meta}` with the same 16-byte body.

**Smuggling demonstration.** When the trailing lines are *valid* SMTP commands rather than garbage, they start a **new transaction**. On `:25`, a smuggled `MAIL FROM:<evil@attacker.example>` after the bare dot is accepted as a fresh message with its own `msg_id` and `incoming message ... sender=evil@attacker.example`. On `:587` the smuggled `MAIL FROM` is likewise accepted (new `msg_id`) but is **tagged with the still-pinned `username=usera@localhost`** (the same connection-scoped identity — see Q2), and its recipient is then rejected by the non-local domain gate (`501 5.1.8`). The mechanism — post-terminator bytes becoming new commands — is the same in both cases.

#### Variant 5 — dot-stuffed content (`..` lines) — UN-STUFFED

Payload lines beginning with a literal dot are dot-stuffed by the client (`..`, `...`). Result and stored body (`:25`, `od -c`):
```
>> BODY PAYLOAD: b'...normal line\r\n..dotstuffed line\r\n...three dots\r\n.\r\n'
<< POST-DATA resp: b'250 2.0.0 OK: queued\r\n'
```
```
0000000   n   o   r   m   a   l       l   i   n   e  \n   .   d   o   t
0000020   s   t   u   f   f   e   d       l   i   n   e  \n   .   .   t
0000040   h   r   e   e       d   o   t   s  \n
0000052
```
Each line's leading dot is removed exactly once: `..dotstuffed line` → `.dotstuffed line`; `...three dots` → `..three dots`. This is the un-stuffing at `reader.go` (`stateDot` → `stateData` when the dot is not the whole line). Identical on `:25` and `:587`, both runs.

### 5.4 Before / during / after (connection state around the bare-dot line)

- **Before:** the connection is in the command loop; `MAIL`/`RCPT` accepted; `DATA` issued.
- **During:** after `354`, the session reads through `newDataReader` → `net/textproto` dot reader. At the **first** bare-dot line the dot reader returns `io.EOF`; `Session.Data` returns; maddy logs `accepted` [smtp.go:L334] and writes `250`.
- **After:** `io.Copy(ioutil.Discard, r)` [conn.go:L521] drains only up to that first EOF, so it does **not** consume trailing bytes; `defer c.reset()` [conn.go:L512, L694] clears recipients and the from-received flag (but **not** authentication — see Q2); the command loop resumes [conn.go:L88] and reads the trailing bytes as new commands. For the bare-`CR` case the dot reader never reaches EOF, so the read blocks until the client closes → `unexpected EOF` and no `250`.

---


## 6. Q2 — Authentication Identity Across Transactions

### 6.1 Direct answer

maddy **does not reject** the identity switch and **does not re-scope** the identity. It **ties the message to the original authenticated identity A** while **accepting the mismatched envelope B at the command layer**. The authenticated identity is connection-scoped and set exactly once at session creation; `RSET` does not clear it. In memory, **A** is consistently the trusted identity (acceptance-log `username`, the `{auth_user}` check, and the relayed downstream SASL credentials all resolve to A); the envelope **B** appears in the `Received` header and the on-disk `.meta`. On disk the authenticated identity is **absent entirely** (the connection state is nulled before serialization). The default response to the mismatch is the DKIM signer **withholding its signature** (not a rejection); a **separate** domain-routing gate rejects non-local sender domains with `501 5.1.8`.

### 6.2 Why the identity is pinned (grounded, re-verified anchors)

```
internal/endpoint/smtp/smtp.go
L643  func (endp *Endpoint) Login(state *smtp.ConnectionState, username, password string) (smtp.Session, error) {
L653      // endp.Auth.CheckPlain(username, password) verifies credentials
L658      return endp.newSession(false, username, password, state), nil
L674  func (endp *Endpoint) newSession(anonymous bool, username, password string, state *smtp.ConnectionState) smtp.Session {
L680          AuthUser:        username,     // <-- set EXACTLY ONCE, connection-scoped
L60   func (s *Session) Reset() {           // RSET -> only aborts an in-flight delivery
L64        // s.log.DebugMsg("reset")
L67   func (s *Session) abort(ctx) {        // clears mailFrom/opts/msgMeta/delivery ...
L74-L80    //   ... but NEVER touches s.connState.AuthUser
L162  func (s *Session) Mail(from string, opts smtp.MailOptions) error {   // NO envelope-vs-auth compare
```
And go-smtp's own reset clears recipients / the from-received flag and calls the session reset, but **never** clears the authenticated session:
```
go-smtp .../conn.go
L694  func (c *Conn) reset() { ... clears recipients, fromReceived; calls Session.Reset(); no auth clearing ... }
```
Submission forces authentication: `endp.authAlwaysRequired = true` [smtp.go:L590]; an anonymous login on submission is refused with `smtp.ErrAuthRequired` [smtp.go:L662-L663].

The identity is carried through the pipeline in memory on the connection state:
```
internal/module/msgmetadata.go     (note: internal/module/, not framework/module/)
L10   type ConnState struct { ... }
L37       AuthUser string
L55   type MsgMetadata struct { ... }
L66       OriginalFrom string     // the MAIL FROM value (envelope sender)
L104      Conn *ConnState         // carries AuthUser through delivery
```

### 6.3 Primary scenario — `AUTH A → MAIL FROM:<A> → RSET → MAIL FROM:<B> → DATA`

The switch is **accepted at the command layer** (no re-authentication, no complaint):
```
<< AUTH (usera@localhost): b'235 2.0.0 Authentication succeeded\r\n'
<< MAIL1 <usera@localhost>: b'250 2.0.0 Roger, accepting mail from <usera@localhost>\r\n'
<< RSET: b'250 2.0.0 Session reset\r\n'
<< MAIL2 <userb@localhost> (no re-auth): b'250 2.0.0 Roger, accepting mail from <userb@localhost>\r\n'
<< RCPT: b"250 2.0.0 I'll make sure <dest@localhost> gets this\r\n"
<< DATA: b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
>> DATA body: b'From: <userb@localhost>\r\nTo: <dest@localhost>\r\nSubject: q2\r\n\r\nbody line\r\n.\r\n'
<< POST-DATA: b'250 2.0.0 OK: queued\r\n'
```
Each named artifact is answered separately below, with its captured evidence and re-verified `file:line`. All results were **byte-identical across both runs** (only `msg_id` differs).

#### (1) Acceptance log — `username` vs `sender`

`startDelivery` logs the `incoming message` line; the `username` field is emitted **only** inside `if s.connState.AuthUser != ""` [smtp.go:L127], as `"username", s.connState.AuthUser` [smtp.go:L133]. Observed:
```
submission: incoming message	{"msg_id":"a0a41bc5","sender":"userb@localhost","src_host":"client.test","src_ip":"127.0.0.1:...","username":"usera@localhost"}
```
The envelope sender is **B** (`sender=userb@localhost`) but the authenticated identity is the pinned **A** (`username=usera@localhost`). Run 2: identical (`sender=userb@localhost`, `username=usera@localhost`, `msg_id=66d1ee80`).

#### (2) `Received` header — records the envelope sender B, not A

`GenerateReceived` [internal/target/received.go:L19] writes the `by <ourhost>` clause [received.go:L62] and the `(envelope-sender <...>)` clause [received.go:L69] **always**, using the `MAIL FROM` value (`s.msgMeta.OriginalFrom`, passed from `prepareBody` at smtp.go:L303). The client "from" trace is suppressed on submission because `submissionPrepare` sets `msgMeta.DontTraceSender = true` [internal/endpoint/smtp/submission.go:L27-L28], gated at received.go:L30. Observed header (captured at the downstream relay):
```
Received:  by localhost (envelope-sender <userb@localhost>) with ESMTP id
 a0a41bc5; Wed, 08 Jul 2026 04:56:30 +0000
Date: Wed, 8 Jul 2026 04:56:30 +0000
Message-Id: <eadc7ce9-6d3d-4f4b-bc52-045c90e0eebc@localhost>
From: <userb@localhost>
To: <dest@localhost>
Subject: q2
```
The `Received` header records **B** (`envelope-sender <userb@localhost>`). It does **not** record the authenticated A.

#### (3) On-disk queue `.meta` — the authenticated identity is ABSENT

`updateMetadataOnDisk` sets `metaCopy.MsgMeta.Conn = nil` [internal/target/queue/queue.go:L752] **before** `json.NewEncoder(file).Encode(metaCopy)` [queue.go:L754]. Because `AuthUser` lives only on `MsgMeta.Conn.AuthUser`, the on-disk `.meta` records the envelope `From`/`OriginalFrom` (B) but **no** authenticated identity. Actual `.meta`:
```
{"MsgMeta":{"ID":"a0a41bc5","OriginalFrom":"userb@localhost","DontTraceSender":true,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"userb@localhost","To":["dest@localhost"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"dest@localhost":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Sink temp-fail on purpose (keep queued)"}},"TriesCount":1,"FirstAttempt":"2026-07-08T04:56:28.38697289Z","LastAttempt":"2026-07-08T04:56:30.519866157Z"}
```
Note `"Conn":null` and the two envelope fields `"OriginalFrom":"userb@localhost"` / `"From":"userb@localhost"`. There is **no** `AuthUser` anywhere in the file. This in-memory-versus-on-disk split is the precise answer to the "queue metadata" sub-part: the queue's persisted metadata attributes the message to the **envelope sender B**, and the authenticated **A is not recoverable from disk**.

#### (4) DKIM enforcement — signature withheld, message still queued

`require_sender_match` defaults to `["envelope","auth"]` [internal/modify/dkim/dkim.go:L151]. `RewriteBody` reads `authUser = s.meta.Conn.AuthUser` [dkim.go:L345] and calls `shouldSign(...)` [dkim.go:L249], which declines on an auth mismatch [dkim.go:L306] or an envelope mismatch [dkim.go:L294]. In the primary scenario (header `From:<B>`, authenticated as A), the **auth** check fires:
```
sign_dkim: not signing, From address is not authenticated identity	{"auth_id":"usera@localhost","from_addr":"userb@localhost","msg_id":"a0a41bc5"}
submission: accepted	{"msg_id":"a0a41bc5"}
submission: 250 2.0.0 OK: queued
```
Crucially, the signer **declines to sign** but the message is **still `accepted` and `250 2.0.0 OK: queued`** — default enforcement is **signature-withholding, not transaction rejection**. Run 2 identical (`auth_id=usera@localhost`, `from_addr=userb@localhost`, `msg_id=66d1ee80`).

The sibling **envelope** decline was also exercised directly (header `From:<A>`, envelope `MAIL FROM:<B>`, both local):
```
sign_dkim: not signing, From address is not envelope address	{"envelope":"userb@localhost","from_addr":"usera@localhost","msg_id":"4135466c"}
```
This confirms the envelope check [dkim.go:L294] fires before the auth check [dkim.go:L306]. Run 2 identical (`msg_id=76e0f293`).

#### (5) `{auth_user}` command check — resolves to A

The `command` check exposes the authenticated identity: `case "{auth_user}":` [internal/check/command/command.go:L145] returns `s.msgMeta.Conn.AuthUser` [command.go:L149]. A scratch `command` check wired to echo `{auth_user}` was run in the primary scenario:
```
COMMAND-CHECK {auth_user} resolved to = [usera@localhost]
```
Even though the envelope is B, the check receives **A**. Run 2 identical (`[usera@localhost]`).

#### (6) Downstream SASL relay — relays A (and A's password)

`internal/target/smtp_downstream/sasl.go` relays the authenticated identity: `sasl.NewPlainClient("", msgMeta.Conn.AuthUser, msgMeta.Conn.AuthPassword)` [sasl.go:L41] (guarded at sasl.go:L31). With `auth forward` on the downstream, the sink captured the relayed SASL exchange and the envelope command:
```
DOWNSTREAM-AUTH: b'\x00usera@localhost\x00password123'
DOWNSTREAM-MAIL: MAIL FROM:<userb@localhost>
DOWNSTREAM-RCPT: RCPT TO:<dest@localhost>
```
The relayed SASL identity is **A** (`usera@localhost`, with A's password) while the relayed envelope is **B** (`userb@localhost`). Run 2 identical (`DOWNSTREAM-AUTH: b'\x00usera@localhost\x00password123'`). This is the sharpest demonstration that A remains the trusted identity in memory across the switch.

#### (7) `ConnState.AuthUser` / `MsgMetadata.Conn` propagation

The four in-memory observations above (log, DKIM, `{auth_user}`, SASL relay) all read `msgMeta.Conn.AuthUser` and all resolved to **A** in the same run — a direct runtime demonstration that the connection-scoped `AuthUser` [msgmetadata.go:L37] propagates through the pipeline via `MsgMetadata.Conn` [msgmetadata.go:L104], unchanged by the `RSET`/`MAIL FROM:<B>` switch.

### 6.4 Controls

**Same-identity control** — `AUTH A → MAIL A → RSET → MAIL A` (header `From:<A>`). No mismatch, so the signer signs:
```
submission: incoming message	{"msg_id":"e0bc9ed1","sender":"usera@localhost",...,"username":"usera@localhost"}
[debug] sign_dkim: signed	{"identifier":"usera@localhost"}
submission: accepted	{"msg_id":"e0bc9ed1"}
submission: 250 2.0.0 OK: queued
```
Run 2 identical (`signed {identifier:usera@localhost}`, `msg_id=440ac997`). This confirms the enforcement in §6.3(4) is specifically about the *mismatch*, not a blanket refusal.

**Anonymous-on-25 control** — connect to `:25`, `MAIL FROM` without auth. The `username` field is absent (the `else` branch at smtp.go:L136 omits it):
```
smtp: incoming message	{"msg_id":"9c204f5e","sender":"other@localhost","src_host":"client.test","src_ip":"127.0.0.1:..."}
smtp: accepted	{"msg_id":"9c204f5e"}
smtp: 250 2.0.0 OK: queued
```
Run 2 identical (`sender=other@localhost`, no `username`, `msg_id=d03cd148`). The unauthenticated `:25` path simply has no authenticated identity to pin; submission's auth requirement (`authAlwaysRequired`) does not apply there.

### 6.5 The domain-routing reject is a SEPARATE gate (do not conflate with DKIM)

The DKIM decline above withholds a signature but still queues. A **distinct** gate — `default_source { reject 501 5.1.8 "Non-local sender domain" }` [maddy.conf:L117-L118] — rejects the *transaction* when the envelope sender's domain is non-local. Exercised with `MAIL FROM:<userb@external.example>`:
```
<< MAIL2 <userb@external.example> (no re-auth): b'250 2.0.0 Roger, accepting mail from <userb@external.example>\r\n'
<< RCPT: b'501 5.1.8 Non-local sender domain (msg ID = 8b766e80)\r\n'
<< DATA: b'502 5.5.1 Missing RCPT TO command.\r\n'
```
Log:
```
submission: incoming message	{"msg_id":"8b766e80","sender":"userb@external.example",...,"username":"usera@localhost"}
submission: RCPT error	{"effective_rcpt":"dest@localhost","rcpt":"dest@localhost","reason":"reject directive used","smtp_code":501,"smtp_enchcode":"5.1.8","smtp_msg":"Non-local sender domain"}
submission: 501 5.1.8 Non-local sender domain (msg ID = 8b766e80)
```
Note the mismatched `MAIL FROM:<userb@external.example>` is still **accepted at the command layer** (`250`), and the message is still logged with the pinned `username=usera@localhost`; the **rejection happens at `RCPT`** via the reject directive — a true `501` transaction failure, not a signature-withholding. Run 2 identical (`RCPT: 501 5.1.8 Non-local sender domain`, `msg ID = e6bc37be`). So the two gates are:
- **DKIM signer** — mismatch of header `From` against envelope or authenticated identity → **declines to sign**, message still queued.
- **Domain routing** — non-local envelope-sender domain → **`501 5.1.8` rejection** at `RCPT`.

---


## 7. Coverage Matrix

Every named sub-part of both questions, mapped to a captured runtime observation. All cells were run **≥2×** with **zero** run-to-run variance.

### Q1 — DATA / message-boundary

| Sub-part | Observation | Evidence |
|---|---|---|
| Stop / continue / unexpected-state | **Stops** at first bare-dot | §5.3 V1/V4; `accepted` + `250 OK: queued` |
| Runtime sign: `354` prompt | `354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>` | §5.3 (both ports) |
| Runtime sign: response codes | `250 OK: queued`; `501 5.5.2`; `554 5.0.0` | §5.3 V1/V3/V4 |
| Runtime sign: `io_debug` dialogue | raw SMTP bytes logged with endpoint prefix | §5.3 (all cells) |
| Runtime sign: `accepted` `msg_id` | `smtp: accepted {"msg_id":...}` [smtp.go:L334] | §5.3 V1/V2/V4 |
| Stored/queued `.body` | byte-exact `od -c`, LF-normalized | §5.3 V1/V2/V4/V5 |
| Post-dot bytes handling | **(c) re-interpreted as commands** | §5.3 V4 (`501` ×3) |
| Variant: canonical CRLF | terminates | §5.3 V1 |
| Variant: bare-`LF` | terminates (lenient) | §5.3 V2 |
| Variant: bare-`CR` | does not terminate; `unexpected EOF` | §5.3 V3 |
| Variant: mid-body dot + trailing | stops; trailing = commands | §5.3 V4 |
| Variant: dot-stuffed | un-stuffed | §5.3 V5 |
| Both ports (25 + 587) | identical behavior | §5.3 (every cell run on both) |

### Q2 — auth identity across transactions

| Sub-part | Observation | Evidence |
|---|---|---|
| reject / tie-to-A / blur | **tie-to-A** (in memory); switch accepted at command layer | §6.3 |
| Acceptance log `username` vs `sender` | `sender=userb`, `username=usera` | §6.3(1) |
| `Received` header identity | envelope **B** (`envelope-sender <userb@localhost>`) | §6.3(2) |
| Queue `.meta` identity | envelope **B**; `"Conn":null` → auth **A absent** | §6.3(3) |
| Enforcement: DKIM (auth mismatch) | `not signing, ... not authenticated identity`; still queued | §6.3(4) |
| Enforcement: DKIM (envelope mismatch) | `not signing, ... not envelope address` | §6.3(4) |
| Enforcement: `{auth_user}` check | resolves to **A** | §6.3(5) |
| Enforcement: downstream SASL relay | relays **A** (`\x00usera@localhost\x00...`) | §6.3(6) |
| `ConnState.AuthUser` / `MsgMetadata.Conn` propagation | A reaches log, DKIM, check, relay in memory | §6.3(7) |
| Control: same-identity | DKIM **signs**; no mismatch | §6.4 |
| Control: anonymous-on-25 | **no** `username` field | §6.4 |
| Separate gate: domain routing | `501 5.1.8` at `RCPT` (non-local sender) | §6.5 |

Nothing in this document is labeled "inferred from reading" — every sub-part was exercised at runtime and is backed by the captured output shown above.

---

## 8. Fails-Safe Judgement

The project's own design yardstick is stated in `HACKING.md`: **"Be secure but interoperable."** [HACKING.md:L19] (the surrounding text explains maddy verifies DKIM and applies DMARC by default to be "as secure as possible while maintaining reasonable interoperability. Though, users can configure maddy to be stricter."). The verdict below is framed against that yardstick and two external references used **as context only** — every concrete behavioral claim traces to the captured bytes in §5–§6.

**Context — RFC 5321 §4.5.2 (end-of-data / dot-transparency).** The end-of-mail indicator is a line containing only a single period; senders dot-stuff a leading period and the receiver strips it. This makes maddy's observed Q1 behavior — *stop at the first bare-dot line, un-stuff leading dots, and treat later bytes as fresh protocol input* — the **RFC-compliant canonical behavior**, not a defect. The captured evidence (§5.3 V1, V4, V5) matches the RFC exactly.

**Context — SMTP smuggling (SEC Consult / Timo Longin; US-CERT VU#302671; 2023–2024 CVE cluster; remediations in Sendmail 8.18, Exim 4.97.1, Postfix `smtpd_forbid_bare_newline`).** The risk is end-of-data **uncertainty** when a server accepts **non-standard** terminators (a bare `LF` or bare `CR`), letting two hops disagree on message boundaries and enabling injected/spoofed messages. Because maddy delegates DATA framing to Go's `net/textproto` dot reader (§5.2), that reader's leniency **determines maddy's smuggling posture** — which is why the bare-`LF`/bare-`CR` variants were exercised, not reasoned about.

**Q1 verdict — fails safe on the canonical and bare-`CR` cases; one smuggling-relevant caveat on bare-`LF`.**
- Canonical `<CR><LF>.<CR><LF>`: stops cleanly, stores only pre-dot content, answers trailing bytes as commands (§5.3 V1, V4). Fail-safe and RFC-compliant.
- Bare-`CR` `\r.\r`: does **not** terminate; the transaction aborts with `unexpected EOF` and **nothing is stored** (§5.3 V3). Fail-safe (a malformed terminator cannot smuggle a second message — it fails the whole transaction).
- Bare-`LF` `\n.\n`: **does** terminate (§5.3 V2). This is the lenient case, inherited from `net/textproto` [reader.go:L351], and is the exact condition the smuggling advisories warn about. maddy at this pinned version does **not** reject bare-`LF` end-of-data. In deployments where an upstream hop frames differently, this leniency is the theoretical smuggling vector. Observed as-is; not a crash or a spoof by itself, but the one place where maddy is *interoperable rather than strict*.
- Post-terminator bytes always fall back to the command parser (§5.3 V4), so garbage is safely rejected (`501`) and only well-formed SMTP commands start a new transaction — and on `:587` even a smuggled `MAIL FROM` is still bound to the pinned authenticated identity (§5.3 smuggling demonstration; §6).

**Q2 verdict — fails safe for live enforcement and in-memory accountability; a forensic gap on disk.**
- The mismatched `MAIL FROM:<B>` is accepted at the command layer, but **A remains the trusted identity in memory** and the mismatch is actively caught: the DKIM signer withholds its signature (§6.3(4)), the `{auth_user}` check sees A (§6.3(5)), the downstream SASL relay carries A (§6.3(6)), and a non-local envelope domain is rejected outright (§6.5). Accountability to the authenticated user is **not blurred at enforcement time** — this is fail-safe and matches "secure but interoperable" (accept the command, enforce at the decision points).
- The one caveat: the **on-disk queue `.meta` drops the authenticated identity** (`"Conn":null`, §6.3(3)), recording only the envelope sender B. Live enforcement is unaffected (it reads the in-memory `Conn`), but **post-hoc disk forensics cannot recover which authenticated user sent a queued message**. The code comment at the nulling site frames this as credential hygiene (avoid serializing secrets), and it does prevent the SASL password from touching disk — but it also removes the accountable identity from the persisted record. This is the single place where Q2's answer is "accountability is weaker than it appears," and it is a persistence/forensics limitation rather than an enforcement bypass.

**Overall:** maddy **fails safe** under both forms of protocol misuse in the sense that matters most — it never silently accepts a smuggled second message on the canonical/bare-`CR` paths, and it never lets an unauthenticated identity switch escape live enforcement. The two honest caveats, both observed at runtime, are (1) bare-`LF` end-of-data leniency inherited from `net/textproto`, and (2) the authenticated identity being absent from on-disk queue metadata.

---

## 9. Appendix

### 9.1 Exact commands

```
# toolchain
go version                                                     # go1.13.15 linux/amd64
# canonical build (real entry point cmd/maddy)
go build -tags 'nopam nosqlite3' -o /tmp/maddy-bin ./cmd/maddy # exit 0; 18859700 bytes
/tmp/maddy-bin -v                                              # maddy unknown (built from source tree)
# run (ephemeral; -debug required for io_debug dialogue)
nohup /tmp/maddy-bin -debug -config /tmp/maddy-scratch/maddy.conf > /tmp/maddy-scratch/maddy.log 2>&1 &
# Q1 driver (per variant, per port): canonical | bare_lf | bare_cr | middot | dotstuffed
python3 q1_driver.py Q1 25  <variant>
python3 q1_driver.py Q1 587 <variant>
# Q2 driver (JSON config): AUTH A -> MAIL1 -> RSET -> MAIL2 -> RCPT -> DATA
python3 q2_driver.py '<json-scenario>'
```

### 9.2 Re-verified `file:line` anchor index

- **Q1 (maddy):** `internal/endpoint/smtp/smtp.go` — `Data` L312, `prepareBody` L283, `ReadHeader` L285, `BufferInMemory` call L298, `GenerateReceived` call L303, `accepted` log **L334**, `max_message_size` L561, `insecure_auth` L564, `io_debug` L565. `internal/buffer/memory.go` — `BufferInMemory` L27.
- **Q1 (delegated reader chain):** `go-smtp@v0.12.1-0.20191206174923-1f576e0ec85c/conn.go` — `handleData` L498, `354` prompt L510, `defer c.reset()` L512, `newDataReader(c)` L519, `Session().Data(r)` L520, `io.Copy(ioutil.Discard, r)` L521, `reset()` L694, command loop `handle` L88. `.../data.go` — `newDataReader` L51, `DotReader()` L53, `MaxMessageBytes` L56. `net/textproto/reader.go` (Go 1.13.15) — `DotReader` L299, `stateDotCR` L318, `stateEOF` L321, loop L324, `stateDot`→`stateDotCR` L347, `stateDot`+`'\n'`→`stateEOF` **L351** (bare-`LF` terminates), `stateDotCR` L356/L358, bare-`CR` un-read L362-L365, EOF surface L389.
- **Q2:** `internal/endpoint/smtp/smtp.go` — `Reset` L60, `abort` L67, `startDelivery` L83, `if AuthUser!=""` L127, `username` log **L133**, else-branch L136, `Mail` L162, `authAlwaysRequired` set L590 / checked L662, `Login` L643, `newSession` L674, `AuthUser: username` **L680**. `internal/target/received.go` — `GenerateReceived` L19, `DontTraceSender` guard L30, `by` clause L62, `envelope-sender` clause L69. `internal/endpoint/smtp/submission.go` — `submissionPrepare` L27, `DontTraceSender=true` L28. `internal/target/queue/queue.go` — `QueueMetadata` L149, `From` L151, `updateMetadataOnDisk` L742, `Conn = nil` **L752**, `Encode` **L754**. `internal/modify/dkim/dkim.go` — `require_sender_match` L151, `shouldSign` L249, not-envelope L294, not-authenticated-identity L306, `RewriteBody` L340, `authUser` read L345. `internal/check/command/command.go` — `{auth_user}` case **L145**, return L149. `internal/target/smtp_downstream/sasl.go` — guard L31, `NewPlainClient` L41. `internal/module/msgmetadata.go` — `ConnState` L10, `AuthUser` L37, `MsgMetadata` L55, `OriginalFrom` L66, `Conn` L104. `maddy.conf` — `submission` `:465` L93, `default_source` reject L117-L118. `HACKING.md` — yardstick L19. `go.mod` — module L1, `go 1.13` L3.

### 9.3 Read-only compliance

This task created exactly one file — this document, `blitzy/documentation/maddy_26452dd8dd78.md`. No existing source file was modified; no dependency, build file, or the tracked `maddy.conf` was changed. All runtime scaffolding (the scratch `maddy.conf`, the `extauth` helper, the DKIM key, the driver scripts, the compiled `/tmp/maddy-bin`, the sink, and the scratch state/queue directories) lived entirely under `/tmp/maddy-scratch` and `/tmp`, outside the tracked tree, and was removed after observation. The clean `git status` (only this new file present) is the proof of compliance.

