# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that provides an empirical, code-grounded analysis of maddy mail server's message processing pipeline — specifically focusing on the retry, queuing, and scheduling subsystems as they behave under adverse conditions (non-responsive SMTP destinations with timeout failures).

- **Documentation Type:** Technical investigation document (Q&A-style deep-dive with rationale derived from source code analysis)
- **Category:** Create new documentation
- **Target Output:** A single comprehensive Markdown document named after the project, placed in `blitzy/documentation/`, that answers the user's series of operational questions about maddy's pipeline behavior

The user's requirements decompose into the following specific documentation objectives:

- **Message Processing Pipeline Flow** — Document the exact sequence a message traverses from SMTP ingress through the message pipeline, into the queue, out to remote delivery, covering every component boundary and handoff
- **Retry Sequence Under Timeout** — When `max_tries` is set to a small value (e.g., 2–3) and the SMTP destination is non-responsive, document the exact sequence of connection attempts, including how the `net.Dialer` timeout manifests, how the queue interprets the failure classification (temporary vs. permanent), and the exponential backoff formula `initialRetryTime * retryTimeScale^(TriesCount-1)`
- **Timeout Duration Analysis** — Document the actual timeout durations observed at each layer: the Go `net.Dialer` default timeout, the SMTP client-level timeout, and any context-based deadline propagation
- **Log Entry Anatomy** — Document the exact structured log fields emitted at each retry stage, including the JSON format from `internal/log/orderedjson.go`, the `msg_id` field from `DeliveryLogger`, attempt counts, error reason fields, SMTP code/enhanced code fields, and timestamp formatting (`2006-01-02T15:04:05.000Z`)
- **Queue Filesystem Layout** — Document the on-disk file structure (`.header`, `.body`, `.meta` triplets), naming patterns (message ID as prefix), the atomic metadata update strategy (`.meta.new` → rename), and the JSON contents of `.meta` files including `TriesCount`, `FirstAttempt`, `LastAttempt`, recipient lists, and serialized `SMTPError` objects
- **Queue Scheduler Behavior** — Document how `TimeWheel` (the linked-list-based time-wheel scheduler in `internal/target/queue/timewheel.go`) handles multiple simultaneous messages, its single-goroutine tick loop, nearest-deadline selection, and the interaction with the `deliverySemaphore` (bounded channel of size `max_parallelism`)
- **Cross-Message Retry Independence** — Document whether message A's retry timing is affected when message B to a different destination is blocking on a slow timeout, analyzing the goroutine-per-dispatch model and the semaphore-based parallelism control
- **Queue Starvation Analysis** — Document whether and how starvation can occur when the delivery semaphore is saturated by slow-timeout connections, observable patterns in logs, and the relationship between `max_parallelism` and starvation risk

### 0.1.2 Special Instructions and Constraints

- **CRITICAL: No codebase modifications.** The user explicitly states: "Avoid modifying the codebase." and "repository files should remain unchanged." This is reinforced by the implementation rule: "Do not modify any existing files in the source repository."
- **Test configurations allowed:** "You can create test configurations and intentionally failing destinations" — however, these should be presented as documented examples within the output markdown, not as committed repository changes
- **Focus on actual behavior:** "Focus on what actually happens during execution, not theoretical behavior." — all answers must be grounded in specific source code lines, data structures, and control flow paths
- **Output location:** The generated document must be placed in `blitzy/documentation/` as specified by the `SWE-AtlasQnA-Repo` implementation rule
- **Document naming:** Must be named `<project name>.md` per the implementation rule — thus `maddy.md`
- **Rationale required:** "Provide thinking / rationale behind the answers" — each answer must cite specific source files and line numbers
- **No assumptions:** "Do not make assumptions, base your answers on the code as the truth"

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document the message pipeline flow**, we will create a narrative walkthrough tracing execution from `internal/endpoint/smtp/smtp.go` (Session.Mail, Session.Rcpt, Session.Data) → `internal/msgpipeline/msgpipeline.go` (MsgPipeline.Start, check_runner, routing) → `internal/target/queue/queue.go` (Queue.Start, queueDelivery.Commit) → `internal/target/queue/timewheel.go` (dispatch scheduling) → `internal/target/remote/remote.go` + `connect.go` (MX resolution, SMTP connection) → `internal/smtpconn/smtpconn.go` (actual TCP/SMTP delivery)
- To **document retry behavior**, we will analyze the `tryDelivery()` and `deliver()` methods in `queue.go`, the backoff formula at line 414, the `maxTries` comparison at line 390, and the `TimeWheel.Add()` re-scheduling at lines 420–428
- To **document log entries**, we will trace every `dl.Msg()`, `dl.Error()`, `dl.Debugf()`, and `q.Log.Printf()` call in the queue dispatch path, documenting the structured JSON fields emitted by `internal/log/log.go` and formatted by `internal/log/orderedjson.go`
- To **document filesystem layout**, we will analyze `storeNewMessage()` (lines 690–740), `updateMetadataOnDisk()` (lines 742–767), `readMessageMeta()` (lines 769–791), and `removeFromDisk()` (lines 600–620) in `queue.go`
- To **document scheduler behavior**, we will analyze the `TimeWheel` tick loop (lines 71–128 in `timewheel.go`), the linked-list slot management, and the `deliverySemaphore` channel in the `dispatch()` goroutine
- To **document cross-message independence and starvation**, we will analyze the goroutine-per-dispatch model, the buffered semaphore channel, and the single-goroutine TimeWheel architecture

### 0.1.4 Inferred Documentation Needs

Based on code analysis, additional documentation needs that the answers must address include:

- **DSN/Bounce generation:** The `emitDSN()` method (lines 849–953 in `queue.go`) is invoked when retries are exhausted — this completes the lifecycle story and must be documented
- **Panic recovery in dispatch:** The `recover()` handler in `dispatch()` (lines 284–296) and the `.meta_broken` file renaming mechanism need documentation as they affect what happens to messages during failures
- **Post-init delay:** The `postInitDelay` mechanism (default 10 seconds, lines 129–138 in `queue.go`) that delays deliveries after server restart needs documentation as it affects observed retry timing
- **Partial delivery semantics:** The `PartialDelivery` interface and per-recipient error tracking (lines 498–508) affect how retry decisions are made for multi-recipient messages
- **Error classification chain:** The `exterrors.IsTemporaryOrUnspec()` function and the `toSMTPErr()` conversion (lines 325–363) determine whether a connection timeout results in a retry or permanent failure — critical for understanding the timeout scenario
- **Connection-per-domain caching:** The `remoteDelivery.connections` map (line 182 in `remote.go`) means that multiple recipients at the same domain share a single SMTP connection, affecting how timeouts impact multi-recipient messages


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **MkDocs-based documentation system** with moderate coverage of setup, tutorials, and man pages, but no existing documentation specifically addressing the message processing pipeline's retry mechanics, queue filesystem layout, or scheduler behavior under adverse conditions.

- **Documentation framework:** MkDocs (configured in `.mkdocs.yml`)
  - Theme: `readthedocs`
  - Markdown extensions: `codehilite` (with `guess_lang: false`)
  - Site name: `maddy documentation`
  - Published URL: `https://foxcpp.dev/maddy/`
- **Documentation generator configuration:** `.mkdocs.yml` at repository root
- **Man page tooling:** `scdoc` format sources in `docs/man/`, with `prepare_md.py` converter to Markdown
- **Diagram tools detected:** None in existing documentation infrastructure; Mermaid diagrams are used in the technical specification but not in the repository's own docs
- **Documentation hosting:** External site at `foxcpp.dev/maddy/`, built from MkDocs

**Existing documentation structure:**

| Path | Type | Coverage |
|------|------|----------|
| `docs/README.md` | Landing page | Product overview, features, links to setup |
| `docs/tutorials/setting-up.md` | Tutorial | End-to-end deployment guide |
| `docs/tutorials/manual-installation.md` | Tutorial | Source build and first-run setup |
| `docs/tutorials/alias-to-remote.md` | Tutorial | Alias routing to remote MX |
| `docs/get.sh-script.md` | Reference | Bootstrap installer documentation |
| `docs/internals/sqlite.md` | Internal reference | SQLite backend behavior |
| `docs/internals/quirks.md` | Internal reference | SMTP/IMAP protocol quirks |
| `docs/man/` | Man pages | scdoc-format reference pages |
| `HACKING.md` | Contributor guide | Architecture, module patterns, error handling |
| `README.md` | Project overview | Features, installation, community links |
| `examples/` | Configuration examples | Advanced config samples |

**Key gap identified:** No existing document covers the queue retry mechanism, timewheel scheduler, filesystem storage patterns, or structured log anatomy during delivery failures. The `docs/internals/` folder contains only `sqlite.md` and `quirks.md` — there is no `queue.md`, `pipeline.md`, or `delivery.md`.

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used to identify code requiring documentation for this task:

- **Queue subsystem:** `internal/target/queue/queue.go` (958 lines) — core queue implementation with retry scheduling, disk persistence, DSN generation
- **Queue scheduler:** `internal/target/queue/timewheel.go` (129 lines) — concurrent time-wheel for delayed dispatch
- **Remote delivery:** `internal/target/remote/remote.go` (487 lines) + `connect.go` (277 lines) — MX resolution, connection establishment, TLS policy
- **SMTP connection wrapper:** `internal/smtpconn/smtpconn.go` (339 lines) — outbound SMTP client with error wrapping
- **Message pipeline:** `internal/msgpipeline/msgpipeline.go` — routing orchestrator
- **SMTP endpoint:** `internal/endpoint/smtp/smtp.go` — inbound SMTP session handling
- **Logging subsystem:** `internal/log/log.go`, `orderedjson.go`, `writer.go` — structured log format
- **Error types:** `internal/exterrors/smtp.go`, `temporary.go`, `fields.go` — error classification and structured fields
- **Buffer abstraction:** `internal/buffer/buffer.go` — FileBuffer for on-disk message body
- **Module interfaces:** `internal/module/delivery_target.go`, `msgmetadata.go` — DeliveryTarget, Delivery, MsgMetadata contracts
- **Default configuration:** `maddy.conf` — production-ready config showing queue with `max_tries 8`, `max_parallelism 16`
- **Test coverage:** `internal/target/queue/queue_test.go` (823 lines) — behavioral tests for retry, serialization, DSN, cleanup

**Key directories examined:**

| Directory | Relevance |
|-----------|-----------|
| `internal/target/queue/` | Primary — queue lifecycle, retry, filesystem persistence |
| `internal/target/remote/` | Primary — outbound SMTP connection, MX failover |
| `internal/smtpconn/` | Primary — SMTP connection wrapper, timeout behavior |
| `internal/msgpipeline/` | Supporting — message routing into delivery targets |
| `internal/endpoint/smtp/` | Supporting — message ingress point |
| `internal/log/` | Supporting — log format, structured JSON output |
| `internal/exterrors/` | Supporting — error classification (temp vs. perm) |
| `internal/limiters/` | Supporting — semaphore and rate limiting |
| `internal/future/` | Supporting — async result primitive (MTA-STS fetching) |
| `internal/dsn/` | Supporting — bounce message generation |
| `internal/module/` | Reference — interface contracts |

### 0.2.3 Web Search Research Conducted

No external web research was required for this task. All necessary information was derived directly from the source code analysis, which is consistent with the user's directive to "base your answers on the code as the truth" and avoid assumptions. The codebase itself provides complete information about:

- Queue retry mechanics (exponential backoff formula, max_tries logic)
- Filesystem storage format (JSON metadata, textproto headers, raw body)
- Structured log format (ordered JSON with deterministic field ordering)
- Scheduler algorithm (linked-list time-wheel with nearest-deadline selection)
- Concurrency model (goroutine-per-dispatch, channel-based semaphore)
- Timeout behavior (Go net.Dialer defaults, SMTP-level error wrapping)


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require detailed documentation for the user's questions, mapped to their specific public APIs and the documentation needed:

- **Module: `internal/target/queue/queue.go`**
  - Public APIs: `NewQueue()`, `Queue.Init()`, `Queue.Start()`, `Queue.Close()`, `queueDelivery.AddRcpt()`, `queueDelivery.Body()`, `queueDelivery.Commit()`, `queueDelivery.Abort()`
  - Internal methods requiring documentation: `tryDelivery()`, `deliver()`, `dispatch()`, `storeNewMessage()`, `updateMetadataOnDisk()`, `readMessageMeta()`, `readDiskQueue()`, `removeFromDisk()`, `openMessage()`, `emitDSN()`, `toSMTPErr()`
  - Data structures: `Queue`, `QueueMetadata`, `queueSlot`, `partialError`
  - Current documentation: **Missing** — no dedicated documentation exists for queue behavior
  - Documentation needed: Complete lifecycle narrative, retry formula, filesystem layout, metadata JSON schema, log entries at each phase

- **Module: `internal/target/queue/timewheel.go`**
  - Public APIs: `NewTimeWheel()`, `TimeWheel.Add()`, `TimeWheel.Close()`
  - Internal methods: `tick()` — the core scheduling loop
  - Data structures: `TimeSlot`, `TimeWheel`
  - Current documentation: **Missing**
  - Documentation needed: Scheduler algorithm explanation, nearest-deadline selection, update notification, dispatch coordination

- **Module: `internal/target/remote/remote.go` + `connect.go`**
  - Public APIs: `Target.Start()`, `remoteDelivery.AddRcpt()`, `remoteDelivery.Body()`, `remoteDelivery.BodyNonAtomic()`, `remoteDelivery.Commit()`, `remoteDelivery.Abort()`
  - Internal methods: `connectionForDomain()`, `lookupMX()`, `checkPolicies()`
  - Current documentation: **Missing** from internals docs
  - Documentation needed: MX connection flow, per-domain connection caching, TLS/auth policy cascade, failover behavior

- **Module: `internal/smtpconn/smtpconn.go`**
  - Public APIs: `New()`, `C.Connect()`, `C.Mail()`, `C.Rcpt()`, `C.Data()`, `C.Close()`
  - Internal methods: `attemptConnect()`, `wrapClientErr()`
  - Current documentation: **Missing**
  - Documentation needed: Connection establishment, error wrapping logic, timeout sources

- **Module: `internal/log/` (log.go, orderedjson.go, writer.go)**
  - Public APIs: `Logger.Msg()`, `Logger.Error()`, `Logger.Debugf()`, `Logger.Printf()`
  - Current documentation: **Missing** from user docs
  - Documentation needed: Structured log format anatomy, field ordering, timestamp format, JSON structure

- **Module: `internal/exterrors/` (smtp.go, temporary.go, fields.go)**
  - Public APIs: `SMTPError`, `IsTemporaryOrUnspec()`, `IsTemporary()`, `WithFields()`, `Fields()`
  - Current documentation: Partially covered in `HACKING.md`
  - Documentation needed: Error classification chain, how timeout errors are classified as temporary

- **Configuration options requiring documentation:**
  - `maddy.conf` queue block: `max_tries` (default: 8), `max_parallelism` (default: 16), `target remote`, `bounce`
  - Queue internal defaults: `initialRetryTime` (15 minutes), `retryTimeScale` (2.0), `postInitDelay` (10 seconds)

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented queue internals:** No existing document describes the retry backoff formula, filesystem storage patterns, or the timewheel scheduler algorithm. The `docs/internals/` folder has only `sqlite.md` and `quirks.md`
- **Missing log format reference:** No document describes the structured JSON log format, field names, or how to parse queue-related log entries
- **Missing pipeline trace:** No end-to-end walkthrough exists showing a message's journey from SMTP ingress through queue to remote delivery failure and retry
- **Missing scheduler documentation:** The `TimeWheel` implementation and its single-goroutine dispatch model are completely undocumented
- **Missing concurrency analysis:** No documentation addresses how `max_parallelism`, the delivery semaphore, and goroutine-per-dispatch interact under high load or timeout conditions
- **Missing configuration deep-dive:** While `maddy.conf` shows the queue config surface, internal defaults (`initialRetryTime`, `retryTimeScale`, `postInitDelay`) are only visible in source code
- **Missing error classification reference:** The chain from `net.OpError` → `wrapClientErr()` → `toSMTPErr()` → `IsTemporaryOrUnspec()` that determines retry behavior is undocumented outside of `HACKING.md`'s brief mention


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document will be a single comprehensive Markdown file following the structure below:

```
blitzy/documentation/
└── maddy.md
    ├── Introduction and Scope
    ├── Q1: End-to-End Message Processing Pipeline
    │   ├── SMTP Ingress (endpoint/smtp)
    │   ├── Message Pipeline Routing (msgpipeline)
    │   ├── Queue Acceptance and Disk Persistence (target/queue)
    │   ├── Remote Delivery Dispatch (target/remote)
    │   └── SMTP Connection Layer (smtpconn)
    ├── Q2: Retry Sequence with Small max_tries and Timeout
    │   ├── Configuration Setup
    │   ├── First Attempt — Connection Timeout
    │   ├── Error Classification Chain
    │   ├── Backoff Calculation
    │   └── Subsequent Attempts and Final Failure
    ├── Q3: Timeout Duration Analysis
    │   ├── Go net.Dialer Default Timeout
    │   ├── SMTP Client-Level Timeouts
    │   └── Context Propagation
    ├── Q4: Log Entry Anatomy During Retries
    │   ├── Structured Log Format
    │   ├── Log Entries Per Delivery Phase
    │   └── Example Log Sequences
    ├── Q5: Queue Filesystem Layout
    │   ├── File Naming Convention
    │   ├── .meta JSON Schema
    │   ├── .header and .body Contents
    │   ├── Atomic Metadata Updates
    │   └── Filesystem State Between Retries
    ├── Q6: Queue Scheduler and Multi-Message Prioritization
    │   ├── TimeWheel Architecture
    │   ├── Nearest-Deadline Selection
    │   ├── Delivery Semaphore
    │   └── Multi-Message Scheduling Behavior
    ├── Q7: Cross-Message Retry Independence
    │   ├── Goroutine-Per-Dispatch Model
    │   ├── Semaphore Saturation Scenario
    │   └── Timing Independence Analysis
    ├── Q8: Queue Starvation Analysis
    │   ├── Starvation Conditions
    │   ├── Observable Patterns in Logs
    │   └── Mitigation via max_parallelism
    └── Source Citations
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract queue lifecycle details from `internal/target/queue/queue.go`, tracing every function call in the `Commit → dispatch → tryDelivery → deliver` chain
- Extract retry backoff formula from line 414 of `queue.go`: `nextTryTime.Add(q.initialRetryTime * time.Duration(math.Pow(q.retryTimeScale, float64(meta.TriesCount-1))))`
- Extract filesystem storage patterns from `storeNewMessage()` (lines 690–740) and `updateMetadataOnDisk()` (lines 742–767)
- Extract log entry formats from `internal/log/log.go` (Msg, Error methods) and `orderedjson.go` (JSON field formatting)
- Extract error classification chain from `internal/exterrors/temporary.go` (`IsTemporaryOrUnspec`) and `smtpconn.go` (`wrapClientErr`)
- Extract scheduler behavior from `internal/target/queue/timewheel.go` (tick loop, lines 71–128)
- Generate examples by analyzing test patterns in `internal/target/queue/queue_test.go`
- Create diagrams by mapping component relationships across `queue.go`, `timewheel.go`, `remote.go`, `connect.go`, and `smtpconn.go`

**Documentation Standards Applied:**

- Markdown formatting with proper headers (`#`, `##`, `###`)
- Mermaid diagrams for pipeline flow, state machines, and scheduler algorithm
- Code examples using fenced blocks with Go syntax highlighting
- Source citations as inline references: `Source: internal/target/queue/queue.go:414`
- Tables for parameter descriptions, configuration options, and log field inventories
- Consistent terminology aligned with codebase naming (e.g., "delivery semaphore" not "concurrency limiter")

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create within the output document:

- **End-to-end pipeline flowchart:** SMTP endpoint → MsgPipeline → Queue → Remote Target → SmtpConn → external MTA
- **Queue state machine:** Persisted → Scheduled → Dispatching → Delivering → [Success/TempFail/PermFail/Exhausted]
- **TimeWheel tick loop flowchart:** Scan slots → find nearest → wait timer/update/stop → dispatch or recalculate
- **Retry timeline sequence diagram:** Showing attempt #1, backoff, attempt #2, backoff, final failure with timestamps
- **Filesystem layout diagram:** ASCII art showing `.header`, `.body`, `.meta` files per message ID
- **Semaphore saturation diagram:** Showing goroutines blocked on semaphore while timeout connections hold slots


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/maddy.md` | CREATE | `internal/target/queue/queue.go`, `internal/target/queue/timewheel.go`, `internal/target/remote/remote.go`, `internal/target/remote/connect.go`, `internal/smtpconn/smtpconn.go`, `internal/msgpipeline/msgpipeline.go`, `internal/endpoint/smtp/smtp.go`, `internal/log/log.go`, `internal/log/orderedjson.go`, `internal/log/writer.go`, `internal/exterrors/smtp.go`, `internal/exterrors/temporary.go`, `internal/exterrors/fields.go`, `internal/module/delivery_target.go`, `internal/module/msgmetadata.go`, `internal/buffer/buffer.go`, `internal/dsn/dsn.go`, `internal/target/delivery.go`, `internal/limiters/concurrency.go`, `internal/future/future.go`, `maddy.conf`, `HACKING.md`, `internal/target/queue/queue_test.go` | Comprehensive Q&A document answering all user questions about message processing pipeline, retry mechanics, queue filesystem layout, log entry anatomy, scheduler behavior, cross-message independence, and starvation analysis — grounded entirely in source code with citations |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/maddy.md
Type: Technical Investigation Document (Q&A with code-grounded rationale)
Source Code Files:
    - internal/target/queue/queue.go (primary — queue lifecycle, retry, persistence)
    - internal/target/queue/timewheel.go (primary — scheduler algorithm)
    - internal/target/remote/remote.go (primary — remote delivery target)
    - internal/target/remote/connect.go (primary — MX connection, failover)
    - internal/smtpconn/smtpconn.go (primary — SMTP connection wrapper)
    - internal/msgpipeline/msgpipeline.go (supporting — pipeline routing)
    - internal/endpoint/smtp/smtp.go (supporting — SMTP ingress)
    - internal/log/log.go (supporting — structured logging API)
    - internal/log/orderedjson.go (supporting — JSON serialization)
    - internal/log/writer.go (supporting — timestamp formatting)
    - internal/exterrors/smtp.go (supporting — SMTPError type)
    - internal/exterrors/temporary.go (supporting — error classification)
    - internal/exterrors/fields.go (supporting — structured error fields)
    - internal/module/delivery_target.go (reference — DeliveryTarget interface)
    - internal/module/msgmetadata.go (reference — MsgMetadata, ConnState)
    - internal/buffer/buffer.go (reference — Buffer interface, FileBuffer)
    - internal/dsn/dsn.go (supporting — DSN generation)
    - internal/target/delivery.go (supporting — DeliveryLogger)
    - internal/limiters/concurrency.go (supporting — Semaphore)
    - internal/future/future.go (reference — Future primitive)
    - maddy.conf (reference — default production configuration)
    - HACKING.md (reference — architecture and error handling guide)
    - internal/target/queue/queue_test.go (reference — behavioral test patterns)
Sections:
    - Introduction and Scope
    - Q1: End-to-End Message Processing Pipeline (with Mermaid flowchart)
    - Q2: Retry Sequence with Small max_tries and Timeout (with timeline diagram)
    - Q3: Timeout Duration Analysis (with Go net.Dialer defaults)
    - Q4: Log Entry Anatomy During Retries (with JSON examples and field table)
    - Q5: Queue Filesystem Layout (with file listing, .meta JSON schema)
    - Q6: Queue Scheduler and Multi-Message Prioritization (with TimeWheel diagram)
    - Q7: Cross-Message Retry Independence (with goroutine model analysis)
    - Q8: Queue Starvation Analysis (with semaphore saturation scenario)
    - Source Citations
Diagrams:
    - End-to-end pipeline flow (Mermaid flowchart)
    - Queue message state machine (Mermaid stateDiagram)
    - TimeWheel tick loop (Mermaid flowchart)
    - Retry timeline (Mermaid sequence diagram)
    - Semaphore saturation (Mermaid sequence diagram)
Key Citations:
    - internal/target/queue/queue.go (lines 112–147, 182–198, 275–323, 365–429, 571–587, 600–620, 622–688, 690–740, 742–767, 849–953)
    - internal/target/queue/timewheel.go (lines 10–128)
    - internal/target/remote/remote.go (lines 49–70, 175–193, 381–440)
    - internal/target/remote/connect.go (lines 113–225, 240–276)
    - internal/smtpconn/smtpconn.go (lines 31–63, 65–120, 122–200)
    - internal/log/log.go (lines 59–104, 135–155)
    - internal/log/orderedjson.go (lines 16–62)
    - internal/log/writer.go (lines 16–29)
    - internal/exterrors/smtp.go (lines 19–128)
    - internal/exterrors/temporary.go (IsTemporaryOrUnspec, IsTemporary)
```

### 0.5.3 Documentation Configuration Updates

No documentation infrastructure configuration changes are required. The output file (`blitzy/documentation/maddy.md`) is a standalone Markdown document that does not need to be integrated into the existing MkDocs navigation (`.mkdocs.yml`). It lives outside the MkDocs-managed `docs/` folder by design, following the implementation rule requirement for placement in `blitzy/documentation/`.


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following tools and packages are relevant to creating the documentation deliverable:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| Go module | `github.com/foxcpp/maddy` | `go 1.13` (module) | Primary codebase under analysis |
| Go module | `github.com/emersion/go-smtp` | `v0.12.1-0.20191206174923-1f576e0ec85c` | SMTP client library used by `smtpconn` — source of `smtp.Client`, `smtp.SMTPError` types |
| Go module | `github.com/emersion/go-message` | `v0.10.9-0.20191116124005-65fd0119e899` | `textproto.Header` type used for message header serialization in queue |
| Go module | `golang.org/x/net` | `v0.0.0-20191126235420-ef20fe5d7933` | `idna` package for hostname normalization in remote delivery |
| Go module | `github.com/miekg/dns` | `v1.1.22` | DNS library used for DNSSEC-aware MX resolution |
| Go module | `github.com/foxcpp/go-mockdns` | `v0.0.0-20191123143003-02edb10da1e3` | Mock DNS resolver used in tests — relevant to understanding test patterns |
| Go module | `golang.org/x/net/publicsuffix` | (part of `golang.org/x/net`) | Common domain check in MX authentication |
| pip | `mkdocs` | (as configured in `.mkdocs.yml`) | Documentation site generator — existing infrastructure, not required for this deliverable |
| system | `scdoc` | (latest in Arch Linux repos) | Man page source format — existing infrastructure, not required for this deliverable |
| Go stdlib | `net` | (Go 1.13+) | `net.Dialer` — source of connection timeout behavior |
| Go stdlib | `encoding/json` | (Go 1.13+) | `.meta` file serialization format |
| Go stdlib | `math` | (Go 1.13+) | `math.Pow` — used in backoff formula |
| Go stdlib | `container/list` | (Go 1.13+) | Linked list for TimeWheel slot management |

### 0.6.2 Key Version Notes

- The Go module specifies `go 1.13` as the minimum toolchain version in `go.mod`. The build configuration (`.build.yml`) uses Arch Linux's system Go package without pinning a version, implying compatibility with the latest stable Go release
- All Go module dependencies use pre-release or commit-pinned versions (e.g., `v0.12.1-0.20191206174923`), which is consistent with the project's pre-1.0 development status
- The `net.Dialer` used in `smtpconn.New()` (line 59) is instantiated with default settings: `(&net.Dialer{}).DialContext`. In Go 1.13+, the default `net.Dialer` has no explicit timeout — the TCP connection timeout is governed by the operating system's SYN retry behavior (typically 75–130 seconds on Linux depending on `tcp_syn_retries`)
- The `go-smtp` library version `v0.12.1` provides the `smtp.Client` type that wraps the underlying TCP connection. Its timeout behavior inherits from the connection provided by the dialer

### 0.6.3 Documentation Reference Updates

No documentation link updates are required for this deliverable. The output document is standalone and does not create cross-references to existing MkDocs-managed pages. All internal references within `maddy.md` will use relative source code paths (e.g., `internal/target/queue/queue.go:414`).


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis for the specific topics the user asked about:**

| Topic Area | Current Documentation | Gap |
|------------|----------------------|-----|
| End-to-end pipeline flow | Partially covered in tech spec (4.1, 4.3, 4.4) but not in repository docs | Full narrative walkthrough with source citations |
| Queue retry mechanics | **None** | Backoff formula, max_tries logic, error classification chain |
| Queue filesystem layout | **None** | File naming patterns, .meta JSON schema, atomic update strategy |
| Structured log format | **None** | JSON field anatomy, timestamp format, per-phase log entries |
| TimeWheel scheduler | **None** | Algorithm, nearest-deadline selection, dispatch coordination |
| Multi-message scheduling | **None** | Semaphore behavior, goroutine model, cross-message independence |
| Queue starvation | **None** | Starvation conditions, observable patterns, mitigation |
| Timeout duration analysis | **None** | net.Dialer defaults, OS-level SYN retries, SMTP client timeouts |

- **Target coverage:** 100% of the user's questions must be answered with code-grounded rationale
- **All 8 question areas** (pipeline flow, retry sequence, timeout durations, log entries, filesystem layout, scheduler behavior, cross-message independence, starvation) must be fully addressed

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**

- Every answer must reference specific source files with line numbers
- All code paths in the retry lifecycle must be traced (success, temporary failure, permanent failure, exhaustion, panic)
- The `.meta` JSON schema must document every field in `QueueMetadata` (from `queue.go` lines 149–170)
- Log entries must be shown with actual field names derived from the `Logger.Msg()`, `Logger.Error()`, and `DeliveryLogger()` call sites
- The backoff formula must be documented with concrete numerical examples
- The TimeWheel algorithm must be explained step-by-step with the actual linked-list scan logic

**Accuracy validation:**

- Every claimed behavior must be traceable to a specific source code location
- No theoretical behavior — only documented behavior based on code analysis
- Error classification must correctly reflect the `IsTemporaryOrUnspec()` default-to-temporary semantics
- Timeout values must accurately reflect Go's `net.Dialer` defaults (no hardcoded timeout) and OS-level TCP behavior
- Log format must accurately reflect the `marshalOrderedJSON` output with deterministic field ordering

**Clarity standards:**

- Progressive disclosure: start with high-level pipeline overview, then drill into each subsystem
- Use concrete examples with realistic message IDs, timestamps, and error messages
- Include Mermaid diagrams for all complex flows and state machines
- Use tables for structured data (configuration options, log fields, file contents)
- Maintain consistent terminology from the codebase (e.g., "delivery semaphore" not "connection pool")

### 0.7.3 Example and Diagram Requirements

- **Minimum examples per question:** At least one concrete, realistic example with actual field values
- **Diagram types required:** Mermaid flowcharts for pipeline flow and scheduler, Mermaid stateDiagram for queue states, Mermaid sequence diagram for retry timeline
- **Code example testing:** All code snippets represent actual Go source code from the repository (read directly, not generated)
- **Log entry examples:** Must show the exact format produced by `internal/log/writer.go` with timestamps in `2006-01-02T15:04:05.000Z` format and JSON payloads with deterministic field ordering


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**

- `blitzy/documentation/maddy.md` — the sole deliverable, containing comprehensive answers to all user questions

**Source code files analyzed (read-only, for documentation purposes):**

- `internal/target/queue/queue.go` — queue lifecycle, retry scheduling, filesystem persistence, DSN generation
- `internal/target/queue/timewheel.go` — time-wheel scheduler algorithm
- `internal/target/queue/queue_test.go` — behavioral test patterns for validation
- `internal/target/queue/timewheel_test.go` — scheduler test patterns
- `internal/target/remote/remote.go` — remote delivery target module
- `internal/target/remote/connect.go` — MX connection, failover, policy enforcement
- `internal/smtpconn/smtpconn.go` — SMTP connection wrapper, error handling
- `internal/msgpipeline/msgpipeline.go` — message pipeline orchestrator
- `internal/endpoint/smtp/smtp.go` — SMTP inbound endpoint, session handling
- `internal/log/log.go` — structured logging API
- `internal/log/orderedjson.go` — deterministic JSON serialization
- `internal/log/writer.go` — log output formatting with timestamps
- `internal/exterrors/smtp.go` — SMTPError type with Fields method
- `internal/exterrors/temporary.go` — error temporary classification
- `internal/exterrors/fields.go` — structured error field extraction
- `internal/module/delivery_target.go` — DeliveryTarget and Delivery interfaces
- `internal/module/msgmetadata.go` — MsgMetadata and ConnState types
- `internal/module/partial_delivery.go` — PartialDelivery interface
- `internal/buffer/buffer.go` — Buffer interface (FileBuffer for disk storage)
- `internal/dsn/dsn.go` — DSN/bounce message generation
- `internal/target/delivery.go` — DeliveryLogger utility
- `internal/limiters/concurrency.go` — Semaphore implementation
- `internal/future/future.go` — Future concurrency primitive
- `maddy.conf` — default server configuration
- `HACKING.md` — contributor architecture guide
- `go.mod` — module identity and dependency versions

**Documentation topics in scope:**

- Complete message processing pipeline from SMTP ingress to remote delivery
- Queue retry mechanics: backoff formula, max_tries, error classification
- Queue filesystem layout: .header, .body, .meta files and their contents
- Structured log entry anatomy: JSON format, field names, timestamps
- TimeWheel scheduler algorithm and multi-message coordination
- Delivery semaphore (max_parallelism) and concurrency behavior
- Cross-message retry timing independence under timeout conditions
- Queue starvation analysis under semaphore saturation
- DSN/bounce generation when retries are exhausted

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — explicitly prohibited by user: "Avoid modifying the codebase" and "repository files should remain unchanged"
- **Test file modifications** — no test files will be created or modified in the repository
- **IMAP storage subsystem** — not relevant to the message processing pipeline questions
- **Authentication subsystem** — PAM, shadow, SQL auth backends are not part of the delivery pipeline under investigation
- **DKIM/SPF/DMARC check details** — mentioned only in the context of pipeline routing, not as primary documentation targets
- **Configuration file changes** — `maddy.conf` will not be modified; test configurations are described as documentation examples only
- **MkDocs infrastructure changes** — `.mkdocs.yml` will not be updated
- **Man page generation or updates** — `docs/man/` content is not in scope
- **Existing tutorial updates** — `docs/tutorials/` files will not be modified
- **Deployment or packaging changes** — `get.sh`, `package.sh`, `dist/` are not in scope
- **Storage backend internals** — `internal/storage/sql/` is not relevant to outbound delivery pipeline
- **SMTP submission endpoint specifics** — only inbound SMTP (port 25) and submission (port 465) are mentioned for context; the focus is on outbound delivery through the queue


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the deliverable is a standalone Markdown file (`blitzy/documentation/maddy.md`) that does not require a build step
- **Documentation preview command:** Any Markdown viewer or `cat blitzy/documentation/maddy.md` — no MkDocs integration needed
- **Diagram generation command:** Mermaid diagrams are embedded inline in the Markdown using fenced code blocks (```` ```mermaid ... ``` ````) and rendered by any Mermaid-compatible Markdown viewer
- **Default format:** Markdown with embedded Mermaid diagrams
- **Citation requirement:** Every section must reference specific source files with line numbers using the format `Source: path/to/file.go:LineNumber`
- **Style guide to follow:** The document should match the style of existing `docs/internals/` documents (technical prose with code references) while being more extensive. Answer format follows the Q&A structure implied by the user's questions, with rationale/thinking provided per the `SWE-AtlasQnA-Repo` implementation rule
- **Documentation validation:** Manual review for accuracy against source code — no automated link checking needed since the document contains source path references, not hyperlinks

### 0.9.2 Key Configuration Values to Document

The following configuration values and internal defaults are critical to the documentation and must be accurately presented:

| Parameter | Default Value | Source | Relevance |
|-----------|--------------|--------|-----------|
| `max_tries` | 8 | `queue.go:204` (`cfg.Int("max_tries", false, false, 8, ...)`) | Maximum delivery attempts before permanent failure |
| `max_parallelism` | 16 | `queue.go:205` (`cfg.Int("max_parallelism", false, false, 16, ...)`) | Bounded concurrent deliveries via semaphore |
| `initialRetryTime` | 15 minutes | `queue.go:184` (`15 * time.Minute`) | Base delay for first retry |
| `retryTimeScale` | 2.0 | `queue.go:185` | Exponential backoff multiplier |
| `postInitDelay` | 10 seconds | `queue.go:187` (`10 * time.Second`) | Minimum delay after server restart |
| `smtpPort` (remote) | "25" | `remote.go:41` (`var smtpPort = "25"`) | Outbound SMTP port |
| `net.Dialer` timeout | OS default (no explicit timeout) | `smtpconn.go:59` (`(&net.Dialer{}).DialContext`) | TCP connection timeout |

### 0.9.3 Backoff Formula Reference

The exact retry delay formula documented in the source code:

```
nextTryTime = now + initialRetryTime × retryTimeScale^(TriesCount - 1)
```

With defaults (`initialRetryTime=15m`, `retryTimeScale=2`):

| Attempt | TriesCount | Delay Formula | Delay |
|---------|-----------|---------------|-------|
| 1st retry | 1 | 15m × 2^0 | 15 minutes |
| 2nd retry | 2 | 15m × 2^1 | 30 minutes |
| 3rd retry | 3 | 15m × 2^2 | 60 minutes |
| 4th retry | 4 | 15m × 2^3 | 120 minutes |
| 5th retry | 5 | 15m × 2^4 | 240 minutes |
| 6th retry | 6 | 15m × 2^5 | 480 minutes |
| 7th retry | 7 | 15m × 2^6 | 960 minutes |

Source: `internal/target/queue/queue.go:413-414`


## 0.10 Rules for Documentation

The following rules are explicitly specified by the user or derived from the implementation rule `SWE-AtlasQnA-Repo`:

- **Do not modify any existing files in the source repository.** All output is confined to `blitzy/documentation/maddy.md`
- **Base all answers on the code as the truth.** No assumptions, no theoretical behavior — every claim must be traceable to a specific source code location
- **Provide thinking/rationale behind the answers.** Each answer must explain why the behavior occurs, citing the code that causes it
- **Name the document `maddy.md`** and place it in the `blitzy/documentation` directory
- **Focus on what actually happens during execution, not theoretical behavior.** This means analyzing the actual control flow, data structures, and Go runtime behavior rather than describing intended behavior from comments alone
- **Test configurations may be described as examples within the document** but must not be committed as repository files
- **Document creation of intentionally failing destinations** as illustrative scenarios within the narrative, showing how to configure maddy to observe the behaviors described
- **Source citations must use the format** `Source: relative/path/to/file.go:LineNumber` or `Source: relative/path/to/file.go:StartLine-EndLine` for ranges
- **Mermaid diagrams should be used** for all complex flows (pipeline, state machines, scheduler) to make the document visually navigable
- **All log entry examples must accurately reflect** the actual structured JSON format produced by `internal/log/orderedjson.go` with deterministic alphabetical field ordering
- **The document must address all 8 question areas** raised by the user, with no question left unanswered or deferred


## 0.11 References

### 0.11.1 Codebase Files and Folders Searched

The following files and folders were systematically searched and analyzed to derive the conclusions in this Agent Action Plan:

**Primary source files (read in full):**

| File Path | Lines | Purpose in Analysis |
|-----------|-------|---------------------|
| `internal/target/queue/queue.go` | 958 | Queue lifecycle, retry scheduling, persistence, DSN generation — central to all user questions |
| `internal/target/queue/timewheel.go` | 129 | Time-wheel scheduler algorithm — critical for scheduler and multi-message questions |
| `internal/target/queue/queue_test.go` | 823 | Behavioral tests — validates understanding of retry, serialization, cleanup |
| `internal/target/remote/remote.go` | 487 | Remote delivery target — connection per domain, body delivery |
| `internal/target/remote/connect.go` | 277 | MX connection establishment, failover, TLS policy — timeout source |
| `internal/smtpconn/smtpconn.go` | 339 | SMTP connection wrapper — error wrapping, timeout origin (net.Dialer) |
| `internal/msgpipeline/msgpipeline.go` | ~100 | Message pipeline entry point — routing and delivery start |
| `internal/endpoint/smtp/smtp.go` | ~280 | SMTP endpoint — message ingress, session lifecycle |
| `internal/log/log.go` | 208 | Logger API — Msg, Error, Debugf methods, field formatting |
| `internal/log/orderedjson.go` | 63 | JSON serialization — deterministic field ordering |
| `internal/log/writer.go` | 78 | Log output — timestamp format `2006-01-02T15:04:05.000Z` |
| `internal/exterrors/smtp.go` | 129 | SMTPError type — Fields(), Temporary(), error structure |
| `internal/exterrors/temporary.go` | (summary) | IsTemporaryOrUnspec, IsTemporary — error classification |
| `internal/exterrors/fields.go` | (summary) | WithFields, Fields — structured error metadata |
| `internal/module/delivery_target.go` | 72 | DeliveryTarget, Delivery interfaces — contract definition |
| `internal/module/msgmetadata.go` | 118 | MsgMetadata, ConnState — message metadata schema |
| `internal/buffer/buffer.go` | 43 | Buffer interface — FileBuffer for on-disk body |
| `internal/dsn/dsn.go` | ~60 | ReportingMTAInfo, DSN generation — bounce message structure |
| `internal/target/delivery.go` | (summary) | DeliveryLogger — adds msg_id to logger fields |
| `internal/limiters/concurrency.go` | (summary) | Semaphore — channel-based concurrency limiter |
| `internal/future/future.go` | 70 | Future — async result primitive for MTA-STS |
| `maddy.conf` | 153 | Default production configuration — queue settings |
| `go.mod` | 38 | Module identity, Go version, dependency versions |
| `HACKING.md` | 131 | Architecture guide, error handling patterns |
| `.mkdocs.yml` | 31 | Documentation infrastructure configuration |
| `.build.yml` | 26 | CI/build configuration |

**Folders explored (with get_source_folder_contents):**

| Folder Path | Depth | Reason |
|-------------|-------|--------|
| `` (root) | 0 | Repository structure discovery |
| `internal/` | 1 | All internal packages identification |
| `internal/target/` | 2 | Delivery target subsystems |
| `internal/target/queue/` | 3 | Queue implementation files |
| `internal/target/remote/` | 3 | Remote delivery implementation |
| `internal/smtpconn/` | 2 | SMTP connection layer |
| `internal/msgpipeline/` | 2 | Message pipeline orchestrator |
| `internal/endpoint/smtp/` | 3 | SMTP endpoint |
| `internal/log/` | 2 | Logging subsystem |
| `internal/exterrors/` | 2 | Error handling utilities |
| `internal/limiters/` | 2 | Concurrency control |
| `internal/module/` | 2 | Module interfaces |
| `docs/` | 1 | Existing documentation structure |
| `docs/internals/` | 2 | Internal documentation |
| `docs/man/` | 2 | Man page tooling |
| `cmd/` | 1 | Command binaries |

**Tech spec sections consulted:**

| Section | Relevance |
|---------|-----------|
| 1.1 EXECUTIVE SUMMARY | Project context, design goals |
| 4.1 HIGH-LEVEL SYSTEM WORKFLOW | Server lifecycle, end-to-end message flow |
| 4.3 MESSAGE PIPELINE AND ROUTING | Pipeline architecture, check runner, modifier chain |
| 4.4 DELIVERY AND QUEUE MANAGEMENT | Queue lifecycle, remote MX delivery, DSN generation |

### 0.11.2 Attachments

No attachments were provided by the user for this project.

### 0.11.3 External References

No Figma screens or external URLs were provided by the user. All analysis is based entirely on the repository source code.


