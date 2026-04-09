# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that answers a detailed empirical question about how the Maddy mail server enforces sender identity and message authentication at runtime.

**Category:** Create new documentation
**Documentation Type:** Technical investigation report — a runtime-behavior analysis document grounded in direct observation rather than source-code inference

The user requires a comprehensive investigation-based document that:

- Establishes a working Maddy deployment from this repository with submission (authenticated SMTP), DKIM signing, and local mailbox delivery
- Creates at least two user accounts to enable cross-user testing
- Conducts controlled experiments that probe security boundary behavior across three axes:
  - **Envelope sender spoofing**: Authenticate as one user but specify a different user's address in `MAIL FROM`
  - **Domain mismatch**: Use a `MAIL FROM` domain that does not match any configured DKIM signing domain
  - **Legitimate baseline**: Send a correctly aligned message that should be signed and accepted
- Captures and presents **raw runtime evidence**: actual SMTP response codes, actual stored message headers (DKIM-Signature, Received, Authentication-Results), and actual SMTP protocol transcripts
- Tests the interaction between the From header, authenticated identity, and DKIM signing — specifically whether signing still occurs when headers are misaligned
- Determines whether Maddy's default configuration enforces sender alignment with authenticated identity, or whether this requires explicit policy configuration
- Rules out at least one plausible but incorrect interpretation of observed behavior using specific evidence
- Identifies one case where Maddy's security behavior differs from what a reasonable configuration reading would suggest
- Leaves the repository source code unchanged, noting any configuration files or test artifacts created, and cleaning up database state afterward

### 0.1.2 Special Instructions and Constraints

- **Repository integrity rule (SWE-AtlasQnA-Repo):** The generated document must be a new markdown file named after the project and placed in `blitzy/documentation/`. No existing source files in the repository may be modified. Answers must be based on the code as the source of truth, not assumptions.
- **Runtime evidence requirement:** The user explicitly states "Based only on what you can observe at runtime" — no conclusions drawn from source-reading alone are acceptable without corroborating runtime evidence
- **Raw header capture:** The user requires "the actual header content, not a summary of what you expect to find"
- **SMTP transaction capture:** At least one rejected and one accepted message transaction must be captured
- **Cleanup requirement:** Configuration files and test artifacts are allowed, but database state must be cleaned up afterward
- **Alternative approaches:** If Maddy's debug logging is insufficient, the document must show what was tried and use alternative approaches for protocol visibility
- **Falsification requirement:** At least one plausible but incorrect interpretation must be explicitly ruled out with evidence
- **Surprise finding:** One case where behavior differs from a reasonable config reading must be identified and supported by runtime observation

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **set up Maddy from this repository**, we will build the Go binary from source, create a non-TLS submission endpoint (for local testing), configure DKIM signing for a test domain, initialize the SQLite storage backend, and create two user accounts using `maddyctl`
- To **test sender identity enforcement**, we will use SMTP client tools (e.g., `swaks`, raw `nc`/telnet, or Python `smtplib`) to send messages through the submission endpoint with varying MAIL FROM and From header combinations against the two accounts
- To **capture SMTP transactions**, we will enable Maddy's `io_debug` mode (which logs full protocol I/O at the endpoint level, per `internal/endpoint/smtp/smtp.go` lines 603-606) and/or use network capture tools
- To **inspect delivered message headers**, we will use `maddyctl` to dump messages from local mailboxes or directly inspect the SQLite storage
- To **document DKIM signing behavior**, we will analyze the `sign_dkim` modifier's `shouldSign()` function behavior at runtime by observing which messages get DKIM-Signature headers and which do not, and correlating with debug logs
- To **answer the sender alignment question**, we will observe whether the `source $(local_domains)` routing in the default `maddy.conf` submission block rejects or accepts cross-user MAIL FROM addresses, and whether the `sign_dkim` modifier's default `require_sender_match` policy (`["envelope", "auth"]`) silently skips signing vs. rejecting messages

### 0.1.4 Inferred Documentation Needs

Based on deep code analysis, the following implicit documentation needs have been identified:

- **Layered enforcement model explanation:** Maddy enforces sender identity at two distinct layers that must be documented separately — (1) the message pipeline's `source` routing rejects based on *domain* in MAIL FROM, not per-user identity (`internal/msgpipeline/msgpipeline.go` `srcBlockForAddr()`), and (2) the `sign_dkim` modifier's `shouldSign()` function applies per-user identity checks but only determines whether to *sign*, not whether to *reject* (`internal/modify/dkim/dkim.go` lines 249-330)
- **Default require_sender_match values:** The default `require_sender_match` is `["envelope", "auth"]` (`internal/modify/dkim/dkim.go` line 152), meaning by default DKIM signing requires both envelope and authenticated identity alignment — but this is a signing decision, not a delivery decision
- **Gap in user-level sender enforcement:** The default `maddy.conf` submission block enforces domain-level sender restrictions (`source $(local_domains)` with `default_source { reject }`) but does NOT enforce that authenticated user A cannot send as user B within the same local domain — this is a key finding that will need runtime verification
- **DontTraceSender behavior:** The submission endpoint sets `DontTraceSender = true` (`internal/endpoint/smtp/submission.go` line 28), which suppresses source host/IP from Received headers — this will be visible in captured headers and needs explanation
- **Authentication-Results generation:** The check runner in `internal/msgpipeline/check_runner.go` (lines 262-309) generates Authentication-Results headers using the `authres.Format()` function — but the submission endpoint's default config has no `check {}` block, so Authentication-Results may be absent from submission-originated messages


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a documentation structure centered on MkDocs with a `readthedocs` theme, configured via `.mkdocs.yml` at the repository root. The documentation is organized into four main sections: Tutorials, manual pages (generated from `scdoc` sources), an installation script reference, and an internals knowledge base.

- **Current documentation framework:** MkDocs (version not pinned in repository; configured in `.mkdocs.yml`)
- **Documentation generator configuration:** `.mkdocs.yml` at repository root — defines site name (`maddy documentation`), repo URL, theme (`readthedocs`), markdown extensions (`codehilite`), and full navigation tree
- **API documentation tools in use:** None — Maddy does not use Go doc generators (godoc) for user-facing documentation; internal code documentation is through Go comments and `HACKING.md`
- **Diagram tools detected:** None configured in existing documentation infrastructure; Mermaid diagrams are used in the tech spec but not in the repository's own docs
- **Manual page toolchain:** `scdoc` format compiled to man pages; `docs/man/prepare_md.py` converts scdoc sources to Markdown for the MkDocs site; generated files are prefixed with `_generated_`
- **Documentation hosting:** Published at `https://foxcpp.dev/maddy/` per `README.md`

**Existing documentation files discovered:**

| File | Type | Description |
|------|------|-------------|
| `docs/README.md` | Landing page | Project overview, capabilities, links |
| `docs/tutorials/setting-up.md` | Tutorial | End-to-end deployment guide with DNS, TLS, user creation |
| `docs/tutorials/manual-installation.md` | Tutorial | Source build and first-run procedure |
| `docs/tutorials/alias-to-remote.md` | Tutorial | Mail routing via aliases |
| `docs/get.sh-script.md` | Reference | Bootstrap installer documentation |
| `docs/internals/sqlite.md` | Internal | SQLite backend behavior and constraints |
| `docs/internals/quirks.md` | Internal | SMTP/IMAP protocol quirks |
| `docs/man/README.md` | Guide | Manpage toolchain workflow |
| `docs/man/prepare_md.py` | Tool | scdoc-to-Markdown converter |
| `HACKING.md` | Contributor guide | Architecture, module patterns, design goals |
| `README.md` | Project overview | Feature summary, installation links, community |
| `examples/multitentant-dkim.conf` | Example | Multi-domain DKIM signing configuration |
| `examples/README.md` | Index | Advanced configuration example guidance |

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used for code relevant to sender identity and DKIM:

- **SMTP endpoint handling:** `internal/endpoint/smtp/smtp.go` — Session lifecycle, MAIL FROM processing, authentication flow, submission detection
- **Submission-specific behavior:** `internal/endpoint/smtp/submission.go` — Header validation, `DontTraceSender` flag, From/Sender/Date enforcement
- **DKIM signing module:** `internal/modify/dkim/dkim.go` — `shouldSign()` decision logic, `require_sender_match` configuration, field-to-sign selection, signing execution
- **DKIM key management:** `internal/modify/dkim/keys.go` — Key generation (Ed25519/RSA), filesystem persistence, DNS record output
- **DKIM signing tests:** `internal/modify/dkim/dkim_test.go` — Unit tests for `shouldSign()` with envelope/auth match methods, field selection
- **Message pipeline:** `internal/msgpipeline/msgpipeline.go` — Source block routing via `srcBlockForAddr()`, modifier application at body stage
- **Pipeline configuration:** `internal/msgpipeline/config.go` — Source/destination/reject directive parsing, modifier group wiring
- **Authentication framework:** `internal/auth/auth.go` — `CheckDomainAuth()` for username normalization and domain allow-list
- **SQL storage authentication:** `internal/storage/sql/sql.go` (lines 351-392) — `prepareUsername()` with PRECIS, `CheckPlain()` with bcrypt
- **User management CLI:** `internal/storage/sql/maddyctl.go` — `CreateUser()`, `SetUserPassword()`, `ListUsers()`, `DeleteUser()`
- **Module metadata:** `internal/module/msgmetadata.go` — `ConnState` (with `AuthUser` field), `MsgMetadata` (with `OriginalFrom`, `DontTraceSender`)
- **Received header generation:** `internal/target/received.go` — `GenerateReceived()` function with sender tracing suppression
- **Check runner and AuthResults:** `internal/msgpipeline/check_runner.go` — Authentication-Results header generation via `authres.Format()`
- **Default configuration:** `maddy.conf` — Full default server config with submission, DKIM, source routing, reject directives

**Key directories examined:**

| Directory | Relevance |
|-----------|-----------|
| `internal/endpoint/smtp/` | SMTP/submission session handling, auth enforcement |
| `internal/modify/dkim/` | DKIM signing logic and policy gates |
| `internal/auth/` | Authentication domain logic |
| `internal/storage/sql/` | SQL auth backend and user management |
| `internal/msgpipeline/` | Message routing, source matching, modifier execution |
| `internal/module/` | Core interfaces, ConnState, MsgMetadata |
| `internal/target/` | Received header generation |
| `internal/check/dkim/` | Inbound DKIM verification |
| `cmd/maddyctl/` | Administrative CLI for user/mailbox management |
| `examples/` | Multi-tenant DKIM configuration example |
| `docs/tutorials/` | Deployment tutorials |
| `docs/internals/` | Implementation notes |

### 0.2.3 Web Search Research Conducted

No external web searches are required for this documentation task. All information needed to understand Maddy's sender identity enforcement and DKIM signing behavior is fully contained within the repository source code, configuration files, and tests. The investigation is explicitly grounded in runtime observations from this specific codebase rather than external documentation or best-practice guides.

The Go toolchain version (go 1.13 per `go.mod`) and all dependencies are pinned in `go.mod` and `go.sum`, providing a fully reproducible build environment.


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules contain the logic directly relevant to the documentation's subject matter — sender identity enforcement and DKIM signing at runtime:

- **Module: `internal/endpoint/smtp/smtp.go`**
  - Public APIs: `Endpoint.Login()`, `Endpoint.AnonymousLogin()`, `Session.Mail()`, `Session.startDelivery()`, `Session.Data()`, `Session.prepareBody()`
  - Current documentation: No dedicated documentation exists for submission sender enforcement behavior
  - Documentation needed: Runtime behavior description of how the authenticated user identity (`ConnState.AuthUser`) flows through the pipeline and where (or if) it is checked against MAIL FROM

- **Module: `internal/endpoint/smtp/submission.go`**
  - Public APIs: `Session.submissionPrepare()`
  - Current documentation: Code comments only; no external documentation
  - Documentation needed: Description of what submission preparation validates (From, Sender, Date, Message-ID headers) and what it does NOT validate (no MAIL FROM vs. AuthUser comparison)

- **Module: `internal/modify/dkim/dkim.go`**
  - Public APIs: `Modifier.shouldSign()`, `state.RewriteBody()`, `Modifier.Init()` (particularly `require_sender_match` configuration)
  - Current documentation: Man pages reference `sign_dkim` directive but do not explain the runtime sender-match decision tree
  - Documentation needed: Full decision tree for `shouldSign()` — From header domain check, envelope match, auth identity match, and the critical insight that this is a *sign-or-skip* decision, not a *sign-or-reject* decision

- **Module: `internal/msgpipeline/msgpipeline.go`**
  - Public APIs: `MsgPipeline.Start()`, `msgpipelineDelivery.srcBlockForAddr()`, `msgpipelineDelivery.Body()`
  - Current documentation: Architecture described in `HACKING.md` at a high level only
  - Documentation needed: How `source` routing uses domain matching from MAIL FROM (not authenticated identity), and how modifiers execute during the Body phase

- **Module: `maddy.conf` (submission block, lines 93-120)**
  - Configuration directives: `submission`, `auth`, `source $(local_domains)`, `sign_dkim`, `default_source { reject }`
  - Current documentation: Comments in config file; `docs/tutorials/setting-up.md` covers basic setup but not sender enforcement semantics
  - Documentation needed: Analysis of what the default config actually enforces at each pipeline stage

- **Module: `internal/storage/sql/maddyctl.go`**
  - Public APIs: `CreateUser()`, `DeleteUser()`, `ListUsers()`
  - Current documentation: `cmd/maddyctl/` README and man page reference
  - Documentation needed: Commands to create/delete test users for the investigation

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the following documentation gaps exist that this task will address:

- **No existing document** explains how Maddy decides whether to accept or reject a sender address from an authenticated session at the SMTP protocol level
- **No existing document** captures actual SMTP response codes and enhanced status codes for various sender mismatch scenarios
- **No existing document** shows raw DKIM-Signature headers as stored in delivered messages
- **No existing document** explains the distinction between domain-level enforcement (pipeline source routing) and user-level enforcement (DKIM sender match) — and the fact that user-level sender spoofing within the same domain is accepted by the pipeline but results in unsigned messages
- **No existing document** presents the `require_sender_match` default behavior with runtime evidence
- **No existing document** addresses what happens to DKIM signing when the From header diverges from the authenticated identity
- **No existing document** captures Received header differences between submission (with `DontTraceSender`) and inbound SMTP delivery
- **The `docs/tutorials/setting-up.md`** covers deployment but does not address security posture testing or sender enforcement verification
- **The `examples/multitentant-dkim.conf`** demonstrates per-domain DKIM but does not document per-user enforcement behavior


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document will be a single comprehensive markdown file placed at `blitzy/documentation/maddy.md` per the SWE-AtlasQnA-Repo rule. The document follows an investigation-report structure:

```
blitzy/
└── documentation/
    └── maddy.md
        ├── Introduction and Objectives
        ├── Test Environment Setup
        │   ├── Build from Source
        │   ├── Configuration (maddy.conf for testing)
        │   ├── DKIM Key Generation
        │   ├── User Account Creation
        │   └── Environment Verification
        ├── Test 1: Cross-User Sender Spoofing
        │   ├── Test Description
        │   ├── SMTP Transaction Capture
        │   ├── Observed Behavior and Response Codes
        │   └── Analysis
        ├── Test 2: Non-Local Domain MAIL FROM
        │   ├── Test Description
        │   ├── SMTP Transaction Capture
        │   ├── Observed Behavior and Response Codes
        │   └── Analysis
        ├── Test 3: Legitimate Aligned Message
        │   ├── Test Description
        │   ├── SMTP Transaction Capture
        │   ├── Raw Stored Headers
        │   └── Analysis
        ├── Test 4: From Header vs. Auth Identity for DKIM
        │   ├── Test Description
        │   ├── SMTP Transaction Capture
        │   ├── Raw Stored Headers (DKIM-Signature)
        │   └── Analysis
        ├── DKIM Signing Decision Analysis
        │   ├── Default require_sender_match Behavior
        │   ├── Fields Covered by Signature
        │   └── Recipient Verification Perspective
        ├── Sender Enforcement Model
        │   ├── Layer 1: Pipeline Source Routing (Domain-Level)
        │   ├── Layer 2: DKIM Sender Match (User-Level Signing)
        │   ├── Gap: No User-Level MAIL FROM Rejection
        │   └── Ruling Out Incorrect Interpretation
        ├── Surprise Finding
        │   ├── Observed Behavior vs. Configuration Expectation
        │   └── Supporting Evidence
        ├── Cleanup and Artifacts
        │   ├── Files Created
        │   └── Database State Cleanup
        └── Conclusions
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract the submission endpoint's sender routing behavior by building and running Maddy with `io_debug` enabled, then sending test messages via SMTP and capturing the complete protocol exchange from debug logs
- Extract DKIM signing decisions by enabling debug logging on the `sign_dkim` modifier (the `Modifier.log` with `Debug: true`), which emits `shouldSign()` decision messages including "not signing, From address is not authenticated identity" and "not signing, From domain is not key domain"
- Extract delivered message headers by using `maddyctl` message dump commands or direct SQLite inspection of the `all.db` database
- Generate test messages using command-line SMTP tools (Python `smtplib`, `swaks`, or raw TCP via `openssl s_client` / `nc`) to maintain full control over MAIL FROM, From headers, and authentication credentials

**Test Configuration Approach:**

The test configuration will adapt the default `maddy.conf` for local testing:
- Replace `tls://0.0.0.0:465` with `tcp://127.0.0.1:587` (plaintext submission on localhost for testability; `insecure_auth` enabled)
- Set `$(hostname)` and `$(primary_domain)` to a test domain (e.g., `test.local`)
- Keep all default pipeline routing (source matching, DKIM signing, reject rules) intact to test default behavior
- Enable `io_debug` on the submission endpoint for protocol capture
- Enable `debug` on the `sign_dkim` modifier for signing decision logs

### 0.4.3 Diagram and Visual Strategy

The document will include the following Mermaid diagrams to illustrate runtime findings:

- **Sender Enforcement Decision Tree:** A flowchart showing the two-layer enforcement model discovered at runtime — pipeline source routing (domain check) followed by DKIM sender match (user check), with outcomes at each decision point
- **DKIM Signing Decision Flow:** A flowchart mapping the `shouldSign()` logic as confirmed by runtime observation — From domain check → envelope match → auth identity match → sign/skip outcome
- **Message Flow for Each Test Case:** Sequence diagrams showing the SMTP dialog, pipeline processing, and delivery outcome for each of the four test scenarios

### 0.4.4 Documentation Standards

- Markdown formatting with proper headers (`#`, `##`, `###`)
- All SMTP transcripts presented in fenced code blocks with transcript-style formatting
- All raw headers presented in fenced code blocks preserving exact content
- Source citations as inline references: `Source: internal/modify/dkim/dkim.go:249-330`
- Tables for parameter descriptions, test case summaries, and response code mappings
- Consistent terminology: "MAIL FROM" (envelope sender), "From header" (message header), "authenticated identity" (SASL username), "signing domain" (DKIM domain)


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/maddy.md` | CREATE | `internal/endpoint/smtp/smtp.go`, `internal/endpoint/smtp/submission.go`, `internal/modify/dkim/dkim.go`, `internal/modify/dkim/dkim_test.go`, `internal/msgpipeline/msgpipeline.go`, `internal/msgpipeline/config.go`, `internal/module/msgmetadata.go`, `internal/target/received.go`, `internal/msgpipeline/check_runner.go`, `internal/auth/auth.go`, `internal/storage/sql/sql.go`, `internal/storage/sql/maddyctl.go`, `maddy.conf`, `examples/multitentant-dkim.conf` | Complete runtime investigation document answering how Maddy enforces sender identity and DKIM signing, with raw SMTP transcripts, stored headers, response codes, and analysis |

This is the sole documentation file to be created. No existing documentation files are updated, deleted, or used as structural references — the SWE-AtlasQnA-Repo rule explicitly prohibits modifying existing repository files.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/maddy.md
Type: Technical Investigation Report (QnA)
Source Code: 
  - internal/endpoint/smtp/smtp.go (session handling, auth, MAIL FROM)
  - internal/endpoint/smtp/submission.go (submission prepare, header validation)
  - internal/modify/dkim/dkim.go (shouldSign decision logic, signing)
  - internal/modify/dkim/keys.go (key generation, DNS record)
  - internal/modify/dkim/dkim_test.go (shouldSign test cases)
  - internal/msgpipeline/msgpipeline.go (source routing, modifier execution)
  - internal/msgpipeline/config.go (source/reject directive parsing)
  - internal/module/msgmetadata.go (ConnState.AuthUser, DontTraceSender)
  - internal/target/received.go (Received header generation)
  - internal/msgpipeline/check_runner.go (Authentication-Results)
  - internal/auth/auth.go (CheckDomainAuth)
  - internal/storage/sql/sql.go (CheckPlain, prepareUsername)
  - internal/storage/sql/maddyctl.go (CreateUser, DeleteUser)
  - maddy.conf (default submission config, source routing, DKIM)
  - examples/multitentant-dkim.conf (multi-domain DKIM reference)
Sections:
  - Introduction and Objectives (problem statement, methodology)
  - Test Environment Setup (build, config, DKIM keys, user accounts)
  - Test 1: Cross-User Sender Spoofing (auth as user A, MAIL FROM user B)
  - Test 2: Non-Local Domain MAIL FROM (domain not in local_domains)
  - Test 3: Legitimate Aligned Message (properly authenticated, signed)
  - Test 4: From Header vs Auth Identity for DKIM (header mismatch signing)
  - DKIM Signing Decision Analysis (require_sender_match default behavior)
  - Sender Enforcement Model (two-layer analysis with gap identification)
  - Surprise Finding (behavior differing from config reading)
  - Cleanup and Artifacts (files created, database cleanup)
  - Conclusions (answers to all posed questions)
Diagrams:
  - Sender Enforcement Decision Tree (Mermaid flowchart)
  - DKIM shouldSign Decision Flow (Mermaid flowchart)
  - SMTP Transaction Sequence (Mermaid sequence diagrams per test)
Key Citations:
  - internal/modify/dkim/dkim.go:152 (require_sender_match default)
  - internal/modify/dkim/dkim.go:249-330 (shouldSign logic)
  - internal/endpoint/smtp/smtp.go:83-160 (startDelivery, MAIL FROM handling)
  - internal/endpoint/smtp/submission.go:27-130 (submissionPrepare)
  - internal/msgpipeline/msgpipeline.go:155-202 (srcBlockForAddr)
  - maddy.conf:93-120 (submission block)
  - maddy.conf:117-119 (default_source reject)
  - internal/module/msgmetadata.go:37 (AuthUser)
  - internal/target/received.go:19-87 (GenerateReceived with DontTraceSender)
```

### 0.5.3 Runtime Artifacts (Temporary, Not Committed)

The following temporary artifacts will be created during testing and documented in the output, but are NOT documentation files:

| Artifact | Purpose | Lifecycle |
|----------|---------|-----------|
| Test `maddy.conf` | Modified config for local testing (plaintext submission, io_debug) | Created for testing, documented in report, not committed |
| DKIM key files (`dkim_keys/test.local_default.key`, `.dns`) | Auto-generated by `sign_dkim` on first run | Documented in report, cleaned up |
| SQLite database (`all.db`) | Stores user accounts and mailboxes | User accounts deleted, database state cleaned up |
| SMTP debug logs | Protocol transcripts from `io_debug` | Captured in report, logs ephemeral |
| Test email messages | Messages sent through the submission endpoint | Inspected via maddyctl, documented, then cleaned up |

### 0.5.4 Documentation Configuration Updates

No documentation configuration updates are needed. The new file `blitzy/documentation/maddy.md` is a standalone document that does not integrate into the MkDocs navigation tree (`.mkdocs.yml`) or any other documentation build system. Per the SWE-AtlasQnA-Repo rule, existing files are not modified.

### 0.5.5 Cross-Documentation Dependencies

- The document will reference code paths by their repository-relative file paths for traceability
- Internal cross-references within the document will use markdown anchor links between sections
- No shared content/includes, no navigation link updates, no index/glossary changes required


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following tools and packages are required to execute the runtime investigation and produce the documentation:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| Go module | `github.com/foxcpp/maddy` | HEAD (repository) | Mail server under investigation |
| Go toolchain | `go` | 1.13+ (per `go.mod`) | Build the maddy binary from source |
| Go module | `github.com/emersion/go-smtp` | v0.12.1 | SMTP protocol handling (dependency) |
| Go module | `github.com/emersion/go-msgauth` | v0.3.2 | DKIM signing/verification (dependency) |
| Go module | `github.com/emersion/go-imap` | v1.0.1 | IMAP protocol handling (dependency) |
| Go module | `github.com/mattn/go-sqlite3` | v1.11.0 | SQLite3 storage driver (requires CGO) |
| Go module | `github.com/foxcpp/go-imap-sql` | v0.3.2 | SQL-backed IMAP storage backend |
| Go module | `github.com/urfave/cli` | v1.22.1 | CLI framework for maddyctl |
| Go module | `golang.org/x/crypto` | v0.0.0-20191108234033 | Bcrypt password hashing |
| Go module | `golang.org/x/text` | v0.3.2 | PRECIS username/password normalization |
| Go module | `github.com/miekg/dns` | v1.1.22 | DNS with DNSSEC support |
| Go module | `blitiri.com.ar/go/spf` | v0.0.0-20191018194539 | SPF evaluation |
| System | `gcc` / `build-essential` | System | C compiler required for SQLite3 CGO |
| System | `sqlite3` | System | Direct database inspection (optional) |
| Tool | `swaks` or Python `smtplib` | Any | SMTP test client for sending test messages |
| Tool | `openssl s_client` or `nc` | System | Raw SMTP protocol interaction (fallback) |

### 0.6.2 Build Dependencies from go.mod

All Go dependencies are pinned in `go.mod` (Go 1.13 minimum) and verified via `go.sum`. The complete dependency list includes 37 direct and indirect modules. Key security-relevant dependencies for this investigation:

| Dependency | Pinned Version | Role in Investigation |
|------------|---------------|----------------------|
| `emersion/go-msgauth` | v0.3.2-20191028231513 | Implements DKIM signing via `dkim.NewSigner()` and `dkim.SignOptions` — the actual signing operation observed at runtime |
| `emersion/go-smtp` | v0.12.1-20191206174923 | Implements SMTP session handling, AUTH, MAIL FROM, DATA commands — the protocol layer where rejection occurs |
| `emersion/go-sasl` | v0.0.0-20190817083125 | SASL PLAIN authentication mechanism used by submission endpoint |
| `mattn/go-sqlite3` | v1.11.0 | SQLite driver for storing users, mailboxes, and messages locally — the storage inspected for delivered headers |
| `foxcpp/go-imap-sql` | v0.3.2-20191208094750 | SQL-backed IMAP storage with user creation, message storage, mailbox management |

### 0.6.3 Documentation Reference Updates

Not applicable. No existing documentation links need updating as the new document is standalone and no existing files are modified.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage of sender identity enforcement documentation:** 0% — no existing document in the repository addresses the runtime sender enforcement behavior of the submission endpoint or the DKIM signing decision tree with empirical evidence.

**Target coverage after this task:**

| Topic | Current | Target | Method |
|-------|---------|--------|--------|
| Sender identity enforcement at SMTP level | 0% | 100% | Runtime testing with response code capture |
| DKIM signing decision tree (shouldSign) | 0% (code comments only) | 100% | Runtime observation with debug logs |
| Stored message header analysis (DKIM-Signature, Received, Auth-Results) | 0% | 100% | Direct mailbox inspection |
| Cross-user sender spoofing behavior | 0% | 100% | Controlled test with two accounts |
| Non-local domain rejection behavior | 0% | 100% | Test with mismatched domain |
| Default require_sender_match behavior | 0% | 100% | Runtime signing observation |
| DontTraceSender effect on Received headers | 0% | 100% | Header comparison |
| Surprise finding (config vs. behavior divergence) | 0% | 100% | Empirical observation |

**Coverage gaps to address:**
- The entire subject domain (sender enforcement at runtime) is currently undocumented
- Focus areas: SMTP response codes for rejection scenarios, DKIM signing vs. non-signing differentiation, user-level vs. domain-level enforcement distinction

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every question posed in the user's prompt must be answered with runtime evidence
- All SMTP response codes and enhanced status codes must be captured from actual protocol exchanges, not inferred from source code
- Raw headers must be presented verbatim from stored messages, not summarized or paraphrased
- At least one rejected and one accepted SMTP transaction must be captured in full
- At least one plausible but incorrect interpretation must be explicitly ruled out
- One case of behavior diverging from config reading must be identified and evidenced

**Accuracy validation:**
- Every claim about runtime behavior must be supported by captured logs, SMTP transcripts, or stored headers
- Source code citations are used to explain WHY observed behavior occurs, but the behavior itself must be established through runtime observation
- Where debug logging is insufficient, alternative capture methods must be attempted and documented

**Clarity standards:**
- Technical accuracy with accessible language — a reader familiar with SMTP but not Maddy internals should understand the findings
- Progressive disclosure: test setup → observations → analysis → conclusions
- Consistent terminology throughout (MAIL FROM vs. From header vs. authenticated identity)

**Maintainability:**
- Source code file paths cited for all technical claims
- Line numbers referenced for critical code paths
- Test methodology documented so tests can be reproduced

### 0.7.3 Example and Diagram Requirements

- **Minimum SMTP transcript examples:** 4 (one per test case: cross-user spoof, non-local domain, legitimate, From header mismatch)
- **Minimum raw header captures:** 2 (at least one properly signed message, one unsigned/rejected message)
- **Diagram types required:**
  - 1 flowchart: Sender enforcement decision tree (2 layers)
  - 1 flowchart: DKIM `shouldSign()` decision flow
  - At least 1 sequence diagram: SMTP transaction flow
- **Code example testing:** All SMTP test commands must be executable and reproducible
- **Visual content freshness:** All diagrams reflect the actual code at HEAD of this repository


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation file:**
- `blitzy/documentation/maddy.md` — The sole deliverable: a comprehensive markdown document answering all questions about sender identity enforcement and DKIM signing at runtime

**Runtime testing activities (produce evidence for the document):**
- Building Maddy from repository source (`go build`)
- Creating a test configuration file adapted from `maddy.conf` for local testing
- Generating DKIM keys (auto-generated by `sign_dkim` on first run)
- Creating two test user accounts via `maddyctl users create`
- Sending test messages through the submission endpoint with various MAIL FROM/From header combinations
- Capturing SMTP protocol transcripts from `io_debug` logs
- Inspecting delivered message headers in the local mailbox storage
- Capturing signing decision logs from `sign_dkim` debug output
- Analyzing and documenting all observed SMTP response codes and error messages
- Cleaning up test artifacts (user accounts, database state)

**Source code files analyzed for explanation of observed behavior:**
- `internal/endpoint/smtp/smtp.go` — MAIL FROM handling, auth flow
- `internal/endpoint/smtp/submission.go` — Submission header validation
- `internal/modify/dkim/dkim.go` — DKIM signing decision logic
- `internal/modify/dkim/keys.go` — Key generation
- `internal/modify/dkim/dkim_test.go` — Signing decision test cases
- `internal/msgpipeline/msgpipeline.go` — Source routing, modifier execution
- `internal/msgpipeline/config.go` — Pipeline configuration parsing
- `internal/msgpipeline/check_runner.go` — Authentication-Results generation
- `internal/module/msgmetadata.go` — ConnState, MsgMetadata structures
- `internal/target/received.go` — Received header generation
- `internal/auth/auth.go` — Domain auth checking
- `internal/storage/sql/sql.go` — Authentication implementation
- `internal/storage/sql/maddyctl.go` — User management
- `maddy.conf` — Default configuration
- `examples/multitentant-dkim.conf` — Multi-domain DKIM example

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No Go source files, test files, or build files will be modified (per SWE-AtlasQnA-Repo rule)
- **Existing documentation modifications:** No changes to `README.md`, `HACKING.md`, `docs/**`, `examples/**`, or any other existing file
- **MkDocs configuration changes:** `.mkdocs.yml` will not be modified
- **Feature additions or code refactoring:** No functional changes to Maddy's behavior
- **Inbound SMTP (port 25) testing:** The investigation focuses on the submission endpoint (authenticated sending), not inbound MX delivery
- **IMAP functionality testing:** Mailbox access is only used to inspect delivered messages, not to test IMAP behavior
- **Remote delivery testing:** No messages will be sent to external servers; all delivery is local
- **TLS certificate management:** The test configuration uses plaintext on localhost; TLS setup is out of scope
- **DNS record configuration:** No actual DNS records are created; DKIM DNS validation is not tested (signing behavior is the focus)
- **Fail2ban integration testing:** Not relevant to sender identity enforcement
- **SPF/DMARC verification testing:** Inbound verification checks are out of scope; the focus is outbound signing and submission sender enforcement
- **Performance testing or benchmarking:** Not relevant to this investigation
- **Multi-tenant/multi-domain configuration:** While `examples/multitentant-dkim.conf` is referenced for context, multi-domain setup is not tested
- **PAM or shadow authentication backends:** Only the SQL backend is tested


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Build command:** `CGO_ENABLED=1 go build -o maddy ./cmd/maddy/` and `CGO_ENABLED=1 go build -o maddyctl ./cmd/maddyctl/` (CGO required for SQLite3 support per `internal/storage/sql/sqlite3.go` build tag `!nosqlite3,cgo`)
- **Runtime command:** `./maddy -config <test-config-path>` (run with custom test configuration pointing to local directories)
- **User creation:** `./maddyctl -config <test-config-path> users create user1@test.local` and `./maddyctl -config <test-config-path> users create user2@test.local`
- **Message inspection:** `./maddyctl -config <test-config-path> imap-msgs list user1@test.local INBOX` and `./maddyctl -config <test-config-path> imap-msgs dump`
- **User cleanup:** `./maddyctl -config <test-config-path> users delete user1@test.local` and `./maddyctl -config <test-config-path> users delete user2@test.local`
- **SMTP test tool:** Python `smtplib` (available in standard library) or `swaks` for SMTP transactions with custom MAIL FROM and AUTH credentials
- **Protocol capture:** Maddy's `io_debug` directive on the submission endpoint logs full SMTP I/O; alternatively, `tcpdump` on the loopback interface for raw packet capture

### 0.9.2 Test Configuration Strategy

The test configuration adapts the default `maddy.conf` for local, non-TLS testing:

- **Hostname/Domain:** `$(hostname) = test.local`, `$(primary_domain) = test.local`
- **Submission endpoint:** `submission tcp://127.0.0.1:587` (plaintext, localhost-only)
  - `insecure_auth` enabled (required for plaintext auth)
  - `io_debug yes` (captures full SMTP protocol exchange)
  - `debug yes` (enables debug logging on the pipeline)
- **DKIM signing:** `sign_dkim test.local default` within the `source test.local` block
  - Debug enabled for signing decision logging
  - Default `require_sender_match` (`envelope`, `auth`) retained to test default behavior
- **Storage:** `sql local_mailboxes local_authdb { driver sqlite3; dsn /tmp/maddy-test/all.db }` (test directory)
- **State directory:** `/tmp/maddy-test/` (isolates test state from any system installation)
- **IMAP endpoint:** Retained for optional message inspection via IMAP, or excluded for simplicity
- **Inbound SMTP (port 25):** Excluded from test config (not needed for submission testing)

### 0.9.3 Default Format and Citation Requirements

- **Default format:** Markdown with Mermaid diagrams
- **Citation requirement:** Every section must reference source files by repository-relative path; runtime observations cite log output or SMTP transcript line numbers
- **Style guide:** Technical investigation report format — setup, observations, analysis, conclusions
- **Validation:** SMTP transcripts validated by actual protocol execution; headers validated by actual storage inspection


## 0.10 Rules for Documentation

### 0.10.1 User-Specified Rules

The following rules are explicitly specified by the user and the SWE-AtlasQnA-Repo implementation rule:

- **Create a new markdown document named `maddy.md`** placed in the `blitzy/documentation` directory
- **Provide thinking / rationale behind the answers** — every conclusion must include the reasoning chain that led to it
- **Do not make assumptions, base answers on the code as the truth** — source code is the definitive reference for explaining observed behavior
- **Do not modify any existing files in the source repository** — no changes to Go source, configuration templates, documentation, or any other committed file
- **Keep the repository unchanged** — configuration files and test artifacts are allowed as temporary files but must be noted
- **Clean up database state afterward** — test user accounts and mailbox data must be removed after testing
- **Based only on what you can observe at runtime** — no conclusions from source reading alone without corroborating runtime evidence
- **Show the raw header content, not a summary of what you expect to find** — actual header text must be presented verbatim
- **Capture the SMTP transaction for at least one rejected and one accepted message** — full protocol exchange required
- **If debug logging doesn't provide enough detail, show what you tried and use an alternative approach** — methodology transparency required
- **Rule out at least one plausible but incorrect interpretation** with specific runtime evidence
- **Identify one case where behavior differs from a reasonable config reading** supported by runtime observations

### 0.10.2 Derived Documentation Rules

Based on the investigation nature of this task, the following additional rules apply:

- **Evidence-first methodology:** Present observed data before analysis; never lead with conclusions
- **Reproducibility:** Document all commands used to set up the environment, send test messages, and inspect results so the investigation can be independently reproduced
- **Separation of layers:** Clearly distinguish between domain-level enforcement (pipeline source routing) and user-level enforcement (DKIM sender match) — these are separate mechanisms with different behaviors
- **Accurate terminology:** Use "MAIL FROM" for envelope sender, "From" for header sender, "authenticated identity" for SASL username; never conflate these distinct concepts
- **Source code is explanatory, not evidentiary:** Source code is cited to explain WHY behavior occurs; the behavior itself must be established through runtime observation
- **No speculation beyond evidence:** If a particular scenario cannot be tested (e.g., due to environment limitations), state this explicitly rather than extrapolating


## 0.11 References

### 0.11.1 Repository Files and Folders Searched

The following files were directly read and analyzed to derive the conclusions in this Agent Action Plan:

**Configuration and Project Files:**

| File | Purpose in Analysis |
|------|-------------------|
| `maddy.conf` | Default server configuration — submission block, source routing, DKIM signing, reject directives |
| `go.mod` | Go module definition, dependency versions, minimum Go version (1.13) |
| `go.sum` | Dependency verification checksums |
| `.mkdocs.yml` | Documentation site configuration, navigation tree, theme |
| `HACKING.md` | Design goals, module architecture, contributor guide |
| `README.md` | Project overview and capabilities |

**Core Source Files (Sender Identity Enforcement):**

| File | Lines Read | Key Findings |
|------|-----------|--------------|
| `internal/endpoint/smtp/smtp.go` | 1-721 | Session lifecycle, MAIL FROM handling via `startDelivery()`, auth flow in `Login()`, `AnonymousLogin()`, submission flag (`authAlwaysRequired`), `wrapErr()` SMTP error translation |
| `internal/endpoint/smtp/submission.go` | 1-131 | `submissionPrepare()` validates From/Sender/Date headers, sets `DontTraceSender=true`, does NOT validate MAIL FROM vs AuthUser |
| `internal/modify/dkim/dkim.go` | 1-420 | `shouldSign()` decision tree (lines 249-330), `require_sender_match` default `["envelope", "auth"]` (line 152), `fieldsToSign()` with oversign list, `RewriteBody()` signing execution |
| `internal/modify/dkim/dkim_test.go` | 1-209 | Unit tests confirming shouldSign behavior for envelope/auth match methods, cross-domain rejection, auth identity mismatch |
| `internal/modify/dkim/keys.go` | Full | Key generation (Ed25519/RSA), filesystem persistence, DNS record output |
| `internal/msgpipeline/msgpipeline.go` | 1-546 | `srcBlockForAddr()` domain-based source routing (lines 155-202), modifier execution in `Body()` (lines 307-336), `Start()` pipeline entry |
| `internal/msgpipeline/config.go` | 1-379 | Source/destination/reject directive parsing, `parseMsgPipelineRootCfg()`, `parseRejectDirective()` with SMTP error codes |
| `internal/msgpipeline/check_runner.go` | 1-317 | `applyResults()` Authentication-Results header generation (lines 262-309), DMARC integration |

**Authentication and Storage Files:**

| File | Lines Read | Key Findings |
|------|-----------|--------------|
| `internal/auth/auth.go` | Full | `CheckDomainAuth()` — domain allow-list, per-domain vs non-per-domain mode, case-insensitive comparison |
| `internal/storage/sql/sql.go` | 330-420 | `prepareUsername()` with PRECIS normalization, `CheckPlain()` with bcrypt, `GetOrCreateUser()` |
| `internal/storage/sql/maddyctl.go` | 1-79 | `CreateUser()`, `DeleteUser()`, `ListUsers()`, `SetUserPassword()` — user management wrappers |

**Module and Infrastructure Files:**

| File | Lines Read | Key Findings |
|------|-----------|--------------|
| `internal/module/msgmetadata.go` | 1-118 | `ConnState` with `AuthUser`, `AuthPassword` fields; `MsgMetadata` with `OriginalFrom`, `DontTraceSender`, `Quarantine` |
| `internal/target/received.go` | 1-87 | `GenerateReceived()` — suppresses src host/IP when `DontTraceSender` is true |

**Documentation and Example Files:**

| File | Purpose in Analysis |
|------|-------------------|
| `docs/README.md` | Documentation landing page, project capabilities |
| `docs/tutorials/setting-up.md` | Deployment guide — user creation, DKIM key setup, DNS |
| `docs/tutorials/manual-installation.md` | Build and first-run procedure |
| `docs/internals/sqlite.md` | SQLite backend constraints and behavior |
| `docs/internals/quirks.md` | SMTP/IMAP protocol quirks |
| `docs/man/README.md` | Manpage toolchain documentation |
| `examples/multitentant-dkim.conf` | Multi-domain DKIM signing configuration pattern |
| `examples/README.md` | Advanced configuration example guidance |
| `cmd/README.md` | Command binary descriptions |

**Folders Explored:**

| Folder | Depth | Purpose |
|--------|-------|---------|
| `` (root) | 0 | Repository structure, top-level files |
| `internal/` | 1 | Internal package organization |
| `internal/endpoint/smtp/` | 2 | SMTP/submission endpoint implementation |
| `internal/modify/dkim/` | 2 | DKIM signing subsystem |
| `internal/modify/` | 1 | Modifier framework |
| `internal/auth/` | 1 | Authentication subsystem |
| `internal/storage/` | 1 | Storage namespace |
| `internal/storage/sql/` | 2 | SQL storage backend |
| `internal/msgpipeline/` | 1 | Message pipeline orchestration |
| `internal/module/` | 1 | Core interfaces and types |
| `internal/check/dkim/` | 2 | Inbound DKIM verification |
| `docs/` | 1 | Documentation hub |
| `docs/tutorials/` | 2 | Deployment tutorials |
| `docs/internals/` | 2 | Implementation notes |
| `docs/man/` | 2 | Manpage toolchain |
| `cmd/` | 1 | Command binaries |
| `cmd/maddyctl/` | 2 | Administrative CLI |
| `examples/` | 1 | Advanced configuration examples |

### 0.11.2 Tech Spec Sections Consulted

| Section | Relevance |
|---------|-----------|
| 1.1 EXECUTIVE SUMMARY | Project overview, design goals, value proposition |
| 6.4 Security Architecture | Authentication framework, authorization system, DKIM signing/verification, pipeline-based authorization model, anti-spoofing policies, SMTP response codes |

### 0.11.3 Attachments and External References

- **Attachments provided by user:** None (0 attachments)
- **Figma URLs:** None provided
- **External URLs:** None referenced; all analysis is based on repository contents
- **Environment files:** No environment-specific files provided in `/tmp/environments_files`
- **Environment variables:** None specified
- **Secrets:** None specified


