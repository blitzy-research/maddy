# maddy — How the SMTP server detects the end of the DATA phase under adversarial line-ending framing

**Repository:** `github.com/foxcpp/maddy`
**Commit:** `26452dd8dd787dc455278b0fdd296f4a5432c768`
**Source branch:** `maddy_26452dd8dd78`
**Document type:** investigative, runtime-observed Q&A (read-only analysis — no production code changed)

---

## Scope note

This document answers, with **observed runtime evidence**, how maddy's inbound SMTP server decides that a message body is finished — the instant it stops reading DATA and returns to command parsing — when a client frames the end-of-DATA boundary in conformant and adversarial ways (bare `LF`, bare `CR`, `LF.CRLF`, dot-stuffing, over-length lines, and pipelined SMTP smuggling), and how the story changes behind a small framing-normalizing / framing-blocking front proxy.

Everything below was produced by **building and running maddy at this exact commit** and driving it with a **raw, byte-exact TCP client** (never a normalizing SMTP library), then quoting the captured output verbatim. Each claim carries a `file:line` citation into the source or a reference to the observed output that produced it. Values that are **random per run** (the 8-hex-char `msg_id`, the client's ephemeral source port, wall-clock timings) come from the coherent investigation described here — each scenario captured against a **freshly restarted** maddy instance with a wiped queue, so the values quoted within any one scenario are internally consistent — and are quoted exactly as observed; values that are **deterministic** (byte counts, SHA-256 digests, status codes, prompt/error literals, config keys, `file:line` citations) reproduce on every run.

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
$ stat -c%s /tmp/maddy_investigation/maddy_bin
21846744
```

The build is the first module fetch; it resolves `github.com/emersion/go-smtp v0.12.1-0.20191206174923-1f576e0ec85c` (`go.mod:19`) into the module cache. All go-smtp `file:line` citations in this document were re-verified against the fetched source at:

```
/root/go/pkg/mod/github.com/emersion/go-smtp@v0.12.1-0.20191206174923-1f576e0ec85c
```

No dependency was added, upgraded, removed, or re-imported; `go.mod`/`go.sum` were used as-is.

### 1.2 Minimal out-of-repository `maddy.conf`

A minimal config was written under `/tmp` (not in the repo). It opens exactly one **plaintext inbound** `smtp` listener (pattern from `maddy.conf:53`), omits every default check (the stock `maddy.conf` smtp block at lines 54–70 has `require_matching_ehlo`, `require_mx_record`, `verify_dkim`, `apply_spf`, `dmarc yes`, which would reject a localhost raw-client probe before DATA is reached), and routes `deliver_to` a `queue` target. In the queue target, `storeNewMessage` creates the header file with `os.Create` at `internal/target/queue/queue.go:694`, creates the body file with `os.Create` at `queue.go:713`, and copies the delivered, dot-unstuffed body bytes into it with `io.Copy(bodyFile, bodyReader)` at `queue.go:719` — so the on-disk `<id>.body` file is exactly the body maddy parsed. The queue's onward `smtp_downstream` target points at a tiny local capture server that returns a temporary `451`, so the queue **defers** and the `<id>.body`/`<id>.header`/`<id>.meta` files **persist** for inspection (the queue removes a message from disk only when `TriesCount == q.maxTries` **or** there are no temporary failures — `queue.go:390`; a `451` is a temporary failure, so the files are retained).

`read_timeout 5s` is set so the non-terminating cases (O4b, O6) fail fast; the default is **10 minutes** (`internal/endpoint/smtp/smtp.go:560` `cfg.Duration("read_timeout", false, false, 10*time.Minute, &endp.serv.ReadTimeout)`). Every accepting variant terminates immediately, so this shortened timeout does not affect them.

```
state /tmp/maddy_investigation/state
runtime /tmp/maddy_investigation/runtime

hostname test.local
tls off

smtp tcp://127.0.0.1:2525 {
    io_debug
    debug
    read_timeout 5s
    deliver_to queue {
        location /tmp/maddy_investigation/queue_dbg
        hostname test.local
        max_tries 8
        target smtp_downstream {
            targets tcp://127.0.0.1:19999
            require_tls no
            attempt_starttls no
        }
    }
}
```

(The `smtp_downstream` bools must be `no`, not `off` — an `off` value produces the parse error `bool argument should be 'yes' or 'no'` and maddy refuses to start.)

Launch (maddy logs to stderr by default — `internal/log/log.go:202` `var DefaultLogger = Logger{Out: WriterOutput(os.Stderr, false)}`; the `-debug` flag is `flag.BoolVar(&log.DefaultLogger.Debug, "debug", false, "enable debug logging early")` at `maddy.go:104`). At startup exactly two lines are printed; the `220` greeting is emitted per-connection (seen in O1's transcript):

```console
$ /tmp/maddy_investigation/maddy_bin -debug -config /tmp/maddy_investigation/maddy.conf
smtp: I/O debugging is on! It may leak passwords in logs, be careful!
smtp: listening on tcp://127.0.0.1:2525
```

### 1.3 A key, non-obvious finding: the wire transcript is gated by **both** `io_debug` **and** `debug`

The go-smtp wire-transcript tee is wired only when the **`io_debug`** directive is set:

```go
// internal/endpoint/smtp/smtp.go
565:	cfg.Bool("io_debug", false, false, &ioDebug)
566:	cfg.Bool("debug", true, false, &endp.Log.Debug)
// (lines 567–602 omitted)
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

The writer maddy installs is consumed on the go-smtp side in the connection's `init()`: when `c.server.Debug != nil`, go-smtp tees every read and write through it — `io.TeeReader(rwc.Reader, c.server.Debug)` / `io.MultiWriter(rwc.Writer, c.server.Debug)` (go-smtp `conn.go:62-71`) — before wrapping the stream in `textproto.NewConn(rwc)` (`conn.go:74`), whose `DotReader()` performs the end-of-DATA decision. (The same `init()` first wraps the raw connection in `lineLimitReader{R: c.conn, LineLimit: c.server.MaxLineLength}` at `conn.go:54-57` — relevant to the over-length-line behavior in O2.4.) So the protocol transcript surfaces **only when `io_debug` wires the tee (maddy side) AND `debug` (or the `-debug` CLI flag) makes `DebugWriter()` non-discarding.** This was verified empirically. With a `debug`-only config (no `io_debug`), the same conformant message produced structured logs but **zero wire bytes**:

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

The go-smtp **client** and `internal/testutils` (`internal/endpoint/smtp/smtp_test.go`, `internal/testutils/smtp_server.go`) normalize line endings, which would **mask the very framing under study**. All probes therefore used a raw `socket` client that writes exact bytes and sends `EHLO probe.local` (hence `src_host":"probe.local"` in every structured log below). Its framing matrix (`scripts/probe.py`):

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

After sending the payload the probe reads the post-DATA response line, prints it verbatim with the elapsed time, then issues `QUIT` (so accepting variants do not wait for the idle timeout). A companion `scripts/capture_server.py` acts as the queue's downstream target: in **defer** mode it answers with a temporary `451` so maddy's queue classifies the delivery as a **temporary failure** and **retains** the `<id>.body` file for inspection (per the disk-removal condition at `queue.go:390`).

Restarting maddy fresh and wiping the queue between variants (so exactly one delivered body is present per capture) is done by `scripts/fresh.sh`, which kills only the previously recorded maddy PID, deletes `state`/`runtime`/`queue_dbg`, relaunches maddy with `-debug`, and waits for the `listening on tcp://127.0.0.1:2525` line:

```bash
# usage: scripts/fresh.sh LOGFILE   (exact-PID management; never a broad pkill)
kill "$(cat "$ROOT/maddy.pid")" 2>/dev/null            # stop prior instance
rm -rf "$ROOT/state" "$ROOT/runtime" "$ROOT/queue_dbg"  # wipe state + delivered bodies
mkdir -p "$ROOT/state" "$ROOT/runtime" "$ROOT/queue_dbg"
nohup "$ROOT/maddy_bin" -config "$ROOT/maddy.conf" -debug > "$1" 2>&1 &
echo $! > "$ROOT/maddy.pid"
```

The O5 front proxy (`scripts/proxy.py`) sits between client and maddy. maddy has **no native PROXY-protocol / framing-normalization support** — a repo-wide search for `proxy_protocol`/`proxyproto` returns **zero** matches — so the proxy is a throwaway external tool with two modes:

```python
# mode "clean": normalize every lone CR and lone LF to CRLF
elif c == 0x0A:               # bare LF
    out += b"\r\n"            # covers CRLF and lone LF
# mode "block": reject a connection that carries a bare newline
if find_bare_newline(chunk, state):
    client.sendall(b"521 5.5.2 bare newline rejected (proxy block mode)\r\n")
    server.shutdown(socket.SHUT_WR)   # signal EOF to maddy mid-DATA
    break
```

---

## O1 — The boundary moment: what you see at the instant DATA ends

**Command.** Conformant control payload — body `This is the body.` terminated by `<CR><LF>.<CR><LF>` — full transcript captured with `io_debug`+`debug`:

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

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"fb790a5c"}
smtp: accepted	{"msg_id":"fb790a5c"}
smtp: 250 2.0.0 OK: queued

[debug] smtp: reset
```

**Surface 2 — the structured log markers** (msg_id assigned at RCPT, then `accepted`):

```
smtp: incoming message	{"msg_id":"fb790a5c","sender":"sender@test.local","src_host":"probe.local","src_ip":"127.0.0.1:37550"}
smtp: RCPT ok	{"msg_id":"fb790a5c","rcpt":"rcpt@test.local"}
smtp: accepted	{"msg_id":"fb790a5c"}
```

**Surface 3 — the delivered body on disk** (`<id>.body`, 18 bytes):

```console
$ od -c queue_dbg/fb790a5c.body
0000000   T   h   i   s       i   s       t   h   e       b   o   d   y
0000020   .  \n
0000022
$ wc -c queue_dbg/fb790a5c.body
18 queue_dbg/fb790a5c.body
```

**What is observed at the boundary.** The exact literals:

- the DATA go-ahead prompt is `354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>` — matching go-smtp `conn.go:510` `c.WriteResponse(354, EnhancedCode{2, 0, 0}, "Go ahead. End your data with <CR><LF>.<CR><LF>")`;
- the post-DATA status line is `250 2.0.0 OK: queued`;
- the debug marker `[debug] smtp: reset` (emitted from maddy's `Reset()` handler at `smtp.go:64`) is the observable instant the connection returns to command parsing — it appears **after** `accepted` and the `250`.

**Reasoning (mechanism, tied to code).** go-smtp's `handleData` (go-smtp `conn.go`) does, in order: write the `354` (`conn.go:510`), `defer c.reset()` (`conn.go:512`), build the DATA reader `r := newDataReader(c)` (`conn.go:519`), call the backend `c.Session().Data(r)` (`conn.go:520`), and then **`io.Copy(ioutil.Discard, r)`** (`conn.go:521`, commented `// Make sure all the data has been consumed`) to drain any bytes the backend did not read up to the boundary, before the deferred `reset()` fires and the command loop resumes. On maddy's side, `func (s *Session) Data(r io.Reader) error` (`smtp.go:312`) reads the header with `textproto.ReadHeader(bufr)` (`smtp.go:285`, inside `prepareBody` at `smtp.go:283`), buffers the body with `buffer.BufferInMemory(bufr)` (`smtp.go:298`), delivers via `s.delivery.Body(bodyCtx, header, buf)` (`smtp.go:326`) / `s.delivery.Commit(bodyCtx)` (`smtp.go:330`), and logs `s.log.Msg("accepted", "msg_id", s.msgMeta.ID)` (`smtp.go:334`). The `354` prompt advertises the conformant terminator `<CR><LF>.<CR><LF>`, but what the reader actually treats as the terminator is decided one layer down (see O2/O3). Note the server also advertises `250 SIZE 33554432` in the EHLO reply — `33554432 = 32*1024*1024`, the `max_message_size` default (`smtp.go:561`).

---

## O2 — Identical-except-for-framing payloads: how the three surfaces diverge (or don't)

The body content is fixed (`This is the body.`); only the DATA-boundary framing differs. This section walks **every** framing variant across the **three observable surfaces** — (1) the SMTP response on the wire, (2) the `io_debug` debug transcript / structured log, and (3) the delivered `<id>.body` bytes (or, for failures, explicit no-body / queue-empty evidence). Each variant was run against a **freshly restarted maddy with a wiped queue** — the exact restart command per variant is `bash scripts/fresh.sh captures/O2_wire_$v.log` with `$v` ranging over `crlf lflf crcr lfcrlf dotstuff longline purelf` (the concrete log files are `captures/O2_wire_crlf.log`, `captures/O2_wire_lflf.log`, and so on) — so exactly one delivered body is attributable to each.

### O2 summary matrix

| Variant | Payload boundary | Surface 1: SMTP response | Surface 3: delivered `<id>.body` | msg_id (this run) |
|---|---|---|---|---|
| `crlf` (control) | `…<CR><LF>.<CR><LF>` | `250 2.0.0 OK: queued` | 18 B — `This is the body.\n` — SHA `5416a9e2…` | `165e427e` |
| `lflf` | `…<CR><LF>.<LF>` | `250 2.0.0 OK: queued` | 18 B — SHA `5416a9e2…` (**identical**) | `ba22525a` |
| `lfcrlf` | `…<LF>.<CR><LF>` | `250 2.0.0 OK: queued` | 18 B — SHA `5416a9e2…` (**identical**) | `dd3d0e54` |
| `purelf` | `…<LF>.<LF>` (no CR anywhere) | `250 2.0.0 OK: queued` | 18 B — SHA `5416a9e2…` (**identical**) | `92e2470e` |
| `dotstuff` | `..text` line, `<CR><LF>.<CR><LF>` | `250 2.0.0 OK: queued` | 32 B — `.hidden leading dot\nnormal line\n` — SHA `827b8731…` | `8fcba1db` |
| `crcr` | `…<CR><LF>.<CR>` (bare CR) | `451 4.0.0 Internal server error` after 5.00 s | **no body — queue empty** | `3a6263b9` |
| `longline` N=2100 | over-`2000` line, `<CR><LF>.<CR><LF>` | `451 4.0.0 Internal server error` (i/o timeout) | **no body — queue empty** | `20cffde0` |
| `longline` N=50000 | over-`2000` line, `<CR><LF>.<CR><LF>` | `554 5.0.0 Internal server error` + `500 5.4.0 Too long line, closing connection` | **no body — queue empty** | `07dc92fa` |
| over-size | body > 32 MiB, `<CR><LF>.<CR><LF>` | `552 5.3.4 Maximum message size exceeded` | **no body — queue empty** | `ac4c99fc` |

The per-variant evidence for all three surfaces follows.

### O2.1 — `crlf` (conformant control), `lflf`, `lfcrlf`, `purelf`: byte-identical delivery

**Surface 1 (wire) + Surface 2 (transcript).** Each of the four accepting framings returns `250 2.0.0 OK: queued` and logs the same three markers. For example `lflf` (bare `<LF>.<LF>`), which a strict peer would reject:

```console
$ python3 scripts/probe.py lflf 2525
<<< [data] b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
>>> [payload scenario=lflf len=46] b'Subject: Framing test\r\n\r\nThis is the body.\r\n.\n'
<<< [post-data-response after 0.05s] b'250 2.0.0 OK: queued\r\n'
<<< [quit] b'221 2.0.0 Goodnight and good luck\r\n'
```

Its transcript markers (`captures/O2_wire_lflf.log`):

```
smtp: incoming message	{"msg_id":"ba22525a","sender":"sender@test.local","src_host":"probe.local","src_ip":"127.0.0.1:33040"}
smtp: accepted	{"msg_id":"ba22525a"}
smtp: 250 2.0.0 OK: queued
[debug] smtp: reset
```

**Surface 3 (delivered bytes) — the byte-identity proof.** Restarting fresh and hashing the delivered body for each accepting framing:

This single loop is the authoritative capture for the four accepting variants: for each it restarts maddy fresh (writing the per-variant wire log `captures/O2_wire_$v.log` that O2.1's transcript excerpt and O6's greps quote), sends the payload (client-side output saved to `captures/O2_client_$v.log`), then preserves the delivered body under a stable name (`captures/body_$v`) **before** the next `fresh` wipes the queue, and hashes it:

```console
$ for v in crlf lflf lfcrlf purelf; do
    bash scripts/fresh.sh "captures/O2_wire_${v}.log" >/dev/null
    python3 scripts/probe.py "$v" 2525 > "captures/O2_client_${v}.log" 2>&1
    bf=$(ls queue_dbg/*.body | head -1)
    cp "$bf" "captures/body_${v}"          # preserve before the next fresh wipes the queue
    printf "%-8s | %-8s | bytes=%s | sha256=%s\n" "$v" "$(basename "$bf" .body)" \
        "$(wc -c < "$bf")" "$(sha256sum "$bf" | cut -d' ' -f1)"
  done
crlf     | 165e427e | bytes=18 | sha256=5416a9e2ef754dd0d354b8e474df6cf24a71e3072cef39635e6b9197e2b8b438
lflf     | ba22525a | bytes=18 | sha256=5416a9e2ef754dd0d354b8e474df6cf24a71e3072cef39635e6b9197e2b8b438
lfcrlf   | dd3d0e54 | bytes=18 | sha256=5416a9e2ef754dd0d354b8e474df6cf24a71e3072cef39635e6b9197e2b8b438
purelf   | 92e2470e | bytes=18 | sha256=5416a9e2ef754dd0d354b8e474df6cf24a71e3072cef39635e6b9197e2b8b438
```

All four are **byte-identical** — 18 bytes, SHA-256 `5416a9e2ef754dd0d354b8e474df6cf24a71e3072cef39635e6b9197e2b8b438`. The delivered `crlf` body (preserved as `captures/body_crlf`) verbatim:

```console
$ od -c captures/body_crlf
0000000   T   h   i   s       i   s       t   h   e       b   o   d   y
0000020   .  \n
0000022
```

**This is the crux of O2:** a payload terminated with a **bare `<LF>.<LF>`** (`lflf`), or with a bare-LF last content line (`lfcrlf`), or with **no `<CR>` anywhere** (`purelf`), produces a delivered body **indistinguishable at the byte level** from the conformant `<CR><LF>.<CR><LF>` control — same SMTP response, same structured markers, same bytes. The body ends in a bare `\n` in every case (the `\r` was rewritten away — see reasoning in O3).

### O2.2 — `dotstuff` (`..text` → `.text`)

**Surface 1 (wire):** `250 2.0.0 OK: queued`. **Surface 3 (delivered body, 32 bytes):**

```console
$ od -c queue_dbg/8fcba1db.body
0000000   .   h   i   d   d   e   n       l   e   a   d   i   n   g    
0000020   d   o   t  \n   n   o   r   m   a   l       l   i   n   e  \n
0000040
$ wc -c queue_dbg/8fcba1db.body
32 queue_dbg/8fcba1db.body
$ sha256sum queue_dbg/8fcba1db.body
827b8731fd4404a9bbf61ed2603e71697e6b4e21d074b45e7d003851564558c8  queue_dbg/8fcba1db.body
```

The leading `..` was **unstuffed to a single `.`** and each `\r\n` was rewritten to `\n`. Delivered: `.hidden leading dot\nnormal line\n` (32 bytes, SHA-256 `827b8731…`), distinct from the 18-byte control — proof that content, not framing, drives the delivered bytes.

### O2.3 — `crcr` (bare `<CR>.<CR>`): non-terminating, no body

A bare `<CR>` is **not** a line terminator, so `.\r` (no `<LF>`) is never recognized as end-of-DATA. **Surface 1 (wire):** after the full `read_timeout` (5 s), `451`:

```console
$ python3 scripts/probe.py crcr 2525
>>> [payload scenario=crcr len=46] b'Subject: Framing test\r\n\r\nThis is the body.\r\n.\r'
<<< [post-data-response after 5.00s] b'451 4.0.0 Internal server error (msg ID = 3a6263b9)\r\n'
<<< [quit] b'221 2.0.0 Goodnight and good luck\r\n'
```

**Surface 2 (transcript):**

```
smtp: incoming message	{"msg_id":"3a6263b9","sender":"sender@test.local","src_host":"probe.local","src_ip":"127.0.0.1:33078"}
smtp: DATA error	{"msg_id":"3a6263b9","reason":"read tcp 127.0.0.1:2525->127.0.0.1:33078: i/o timeout"}
smtp: aborted	{"msg_id":"3a6263b9"}
[debug] smtp: reset
```

**Surface 3 (delivered body): none.** The queue directory is empty:

```console
$ ls queue_dbg/ | wc -l
0
```

Mechanism is in O4(b)/O6: the `net/textproto` machine only re-enters begin-of-line on a real `'\n'`, so the `.` after a lone `<CR>` is ordinary body data; the reader blocks awaiting more bytes until `read_timeout` fires with `i/o timeout`.

### O2.4 — Over-length line (`MaxLineLength` default `2000`): behavior depends on delivery framing

maddy does not override go-smtp's `MaxLineLength` (default `2000` at go-smtp `server.go:76`, with the preceding comment *"Doubled maximum line length per RFC 5321 (Section 4.5.3.1.6)"* at `server.go:75`; field declared `server.go:40`). **Threshold, pinned by experiment** — invoked as `python3 scripts/probe_longline.py N 2525`, which sends a single content line of `N` bytes followed by `<CR><LF>.<CR><LF>`, run for `N` in `1998 1999 2000 2001`:

```
N=1998 -> 250 2.0.0 OK: queued        (1 body delivered)
N=1999 -> DATA error -> 451           (0 bodies)
N=2000 -> DATA error -> 451           (0 bodies)
N=2001 -> DATA error -> 451           (0 bodies)
```

So the accept/reject boundary sits at `N=1998` (accepted) vs `N=1999` (rejected). But the **error that surfaces depends on how the over-limit bytes arrive**, because the limiter's `Read` has a **value receiver** — `func (r lineLimitReader) Read(b []byte)` (go-smtp `lengthlimit_reader.go:21`) — so `curLineLength` **does not persist across separate underlying `Read` calls**; the limit only trips within a single read, and when it trips it returns `(0, ErrTooLongLine)` (`lengthlimit_reader.go:22-23,41-42`), discarding the bytes it already consumed from the connection. Two observed outcomes:

**(a) small over-limit line (N=2100, delivered in one ~2 KB TCP segment) → i/o timeout → `451`, no body.** The tripping read discards the whole chunk — *including the trailing `.\r\n` terminator* — so the dot reader never sees end-of-DATA and blocks until `read_timeout`:

```console
$ python3 scripts/probe_longline.py 2100 2525
<<< [post-data] b'451 4.0.0 Internal server error (msg ID = 20cffde0)\r\n'
<<< [post-data] b'221 2.4.2 Idle timeout, bye bye\r\n'
```
```
smtp: DATA error	{"msg_id":"20cffde0","reason":"read tcp 127.0.0.1:2525->127.0.0.1:41810: i/o timeout"}
smtp: aborted	{"msg_id":"20cffde0"}
[debug] smtp: reset
smtp: 221 2.4.2 Idle timeout, bye bye
```

Direct corroboration that the over-limit bytes were **discarded, not teed**: only 7 stray `A` bytes (from unrelated text) appear in the whole debug transcript — not the 2100 that were sent — and the `.\r\n` terminator bytes never appear:

```console
$ python3 -c "d=open('captures/O2_wire_longline2100.log','rb').read(); print('A bytes teed:', d.count(b'A'), '| .CRLF present:', b'.\r\n' in d)"
A bytes teed: 7 | .CRLF present: False
```
```console
$ ls queue_dbg/ | wc -l
0
```

**(b) large over-limit line (N=50000, spanning many 4096-byte buffer fills of pure `A`) → `ErrTooLongLine` → `554`, no body.** Here a single read returns >2000 non-newline bytes, so the limit trips cleanly and the exact error literal surfaces:

```console
$ python3 scripts/probe_longline.py 50000 2525
<<< [post-data] b'554 5.0.0 Internal server error (msg ID = 07dc92fa)\r\n'
<<< [post-data] b'500 5.4.0 Too long line, closing connection\r\n'
```
```
smtp: DATA error	{"msg_id":"07dc92fa","reason":"smtp: too longer line in input stream"}
smtp: 554 5.0.0 Internal server error (msg ID = 07dc92fa)
smtp: 500 5.4.0 Too long line, closing connection
```
```console
$ ls queue_dbg/ | wc -l
0
```

The reason literal is exactly `smtp: too longer line in input stream` — go-smtp `lengthlimit_reader.go:8` `var ErrTooLongLine = errors.New("smtp: too longer line in input stream")`. After the DATA abort resets the connection, the command reader hits the same over-limit condition and go-smtp writes `500 5.4.0 Too long line, closing connection` and closes (go-smtp `server.go:154-156` `if err == ErrTooLongLine { c.WriteResponse(500, EnhancedCode{5, 4, 0}, "Too long line, closing connection") }`). The command-phase net timeout path, by contrast, writes `221 2.4.2 Idle timeout, bye bye` (`server.go:159-162`) — which is the trailing line seen in case (a).

**Net finding:** an over-length line is always rejected with **no delivered body**, but the surfaced error is **not stable** — small over-limit lines fail as an `i/o timeout` (`451`), large ones as `ErrTooLongLine` (`554` + `500`). This unreliability is a direct consequence of the value-receiver `lineLimitReader` (`lengthlimit_reader.go:21`).

### O2.5 — Message size limit (`max_message_size` default 32 MiB)

**Command:** stream `1000×'A'` lines past 32 MiB (`scripts/probe_size.py`). **Surface 1 (wire) + Surface 3 (no body):**

```console
$ python3 scripts/probe_size.py 2525
>>> [streamed approx 37424716 body bytes (35.69 MiB)]
<<< [post-data] b'552 5.3.4 Maximum message size exceeded (msg ID = ac4c99fc)\r\n'
<<< [post-data] b'221 2.4.2 Idle timeout, bye bye\r\n'
$ ls queue_dbg/ | wc -l
0
```

**Surface 2 (transcript):**

```
smtp: incoming message	{"msg_id":"ac4c99fc","sender":"sender@test.local","src_host":"probe.local","src_ip":"127.0.0.1:49410"}
smtp: DATA error	{"msg_id":"ac4c99fc","reason":"Maximum message size exceeded"}
smtp: 552 5.3.4 Maximum message size exceeded (msg ID = ac4c99fc)
smtp: aborted	{"msg_id":"ac4c99fc"}
```

The status is exactly `552 5.3.4 Maximum message size exceeded` — go-smtp `data.go:38-41` `var ErrDataTooLarge = &SMTPError{ Code: 552, EnhancedCode: EnhancedCode{5, 3, 4}, Message: "Maximum message size exceeded" }`, returned from the data reader at `data.go:67`. The `250 SIZE 33554432` advertised in EHLO equals the 32 MiB default (`smtp.go:561` `cfg.DataSize("max_message_size", false, false, 32*1024*1024, &endp.serv.MaxMessageBytes)`).

---

## O3 — Pipelined pressure / SMTP smuggling: bytes a stricter peer would keep in the body

**Command.** One connection. Message #1's DATA is terminated with a **bare `<LF>.<LF>`**; immediately after (same `sendall`, pipelined) comes a second `MAIL FROM` / `RCPT TO` / `DATA` sequence with a **forged** sender:

```console
$ python3 scripts/probe_smuggle.py 2525
```

The 168-byte blast sent on the wire (verbatim `repr`, note `\n.\n` mid-stream then trailing commands):

```
>>> [blast len=168] b'Subject: First message\r\n\r\nLegitimate body line\n.\nMAIL FROM:<attacker@evil.test>\r\nRCPT TO:<victim@test.local>\r\nDATA\r\nSubject: SMUGGLED SPOOF\r\n\r\nSpoofed body content\r\n.\r\n'
```

**Surface 1 — the client received six response lines after the single blast** — the five SMTP command-responses prove the trailing bytes were parsed as new commands (`250` for message #1, then `250`/`250`/`354`/`250` for the smuggled `MAIL`/`RCPT`/`DATA`/end-of-data), followed by the idle-timeout close:

```
<<< [all-post-data-responses] b"250 2.0.0 OK: queued\r\n250 2.0.0 Roger, accepting mail from <attacker@evil.test>\r\n250 2.0.0 I'll make sure <victim@test.local> gets this\r\n354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n250 2.0.0 OK: queued\r\n221 2.4.2 Idle timeout, bye bye\r\n"
```

**Surface 2 — TWO `incoming message` events on the SAME connection** (identical `src_ip 127.0.0.1:34712`), the second with the forged sender:

```
smtp: incoming message	{"msg_id":"48089fe5","sender":"sender@test.local","src_host":"probe.local","src_ip":"127.0.0.1:34712"}
smtp: incoming message	{"msg_id":"2abdd736","sender":"attacker@evil.test","src_host":"probe.local","src_ip":"127.0.0.1:34712"}
```

**Surface 3 — TWO delivered bodies, with distinct envelopes** (same run; the `.meta` JSON records the envelope sender at `MsgMeta.OriginalFrom`):

```console
$ for m in queue_dbg/*.meta; do
    id=$(basename "$m" .meta)
    from=$(python3 -c "import json,sys;print(json.load(open(sys.argv[1]))['MsgMeta']['OriginalFrom'])" "$m")
    echo "id=$id From=$from bytes=$(wc -c < "queue_dbg/$id.body")"
    od -c "queue_dbg/$id.body"
  done
id=48089fe5 From=sender@test.local bytes=21
0000000   L   e   g   i   t   i   m   a   t   e       b   o   d   y    
0000020   l   i   n   e  \n
0000025
id=2abdd736 From=attacker@evil.test bytes=21
0000000   S   p   o   o   f   e   d       b   o   d   y       c   o   n
0000020   t   e   n   t  \n
0000025
```

**What this proves.** A single connection carrying one pipelined blast produced **two accepted messages**: the legitimate `From=sender@test.local` (`msg_id=48089fe5`) and a **smuggled** `From=attacker@evil.test`, `To=victim@test.local` (`msg_id=2abdd736`). The bytes after the bare-LF dot — which an RFC-5321-strict, `<CR><LF>.<CR><LF>`-only peer would have kept **inside the first message's body** — were instead parsed by maddy as fresh SMTP commands.

**Reasoning (mechanism, tied to code).** maddy's DATA reader is delegated: go-smtp builds it as `r: c.text.DotReader()` (go-smtp `data.go:53`), i.e. Go's `net/textproto` dot reader (`/usr/local/go/src/net/textproto/reader.go:299`). That state machine (`func (d *dotReader) Read` at `reader.go:311`, commented `elide leading dots, rewrite trailing \r\n into \n, and detect ending .\r\n line` at `reader.go:313`) transitions on a lone `'\n'` at the start of a line: from `stateDot` on `'\n'` it goes to `stateEOF` (`reader.go:351`) — i.e. a bare `<LF>.<LF>` is accepted as end-of-DATA exactly like `<CR><LF>.<CR><LF>` (`stateDotCR` + `'\n'` → `stateEOF` at `reader.go:358`). When `Data()` returns, go-smtp's `defer c.reset()` (`conn.go:512`) restores the command loop, and the still-buffered trailing bytes are read as the next command — the smuggling surface. This is the SMTP-smuggling class (CVE-2023-51764 and the related Sendmail/Exim CVE-2023-51765/51766): a parsing differential where a receiver accepts bare-`<LF>` end-of-data that RFC 5321 §4.5.2 does not sanction, enabling a spoofed `MAIL FROM` that bypasses SPF alignment. **This is analysis only — no fix is prescribed.**

---

## O4 — Back-to-back stability, and the failure mode when framing never terminates

### O4(a) — Several well-formed messages on one connection

**Command:** four conformant messages back-to-back on a single connection, with client-side per-message timing (`scripts/probe_backtoback.py`):

```console
$ python3 scripts/probe_backtoback.py 2525 4
msg=1 elapsed=46.5ms resp=b'250 2.0.0 OK: queued\r\n'
msg=2 elapsed=50.7ms resp=b'250 2.0.0 OK: queued\r\n'
msg=3 elapsed=55.2ms resp=b'250 2.0.0 OK: queued\r\n'
msg=4 elapsed=59.7ms resp=b'250 2.0.0 OK: queued\r\n'
```

**Server side — four distinct msg_ids (in arrival order) and four correct distinct bodies:**

```
msg_id":"39ad1d4b"   -> body "Back-to-back body 1"  (20 bytes)
msg_id":"b51c859e"   -> body "Back-to-back body 2"  (20 bytes)
msg_id":"9a7bad41"   -> body "Back-to-back body 3"  (20 bytes)
msg_id":"48b70610"   -> body "Back-to-back body 4"  (20 bytes)
```

One delivered body verbatim (message #4):

```console
$ od -c queue_dbg/48b70610.body
0000000   B   a   c   k   -   t   o   -   b   a   c   k       b   o   d
0000020   y       4  \n
0000024
```

**Observation.** The boundary decision does **not** wobble across back-to-back messages: each message got its own `250 2.0.0 OK: queued`, a fresh distinct `msg_id`, and its own correct 20-byte body. The client-side per-message elapsed times are tight and monotonic (46.5 → 50.7 → 55.2 → 59.7 ms cumulative wall-clock from the start of the back-to-back send loop, a per-message spacing of ~4–5 ms) — consistent, no drift. These are **client** round-trip windows, quoted to show consistency, not to measure server latency.

**Reasoning.** Each message body is **fully buffered in memory** before pipeline hand-off: `buffer.BufferInMemory(bufr)` (`internal/buffer/memory.go:27`, called at `smtp.go:298`) reads the whole body into memory, after which `s.endp.pipeline.Start(mailCtx, msgMeta, cleanFrom)` (`smtp.go:148`) hands it to `internal/msgpipeline`. Each message's id is generated independently (8 hex chars from `crypto/rand`, `internal/msgpipeline/msgid.go:12-16`), so ids stay distinct. Between messages, go-smtp's `defer c.reset()` (`conn.go:512`) plus maddy's `[debug] smtp: reset` (`smtp.go:64`) cleanly re-arm the command loop.

### O4(b) — Non-terminating framing: bare `<CR>.<CR>` hangs until the read timeout

A bare `<CR>` is **not** a line terminator, so `.\r` (no `<LF>`) is never recognized as end-of-DATA; the reader keeps waiting. With `read_timeout 5s`, this is observed quickly.

**Command:** send the body `This is the body.` terminated by `\r\n.\r` (a bare `<CR>` with no final `<LF>`), then hold the connection open, sending nothing further (`scripts/probe_hang.py`):

```console
$ python3 scripts/probe_hang.py 2525
<<< [data] b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
elapsed=10.00s closed_by_server=True data=b'451 4.0.0 Internal server error (msg ID = 398b60fb)\r\n221 2.4.2 Idle timeout, bye bye\r\n'
```

**Server side:**

```
smtp: incoming message	{"msg_id":"398b60fb","sender":"sender@test.local","src_host":"probe.local","src_ip":"127.0.0.1:35068"}
smtp: DATA error	{"msg_id":"398b60fb","reason":"read tcp 127.0.0.1:2525->127.0.0.1:35068: i/o timeout"}
smtp: 451 4.0.0 Internal server error (msg ID = 398b60fb)
smtp: aborted	{"msg_id":"398b60fb"}
[debug] smtp: reset
smtp: 221 2.4.2 Idle timeout, bye bye
```

**Observation & reasoning.** With no valid boundary, the server blocked in the DATA read for the full `read_timeout` (5 s) and then failed the read with `i/o timeout`, returning `451 4.0.0 Internal server error`. It then returned to the command loop, waited another 5 s idle, and disconnected with `221 2.4.2 Idle timeout, bye bye` — the client therefore measured `elapsed=10.00s` end-to-end (two sequential 5 s timeouts). maddy logs `DATA error` (`smtp.go:317`) and `aborted` (`smtp.go:72`); the idle-timeout close is go-smtp `server.go:159-162`. The bare `<CR>` never reaches the dot state at begin-of-line because the `net/textproto` machine only enters `stateBeginLine` after a real `'\n'` (`stateCR` advances to `stateBeginLine` only on `'\n'` — `reader.go:367`), so the `.` after a lone `<CR>` is treated as ordinary body data, not a terminator.

A closely related variant (used in O5): when the probe sends the bare-CR-framed body and then **closes** the socket (rather than holding it open), the read ends in `unexpected EOF` instead of a timeout — `smtp: DATA error {"reason":"unexpected EOF"}` → `554 5.0.0 Internal server error` → `aborted`. Either way, the non-terminating framing never produces a delivered body.

---

## O5 — Front proxy: how "clean" vs "block" changes the story

The same smuggling blast (O3) was run three ways: **direct** to maddy (baseline), through the **clean** proxy (rewrites bare `CR`/`LF` → `CRLF`), and through the **block** proxy (rejects a connection carrying a bare newline). Delivered-message count is the decisive surface.

**Commands:**

```console
# baseline (direct to maddy)
$ python3 scripts/probe_smuggle.py 2525
# clean proxy in front (client -> 2526 -> maddy 2525)
$ python3 scripts/proxy.py clean 2526 2525 &
$ python3 scripts/probe_smuggle.py 2526
# block proxy in front
$ python3 scripts/proxy.py block 2526 2525 &
$ python3 scripts/probe_smuggle.py 2526
```

**Observed delivered-message counts** (via `ls queue_dbg/*.body | wc -l` and the count of `incoming message` events, after a fresh restart before each run):

```
DIRECT delivered: 2   ("From":"sender@test.local"  +  "From":"attacker@evil.test")
CLEAN  delivered: 2   ("From":"sender@test.local"  +  "From":"attacker@evil.test")
BLOCK  delivered: 0
```

**DIRECT** and **CLEAN** — the client received the same five SMTP command-responses (two `250 2.0.0 OK: queued`, i.e. two messages accepted):

```
b"250 2.0.0 OK: queued\r\n250 2.0.0 Roger, accepting mail from <attacker@evil.test>\r\n250 2.0.0 I'll make sure <victim@test.local> gets this\r\n354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n250 2.0.0 OK: queued\r\n"
```

**BLOCK** — the client received the proxy's synthetic rejection followed by maddy aborting the truncated message:

```
b'521 5.5.2 bare newline rejected (proxy block mode)\r\n554 5.0.0 Internal server error (msg ID = 5884dd78)\r\n'
```

…and on maddy's side (the proxy dropped the connection mid-DATA, so maddy hit EOF):

```
smtp: incoming message	{"msg_id":"5884dd78","sender":"sender@test.local","src_host":"probe.local","src_ip":"127.0.0.1:35534"}
smtp: DATA error	{"msg_id":"5884dd78","reason":"unexpected EOF"}
smtp: aborted	{"msg_id":"5884dd78"}
[debug] smtp: reset
```

**Reasoning — the important nuance.** The **block** proxy is the only mode that prevents the smuggle: it detects the bare newline before maddy sees the body, returns `521 5.5.2 bare newline rejected (proxy block mode)`, and drops the connection, so **zero** messages are delivered (the `554` the client also sees is maddy reacting to the truncated stream — `unexpected EOF` — not a delivery). The **clean** proxy does **not** help — and is arguably worse: by rewriting the client's ambiguous `\n.\n` into a conformant `\r\n.\r\n`, it hands maddy an **explicit, unambiguous** end-of-DATA, so the second (spoofed) message is still delivered (`CLEAN delivered: 2`). This is exactly the documented SMTP-smuggling nuance: a gateway that *normalizes* bare newlines can *manufacture* a clean boundary that even a strict backend would honor, which is why the Postfix mitigation for CVE-2023-51764 offers an explicit *reject* option (`smtpd_forbid_bare_newline`) rather than relying on normalization alone. maddy itself has no such control (no `proxy_protocol`/`proxyproto`; no bare-newline directive) — the proxy is entirely external. **Analysis only; no remediation prescribed.**

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
smtp: DATA error	{"msg_id":"5e9e7dea","reason":"read tcp 127.0.0.1:2525->127.0.0.1:45628: i/o timeout"}
smtp: 451 4.0.0 Internal server error (msg ID = 5e9e7dea)
smtp: aborted	{"msg_id":"5e9e7dea"}
```

**What lingers in memory: nothing observable.** The body is buffered in memory (`buffer.BufferInMemory`, `smtp.go:298`); on a failed/short read it is never committed, and go-smtp drains any unread bytes with `io.Copy(ioutil.Discard, r)` (`conn.go:521`) and resets via `defer c.reset()` (`conn.go:512`). The observable end-of-transaction signal is the `[debug] smtp: reset` line (`smtp.go:64`), which marks the return to command parsing on both success and failure.

**What you expect but never see: any "bare `<LF>` rejected" signal.** A search across the accept-variant wire logs for any conformance/bare-newline/rejection wording returns **zero** hits:

```console
$ grep -aEi "bare|conforman|rfc|reject|not.*(valid|conform)" \
      captures/O2_wire_crlf.log captures/O2_wire_lflf.log \
      captures/O2_wire_lfcrlf.log captures/O2_wire_purelf.log | wc -l
0
```

There is no code path in maddy or go-smtp that flags a bare `<LF>.<LF>` end-of-DATA — it is accepted **silently**. As direct corroboration, the structured log markers for the bare-LF (`lflf`) message are **identical** in count to the conformant (`crlf`) message:

```console
$ for v in crlf lflf; do
    echo "$v: incoming=$(grep -ac 'incoming message' captures/O2_wire_${v}.log) \
accepted=$(grep -ac 'smtp: accepted' captures/O2_wire_${v}.log) \
queued=$(grep -ac '250 2.0.0 OK: queued' captures/O2_wire_${v}.log) \
reset=$(grep -ac 'smtp: reset' captures/O2_wire_${v}.log)"
  done
crlf: incoming=1 accepted=1 queued=1 reset=1
lflf: incoming=1 accepted=1 queued=1 reset=1
```

**Reasoning.** Because the boundary decision is delegated to `net/textproto` (which accepts bare `<LF>.<LF>` — `reader.go:351`), neither maddy nor go-smtp ever *knows* the framing was non-conformant; there is nothing to log. The only failure-time output an operator sees is the generic `DATA error` / `aborted` pair (for reads that never terminate or are cut short) — never a framing-specific diagnostic. This absence is itself the security-relevant observation: the leniency is invisible in the logs.

**Explicitly labeled as source-grounded, not directly runtime-probed:** the exact bytes that `net/textproto` retains vs. emits internally are grounded in the source state machine (`reader.go:311-379`) and are corroborated by the delivered-body captures above (bare `\n` endings, dot-unstuffing, byte-identical bodies); the internal `stateEOF`/`stateData` transitions themselves are not observable at the wire and are cited from source rather than asserted from a runtime probe of the reader internals.

---

## Summary — the central finding

At commit `26452dd8dd787dc455278b0fdd296f4a5432c768`, maddy does **not** implement the end-of-DATA decision. Its SMTP endpoint delegates the DATA body to `github.com/emersion/go-smtp` (`func (s *Session) Data(r io.Reader)` at `smtp.go:312`), which constructs the reader as `r: c.text.DotReader()` (go-smtp `data.go:53`) — Go's standard-library `net/textproto` dot reader (`reader.go:299`). That state machine decides what terminates DATA, and it is lenient by RFC-5321 standards:

- **(a) bare `<LF>.<LF>` is a valid end-of-DATA.** Bodies delivered from `crlf`, `lflf`, `lfcrlf`, and `purelf` framings are **byte-identical** — 18 bytes, SHA-256 `5416a9e2ef754dd0d354b8e474df6cf24a71e3072cef39635e6b9197e2b8b438` — and the trailing bytes after a bare-LF dot are parsed as new commands, delivering a **second, spoofed** message on one connection (`From=attacker@evil.test`, `msg_id=2abdd736`).
- **(b) bare `<CR>.<CR>` is NOT a boundary.** Those bytes stay in the body and the reader blocks until `read_timeout` (`i/o timeout` → `451 4.0.0`), or ends in `unexpected EOF` if the client closes; nothing is delivered and nothing lingers on disk.
- **(c) dot-unstuffing is applied.** A leading `..` is delivered as a single `.` (`.hidden leading dot\nnormal line\n`, 32 bytes, SHA-256 `827b8731…`), and `\r\n` line endings are rewritten to `\n` in the delivered body.

Two additional runtime-observed limits bound the DATA phase: an over-length line (default `MaxLineLength 2000`, `server.go:76`) is always rejected with no body, though the surfaced error is **not stable** — small over-limit lines fail as `i/o timeout`/`451`, large ones as `ErrTooLongLine`/`554` (a consequence of the value-receiver `lineLimitReader`, `lengthlimit_reader.go:21`); and a body over 32 MiB (`max_message_size`, `smtp.go:561`) is rejected with `552 5.3.4 Maximum message size exceeded` (`data.go:38-41`).

This leniency is exactly the **SMTP-smuggling** condition (CVE-2023-51764 for Postfix, with sibling CVE-2023-51765/51766 for Sendmail/Exim): RFC 5321 §4.5.2 defines `<CR><LF>.<CR><LF>` as the end-of-mail indicator, and accepting a bare-`<LF>` equivalent creates a parsing differential that lets an attacker inject a spoofed `MAIL FROM` and bypass SPF alignment. An external **block**-mode proxy prevents it (0 delivered); a **clean**-mode proxy does not (it normalizes the ambiguous boundary into a conformant one and the spoof still lands). **This document is analysis only — no remediation is proposed — and every conclusion is scoped to this exact commit and its pinned dependency versions.**

---

## Coverage-pass checklist

| Sub-question | Answered with observed evidence | Key literals (with citation) |
|---|---|---|
| **O1** — the boundary moment | ✅ runtime-observed: full transcript (`354` prompt, body consumed, `250 2.0.0 OK: queued`, `[debug] smtp: reset`), 3 surfaces, delivered 18 B body | `354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>` (conn.go:510); `accepted` (smtp.go:334); reset (smtp.go:64) |
| **O2** — identical-except-for-framing | ✅ runtime-observed: all six variants × three surfaces — `crlf`/`lflf`/`lfcrlf`/`purelf` byte-identical 18 B (SHA `5416a9e2…`); `dotstuff` 32 B (SHA `827b8731…`); `crcr` no body (451/i-o-timeout); over-limit no body (`451` small / `554`+`500` large); over-size no body (`552 5.3.4`) | SHA `5416a9e2…`; `smtp: too longer line in input stream` (lengthlimit_reader.go:8) + value receiver (lengthlimit_reader.go:21); `MaxLineLength 2000` (server.go:76); `500 5.4.0 Too long line…` (server.go:154-156); `552 5.3.4` (data.go:38-41) |
| **O3** — pipelined smuggling | ✅ runtime-observed: one 168 B blast → 5 command-responses, 2 delivered messages, spoofed `attacker@evil.test` (`2abdd736`) on one `src_ip` | `c.text.DotReader()` (data.go:53); stateDot+`\n`→stateEOF (reader.go:351) |
| **O4** — back-to-back stability | ✅ runtime-observed: 4 distinct msg_ids, 4 correct 20 B bodies, ~4–5 ms/msg spacing; bare-CR hang → `i/o timeout`/`451` then idle `221` (elapsed 10.00 s) | `BufferInMemory` (memory.go:27, smtp.go:298); `read_timeout` 10 min default (smtp.go:560); `451 4.0.0` + `221 2.4.2 Idle timeout, bye bye` (server.go:159-162) |
| **O5** — front proxy | ✅ runtime-observed: DIRECT=2, CLEAN=2, BLOCK=0; `521 5.5.2 bare newline rejected` + maddy `unexpected EOF` → `554 5.0.0` | no native proxy support (repo grep `proxy_protocol`/`proxyproto` = 0) |
| **O6** — negative space | ✅ runtime-observed: 0 files linger after failure; 0 bare-LF rejections in any log; `lflf` markers ≡ `crlf` markers | `DATA error` (smtp.go:317); `aborted` (smtp.go:72); `io.Copy(ioutil.Discard, r)` (conn.go:521) |

**Items grounded in source rather than a direct runtime probe (explicitly labeled):** the internal `net/textproto` state transitions (`reader.go:311-379`) are cited from source; their *effects* (bare-LF acceptance, bare-CR non-termination, dot-unstuffing, `\r\n`→`\n`) are all confirmed by the observed delivered bytes and transcripts above. Everything else in the table — including the over-length-line limit, the 32 MiB size limit, and the smuggling delivery — **was** exercised at runtime and the quoted responses/logs/bytes are verbatim captures.

---

## Cleanup note

All observation artifacts lived outside the repository under `/tmp/maddy_investigation/` (the built `maddy_bin`, the minimal config, the `scripts/` probes and proxy, the `queue_dbg`/`state`/`runtime` directories, the capture server, and the captured logs — including a multi-MB size-limit stream log). These were deleted after evidence capture. The only change introduced to the repository is this document; `git status --porcelain` shows solely `blitzy/documentation/maddy_26452dd8dd78.md`, and `go.mod`/`go.sum` are unchanged.
