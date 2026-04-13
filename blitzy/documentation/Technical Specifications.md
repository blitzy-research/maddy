# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the new feature requirement is to **provide a comprehensive runtime behavior analysis of the maddy mail server** by executing its test suites, capturing actual log output, and documenting the exact formats, fields, and sequences observed. This is an onboarding and codebase-understanding exercise, not a code-modification task.

The specific deliverables are:

- **SMTP endpoint log analysis**: Run SMTP endpoint tests with verbose output to capture the exact log line format for successful delivery (including all JSON fields in the `RCPT ok` and `accepted` messages) versus aborted deliveries (including the `DATA error` and `aborted` log lines). Identify the module name that appears in the log prefix for SMTP operations, and document the exact `msg_id` format observed.

- **Queue delivery trace**: Run the queue delivery tests to capture the complete sequence of log messages from initial message acceptance through retry scheduling. Quote exact log lines showing `delivered`, `delivery attempt failed`, and `will retry` messages with all their JSON fields.

- **Remote delivery MX authentication failure analysis**: Run remote delivery tests that exercise MX authenticity check failures. Capture the exact error message string when MX authenticity verification fails, the precise SMTP enhanced status code in `X.Y.Z` format, and the complete reply text from the test output.

- **TLS fallback log capture**: Run the TLS error fallback test and quote the exact log message that appears when the system falls back from TLS to plaintext, including every JSON field present in the log line.

- **Retry scheduling field identification**: From the queue retry test output, quote the log line showing retry scheduling and identify the exact JSON field name that contains the retry delay value.

- **Documentation output**: Create a markdown document named `maddy_26452dd8dd78.md` in `blitzy/documentation/` containing all findings, with exact log lines and analysis.

Implicit requirements detected:
- The Go build environment must be operational to compile and execute tests
- PAM development headers are required for the full build (`maddy-pam-helper` CGo dependency)
- All test execution must use `-v` (verbose) flag to surface `t.Log()` output from the test utility logger
- No existing repository files may be modified; only the new markdown document in `blitzy/documentation/` is created

### 0.1.2 Special Instructions and Constraints

- **Read-only constraint on repository files**: The user explicitly stated: "Don't modify repository files; temporary scripts are fine but clean them up afterward." The implementation rule further specifies: "Do not modify any existing files in the source repository."
- **Output document requirement**: Per implementation rule `SWE-AtlasQnA-Repo`, create `blitzy/documentation/maddy_26452dd8dd78.md` containing comprehensive answers with rationale, grounded in code as truth.
- **Exact quoting required**: The user requests exact log lines, exact field names, exact status codes — not paraphrased summaries.
- **Architectural convention**: Maddy uses a structured logging subsystem (`internal/log/`) that formats messages as `module_name: message_text\t{JSON_fields}` where JSON keys are alphabetically sorted via `marshalOrderedJSON`.

### 0.1.3 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- To **capture SMTP endpoint log output**, we will execute `go test -v -count=1 -run "TestSMTPDelivery$|TestSMTPDelivery_AbortData|TestSMTPDelivery_AbortLogout" ./internal/endpoint/smtp/` and analyze the `t.Log()` output emitted by the `testutils.Logger(t, "smtp")` log adapter.

- To **trace queue delivery**, we will execute `go test -v -count=1 -run "TestQueueDelivery$|TestQueueDelivery_TemporaryFail$|TestQueueDelivery_MultipleAttempts" ./internal/target/queue/` and document the `queue: delivered`, `queue: delivery attempt failed`, and `queue: will retry` log messages.

- To **capture MX auth failure details**, we will execute `go test -v -count=1 -run "TestRemoteDelivery_AuthMX_Fail$|TestRemoteDelivery_AuthMX_DNSSEC_Fail" ./internal/target/remote/` and extract the error message, enhanced status code, and reply text from the test output.

- To **capture TLS fallback behavior**, we will execute `go test -v -count=1 -run "TestRemoteDelivery_TLSErrFallback" ./internal/target/remote/` and quote the `remote: TLS error, falling back to plaintext` log line with its complete JSON fields.

- To **produce the answer document**, we will create `blitzy/documentation/maddy_26452dd8dd78.md` containing all test outputs with analysis and exact quotations.

## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

The following repository files were examined and are relevant to executing the tests and understanding log output:

**SMTP Endpoint Test & Source Files:**

| File Path | Purpose | Relevance |
|-----------|---------|-----------|
| `internal/endpoint/smtp/smtp_test.go` (533 lines) | SMTP endpoint test suite — 12 test functions covering delivery, abort, reset, authentication, and UTF8 | Primary: Contains `TestSMTPDelivery`, `TestSMTPDelivery_AbortData`, `TestSMTPDelivery_AbortLogout` |
| `internal/endpoint/smtp/smtp.go` (720 lines) | Core SMTP endpoint implementation including session handling, logging, and delivery orchestration | Primary: Contains all `s.log.Msg(...)` and `s.endp.Log.Msg(...)` calls that produce the observed log output |
| `internal/endpoint/smtp/submission.go` | SMTP submission mode logic for authenticated clients | Supporting: Adds Message-ID and Date headers in submission mode |
| `internal/endpoint/smtp/smtp_test.go:522` | `TestMain` — assigns random port for test SMTP server | Infrastructure: Port allocation for test harness |

**Queue Test & Source Files:**

| File Path | Purpose | Relevance |
|-----------|---------|-----------|
| `internal/target/queue/queue_test.go` (746+ lines) | Queue delivery test suite — 16 test functions covering delivery, failure modes, retries, DSN generation | Primary: Contains `TestQueueDelivery`, `TestQueueDelivery_TemporaryFail`, `TestQueueDelivery_MultipleAttempts` |
| `internal/target/queue/queue.go` | Queue implementation including `tryDelivery()`, retry scheduling, DSN emission | Primary: Contains `dl.Msg("delivered"...)`, `dl.Error("delivery attempt failed"...)`, `dl.Msg("will retry"...)` |
| `internal/target/queue/timewheel.go` | Time-wheel scheduling for retry delays | Supporting: Manages when retries fire |

**Remote Delivery Test & Source Files:**

| File Path | Purpose | Relevance |
|-----------|---------|-----------|
| `internal/target/remote/remote_test.go` (975+ lines) | Remote delivery test suite — 19 test functions covering MX resolution, TLS, body errors, splits | Primary: Contains `TestRemoteDelivery_TLSErrFallback`, `TestRemoteDelivery_RequireTLS_*` |
| `internal/target/remote/mxauth_test.go` (550+ lines) | MX authentication test suite — 13 test functions for MTA-STS, DNSSEC, common domain, IP literal auth | Primary: Contains `TestRemoteDelivery_AuthMX_Fail`, `TestRemoteDelivery_AuthMX_DNSSEC_Fail` |
| `internal/target/remote/remote.go` | Remote delivery target implementation — MX lookup, connection management, policy checks | Primary: Contains `remoteDelivery.checkPolicies()` with MX auth error messages |
| `internal/target/remote/connect.go` (277 lines) | Connection establishment — MX resolution, TLS negotiation, STARTTLS fallback, policy enforcement | Primary: Contains `TLS error, falling back to plaintext` log line and `checkPolicies()` MX auth error returns |

**Logging Infrastructure:**

| File Path | Purpose | Relevance |
|-----------|---------|-----------|
| `internal/log/log.go` (208 lines) | Structured logging library — `Logger`, `Msg()`, `Error()`, format: `name: msg\t{JSON}` | Core: Defines the log line format observed in all test output |
| `internal/log/orderedjson.go` (62 lines) | Alphabetically-sorted JSON serialization for deterministic log output | Core: Explains why JSON keys appear in sorted order |
| `internal/log/output.go` | Output interface — `FuncOutput`, `NopOutput`, `multiOut` | Supporting: Test logger adapter mechanism |
| `internal/testutils/logger.go` (41 lines) | Test log adapter — routes `log.Logger` output to `t.Log()` for verbose test capture | Core: The bridge enabling log output to appear in `go test -v` |

**Error & Message ID Infrastructure:**

| File Path | Purpose | Relevance |
|-----------|---------|-----------|
| `internal/exterrors/smtp.go` (128 lines) | `SMTPError` type with `EnhancedCode`, `Fields()` method, and `FormatLog()` for `X.Y.Z` format | Core: Defines the error structure producing enhanced status codes |
| `internal/msgpipeline/msgid.go` (16 lines) | `GenerateMsgID()` — 4 random bytes → 8-character hex string | Core: Defines SMTP endpoint msg_id format |
| `internal/target/delivery.go` (16 lines) | `DeliveryLogger()` — wraps logger with `msg_id` field for delivery contexts | Core: Adds `msg_id` to all queue/remote delivery log messages |
| `internal/testutils/target.go` | `DoTestDelivery()`, `CheckMsgID()` — SHA1(test name) → 40-char hex for queue/remote test IDs | Core: Explains queue msg_id format difference from SMTP |

**SMTP Connection Infrastructure:**

| File Path | Purpose | Relevance |
|-----------|---------|-----------|
| `internal/smtpconn/smtpconn.go` | Outbound SMTP client wrapper — `Connect()`, `TLSError` type, STARTTLS logic | Primary: Defines `TLSError` that triggers the fallback path in `connect.go` |
| `internal/smtpconn/smtpconn_test.go` | SMTP connection tests | Supporting |
| `internal/testutils/smtp_server.go` | Mock SMTP server for test use — `SMTPServer()`, `SMTPServerSTARTTLS()` | Supporting: Provides test SMTP backends |

### 0.2.2 Web Search Research Conducted

No external web searches were required for this task. All answers are derived directly from the codebase source and actual test execution output. The maddy project is a well-documented Go codebase with self-contained test infrastructure.

### 0.2.3 New File Requirements

**New source files to create:**

| File Path | Purpose |
|-----------|---------|
| `blitzy/documentation/maddy_26452dd8dd78.md` | Comprehensive markdown document answering all user questions with exact log lines, analysis, and rationale based on actual test execution output |

No other new files are required. This is a read-only analysis task with a single documentation output.

## 0.3 Dependency Inventory

### 0.3.1 Private and Public Packages

The following packages are relevant to this test execution and analysis exercise, as identified from `go.mod`:

| Registry | Package | Version | Purpose |
|----------|---------|---------|---------|
| Go Module | `github.com/foxcpp/maddy` | (root module) | The maddy mail server — project under analysis |
| Go Module | `github.com/emersion/go-smtp` | `v0.12.1-0.20191206174923-1f576e0ec85c` | SMTP protocol library used by endpoint and smtpconn; defines `smtp.Client`, `smtp.SMTPError`, `smtp.EnhancedCode` |
| Go Module | `github.com/emersion/go-sasl` | `v0.0.0-20190817083125-240c8404624e` | SASL authentication for SMTP submission tests |
| Go Module | `github.com/emersion/go-message` | `v0.10.9-0.20191116124005-65fd0119e899` | MIME message parsing — `textproto.Header` used throughout delivery |
| Go Module | `github.com/foxcpp/go-mockdns` | `v0.0.0-20191123143003-02edb10da1e3` | Mock DNS resolver for test MX lookups, rDNS, and zone simulation |
| Go Module | `github.com/emersion/go-imap` | `v1.0.1` | IMAP protocol library (not directly exercised in SMTP/queue/remote tests) |
| Go Module | `github.com/foxcpp/go-imap-sql` | `v0.3.2-0.20191208094750-8b4ec6b19a78` | SQL-backed IMAP storage (not directly exercised) |
| Go Module | `github.com/mattn/go-sqlite3` | `v1.11.0` | SQLite3 driver for storage tests (CGo dependency) |
| Go Module | `github.com/miekg/dns` | `v1.1.22` | DNS library used by `internal/dns` for DNSSEC-aware resolution |
| Go Module | `github.com/google/uuid` | `v1.1.1` | UUID generation for message identifiers |
| Go Module | `golang.org/x/net` | `v0.0.0-20191126235420-ef20fe5d7933` | Extended networking — `publicsuffix` package used in `commonDomainCheck` |
| Go Module | `golang.org/x/crypto` | `v0.0.0-20191108234033-bd318be0434a` | Cryptographic primitives for TLS and authentication |
| System | `libpam0g-dev` | system package | PAM development headers required for `cmd/maddy-pam-helper` CGo compilation |
| Runtime | Go | `1.22.2` (installed; `go.mod` specifies minimum `1.13`) | Go toolchain for building and running tests |

### 0.3.2 Dependency Updates

No dependency updates are required. This is a read-only analysis task. All dependencies are resolved from the existing `go.mod` and `go.sum` lock files using `go mod` with `-mod=mod` to allow automatic download of missing modules.

## 0.4 Integration Analysis

### 0.4.1 Existing Code Touchpoints

This task is a read-only test execution and analysis exercise. No code modifications are required. However, the following integration points between modules are critical to understanding the log output and message flow:

**SMTP Endpoint → Logging Pipeline:**
- `internal/endpoint/smtp/smtp.go` (line 49): `endp.Log = testutils.Logger(t, "smtp")` — sets the logger name to `"smtp"`, which becomes the prefix `smtp:` in all SMTP endpoint log lines
- `internal/endpoint/smtp/smtp.go` (line 85): `endp.pipeline.Log = testutils.Logger(t, "smtp/pipeline")` — sets a separate logger for the message pipeline within the SMTP endpoint
- `internal/log/log.go` (line 182): `s = l.Name + ": " + s` — the `log()` method prepends the logger `Name` field, creating the `module: message` format

**Queue → Delivery Logger Integration:**
- `internal/target/queue/queue.go` (line 366): `dl := target.DeliveryLogger(q.Log, meta.MsgMeta)` — wraps the queue logger with a `msg_id` field from the message metadata
- `internal/target/delivery.go` (line 13): `fields["msg_id"] = msgMeta.ID` — injects `msg_id` as a persistent JSON field on every subsequent log call
- `internal/target/queue/queue_test.go` (line 58): `q.Log = testutils.Logger(t, "queue")` — sets the logger name to `"queue"`, visible only when `-v` flag is used

**Remote Delivery → MX Auth Policy Chain:**
- `internal/target/remote/remote.go` (line 190): `Log: target.DeliveryLogger(rt.Log, msgMeta)` — wraps remote logger with `msg_id`
- `internal/target/remote/connect.go` (lines 94–100): `checkPolicies()` returns `SMTPError{Code: 550, EnhancedCode: {5, 7, 0}, Message: "Failed to estabilish the MX record (%s) authenticity"}` when `requireMXAuth && !authenticated`
- `internal/target/remote/connect.go` (lines 175–177): TLS error fallback — `rd.Log.Error("TLS error, falling back to plaintext", err, "mx", record.Host, "domain", domain)` — produces the TLS fallback log line with `reason`, `mx`, and `domain` JSON fields

**Message ID Generation Paths:**
- `internal/msgpipeline/msgid.go` (line 12): `GenerateMsgID()` — generates 4 random bytes encoded as 8-character hex, used by the SMTP endpoint for inbound messages
- `internal/testutils/target.go` (line 239): `sha1.Sum([]byte(t.Name()))` → 40-character hex, used by test harness `DoTestDelivery()` for queue and remote delivery tests
- `internal/target/queue/queue.go` (line 439): `msgMeta.ID = msgMeta.ID + "-" + strconv.Itoa(meta.TriesCount+1)` — appends attempt number to the base msg_id for each delivery attempt

**Error Field Propagation:**
- `internal/exterrors/smtp.go` (line 72): `SMTPError.Fields()` returns a map with keys `smtp_code`, `smtp_enchcode`, `smtp_msg`, `target`, and `reason`
- `internal/exterrors/smtp.go` (line 11): `EnhancedCode.FormatLog()` returns `fmt.Sprintf("%d.%d.%d", ec[0], ec[1], ec[2])` — this is the `X.Y.Z` format seen in log output as the `smtp_enchcode` value
- `internal/log/log.go` (line 89): `Logger.Error()` merges `exterrors.Fields(err)` into the log output, automatically extracting and including all structured error fields

## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

**Group 1 — Environment Setup:**
- VERIFY: `go.mod` — Confirm Go module identity and minimum toolchain version (`go 1.13`)
- INSTALL: `libpam0g-dev` — System package required for CGo compilation of `cmd/maddy-pam-helper`
- BUILD: `go build ./...` — Compile all packages to download dependencies and verify build integrity

**Group 2 — SMTP Endpoint Test Execution:**
- EXECUTE: `go test -v -count=1 -run "TestSMTPDelivery$|TestSMTPDelivery_AbortData|TestSMTPDelivery_AbortLogout|TestSMTPDelivery_Reset" ./internal/endpoint/smtp/`
  - Captures: `incoming message`, `RCPT ok`, `accepted`, `DATA error`, `aborted` log lines
  - Captures: Module prefix `smtp:` and 8-character hex `msg_id` format

**Group 3 — Queue Delivery Test Execution:**
- EXECUTE: `go test -v -count=1 -run "TestQueueDelivery$|TestQueueDelivery_TemporaryFail$|TestQueueDelivery_MultipleAttempts|TestQueueDelivery_PermanentFail_NonPartial" ./internal/target/queue/`
  - Captures: `delivered`, `delivery attempt failed`, `will retry`, `not delivered, permanent error` log lines
  - Captures: Complete retry sequence from acceptance through failure and re-delivery

**Group 4 — Remote Delivery & MX Auth Test Execution:**
- EXECUTE: `go test -v -count=1 -run "TestRemoteDelivery_AuthMX_Fail$|TestRemoteDelivery_AuthMX_DNSSEC_Fail|TestRemoteDelivery_TLSErrFallback|TestRemoteDelivery_RequireTLS_Missing|TestRemoteDelivery_RequireTLS_NoErrFallback" ./internal/target/remote/`
  - Captures: MX authenticity failure error message and enhanced status code `5.7.0`
  - Captures: TLS fallback log line with all JSON fields
  - Captures: TLS required but unsupported error

**Group 5 — Documentation Output:**
- CREATE: `blitzy/documentation/maddy_26452dd8dd78.md` — Comprehensive answer document with all findings

### 0.5.2 Implementation Approach per File

The implementation approach follows a strictly sequential test execution and documentation pattern:

- **Establish build environment** by installing Go toolchain (1.22, backward-compatible with `go 1.13` minimum), PAM dev headers, and downloading all Go module dependencies via `go build ./...`
- **Execute SMTP endpoint tests** with `-v` flag, capturing the structured log output from `testutils.Logger(t, "smtp")` which routes all `log.Msg()`, `log.Error()`, and `log.DebugMsg()` calls through `t.Log()` for verbose capture
- **Execute queue delivery tests** with `-v` flag, noting that the queue test infrastructure (`newTestQueueDir`) only enables logging when `testing.Verbose()` returns true — confirming the `-v` flag requirement
- **Execute remote delivery tests** with `-v` flag, capturing error propagation through `exterrors.SMTPError.Fields()` into the test harness output
- **Analyze all captured output** to extract exact log line formats, JSON field names, module prefixes, msg_id formats, enhanced status codes, and error messages
- **Generate the answer document** with exact quotations and rationale grounded in the source code

### 0.5.3 Key Findings from Test Execution

**SMTP Log Format** (from `TestSMTPDelivery`):
```
smtp: RCPT ok\t{"msg_id":"<8-char-hex>","rcpt":"<address>"}
```

**Queue Retry Format** (from `TestQueueDelivery_TemporaryFail`):
```
queue: will retry\t{"attempts_count":1,"msg_id":"<40-char-hex>","next_try_delay":"<duration>","rcpts":["<addr>"]}
```

**MX Auth Failure** (from `TestRemoteDelivery_AuthMX_Fail`):
- Error message: `Failed to estabilish the MX record (mx.example.invalid.) authenticity`
- Enhanced status code: `5.7.0` (rendered from `EnhancedCode{5, 7, 0}`)
- Reply text: `No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity`

**TLS Fallback** (from `TestRemoteDelivery_TLSErrFallback`):
```
remote: TLS error, falling back to plaintext\t{"domain":"...","msg_id":"...","mx":"...","reason":"smtpconn: tls: ..."}
```

**Retry Delay Field**: The JSON field containing the retry delay is `next_try_delay`.

## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

**Test packages executed:**
- `internal/endpoint/smtp/` — All SMTP endpoint tests (12 test functions)
- `internal/target/queue/` — All queue delivery tests (16 test functions)
- `internal/target/remote/` — All remote delivery and MX auth tests (32 test functions)

**Source files analyzed for log output understanding:**
- `internal/endpoint/smtp/smtp.go` — SMTP session log messages
- `internal/target/queue/queue.go` — Queue delivery and retry log messages
- `internal/target/remote/remote.go` — Remote delivery log messages
- `internal/target/remote/connect.go` — MX auth policy and TLS fallback log messages

**Logging infrastructure analyzed:**
- `internal/log/log.go` — Logger format: `name: msg\tJSON`
- `internal/log/orderedjson.go` — Alphabetical JSON key ordering
- `internal/log/output.go` — Output adapters including `FuncOutput`
- `internal/testutils/logger.go` — Test log adapter routing to `t.Log()`

**Error infrastructure analyzed:**
- `internal/exterrors/smtp.go` — `SMTPError`, `EnhancedCode.FormatLog()`
- `internal/exterrors/fields.go` — `Fields()` extraction for log enrichment

**Message ID generation analyzed:**
- `internal/msgpipeline/msgid.go` — 8-char hex for SMTP endpoint
- `internal/testutils/target.go` — SHA1 40-char hex for test harness
- `internal/target/delivery.go` — `DeliveryLogger` msg_id injection

**Connection infrastructure analyzed:**
- `internal/smtpconn/smtpconn.go` — `TLSError` type and STARTTLS fallback
- `internal/testutils/smtp_server.go` — Mock SMTP server for test backends

**Output artifact:**
- `blitzy/documentation/maddy_26452dd8dd78.md` — Answer document

### 0.6.2 Explicitly Out of Scope

- **IMAP endpoint tests** (`internal/endpoint/imap/`) — Not requested by the user
- **Storage/SQL tests** (`internal/storage/sql/`) — Not related to SMTP/queue/remote delivery
- **Check module tests** (`internal/check/`) — DKIM, SPF, DNSBL tests not requested
- **Modifier tests** (`internal/modify/`) — DKIM signing, alias tests not requested
- **Message pipeline tests** (`internal/msgpipeline/`) — Not directly requested, though pipeline is involved in SMTP endpoint flow
- **Code modifications** — User explicitly prohibits modifying repository files
- **Performance testing or benchmarks** — Not requested
- **Configuration file changes** (`maddy.conf`) — Not applicable to test execution
- **Docker or deployment testing** — Not requested
- **Any refactoring of existing code** — Not applicable

## 0.7 Rules for Feature Addition

### 0.7.1 User-Specified Rules

- **SWE-AtlasQnA-Repo**: Create a new markdown document named `maddy_26452dd8dd78.md` that comprehensively answers the questions posed in the prompt. Provide thinking/rationale behind the answers. Do not make assumptions — base answers on the code as the truth. Do not modify any existing files in the source repository. Place the generated document in the `blitzy/documentation` directory.

- **No repository file modification**: The user stated "Don't modify repository files; temporary scripts are fine but clean them up afterward." This means the only new file permitted is the answer document in `blitzy/documentation/`.

- **Exact quotation requirement**: The user explicitly requests "quote the exact log lines," "show me the complete log line format," "quote the exact log message," and "identify the exact JSON field name." All answers must contain verbatim output from actual test runs, not inferred or paraphrased content.

### 0.7.2 Code-Derived Conventions

- **Verbose logging is conditional in queue tests**: The queue test infrastructure (`internal/target/queue/queue_test.go`, lines 57–61) only enables logging when `testing.Verbose()` is true. Tests **must** be run with `-v` to capture queue log output.

- **Log format convention**: All maddy log lines follow the pattern `module_name: event_message\t{alphabetically_sorted_json_fields}` as implemented in `internal/log/log.go` (`formatMsg` method, line 135) and `internal/log/orderedjson.go`.

- **Error field propagation**: When `Logger.Error()` is called, it automatically extracts fields from the error using `exterrors.Fields(err)` and merges them into the JSON output. The `reason` field defaults to `err.Error()` unless explicitly overridden.

- **Test msg_id determinism**: Queue and remote delivery tests use `sha1.Sum([]byte(t.Name()))` to generate deterministic 40-character hex msg_ids. SMTP endpoint tests use the runtime `GenerateMsgID()` which produces random 8-character hex values, so those will differ between runs.

## 0.8 References

### 0.8.1 Repository Files and Folders Searched

The following files and folders were systematically explored to derive all conclusions in this action plan:

**Root-level files:**
- `go.mod` — Go module identity, minimum toolchain version (`go 1.13`), and complete dependency manifest
- `go.sum` — Dependency checksums for reproducible builds
- `.build.yml` — CI configuration: build, test (`go test ./... -cover -race`), and man-page generation tasks
- `maddy.go` — Main server bootstrap with module registration (lines 20–38)
- `HACKING.md` — Contributor design guide describing architecture, module patterns, error handling

**SMTP endpoint:**
- `internal/endpoint/smtp/smtp.go` — Core SMTP session handling with all log calls (720 lines)
- `internal/endpoint/smtp/smtp_test.go` — 12 test functions for SMTP delivery scenarios (533 lines)
- `internal/endpoint/smtp/submission.go` — Submission mode with Message-ID/Date injection
- `internal/endpoint/smtp/smtputf8_test.go` — UTF8 SMTP extension tests

**Queue delivery:**
- `internal/target/queue/queue.go` — Queue implementation: delivery, retry, DSN emission
- `internal/target/queue/queue_test.go` — 16 test functions for queue delivery scenarios
- `internal/target/queue/timewheel.go` — Time-wheel scheduling for retry delays

**Remote delivery:**
- `internal/target/remote/remote.go` — Remote delivery target with MX resolution and policy checks
- `internal/target/remote/connect.go` — Connection establishment, TLS negotiation, MX auth policy chain (277 lines)
- `internal/target/remote/remote_test.go` — 19 test functions for remote delivery (975+ lines)
- `internal/target/remote/mxauth_test.go` — 13 test functions for MX authentication (550+ lines)

**Logging infrastructure:**
- `internal/log/log.go` — Logger struct, `Msg()`, `Error()`, `formatMsg()` methods (208 lines)
- `internal/log/orderedjson.go` — Deterministic alphabetical JSON serialization (62 lines)
- `internal/log/output.go` — Output interface: `FuncOutput`, `NopOutput`, `multiOut`

**Error handling:**
- `internal/exterrors/smtp.go` — `SMTPError` type, `EnhancedCode`, `Fields()` method (128 lines)
- `internal/exterrors/fields.go` — `Fields()` extraction from errors
- `internal/exterrors/temporary.go` — Temporary/permanent error classification

**Test utilities:**
- `internal/testutils/logger.go` — Test log adapter: `Logger(t, name)` → `t.Log()` (41 lines)
- `internal/testutils/target.go` — `DoTestDelivery()`, `DoTestDeliveryErr()`, `CheckMsgID()` — test delivery harness
- `internal/testutils/smtp_server.go` — `SMTPServer()`, `SMTPServerSTARTTLS()` mock backends

**Connection infrastructure:**
- `internal/smtpconn/smtpconn.go` — `Connect()`, `TLSError` type, STARTTLS handling

**Message ID and delivery:**
- `internal/msgpipeline/msgid.go` — `GenerateMsgID()`: 4 random bytes → 8-char hex (16 lines)
- `internal/target/delivery.go` — `DeliveryLogger()`: injects `msg_id` into logger fields (16 lines)

**Internal documentation:**
- `internal/README.md` — Architectural index of internal subsystem boundaries

### 0.8.2 Attachments

No attachments were provided with this project.

### 0.8.3 External References

No Figma URLs or external design references are applicable to this task. All information is derived exclusively from the maddy mail server source code and actual test execution output.

