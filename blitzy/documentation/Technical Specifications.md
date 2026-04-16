# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification


### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the new feature requirement is to **onboard a developer to the maddy email server codebase by executing tests and capturing runtime log output** to answer specific behavioral questions about the system. This is an investigative, read-only exercise that produces a documentation artifact, not a code change. The specific requirements are:

- **SMTP Endpoint Log Tracing**: Run SMTP endpoint tests (`internal/endpoint/smtp/smtp_test.go`) with verbose and debug output to capture log messages for successful deliveries versus aborted deliveries. Identify the complete JSON field structure of key log events, the module name in the log prefix, and the exact `msg_id` format.
- **Queue Delivery Log Tracing**: Run queue delivery tests (`internal/target/queue/queue_test.go`) with verbose output to trace a message from initial acceptance through retry scheduling. Quote the exact log lines showing the delivery attempt, failure, and retry scheduling sequence.
- **Remote Delivery MX Auth and TLS Fallback**: Run remote delivery tests (`internal/target/remote/mxauth_test.go` and `internal/target/remote/remote_test.go`) covering MX authenticity check failures and TLS fallback. Capture the exact error message string, the SMTP enhanced status code in `X.Y.Z` format, and the complete reply text. For TLS fallback, quote the exact log message including all JSON fields.
- **Queue Retry Scheduling**: From queue retry tests, quote the log line showing retry scheduling and identify the exact JSON field name that contains the retry delay value.

Implicit requirements detected:
- The `-test.debuglog` flag must be passed to the Go test runner to activate debug-level logging within the test logger
- Tests must run with `-v` (verbose) to surface `t.Log()` output from the test framework
- No repository files may be modified; only a documentation artifact (markdown) may be created
- The documentation goes to `blitzy/documentation/` in the destination repository

### 0.1.2 Special Instructions and Constraints

- **Read-only constraint**: The user explicitly stated "Don't modify repository files; temporary scripts are fine but clean them up afterward." This limits the work to executing existing tests and capturing output.
- **Architectural requirement**: Use the existing Go test infrastructure (`go test`) with the project's own `testutils.Logger` which routes log output through `t.Log()` when running in verbose mode.
- **Repository convention**: The maddy project uses structured JSON logging via `internal/log/log.go`, with deterministic key ordering (alphabetical via `marshalOrderedJSON`) and a `name: msg\t{JSON}` format. This convention is critical for interpreting the test output.
- **Output rules**: The user requires exact log lines quoted from real test runs — not constructed or approximated from source code reading.

### 0.1.3 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- To **capture SMTP endpoint log behavior**, we will run `go test -v -count=1 -run "TestSMTPDelivery" ./internal/endpoint/smtp/ -test.debuglog` and analyze the `t.Log()` output that surfaces through the `testutils.Logger` function, which wraps `internal/log/log.go`'s structured logging.
- To **trace queue delivery through retry**, we will run `go test -v -count=1 -run "TestQueueDelivery_TemporaryRcptReject" ./internal/target/queue/ -test.debuglog` (and related tests like `TestQueueDelivery_TemporaryFail`, `TestQueueDelivery_MultipleAttempts`) and extract the full log sequence.
- To **capture MX authenticity failure details**, we will run `go test -v -count=1 -run "TestRemoteDelivery_AuthMX" ./internal/target/remote/ -test.debuglog` and extract the error fields from the test helper's `t.Log()` output.
- To **capture TLS fallback behavior**, we will run `go test -v -count=1 -run "TestRemoteDelivery_TLSErrFallback" ./internal/target/remote/ -test.debuglog` and quote the exact log line.
- To **identify retry scheduling fields**, we will examine the `"will retry"` log messages from the queue tests and document the JSON field name for the delay value.
- To **produce the deliverable**, we will create a comprehensive markdown document at `blitzy/documentation/maddy_26452dd8dd78.md` answering all the user's questions with exact log line evidence.


## 0.2 Repository Scope Discovery


### 0.2.1 Comprehensive File Analysis

The maddy codebase is a Go email server project (`github.com/foxcpp/maddy`). The following files were identified as directly relevant to the user's investigative requirements:

**SMTP Endpoint (source and test files examined)**

| File Path | Type | Relevance |
|-----------|------|-----------|
| `internal/endpoint/smtp/smtp.go` | Source | SMTP session handling: `startDelivery`, `abort`, RCPT, DATA, and accepted log emission |
| `internal/endpoint/smtp/smtp_test.go` | Test | 533 lines; contains `TestSMTPDelivery`, `TestSMTPDelivery_AbortData`, `TestSMTPDelivery_AbortLogout`, `TestSMTPDelivery_Reset`, `TestSMTPDelivery_Multi`, check error tests |
| `internal/endpoint/smtp/submission.go` | Source | Submission-mode handling (authentication required paths) |
| `internal/endpoint/smtp/date.go` | Source | Date header injection for submission mode |

**Queue System (source and test files examined)**

| File Path | Type | Relevance |
|-----------|------|-----------|
| `internal/target/queue/queue.go` | Source | Core queue logic: `tryDelivery`, `deliver`, retry scheduling with `will retry` log, DSN generation |
| `internal/target/queue/queue_test.go` | Test | 822 lines; contains `TestQueueDelivery`, `TestQueueDelivery_TemporaryFail`, `TestQueueDelivery_MultipleAttempts`, `TestQueueDelivery_TemporaryRcptReject`, DSN tests |
| `internal/target/queue/timewheel_test.go` | Test | TimeWheel scheduling mechanism tests |

**Remote Delivery (source and test files examined)**

| File Path | Type | Relevance |
|-----------|------|-----------|
| `internal/target/remote/remote.go` | Source | 486 lines; `Target` struct, `Start`, `AddRcpt`, `Body`, `BodyNonAtomic`, MX lookup, TLS handling |
| `internal/target/remote/connect.go` | Source | `connectionForDomain`, `checkPolicies` (MX authentication), TLS fallback logic |
| `internal/target/remote/remote_test.go` | Test | TLS fallback test (`TestRemoteDelivery_TLSErrFallback`), `RequireTLS` tests, split delivery tests |
| `internal/target/remote/mxauth_test.go` | Test | MX authenticity tests: MTA-STS, DNSSEC, CommonDomain, IP literal tests |

**Logging Infrastructure (source files examined)**

| File Path | Type | Relevance |
|-----------|------|-----------|
| `internal/log/log.go` | Source | Core `Logger` struct, `Msg()`, `Error()`, `Debugf()` methods, `formatMsg` producing `name: msg\t{JSON}` format |
| `internal/log/orderedjson.go` | Source | `marshalOrderedJSON` — deterministic alphabetical JSON key ordering |
| `internal/testutils/logger.go` | Source | `Logger(t, name)` function — routes log output through `t.Log()` in test mode |
| `internal/testutils/target.go` | Source | `Target`, `DoTestDelivery`, `DoTestDeliveryErr`, `CheckMsgID` test helpers |
| `internal/testutils/smtp_server.go` | Source | `SMTPServer`, `SMTPServerSTARTTLS`, `CheckSMTPErr` test helpers |

**Supporting Modules (examined for context)**

| File Path | Type | Relevance |
|-----------|------|-----------|
| `internal/target/delivery.go` | Source | `DeliveryLogger` — adds `msg_id` field to logger for delivery correlation |
| `internal/exterrors/smtp.go` | Source | `SMTPError` struct with `Code`, `EnhancedCode`, `Message`, `Fields()` method |
| `internal/msgpipeline/msgid.go` | Source | `GenerateMsgID()` — generates 4-byte random hex IDs (8 hex chars) |
| `internal/module/module.go` | Source | `MsgMetadata`, `Delivery`, `DeliveryTarget` interfaces |
| `internal/smtpconn/smtpconn.go` | Source | SMTP connection wrapper with `TLSError` type, `Connect`, STARTTLS handling |

### 0.2.2 Integration Point Discovery

- **Log formatting pipeline**: `Logger.Msg()` → `formatMsg()` → `marshalOrderedJSON()` → tab-delimited `name: msg\t{JSON}`
- **Delivery logger enrichment**: `target.DeliveryLogger(q.Log, meta.MsgMeta)` adds `msg_id` field to all subsequent log calls
- **Test output routing**: `testutils.Logger(t, name)` creates a `log.FuncOutput` that calls `t.Log(str)`, making log output visible only when `-v` is passed
- **Message ID generation**: SMTP endpoint uses `msgpipeline.GenerateMsgID()` producing 8-char hex; queue appends `-N` suffix for attempt number (e.g., `c0e0f73a-1`)
- **Error field propagation**: `exterrors.SMTPError.Fields()` returns `smtp_code`, `smtp_enchcode`, `smtp_msg`, `target`, and `reason` keys

### 0.2.3 New File Requirements

- **CREATE**: `blitzy/documentation/maddy_26452dd8dd78.md` — The comprehensive answers document as required by the project rules

No other new source files, test files, or configuration files are needed. This is a documentation-only deliverable.


## 0.3 Dependency Inventory


### 0.3.1 Key Packages

All dependencies are defined in `go.mod` with Go 1.13 as the language version. The packages directly relevant to this investigation are:

| Registry | Package | Version | Purpose |
|----------|---------|---------|---------|
| Go module | `go` (language) | 1.13 | Go runtime used for building and running tests |
| Go module | `github.com/emersion/go-smtp` | v0.12.1-0.20191206174923-1f576e0ec85c | SMTP client/server library used in tests and production SMTP handling |
| Go module | `github.com/emersion/go-sasl` | v0.0.0-20190817083125-240c8404624e | SASL authentication library for SMTP submission tests |
| Go module | `github.com/foxcpp/go-mockdns` | v0.0.0-20191123143003-02edb10da1e3 | Mock DNS resolver for MX lookups in remote delivery and SMTP tests |
| Go module | `github.com/emersion/go-message` | v0.10.9-0.20191116124005-65fd0119e899 | MIME message parsing/generation and `textproto.Header` used in all delivery pipelines |
| Go module | `github.com/google/uuid` | v1.1.1 | UUID generation (used indirectly in the codebase) |
| Go module | `github.com/mattn/go-sqlite3` | v1.11.0 | SQLite3 driver (CGO dependency requiring gcc) |
| Go module | `github.com/miekg/dns` | v1.1.22 | DNS library for DNSSEC-aware resolution in remote MX auth |
| Go module | `golang.org/x/net` | v0.0.0-20191126235420-ef20fe5d7933 | IDNA/Punycode handling, `publicsuffix` for common domain auth |
| Go module | `github.com/stretchr/testify` | v1.4.0 | Test assertion library (indirect) |

### 0.3.2 Build Dependencies

| Tool | Version | Purpose |
|------|---------|---------|
| `gcc` / `build-essential` | System | Required for CGO compilation of `mattn/go-sqlite3` |

### 0.3.3 Dependency Updates

No dependency updates are required. This is a read-only investigation producing only documentation output.


## 0.4 Integration Analysis


### 0.4.1 Existing Code Touchpoints

No source code modifications are required. The analysis integrates with the following existing code paths that were exercised during test execution:

**SMTP Endpoint Flow** (exercised by `TestSMTPDelivery*` tests):
- `internal/endpoint/smtp/smtp.go` — `Session.startDelivery()` generates `msg_id` via `msgpipeline.GenerateMsgID()`, logs `"incoming message"` with `msg_id`, `sender`, `src_host`, `src_ip` fields
- `internal/endpoint/smtp/smtp.go` — `Session.Rcpt()` calls pipeline, logs `"RCPT ok"` with `msg_id` and `rcpt` fields
- `internal/endpoint/smtp/smtp.go` — `Session.Data()` / `Session.LMTPData()` logs `"accepted"` with `msg_id` on success
- `internal/endpoint/smtp/smtp.go` — `Session.abort()` logs `"aborted"` with `msg_id` on delivery abort

**Queue Delivery Flow** (exercised by `TestQueueDelivery*` tests):
- `internal/target/queue/queue.go` — `tryDelivery()` calls `target.DeliveryLogger(q.Log, meta.MsgMeta)` to create a msg_id-enriched logger
- `internal/target/queue/queue.go` — `deliver()` appends `-N` to the msg_id for each attempt: `msgMeta.ID + "-" + strconv.Itoa(meta.TriesCount+1)`
- `internal/target/queue/queue.go` — `tryDelivery()` logs `"delivered"` (with `attempt`, `rcpt` fields), `"delivery attempt failed"` (with `rcpt`, `reason`), `"will retry"` (with `attempts_count`, `next_try_delay`, `rcpts`), and `"not delivered, permanent error"` / `"not delivered, temporary error"`

**Remote Delivery Flow** (exercised by `TestRemoteDelivery_AuthMX*` and `TestRemoteDelivery_TLSErrFallback` tests):
- `internal/target/remote/connect.go` — `checkPolicies()` returns `SMTPError{Code: 550, EnhancedCode: {5, 7, 0}, Message: "Failed to estabilish the MX record (...) authenticity"}` on authentication failure
- `internal/target/remote/connect.go` — `connectionForDomain()` catches `smtpconn.TLSError` and logs `"TLS error, falling back to plaintext"` via `rd.Log.Error()` with `mx`, `domain`, and `reason` fields

### 0.4.2 Logging Infrastructure Integration

The test-time logging stack is:

```
testutils.Logger(t, "smtp") → log.Logger{Out: FuncOutput(t.Log)} → t.Log(str)
                                         ↓
                              Logger.Msg(msg, fields...) → formatMsg → marshalOrderedJSON
                                         ↓
                              "smtp: incoming message\t{\"msg_id\":\"...\",\"sender\":\"...\"}"
```

- `Logger.log()` prepends `l.Name + ": "` to all messages
- `Logger.Fields` (set by `DeliveryLogger`) are merged into every `Msg()` / `Error()` call's JSON output
- JSON keys are sorted alphabetically by `marshalOrderedJSON()` for deterministic output

### 0.4.3 Database / Schema Updates

None. No schema or migration changes are involved.


## 0.5 Technical Implementation


### 0.5.1 File-by-File Execution Plan

The implementation consists of running tests, analyzing their output, and producing a single documentation file.

**Group 1 — Test Execution (no file modifications)**

- **EXECUTE**: `go test -v -count=1 -run "TestSMTPDelivery" ./internal/endpoint/smtp/ -test.debuglog` — Captures SMTP endpoint log messages for successful delivery, abort, reset, multi-message, and check error scenarios
- **EXECUTE**: `go test -v -count=1 -run "TestSMTPDeliver_Check" ./internal/endpoint/smtp/ -test.debuglog` — Captures MAIL FROM error handling log output
- **EXECUTE**: `go test -v -count=1 -run "TestQueueDelivery" ./internal/target/queue/ -test.debuglog` — Captures queue delivery flow including accept, retry, and permanent failure log sequences
- **EXECUTE**: `go test -v -count=1 -run "TestQueueDelivery_MultipleAttempts" ./internal/target/queue/ -test.debuglog` — Captures multi-attempt retry cycle with mixed outcomes
- **EXECUTE**: `go test -v -count=1 -run "TestRemoteDelivery_AuthMX|TestRemoteDelivery_TLS|TestRemoteDelivery_RequireTLS" ./internal/target/remote/ -test.debuglog` — Captures MX auth failure errors and TLS fallback log messages
- **EXECUTE**: `go test -v -count=1 -run "TestRemoteDelivery_MXAuth_IPLiteral" ./internal/target/remote/ -test.debuglog` — Captures IP literal MX auth tests

**Group 2 — Documentation Artifact**

- **CREATE**: `blitzy/documentation/maddy_26452dd8dd78.md` — Comprehensive answers to all user questions, structured by topic, with exact log lines quoted from test output

### 0.5.2 Implementation Approach

- Establish investigative foundation by executing each test suite in verbose mode with debug logging enabled
- Capture and parse the `t.Log()` output that surfaces structured log lines from `internal/log/log.go`
- Cross-reference observed log lines with the source code to explain the format, field names, and module prefixes
- Document findings in a structured markdown file that directly answers each user question with evidence

### 0.5.3 Key Findings from Test Execution

**SMTP Endpoint Log Behavior:**
- The module name in the log prefix for SMTP operations is **`smtp`** (with pipeline operations prefixed as **`smtp/pipeline`**)
- The `msg_id` format at the SMTP endpoint level is an **8-character lowercase hexadecimal string** (e.g., `c0e0f73a`), generated by `msgpipeline.GenerateMsgID()` which reads 4 random bytes and hex-encodes them
- Successful RCPT log: `smtp: RCPT ok\t{"msg_id":"...","rcpt":"..."}`
- Successful delivery log: `smtp: accepted\t{"msg_id":"..."}`
- Aborted delivery log: `smtp: aborted\t{"msg_id":"..."}`

**Queue Delivery Log Behavior:**
- The module prefix is **`queue`**
- The `msg_id` for queue operations is the **SHA-1 hex of the test name** (40-character hex in tests), with a `-N` attempt suffix appended (e.g., `10a443bb...-1`)
- Delivered: `queue: delivered\t{"attempt":1,"msg_id":"...","rcpt":"..."}`
- Failed: `queue: delivery attempt failed\t{"msg_id":"...","rcpt":"...","reason":"..."}`
- Retry: `queue: will retry\t{"attempts_count":1,"msg_id":"...","next_try_delay":"...","rcpts":[...]}`
- The JSON field name containing the retry delay value is **`next_try_delay`**

**Remote Delivery — MX Auth Failure:**
- Error message string: `Failed to estabilish the MX record (mx.example.invalid.) authenticity`
- SMTP enhanced status code: `5.7.0`
- Complete reply: `No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity`

**Remote Delivery — TLS Fallback:**
- Log message: `remote: TLS error, falling back to plaintext\t{"domain":"example.invalid","msg_id":"...","mx":"mx.example.invalid.","reason":"smtpconn: x509: certificate signed by unknown authority"}`


## 0.6 Scope Boundaries


### 0.6.1 Exhaustively In Scope

- Test execution of all relevant test functions across three packages:
  - `internal/endpoint/smtp/*_test.go` — SMTP endpoint tests
  - `internal/target/queue/*_test.go` — Queue delivery and retry tests
  - `internal/target/remote/*_test.go` — Remote delivery, MX auth, TLS fallback tests
- Source code reading for log format understanding:
  - `internal/log/*.go` — Logger implementation and JSON formatting
  - `internal/testutils/*.go` — Test helper infrastructure
  - `internal/target/delivery.go` — DeliveryLogger msg_id enrichment
  - `internal/exterrors/smtp.go` — SMTPError field structure
  - `internal/msgpipeline/msgid.go` — Message ID generation
  - `internal/smtpconn/smtpconn.go` — TLSError type and STARTTLS handling
  - `internal/target/remote/connect.go` — MX authentication and TLS fallback logic
- Documentation artifact creation:
  - `blitzy/documentation/maddy_26452dd8dd78.md` — Comprehensive answers file

### 0.6.2 Explicitly Out of Scope

- Modification of any existing repository source files
- Creation of new Go source code, test files, or configuration files in the repository
- IMAP endpoint tests (`internal/endpoint/imap/`)
- Storage layer tests (`internal/storage/sql/`)
- DKIM, DMARC, SPF check tests
- Message pipeline routing tests (`internal/msgpipeline/`)
- Performance optimization or benchmarking
- DNS check tests (`internal/check/dns/`, `internal/check/dnsbl/`)
- Address normalization tests (`internal/address/`)
- Build or deployment configuration changes


## 0.7 Rules for Feature Addition


### 0.7.1 User-Specified Rules

The following rules were explicitly provided by the user or specified in the project configuration:

- **SWE-AtlasQnA-Repo Rule**: Create a new markdown document named `<source_branch_name>.md` that comprehensively answers the question(s) posed in the prompt. The source branch name is `maddy_26452dd8dd78`, so the output file is `maddy_26452dd8dd78.md`.
- **Build and run source code**: The rule requires building and running the source code to analyze repository behavior as needed — base answers on the code as the truth, not assumptions.
- **Do not modify existing files**: No existing files in the source repository may be modified.
- **Do not add other code**: No code besides the requested documentation may be added to the source repository.
- **Placement**: The generated document must be placed in the `blitzy/documentation` directory in the destination repo.
- **User's explicit constraint**: "Don't modify repository files; temporary scripts are fine but clean them up afterward."

### 0.7.2 Investigation-Specific Requirements

- All log lines quoted in the documentation must come from actual test execution output, not from source code inference alone
- The `-test.debuglog` flag must be used to enable debug-level log messages during testing
- The `-v` flag must be used to ensure `t.Log()` output is visible in test results
- Answer each question with both the exact quoted log line and an explanation of its structure traced back to the source code


## 0.8 References


### 0.8.1 Files and Folders Searched

The following files and folders were searched and retrieved across the codebase to derive the conclusions in this Agent Action Plan:

**Source Files Read:**
- `internal/endpoint/smtp/smtp.go` — SMTP session handling, log calls, msg_id generation
- `internal/endpoint/smtp/smtp_test.go` — SMTP endpoint test cases (533 lines)
- `internal/target/queue/queue.go` — Queue delivery logic, retry scheduling, log messages
- `internal/target/queue/queue_test.go` — Queue delivery tests (822 lines)
- `internal/target/remote/remote.go` — Remote delivery target (486 lines)
- `internal/target/remote/connect.go` — MX connection, authentication, TLS fallback
- `internal/target/remote/remote_test.go` — Remote delivery tests (TLS, RequireTLS, split delivery)
- `internal/target/remote/mxauth_test.go` — MX authenticity tests (MTA-STS, DNSSEC, CommonDomain, IPLiteral)
- `internal/target/delivery.go` — `DeliveryLogger` function
- `internal/log/log.go` — Core logging implementation
- `internal/log/orderedjson.go` — Deterministic JSON key ordering
- `internal/testutils/logger.go` — Test logger configuration
- `internal/testutils/target.go` — Test delivery target and helpers
- `internal/testutils/smtp_server.go` — Test SMTP server utilities
- `internal/exterrors/smtp.go` — SMTPError struct and Fields method
- `internal/msgpipeline/msgid.go` — Message ID generation
- `internal/smtpconn/smtpconn.go` — SMTP connection wrapper, TLSError type
- `internal/smtpconn/smtpconn_test.go` — SMTP connection test setup
- `go.mod` — Dependency manifest

**Folders Explored:**
- Root directory (`/tmp/blitzy/maddy/maddy_26452dd8dd78_76465c/`)
- `internal/` — All 24 sub-directories scanned for test files
- `internal/endpoint/smtp/` — SMTP endpoint package
- `internal/target/queue/` — Queue system package
- `internal/target/remote/` — Remote delivery package
- `internal/log/` — Logging infrastructure
- `internal/testutils/` — Test utilities
- `internal/exterrors/` — Extended error types
- `internal/smtpconn/` — SMTP connection package
- `internal/msgpipeline/` — Message pipeline and ID generation

### 0.8.2 Attachments

No user attachments were provided for this project. No Figma URLs were referenced.

### 0.8.3 Environment Setup

- **Runtime**: Go 1.13.15 (linux/amd64) — installed from `go.dev/dl/go1.13.15.linux-amd64.tar.gz`
- **Build tools**: `gcc` and `build-essential` installed for CGO compilation of `mattn/go-sqlite3`
- **Dependencies**: Downloaded via `go mod download` with all modules resolved from `go.sum`
- **Source branch**: `maddy_26452dd8dd78` (commit `26452dd`)


