# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification


### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that captures the runtime behavior of the Maddy mail server as observed through actual test execution and source-code tracing. The user is onboarding to the codebase and requires an authoritative reference that bridges the gap between source code structure and observable runtime output.

**Documentation Category:** Create new documentation

**Documentation Type:** Technical onboarding guide / Runtime behavior reference

The user's requirements decompose into the following specific documentation objectives:

- **SMTP Endpoint Log Taxonomy** — Document the complete log line format produced by `internal/endpoint/smtp/` during successful delivery versus aborted delivery, including all JSON fields, the module name prefix, and the `msg_id` format
- **Queue Delivery Trace** — Trace a message through `internal/target/queue/` from initial acceptance through retry scheduling, quoting exact log lines for delivery attempt, failure, and retry scheduling including the JSON field name that contains the retry delay value
- **Remote Delivery MX Authentication Failures** — Document the exact error message string when MX authenticity checks fail in `internal/target/remote/`, the precise SMTP enhanced status code in X.Y.Z format, and the complete reply text
- **TLS Fallback Behavior** — Quote the exact log message emitted when the system falls back from TLS to plaintext in `internal/target/remote/connect.go`, including all JSON fields
- **Retry Scheduling JSON Fields** — Identify the exact JSON field name that contains the retry delay value in queue retry log output

### 0.1.2 Special Instructions and Constraints

The user has specified the following critical directives that must be strictly observed:

- **No source repository modifications:** "Don't modify repository files; temporary scripts are fine but clean them up afterward." — No files in the maddy source tree may be modified. Temporary helper scripts used during test execution must be removed after use.
- **Implementation Rule — Single Output File:** The user-specified implementation rule states: "Use the blitzy-research/AtlasQnA repository as the destination. Write your complete answer as a single markdown file named `<project_name>.md`. Do not modify any files in the source repository." This means all documentation must be consolidated into a single `maddy.md` file written to the `blitzy-research/AtlasQnA` repository.
- **Evidence-Based Documentation:** All log lines, field names, error messages, and status codes must be quoted from actual test execution output, not inferred from source code alone. The user demands observable evidence: "From the test output, identify...", "Quote the exact log lines...", "From the actual test output, what is..."
- **Environment Constraint:** The build environment lacks a C compiler (`gcc`), requiring `CGO_ENABLED=0` for all Go test execution. This excludes SQLite-dependent packages but does not affect any of the target test packages (`internal/endpoint/smtp/`, `internal/target/queue/`, `internal/target/remote/`).

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document SMTP endpoint log behavior**, we will execute `go test -v -count=1 -test.debuglog ./internal/endpoint/smtp/` targeting `TestSMTPDelivery`, `TestSMTPDelivery_AbortData`, `TestSMTPDelivery_AbortLogout`, and `TestSMTPDelivery_SubmissionAuthOK`, then capture and annotate the structured log output emitted by the `internal/log` library through `testutils.Logger`
- To **trace queue delivery mechanics**, we will execute `go test -v -count=1 ./internal/target/queue/` targeting `TestQueueDelivery`, `TestQueueDelivery_PermanentFail_NonPartial`, `TestQueueDelivery_TemporaryFail`, `TestQueueDelivery_MultipleAttempts`, and `TestQueueDelivery_SerializationRoundtrip`, then extract the complete log sequence from acceptance through retry scheduling
- To **document MX authentication failures and TLS fallback**, we will execute `go test -v -count=1 -test.debuglog ./internal/target/remote/` targeting `TestRemoteDelivery_AuthMX_Fail`, `TestRemoteDelivery_TLSErrFallback`, `TestRemoteDelivery_RequireTLS`, `TestRemoteDelivery_AuthMX_DNSSEC`, and `TestRemoteDelivery_AuthMX_CommonDomain`, then extract exact error strings, SMTP enhanced status codes, and TLS fallback log messages
- To **create the output documentation**, we will synthesize all captured test output into a single `maddy.md` file organized by subsystem (SMTP endpoint, queue, remote delivery), with each section containing annotated log line examples, JSON field inventories, and Mermaid diagrams showing message flow

### 0.1.4 Inferred Documentation Needs

Based on code analysis and test output, the following implicit documentation needs are surfaced:

- **Log Format Specification:** The `internal/log` library uses a custom format `name: msg\t{"key":"value",...}` with tab-separated ordered JSON. No existing documentation describes this format. The onboarding guide must include a formal specification of the log line grammar.
- **Message ID Generation and Mutation:** The `msg_id` field observed in test output is an 8-character hexadecimal string (first 4 bytes of SHA-1 hash of `t.Name()` in tests, `google/uuid` in production). For queue retries, the ID mutates to `originalID + "-" + attemptNumber` (e.g., `"abc123-2"`). This dual-format behavior requires explicit documentation.
- **Error Wrapping Taxonomy:** The `internal/exterrors` package decorates errors with structured fields (`smtp_code`, `smtp_enchcode`, `smtp_msg`, `target`). Remote delivery errors wrap connection failures into this format. The documentation must show the complete error field hierarchy.
- **Delivery Lifecycle State Machine:** The SMTP session lifecycle (`MAIL FROM → RCPT TO → DATA → Commit/Abort`) produces a deterministic sequence of log messages. A state diagram connecting session events to log output will aid onboarding.
- **TLS Policy Decision Tree:** The remote delivery module implements a multi-layer security policy check (implicit auth → MTA-STS → DNSSEC → common domain). The documentation must include a decision tree showing how each policy gate produces its respective log messages and error strings.
- **Queue Retry Formula:** The retry delay follows `initialRetryTime × retryTimeScale^(TriesCount-1)` with defaults of 15 minutes initial and 2× scale factor. This formula is not documented outside the source code and must be included in the onboarding reference.


## 0.2 Documentation Discovery and Analysis


### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **MkDocs-based documentation site** with limited runtime behavior coverage and no existing documentation about log message formats or test execution traces.

**Documentation Framework:**

| Attribute | Value | Source |
|-----------|-------|--------|
| Generator | MkDocs | `.mkdocs.yml` |
| Theme | ReadTheDocs | `.mkdocs.yml` line 5 |
| Markdown extensions | `codehilite` (syntax highlighting) | `.mkdocs.yml` lines 7–9 |
| Published URL | `https://foxcpp.dev/maddy/` | `.mkdocs.yml` line 1 |
| Man page compiler | `scdoc` | `docs/man/README.md`, `.build.yml` |

**Documentation file inventory discovered:**

| File Path | Type | Content |
|-----------|------|---------|
| `README.md` | Project overview | Features, badges, community links, planned features |
| `HACKING.md` | Developer guide | Design goals, module architecture, error handling conventions |
| `docs/tutorials/setting-up.md` | Tutorial | Initial server setup walkthrough |
| `docs/tutorials/manual-installation.md` | Tutorial | Manual build/install instructions |
| `docs/tutorials/alias-to-remote.md` | Tutorial | Remote alias configuration |
| `docs/get.sh-script.md` | Reference | Bootstrap installer documentation |
| `docs/internals/quirks.md` | Internals | Protocol implementation deviations |
| `docs/internals/sqlite.md` | Internals | SQLite configuration and maintenance |
| `docs/man/*.scd` | Man pages | 9 scdoc source files covering all modules |

**Documentation gaps relevant to this task:**
- No documentation exists describing the structured log format used by `internal/log`
- No documentation covers the JSON fields emitted by each module during operation
- No documentation traces message lifecycle through the queue retry system
- No documentation describes the MX authentication policy decision tree or TLS fallback behavior
- No documentation provides annotated test execution output for onboarding purposes

### 0.2.2 Repository Code Analysis for Documentation

The following source files and packages were examined to extract documentable runtime behavior:

**Log infrastructure (`internal/log/`):**
- `log.go` (261 lines): `Logger` struct with `Name`, `Debug`, `Fields` map. Methods: `Msg(msg, fields...)`, `Error(msg, err, fields...)`, `DebugMsg(kind, fields...)`, `Printf(format, args...)`. Custom `marshalOrderedJSON` serializer produces alphabetically-ordered JSON. `formatMsg` merges logger-level `Fields` with per-call fields.
- `output.go` (42 lines): `Output` interface with `Write(stamp time.Time, debug bool, msg string)`. Implementations: `FuncOutput`, `NopOutput`, `MultiOutput`.

**SMTP endpoint (`internal/endpoint/smtp/`):**
- `smtp.go` (721 lines): Session lifecycle with log points at `incoming message`, `RCPT ok`, `RCPT error`, `MAIL FROM error`, `DATA error`, `accepted`, `aborted`. Authenticated sessions add `"username"` field. `wrapErr` appends `(msg ID = ...)` to SMTP error text.
- `smtp_test.go` (534 lines): Tests use `testEndpoint()` with mock DNS, mock pipeline target, random port assignment via `TestMain`.

**Queue delivery (`internal/target/queue/`):**
- `queue.go` (958 lines): `tryDelivery` method with log points at `delivered`, `delivery attempt failed`, `not delivered, permanent error`, `not delivered, temporary error`, `will retry`, `generated failed DSN`. Retry ID format: `msgMeta.ID + "-" + strconv.Itoa(meta.TriesCount+1)`. Exponential backoff: `initialRetryTime * retryTimeScale^(TriesCount-1)`.
- `queue_test.go` (823 lines): `unreliableTarget` mock with configurable `bodyFailures`, `bodyFailuresPartial`, `rcptFailures` for fault injection.
- `delivery.go`: `DeliveryLogger(l, msgMeta)` enriches logger with `msg_id` from metadata.

**Remote delivery (`internal/target/remote/`):**
- `connect.go` (277 lines): MX connection logic with TLS fallback. Log points: `trying`, `TLS error, falling back to plaintext`, `connected`, `TLS required by MTA-STS`, `authenticated MX using...`, `skipping MX not matching MTA-STS`. Policy errors: "Failed to estabilish the MX record authenticity" (550/5.7.0), "TLS is required but unsupported or failed" (550/5.7.1), "Domain does not accept email (null MX)" (556/5.1.10).
- `remote.go` (482 lines): Module registration, MTA-STS cache updater, per-domain delivery dispatch.
- `remote_test.go` (982 lines) and `mxauth_test.go` (606 lines): Comprehensive integration tests using mock DNS, in-process SMTP servers with TLS.

**SMTP connection wrapper (`internal/smtpconn/`):**
- `smtpconn.go` (339 lines): `Connect` handles dial, implicit TLS, STARTTLS, EHLO/HELO. `TLSError` type wraps STARTTLS failures. `wrapClientErr` converts `*smtp.SMTPError` into `*exterrors.SMTPError` with `"serverName said: msg"` format.

**Test utilities (`internal/testutils/`):**
- `logger.go`: `Logger(t, name)` wires logger output to `t.Log()` with debug prefix `[debug]`. Flags: `-test.debuglog`, `-test.directlog`.
- `target.go`: `Target` mock with `DoTestDelivery` generating deterministic message IDs from `sha1(t.Name())`.

### 0.2.3 Web Search Research Conducted

No web search is required for this documentation task. All content is derived from actual repository source code analysis and test execution output. The documentation type is a runtime behavior reference based entirely on first-party evidence from the codebase, and no external best practices or tooling recommendations need to be validated.


## 0.3 Documentation Scope Analysis


### 0.3.1 Code-to-Documentation Mapping

**Module: `internal/log/` — Logging Infrastructure**
- Public APIs: `Logger.Msg()`, `Logger.Error()`, `Logger.DebugMsg()`, `Logger.Printf()`, `marshalOrderedJSON()`
- Current documentation: Missing — no existing documentation describes the log line format
- Documentation needed: Log format grammar specification, JSON field ordering rules, debug message prefix convention

**Module: `internal/endpoint/smtp/` — SMTP Endpoint**
- Public log messages: `incoming message`, `RCPT ok`, `RCPT error`, `MAIL FROM error`, `DATA error`, `accepted`, `aborted`, `authentication failed`, `adding missing Message-ID`, `adding missing Date header`
- Current documentation: Man page `maddy-smtp.5.scd` covers configuration only, not runtime log behavior
- Documentation needed: Complete log message catalog with JSON field inventories per message type, annotated examples from test output comparing successful delivery vs. aborted delivery

**Module: `internal/target/queue/` — Delivery Queue**
- Public log messages: `delivered`, `delivery attempt failed`, `not delivered, permanent error`, `not delivered, temporary error`, `will retry`, `generated failed DSN`, `loaded N saved queue entries`
- Current documentation: Man page `maddy-targets.5.scd` covers configuration only
- Documentation needed: Complete message lifecycle trace from acceptance through retry, retry scheduling formula documentation, JSON field inventory including `next_try_delay`, `attempts_count`, `attempt`, `rcpts`

**Module: `internal/target/remote/` — Remote MX Delivery**
- Public log messages: `trying`, `connected`, `TLS error, falling back to plaintext`, `TLS required by MTA-STS`, `TLS required by local policy`, `authenticated MX using DNSSEC`, `authenticated MX using common domain rule`, `skipping MX not matching MTA-STS`
- Error messages: "Failed to estabilish the MX record authenticity" (550/5.7.0), "TLS is required but unsupported or failed" (550/5.7.1), "Domain does not accept email (null MX)" (556/5.1.10)
- Current documentation: Man page `maddy-targets.5.scd` covers configuration, no log/error reference
- Documentation needed: MX authentication error catalog with exact SMTP codes and enhanced status codes, TLS fallback behavior documentation with annotated log output, policy decision tree

**Module: `internal/smtpconn/` — SMTP Connection Wrapper**
- Public log messages: `connected` (with `remote_server` field)
- Error wrapping: `TLSError` type, `wrapClientErr` producing `"serverName said: msg"` format
- Current documentation: None
- Documentation needed: Connection-level log messages, TLS error wrapping behavior, error format documentation

**Module: `internal/testutils/` — Test Infrastructure**
- Key functions: `Logger(t, name)`, `DoTestDelivery()`, `SMTPServer()` variants
- Current documentation: None
- Documentation needed: Test execution flag reference (`-test.debuglog`, `-test.directlog`), test ID generation mechanism, test environment setup instructions

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

**Undocumented runtime behavior (critical gaps):**
- Log line format grammar: `name: msg\t{"key":"value",...}` — nowhere documented
- Per-module log message catalog — no reference for any module's log output
- JSON field inventory per log message type — fields like `msg_id`, `rcpt`, `attempt`, `next_try_delay`, `domain`, `mx`, `reason` are undocumented
- Message ID format: 8-character hex string (production: first 4 bytes of UUID; tests: first 4 bytes of SHA-1)
- Queue retry ID mutation: `originalID-N` format for retry attempts
- Retry delay formula: `initialRetryTime × retryTimeScale^(TriesCount-1)` — exists only in source code
- MX authentication error text and SMTP status codes — exist only in `connect.go` string literals
- TLS fallback decision logic — exists only in `connect.go` control flow

**Undocumented test execution patterns:**
- How to run individual test suites with debug logging enabled
- How to interpret test log output in the context of message tracing
- The `CGO_ENABLED=0` workaround for environments without a C compiler
- Test port management via `-test.smtpport` flag

**Missing onboarding material:**
- No "how to trace a message" guide exists
- No annotated examples of log output for common scenarios
- No comparison of success vs. failure log patterns
- No architectural diagrams connecting modules to their log output


## 0.4 Documentation Implementation Design


### 0.4.1 Documentation Structure Planning

The output is a single comprehensive markdown file (`maddy.md`) written to the `blitzy-research/AtlasQnA` destination repository. The file structure follows the user's request sequence:

```
maddy.md
├── Introduction (onboarding context, environment setup)
├── Log Format Specification
│   ├── Grammar definition
│   ├── JSON field ordering rules
│   └── Debug message convention
├── SMTP Endpoint Runtime Behavior
│   ├── Successful delivery log trace (annotated)
│   ├── Aborted delivery log trace (annotated)
│   ├── Comparison: success vs. abort
│   ├── Module name identification
│   ├── msg_id format specification
│   └── Submission authentication log trace
├── Queue Delivery Mechanics
│   ├── Successful delivery log trace
│   ├── Permanent failure log trace
│   ├── Temporary failure with retry log trace
│   ├── Complete acceptance-to-retry sequence
│   ├── Retry scheduling formula
│   └── JSON field inventory (next_try_delay, etc.)
├── Remote Delivery: MX Auth and TLS
│   ├── MX authentication failure error catalog
│   ├── SMTP enhanced status codes (X.Y.Z)
│   ├── TLS fallback log trace (all JSON fields)
│   ├── DNSSEC authentication log trace
│   ├── Common domain authentication log trace
│   ├── RequireTLS enforcement log trace
│   └── Policy decision tree diagram
├── Cross-Module JSON Field Reference
│   ├── SMTP endpoint fields
│   ├── Queue fields
│   ├── Remote delivery fields
│   └── Error wrapping fields
├── Test Execution Reference
│   ├── Commands to reproduce each test scenario
│   ├── Environment setup (CGO_ENABLED=0)
│   └── Debug logging flags
└── Architectural Diagrams
    ├── Message flow through pipeline
    ├── SMTP session state machine
    ├── Queue retry lifecycle
    └── Remote delivery TLS decision tree
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract log format grammar from `internal/log/log.go` (`formatMsg`, `marshalOrderedJSON` functions)
- Extract all log message strings from `internal/endpoint/smtp/smtp.go` (`s.log.Msg(...)` and `s.endp.Log.Msg(...)` call sites)
- Extract queue log messages from `internal/target/queue/queue.go` (`dl.Msg(...)`, `dl.Error(...)`, `dl.Debugf(...)` call sites)
- Extract remote delivery log messages from `internal/target/remote/connect.go` (`rd.Log.DebugMsg(...)`, `rd.Log.Error(...)`, `rd.Log.Msg(...)` call sites)
- Capture exact test output from all executed test runs with `-v -test.debuglog` flags

**Template Application:**
- Each log trace section follows a consistent pattern: test command → raw output → annotated breakdown → JSON field table
- Error catalog sections use a consistent table format: error text → SMTP code → enhanced code → triggering condition → source file reference

**Documentation Standards:**
- All log lines quoted verbatim from test output, formatted in `code blocks`
- JSON fields documented in tables with name, type, description, and example value
- Mermaid diagrams for all state machines and decision trees
- Source citations as inline references: `Source: internal/endpoint/smtp/smtp.go:L445`
- Test commands provided as reproducible copy-paste snippets

### 0.4.3 Diagram and Visual Strategy

**Mermaid diagrams to create:**

- **SMTP Session State Machine:** Flowchart showing session lifecycle from connection through `MAIL FROM → RCPT TO → DATA → Commit/Abort`, with log messages emitted at each transition
- **Queue Retry Lifecycle:** Sequence diagram showing message acceptance, initial delivery attempt, failure, retry scheduling with `will retry` log message, and subsequent attempts
- **Remote Delivery TLS Decision Tree:** Flowchart showing the policy check sequence: implicit auth → MTA-STS check → DNSSEC check → common domain check → TLS negotiation → fallback decision, with corresponding log messages and error codes at each branch
- **Cross-Module Message Flow:** Flowchart tracing a message from SMTP endpoint acceptance through pipeline routing to queue enqueue to remote delivery attempt, annotated with the module name prefix that appears in each log line (`smtp:`, `queue:`, `remote:`)

All diagrams use `mermaid` code blocks with clear node labels derived from actual log message text observed in test output.


## 0.5 Documentation File Transformation Mapping


### 0.5.1 File-by-File Documentation Plan

Per the user's implementation rule, all documentation is consolidated into a single markdown file in the destination repository. No source repository files are modified.

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `maddy.md` (in `blitzy-research/AtlasQnA`) | CREATE | `internal/log/log.go`, `internal/endpoint/smtp/smtp.go`, `internal/target/queue/queue.go`, `internal/target/remote/connect.go`, `internal/target/remote/remote.go`, `internal/smtpconn/smtpconn.go`, `internal/testutils/logger.go`, `internal/testutils/target.go`, `internal/target/delivery.go` | Comprehensive runtime behavior documentation with annotated test output, log format specification, JSON field inventories, error catalogs, and architectural diagrams. Sections cover SMTP endpoint log taxonomy, queue delivery trace, MX auth failure errors with SMTP codes, TLS fallback log output, and retry scheduling field identification |

**No other documentation files are created, updated, or deleted.** The user's instruction explicitly constrains output to a single markdown file and prohibits source repository modifications.

### 0.5.2 New Documentation File Detail

```
File: maddy.md
Location: blitzy-research/AtlasQnA/maddy.md
Type: Runtime Behavior Reference / Onboarding Guide

Source Code Files (for content extraction):
  - internal/log/log.go (log format grammar, JSON serialization)
  - internal/log/output.go (Output interface)
  - internal/endpoint/smtp/smtp.go (SMTP session log points)
  - internal/endpoint/smtp/smtp_test.go (test fixtures for SMTP scenarios)
  - internal/target/queue/queue.go (queue delivery log points, retry formula)
  - internal/target/queue/queue_test.go (test fixtures for queue scenarios)
  - internal/target/delivery.go (DeliveryLogger msg_id enrichment)
  - internal/target/remote/connect.go (MX connection, TLS fallback, policy errors)
  - internal/target/remote/remote.go (module registration, STS cache)
  - internal/target/remote/remote_test.go (remote delivery test fixtures)
  - internal/target/remote/mxauth_test.go (MX auth policy test fixtures)
  - internal/smtpconn/smtpconn.go (SMTP client wrapper, TLSError, error wrapping)
  - internal/testutils/logger.go (test logger configuration)
  - internal/testutils/target.go (mock target, deterministic ID generation)

Sections:
  - Environment Setup and Test Execution
    * Go 1.13 runtime requirement (Source: go.mod:L3)
    * CGO_ENABLED=0 workaround for environments without gcc
    * Debug logging flags: -test.debuglog, -test.directlog
  - Log Format Specification
    * Grammar: name: msg\t{"key":"value",...} (Source: internal/log/log.go formatMsg())
    * JSON ordering: alphabetical key sorting via marshalOrderedJSON()
    * Debug prefix: [debug] for DebugMsg output
    * Error field injection: "reason" key from err.Error()
  - SMTP Endpoint Log Taxonomy
    * Successful delivery: incoming message → RCPT ok → accepted
    * Aborted delivery: incoming message → DATA error → aborted
    * Module name prefix: "smtp" (Source: internal/endpoint/smtp/smtp.go:L109 s.endp.Log)
    * msg_id format: 8-char hex (Source: internal/testutils/target.go DoTestDelivery sha1)
    * Submission auth: "username" field added (Source: smtp.go:L445)
  - Queue Delivery Trace
    * Acceptance → delivered (single attempt success)
    * Acceptance → delivery attempt failed → will retry → delivered (retry success)
    * Acceptance → delivery attempt failed → not delivered, permanent error (permanent fail)
    * JSON fields: msg_id, rcpt, attempt, attempts_count, next_try_delay, rcpts, reason
    * Retry formula: initialRetryTime × retryTimeScale^(TriesCount-1)
  - Remote Delivery: MX Auth Failures
    * Error: "Failed to estabilish the MX record authenticity" (smtp_code:550, smtp_enchcode:5.7.0)
    * Error: "TLS is required but unsupported or failed" (smtp_code:550, smtp_enchcode:5.7.1)
    * Error: "Domain does not accept email (null MX)" (smtp_code:556, smtp_enchcode:5.1.10)
  - Remote Delivery: TLS Fallback
    * Log: "TLS error, falling back to plaintext" with fields: domain, msg_id, mx, reason
    * Trigger: smtpconn.TLSError when no auth policy prevents fallback
    * RequireTLS override: prevents fallback, returns 550/5.7.1
  - Cross-Module JSON Field Reference Table
  - Architectural Diagrams (Mermaid)

Diagrams:
  - SMTP session lifecycle with log message annotations
  - Queue retry lifecycle sequence diagram
  - Remote delivery TLS policy decision tree
  - Cross-module message flow with log prefix annotations

Key Citations:
  internal/log/log.go, internal/endpoint/smtp/smtp.go,
  internal/target/queue/queue.go, internal/target/remote/connect.go,
  internal/smtpconn/smtpconn.go, internal/testutils/logger.go,
  internal/testutils/target.go, internal/target/delivery.go
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The output is a standalone markdown file that does not participate in MkDocs navigation, man page compilation, or any existing documentation build pipeline. The `.mkdocs.yml` file in the maddy source repository is not modified per the user's explicit instruction.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content or includes**: The output file is self-contained with no cross-references to the existing maddy documentation site
- **No navigation link updates**: The file lives in a separate repository (`blitzy-research/AtlasQnA`)
- **Source code citations**: All sections reference source files by path, enabling readers to cross-reference with the maddy source tree. These are informational citations, not hyperlinks that require maintenance.
- **Test command reproducibility**: Each section includes the exact `go test` command used to generate the output, allowing readers to independently verify all quoted log lines


## 0.6 Dependency Inventory


### 0.6.1 Documentation Dependencies

The documentation task requires only the Go toolchain and the project's own dependencies for test execution. No dedicated documentation generation tools are needed since the output is a plain markdown file.

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| golang.org | Go toolchain | 1.13.15 | Runtime for test execution; minimum version per `go.mod` line 3 |
| go module | `github.com/emersion/go-smtp` | v0.12.1-0.20191206174923 | SMTP protocol library used by endpoint and connection tests |
| go module | `github.com/foxcpp/go-mockdns` | v0.0.0-20191123143003 | In-memory DNS mock for remote delivery and SMTP endpoint tests |
| go module | `github.com/emersion/go-msgauth` | v0.3.2-0.20191028231513 | DKIM/DMARC message authentication used by remote delivery |
| go module | `github.com/miekg/dns` | v1.1.22 | DNS library for MX resolution and DNSSEC verification |
| go module | `github.com/google/uuid` | v1.1.1 | Message ID generation in production code path |
| go module | `github.com/emersion/go-sasl` | v0.0.0-20190817083125 | SASL authentication for submission tests |
| go module | `github.com/emersion/go-message` | v0.10.9-0.20191116124005 | MIME message parsing and header handling |

**Build environment constraint:** `CGO_ENABLED=0` is required because `gcc` is unavailable. This disables the `mattn/go-sqlite3` driver (v1.11.0), which is irrelevant for the target test packages. All three test packages (`internal/endpoint/smtp/`, `internal/target/queue/`, `internal/target/remote/`) compile and pass cleanly without CGO.

### 0.6.2 Documentation Reference Updates

No documentation reference updates are applicable. The output file is a new standalone document in a separate repository. No existing documentation links need modification, and no link transformation rules apply.


## 0.7 Coverage and Quality Targets


### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis (before this documentation task):**

| Category | Documented | Total | Coverage |
|----------|-----------|-------|----------|
| SMTP endpoint log messages documented | 0 | 10 | 0% |
| Queue delivery log messages documented | 0 | 7 | 0% |
| Remote delivery log messages documented | 0 | 8 | 0% |
| JSON field names formally cataloged | 0 | 22 | 0% |
| Error messages with SMTP codes cataloged | 0 | 3 | 0% |
| Test execution guides for runtime tracing | 0 | 3 | 0% |

**Target coverage (after this documentation task):**

| Category | Target | Method |
|----------|--------|--------|
| SMTP endpoint log messages | 100% — all 10 message types | Extract from `smtp.go` log call sites + test output |
| Queue delivery log messages | 100% — all 7 message types | Extract from `queue.go` log call sites + test output |
| Remote delivery log messages | 100% — all 8 message types | Extract from `connect.go` and `remote.go` log call sites + test output |
| JSON field names | 100% — all 22 fields across modules | Catalog from source + verify in test output |
| Error messages with SMTP codes | 100% — all 3 policy error types | Extract from `connect.go` string literals + test output |
| Test execution guides | 100% — SMTP, queue, remote | Document exact commands with flags and expected output |

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every log message type observed in test output has a dedicated entry with: the exact log line text, a table of JSON fields with types and descriptions, the source file and line number where the log call occurs, and at least one annotated example from actual test execution
- Every error message includes: the exact error string, SMTP status code, SMTP enhanced status code (X.Y.Z format), the triggering condition, and the source code location
- Every test execution section includes: the exact `go test` command, environment variables required, and a description of what to expect in the output

**Accuracy validation:**
- All log lines are quoted from captured test output verbatim — no paraphrasing or reconstruction from source code alone
- All JSON field names are verified against both the source code (`Msg()` call arguments) and the actual test output
- All SMTP enhanced status codes are verified against `connect.go` string literals and confirmed in test error output
- The `msg_id` format is verified: 8-character hex in test output (e.g., `"35d681c0"`) matches the first 4 bytes of SHA-1 hash of `t.Name()` in `testutils.DoTestDelivery`

**Clarity standards:**
- Log line examples use consistent formatting: raw log line in code block, followed by field-by-field annotation table
- Comparison sections (success vs. abort) use side-by-side presentation or clearly labeled sequential blocks
- Technical terms are defined on first use (e.g., "enhanced status code", "MX authenticity", "TLS fallback")
- Diagrams accompany all multi-step processes (session lifecycle, retry sequence, policy decisions)

**Maintainability:**
- Every claim cites its source file path and line range
- Test commands are reproducible with no hidden state
- The retry formula is presented both in mathematical notation and with a worked example

### 0.7.3 Example and Diagram Requirements

| Content Type | Minimum Count | Verification Method |
|--------------|---------------|---------------------|
| Annotated log line examples (SMTP endpoint) | 6 (incoming message, RCPT ok, DATA error, aborted, accepted, submission with username) | Quoted from test output of `TestSMTPDelivery`, `TestSMTPDelivery_AbortData`, `TestSMTPDelivery_SubmissionAuthOK` |
| Annotated log line examples (queue) | 5 (delivered, delivery attempt failed, permanent error, will retry, loaded entries) | Quoted from test output of `TestQueueDelivery`, `TestQueueDelivery_PermanentFail_NonPartial`, `TestQueueDelivery_TemporaryFail` |
| Annotated log line examples (remote) | 5 (trying, connected, TLS fallback, authenticated MX, policy error) | Quoted from test output of `TestRemoteDelivery_TLSErrFallback`, `TestRemoteDelivery_AuthMX_Fail`, `TestRemoteDelivery_AuthMX_DNSSEC` |
| Mermaid diagrams | 4 (SMTP session, queue retry, TLS decision tree, cross-module flow) | Visual review for accuracy against source code control flow |
| JSON field reference tables | 4 (per-module: SMTP, queue, remote, error wrapping) | Cross-checked against source code `Msg()`/`Error()` call arguments |
| Reproducible test commands | 3 (one per test package) | Execute and verify output matches documented examples |


## 0.8 Scope Boundaries


### 0.8.1 Exhaustively In Scope

**New documentation file:**
- `maddy.md` in `blitzy-research/AtlasQnA` — the single output artifact containing all runtime behavior documentation

**Test execution and output capture (read-only investigation):**
- `CGO_ENABLED=0 go test -v -count=1 -test.debuglog ./internal/endpoint/smtp/` — SMTP endpoint tests
- `CGO_ENABLED=0 go test -v -count=1 ./internal/target/queue/` — Queue delivery tests
- `CGO_ENABLED=0 go test -v -count=1 -test.debuglog ./internal/target/remote/` — Remote delivery tests

**Source files analyzed for documentation content (read-only):**
- `internal/log/log.go` — log format grammar
- `internal/log/output.go` — output interface
- `internal/endpoint/smtp/smtp.go` — SMTP session log points
- `internal/endpoint/smtp/smtp_test.go` — SMTP test fixtures
- `internal/target/queue/queue.go` — queue delivery log points and retry formula
- `internal/target/queue/queue_test.go` — queue test fixtures
- `internal/target/delivery.go` — DeliveryLogger msg_id enrichment
- `internal/target/remote/connect.go` — MX connection, TLS fallback, policy errors
- `internal/target/remote/remote.go` — module registration, STS cache
- `internal/target/remote/remote_test.go` — remote delivery test fixtures
- `internal/target/remote/mxauth_test.go` — MX auth policy test fixtures
- `internal/smtpconn/smtpconn.go` — SMTP client wrapper, TLSError, error format
- `internal/testutils/logger.go` — test logger configuration
- `internal/testutils/target.go` — mock target, deterministic ID generation

**Documentation content areas covered:**
- Log format specification (grammar, JSON ordering, debug convention)
- SMTP endpoint log taxonomy (all 10 message types)
- Queue delivery log trace (all 7 message types, retry formula, JSON field inventory)
- Remote delivery MX auth error catalog (3 error types with SMTP codes)
- Remote delivery TLS fallback log behavior (all JSON fields)
- Cross-module JSON field reference (22 fields total)
- Test execution reference (3 test suites with exact commands)
- Architectural diagrams (4 Mermaid diagrams)

### 0.8.2 Explicitly Out of Scope

**Source code modifications:**
- No files in the maddy source repository (`/tmp/blitzy/maddy/maddy_26452dd8dd78_76465c/`) are modified per the user's explicit instruction: "Don't modify repository files"
- No docstrings, comments, or inline documentation are added to source files

**Test file modifications:**
- No test files are modified or created in the source repository
- Temporary scripts used during test execution must be cleaned up afterward

**Existing documentation updates:**
- `.mkdocs.yml` is not modified
- `docs/**/*.md` files in the maddy source tree are not modified
- `docs/man/*.scd` man pages are not modified
- `README.md` and `HACKING.md` are not modified

**Feature additions or code refactoring:**
- No functional changes to any Go source files
- No build system modifications (`go.mod`, `go.sum`, `.build.yml`)
- No configuration file changes (`maddy.conf`)

**Packages not covered in documentation:**
- `internal/msgpipeline/` — pipeline routing (not requested by user)
- `internal/check/*` — security checks (not requested)
- `internal/modify/*` — message modifiers (not requested)
- `internal/storage/sql/` — SQL storage (not requested)
- `internal/endpoint/imap/` — IMAP endpoint (not requested)
- `internal/auth/*` — authentication backends (not requested)
- `internal/dmarc/` — DMARC evaluation (not requested)
- `internal/mtasts/` — MTA-STS policy (not requested)
- `pkg/cfgparser/`, `pkg/logparser/` — public libraries (not requested)

**Deployment and infrastructure:**
- No Docker image changes
- No systemd unit modifications
- No CI pipeline modifications
- No documentation site deployment


## 0.9 Execution Parameters


### 0.9.1 Documentation-Specific Instructions

**Environment setup commands:**
```bash
export PATH=/usr/local/go/bin:$PATH
export CGO_ENABLED=0
cd /tmp/blitzy/maddy/maddy_26452dd8dd78_76465c
go mod download
```

**Test execution commands (for capturing documentation content):**

SMTP endpoint tests (successful delivery, abort, submission auth):
```bash
CGO_ENABLED=0 go test -v -count=1 \
  -run "TestSMTPDelivery$|TestSMTPDelivery_AbortData|TestSMTPDelivery_SubmissionAuthOK" \
  ./internal/endpoint/smtp/ -test.debuglog
```

Queue delivery tests (success, permanent fail, temporary fail, retry, serialization):
```bash
CGO_ENABLED=0 go test -v -count=1 \
  -run "TestQueueDelivery$|TestQueueDelivery_PermanentFail_NonPartial|TestQueueDelivery_TemporaryFail$|TestQueueDelivery_MultipleAttempts|TestQueueDelivery_SerializationRoundtrip" \
  ./internal/target/queue/
```

Remote delivery tests (MX auth failure, TLS fallback, RequireTLS, DNSSEC, CommonDomain):
```bash
CGO_ENABLED=0 go test -v -count=1 \
  -run "TestRemoteDelivery_AuthMX_Fail|TestRemoteDelivery_TLSErrFallback|TestRemoteDelivery_RequireTLS|TestRemoteDelivery_AuthMX_DNSSEC|TestRemoteDelivery_AuthMX_CommonDomain" \
  ./internal/target/remote/ -test.debuglog
```

**Default output format:** Markdown with Mermaid diagrams. No documentation build system is needed — the output is a self-contained `.md` file.

**Citation requirement:** Every section references source files by path and line number where applicable. Log line examples cite the test function name from which output was captured.

**Style guide:** Follows a runtime-tracing onboarding reference style — each section presents the exact test command, the raw log output, and an annotated breakdown. Technical terms are defined on first use. Code blocks use `go` or `json` syntax highlighting where appropriate. Log output uses no syntax highlighting to preserve fidelity.

**Documentation validation:** The output file can be validated by re-running the documented test commands and comparing the log output structure against the quoted examples. Field names and SMTP codes can be cross-referenced against the cited source file locations.


## 0.10 Rules for Documentation


The following rules are explicitly specified by the user or derived from user instructions and must be strictly enforced during documentation generation:

- **Do not modify any files in the source repository.** The maddy source tree is read-only. All test execution is observational. The only writable artifact is the output markdown file in the destination repository.
- **Write the complete answer as a single markdown file named `maddy.md`.** All documentation is consolidated into one file. No auxiliary files, images, or supplementary documents are created.
- **Use the `blitzy-research/AtlasQnA` repository as the destination.** The output file is written to this repository, not to the maddy source tree.
- **Temporary scripts are fine but clean them up afterward.** If any helper scripts are created during test execution, they must be deleted before the task completes.
- **Quote exact log lines from actual test output.** All log messages, error strings, JSON field names, and SMTP codes must be quoted from captured test execution output. Do not fabricate or reconstruct log lines from source code inspection alone.
- **Show the complete log line format including all JSON fields.** When the user requests "all JSON fields," every key-value pair in the tab-separated JSON payload must be listed and annotated.
- **Identify module names from log prefixes.** The module name prefix (e.g., `smtp`, `queue`, `remote`) appears before the colon in each log line and must be explicitly called out.
- **Report SMTP enhanced status codes in X.Y.Z format.** Enhanced codes like `5.7.0` must be presented in the standard three-number dotted format.
- **Include the exact JSON field name for retry delay.** The user specifically asks to "identify the exact JSON field name that contains the retry delay value" — this is `next_try_delay`.
- **Compare success vs. failure log output explicitly.** The user requests comparison between successful delivery and aborted delivery log patterns. Present these as clearly labeled, contrasting sections.


## 0.11 References


### 0.11.1 Source Files Examined

**Logging infrastructure:**
- `internal/log/log.go` — Logger struct, `Msg()`, `Error()`, `DebugMsg()`, `formatMsg()`, `marshalOrderedJSON()` (261 lines)
- `internal/log/output.go` — `Output` interface, `FuncOutput`, `NopOutput`, `MultiOutput` (42 lines)

**SMTP endpoint:**
- `internal/endpoint/smtp/smtp.go` — Session lifecycle, log points, `wrapErr`, rate limiting, deferred reject (721 lines)
- `internal/endpoint/smtp/smtp_test.go` — Integration tests: delivery, abort, reset, multi-message, rDNS (534 lines)
- `internal/endpoint/smtp/submission.go` — Submission personality, auth enforcement, header normalization

**Queue delivery:**
- `internal/target/queue/queue.go` — Queue module, `tryDelivery`, retry scheduling, disk persistence, DSN generation (958 lines)
- `internal/target/queue/queue_test.go` — Unit/integration tests: success, permanent/temporary fail, multiple attempts, serialization (823 lines)
- `internal/target/delivery.go` — `DeliveryLogger` helper, msg_id enrichment

**Remote delivery:**
- `internal/target/remote/connect.go` — MX connection, TLS negotiation, policy checks, fallback logic (277 lines)
- `internal/target/remote/remote.go` — Module registration, per-domain dispatch, MTA-STS cache updater (482 lines)
- `internal/target/remote/remote_test.go` — Integration tests: TLS fallback, RequireTLS, NullMX, delivery flows (982 lines)
- `internal/target/remote/mxauth_test.go` — MX auth policy tests: DNSSEC, CommonDomain, MTA-STS, fail cases (606 lines)

**SMTP connection wrapper:**
- `internal/smtpconn/smtpconn.go` — `Connect`, STARTTLS, `TLSError`, `wrapClientErr`, EHLO/HELO negotiation (339 lines)

**Test utilities:**
- `internal/testutils/logger.go` — `Logger(t, name)`, debug/direct flags, `FuncOutput` wiring
- `internal/testutils/target.go` — `Target` mock, `DoTestDelivery`, `CheckMsg`, `CheckMsgID`, deterministic SHA-1 IDs
- `internal/testutils/smtp_server.go` — In-process SMTP server with TLS variants, connection leak detection

**Project configuration and documentation:**
- `go.mod` — Module path `github.com/foxcpp/maddy`, Go 1.13 minimum, all dependency versions
- `.mkdocs.yml` — MkDocs site configuration, ReadTheDocs theme, navigation structure
- `HACKING.md` — Developer guide: design goals, module architecture, error handling conventions (130 lines)
- `README.md` — Project overview, features, community links (78 lines)
- `docs/internals/quirks.md` — Protocol implementation deviations
- `docs/internals/sqlite.md` — SQLite configuration and maintenance

### 0.11.2 Folders Explored

- Repository root (`""`) — project overview, configuration files, build scripts
- `internal/` — private implementation packages (23 packages)
- `internal/log/` — structured logging library
- `internal/endpoint/smtp/` — SMTP/submission/LMTP endpoint
- `internal/target/queue/` — disk-backed delivery queue with retry
- `internal/target/remote/` — remote MX delivery with TLS and policy enforcement
- `internal/target/` — delivery target abstractions
- `internal/smtpconn/` — outbound SMTP client wrapper
- `internal/testutils/` — centralized test infrastructure (7 files)
- `docs/` — documentation source tree (tutorials, man pages, internals)
- `docs/man/` — 9 scdoc man page sources
- `docs/internals/` — implementation quirks and SQLite documentation
- `docs/tutorials/` — setup and configuration tutorials

### 0.11.3 Test Suites Executed

| Test Suite | Package | Tests Run | Result | Key Output Captured |
|------------|---------|-----------|--------|---------------------|
| SMTP Endpoint | `internal/endpoint/smtp/` | `TestSMTPDelivery`, `TestSMTPDelivery_AbortData`, `TestSMTPDelivery_AbortLogout`, `TestSMTPDelivery_Reset`, `TestSMTPDelivery_Multi`, `TestSMTPDelivery_SubmissionAuthOK` | All PASS | Log prefixes, msg_id format, JSON fields for incoming/RCPT ok/accepted/aborted/DATA error, submission username field |
| Queue Delivery | `internal/target/queue/` | `TestQueueDelivery`, `TestQueueDelivery_PermanentFail_NonPartial`, `TestQueueDelivery_TemporaryFail`, `TestQueueDelivery_MultipleAttempts`, `TestQueueDelivery_TemporaryRcptReject`, `TestQueueDelivery_SerializationRoundtrip` | All PASS | Delivered/failed/retry log lines, `next_try_delay` field, `attempts_count` field, serialization roundtrip log |
| Remote Delivery (TLS) | `internal/target/remote/` | `TestRemoteDelivery_TLSErrFallback`, `TestRemoteDelivery_RequireTLS` | All PASS | TLS fallback log with all JSON fields, RequireTLS enforcement error |
| Remote Delivery (MX Auth) | `internal/target/remote/` | `TestRemoteDelivery_AuthMX_Fail`, `TestRemoteDelivery_AuthMX_DNSSEC`, `TestRemoteDelivery_AuthMX_CommonDomain`, `TestRemoteDelivery_NullMX` | All PASS | MX authenticity error string, SMTP 550/5.7.0, DNSSEC auth log, common domain auth log, NullMX 556/5.1.10 |

### 0.11.4 Attachments

No attachments were provided for this project. No Figma screens or external design documents are referenced.


