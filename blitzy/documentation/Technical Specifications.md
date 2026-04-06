# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new, self-contained technical investigation report** that walks a newcomer through a concrete DKIM signing execution path in the foxcpp/maddy mail server codebase. This report will be produced by building the project from source, exercising its DKIM key generation and header-signing logic, and its Message-ID generation logic, then documenting the verbatim results.

- **Category:** Create new documentation
- **Documentation type:** Technical investigation walkthrough / onboarding guide
- **Delivery format:** A single Markdown file named `maddy.md` placed in the `blitzy-research/AtlasQnA` repository

The documentation requirements, with enhanced clarity, are:

- **R1 – Binary build and version attestation:** Build the maddy server binary from the checked-out source tree, place it outside the repository at `/tmp/maddy-bin/maddy`, and include the verbatim `maddy -v` output (`maddy unknown (built from source tree)`) in the report.
- **R2 – Sample message construction:** Compose a sample email containing exactly three `From` headers, exactly three `List-Id` headers, no other headers, and a single non-empty body line. Reproduce the exact message verbatim.
- **R3 – DKIM key generation (dual-algorithm):** Generate an RSA-2048 key (the codebase default `newkey_algo`) and an Ed25519 key in the same run. For each, copy the verbatim DNS TXT record string from the generated `.dns` file, report the DNS TXT record format, the observed base64 length, the raw public-key byte length, whether the base64 string ends with `=` padding, and the absolute `.dns` file path. State the selector and domain used.
- **R4 – Fields-to-sign analysis:** Produce the fields-to-sign list for the sample message, printed one header per line. Report the total entry count, the number of unique header names, and explain the numeric difference between an oversigned header (`From`) and a signed-only header (`List-Id`).
- **R5 – Message-ID observation:** Run the code path that produces Message-ID values, show at least five real outputs, and describe the observed format and length.

### 0.1.2 Special Instructions and Constraints

- **CRITICAL: No repository modification.** The user explicitly stated: "do not modify repository files." All temporary scripts, configs, and binaries must reside outside the repo tree.
- **CRITICAL: No web search.** The user stated: "do not use web search, only what you can verify by building and running this code."
- **Binary placement:** The built binary must be placed outside the repository (`/tmp/maddy-bin/maddy`).
- **Verbatim output requirement:** The `maddy version` output, DNS TXT record strings, fields-to-sign list, and Message-ID samples must be reproduced verbatim in the documentation.
- **Temporary scripts allowed:** Temporary Go programs and configuration files outside the repo are permitted for observation and measurement.
- **Output destination rule:** User-specified implementation rule: "Use the blitzy-research/AtlasQnA repository as the destination. Write your complete answer as a single markdown file named `maddy.md`. Do not modify any files in the source repository."

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document the binary build (R1), we will build `./cmd/maddy/` with `go build` targeting `/tmp/maddy-bin/maddy` and capture the verbatim version string.
- To document the sample message (R2), we will construct a `textproto.Header` with exactly 3 `From` and 3 `List-Id` headers, no other headers, and a single body line, using the same `go-message/textproto` library the codebase uses.
- To document DKIM key generation (R3), we will replicate the `generateAndWrite` and `writeDNSRecord` logic from `internal/modify/dkim/keys.go`, generating both RSA-2048 (default) and Ed25519 keys, reading back the `.dns` files, and analyzing the base64 content.
- To document fields-to-sign (R4), we will replicate the `fieldsToSign` function from `internal/modify/dkim/dkim.go` using the default `oversignDefault` and `signDefault` lists and apply it to the sample message header.
- To document Message-ID generation (R5), we will replicate the `msgIDField` function from `internal/endpoint/smtp/submission.go` using `google/uuid` v1.1.1.

### 0.1.4 Inferred Documentation Needs

- Based on code analysis: The `internal/modify/dkim/dkim.go` file defines two distinct header lists (`oversignDefault` and `signDefault`) whose behavioral difference is the core pedagogical content for this investigation.
- Based on structure: The investigation spans `internal/modify/dkim/` (signing + key management), `internal/endpoint/smtp/submission.go` (Message-ID), and `cmd/maddy/` (build entrypoint).
- Based on user journey: The report must include the exact sample message, build output, DNS records, fields-to-sign list, and Message-ID samples so the reader can reproduce or verify each step independently.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **MkDocs-based documentation site** with the ReadTheDocs theme, configured in `.mkdocs.yml`. The documentation ecosystem also relies on `scdoc` for man page generation. No ReadTheDocs deployment configuration (`.readthedocs.yml`) was found; the docs are published at `https://foxcpp.dev/maddy/`.

| Attribute | Value |
|---|---|
| Documentation framework | MkDocs (ReadTheDocs theme) |
| Documentation generator config | `.mkdocs.yml` |
| Man page tool | `scdoc` |
| Markdown extensions | codehilite (syntax highlighting) |
| API documentation tools | None (Go doc comments only) |
| Diagram tools detected | None in codebase |
| Documentation hosting | `https://foxcpp.dev/maddy/` |

Existing documentation files in the repository:

| Path | Purpose |
|---|---|
| `docs/README.md` | Project landing page and feature overview |
| `docs/get.sh-script.md` | Bootstrap installer documentation |
| `docs/tutorials/setting-up.md` | End-to-end deployment walkthrough |
| `docs/tutorials/manual-installation.md` | Step-by-step source build and first-run guide |
| `docs/tutorials/alias-to-remote.md` | Routing aliases to remote MX destinations |
| `docs/internals/quirks.md` | SMTP/IMAP implementation quirks |
| `docs/internals/sqlite.md` | SQLite backend behavior |
| `docs/man/*.scd` | 9 scdoc man page sources for all modules |
| `HACKING.md` | Contributor design guide |
| `README.md` | Project overview |
| `examples/multitentant-dkim.conf` | Multi-domain DKIM signing example config |

### 0.2.2 Repository Code Analysis for Documentation

The code modules relevant to this DKIM signing investigation were examined:

- **DKIM signing subsystem:** `internal/modify/dkim/dkim.go` — defines `oversignDefault` (15 headers) and `signDefault` (12 headers), implements `fieldsToSign()` which computes the signed-header list, and orchestrates signing via `RewriteBody`.
- **DKIM key management:** `internal/modify/dkim/keys.go` — implements `loadOrGenerateKey()` and `generateAndWrite()` for key lifecycle, plus `writeDNSRecord()` which writes `v=DKIM1; k=<algo>; p=<base64>` format DNS TXT records.
- **DKIM signing tests:** `internal/modify/dkim/dkim_test.go` — validates `fieldsToSign` and `shouldSign` behavior with table-driven tests.
- **DKIM key tests:** `internal/modify/dkim/keys_test.go` — validates Ed25519 key generation, PKCS#8 loading, and PKCS#1 RSA loading.
- **Message-ID generation:** `internal/endpoint/smtp/submission.go` — uses `github.com/google/uuid` v1.1.1 `NewRandom()` to generate UUIDv4 strings, formatted as `<UUID@domain>`.
- **Server entrypoint and version:** `maddy.go` — `BuildInfo()` returns the version string; `Version` variable defaults to `"unknown (built from source tree)"`.
- **Main binary launcher:** `cmd/maddy/main.go` — minimal launcher that imports root package and calls `maddy.Run()`.

### 0.2.3 DKIM-Relevant Configuration Documented in Man Pages

The DKIM signing module (`sign_dkim`) is documented in `docs/man/maddy-filters.5.scd` (lines 373–520). Key configuration defaults from the documentation:

| Directive | Default | Source |
|---|---|---|
| `key_path` | `dkim_keys/{domain}_{selector}.key` | `internal/modify/dkim/dkim.go:137` |
| `newkey_algo` | `rsa2048` | `internal/modify/dkim/dkim.go:150` |
| `header_canon` | `relaxed` | `internal/modify/dkim/dkim.go:141` |
| `body_canon` | `relaxed` | `internal/modify/dkim/dkim.go:144` |
| `sig_expiry` | `120h` (5 days) | `internal/modify/dkim/dkim.go:146` |
| `hash` | `sha256` | `internal/modify/dkim/dkim.go:148` |

### 0.2.4 Build and Runtime Verification

The binary was built from the checked-out source and placed outside the repository:

- **Go version used:** go1.13.15 linux/amd64 (matching the `go 1.13` directive in `go.mod`)
- **Build command:** `go build -o /tmp/maddy-bin/maddy ./cmd/maddy/`
- **Build dependency:** `gcc` required for `github.com/mattn/go-sqlite3` CGO compilation
- **Verbatim `maddy version` output:**

```
maddy unknown (built from source tree)
```

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following source modules are exercised by this investigation and must be documented:

- **Module: `internal/modify/dkim/dkim.go`**
  - Public APIs exercised: `fieldsToSign(h *textproto.Header) []string`
  - Data structures documented: `oversignDefault` (15 headers), `signDefault` (12 headers)
  - Current documentation: Man page in `docs/man/maddy-filters.5.scd` covers configuration but not runtime behavior
  - Documentation needed: Walkthrough of `fieldsToSign` behavior with a concrete multi-header message

- **Module: `internal/modify/dkim/keys.go`**
  - Public APIs exercised: `loadOrGenerateKey()`, `generateAndWrite()`, `writeDNSRecord()`
  - Current documentation: Man page covers `key_path` and `newkey_algo` directives
  - Documentation needed: Verbatim DNS TXT records for RSA-2048 and Ed25519, base64 analysis

- **Module: `internal/endpoint/smtp/submission.go`**
  - Public APIs exercised: `msgIDField` function variable (UUIDv4 via `google/uuid`)
  - Current documentation: None beyond code comments
  - Documentation needed: Format description and real output samples of generated Message-ID values

- **Module: `cmd/maddy/main.go` + `maddy.go`**
  - Build entrypoint and version reporting
  - Documentation needed: Build instructions and verbatim version output

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps addressed by this investigation include:

- **No existing DKIM walkthrough:** The repository contains a multi-tenant DKIM config example (`examples/multitentant-dkim.conf`) and man page directives, but no concrete walkthrough showing the runtime behavior of oversigning vs. signing for a specific message.
- **No DNS record format documentation:** While `keys.go` generates `.dns` files, no existing documentation describes the exact format, base64 encoding properties, or byte-length characteristics for RSA vs. Ed25519.
- **No Message-ID format documentation:** The UUID-based Message-ID generation in `submission.go` is undocumented beyond the source code itself.
- **No onboarding investigation guide:** No existing document walks a newcomer through building the binary and observing DKIM behavior end-to-end.

### 0.3.3 Investigation Results to Document

The following concrete results were obtained from building and running the code, and must be reproduced verbatim in the output documentation:

**Sample message (exact):**
```
From: alice@example.com
From: bob@example.com
From: charlie@example.com
List-Id: list1.example.com
List-Id: list2.example.com
List-Id: list3.example.com

Hello, this is a test message body for DKIM signing investigation.
```

**DKIM keys generated:**
- RSA-2048 key at `/tmp/dkim-keys/example.com_default_rsa.key`
- Ed25519 key at `/tmp/dkim-keys/example.com_default_ed25519.key`
- Selector: `default`, Domain: `example.com`

**Fields-to-sign list (21 entries, 16 unique header names):**
- `From` appears 4 times (3 present + 1 oversign)
- `List-Id` appears 3 times (3 present, no oversign)

**Message-ID format:** `<UUIDv4@domain>` — 36-character UUID, 50-character total Message-ID

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output is a single self-contained Markdown file. The planned structure:

```
maddy.md
├── Title and Introduction
├── 1. Build Verification
│   ├── Build environment
│   ├── Build command
│   └── Verbatim `maddy version` output
├── 2. DKIM Key Generation
│   ├── Key generation method (RSA-2048 default + Ed25519)
│   ├── DNS TXT record for RSA-2048 (verbatim)
│   ├── DNS TXT record for Ed25519 (verbatim)
│   ├── Base64 length and padding analysis
│   ├── Raw public key byte lengths
│   └── File paths, selector, and domain
├── 3. Sample Message
│   ├── Exact message (headers + body)
│   └── Header count verification
├── 4. Fields-to-Sign Analysis
│   ├── Verbatim fields-to-sign list
│   ├── Total entries and unique header names
│   └── Oversigned vs. signed-only explanation
├── 5. Message-ID Observation
│   ├── Five real Message-ID outputs
│   └── Format and length description
└── 6. Source Code References
```

### 0.4.2 Content Generation Strategy

- **Information Extraction Approach:**
  - Extract `oversignDefault` and `signDefault` arrays from `internal/modify/dkim/dkim.go:31-72`
  - Replicate `fieldsToSign` logic from `internal/modify/dkim/dkim.go:202-233` in an external Go test program
  - Replicate `writeDNSRecord` logic from `internal/modify/dkim/keys.go:136-163` for key generation
  - Replicate `msgIDField` from `internal/endpoint/smtp/submission.go:16-22` for Message-ID generation
  - All observations produced by compiling and running code using the same libraries as the codebase

- **Documentation Standards:**
  - All output values are verbatim from program execution, not manually constructed
  - Code blocks use triple-backtick fenced blocks with language annotations
  - Source citations reference exact file paths and line numbers from the repository
  - Tables for structured comparisons (RSA vs. Ed25519 properties)

### 0.4.3 Diagram and Visual Strategy

No Mermaid diagrams are required for this investigation. The document is centered on verbatim program output, tables comparing algorithm properties, and inline code blocks showing the sample message and fields-to-sign list. The investigation is observational in nature and best served by concrete textual evidence rather than abstract diagrams.

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---|---|---|---|
| `maddy.md` | CREATE | `internal/modify/dkim/dkim.go`, `internal/modify/dkim/keys.go`, `internal/endpoint/smtp/submission.go`, `cmd/maddy/main.go`, `maddy.go` | Complete DKIM signing investigation report with build verification, key generation, DNS TXT records, sample message, fields-to-sign analysis, and Message-ID observation |

**Note:** This is the sole documentation file to be produced. No existing documentation files are modified, deleted, or used as templates. The user's constraint ("do not modify repository files") means zero UPDATE or DELETE transformations apply.

### 0.5.2 New Documentation File Detail

```
File: maddy.md
Type: Technical investigation walkthrough / onboarding guide
Destination: blitzy-research/AtlasQnA repository root
Source Code:
  - internal/modify/dkim/dkim.go (oversignDefault, signDefault, fieldsToSign)
  - internal/modify/dkim/keys.go (loadOrGenerateKey, generateAndWrite, writeDNSRecord)
  - internal/modify/dkim/dkim_test.go (TestFieldsToSign reference)
  - internal/modify/dkim/keys_test.go (TestKeyLoad_new reference)
  - internal/endpoint/smtp/submission.go (msgIDField, Message-ID format)
  - cmd/maddy/main.go (build entrypoint)
  - maddy.go (Version variable, BuildInfo function)
  - go.mod (Go version requirement, dependency versions)
Sections:
  - Build Verification (binary build and version output)
  - DKIM Key Generation (RSA-2048 and Ed25519 with DNS records)
  - Sample Message (exact headers and body)
  - Fields-to-Sign Analysis (verbatim list, counts, oversign vs. sign explanation)
  - Message-ID Observation (5 real outputs, format description)
  - Source Code References (all files consulted)
Key Citations:
  - internal/modify/dkim/dkim.go:31-72 (header lists)
  - internal/modify/dkim/dkim.go:202-233 (fieldsToSign function)
  - internal/modify/dkim/keys.go:77-133 (generateAndWrite)
  - internal/modify/dkim/keys.go:136-163 (writeDNSRecord)
  - internal/endpoint/smtp/submission.go:16-22 (msgIDField)
  - maddy.go:41 (Version variable)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration files need to be updated. The output file `maddy.md` is a standalone artifact placed in the `blitzy-research/AtlasQnA` repository, not integrated into the maddy project's MkDocs site or man page system.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content/includes:** The output file is self-contained.
- **No navigation links:** Not part of the maddy MkDocs nav tree.
- **No table of contents updates:** No existing TOC references this file.
- **No index/glossary updates:** No glossary integration required.

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following tools and packages are required to build the maddy binary and execute the DKIM investigation:

| Registry | Package Name | Version | Purpose |
|---|---|---|---|
| golang.org | go | 1.13.15 | Go compiler matching the `go 1.13` directive in `go.mod` |
| apt | build-essential | 12.10ubuntu1 | Provides gcc required by `mattn/go-sqlite3` CGO compilation |
| go module | github.com/emersion/go-message | v0.10.9-0.20191116124005-65fd0119e899 | `textproto.Header` used for message header construction and iteration |
| go module | github.com/emersion/go-msgauth | v0.3.2-0.20191028231513-55b75676976c | `dkim` package used by the signing subsystem (not directly exercised in test program) |
| go module | github.com/google/uuid | v1.1.1 | UUIDv4 generation for Message-ID values |
| go module | github.com/mattn/go-sqlite3 | v1.11.0 | SQLite driver (CGO, required for successful binary build) |
| go module | golang.org/x/crypto | v0.0.0-20191108234033-bd318be0434a | Ed25519 cryptographic primitives |
| go stdlib | crypto/rsa | (Go 1.13) | RSA-2048 key generation |
| go stdlib | crypto/x509 | (Go 1.13) | PKCS#1 and PKCS#8 key marshaling |
| go stdlib | encoding/base64 | (Go 1.13) | Base64 encoding for DNS TXT records |

### 0.6.2 Documentation Reference Updates

Not applicable. No existing documentation files require link updates, as the output file `maddy.md` is a standalone artifact in a separate repository.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

| Requirement | Coverage Target | Verification Method |
|---|---|---|
| R1 – Binary build and version | 100% — verbatim `maddy version` output included | Compare output string to `maddy.go:41` default |
| R2 – Sample message | 100% — exact message with 3 From, 3 List-Id, no others, non-empty body | Count header instances in reproduced message |
| R3 – DKIM key generation (dual) | 100% — both RSA-2048 and Ed25519 DNS records, base64 analysis, byte lengths, padding, file paths, selector, domain | Verify `.dns` file contents match `writeDNSRecord` output format |
| R4 – Fields-to-sign analysis | 100% — verbatim list, total count, unique count, oversign vs. sign explanation | Cross-reference with `fieldsToSign` logic and `TestFieldsToSign` |
| R5 – Message-ID observation | 100% — at least 5 real outputs, format and length description | Verify UUID format via `google/uuid` v1.1.1 |

### 0.7.2 Documentation Quality Criteria

- **Completeness requirements:**
  - Every user-specified question is answered with verbatim program output
  - All five requirements (R1–R5) are fully addressed in the output document
  - DNS TXT record strings are copied character-for-character from `.dns` files
  - Base64 length, padding presence, and raw byte length are reported for both key types

- **Accuracy validation:**
  - All output values are produced by compiling and executing Go code that uses the same libraries as the maddy codebase (`go-message/textproto`, `google/uuid`, `crypto/ed25519`, `crypto/rsa`, `crypto/x509`)
  - The `fieldsToSign` function is replicated verbatim from `internal/modify/dkim/dkim.go:202-233`
  - The `writeDNSRecord` function is replicated verbatim from `internal/modify/dkim/keys.go:136-163`
  - Key generation matches the codebase defaults: RSA-2048 via `rsa.GenerateKey(rand.Reader, 2048)`, Ed25519 via `ed25519.GenerateKey(rand.Reader)`

- **Clarity standards:**
  - Each section starts with what was done, followed by the exact output, then interpretation
  - Technical terms (oversigning, PKCS#1, PKCS#8, UUIDv4) are used precisely
  - The difference between oversigned and signed-only headers is explained with concrete counts

### 0.7.3 Example and Diagram Requirements

- **Minimum examples:** 1 complete sample message, 2 DNS TXT records, 1 fields-to-sign list, 5 Message-ID values
- **Diagram types required:** None (investigation is output-driven)
- **Code example testing:** All examples were produced by compiling and running Go programs
- **Visual content freshness:** Not applicable (single-run investigation)

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

- **New documentation file:**
  - `maddy.md` — complete DKIM signing investigation report (CREATE)

- **Source code modules examined** (read-only):
  - `internal/modify/dkim/dkim.go` — oversign/sign defaults, `fieldsToSign`
  - `internal/modify/dkim/keys.go` — key generation and DNS record writing
  - `internal/modify/dkim/dkim_test.go` — test reference for `fieldsToSign`
  - `internal/modify/dkim/keys_test.go` — test reference for key loading
  - `internal/endpoint/smtp/submission.go` — Message-ID generation
  - `cmd/maddy/main.go` — build entrypoint
  - `maddy.go` — version string and `BuildInfo()`
  - `go.mod` — Go version and dependency manifest
  - `docs/man/maddy-filters.5.scd` — existing DKIM documentation reference

- **Temporary artifacts** (outside repo):
  - `/tmp/maddy-bin/maddy` — built binary
  - `/tmp/dkim-investigation/` — Go test program for DKIM + Message-ID investigation
  - `/tmp/dkim-keys/` — generated DKIM key files and DNS record files

- **Investigation outputs to document:**
  - Verbatim `maddy -v` output
  - Exact sample message (3 From, 3 List-Id, body line)
  - Verbatim DNS TXT record strings for RSA-2048 and Ed25519
  - Base64 length, padding, and raw byte length for each algorithm
  - Absolute `.dns` file paths
  - DKIM selector and domain used
  - Verbatim fields-to-sign list with counts and explanation
  - At least 5 Message-ID values with format description

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No files in the maddy repository may be modified, per user constraint
- **Test file modifications:** No test files modified
- **Feature additions or code refactoring:** Not applicable
- **Deployment configuration changes:** Not applicable
- **Unrelated documentation:** No updates to existing maddy docs, man pages, or MkDocs config
- **DKIM signature verification:** Only signing behavior is investigated, not the `verify_dkim` check module
- **Actual SMTP message delivery:** No live SMTP transactions; the investigation uses library-level code paths
- **Web search:** Explicitly prohibited by user instruction

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

| Parameter | Value |
|---|---|
| **Build command** | `CGO_ENABLED=1 go build -o /tmp/maddy-bin/maddy ./cmd/maddy/` |
| **Version command** | `/tmp/maddy-bin/maddy -v` |
| **Go toolchain** | go1.13.15 linux/amd64 |
| **Investigation program** | `/tmp/dkim-investigation/main.go` (standalone Go program replicating codebase logic) |
| **Investigation build** | `cd /tmp/dkim-investigation && go build -o investigate .` |
| **Investigation run** | `cd /tmp/dkim-investigation && ./investigate` |
| **Key output directory** | `/tmp/dkim-keys/` |
| **Output format** | Single Markdown file (`maddy.md`) |
| **Output destination** | `blitzy-research/AtlasQnA` repository |
| **Citation requirement** | Every technical claim references a source file path and line number |
| **Style guide** | Standalone technical investigation report with verbatim outputs |
| **Validation method** | All outputs produced by compiling and running Go code against the same library versions used by the maddy codebase |

## 0.10 Rules for Documentation

The following rules are explicitly specified or directly implied by the user's instructions:

- **Do not modify repository files.** All temporary scripts, configs, and binaries must reside outside the repository tree. The source repository must remain byte-identical after the investigation.
- **Do not use web search.** All information must be verified by building and running the code from the checked-out source.
- **Place the built binary outside the repo.** The binary resides at `/tmp/maddy-bin/maddy`.
- **Include verbatim `maddy version` output.** The exact string `maddy unknown (built from source tree)` must appear in the documentation.
- **Sample message must include exactly three `From` headers and exactly three `List-Id` headers, no other headers, and a non-empty body line.** The exact message must be reproduced.
- **Generate keys using default settings from the codebase.** The default `newkey_algo` is `rsa2048` (from `internal/modify/dkim/dkim.go:150`). An Ed25519 key must also be generated in the same run.
- **Copy DNS TXT record strings verbatim from `.dns` files.** Report the format, base64 length, raw byte length, padding presence, absolute file path, selector, and domain.
- **Report fields-to-sign verbatim, one header per line.** Explain the count difference between oversigned (`From`) and signed-only (`List-Id`). Report total entries and unique header names.
- **Show at least five real Message-ID outputs.** Describe the observed format and length.
- **Temporary scripts may be used for observation.** The repository itself must remain unchanged.
- **Write the complete answer as a single markdown file named `maddy.md`.** Place it in the `blitzy-research/AtlasQnA` repository. Do not modify any files in the source repository.

## 0.11 References

### 0.11.1 Repository Files and Folders Searched

The following files and folders were retrieved and analyzed to derive the conclusions in this Agent Action Plan:

| Path | Purpose |
|---|---|
| `go.mod` | Go module identity (`github.com/foxcpp/maddy`), Go version requirement (`go 1.13`), and dependency manifest |
| `maddy.go` | Server bootstrap, `Version` variable (line 41), `BuildInfo()` function (line 89) |
| `cmd/maddy/main.go` | Build entrypoint for the maddy binary |
| `internal/modify/dkim/dkim.go` | Core DKIM signing module: `oversignDefault` (lines 31-54), `signDefault` (lines 55-72), `fieldsToSign` (lines 202-233), `Modifier.Init` (lines 126-200), `RewriteBody` (lines 340-411) |
| `internal/modify/dkim/keys.go` | DKIM key lifecycle: `loadOrGenerateKey` (lines 19-75), `generateAndWrite` (lines 77-134), `writeDNSRecord` (lines 136-163) |
| `internal/modify/dkim/dkim_test.go` | Unit tests: `TestFieldsToSign` (lines 16-36), `TestShouldSign` (lines 38-208) |
| `internal/modify/dkim/keys_test.go` | Key tests: `TestKeyLoad_new` (lines 16-58), `TestKeyLoad_existing_pkcs8` (lines 66-92), `TestKeyLoad_existing_pkcs1` (lines 122-151) |
| `internal/endpoint/smtp/submission.go` | Message-ID generation: `msgIDField` (lines 16-22), `submissionPrepare` (lines 27-130) |
| `internal/module/msgmetadata.go` | `MsgMetadata` and `ConnState` type definitions |
| `internal/testutils/logger.go` | Test logger helper |
| `internal/check/dkim/dkim.go` | DKIM verification module (out of scope, inspected for context) |
| `.mkdocs.yml` | MkDocs documentation site configuration |
| `.build.yml` | CI/build configuration (Arch Linux, Go, scdoc) |
| `HACKING.md` | Contributor design guide and architecture overview |
| `README.md` | Project overview |
| `docs/README.md` | Documentation landing page |
| `docs/man/maddy-filters.5.scd` | Man page source covering `sign_dkim` module (lines 373-520) |
| `docs/tutorials/manual-installation.md` | Manual installation tutorial (context) |
| `examples/multitentant-dkim.conf` | Multi-domain DKIM signing example configuration |

**Folders traversed:**

| Folder | Depth | Purpose |
|---|---|---|
| `` (root) | 0 | Repository root — identified all top-level children |
| `cmd/` | 1 | Command binaries — located `maddy/` build entrypoint |
| `internal/` | 1 | Private implementation packages — identified DKIM and SMTP subsystems |
| `internal/modify/` | 2 | Message modifiers — located `dkim/` signing subpackage |
| `internal/modify/dkim/` | 3 | DKIM signing implementation — all 4 files examined |
| `internal/check/dkim/` | 3 | DKIM verification — 1 file examined for context |
| `docs/` | 1 | Documentation tree — inventoried all documentation assets |

### 0.11.2 Attachments

No attachments were provided for this project.

### 0.11.3 Figma Screens

No Figma screens were provided for this project.

### 0.11.4 External References

No web search was conducted per user instruction. All information was derived exclusively from building and running the repository source code.

