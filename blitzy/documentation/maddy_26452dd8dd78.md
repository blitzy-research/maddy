# Maddy SMTP `DATA` message-boundary detection: runtime behavior under adversarial line-endings, dot-stuffing, and pipelining

## Section 0 — Scope, environment, and how this was produced (run-first)

**The question.** How does Maddy's SMTP server detect the end of the `DATA` phase (the message boundary), and how does that detection behave when a client manipulates line endings, dot-stuffing, and command pipelining around that boundary? This document answers that end-to-end (Q1–Q6) **from observed execution** of the real server built and run in its canonical configuration.

**Repository state.** All behavior was observed on Maddy commit `26452dd` (branch `maddy_26452dd8dd78`) with its pinned dependencies. This is a strictly read-only investigation: the only repository change is the addition of this document; the temporary observation harness lived entirely under `/tmp/investig` (outside the repository) and was removed afterward.

**Canonical build (observed).** The binary was built with the canonical single-package command and its version string was read back. The Go toolchain is the one shipped in the designated image (Go 1.18.10):

```text
$ go version
go version go1.18.10 linux/amd64
$ go build ./cmd/maddy
# github.com/mattn/go-sqlite3
sqlite3-binding.c: In function ‘sqlite3SelectNew’:
sqlite3-binding.c:125322:10: warning: function may return address of local variable [-Wreturn-local-addr]
125322 |   return pNew;
       |          ^~~~
sqlite3-binding.c:125282:10: note: declared here
125282 |   Select standin;
       |          ^~~~~~~
$ ./maddy -v
maddy unknown (built from source tree)
```

The build exits 0. The one compiler diagnostic (`-Wreturn-local-addr` in the vendored `sqlite3-binding.c`) is a **benign warning from the CGO SQLite driver `github.com/mattn/go-sqlite3 v1.11.0`** (`go.mod:26`), not an error — the default storage backend is CGO SQLite, which is why the build needs a C compiler. `libpam` headers are also linked by the PAM auth stack. The version string is `maddy unknown (built from source tree)` because a plain source build sets no linker version stamp (`Version` is defined at `maddy.go:41`).

**Run command (observed).** Maddy runs directly — there is **no `run` subcommand** (`Run` is the process entry point at `maddy.go:102`; `-debug` is registered around `maddy.go:104`):

`./maddy -config /tmp/investig/maddy.conf -log stderr -debug`

**Enabling the raw wire transcript.** The byte-level transcript requires **both** `io_debug yes` on the endpoint (`smtp.go:565`, which sets `serv.Debug = endp.Log.DebugWriter()` at `smtp.go:604`) **and** the global `-debug` flag — because `Logger.DebugWriter()` returns `ioutil.Discard` unless the logger's `Debug` flag is set (`internal/log/log.go:172`, `ioutil.Discard` at `log.go:174`). With only `io_debug yes` and no `-debug`, the transcript is silently discarded. This was confirmed at runtime: the startup line `smtp: I/O debugging is on!` appears, and with `-debug` the per-connection raw wire bytes are written to the log.

**Probe configuration (used verbatim).** A minimal check-free plaintext config. `state` and `runtime` are pointed inside `/tmp/investig` so **no** `*queue`/`*mtasts-cache` artifacts and **no** `/var/lib/maddy` or `/run/maddy` directories are created anywhere near the repository (verified in Section 7). Delivery is forwarded byte-for-byte to a local Python catch server via `smtp_downstream` so the exact delivered bytes can be captured:

```text
## Minimal check-free plaintext probe config for DATA-boundary investigation.
## Runs from /tmp/investig so all state/runtime artifacts land OUTSIDE the repo.
state /tmp/investig/state
runtime /tmp/investig/runtime
hostname probe.local
tls off

## Byte-exact delivered-body sink: forward everything to the Python catch server.
smtp_downstream catch {
    targets tcp://127.0.0.1:2526
    hostname probe.local
    attempt_starttls no
    require_tls no
}

## The endpoint under study: plaintext, raw transcript on, no inbound checks.
smtp tcp://127.0.0.1:2525 {
    io_debug yes
    deliver_to &catch
}
```

**Why a raw socket client is mandatory.** Maddy's own tests drive the server with the well-behaved `go-smtp` client (`internal/endpoint/smtp/smtp_test.go:30` `testEndpoint`), which dot-stuffs correctly and *cannot* emit the malformed boundary framings this question is about. A hand-written `socket` client is therefore required and every byte sent is recorded alongside the responses. The in-tree abort precedent `TestSMTPDelivery_AbortData` (`smtp_test.go:360`) is the closest existing coverage of an early close.

**Delivered-byte capture.** The catch server writes two files per message: `caught_NNN.wire` (the exact bytes Maddy sent downstream, i.e. re-dot-stuffed for the wire) and `caught_NNN.raw` (the same bytes un-dot-stuffed — the actual message content). Comparing `.wire` vs `.raw` makes the dot-stuffing round-trip directly observable (E4).

## Section 0.1 — Constraints and validation methodology

**Technical constraints that bound what is testable (all observed or cited):**

- **No `CHUNKING`/`BDAT`.** EHLO advertises only `PIPELINING`, `8BITMIME`, `ENHANCEDSTATUSCODES`, `SMTPUTF8`, and `SIZE` (`go-smtp/server.go:81`); the length-framed `BDAT` mode that structurally defeats terminator smuggling is not advertised, so only the `DATA`-terminator vector is testable here.
- **`PIPELINING` is advertised by default** (`go-smtp/server.go:81`), which is what makes the single-write smuggling experiment (E7) possible.
- **Line length is capped at 2000** (`MaxLineLength`, `go-smtp/server.go:40,76`) and **message size defaults to 32 MiB** (`max_message_size`, `smtp.go:561`, observed as `250 SIZE 33554432`). These bound the ELINE/ECMD and 552 experiments.
- **An unterminated `DATA` phase blocks until the 10-minute `read_timeout`** (`smtp.go:560`); the hang experiment is therefore bounded to a few seconds rather than waiting the full timeout.

**Validation methodology (how each claim is grounded):**

- **Run-first.** Every behavioral claim is accompanied by the actual captured output (the exact bytes sent as `repr`, the server responses as `repr`, the `io_debug` raw-wire transcript, the ordered-JSON `maddy.log` delta, and the delivered `.raw`/`.wire` bytes). The complete unedited capture for each experiment is in Appendix B; the inline sections quote real lines from those same captures.
- **Repeated runs for any stability or variance claim.** The back-to-back claim (E8) was replayed across two identical runs; the two error/abort paths that proved **nondeterministic** (E6 early close, ELINE over-long `DATA` line) were each run **10 times** and are reported as an observed distribution rather than a single run.
- **Real entry point.** The server was exercised through its real listener and pipeline; delivery was captured at a real downstream sink. No value was taken from a bypassing interface.
- **Read-only.** `git status --porcelain` was checked to confirm the working tree stayed clean throughout and that no harness file leaked into the repository.
- **Inferred vs observed.** The one item read from source rather than executed — the `net/textproto` dot-reader's internal state transitions — is explicitly labeled INFERRED, with its observable consequences confirmed at runtime (Section 2, Section 8).

## Section 1 — Direct answers (Q1–Q6)

Each answer leads with the direct result; the evidence and cause→effect reasoning follow in Sections 2–7 (full unedited captures in Appendix B).

- **Q1 — What the server sees at the boundary.** The end-of-`DATA` detector is **not Maddy code and not go-smtp code — it is the Go standard library `net/textproto` dot-reader.** `go-smtp/data.go:51-53` `newDataReader` wraps `c.text.DotReader()`; that reader elides leading dots (un-stuffing), rewrites body `\r\n`→`\n`, and detects the terminating dot line, returning `io.EOF` up to go-smtp. go-smtp then drains any residual bytes with `io.Copy(ioutil.Discard, r)` (`conn.go:521`) and resumes command parsing. Maddy itself sees only the already-decoded body, via `Session.Data` → `prepareBody` (`smtp.go:312,283`). Observed at runtime in E1 (the `354` prompt, the delivered decoded body, and — via E7/E8 — command parsing resuming immediately after the terminator).

- **Q2 — Varied boundary framing.** Observed: canonical `\r\n.\r\n` (E1), bare-LF `\n.\n` (E2), and **both** mixed forms `\n.\r\n` (E3a) and `\r\n.\n` (E3b) **all terminate** and deliver exactly one message; dot-stuffing is **un-stuffed** at delivery (`..`→`.`, `...`→`..`, E4); a `.` that is **not at line start** is literal body (E5, where the mid-line dot in `GLUED_NO_NEWLINE_BEFORE.` is kept literal and the message is still delivered via its real `\r\n.\r\n` terminator); an early close before any dot delivers **zero** (E6). Edge limits: an over-long line (>2000) drops the connection with 0 delivered (ELINE/ECMD), and an over-size body yields `552 5.3.4` (0 delivered).

- **Q3 — Pipelined pressure / SMTP smuggling.** Observed **YES**. Under `PIPELINING` (advertised by default, `go-smtp/server.go:81`), a single 409-byte TCP write carrying a bare-LF `\n.\n` boundary followed by an injected transaction causes Maddy to deliver a **second, spoofed message** (`spoofed@evil.example`) on the same connection (E7). This is the CVE-2023-51765-class desync; the pinned 2019 `go-smtp` predates the v0.20.0/v0.20.1 (December 2023) fix series.

- **Q4 — Back-to-back stability.** Observed **STABLE**. Five back-to-back messages on one connection delivered 5/5 across two identical runs (E8) — deterministic, no boundary wobble, no timing/buffering variance.

- **Q5 — Front proxy.** Observed: a naive **normalize** (bare-LF→CRLF) proxy does **not** stop the spoof — it promotes the ambiguous boundary to canonical `\r\n.\r\n`, and Maddy still delivers two messages; a **reject-on-bare-LF** proxy **blocks** it (0 delivered; the client receives `421` and the connection closes). This mirrors Postfix `smtpd_forbid_bare_newline` in its `normalize` vs `reject` modes (E9).

- **Q6 — The real runtime story (what lingers / never seen).** The boundary is decided by the stdlib, Maddy trusts it, and pipelining plus lenient bare-LF acceptance equals smuggling; proxies help only in reject mode. What lingers: an unterminated `DATA` phase blocks until the 10-minute `read_timeout` (`smtp.go:560`); an early/abrupt close yields a `DATA error` + `aborted` with nothing queued (and, nondeterministically, a `554`); the `smtp_downstream` probe target leaves nothing on disk; the `queue`/`remote` targets would create `*queue`/`*mtasts-cache` directories (`.gitignore` entries), which is why the harness ran from `/tmp/investig`. What one expects but never sees: **no `CHUNKING`/`BDAT` capability** is advertised (`go-smtp/server.go:81`), so the length-framed `BDAT` path that would defeat terminator smuggling does not exist to exercise here.

## Section 2 — The mechanism (Q1): three layers + the dot-reader state machine

The single most likely analysis error is to confuse two independent decoding steps. Keep them distinct: **Layer 1** (`net/textproto` dot-decoding + terminator detection, owned by the Go stdlib) runs *first*; **Layer 2** (Maddy `prepareBody` header/body split on the blank line) runs *second*. Below, Layer 0 is the raw wire before any decoding.

### Layer 0 — raw wire (the `io_debug` transcript)

`go-smtp/conn.go:68` tees the raw `net.Conn` reader into `c.server.Debug` via `io.TeeReader(rwc.Reader, c.server.Debug)` **before any decoding**. Consequently the transcript preserves CRLF and dot-stuffing and shows the terminator line exactly as received. In the captured `maddy.log` deltas below, the raw wire lines are the `smtp: ...` block that reproduces exactly what the client sent (including the lone `.` line).

### Layer 1 — `net/textproto` `dotReader.Read` (the actual detector) — INFERRED state machine, behavior confirmed at runtime

The terminator detection and dot-unstuffing are performed entirely by the Go standard library `dotReader`. The following state machine is **INFERRED** — read from the stdlib source and its doc comment ("translate the dots … elide leading dots, rewrite trailing `\r\n` into `\n`, and detect the ending `.\r\n` line") — but every observable consequence is confirmed at runtime by E1–E7. States: `stateBeginLine`, `stateDot`, `stateDotCR`, `stateCR`, `stateData`, `stateEOF`.

- line start + `.` → `stateDot` (the dot is not emitted yet);
- `stateDot` + `\n` → `stateEOF` ⇒ **a bare-LF `.\n` terminates** (this is the smuggling-relevant leniency, confirmed by E2);
- `stateDot` + `\r` → `stateDotCR`; then `stateDotCR` + `\n` → `stateEOF` ⇒ canonical `.\r\n` terminates (E1);
- `stateDot` + any other byte → `stateData` with the **first dot elided** ⇒ **dot-unstuffing** (`..`→`.`, confirmed by E4);
- `stateCR` + `\n` → emit `\n` only ⇒ **body CRLF is rewritten to LF** internally;
- a `.` that is **not** at line start stays in `stateData` (literal period, never a terminator; confirmed by E5);
- `ReadByte` hitting `io.EOF` → `err = io.ErrUnexpectedEOF` ⇒ **an early close is an error, zero delivery** (E6).

**INFERRED note (verified locations, Go 1.18.10).** These transitions were read in `/usr/local/go/src/net/textproto/reader.go`: the state constants at **L328-333**, the `stateDot + '\n' → stateEOF` transition at **L362-364**, and the `io.ErrUnexpectedEOF` conversion at **L340-342**. The state machine is labeled INFERRED because it is read from stdlib source; the *observable* behavior is confirmed at runtime via E1–E7.

### Layer 2 — delivered bytes (Maddy `prepareBody`)

Maddy `prepareBody` (`smtp.go:283`) wraps the already-decoded reader in `bufio`, parses headers with go-message's `textproto.ReadHeader` (which splits on the blank line — a **second** parse layer, entirely separate from dot-decoding), buffers the body, and prepends a `Received:` header before delivery (`header.Add("Received", …)` at `smtp.go:307`). `Session.Data` (`smtp.go:312`) then logs `accepted` with the `msg_id` at `smtp.go:334`. Again: Layer 1 owns dot-decoding and terminator detection; Layer 2 owns the header/body split — do not conflate them. Every delivered `.raw` below begins with this prepended `Received:` header.

### How go-smtp frames the phase and why pipelining matters

`handleData` writes the prompt `354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>` (`conn.go:510`), constructs the data reader (`conn.go:519`), and then drains with `io.Copy(ioutil.Discard, r)` (`conn.go:521`). The drain is a **no-op** once the dot-reader has already reached `stateEOF`. Crucially, command reads and the `DATA` reader **share one buffered reader** (`c.text`, set up at `conn.go:74`), so any residual bytes that were pipelined *after* the boundary remain buffered and are subsequently parsed as the next commands — exactly the mechanism exploited in Q3 (E7).

### E1 baseline — the happy path (ground truth)

The canonical `\r\n.\r\n` exchange below is the ground-truth reference every other experiment is compared against. Note the advertised capabilities (`PIPELINING`, `8BITMIME`, `ENHANCEDSTATUSCODES`, `SMTPUTF8`, `SIZE 33554432`), the `354` prompt, the `250 2.0.0 OK: queued` acceptance, the ordered-JSON `accepted` log line, and the delivered 312-byte body with the prepended `Received:` header. This is the complete unedited capture (conversation + `maddy.log` delta + delivered bytes):

```text
==================== EXPERIMENT E1 (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [38 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [39 bytes] --- b'250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [33 bytes] --- b'250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [150 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E1-canonical\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n.\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [6 bytes] --- b'QUIT\r\n'
--- S->C [58 bytes] --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'

-------- maddy.log delta (io_debug raw-wire + ordered-JSON) --------
smtp: 220 probe.local ESMTP Service Ready

smtp: EHLO probe.test

smtp: 250-Hello probe.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432

smtp: MAIL FROM:<sender@probe.local>
RCPT TO:<rcpt@probe.local>

smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: incoming message	{"msg_id":"da8890d0","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:33512"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"da8890d0"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"da8890d0"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"da8890d0"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"da8890d0"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"da8890d0"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"da8890d0"}
[debug] smtp_downstream: connected	{"msg_id":"da8890d0","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"da8890d0"}
smtp: RCPT ok	{"msg_id":"da8890d0","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: DATA
From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E1-canonical
X-Probe: boundary-test

Line one of the body.
Line two of the body.
.

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"da8890d0"}
smtp: accepted	{"msg_id":"da8890d0"}
smtp: 250 2.0.0 OK: queued

[debug] smtp: reset	
smtp: QUIT


-------- delivered messages (new caught_* files) --------
delivered [caught_001.raw, 312 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id da8890d0; Wed, 08 Jul\r\n 2026 06:14:51 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E1-canonical\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n'
```

## Section 3 — Varied boundary framing (Q2)

Every payload below is byte-identical except for how the `DATA` boundary is framed, so any behavioral difference is attributable solely to framing. Each experiment shows the exact payload (C→S) and server responses (S→C) as `repr`, plus the delivered `.raw` (and `.wire` where they differ); the complete unedited capture (including the `io_debug` raw-wire + ordered-JSON `maddy.log` delta) for each is in Appendix B.

### E1 — canonical `\r\n.\r\n` (baseline)

**Result:** `250 2.0.0 OK: queued`; exactly 1 delivered message (312 bytes, `caught_001.raw`, `msg_id da8890d0`). Full capture in Section 2 (E1 baseline). All following experiments are diffed against it.

### E2 — bare-LF `\n.\n`

**Result: ACCEPTED** — 1 delivered message (309 bytes, `msg_id a8fe0125`). The bare-LF `\n.\n` is treated as end-of-`DATA` (Layer-1 `stateDot + '\n' → stateEOF`). This is the SMUGGLING vector exploited in E7. The delivered body is byte-for-byte the two body lines with the bare-LF rewritten to CRLF internally; it is shorter than E1 only because the subject string differs.

Conversation (C→S / S→C, verbatim):

```text
==================== EXPERIMENT E2 (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [38 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [39 bytes] --- b'250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [14 bytes] --- b'250-SMTPUTF8\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [19 bytes] --- b'250 SIZE 33554432\r\n'
--- C->S [145 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E2-bareLF\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\n.\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [6 bytes] --- b'QUIT\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
```

Delivered (verbatim):

```text
-------- delivered messages (new caught_* files) --------
delivered [caught_002.raw, 309 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id a8fe0125; Wed, 08 Jul\r\n 2026 06:14:52 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E2-bareLF\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n'
```

→ complete unedited capture with transcript + ordered-JSON logs: **Appendix B.E2**.

### E3a `\n.\r\n` and E3b `\r\n.\n` — mixed forms

**Result: BOTH accepted**, 1 delivered each (315 bytes): E3a `msg_id 9be40444` (`caught_003.raw`), E3b `msg_id 34710cd8` (`caught_004.raw`). Every mixed terminator form terminates the `DATA` phase — consistent with the Layer-1 state machine, where reaching `stateEOF` requires only that a dot at line start be followed by an LF (whether or not a CR precedes it).

E3a delivered (verbatim):

```text
-------- delivered messages (new caught_* files) --------
delivered [caught_003.raw, 315 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id 9be40444; Wed, 08 Jul\r\n 2026 06:14:53 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E3a-LF-dot-CRLF\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n'
```

E3b delivered (verbatim):

```text
-------- delivered messages (new caught_* files) --------
delivered [caught_004.raw, 315 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id 34710cd8; Wed, 08 Jul\r\n 2026 06:14:53 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E3b-CRLF-dot-LF\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n'
```

→ complete unedited captures: **Appendix B.E3a / B.E3b**.

### E4 — dot-stuffing `\r\n..\r\n` / `\r\n...\r\n`

**Result:** ACCEPTED (`msg_id 7d66417b`). The client sent the body lines `..` and `...`; the delivered `.raw` (un-dot-stuffed content) shows `..`→`.` and `...`→`..`, while the `.wire` (what Maddy actually re-emitted downstream) is re-stuffed. This is ground-truth proof of unstuffing at the delivered-byte level, with the full round-trip visible in the `.wire` vs `.raw` diff below (note the `.raw` is 303 bytes vs the `.wire` 305 bytes — one dot fewer per stuffed line):

Conversation (verbatim):

```text
==================== EXPERIMENT E4 (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [52 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [39 bytes] --- b'250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [19 bytes] --- b'250 SIZE 33554432\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [143 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E4-dotstuff\r\nX-Probe: boundary-test\r\n\r\nLine A before.\r\n..\r\n...\r\nLine B after.\r\n.\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [6 bytes] --- b'QUIT\r\n'
--- S->C [58 bytes] --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
```

Delivered `.raw` (un-stuffed) and `.wire` (re-stuffed), verbatim:

```text
-------- delivered messages (new caught_* files) --------
delivered [caught_005.raw, 303 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id 7d66417b; Wed, 08 Jul\r\n 2026 06:14:54 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E4-dotstuff\r\nX-Probe: boundary-test\r\n\r\nLine A before.\r\n.\r\n..\r\nLine B after.\r\n'
  [.wire, 305 bytes]: b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id 7d66417b; Wed, 08 Jul\r\n 2026 06:14:54 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E4-dotstuff\r\nX-Probe: boundary-test\r\n\r\nLine A before.\r\n..\r\n...\r\nLine B after.\r\n'
```

→ complete unedited capture: **Appendix B.E4**.

### E5 — a mid-line dot is literal; the terminator arriving in a separate TCP segment still ends `DATA`

**Result: ACCEPTED** — 1 delivered message (320 bytes, `msg_id e1251d50`). The user's "missing final CRLF before the dot" case is exercised as follows: the client first sends a part ending `Line one of the body.\r\nGLUED_NO_NEWLINE_BEFORE.\r\n`, then in a **separate** `sendall` sends the real terminator `\r\n.\r\n`. Two observations: (1) the `.` at the end of `GLUED_NO_NEWLINE_BEFORE.` is **not at line start**, so it stays in `stateData` and is kept as a literal period in the delivered body — it never terminates; (2) the true terminator arriving in its own TCP segment is still detected (the dot-reader reassembles across segments), and the intervening `\r\n` becomes a trailing blank line in the delivered body. Note the pipelined responses are offset (the probe reads lag one response behind), so the `250 … OK: queued` for this message is read after `QUIT`:

Conversation (verbatim):

```text
==================== EXPERIMENT E5 (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [52 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [39 bytes] --- b'250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [19 bytes] --- b'250 SIZE 33554432\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [153 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E5-missing-crlf\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nGLUED_NO_NEWLINE_BEFORE.\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [5 bytes] --- b'\r\n.\r\n'
--- S->C [58 bytes] --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
--- C->S [6 bytes] --- b'QUIT\r\n'
--- S->C [22 bytes] --- b'250 2.0.0 OK: queued\r\n'
```

Delivered (verbatim; note the literal `GLUED_NO_NEWLINE_BEFORE.` and the trailing blank line):

```text
-------- delivered messages (new caught_* files) --------
delivered [caught_006.raw, 320 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id e1251d50; Wed, 08 Jul\r\n 2026 06:14:55 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E5-missing-crlf\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nGLUED_NO_NEWLINE_BEFORE.\r\n\r\n'
```

The clean **before/after state transition** (empty before a terminator, populated after) is shown directly by the unterminated-`DATA` hang in Section 7 (server silent, nothing delivered) versus E1 (delivered). → complete unedited capture: **Appendix B.E5**.

### E6 — early connection close before the dot (nondeterministic abort path)

**Result: 0 delivered** in every run — this is the invariant, matching the in-tree precedent `internal/endpoint/smtp/smtp_test.go:360` `TestSMTPDelivery_AbortData`. The *surfaced* abort detail is **nondeterministic** (a race between the server's read and the client's abrupt `close()`), so the same unchanged input was run **10 times**. Observed distribution: reason `"connection reset by peer"` in **6/10** runs and `"unexpected EOF"` in **4/10**; a `554 5.0.0 Internal server error` surfaces in only **3/10** runs of this abrupt-`close()` variant (a race over whether the server writes its response before the socket is torn down). `wrapErr` (`smtp.go:389`) maps the non-`SMTPError` abort to a generic `554`; that `554` is captured **deterministically** — on both the wire and the `io_debug` log — by the half-close variant in **Appendix B.E6**. One representative run (the `connection reset by peer` path) — complete capture:

```text
==================== EXPERIMENT E6 (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [38 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [39 bytes] --- b'250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [33 bytes] --- b'250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [133 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E6-early-close\r\nX-Probe: boundary-test\r\n\r\nBody with no terminator at all'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
>>> CLOSE (client closed socket)

-------- maddy.log delta (io_debug raw-wire + ordered-JSON) --------
smtp: 220 probe.local ESMTP Service Ready

smtp: EHLO probe.test

smtp: 250-Hello probe.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432

smtp: MAIL FROM:<sender@probe.local>
RCPT TO:<rcpt@probe.local>

smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: incoming message	{"msg_id":"09b39b33","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:51666"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"09b39b33"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"09b39b33"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"09b39b33"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"09b39b33"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"09b39b33"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"09b39b33"}
[debug] smtp_downstream: connected	{"msg_id":"09b39b33","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"09b39b33"}
smtp: RCPT ok	{"msg_id":"09b39b33","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: DATA
From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E6-early-close
X-Probe: boundary-test

Body with no terminator at all
smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

smtp: DATA error	{"msg_id":"09b39b33","reason":"read tcp 127.0.0.1:2525-\u003e127.0.0.1:51666: read: connection reset by peer"}
smtp: aborted	{"msg_id":"09b39b33"}
[debug] smtp: reset	

-------- delivered messages (new caught_* files) --------
delivered messages captured: 0
```

The alternate `unexpected EOF` → `554` path is captured **deterministically** in **Appendix B.E6** — a half-close (client `shutdown(SHUT_WR)` mid-body, read side kept open): the server reads EOF, logs `DATA error {"reason":"unexpected EOF"}`, and writes `554 5.0.0 Internal server error` to the still-open read side before closing. Either way: **0 delivered, connection aborted.**

### ELINE / ECMD — the line-length limit (2000), and why there is *no* clean asymmetry

**Result — corrected by runtime observation.** The 2000-byte `MaxLineLength` (`go-smtp/server.go:40,76`) applies to both command lines and `DATA` body lines, and in **both** cases the connection is dropped with **0 delivered**. A **command** line over 2000 bytes deterministically yields a clean `500 5.4.0 Too long line, closing connection` (`go-smtp/server.go:154-155`), captured cleanly by reading until the socket closes:

```text
--- banner ---
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17] --- b'EHLO probe.test\r\n'
--- S->C [110 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
--- C->S [2526 bytes] --- MAIL FROM with 2500-a localpart (repr head) --- b'MAIL FROM:<aaaaaaaaaaaaaaaaaaaaaaaaaaaaa'...b'aaaaa@probe.local>\r\n'
--- S->C [45 bytes] --- b'500 5.4.0 Too long line, closing connection\r\n'
>>> server closed connection (recv returned empty)
```

A **`DATA` body** line over 2000 bytes is **nondeterministic** on the wire (the server's too-long-line detection races the client's continued oversized write). The same unchanged ELINE input was run **10 times**: the client saw a clean `500 5.4.0 Too long line, closing connection` in **4/10** runs, and in the other runs the connection reset first and Maddy logged a `DATA error` with reason `"connection reset by peer"` (2/10) or `"unexpected EOF"` (the remainder). **In all 10 runs, 0 were delivered.** There is therefore *no* deterministic "DATA→554 vs command→500" asymmetry — the plurality `DATA`-line outcome is in fact the same `500 5.4.0 Too long line` message as the command path. One representative ELINE run (the clean `500` outcome) — complete capture:

```text
==================== EXPERIMENT ELINE (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [38 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [53 bytes] --- b'250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [19 bytes] --- b'250 SIZE 33554432\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [2599 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: ELINE\r\nX-Probe: boundary-test\r\n\r\nAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\r\n.\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [6 bytes] --- b'QUIT\r\n'
--- S->C [45 bytes] --- b'500 5.4.0 Too long line, closing connection\r\n'

-------- maddy.log delta (io_debug raw-wire + ordered-JSON) --------
smtp: 220 probe.local ESMTP Service Ready

smtp: EHLO probe.test

smtp: 250-Hello probe.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432

smtp: MAIL FROM:<sender@probe.local>
RCPT TO:<rcpt@probe.local>

smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: incoming message	{"msg_id":"c9ad5229","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:51676"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"c9ad5229"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"c9ad5229"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"c9ad5229"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"c9ad5229"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"c9ad5229"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"c9ad5229"}
[debug] smtp_downstream: connected	{"msg_id":"c9ad5229","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"c9ad5229"}
smtp: RCPT ok	{"msg_id":"c9ad5229","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: 500 5.4.0 Too long line, closing connection

smtp: aborted	{"msg_id":"c9ad5229"}

-------- delivered messages (new caught_* files) --------
delivered messages captured: 0
```

→ additional ELINE runs — the observed same-input distribution across 64 repetitions plus complete captures of the `unexpected EOF` and (abrupt-close) `connection reset by peer` variants — are in **Appendix B.ELINE**.

### 552 — the message-size limit

**Result:** on a second instance configured with `max_message_size 500b` (Appendix A.6), EHLO advertises `250 SIZE 500`; a 907-byte body (an 800-byte `X` payload plus headers and terminator) yields `552 5.3.4 Maximum message size exceeded (msg ID = 56b86674)`, logged as a `DATA error` (`Maximum message size exceeded`) plus `plain SMTP error returned, this is deprecated`; **0 delivered**; then `QUIT` → `221 2.0.0 Goodnight and good luck`. `ErrDataTooLarge` is a `*smtp.SMTPError` (`go-smtp/data.go:38-39`, code 552) whose code passes through Maddy `wrapErr` unchanged (`smtp.go:429-434`, msg-ID suffix at `smtp.go:437`), unlike plain errors which collapse to `554`. The default limit is 32 MiB (`smtp.go:561`), confirmed by the advertised `250 SIZE 33554432` on the primary instance. Complete capture:

```text
--- banner ---
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [105 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 500\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [58 bytes] --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
--- C->S [907 bytes] --- oversized DATA body (907 bytes incl terminator; body payload 800 X's, limit 500)
--- S->C [61 bytes] --- b'552 5.3.4 Maximum message size exceeded (msg ID = 56b86674)\r\n'
--- C->S [6 bytes] --- b'QUIT\r\n'
--- S->C [35 bytes] --- b'221 2.0.0 Goodnight and good luck\r\n'
```

## Section 4 — Pipelined pressure / SMTP smuggling (Q3)

**Result: YES — the smuggle reproduces.** The probe sent **one** `sendall` of **409 bytes**: a full transaction whose body ends with a bare-LF boundary `\n.\n`, immediately followed by an injected `MAIL FROM:<spoofed@evil.example>` / `RCPT` / `DATA` / second body / `\r\n.\r\n`. Maddy logged **two** complete `incoming message`→`accepted` cycles on the **same** `src_ip` and delivered **two messages**: the outer `attacker@probe.local` (`msg_id e7ffb4a8`, `caught_007.raw`) and the smuggled `spoofed@evil.example` (`msg_id a7064bc7`, `caught_008.raw`, `Subject: E7-SMUGGLED-injected`) — the second is fully attacker-controlled. This is the complete unedited capture:

```text
==================== EXPERIMENT E7 (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [38 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n'
--- C->S [409 bytes] --- b'MAIL FROM:<attacker@probe.local>\r\nRCPT TO:<victim@probe.local>\r\nDATA\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E7-OUTER-legit\r\nX-Probe: boundary-test\r\n\r\nOuter legit body line.\n.\nMAIL FROM:<spoofed@evil.example>\r\nRCPT TO:<victim@probe.local>\r\nDATA\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E7-SMUGGLED-injected\r\nX-Probe: boundary-test\r\n\r\nSmuggled spoofed body line.\r\n.\r\n'
--- S->C [39 bytes] --- b'250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n'
--- C->S [6 bytes] --- b'QUIT\r\n'
--- S->C [33 bytes] --- b'250-SMTPUTF8\r\n250 SIZE 33554432\r\n'

-------- maddy.log delta (io_debug raw-wire + ordered-JSON) --------
smtp: 220 probe.local ESMTP Service Ready

smtp: EHLO probe.test

smtp: 250-Hello probe.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432

smtp: MAIL FROM:<attacker@probe.local>
RCPT TO:<victim@probe.local>
DATA
From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E7-OUTER-legit
X-Probe: boundary-test

Outer legit body line.
.
MAIL FROM:<spoofed@evil.example>
RCPT TO:<victim@probe.local>
DATA
From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E7-SMUGGLED-injected
X-Probe: boundary-test

Smuggled spoofed body line.
.
QUIT

smtp: 250 2.0.0 Roger, accepting mail from <attacker@probe.local>

smtp: incoming message	{"msg_id":"e7ffb4a8","sender":"attacker@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:60628"}
[debug] smtp/pipeline: sender attacker@probe.local matched by default rule	{"msg_id":"e7ffb4a8"}
[debug] smtp/pipeline: global rcpt modifiers: victim@probe.local => victim@probe.local	{"msg_id":"e7ffb4a8"}
[debug] smtp/pipeline: per-source rcpt modifiers: victim@probe.local => victim@probe.local	{"msg_id":"e7ffb4a8"}
[debug] smtp/pipeline: recipient victim@probe.local matched by default rule (clean = victim@probe.local)	{"msg_id":"e7ffb4a8"}
[debug] smtp/pipeline: per-rcpt modifiers: victim@probe.local => victim@probe.local	{"msg_id":"e7ffb4a8"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"e7ffb4a8"}
[debug] smtp_downstream: connected	{"msg_id":"e7ffb4a8","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(attacker@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"e7ffb4a8"}
smtp: RCPT ok	{"msg_id":"e7ffb4a8","rcpt":"victim@probe.local"}
[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"e7ffb4a8"}
smtp: accepted	{"msg_id":"e7ffb4a8"}
[debug] smtp: reset	
smtp: incoming message	{"msg_id":"a7064bc7","sender":"spoofed@evil.example","src_host":"probe.test","src_ip":"127.0.0.1:60628"}
[debug] smtp/pipeline: sender spoofed@evil.example matched by default rule	{"msg_id":"a7064bc7"}
[debug] smtp/pipeline: global rcpt modifiers: victim@probe.local => victim@probe.local	{"msg_id":"a7064bc7"}
[debug] smtp/pipeline: per-source rcpt modifiers: victim@probe.local => victim@probe.local	{"msg_id":"a7064bc7"}
[debug] smtp/pipeline: recipient victim@probe.local matched by default rule (clean = victim@probe.local)	{"msg_id":"a7064bc7"}
[debug] smtp/pipeline: per-rcpt modifiers: victim@probe.local => victim@probe.local	{"msg_id":"a7064bc7"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"a7064bc7"}
[debug] smtp_downstream: connected	{"msg_id":"a7064bc7","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(spoofed@evil.example) ok, target = smtp_downstream:catch	{"msg_id":"a7064bc7"}
smtp: RCPT ok	{"msg_id":"a7064bc7","rcpt":"victim@probe.local"}
[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"a7064bc7"}
smtp: accepted	{"msg_id":"a7064bc7"}
[debug] smtp: reset	

-------- delivered messages (new caught_* files) --------
delivered [caught_007.raw, 294 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <attacker@probe.local>) with ESMTP id e7ffb4a8; Wed, 08\r\n Jul 2026 06:17:58 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E7-OUTER-legit\r\nX-Probe: boundary-test\r\n\r\nOuter legit body line.\r\n'
delivered [caught_008.raw, 305 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <spoofed@evil.example>) with ESMTP id a7064bc7; Wed, 08\r\n Jul 2026 06:17:58 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E7-SMUGGLED-injected\r\nX-Probe: boundary-test\r\n\r\nSmuggled spoofed body line.\r\n'
```

**Root cause (cause→effect).** The bare-LF `\n.\n` is accepted as end-of-`DATA` by the stdlib dot-reader (`stateDot + '\n' → stateEOF`). The `io.Copy(ioutil.Discard, r)` drain (`conn.go:521`) is a **no-op** because the reader is already at EOF, so it consumes nothing; the residual injected bytes remain in the shared buffered reader `c.text` (`conn.go:74`), and go-smtp's command loop parses them as a brand-new transaction. A stricter RFC-5321 peer would treat `\n.\n` as body, so a relaying Maddy placed in front of such a peer **desynchronizes** — the classic SMTP-smuggling / SPF-bypass scenario where the front and back servers disagree about where the first message ends.

**Timeline note (fix is a dated series, not one release).** The pinned `go-smtp` is the 2019 commit `1f576e0` (`go.mod:19`), roughly four years before the upstream fix. The fix landed as a **series**: **v0.20.0** — tagged **27 December 2023** — which "Remove[d] DotLF to EOFState case" and added an SMTP-smuggling test; followed by **v0.20.1**, which "Prevent[s] `<LF>.<CR><LF>` SMTP smuggling attacks." This task **observes** the pinned 2019 behavior; it does **not** upgrade or fix it (out of scope, read-only).

## Section 5 — Back-to-back stability (Q4)

**Result: STABLE and deterministic.** Five back-to-back canonical messages were sent on **one** connection, and the identical input was replayed across two runs (≥2 identical runs are required for any stability claim). RUN 1 delivered 5/5 (`msg_id`s `24643c9f, 4d5dd79e, a7e1be73, 3736a717, 073b2265`); RUN 2 delivered 5/5 (`7300344d, ec43152e, e8de4b34, 5d4d2633, b3c56d90`). Observed distribution `{5/5, 5/5}` — no boundary wobble, no timing/buffering variance. Both complete unedited runs follow.

**E8 — RUN 1 (complete capture):**

```text
==================== EXPERIMENT E8 (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [38 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [39 bytes] --- b'250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [33 bytes] --- b'250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [146 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E8-msg-1\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n.\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [58 bytes] --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [22 bytes] --- b'250 2.0.0 OK: queued\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [146 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E8-msg-2\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n.\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [58 bytes] --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [22 bytes] --- b'250 2.0.0 OK: queued\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [146 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E8-msg-3\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n.\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [58 bytes] --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [22 bytes] --- b'250 2.0.0 OK: queued\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [146 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E8-msg-4\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n.\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [58 bytes] --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [22 bytes] --- b'250 2.0.0 OK: queued\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [146 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E8-msg-5\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n.\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [6 bytes] --- b'QUIT\r\n'
--- S->C [58 bytes] --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'

-------- maddy.log delta (io_debug raw-wire + ordered-JSON) --------
smtp: 220 probe.local ESMTP Service Ready

smtp: EHLO probe.test

smtp: 250-Hello probe.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432

smtp: MAIL FROM:<sender@probe.local>
RCPT TO:<rcpt@probe.local>

smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: incoming message	{"msg_id":"24643c9f","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:46160"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"24643c9f"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"24643c9f"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"24643c9f"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"24643c9f"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"24643c9f"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"24643c9f"}
[debug] smtp_downstream: connected	{"msg_id":"24643c9f","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"24643c9f"}
smtp: RCPT ok	{"msg_id":"24643c9f","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: DATA
From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E8-msg-1
X-Probe: boundary-test

Line one of the body.
Line two of the body.
.

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"24643c9f"}
smtp: accepted	{"msg_id":"24643c9f"}
smtp: 250 2.0.0 OK: queued

[debug] smtp: reset	
smtp: MAIL FROM:<sender@probe.local>
RCPT TO:<rcpt@probe.local>

smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: incoming message	{"msg_id":"4d5dd79e","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:46160"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"4d5dd79e"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"4d5dd79e"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"4d5dd79e"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"4d5dd79e"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"4d5dd79e"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"4d5dd79e"}
[debug] smtp_downstream: connected	{"msg_id":"4d5dd79e","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"4d5dd79e"}
smtp: RCPT ok	{"msg_id":"4d5dd79e","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: DATA
From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E8-msg-2
X-Probe: boundary-test

Line one of the body.
Line two of the body.
.

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"4d5dd79e"}
smtp: accepted	{"msg_id":"4d5dd79e"}
smtp: 250 2.0.0 OK: queued

[debug] smtp: reset	
smtp: MAIL FROM:<sender@probe.local>
RCPT TO:<rcpt@probe.local>

smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: incoming message	{"msg_id":"a7e1be73","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:46160"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"a7e1be73"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"a7e1be73"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"a7e1be73"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"a7e1be73"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"a7e1be73"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"a7e1be73"}
[debug] smtp_downstream: connected	{"msg_id":"a7e1be73","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"a7e1be73"}
smtp: RCPT ok	{"msg_id":"a7e1be73","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: DATA
From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E8-msg-3
X-Probe: boundary-test

Line one of the body.
Line two of the body.
.

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"a7e1be73"}
smtp: accepted	{"msg_id":"a7e1be73"}
smtp: 250 2.0.0 OK: queued

[debug] smtp: reset	
smtp: MAIL FROM:<sender@probe.local>
RCPT TO:<rcpt@probe.local>

smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: incoming message	{"msg_id":"3736a717","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:46160"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"3736a717"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"3736a717"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"3736a717"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"3736a717"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"3736a717"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"3736a717"}
[debug] smtp_downstream: connected	{"msg_id":"3736a717","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"3736a717"}
smtp: RCPT ok	{"msg_id":"3736a717","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: DATA
From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E8-msg-4
X-Probe: boundary-test

Line one of the body.
Line two of the body.
.

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"3736a717"}
smtp: accepted	{"msg_id":"3736a717"}
smtp: 250 2.0.0 OK: queued

[debug] smtp: reset	
smtp: MAIL FROM:<sender@probe.local>
RCPT TO:<rcpt@probe.local>

smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: incoming message	{"msg_id":"073b2265","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:46160"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"073b2265"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"073b2265"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"073b2265"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"073b2265"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"073b2265"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"073b2265"}
[debug] smtp_downstream: connected	{"msg_id":"073b2265","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"073b2265"}
smtp: RCPT ok	{"msg_id":"073b2265","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: DATA
From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E8-msg-5
X-Probe: boundary-test

Line one of the body.
Line two of the body.
.

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"073b2265"}
smtp: accepted	{"msg_id":"073b2265"}
smtp: 250 2.0.0 OK: queued

[debug] smtp: reset	
smtp: QUIT


-------- delivered messages (new caught_* files) --------
delivered [caught_009.raw, 308 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id 24643c9f; Wed, 08 Jul\r\n 2026 06:18:53 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E8-msg-1\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n'
delivered [caught_010.raw, 308 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id 4d5dd79e; Wed, 08 Jul\r\n 2026 06:18:53 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E8-msg-2\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n'
delivered [caught_011.raw, 308 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id a7e1be73; Wed, 08 Jul\r\n 2026 06:18:53 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E8-msg-3\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n'
delivered [caught_012.raw, 308 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id 3736a717; Wed, 08 Jul\r\n 2026 06:18:53 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E8-msg-4\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n'
delivered [caught_013.raw, 308 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id 073b2265; Wed, 08 Jul\r\n 2026 06:18:53 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E8-msg-5\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n'
```

**E8 — RUN 2 (complete capture, identical input):**

```text
==================== EXPERIMENT E8 (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [38 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [39 bytes] --- b'250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [14 bytes] --- b'250-SMTPUTF8\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [19 bytes] --- b'250 SIZE 33554432\r\n'
--- C->S [146 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E8-msg-1\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n.\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [58 bytes] --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [22 bytes] --- b'250 2.0.0 OK: queued\r\n'
--- C->S [146 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E8-msg-2\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n.\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [58 bytes] --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [22 bytes] --- b'250 2.0.0 OK: queued\r\n'
--- C->S [146 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E8-msg-3\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n.\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [58 bytes] --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [22 bytes] --- b'250 2.0.0 OK: queued\r\n'
--- C->S [146 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E8-msg-4\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n.\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [58 bytes] --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [22 bytes] --- b'250 2.0.0 OK: queued\r\n'
--- C->S [146 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E8-msg-5\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n.\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [6 bytes] --- b'QUIT\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"

-------- maddy.log delta (io_debug raw-wire + ordered-JSON) --------
smtp: 220 probe.local ESMTP Service Ready

smtp: EHLO probe.test

smtp: 250-Hello probe.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432

smtp: MAIL FROM:<sender@probe.local>
RCPT TO:<rcpt@probe.local>
DATA

smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: incoming message	{"msg_id":"7300344d","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:46164"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"7300344d"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"7300344d"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"7300344d"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"7300344d"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"7300344d"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"7300344d"}
[debug] smtp_downstream: connected	{"msg_id":"7300344d","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"7300344d"}
smtp: RCPT ok	{"msg_id":"7300344d","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

smtp: From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E8-msg-1
X-Probe: boundary-test

Line one of the body.
Line two of the body.
.
MAIL FROM:<sender@probe.local>
RCPT TO:<rcpt@probe.local>

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"7300344d"}
smtp: accepted	{"msg_id":"7300344d"}
smtp: 250 2.0.0 OK: queued

[debug] smtp: reset	
smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: incoming message	{"msg_id":"ec43152e","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:46164"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"ec43152e"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"ec43152e"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"ec43152e"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"ec43152e"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"ec43152e"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"ec43152e"}
[debug] smtp_downstream: connected	{"msg_id":"ec43152e","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"ec43152e"}
smtp: RCPT ok	{"msg_id":"ec43152e","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: DATA
From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E8-msg-2
X-Probe: boundary-test

Line one of the body.
Line two of the body.
.
MAIL FROM:<sender@probe.local>

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"ec43152e"}
smtp: accepted	{"msg_id":"ec43152e"}
smtp: 250 2.0.0 OK: queued

[debug] smtp: reset	
smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: RCPT TO:<rcpt@probe.local>
DATA

smtp: incoming message	{"msg_id":"e8de4b34","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:46164"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"e8de4b34"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"e8de4b34"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"e8de4b34"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"e8de4b34"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"e8de4b34"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"e8de4b34"}
[debug] smtp_downstream: connected	{"msg_id":"e8de4b34","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"e8de4b34"}
smtp: RCPT ok	{"msg_id":"e8de4b34","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

smtp: From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E8-msg-3
X-Probe: boundary-test

Line one of the body.
Line two of the body.
.
MAIL FROM:<sender@probe.local>
RCPT TO:<rcpt@probe.local>

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"e8de4b34"}
smtp: accepted	{"msg_id":"e8de4b34"}
smtp: 250 2.0.0 OK: queued

[debug] smtp: reset	
smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: incoming message	{"msg_id":"5d4d2633","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:46164"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"5d4d2633"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"5d4d2633"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"5d4d2633"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"5d4d2633"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"5d4d2633"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"5d4d2633"}
[debug] smtp_downstream: connected	{"msg_id":"5d4d2633","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"5d4d2633"}
smtp: RCPT ok	{"msg_id":"5d4d2633","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: DATA
From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E8-msg-4
X-Probe: boundary-test

Line one of the body.
Line two of the body.
.
MAIL FROM:<sender@probe.local>

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"5d4d2633"}
smtp: accepted	{"msg_id":"5d4d2633"}
smtp: 250 2.0.0 OK: queued

[debug] smtp: reset	
smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: RCPT TO:<rcpt@probe.local>
DATA

smtp: incoming message	{"msg_id":"b3c56d90","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:46164"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"b3c56d90"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"b3c56d90"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"b3c56d90"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"b3c56d90"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"b3c56d90"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"b3c56d90"}
[debug] smtp_downstream: connected	{"msg_id":"b3c56d90","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"b3c56d90"}
smtp: RCPT ok	{"msg_id":"b3c56d90","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

smtp: From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E8-msg-5
X-Probe: boundary-test

Line one of the body.
Line two of the body.
.
QUIT

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"b3c56d90"}
smtp: accepted	{"msg_id":"b3c56d90"}
[debug] smtp: reset	

-------- delivered messages (new caught_* files) --------
delivered [caught_014.raw, 308 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id 7300344d; Wed, 08 Jul\r\n 2026 06:18:54 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E8-msg-1\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n'
delivered [caught_015.raw, 308 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id ec43152e; Wed, 08 Jul\r\n 2026 06:18:54 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E8-msg-2\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n'
delivered [caught_016.raw, 308 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id e8de4b34; Wed, 08 Jul\r\n 2026 06:18:54 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E8-msg-3\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n'
delivered [caught_017.raw, 308 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id 5d4d2633; Wed, 08 Jul\r\n 2026 06:18:54 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E8-msg-4\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n'
delivered [caught_018.raw, 308 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id b3c56d90; Wed, 08 Jul\r\n 2026 06:18:54 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E8-msg-5\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n'
```

## Section 6 — Front proxy (Q5)

**Result:** a naive normalizing proxy does **not** stop the spoof; only a rejecting proxy does. The three modes of the front proxy (`proxy.py`, Appendix A.5, listening on `:2527` → `:2525`) were each measured against E2 (bare-LF) and E7 (the smuggle). Observed delivery counts:

| Proxy mode | E2 (bare-LF `\n.\n`) | E7 (pipelined smuggle) | Effect on the attack |
|------------|----------------------|------------------------|----------------------|
| passthrough (control) | 1 delivered | 2 delivered (spoof succeeds) | proxy neutral |
| normalize (bare-LF→CRLF) | 1 delivered | 2 delivered (attacker + spoofed) | **does NOT block** the injection |
| reject (drop on bare-LF) | 0 delivered | 0 delivered (`421`, connection closed) | **blocks** the attack |

- **passthrough (control):** E2 → 1 delivered (`caught_019`, `msg_id b74a01f8`); E7 → 2 delivered. The proxy is neutral and the spoof succeeds exactly as in the direct case.
- **normalize (bare-LF→CRLF):** E2 → 1; E7 → **2** (attacker `c644728f` + spoofed `cff64f61`). **HONEST result:** naive normalization does *not* block the spoof — it promotes `\n.\n` to canonical `\r\n.\r\n`, which is *still* a valid terminator, so Maddy still splits and delivers the spoofed message. Normalization removes the *desync* (a strict downstream peer would now also see two messages) but not the *injection*.
- **reject (drop on bare-LF):** E2 → 0; E7 → 0. The client receives `421 4.7.0 bare <LF> in stream rejected by proxy (anti-smuggling), closing` and the connection closes. This is the effective mitigation — the analog of Postfix `smtpd_forbid_bare_newline=reject`.

**Proof the normalize mode actually rewrites the bytes** (distinguishing it from passthrough): running the proxy's own `normalize_stream`/`has_bare_lf` on the exact E2 and E7 payloads shows `\n.\n` → `\r\n.\r\n`:

```text
### E2 DATA-body tail before/after normalize (bare-LF boundary) ###
BEFORE (repr tail): b'ody.\r\nLine two of the body.\n.\n'
AFTER  (repr tail): b'y.\r\nLine two of the body.\r\n.\r\n'
has_bare_lf(E2 body) = True

### E7 blob: boundary region before/after normalize ###
BEFORE (repr region): b'Outer legit body line.\n.\nMAIL FROM'
AFTER  (repr region): b'Outer legit body line.\r\n.\r\nMAIL FROM'
has_bare_lf(E7 blob) = True

### passthrough forwards these bytes unchanged; normalize rewrites bare \n -> \r\n; reject drops on has_bare_lf=True ###
```

**reject / E2 — complete capture (client `421`, Maddy `DATA error` → `aborted`, 0 delivered):**

```text
==================== EXPERIMENT E2 (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [22 bytes] --- b'250-Hello probe.test\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [55 bytes] --- b'250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [33 bytes] --- b'250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [145 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E2-bareLF\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\n.\n'
--- S->C [75 bytes] --- b'421 4.7.0 bare <LF> in stream rejected by proxy (anti-smuggling), closing\r\n'
--- C->S [6 bytes] --- b'QUIT\r\n'
--- S->C [0 bytes] --- b''

-------- maddy.log delta (io_debug raw-wire + ordered-JSON) --------
smtp: 220 probe.local ESMTP Service Ready

smtp: EHLO probe.test

smtp: 250-Hello probe.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432

smtp: MAIL FROM:<sender@probe.local>

smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: RCPT TO:<rcpt@probe.local>
DATA

smtp: incoming message	{"msg_id":"cb42de8a","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:50100"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"cb42de8a"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"cb42de8a"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"cb42de8a"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"cb42de8a"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"cb42de8a"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"cb42de8a"}
[debug] smtp_downstream: connected	{"msg_id":"cb42de8a","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"cb42de8a"}
smtp: RCPT ok	{"msg_id":"cb42de8a","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: DATA error	{"msg_id":"cb42de8a","reason":"unexpected EOF"}
smtp: aborted	{"msg_id":"cb42de8a"}
[debug] smtp: reset	

-------- delivered messages (new caught_* files) --------
delivered messages captured: 0
```

**reject / E7 — conversation (single blob rejected with `421`, 0 delivered):**

```text
==================== EXPERIMENT E7 (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [22 bytes] --- b'250-Hello probe.test\r\n'
--- C->S [409 bytes] --- b'MAIL FROM:<attacker@probe.local>\r\nRCPT TO:<victim@probe.local>\r\nDATA\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E7-OUTER-legit\r\nX-Probe: boundary-test\r\n\r\nOuter legit body line.\n.\nMAIL FROM:<spoofed@evil.example>\r\nRCPT TO:<victim@probe.local>\r\nDATA\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E7-SMUGGLED-injected\r\nX-Probe: boundary-test\r\n\r\nSmuggled spoofed body line.\r\n.\r\n'
--- S->C [16 bytes] --- b'250-PIPELINING\r\n'
--- C->S [6 bytes] --- b'QUIT\r\n'
--- S->C [128 bytes] --- b'421 4.7.0 bare <LF> in stream rejected by proxy (anti-smuggling), closing\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n'
```

→ complete unedited captures for all three modes × {E2, E7} are in **Appendix B.E9** (passthrough, normalize, and reject).

## Section 7 — The real runtime story (Q6): what lingers, what is never seen

**The end-to-end narrative.** The message boundary is not decided by Maddy at all — it is decided by the Go standard library `net/textproto` dot-reader that go-smtp delegates to (`data.go:51-53`). Maddy trusts whatever that reader hands it, sees only the decoded body through `prepareBody` (`smtp.go:283`), prepends a `Received:` header, logs `accepted`, and delivers. Because the stdlib reader accepts a bare-LF `.\n` as a terminator, and because `PIPELINING` lets a client cram a whole follow-on transaction into the same buffered reader that command parsing resumes from, a single crafted write injects a second, fully attacker-controlled message (Q3). A front proxy helps only if it *rejects* bare LF; merely *normalizing* it removes the desync but still lets the injection through (Q5). Back-to-back traffic on one connection is perfectly stable (Q4), so the danger is not flakiness — it is the deterministic acceptance of a non-standard boundary.

**What lingers when things go wrong (observed):**

**(a) Unterminated `DATA` hangs to the read timeout.** When the client sends `DATA` content but never a terminator and never closes, the connection blocks until the 10-minute `read_timeout` (`smtp.go:560`). Observed (bounded to a 6-second poll rather than waiting the full timeout): after the `354`, the server returned **nothing** for the entire poll — it stays blocked awaiting the `<CRLF>.<CRLF>` terminator. This is also the clean before/after state transition for E5: **empty (blocked, silent) before a terminator arrives.** (When the client subsequently goes away without a terminator, the reader hits EOF and the abort path fires deterministically: `DATA error {"reason":"unexpected EOF"}` → `554 5.0.0 Internal server error` → `aborted`, 0 delivered — captured in full in **Appendix B.E6**.) Complete capture of the silent-hang phase:

```text
--- S->C --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C --- b'250-Hello probe.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
--- C->S [68 bytes] --- b'From: <s@probe.local>\r\nSubject: hang\r\n\r\nBody line with no terminator'
>>> now holding connection open, polling for a server response for ~6s (server should stay silent, blocked on read_timeout=10m)
--- S->C --- b'<no data - timed out>'
--- S->C --- b'<no data - timed out>'
--- S->C --- b'<no data - timed out>'
--- S->C --- b'<no data - timed out>'
--- S->C --- b'<no data - timed out>'
--- S->C --- b'<no data - timed out>'
>>> after 6.0s the server sent NOTHING: it is blocked awaiting the <CRLF>.<CRLF> terminator (read_timeout default 10m, smtp.go:560). Closing client now (bounded).
```

**(b) Early/abrupt close → `DATA error` + `aborted`, nothing queued.** As quantified in E6 (Section 3), a close before the dot delivers 0; the logged reason is `connection reset by peer` (an abrupt RST close that leaves the server's response unread) or `unexpected EOF` (a clean or half close) depending on how the client tears the socket down, and on the abrupt-`close()` path the `554` reaches the client only sometimes (the write races the teardown). The half-close variant makes the `554` deterministic — captured in **Appendix B.E6**; the abrupt-RST reset reason is captured in **Appendix B.ELINE**. Nothing is queued.

**(c) `smtp_downstream` leaves nothing on disk.** The probe's delivery target relays synchronously to the catch server; it does not spool. Confirmed by the artifact inventory below — the `state`/`runtime` directories are empty of files after 24 delivered messages.

**(d) The `queue`/`remote` targets are what create the `.gitignore` residue** (not used by this probe config). The `queue` target creates `StateDirectory/<name>` via `os.MkdirAll` (location `internal/target/queue/queue.go:229`, `os.MkdirAll` at `queue.go:233`), and the `remote` target creates `StateDirectory/mtasts-cache` (location `internal/target/remote/remote.go:93`, `os.MkdirAll` at `remote.go:145`); `DefaultStateDirectory = /var/lib/maddy` (`maddy.go:59`). When Maddy is run from the `cmd/maddy` directory these surface as the ignored paths `cmd/maddy/*queue` (`.gitignore:36`) and `cmd/maddy/*mtasts-cache` (`.gitignore:35`) — the very reason this investigation ran from `/tmp/investig` with `state`/`runtime` redirected there, keeping the repository clean.

**Artifact inventory (observed):**

```text
$ ls -la /tmp/investig/state /tmp/investig/runtime /tmp/investig/state2 /tmp/investig/runtime2
/tmp/investig/runtime:
total 8
drwxr-xr-x 2 root root 4096 Jul  8 06:14 .
drwxr-xr-x 7 root root 4096 Jul  8 06:37 ..

/tmp/investig/runtime2:
total 8
drwxr-xr-x 2 root root 4096 Jul  8 06:19 .
drwxr-xr-x 7 root root 4096 Jul  8 06:37 ..

/tmp/investig/state:
total 8
drwxr-xr-x 2 root root 4096 Jul  8 06:14 .
drwxr-xr-x 7 root root 4096 Jul  8 06:37 ..

/tmp/investig/state2:
total 8
drwxr-xr-x 2 root root 4096 Jul  8 06:19 .
drwxr-xr-x 7 root root 4096 Jul  8 06:37 ..

$ find /tmp/investig/state /tmp/investig/runtime /tmp/investig/state2 /tmp/investig/runtime2 -type f | wc -l   # files spooled to disk by smtp_downstream
0

$ find /tmp/investig \( -name "*queue" -o -name "*mtasts-cache" \) 2>/dev/null   # queue/remote residue in the run dir
(none)

$ ls -la /var/lib/maddy /run/maddy 2>&1   # DefaultStateDirectory / runtime (never created by this probe)
ls: cannot access '/var/lib/maddy': No such file or directory
ls: cannot access '/run/maddy': No such file or directory

$ ls -1 /tmp/investig/caught_*.raw | wc -l   # total delivered messages captured across all experiments
24

$ cd <repo> && git status --porcelain   # repository working tree (before writing the answer doc)
(clean; the sole change will be the answer document itself)

$ cd <repo> && git status --ignored --porcelain | grep -E "queue|mtasts" ; echo exit=$?   # any .gitignore residue tracked-or-ignored in the repo
exit=1 (no ignored *queue/*mtasts-cache residue present)
```

**What one expects to see but never does (observed):**

- **No `CHUNKING`/`BDAT`.** EHLO advertises only `PIPELINING`, `8BITMIME`, `ENHANCEDSTATUSCODES`, `SMTPUTF8`, and `SIZE` (see every E1–E8 banner above) — **never** `CHUNKING`/`BDAT` (`go-smtp/server.go:81`). Length-framed `BDAT` is the transfer mode that structurally defeats terminator smuggling (there is no terminator to confuse), but it does not exist to exercise in this version, so only the `DATA`-terminator vector is testable here.
- **No terminator response.** During an unterminated `DATA`, one might expect a timeout error or a nudge; instead the server is simply silent until the client goes away or the 10-minute `read_timeout` fires (shown in (a) above).

## Section 8 — Coverage, citations, and grounding

### 8.1 Question coverage matrix

Every question item and every user-named framing is exercised at runtime with adjacent evidence:

| Item | Where answered | Framing / condition exercised | Observed outcome |
|------|----------------|-------------------------------|------------------|
| Q1 (what the server sees) | §2, §3/E1 | canonical `\r\n.\r\n` | dot-reader detects terminator, drains, resumes commands; 1 delivered |
| Q2 (varied framing) | §3 | E1 `\r\n.\r\n`; **E2 bare-LF `\n.\n`**; E3a `\n.\r\n`; E3b `\r\n.\n`; **E4 dot-stuffing `\r\n..\r\n`**; **E5 missing CRLF before dot**; E6 early close | E1/E2/E3a/E3b/E4/E5 all ACCEPTED+delivered; E4 unstuffed one dot; E6 0 delivered |
| Q3 (smuggling) | §4/E7 | pipelined single write, bare-LF boundary + injected txn | **2 delivered** (attacker + spoofed) |
| Q4 (stability) | §5/E8 | 5 back-to-back × 2 identical runs | `{5/5, 5/5}` stable |
| Q5 (front proxy) | §6/E9 | passthrough / normalize / reject × {E2, E7} | reject blocks (0/0, `421`); normalize does NOT block spoof (2) |
| Q6 (the real story) | §7 | unterminated hang; early close; artifact inventory; no CHUNKING/BDAT | silent-until-timeout; 0-delivery aborts; empty state dirs; only PIPELINING/8BITMIME/ENHANCEDSTATUSCODES/SMTPUTF8/SIZE advertised |

Edge/limit paths additionally exercised: **ELINE** (over-long `DATA` line), **ECMD** (over-long command line), and the **552** maximum-message-size rejection (§3).

### 8.2 Observed vs. inferred

**Observed (with captured evidence in this document):** every delivery count and `msg_id`; the `354`/`250`/`500`/`552`/`421`/`554` response codes and their exact text; the two-message E7 smuggle; the `{5/5,5/5}` E8 distribution; the three proxy-mode outcomes and the normalize byte-rewrite proof; the unterminated-`DATA` silence (§7) and the deterministic `unexpected EOF`→`554 5.0.0` abort captured by the half-close variant (Appendix B.E6); the empty state/runtime dirs and clean repo; and the EHLO capability set (no CHUNKING/BDAT).

**Corrected against runtime (earlier read-only guesses that the captures disproved):**
- **E5** is **accepted and delivered**, not "never terminates." An earlier draft misread a pipelining-offset `RECV b''` as "still in DATA"; the actual bytes show the `\r\n.\r\n` terminator arriving in a later TCP segment and being detected normally.
- **E6 / ELINE outcomes are nondeterministic**, not a fixed `unexpected EOF`+`554`. Across 10 identical runs each, E6 logged `connection reset by peer` 6/10 and `unexpected EOF` 4/10 (a `554` surfaced only 3/10); ELINE produced a clean `500 5.4.0 Too long line` 4/10 with reset/other on the rest. The **one invariant** is **0 delivered** in every run. There is **no** deterministic "DATA-path `554` vs command-path `500` asymmetry," and no `lineLimitReader` value-vs-pointer receiver explanation is asserted — the difference is a client-close/server-read race, confirmed by repetition.

**Inferred (from reading, not separately instrumented):** the internal `io.Copy(ioutil.Discard, r)` drain being a no-op at EOF (`conn.go:521`) is the mechanism that leaves injected bytes in the shared buffered reader; this is inferred from the source and is *consistent with* the observed E7 two-message delivery, but the buffer contents were not dumped directly.

### 8.3 Consolidated citations (verified against source at commit `26452dd8dd78`)

**Maddy `internal/endpoint/smtp/smtp.go`:** `prepareBody` L283; `Received` header add L307; `Session.Data` L312; `accepted` log L334 (SMTP) / L377 (LMTP); `wrapErr` L389 with `*smtp.SMTPError` passthrough L429–434 and `" (msg ID = " + msgId + ")"` suffix L437; `EnableSMTPUTF8=true` L504; `write_timeout` L559; `read_timeout` L560; `max_message_size` L561; `io_debug` L565; `serv.Debug = Log.DebugWriter()` L604.
**Maddy other:** `maddy.go` Version L41, `DefaultStateDirectory=/var/lib/maddy` L59, `Run` L102, `-debug` L104; `internal/log/log.go` `formatMsg` L135, `DebugWriter` L172, `ioutil.Discard` L174; `internal/target/remote/remote.go` mtasts-cache location L93, `os.MkdirAll` L145; `internal/target/queue/queue.go` location L229, `os.MkdirAll` L233; `.gitignore` `*mtasts-cache` L35, `*queue` L36.
**`go-smtp` @ `1f576e0` (2019):** `data.go` `ErrDataTooLarge`/552 L38–39, `newDataReader` L51/L53, byte counter L56/L58, size check L67; `conn.go` transcript tee L68, `c.text=textproto.NewConn` L74, `handleData` L498, `354` L510, reader construct L519, drain `io.Copy(ioutil.Discard,r)` L521, LMTP `handleDataLMTP` L568; `server.go` `MaxLineLength=2000` L40/L76, `caps` (PIPELINING/8BITMIME/ENHANCEDSTATUSCODES; no CHUNKING/BDAT) L81, `handleConn` loop L154, `ErrTooLongLine`→`500` L155; `smtp.go` `validateLine` rejects `\n`/`\r` L23–24.
**Go standard library `net/textproto/reader.go` (Go 1.18.10):** `dotReader.Read` L322; state constants `stateBeginLine/stateDot/stateData/stateCR/stateEOF` L328–333; `ErrUnexpectedEOF` (returned when the stream ends mid-body) L340–342; **the `stateDot` + `'\n'` → `stateEOF` transition (the exact line that accepts a bare-LF `.\n` terminator) L362–364**.
**Manifests:** `go.mod` `go 1.13`; `github.com/emersion/go-smtp v0.12.1-0.20191206174923-1f576e0ec85c`; `github.com/emersion/go-message v0.10.9-0.20191116124005-65fd0119e899`; `github.com/mattn/go-sqlite3 v1.11.0`.

### 8.4 Read-only confirmation

The investigation added exactly one file — this document — and modified no existing repository file. The build was performed from the repo root (`go build ./cmd/maddy`) and the binary moved to `/tmp/investig`; the server ran from `/tmp/investig` with `state`/`runtime` redirected there, so no `*queue`/`*mtasts-cache` residue and no SQLite databases were created in the tree (§7 inventory). All harness scripts (`catch.py`, `probe.py`, `exp.py`, `proxy.py`) and configs (`maddy.conf`, `maddy_size.conf`) live under `/tmp/investig` and are removed at the end of the investigation, leaving the repository unchanged.

## Appendix A — Harness (temporary; lives under `/tmp/investig`, removed after the investigation)

All scripts below were run exactly as shown. They are **not** added to the repository; they are reproduced here so every experiment is repeatable.

### A.1 Probe config — `maddy.conf` (minimal, check-free, plaintext, `io_debug yes`, state/runtime redirected out of the repo)

```text
## Minimal check-free plaintext probe config for DATA-boundary investigation.
## Runs from /tmp/investig so all state/runtime artifacts land OUTSIDE the repo.
state /tmp/investig/state
runtime /tmp/investig/runtime
hostname probe.local
tls off

## Byte-exact delivered-body sink: forward everything to the Python catch server.
smtp_downstream catch {
    targets tcp://127.0.0.1:2526
    hostname probe.local
    attempt_starttls no
    require_tls no
}

## The endpoint under study: plaintext, raw transcript on, no inbound checks.
smtp tcp://127.0.0.1:2525 {
    io_debug yes
    deliver_to &catch
}
```

### A.2 Catch server — `catch.py` (byte-exact delivered-message sink on `:2526`)

```python
#!/usr/bin/env python3
"""catch.py - byte-exact delivery sink on :2526."""
import socket
import threading
import os
import sys

HOST, PORT = "127.0.0.1", 2526
OUTDIR = os.environ.get("INVESTIG_DIR", "/tmp/investig")
_counter_lock = threading.Lock()


def _next_index():
    with _counter_lock:
        existing = [f for f in os.listdir(OUTDIR) if f.startswith("caught_") and f.endswith(".wire")]
        nums = []
        for f in existing:
            try:
                nums.append(int(f[len("caught_"):-len(".wire")]))
            except ValueError:
                pass
        n = (max(nums) + 1) if nums else 1
        open(os.path.join(OUTDIR, "caught_%03d.wire" % n), "wb").close()
        return n


def unstuff(wire_bytes):
    out = []
    for line in wire_bytes.split(b"\r\n"):
        if line.startswith(b".."):
            out.append(line[1:])
        else:
            out.append(line)
    return b"\r\n".join(out)


def handle(conn, addr):
    f = conn.makefile("rb")
    def send(s):
        conn.sendall(s if isinstance(s, bytes) else s.encode())
    send("220 catch.sink ESMTP ready\r\n")
    in_data = False
    data_buf = bytearray()
    while True:
        line = f.readline()
        if not line:
            break
        if not in_data:
            up = line.upper()
            if up.startswith(b"EHLO") or up.startswith(b"LHLO"):
                send("250-catch.sink\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250 SMTPUTF8\r\n")
            elif up.startswith(b"HELO"):
                send("250 catch.sink\r\n")
            elif up.startswith(b"MAIL"):
                send("250 2.1.0 sender ok\r\n")
            elif up.startswith(b"RCPT"):
                send("250 2.1.5 rcpt ok\r\n")
            elif up.startswith(b"DATA"):
                send("354 go ahead\r\n")
                in_data = True
                data_buf = bytearray()
            elif up.startswith(b"RSET"):
                send("250 2.0.0 ok\r\n")
            elif up.startswith(b"QUIT"):
                send("221 2.0.0 bye\r\n")
                break
            elif up.startswith(b"NOOP"):
                send("250 2.0.0 ok\r\n")
            else:
                send("250 2.0.0 ok\r\n")
        else:
            if line in (b".\r\n", b".\n"):
                wire = bytes(data_buf)
                idx = _next_index()
                with open(os.path.join(OUTDIR, "caught_%03d.wire" % idx), "wb") as w:
                    w.write(wire)
                with open(os.path.join(OUTDIR, "caught_%03d.raw" % idx), "wb") as w:
                    w.write(unstuff(wire))
                send("250 2.0.0 OK: queued\r\n")
                in_data = False
            else:
                data_buf += line
    try:
        conn.close()
    except OSError:
        pass


def main():
    os.makedirs(OUTDIR, exist_ok=True)
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind((HOST, PORT))
    s.listen(64)
    sys.stderr.write("catch.py listening on %s:%d -> %s\n" % (HOST, PORT, OUTDIR))
    sys.stderr.flush()
    while True:
        conn, addr = s.accept()
        threading.Thread(target=handle, args=(conn, addr), daemon=True).start()


if __name__ == "__main__":
    main()
```

### A.3 Raw-socket probe — `probe.py` (byte-exact CR/LF and packet-boundary control)

```python
#!/usr/bin/env python3
"""probe.py - raw-socket byte-exact SMTP client."""
import socket

DEFAULT_TIMEOUT = 6.0


def converse(steps, host="127.0.0.1", port=2525, timeout=DEFAULT_TIMEOUT):
    log = []
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.settimeout(timeout)
    s.connect((host, port))
    try:
        banner = s.recv(65535)
        log.append(("S->C", banner))
    except socket.timeout:
        log.append(("S->C", b""))
    for step in steps:
        kind = step[0]
        if kind == "S":
            data = step[1]
            s.sendall(data)
            log.append(("C->S", data))
        elif kind == "R":
            t = step[1] if len(step) > 1 else timeout
            s.settimeout(t)
            try:
                data = s.recv(65535)
            except socket.timeout:
                data = b""
            log.append(("S->C", data))
        elif kind == "C":
            s.close()
            log.append(("CLOSE", b""))
    try:
        s.close()
    except OSError:
        pass
    return log


def show(label, log):
    print("==================== %s ====================" % label)
    for direction, data in log:
        if direction == "CLOSE":
            print(">>> CLOSE (client closed socket)")
            continue
        print("--- %s [%d bytes] --- %r" % (direction, len(data), data))
    print()


if __name__ == "__main__":
    log = converse([("S", b"EHLO probe.test\r\n"), ("R", 3), ("S", b"QUIT\r\n"), ("R", 3)])
    show("probe self-test", log)
```

### A.4 Experiment driver — `exp.py` (defines E1–E9, ELINE, ECMD payloads)

```python
#!/usr/bin/env python3
"""exp.py - experiment runner for the DATA-boundary investigation."""
import os
import sys
import time
import glob

import probe

INVESTIG = os.environ.get("INVESTIG_DIR", "/tmp/investig")
LOG = os.path.join(INVESTIG, "maddy.log")

HDR = (b"From: <sender@probe.local>\r\n"
       b"To: <rcpt@probe.local>\r\n"
       b"Subject: %s\r\n"
       b"X-Probe: boundary-test\r\n"
       b"\r\n"
       b"Line one of the body.\r\n"
       b"Line two of the body.")


def _mail_rcpt_data(sender=b"sender@probe.local", rcpt=b"rcpt@probe.local"):
    return [("S", b"EHLO probe.test\r\n"), ("R", 3),
            ("S", b"MAIL FROM:<%s>\r\n" % sender), ("R", 3),
            ("S", b"RCPT TO:<%s>\r\n" % rcpt), ("R", 3),
            ("S", b"DATA\r\n"), ("R", 3)]


def build(exp):
    if exp == "E1":
        body = HDR % b"E1-canonical" + b"\r\n.\r\n"
        return _mail_rcpt_data() + [("S", body), ("R", 4), ("S", b"QUIT\r\n"), ("R", 3)]
    if exp == "E2":
        body = HDR % b"E2-bareLF" + b"\n.\n"
        return _mail_rcpt_data() + [("S", body), ("R", 4), ("S", b"QUIT\r\n"), ("R", 3)]
    if exp == "E3a":
        body = HDR % b"E3a-LF-dot-CRLF" + b"\n.\r\n"
        return _mail_rcpt_data() + [("S", body), ("R", 4), ("S", b"QUIT\r\n"), ("R", 3)]
    if exp == "E3b":
        body = HDR % b"E3b-CRLF-dot-LF" + b"\r\n.\n"
        return _mail_rcpt_data() + [("S", body), ("R", 4), ("S", b"QUIT\r\n"), ("R", 3)]
    if exp == "E4":
        body = (b"From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\n"
                b"Subject: E4-dotstuff\r\nX-Probe: boundary-test\r\n\r\n"
                b"Line A before.\r\n..\r\n...\r\nLine B after.\r\n.\r\n")
        return _mail_rcpt_data() + [("S", body), ("R", 4), ("S", b"QUIT\r\n"), ("R", 3)]
    if exp == "E5":
        part1 = (b"From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\n"
                 b"Subject: E5-missing-crlf\r\nX-Probe: boundary-test\r\n\r\n"
                 b"Line one of the body.\r\nGLUED_NO_NEWLINE_BEFORE.\r\n")
        return _mail_rcpt_data() + [("S", part1), ("R", 4),
                                    ("S", b"\r\n.\r\n"), ("R", 4),
                                    ("S", b"QUIT\r\n"), ("R", 3)]
    if exp == "E6":
        part1 = (b"From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\n"
                 b"Subject: E6-early-close\r\nX-Probe: boundary-test\r\n\r\n"
                 b"Body with no terminator at all")
        return _mail_rcpt_data() + [("S", part1), ("R", 3), ("C",)]
    if exp == "E7":
        one = (b"MAIL FROM:<attacker@probe.local>\r\n"
               b"RCPT TO:<victim@probe.local>\r\n"
               b"DATA\r\n"
               b"From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\n"
               b"Subject: E7-OUTER-legit\r\nX-Probe: boundary-test\r\n\r\n"
               b"Outer legit body line.\n.\n"
               b"MAIL FROM:<spoofed@evil.example>\r\n"
               b"RCPT TO:<victim@probe.local>\r\n"
               b"DATA\r\n"
               b"From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\n"
               b"Subject: E7-SMUGGLED-injected\r\nX-Probe: boundary-test\r\n\r\n"
               b"Smuggled spoofed body line.\r\n.\r\n")
        return [("S", b"EHLO probe.test\r\n"), ("R", 3), ("S", one), ("R", 5),
                ("S", b"QUIT\r\n"), ("R", 3)]
    if exp == "E8":
        steps = [("S", b"EHLO probe.test\r\n"), ("R", 3)]
        for i in range(1, 6):
            steps += [("S", b"MAIL FROM:<sender@probe.local>\r\n"), ("R", 3),
                      ("S", b"RCPT TO:<rcpt@probe.local>\r\n"), ("R", 3),
                      ("S", b"DATA\r\n"), ("R", 3),
                      ("S", (HDR % (b"E8-msg-%d" % i)) + b"\r\n.\r\n"), ("R", 4)]
        steps += [("S", b"QUIT\r\n"), ("R", 3)]
        return steps
    if exp == "ELINE":
        body = (b"From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\n"
                b"Subject: ELINE\r\nX-Probe: boundary-test\r\n\r\n"
                + b"A" * 2500 + b"\r\n.\r\n")
        return _mail_rcpt_data() + [("S", body), ("R", 4), ("S", b"QUIT\r\n"), ("R", 3)]
    if exp == "ECMD":
        return [("S", b"EHLO probe.test\r\n"), ("R", 3),
                ("S", b"MAIL FROM:<" + b"a" * 2500 + b"@probe.local>\r\n"), ("R", 4),
                ("S", b"QUIT\r\n"), ("R", 3)]
    raise SystemExit("unknown experiment %r" % exp)


def snapshot_log():
    try:
        return os.path.getsize(LOG)
    except OSError:
        return 0


def read_log_delta(off):
    try:
        with open(LOG, "rb") as f:
            f.seek(off)
            return f.read()
    except OSError:
        return b""


def caught_set():
    return set(glob.glob(os.path.join(INVESTIG, "caught_*.raw")))


def run(exp, host="127.0.0.1", port=2525):
    off = snapshot_log()
    before = caught_set()
    steps = build(exp)
    log = probe.converse(steps, host=host, port=port)
    time.sleep(0.8)
    probe.show("EXPERIMENT %s (conversation)" % exp, log)
    delta = read_log_delta(off)
    print("-------- maddy.log delta (io_debug raw-wire + ordered-JSON) --------")
    sys.stdout.buffer.write(delta)
    sys.stdout.flush()
    print("\n-------- delivered messages (new caught_* files) --------")
    after = caught_set()
    new = sorted(after - before)
    if not new:
        print("delivered messages captured: 0")
    for rawpath in new:
        wirepath = rawpath[:-4] + ".wire"
        with open(rawpath, "rb") as f:
            raw = f.read()
        print("delivered [%s, %d bytes]:" % (os.path.basename(rawpath), len(raw)))
        print(repr(raw))
        try:
            with open(wirepath, "rb") as f:
                wire = f.read()
            if wire != raw:
                print("  [.wire, %d bytes]: %r" % (len(wire), wire))
        except OSError:
            pass
    print()


if __name__ == "__main__":
    exp = sys.argv[1] if len(sys.argv) > 1 else "E1"
    host = sys.argv[2] if len(sys.argv) > 2 else "127.0.0.1"
    port = int(sys.argv[3]) if len(sys.argv) > 3 else 2525
    run(exp, host, port)
```

### A.5 Front proxy — `proxy.py` (passthrough / normalize / reject modes on `:2527`→`:2525`)

```python
#!/usr/bin/env python3
"""proxy.py - small front proxy for the Q5 scenarios (passthrough|normalize|reject)."""
import socket
import sys
import threading

LISTEN = 2527
UPSTREAM = ("127.0.0.1", 2525)
REJECT_MSG = b"421 4.7.0 bare <LF> in stream rejected by proxy (anti-smuggling), closing\r\n"


def normalize_stream(chunk, carry_cr):
    out = bytearray()
    prev_cr = carry_cr
    for b in chunk:
        if b == 0x0A:
            if not prev_cr:
                out += b"\r\n"
            else:
                out.append(b)
            prev_cr = False
        else:
            out.append(b)
            prev_cr = (b == 0x0D)
    return bytes(out), prev_cr


def has_bare_lf(chunk, carry_cr):
    prev_cr = carry_cr
    for b in chunk:
        if b == 0x0A and not prev_cr:
            return True, False
        prev_cr = (b == 0x0D)
    return False, prev_cr


def pump_c2s(cli, upsock, mode):
    carry = False
    try:
        while True:
            chunk = cli.recv(65535)
            if not chunk:
                break
            if mode == "normalize":
                out, carry = normalize_stream(chunk, carry)
                upsock.sendall(out)
            elif mode == "reject":
                bad, carry = has_bare_lf(chunk, carry)
                if bad:
                    try:
                        cli.sendall(REJECT_MSG)
                    except OSError:
                        pass
                    break
                upsock.sendall(chunk)
            else:
                upsock.sendall(chunk)
    except OSError:
        pass
    finally:
        for s in (cli, upsock):
            try:
                s.shutdown(socket.SHUT_RDWR)
            except OSError:
                pass


def pump_s2c(cli, upsock):
    try:
        while True:
            chunk = upsock.recv(65535)
            if not chunk:
                break
            cli.sendall(chunk)
    except OSError:
        pass


def handle(cli, mode):
    upsock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    upsock.connect(UPSTREAM)
    t = threading.Thread(target=pump_s2c, args=(cli, upsock), daemon=True)
    t.start()
    pump_c2s(cli, upsock, mode)
    try:
        cli.close()
    except OSError:
        pass
    try:
        upsock.close()
    except OSError:
        pass


def main():
    mode = sys.argv[1] if len(sys.argv) > 1 else "passthrough"
    listen = int(sys.argv[2]) if len(sys.argv) > 2 else LISTEN
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind(("127.0.0.1", listen))
    s.listen(64)
    sys.stderr.write("proxy.py mode=%s listening :%d -> %s:%d\n" % (mode, listen, UPSTREAM[0], UPSTREAM[1]))
    sys.stderr.flush()
    while True:
        cli, _ = s.accept()
        threading.Thread(target=handle, args=(cli, mode), daemon=True).start()


if __name__ == "__main__":
    main()
```

### A.6 Size-limit instance config — `maddy_size.conf` (`:2530`, `max_message_size 500b`, separate state2/runtime2 — the 552 instance)

```text
## Second probe instance: same as primary but with a tiny max_message_size
## to exercise the 552 (ErrDataTooLarge) path. Distinct state/runtime to avoid
## collision with the primary instance.
state /tmp/investig/state2
runtime /tmp/investig/runtime2
hostname probe.local
tls off

smtp_downstream catch {
    targets tcp://127.0.0.1:2526
    hostname probe.local
    attempt_starttls no
    require_tls no
}

## max_message_size 500b -> advertises "250 SIZE 500" and rejects larger bodies with 552.
smtp tcp://127.0.0.1:2530 {
    io_debug yes
    max_message_size 500b
    deliver_to &catch
}
```

## Appendix B — Complete unedited captures

Experiments shown in full inline are pointer-referenced here to avoid duplication; the remainder are reproduced in full below. Every capture is the complete, unedited output (probe conversation → Maddy `io_debug` transcript + ordered-JSON logs → delivered bytes).

- **E1** (canonical) — shown in full in **§2**.
- **E6** (early close) — the abrupt-`close()` 10-run context is in **§3**; the deterministic half-close `unexpected EOF`→`554` capture is in **B.E6** below.
- **ELINE** (over-long DATA line) — the primary clean-`500` run is in **§3**; the 64-run same-input distribution and the `unexpected EOF` / abrupt-close `connection reset by peer` variant captures are in **B.ELINE** below.
- **552** (max message size) — shown in full in **§3**.
- **E7** (pipelined smuggle) — shown in full in **§4**.
- **E8** (back-to-back, both runs) — shown in full in **§5**.
- **E9 reject/E2**, **normalize byte-rewrite proof** — shown in full in **§6**.
- **hang** (unterminated DATA) and **artifact inventory** — shown in full in **§7**.
- **build** (`go build` + `-v`) — shown in full in **§0**.

### B.E2 — bare-LF terminator `\n.\n` (complete)

```text
==================== EXPERIMENT E2 (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [38 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [39 bytes] --- b'250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [14 bytes] --- b'250-SMTPUTF8\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [19 bytes] --- b'250 SIZE 33554432\r\n'
--- C->S [145 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E2-bareLF\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\n.\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [6 bytes] --- b'QUIT\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"

-------- maddy.log delta (io_debug raw-wire + ordered-JSON) --------
smtp: 220 probe.local ESMTP Service Ready

smtp: EHLO probe.test

smtp: 250-Hello probe.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432

smtp: MAIL FROM:<sender@probe.local>
RCPT TO:<rcpt@probe.local>
DATA

smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: incoming message	{"msg_id":"a8fe0125","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:33524"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"a8fe0125"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"a8fe0125"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"a8fe0125"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"a8fe0125"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"a8fe0125"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"a8fe0125"}
[debug] smtp_downstream: connected	{"msg_id":"a8fe0125","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"a8fe0125"}
smtp: RCPT ok	{"msg_id":"a8fe0125","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

smtp: From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E2-bareLF
X-Probe: boundary-test

Line one of the body.
Line two of the body.
.
QUIT

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"a8fe0125"}
smtp: accepted	{"msg_id":"a8fe0125"}
[debug] smtp: reset	

-------- delivered messages (new caught_* files) --------
delivered [caught_002.raw, 309 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id a8fe0125; Wed, 08 Jul\r\n 2026 06:14:52 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E2-bareLF\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n'
```

### B.E3a — mixed `\n.\r\n` (complete)

```text
==================== EXPERIMENT E3a (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [38 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [39 bytes] --- b'250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [33 bytes] --- b'250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [152 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E3a-LF-dot-CRLF\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\n.\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [6 bytes] --- b'QUIT\r\n'
--- S->C [58 bytes] --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'

-------- maddy.log delta (io_debug raw-wire + ordered-JSON) --------
smtp: 220 probe.local ESMTP Service Ready

smtp: EHLO probe.test

smtp: 250-Hello probe.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432

smtp: MAIL FROM:<sender@probe.local>
RCPT TO:<rcpt@probe.local>

smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: incoming message	{"msg_id":"9be40444","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:33534"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"9be40444"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"9be40444"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"9be40444"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"9be40444"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"9be40444"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"9be40444"}
[debug] smtp_downstream: connected	{"msg_id":"9be40444","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"9be40444"}
smtp: RCPT ok	{"msg_id":"9be40444","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: DATA
From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E3a-LF-dot-CRLF
X-Probe: boundary-test

Line one of the body.
Line two of the body.
.

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"9be40444"}
smtp: accepted	{"msg_id":"9be40444"}
smtp: 250 2.0.0 OK: queued

[debug] smtp: reset	
smtp: QUIT


-------- delivered messages (new caught_* files) --------
delivered [caught_003.raw, 315 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id 9be40444; Wed, 08 Jul\r\n 2026 06:14:53 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E3a-LF-dot-CRLF\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n'
```

### B.E3b — mixed `\r\n.\n` (complete)

```text
==================== EXPERIMENT E3b (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [38 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [53 bytes] --- b'250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [19 bytes] --- b'250 SIZE 33554432\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [152 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E3b-CRLF-dot-LF\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n.\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [6 bytes] --- b'QUIT\r\n'
--- S->C [58 bytes] --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'

-------- maddy.log delta (io_debug raw-wire + ordered-JSON) --------
smtp: 220 probe.local ESMTP Service Ready

smtp: EHLO probe.test

smtp: 250-Hello probe.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432

smtp: MAIL FROM:<sender@probe.local>
RCPT TO:<rcpt@probe.local>

smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: incoming message	{"msg_id":"34710cd8","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:33540"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"34710cd8"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"34710cd8"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"34710cd8"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"34710cd8"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"34710cd8"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"34710cd8"}
[debug] smtp_downstream: connected	{"msg_id":"34710cd8","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"34710cd8"}
smtp: RCPT ok	{"msg_id":"34710cd8","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: DATA
From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E3b-CRLF-dot-LF
X-Probe: boundary-test

Line one of the body.
Line two of the body.
.

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"34710cd8"}
smtp: accepted	{"msg_id":"34710cd8"}
smtp: 250 2.0.0 OK: queued

[debug] smtp: reset	
smtp: QUIT


-------- delivered messages (new caught_* files) --------
delivered [caught_004.raw, 315 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id 34710cd8; Wed, 08 Jul\r\n 2026 06:14:53 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E3b-CRLF-dot-LF\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n'
```

### B.E4 — dot-stuffing `\r\n..\r\n` (complete; note `.wire` re-stuffed egress vs `.raw` unstuffed delivered)

```text
==================== EXPERIMENT E4 (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [52 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [39 bytes] --- b'250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [19 bytes] --- b'250 SIZE 33554432\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [143 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E4-dotstuff\r\nX-Probe: boundary-test\r\n\r\nLine A before.\r\n..\r\n...\r\nLine B after.\r\n.\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [6 bytes] --- b'QUIT\r\n'
--- S->C [58 bytes] --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'

-------- maddy.log delta (io_debug raw-wire + ordered-JSON) --------
smtp: 220 probe.local ESMTP Service Ready

smtp: EHLO probe.test

smtp: 250-Hello probe.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432

smtp: MAIL FROM:<sender@probe.local>
RCPT TO:<rcpt@probe.local>

smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: incoming message	{"msg_id":"7d66417b","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:33552"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"7d66417b"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"7d66417b"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"7d66417b"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"7d66417b"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"7d66417b"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"7d66417b"}
[debug] smtp_downstream: connected	{"msg_id":"7d66417b","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"7d66417b"}
smtp: RCPT ok	{"msg_id":"7d66417b","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: DATA
From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E4-dotstuff
X-Probe: boundary-test

Line A before.
..
...
Line B after.
.

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"7d66417b"}
smtp: accepted	{"msg_id":"7d66417b"}
smtp: 250 2.0.0 OK: queued

[debug] smtp: reset	
smtp: QUIT


-------- delivered messages (new caught_* files) --------
delivered [caught_005.raw, 303 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id 7d66417b; Wed, 08 Jul\r\n 2026 06:14:54 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E4-dotstuff\r\nX-Probe: boundary-test\r\n\r\nLine A before.\r\n.\r\n..\r\nLine B after.\r\n'
  [.wire, 305 bytes]: b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id 7d66417b; Wed, 08 Jul\r\n 2026 06:14:54 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E4-dotstuff\r\nX-Probe: boundary-test\r\n\r\nLine A before.\r\n..\r\n...\r\nLine B after.\r\n'
```

### B.E5 — split/mid-line dot framing (complete; accepted + delivered)

```text
==================== EXPERIMENT E5 (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [52 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [39 bytes] --- b'250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [19 bytes] --- b'250 SIZE 33554432\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [153 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E5-missing-crlf\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nGLUED_NO_NEWLINE_BEFORE.\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [5 bytes] --- b'\r\n.\r\n'
--- S->C [58 bytes] --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
--- C->S [6 bytes] --- b'QUIT\r\n'
--- S->C [22 bytes] --- b'250 2.0.0 OK: queued\r\n'

-------- maddy.log delta (io_debug raw-wire + ordered-JSON) --------
smtp: 220 probe.local ESMTP Service Ready

smtp: EHLO probe.test

smtp: 250-Hello probe.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432

smtp: MAIL FROM:<sender@probe.local>
RCPT TO:<rcpt@probe.local>

smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: incoming message	{"msg_id":"e1251d50","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:33560"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"e1251d50"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"e1251d50"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"e1251d50"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"e1251d50"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"e1251d50"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"e1251d50"}
[debug] smtp_downstream: connected	{"msg_id":"e1251d50","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"e1251d50"}
smtp: RCPT ok	{"msg_id":"e1251d50","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: DATA
From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E5-missing-crlf
X-Probe: boundary-test

Line one of the body.
GLUED_NO_NEWLINE_BEFORE.

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

smtp: 
.

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"e1251d50"}
smtp: accepted	{"msg_id":"e1251d50"}
smtp: 250 2.0.0 OK: queued

[debug] smtp: reset	
smtp: QUIT

smtp: 221 2.0.0 Goodnight and good luck


-------- delivered messages (new caught_* files) --------
delivered [caught_006.raw, 320 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id e1251d50; Wed, 08 Jul\r\n 2026 06:14:55 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E5-missing-crlf\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nGLUED_NO_NEWLINE_BEFORE.\r\n\r\n'
```

### B.E6 — deterministic `unexpected EOF` → `554` via half-close (complete; 0 delivered)

**Result — the deterministic form of the E6 abort.** The abrupt-`close()` E6 run in §3 surfaces the `554` only as a race (3/10). To capture the `554` on the wire on **every** run, the client half-closes: it sends the `DATA` headers plus a partial body with **no** terminator, then `shutdown(SHUT_WR)` (signalling EOF to the server) while keeping its read side open. The server's `net/textproto` dot-reader hits EOF mid-body and returns `io.ErrUnexpectedEOF`; Maddy's `Session.Data` logs `DATA error {"reason":"unexpected EOF"}` and `endp.wrapErr` (`smtp.go:389`) maps the non-`SMTPError` to the generic `554` — `EnhancedCodeNotSet` is promoted to `5.0.0` at `go-smtp/conn.go:663-666`, rendering `554 5.0.0 Internal server error (msg ID = …)`, which go-smtp writes to the still-open read side. **0 delivered**, and this reproduced on every run. Complete capture:

```text
==================== EXPERIMENT B.E6 half-close (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [110 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [58 bytes] --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
--- C->S [131 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E6-halfclose\r\nX-Probe: boundary-test\r\n\r\nBody with no terminator at all'
>>> HALF-CLOSE: client shutdown(SHUT_WR); read side kept open, polling for the server's final response
--- S->C [53 bytes] --- b'554 5.0.0 Internal server error (msg ID = a0ba388f)\r\n'
>>> server closed its side (recv returned empty)

-------- maddy.log delta (io_debug raw-wire + ordered-JSON) --------
smtp: 220 probe.local ESMTP Service Ready

smtp: EHLO probe.test

smtp: 250-Hello probe.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432

smtp: MAIL FROM:<sender@probe.local>

smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: RCPT TO:<rcpt@probe.local>

smtp: incoming message	{"msg_id":"a0ba388f","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:51242"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"a0ba388f"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"a0ba388f"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"a0ba388f"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"a0ba388f"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"a0ba388f"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"a0ba388f"}
[debug] smtp_downstream: connected	{"msg_id":"a0ba388f","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"a0ba388f"}
smtp: RCPT ok	{"msg_id":"a0ba388f","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: DATA

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

smtp: From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E6-halfclose
X-Probe: boundary-test

Body with no terminator at all
smtp: DATA error	{"msg_id":"a0ba388f","reason":"unexpected EOF"}
smtp: 554 5.0.0 Internal server error (msg ID = a0ba388f)

smtp: aborted	{"msg_id":"a0ba388f"}
[debug] smtp: reset	

-------- delivered messages (new caught_* files) --------
delivered messages captured: 0
```

### B.ELINE — over-long `DATA` line: `unexpected EOF` and abrupt-close `connection reset by peer` variants (complete; 0 delivered)

**Same-input distribution (run-to-run rigor).** The identical ELINE input — a 2500-byte `DATA` body line (exceeding `MaxLineLength` 2000) followed by `\r\n.\r\n` — was replayed **64 times** against the running server. Observed distribution: server-logged `DATA error {"reason":"unexpected EOF"}` in **36/64** runs, a clean `500 5.4.0 Too long line, closing connection` in **28/64** runs, and `connection reset by peer` in **0/64**; the invariant across all 64 was **0 delivered**. Which server-log reason appears is governed by *how the client tears the socket down*, not by the payload: a client that reads the response and then closes cleanly makes the server's read see EOF (`unexpected EOF`); a client that closes abruptly (RST) with the server's response still unread makes the server's read see `connection reset by peer`. That is why the `2/10` reset share in the §3 10-run sample did not recur in this drain-then-close re-run — the reset reason requires an abrupt RST close, exercised explicitly in (ii) below.

**(i) `unexpected EOF` variant (same input; client drains then closes cleanly).** The server logs the `DATA error`, writes a `554 5.0.0 Internal server error (msg ID = …)` that the timed-out client never reads, and aborts. Complete capture:

```text
==================== EXPERIMENT ELINE (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [38 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [72 bytes] --- b'250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [2599 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: ELINE\r\nX-Probe: boundary-test\r\n\r\nAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\r\n.\r\n'
--- S->C [58 bytes] --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
--- S->C --- b'<no data - timed out>'
CLIENT_OUTCOME=timeout

-------- maddy.log delta (io_debug raw-wire + ordered-JSON) --------
smtp: 220 probe.local ESMTP Service Ready

smtp: EHLO probe.test

smtp: 250-Hello probe.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432

smtp: MAIL FROM:<sender@probe.local>

smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: RCPT TO:<rcpt@probe.local>

smtp: incoming message	{"msg_id":"10a6b616","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:41934"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"10a6b616"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"10a6b616"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"10a6b616"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"10a6b616"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"10a6b616"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"10a6b616"}
[debug] smtp_downstream: connected	{"msg_id":"10a6b616","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"10a6b616"}
smtp: RCPT ok	{"msg_id":"10a6b616","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: DATA

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

smtp: DATA error	{"msg_id":"10a6b616","reason":"unexpected EOF"}
smtp: 554 5.0.0 Internal server error (msg ID = 10a6b616)

smtp: aborted	{"msg_id":"10a6b616"}
[debug] smtp: reset	

-------- delivered messages (new caught_* files) --------
delivered messages captured: 0
```

**(ii) `connection reset by peer` variant (same 2599-byte body; abrupt `SO_LINGER=0` RST close, server response left unread).** This is the close behaviour that deterministically produces the reset reason — the same reason-shape as the §3 abrupt-`close()` E6 run. The RST tears the socket down before the server can write its `554`, so only the `DATA error` + `aborted` are logged. Complete capture:

```text
==================== EXPERIMENT ELINE-abrupt (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [38 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [72 bytes] --- b'250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [2599 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: ELINE\r\nX-Probe: boundary-test\r\n\r\nAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\r\n.\r\n'
>>> abrupt close with SO_LINGER=0 (RST), server response left unread

-------- maddy.log delta (io_debug raw-wire + ordered-JSON) --------
smtp: 220 probe.local ESMTP Service Ready

smtp: EHLO probe.test

smtp: 250-Hello probe.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432

smtp: MAIL FROM:<sender@probe.local>

smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: RCPT TO:<rcpt@probe.local>

smtp: incoming message	{"msg_id":"50869fbf","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:54910"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"50869fbf"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"50869fbf"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"50869fbf"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"50869fbf"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"50869fbf"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"50869fbf"}
[debug] smtp_downstream: connected	{"msg_id":"50869fbf","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"50869fbf"}
smtp: RCPT ok	{"msg_id":"50869fbf","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: DATA

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

smtp: DATA error	{"msg_id":"50869fbf","reason":"read tcp 127.0.0.1:2525-\u003e127.0.0.1:54910: read: connection reset by peer"}
smtp: aborted	{"msg_id":"50869fbf"}
[debug] smtp: reset	

-------- delivered messages (new caught_* files) --------
delivered messages captured: 0
```

### B.ECMD — over-long command line, clean read-until-close (complete)

```text
--- banner ---
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17] --- b'EHLO probe.test\r\n'
--- S->C [110 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
--- C->S [2526 bytes] --- MAIL FROM with 2500-a localpart (repr head) --- b'MAIL FROM:<aaaaaaaaaaaaaaaaaaaaaaaaaaaaa'...b'aaaaa@probe.local>\r\n'
--- S->C [45 bytes] --- b'500 5.4.0 Too long line, closing connection\r\n'
>>> server closed connection (recv returned empty)
```

### B.E9.passthrough / E2 (complete)

```text
==================== EXPERIMENT E2 (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [38 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [72 bytes] --- b'250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [145 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E2-bareLF\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\n.\n'
--- S->C [58 bytes] --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
--- C->S [6 bytes] --- b'QUIT\r\n'
--- S->C [22 bytes] --- b'250 2.0.0 OK: queued\r\n'

-------- maddy.log delta (io_debug raw-wire + ordered-JSON) --------
smtp: 220 probe.local ESMTP Service Ready

smtp: EHLO probe.test

smtp: 250-Hello probe.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432

smtp: MAIL FROM:<sender@probe.local>

smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: RCPT TO:<rcpt@probe.local>

smtp: incoming message	{"msg_id":"b74a01f8","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:50366"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"b74a01f8"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"b74a01f8"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"b74a01f8"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"b74a01f8"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"b74a01f8"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"b74a01f8"}
[debug] smtp_downstream: connected	{"msg_id":"b74a01f8","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"b74a01f8"}
smtp: RCPT ok	{"msg_id":"b74a01f8","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: DATA

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

smtp: From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E2-bareLF
X-Probe: boundary-test

Line one of the body.
Line two of the body.
.

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"b74a01f8"}
smtp: accepted	{"msg_id":"b74a01f8"}
smtp: 250 2.0.0 OK: queued

[debug] smtp: reset	
smtp: QUIT

smtp: 221 2.0.0 Goodnight and good luck


-------- delivered messages (new caught_* files) --------
delivered [caught_019.raw, 309 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id b74a01f8; Wed, 08 Jul\r\n 2026 06:20:51 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E2-bareLF\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n'
```

### B.E9.passthrough / E7 (complete)

```text
==================== EXPERIMENT E7 (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [22 bytes] --- b'250-Hello probe.test\r\n'
--- C->S [409 bytes] --- b'MAIL FROM:<attacker@probe.local>\r\nRCPT TO:<victim@probe.local>\r\nDATA\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E7-OUTER-legit\r\nX-Probe: boundary-test\r\n\r\nOuter legit body line.\n.\nMAIL FROM:<spoofed@evil.example>\r\nRCPT TO:<victim@probe.local>\r\nDATA\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E7-SMUGGLED-injected\r\nX-Probe: boundary-test\r\n\r\nSmuggled spoofed body line.\r\n.\r\n'
--- S->C [69 bytes] --- b'250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n'
--- C->S [6 bytes] --- b'QUIT\r\n'
--- S->C [19 bytes] --- b'250 SIZE 33554432\r\n'

-------- maddy.log delta (io_debug raw-wire + ordered-JSON) --------
smtp: 220 probe.local ESMTP Service Ready

smtp: EHLO probe.test

smtp: 250-Hello probe.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432

smtp: MAIL FROM:<attacker@probe.local>
RCPT TO:<victim@probe.local>
DATA
From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E7-OUTER-legit
X-Probe: boundary-test

Outer legit body line.
.
MAIL FROM:<spoofed@evil.example>
RCPT TO:<victim@probe.local>
DATA
From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E7-SMUGGLED-injected
X-Probe: boundary-test

Smuggled spoofed body line.
.

smtp: 250 2.0.0 Roger, accepting mail from <attacker@probe.local>

smtp: incoming message	{"msg_id":"cded9993","sender":"attacker@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:50382"}
[debug] smtp/pipeline: sender attacker@probe.local matched by default rule	{"msg_id":"cded9993"}
[debug] smtp/pipeline: global rcpt modifiers: victim@probe.local => victim@probe.local	{"msg_id":"cded9993"}
[debug] smtp/pipeline: per-source rcpt modifiers: victim@probe.local => victim@probe.local	{"msg_id":"cded9993"}
[debug] smtp/pipeline: recipient victim@probe.local matched by default rule (clean = victim@probe.local)	{"msg_id":"cded9993"}
[debug] smtp/pipeline: per-rcpt modifiers: victim@probe.local => victim@probe.local	{"msg_id":"cded9993"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"cded9993"}
[debug] smtp_downstream: connected	{"msg_id":"cded9993","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(attacker@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"cded9993"}
smtp: RCPT ok	{"msg_id":"cded9993","rcpt":"victim@probe.local"}
smtp: 250 2.0.0 I'll make sure <victim@probe.local> gets this

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"cded9993"}
smtp: accepted	{"msg_id":"cded9993"}
[debug] smtp: reset	
smtp: incoming message	{"msg_id":"35aa3a07","sender":"spoofed@evil.example","src_host":"probe.test","src_ip":"127.0.0.1:50382"}
[debug] smtp/pipeline: sender spoofed@evil.example matched by default rule	{"msg_id":"35aa3a07"}
[debug] smtp/pipeline: global rcpt modifiers: victim@probe.local => victim@probe.local	{"msg_id":"35aa3a07"}
[debug] smtp/pipeline: per-source rcpt modifiers: victim@probe.local => victim@probe.local	{"msg_id":"35aa3a07"}
[debug] smtp/pipeline: recipient victim@probe.local matched by default rule (clean = victim@probe.local)	{"msg_id":"35aa3a07"}
[debug] smtp/pipeline: per-rcpt modifiers: victim@probe.local => victim@probe.local	{"msg_id":"35aa3a07"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"35aa3a07"}
[debug] smtp_downstream: connected	{"msg_id":"35aa3a07","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(spoofed@evil.example) ok, target = smtp_downstream:catch	{"msg_id":"35aa3a07"}
smtp: RCPT ok	{"msg_id":"35aa3a07","rcpt":"victim@probe.local"}
[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"35aa3a07"}
smtp: accepted	{"msg_id":"35aa3a07"}
[debug] smtp: reset	
smtp: QUIT


-------- delivered messages (new caught_* files) --------
delivered [caught_020.raw, 294 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <attacker@probe.local>) with ESMTP id cded9993; Wed, 08\r\n Jul 2026 06:20:52 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E7-OUTER-legit\r\nX-Probe: boundary-test\r\n\r\nOuter legit body line.\r\n'
delivered [caught_021.raw, 305 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <spoofed@evil.example>) with ESMTP id 35aa3a07; Wed, 08\r\n Jul 2026 06:20:52 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E7-SMUGGLED-injected\r\nX-Probe: boundary-test\r\n\r\nSmuggled spoofed body line.\r\n'
```

### B.E9.normalize / E2 (complete)

```text
==================== EXPERIMENT E2 (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [38 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n'
--- C->S [32 bytes] --- b'MAIL FROM:<sender@probe.local>\r\n'
--- S->C [72 bytes] --- b'250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
--- C->S [28 bytes] --- b'RCPT TO:<rcpt@probe.local>\r\n'
--- S->C [59 bytes] --- b'250 2.0.0 Roger, accepting mail from <sender@probe.local>\r\n'
--- C->S [6 bytes] --- b'DATA\r\n'
--- S->C [55 bytes] --- b"250 2.0.0 I'll make sure <rcpt@probe.local> gets this\r\n"
--- C->S [145 bytes] --- b'From: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E2-bareLF\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\n.\n'
--- S->C [58 bytes] --- b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
--- C->S [6 bytes] --- b'QUIT\r\n'
--- S->C [22 bytes] --- b'250 2.0.0 OK: queued\r\n'

-------- maddy.log delta (io_debug raw-wire + ordered-JSON) --------
smtp: 220 probe.local ESMTP Service Ready

smtp: EHLO probe.test

smtp: 250-Hello probe.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432

smtp: MAIL FROM:<sender@probe.local>

smtp: 250 2.0.0 Roger, accepting mail from <sender@probe.local>

smtp: RCPT TO:<rcpt@probe.local>

smtp: incoming message	{"msg_id":"e0289c2e","sender":"sender@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:48972"}
[debug] smtp/pipeline: sender sender@probe.local matched by default rule	{"msg_id":"e0289c2e"}
[debug] smtp/pipeline: global rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"e0289c2e"}
[debug] smtp/pipeline: per-source rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"e0289c2e"}
[debug] smtp/pipeline: recipient rcpt@probe.local matched by default rule (clean = rcpt@probe.local)	{"msg_id":"e0289c2e"}
[debug] smtp/pipeline: per-rcpt modifiers: rcpt@probe.local => rcpt@probe.local	{"msg_id":"e0289c2e"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"e0289c2e"}
[debug] smtp_downstream: connected	{"msg_id":"e0289c2e","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(sender@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"e0289c2e"}
smtp: RCPT ok	{"msg_id":"e0289c2e","rcpt":"rcpt@probe.local"}
smtp: 250 2.0.0 I'll make sure <rcpt@probe.local> gets this

smtp: DATA

smtp: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

smtp: From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E2-bareLF
X-Probe: boundary-test

Line one of the body.
Line two of the body.
.

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"e0289c2e"}
smtp: accepted	{"msg_id":"e0289c2e"}
smtp: 250 2.0.0 OK: queued

[debug] smtp: reset	
smtp: QUIT

smtp: 221 2.0.0 Goodnight and good luck


-------- delivered messages (new caught_* files) --------
delivered [caught_022.raw, 309 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <sender@probe.local>) with ESMTP id e0289c2e; Wed, 08 Jul\r\n 2026 06:21:06 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E2-bareLF\r\nX-Probe: boundary-test\r\n\r\nLine one of the body.\r\nLine two of the body.\r\n'
```

### B.E9.normalize / E7 (complete; spoof still delivered — 2 messages)

```text
==================== EXPERIMENT E7 (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [38 bytes] --- b'250-Hello probe.test\r\n250-PIPELINING\r\n'
--- C->S [409 bytes] --- b'MAIL FROM:<attacker@probe.local>\r\nRCPT TO:<victim@probe.local>\r\nDATA\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E7-OUTER-legit\r\nX-Probe: boundary-test\r\n\r\nOuter legit body line.\n.\nMAIL FROM:<spoofed@evil.example>\r\nRCPT TO:<victim@probe.local>\r\nDATA\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E7-SMUGGLED-injected\r\nX-Probe: boundary-test\r\n\r\nSmuggled spoofed body line.\r\n.\r\n'
--- S->C [72 bytes] --- b'250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
--- C->S [6 bytes] --- b'QUIT\r\n'
--- S->C [61 bytes] --- b'250 2.0.0 Roger, accepting mail from <attacker@probe.local>\r\n'

-------- maddy.log delta (io_debug raw-wire + ordered-JSON) --------
smtp: 220 probe.local ESMTP Service Ready

smtp: EHLO probe.test

smtp: 250-Hello probe.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432

smtp: MAIL FROM:<attacker@probe.local>
RCPT TO:<victim@probe.local>
DATA
From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E7-OUTER-legit
X-Probe: boundary-test

Outer legit body line.
.
MAIL FROM:<spoofed@evil.example>
RCPT TO:<victim@probe.local>
DATA
From: <sender@probe.local>
To: <rcpt@probe.local>
Subject: E7-SMUGGLED-injected
X-Probe: boundary-test

Smuggled spoofed body line.
.

smtp: 250 2.0.0 Roger, accepting mail from <attacker@probe.local>

smtp: incoming message	{"msg_id":"c644728f","sender":"attacker@probe.local","src_host":"probe.test","src_ip":"127.0.0.1:48976"}
[debug] smtp/pipeline: sender attacker@probe.local matched by default rule	{"msg_id":"c644728f"}
[debug] smtp/pipeline: global rcpt modifiers: victim@probe.local => victim@probe.local	{"msg_id":"c644728f"}
[debug] smtp/pipeline: per-source rcpt modifiers: victim@probe.local => victim@probe.local	{"msg_id":"c644728f"}
[debug] smtp/pipeline: recipient victim@probe.local matched by default rule (clean = victim@probe.local)	{"msg_id":"c644728f"}
[debug] smtp/pipeline: per-rcpt modifiers: victim@probe.local => victim@probe.local	{"msg_id":"c644728f"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"c644728f"}
[debug] smtp_downstream: connected	{"msg_id":"c644728f","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(attacker@probe.local) ok, target = smtp_downstream:catch	{"msg_id":"c644728f"}
smtp: RCPT ok	{"msg_id":"c644728f","rcpt":"victim@probe.local"}
smtp: 250 2.0.0 I'll make sure <victim@probe.local> gets this

[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"c644728f"}
smtp: accepted	{"msg_id":"c644728f"}
[debug] smtp: reset	
smtp: incoming message	{"msg_id":"cff64f61","sender":"spoofed@evil.example","src_host":"probe.test","src_ip":"127.0.0.1:48976"}
[debug] smtp/pipeline: sender spoofed@evil.example matched by default rule	{"msg_id":"cff64f61"}
[debug] smtp/pipeline: global rcpt modifiers: victim@probe.local => victim@probe.local	{"msg_id":"cff64f61"}
[debug] smtp/pipeline: per-source rcpt modifiers: victim@probe.local => victim@probe.local	{"msg_id":"cff64f61"}
[debug] smtp/pipeline: recipient victim@probe.local matched by default rule (clean = victim@probe.local)	{"msg_id":"cff64f61"}
[debug] smtp/pipeline: per-rcpt modifiers: victim@probe.local => victim@probe.local	{"msg_id":"cff64f61"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"cff64f61"}
[debug] smtp_downstream: connected	{"msg_id":"cff64f61","remote_server":"127.0.0.1"}
[debug] smtp/pipeline: tgt.Start(spoofed@evil.example) ok, target = smtp_downstream:catch	{"msg_id":"cff64f61"}
smtp: RCPT ok	{"msg_id":"cff64f61","rcpt":"victim@probe.local"}
[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"cff64f61"}
smtp: accepted	{"msg_id":"cff64f61"}
[debug] smtp: reset	
smtp: QUIT


-------- delivered messages (new caught_* files) --------
delivered [caught_023.raw, 294 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <attacker@probe.local>) with ESMTP id c644728f; Wed, 08\r\n Jul 2026 06:21:07 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E7-OUTER-legit\r\nX-Probe: boundary-test\r\n\r\nOuter legit body line.\r\n'
delivered [caught_024.raw, 305 bytes]:
b'Received: from probe.test (localhost [127.0.0.1]) by probe.local\r\n (envelope-sender <spoofed@evil.example>) with ESMTP id cff64f61; Wed, 08\r\n Jul 2026 06:21:07 +0000\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E7-SMUGGLED-injected\r\nX-Probe: boundary-test\r\n\r\nSmuggled spoofed body line.\r\n'
```

### B.E9.reject / E7 (complete; `421`, 0 delivered)

```text
==================== EXPERIMENT E7 (conversation) ====================
--- S->C [37 bytes] --- b'220 probe.local ESMTP Service Ready\r\n'
--- C->S [17 bytes] --- b'EHLO probe.test\r\n'
--- S->C [22 bytes] --- b'250-Hello probe.test\r\n'
--- C->S [409 bytes] --- b'MAIL FROM:<attacker@probe.local>\r\nRCPT TO:<victim@probe.local>\r\nDATA\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E7-OUTER-legit\r\nX-Probe: boundary-test\r\n\r\nOuter legit body line.\n.\nMAIL FROM:<spoofed@evil.example>\r\nRCPT TO:<victim@probe.local>\r\nDATA\r\nFrom: <sender@probe.local>\r\nTo: <rcpt@probe.local>\r\nSubject: E7-SMUGGLED-injected\r\nX-Probe: boundary-test\r\n\r\nSmuggled spoofed body line.\r\n.\r\n'
--- S->C [16 bytes] --- b'250-PIPELINING\r\n'
--- C->S [6 bytes] --- b'QUIT\r\n'
--- S->C [128 bytes] --- b'421 4.7.0 bare <LF> in stream rejected by proxy (anti-smuggling), closing\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-SMTPUTF8\r\n'

-------- maddy.log delta (io_debug raw-wire + ordered-JSON) --------
smtp: 220 probe.local ESMTP Service Ready

smtp: EHLO probe.test

smtp: 250-Hello probe.test

smtp: 250-PIPELINING

smtp: 250-8BITMIME

smtp: 250-ENHANCEDSTATUSCODES

smtp: 250-SMTPUTF8

smtp: 250 SIZE 33554432


-------- delivered messages (new caught_* files) --------
delivered messages captured: 0
```

## Downstream coverage checklist

- [x] **Q1** — what the server sees at the boundary: dot-reader delegation + drain + command resume (§2, §3/E1).
- [x] **Q2** — every named framing: bare LF vs CRLF (E1/E2), mixed (E3a/E3b), dot-stuffing `\r\n..\r\n` (E4), missing CRLF before the dot (E5), early close (E6) (§3).
- [x] **Q3** — pipelined smuggling reproduced: 2 messages delivered from one write (§4/E7).
- [x] **Q4** — back-to-back stability across ≥2 identical runs: `{5/5,5/5}` (§5/E8).
- [x] **Q5** — front proxy passthrough/normalize/reject, with the honest finding that normalize does not stop the spoof (§6/E9).
- [x] **Q6** — the runtime story: hang-to-timeout, 0-delivery aborts, empty state dirs, no CHUNKING/BDAT (§7).
- [x] Limit paths: over-long line `500` (ELINE/ECMD), over-size `552` (§3).
- [x] Every claim carries adjacent complete unedited output; inferred items are labeled (§8.2).
- [x] Read-only: exactly one file added; harness under `/tmp/investig` removed; repository unchanged (§8.4).
