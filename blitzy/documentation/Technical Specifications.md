# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification


### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively answers two interrelated security-review questions about the maddy mail server's SMTP handling behavior under edge-case conditions. The document must be grounded entirely in code-level evidence and runtime-observable behavior rather than design intent or specification compliance claims.

- **Category:** Create new documentation
- **Documentation type:** Security analysis / Q&A reference document
- **Output file:** `blitzy/documentation/maddy_26452dd8dd78.md`

The user's requirements decompose into the following discrete documentation objectives:

- **Objective 1 — SMTP Message Boundary Behavior (Dot-Stuffing Edge Case):** Document how maddy (via its `go-smtp` dependency and Go's `net/textproto.DotReader`) handles a message body where a line containing only a period (`.\r\n`) appears mid-stream, before the actual end-of-data terminator. The documentation must explain whether maddy stops reading at the first dot, continues consuming input, or leaves the connection in an unexpected state. It must also describe what runtime evidence (logs, stored messages, queue metadata) reveals which code path was taken.

- **Objective 2 — Authentication State Persistence Across RSET:** Document what happens when a client authenticates as one user on port 587 (submission), begins a MAIL FROM transaction, issues RSET, then immediately issues a new MAIL FROM claiming a different sender identity without re-authenticating. The documentation must explain whether maddy rejects the command, ties the new transaction to the original identity, or permits identity confusion. It must cite the specific code structures (session objects, `ConnState.AuthUser`, `go-smtp` `Conn.reset()`) that determine this behavior.

- **Objective 3 — Behavioral Observation Guidance:** Provide concrete guidance on what runtime artifacts (structured JSON logs, queue `.meta`/`.header`/`.body` files, `Received` headers, SMTP response codes) constitute evidence of each behavior, enabling the reader to observe these behaviors directly on a running instance.

### 0.1.2 Special Instructions and Constraints

- **No source repository modifications:** The user explicitly states that the repository itself should remain unchanged. Any temporary scripts used for observation must be cleaned up afterward.
- **Code-as-truth basis:** All answers must be derived from actual code analysis, not assumptions about intended behavior. The document must cite specific source files, line numbers, and library versions.
- **Runtime observability emphasis:** The user wants to judge from what the running system reveals, not from what the code intends. The document should guide the reader to specific log fields, queue artifacts, and SMTP response codes.
- **Branch-specific output naming:** Per the implementation rule `SWE-AtlasQnA-Repo`, the output file must be named `maddy_26452dd8dd78.md` and placed in `blitzy/documentation/`.
- **Thinking and rationale required:** The document must provide the reasoning behind each answer, not just conclusions.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document the **dot-stuffing message boundary behavior**, we will trace the data flow from the `go-smtp` library's `Conn.handleData()` method through Go's `net/textproto.DotReader()` into maddy's `Session.Data()` and `Session.prepareBody()`, explaining how the `DotReader` treats `.\r\n` as EOF and how the `io.Copy(ioutil.Discard, r)` drain in `handleData` ensures no residual data corrupts the command stream. Source: `go-smtp@v0.12.1 conn.go` lines 512-521, `internal/endpoint/smtp/smtp.go` lines 283-344, Go stdlib `net/textproto/reader.go` `DotReader()`.

- To document the **authentication state persistence across RSET**, we will trace the session lifecycle through `go-smtp`'s `Conn.SetSession()`, `Conn.reset()`, and maddy's `Session.Reset()`, showing that `reset()` preserves the session object (and its `ConnState.AuthUser`), that `handleMail()` only creates a new anonymous session when `c.Session() == nil`, and that RSET therefore cannot change or clear the authenticated identity. Source: `go-smtp@v0.12.1 conn.go` lines 131-133, 263-275, 694-701; `internal/endpoint/smtp/smtp.go` lines 60-81, 162-178, 674-706.

- To document **runtime observation**, we will catalog the structured JSON log fields emitted by maddy's `log.Logger.Msg()` calls (msg_id, sender, username, src_ip), the queue metadata written by `internal/target/queue/queue.go`, and the `Received` header generation in `internal/target/target.go`.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs are surfaced:

- **go-smtp library version specificity:** The exact behavior depends on `github.com/emersion/go-smtp v0.12.1-0.20191206174923-1f576e0ec85c`. The document must note this pinned version because newer versions may have different session lifecycle behavior.
- **Sender domain enforcement on submission:** The default `maddy.conf` rejects non-local sender domains on submission (lines 117-119), which is an additional enforcement layer beyond authentication state. The document should explain this pipeline-level authorization as it bears on the identity-confusion question.
- **DontTraceSender metadata flag:** On submission endpoints, `submissionPrepare()` sets `msgMeta.DontTraceSender = true` (line 28 of `submission.go`), which suppresses source hostname/IP in `Received` headers for authenticated senders. This affects what evidence appears in stored messages.
- **Deferred sender reject behavior:** The `defer_sender_reject` option (default `true`) defers MAIL FROM errors to the RCPT TO phase, which affects the observable timing of rejection when sender domain checks fail after RSET.


## 0.2 Documentation Discovery and Analysis


### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a mature but minimal documentation structure, primarily oriented toward operator guides and man pages rather than security internals. The project uses **MkDocs** for documentation site generation (confirmed by `.mkdocs.yml` at repository root) and has no dedicated security analysis documents.

**Documentation framework:** MkDocs (configuration at `.mkdocs.yml`)

**Existing documentation files discovered:**

| Path | Type | Relevance |
|------|------|-----------|
| `docs/README.md` | Documentation index | Low — site overview only |
| `docs/man/` | Man pages (maddy.1, maddy.5, maddy-filters.5, maddy-targets.5, maddy-auth.5, maddy-storage.5, maddy-tls.5) | Medium — describes configuration directives |
| `docs/internals/quirks.md` | Known behavioral quirks | Medium — documents Received header omission, IMAP sequence number issues |
| `docs/internals/sqlite.md` | SQLite usage notes | Low |
| `docs/tutorials/setting-up.md` | Setup tutorial | Low |
| `docs/tutorials/manual-installation.md` | Manual install guide | Low |
| `docs/tutorials/alias-to-remote.md` | Alias forwarding tutorial | Low |
| `docs/get.sh-script.md` | Installer script docs | Low |
| `HACKING.md` | Developer internals guide | High — describes module architecture, error handling, check/modifier patterns |
| `maddy.conf` | Reference configuration | High — default SMTP/submission/IMAP configuration with security pipeline |

**Key finding:** No existing documentation covers SMTP protocol-level edge-case behavior, session lifecycle, or authentication state management in depth. The `docs/internals/quirks.md` file documents minor protocol deviations but does not address the specific security questions raised. The `HACKING.md` developer guide describes module architecture at a high level but does not trace data flows through the `go-smtp` dependency.

**API documentation tools in use:** None detected. No JSDoc, Godoc annotations, or auto-generated API references.

**Diagram tools detected:** None in existing documentation. The new document will introduce Mermaid diagrams.

### 0.2.2 Repository Code Analysis for Documentation

The following search patterns were used to identify all code components relevant to the security questions:

**SMTP Endpoint (core focus):**
- `internal/endpoint/smtp/smtp.go` — Main SMTP session handler with `Session` struct, `Mail()`, `Data()`, `Reset()`, `prepareBody()`, `Login()`, `AnonymousLogin()`, `newSession()` (721 lines)
- `internal/endpoint/smtp/submission.go` — Submission-mode header preparation with `submissionPrepare()`, sender domain validation
- `internal/endpoint/smtp/date.go` — Date header utilities
- `internal/endpoint/smtp/smtp_test.go` — Integration tests covering RSET, multi-message delivery, abort scenarios
- `internal/endpoint/smtp/submission_test.go` — Header manipulation tests
- `internal/endpoint/smtp/smtputf8_test.go` — UTF-8 address handling tests

**go-smtp library (session lifecycle management):**
- `go-smtp@v0.12.1/conn.go` — Connection handler with `handleData()`, `handleAuth()`, `handleMail()`, `reset()`, `SetSession()` (~700 lines)
- `go-smtp@v0.12.1/data.go` — `dataReader` wrapping `text.DotReader()` with optional size limiting
- `go-smtp@v0.12.1/backend.go` — `Session` interface definition (`Reset`, `Logout`, `Mail`, `Rcpt`, `Data`)
- `go-smtp@v0.12.1/server.go` — Server initialization with SASL PLAIN factory

**Go standard library (dot-encoding):**
- `net/textproto/reader.go` — `DotReader()` implementation: state machine reading dot-encoded blocks, consuming `.\r\n` terminators, returning `io.EOF`

**Module interfaces:**
- `internal/module/msgmetadata.go` — `ConnState` struct with `AuthUser`/`AuthPassword` fields; `MsgMetadata` struct with `Conn *ConnState`
- `internal/module/auth.go` — `AuthProvider` interface with `CheckPlain(username, password) bool`
- `internal/module/module.go` — Module registration interface

**Message pipeline:**
- `internal/msgpipeline/msgpipeline.go` — Pipeline orchestration with source/destination block routing
- `internal/target/queue/queue.go` — Disk-backed delivery queue with `.meta`, `.header`, `.body` file persistence
- `internal/buffer/buffer.go` — Immutable message buffer interface

**Configuration:**
- `maddy.conf` — Default configuration showing port 25 (no auth, `check_source_hostname`, `check_source_mx`, `check_source_rdns`) and port 465 (auth required, DKIM signing)

### 0.2.3 Web Search Research Conducted

- **Go `net/textproto.DotReader` behavior:** Confirmed that `DotReader()` returns `io.EOF` after consuming and discarding the `.\r\n` end-of-sequence line. Lines beginning with a dot are unescaped by removing the leading dot. The reader's `closeDot()` method drains remaining data if the reader is replaced before being fully consumed.

- **RFC 5321 Section 4.5.2 (Transparency):** Confirmed the transparency procedure: sending clients prepend a dot to lines beginning with a dot; receiving servers strip the leading dot from lines beginning with a dot, and treat a line containing only a single dot as the end-of-data marker. This is the canonical specification governing the edge case under review.

- **RFC 821 Section 4.5.2 (original transparency spec):** The original specification established the same dot-stuffing mechanism, which has remained unchanged through RFC 2821 and RFC 5321.


## 0.3 Documentation Scope Analysis


### 0.3.1 Code-to-Documentation Mapping

The documentation must trace two complete code paths through three layers: the `go-smtp` library, maddy's SMTP endpoint, and the message pipeline. Every module below contributes evidence to the security answers.

**Module: go-smtp `Conn` (Session Lifecycle)**
- File: `go-smtp@v0.12.1/conn.go`
- Key functions: `handleData()`, `handleAuth()`, `handleMail()`, `handleRcpt()`, `reset()`, `SetSession()`, `Close()`
- Current documentation: None (third-party library, no inline docs for session state semantics)
- Documentation needed: Trace of `reset()` behavior showing session preservation; `handleData()` drain-and-reset sequence; `handleMail()` anonymous session fallback logic

**Module: go-smtp `dataReader` (Dot-Encoding Layer)**
- File: `go-smtp@v0.12.1/data.go`
- Key functions: `newDataReader()`, `Read()`, size-limiting wrapper
- Current documentation: None specific to security implications
- Documentation needed: Explanation of `DotReader()` delegation and how EOF terminates data consumption

**Module: Go stdlib `net/textproto.DotReader` (RFC 5321 §4.5.2 Implementation)**
- File: `net/textproto/reader.go` (Go standard library)
- Key functions: `DotReader()`, `closeDot()`, `dotReader.Read()`
- Current documentation: Godoc exists but does not address security implications of mid-stream dot lines
- Documentation needed: Security analysis of state machine behavior when `.\r\n` appears mid-message-body

**Module: maddy SMTP Session (`internal/endpoint/smtp/smtp.go`)**
- Key structs: `Session`, `Endpoint`
- Key functions: `Reset()`, `Mail()`, `Data()`, `prepareBody()`, `Login()`, `AnonymousLogin()`, `newSession()`, `startDelivery()`
- Current documentation: None covering edge-case behavior
- Documentation needed: Trace of `connState.AuthUser` binding at session creation and its persistence through `Reset()`; body processing pipeline through `prepareBody()` showing DotReader output consumption

**Module: maddy Submission Preparation (`internal/endpoint/smtp/submission.go`)**
- Key functions: `submissionPrepare()`
- Current documentation: None
- Documentation needed: Explanation of sender domain validation, `DontTraceSender` flag, header synthesis — all relevant to what identity evidence appears in stored messages

**Module: maddy Message Metadata (`internal/module/msgmetadata.go`)**
- Key structs: `ConnState` (AuthUser, AuthPassword, RDNSName), `MsgMetadata` (ID, OriginalFrom, Conn)
- Current documentation: None
- Documentation needed: How `MsgMetadata.Conn` points to the shared `ConnState` and carries authentication identity into the pipeline

**Module: maddy Message Pipeline (`internal/msgpipeline/msgpipeline.go`)**
- Key functions: `Start()`, `RunEarlyChecks()`, source/destination block routing
- Current documentation: HACKING.md provides architectural overview but no per-transaction detail
- Documentation needed: How pipeline checks (source_hostname, source_mx, source_rdns on port 25; authorize_sender on port 587) enforce identity at delivery time

**Module: maddy Queue (`internal/target/queue/queue.go`)**
- Key behavior: Persists messages as `.meta`, `.header`, `.body` files with retry scheduling
- Current documentation: None covering file format or metadata fields
- Documentation needed: What queue artifacts reveal about which identity was trusted

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the following documentation gaps are identified:

- **No security-focused SMTP behavioral documentation exists.** The existing `docs/internals/quirks.md` covers only minor RFC deviations, not edge-case protocol behavior under adversarial conditions.
- **No documentation of go-smtp session lifecycle semantics.** The interaction between `go-smtp`'s `Conn.reset()` and maddy's `Session.Reset()` is not documented anywhere in the repository.
- **No documentation of authentication state persistence.** The fact that `ConnState.AuthUser` is set once in `newSession()` and never modified by `Reset()` is implicit in the code but never explicitly stated.
- **No documentation of the DotReader → prepareBody → buffer pipeline.** The multi-layer data flow from TCP socket through dot-decoding, header parsing, body buffering, and queue persistence is not documented.
- **No documentation of runtime observation artifacts.** Log fields, queue file formats, and Received header generation rules relevant to security review are not documented together in any single reference.

All of these gaps will be addressed by the new documentation file.


## 0.4 Documentation Implementation Design


### 0.4.1 Documentation Structure Planning

The output document `blitzy/documentation/maddy_26452dd8dd78.md` will be organized as a single comprehensive Markdown file with the following structure:

```
blitzy/documentation/
└── maddy_26452dd8dd78.md
    ├── # Maddy SMTP Security Review: Edge-Case Behavior Analysis
    ├── ## 1. Executive Summary
    ├── ## 2. Environment and Versioning Context
    │   ├── ### 2.1 Maddy Server Version and Module
    │   ├── ### 2.2 go-smtp Library Version
    │   └── ### 2.3 Go Standard Library (net/textproto)
    ├── ## 3. Question 1: SMTP Message Boundary Behavior (Dot-Stuffing Edge Case)
    │   ├── ### 3.1 The Scenario
    │   ├── ### 3.2 RFC 5321 §4.5.2 — The Transparency Procedure
    │   ├── ### 3.3 Code Path Trace: DotReader → handleData → prepareBody
    │   ├── ### 3.4 What Happens in Practice
    │   ├── ### 3.5 What Ends Up Stored or Queued
    │   ├── ### 3.6 Runtime Observable Evidence
    │   └── ### 3.7 Security Assessment
    ├── ## 4. Question 2: Authentication State Persistence Across RSET
    │   ├── ### 4.1 The Scenario
    │   ├── ### 4.2 Session Lifecycle: AUTH → Session Creation → ConnState Binding
    │   ├── ### 4.3 RSET Behavior: What Resets and What Persists
    │   ├── ### 4.4 MAIL FROM After RSET: Identity Enforcement
    │   ├── ### 4.5 Pipeline-Level Authorization (Submission Mode)
    │   ├── ### 4.6 What Evidence Shows Which Identity Was Trusted
    │   └── ### 4.7 Security Assessment
    ├── ## 5. Fail-Safe Assessment
    │   ├── ### 5.1 Dot-Stuffing: Does the System Fail Safely?
    │   └── ### 5.2 Authentication State: Does the System Fail Safely?
    ├── ## 6. Observation Playbook
    │   ├── ### 6.1 Test Setup (Temporary, Non-Modifying)
    │   ├── ### 6.2 Observing Dot-Stuffing Behavior
    │   └── ### 6.3 Observing Authentication State Persistence
    └── ## 7. Source File Reference Index
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract session lifecycle semantics from `go-smtp@v0.12.1/conn.go` by tracing the `handleData()`, `handleAuth()`, `handleMail()`, and `reset()` methods with exact line references
- Extract dot-encoding behavior from Go stdlib `net/textproto/reader.go` `DotReader()` implementation and its documented contract
- Extract authentication state binding from `internal/endpoint/smtp/smtp.go` `newSession()` (line 674) and `Reset()` (line 60)
- Generate security conclusions by cross-referencing code behavior against the user's adversarial scenarios
- Extract runtime observation guidance from `internal/target/queue/queue.go` queue file format, log infrastructure, and `Received` header generation in `internal/target/target.go`

**Template Application:** Not applicable — no user template was provided. The document will follow a question-and-answer structure with embedded code-path traces.

**Documentation Standards:**
- Markdown formatting with `#`, `##`, `###` hierarchy
- Mermaid diagrams for data flow and session lifecycle visualization
- Code path references using format: `Source: /path/to/file.go:LineNumber`
- Tables for side-by-side comparison of state before/after RSET
- Source citations for every technical claim

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams will be created:

- **Sequence diagram: DATA command dot-encoding flow** — Shows the message data path from TCP socket through `go-smtp Conn.handleData()` → `net/textproto DotReader()` → `maddy Session.Data()` → `prepareBody()` → `buffer.BufferInMemory()` → `delivery.Body()` → `delivery.Commit()`, with the drain-and-reset step after Session.Data returns

- **Sequence diagram: AUTH → RSET → MAIL FROM identity flow** — Shows a client performing AUTH PLAIN → MAIL FROM user1 → RCPT TO → RSET → MAIL FROM user2, tracing which session object and ConnState.AuthUser are active at each step

- **State diagram: go-smtp Conn session states** — Shows state transitions between no-session, authenticated-session, mid-transaction, and post-reset states, with clear annotation of what `reset()` clears vs. preserves

- **Flowchart: handleMail() decision logic** — Shows the `Session() == nil` check that determines whether a new anonymous session is created or the existing authenticated session is reused


## 0.5 Documentation File Transformation Mapping


### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/maddy_26452dd8dd78.md` | CREATE | `internal/endpoint/smtp/smtp.go`, `internal/endpoint/smtp/submission.go`, `go-smtp@v0.12.1/conn.go`, `go-smtp@v0.12.1/data.go`, `go-smtp@v0.12.1/backend.go`, `net/textproto/reader.go`, `internal/module/msgmetadata.go`, `internal/target/queue/queue.go`, `maddy.conf`, `internal/msgpipeline/msgpipeline.go` | Comprehensive security review document answering both SMTP edge-case questions with full code-path traces, Mermaid diagrams, runtime observation guidance, and fail-safe assessment |

No other documentation files are created, updated, or deleted. The implementation rule `SWE-AtlasQnA-Repo` explicitly prohibits modifying existing files.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/maddy_26452dd8dd78.md
Type: Security Analysis / Q&A Reference Document
Source Code:
    - internal/endpoint/smtp/smtp.go (Session struct, Reset, Mail, Data, prepareBody, Login, newSession)
    - internal/endpoint/smtp/submission.go (submissionPrepare, sender domain validation)
    - go-smtp@v0.12.1/conn.go (handleData, handleAuth, handleMail, reset, SetSession)
    - go-smtp@v0.12.1/data.go (dataReader wrapping textproto.DotReader)
    - go-smtp@v0.12.1/backend.go (Session interface: Reset, Logout, Mail, Rcpt, Data)
    - go-smtp@v0.12.1/server.go (SASL PLAIN factory, server loop)
    - net/textproto/reader.go (DotReader state machine, closeDot drain)
    - internal/module/msgmetadata.go (ConnState with AuthUser/AuthPassword, MsgMetadata)
    - internal/module/auth.go (AuthProvider.CheckPlain interface)
    - internal/target/queue/queue.go (disk-backed queue with .meta/.header/.body files)
    - internal/buffer/buffer.go (immutable buffer interface)
    - internal/msgpipeline/msgpipeline.go (pipeline orchestration, source/dest routing)
    - maddy.conf (default SMTP port 25 / submission port 465 configuration)
    - internal/endpoint/smtp/smtp_test.go (RSET, multi-message, abort tests)
Sections:
    - Executive Summary (concise answers to both questions)
    - Environment and Versioning Context (Go module, go-smtp version, stdlib version)
    - Question 1: SMTP Message Boundary Behavior
        - Scenario description
        - RFC 5321 §4.5.2 reference
        - Code path trace with line-level citations
        - Practical behavior description
        - Stored/queued artifacts analysis
        - Runtime observable evidence catalog
        - Security assessment
    - Question 2: Authentication State Persistence Across RSET
        - Scenario description
        - Session lifecycle trace (AUTH → newSession → ConnState binding)
        - RSET behavior trace (what resets, what persists)
        - MAIL FROM after RSET (identity enforcement via session reuse)
        - Pipeline-level authorization (submission-mode sender domain checks)
        - Identity evidence in headers, queue metadata, enforcement checks
        - Security assessment
    - Fail-Safe Assessment (overall verdict for both scenarios)
    - Observation Playbook (temporary test scripts using openssl/netcat, non-modifying)
    - Source File Reference Index (all files cited with purpose)
Diagrams:
    - Sequence diagram: DATA command dot-encoding flow through all layers
    - Sequence diagram: AUTH → RSET → MAIL FROM identity persistence
    - State diagram: go-smtp Conn session states
    - Flowchart: handleMail() decision logic for session reuse
Key Citations:
    - go-smtp@v0.12.1/conn.go:512-521 (handleData drain-and-reset)
    - go-smtp@v0.12.1/conn.go:694-701 (reset preserves session)
    - go-smtp@v0.12.1/conn.go:263-275 (handleMail anonymous fallback)
    - go-smtp@v0.12.1/conn.go:393-430 (handleAuth session creation)
    - internal/endpoint/smtp/smtp.go:60-81 (Session.Reset and abort)
    - internal/endpoint/smtp/smtp.go:162-178 (Session.Mail)
    - internal/endpoint/smtp/smtp.go:283-344 (Session.Data and prepareBody)
    - internal/endpoint/smtp/smtp.go:643-706 (Login, AnonymousLogin, newSession)
    - internal/endpoint/smtp/submission.go:1-28 (submissionPrepare, DontTraceSender)
    - internal/module/msgmetadata.go:1-50 (ConnState, MsgMetadata)
    - net/textproto/reader.go:334-337 (DotReader factory)
    - net/textproto/reader.go:435-444 (closeDot drain)
    - maddy.conf:1-138 (default SMTP/submission configuration)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The `blitzy/documentation/` directory is a standalone output location per the `SWE-AtlasQnA-Repo` rule. No MkDocs, Docusaurus, or other documentation generator configurations are affected because the output document is not integrated into maddy's existing documentation site.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content/includes:** The new document is self-contained.
- **No navigation links to update:** The document lives in `blitzy/documentation/`, outside the repository's `docs/` tree.
- **No table of contents updates:** The document does not integrate into `.mkdocs.yml`.
- **Internal cross-references within the document:** Section 3 (dot-stuffing) and Section 4 (auth state) will cross-reference each other where the behaviors interact (e.g., post-DATA `reset()` also clearing transaction state but preserving session).


## 0.6 Dependency Inventory


### 0.6.1 Documentation Dependencies

The output document is a standalone Markdown file requiring no documentation build tools. However, the following project dependencies are directly relevant to the security analysis and are cited as authoritative sources throughout the document:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| Go module | `github.com/foxcpp/maddy` | v0.0.0 (pre-1.0, HEAD) | Primary subject of analysis; SMTP endpoint, session handler, message pipeline |
| Go module | `github.com/emersion/go-smtp` | v0.12.1-0.20191206174923-1f576e0ec85c | SMTP server library; manages connection state, command parsing, session lifecycle, DATA handling |
| Go module | `github.com/emersion/go-sasl` | v0.0.0-20191210011802-430746ea8b9b | SASL authentication (PLAIN mechanism) used by go-smtp for AUTH command handling |
| Go module | `github.com/emersion/go-message` | v0.11.2-0.20200422153558-d8abc9f4b81a | RFC 5322 message parsing used by maddy's `prepareBody()` for header extraction |
| Go stdlib | `net/textproto` | (Go 1.13+) | Dot-encoding reader/writer; `DotReader()` implements RFC 5321 §4.5.2 transparency |
| Go stdlib | `bufio` | (Go 1.13+) | Buffered I/O wrapping the dot-decoded stream in `prepareBody()` |
| Go stdlib | `io/ioutil` | (Go 1.13+) | `ioutil.Discard` used by go-smtp to drain unconsumed DATA after `Session.Data()` returns |
| Go minimum | Go runtime | 1.13 | Minimum Go version specified in `go.mod` |

### 0.6.2 Documentation Reference Updates

No documentation link updates are required. The new document is self-contained within `blitzy/documentation/` and does not modify or link into any existing documentation files in the repository. All internal references use relative source paths (e.g., `internal/endpoint/smtp/smtp.go:283`) that point to repository-local files.


## 0.7 Coverage and Quality Targets


### 0.7.1 Documentation Coverage Metrics

The coverage scope is defined by the two security questions. Every code module involved in answering those questions must be traced and cited.

**Question 1 — Dot-Stuffing / Message Boundary Behavior:**

| Code Component | Coverage Target | Coverage Strategy |
|----------------|-----------------|-------------------|
| `net/textproto DotReader()` | 100% of state machine behavior | Trace Godoc contract + `closeDot()` drain semantics |
| `go-smtp data.go newDataReader()` | 100% of wrapper behavior | Document `DotReader()` delegation and size-limiting |
| `go-smtp conn.go handleData()` | 100% of post-DATA sequence | Trace: `Session.Data(r)` → `io.Copy(Discard, r)` → `c.reset()` |
| `maddy smtp.go Data()` | 100% of body processing path | Trace: `prepareBody()` → header read → body buffer → `delivery.Body()` → `Commit()` |
| `maddy smtp.go prepareBody()` | 100% of body consumption | Document `bufio.NewReader` → `textproto.ReadHeader` → `BufferInMemory` |
| `maddy buffer.go BufferInMemory()` | Sufficient for stored-content analysis | Document that buffered content comes from the already-dot-decoded stream |

**Question 2 — Authentication State Persistence:**

| Code Component | Coverage Target | Coverage Strategy |
|----------------|-----------------|-------------------|
| `go-smtp conn.go handleAuth()` | 100% of SASL flow | Trace: PLAIN negotiation → `be.Login()` → `SetSession()` |
| `go-smtp conn.go reset()` | 100% of cleared/preserved state | Document: clears `fromReceived`, `recipients`; calls `session.Reset()`; preserves session object |
| `go-smtp conn.go handleMail()` | 100% of session reuse logic | Document: `Session() == nil` → `AnonymousLogin()` vs. reuse existing |
| `maddy smtp.go newSession()` | 100% of identity binding | Document: `connState.AuthUser = username` set once, never modified |
| `maddy smtp.go Reset()` | 100% of transaction cleanup | Document: calls `abort()` on delivery; clears `mailFrom`, `opts`, `msgMeta`, `delivery`; preserves `connState` |
| `maddy smtp.go Mail()` | 100% of sender validation | Document: `startDelivery()` with `MsgMetadata.Conn = &s.connState` |
| `maddy submission.go submissionPrepare()` | Sufficient for identity evidence | Document sender domain validation and `DontTraceSender` flag |
| `maddy.conf` pipeline checks | Sufficient for enforcement context | Document `check_source_hostname`, `authorize_sender` checks |

**Target coverage:** 100% of code paths directly involved in answering both security questions, with explicit citations for every claim.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every code-level claim must cite a specific file path and line number (or line range)
- Both questions must receive a clear, unambiguous answer with supporting code evidence
- Runtime observation guidance must be actionable (specific commands, expected output patterns)
- The fail-safe assessment must render a definitive judgment for each scenario

**Accuracy validation:**
- All line number citations verified against the actual source files in the repository at `HEAD`
- All go-smtp citations verified against the exact pinned version `v0.12.1-0.20191206174923-1f576e0ec85c`
- Go stdlib `DotReader()` behavior validated against the official Godoc documentation
- RFC 5321 §4.5.2 transparency procedure cited as the authoritative standard

**Clarity standards:**
- The document must be readable by a security reviewer who is not a Go developer, with sufficient context to understand each code path
- Technical terms (DotReader, dot-stuffing, SASL PLAIN, ConnState) must be defined or explained on first use
- Each section must begin with a plain-language summary before diving into code traces
- Diagrams must make the multi-layer data flow visually clear

**Maintainability:**
- All citations use the format `Source: path/to/file.go:LineNumber` for traceability
- The document includes a complete Source File Reference Index as its final section
- Version-specific dependencies are called out explicitly so the document can be validated against future versions

### 0.7.3 Example and Diagram Requirements

- **Minimum diagrams:** 4 Mermaid diagrams (two sequence diagrams, one state diagram, one flowchart) as specified in Section 0.4.3
- **Code path examples:** Each question section includes a condensed pseudo-trace of the relevant code path, not full source listings
- **SMTP session transcripts:** The Observation Playbook section includes example SMTP dialog transcripts showing the exact commands and expected responses for each test scenario
- **Table comparisons:** State-before and state-after tables for RSET showing which fields are cleared vs. preserved


## 0.8 Scope Boundaries


### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/maddy_26452dd8dd78.md` — The sole deliverable

**Source code analyzed (read-only, for citation purposes):**
- `internal/endpoint/smtp/smtp.go` — Session handler, core SMTP logic
- `internal/endpoint/smtp/submission.go` — Submission mode header preparation and sender validation
- `internal/endpoint/smtp/date.go` — Date header utilities
- `internal/endpoint/smtp/smtp_test.go` — Test cases for RSET, multi-message, abort scenarios
- `internal/endpoint/smtp/submission_test.go` — Submission header manipulation tests
- `internal/endpoint/smtp/smtputf8_test.go` — UTF-8 address handling tests
- `internal/module/msgmetadata.go` — ConnState and MsgMetadata struct definitions
- `internal/module/auth.go` — AuthProvider interface
- `internal/module/module.go` — Module registration interface
- `internal/msgpipeline/msgpipeline.go` — Pipeline orchestration
- `internal/target/queue/queue.go` — Disk-backed delivery queue
- `internal/buffer/buffer.go` — Immutable message buffer interface
- `maddy.conf` — Default configuration
- `HACKING.md` — Developer internals guide
- `docs/internals/quirks.md` — Known behavioral quirks
- `go.mod` — Go module definition with dependency versions

**Third-party library code analyzed (read-only):**
- `go-smtp@v0.12.1/conn.go` — Connection handler, session lifecycle
- `go-smtp@v0.12.1/data.go` — DATA reader with DotReader delegation
- `go-smtp@v0.12.1/backend.go` — Session interface definition
- `go-smtp@v0.12.1/server.go` — Server initialization, SASL factory

**Standard library analyzed (read-only):**
- `net/textproto/reader.go` — DotReader implementation

**External references:**
- RFC 5321 Section 4.5.2 (SMTP Transparency)
- Go official `net/textproto` package documentation

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No existing files in the source repository will be modified (per `SWE-AtlasQnA-Repo` rule and user instruction)
- **Test file modifications:** No test files will be created or modified in the repository
- **Existing documentation updates:** No changes to `docs/`, `HACKING.md`, `.mkdocs.yml`, or any other existing documentation
- **Feature additions or code refactoring:** No code changes of any kind
- **Deployment configuration changes:** No changes to `maddy.conf`, systemd units, or Docker configurations
- **IMAP endpoint analysis:** The user's questions are scoped exclusively to SMTP/submission behavior; IMAP internals are not examined
- **LMTP endpoint analysis:** Although maddy registers an `lmtp` endpoint type using the same constructor, LMTP is not part of the user's scenario
- **TLS/STARTTLS negotiation analysis:** Transport-layer encryption is not part of the dot-stuffing or auth-state questions
- **DKIM/SPF/DMARC check internals:** While the pipeline runs these checks, their internal logic is not part of the scoped questions. Only their role as pipeline enforcement points is noted
- **Queue retry and DSN generation logic:** The queue's retry scheduling and bounce handling are not part of the user's questions
- **Remote delivery target internals:** `internal/target/remote/` is not analyzed as the questions concern message acceptance, not outbound delivery
- **Permanent test scripts or infrastructure:** Any observation scripts described in the Playbook section are ephemeral guidance, not repository artifacts


## 0.9 Execution Parameters


### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the output is a standalone Markdown file, not part of a generated documentation site
- **Documentation preview command:** Any Markdown viewer or `cat blitzy/documentation/maddy_26452dd8dd78.md` for terminal review
- **Diagram generation command:** Mermaid diagrams are embedded inline in the Markdown using fenced code blocks (`\`\`\`mermaid ... \`\`\``); they render natively in GitHub, GitLab, and most Markdown previewers
- **Documentation deployment command:** Not applicable — file is committed to the repository via standard git workflow
- **Default format:** Markdown with embedded Mermaid diagrams
- **Citation requirement:** Every section referencing source code must include explicit file path and line number citations in the format `Source: path/to/file.go:LineNumber`
- **Style guide:** Question-and-answer structure with embedded code-path traces, following the repository's existing Markdown conventions (as observed in `HACKING.md` and `docs/internals/quirks.md`)
- **Documentation validation:** Standard Markdown linting; verify all cited line numbers match current HEAD

### 0.9.2 Output File Specification

- **Repository branch:** `maddy_26452dd8dd78`
- **Output directory:** `blitzy/documentation/`
- **Output filename:** `maddy_26452dd8dd78.md` (derived from branch name per `SWE-AtlasQnA-Repo` rule)
- **Full output path:** `blitzy/documentation/maddy_26452dd8dd78.md`
- **File encoding:** UTF-8
- **Line endings:** LF (Unix-style)

### 0.9.3 Repository Preservation Constraints

- **No existing files modified:** The `SWE-AtlasQnA-Repo` rule explicitly states "Do not modify any existing files in the source repository"
- **No temporary artifacts left behind:** The user states "anything temporary should be cleaned up afterward" — the Observation Playbook section in the document will describe temporary test procedures but no scripts will be persisted in the repository
- **Directory creation:** The `blitzy/documentation/` directory will be created if it does not already exist, as a new output location


## 0.10 Rules for Documentation


The following rules are derived directly from the user's requirements and the `SWE-AtlasQnA-Repo` implementation rule:

- **Base all answers on the code as the truth.** Do not make assumptions about intended behavior. Every claim must trace to a specific source file and line range. The document answers "what does the code actually do" rather than "what should the code do."

- **Provide thinking and rationale behind the answers.** Each security conclusion must be preceded by the chain of reasoning that leads to it, citing the code path that produces the behavior.

- **Do not modify any existing files in the source repository.** The only output is the new file `blitzy/documentation/maddy_26452dd8dd78.md`. No source code, test files, configuration files, or existing documentation files are altered.

- **Place the generated document in the `blitzy/documentation` directory.** The filename must match the source branch name: `maddy_26452dd8dd78.md`.

- **Repository must remain unchanged.** Any temporary scripts described in the Observation Playbook are instructional guidance for the reader, not committed artifacts. The repository state after document creation must differ from before only by the addition of the `blitzy/documentation/maddy_26452dd8dd78.md` file (and the `blitzy/documentation/` directory if it did not exist).

- **Rely on what the running system reveals rather than what the code intends.** The document must describe observable behaviors (SMTP response codes, log output, stored message content, queue metadata files) rather than theoretical design goals.

- **Temporary scripts may be used for observation but must be cleaned up afterward.** The Observation Playbook section will provide ephemeral shell commands and `openssl s_client` / raw TCP examples that leave no trace on the system.

- **Pin all analysis to exact dependency versions.** The go-smtp version is `v0.12.1-0.20191206174923-1f576e0ec85c` as specified in `go.mod`. The Go minimum version is 1.13. All behavioral claims are scoped to these versions.

- **Address both the unauthenticated (port 25) and authenticated submission (port 587/465) contexts.** The dot-stuffing question applies to both ports; the authentication state question applies specifically to the submission endpoint where `authAlwaysRequired = true`.


## 0.11 References


### 0.11.1 Repository Files and Folders Searched

The following files and folders were retrieved and analyzed during context gathering to derive the conclusions in this Agent Action Plan:

**Root-level files:**
- `go.mod` — Go module definition confirming `github.com/foxcpp/maddy` module path, Go 1.13 minimum, and `go-smtp v0.12.1-0.20191206174923-1f576e0ec85c` dependency
- `maddy.conf` — Default configuration defining SMTP on port 25 (no auth, anti-relay checks) and submission on port 465 (auth required, DKIM signing)
- `HACKING.md` — Developer internals guide describing module architecture, error handling, check/modifier extension patterns
- `.mkdocs.yml` — MkDocs documentation site configuration

**SMTP endpoint (`internal/endpoint/smtp/`):**
- `internal/endpoint/smtp/smtp.go` (721 lines) — Session struct, Reset(), Mail(), Data(), prepareBody(), Login(), AnonymousLogin(), newSession(), startDelivery(), Endpoint.Init()
- `internal/endpoint/smtp/submission.go` — submissionPrepare(), sender domain validation, DontTraceSender flag
- `internal/endpoint/smtp/date.go` — Date header utilities
- `internal/endpoint/smtp/smtp_test.go` (534 lines) — Tests for delivery, RSET, multi-message, abort, submission auth
- `internal/endpoint/smtp/submission_test.go` — Header manipulation tests
- `internal/endpoint/smtp/smtputf8_test.go` — UTF-8 address handling tests

**Module interfaces (`internal/module/`):**
- `internal/module/msgmetadata.go` — ConnState struct (AuthUser, AuthPassword, RDNSName), MsgMetadata struct (ID, OriginalFrom, Conn)
- `internal/module/auth.go` — AuthProvider interface with CheckPlain()
- `internal/module/module.go` — Module registration interface

**Message pipeline (`internal/msgpipeline/`):**
- `internal/msgpipeline/msgpipeline.go` — Pipeline orchestration with source/destination block routing, RunEarlyChecks(), Start()

**Queue and buffer (`internal/target/queue/`, `internal/buffer/`):**
- `internal/target/queue/queue.go` — Disk-backed delivery queue with .meta/.header/.body file persistence and retry scheduling
- `internal/buffer/buffer.go` — Immutable message buffer interface (MemoryBuffer, FileBuffer)

**Authentication (`internal/auth/`):**
- `internal/auth/` folder — CheckDomainAuth function, external/pam/shadow authentication backends

**Existing documentation (`docs/`):**
- `docs/README.md` — Documentation site index
- `docs/internals/quirks.md` — Known SMTP quirk (Received header omits `for` field), IMAP quirks
- `docs/internals/sqlite.md` — SQLite usage notes
- `docs/tutorials/setting-up.md`, `docs/tutorials/manual-installation.md`, `docs/tutorials/alias-to-remote.md` — Operator tutorials
- `docs/man/` — Man pages for maddy(1), maddy(5), maddy-filters(5), maddy-targets(5), maddy-auth(5), maddy-storage(5), maddy-tls(5)

**Test utilities (`internal/testutils/`):**
- `internal/testutils/` — Mock implementations for checks, modifiers, targets, SMTP servers

**go-smtp library (downloaded module cache):**
- `go-smtp@v0.12.1/conn.go` (~700 lines) — Connection handler: handleData(), handleAuth(), handleMail(), handleRcpt(), reset(), SetSession(), Close(), RSET/QUIT handlers
- `go-smtp@v0.12.1/data.go` — dataReader wrapping textproto.DotReader() with optional size limiting
- `go-smtp@v0.12.1/backend.go` — Backend interface (Login, AnonymousLogin), Session interface (Reset, Logout, Mail, Rcpt, Data), LMTPSession interface
- `go-smtp@v0.12.1/server.go` — Server initialization, SASL PLAIN factory, connection goroutine management

**Root-level folder exploration:**
- Repository root (`""`) — Identified all top-level directories and files
- `internal/` — Full private implementation tree
- `internal/endpoint/` — SMTP, IMAP, and other endpoint registrations
- `internal/endpoint/smtp/` — Six files comprising the SMTP endpoint
- `docs/` — Documentation tree
- `docs/internals/` — Two internal documentation files
- `docs/tutorials/` — Three tutorial files

### 0.11.2 External References

- **RFC 5321 — Simple Mail Transfer Protocol** (October 2008, J. Klensin): Section 4.5.2 (Transparency) defines the dot-stuffing procedure. URL: `https://www.rfc-editor.org/rfc/rfc5321`
- **RFC 821 — Simple Mail Transfer Protocol** (August 1982, J. Postel): Section 4.5.2 (Transparency) — original dot-stuffing specification. URL: `https://www.rfc-editor.org/rfc/rfc821`
- **Go `net/textproto` package documentation**: DotReader() contract specifying EOF behavior at end-of-sequence line. URL: `https://pkg.go.dev/net/textproto`
- **Go `net/textproto/reader.go` source**: DotReader state machine implementation and closeDot() drain logic. URL: `https://go.dev/src/net/textproto/reader.go`

### 0.11.3 Attachments

No attachments were provided for this project. No Figma URLs or external design assets are referenced.


