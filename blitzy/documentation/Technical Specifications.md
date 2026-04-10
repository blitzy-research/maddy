# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **investigate, reproduce, and document counterintuitive SPF/DMARC interaction behavior** in the maddy mail server codebase. The task is categorized as **Create new documentation** of type **Technical Investigation Report / Q&A Document**.

The specific documentation requirements are:

- **Reproduce three SMTP sessions** with controlled DNS records against the maddy mail server pipeline, each using different domain/DMARC combinations
- **Report exact SMTP response lines** (code, enhanced status code, message text) immediately after the DATA terminator `'.'`
- **Identify the outcome stage** (MAIL FROM, RCPT TO, or end of DATA) for each session
- **Extract server log lines** that explain each outcome, with variable IDs replaced by `<id>`
- **Explain the root cause** of why Case B (domain `maddy-dmarc.com` with `p=reject`) is accepted despite `v=spf1 -all`
- **Identify the input condition** that triggers deferral and why Case A does not defer
- **Identify the runtime component** that makes the deferral decision
- **Describe verification methodology** using only observable signals (SMTP responses + server logs)

The user emphasizes that **no tracked repository files may be modified** — only temporary test scripts and configurations are permissible.

### 0.1.2 Special Instructions and Constraints

- **Read-only constraint**: "Don't modify any files in the repository; if you need to create temporary scripts/configs to reproduce, that's fine, but don't edit tracked repo files."
- **Variable ID replacement**: "you may replace variable IDs such as message IDs with `<id>` while keeping everything else literal"
- **Evidence-based**: "answer with runtime evidence" — all conclusions must be backed by actual test execution, not static analysis alone
- **Observable signals only**: "how to verify all of it using only observable signals (SMTP responses + server logs), without modifying the repository"
- **Implementation rule**: Per the `SWE-AtlasQnA-Repo` rule, the deliverable is a markdown document named `maddy_26452dd8dd78.md` placed in `blitzy/documentation/`, with thinking/rationale, grounded in the code as the source of truth

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To reproduce the three SMTP sessions, we will create temporary Go test files that construct a maddy `MsgPipeline` with the actual `apply_spf` check module, `doDMARC=true`, and controlled DNS via `go-mockdns` `PatchNet` interception of `net.DefaultResolver`
- To capture exact SMTP responses, we will exercise the SMTP endpoint (`internal/endpoint/smtp`) with a mock SMTP client and report the `250`/`550` response after DATA close
- To extract server log lines, we will use the `testutils.Logger` infrastructure that captures structured log output during test execution
- To explain the root cause, we will trace the control flow through `relyOnDMARC` → `EvaluateAlignment` → `Apply` with exact line numbers and code references
- To document verification methodology, we will map each behavioral difference to a specific observable signal (log line, Auth-Results header field, quarantine metadata flag)

### 0.1.4 Inferred Documentation Needs

Based on code analysis:

- The interaction between `apply_spf` and DMARC is non-obvious — the `relyOnDMARC` function in `internal/check/spf/spf.go` is the critical decision point, but its behavior depends on whether `policyDomain == fromDomain` and the subdomain policy value
- The DMARC evaluation gap (missing DKIM results → `ResultNone` → `PolicyNone`) is not documented anywhere in the codebase; documenting this gap is an implicit requirement
- The difference between quarantine and rejection at the SMTP wire level (quarantine still returns `250`, not `4xx`/`5xx`) is a key insight that the user needs to understand
- The contrast between "with `verify_dkim`" and "without `verify_dkim`" pipeline configurations must be documented to fully explain the behavior

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a Go project with no dedicated documentation generator or site. Documentation exists as in-code comments and a handful of markdown files:

- `README.md` — Project overview, feature list, installation instructions
- `HACKING.md` — Developer guide with design goals and contribution guidelines
- `CONTRIBUTING.md` — Contribution process
- `.github/SECURITY.md` — Security disclosure policy
- `maddy.conf` — Default configuration file with inline comments (serves as implicit reference documentation)
- `internal/README.md` — Notice that internal packages are not stable API

No documentation generators (mkdocs, docusaurus, sphinx, etc.) are configured. No `docs/` directory exists at the repository root. API documentation is embedded in Go source files as godoc comments.

The project's documentation framework is **none** (plain markdown + godoc). No DMARC/SPF behavior documentation exists beyond code comments.

### 0.2.2 Repository Code Analysis for Documentation

The following source files were examined to understand the SPF/DMARC interaction that forms the basis of this documentation:

| Source File | Lines | Key Findings |
|---|---|---|
| `internal/check/spf/spf.go` | 374 | `Check` struct, `relyOnDMARC` (lines 212–235), `CheckBody` (lines 330–367), default `failAction = {Quarantine: true}` |
| `internal/dmarc/evaluate.go` | 234 | `EvaluateAlignment` (lines 99–184), `!dkimPresent` guard at line 148 returning `ResultNone` |
| `internal/dmarc/verifier.go` | 157 | `Apply` (lines 111–156), `ResultNone → PolicyNone` bypass at line 143 |
| `internal/dmarc/dmarc.go` | ~30 | Type aliases: `Record = dmarc.Record`, `Policy = dmarc.Policy`, `Resolver` interface |
| `internal/msgpipeline/check_runner.go` | 317 | `runAndMergeResults` (lines 141–209), `applyResults` (lines 261–316), `checkBody` (lines 245–259), `doDMARC` flag |
| `internal/msgpipeline/config.go` | ~110 | `doDMARC` parsing: `dmarc yes` or `dmarc` → `cfg.doDMARC = true` |
| `internal/msgpipeline/msgpipeline.go` | ~545 | `MsgPipeline` struct, `Mock` function, pipeline lifecycle |
| `internal/dns/resolver.go` | ~42 | `DefaultResolver()` returns `net.DefaultResolver` |
| `internal/endpoint/smtp/smtp.go` | ~680 | `Data` handler (lines 312–345), `wrapErr` (lines 389–450+) |
| `internal/check/dkim/dkim.go` | ~180 | `verify_dkim` module — absent from pipeline triggers the gap |
| `internal/testutils/check.go` | ~85 | `Check` mock struct for test infrastructure |

Search patterns used:
- `search_files("SPF check implementation")` → found `internal/check/spf/spf.go`
- `search_files("DMARC evaluation and verification")` → found `internal/dmarc/evaluate.go`, `internal/dmarc/verifier.go`
- `search_files("message pipeline check runner")` → found `internal/msgpipeline/check_runner.go`
- `search_folders("DNS resolution and mocking")` → found `internal/dns/`

### 0.2.3 External Dependency Analysis

| Dependency | Location | Relevance |
|---|---|---|
| `github.com/foxcpp/go-mockdns` | `go.sum` (v0.0.0-20191123143003) | DNS interception via `NewServer` + `PatchNet` for test reproduction |
| `github.com/emersion/go-smtp` | `go.sum` (v0.12.1-0.20191206174923) | SMTP server with `toSMTPStatus`, `WriteResponse` for response formatting |
| `github.com/emersion/go-msgauth` | `go.sum` | `authres` package for Authentication-Results formatting |
| `github.com/foxcpp/go-imap-sql` | `go.sum` | Not relevant to this investigation |
| `github.com/miekg/dns` | `go.sum` | Underlying DNS library used by go-mockdns |

### 0.2.4 Web Search Research Conducted

No external web search was required for this investigation. All findings are derived from source code analysis and runtime test execution within the repository. The behavior is specific to maddy's internal implementation and is not documented in any external RFC or best-practices guide.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The documentation maps the following code modules and their interactions:

- **Module: `internal/check/spf/spf.go`**
  - Public APIs: `New`, `Check.Init`, `Check.CheckStateForMsg`
  - Internal methods documented: `state.relyOnDMARC` (lines 212–235), `state.CheckBody` (lines 330–367), `state.spfResult` (lines 116–210)
  - Current documentation: Godoc comments only; no behavioral documentation for DMARC interaction
  - Documentation needed: Decision logic of `relyOnDMARC`, enforcement clearing behavior, log output mapping

- **Module: `internal/dmarc/evaluate.go`**
  - Public APIs: `EvaluateAlignment`, `FetchRecord`, `ExtractFromDomain`
  - Current documentation: Godoc comments only
  - Documentation needed: The `!dkimPresent` guard condition and its `ResultNone` output

- **Module: `internal/dmarc/verifier.go`**
  - Public APIs: `NewVerifier`, `Verifier.FetchRecord`, `Verifier.Apply`
  - Current documentation: Godoc comments only
  - Documentation needed: The `ResultNone → PolicyNone` bypass in `Apply`

- **Module: `internal/msgpipeline/check_runner.go`**
  - Internal methods documented: `runAndMergeResults`, `applyResults`, `checkBody`
  - Current documentation: Code comments only
  - Documentation needed: How `doDMARC` flag triggers DMARC evaluation, the "no check action" log case

- **Module: `internal/endpoint/smtp/smtp.go`**
  - Public APIs: `Session.Data`
  - Current documentation: Minimal
  - Documentation needed: Error-to-SMTP-response mapping via `wrapErr` → `toSMTPStatus`

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the following documentation gaps are addressed by this task:

- **Undocumented behavioral interaction**: The `relyOnDMARC` → `EvaluateAlignment` → `Apply` chain and the enforcement gap when `verify_dkim` is absent is not documented anywhere
- **No existing SPF/DMARC troubleshooting guide**: Operators encountering the described behavior have no reference material
- **Missing pipeline configuration documentation**: The relationship between `dmarc yes`, `apply_spf`, and `verify_dkim` in determining enforcement outcomes is implicit
- **No observable-signals reference**: No documentation maps specific log lines and Auth-Results fields to pipeline decision points
- **Subdomain policy behavior undocumented**: How `sp=none` prevents deferral for subdomains while `p=reject` triggers it for the organizational domain

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The deliverable is a single markdown document at `blitzy/documentation/maddy_26452dd8dd78.md` with the following structure:

```
blitzy/documentation/maddy_26452dd8dd78.md
├── Summary (executive overview of the finding)
├── Controlled DNS Configuration (table of all DNS records used)
├── Case A — Subdomain with Inherited DMARC (sp=none)
│   ├── (1) SMTP Response After End-of-DATA
│   ├── (2) Outcome Stage
│   ├── (3) Relevant Server Log Lines
│   └── Rationale
├── Case B — Organizational Domain with p=reject
│   ├── (1) SMTP Response After End-of-DATA
│   ├── (2) Outcome Stage
│   ├── (3) Relevant Server Log Lines
│   └── Rationale
├── Case C — Domain with No DMARC Record
│   ├── (1) SMTP Response After End-of-DATA
│   ├── (2) Outcome Stage
│   ├── (3) Relevant Server Log Lines
│   └── Rationale
├── Comparison Summary (table contrasting all three cases)
├── Why Case B Is Accepted Despite SPF -all
│   ├── The Deferral Mechanism
│   ├── The Clearing of SPF Enforcement
│   ├── The DMARC Evaluation Gap
│   ├── Why Case A Does NOT Defer
│   └── The Decision-Making Component
├── How to Verify Using Only Observable Signals
│   ├── SMTP Responses
│   ├── Server Logs
│   ├── Authentication-Results Header
│   └── Quarantine Metadata
├── Root Cause Summary
├── Test Reproduction Details
└── Source Files Examined
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract SMTP response format from `go-smtp/conn.go`'s `toSMTPStatus` and `WriteResponse` functions
- Extract SPF decision logic from `internal/check/spf/spf.go` with exact line references
- Extract DMARC evaluation logic from `internal/dmarc/evaluate.go` and `internal/dmarc/verifier.go`
- Generate runtime evidence by executing Go test files that exercise the actual pipeline code paths
- Capture server log output using `testutils.Logger` which records structured log entries during test execution

**Documentation Standards:**
- Markdown with fenced code blocks for SMTP responses and log output
- Tables for comparison summaries and DNS configuration
- Source citations as inline references: `spf.go:212`, `evaluate.go:148`, `verifier.go:143`
- All variable IDs (message IDs) replaced with `<id>` per user instruction
- All SMTP response lines given literally with code, enhanced status code, and message text

### 0.4.3 Diagram and Visual Strategy

No Mermaid diagrams are included in the deliverable document per the user's focus on exact SMTP responses and log lines. The comparison summary table serves as the primary visual aid. The code flow is described narratively with line number references rather than as diagrams.

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---|---|---|---|
| `blitzy/documentation/maddy_26452dd8dd78.md` | CREATE | `internal/check/spf/spf.go`, `internal/dmarc/evaluate.go`, `internal/dmarc/verifier.go`, `internal/msgpipeline/check_runner.go`, `internal/endpoint/smtp/smtp.go` | Complete Q&A document answering all three SMTP session reproduction cases with runtime evidence, root cause analysis, and verification methodology |

No existing documentation files are updated or deleted. No documentation configuration files exist to modify.

### 0.5.2 New Documentation Files Detail

```
File: blitzy/documentation/maddy_26452dd8dd78.md
Type: Technical Investigation Report / Q&A
Source Code:
    - internal/check/spf/spf.go (relyOnDMARC, CheckBody, spfResult)
    - internal/dmarc/evaluate.go (EvaluateAlignment, FetchRecord)
    - internal/dmarc/verifier.go (Apply)
    - internal/msgpipeline/check_runner.go (applyResults, runAndMergeResults)
    - internal/endpoint/smtp/smtp.go (Data, wrapErr)
    - go-smtp vendor (toSMTPStatus, WriteResponse)
Sections:
    - Summary
    - Controlled DNS Configuration
    - Case A reproduction (SMTP response, stage, logs, rationale)
    - Case B reproduction (SMTP response, stage, logs, rationale)
    - Case C reproduction (SMTP response, stage, logs, rationale)
    - Comparison Summary table
    - Root cause explanation (deferral mechanism, enforcement clearing, DMARC evaluation gap)
    - Verification methodology (observable signals)
    - Test reproduction details
    - Source files examined
Key Citations:
    - spf.go:212-235 (relyOnDMARC)
    - spf.go:351-363 (CheckBody deferral path)
    - evaluate.go:148 (!dkimPresent guard)
    - verifier.go:143 (ResultNone → PolicyNone)
    - check_runner.go:194 (no check action log)
    - check_runner.go:269 (DMARC Apply call)
```

### 0.5.3 Temporary Test Files Created (Not Tracked)

These files were created solely to gather runtime evidence and are not deliverables:

| File | Package | Purpose |
|---|---|---|
| `internal/msgpipeline/spf_dmarc_repro_test.go` | `msgpipeline` | Pipeline-level reproduction with DKIM present (3 cases + relyOnDMARC direct tests) |
| `internal/msgpipeline/spf_dmarc_nodkim_test.go` | `msgpipeline` | Pipeline-level reproduction with/without DKIM (6 sub-cases: the critical evidence) |
| `internal/msgpipeline/mock_dmarc.go` | `msgpipeline` | Exports `MockWithDMARC` function for cross-package test access |
| `internal/endpoint/smtp/spf_dmarc_smtp_test.go` | `smtp` | SMTP endpoint wire-level test with mock check injection |
| `internal/endpoint/smtp/spf_dmarc_real_test.go` | `smtp` | SMTP endpoint wire-level test with actual `apply_spf` check and DNS interception |

### 0.5.4 Cross-Documentation Dependencies

- No cross-documentation dependencies exist. The deliverable is a standalone document.
- No navigation links, table of contents updates, or index changes are required.
- No documentation configuration files (mkdocs.yml, docusaurus.config.js, etc.) exist in the repository.

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The documentation exercise relies on the following tools and packages for building and running the test reproduction:

| Registry | Package Name | Version | Purpose |
|---|---|---|---|
| system | `golang-go` | 1.22.2 | Go compiler and test runner for executing reproduction tests |
| go module | `github.com/foxcpp/go-mockdns` | v0.0.0-20191123143003 | Mock DNS server with `PatchNet` to intercept `net.DefaultResolver` for controlled DNS |
| go module | `github.com/emersion/go-smtp` | v0.12.1-0.20191206174923 | SMTP server library providing `toSMTPStatus`, `WriteResponse`, and `SMTPError` types |
| go module | `github.com/emersion/go-msgauth` | (pinned in go.sum) | `authres` package for `Authentication-Results` header formatting and result types |
| go module | `github.com/miekg/dns` | (pinned in go.sum) | Underlying DNS library used by go-mockdns server |
| go module | `github.com/foxcpp/maddy` (self) | (repo HEAD) | The mail server codebase under investigation |

All versions are the exact versions pinned in the repository's `go.sum` file. No additional dependencies were installed.

### 0.6.2 Documentation Reference Updates

Not applicable. No existing documentation files contain links that require updating. The new document is standalone and does not reference or link to other documentation files.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

The user's question poses exactly **7 deliverables** across the three cases and the explanation section:

| Deliverable | Status |
|---|---|
| Case A: SMTP response line after DATA `'.'` | Covered with runtime evidence |
| Case A: Outcome stage | Covered |
| Case A: Relevant server log lines | Covered |
| Case B: SMTP response line after DATA `'.'` | Covered with runtime evidence |
| Case B: Outcome stage | Covered |
| Case B: Relevant server log lines | Covered |
| Case C: SMTP response line after DATA `'.'` | Covered with runtime evidence |
| Case C: Outcome stage | Covered |
| Case C: Relevant server log lines | Covered |
| Why Case B is accepted despite SPF -all | Covered with code trace |
| What input condition triggers deferral | Covered (`policyDomain == fromDomain` + non-none policy) |
| Why Case A does not defer | Covered (`sp=none` substitution) |
| Which runtime component makes the decision | Covered (`relyOnDMARC` in `spf.go`) |
| How to verify using only observable signals | Covered (logs, Auth-Results, quarantine flag) |

Coverage: **14/14 (100%)** of user-requested items are documented with runtime evidence.

### 0.7.2 Documentation Quality Criteria

- **Completeness**: Every question posed by the user is answered with specific evidence
- **Accuracy**: All SMTP response lines, log lines, and Auth-Results headers are captured from actual test execution, not approximated
- **Traceability**: Every behavioral claim cites specific source file and line number (e.g., `spf.go:212`, `evaluate.go:148`)
- **Reproducibility**: The test files and their execution commands are documented so the investigation can be independently reproduced
- **Clarity**: The comparison summary table provides an at-a-glance view of all three cases; the root cause section builds progressively from mechanism to gap to impact

### 0.7.3 Example and Diagram Requirements

- **SMTP response examples**: 3 (one per case), presented as fenced code blocks with exact literal text
- **Log line examples**: 5 distinct log lines across the three cases, presented as fenced code blocks
- **Auth-Results examples**: 3 (one per case), presented as fenced code blocks
- **Code snippet references**: 3 critical code paths cited with exact line numbers
- **Comparison table**: 1 comprehensive table with 10 comparison dimensions across 3 cases
- **No diagrams required**: The user's questions are focused on exact SMTP responses and log lines, not architectural visualization

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

- **New documentation file**:
  - `blitzy/documentation/maddy_26452dd8dd78.md` — the primary deliverable answering the user's questions

- **Temporary test files created for evidence gathering** (not tracked, not deliverables):
  - `internal/msgpipeline/spf_dmarc_repro_test.go`
  - `internal/msgpipeline/spf_dmarc_nodkim_test.go`
  - `internal/msgpipeline/mock_dmarc.go`
  - `internal/endpoint/smtp/spf_dmarc_smtp_test.go`
  - `internal/endpoint/smtp/spf_dmarc_real_test.go`

- **Source files analyzed** (read-only, not modified):
  - `internal/check/spf/spf.go`
  - `internal/dmarc/evaluate.go`
  - `internal/dmarc/verifier.go`
  - `internal/dmarc/dmarc.go`
  - `internal/msgpipeline/check_runner.go`
  - `internal/msgpipeline/config.go`
  - `internal/msgpipeline/msgpipeline.go`
  - `internal/dns/resolver.go`
  - `internal/endpoint/smtp/smtp.go`
  - `internal/check/dkim/dkim.go`
  - `internal/testutils/check.go`
  - `internal/msgpipeline/dmarc_test.go`
  - `internal/dmarc/verifier_test.go`
  - `internal/endpoint/smtp/smtp_test.go`
  - `internal/msgpipeline/bench_test.go`
  - `go-smtp/conn.go` (vendor dependency)
  - `go-smtp/data.go` (vendor dependency)

### 0.8.2 Explicitly Out of Scope

- **Source code modifications**: No tracked repository files are modified per the user's explicit constraint
- **Feature additions or bug fixes**: The deferral gap is documented, not fixed
- **Test file modifications**: Existing test files (`dmarc_test.go`, `verifier_test.go`, `smtp_test.go`) are read-only
- **Configuration changes**: No `maddy.conf` or build configuration changes
- **Deployment configuration**: No systemd, Docker, or CI changes
- **IMAP, queue, storage, or other subsystem documentation**: Only SPF/DMARC interaction is in scope
- **Documentation for other checks**: `verify_dkim`, DNSBL, MTA-STS, and other check modules are mentioned only in context of the SPF/DMARC interaction
- **External documentation or website updates**: No upstream documentation changes

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Build command**: `cd /tmp/blitzy/maddy/maddy_26452dd8dd78_76465c && go build ./...`
- **Test execution (pipeline-level)**: `go test -v -run TestSPF_DMARC ./internal/msgpipeline/`
- **Test execution (SMTP endpoint)**: `go test -v -run TestSMTP_RealSPF_DMARC ./internal/endpoint/smtp/`
- **All existing tests (baseline verification)**: `go test ./internal/dmarc/ ./internal/msgpipeline/`
- **Default format**: Markdown with fenced code blocks for SMTP responses and log lines
- **Citation requirement**: Every behavioral claim references a source file and line number
- **Style guide**: Technical investigation report style — concise, evidence-first, with comparison tables
- **Variable ID handling**: All message IDs replaced with `<id>` per user instruction; all other content preserved literally

## 0.10 Rules for Documentation

The following documentation-specific rules are derived from the user's explicit instructions:

- **Do not modify any tracked repository files**: Only create new temporary test scripts/configs for reproduction; the deliverable markdown file is placed in `blitzy/documentation/` which is a new directory
- **Answer with runtime evidence**: All SMTP responses, log lines, and behavioral claims must be backed by actual test execution output, not static code analysis alone
- **Replace variable IDs with `<id>`**: Message IDs and other variable identifiers in SMTP responses and log lines are replaced with `<id>` while keeping everything else literal
- **Include exact SMTP response lines**: Report the full `code enhanced_code message` format (e.g., `250 2.0.0 OK: queued`) immediately after the DATA `'.'` terminator
- **Identify outcome stage precisely**: Report whether the outcome occurs at MAIL FROM, RCPT TO, or end of DATA
- **Report most relevant server log lines**: Extract the log lines that directly explain the pipeline's decision for each case
- **Explain using only observable signals**: The verification methodology must rely solely on SMTP responses and server logs, without requiring source code modification
- **Provide thinking and rationale**: Per the `SWE-AtlasQnA-Repo` implementation rule, include reasoning behind all answers
- **Base answers on the code as truth**: Do not make assumptions; all conclusions are grounded in actual source code and runtime behavior

## 0.11 References

### 0.11.1 Files and Folders Searched

The following files were retrieved and analyzed during the investigation:

**Core SPF/DMARC source files:**
- `internal/check/spf/spf.go` — `apply_spf` check module, `relyOnDMARC` function, `CheckBody`, `spfResult`, default fail actions
- `internal/dmarc/evaluate.go` — `EvaluateAlignment` function, `FetchRecord`, `ExtractFromDomain`
- `internal/dmarc/verifier.go` — `Verifier` struct, `Apply` method, `FetchRecord` async channel pattern
- `internal/dmarc/dmarc.go` — Type aliases (`Record`, `Policy`), `Resolver` interface

**Pipeline and endpoint source files:**
- `internal/msgpipeline/check_runner.go` — `checkRunner` struct, `runAndMergeResults`, `applyResults`, `checkBody`, `doDMARC` flag
- `internal/msgpipeline/config.go` — `msgpipelineCfg` struct, `doDMARC` parsing from `dmarc yes` directive
- `internal/msgpipeline/msgpipeline.go` — `MsgPipeline` struct, `Mock` function, pipeline lifecycle
- `internal/endpoint/smtp/smtp.go` — `Session.Data` handler, `wrapErr` error-to-SMTP mapping
- `internal/dns/resolver.go` — `DefaultResolver()` returning `net.DefaultResolver`
- `internal/check/dkim/dkim.go` — `verify_dkim` module (for understanding its absence)

**Test infrastructure files:**
- `internal/testutils/check.go` — `Check` mock struct with `BodyRes`, `ConnRes`, etc.
- `internal/msgpipeline/dmarc_test.go` — Existing DMARC pipeline tests (baseline verification)
- `internal/dmarc/verifier_test.go` — Existing verifier unit tests
- `internal/endpoint/smtp/smtp_test.go` — Existing SMTP endpoint test patterns (`testEndpoint`, `submitMsg`)
- `internal/msgpipeline/bench_test.go` — Pipeline construction patterns with `msgpipelineCfg`
- `internal/msgpipeline/bodynonatomic_test.go` — Additional pipeline construction patterns
- `internal/msgpipeline/check_test.go` — Check pipeline test patterns

**Vendor/dependency files:**
- `go-smtp/conn.go` (at `$GOMODCACHE/.../go-smtp@.../conn.go`) — `toSMTPStatus`, `WriteResponse`, `handleData`
- `go-smtp/data.go` — `SMTPError` type definition

**Folder searches:**
- Repository root (`""`) — initial structure discovery
- `internal/check/` — check modules folder
- `internal/dmarc/` — DMARC library folder
- `internal/msgpipeline/` — pipeline implementation folder
- `internal/endpoint/smtp/` — SMTP endpoint folder
- `internal/dns/` — DNS resolver wrapper folder
- `internal/testutils/` — test utilities folder

### 0.11.2 Attachments and External Resources

No attachments were provided by the user. No Figma screens were referenced. No external URLs were specified.

### 0.11.3 Runtime Evidence Sources

All runtime evidence was generated by executing Go tests within the repository:

| Test Suite | Command | Result |
|---|---|---|
| Existing DMARC tests (baseline) | `go test ./internal/dmarc/ ./internal/msgpipeline/` | PASS |
| Pipeline SPF/DMARC with DKIM | `go test -v -run TestSPF_DMARC_Interaction ./internal/msgpipeline/` | PASS |
| Pipeline SPF/DMARC ± DKIM | `go test -v -run TestSPF_DMARC_NoDKIM ./internal/msgpipeline/` | PASS |
| relyOnDMARC direct test | `go test -v -run TestRelyOnDMARC_ThreeCases ./internal/msgpipeline/` | PASS |
| SMTP endpoint with mock checks | `go test -v -run TestSMTP_SPF_DMARC_ThreeCases ./internal/endpoint/smtp/` | PASS |
| SMTP endpoint with real SPF check | `go test -v -run TestSMTP_RealSPF_DMARC ./internal/endpoint/smtp/` | PASS |

