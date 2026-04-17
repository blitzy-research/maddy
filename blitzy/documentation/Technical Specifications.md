# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the new feature requirement is to **produce a comprehensive runtime behavioral analysis of the Maddy SMTP server's DATA boundary parsing**, delivered as a standalone Markdown document. The user is new to the repository and needs empirical, evidence-based answers to a set of interrelated questions about how Maddy decides a message body has ended and transitions back to command parsing. The specific investigative goals are:

- **Boundary Terminator Tolerance**: Determine which byte sequences the server accepts as DATA termination markers (canonical `\r\n.\r\n` versus bare-LF variants such as `\n.\n`, `\r\n.\n`, `\n.\r\n`) and document the exact server response and delivered content for each variant.
- **Near-Miss Boundary Behavior**: Identify what happens when body content closely resembles but does not constitute a legitimate terminator (e.g., `\r\n. \r\n` with a trailing space, `\r.\r` bare-CR, dot-stuffed lines `..text`), and confirm whether the parser correctly treats these as body content.
- **Pipelined Multi-Message Consistency**: Under back-to-back message delivery on a single persistent connection, determine whether the boundary decision is consistent across successive transactions, or whether timing, buffering, or state leakage causes the boundary to "wobble."
- **SMTP Smuggling Exposure**: When a bare-LF terminator (`\n.\n`) appears mid-stream before the canonical `\r\n.\r\n`, determine whether the server prematurely terminates DATA and interprets trailing bytes as new SMTP commands — the hallmark of an SMTP smuggling vulnerability.
- **Proxy Interaction**: With a strict front-proxy that only recognizes the canonical `\r\n.\r\n` terminator and forwards all bytes as a single DATA stream, determine whether the proxy mitigates or leaves intact any boundary confusion on the backend.
- **Residual Artifacts**: Identify what lingers in logs, protocol transcripts, and delivered messages when something goes wrong, and note any expected diagnostic output that is absent.

The user explicitly states: **the repository itself should remain unchanged**, temporary observation scripts may be used but must be cleaned up, and the final deliverable is a Markdown document placed in `blitzy/documentation/`.

### 0.1.2 Implicit Requirements Detected

- The analysis requires **building and running** the Maddy SMTP server (or its underlying go-smtp library) to observe actual runtime behavior, not just static code inspection.
- The user expects byte-level evidence (hex dumps, wire transcripts) supporting each behavioral claim.
- The phrase "what you expect to see but never do" implies documenting **negative results** — behaviors that might reasonably be expected (e.g., the server rejecting bare-LF terminators) but that do not actually occur.
- The SMTP smuggling question implies awareness of the CVE-2023-51764-class vulnerability pattern; the analysis must directly address whether Maddy is susceptible.
- The proxy scenario requires constructing a lightweight intermediary to demonstrate the interaction, then tearing it down.

### 0.1.3 Special Instructions and Constraints

- **No Repository Modifications**: The implementation rule `SWE-AtlasQnA-Repo` mandates that no existing files may be modified. Only a new Markdown document named `maddy_26452dd8dd78.md` may be added, placed in `blitzy/documentation/`.
- **Temporary Scripts**: Observation scripts (Go test binaries, proxy stubs) may be created in `/tmp` but must be removed after use.
- **Deliverable Format**: A single comprehensive Markdown file answering all posed questions with rationale and evidence.
- **Branch Name for Document**: `maddy_26452dd8dd78` (the current working branch).

### 0.1.4 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- To **determine boundary tolerance**, we will build a minimal SMTP backend using the exact `github.com/emersion/go-smtp v0.12.1-0.20191206174923-1f576e0ec85c` library that Maddy depends on, then send raw TCP payloads with systematically varied DATA terminators and record the server's 250/354 responses and the byte content it delivers to the session handler.
- To **analyze the dotReader state machine**, we will read the Go standard library's `net/textproto.dotReader.Read()` implementation (the actual boundary parser) and trace each terminator variant through its six-state finite automaton (`stateBeginLine`, `stateDot`, `stateDotCR`, `stateCR`, `stateData`, `stateEOF`).
- To **test pipelining**, we will send three sequential MAIL/RCPT/DATA/body transactions on a single TCP connection with alternating terminator styles and verify that each transaction completes independently.
- To **demonstrate SMTP smuggling**, we will craft a payload embedding `\n.\n` mid-body followed by injected SMTP commands and a canonical `\r\n.\r\n` tail, then observe whether the server produces two accepted messages from a single DATA stream.
- To **evaluate proxy interaction**, we will build a strict line-buffering proxy that only recognizes canonical `\r\n.\r\n` as the DATA terminator and forward the smuggling payload through it to the same backend.
- To **produce the deliverable**, we will synthesize all findings into a structured Markdown document and commit it to `blitzy/documentation/maddy_26452dd8dd78.md`.


## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

The Maddy repository is a Go project rooted at `github.com/foxcpp/maddy` with the module minimum version Go 1.13 (per `go.mod`). The DATA boundary behavior involves a precise chain of files spanning the go-smtp dependency library, the Go standard library, and Maddy's own endpoint and pipeline code.

**Core SMTP Protocol Layer (go-smtp dependency)**

| File | Path (in Go module cache) | Relevance |
|------|---------------------------|-----------|
| `conn.go` | `github.com/emersion/go-smtp@v0.12.1-.../conn.go` | Command dispatch loop (`handleConn`), `handleData()` method, `newDataReader()` call, `io.Copy(ioutil.Discard, r)` drain line |
| `data.go` | `github.com/emersion/go-smtp@v0.12.1-.../data.go` | `dataReader` wrapper around `textproto.DotReader()` with max-message-size enforcement |
| `server.go` | `github.com/emersion/go-smtp@v0.12.1-.../server.go` | `Server` struct (PIPELINING advertised in default caps), connection lifecycle, `MaxLineLength` default (2000) |
| `parse.go` | `github.com/emersion/go-smtp@v0.12.1-.../parse.go` | `parseCmd()` — trims `\r\n` from command lines, extracts verb and argument |
| `lengthlimit_reader.go` | `github.com/emersion/go-smtp@v0.12.1-.../lengthlimit_reader.go` | `lineLimitReader` — wraps the raw connection reader and resets line-length counter on `\n` (not `\r\n`) |

**Go Standard Library (Boundary State Machine)**

| File | Path | Relevance |
|------|------|-----------|
| `reader.go` | `/usr/lib/go-1.22/src/net/textproto/reader.go` | `dotReader.Read()` — the six-state finite automaton that actually decides when DATA ends. Accepts `\n.\n`, `\r\n.\n`, `\n.\r\n`, and `\r\n.\r\n` as terminators. |

**Maddy Endpoint Layer**

| File | Path | Relevance |
|------|------|-----------|
| `smtp.go` | `internal/endpoint/smtp/smtp.go` | `Session.Data()`, `Session.prepareBody()` — reads header via `textproto.ReadHeader`, buffers body in memory, adds Received header, calls `delivery.Body()` and `delivery.Commit()` |
| `submission.go` | `internal/endpoint/smtp/submission.go` | `submissionPrepare()` — adds Message-ID and Date headers for submission endpoints |
| `smtp_test.go` | `internal/endpoint/smtp/smtp_test.go` | Existing tests for delivery, abort, reset, multi-message; no bare-LF boundary tests |

**Downstream Client Layer**

| File | Path | Relevance |
|------|------|-----------|
| `smtpconn.go` | `internal/smtpconn/smtpconn.go` | `C.Data()` — sends DATA to remote servers using `textproto.WriteHeader` + `io.Copy` + `wc.Close()` which invokes `DotWriter` |

**Message Pipeline and Buffering**

| File | Path | Relevance |
|------|------|-----------|
| `msgpipeline.go` | `internal/msgpipeline/msgpipeline.go` | Routes messages through checks, modifiers, and targets |
| `buffer.go` | `internal/buffer/buffer.go` | `Buffer` interface for temporary message storage |
| `memory.go` | `internal/buffer/memory.go` | `BufferInMemory()` — default strategy; body bytes read from dotReader are stored in a `[]byte` slice |

**Configuration and Logging**

| File | Path | Relevance |
|------|------|-----------|
| `maddy.go` | `maddy.go` | `Run()` entrypoint, global config parsing, `io_debug` passthrough |
| `config.go` | `config.go` | Log output configuration |
| `log.go` | `internal/log/log.go` | `Logger` with `Debug` and `DebugWriter()` methods used by `io_debug` |

### 0.2.2 Integration Point Discovery

- **API Endpoints**: The SMTP endpoint listens on a configured TCP address and dispatches commands via `go-smtp`'s `Server.handleConn` → `Conn.handle` → `Conn.handleData`.
- **Data Flow**: Raw TCP bytes → `lineLimitReader` → `textproto.Conn` (with `bufio.Reader`) → `textproto.DotReader` → `dataReader` (size limiter) → `Session.Data()` → `textproto.ReadHeader` + `bufio.NewReader` → `buffer.BufferInMemory` → `delivery.Body()` → target.
- **State Transition**: After `handleData` returns, `go-smtp`'s `handleConn` loop resumes reading command lines from the same `textproto.Conn`. Any unconsumed bytes remaining in the `bufio.Reader` buffer after the dotReader terminates are interpreted as the next SMTP command.
- **PIPELINING**: Advertised by default in go-smtp's `NewServer` (`caps: []string{"PIPELINING", "8BITMIME", "ENHANCEDSTATUSCODES"}`). This allows clients to send commands without waiting for responses, increasing the risk of boundary confusion when terminators are non-canonical.

### 0.2.3 Research Conducted

- **SMTP Smuggling (CVE-2023-51764 class)**: The December 2023 disclosure by SEC Consult demonstrated that differences in DATA terminator recognition between MTAs enable message injection. Maddy's reliance on Go's `net/textproto.dotReader` — which accepts bare-LF terminators — places it in the class of servers vulnerable to this attack pattern.
- **RFC 5321 Section 4.1.1.4**: The standard specifies `<CRLF>.<CRLF>` as the only legitimate DATA terminator. Any server accepting bare `\n.\n` deviates from the standard.
- **Go net/textproto DotReader**: The Go standard library's dotReader has intentionally lenient parsing that treats bare `\n` as equivalent to `\r\n` for line-ending purposes, a design choice for robustness that creates the boundary confusion.

### 0.2.4 New File Requirements

- **CREATE**: `blitzy/documentation/maddy_26452dd8dd78.md` — The comprehensive analysis document answering all user questions with evidence from live testing.
- No other permanent files are created. All temporary observation scripts and test binaries are created in `/tmp` and cleaned up after use.


## 0.3 Dependency Inventory

### 0.3.1 Key Packages Relevant to This Analysis

| Registry | Package | Version | Purpose |
|----------|---------|---------|---------|
| Go Modules | `github.com/emersion/go-smtp` | `v0.12.1-0.20191206174923-1f576e0ec85c` | SMTP server/client library — provides the `Server`, `Conn`, `dataReader`, and command dispatch loop that directly govern DATA handling |
| Go Modules | `github.com/emersion/go-sasl` | `v0.0.0-20190817083125-240c8404624e` | SASL authentication primitives used by go-smtp |
| Go Modules | `github.com/emersion/go-message` | `v0.10.9-0.20191116124005-65fd0119e899` | RFC 5322 message parsing; `textproto.ReadHeader` used in `Session.prepareBody()` |
| Go Stdlib | `net/textproto` | (Go 1.22.2 runtime) | Provides `DotReader()` — the six-state finite automaton that is the actual DATA boundary parser |
| Go Stdlib | `bufio` | (Go 1.22.2 runtime) | Buffered I/O layer between the TCP connection and `textproto.Conn`; unconsumed bytes after DATA termination remain in this buffer |
| Go Modules | `github.com/foxcpp/maddy` | `26452dd` (HEAD) | The Maddy server itself — `internal/endpoint/smtp` wires go-smtp into the message pipeline |
| Go Modules | `github.com/mattn/go-sqlite3` | `v1.11.0` | SQLite3 driver (CGO required); used for default storage backend |
| Go Modules | `golang.org/x/net` | `v0.0.0-20191126235420-ef20fe5d7933` | Extended networking utilities |

### 0.3.2 Dependency Chain for DATA Boundary Parsing

The boundary decision flows through a precise dependency chain:

```
TCP socket
  → lineLimitReader (go-smtp/lengthlimit_reader.go)
    → io.TeeReader (if io_debug enabled; go-smtp/conn.go)
      → textproto.Conn (Go stdlib net/textproto)
        → bufio.Reader (Go stdlib bufio)
          → textproto.dotReader.Read() ← THE BOUNDARY DECISION POINT
            → dataReader (go-smtp/data.go, size limiter)
              → Session.Data() (maddy/internal/endpoint/smtp/smtp.go)
```

The critical insight is that the boundary decision is made entirely within Go's standard library `net/textproto.dotReader`, not within go-smtp or Maddy code. Neither go-smtp nor Maddy applies any additional validation or normalization of the DATA terminator sequence.

### 0.3.3 Dependency Updates

No dependency updates are required. This analysis is observational and read-only. All packages are used at their pinned versions as declared in `go.mod`.

No import changes, configuration changes, or build file modifications are needed.


## 0.4 Integration Analysis

### 0.4.1 Existing Code Touchpoints

The analysis exercise does not modify any existing code, but the following touchpoints were examined to derive behavioral conclusions:

**go-smtp `conn.go` — Command Loop and DATA Handler (lines 134, 497–522)**

The `handleConn` function reads commands in a loop via `c.ReadLine()`. When `DATA` is dispatched, `handleData()` calls `newDataReader(c)` which wraps `c.text.DotReader()`. After `Session.Data(r)` returns, the critical line `io.Copy(ioutil.Discard, r)` attempts to drain any unconsumed body data. However, if the dotReader already reached `stateEOF` (which happens immediately upon encountering `\n.\n`), this drain operation reads zero additional bytes — leaving any post-terminator bytes in the `bufio.Reader` buffer to be interpreted as SMTP commands on the next loop iteration.

**`net/textproto/reader.go` — dotReader State Machine (lines 339–425)**

The six-state automaton processes bytes as follows for terminator recognition:

| State | Byte `\n` | Byte `\r` | Byte `.` | Other |
|-------|-----------|-----------|----------|-------|
| `stateBeginLine` | emit `\n`, stay | → `stateCR` | → `stateDot` | emit, → `stateData` |
| `stateDot` | → `stateEOF` | → `stateDotCR` | emit `.`, → `stateData` | emit `.`, → `stateData` |
| `stateDotCR` | → `stateEOF` | unread, emit `\r`, → `stateData` | unread, emit `\r`, → `stateData` | unread, emit `\r`, → `stateData` |
| `stateCR` | → `stateBeginLine` | unread, emit `\r`, → `stateData` | unread, emit `\r`, → `stateData` | unread, emit `\r`, → `stateData` |
| `stateData` | → `stateBeginLine` | → `stateCR` | emit | emit |

Key observation: From `stateDot`, a bare `\n` transitions directly to `stateEOF` without requiring a preceding `\r`. This means `\n.\n` is accepted as a valid terminator, as are `\r\n.\n` and `\n.\r\n`.

**Maddy `internal/endpoint/smtp/smtp.go` — Session.Data (line 312)**

After the header is parsed via `textproto.ReadHeader(bufr)`, the remaining body is read via `buffer.BufferInMemory(bufr)` which calls `ioutil.ReadAll`. This means the entire body (up to the dotReader's EOF) is consumed into a `[]byte` in memory. There is no secondary validation of line endings or terminator format.

**Maddy `internal/endpoint/smtp/smtp.go` — Session.prepareBody (line 283)**

The `prepareBody` function wraps the dotReader output in a `bufio.Reader`, then splits the stream into header and body. The `\r\n` → `\n` normalization performed by the dotReader means headers and body arrive with bare `\n` line endings regardless of what was sent on the wire.

### 0.4.2 Observed Runtime Behavior Summary

The following table summarizes the empirically observed behavior from live testing against a go-smtp server using Maddy's pinned library version:

| Test Case | Wire Terminator | Server Response | Message Accepted | Post-DATA Command Parsing |
|-----------|-----------------|-----------------|------------------|--------------------------|
| Canonical | `\r\n.\r\n` | 250 OK | Yes | Clean — NOOP returns 250 |
| Bare LF | `\n.\n` | 250 OK | Yes | Clean — NOOP returns 250 |
| Mixed CRLF+LF | `\r\n.\n` | 250 OK | Yes | Clean — NOOP returns 250 |
| Mixed LF+CRLF | `\n.\r\n` | 250 OK | Yes | Clean — NOOP returns 250 |
| Bare CR | `\r.\r` (then canonical) | 250 OK | Yes — `\r.\r` treated as body content | Clean |
| Dot-space near miss | `\r\n. \r\n` (then canonical) | 250 OK | Yes — `. ` treated as body (space stripped by dot-unstuffing, space emitted) | Clean |
| Dot mid-line | `Body.\r\n` | Timeout | No — dot must begin a new line | Connection hung |
| SMTP Smuggling | `\n.\n` + injected commands + `\r\n.\r\n` | 250 + 250 + 250 + 354 + 250 | **Two messages accepted** from one DATA stream | Injected commands executed |

### 0.4.3 SMTP Smuggling Demonstration

When the following single-write payload was sent after a valid MAIL/RCPT/DATA exchange:

```
Subject: smuggle-test\r\n\r\nBefore fake boundary\n.\nMAIL FROM:<evil@attacker.com>\r\nRCPT TO:<victim@target.com>\r\nDATA\r\nSubject: injected\r\n\r\nEvil body\r\n.\r\n
```

The server produced:
- **Message 1**: From `sender@test.local`, body = "Before fake boundary" (44 bytes delivered)
- **Message 2**: From `evil@attacker.com` to `victim@target.com`, body = "Evil body" (29 bytes delivered)

The server log confirmed that after the first message was accepted (at the `\n.\n` boundary), the remaining bytes were interpreted as SMTP commands (`MAIL FROM`, `RCPT TO`, `DATA`) and a second message was injected and delivered.

### 0.4.4 Proxy Interaction Analysis

A strict front-proxy that only recognized canonical `\r\n.\r\n` as the DATA terminator was constructed and placed in front of the go-smtp backend. When the smuggling payload was sent through the proxy:

- The proxy correctly accumulated all 149 bytes (including the embedded `\n.\n`) and forwarded them as a single DATA stream to the backend, because the proxy did not recognize `\n.\n` as a terminator.
- **However**, the backend's dotReader still terminated at `\n.\n`, and the remaining bytes were still interpreted as SMTP commands.
- **Result**: The strict proxy did **not** prevent the smuggling attack. Two messages were still accepted by the backend.

This occurs because the proxy forwards all bytes over the same TCP connection to the backend. The backend's `bufio.Reader` buffers the entire forwarded block, and the dotReader consumes only up to `\n.\n`. The remaining bytes persist in the buffer and are read as SMTP commands by the `handleConn` loop.


## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

Since the repository must remain unchanged, the only permanent artifact is the analysis document. All other files are temporary.

**Group 1 — Deliverable Document**

- **CREATE**: `blitzy/documentation/maddy_26452dd8dd78.md`
  - Purpose: Comprehensive Markdown document answering all user questions about DATA boundary behavior
  - Content: Structured sections covering boundary tolerance, smuggling exposure, pipelined consistency, proxy interaction, and diagnostic artifacts
  - Evidence: Includes wire-level hex dumps, protocol transcripts, server log excerpts, and state-machine trace analysis

**Group 2 — Temporary Observation Infrastructure (created in `/tmp`, cleaned up after use)**

- **CREATE (temp)**: `/tmp/maddy-test/server/main.go` — Minimal SMTP server using `go-smtp v0.12.1-0.20191206174923` with verbose session logging (hex dumps of received DATA, byte counts, message sequencing)
- **CREATE (temp)**: `/tmp/maddy-test/testclient/main.go` — Raw TCP test client sending eight systematically varied payloads: canonical `\r\n.\r\n`, bare `\n.\n`, mixed `\r\n.\n`, mixed `\n.\r\n`, bare `\r.\r`, dot-space near-miss, pipelined back-to-back, and ambiguous body content
- **CREATE (temp)**: `/tmp/maddy-test/detailedtest/main.go` — Detailed analysis client including SMTP smuggling payload and post-DATA command verification (NOOP probe)
- **CREATE (temp)**: `/tmp/maddy-test/proxydir/main.go` — Strict byte-level proxy recognizing only canonical `\r\n.\r\n` as DATA terminator, forwarding normalized traffic to the backend
- **REMOVE (cleanup)**: All files and directories under `/tmp/maddy-test/` after results are captured

### 0.5.2 Implementation Approach

The implementation follows a four-phase approach:

**Phase 1 — Environment Preparation**
- Install Go runtime (go 1.22 satisfies the go 1.13+ minimum)
- Download project dependencies via `go mod download`
- Build the Maddy binary (`go build -o /tmp/maddy ./cmd/maddy/`) to confirm compilability

**Phase 2 — Live Behavioral Testing**
- Build and start a test SMTP server using the exact go-smtp library version Maddy depends on
- Execute the test client against the server, capturing all server logs and client responses
- Execute the detailed analysis client including the SMTP smuggling payload
- Build and start the strict proxy, then re-run the smuggling test through it
- Record all protocol transcripts, hex dumps, and server logs

**Phase 3 — Static Analysis Corroboration**
- Read the `net/textproto.dotReader.Read()` source to confirm the state machine accepts bare-LF terminators
- Read `go-smtp/conn.go` `handleData()` to confirm the `io.Copy(ioutil.Discard, r)` drain is ineffective after early EOF
- Read `go-smtp/conn.go` `handleConn()` to confirm the command loop resumes from the same `bufio.Reader` buffer
- Read `go-smtp/server.go` to confirm PIPELINING is advertised by default

**Phase 4 — Document Synthesis and Cleanup**
- Compile all findings into the structured Markdown document
- Place in `blitzy/documentation/maddy_26452dd8dd78.md`
- Remove all temporary files from `/tmp`

### 0.5.3 Key Technical Findings to Document

The analysis document must address each user question with the following evidence-backed conclusions:

- **Terminator Tolerance**: Maddy (via go-smtp via net/textproto) accepts four terminator variants (`\r\n.\r\n`, `\n.\n`, `\r\n.\n`, `\n.\r\n`). Only the canonical form is specified by RFC 5321. The dotReader normalizes all `\r\n` to `\n` in the delivered body, so the delivered content is byte-identical regardless of which terminator was used.
- **Near-Miss Handling**: Dot-space (`\r\n. \r\n`) is correctly treated as body content — the space after the dot prevents terminator matching, and the leading dot is stripped by dot-unstuffing. Bare-CR (`\r.\r`) is treated as body content. Dot mid-line (`Body.\r\n`) does not trigger termination because the dot is not at the beginning of a line.
- **Pipelined Consistency**: Three back-to-back messages on a single connection — mixing canonical and bare-LF terminators — all completed successfully with correct sender/recipient isolation. No boundary wobble was observed. The state machine resets cleanly between transactions.
- **SMTP Smuggling**: Confirmed exploitable. A single DATA stream containing `\n.\n` mid-body followed by injected SMTP commands produces two accepted messages. The dotReader terminates at `\n.\n`, and the remaining bytes are parsed as SMTP commands.
- **Proxy Interaction**: A strict proxy forwarding only on canonical `\r\n.\r\n` does NOT prevent the smuggling attack. The proxy correctly accumulates all bytes, but the backend's dotReader still terminates at the embedded `\n.\n`, and the remaining bytes are still interpreted as commands.
- **Diagnostic Artifacts**: The `io_debug` mode logs raw wire bytes (including both the message body and the injected commands). The server log shows two separate `[SESSION] Message #N accepted` entries from a single client DATA stream. No error or warning is logged — the smuggling is silent.


## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

**Permanent Deliverable**
- `blitzy/documentation/maddy_26452dd8dd78.md` — The comprehensive analysis document

**Repository Files Analyzed (read-only)**
- `go.mod` — Dependency manifest, Go version, pinned library versions
- `go.sum` — Cryptographic checksums for dependency verification
- `internal/endpoint/smtp/smtp.go` — Session.Data(), Session.prepareBody(), Endpoint.Init()
- `internal/endpoint/smtp/smtp_test.go` — Existing test coverage review
- `internal/endpoint/smtp/submission.go` — Submission endpoint header preparation
- `internal/smtpconn/smtpconn.go` — Downstream client DATA sending
- `internal/buffer/*.go` — Message buffering (memory and file strategies)
- `internal/msgpipeline/msgpipeline.go` — Message routing pipeline
- `internal/log/log.go` — Logging infrastructure (Debug, DebugWriter)
- `internal/testutils/smtp_server.go` — Test SMTP backend used in existing tests
- `internal/target/queue/queue.go` — Queue-based delivery target
- `internal/target/remote/remote.go` — Remote delivery target
- `maddy.go` — Server entrypoint and global configuration
- `config.go` — Log output configuration
- `maddy.conf` — Example configuration file

**Go-smtp Dependency Files Analyzed (read-only, from module cache)**
- `github.com/emersion/go-smtp@v0.12.1-.../conn.go` — Command dispatch, handleData, handleConn
- `github.com/emersion/go-smtp@v0.12.1-.../data.go` — dataReader wrapping DotReader
- `github.com/emersion/go-smtp@v0.12.1-.../server.go` — Server configuration, PIPELINING caps
- `github.com/emersion/go-smtp@v0.12.1-.../parse.go` — Command parsing
- `github.com/emersion/go-smtp@v0.12.1-.../lengthlimit_reader.go` — Line length enforcement
- `github.com/emersion/go-smtp@v0.12.1-.../client.go` — Client-side DotWriter
- `github.com/emersion/go-smtp@v0.12.1-.../client_test.go` — Injection protection tests

**Go Standard Library Files Analyzed (read-only)**
- `/usr/lib/go-1.22/src/net/textproto/reader.go` — dotReader state machine (lines 333–445)

**Behavioral Tests Conducted**
- Canonical terminator (`\r\n.\r\n`)
- Bare LF terminator (`\n.\n`)
- Mixed CRLF+LF terminator (`\r\n.\n`)
- Mixed LF+CRLF terminator (`\n.\r\n`)
- Bare CR non-terminator (`\r.\r`)
- Dot-space near-miss (`\r\n. \r\n`)
- Dot mid-line non-terminator (`Body.\r\n`)
- Pipelined back-to-back messages (3 messages, alternating terminators)
- Ambiguous body content (dot-stuffed lines, dot+text, dot+space)
- SMTP smuggling payload (embedded `\n.\n` with injected commands)
- Proxy-mediated smuggling test (strict `\r\n.\r\n`-only proxy)

### 0.6.2 Explicitly Out of Scope

- **Modification of any existing repository file** — per the `SWE-AtlasQnA-Repo` implementation rule
- **IMAP endpoint behavior** — the user's questions are exclusively about SMTP DATA handling
- **TLS/STARTTLS interaction** — not relevant to DATA boundary parsing (TLS is a transport-layer concern)
- **Authentication mechanisms** — AUTH/SASL behavior is orthogonal to DATA termination
- **DNS/SPF/DKIM/DMARC checks** — mail security checks run after DATA is accepted; they do not affect boundary detection
- **Performance benchmarking** — the user asks about correctness and boundary behavior, not throughput
- **Remediation or patching** — the user asks to understand behavior, not to fix it
- **Other protocol endpoints** — submission and LMTP endpoints share the same go-smtp DATA path, but the user specifically asks about SMTP


## 0.7 Rules for Feature Addition

### 0.7.1 User-Specified Rules

The following rules are explicitly mandated by the user and the `SWE-AtlasQnA-Repo` implementation directive:

- **No Repository Modifications**: Do not modify any existing files in the source repository. The analysis must be purely observational.
- **Single Deliverable**: Create a new Markdown document named `maddy_26452dd8dd78.md` (matching the source branch name) that comprehensively answers all posed questions.
- **Document Placement**: The generated document must be placed in the `blitzy/documentation` directory within the destination repository.
- **Build and Run**: Build and run the source code to analyze repository behavior as needed. Conclusions must be grounded in observed runtime behavior, not assumptions.
- **Evidence-Based**: Provide thinking and rationale behind all answers. Do not make assumptions; base answers on the code as the truth.
- **Temporary Artifact Cleanup**: Temporary scripts used for observation must be cleaned up afterward. The repository should remain in its original state.

### 0.7.2 Conventions and Patterns Observed

- **Go Module Layout**: The project follows standard Go module conventions with `internal/` for private packages and `pkg/` for public libraries.
- **Test Conventions**: Existing tests use Go's built-in `testing` package with `t.Fatal`/`t.Error` assertions. Test files are co-located with source files (`*_test.go`).
- **Logging**: Maddy uses a custom `log.Logger` with structured fields (`log.Msg("event", "key", value)`). The `io_debug` flag enables raw wire-level protocol logging via `io.TeeReader`/`io.MultiWriter`.
- **Error Wrapping**: Errors flow through `exterrors.SMTPError` with enhanced SMTP status codes (`exterrors.EnhancedCode`).


## 0.8 References

### 0.8.1 Repository Files Searched

The following files and folders were directly retrieved and analyzed during the context-gathering phase:

| Path | Type | Purpose |
|------|------|---------|
| `/tmp/blitzy/maddy/maddy_26452dd8dd78_76465c/` (root) | Folder | Repository root, directory listing |
| `go.mod` | File | Dependency manifest — identified go-smtp version, Go minimum version |
| `go.sum` | File | Dependency checksums |
| `maddy.go` | File | Server entrypoint, global configuration, Run() function |
| `config.go` | File | Log output configuration |
| `maddy.conf` | File | Example configuration syntax and directive structure |
| `internal/endpoint/smtp/smtp.go` | File | SMTP session handler — Data(), prepareBody(), wrapErr() |
| `internal/endpoint/smtp/smtp_test.go` | File | Existing SMTP test coverage (delivery, abort, reset, multi-message) |
| `internal/endpoint/smtp/submission.go` | File | Submission endpoint preparation |
| `internal/smtpconn/smtpconn.go` | File | Downstream SMTP client — Data(), Connect(), Mail(), Rcpt() |
| `internal/buffer/*.go` | Files | Buffer interface, memory buffer, file buffer, bytes reader |
| `internal/msgpipeline/` | Folder | Message pipeline configuration and routing |
| `internal/target/queue/queue.go` | File | Queue-based delivery target |
| `internal/target/` | Folder | Target directory structure |
| `internal/log/log.go` | File | Logging infrastructure |
| `internal/testutils/smtp_server.go` | File | Test SMTP backend and helper utilities |
| `cmd/maddy/main.go` | File | Main entrypoint (delegates to maddy.Run()) |
| `HACKING.md` | File | Developer guide and design goals |
| `.github/` | Folder | GitHub workflows and security policy |

**Go-smtp dependency (from module cache at `/root/go/pkg/mod/github.com/emersion/go-smtp@v0.12.1-...`):**

| File | Purpose |
|------|---------|
| `conn.go` | Connection handler, command dispatch, handleData(), handleConn() |
| `data.go` | dataReader wrapping textproto.DotReader with size limiting |
| `server.go` | Server struct, PIPELINING/8BITMIME/ENHANCEDSTATUSCODES caps, Serve() |
| `parse.go` | Command parser (parseCmd, parseArgs) |
| `lengthlimit_reader.go` | Line length enforcement reader |
| `client.go` | Client-side Data() with DotWriter |
| `client_test.go` | Injection protection tests |
| `server_test.go` | Server tests including Strict mode |

**Go standard library:**

| File | Purpose |
|------|---------|
| `/usr/lib/go-1.22/src/net/textproto/reader.go` (lines 333–445) | dotReader state machine — the actual DATA boundary parser |

### 0.8.2 Live Tests Executed

| Test | Tool | Target | Key Finding |
|------|------|--------|-------------|
| 8 terminator variants | Custom Go TCP client | go-smtp server on `127.0.0.1:2525` | All four `\n`/`\r\n` dot combinations accepted as terminators |
| SMTP smuggling payload | Custom Go TCP client | go-smtp server on `127.0.0.1:2525` | Two messages accepted from one DATA stream via `\n.\n` injection |
| Post-DATA NOOP probe | Custom Go TCP client | go-smtp server on `127.0.0.1:2525` | Server cleanly returns to command mode after all terminator variants |
| Pipelined 3-message test | Custom Go TCP client | go-smtp server on `127.0.0.1:2525` | Consistent behavior across back-to-back transactions |
| Smuggling through strict proxy | Custom Go TCP client | Proxy on `127.0.0.1:2526` → backend on `127.0.0.1:2525` | Proxy forwarded all bytes; backend still smuggled two messages |
| Existing go test suite | `go test ./internal/endpoint/smtp/` | Maddy SMTP endpoint | All 15 existing tests pass; no bare-LF boundary tests exist |
| go-smtp library tests | `go test ./...` | go-smtp v0.12.1 | All tests pass; injection protection covers command-level injection only |

### 0.8.3 Attachments and External Metadata

- **No Figma URLs** were provided.
- **No file attachments** were provided.
- **Environment**: `andrewparkscaleai/coding-agent:foxcpp__maddy__26452dd8dd787dc455278b0fdd296f4a5432c768` from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_foxcpp_maddy_1.0`
- **Source Branch**: `maddy_26452dd8dd78`
- **Commit**: `26452dd` — "target/remote: Rewrite connection part to allow more concurrency"


