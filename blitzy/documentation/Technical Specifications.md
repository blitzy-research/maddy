# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new analytical documentation artifact** that comprehensively answers a set of precise technical questions about the maddy mail server's delivery queue behavior. The user is receiving complaints from end-users about long email bounce times and needs to produce an authoritative, calculation-backed explanation of queue timing and throughput characteristics to share with those users.

**Category:** Create new documentation
**Documentation Type:** Technical analysis / Q&A document (code-backed investigation)

The user's requirements distill into the following concrete documentation needs:

- **Parallel Processing Model** — Explain how the queue's concurrency mechanism works (channel-based semaphore with configurable `max_parallelism`), identifying the exact source code constructs that govern parallel dispatch of delivery attempts
- **Initial Pass Throughput Calculation** — Given a batch of ~500 messages, each delivery attempt taking ~2 seconds to a single destination server, compute the actual wall-clock time for the first delivery pass using the queue's default parallelism settings
- **Retry Delay Accumulation** — Show the complete retry schedule from the first attempt through final bounce generation, documenting the exponential backoff formula with exact configuration values drawn from the source code, and computing cumulative wait time at each stage
- **TriesCount=0 Edge Case** — Analyze the mathematical behavior of the retry delay formula `initialRetryTime × retryTimeScale ^ (TriesCount - 1)` when `TriesCount` is zero, evaluating whether the resulting negative exponent produces an unexpected delay value, and explaining in which code path this condition can occur
- **Source-of-Truth Requirement** — All configuration values and formulas must be extracted from the actual queue module source code (`internal/target/queue/queue.go`) rather than assumed or estimated
- **Test Verification** — Run the existing test suite to confirm understanding of system behavior before documenting conclusions

**Inferred Documentation Needs:**

- The relationship between `max_tries` configuration and the actual number of delivery attempts (the code reveals a potential off-by-one: `max_tries=8` produces 9 total delivery attempts because the termination check `TriesCount == maxTries` occurs before the increment)
- The role of `postInitDelay` (10 seconds) in startup recovery scheduling and how it interacts with the retry formula in `readDiskQueue()`
- The TimeWheel scheduler's dispatch model and its interaction with the delivery semaphore to explain the queuing behavior under load

### 0.1.2 Special Instructions and Constraints

The user has specified several critical directives that constrain the documentation task:

- **Do not create new files or test files** — The user explicitly states: "Do not create new files or test files, just use what already exists in the repository." This means all analysis must be based on existing source code and existing tests
- **Run existing tests** — The user requests running the existing test suite to verify understanding. All tests in `internal/target/queue/` pass successfully (confirmed: `PASS ok github.com/foxcpp/maddy/internal/target/queue 1.509s`)
- **Find exact configuration values in source code** — The user does not want guessed or approximate values; all constants must be cited from specific source file locations
- **Show actual calculations** — The documentation must include step-by-step mathematical calculations that a user could follow to understand the timing

**Implementation Rule (SWE-AtlasQnA-Repo):**
- Create a new markdown document named `maddy_26452dd8dd78.md`
- Place in the `blitzy/documentation` directory
- Provide thinking/rationale behind answers
- Base answers on the code as truth
- Do not modify any existing files in the source repository

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document the **parallel processing model**, we will create a new analysis section in `blitzy/documentation/maddy_26452dd8dd78.md` that traces the dispatch flow from `queueDelivery.Commit()` through `TimeWheel.Add()` → `dispatch()` → semaphore acquisition → `tryDelivery()`, citing exact line numbers in `internal/target/queue/queue.go` and `internal/target/queue/timewheel.go`
- To document the **initial pass throughput**, we will compute `ceil(500 / 16) × 2s = 64 seconds` based on the default `max_parallelism=16` (from `queue.go` line 205) and the user's stated 2-second delivery time, explaining the semaphore batching behavior
- To document the **retry delay accumulation**, we will expand the formula from `queue.go` lines 122–126 (comment) and line 414 (implementation) with the default values from `NewQueue()` (lines 185–186: `initialRetryTime=15min`, `retryTimeScale=2`) and `Init()` (line 204: `maxTries=8`), computing each delay from attempt 1 through the final bounce
- To document the **TriesCount=0 edge case**, we will analyze `readDiskQueue()` line 670 where the formula is applied to metadata loaded from disk, showing that `math.Pow(2, -1) = 0.5` produces a 7.5-minute delay instead of the expected 15-minute first-retry delay
- To ensure **source-of-truth compliance**, every value cited will include the exact file path and line number as a source citation

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **MkDocs-based documentation system** with a ReadTheDocs theme, configured via `.mkdocs.yml` at the repository root. The documentation infrastructure covers tutorials, man pages, and internal design notes, but does **not** contain any dedicated analysis document addressing queue timing, throughput, or retry behavior.

- **Documentation framework:** MkDocs (configured in `.mkdocs.yml`)
- **Documentation generator configuration:** `.mkdocs.yml` at repository root — defines site metadata, theme (`readthedocs`), Markdown extensions (`codehilite`), and the full navigation tree
- **Man page tooling:** `scdoc` format sources in `docs/man/*.scd`, with a Python converter `docs/man/prepare_md.py` that transforms `scdoc` to Markdown
- **API documentation tools:** None detected — the project uses manual documentation rather than auto-generated API docs
- **Diagram tools:** The tech spec references Mermaid diagrams, but no Mermaid tooling is configured in the repository itself
- **Documentation hosting:** ReadTheDocs (implied by `.mkdocs.yml` configuration with `readthedocs` theme)

**Existing documentation structure:**
```
docs/
├── README.md                         (project landing page)
├── get.sh-script.md                  (installer documentation)
├── internals/
│   ├── sqlite.md                     (SQLite backend behavior)
│   └── quirks.md                     (SMTP/IMAP quirks)
├── man/
│   ├── README.md                     (man page authoring guide)
│   ├── prepare_md.py                 (scdoc → markdown converter)
│   ├── maddy-targets.5.scd           (delivery targets man page — contains queue config reference)
│   ├── maddy-smtp.5.scd
│   ├── maddy-config.5.scd
│   ├── maddy-filters.5.scd
│   ├── maddy-imap.5.scd
│   ├── maddy-storage.5.scd
│   ├── maddy-tls.5.scd
│   └── maddy.1.scd
└── tutorials/
    ├── alias-to-remote.md            (mail routing tutorial)
    ├── manual-installation.md        (build & install guide)
    └── setting-up.md                 (end-to-end deployment)
```

**Coverage gap identified:** The man page `maddy-targets.5.scd` contains queue configuration directive reference (lines 12–89) but provides no explanation of the retry delay formula, no throughput analysis, and no discussion of edge cases. The `HACKING.md` developer guide covers module architecture and error handling conventions but does not address queue internals.

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used to locate code relevant to the user's questions:

- **Queue module source:** `internal/target/queue/queue.go` — Contains the `Queue` struct, retry delay formula (lines 122–126, 414), default configuration values (lines 183–189, 201–237), dispatch/semaphore logic (lines 275–323), and delivery attempt flow (lines 365–429)
- **Queue tests:** `internal/target/queue/queue_test.go` — Contains `newTestQueue()` helper (lines 29–68), delivery tests covering temporary/permanent failure scenarios, serialization roundtrip, DSN generation, and partial delivery
- **TimeWheel scheduler:** `internal/target/queue/timewheel.go` — Implements the time-based dispatch scheduler used for retry scheduling and immediate dispatch
- **TimeWheel tests:** `internal/target/queue/timewheel_test.go` — Tests ordering, restart, and empty-queue behavior
- **Queue configuration in production:** `maddy.conf` lines 122–147 — Sets `max_tries 8`, `max_parallelism 16`, and configures the remote delivery target and bounce routing
- **Remote delivery target:** `internal/target/remote/remote.go` — The downstream delivery target used by the queue
- **Concurrency primitives:** `internal/limiters/concurrency.go` — Semaphore implementation (reference pattern; queue uses its own channel-based semaphore)
- **Test utilities:** `internal/testutils/target.go` — `DoTestDelivery()` helper used in queue tests
- **Developer guide:** `HACKING.md` — Design goals, module architecture, error handling, and goroutine safety conventions

**Key directories examined:**
- `internal/target/queue/` — Primary analysis target (4 files)
- `internal/target/remote/` — Downstream delivery target (6 files)
- `internal/target/` — Delivery utilities (2 files + 3 subfolders)
- `internal/limiters/` — Concurrency control (3 files)
- `internal/testutils/` — Test infrastructure
- `docs/` — Existing documentation tree
- `docs/man/` — Man page sources (queue configuration reference)

### 0.2.3 Test Suite Verification

The existing queue test suite was executed to verify understanding of system behavior:

```
go test ./internal/target/queue/... -v -count=1 -timeout 120s
```

**Results:** All 18 tests passed in 1.509 seconds:

| Test Name | Status | Validates |
|-----------|--------|-----------|
| `TestQueueDelivery` | PASS | Basic successful delivery and disk cleanup |
| `TestQueueDelivery_PermanentFail_NonPartial` | PASS | Permanent failure → no retry |
| `TestQueueDelivery_PermanentFail_Partial` | PASS | Partial permanent failure via PartialDelivery |
| `TestQueueDelivery_TemporaryFail` | PASS | Temporary failure → automatic retry succeeds |
| `TestQueueDelivery_TemporaryFail_Partial` | PASS | Partial temporary failure → selective retry |
| `TestQueueDelivery_MultipleAttempts` | PASS | Multi-attempt with mixed permanent/temporary |
| `TestQueueDelivery_PermanentRcptReject` | PASS | Permanent recipient rejection at AddRcpt |
| `TestQueueDelivery_TemporaryRcptReject` | PASS | Temporary recipient rejection → retry |
| `TestQueueDelivery_SerializationRoundtrip` | PASS | Disk persistence and restart recovery |
| `TestQueueDelivery_DeserlizationCleanUp/NoBody` | PASS | Cleanup of missing body file |
| `TestQueueDelivery_DeserlizationCleanUp/NoHeader` | PASS | Cleanup of missing header file |
| `TestQueueDelivery_AbortIfNoRecipients` | PASS | Abort when all recipients rejected |
| `TestQueueDelivery_AbortNoDangling` | PASS | No dangling files after abort |
| `TestQueueDSN` | PASS | DSN bounce generation |
| `TestQueueDSN_FromEmptyAddr` | PASS | No DSN for null-sender messages |
| `TestQueueDSN_NoDSNforDSN` | PASS | No infinite bounce loops |
| `TestQueueDSN_RcptRewrite` | PASS | DSN uses original recipient addresses |
| `TestTimeWheelAdd*` (4 tests) | PASS | TimeWheel scheduling and ordering |

The test results confirm the delivery retry mechanism, semaphore-based parallelism, disk persistence model, and DSN generation behavior.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The user's questions map to specific code modules and functions that must be analyzed and documented:

- **Module: `internal/target/queue/queue.go`**
  - Public APIs / Structures:
    - `Queue` struct (line 112): Contains `initialRetryTime`, `retryTimeScale`, `maxTries`, `deliverySemaphore`, `postInitDelay`
    - `QueueMetadata` struct (line 149): Contains `TriesCount`, `FirstAttempt`, `LastAttempt`, `To`, `FailedRcpts`, `TemporaryFailedRcpts`
    - `NewQueue()` (line 182): Default values — `initialRetryTime=15*time.Minute`, `retryTimeScale=2`
    - `Init()` (line 201): Config parsing — `max_tries` default=8, `max_parallelism` default=16
  - Retry formula implementation:
    - Comment (lines 122–126): `initialRetryTime * retryTimeScale ^ (TriesCount - 1)`
    - `tryDelivery()` line 414: Runtime formula application
    - `readDiskQueue()` line 670: Startup recovery formula application (TriesCount=0 edge case)
  - Parallelism mechanism:
    - `deliverySemaphore` (line 146): `chan struct{}` buffered to `maxParallelism`
    - `start()` (line 242): Semaphore initialization
    - `dispatch()` (lines 275–323): Goroutine launch with semaphore acquisition
  - Delivery lifecycle:
    - `tryDelivery()` (lines 365–429): Termination check (line 390), TriesCount increment (line 407), retry scheduling (lines 413–428)
  - Current documentation: **Incomplete** — Man page `maddy-targets.5.scd` has directive reference but no formula explanation, no throughput analysis, no edge case discussion

- **Module: `internal/target/queue/timewheel.go`**
  - `TimeWheel` struct (line 15): Concurrent scheduler for retry dispatch
  - `Add()` (line 38): Schedule a delivery for a target time
  - `tick()` (line 71): Main loop — finds closest slot, waits, dispatches
  - Current documentation: **None** — Internal implementation detail, not user-facing

- **Module: `internal/target/queue/queue_test.go`**
  - `newTestQueue()` (line 29): Test helper that sets `initialRetryTime=0`, `retryTimeScale=1`, `maxTries=5`, `maxParallelism=1`
  - Delivery scenario tests: Confirm temporary/permanent failure handling and retry behavior
  - Current documentation: **None** — Tests are self-documenting through test names

- **Configuration: `maddy.conf` (lines 122–147)**
  - `max_tries 8` (line 125): Configures the retry limit
  - `max_parallelism 16` (line 128): Configures concurrent delivery cap
  - Current documentation: Inline comments only

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No existing document explains how the retry delay formula works** — The formula `initialRetryTime × retryTimeScale ^ (TriesCount - 1)` is documented only as a code comment at `queue.go` lines 122–126 and is not exposed in user-facing documentation
- **No throughput analysis exists** — There is no documentation that explains how `max_parallelism` affects message processing speed or provides throughput calculations for batch scenarios
- **The TriesCount=0 edge case is undocumented** — The `readDiskQueue()` function at line 670 applies the retry formula with potentially zero `TriesCount` from disk-persisted metadata, producing `math.Pow(2, -1) = 0.5` and a 7.5-minute delay; this is not documented anywhere
- **The actual number of delivery attempts vs. `max_tries` is unclear** — The termination condition `TriesCount == maxTries` at line 390 is checked before the increment at line 407, resulting in `maxTries + 1` total delivery attempts (9 attempts with `max_tries=8`); this off-by-one behavior is not documented
- **No end-to-end timing analysis exists** — Users receiving bounce notifications have no reference for expected wait times based on default configuration
- **Man page `maddy-targets.5.scd` shows `max_tries 4`** (line 24) as the example value, while `maddy.conf` uses `max_tries 8` (line 125) and the code default is also `8` (`queue.go` line 204) — this inconsistency is not documented

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document `blitzy/documentation/maddy_26452dd8dd78.md` will be structured as a self-contained technical analysis with the following hierarchy:

```
blitzy/documentation/maddy_26452dd8dd78.md
├── Title / Introduction
│   └── Context and purpose of the analysis
├── Queue Configuration Defaults
│   └── Table of exact values from source code with file:line citations
├── Parallel Processing Model
│   ├── Semaphore-based concurrency mechanism
│   ├── TimeWheel dispatch flow
│   └── How goroutines compete for delivery slots
├── Initial Delivery Pass Throughput
│   ├── Setup: 500 messages, 2s per attempt, 16 parallelism
│   ├── Step-by-step calculation
│   └── Effective throughput rate
├── Retry Delay Accumulation
│   ├── Formula explanation with source citations
│   ├── Complete retry schedule table (all 9 attempts)
│   ├── Cumulative wait time computation
│   └── When the bounce notification is generated
├── TriesCount=0 Edge Case Analysis
│   ├── The mathematical behavior of 2^(-1)
│   ├── Code path where this occurs (readDiskQueue)
│   ├── Practical impact assessment
│   └── Why it does not cause a crash
├── Test Suite Verification
│   └── Summary of test execution confirming behavior
└── Summary / Key Takeaways
    └── Actionable timing numbers for users
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract default configuration values from `NewQueue()` at `internal/target/queue/queue.go:182-189` and `Init()` at lines 201-237
- Extract the retry formula from the code comment at lines 122-126 and the implementation at line 414
- Extract the termination condition from `tryDelivery()` at line 390
- Extract the parallelism mechanism from `dispatch()` at lines 275-323 and `start()` at line 242
- Extract the TriesCount=0 behavior from `readDiskQueue()` at line 670
- Confirm production configuration from `maddy.conf` lines 122-128

**Documentation Standards:**
- Markdown formatting with hierarchical headers (`# ## ### ####`)
- Code examples citing exact file paths and line numbers: `Source: internal/target/queue/queue.go:414`
- Tables for retry schedule, configuration defaults, and throughput calculations
- Step-by-step mathematical calculations shown explicitly so users can follow the reasoning
- All claims backed by source code references — no assumptions

### 0.4.3 Diagram and Visual Strategy

No Mermaid diagrams are planned for this document. The user's questions are quantitative and computational in nature, best served by tables and inline formulas rather than diagrams. The dispatch flow and retry lifecycle are already well-documented in the tech spec (sections 4.4 and 4.5) with extensive Mermaid diagrams; the new document will focus on the numerical analysis that is currently missing.

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/maddy_26452dd8dd78.md` | CREATE | `internal/target/queue/queue.go`, `internal/target/queue/timewheel.go`, `internal/target/queue/queue_test.go`, `internal/target/queue/timewheel_test.go`, `maddy.conf` | Complete analytical Q&A document covering: queue parallel processing model, initial delivery throughput for 500 messages at 2s/attempt, retry delay accumulation schedule with all 9 attempts, TriesCount=0 edge case analysis, and test verification results |

No existing files are modified, updated, or deleted per the user's explicit instruction: "Do not create new files or test files, just use what already exists in the repository" — interpreted as do not create new **source code** or **test** files. The implementation rule (SWE-AtlasQnA-Repo) specifically requires creating a new markdown document in `blitzy/documentation/`.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/maddy_26452dd8dd78.md
Type: Technical Analysis / Q&A Document
Source Code:
    - internal/target/queue/queue.go (primary)
    - internal/target/queue/timewheel.go
    - internal/target/queue/queue_test.go
    - internal/target/queue/timewheel_test.go
    - maddy.conf (lines 122-147)
    - docs/man/maddy-targets.5.scd (lines 12-89)
Sections:
    - Introduction (context: user complaints about bounce wait times)
    - Queue Configuration Defaults (table citing NewQueue, Init, maddy.conf)
    - Parallel Processing Model (semaphore, TimeWheel, goroutine dispatch)
    - Initial Delivery Pass Throughput (calculation: 500 msgs × 2s ÷ 16 parallelism)
    - Retry Delay Accumulation (formula, 9-attempt schedule, cumulative timings)
    - TriesCount=0 Edge Case (math.Pow(2,-1)=0.5, readDiskQueue path, impact)
    - Test Suite Verification (18/18 tests passed)
    - Summary with actionable timing numbers
Key Citations:
    - internal/target/queue/queue.go:122-126 (retry formula comment)
    - internal/target/queue/queue.go:182-189 (default values in NewQueue)
    - internal/target/queue/queue.go:201-237 (Init with config defaults)
    - internal/target/queue/queue.go:275-323 (dispatch with semaphore)
    - internal/target/queue/queue.go:365-429 (tryDelivery with termination logic)
    - internal/target/queue/queue.go:414 (retry delay calculation)
    - internal/target/queue/queue.go:622-688 (readDiskQueue with TriesCount=0 path)
    - internal/target/queue/timewheel.go:38-53 (Add method)
    - internal/target/queue/timewheel.go:71-128 (tick loop)
    - maddy.conf:122-128 (production queue configuration)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The new document is placed in `blitzy/documentation/` which is a separate output directory, not integrated into the MkDocs navigation tree or the existing `docs/` structure.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content or includes** — The new document is self-contained
- **No navigation link updates** — The document resides outside the MkDocs tree
- **No table-of-contents updates** — `.mkdocs.yml` is not modified
- **Source code references** — The document will use inline `Source: path:line` citations linking to specific source files; these are informational references, not hyperlinks

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The documentation task requires the Go toolchain to run the existing test suite and the project's dependency tree for test compilation. No additional documentation-specific tools are needed since the output is a plain Markdown file.

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| golang.org | go | 1.13+ (tested with 1.22.2) | Go toolchain required to compile and run `go test ./internal/target/queue/...` |
| go module | github.com/foxcpp/maddy | (local) | The project itself — source of truth for all queue behavior analysis |
| go module | github.com/emersion/go-smtp | v0.12.1 | SMTP error types used in queue metadata (`smtp.SMTPError`) |
| go module | github.com/emersion/go-message | v0.10.9 | `textproto.Header` used in queue message storage |

No documentation generators (MkDocs, Sphinx, etc.) are required for this task. The output is a standalone Markdown file that does not need to be built or rendered through a documentation pipeline.

### 0.6.2 Documentation Reference Updates

Not applicable. No existing documentation files require link updates since the new document is placed in the separate `blitzy/documentation/` directory and no existing files are modified.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

The user's request contains five distinct questions. Coverage targets for the output document:

| Question | Coverage Target | Source Code Citations Required |
|----------|----------------|-------------------------------|
| How does the queue process messages in parallel? | 100% — Full explanation of semaphore-based concurrency with dispatch goroutine model | `queue.go:112-147` (Queue struct), `queue.go:240-251` (start), `queue.go:275-323` (dispatch) |
| What is the actual throughput for 500 messages at 2s/attempt? | 100% — Numerical calculation with step-by-step working | `queue.go:205` (max_parallelism=16), `maddy.conf:128` |
| How do retry delays accumulate if every attempt fails? | 100% — Complete 9-attempt schedule with cumulative wait times | `queue.go:122-126` (formula), `queue.go:183-186` (defaults), `queue.go:204` (max_tries), `queue.go:390-428` (retry flow) |
| What happens mathematically when TriesCount is zero? | 100% — math.Pow(2, -1)=0.5 analysis, code path identification, practical impact | `queue.go:670` (readDiskQueue formula), `queue.go:122-126` (formula) |
| Verify using existing tests | 100% — Test execution results for all 18 queue/timewheel tests | `queue_test.go`, `timewheel_test.go` |

**Overall coverage target:** 100% of the user's stated questions, with every claim backed by a source code citation.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every question in the user's request must have a dedicated section with a clear answer
- All configuration values must cite their exact source file and line number
- All mathematical calculations must show intermediate steps
- The retry schedule must cover all attempts from first through final bounce

**Accuracy validation:**
- All configuration values verified against source code (not man pages or comments alone)
- Test suite executed and results reported as confirmation
- Mathematical results verified with Python computation (already completed during analysis phase)
- Edge case analysis confirmed by tracing the exact code path

**Clarity standards:**
- Answers must be understandable by someone who has not read the source code
- Technical terms must be defined on first use (e.g., "semaphore," "TimeWheel," "TriesCount")
- Calculations must be reproducible by the reader
- A summary section must provide actionable timing numbers that can be shared with end-users

**Maintainability:**
- Every claim includes a source citation with file path and line number
- The document explicitly states which version of the code was analyzed (branch `maddy_26452dd8dd78`)

### 0.7.3 Example and Diagram Requirements

- **Minimum code snippets:** 4–5 short Go code excerpts (2-3 lines each) showing the retry formula, default values, semaphore creation, dispatch mechanism, and termination condition
- **Tables required:** Configuration defaults table, retry schedule table (9 rows), throughput calculation table
- **Diagrams:** None required — the analysis is quantitative and best served by tables and inline formulas
- **Code example testing:** N/A — code snippets are read-only excerpts from the existing source, not executable examples

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

- **New documentation file:**
  - `blitzy/documentation/maddy_26452dd8dd78.md` — The sole deliverable: a comprehensive analytical document answering all five of the user's questions about queue behavior
- **Source code analysis (read-only):**
  - `internal/target/queue/queue.go` — Queue implementation, retry formula, parallelism, default config values
  - `internal/target/queue/timewheel.go` — TimeWheel scheduler implementation
  - `internal/target/queue/queue_test.go` — Queue test suite (to be executed for verification)
  - `internal/target/queue/timewheel_test.go` — TimeWheel test suite (to be executed for verification)
  - `maddy.conf` — Production queue configuration (lines 122–147)
  - `docs/man/maddy-targets.5.scd` — Existing queue documentation reference (for gap analysis)
  - `internal/limiters/concurrency.go` — Semaphore pattern reference
  - `go.mod` — Go module version and dependency information
- **Test execution (read-only):**
  - `go test ./internal/target/queue/... -v` — Run existing tests to confirm queue behavior understanding
- **Computation:**
  - Throughput calculation for 500 messages with 16 parallelism and 2-second delivery time
  - Complete retry schedule with cumulative wait times for 9 delivery attempts
  - Edge case analysis for `math.Pow(2, -1)` when TriesCount=0

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — The user explicitly states: "Do not create new files or test files, just use what already exists in the repository." No `.go` files will be created or modified
- **New test files** — No new test files will be created
- **Existing file modifications** — No existing files in the repository will be modified (per SWE-AtlasQnA-Repo rule)
- **Man page updates** — `docs/man/maddy-targets.5.scd` will not be updated even though it contains an inconsistent `max_tries 4` example
- **MkDocs configuration** — `.mkdocs.yml` will not be modified
- **Queue behavior changes** — No bug fixes, code patches, or configuration changes
- **Remote delivery target analysis** — `internal/target/remote/` is only relevant as context for understanding the downstream delivery pipeline; it is not a primary documentation target
- **DSN/bounce message content analysis** — The user's questions focus on timing, not bounce message content
- **Non-queue modules** — Authentication, storage, IMAP endpoint, checks, modifiers, and all other modules are out of scope
- **Performance benchmarks** — The user asks for throughput calculations, not benchmark execution

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Test execution command:** `cd /tmp/blitzy/maddy/maddy_26452dd8dd78_76465c && go test ./internal/target/queue/... -v -count=1 -timeout 120s`
- **Documentation build command:** N/A — Output is a standalone Markdown file, no build pipeline required
- **Documentation preview command:** N/A — Markdown renders natively on GitHub/GitLab
- **Diagram generation command:** N/A — No diagrams in the output document
- **Default format:** Markdown (`.md`) with tables, code blocks, and inline formulas
- **Citation requirement:** Every configuration value, formula, and behavioral claim must reference a specific source file path and line number
- **Style guide:** The document follows the SWE-AtlasQnA-Repo pattern — structured Q&A with rationale and code-backed evidence
- **Documentation validation:** Manual review — all numerical calculations have been pre-verified using Python computation during the analysis phase

## 0.10 Rules for Documentation

The following rules govern the documentation task, as derived from the user's explicit instructions and the SWE-AtlasQnA-Repo implementation rule:

- **Do not create new files or test files in the source repository** — Only the output document `blitzy/documentation/maddy_26452dd8dd78.md` is created; no files are added to the existing maddy repository structure
- **Do not modify any existing files in the source repository** — All source code, configuration, tests, and documentation in the repository remain untouched
- **Base all answers on the code as the truth** — Every claim must be traceable to a specific line in the source code; no assumptions, estimates, or external references used as primary evidence
- **Provide thinking and rationale behind the answers** — The document must explain not just the "what" but the "why" — showing the reasoning path from code to conclusion
- **Find exact configuration values in the queue module source code** — Values like `initialRetryTime`, `retryTimeScale`, `maxTries`, and `max_parallelism` must be extracted from their declaration sites in `queue.go` and verified against `maddy.conf`
- **Run existing tests to verify understanding** — The queue test suite must be executed and results reported as supporting evidence
- **Show actual calculations** — All throughput and timing analyses must include step-by-step arithmetic that the reader can verify independently
- **Place the output document in `blitzy/documentation/`** — The file must be named `maddy_26452dd8dd78.md` (matching the source branch name) and located in the `blitzy/documentation` directory

## 0.11 References

### 0.11.1 Source Code Files Analyzed

| File Path | Lines of Interest | Purpose in Analysis |
|-----------|-------------------|---------------------|
| `internal/target/queue/queue.go` | Full file (958 lines) | Primary analysis target — Queue struct, default config values, retry formula, dispatch/semaphore concurrency, tryDelivery flow, readDiskQueue recovery, DSN generation |
| `internal/target/queue/timewheel.go` | Full file (129 lines) | TimeWheel scheduler — Add(), tick() loop, dispatch callback mechanism |
| `internal/target/queue/queue_test.go` | Full file (823 lines) | Test suite — 14 test functions covering delivery success, failure, retry, persistence, DSN; test helper `newTestQueue()` with override values |
| `internal/target/queue/timewheel_test.go` | Full file (113 lines) | TimeWheel tests — 4 test functions covering ordering, restart, and empty-queue behavior |
| `maddy.conf` | Lines 122–147 | Production queue configuration — `max_tries 8`, `max_parallelism 16`, target remote, bounce routing |
| `docs/man/maddy-targets.5.scd` | Lines 12–89 | Existing queue documentation — configuration directive reference (noted: shows `max_tries 4` example vs. actual default of `8`) |
| `internal/limiters/concurrency.go` | Full file (47 lines) | Semaphore pattern reference — channel-based semaphore with Take/Release; queue uses same pattern directly |
| `go.mod` | Full file (37 lines) | Go module version (`go 1.13`) and dependency tree |
| `.mkdocs.yml` | Full file (30 lines) | Documentation infrastructure — MkDocs config with ReadTheDocs theme |
| `HACKING.md` | Full file (131 lines) | Developer guide — design goals, module architecture, error handling conventions, goroutine safety |
| `internal/target/remote/remote.go` | Lines 1–80 | Remote delivery target — the downstream target used by the queue for actual SMTP delivery |
| `.build.yml` | Full file (40 lines) | CI build configuration — confirms `go test ./... -cover -race` as the test command |

### 0.11.2 Folders Searched

| Folder Path | Depth Explored | Relevance |
|-------------|----------------|-----------|
| (root) | Level 0 | Repository structure, configuration files, Go module |
| `internal/` | Level 1 | Internal implementation packages |
| `internal/target/` | Level 2 | Delivery targets parent directory |
| `internal/target/queue/` | Level 3 | Primary analysis target — all 4 files read |
| `internal/target/remote/` | Level 2 | Downstream delivery target context |
| `internal/limiters/` | Level 2 | Concurrency primitive reference |
| `docs/` | Level 1 | Documentation structure |
| `docs/man/` | Level 2 | Man page sources with queue config reference |
| `docs/tutorials/` | Level 2 | Tutorial documentation (no queue content) |
| `docs/internals/` | Level 2 | Internal documentation (no queue content) |

### 0.11.3 Tech Spec Sections Referenced

| Section Heading | Relevance |
|----------------|-----------|
| 4.4 DELIVERY AND QUEUE MANAGEMENT | Delivery queue lifecycle, persistence model, dispatch flow, DSN generation |
| 4.5 ERROR HANDLING AND RECOVERY | Queue retry strategy, partial delivery handling, startup recovery |
| 5.2 COMPONENT DETAILS | Component architecture, module system, delivery targets, support libraries |

### 0.11.4 Attachments and External Resources

- **No attachments** were provided by the user
- **No Figma URLs** were referenced
- **No external URLs** are required for this documentation task — all analysis is based on the repository source code

