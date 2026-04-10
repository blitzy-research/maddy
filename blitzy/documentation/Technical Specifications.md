# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new Q&A-style investigative document** that comprehensively answers a series of interrelated questions about the runtime behavior of Maddy's SMTP server when handling DATA boundary conditions, line-ending ambiguity, pipelined multi-message sessions, and proxy-mediated traffic.

**Category:** Create new documentation
**Documentation type:** Investigative technical Q&A / architectural deep-dive

The documentation requirements, restated with enhanced clarity, are:

- **DATA boundary state-machine behavior:** Document how the go-smtp library's `dataReader` state machine (delegated to by Maddy's `Session.Data(io.Reader)`) detects the `<CRLF>.<CRLF>` end-of-data marker, how it handles dot-stuffing/elision, and at what exact moment the server transitions back to command parsing after DATA completes.
- **Line-ending edge cases:** Analyze what happens when a client sends payloads that use non-standard line endings (bare `<LF>`, bare `<CR>`, `<LF>.<LF>`, `<CR>.<CR>`, `<LF>.<CRLF>`) that *almost* look like the end-of-data marker but differ in framing, and whether go-smtp v0.12.1 (the version pinned by Maddy) accepts any of these as a valid DATA terminator.
- **Observable differences across payload variants:** Describe what artifacts surface in logs, delivered message bodies, Received headers, or rejection responses when identical payloads are sent with only the DATA boundary framing changed (correct `<CRLF>.<CRLF>` vs. smuggling-style alternatives).
- **Pipelining and back-to-back messages:** Assess whether boundary detection remains deterministic when multiple messages traverse a single SMTP connection in rapid succession (`TestSMTPDelivery_Multi` pattern), and whether buffering, timing, or connection state could cause boundary detection to "wobble" between transactions.
- **Proxy-mediated traffic:** Reason about how inserting a front proxy that normalizes or rejects ambiguous line endings would change the protocol exchange, error responses, and what Maddy ultimately delivers or rejects.
- **Residual state on failure:** Document what persists (or fails to persist) when things go wrong mid-DATA — semaphore releases, delivery abort paths, message metadata cleanup — and what expected artifacts (log entries, bounce messages, partial deliveries) are notably absent.

### 0.1.2 Special Instructions and Constraints

- **Repository immutability:** The user explicitly states: "the repository itself should remain unchanged, and anything temporary should be cleaned up afterward." This means the document must be created in `blitzy/documentation/` in the destination repository, and no existing files in the source repository may be modified.
- **Implementation rule (SWE-AtlasQnA-Repo):** The output must be a single markdown document named `maddy_26452dd8dd78.md` placed in the `blitzy/documentation/` directory. It must provide thinking/rationale behind each answer, base all conclusions on the code as truth (not assumptions), and must not modify any existing source files.
- **Temporary scripts allowed:** The user permits temporary observation scripts but requires cleanup. Since this is a documentation exercise (not a live-test exercise), the document itself should describe what such scripts *would* do, grounded in code analysis, without requiring actual execution.
- **Tone and depth:** The user is "new to the Maddy repository" and wants to "piece together the real runtime story." This implies a narrative, tutorial-like tone that walks through code paths with specific file/line citations, explains the *why* behind each behavior, and distinguishes between what is observable versus what is inferred.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document **DATA boundary detection**, we will trace the complete code path from go-smtp's `conn.go:handleData()` → `newDataReader()` → `dataReader.Read()` state machine (in `data.go`) → Maddy's `Session.Data(r io.Reader)` → `prepareBody()` → `buffer.BufferInMemory()` → `delivery.Body()` → `delivery.Commit()`, citing the exact state transitions and what triggers EOF on the reader.
- To document **line-ending edge cases**, we will analyze the go-smtp `dataReader` state machine's six states (`stateBeginLine`, `stateDot`, `stateDotCR`, `stateCR`, `stateData`, `stateEOF`) and determine which byte sequences cause transitions to `stateEOF` versus which are passed through as body data, referencing the RFC 5321 §4.5.2 requirements and the library's comment that it is "taken from net/textproto with only one modification to not rewrite CRLF → LF."
- To document **observable differences**, we will catalog the logging statements in `smtp.go` (`s.log.Msg("accepted")`, `s.log.Error("DATA error")`), the Received header generation in `target/received.go`, and the error wrapping in `wrapErr()` to enumerate every artifact that would distinguish a successful delivery from a boundary confusion scenario.
- To document **pipelining behavior**, we will analyze `TestSMTPDelivery_Multi` and the session state cleanup path (`s.delivery = nil`, `s.endp.semaphore.Release()`) to determine whether state leaks are possible between consecutive transactions.
- To document **proxy interaction**, we will reason about what happens when a front proxy strips or normalizes bare CR/LF before the traffic reaches go-smtp's `dataReader`, using the Postfix SMTP smuggling documentation (CVE-2023-51764) as a reference framework.
- To document **failure residuals**, we will trace the `abort()` method (lines 67-81 of `smtp.go`), the `TestSMTPDelivery_AbortData` test, and the `io.Copy(ioutil.Discard, r)` drain in go-smtp's `handleData()` to document what cleanup occurs and what artifacts remain.

### 0.1.4 Inferred Documentation Needs

Based on code analysis:

- The go-smtp library (v0.12.1) is the *sole* component responsible for DATA boundary detection; Maddy itself never inspects raw SMTP wire bytes. This architectural fact must be prominently documented because it is the single most important answer to the user's core question.
- The `dataReader` state machine in go-smtp strictly requires `<CR><LF>.<CR><LF>` (state transitions: `stateBeginLine` → `stateDot` on `.` → `stateDotCR` on `\r` → `stateEOF` on `\n`). Bare-LF variations (`<LF>.<LF>`) do not trigger the terminal state, making Maddy non-vulnerable to the classic SMTP smuggling pattern. This must be stated definitively.
- The memory-buffering behavior (`buffer.BufferInMemory` using `ioutil.ReadAll`) means the entire body is held in RAM. There is no partial-write/flush behavior that could introduce timing-dependent boundary wobble.
- The `SanitizeForHeader()` function in `target/received.go` strips newlines from header values, which is relevant to understanding what residual artifacts survive in delivered messages.
- The LMTP path (`LMTPData`) uses `BodyNonAtomic` for partial delivery, which has different commit semantics than the SMTP path — this needs documentation as a contrast point.


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **mkdocs-based documentation site** with limited coverage of SMTP internals and no existing documentation addressing DATA boundary behavior or line-ending handling.

**Documentation framework:** mkdocs (configuration at `.mkdocs.yml` in the repository root)
**Documentation generator configuration:** `.mkdocs.yml` defines a navigation structure with sections for tutorials, configuration references (man pages), and a small "internals" section.
**API documentation tools:** None detected — there are no godoc generation configs, no Sphinx, no JSDoc equivalents.
**Diagram tools:** None detected in the repository; Mermaid diagrams will be introduced in the new document.
**Documentation hosting/deployment:** Not explicitly configured; mkdocs typically deploys via `mkdocs gh-deploy` or CI pipeline (no CI docs deployment detected in `.github/`).

**Existing documentation files discovered:**

| Path | Type | Relevance |
|------|------|-----------|
| `docs/internals/quirks.md` | Internals reference | **High** — Documents known behavioral quirks (Received header `for` field omission, IMAP `\Recent` behavior). No mention of DATA boundary or line-ending handling. |
| `HACKING.md` | Developer guide | **Medium** — Covers module architecture, error handling via `exterrors`, check/modifier patterns. No SMTP protocol-level discussion. |
| `.mkdocs.yml` | Config | **Medium** — Defines site nav: Tutorials, man pages (maddy.conf, maddy-filters, maddy-targets, maddy-auth, maddy-storage, maddy-tls), and internals (quirks, sqlite). |
| `docs/tutorials/` | Tutorials | **Low** — Setup-oriented guides, not protocol behavior. |
| `docs/man/` | Man pages | **Low** — Configuration reference for endpoints, filters, targets. |

### 0.2.2 Repository Code Analysis for Documentation

The following search patterns were employed to identify code relevant to the documentation task:

**Public APIs handling DATA flow:**
- `internal/endpoint/smtp/smtp.go` — `Session.Data(io.Reader)`, `Session.LMTPData(io.Reader, StatusCollector)`, `Session.prepareBody()`, `Session.abort()`, `Endpoint.wrapErr()`
- `internal/endpoint/smtp/submission.go` — `Session.submissionPrepare()` for header validation/synthesis in submission mode

**Message buffering and body handling:**
- `internal/buffer/buffer.go` — `Buffer` interface definition (`Open()`, `Len()`, `Remove()`)
- `internal/buffer/memory.go` — `MemoryBuffer` and `BufferInMemory()` using `ioutil.ReadAll`

**Received header and traceability:**
- `internal/target/received.go` — `GenerateReceived()` building the Received header, `SanitizeForHeader()` stripping newlines

**Outbound relay (DATA re-transmission):**
- `internal/smtpconn/smtpconn.go` — `C.Data()` sends headers and body to downstream via `cl.Data()` (go-smtp client), `textproto.WriteHeader()`, and `io.Copy()`

**Test infrastructure:**
- `internal/testutils/smtp_server.go` — `SMTPMessage`, `SMTPBackend`, injectable errors (`DataErr`, `MailErr`, `RcptErr`)
- `internal/endpoint/smtp/smtp_test.go` — `TestSMTPDelivery_Multi`, `TestSMTPDelivery_AbortData`, `TestSMTPDelivery_AbortLogout`, `TestSMTPDelivery_Reset`
- `internal/endpoint/smtp/submission_test.go` — `TestSubmissionPrepare` for header validation

**Configuration:**
- `maddy.conf` — Default config with `smtp tcp://0.0.0.0:25` and `submission tls://0.0.0.0:465` endpoints, check pipeline (EHLO, MX, DKIM, SPF), DMARC enforcement

**Key directories examined:**
- `internal/endpoint/smtp/` (6 files: `smtp.go`, `submission.go`, `date.go`, `smtp_test.go`, `smtputf8_test.go`, `submission_test.go`)
- `internal/testutils/` (SMTP server, target, check, DNS mocks)
- `internal/buffer/` (buffer interface, memory buffer, file buffer, bytes reader)
- `internal/smtpconn/` (outbound SMTP client wrapper)
- `internal/target/` (received header generation, queue)
- `docs/` (mkdocs site, internals, tutorials)
- Repository root (`go.mod`, `maddy.conf`, `HACKING.md`, `.mkdocs.yml`)

### 0.2.3 External Library Analysis (go-smtp)

The go-smtp library (`github.com/emersion/go-smtp` v0.12.1) is not vendored in the repository and Go is not installed in the build environment. Analysis was conducted via web search of the library's source code and issue tracker:

- **`data.go` — `dataReader` state machine:** The `Read()` method implements a state machine derived from Go's `net/textproto` with the key modification that it does **not** rewrite `CRLF → LF`. The six states are: `stateBeginLine` (initial), `stateDot`, `stateDotCR`, `stateCR`, `stateData`, `stateEOF`. Only the sequence `<CR><LF>.<CR><LF>` transitions to `stateEOF`. Bare `<LF>` does not trigger line-beginning recognition, so `<LF>.<LF>` and `<LF>.<CRLF>` are not recognized as end-of-data.
- **`conn.go` — `handleData()`:** After `Session.Data(r)` returns, go-smtp drains any remaining data with `io.Copy(ioutil.Discard, r)` to ensure the connection is clean before the next command. This is critical for understanding that even if the session handler returns early (e.g., on error), the connection is not left in a broken DATA state.
- **SMTP smuggling test:** go-smtp v0.20.0 explicitly added an "SMTP smuggling test" (per release notes: "Add SMTP smuggling test") and in v0.20.0 also "Remove DotLF to EOFState case" — confirming that earlier versions may have been more permissive, but by v0.12.1 (Maddy's pinned version), the state machine already required strict `<CR><LF>` framing.
- **Issue #196 (DATA timeout):** Documents the pattern where DATA timeout on a slow client causes leftover data to be interpreted as mangled SMTP commands, leading to a proposed fix to close the connection on `ErrDataTimeout`. This is directly relevant to the user's question about failure residuals.

### 0.2.4 Web Search Research Conducted

- **RFC 5321 §4.5.2 (Data Transparency):** The standard requires that `<CRLF>.<CRLF>` is the sole end-of-data indicator. Section 4.1.1.4 explicitly states that `<LF>.<LF>` MUST NOT be treated as equivalent.
- **SMTP Smuggling (Postfix CVE-2023-51764):** The attack exploits servers that accept `<LF>.<LF>` or `<LF>.<CRLF>` as end-of-data. Postfix responded with `smtpd_forbid_bare_newline` options. This provides the reference framework for the user's proxy question.
- **go-smtp release notes (v0.20.0):** Confirmed removal of "DotLF to EOFState" transition and addition of smuggling tests, reinforcing that the state machine is strict.


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules and components must be analyzed and documented to answer the user's questions comprehensively:

**Module: `internal/endpoint/smtp/smtp.go`**
- Public APIs: `Session.Data()`, `Session.LMTPData()`, `Session.prepareBody()`, `Session.abort()`, `Session.Reset()`, `Session.Logout()`, `Endpoint.wrapErr()`
- Current documentation: **Missing** — no doc file covers the DATA flow, abort semantics, or error mapping
- Documentation needed: Complete walkthrough of DATA reception path, state cleanup on success/failure, error-to-SMTP-response translation, semaphore lifecycle

**Module: `internal/endpoint/smtp/submission.go`**
- Public APIs: `Session.submissionPrepare()`
- Current documentation: **Missing** — no doc file covers header validation/synthesis
- Documentation needed: How submission mode header processing interacts with DATA body parsing; relevant because header parse failure in `textproto.ReadHeader()` can abort DATA processing before body buffering

**Module: `internal/buffer/memory.go`**
- Public APIs: `BufferInMemory()`, `MemoryBuffer`
- Current documentation: **Missing**
- Documentation needed: Explanation that body buffering is fully in-memory (`ioutil.ReadAll`), with no streaming or partial-flush behavior, which eliminates a class of timing-dependent boundary issues

**Module: `internal/target/received.go`**
- Public APIs: `GenerateReceived()`, `SanitizeForHeader()`
- Current documentation: **Missing**
- Documentation needed: How the Received header is constructed and how `SanitizeForHeader` strips newlines, relevant to understanding what artifacts survive in delivered messages

**Module: `internal/smtpconn/smtpconn.go`**
- Public APIs: `C.Data()`, `C.Connect()`, `C.Mail()`, `C.Rcpt()`, `C.Close()`
- Current documentation: **Missing**
- Documentation needed: How Maddy relays messages to downstream servers via go-smtp's client-side dot-encoding, relevant to the proxy question

**Module: `internal/testutils/smtp_server.go`**
- Public APIs: `SMTPBackend`, `SMTPMessage`, test server helpers
- Current documentation: **Missing**
- Documentation needed: How tests verify DATA behavior, injectable error points, and what the test infrastructure reveals about expected behaviors

**External Library: `github.com/emersion/go-smtp` v0.12.1**
- Key components: `dataReader` state machine in `data.go`, `handleData()` in `conn.go`, `newDataReader()`, drain pattern
- Current documentation: Library's own godoc; no Maddy-specific documentation of the boundary contract
- Documentation needed: Complete state-machine analysis with state transition table, identification of which byte sequences terminate DATA vs. pass through as body content

**Configuration: `maddy.conf`**
- Endpoints: `smtp tcp://0.0.0.0:25`, `submission tls://0.0.0.0:465`
- Current documentation: Man pages in `docs/man/`
- Documentation needed: Brief context on how the check pipeline (EHLO, MX, DKIM, SPF) runs *before* DATA, not during DATA boundary parsing

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No existing document describes the DATA reception lifecycle** — from wire bytes arriving at go-smtp through boundary detection, dot-stripping, header parsing, body buffering, Received header generation, pipeline delivery, to commit/abort.
- **No existing document addresses line-ending handling** — the `quirks.md` file documents three known quirks (Received `for` field, IMAP `\Recent`, IMAP sequence numbers) but says nothing about CRLF handling, DATA boundary, or SMTP smuggling.
- **No existing document covers the go-smtp `dataReader` contract** — the fact that Maddy delegates all boundary detection to go-smtp and receives a pre-decoded `io.Reader` is not documented anywhere.
- **No existing document explains pipelining behavior** — how session state is managed across multiple transactions on a single connection.
- **No existing document discusses error/abort semantics** — what happens when DATA is interrupted (connection close, timeout, pipeline error), what state is cleaned up, and what artifacts remain.
- **No existing document addresses proxy interaction** — how a front proxy that normalizes line endings would affect Maddy's behavior.

All of these gaps will be addressed by the new `maddy_26452dd8dd78.md` document.


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The new document will be structured as a comprehensive Q&A investigation, organized to follow the user's questions in order while building understanding progressively:

```
blitzy/documentation/
└── maddy_26452dd8dd78.md
    ├── Introduction (context, scope, methodology)
    ├── Q1: DATA Boundary Detection — How Does the Server Decide the Message Is Finished?
    │   ├── Architectural separation: go-smtp vs. Maddy
    │   ├── go-smtp dataReader state machine analysis
    │   ├── State transition table
    │   ├── Maddy's Session.Data() reception path
    │   └── The moment of transition: DATA → command parsing
    ├── Q2: Line-Ending Edge Cases — What Happens with Non-Standard Framing?
    │   ├── Correct termination: CRLF.CRLF
    │   ├── Bare-LF variations: LF.LF, LF.CRLF
    │   ├── Bare-CR variations: CR.CR
    │   ├── Mixed: CRLF.LF, CR.CRLF
    │   └── State-machine trace for each variant
    ├── Q3: Observable Differences — What Shows Up in Logs and Delivered Messages?
    │   ├── Successful delivery artifacts (logs, Received header, body)
    │   ├── Failed/ambiguous delivery artifacts (error logs, SMTP responses)
    │   └── What you expect to see but never do
    ├── Q4: Pipelining — Does Boundary Detection Wobble Under Pressure?
    │   ├── Multi-message session state management
    │   ├── TestSMTPDelivery_Multi analysis
    │   ├── Semaphore and delivery lifecycle
    │   └── Buffering consistency (ioutil.ReadAll atomicity)
    ├── Q5: Proxy-Mediated Traffic — How Does a Front Proxy Change the Story?
    │   ├── Proxy normalization scenarios
    │   ├── Protocol exchange changes
    │   ├── Error surface changes
    │   └── SMTP smuggling context (CVE-2023-51764)
    ├── Q6: Failure Residuals — What Lingers When Something Goes Wrong?
    │   ├── Abort path analysis (Session.abort())
    │   ├── go-smtp drain pattern
    │   ├── TestSMTPDelivery_AbortData analysis
    │   ├── Timeout scenarios (go-smtp Issue #196)
    │   └── What you expect to see but never do
    └── Conclusion (summary of findings, architectural strengths/gaps)
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract the DATA flow from `internal/endpoint/smtp/smtp.go` lines 283-344 (prepareBody and Data methods), tracing each function call with file:line citations
- Extract the state machine from go-smtp `data.go` via web search results, documenting all six states and their transitions
- Extract test behavior from `internal/endpoint/smtp/smtp_test.go` lines 322-396 (Multi, AbortData tests) to ground assertions in tested behavior
- Extract error handling from `smtp.go` lines 389-455 (wrapErr) and connect to SMTP response codes
- Extract Received header construction from `internal/target/received.go` lines 19-87 to document observable artifacts
- Extract outbound relay behavior from `internal/smtpconn/smtpconn.go` lines 298-324 to document the proxy/relay story

**Documentation Standards:**
- Markdown formatting with `#`, `##`, `###` heading hierarchy
- Mermaid diagrams for the DATA flow sequence and state machine transitions
- Code citations in the format `Source: internal/endpoint/smtp/smtp.go:312`
- Tables for state transition matrices and error-to-response mappings
- Consistent terminology: "DATA boundary," "end-of-data marker," "dot-stuffing," "bare LF"

### 0.4.3 Diagram and Visual Strategy

The document will include the following Mermaid diagrams:

- **Sequence diagram:** Complete DATA flow from TCP socket → go-smtp `dataReader` → Maddy `Session.Data()` → `prepareBody()` → `BufferInMemory()` → `delivery.Body()` → `delivery.Commit()` → SMTP 250 response
- **State machine diagram:** The six-state `dataReader` FSM with labeled transitions for each byte class (`\r`, `\n`, `.`, other)
- **Flow chart:** The abort/cleanup decision tree showing paths for connection close, timeout, and pipeline error
- **Comparison table diagram:** Side-by-side showing what happens for `<CRLF>.<CRLF>` vs. `<LF>.<LF>` vs. `<LF>.<CRLF>` through the state machine


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/maddy_26452dd8dd78.md` | CREATE | `internal/endpoint/smtp/smtp.go`, `internal/endpoint/smtp/smtp_test.go`, `internal/endpoint/smtp/submission.go`, `internal/endpoint/smtp/date.go`, `internal/endpoint/smtp/submission_test.go`, `internal/buffer/memory.go`, `internal/buffer/buffer.go`, `internal/target/received.go`, `internal/smtpconn/smtpconn.go`, `internal/testutils/smtp_server.go`, `docs/internals/quirks.md`, `HACKING.md`, `maddy.conf`, `go.mod`, go-smtp `data.go` (external) | Comprehensive Q&A document answering all user questions about SMTP DATA boundary handling, line-ending edge cases, pipelining behavior, proxy interaction, and failure residuals. Full Mermaid diagrams, state transition tables, code-path traces with file:line citations, and test-grounded behavioral assertions. |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/maddy_26452dd8dd78.md
Type: Investigative Q&A / Architectural Deep-Dive
Source Code:
  - internal/endpoint/smtp/smtp.go (lines 67-81, 283-344, 355-387, 389-455)
  - internal/endpoint/smtp/smtp_test.go (lines 25-28, 90-120, 122-158, 322-396, 429-462)
  - internal/endpoint/smtp/submission.go (lines 1-131)
  - internal/endpoint/smtp/submission_test.go (lines 1-147)
  - internal/endpoint/smtp/date.go (lines 1-48)
  - internal/buffer/memory.go (lines 1-33)
  - internal/buffer/buffer.go (lines 1-42)
  - internal/target/received.go (lines 1-87)
  - internal/smtpconn/smtpconn.go (lines 1-339)
  - internal/testutils/smtp_server.go (lines 1-388)
  - docs/internals/quirks.md (lines 1-31)
  - HACKING.md (developer guide)
  - maddy.conf (lines 1-153)
  - go.mod (dependency: go-smtp v0.12.1)
  - External: go-smtp data.go (dataReader state machine, via web search)
  - External: go-smtp conn.go (handleData, drain pattern, via web search/issue #196)
Sections:
  - Introduction (purpose, methodology, codebase context)
  - Q1: DATA Boundary Detection (go-smtp state machine, Maddy's Session.Data path)
  - Q2: Line-Ending Edge Cases (state traces for each variant)
  - Q3: Observable Differences (logs, Received headers, errors)
  - Q4: Pipelining Behavior (multi-message sessions, state cleanup)
  - Q5: Proxy-Mediated Traffic (normalization, SMTP smuggling context)
  - Q6: Failure Residuals (abort paths, timeouts, drain pattern)
  - Conclusion (findings summary, architectural assessment)
Diagrams:
  - Mermaid sequence diagram: DATA reception lifecycle
  - Mermaid state diagram: dataReader FSM with all six states
  - Mermaid flowchart: abort/cleanup decision tree
Key Citations:
  - internal/endpoint/smtp/smtp.go:312 (Session.Data entry point)
  - internal/endpoint/smtp/smtp.go:283 (prepareBody)
  - internal/endpoint/smtp/smtp.go:67 (abort method)
  - internal/endpoint/smtp/smtp.go:336 (comment: "go-smtp will call Reset")
  - internal/endpoint/smtp/smtp_test.go:360 (TestSMTPDelivery_AbortData)
  - internal/endpoint/smtp/smtp_test.go:322 (TestSMTPDelivery_Multi)
  - internal/buffer/memory.go:27 (BufferInMemory)
  - internal/target/received.go:19 (GenerateReceived)
  - internal/target/received.go:15 (SanitizeForHeader)
  - internal/smtpconn/smtpconn.go:303 (outbound C.Data)
  - go.mod: go-smtp v0.12.1 dependency
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The new file is placed in `blitzy/documentation/` which is a standalone output directory, not part of the mkdocs site navigation. The existing `.mkdocs.yml` and `docs/` structure remain untouched per the repository immutability constraint.


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No external documentation tooling is required for this task. The output is a single standalone Markdown file that does not require compilation, rendering through mkdocs, or any documentation generator. Mermaid diagrams embedded in the document are written as fenced code blocks and can be rendered by any Mermaid-compatible Markdown viewer (GitHub, GitLab, VS Code, etc.).

The following project dependencies are relevant to understanding and documenting the SMTP DATA behavior:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| Go modules | `github.com/emersion/go-smtp` | v0.12.1-0.20191206174923-1f576e0ec85c | Core SMTP protocol library; owns DATA boundary detection via `dataReader` state machine |
| Go modules | `github.com/emersion/go-message` | v0.10.9 | Message parsing; `textproto.ReadHeader()` used in `prepareBody()` to parse message headers from the body stream |
| Go modules | `github.com/emersion/go-sasl` | v0.0.0-20191210011802-430746ea8b9b | SASL authentication; used in test helpers (`sasl.NewPlainClient`) |
| Go modules | `github.com/foxcpp/go-mockdns` | v0.0.0-20191226172053-0d87d6003a12 | DNS mocking in tests; provides mock resolver for EHLO/rDNS validation |
| Go runtime | Go | 1.13 | Minimum Go version specified in `go.mod` |

### 0.6.2 Documentation Reference Updates

Not applicable — no existing documentation files require link updates. The new document is self-contained and does not cross-reference other documentation pages.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis:**

- SMTP DATA boundary behavior documented: **0%** — no existing documentation addresses this topic
- go-smtp `dataReader` state machine documented (in Maddy context): **0%** — the library contract is completely undocumented
- Line-ending edge case behavior documented: **0%** — `quirks.md` covers three unrelated quirks
- Pipelining/multi-message session behavior documented: **0%** — only test code serves as implicit documentation
- Error/abort path behavior documented: **0%** — `HACKING.md` discusses `exterrors` patterns but not SMTP-specific abort semantics
- Proxy interaction documented: **0%** — no existing documentation

**Target coverage after this document:** 100% of the user's six question areas, grounded in specific code citations.

**Coverage gaps to address:**

| Topic | Current | Target | Key Sources |
|-------|---------|--------|-------------|
| DATA boundary detection mechanism | 0% | 100% | go-smtp `data.go` state machine, `smtp.go:312` |
| Line-ending variant handling | 0% | 100% | go-smtp state transitions, RFC 5321 §4.5.2 |
| Observable artifacts (logs, headers, errors) | 0% | 100% | `smtp.go:334`, `received.go:19`, `smtp.go:389` |
| Multi-message session state management | 0% | 100% | `smtp_test.go:322`, `smtp.go:336-341` |
| Proxy/relay interaction | 0% | 100% | `smtpconn.go:303`, CVE-2023-51764 context |
| Failure residuals and abort paths | 0% | 100% | `smtp.go:67-81`, `smtp_test.go:360`, go-smtp drain pattern |

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every user question is addressed with a dedicated section containing rationale and code citations
- All code paths are traced with specific file:line references (e.g., `internal/endpoint/smtp/smtp.go:312`)
- The go-smtp state machine is fully enumerated with all six states and all transition conditions
- Each line-ending variant (`<CRLF>.<CRLF>`, `<LF>.<LF>`, `<LF>.<CRLF>`, `<CR>.<CR>`, etc.) is traced through the state machine with a definitive outcome (terminates DATA vs. passes through as body)
- Test-grounded assertions: every behavioral claim is supported by either a test case citation or a direct code-path trace

**Accuracy validation:**
- State machine analysis is cross-referenced between the web search results of go-smtp `data.go` and the behavior observed in Maddy's test suite (e.g., `TestSMTPDelivery_AbortData` confirms that incomplete DATA results in zero delivered messages)
- RFC 5321 requirements are cited to establish the compliance baseline
- go-smtp version is pinned to v0.12.1 (from `go.mod`) — all analysis applies to this specific version

**Clarity standards:**
- Progressive disclosure: start with the architectural separation (go-smtp owns boundary detection, Maddy receives decoded stream), then drill into state machine details, then address each edge case
- Technical accuracy with accessible narrative tone (appropriate for someone "new to the Maddy repository")
- Consistent terminology: "end-of-data marker" (not "termination sequence"), "bare LF" (not "naked newline"), "dot-stuffing" (not "dot-encoding")

**Maintainability:**
- All assertions cite specific source code lines so they can be re-verified when code changes
- External library behavior is clearly marked as version-specific (go-smtp v0.12.1)

### 0.7.3 Example and Diagram Requirements

- Minimum 1 Mermaid diagram per major section (DATA flow sequence, state machine, abort flowchart)
- State transition table for all six `dataReader` states × all input byte classes
- At least one annotated byte-sequence trace per line-ending variant
- Error-to-SMTP-response mapping table


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/maddy_26452dd8dd78.md` — the sole deliverable

**Source code analyzed for documentation (read-only, no modifications):**
- `internal/endpoint/smtp/smtp.go` — DATA reception, abort, error wrapping
- `internal/endpoint/smtp/smtp_test.go` — Multi-message, abort, reset tests
- `internal/endpoint/smtp/submission.go` — Submission mode header preparation
- `internal/endpoint/smtp/submission_test.go` — Submission header validation tests
- `internal/endpoint/smtp/date.go` — Date parsing for submission mode
- `internal/buffer/buffer.go` — Buffer interface
- `internal/buffer/memory.go` — In-memory buffering implementation
- `internal/target/received.go` — Received header generation and sanitization
- `internal/smtpconn/smtpconn.go` — Outbound SMTP client wrapper (relay behavior)
- `internal/testutils/smtp_server.go` — Test SMTP backend and server helpers
- `docs/internals/quirks.md` — Existing quirks documentation
- `HACKING.md` — Developer architecture guide
- `maddy.conf` — Default configuration
- `go.mod` — Dependency versions (go-smtp v0.12.1)
- `.mkdocs.yml` — Documentation site configuration

**External references analyzed:**
- go-smtp `data.go` source (dataReader state machine, via web search)
- go-smtp `conn.go` handleData/drain pattern (via web search and issue #196)
- go-smtp release notes (v0.20.0 smuggling test, DotLF removal)
- RFC 5321 §2.3.8 (line termination), §4.5.2 (data transparency), §4.1.1.4 (DATA command)
- SMTP smuggling (Postfix CVE-2023-51764, Cisco ESA response)

**Topics covered in the document:**
- go-smtp `dataReader` six-state FSM and its `<CRLF>.<CRLF>`-only termination
- Maddy's `Session.Data()` → `prepareBody()` → `BufferInMemory()` → `delivery.Body()` → `Commit()` pipeline
- Line-ending variant behavior for: `<CRLF>.<CRLF>`, `<LF>.<LF>`, `<LF>.<CRLF>`, `<CR>.<CR>`, `<CRLF>.<LF>`, `<CR>.<CRLF>`
- Log output, Received header content, SMTP response codes for success/failure
- Multi-message session state lifecycle (cleanup after each transaction)
- Proxy normalization effects on DATA boundary detection
- Abort/cleanup on connection close, timeout, pipeline error
- Memory buffering atomicity and its effect on timing-dependent behavior

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — no files in the Maddy repository are to be modified; the user explicitly requires the repository to remain unchanged
- **Test file modifications** — existing tests are analyzed but not altered
- **Feature additions or code refactoring** — this is purely a documentation exercise
- **IMAP, SQL storage, or other non-SMTP components** — the document focuses exclusively on SMTP/LMTP DATA handling
- **TLS/STARTTLS negotiation details** — while the SMTP endpoint supports TLS, the user's questions are about DATA boundary behavior which occurs after TLS is established
- **Authentication mechanisms** — SASL/PLAIN auth is used in tests but is not part of the DATA boundary story
- **DKIM, SPF, DMARC check pipeline** — these run before DATA and do not affect boundary detection
- **Queue management and delivery retry logic** (`internal/target/queue/`) — the user's questions focus on the moment of DATA acceptance, not downstream queue behavior
- **Deployment configuration changes** — no infrastructure changes
- **mkdocs site updates** — the new document lives in `blitzy/documentation/`, not in the mkdocs `docs/` tree
- **Running Maddy in a live environment** — the document is a code-analysis-grounded investigation, not a live-test report; Go is not installed and the binary cannot be built
- **go-smtp versions other than v0.12.1** — all analysis is version-specific to Maddy's pinned dependency


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the output is a standalone Markdown file, not part of a documentation build pipeline.
- **Documentation preview command:** Any Mermaid-compatible Markdown viewer (GitHub web UI, VS Code with Mermaid extension, `grip` for local preview).
- **Diagram generation command:** Mermaid diagrams are embedded inline as fenced code blocks; no external generation step is required. For offline rendering: `npx @mermaid-js/mermaid-cli -i maddy_26452dd8dd78.md -o output.pdf` (optional).
- **Documentation deployment command:** Not applicable — the file is committed to `blitzy/documentation/` and does not require separate deployment.
- **Default format:** Markdown with embedded Mermaid diagrams.
- **Citation requirement:** Every behavioral assertion must reference a specific source file and line number in the format `Source: path/to/file.go:LineNumber`.
- **Style guide:** Narrative Q&A format with progressive disclosure (architectural overview → detailed analysis → edge cases). Tone should be accessible to someone new to the repository, with technical precision.
- **Documentation validation:** Manual review; no automated linting or link checking is configured for `blitzy/documentation/`.

### 0.9.2 Environment Constraints

- **Go runtime:** Not installed; the Maddy binary cannot be compiled or executed. All analysis is based on static code reading.
- **go-smtp library:** Not vendored; source code obtained via web search of the public repository.
- **Test execution:** Not possible without Go runtime; test behavior is inferred from reading test source code.
- **Repository immutability:** The source repository at `/tmp/blitzy/maddy/maddy_26452dd8dd78_76465c/` must remain unchanged. The output file is written to `blitzy/documentation/` in the destination repository.


## 0.10 Rules for Documentation

The following rules are derived from the user's explicit instructions and the implementation rules configured for this project:

- **Do not modify any existing files in the source repository.** The repository must remain unchanged; the output is solely the new `maddy_26452dd8dd78.md` file in `blitzy/documentation/`.
- **Base all answers on the code as truth.** Do not make assumptions; every assertion must be traceable to a specific source file, line number, or test case. Where the code is ambiguous or unavailable (e.g., go-smtp internals not locally accessible), state the basis of the conclusion (web search of public source, issue tracker, release notes) and the confidence level.
- **Provide thinking and rationale behind the answers.** Each answer must explain *why* the behavior occurs, not just *what* the behavior is. Trace the code path, explain the state machine transitions, and connect cause to effect.
- **Temporary scripts may be described but not committed.** The user permits temporary observation scripts; however, since Go is not installed and the binary cannot be built, the document should describe what such scripts *would* do and what they would observe, grounded in code analysis. Any hypothetical scripts described in the document must be clearly labeled as such.
- **Clean up anything temporary.** No files should be left in the source repository after document generation. The only persistent output is the documentation file in the destination repository.
- **Name the output document `maddy_26452dd8dd78.md`** and place it in the `blitzy/documentation/` directory, as specified by the SWE-AtlasQnA-Repo implementation rule.
- **Use Mermaid diagrams for all significant workflows** including the DATA reception sequence, the `dataReader` state machine, and the abort/cleanup decision tree.
- **Cite source code for all technical details** using the format `Source: path/to/file.go:LineNumber`.
- **Pin all analysis to go-smtp v0.12.1** as specified in `go.mod`. Note where newer versions have different behavior (e.g., v0.20.0 removed DotLF-to-EOF transition) but do not present newer-version behavior as applicable to Maddy's current build.
- **Address all six user questions comprehensively** — do not skip any question or defer it as "future work."


## 0.11 References

### 0.11.1 Repository Files and Folders Searched

The following files and folders were systematically retrieved and analyzed to derive the conclusions in this Agent Action Plan:

**Core SMTP Endpoint (primary sources):**

| File Path | Purpose | Key Findings |
|-----------|---------|--------------|
| `internal/endpoint/smtp/smtp.go` | SMTP endpoint: session management, DATA handling, error wrapping | `Session.Data()` at line 312 receives `io.Reader` from go-smtp; `prepareBody()` at line 283 parses headers and buffers body; `abort()` at line 67 handles cleanup; `wrapErr()` at line 389 translates errors to SMTP responses |
| `internal/endpoint/smtp/smtp_test.go` | SMTP endpoint tests | `TestSMTPDelivery_Multi` (line 322) validates two messages on one connection; `TestSMTPDelivery_AbortData` (line 360) confirms incomplete DATA yields zero messages; `TestSMTPDelivery_Reset` (line 429) validates state cleanup via RSET |
| `internal/endpoint/smtp/submission.go` | Submission mode header validation | `submissionPrepare()` validates From, Sender, To, Cc, Bcc, Reply-To, Date, Message-ID headers; enforces RFC 5322 §3.6.2 |
| `internal/endpoint/smtp/submission_test.go` | Submission header validation tests | Tests for malformed headers, missing From, multiple From without Sender, date parsing, Message-ID generation |
| `internal/endpoint/smtp/date.go` | Date header parsing | `parseMessageDateTime()` tries 16 RFC 5322 §3.3 layouts; strips trailing CFWS comments |

**Buffer and Body Handling:**

| File Path | Purpose | Key Findings |
|-----------|---------|--------------|
| `internal/buffer/buffer.go` | Buffer interface definition | Defines `Open()`, `Len()`, `Remove()` — immutable blob storage contract |
| `internal/buffer/memory.go` | In-memory buffer implementation | `BufferInMemory()` uses `ioutil.ReadAll()` — atomic, no streaming, no partial flush |

**Received Header and Tracing:**

| File Path | Purpose | Key Findings |
|-----------|---------|--------------|
| `internal/target/received.go` | Received header generation | `GenerateReceived()` builds header with from/by/with/id fields; `SanitizeForHeader()` strips newlines; `DontTraceSender` suppresses sender info in submission mode |

**Outbound Relay:**

| File Path | Purpose | Key Findings |
|-----------|---------|--------------|
| `internal/smtpconn/smtpconn.go` | Outbound SMTP client wrapper | `C.Data()` at line 303 sends headers via `textproto.WriteHeader()` and body via `io.Copy()` through go-smtp client's dot-encoding writer |

**Test Infrastructure:**

| File Path | Purpose | Key Findings |
|-----------|---------|--------------|
| `internal/testutils/smtp_server.go` | Test SMTP backend | `SMTPMessage` captures From, To, Data, State; `SMTPBackend` has injectable `DataErr`, `MailErr`, `RcptErr`; `session.Data()` uses `ioutil.ReadAll(r)` |

**Documentation and Configuration:**

| File Path | Purpose | Key Findings |
|-----------|---------|--------------|
| `docs/internals/quirks.md` | Known behavioral quirks | Three documented quirks (Received `for` field omission, IMAP `\Recent`, IMAP sequence numbers); no SMTP boundary or line-ending content |
| `HACKING.md` | Developer guide | Module architecture, `exterrors` error handling, check/modifier patterns |
| `maddy.conf` | Default configuration | `smtp tcp://0.0.0.0:25` with EHLO/MX/DKIM/SPF checks; `submission tls://0.0.0.0:465` with auth |
| `.mkdocs.yml` | Documentation site config | mkdocs nav structure with tutorials, man pages, internals |
| `go.mod` | Go module dependencies | Go 1.13; go-smtp v0.12.1-0.20191206174923-1f576e0ec85c; go-message v0.10.9 |

**Folders explored:**

| Folder Path | Depth | Relevance |
|-------------|-------|-----------|
| `` (root) | 0 | Project structure overview |
| `internal/` | 1 | All internal packages |
| `internal/endpoint/smtp/` | 2 | Primary SMTP endpoint — 6 files |
| `internal/testutils/` | 2 | Test infrastructure |
| `internal/buffer/` | 2 | Body buffering |
| `internal/target/` | 2 | Received header, queue |
| `internal/smtpconn/` | 2 | Outbound SMTP relay |
| `docs/` | 1 | Documentation site |
| `docs/internals/` | 2 | Quirks, sqlite docs |

### 0.11.2 External Sources

| Source | URL / Identifier | Summary |
|--------|-----------------|---------|
| go-smtp `data.go` (master branch) | `github.com/emersion/go-smtp/blob/master/data.go` | `dataReader` state machine with 6 states; comment: "taken from net/textproto with only one modification to not rewrite CRLF → LF"; only `<CR><LF>.<CR><LF>` triggers `stateEOF` |
| go-smtp Issue #196 | `github.com/emersion/go-smtp/issues/196` | DATA timeout causes leftover data to be interpreted as mangled SMTP commands; proposed fix: close connection on `ErrDataTimeout`; shows `handleData()` drain pattern |
| go-smtp Releases | `github.com/emersion/go-smtp/releases` | v0.20.0: "Remove DotLF to EOFState case", "Add SMTP smuggling test"; v0.20.2: "Parse DATA\\r\\n\\r\\n.\\r\\n as \\r\\n\\r\\n message" |
| go-smtp pkg.go.dev | `pkg.go.dev/github.com/emersion/go-smtp` | API reference: `Session.Data(r io.Reader)` interface, `Client.Data()` returns `io.WriteCloser` |
| RFC 5321 | `rfc-editor.org/rfc/rfc5321.html` | §2.3.8: Lines must be CRLF-terminated; §4.5.2: `<CRLF>.<CRLF>` is end-of-data; `<LF>.<LF>` MUST NOT be treated as equivalent |
| Postfix SMTP Smuggling | `postfix.org/smtp-smuggling.html` | CVE-2023-51764: attack exploits `<LF>.<LF>` and `<LF>.<CRLF>` as non-standard end-of-data; Postfix added `smtpd_forbid_bare_newline` options |
| Postfix False Smuggling Claims | `postfix.org/false-smuggling-claims.html` | Dot-stuffing by compliant MTAs prevents `<CR><LF>..<non-CRLF>` from being confused with end-of-data; false positives from non-viable attack patterns |

### 0.11.3 Attachments

No attachments were provided by the user for this project.


