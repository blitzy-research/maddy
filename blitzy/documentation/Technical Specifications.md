# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively answers a set of investigative questions about Maddy mail server's SMTP DATA boundary handling behavior at runtime. The user is new to the Maddy repository and seeks a thorough, evidence-based runtime analysis document rather than theoretical code review.

- **Documentation Type**: Technical investigation report / Runtime behavior analysis
- **Category**: Create new documentation
- **Output**: A single Markdown document named `maddy_26452dd8dd78.md` placed in the `blitzy/documentation` directory

The user's requirements decompose into the following distinct investigation areas:

- **Boundary Decision Mechanics**: What the server actually does at the precise moment it decides the DATA phase is finished and transitions back to command parsing, particularly when line endings are varied (`\r\n.\r\n` vs `\n.\n` vs mixed variants)
- **Payload Comparison Under Varied Framing**: How behavior changes when payloads are identical except for how the DATA boundary terminator is framed (CRLF vs bare LF vs mixed)
- **Smuggling Risk via Lenient Parsing**: Whether Maddy accepts something a stricter SMTP peer would still consider part of the message body, and how that mismatch surfaces in delivered content or rejection
- **Pipelining Pressure Consistency**: Whether back-to-back messages over a single connection exhibit any boundary "wobble" or timing/buffering inconsistency
- **Proxy Interaction**: How the story changes when the same traffic passes through a small front proxy that normalizes or blocks ambiguous framing
- **Log and Transcript Evidence**: What shows up in logs, protocol transcripts, and delivered messages that reveals the behavioral differences
- **Failure Residue**: What lingers when something goes wrong, and what you expect to see but never do

### 0.1.2 Special Instructions and Constraints

The user has specified critical operational constraints:

- **No repository modifications**: The existing source repository must remain unchanged. Temporary scripts may be used for observation but must be cleaned up afterward.
- **Output location**: The generated document must be placed in `blitzy/documentation/maddy_26452dd8dd78.md`
- **Evidence-based answers**: Do not make assumptions; base all answers on the code as the source of truth
- **Provide rationale**: Include the thinking and reasoning behind all answers
- **Build and run**: The source code should be built and run to analyze repository behavior as needed

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document boundary decision mechanics**, we will trace the code path from `go-smtp`'s `handleData()` through Go's `net/textproto.DotReader()` state machine, identifying exactly which byte sequences trigger the transition from DATA to command mode, and validate by sending crafted payloads to a running server
- To **compare payload behavior under varied framing**, we will send identical message bodies with four terminator variants (`\r\n.\r\n`, `\n.\n`, `\r\n.\n`, `\n.\r\n`) and compare the delivered body bytes, response codes, and timing
- To **assess smuggling risk**, we will construct a payload where a bare-LF dot terminator is followed by SMTP commands that a stricter server would treat as body content, and observe whether Maddy processes them as new commands
- To **test pipelining consistency**, we will send multiple complete message transactions in rapid succession (including in a single TCP write) and measure timing and response code consistency
- To **analyze proxy interaction**, we will simulate a strict proxy that rejects non-RFC terminators and compare the protocol exchange against direct-to-server delivery
- To **capture evidence**, we will enable `io_debug` logging in the go-smtp server layer and record raw protocol transcripts, body hex dumps, and timing measurements

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs have been identified:

- The critical dependency on Go's `net/textproto.DotReader()` state machine (from Go 1.13's standard library) needs to be documented, as it is the sole component responsible for DATA boundary detection; Maddy's own code contains no custom boundary logic
- The PIPELINING capability advertised by go-smtp and how it interacts with the DATA phase needs documentation
- The `io.Copy(ioutil.Discard, r)` drain pattern in go-smtp's `handleData()` that ensures clean state transitions after DATA needs to be explained
- The DotReader's `\r\n` → `\n` normalization behavior that makes all delivered bodies lose their original CRLF line endings needs to be highlighted as it has implications for message integrity verification

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a MkDocs-based documentation system with limited SMTP internals coverage. The documentation infrastructure consists of:

- **Documentation framework**: MkDocs with the ReadTheDocs theme, configured in `.mkdocs.yml`
- **Documentation site**: Published at `https://foxcpp.dev/maddy/`
- **Markdown extensions**: `codehilite` (with `guess_lang: false`)
- **Navigation structure**: Organized into Tutorials, Manual Pages, and Internals sections
- **Man pages**: 9 scdoc-formatted pages with a Python converter (`docs/man/prepare_md.py`)
- **Internals documentation**: Only two pages — `internals/quirks.md` and `internals/sqlite.md`

The existing `docs/internals/quirks.md` documents SMTP quirks at a high level (e.g., `Received` header `for` field omission) but contains **no documentation** about DATA boundary handling, line ending behavior, dot-stuffing, SMTP smuggling risks, or pipelining behavior.

### 0.2.2 Repository Code Analysis for Documentation

The following source code areas were examined to build the investigation:

**Primary SMTP Server Path**:
- `internal/endpoint/smtp/smtp.go` — Endpoint and Session types; `Data()` and `LMTPData()` methods that receive an `io.Reader` from go-smtp's DotReader
- `internal/endpoint/smtp/smtp_test.go` — Existing tests covering delivery, abort, reset, multi-message, submission auth (no DATA boundary edge-case tests)
- `internal/endpoint/smtp/submission.go` — Submission-mode header normalization in `submissionPrepare()`

**go-smtp Library** (version `v0.12.1-0.20191206174923-1f576e0ec85c`):
- `conn.go` — Connection handler with `handleData()`, `handleConn()` main loop, `init()` setting up `textproto.Conn`
- `data.go` — `dataReader` wrapping `textproto.DotReader()` with size limits
- `server.go` — Server initialization advertising `PIPELINING`, `8BITMIME`, `ENHANCEDSTATUSCODES`
- `lengthlimit_reader.go` — Line length enforcement wrapping the raw connection reader

**Go Standard Library** (`net/textproto` from Go 1.13):
- `reader.go` — `DotReader()` and `dotReader.Read()` state machine with 6 states handling boundary detection

**Supporting Code**:
- `internal/buffer/buffer.go` — Immutable buffer abstraction for message bodies
- `internal/testutils/smtp_server.go` — SMTP test harness with in-memory delivery capture
- `internal/smtpconn/smtpconn.go` — Outbound SMTP client wrapper

### 0.2.3 Web Search Research Conducted

No external web search was required for this investigation. The answers are derived entirely from:
- Source code analysis of Maddy's SMTP endpoint and its go-smtp dependency
- Go 1.13 standard library source code (`net/textproto` DotReader implementation)
- Runtime testing against a live go-smtp server instance built from the repository's exact dependency versions
- Existing repository documentation (`docs/internals/quirks.md`, `HACKING.md`)

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The investigation spans a precisely defined chain of modules from the network socket to the delivered message buffer:

- **Module: go-smtp `conn.go`** (`go-smtp@v0.12.1-0.20191206174923-1f576e0ec85c`)
  - Key functions: `handleConn()` main loop, `handleData()`, `init()`, `reset()`
  - Current documentation: None in the Maddy repository; the go-smtp library is a third-party dependency
  - Documentation needed: How `handleData()` creates a DotReader, invokes `Session.Data(r)`, drains residual data with `io.Copy(ioutil.Discard, r)`, and returns to the command loop

- **Module: go-smtp `data.go`**
  - Key functions: `newDataReader()` wrapping `c.text.DotReader()`
  - Current documentation: None
  - Documentation needed: How `dataReader` adds size limiting on top of textproto's DotReader

- **Module: Go stdlib `net/textproto` `reader.go`** (Go 1.13)
  - Key functions: `DotReader()`, `dotReader.Read()` with its 6-state FSM
  - Current documentation: Go standard library godoc exists but is not referenced from Maddy docs
  - Documentation needed: The exact state machine behavior, which terminator variants are accepted, and CRLF-to-LF normalization

- **Module: `internal/endpoint/smtp/smtp.go`**
  - Key functions: `Session.Data()`, `Session.prepareBody()`, `Session.LMTPData()`
  - Current documentation: None beyond code comments
  - Documentation needed: How the Maddy session layer consumes the DotReader output, parses headers, buffers the body, and commits delivery

- **Module: go-smtp `server.go`**
  - Key capabilities: `PIPELINING`, `8BITMIME`, `ENHANCEDSTATUSCODES` advertised by default
  - Current documentation: None specific to pipelining interaction with DATA
  - Documentation needed: How pipelined commands interact with the DATA phase and post-DATA state transition

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented runtime behavior**: No existing documentation describes what happens at the byte level when Maddy receives a DATA terminator. The `quirks.md` file documents `Received` header and IMAP quirks but omits all DATA boundary behavior.
- **Missing smuggling risk assessment**: The repository contains no documentation about SMTP smuggling risks arising from the lenient DotReader. This is a security-relevant gap.
- **No pipelining interaction docs**: While the server advertises PIPELINING, there is no documentation about how pipelined commands interact with DATA mode or how the state machine handles the transition.
- **No proxy interaction guidance**: There is no documentation about how Maddy behaves behind front proxies, or how ambiguous framing propagates through relay chains.
- **No body normalization documentation**: The fact that all CRLF sequences in message bodies are silently converted to bare LF by the DotReader is not documented anywhere, despite having implications for DKIM signature verification and content integrity.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document follows a single-file investigation report structure:

```
blitzy/documentation/
└── maddy_26452dd8dd78.md
    ├── Introduction and Context
    │   ├── Purpose and scope
    │   └── Architecture overview (DotReader chain)
    ├── Boundary Decision Mechanics
    │   ├── DotReader state machine analysis
    │   ├── Accepted terminator variants
    │   └── Code path walkthrough
    ├── Payload Comparison Under Varied Framing
    │   ├── Test methodology
    │   ├── Response code comparison
    │   ├── Delivered body byte-level comparison
    │   └── CRLF normalization behavior
    ├── SMTP Smuggling Risk Assessment
    │   ├── Attack scenario
    │   ├── Runtime evidence
    │   └── Implications
    ├── Pipelining Pressure and Consistency
    │   ├── Single-connection multi-message test
    │   ├── Mega-pipeline test
    │   └── Timing measurements
    ├── Proxy Interaction Analysis
    │   ├── Strict proxy behavior
    │   ├── Normalizing proxy behavior
    │   └── Error surface comparison
    ├── Log and Transcript Evidence
    │   ├── What appears in logs
    │   └── What is expected but absent
    └── Conclusions and Rationale
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach**:
- Extract DotReader state machine logic from Go 1.13's `net/textproto/reader.go` source
- Extract connection handling from `go-smtp@v0.12.1` `conn.go` and `data.go`
- Extract Maddy's DATA handling from `internal/endpoint/smtp/smtp.go` lines 283-344
- Generate protocol transcripts and body hex dumps by running a go-smtp test server built from the project's exact dependencies and sending crafted payloads via raw TCP sockets

**Runtime Test Methodology**:
- Build the go-smtp backend directly from Maddy's module dependencies (Go 1.13, go-smtp v0.12.1-pre)
- Send payloads using raw TCP (Python socket or Go net.Dial) to avoid client-side normalization
- Enable debug output (`s.Debug = os.Stderr`) to capture wire-level protocol transcripts
- Record body content as hex dumps for byte-level comparison
- Measure timing with monotonic clocks across multiple terminator variants

**Documentation Standards**:
- Mermaid diagrams for the DotReader state machine and the DATA-to-command transition
- Markdown tables for payload comparison results
- Inline hex dumps and protocol transcripts in fenced code blocks
- Source citations as `Source: /path/to/file.go:LineNumber` throughout

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams will be created:

- **DotReader State Machine**: A state diagram showing the 6 states (`stateBeginLine`, `stateDot`, `stateDotCR`, `stateCR`, `stateData`, `stateEOF`) with transitions labeled by input byte. Source: Go 1.13 `net/textproto/reader.go:317-393`
- **DATA-to-Command Transition**: A sequence diagram showing the flow from `handleData()` through DotReader consumption, `io.Copy` drain, response write, `reset()`, and return to the main `handleConn()` command loop. Source: `go-smtp conn.go`
- **Smuggling Attack Flow**: A sequence diagram comparing what Maddy sees vs what a strict server would see when processing a `\n.\n` payload with trailing commands

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/maddy_26452dd8dd78.md` | CREATE | `internal/endpoint/smtp/smtp.go`, go-smtp `conn.go`, go-smtp `data.go`, Go stdlib `net/textproto/reader.go`, `docs/internals/quirks.md` | Complete investigation report covering DATA boundary mechanics, payload comparison, smuggling risk, pipelining consistency, proxy interaction, and log evidence |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/maddy_26452dd8dd78.md
Type: Technical Investigation Report / Runtime Behavior Analysis
Source Code:
  - go-smtp (v0.12.1-pre) conn.go — handleData(), handleConn() main loop
  - go-smtp (v0.12.1-pre) data.go — newDataReader(), dataReader.Read()
  - go-smtp (v0.12.1-pre) server.go — PIPELINING capability, NewServer()
  - go-smtp (v0.12.1-pre) lengthlimit_reader.go — line length enforcement
  - Go 1.13 net/textproto/reader.go — DotReader(), dotReader.Read() state machine
  - internal/endpoint/smtp/smtp.go — Session.Data(), Session.prepareBody()
  - internal/endpoint/smtp/smtp_test.go — existing test patterns
  - internal/buffer/buffer.go — Buffer interface for body storage
  - docs/internals/quirks.md — existing known quirks
Sections:
  - Introduction and architectural context
  - DotReader state machine analysis with state diagram
  - Accepted terminator variant catalog (4 variants)
  - Payload comparison with hex dumps and response codes
  - SMTP smuggling demonstration and risk assessment
  - Pipelining pressure test results
  - Proxy interaction analysis (strict vs normalizing)
  - Log evidence catalog (what appears, what is absent)
  - Timing consistency measurements
  - Conclusions with rationale
Diagrams:
  - DotReader 6-state FSM (Mermaid stateDiagram)
  - DATA-to-command transition sequence (Mermaid sequenceDiagram)
  - Smuggling attack comparison (Mermaid sequenceDiagram)
Key Citations:
  - go-smtp conn.go handleData() — DATA command handler
  - Go 1.13 net/textproto/reader.go lines 283-398 — DotReader implementation
  - internal/endpoint/smtp/smtp.go lines 283-344 — Maddy's Data/prepareBody methods
  - go-smtp server.go NewServer() — capability advertisement
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration changes are needed. The output document is a standalone Markdown file in `blitzy/documentation/` and does not require integration with the MkDocs navigation or any existing documentation build system.

### 0.5.4 Cross-Documentation Dependencies

- The investigation references but does not modify `docs/internals/quirks.md`
- No shared content, navigation links, or index updates are required
- The document is self-contained with all source citations inline

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following tools and packages are relevant to this documentation exercise:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| Go module | `github.com/emersion/go-smtp` | `v0.12.1-0.20191206174923-1f576e0ec85c` | SMTP server library that implements the connection handler, DATA command, and DotReader integration; the primary subject of this investigation |
| Go stdlib | `net/textproto` | Go 1.13.15 | Provides `DotReader()` and its 6-state FSM that performs the actual DATA boundary detection and CRLF-to-LF normalization |
| Go module | `github.com/emersion/go-message` | `v0.10.9-0.20191116124005-65fd0119e899` | Provides `textproto.Header` and `textproto.ReadHeader` used in Maddy's `prepareBody()` to parse headers from the DotReader output |
| Go toolchain | `go` | `1.13.15` | Required Go version per `go.mod`; the `net/textproto` behavior is specific to this version |
| System | `gcc` | 13.2.0 | Required for CGO compilation of `go-sqlite3` dependency during build |
| pip | `mkdocs` | (configured in `.mkdocs.yml`) | Existing documentation site generator; not modified by this task |

### 0.6.2 Documentation Reference Updates

No link updates are required. The output document is a new standalone file that does not replace or redirect any existing documentation paths.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

The user's prompt contains seven distinct investigative questions. Coverage targets:

| Question Area | Coverage Target | Validation Method |
|---|---|---|
| Boundary decision mechanics (DotReader FSM) | 100% — all 6 states and all transitions documented | State diagram matches Go 1.13 source code |
| Payload comparison (4 terminator variants) | 100% — all 4 variants tested and compared | Hex dumps of delivered bodies confirm behavior |
| Smuggling risk assessment | 100% — attack demonstrated with protocol transcript | Live test confirms smuggled commands are executed |
| Pipelining pressure consistency | 100% — 10+ back-to-back messages measured | Timing data shows no wobble or inconsistency |
| Proxy interaction | 100% — strict and normalizing proxy behaviors analyzed | Comparison shows how proxy changes error surface |
| Log/transcript evidence | 100% — all observable artifacts cataloged | Debug output captures wire-level protocol |
| Failure residue and missing signals | 100% — expected but absent behaviors documented | Analysis of what logs do NOT show |

### 0.7.2 Documentation Quality Criteria

- **Completeness**: Every question from the user's prompt is answered with code evidence and runtime test results; no question is deferred or answered speculatively
- **Accuracy**: All claims are traced to specific source file locations (file path + line number) and validated against runtime behavior; delivered body content is verified at the hex/byte level
- **Reproducibility**: The testing methodology is documented clearly enough that another developer could reproduce the same results by building from the same commit and sending the same raw payloads
- **Clarity**: Technical explanations use progressive disclosure — starting with a high-level summary, then drilling into the state machine, then showing concrete protocol transcripts
- **Rationale**: Every conclusion includes the reasoning chain that led to it, as required by the user's rules

### 0.7.3 Example and Diagram Requirements

- Minimum 1 Mermaid state diagram (DotReader FSM)
- Minimum 1 Mermaid sequence diagram (DATA transition or smuggling flow)
- Protocol transcript examples for each terminator variant
- Hex dump comparison table for delivered body content
- Timing measurement table for consistency analysis

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

- **New documentation file**:
  - `blitzy/documentation/maddy_26452dd8dd78.md` — the complete investigation report

- **Source code analyzed** (read-only, not modified):
  - `internal/endpoint/smtp/smtp.go` — Maddy's SMTP session and DATA handling
  - `internal/endpoint/smtp/smtp_test.go` — existing test patterns for context
  - `internal/endpoint/smtp/submission.go` — submission header normalization
  - `internal/endpoint/smtp/date.go` — date parsing helper
  - `internal/buffer/buffer.go` — buffer interface
  - `internal/smtpconn/smtpconn.go` — outbound SMTP client
  - `internal/testutils/smtp_server.go` — test harness
  - `docs/internals/quirks.md` — existing known quirks
  - `go.mod` — dependency versions
  - `.mkdocs.yml` — documentation configuration
  - `HACKING.md` — contributor design guide

- **Third-party library source analyzed** (read-only):
  - `go-smtp@v0.12.1-pre` `conn.go`, `data.go`, `server.go`, `lengthlimit_reader.go`
  - Go 1.13 `net/textproto/reader.go` (DotReader implementation)

- **Runtime testing** (temporary, cleaned up):
  - Building and running a go-smtp test server from the project's dependency graph
  - Sending crafted raw TCP payloads to test DATA boundary behavior
  - Collecting protocol transcripts, body hex dumps, and timing measurements

### 0.8.2 Explicitly Out of Scope

- Source code modifications of any kind to the Maddy repository
- Modifications to existing documentation files (e.g., `docs/internals/quirks.md`)
- Test file additions or modifications within the repository
- Any changes to go-smtp, textproto, or other dependencies
- Feature additions, bug fixes, or code refactoring
- Deployment configuration changes
- Documentation for IMAP endpoint behavior
- Documentation for outbound SMTP client (`internal/smtpconn`) behavior (beyond its relevance to understanding the server)
- Performance benchmarking beyond timing consistency measurements
- Full security audit (only the DATA boundary smuggling vector is analyzed)
- Changes to the MkDocs site configuration or navigation

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Build command**: `CGO_ENABLED=1 go build -o /tmp/maddy-bin ./cmd/maddy` (requires Go 1.13.15 and gcc for CGO/sqlite3)
- **Test server command**: Build a standalone go-smtp server from the project's module graph using `go build -o /tmp/test_smtp_server /tmp/test_smtp_server.go` within the module context
- **Runtime test approach**: Raw TCP socket connections (Python `socket` module or Go `net.Dial`) to avoid any client-side CRLF normalization that higher-level SMTP client libraries perform
- **Default format**: Markdown with Mermaid diagrams, fenced code blocks for protocol transcripts and hex dumps
- **Citation requirement**: Every technical claim references the specific source file and line number(s)
- **Cleanup requirement**: All temporary test scripts, binaries, and log files must be removed after testing; only the output document in `blitzy/documentation/` persists

### 0.9.2 Key Runtime Test Parameters

| Parameter | Value | Rationale |
|---|---|---|
| Go version | 1.13.15 | Matches `go.mod` requirement; `net/textproto` behavior is version-specific |
| go-smtp version | v0.12.1-0.20191206174923-1f576e0ec85c | Exact version from `go.mod` |
| Test server port | 9025 | Unprivileged port to avoid root requirement |
| Server debug output | Enabled (`s.Debug = os.Stderr`) | Captures wire-level protocol for transcript analysis |
| Read timeout | 30s | Generous timeout to avoid premature connection drops during slow tests |
| Max message size | 1 MB | Sufficient for test payloads |
| Auth | Disabled | Simplifies testing; not relevant to DATA boundary behavior |

## 0.10 Rules for Documentation

The following rules are explicitly specified by the user and must be observed:

- **No repository modifications**: Do not modify any existing files in the source repository. The Maddy codebase must remain byte-identical to its original state after the task is complete.
- **No code additions to repository**: Do not add any other code in the source repository besides the requested documentation file.
- **Evidence-based answers**: Do not make assumptions. Base all answers on the code as the ground truth. Every conclusion must be traceable to a specific source file, line number, or runtime observation.
- **Provide rationale**: Include the thinking and reasoning behind every answer. Readers should be able to follow the logical chain from evidence to conclusion.
- **Build and run**: Build and run the source code to analyze behavior as needed. Code analysis alone is insufficient; the user explicitly asks about what you "actually see" at runtime.
- **Output placement**: Place the generated document in the `blitzy/documentation` directory in the destination repo, named `maddy_26452dd8dd78.md` (matching the source branch name).
- **Temporary script cleanup**: Any temporary scripts or test binaries used for observation must be cleaned up afterward. No artifacts from testing should persist outside the output document.

## 0.11 References

### 0.11.1 Repository Files and Folders Searched

The following files and folders were examined during context gathering:

**Root-level files**:
- `go.mod` — Go module definition; confirmed Go 1.13, go-smtp v0.12.1-pre dependency
- `go.sum` — Dependency checksums
- `.mkdocs.yml` — MkDocs documentation configuration (ReadTheDocs theme, nav tree)
- `HACKING.md` — Contributor design guide with architecture overview
- `README.md` — Project overview and feature list
- `maddy.conf` — Default server configuration

**SMTP endpoint source** (`internal/endpoint/smtp/`):
- `smtp.go` — Primary endpoint: Endpoint, Session types, Data(), LMTPData(), prepareBody(), Mail(), Rcpt(), wrapErr(), Init(), setConfig()
- `smtp_test.go` — Test suite: delivery, abort, reset, multi-message, submission auth, check error tests
- `submission.go` — Submission header normalization
- `smtputf8_test.go` — UTF-8 address handling tests
- `submission_test.go` — Submission header validation tests
- `date.go` — Date header parsing helper

**go-smtp library** (module cache, v0.12.1-pre):
- `conn.go` — Connection type, init(), handleConn() main loop, handleData(), handleDataLMTP(), handle() command dispatcher, WriteResponse(), ReadLine(), reset()
- `data.go` — SMTPError type, dataReader, newDataReader() wrapping textproto.DotReader()
- `server.go` — Server type, NewServer() with PIPELINING/8BITMIME/ENHANCEDSTATUSCODES caps, Serve(), handleConn()
- `lengthlimit_reader.go` — lineLimitReader enforcing MaxLineLength
- `backend.go` — Backend and Session interfaces
- `smtp.go` — Constants and capability names

**Go 1.13 standard library**:
- `net/textproto/reader.go` — DotReader(), dotReader struct and Read() method (6-state FSM: stateBeginLine, stateDot, stateDotCR, stateCR, stateData, stateEOF), closeDot()

**Internal support packages**:
- `internal/buffer/buffer.go` — Buffer interface (immutable blob storage)
- `internal/smtpconn/smtpconn.go` — Outbound SMTP client wrapper
- `internal/testutils/smtp_server.go` — In-memory SMTP test harness
- `internal/testutils/target.go` — Mock delivery target

**Documentation**:
- `docs/internals/quirks.md` — Existing SMTP/IMAP quirks (no DATA boundary coverage)
- `docs/internals/sqlite.md` — SQLite backend notes
- `docs/README.md` — Documentation landing page
- `docs/man/README.md` — Man page authoring workflow

**Folders traversed**:
- Root (`""`) — Repository root with all children
- `internal/` — Private implementation tree
- `internal/endpoint/` — Protocol endpoints
- `internal/endpoint/smtp/` — SMTP endpoint implementation
- `internal/smtpconn/` — Outbound SMTP connection
- `internal/testutils/` — Test utilities
- `docs/` — Documentation hub
- `docs/internals/` — Internals documentation

### 0.11.2 Attachments

No attachments were provided by the user for this project.

### 0.11.3 Figma Screens

No Figma screens were provided or referenced for this project.

### 0.11.4 Key Source Citations

| Citation | File | Lines | Relevance |
|---|---|---|---|
| DotReader FSM | Go 1.13 `net/textproto/reader.go` | 283-398 | The 6-state machine that decides when DATA ends; accepts bare LF variants |
| handleData | go-smtp `conn.go` | handleData func | Creates DotReader, calls Session.Data(), drains with io.Copy, writes response |
| newDataReader | go-smtp `data.go` | newDataReader func | Wraps textproto.DotReader() with size limit |
| Server capabilities | go-smtp `server.go` | NewServer func | Advertises PIPELINING, 8BITMIME, ENHANCEDSTATUSCODES |
| Maddy Data handler | `internal/endpoint/smtp/smtp.go` | 312-344 | Session.Data() → prepareBody() → delivery.Body() → delivery.Commit() |
| Maddy prepareBody | `internal/endpoint/smtp/smtp.go` | 283-310 | Reads header, buffers body in memory, adds Received header |
| lineLimitReader | go-smtp `lengthlimit_reader.go` | full file | Enforces MaxLineLength (2000 bytes) on raw reads |
| handleConn loop | go-smtp `conn.go` | handleConn func | Main command loop that reads next line after DATA completes |
| Existing quirks | `docs/internals/quirks.md` | full file | Documents `Received` header and IMAP quirks; no DATA boundary docs |

