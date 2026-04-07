# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that provides a deep, empirically-grounded analysis of maddy mail server's message processing pipeline, specifically focused on the queue delivery subsystem, retry scheduling mechanism, timeout behavior, filesystem persistence model, and multi-message queue scheduling dynamics.

**Documentation Category:** Create new documentation
**Documentation Type:** Technical investigation guide / Internals deep-dive analysis

The specific documentation requirements, restated with technical precision, are:

- **Pipeline Flow Documentation** — Trace the end-to-end path of a message from SMTP ingress through `internal/msgpipeline/` routing, into `internal/target/queue/` persistence, through `internal/target/remote/` delivery attempts, to final disposition (delivery success or DSN generation)
- **Retry Mechanics Under Failure** — Document the exact behavior when `max_tries` is set to a small value (e.g., 2–3) and the `remote` delivery target encounters a non-responsive SMTP destination that times out. This requires analyzing the exponential backoff formula `initialRetryTime * retryTimeScale ^ (TriesCount - 1)` with defaults of 15 minutes base and 2x scale factor, as implemented in `queue.go` lines 413–414
- **Connection Timeout Behavior** — Document what occurs at the TCP/SMTP layer when the destination is unreachable, including the OS-level TCP connection timeout inherited via Go's `net.Dialer` (no explicit timeout configured in `smtpconn.go` line 59), and how `smtpconn.C.wrapClientErr()` classifies the resulting `*net.OpError` into structured SMTP errors
- **Log Entry Analysis** — Document the exact structured log output produced during each retry attempt, including the JSON field format from `internal/log/orderedjson.go`, the `msg_id` field injected by `target.DeliveryLogger()`, and the specific log messages emitted by `tryDelivery()` (line 367: `"delivery attempt #%d"`), `deliver()` (debug messages for target.Start, AddRcpt, Body), and retry scheduling (line 415: `"will retry"` with `attempts_count`, `next_try_delay`, `rcpts` fields)
- **Queue Filesystem Layout** — Document the three-file persistence model (`.header`, `.body`, `.meta`) including the JSON structure of `QueueMetadata` containing `From`, `To`, `FailedRcpts`, `TemporaryFailedRcpts`, `RcptErrs`, `TriesCount`, `FirstAttempt`, `LastAttempt`, and the atomic update strategy using `.meta.new` → rename
- **Multi-Message Scheduling** — Document how the `TimeWheel` scheduler in `timewheel.go` handles concurrent messages destined for different domains, specifically the linked-list scan for the nearest deadline (lines 78–84), the single-goroutine dispatch model, and the semaphore-bounded parallelism via `deliverySemaphore` (default 16)
- **Queue Starvation Analysis** — Document whether and how a slow-timeout delivery to domain A can delay dispatch of message B to domain B, considering the TimeWheel's blocking dispatch pattern and the delivery semaphore contention

### 0.1.2 Special Instructions and Constraints

**Critical Directives:**
- **No codebase modification** — The source repository must remain completely unchanged. No files in the maddy repository may be edited, added, or deleted
- **Implementation Rule: SWE-AtlasQnA-Repo** — The complete analysis must be written as a single markdown file named `<project_name>.md` in the blitzy-research/AtlasQnA destination repository
- **Test configurations allowed** — Test configurations and intentionally failing destinations may be created outside the repository for experimental observation
- **Empirical focus** — Documentation must describe what actually happens during execution, not theoretical behavior. This requires tracing actual code paths in the source and correlating them with observable outputs
- **No design system** — No UI component library or design system is relevant to this task

**Style Preferences:**
- Emphasis on precise, verifiable claims backed by source code line references
- Include exact log output formats with field names and JSON structure
- Include filesystem path patterns with concrete examples
- Use Mermaid diagrams for flow visualization
- Provide timing calculations with the actual backoff formula

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document the **message processing pipeline**, we will create a comprehensive flow analysis tracing code paths through `internal/endpoint/smtp/` → `internal/msgpipeline/msgpipeline.go` → `internal/target/queue/queue.go` → `internal/target/remote/remote.go` → `internal/smtpconn/smtpconn.go`, with Mermaid sequence diagrams showing the exact method call chain
- To document the **retry mechanics**, we will analyze `queue.go` lines 365–429 (`tryDelivery`) and lines 431–532 (`deliver`), calculating concrete retry timing tables for small `max_tries` values using the formula `15min × 2^(n-1)`
- To document the **timeout behavior**, we will analyze `smtpconn.go` `New()` (line 57–63) which creates a `net.Dialer` with zero-value timeout (inherits OS TCP default, typically ~2 minutes on Linux), and `connect.go` `connectionForDomain()` which iterates MX hosts with this dialer
- To document the **log entries**, we will extract all `Log.Msg()`, `Log.Error()`, `Log.Debugf()`, and `Log.Debugln()` calls from the queue and remote delivery code paths, mapping them to the JSON output format defined by `marshalOrderedJSON()` in `orderedjson.go`
- To document the **queue filesystem**, we will analyze `storeNewMessage()` (lines 690–740), `updateMetadataOnDisk()` (lines 742–767), `readMessageMeta()` (lines 769–791), and `readDiskQueue()` (lines 622–688) to produce the exact file layout, naming convention, and JSON schema
- To document the **multi-message scheduling**, we will analyze `TimeWheel.tick()` (lines 71–128) including its single-goroutine scanning loop, timer-based waiting, and the `updateNotify` channel for re-evaluation when new slots arrive
- To document the **queue starvation scenario**, we will analyze the interaction between `dispatch()` (lines 275–323) which launches goroutines for delivery, the `deliverySemaphore` (bounded channel of size `max_parallelism`), and the blocking `Dialer` call in `smtpconn.go`

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs are identified:

- **Post-init delay behavior** — The `postInitDelay` field (default 10 seconds, `queue.go` line 188) delays deliveries scheduled within this window after startup, which affects restart-recovery timing and should be documented
- **Error classification logic** — The `exterrors.IsTemporaryOrUnspec()` function defaults unclassified errors to temporary (`temporary.go` lines 15–21), which directly determines whether a failed delivery is retried. This is critical to understand timeout behavior since TCP timeouts produce `*net.OpError` which wraps through `wrapClientErr()` into SMTP code 450 (temporary)
- **Partial delivery semantics** — The `PartialDelivery` interface (`module/partial_delivery.go`) and `partialError` type in `queue.go` enable per-recipient failure tracking, meaning some recipients may succeed while others are retried
- **DSN generation conditions** — The `emitDSN()` function (lines 849–953) has specific suppression conditions (null sender, no bounce pipeline configured) that affect what happens after retry exhaustion
- **Metadata atomic update** — The `.meta.new` → rename pattern in `updateMetadataOnDisk()` (lines 742–767) provides crash-safe metadata persistence, which is important for understanding queue state between failures
- **Dangling file cleanup** — The `readDiskQueue()` function performs consistency checks for orphaned `.header`, `.body`, or `.meta` files during startup recovery, which is relevant to understanding queue robustness

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **MkDocs-based documentation site** with limited internals coverage and no existing documentation about queue retry mechanics, timeout behavior, or message scheduling.

**Documentation Framework:**
- **Generator:** MkDocs (configured via `.mkdocs.yml` at repository root)
- **Theme:** ReadTheDocs
- **Markdown Extensions:** `codehilite` with `guess_lang: false`
- **Site Name:** "maddy documentation"
- **Repository URL:** `https://github.com/foxcpp/maddy`

**Documentation Navigation Structure (from `.mkdocs.yml`):**
- `README.md` — Project overview
- **Tutorials:**
  - `tutorials/setting-up.md` — End-to-end deployment guide
  - `tutorials/manual-installation.md` — Source installation procedure
  - `tutorials/alias-to-remote.md` — Alias-to-remote routing pattern
- `get.sh-script.md` — Bootstrap installer documentation
- **Manual pages** — Generated man pages (`man/_generated_maddy*.md`)
- **Internals:**
  - `internals/quirks.md` — SMTP/IMAP protocol quirks
  - `internals/sqlite.md` — SQLite backend operational details

**API Documentation Tools:** None. No JSDoc, Godoc generation, or similar tooling is configured. Code documentation relies on Go doc comments and the `HACKING.md` contributor guide.

**Diagram Tools:** No Mermaid or PlantUML tooling is configured in the documentation build pipeline. The MkDocs configuration does not include any diagram rendering extensions.

**Coverage Status:** The existing documentation covers deployment, installation, and basic configuration. There is **no documentation** covering:
- Queue delivery internals and retry behavior
- Message processing pipeline flow with code-level detail
- Log format specifications and structured field descriptions
- Queue filesystem layout and metadata schema
- Timeout mechanics and error classification
- Multi-message scheduling behavior

### 0.2.2 Repository Code Analysis for Documentation

**Search patterns used for code to document:**

- **Queue subsystem:** `internal/target/queue/queue.go` (957 lines) — Core queue module with delivery retry, persistence, and DSN generation
- **Queue scheduler:** `internal/target/queue/timewheel.go` (128 lines) — Concurrent time-wheel scheduler for delayed retries
- **Queue tests:** `internal/target/queue/queue_test.go` (822 lines) — Behavioral test suite demonstrating retry, persistence, and cleanup patterns
- **Remote delivery:** `internal/target/remote/remote.go` (486 lines) — Outbound SMTP delivery target with MX resolution
- **MX connection:** `internal/target/remote/connect.go` (276 lines) — MX host connection, TLS policy, and failover logic
- **SMTP connection:** `internal/smtpconn/smtpconn.go` (338 lines) — SMTP client wrapper with error classification
- **Pipeline orchestrator:** `internal/msgpipeline/msgpipeline.go` — Two-level routing engine
- **Logging core:** `internal/log/log.go` (208 lines) — Logger type with structured JSON output
- **Log formatting:** `internal/log/orderedjson.go` (62 lines) — Deterministic JSON serialization
- **Log output:** `internal/log/writer.go` (77 lines) — Timestamp and debug prefix formatting
- **Error model:** `internal/exterrors/smtp.go` (128 lines) — SMTP error type with Fields() method
- **Error classification:** `internal/exterrors/temporary.go` (56 lines) — Temporary/permanent error determination
- **Delivery logger:** `internal/target/delivery.go` (16 lines) — Message-scoped logger enrichment
- **Buffer system:** `internal/buffer/file.go` — File-backed blob storage for message bodies
- **DSN generation:** `internal/dsn/dsn.go` — RFC 3464 bounce message construction
- **Module interfaces:** `internal/module/delivery_target.go`, `internal/module/partial_delivery.go` — Delivery contracts
- **Default config:** `maddy.conf` (153 lines) — Reference configuration with queue settings
- **Design guide:** `HACKING.md` (131 lines) — Contributor architecture overview

**Key directories examined:**
- `internal/target/queue/` — Complete queue subsystem
- `internal/target/remote/` — Remote MX delivery
- `internal/smtpconn/` — SMTP connection handling
- `internal/msgpipeline/` — Message pipeline routing
- `internal/log/` — Logging subsystem
- `internal/exterrors/` — Error handling utilities
- `internal/buffer/` — Message buffer storage
- `internal/dsn/` — DSN generation
- `internal/module/` — Module interfaces and contracts
- `internal/config/` — Configuration and directory management
- `internal/limiters/` — Concurrency and rate limiting primitives
- `docs/` — Existing documentation
- `docs/internals/` — Existing internals documentation

**Related documentation found:**
- `docs/internals/quirks.md` — Documents SMTP protocol deviations but not queue behavior
- `docs/internals/sqlite.md` — Documents SQLite backend but not queue storage
- `HACKING.md` — Provides module architecture overview and error handling guidelines but not queue scheduling specifics

### 0.2.3 Web Search Research Conducted

No web search was required for this task. The documentation requirements are focused on empirical analysis of the existing codebase behavior, and all necessary information was obtained through direct source code inspection. The maddy project documentation at `https://github.com/foxcpp/maddy` and the existing in-repository documentation provide sufficient context for the Go standard library networking defaults (TCP connect timeout) and SMTP protocol behavior.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

**Module: `internal/target/queue/queue.go`**
- Public APIs: `Queue` struct, `NewQueue()`, `Init()`, `Start()`, `Close()`, `Name()`, `InstanceName()`
- Internal APIs: `dispatch()`, `tryDelivery()`, `deliver()`, `storeNewMessage()`, `updateMetadataOnDisk()`, `readMessageMeta()`, `readDiskQueue()`, `openMessage()`, `removeFromDisk()`, `emitDSN()`, `discardBroken()`, `toSMTPErr()`
- Data structures: `Queue`, `QueueMetadata`, `queueSlot`, `queueDelivery`, `partialError`
- Current documentation: **Missing** — No documentation exists covering retry logic, filesystem model, or scheduling behavior
- Documentation needed: Full behavioral documentation covering retry formula, persistence model, dispatch flow, error classification, and DSN generation

**Module: `internal/target/queue/timewheel.go`**
- Public APIs: `TimeWheel`, `TimeSlot`, `NewTimeWheel()`, `Add()`, `Close()`
- Internal APIs: `tick()` — the core scheduling loop
- Current documentation: **Missing** — No documentation covers the scheduling algorithm
- Documentation needed: Scheduling algorithm documentation including nearest-deadline scan, timer-based dispatch, update notification, and single-goroutine execution model

**Module: `internal/target/remote/remote.go` + `connect.go`**
- Public APIs: `Target` struct, `New()`, `Init()`, `Start()`, `Close()`
- Internal APIs: `connectionForDomain()`, `lookupMX()`, `checkPolicies()`, `remoteDelivery` lifecycle
- Current documentation: **Missing** — No documentation on MX connection behavior during failures
- Documentation needed: Connection attempt sequence, TLS fallback logic, error wrapping for timeouts

**Module: `internal/smtpconn/smtpconn.go`**
- Public APIs: `C` struct, `New()`, `Connect()`, `Mail()`, `Rcpt()`, `Data()`, `Close()`
- Internal APIs: `attemptConnect()`, `wrapClientErr()`
- Current documentation: **Missing** — No documentation on timeout behavior or error classification
- Documentation needed: Dialer timeout defaults, error wrapping logic, TLS negotiation flow

**Module: `internal/log/` (log.go, writer.go, orderedjson.go)**
- Public APIs: `Logger`, `Output`, `Msg()`, `Error()`, `Debugf()`, `Printf()`, `WriterOutput()`, `WriteCloserOutput()`
- Current documentation: **Partial** — Go doc comments exist but no user-facing format specification
- Documentation needed: Complete log output format specification with field structure, timestamp format, JSON serialization rules

**Module: `internal/exterrors/` (temporary.go, smtp.go)**
- Public APIs: `IsTemporaryOrUnspec()`, `IsTemporary()`, `SMTPError`, `WithTemporary()`, `WithFields()`
- Current documentation: **Partial** — Go doc comments describe individual functions
- Documentation needed: Error classification decision tree, how error temporality drives retry behavior

**Configuration: `maddy.conf`**
- Queue configuration directives: `max_tries`, `max_parallelism`, `target remote {}`, `bounce {}`
- Current documentation: Documented in man pages but not linked to behavioral analysis
- Documentation needed: Configuration-to-behavior mapping showing how each directive affects queue operation

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

**Undocumented behaviors (critical to user's questions):**
- Complete message lifecycle from SMTP ingress to final delivery or bounce
- Retry timing table with concrete delay calculations for each attempt
- Exact TCP connection timeout duration when contacting non-responsive hosts
- Structured log field reference for queue-related events
- Queue directory filesystem layout with file naming patterns and JSON metadata schema
- TimeWheel scheduling algorithm and its implications for multi-message dispatch
- Delivery semaphore contention model and queue starvation scenarios
- Post-init delay behavior affecting restart recovery timing
- Partial delivery handling (some recipients succeed, others retry)
- DSN generation conditions and bounce suppression rules

**Missing user guides:**
- No guide for diagnosing queue delivery problems using log analysis
- No guide for inspecting queue state via filesystem examination
- No guide for tuning `max_tries`, `max_parallelism`, and retry timing

**Incomplete architecture documentation:**
- No data flow diagram for the queue → remote → smtpconn delivery chain
- No state machine for message lifecycle in the queue
- No documentation of error classification flow from TCP error to retry decision

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output is a single comprehensive markdown file as required by the SWE-AtlasQnA-Repo implementation rule. The document will be structured as follows:

```
<project_name>.md
├── 1. Message Processing Pipeline Overview
│   ├── 1.1 End-to-End Flow (SMTP Ingress → Queue → Remote Delivery)
│   ├── 1.2 Pipeline Routing (source/destination block matching)
│   └── 1.3 Queue Entry Point (queueDelivery lifecycle)
├── 2. Queue Delivery and Retry Mechanics
│   ├── 2.1 Retry Formula and Backoff Calculation
│   ├── 2.2 Connection Attempt Sequence for Non-Responsive Destinations
│   ├── 2.3 TCP Timeout Duration Analysis
│   ├── 2.4 Error Classification: Temporary vs Permanent
│   └── 2.5 Retry Timing Table (for max_tries = 2, 3, 8)
├── 3. Log Entry Analysis
│   ├── 3.1 Log Output Format Specification
│   ├── 3.2 Log Entries Per Retry Attempt (fields, timestamps, errors)
│   ├── 3.3 Debug vs Standard Log Levels
│   └── 3.4 Sample Log Sequences for Failure Scenarios
├── 4. Queue Filesystem and Persistence
│   ├── 4.1 Queue Directory Layout
│   ├── 4.2 File Naming Convention
│   ├── 4.3 Metadata JSON Schema (QueueMetadata)
│   ├── 4.4 Inspecting Queue State Between Failures
│   └── 4.5 Atomic Update and Crash Recovery
├── 5. Multi-Message Queue Scheduling
│   ├── 5.1 TimeWheel Scheduling Algorithm
│   ├── 5.2 Delivery Semaphore and Parallelism
│   ├── 5.3 Queue Starvation Analysis
│   └── 5.4 Impact of Slow Timeouts on Other Messages
├── 6. Experimental Setup Guide
│   ├── 6.1 Test Configuration for Intentionally Failing Destinations
│   ├── 6.2 Observing Queue Behavior Without Codebase Modification
│   └── 6.3 Filesystem Inspection Commands
└── 7. Source Code References
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract retry formula and timing constants from `internal/target/queue/queue.go` lines 122–126 (field declarations) and lines 183–188 (`NewQueue` defaults)
- Extract dispatch flow from `queue.go` `dispatch()` (lines 275–323) and `tryDelivery()` (lines 365–429)
- Extract log message templates from all `Log.Msg()`, `Log.Error()`, `Log.Debugf()`, `Log.Debugln()` calls in `queue.go` and `remote.go`
- Extract connection timeout behavior from `smtpconn.go` `New()` (line 57–63) where `net.Dialer` uses zero-value timeouts
- Extract scheduling algorithm from `timewheel.go` `tick()` (lines 71–128)
- Generate examples by analyzing `queue_test.go` patterns (e.g., `TestQueueDelivery_TemporaryFail`, `TestQueueDelivery_SerializationRoundtrip`)
- Create diagrams by mapping component relationships: Queue → TimeWheel → dispatch → deliver → remote.Target → smtpconn.C

**Documentation Standards:**
- Markdown formatting with proper headers (# ## ### ####)
- Mermaid diagrams for pipeline flow, state machine, and scheduling visualization
- Code examples using Go syntax highlighting for configuration and metadata
- Source citations as inline references: `Source: internal/target/queue/queue.go:414`
- Tables for retry timing calculations and log field specifications
- Consistent terminology aligned with the codebase: "delivery attempt", "retry", "temporary failure", "permanent failure", "dispatch", "slot"

### 0.4.3 Diagram and Visual Strategy

**Mermaid diagrams to create:**

- **Sequence diagram:** End-to-end message flow from SMTP session through pipeline routing to queue persistence and remote delivery attempt
- **State machine:** Queued message lifecycle from `Commit()` through retry cycles to final delivery or DSN generation (adapting the existing tech spec state diagram with timeout-specific states)
- **Flowchart:** Error classification decision tree from TCP `net.OpError` through `wrapClientErr()` and `toSMTPErr()` to `IsTemporaryOrUnspec()` determining retry eligibility
- **Flowchart:** TimeWheel scheduling loop showing nearest-deadline scan, timer wait, update notification, and dispatch callback
- **Timeline diagram:** Concrete retry timing for `max_tries=3` with 15-minute base showing TCP timeout periods interleaved with backoff delays
- **Flowchart:** Multi-message dispatch showing semaphore acquisition, goroutine spawning, and potential starvation when delivery goroutines block on TCP timeouts

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

The SWE-AtlasQnA-Repo implementation rule requires writing the complete answer as a single markdown file. No files in the source repository are modified. The transformation mapping is as follows:

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `<project_name>.md` | CREATE | `internal/target/queue/queue.go`, `internal/target/queue/timewheel.go`, `internal/target/remote/remote.go`, `internal/target/remote/connect.go`, `internal/smtpconn/smtpconn.go`, `internal/log/log.go`, `internal/log/writer.go`, `internal/log/orderedjson.go`, `internal/exterrors/smtp.go`, `internal/exterrors/temporary.go`, `internal/target/delivery.go`, `internal/dsn/dsn.go`, `internal/module/delivery_target.go`, `internal/module/partial_delivery.go`, `internal/msgpipeline/msgpipeline.go`, `internal/buffer/file.go`, `maddy.conf`, `HACKING.md` | Complete technical investigation of maddy's message processing pipeline, queue retry mechanics, timeout behavior, log format analysis, filesystem persistence model, multi-message scheduling, and queue starvation analysis |

**No files in the source repository are modified, updated, or deleted.**

### 0.5.2 New Documentation File Detail

```
File: <project_name>.md
Type: Technical investigation / Internals deep-dive
Source Code: 18 source files across internal/target/queue/,
             internal/target/remote/, internal/smtpconn/,
             internal/log/, internal/exterrors/, internal/module/,
             internal/msgpipeline/, internal/dsn/, internal/buffer/
Sections:
    - Message Processing Pipeline Overview
      (from internal/msgpipeline/msgpipeline.go, internal/endpoint/smtp/)
    - Queue Delivery and Retry Mechanics
      (from internal/target/queue/queue.go lines 365-429, 112-147)
    - Connection Attempt Sequence
      (from internal/target/remote/connect.go lines 113-225)
    - TCP Timeout Duration Analysis
      (from internal/smtpconn/smtpconn.go lines 57-63, 122-133)
    - Error Classification Decision Tree
      (from internal/exterrors/temporary.go, internal/exterrors/smtp.go)
    - Retry Timing Calculations
      (from queue.go line 414: initialRetryTime * retryTimeScale^(TriesCount-1))
    - Log Entry Format Specification
      (from internal/log/log.go, writer.go, orderedjson.go)
    - Log Entries Per Retry Attempt
      (from queue.go tryDelivery(), deliver(), dispatch())
    - Queue Directory Layout and File Naming
      (from queue.go storeNewMessage(), readDiskQueue())
    - Metadata JSON Schema
      (from queue.go QueueMetadata struct lines 149-170)
    - Atomic Update and Crash Recovery
      (from queue.go updateMetadataOnDisk() lines 742-767)
    - TimeWheel Scheduling Algorithm
      (from timewheel.go tick() lines 71-128)
    - Delivery Semaphore Model
      (from queue.go deliverySemaphore, dispatch() lines 281-299)
    - Queue Starvation Analysis
      (from interaction of timewheel.go dispatch blocking, semaphore, and net.Dialer timeouts)
    - Experimental Setup Guide
      (test configuration examples without repository modification)
    - Source Code References
      (comprehensive citations to all analyzed files)
Diagrams:
    - End-to-end message flow sequence diagram
    - Queued message lifecycle state machine
    - Error classification flowchart
    - TimeWheel scheduling algorithm flowchart
    - Multi-message dispatch concurrency diagram
    - Retry timing timeline
Key Citations:
    internal/target/queue/queue.go,
    internal/target/queue/timewheel.go,
    internal/target/remote/remote.go,
    internal/target/remote/connect.go,
    internal/smtpconn/smtpconn.go,
    internal/log/log.go,
    internal/log/writer.go,
    internal/log/orderedjson.go,
    internal/exterrors/smtp.go,
    internal/exterrors/temporary.go,
    internal/target/delivery.go,
    internal/dsn/dsn.go,
    internal/module/delivery_target.go,
    internal/module/partial_delivery.go,
    internal/msgpipeline/msgpipeline.go,
    internal/buffer/file.go,
    maddy.conf,
    HACKING.md
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The output is a standalone markdown file that does not integrate into the maddy MkDocs documentation site. The `.mkdocs.yml` navigation tree is not modified.

### 0.5.4 Cross-Documentation Dependencies

- The new document references source code files but does not link to existing documentation pages
- No navigation, table of contents, or index updates are needed in the source repository
- The document is self-contained with all necessary context embedded inline

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following runtime and build dependencies are relevant to the documented behavior:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| go modules | `github.com/foxcpp/maddy` | v0.0.0 (development) | Main mail server binary — queue, remote, and pipeline modules |
| go modules | `github.com/emersion/go-smtp` | v0.12.1-0.20191206174923 | SMTP client/server library — underlying connection transport |
| go modules | `github.com/emersion/go-message` | v0.10.9-0.20191116124005 | Message parsing — textproto header reading/writing |
| go modules | `github.com/google/uuid` | v1.1.1 | UUID generation for message IDs |
| go modules | `github.com/foxcpp/go-imap-sql` | v0.3.2-0.20191208094750 | SQL-backed IMAP storage for local mailbox delivery |
| go modules | `github.com/mattn/go-sqlite3` | v1.11.0 | SQLite3 driver for local storage (requires CGO) |
| go modules | `github.com/miekg/dns` | v1.1.22 | DNS library for MX lookups and DNSSEC verification |
| go modules | `golang.org/x/net` | v0.0.0-20191126235420 | Extended networking — IDNA support for internationalized domains |
| go modules | `golang.org/x/crypto` | v0.0.0-20191108234033 | Cryptographic primitives for TLS and DKIM |
| go | Go toolchain | 1.13+ (go.mod minimum) | Build toolchain — net.Dialer, crypto/tls, encoding/json |
| system | MkDocs | (configured in .mkdocs.yml) | Documentation site generator for existing docs |
| system | scdoc | (optional, referenced in docs/man/) | Man page generation from scdoc format |

**Key dependency details relevant to documented behavior:**

- `github.com/emersion/go-smtp` v0.12.1 — Provides the `smtp.Client` type that `smtpconn.C` wraps. The SMTP error type `smtp.SMTPError` is the foundation for the error model in the queue system
- `net.Dialer` (Go stdlib) — Used with zero-value timeout in `smtpconn.New()`, inheriting OS TCP connection timeout defaults (typically ~2 minutes on Linux, controlled by `/proc/sys/net/ipv4/tcp_syn_retries`)
- `encoding/json` (Go stdlib) — Used for `QueueMetadata` serialization to `.meta` files and for structured log field serialization in `marshalOrderedJSON()`
- `container/list` (Go stdlib) — Used by `TimeWheel` for the doubly-linked list of pending delivery slots

### 0.6.2 Documentation Reference Updates

No documentation reference or link updates are required. The output document is standalone and does not modify any existing links in the source repository.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis:**
- Queue retry mechanics documented: 0/7 key behaviors (0%)
- Log format specification documented: 0/4 log event types (0%)
- Queue filesystem layout documented: 0/3 file types (0%)
- TimeWheel scheduling documented: 0/4 algorithm phases (0%)
- Error classification paths documented: 0/5 error types (0%)
- Multi-message scheduling documented: 0/3 concurrency aspects (0%)
- Experimental setup guides: 0/3 scenarios (0%)

**Target coverage: 100%** — Every question posed in the user's requirements must have a precise, source-code-backed answer.

**Coverage gaps to address:**

| Topic | Current | Target | Focus Areas |
|-------|---------|--------|-------------|
| Message pipeline flow | 0% | 100% | SMTP ingress → pipeline routing → queue entry → remote delivery |
| Retry formula and timing | 0% | 100% | Backoff calculation, timing tables for max_tries 2/3/8 |
| TCP timeout behavior | 0% | 100% | net.Dialer defaults, OS-level TCP SYN retries, observed durations |
| Log entry format | 0% | 100% | Timestamp format, JSON field structure, per-event field catalog |
| Queue filesystem model | 0% | 100% | .header/.body/.meta files, naming patterns, JSON metadata schema |
| Multi-message scheduling | 0% | 100% | TimeWheel algorithm, semaphore model, starvation analysis |
| Experimental verification | 0% | 100% | Test configurations, observation commands, filesystem inspection |

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every user question must have a direct, unambiguous answer with source code citations
- All retry timing values must be calculated using the actual formula from `queue.go` line 414
- All log field names must be extracted from actual `Log.Msg()` and `Log.Error()` calls
- All filesystem paths must use the actual naming pattern from `storeNewMessage()` and `updateMetadataOnDisk()`
- All error classification paths must trace from the raw Go error type through `wrapClientErr()` and `IsTemporaryOrUnspec()` to the retry/fail decision

**Accuracy validation:**
- Every claim about behavior must reference a specific source file and line range
- Retry timing calculations must be verified against the formula: `initialRetryTime * retryTimeScale ^ (TriesCount - 1)` with defaults `15min * 2^(n-1)`
- Log format examples must match the actual output of `marshalOrderedJSON()` with sorted keys
- Error code mapping must match `toSMTPErr()` logic (lines 325–363): temporary → 451/4.0.0, permanent → 554/5.0.0
- Queue metadata JSON must match the `QueueMetadata` struct fields (lines 149–170)

**Clarity standards:**
- Technical accuracy with precise code references
- Progressive disclosure: overview first, then detailed analysis
- Concrete examples: actual log lines, actual file contents, actual timing values
- Consistent terminology from the codebase (`tryDelivery`, `dispatch`, `TimeSlot`, `deliverySemaphore`)

**Maintainability:**
- Source citations for every technical claim enable future verification
- Structured sections allow readers to jump to specific topics
- Calculation tables enable readers to recompute timing for their own configurations

### 0.7.3 Example and Diagram Requirements

- Minimum 1 Mermaid diagram per major section (6 diagrams total)
- Minimum 1 concrete log output example per retry attempt phase
- Minimum 1 complete metadata JSON example showing queue state between failures
- Minimum 1 retry timing table per `max_tries` scenario (2, 3, 8)
- Minimum 1 test configuration example for experimental verification
- All code examples must use Go or YAML/TOML syntax highlighting as appropriate

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `<project_name>.md` — Single comprehensive investigation document (the only output artifact)

**Source code analyzed for documentation (read-only, not modified):**
- `internal/target/queue/queue.go` — Queue delivery lifecycle, retry logic, persistence, DSN generation
- `internal/target/queue/timewheel.go` — Time-based scheduling algorithm
- `internal/target/queue/queue_test.go` — Behavioral test patterns demonstrating retry and persistence
- `internal/target/remote/remote.go` — Remote delivery target initialization, recipient processing, body delivery
- `internal/target/remote/connect.go` — MX lookup, host connection, TLS policy, failover iteration
- `internal/smtpconn/smtpconn.go` — SMTP connection wrapper, dialer, error wrapping
- `internal/msgpipeline/msgpipeline.go` — Pipeline routing orchestration
- `internal/log/log.go` — Logger type, Msg(), Error(), Debugf() methods
- `internal/log/writer.go` — Timestamp formatting, debug prefix
- `internal/log/orderedjson.go` — JSON field serialization with sorted keys
- `internal/exterrors/temporary.go` — Temporary/permanent error classification
- `internal/exterrors/smtp.go` — SMTPError type, Fields() method, Temporary() method
- `internal/exterrors/fields.go` — Structured error field extraction
- `internal/target/delivery.go` — DeliveryLogger helper (msg_id field injection)
- `internal/dsn/dsn.go` — DSN bounce message generation
- `internal/module/delivery_target.go` — DeliveryTarget and Delivery interfaces
- `internal/module/partial_delivery.go` — PartialDelivery and StatusCollector interfaces
- `internal/module/msgmetadata.go` — MsgMetadata data model
- `internal/buffer/file.go` — FileBuffer for disk-backed message bodies
- `internal/buffer/buffer.go` — Buffer interface contract
- `internal/config/directories.go` — StateDirectory, RuntimeDirectory variables
- `internal/limiters/concurrency.go` — Semaphore primitives
- `maddy.conf` — Default server configuration with queue settings
- `HACKING.md` — Design guide and error handling conventions
- `.mkdocs.yml` — Documentation site configuration

**Topics covered in documentation:**
- End-to-end message processing pipeline from SMTP ingress to final delivery
- Queue retry mechanics with exponential backoff formula
- TCP connection timeout behavior for non-responsive SMTP destinations
- SMTP error classification and retry eligibility determination
- Structured log output format with field specifications
- Log entries produced during each delivery attempt and retry
- Queue filesystem layout: `.header`, `.body`, `.meta` file naming and contents
- `QueueMetadata` JSON schema with all fields documented
- Atomic metadata update strategy (`.meta.new` → rename)
- Post-init delay behavior for restart recovery
- TimeWheel scheduling algorithm (nearest-deadline scan, single-goroutine dispatch)
- Delivery semaphore concurrency control (`max_parallelism`)
- Multi-message scheduling priority (earliest deadline first)
- Queue starvation analysis under slow TCP timeouts
- Partial delivery semantics (per-recipient failure tracking)
- DSN generation conditions and bounce suppression rules
- Experimental setup with test configurations and failing destinations

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — No files in the maddy repository are modified, added, or deleted (per user instruction and SWE-AtlasQnA-Repo rule)
- **Test file modifications** — No test files are altered
- **IMAP storage internals** — The IMAP backend (`internal/storage/`) is not part of the queue/delivery investigation
- **DKIM/SPF/DMARC check internals** — Security check pipeline (`internal/check/`) is mentioned only as context for the message pipeline, not documented in detail
- **Authentication backend** — The SQL auth provider (`internal/auth/`) is outside scope
- **Man page updates** — No man pages in `docs/man/` are modified
- **MkDocs site updates** — The `.mkdocs.yml` navigation is not changed
- **CI/CD configuration** — Build pipeline (`.build.yml`) is not modified
- **Deployment configuration** — systemd units, fail2ban rules (`dist/`) are not modified
- **Feature additions or code refactoring** — Strictly documentation-only exercise
- **LMTP endpoint behavior** — Only SMTP inbound and submission endpoints are in scope
- **Modifier internals** — The modify/ package is mentioned in pipeline context but not documented in depth
- **Third-party SMTP server behavior** — Only maddy's own behavior is documented; remote MTA behavior is out of scope

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

**Documentation build command:** Not applicable — output is a standalone markdown file, not integrated into MkDocs

**Documentation preview command:** Any markdown renderer (e.g., `grip <project_name>.md`, VS Code markdown preview, or GitHub rendering)

**Diagram generation:** Mermaid diagrams are embedded directly in the markdown using fenced code blocks (```` ```mermaid ... ``` ````). They render natively on GitHub and in Mermaid-compatible viewers.

**Default format:** Markdown with Mermaid diagrams

**Citation requirement:** Every technical claim must reference a specific source file and, where applicable, line numbers. Format: `Source: internal/target/queue/queue.go:414`

**Style guide:** Follow the existing maddy documentation conventions observed in `docs/internals/quirks.md` and `docs/internals/sqlite.md`:
- Clear section headings with hierarchical structure
- Concise technical prose with specific references
- Code blocks for configuration examples and metadata samples
- Tables for structured data (timing calculations, field specifications)

**Documentation validation:** Manual review against source code. Each claim must be verifiable by reading the cited source file at the cited line range.

### 0.9.2 Build and Runtime Environment

**Go toolchain:** Go 1.13+ (as specified in `go.mod` line 3). The installed environment has Go 1.22.2 which satisfies this requirement.

**C compiler requirement:** Required for `github.com/mattn/go-sqlite3` CGO dependency. This is needed only if building and running the maddy binary for experimental verification.

**Operating system defaults relevant to timeout behavior:**
- Linux TCP SYN retries: controlled by `/proc/sys/net/ipv4/tcp_syn_retries` (default: 6, resulting in ~127 second timeout)
- Go `net.Dialer` with zero-value `Timeout` field: delegates to OS kernel TCP timeout
- These system-level defaults are documented in the output because they directly affect the observable timeout duration during delivery attempts to non-responsive destinations

### 0.9.3 Key Configuration Parameters for Analysis

The following configuration values from `maddy.conf` and `queue.go` defaults drive the documented behavior:

| Parameter | Default Value | Source | Effect on Documented Behavior |
|-----------|--------------|--------|-------------------------------|
| `max_tries` | 8 | `queue.go` line 204 | Maximum delivery attempts per message |
| `max_parallelism` | 16 | `queue.go` line 205 | Delivery semaphore size — max concurrent deliveries |
| `initialRetryTime` | 15 minutes | `queue.go` line 185 | Base retry delay before exponential scaling |
| `retryTimeScale` | 2.0 | `queue.go` line 186 | Exponential backoff multiplier |
| `postInitDelay` | 10 seconds | `queue.go` line 187 | Minimum delay for deliveries scheduled close to startup |
| `smtpPort` | "25" | `remote.go` line 41 | Target SMTP port for remote delivery |
| `net.Dialer.Timeout` | 0 (zero-value) | `smtpconn.go` line 59 | TCP connection timeout — delegates to OS default |
| `hostname` | (required) | `maddy.conf` line 4 | Used in EHLO greeting and DSN generation |
| `autogenerated_msg_domain` | (required for bounce) | `maddy.conf` line 28 | Domain for auto-generated DSN messages |

## 0.10 Rules for Documentation

### 0.10.1 User-Specified Rules

The following rules are explicitly specified by the user and must be strictly followed:

- **"Avoid modifying the codebase."** — No files in the maddy source repository may be edited, added, or deleted. The repository must remain in its original state after the documentation exercise is complete.
- **"You can create test configurations and intentionally failing destinations, but repository files should remain unchanged."** — External test configurations (e.g., alternative `maddy.conf` files in temporary directories) may be created for experimental observation, but these must not be placed inside the repository tree.
- **"Focus on what actually happens during execution, not theoretical behavior."** — Every documented behavior must be grounded in actual code path analysis with source citations. Speculative or hypothetical behavior is not acceptable.
- **"Use the blitzy-research/AtlasQnA repository as the destination."** — The output document is delivered to the AtlasQnA repository, not to the maddy repository.
- **"Write your complete answer as a single markdown file named `<project_name>.md`."** — All documentation is consolidated into one file. No multi-file documentation structure is used.
- **"Do not modify any files in the source repository."** — Reinforces the read-only constraint on the maddy codebase.

### 0.10.2 Derived Documentation Rules

Based on the nature of the task and codebase analysis, the following rules govern documentation quality:

- **Source code citations are mandatory** — Every technical claim about maddy's behavior must reference a specific file path and line range (e.g., `queue.go:414`)
- **Use exact function and variable names from the codebase** — Do not paraphrase or rename code constructs. Use `tryDelivery()`, `TimeWheel`, `deliverySemaphore`, `QueueMetadata`, `IsTemporaryOrUnspec()`, etc.
- **Calculate concrete timing values** — Do not leave retry delays as formulas alone. Compute and present actual durations for each retry attempt under specified `max_tries` values.
- **Show exact log output format** — Reproduce the structured log format as it would appear in stderr output, including timestamp format, logger name prefix, message text, and JSON field structure.
- **Show exact metadata JSON** — Reproduce the `QueueMetadata` JSON as it would appear in a `.meta` file on disk after serialization via `json.NewEncoder().Encode()`.
- **Document observable vs inferred behavior** — Clearly distinguish between behavior directly observable in logs/filesystem and behavior inferred from code analysis.
- **Include experimental reproduction steps** — For each behavioral claim, describe how an operator could verify it using log inspection, filesystem examination, or test configuration.

## 0.11 References

### 0.11.1 Source Files Analyzed

The following files and folders were searched and analyzed across the codebase to derive all conclusions in this Agent Action Plan:

**Root-level files:**
| File | Purpose | Key Findings |
|------|---------|--------------|
| `go.mod` | Go module definition, minimum toolchain version | Go 1.13+, module path `github.com/foxcpp/maddy`, key deps: go-smtp v0.12.1, go-message v0.10.9, miekg/dns v1.1.22 |
| `maddy.conf` | Default server configuration | Queue configured as `remote_queue` with `max_tries 8`, `max_parallelism 16`, `target remote { authenticate_mx mtasts dnssec }`, bounce pipeline with local-only delivery |
| `HACKING.md` | Contributor design guide | Module architecture, error handling conventions, exterrors usage guidelines |
| `.mkdocs.yml` | MkDocs documentation configuration | ReadTheDocs theme, codehilite extension, nav structure with tutorials and internals |
| `README.md` | Project overview | All-in-one mail server description, feature list |
| `.editorconfig` | Editor configuration | Whitespace and indentation defaults |
| `maddy.go` | Server bootstrap | Module registration, CLI parsing, startup lifecycle |
| `config.go` | Logging configuration glue | Logger reinitialization logic |

**Queue subsystem (`internal/target/queue/`):**
| File | Lines | Key Findings |
|------|-------|--------------|
| `queue.go` | 957 | Complete queue implementation: Queue struct (lines 112-147), QueueMetadata (149-170), NewQueue defaults (182-199), Init (201-238), dispatch (275-323), tryDelivery (365-429), deliver (431-532), storeNewMessage (690-740), updateMetadataOnDisk (742-767), readDiskQueue (622-688), emitDSN (849-953) |
| `timewheel.go` | 128 | TimeWheel scheduler: TimeSlot struct (10-13), Add (38-53), tick loop (71-128) with nearest-deadline scan, timer wait, updateNotify channel |
| `queue_test.go` | 822 | Behavioral tests: retry patterns, serialization roundtrip, deserialization cleanup, DSN tests, abort semantics |
| `timewheel_test.go` | — | Scheduler unit tests: ordering, wake-up, regression cases |

**Remote delivery (`internal/target/remote/`):**
| File | Lines | Key Findings |
|------|-------|--------------|
| `remote.go` | 486 | Target struct, Init, connectionForDomain, per-domain connection caching, MTA-STS refresh, BodyNonAtomic with per-domain goroutines |
| `connect.go` | 276 | MX lookup, sorted iteration, TLS attempt, policy checking (MTA-STS, DNSSEC, common domain), plaintext fallback, "No usable MXs" error |

**SMTP connection (`internal/smtpconn/`):**
| File | Lines | Key Findings |
|------|-------|--------------|
| `smtpconn.go` | 338 | C struct with net.Dialer (zero-value timeout), Connect → attemptConnect, wrapClientErr (net.OpError → SMTP 450/4.4.2 "Network I/O error"), EHLO, STARTTLS negotiation |

**Logging (`internal/log/`):**
| File | Lines | Key Findings |
|------|-------|--------------|
| `log.go` | 208 | Logger struct (Name, Debug, Fields), Msg() → formatMsg → JSON fields, Error() → exterrors.Fields extraction + reason field, Debugf/Debugln gated by Debug flag |
| `writer.go` | 77 | Timestamp format: `2006-01-02T15:04:05.000Z`, debug prefix: `[debug]`, newline-terminated |
| `orderedjson.go` | 62 | Sorted keys, time.Time → ISO format, time.Duration → String(), error → Error() |

**Error handling (`internal/exterrors/`):**
| File | Lines | Key Findings |
|------|-------|--------------|
| `temporary.go` | 56 | IsTemporaryOrUnspec defaults to true for unclassified errors, IsTemporary defaults to false |
| `smtp.go` | 128 | SMTPError with Code, EnhancedCode, Message, Reason, Misc; Temporary() → Code/100==4; Fields() returns structured map |
| `fields.go` | — | WithFields wrapper, Fields() chain walker merging field maps |

**Pipeline and module interfaces:**
| File | Key Findings |
|------|--------------|
| `internal/msgpipeline/msgpipeline.go` | MsgPipeline struct, two-level source/destination routing, Start → srcBlockForAddr → rcptBlockForAddr |
| `internal/module/delivery_target.go` | DeliveryTarget interface: Start, Delivery interface: AddRcpt, Body, Abort, Commit |
| `internal/module/partial_delivery.go` | PartialDelivery interface: BodyNonAtomic, StatusCollector: SetStatus |
| `internal/module/msgmetadata.go` | MsgMetadata: ID, Quarantine, OriginalRcpts, SMTPOpts, Conn, DeepCopy() |
| `internal/target/delivery.go` | DeliveryLogger: copies logger fields, adds msg_id from MsgMetadata.ID |

**Supporting modules:**
| File/Folder | Key Findings |
|-------------|--------------|
| `internal/buffer/file.go` | FileBuffer: Path + LenHint, Open → os.Open, used for queue body storage |
| `internal/buffer/buffer.go` | Buffer interface: Open, Len, Remove |
| `internal/dsn/dsn.go` | DSN generation: RFC 3464 multipart/report, ReportingMTAInfo, RecipientInfo |
| `internal/config/directories.go` | StateDirectory, RuntimeDirectory variables |
| `internal/limiters/concurrency.go` | Semaphore using buffered channel (same pattern as queue's deliverySemaphore) |

**Existing documentation (`docs/`):**
| File | Key Findings |
|------|--------------|
| `docs/README.md` | Project overview landing page |
| `docs/internals/quirks.md` | SMTP/IMAP protocol quirks, no queue documentation |
| `docs/internals/sqlite.md` | SQLite backend details, no queue documentation |
| `docs/tutorials/setting-up.md` | End-to-end deployment guide |
| `docs/tutorials/manual-installation.md` | Source installation, Go 1.13+ requirement |
| `docs/tutorials/alias-to-remote.md` | Alias-to-remote routing with reroute block |

**Tech spec sections retrieved:**
| Section | Key Findings Used |
|---------|-------------------|
| 4.1 HIGH-LEVEL SYSTEM WORKFLOW | Startup lifecycle, end-to-end message processing overview, system boundary actors |
| 4.3 MESSAGE PIPELINE AND ROUTING | Two-level routing architecture, check runner, modifier chain execution |
| 4.4 DELIVERY AND QUEUE MANAGEMENT | Queue lifecycle state machine, dispatch flow, remote MX delivery, DSN generation |

### 0.11.2 Attachments

No attachments were provided by the user. No Figma screens, design documents, or external files accompany this task.

### 0.11.3 External References

- **maddy repository:** `https://github.com/foxcpp/maddy` — Primary source of all analyzed code
- **RFC 3464** — Delivery Status Notification format standard (referenced by `internal/dsn/dsn.go`)
- **RFC 5321 Section 5.1** — SMTP MX fallback to A/AAAA records (implemented in `connect.go` lines 268–274)
- **RFC 2821** — SMTP protocol (referenced in `docs/internals/quirks.md`)
- **Linux TCP documentation** — `/proc/sys/net/ipv4/tcp_syn_retries` controls TCP connection timeout (relevant to `net.Dialer` with zero-value timeout)

