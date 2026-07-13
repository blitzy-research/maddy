# maddy inbound SMTP `DATA` boundary decision — a runtime-grounded investigation

**Branch:** `maddy_26452dd8dd78` (source), HEAD short `26452dd`
**Scope:** read-only investigation. The only file added to the repository is this document. No existing source file was modified; all observation scripts lived under `/tmp` and were removed on completion (see **Caveats**).

This document answers, from **live runtime observation** (not code reading alone), how maddy's inbound SMTP endpoint decides that a `DATA` message body has ended and resumes command parsing, and how ambiguous message-boundary framing (line endings, dot-stuffing, end-of-`DATA` sequences) changes that decision. Every behavioural claim is tagged **(observed)** — backed by captured output plus the exact command that produced it — or **(inferred)** — derived from reading code. Code claims carry `file:line` references; standards/advisory facts are attributed in prose.

The single most important finding, established below and reconfirmed at runtime: **maddy does not decide the `DATA` boundary itself.** The decision is made two layers down, in the Go standard library's `net/textproto` dot-reader finite-state machine. maddy receives an *already-decoded* body reader. That FSM leniently accepts a bare-LF `<LF>.<LF>` as end-of-mail-data, which is exactly the SMTP-smuggling condition, and maddy ships **no** bare-newline rejection or normalization knob.

---

## 1. Direct-answer summary

**Overall (observed).** At the byte level the end-of-`DATA` decision is taken by the Go stdlib `dotReader` state machine in `net/textproto/reader.go` (`dotReader.Read`, reached by go-smtp's `newDataReader` → `c.text.DotReader()` in `data.go:L53`). maddy's `Session.Data` (`internal/endpoint/smtp/smtp.go:L312`) only ever sees the decoded body; it never scans for the `.` terminator. Because that FSM treats both canonical `<CRLF>.<CRLF>` **and** bare-LF `<LF>.<LF>` as terminators, the framing an attacker chooses changes maddy's *acceptance decision* but not the *delivered bytes* (which are re-normalized to canonical CRLF on relay).

- **Q1 — the boundary-decision moment (observed).** When the terminator is consumed the `dotReader` reaches `stateEOF`, `Session.Data` returns, and go-smtp writes `250 2.0.0 OK: queued` and resets the envelope. The observable signals, in order, are: the `354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>` greeting; the terminator bytes in the `io_debug` transcript; the `smtp: accepted {"msg_id":...}` log line (`smtp.go:L334`); the `250 ... OK: queued` reply; and the message arriving at the downstream sink.
- **Q2 — framing-variation payloads (observed).** Canonical `<CRLF>.<CRLF>` and bare-LF `<LF>.<LF>` are **both accepted**, and their delivered bodies are **byte-identical** (same SHA-256) because `dotReader` rewrites `\r\n`→`\n` on decode and the `smtp_downstream` relay re-encodes to canonical CRLF. Dot-stuffed leading dots (`..`→`.`) are un-stuffed on decode and re-stuffed on relay; a lone-looking-but-not-lone dot line (` .`, `.text`, `embedded . dot`) does **not** terminate.
- **Q3 — pipelined pressure vs a stricter peer (observed).** Terminating the first transaction with a bare-LF `<LF>.<LF>` and pipelining a second `MAIL/RCPT/DATA` in one `send()` makes maddy deliver **two** messages — the second with a spoofed sender — where an RFC-5321-strict peer would treat `<LF>.<LF>` as body and deliver **one**. This is SMTP smuggling.
- **Q4 — back-to-back stability (observed).** The decision is deterministic. Ten identical canonical messages across two independent 5-message batches were all accepted with byte-identical delivered bodies; three back-to-back transactions on one connection all delivered, with an envelope reset between each. No run-to-run wobble was observed.
- **Q5 — a cleaning / blocking front proxy (observed).** A **normalize** proxy (bare `\n`→`\r\n`) does not stop the smuggle — it *manufactures* a canonical `<CRLF>.<CRLF>` boundary, so maddy still delivers two messages. A **reject** proxy (refuse bare `\n`) blocks the traffic upstream: the client gets a `500` from the proxy and maddy delivers nothing, while a fully-canonical message still passes.
- **Q6 — what lingers, and what is expected but never seen (observed).** A connection dropped mid-`DATA` yields `io.ErrUnexpectedEOF` → `DATA error {"reason":"unexpected EOF"}` → `554`, `aborted`, envelope cleared (nothing lingers, nothing delivered). An oversize message yields `552 5.3.4 Maximum message size exceeded`; an over-long line yields `DATA error {"reason":"smtp: too longer line in input stream"}` → `554`. The headline **absence**: maddy emits **no** bare-newline `5xx` rejection — a repository-wide search for any such control returns nothing — the RFC-conformant refusal that peers like Postfix added (`smtpd_forbid_bare_newline`) is exactly what maddy never exhibits.

---
## 2. Per-question findings (Q1–Q6)

For every experiment three observation points are captured: **(1)** the client-side transcript (raw bytes both directions, control chars made visible), **(2)** maddy's log — the `io_debug` raw wire transcript (each chunk prefixed `smtp: `) plus the structured `smtp: accepted` / `smtp: DATA error` lines, and **(3)** the delivered bytes at the capturing sink (post decode-then-re-encode), shown with visible control characters and a hexdump. The faithful code regions for all three layers are collected in **§3**; the questions below cite them by `file:line`.

### Q1 — The boundary-decision moment

**Mechanism (cause → effect).** The client sends the body then the end-of-`DATA` terminator. **Layer 3** — the Go stdlib `dotReader.Read` (`net/textproto/reader.go:L323`) — consumes the terminator: a canonical `.\r\n` walks `stateBeginLine`→`stateDot` (`:L347-348`) →`stateDotCR` (`:L358-359`) →`stateEOF` (`:L369-370`); reaching `stateEOF` makes `Read` return `io.EOF` (`:L401-402`). That EOF is what the reader maddy holds observes. **Layer 2** — go-smtp `Conn.handleData` (`conn.go:L498`) — has already written the `354` greeting (`conn.go:L510`), built the data reader via `newDataReader` → `c.text.DotReader()` (`data.go:L51,L53`), and called `c.Session().Data(r)` (`conn.go:L520`); after `Data` returns it drains the reader with `io.Copy(ioutil.Discard, r)` (`conn.go:L521`), writes the final reply (`conn.go:L522`), and (deferred at `conn.go:L512`) resets the envelope via `Conn.reset` (`conn.go:L694`). **Layer 1** — maddy `Session.Data` (`smtp.go:L312`) — calls `prepareBody` (`smtp.go:L283`), which reads the header with `textproto.ReadHeader` (`smtp.go:L285`), buffers the body with `buffer.BufferInMemory` (`smtp.go:L298`, impl `internal/buffer/memory.go:L27`), and adds a `Received` header via `target.GenerateReceived` (`smtp.go:L303`, impl `internal/target/received.go:L19`; add at `smtp.go:L307`); then it delivers (`delivery.Body` `internal/msgpipeline/msgpipeline.go:L307`, `delivery.Commit` `:L406`) and finally logs `s.log.Msg("accepted", ...)` (`smtp.go:L334`). maddy itself **never** parses the `.` — consistent with `HACKING.md:L123-124` ("this is not possible to modify the body contents, only header can be modified"), i.e. the boundary is decided upstream of maddy. **(observed + inferred as noted).**

The exact byte at which command parsing resumes is the byte immediately after the terminator line (`.\r\n` here): once `dotReader` returns EOF, `handleData` writes `250` and the connection's `net/textproto` reader parses whatever bytes follow as the next command (see Q3 for the pipelined case).

**Exact command.**
```text
$ python3 /tmp/maddy-smtp/rawclient.py canonical 127.0.0.1 2525

--- client: DATA phase — 354 greeting, canonical <CRLF>.<CRLF> terminator, then 250 queued ---
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

--- maddy structured log: the boundary decision + acceptance (msg_id 3eab2683) ---
smtp: incoming message    {"msg_id":"3eab2683","sender":"sender@example.com","src_host":"client.test","src_ip":"127.0.0.1:35154"}
smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery    {"msg_id":"3eab2683"}
smtp: accepted    {"msg_id":"3eab2683"}
smtp: 250 2.0.0 OK: queued
[debug] smtp: reset    

--- sink: the delivered message (post-decode, re-encoded to canonical CRLF) ---
========== DELIVERED MESSAGE #2 from 127.0.0.1:46226 ==========
ENVELOPE MAIL FROM:<sender@example.com> BODY=8BITMIME
ENVELOPE RCPT TO:<rcpt@example.com>
--- delivered body (unstuffed, 296 bytes) visible-control-chars ---
Received: from client.test (localhost [127.0.0.1]) by test.local<CR><LF>
 (envelope-sender <sender@example.com>) with ESMTP id 3eab2683; Mon, 13 Jul<CR><LF>
 2026 16:42:32 +0000<CR><LF>
From: sender@example.com<CR><LF>
To: rcpt@example.com<CR><LF>
Subject: framing probe<CR><LF>
<CR><LF>
BODYLINE-ONE canonical body.<CR><LF>
BODYLINE-TWO second line.<CR><LF>


# stability: 2/2 runs accepted; msg_ids 3eab2683 (run1), 14c1bf14 (run2); 296-byte delivered body each.
```

**Stability (≥2 runs).** Run twice on fresh connections: **2/2 accepted, 1 delivered each.** msg_id `3eab2683` (run 1) and `14c1bf14` (run 2); the delivered message portion (everything from `From:` onward, excluding the intentionally variable `Received` id+date) was 296 bytes in both runs. No variation observed.

---

### Q2 — Framing-variation payloads (byte-identical except the framing)

**Mechanism (cause → effect).** All five variants share an identical body and differ only in framing. The decision path inside `dotReader` differs by variant — canonical `.\r\n` terminates via `stateDotCR`→`stateEOF` (`reader.go:L369-370`) whereas a bare-LF `.\n` terminates **directly** from `stateDot` on `\n` (`reader.go:L362-363`) — but the *delivered* body does not, because (a) `dotReader` rewrites `\r\n`→`\n` on decode (a `\r` in `stateData` moves to `stateCR` with `continue`, emitting nothing, then `\n` emits a single `\n`: `reader.go:L390-391,L380-381`) and un-stuffs a leading dot (`stateBeginLine`+`.`→`stateDot`, `continue`, dot elided: `reader.go:L347-348`), and (b) the `smtp_downstream` relay re-encodes the buffered body to canonical CRLF and re-stuffs dots on the wire (`internal/target/smtp_downstream/smtp_downstream.go:L207` opens the buffer; `Commit`→`conn.Data(ctx, d.hdr, d.body)` just below serializes via the go-smtp client's dot-writer). **(observed).**

Variants: **V-A** canonical `<CRLF>.<CRLF>` with CRLF body lines; **V-B** bare-LF `<LF>.<LF>` with bare-LF body lines; **V-C** canonical terminator but bare-LF body lines; **V-D** dot-stuffed leading dots (`..text`, `...text`); **V-E** body lines that resemble the terminator (` .`, `.not-a-terminator`, `embedded . dot in middle`, `trailing dot.`). Each was run twice; **all five accepted and delivered exactly one message in both runs.**

**Exact commands.**
```bash
python3 /tmp/maddy-smtp/rawclient.py canonical   127.0.0.1 2525   # V-A
python3 /tmp/maddy-smtp/rawclient.py barelf       127.0.0.1 2525   # V-B
python3 /tmp/maddy-smtp/rawclient.py barelf-body  127.0.0.1 2525   # V-C
python3 /tmp/maddy-smtp/rawclient.py dotstuff     127.0.0.1 2525   # V-D
python3 /tmp/maddy-smtp/rawclient.py resembles    127.0.0.1 2525   # V-E
```

**KEY RESULT — V-A vs V-B deliver byte-identical bodies despite different wire framing.** The client sent CRLF body lines + `<CRLF>.<CRLF>` (V-A) versus bare-LF body lines + `<LF>.<LF>` (V-B); both were accepted, and the delivered message portions are identical byte-for-byte (identical SHA-256; note the `0d 0a` line endings in **both** hexdumps):
```text
$ # message portion (from "From:" onward) of the delivered body, canonical (V-A) vs bare-LF (V-B)
$ sha256sum VA_body.bin VB_body.bin
6fafed38f64a7386bbc798e3f3f801fc9d023b71bf1a6e00f8af8b402fc47932  VA_body.bin
6fafed38f64a7386bbc798e3f3f801fc9d023b71bf1a6e00f8af8b402fc47932  VB_body.bin
$ cmp VA_body.bin VB_body.bin && echo IDENTICAL
IDENTICAL (byte-for-byte)

$ hexdump -C VA_body.bin   # V-A canonical CRLF.CRLF terminator on the wire
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

$ hexdump -C VB_body.bin   # V-B bare-LF <LF>.<LF> terminator on the wire
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

And the wire-level confirmation that the bare-LF terminator `<LF>.<LF>` (V-B) is itself accepted with a `250` — i.e. the acceptance *decision* fired on bare-LF framing, not merely that its delivered bytes matched V-A:
```text
$ python3 /tmp/maddy-smtp/rawclient.py barelf 127.0.0.1 2525
--- client transcript: DATA phase (bare <LF> body lines AND the ".<LF>" terminator) ---
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
--- maddy structured log for this run: ACCEPTED ---
    smtp: incoming message	{"msg_id":"1f057472","sender":"sender@example.com","src_host":"client.test","src_ip":"127.0.0.1:47628"}
    smtp: accepted	{"msg_id":"1f057472"}
    smtp: 250 2.0.0 OK: queued
```

**V-C — bare-LF body lines, canonical terminator.** The bare-LF body lines are delivered normalized to CRLF (`0d 0a`). (A trailing blank line `0d 0a 0d 0a` appears because the payload ended a line with bare `\n` and then sent the canonical `<CRLF>.<CRLF>` terminator, so both the bare `\n` and the terminator's leading CRLF each produced a line boundary — an honest consequence of that specific byte sequence.)
```text
$ python3 /tmp/maddy-smtp/rawclient.py barelf-body 127.0.0.1 2525
--- delivered message portion (bare-LF body lines normalized to CRLF on relay) ---
00000000  46 72 6f 6d 3a 20 73 65  6e 64 65 72 40 65 78 61  |From: sender@exa|
00000010  6d 70 6c 65 2e 63 6f 6d  0d 0a 54 6f 3a 20 72 63  |mple.com..To: rc|
00000020  70 74 40 65 78 61 6d 70  6c 65 2e 63 6f 6d 0d 0a  |pt@example.com..|
00000030  53 75 62 6a 65 63 74 3a  20 66 72 61 6d 69 6e 67  |Subject: framing|
00000040  20 70 72 6f 62 65 0d 0a  0d 0a 42 4f 44 59 4c 49  | probe....BODYLI|
00000050  4e 45 2d 4f 4e 45 20 63  61 6e 6f 6e 69 63 61 6c  |NE-ONE canonical|
00000060  20 62 6f 64 79 2e 0d 0a  42 4f 44 59 4c 49 4e 45  | body...BODYLINE|
00000070  2d 54 57 4f 20 73 65 63  6f 6e 64 20 6c 69 6e 65  |-TWO second line|
00000080  2e 0d 0a 0d 0a                                    |.....|
00000085
```

**V-D — dot-stuffing / transparency (RFC 5321 §4.5.2).** The client sent doubled leading dots; `dotReader` un-stuffs one dot on decode; the `smtp_downstream` relay re-stuffs on the wire; the sink un-stuffs again for the final delivered form. The full round trip (client 2 dots → decode 1 → relay re-stuff 2 → sink 1) is visible below:
```text
$ python3 /tmp/maddy-smtp/rawclient.py dotstuff 127.0.0.1 2525
--- (client) body lines SENT on the wire to maddy (leading dots doubled) ---
[C->S body (dot-stuffed leading dots)]
From: sender@example.com<CR><LF>
To: rcpt@example.com<CR><LF>
Subject: framing probe<CR><LF>
<CR><LF>
..leading-double-dot should un-stuff to one dot<CR><LF>
...three dots become two<CR><LF>
normal line<CR><LF>

[C->S end-of-DATA terminator (canonical)]
--- (sink) DELIVERED body, un-stuffed by the sink (one leading dot) ---

.leading-double-dot should un-stuff to one dot
..three dots become two
normal line
--- (sink) RAW WIRE bytes maddys relay actually sent to the sink (dots RE-STUFFED) ---
Received: from client.test (localhost [127.0.0.1]) by test.local<CR><LF>
 (envelope-sender <sender@example.com>) with ESMTP id 7dafdd31; Mon, 13 Jul<CR><LF>
 2026 16:43:24 +0000<CR><LF>
From: sender@example.com<CR><LF>
To: rcpt@example.com<CR><LF>
Subject: framing probe<CR><LF>
<CR><LF>
..leading-double-dot should un-stuff to one dot<CR><LF>
...three dots become two<CR><LF>
normal line<CR><LF>
.<CR><LF>

```

**V-E — lines that resemble the end-of-`DATA` marker.** None of ` .` (space then dot), `.not-a-terminator` (a single leading dot, un-stuffed to `not-a-terminator`), `embedded . dot in middle`, or `trailing dot.` terminates the message; all four survive in one delivered message. Only a line consisting of exactly one dot terminates — grounded in the FSM: at `stateBeginLine` a `.` moves to `stateDot` (`reader.go:L347-348`); termination requires the *next* byte to be `\n` (`:L362-363`) or `\r\n` (`:L358-359,L369-370`); any other byte moves to `stateData` (`:L366`) and the line is ordinary body.
```text
$ python3 /tmp/maddy-smtp/rawclient.py resembles 127.0.0.1 2525
--- delivered message portion: all four resembling lines survived as ONE message ---
    (client sent lines: " .", ".not-a-terminator", "embedded . dot in middle", "trailing dot.")
00000000  46 72 6f 6d 3a 20 73 65  6e 64 65 72 40 65 78 61  |From: sender@exa|
00000010  6d 70 6c 65 2e 63 6f 6d  0d 0a 54 6f 3a 20 72 63  |mple.com..To: rc|
00000020  70 74 40 65 78 61 6d 70  6c 65 2e 63 6f 6d 0d 0a  |pt@example.com..|
00000030  53 75 62 6a 65 63 74 3a  20 66 72 61 6d 69 6e 67  |Subject: framing|
00000040  20 70 72 6f 62 65 0d 0a  0d 0a 20 2e 0d 0a 6e 6f  | probe.... ...no|
00000050  74 2d 61 2d 74 65 72 6d  69 6e 61 74 6f 72 0d 0a  |t-a-terminator..|
00000060  65 6d 62 65 64 64 65 64  20 2e 20 64 6f 74 20 69  |embedded . dot i|
00000070  6e 20 6d 69 64 64 6c 65  0d 0a 74 72 61 69 6c 69  |n middle..traili|
00000080  6e 67 20 64 6f 74 2e 0d  0a                       |ng dot...|
00000089
```

**Stability (≥2 runs).** All five variants were run twice; each accepted and delivered exactly one message in both runs, with identical delivered message portions per variant (V-A 131 B, V-B 131 B — identical to V-A, V-C 133 B, V-D 160 B, V-E 137 B). No variation observed.

---

### Q3 — Pipelined pressure vs a stricter peer (SMTP smuggling)

**Mechanism (cause → effect).** `net/textproto` reads through a buffered reader (`bufio`), so bytes sent **after** the terminator in the same `send()` are already sitting in that buffer. A bare-LF `<LF>.<LF>` is accepted as end-of-data by `dotReader` (directly `stateDot`+`\n`→`stateEOF`, `reader.go:L362-363`); `handleData` then drains the data reader (`conn.go:L521`), writes `250`, and `Conn.reset` (`conn.go:L694`, deferred at `:L512`) clears `fromReceived`/`recipients` (`conn.go:L700-702`). The next read off the still-buffered stream is parsed as a **new** command, so a pipelined `MAIL/RCPT/DATA` after the bare-LF terminator becomes a **second transaction**. **(observed).**

**Exact command.** One connection, one `send()`: transaction 1 (`legit@example.com`) ends with a bare-LF `<LF>.<LF>`, immediately followed (pipelined) by transaction 2 with a **spoofed** `MAIL FROM:<SPOOFED-attacker@evil.example>` / `RCPT TO:<victim@example.com>` and a canonical `<CRLF>.<CRLF>`:
```bash
python3 /tmp/maddy-smtp/rawclient.py smuggle 127.0.0.1 2525
```

**Result — maddy accepts TWO transactions and delivers TWO messages** (the second with the spoofed envelope). Client transcript and maddy logs:
```text
$ python3 /tmp/maddy-smtp/rawclient.py smuggle 127.0.0.1 2525

--- client: ONE pipelined send() (bare-LF ".<LF>" ends txn1; txn2 pipelined behind it) ---
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

--- server replies (note TWO "250 2.0.0 OK: queued") ---
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

--- maddy structured logs: TWO accepted, distinct msg_id, second sender SPOOFED ---
smtp: incoming message    {"msg_id":"d34a6486","sender":"legit@example.com","src_host":"client.test","src_ip":"127.0.0.1:33696"}
smtp: accepted    {"msg_id":"d34a6486"}
smtp: incoming message    {"msg_id":"772526e1","sender":"SPOOFED-attacker@evil.example","src_host":"client.test","src_ip":"127.0.0.1:33696"}
smtp: accepted    {"msg_id":"772526e1"}

```

The two delivered messages at the sink — note the second carries the spoofed sender and a different recipient, and the first message's body does **not** contain the smuggled commands (proving the bare-LF `<LF>.<LF>` genuinely terminated transaction 1):
```text
--- sink: TWO delivered messages (envelopes) ---
========== DELIVERED MESSAGE #14 from 127.0.0.1:41442 ==========
ENVELOPE MAIL FROM:<legit@example.com> BODY=8BITMIME
ENVELOPE RCPT TO:<rcpt@example.com>
========== DELIVERED MESSAGE #15 from 127.0.0.1:41450 ==========
ENVELOPE MAIL FROM:<SPOOFED-attacker@evil.example> BODY=8BITMIME
ENVELOPE RCPT TO:<victim@example.com>
```

**Conformance contrast (prose; RFC 5321 §4.1.1.4).** A strictly conformant receiver treats `<LF>.<LF>` as ordinary body content — it is **not** a valid end-of-mail-data indication — and would therefore read the pipelined `MAIL FROM:<SPOOFED…>` etc. as *body text of the first message*, delivering exactly **one** message with no spoofed envelope. maddy delivers **two**. That divergence between "what maddy accepts as a new transaction" and "what a strict peer still treats as body" is precisely the SMTP-smuggling condition.

**Stability (≥2 runs).** Run twice: **2 accepted + 2 delivered each run.** msg_ids `d34a6486`+`772526e1` (run 1) and `5e94a9ad`+`7b7b28f7` (run 2). The spoofed envelope (`SPOOFED-attacker@evil.example` → `victim@example.com`) appeared in both runs. No variation observed.

---

### Q4 — Back-to-back stability

**Mechanism (cause → effect).** The terminator decision is a deterministic FSM (`dotReader`, `reader.go:L323-408`) driven only by the input bytes, so identical input must yield identical outcomes. Between transactions on one connection, go-smtp's `Conn.reset` (`conn.go:L694`) clears the envelope and maddy's `Session.Reset` (`smtp.go:L60`) logs `[debug] smtp: reset`; a fresh `MAIL FROM` starts the next transaction cleanly. **(observed).**

**Exact commands.** (a) five identical canonical messages on **fresh** connections, as two independent batches; (b) three transactions **back-to-back on one** connection:
```bash
# (a) two independent batches of 5 fresh-connection canonical messages:
for i in 1 2 3 4 5; do python3 /tmp/maddy-smtp/rawclient.py canonical 127.0.0.1 2525; done   # batch A
for i in 1 2 3 4 5; do python3 /tmp/maddy-smtp/rawclient.py canonical 127.0.0.1 2525; done   # batch B
# (b) three back-to-back transactions on ONE connection:
python3 /tmp/maddy-smtp/rawclient.py backtoback 127.0.0.1 2525
```

**Result — deterministic, no wobble.** The two batches and the back-to-back run:
```text
$ # (a) two independent batches of 5 fresh-connection canonical messages:
$ for i in 1 2 3 4 5; do python3 /tmp/maddy-smtp/rawclient.py canonical 127.0.0.1 2525; done   # x2 batches
batchA: accepted=5/5  delivered_files=5  distinct-body-sha256(excl Received)=1
batchB: accepted=5/5  delivered_files=5  distinct-body-sha256(excl Received)=1

$ # (b) three back-to-back transactions on ONE connection:
$ python3 /tmp/maddy-smtp/rawclient.py backtoback 127.0.0.1 2525

--- maddy structured logs: 3 accepted, 3 envelope resets between transactions (one connection) ---
smtp: incoming message    {"msg_id":"0c494fad","sender":"sender1@example.com","src_host":"client.test","src_ip":"127.0.0.1:56442"}
smtp: accepted    {"msg_id":"0c494fad"}
[debug] smtp: reset    
smtp: incoming message    {"msg_id":"6cb9caa5","sender":"sender2@example.com","src_host":"client.test","src_ip":"127.0.0.1:56442"}
smtp: accepted    {"msg_id":"6cb9caa5"}
[debug] smtp: reset    
smtp: incoming message    {"msg_id":"3fa76838","sender":"sender3@example.com","src_host":"client.test","src_ip":"127.0.0.1:56442"}
smtp: accepted    {"msg_id":"3fa76838"}
[debug] smtp: reset    

--- sink: 3 delivered messages, distinct senders ---
========== DELIVERED MESSAGE #28 from 127.0.0.1:46828 ==========
ENVELOPE MAIL FROM:<sender1@example.com> BODY=8BITMIME
========== DELIVERED MESSAGE #29 from 127.0.0.1:50852 ==========
ENVELOPE MAIL FROM:<sender2@example.com> BODY=8BITMIME
========== DELIVERED MESSAGE #30 from 127.0.0.1:50862 ==========
ENVELOPE MAIL FROM:<sender3@example.com> BODY=8BITMIME

# reset/accepted counts across the two back-to-back runs:
run1: resets=3 accepted=3    run2: resets=3 accepted=3
```

`distinct-body-sha256(excl Received)=1` means all five delivered bodies in a batch hashed identically (excluding the intentionally-variable `Received` id+date). The back-to-back run shows three distinct senders delivered and the `[debug] smtp: reset` between each transaction.

**Stability (≥2 runs).** (a) two independent 5-message batches → **5/5 accepted each (10/10 total), one distinct delivered-body hash per batch.** (b) back-to-back run twice → **3/3 accepted+delivered each, 3 envelope resets.** Observed outcome distribution: a single stable outcome (100% accepted); no run-to-run variation.

---

### Q5 — A cleaning / blocking front proxy

A small stdlib TCP proxy (`/tmp/maddy-smtp/proxy.py`, full source in §7) sits in front of maddy on `127.0.0.1:2524 → 127.0.0.1:2525`, in two modes. The **same** ambiguous bare-LF smuggling payload from Q3 is routed through each.

**Q5a — normalize (bare `\n`→`\r\n`).** **Mechanism:** rewriting every bare `\n` to `\r\n` turns the first transaction's bare-LF `<LF>.<LF>` into a canonical `<CRLF>.<CRLF>` before maddy sees it. **Effect (observed): the smuggle is not prevented — it is *manufactured* into a fully canonical boundary,** so maddy still delivers two messages including the spoofed one. This is the "clean-up" front-end (the Cisco-normalizes-bare-LF analogue) that can *create* an unambiguous smuggling boundary out of ambiguous input.
```bash
python3 /tmp/maddy-smtp/proxy.py normalize 2524 127.0.0.1 2525 &   # front proxy
python3 /tmp/maddy-smtp/rawclient.py smuggle 127.0.0.1 2524        # send THROUGH the proxy
```
```text
$ python3 /tmp/maddy-smtp/proxy.py normalize 2524 127.0.0.1 2525 &
$ python3 /tmp/maddy-smtp/rawclient.py smuggle 127.0.0.1 2524   # bare-LF payload THROUGH normalize proxy

--- maddy structured logs: bare-LF rewritten to CRLF upstream => STILL 2 accepted (boundary manufactured) ---
smtp: incoming message    {"msg_id":"f0dd107b","sender":"legit@example.com","src_host":"client.test","src_ip":"127.0.0.1:53640"}
smtp: accepted    {"msg_id":"f0dd107b"}
smtp: incoming message    {"msg_id":"3d34fb81","sender":"SPOOFED-attacker@evil.example","src_host":"client.test","src_ip":"127.0.0.1:53640"}
smtp: accepted    {"msg_id":"3d34fb81"}

--- sink: 2 delivered messages, including the spoofed envelope ---
========== DELIVERED MESSAGE #34 from 127.0.0.1:45546 ==========
ENVELOPE MAIL FROM:<legit@example.com> BODY=8BITMIME
========== DELIVERED MESSAGE #35 from 127.0.0.1:45558 ==========
ENVELOPE MAIL FROM:<SPOOFED-attacker@evil.example> BODY=8BITMIME

# stability: normalize run1 accepted=2, run2 accepted=2
```

**Q5b — reject (refuse bare `\n`).** **Mechanism:** the proxy scans the client→server stream and, on the first bare `\n`, returns a `5xx` to the client and closes without forwarding. **Effect (observed): the ambiguous traffic is blocked upstream of maddy** — the client receives the proxy's `500`, and maddy delivers nothing (it only ever emitted its `220` greeting). This is RFC-strict enforcement in front of maddy.
```bash
python3 /tmp/maddy-smtp/proxy.py reject 2524 127.0.0.1 2525 &   # front proxy
python3 /tmp/maddy-smtp/rawclient.py smuggle 127.0.0.1 2524     # send THROUGH the proxy
```
```text
$ python3 /tmp/maddy-smtp/proxy.py reject 2524 127.0.0.1 2525 &
$ python3 /tmp/maddy-smtp/rawclient.py smuggle 127.0.0.1 2524   # same bare-LF payload THROUGH reject proxy

--- client: proxy refuses on first bare LF with a 5xx and closes ---
500 5.5.2 bare newline rejected by front proxy (RFC 5321 requires CRLF)<CR><LF>

--- maddy.log for this run: ONLY its 220 greeting; nothing forwarded ---
smtp: 220 test.local ESMTP Service Ready

--- counts ---
accepted-count this run: 0 ; delivered-count: 0   (stable: run1=0/0, run2=0/0)
```

The reject proxy is **selective**: a fully-canonical message (no bare newlines) passes straight through and is delivered normally, confirming the proxy blocks only the ambiguous framing, not all traffic:
```text
$ python3 /tmp/maddy-smtp/proxy.py reject 2524 127.0.0.1 2525 &
$ python3 /tmp/maddy-smtp/rawclient.py canonical 127.0.0.1 2524   # fully-canonical message THROUGH reject proxy

--- canonical (no bare newline) passes straight through; maddy accepts + delivers ---
smtp: incoming message    {"msg_id":"eee2abb4","sender":"sender@example.com","src_host":"client.test","src_ip":"127.0.0.1:35800"}
smtp: accepted    {"msg_id":"eee2abb4"}
========== DELIVERED MESSAGE #38 from 127.0.0.1:33242 ==========
ENVELOPE MAIL FROM:<sender@example.com> BODY=8BITMIME
client final reply: 250 2.0.0 OK: queued<CR><LF>   => proxy is selective (blocks only bare newlines)
```


**Contrast & which errors appear/disappear.** normalize = *repair* (no new error surfaces; maddy still accepts — and the smuggle boundary is made canonical); reject = *enforcement* (a **new** `500 5.5.2 bare newline rejected …` appears at the client and maddy's `accepted` **disappears** entirely). **Stability (≥2 runs):** normalize 2/2 → 2 accepted+2 delivered; reject 2/2 → 0 accepted+0 delivered. No variation observed.

---

### Q6 — The real runtime story: what lingers, and what is expected but never seen

Each edge/error branch is triggered directly (not inferred). **What lingers on failure:** nothing — maddy's `Session.Reset` (`smtp.go:L60`) sees a still-live delivery and calls `Session.abort` (`smtp.go:L67`), which invokes `delivery.Abort`, logs `s.log.Msg("aborted", …)` (`smtp.go:L72`), and clears `mailFrom`/`msgMeta`/`delivery` (`smtp.go:L74-79`). No `accepted` is logged and nothing reaches the sink.

**Q6a — dropped connection mid-`DATA`.** **Mechanism:** closing the socket after `354` and some body bytes but before any terminator makes `dotReader.Read` hit `io.EOF` on the underlying reader and convert it to `io.ErrUnexpectedEOF` (`reader.go:L340-341`); `Session.Data`'s `wrapErr` logs `s.log.Error("DATA error", …)` (`smtp.go:L317`). **(observed).**
```bash
python3 /tmp/maddy-smtp/rawclient.py dropmid 127.0.0.1 2525
```
```text
$ python3 /tmp/maddy-smtp/rawclient.py dropmid 127.0.0.1 2525

--- client: after 354, send partial body then CLOSE the socket (no terminator) ---
[C->S partial body then DROP (no terminator)]
Subject: will be dropped<CR><LF>
<CR><LF>
partial body with no terminator<CR><LF>

[C->S] connection closed mid-DATA (no <CRLF>.<CRLF> sent)
--- maddy structured log: unexpected EOF => DATA error => 554 => aborted => reset ---
smtp: incoming message    {"msg_id":"155902f6","sender":"sender@example.com","src_host":"client.test","src_ip":"127.0.0.1:46964"}
smtp: DATA error    {"msg_id":"155902f6","reason":"unexpected EOF"}
smtp: 554 5.0.0 Internal server error (msg ID = 155902f6)
smtp: aborted    {"msg_id":"155902f6"}
[debug] smtp: reset    

accepted-count this run: 0 ; delivered-count: 0   (stable: run1=0/0, run2=0/0)
```

maddy replies `554 5.0.0 Internal server error`, logs `aborted`, and resets; **0 accepted, 0 delivered.** Stable across 2 runs (`DATA error` present both times).

**Q6b — oversize message.** **Mechanism:** `max_message_size` maps to go-smtp's `MaxMessageBytes` (`smtp.go:L561`); `newDataReader` arms the limit (`data.go:L56-58`) and the wrapping `dataReader.Read` returns `ErrDataTooLarge` (`552`, `EnhancedCode{5,3,4}`, `data.go:L38-42,L66-67`) once the cap is exceeded. To trigger it quickly this used a **labelled configured-limit variant** config `max_message_size 2K` (the canonical default is 32 MiB — advertised as `SIZE 33554432` in EHLO). **(observed).**
```bash
# variant config differs from the canonical maddy.conf ONLY by: max_message_size 2K
python3 /tmp/maddy-smtp/rawclient.py oversize 127.0.0.1 2525
```
```text
$ # variant config differs from canonical maddy.conf ONLY by: max_message_size 2K
$ ./maddy -config /tmp/maddy-smtp/maddy-oversize.conf     # EHLO now advertises SIZE 2048
$ python3 /tmp/maddy-smtp/rawclient.py oversize 127.0.0.1 2525

--- client: EHLO shows the configured 2K limit; body >2K => 552 ---
250 SIZE 2048<CR><LF>
552 5.3.4 Maximum message size exceeded (msg ID = 3c3ac391)<CR><LF>

--- maddy structured log: MaxMessageBytes trips => DATA error => 552 => aborted => reset ---
smtp: incoming message    {"msg_id":"3c3ac391","sender":"sender@example.com","src_host":"client.test","src_ip":"127.0.0.1:42490"}
smtp: DATA error    {"msg_id":"3c3ac391","reason":"Maximum message size exceeded"}
smtp: 552 5.3.4 Maximum message size exceeded (msg ID = 3c3ac391)
smtp: aborted    {"msg_id":"3c3ac391"}
[debug] smtp: reset    

accepted this run: 0   (stable: run1 552+DATA error, run2 552+DATA error, 0 accepted both)
```

**0 accepted**; stable across 2 runs (client `552` + maddy `552` both runs).

**Q6c — over-long line.** **Mechanism:** go-smtp's per-line cap `MaxLineLength` defaults to `2000` (`server.go:L76`) and maddy does **not** override it; the `lineLimitReader` returns `ErrTooLongLine` ("smtp: too longer line in input stream", `lengthlimit_reader.go:L8,L22-23,L41-42`). Sending one ~5000-byte line trips it. **(observed).**
```bash
python3 /tmp/maddy-smtp/rawclient.py longline 127.0.0.1 2525
```
```text
$ python3 /tmp/maddy-smtp/rawclient.py longline 127.0.0.1 2525   # one ~5000-byte line > MaxLineLength(2000)

--- client final reply ---
554 5.0.0 Internal server error (msg ID = 8916ebe2)<CR><LF>

--- maddy structured log: lineLimitReader ErrTooLongLine => DATA error => 554 => aborted ---
smtp: incoming message    {"msg_id":"8916ebe2","sender":"sender@example.com","src_host":"client.test","src_ip":"127.0.0.1:33736"}
smtp: DATA error    {"msg_id":"8916ebe2","reason":"smtp: too longer line in input stream"}
smtp: 554 5.0.0 Internal server error (msg ID = 8916ebe2)
smtp: aborted    {"msg_id":"8916ebe2"}

accepted this run: 0  (stable 2/2)
```

maddy replies `554`, logs `aborted`, resets; **0 accepted.** Stable across 2 runs.

**Q6d — the critical ABSENCE (the headline).** An RFC-conformant server would refuse a bare-LF line / `<LF>.<LF>` end-of-data with a `5xx`. maddy emits **no such rejection** and exposes **no mitigation knob**. Grounded in a repository-wide negative search (re-run here; empty output is the evidence):
```bash
grep -rniE 'bare(lf|cr|newline)|smuggl|forbid_bare' internal/ cmd/ pkg/ ; echo "exit=$?"
# broadened across the whole repo:
grep -rniE 'forbid_bare|bare_newline|barenewline|smtp_smuggl|smuggling' . \
     --include='*.go' --include='*.conf' --include='*.md' ; echo "exit=$?"
```
```text
$ grep -rniE 'bare(lf|cr|newline)|smuggl|forbid_bare' internal/ cmd/ pkg/ ; echo "exit=$?"
exit=1
$ grep -rniE 'forbid_bare|bare_newline|barenewline|smtp_smuggl|smuggling' . --include='*.go' --include='*.conf' --include='*.md' ; echo "exit=$?"
exit=1
```
Both searches return **no matches** (`grep` exit `1`). Contrast (prose): after the December 2023 SMTP-smuggling disclosure, peers added explicit controls — e.g. Postfix's `smtpd_forbid_bare_newline` with `normalize`/`reject` modes (CVE-2023-51764), with fixes also shipping in Exim and Sendmail. The pinned 2019 go-smtp plus the Go stdlib `dotReader` used by maddy accept bare-LF terminators with no option to normalize or reject them — the RFC-conformant behaviour maddy never exhibits.

---

## 3. Three-layer code-path map — where the boundary is actually decided

The end-of-`DATA` decision is not made in maddy. It is made three layers deep, in the Go standard library. maddy's own code receives an **already-decoded** body reader and never inspects the `.` terminator.

```
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

On any `DATA` failure (dropped connection, oversize, over-long line), maddy's `Session.Reset`/`Session.abort` clears all per-transaction state — this is why nothing lingers (`internal/endpoint/smtp/smtp.go`):
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


## 4. RFC 5321 / RFC 5322 and the SMTP-smuggling context

The following standards and advisory facts frame *why* maddy's observed behaviour matters. They are attributed in prose; the inline `file:line` citations elsewhere in this document are reserved for code claims only.

- **RFC 5321 §2.3.8 (Lines).** In SMTP, a "line" is a sequence of characters terminated **only** by `<CRLF>`. Conforming implementations are required not to recognize any other sequence — in particular a bare `<LF>` or a bare `<CR>` — as a line terminator.
- **RFC 5321 §4.1.1.4 (DATA).** The end of mail data is indicated by a line containing only a single period, i.e. the five-character sequence `<CRLF>.<CRLF>`. The standard is explicit that a bare `<LF>.<LF>` sequence **must not** be treated as the end-of-mail-data indication. A strictly conforming receiver therefore treats `<LF>.<LF>` as ordinary body content.
- **RFC 5321 §4.5.2 (Transparency / dot-stuffing).** When a line of mail text begins with a period, the sender doubles it (`.`→`..`); the receiver strips one leading period from any line beginning with a period before the body is stored. This is exactly the un-stuff/re-stuff round trip observed in Q2 V-D.
- **RFC 5322 §2.3.** In a message body, the CR and LF octets are permitted to occur **only together, as CRLF**; they must not appear independently. A bare `<LF>` inside a body is thus non-conformant framing.
- **SMTP smuggling (SEC Consult disclosure, December 2023; Postfix CVE-2023-51764).** A receiver that leniently accepts a bare-LF (or bare-CR) sequence as end-of-data lets an attacker "smuggle" a second message — with a spoofed envelope sender — inside what a conforming peer treats as a single message body, bypassing SPF/DKIM/DMARC alignment on the smuggled message. In response, peer MTAs added explicit controls: Postfix introduced `smtpd_forbid_bare_newline` with `normalize` and `reject`-style handling, and fixes shipped in Exim and Sendmail.

**Mapping to maddy (observed + inferred).** maddy pins `github.com/emersion/go-smtp v0.12.1-0.20191206174923-1f576e0ec85c` (a 2019 revision; `go.mod:L19`) and delegates the end-of-`DATA` decision to that library, which in turn delegates it to the Go standard library's `net/textproto` `dotReader`. As shown in §3 and demonstrated at runtime in Q2/Q3, that FSM accepts **both** canonical `<CRLF>.<CRLF>` and bare-LF `<LF>.<LF>` as terminators (`reader.go:L362-363`). maddy neither overrides this nor layers any bare-newline check on top: the repository-wide search in Q6 for any such control returns nothing. That places maddy, at this revision, in the class of receivers vulnerable to SMTP smuggling, with **no** normalization or rejection knob — the RFC-conformant `5xx` refusal that Postfix et al. added is precisely the behaviour maddy never exhibits.

## 5. Caveats — observed vs inferred, environment, and reproducibility

**Observed vs inferred.** Every behavioural claim tagged **(observed)** is backed by the captured transcript / log / delivered-bytes / error text shown inline together with the exact command that produced it. Claims tagged **(inferred)** are read from the cited code and were not independently instrumented — specifically: the precise internal state-variable transitions of the `dotReader` FSM are inferred from `net/textproto/reader.go` (their *effects* — acceptance, normalization, un-stuffing, `ErrUnexpectedEOF` — are observed); and the `HACKING.md` "body cannot be modified" invariant is a documentation claim, not a runtime measurement (its consequence — maddy never parsing `.` — is corroborated by the fact that maddy only ever receives an already-decoded reader).

**Confirmed toolchain and paths (this container).**
- `go version` → `go version go1.18.10 linux/amd64` (this is the exact stdlib whose `net/textproto/reader.go` `dotReader` was exercised; the file is `/usr/local/go/src/net/textproto/reader.go`, dated Jan 9 2023).
- `GOROOT` = `/usr/local/go` ; `GOMODCACHE` = `/go/pkg/mod`.
- go-smtp resolved pin: `go list -m github.com/emersion/go-smtp` → `github.com/emersion/go-smtp v0.12.1-0.20191206174923-1f576e0ec85c` (matches `go.mod:L19`); cache dir `/go/pkg/mod/github.com/emersion/go-smtp@v0.12.1-0.20191206174923-1f576e0ec85c`.
- Repository: git branch `blitzy-438524cc-5240-4dd1-904e-f2f199faf5de`, HEAD short `26452dd`. (The **source** branch, per the SWE-AtlasQnA-Repo naming rule, is `maddy_26452dd8dd78`, which is why this document is named `maddy_26452dd8dd78.md`.)

> **Note on the Go version.** The task-analysis notes anticipated a Go 1.21.13 feasibility build; the actual canonical container ships **Go 1.18.10**, so that is the version reported here. The `dotReader` FSM regions cited (`reader.go` L311–408) were re-verified by inspection in this container's 1.18.10 stdlib and are stable across Go 1.13–1.21.

**Exact build and run commands.**
- Build (canonical, default configuration): `go build -o /tmp/maddy-smtp/maddy ./cmd/maddy` — exit 0 (only a harmless `mattn/go-sqlite3` cgo compiler notice, from a storage dependency unrelated to the `DATA` path; `CGO_ENABLED=1` is set by `/etc/profile.d/go.sh`).
- Run (canonical): `/tmp/maddy-smtp/maddy -config /tmp/maddy-smtp/maddy.conf` (this build takes configuration via the `-config` flag; there is no `run` subcommand). `io_debug yes` plus a top-level `debug yes` are required for the raw-wire transcript, because `Log.DebugWriter()` returns a discard writer unless debug logging is on.
- Oversize variant (Q6b only, clearly labelled): the same config with `max_message_size 2K` added, run as `/tmp/maddy-smtp/maddy -config /tmp/maddy-smtp/maddy-oversize.conf`; the canonical instance (advertising `SIZE 33554432` = 32 MiB) was restored immediately afterward and re-verified.

**Non-canonical labels.** The only non-default configuration used anywhere is the Q6b `max_message_size 2K` variant, explicitly labelled as such at its use site; every other experiment ran against the canonical default configuration through the real TCP listener.

**Run-to-run variation.** None observed. Every experiment was run at least twice; Q4 was additionally run as two independent 5-message batches plus repeated back-to-back transactions. All outcomes were a single stable value (see each Q's "Stability" line).

**Read-only & cleanup.** All observation artifacts (the built binary, `maddy.conf`, the oversize variant, the raw client, the sink, the front proxy, and all captured logs) lived under `/tmp/maddy-smtp/` only; none was added to the repository. On completion the temporary processes were stopped and `/tmp/maddy-smtp/` removed. `git status --porcelain` then reported only the single new file (and the new directories that hold it):
```text
?? blitzy/documentation/maddy_26452dd8dd78.md
```
`git diff --stat` against HEAD shows no modification to any existing tracked file.

## 6. Coverage pass — every question and named item, confirmed addressed

**The six sub-questions (each answered by name, with observed output):**

- [x] **Q1 — the boundary-decision moment.** §2 Q1: `354` greeting, terminator consumed → `stateEOF`, `250 OK: queued`, `smtp: accepted {msg_id}` (`smtp.go:L334`), delivered body at sink. (observed)
- [x] **Q2 — framing-variation payloads.** §2 Q2 V-A…V-E: canonical vs bare-LF terminator, bare-LF body lines, dot-stuffing, terminator-resembling lines; V-A≡V-B byte-identical delivery (SHA-256). (observed)
- [x] **Q3 — pipelined pressure vs a stricter peer.** §2 Q3: bare-LF `<LF>.<LF>` + pipelined second block ⇒ two delivered messages incl. spoofed sender; strict-peer contrast in prose. (observed + prose)
- [x] **Q4 — back-to-back stability.** §2 Q4: two 5-message batches (10/10 accepted, 1 body hash each) + 3 back-to-back with 3 resets; deterministic. (observed)
- [x] **Q5 — a cleaning / blocking front proxy.** §2 Q5: normalize (manufactures boundary, still 2 delivered) vs reject (`500`, 0 delivered; canonical control passes). (observed)
- [x] **Q6 — what lingers, and what is expected but never seen.** §2 Q6: dropped-mid-`DATA`→`unexpected EOF`/`554`; oversize→`552`; over-long line→`554`; the **absence** of any bare-newline `5xx` (empty grep). (observed)

**Named mechanisms (function/method/struct), each cited by `file:line` and shown in §3:**

- [x] `Session.Data` (`smtp.go:L312`) · [x] `prepareBody` (`smtp.go:L283`) · [x] `BufferInMemory` (`memory.go:L27`, called `smtp.go:L298`) · [x] `GenerateReceived` (`received.go:L19`, called `smtp.go:L303`; add `smtp.go:L307`)
- [x] `dotReader` states — `stateBeginLine`/`stateDot`/`stateDotCR`/`stateCR`/`stateData`/`stateEOF` (`reader.go:L327-333`, transitions L346-402)
- [x] `handleData` (`conn.go:L498`) · [x] drain `io.Copy(ioutil.Discard, r)` (`conn.go:L521`) · [x] `Conn.reset` (`conn.go:L694`) · [x] `io.TeeReader`→`Debug` (`conn.go:L68`)
- [x] `newDataReader`→`c.text.DotReader()` (`data.go:L51,L53`) · [x] `Session.abort`/`Session.Reset` (`smtp.go:L60,L67`, `aborted` log L72) · [x] delivery `Body`/`Commit` (`msgpipeline.go:L307,L406`); relay `conn.Data` (`smtp_downstream.go:L207`+)

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

These four scripts and the minimal config are the exact artifacts that produced every transcript above. They lived under `/tmp/maddy-smtp/` only and were removed on completion (see §5); they are reproduced here so the evidence is fully reproducible.

### 7.1 Minimal canonical config — `/tmp/maddy-smtp/maddy.conf`
```
## Minimal maddy config for the DATA-boundary investigation (temporary; /tmp only).
## Inbound smtp endpoint -> smtp_downstream relay -> capturing Python sink (:2526).
## io_debug yes + global debug yes => raw wire transcript is written to the log.
## All production reject rules (require_matching_ehlo, require_mx_record, verify_dkim,
## apply_spf, dmarc, source/destination rules) are OMITTED so raw test traffic is accepted.
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
```
## Oversize VARIANT config (temporary; /tmp only) — identical to maddy.conf but with a
## small max_message_size (2K) mapped to go-smtp MaxMessageBytes to trip a 552 quickly.
## LABEL: this is a configured-limit variant, NOT the 32 MiB canonical default.
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
The sink accepts `EHLO/HELO/MAIL/RCPT/DATA`, reads `DATA` until the canonical `<CRLF>.<CRLF>`, un-stuffs leading dots, and records the exact delivered bytes (visible control chars + hexdump). It advertises **no** STARTTLS, matching `attempt_starttls no` on the relay.
```python
#!/usr/bin/env python3
# Capturing SMTP sink (stdlib raw sockets only; smtpd removed in Py3.12+).
# Listens on 127.0.0.1:<port>, speaks minimal ESMTP, and for every DATA message
# records the EXACT delivered bytes (post decode+re-encode produced by maddy's
# smtp_downstream relay via conn.Data) to a per-message .bin file plus a human log.
# Does NOT advertise STARTTLS. Handles multiple transactions per connection.
import socket, socketserver, sys, os, threading, time

PORT = int(sys.argv[1]) if len(sys.argv) > 1 else 2526
DELIV_DIR = sys.argv[2] if len(sys.argv) > 2 else "/tmp/maddy-smtp/delivered"
HUMAN_LOG = sys.argv[3] if len(sys.argv) > 3 else "/tmp/maddy-smtp/sink_delivered.log"
os.makedirs(DELIV_DIR, exist_ok=True)
_seq = [0]
_seq_lock = threading.Lock()

def vis(b: bytes) -> str:
    # make control chars visible: CR -> <CR>, LF -> <LF>\n, tab -> <TAB>
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
        # read one CRLF-terminated line from a buffer object (list holding bytes)
        while b"\r\n" not in buf[0]:
            chunk = self.request.recv(4096)
            if not chunk:
                return None
            buf[0] += chunk
        line, _, rest = buf[0].partition(b"\r\n")
        buf[0] = rest
        return line + b"\r\n"
    def read_data(self, buf):
        # read until canonical <CRLF>.<CRLF> (RFC-strict sink); returns raw bytes incl terminator handling
        data = b""
        # account for data already buffered
        while True:
            if buf[0]:
                data += buf[0]; buf[0] = b""
            # check terminator
            if data == b".\r\n" or data.endswith(b"\r\n.\r\n"):
                break
            chunk = self.request.recv(4096)
            if not chunk:
                break
            data += chunk
        # strip the trailing dot-line terminator, unstuff leading dots per RFC 5321 4.5.2
        if data.endswith(b"\r\n.\r\n"):
            body = data[:-3]  # keep trailing CRLF of last body line, drop ".\r\n"
        elif data == b".\r\n":
            body = b""
        else:
            body = data
        # unstuff: any line beginning with '.' has one dot removed
        out = bytearray(); i = 0; atbol = True
        while i < len(body):
            c = body[i]
            if atbol and c == 0x2e and i+1 < len(body):
                i += 1  # drop one stuffing dot
                atbol = False
                continue
            out.append(c)
            atbol = (c == 0x0a)
            i += 1
        return bytes(out), data
    def handle(self):
        peer = "%s:%d" % self.client_address
        self.send("220 sink.local ESMTP capturing-sink ready\r\n")
        buf = [b""]
        mail_from = None; rcpts = []
        while True:
            line = self.readline(buf)
            if line is None:
                break
            cmd = line.rstrip(b"\r\n")
            up = cmd.upper()
            if up.startswith(b"EHLO"):
                # multiline 250; deliberately NO STARTTLS advertised
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
                    _seq[0] += 1
                    n = _seq[0]
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
                mail_from = None; rcpts = []
                self.send("250 2.0.0 Ok\r\n")
            elif up.startswith(b"NOOP"):
                self.send("250 2.0.0 Ok\r\n")
            elif up.startswith(b"QUIT"):
                self.send("221 2.0.0 Bye\r\n")
                break
            else:
                self.send("250 2.0.0 Ok\r\n")

class Srv(socketserver.ThreadingTCPServer):
    allow_reuse_address = True
    daemon_threads = True

if __name__ == "__main__":
    with Srv(("127.0.0.1", PORT), Handler) as s:
        sys.stderr.write("sink listening on 127.0.0.1:%d (delivered->%s)\n" % (PORT, DELIV_DIR)); sys.stderr.flush()
        s.serve_forever()
```

### 7.4 Byte-controlled raw SMTP client — `/tmp/maddy-smtp/rawclient.py`
Full control over `\r\n` vs bare `\n`, dot-stuffing, and the end-of-`DATA` sequence. Modes: `canonical`, `barelf`, `barelf-body`, `dotstuff`, `resembles`, `smuggle` (one pipelined `send()`), `backtoback`, `dropmid`, `longline`, `oversize`.
```python
#!/usr/bin/env python3
# Byte-controlled raw-socket SMTP client (stdlib socket only).
# Full control over \r\n vs bare \n, dot-stuffing, end-of-DATA sequence, and
# pipelining. Prints the COMPLETE client-side transcript with control chars visible.
# Usage: rawclient.py <mode> [host] [port]
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
        # drain whatever the server sends within the settle window
        self.s.settimeout(settle)
        data = b""
        try:
            while True:
                chunk = self.s.recv(4096)
                if not chunk:
                    break
                data += chunk
        except socket.timeout:
            pass
        if data:
            sys.stdout.write("[%s]\n%s\n" % (label, vis(data)))
            sys.stdout.flush()
        return data
    def send(self, b: bytes, label="C->S"):
        sys.stdout.write("[%s]\n%s\n" % (label, vis(b)))
        sys.stdout.flush()
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
            b"Subject: framing probe" + CRLF +
            CRLF +
            b"BODYLINE-ONE canonical body." + CRLF +
            b"BODYLINE-TWO second line." + CRLF)
    c.send(body, "C->S body")
    # end-of-DATA terminator: '.' + term, preceded by a line break of the chosen kind
    c.send(b"." + term, "C->S end-of-DATA terminator")
    c.recv_all("S->C 250?")
    c.send(b"QUIT" + CRLF); c.recv_all()

def barelf(c):
    # bare-LF everywhere in the DATA phase, incl the dot terminator: <LF>.<LF>
    c.recv_all("S->C greeting")
    c.send(b"EHLO client.test" + CRLF); c.recv_all()
    c.send(b"MAIL FROM:<sender@example.com>" + CRLF); c.recv_all()
    c.send(b"RCPT TO:<rcpt@example.com>" + CRLF); c.recv_all()
    c.send(b"DATA" + CRLF); c.recv_all("S->C 354")
    body = (b"From: sender@example.com" + LF +
            b"To: rcpt@example.com" + LF +
            b"Subject: framing probe" + LF +
            LF +
            b"BODYLINE-ONE canonical body." + LF +
            b"BODYLINE-TWO second line." + LF)
    c.send(body, "C->S body (bare-LF lines)")
    c.send(b"." + LF, "C->S end-of-DATA terminator (bare-LF <LF>.<LF>)")
    c.recv_all("S->C 250?")
    c.send(b"QUIT" + CRLF); c.recv_all()

def barelf_body(c):
    # canonical terminator, but some BODY lines use bare LF
    c.recv_all("S->C greeting")
    c.send(b"EHLO client.test" + CRLF); c.recv_all()
    c.send(b"MAIL FROM:<sender@example.com>" + CRLF); c.recv_all()
    c.send(b"RCPT TO:<rcpt@example.com>" + CRLF); c.recv_all()
    c.send(b"DATA" + CRLF); c.recv_all("S->C 354")
    body = (b"From: sender@example.com" + CRLF +
            b"To: rcpt@example.com" + CRLF +
            b"Subject: framing probe" + CRLF +
            CRLF +
            b"BODYLINE-ONE canonical body." + LF +   # bare LF
            b"BODYLINE-TWO second line." + LF)        # bare LF
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
            b"Subject: framing probe" + CRLF +
            CRLF +
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
            b"Subject: framing probe" + CRLF +
            CRLF +
            b" ." + CRLF +                 # leading space then dot: NOT a terminator
            b".not-a-terminator" + CRLF +  # dot followed by text (stuffed->'.not...')? no: single dot+text = unstuffed to 'not...'
            b"embedded . dot in middle" + CRLF +
            b"trailing dot." + CRLF)
    c.send(body, "C->S body (lines resembling terminator)")
    c.send(b"." + CRLF, "C->S end-of-DATA terminator (canonical)")
    c.recv_all("S->C 250?")
    c.send(b"QUIT" + CRLF); c.recv_all()

def smuggle(c):
    # SINGLE send(): first transaction ends with bare-LF <LF>.<LF>, then a pipelined
    # second MAIL/RCPT/DATA block with a SPOOFED sender that a strict peer would treat as body.
    c.recv_all("S->C greeting")
    payload = (
        b"EHLO client.test" + CRLF +
        b"MAIL FROM:<legit@example.com>" + CRLF +
        b"RCPT TO:<rcpt@example.com>" + CRLF +
        b"DATA" + CRLF +
        b"Subject: first (legit) message" + CRLF +
        CRLF +
        b"This is the visible first message body." + LF +
        b"." + LF +                                   # <LF>.<LF> bare-LF terminator (the smuggling boundary)
        b"MAIL FROM:<SPOOFED-attacker@evil.example>" + CRLF +
        b"RCPT TO:<victim@example.com>" + CRLF +
        b"DATA" + CRLF +
        b"Subject: SMUGGLED second message" + CRLF +
        CRLF +
        b"This body was smuggled past the DATA boundary." + CRLF +
        b"." + CRLF
    )
    c.send(payload, "C->S ONE pipelined send() (smuggling)")
    c.recv_all("S->C all replies", settle=1.5)
    c.send(b"QUIT" + CRLF); c.recv_all()

def dropmid(c):
    # open DATA, send some body, then DROP the connection without any terminator
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
    # one line far exceeding MaxLineLength (2000). No CRLF until after >2000 bytes.
    c.recv_all("S->C greeting")
    c.send(b"EHLO client.test" + CRLF); c.recv_all()
    c.send(b"MAIL FROM:<sender@example.com>" + CRLF); c.recv_all()
    c.send(b"RCPT TO:<rcpt@example.com>" + CRLF); c.recv_all()
    c.send(b"DATA" + CRLF); c.recv_all("S->C 354")
    c.send(b"Subject: long line probe" + CRLF + CRLF, "C->S header")
    longln = b"X" * 5000 + CRLF
    c.send(b"[5000 X chars follows] " + longln, "C->S over-long line (>2000)")
    c.send(b"." + CRLF, "C->S end-of-DATA terminator (canonical)")
    c.recv_all("S->C reply", settle=1.5)
    c.send(b"QUIT" + CRLF); c.recv_all()

def oversize(c):
    # a body larger than a (small) configured max_message_size to trigger 552
    c.recv_all("S->C greeting")
    c.send(b"EHLO client.test" + CRLF); c.recv_all()
    c.send(b"MAIL FROM:<sender@example.com>" + CRLF); c.recv_all()
    c.send(b"RCPT TO:<rcpt@example.com>" + CRLF); c.recv_all()
    c.send(b"DATA" + CRLF); c.recv_all("S->C 354")
    c.send(b"Subject: oversize probe" + CRLF + CRLF, "C->S header")
    # send ~5000 bytes of body in canonical lines
    chunk = (b"A" * 60 + CRLF) * 90   # ~5580 bytes
    c.send(chunk, "C->S oversize body (~5.5 KB)")
    c.send(b"." + CRLF, "C->S end-of-DATA terminator (canonical)")
    c.recv_all("S->C reply", settle=1.5)
    c.send(b"QUIT" + CRLF); c.recv_all()

def backtoback(c):
    # >=2 full transactions on ONE connection (no reconnect), lockstep.
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
    "dropmid": dropmid, "longline": longline, "oversize": oversize, "backtoback": backtoback,
}

if __name__ == "__main__":
    sys.stdout.write("### rawclient mode=%s target=%s:%d\n" % (MODE, HOST, PORT)); sys.stdout.flush()
    c = Client(HOST, PORT)
    try:
        MODES[MODE](c)
    finally:
        c.close()
    sys.stdout.write("### rawclient done (mode=%s)\n" % MODE); sys.stdout.flush()
```

### 7.5 Front proxy (normalize / reject) — `/tmp/maddy-smtp/proxy.py`
Listens on `127.0.0.1:2524`, forwards to maddy on `127.0.0.1:2525`. `normalize` rewrites bare `\n`→`\r\n` (tracking a `\r` across chunk boundaries); `reject` returns `500 5.5.2 bare newline rejected …` and closes on the first bare `\n`.
```python
#!/usr/bin/env python3
# Small TCP front proxy (stdlib socket/threading only).
# Listens on 127.0.0.1:<lport>, forwards to maddy 127.0.0.1:<rport>, in two modes:
#   normalize : rewrite bare '\n' (LF not preceded by CR) to '\r\n' on the client->server
#               stream before forwarding (the "clean" analogue; can MANUFACTURE a
#               canonical <CRLF>.<CRLF> boundary out of ambiguous bare-LF framing).
#   reject    : detect a bare '\n' in the client->server stream, emit a 5xx to the client
#               and close (RFC-strict enforcement UPSTREAM of maddy; maddy sees nothing).
import socket, sys, threading, selectors

MODE  = sys.argv[1] if len(sys.argv) > 1 else "normalize"
LPORT = int(sys.argv[2]) if len(sys.argv) > 2 else 2524
RHOST = sys.argv[3] if len(sys.argv) > 3 else "127.0.0.1"
RPORT = int(sys.argv[4]) if len(sys.argv) > 4 else 2525

def normalize_bare_lf(data: bytes, carry_prev_cr: bool):
    # convert any LF not immediately preceded by CR into CRLF.
    # carry_prev_cr tracks whether the previous byte (across chunk boundary) was CR.
    out = bytearray()
    prev_cr = carry_prev_cr
    for by in data:
        if by == 0x0a and not prev_cr:
            out += b"\r\n"
        else:
            out.append(by)
        prev_cr = (by == 0x0d)
    return bytes(out), prev_cr

def has_bare_lf(data: bytes, carry_prev_cr: bool):
    prev_cr = carry_prev_cr
    for by in data:
        if by == 0x0a and not prev_cr:
            return True, (by == 0x0d)
        prev_cr = (by == 0x0d)
    return False, prev_cr

def handle(client):
    try:
        upstream = socket.create_connection((RHOST, RPORT), timeout=10)
    except OSError as e:
        client.sendall(b"421 proxy cannot reach upstream\r\n"); client.close(); return
    sel = selectors.DefaultSelector()
    sel.register(client, selectors.EVENT_READ, "c")
    sel.register(upstream, selectors.EVENT_READ, "u")
    prev_cr = False
    alive = True
    while alive:
        for key, _ in sel.select(timeout=5):
            who = key.data
            sock = key.fileobj
            try:
                data = sock.recv(4096)
            except OSError:
                alive = False; break
            if not data:
                alive = False; break
            if who == "c":
                # client -> server (apply policy)
                if MODE == "reject":
                    bad, prev_cr = has_bare_lf(data, prev_cr)
                    if bad:
                        client.sendall(b"500 5.5.2 bare newline rejected by front proxy (RFC 5321 requires CRLF)\r\n")
                        alive = False; break
                    upstream.sendall(data)
                elif MODE == "normalize":
                    conv, prev_cr = normalize_bare_lf(data, prev_cr)
                    upstream.sendall(conv)
                else:
                    upstream.sendall(data)
            else:
                # server -> client (verbatim)
                client.sendall(data)
    try: upstream.close()
    except OSError: pass
    try: client.close()
    except OSError: pass

def main():
    ls = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    ls.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    ls.bind(("127.0.0.1", LPORT)); ls.listen(16)
    sys.stderr.write("proxy mode=%s listening 127.0.0.1:%d -> %s:%d\n" % (MODE, LPORT, RHOST, RPORT)); sys.stderr.flush()
    while True:
        conn, _ = ls.accept()
        threading.Thread(target=handle, args=(conn,), daemon=True).start()

if __name__ == "__main__":
    main()
```

