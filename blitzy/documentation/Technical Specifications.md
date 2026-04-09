# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new investigative technical document** that traces, explains, and corrects misconceptions about how the maddy mail server tracks recipient address transformations during the message delivery pipeline.

**Documentation Category:** Create new documentation
**Documentation Type:** Technical internals / architecture clarification document

The user is onboarding to the maddy mail server codebase and received a colleague's explanation of how recipient address rewrites are tracked. That explanation contained several factual errors about the data structure, mapping direction, and construction mechanism. The user independently searched the `modify` and `dsn` packages but could not locate the tracking mechanism, confirming that the colleague's mental model pointed to the wrong packages.

**Documentation Requirements with Enhanced Clarity:**

- **Requirement 1 — Trace the address tracking mechanism:** Identify where in the codebase the tracking of recipient address transformations actually resides, since it is NOT in the `internal/modify/` or `internal/dsn/` packages the user inspected. The tracking is orchestrated by `internal/msgpipeline/msgpipeline.go` and defined in `internal/module/msgmetadata.go`.
- **Requirement 2 — Identify and run verifying tests:** Locate the test functions that exercise the `OriginalRcpts` mapping, run them, and report results as proof that the documented behavior is accurate.
- **Requirement 3 — Show the actual data structure and direction:** Document the Go type `OriginalRcpts map[string]string` in `MsgMetadata`, clarify its direction (final→original, not original→final), and explain why the colleague's "forward mapping" description is inverted.
- **Requirement 4 — Clarify what the system actually does:** Provide a definitive correction of each misconception from the colleague's explanation, grounded in the source code.

**Inferred Documentation Needs:**

- Based on code analysis: The `MsgMetadata.OriginalRcpts` field in `internal/module/msgmetadata.go` is the sole data structure for tracking recipient transformations, and its population logic lives in `internal/msgpipeline/msgpipeline.go:AddRcpt()` — neither of which is in the `modify` or `dsn` packages the user searched.
- Based on architecture: The mapping spans three packages (`module`, `msgpipeline`, `target/queue`) that need consolidated documentation to form a coherent understanding.
- Based on user journey: The document must explicitly address the four misconceptions from the colleague (direction, incrementality, intermediate chain, package location) and replace them with source-cited truths.
- Based on bounce usage: The `internal/target/queue/queue.go:emitDSN()` function and the `statusCollector` wrapper in `msgpipeline.go` consume `OriginalRcpts` to reverse-translate addresses for DSN generation and LMTP status reporting, which connects to the colleague's mention of "using this for bounces."

### 0.1.2 Special Instructions and Constraints

- **Implementation Rule — SWE-AtlasQnA-Repo:** The output must be a new markdown document named after the project, placed in the `blitzy/documentation` directory in the destination repository. It must provide thinking/rationale behind all answers, base all answers on the code as truth, and must NOT modify any existing files in the source repository.
- **Style:** The document should be investigative in nature — tracing the actual behavior through the codebase, citing exact file paths and line numbers, and directly contrasting the colleague's claims with source-of-truth code.
- **Preservation of user context:** The user explicitly described four claims from a colleague:
  - User Example: "when aliases rewrite addresses, the system maintains a 'forward mapping' that stores what each original address was transformed into"
  - User Example: "you can look up user@domain.com and find it became alias@domain.com"
  - User Example: "this mapping is built incrementally by each modifier as it runs, creating a chain of all intermediate transformations"
  - User Example: "something about using this for bounces"
- **No source modification:** Per the implementation rule, no existing repository files may be modified.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **trace how address tracking works**, we will create a new document that walks through the `AddRcpt` method in `internal/msgpipeline/msgpipeline.go` (lines 227–305), showing how the original address is captured at line 235, how three phases of modifiers sequentially rewrite `to`, and how the mapping is written once at line 289 as `OriginalRcpts[finalAddr] = originalAddr`.
- To **find and run tests**, we will document the four key test functions that exercise `OriginalRcpts`: `TestMsgPipeline_RcptModifier_OriginalRcpt`, `TestMsgPipeline_RcptModifier_OriginalRcpt_Multiple`, `TestMsgPipeline_BodyNonAtomic_ModifiedRcpt`, and `TestQueueDSN_RcptRewrite`, all of which pass successfully.
- To **show the data structure and direction**, we will cite the type declaration in `internal/module/msgmetadata.go:89` (`OriginalRcpts map[string]string`) and its comment at lines 80–88 which unambiguously states the mapping goes from "final recipient" (key) to "recipient that was presented by the client" (value).
- To **clarify what the system actually does**, we will construct a side-by-side comparison of the colleague's four claims versus the code-evidenced behavior, with file-path citations for every correction.
- To **explain the bounce connection**, we will document how `internal/target/queue/queue.go:emitDSN()` (lines 882–897) uses `OriginalRcpts` to reverse-translate final addresses back to original addresses in DSN recipient reports, preventing alias disclosure.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a MkDocs-based documentation structure with moderate coverage of operational topics but no documentation of the address-tracking internals relevant to this task.

**Documentation framework:** MkDocs, configured via `.mkdocs.yml` at the repository root
- Theme: `readthedocs`
- Markdown extensions: `codehilite` with `guess_lang: false`
- Site name: "maddy documentation"
- Navigation tree covers: README, tutorials (3 pages), get.sh script, manual pages (9 scdoc-generated pages), and internals (2 pages)

**Documentation generator configuration:** `.mkdocs.yml` (root)
- Man page conversion utility: `docs/man/prepare_md.py` (Python 3 script converting `scdoc` sources to Markdown)

**Existing documentation inventory:**

| Path | Type | Content | Relevance |
|------|------|---------|-----------|
| `README.md` | Project overview | Feature summary, links, community channels | Low — does not cover internals |
| `HACKING.md` | Developer guide | Architecture, module patterns, modifier overview | Medium — describes modifier structure but not OriginalRcpts |
| `internal/README.md` | Architecture index | Package directory structure guide | Medium — maps subsystem boundaries |
| `docs/README.md` | Docs landing page | Product capabilities, links | Low |
| `docs/tutorials/alias-to-remote.md` | Tutorial | Alias forwarding to remote MX with bounce handling | High — discusses alias-related delivery but not OriginalRcpts mapping |
| `docs/tutorials/setting-up.md` | Tutorial | End-to-end deployment guide | Low |
| `docs/tutorials/manual-installation.md` | Tutorial | Source build and first-run | Low |
| `docs/internals/sqlite.md` | Internals | SQLite backend behavior | Low |
| `docs/internals/quirks.md` | Internals | SMTP/IMAP protocol quirks | Low |
| `docs/get.sh-script.md` | Reference | Installer script documentation | Low |
| `docs/man/README.md` | Reference | Man page authoring workflow | Low |

**Key finding:** There is no existing documentation anywhere in the repository that describes the `OriginalRcpts` mapping, its data structure, its direction, or how it integrates with modifier pipelines and DSN generation. The `HACKING.md` developer guide mentions modifiers but stops at describing their interface — it does not cover the pipeline's address bookkeeping. The `docs/tutorials/alias-to-remote.md` tutorial discusses alias behavior from a configuration perspective but does not explain the underlying tracking mechanism.

**API documentation tools:** None detected — the project does not use godoc generation, and code comments serve as inline documentation.
**Diagram tools:** None configured in the repository; Mermaid is appropriate for the output document.

### 0.2.2 Repository Code Analysis for Documentation

The following search patterns and directories were examined to map the code modules relevant to address tracking:

**Search 1 — Data structure definition:**
- `internal/module/msgmetadata.go` — `MsgMetadata` struct with `OriginalRcpts map[string]string` (line 89)
- Comment at lines 80–88 defines the mapping direction and purpose

**Search 2 — Population logic:**
- `internal/msgpipeline/msgpipeline.go` — `AddRcpt()` method (lines 227–305)
- Line 235: `originalTo := to` captures the pre-modification address
- Lines 237–280: Three modifier phases (global, source, per-recipient) sequentially rewrite `to`
- Line 288–289: Single conditional write `if originalTo != to { dd.msgMeta.OriginalRcpts[to] = originalTo }`
- Line 90–92: Lazy initialization of the map in `Start()`

**Search 3 — Consumption for LMTP status reporting:**
- `internal/msgpipeline/msgpipeline.go` — `statusCollector` struct (lines 346–357)
- Wraps `OriginalRcpts` to reverse-translate final addresses back to original addresses when reporting per-recipient statuses

**Search 4 — Consumption for DSN bounce generation:**
- `internal/target/queue/queue.go` — `emitDSN()` method (lines 849–950)
- Lines 882–897: Iterates `RcptErrs`, looks up `OriginalRcpts[rcpt]` to replace final addresses with original addresses in the DSN recipient info

**Search 5 — Modifier interface contract:**
- `internal/module/modifier.go` — `ModifierState` interface (lines 32–63)
- Lines 49–50: Explicit comment "MsgPipeline will take care of populating MsgMeta.OriginalRcpts. RewriteRcpt doesn't do it."

**Search 6 — Modifier composition layer:**
- `internal/modify/group.go` — `Group` struct serializes multiple modifiers
- `RewriteRcpt` runs each child modifier sequentially, passing output as next input (lines 49–57)

**Search 7 — Concrete modifiers (where user searched):**
- `internal/modify/alias_file.go` — `RewriteRcpt` does address lookup in alias map, returns new address. Does NOT touch `OriginalRcpts`.
- `internal/modify/replace_addr.go` — `RewriteRcpt` does string/regex replacement. Does NOT touch `OriginalRcpts`.
- `internal/dsn/dsn.go` — Pure DSN serializer. Receives `RecipientInfo` with `FinalRecipient` already translated. Does NOT access `OriginalRcpts`.

**Search 8 — Test coverage:**
- `internal/msgpipeline/modifier_test.go` — `TestMsgPipeline_RcptModifier_OriginalRcpt` (line 253) and `TestMsgPipeline_RcptModifier_OriginalRcpt_Multiple` (line 299)
- `internal/msgpipeline/bodynonatomic_test.go` — `TestMsgPipeline_BodyNonAtomic_ModifiedRcpt` (line 50)
- `internal/target/queue/queue_test.go` — `TestQueueDSN_RcptRewrite` (line 746)

**Codebase-wide grep for `OriginalRcpts`:** 18 occurrences across 5 files — confirming that usage is confined to the module definition, pipeline orchestration, pipeline tests, and queue DSN generation.

### 0.2.3 Web Search Research Conducted

No web search was needed for this task. The codebase itself is the authoritative source for answering the user's question. All findings are derived from direct code inspection, file reading, and test execution against the repository source.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules and files form the complete scope of code that must be analyzed and documented to answer the user's question:

**Module: `internal/module` — Core Data Structures**

- File: `internal/module/msgmetadata.go`
  - Public type: `MsgMetadata` (struct)
  - Key field: `OriginalRcpts map[string]string` (line 89)
  - Key method: `DeepCopy()` (line 112)
  - Current documentation: Inline code comments at lines 80–88 define the field's purpose and direction. No standalone documentation exists.
  - Documentation needed: Explanation of the data structure, map direction, initialization, and why it lives in the metadata rather than the modifier package.

- File: `internal/module/modifier.go`
  - Public interface: `Modifier`, `ModifierState`
  - Key method: `RewriteRcpt(ctx, rcptTo) (string, error)` (line 45)
  - Key contract: Comment at lines 49–50 explicitly states that `MsgPipeline` populates `OriginalRcpts`, not `RewriteRcpt`.
  - Current documentation: Inline interface comments only.
  - Documentation needed: Explanation of the responsibility boundary between modifiers and the pipeline for address bookkeeping.

- File: `internal/module/partial_delivery.go`
  - Public interface: `StatusCollector`, `PartialDelivery`
  - Key method: `SetStatus(rcptTo string, err error)` (line 26)
  - Current documentation: Inline comments. Line 17 notes translations should not affect `rcptTo`.
  - Documentation needed: How the pipeline's `statusCollector` wraps this interface to reverse-translate addresses.

**Module: `internal/msgpipeline` — Pipeline Orchestration (Owner of OriginalRcpts Population)**

- File: `internal/msgpipeline/msgpipeline.go`
  - Public type: `MsgPipeline` (struct implementing `module.DeliveryTarget`)
  - Key method: `Start(ctx, msgMeta, mailFrom)` — initializes `OriginalRcpts` map (line 90–92)
  - Key method: `AddRcpt(ctx, to)` — runs three modifier phases and writes `OriginalRcpts[finalAddr] = originalAddr` (line 289)
  - Internal type: `statusCollector` — wraps `OriginalRcpts` to reverse-translate addresses in LMTP status reports (lines 346–357)
  - Internal method: `BodyNonAtomic(ctx, c, header, body)` — passes `statusCollector` to downstream partial delivery targets (line 391–394)
  - Current documentation: Inline comments only.
  - Documentation needed: Step-by-step trace of the `AddRcpt` flow, data flow diagram, and explanation of the `statusCollector` reverse-translation.

**Module: `internal/modify` — Modifier Implementations (Where User Searched)**

- File: `internal/modify/group.go`
  - Public type: `Group` — serial modifier composition
  - Key method: `RewriteRcpt(ctx, rcptTo)` — chains modifiers sequentially (lines 49–57)
  - Documentation needed: Explanation of how modifiers are composed and why they do NOT contain address tracking logic.

- File: `internal/modify/alias_file.go`
  - Public type: `Modifier` — file-based alias rewriting
  - Key method: `RewriteRcpt(ctx, rcptTo)` — looks up alias, returns new address (lines 257–296)
  - Documentation needed: Explain this is a pure rewriter with no bookkeeping responsibility.

- File: `internal/modify/replace_addr.go`
  - Public type: `replaceAddr` — string/regex address replacement
  - Key method: `RewriteRcpt(ctx, rcptTo)` (lines 137–142)
  - Documentation needed: Same as alias_file — pure rewriter, no tracking.

**Module: `internal/dsn` — DSN Generation (Where User Searched)**

- File: `internal/dsn/dsn.go`
  - Public function: `GenerateDSN(utf8, envelope, mtaInfo, rcptsInfo, failedHeader, outWriter)`
  - Public type: `RecipientInfo` with `FinalRecipient string` field
  - Documentation needed: Explain that this is a pure serializer that receives already-translated addresses; the translation happens upstream in the queue's `emitDSN()`.

**Module: `internal/target/queue` — Queue and DSN Emission (Consumer of OriginalRcpts for Bounces)**

- File: `internal/target/queue/queue.go`
  - Private method: `emitDSN(meta, header)` — lines 849–950
  - Key logic: Lines 882–897 — iterates `RcptErrs`, looks up `OriginalRcpts[rcpt]`, substitutes original address into DSN `RecipientInfo.FinalRecipient`
  - Documentation needed: Explain the bounce flow and how OriginalRcpts prevents alias disclosure in DSN messages.

**Module: `internal/testutils` — Test Support**

- File: `internal/testutils/modifier.go` — Mock modifier with configurable `RcptTo map[string]string`
- File: `internal/testutils/target.go` — Mock delivery target with `Msg` capture
- Documentation needed: Referenced as test infrastructure supporting the verification tests.

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the documentation gap is comprehensive and specific:

- **No existing document describes `OriginalRcpts`:** The field is documented only through inline Go comments in `internal/module/msgmetadata.go`. There is no standalone document, tutorial, or internals page that explains this mechanism.
- **No existing document traces the modifier-pipeline interaction:** `HACKING.md` describes modifiers at the interface level but does not explain how the pipeline wraps them with bookkeeping.
- **No existing document explains DSN address translation:** The `docs/tutorials/alias-to-remote.md` tutorial discusses bounce handling at the configuration level but does not describe the code-level reverse-translation via `OriginalRcpts`.
- **No existing document corrects the "forward mapping" misconception:** The user's colleague's description contradicts the actual implementation, and no documentation prevents this misunderstanding.
- **No test documentation:** The four tests that verify `OriginalRcpts` behavior are undocumented beyond their function names and inline assertions.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

Per the implementation rule **SWE-AtlasQnA-Repo**, the output is a single comprehensive markdown document placed in `blitzy/documentation/`. The document structure will follow an investigative Q&A format that systematically addresses each part of the user's question.

Planned file tree:

    blitzy/
    └── documentation/
        └── maddy.md

**Document internal structure:**

    maddy.md
    ├── Introduction (context, colleague's claims, what we'll investigate)
    ├── The Actual Data Structure
    │   ├── Where it lives (internal/module/msgmetadata.go)
    │   ├── Type and direction (map[string]string, final→original)
    │   └── Why the direction matters
    ├── How the Mapping is Built
    │   ├── Pipeline initialization (Start method)
    │   ├── The AddRcpt flow (step-by-step trace)
    │   ├── Three modifier phases (global, source, per-recipient)
    │   ├── The single-assignment write (line 289)
    │   └── Mermaid sequence diagram of the flow
    ├── Why You Didn't Find It in modify/ or dsn/
    │   ├── What modify/ actually does (pure rewriting)
    │   ├── What dsn/ actually does (pure serialization)
    │   └── The responsibility boundary (modifier.go comment)
    ├── How It's Used for Bounces
    │   ├── Queue emitDSN() flow
    │   ├── statusCollector for LMTP
    │   └── Alias privacy protection
    ├── Colleague's Claims vs. Reality
    │   ├── Claim 1: "forward mapping" → Reverse mapping
    │   ├── Claim 2: "built incrementally" → Single assignment
    │   ├── Claim 3: "chain of intermediates" → First-and-last only
    │   ├── Claim 4: "used for bounces" → Correct (with nuance)
    │   └── Summary table
    ├── Test Evidence
    │   ├── Test inventory and descriptions
    │   ├── Test execution results
    │   └── What each test proves
    └── Key Source File Citations

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract the `OriginalRcpts` field declaration and comment from `internal/module/msgmetadata.go:80-89`
- Trace the `AddRcpt` method line-by-line from `internal/msgpipeline/msgpipeline.go:227-305`
- Extract the `statusCollector` reverse-translation logic from `internal/msgpipeline/msgpipeline.go:346-357`
- Extract the DSN emission logic from `internal/target/queue/queue.go:882-897`
- Extract the modifier interface contract comment from `internal/module/modifier.go:49-50`
- Run and capture results from the four test functions

**Template Application:**
- The document follows the SWE-AtlasQnA-Repo rule: comprehensive Q&A with thinking/rationale, code-grounded answers, no modifications to source files
- Each section provides the rationale behind the answer before presenting the answer itself

**Documentation Standards:**
- Markdown formatting with proper headers (# ## ###)
- Mermaid diagram for the AddRcpt flow showing data flow through modifier phases
- Code examples using fenced code blocks with Go syntax highlighting
- Source citations as inline references: `Source: internal/module/msgmetadata.go:89`
- Tables for claim-vs-reality comparisons
- Consistent terminology: "final address" (post-rewrite), "original address" (pre-rewrite)

### 0.4.3 Diagram and Visual Strategy

**Mermaid diagrams to create:**

- **Sequence diagram:** The `AddRcpt` flow showing `originalTo` capture, three modifier phases, conditional `OriginalRcpts` write, and delivery target dispatch
- **Flowchart:** The DSN generation flow showing how `emitDSN()` consumes `OriginalRcpts` to reverse-translate addresses before passing them to the DSN serializer
- **Data structure diagram:** Visual representation of the `OriginalRcpts` map direction with concrete example values from the test suite

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/maddy.md` | CREATE | `internal/module/msgmetadata.go`, `internal/module/modifier.go`, `internal/module/partial_delivery.go`, `internal/msgpipeline/msgpipeline.go`, `internal/modify/group.go`, `internal/modify/alias_file.go`, `internal/modify/replace_addr.go`, `internal/dsn/dsn.go`, `internal/target/queue/queue.go`, `internal/msgpipeline/modifier_test.go`, `internal/msgpipeline/bodynonatomic_test.go`, `internal/target/queue/queue_test.go`, `internal/testutils/modifier.go`, `internal/testutils/target.go`, `HACKING.md`, `internal/README.md` | Comprehensive Q&A document tracing recipient address transformation tracking: data structure definition, mapping direction, population logic in AddRcpt, modifier pipeline architecture, DSN bounce integration, statusCollector reverse-translation, test evidence with execution results, and systematic correction of four misconceptions from colleague's explanation |

This is the only file to be created. Per the SWE-AtlasQnA-Repo implementation rule, no existing files in the source repository are modified.

### 0.5.2 New Documentation File Detail

**File:** `blitzy/documentation/maddy.md`
**Type:** Technical investigation / Q&A document
**Source Code:** Multiple files across `internal/module/`, `internal/msgpipeline/`, `internal/modify/`, `internal/dsn/`, `internal/target/queue/`, `internal/testutils/`

**Sections:**

- **Introduction**
  - Restate the user's question and colleague's four claims
  - Explain the investigation approach
  - Source: User prompt

- **The Actual Data Structure: OriginalRcpts**
  - Field declaration: `OriginalRcpts map[string]string`
  - Comment text explaining direction: final→original
  - Why it lives in MsgMetadata (shared across pipeline phases)
  - Source: `internal/module/msgmetadata.go:80-89`

- **How the Mapping is Built: The AddRcpt Flow**
  - Step 1: `originalTo := to` captures pre-modification address
  - Step 2: Global modifiers rewrite via `RewriteRcpt`
  - Step 3: Source modifiers rewrite via `RewriteRcpt`
  - Step 4: Per-recipient modifiers rewrite via `RewriteRcpt`
  - Step 5: `if originalTo != to { OriginalRcpts[to] = originalTo }`
  - Mermaid sequence diagram of the complete flow
  - Source: `internal/msgpipeline/msgpipeline.go:227-305`

- **Why You Didn't Find It in modify/ or dsn/**
  - Modifiers are pure rewriters; the pipeline manages bookkeeping
  - Key contract: modifier.go lines 49-50 comment
  - DSN package is a pure serializer; queue manages reverse-translation
  - Source: `internal/module/modifier.go:49-50`, `internal/modify/alias_file.go`, `internal/modify/replace_addr.go`, `internal/dsn/dsn.go`

- **How It's Used for Bounces**
  - Queue's `emitDSN()` reverse-translates addresses via `OriginalRcpts`
  - `statusCollector` wraps `OriginalRcpts` for LMTP partial delivery
  - Privacy motivation: prevents alias disclosure in bounce messages
  - Source: `internal/target/queue/queue.go:882-897`, `internal/msgpipeline/msgpipeline.go:346-357`

- **Colleague's Claims vs. Reality**
  - Side-by-side comparison table
  - Source citations for each correction

- **Test Evidence**
  - `TestMsgPipeline_RcptModifier_OriginalRcpt`: single modifier mapping verification
  - `TestMsgPipeline_RcptModifier_OriginalRcpt_Multiple`: chained modifiers with first-and-last tracking
  - `TestMsgPipeline_BodyNonAtomic_ModifiedRcpt`: LMTP status reverse-translation
  - `TestQueueDSN_RcptRewrite`: DSN bounce message uses original addresses, not aliases
  - Execution results showing all PASS
  - Source: `internal/msgpipeline/modifier_test.go`, `internal/msgpipeline/bodynonatomic_test.go`, `internal/target/queue/queue_test.go`

- **Key Source File Citations**
  - Complete listing of all files examined with paths and line numbers

**Diagrams:**
- Mermaid sequence diagram: AddRcpt modifier pipeline flow
- Mermaid flowchart: DSN emission with address reverse-translation

**Key Citations:** `internal/module/msgmetadata.go`, `internal/msgpipeline/msgpipeline.go`, `internal/target/queue/queue.go`, `internal/module/modifier.go`, `internal/modify/group.go`, `internal/modify/alias_file.go`, `internal/dsn/dsn.go`

### 0.5.3 Documentation Files to Update Detail

No existing documentation files will be updated. Per the SWE-AtlasQnA-Repo implementation rule, the task explicitly prohibits modifying any existing files in the source repository.

### 0.5.4 Documentation Configuration Updates

No documentation configuration files (`.mkdocs.yml`, etc.) will be modified. The new document is placed in `blitzy/documentation/` which is outside the existing documentation infrastructure.

### 0.5.5 Cross-Documentation Dependencies

- The new document references internal code comments as primary sources of truth
- No shared content/includes with existing documentation
- No navigation or table of contents updates needed (the document lives in `blitzy/documentation/`, not `docs/`)
- No index or glossary updates needed

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following tools and packages are relevant to this documentation exercise:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| go | go toolchain | 1.13 (minimum per go.mod) | Required to compile and run Go test suite for test evidence collection |
| go module | github.com/foxcpp/maddy | HEAD (local) | The mail server project being documented |
| go module | github.com/emersion/go-smtp | 0.12.1-0.20191206174923 | SMTP library; defines `smtp.MailOptions`, `smtp.ConnectionState`, `smtp.EnhancedCode` types used in MsgMetadata and DSN |
| go module | github.com/emersion/go-message | 0.10.9-0.20191116124005 | Message handling; provides `textproto.Header` used by modifiers and DSN |
| pip (docs infra) | mkdocs | (configured in .mkdocs.yml) | Existing documentation generator — NOT modified by this task |

**Note:** The Go toolchain is the only runtime dependency needed for this documentation task, specifically for running the four test functions that provide evidence for the document. All Go module dependencies are pulled automatically via `go test`. The documentation output itself (`maddy.md`) is a standalone Markdown file with no build-time dependencies.

### 0.6.2 Documentation Reference Updates

No documentation reference updates are applicable. The new document is a standalone Q&A file in `blitzy/documentation/` and does not require link updates in any existing files. Per the SWE-AtlasQnA-Repo rule, no existing files are modified.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis of the address tracking mechanism:**

| Documentation Area | Files Involved | Currently Documented | Target |
|--------------------|---------------|---------------------|--------|
| `OriginalRcpts` data structure definition | `internal/module/msgmetadata.go` | Inline code comment only (8 lines) | Full standalone explanation with direction clarification |
| `AddRcpt` population logic | `internal/msgpipeline/msgpipeline.go` | Inline debug log statements only | Step-by-step flow trace with diagram |
| Modifier responsibility boundary | `internal/module/modifier.go` | 2-line inline comment (lines 49–50) | Expanded explanation of why modifiers don't track |
| DSN reverse-translation | `internal/target/queue/queue.go` | No documentation | Complete flow explanation with privacy rationale |
| LMTP `statusCollector` | `internal/msgpipeline/msgpipeline.go` | 5-line inline comment (lines 338–345) | Full explanation of reverse-translation for partial delivery |
| Test coverage documentation | 4 test files | Function names only | Descriptions, execution results, and what each proves |

**Coverage gaps to address:**
- `internal/msgpipeline/`: 0% externally documented for address tracking logic — target: 100% of the `AddRcpt` flow and `statusCollector`
- `internal/target/queue/`: 0% documented for the DSN-to-OriginalRcpts relationship — target: 100% of the `emitDSN` logic
- Misconception correction: 0% documented — target: all four claims addressed with source citations

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every factual claim in the document must cite a specific file path and line number
- The `AddRcpt` trace must account for all three modifier phases (global, source, per-recipient)
- The DSN flow must show the complete chain from `RcptErrs` iteration through `OriginalRcpts` lookup to `RecipientInfo.FinalRecipient` assignment
- All four of the colleague's claims must be explicitly addressed

**Accuracy validation:**
- Test execution results must be captured from a real `go test` run against the repository (already completed: all 4 test groups pass)
- The data structure direction claim ("final→original") must be verified against both the code comment at `msgmetadata.go:80-81` and the test assertions at `modifier_test.go:285-292`
- The "not in modify/" claim must be verified by showing that no file in `internal/modify/` references `OriginalRcpts` (confirmed via codebase-wide grep)

**Clarity standards:**
- Technical accuracy with accessible language appropriate for a developer onboarding to the codebase
- Progressive disclosure: start with the data structure, then how it's built, then how it's used
- Consistent terminology: "final address" = post-rewrite, "original address" = pre-rewrite, "reverse mapping" = the actual direction

**Maintainability:**
- Source citations with exact file paths and line numbers for traceability
- All diagrams in Mermaid format for easy updates
- Standalone document with no cross-dependencies on other documentation

### 0.7.3 Example and Diagram Requirements

- **Minimum code examples:** At least 3 — the `OriginalRcpts` field declaration, the key `AddRcpt` assignment line, and the `emitDSN` lookup
- **Diagram types required:** 1 Mermaid sequence diagram (AddRcpt modifier pipeline flow), 1 Mermaid flowchart (DSN emission reverse-translation)
- **Concrete data examples:** Use values from the test suite (e.g., `rcpt1@example.com` → `rcpt1-alias@example.com` from `TestMsgPipeline_RcptModifier_OriginalRcpt`)
- **Test evidence:** All four test runs captured with PASS/FAIL status

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/maddy.md` — The sole deliverable: a comprehensive investigative document answering the user's question about recipient address transformation tracking

**Source code to analyze and cite (read-only):**
- `internal/module/msgmetadata.go` — `OriginalRcpts` field definition and `MsgMetadata` struct
- `internal/module/modifier.go` — `ModifierState` interface and responsibility boundary comment
- `internal/module/partial_delivery.go` — `StatusCollector` and `PartialDelivery` interfaces
- `internal/msgpipeline/msgpipeline.go` — `AddRcpt` population logic, `statusCollector`, `BodyNonAtomic`, `Start` initialization
- `internal/modify/group.go` — Modifier group composition (serial chaining)
- `internal/modify/alias_file.go` — Alias file modifier implementation
- `internal/modify/replace_addr.go` — Replace address modifier implementation
- `internal/dsn/dsn.go` — DSN generation (pure serializer)
- `internal/target/queue/queue.go` — Queue `emitDSN()` method consuming `OriginalRcpts`
- `internal/testutils/modifier.go` — Mock modifier used in tests
- `internal/testutils/target.go` — Mock delivery target used in tests

**Test files to execute and document:**
- `internal/msgpipeline/modifier_test.go` — `TestMsgPipeline_RcptModifier_OriginalRcpt`, `TestMsgPipeline_RcptModifier_OriginalRcpt_Multiple`
- `internal/msgpipeline/bodynonatomic_test.go` — `TestMsgPipeline_BodyNonAtomic_ModifiedRcpt`
- `internal/target/queue/queue_test.go` — `TestQueueDSN_RcptRewrite`

**Contextual documentation files to reference:**
- `HACKING.md` — Developer guide with modifier architecture overview
- `internal/README.md` — Package directory structure
- `docs/tutorials/alias-to-remote.md` — Alias forwarding tutorial with bounce handling section
- `.mkdocs.yml` — Existing documentation configuration

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No files in the repository will be modified, added to, or deleted. This includes no changes to code comments, no docstring additions, and no test modifications. Per the SWE-AtlasQnA-Repo rule.
- **Test file modifications:** No test files will be created or modified.
- **Feature additions or code refactoring:** This is a documentation-only task.
- **Existing documentation updates:** No changes to `README.md`, `HACKING.md`, `docs/` files, or `.mkdocs.yml`.
- **Deployment configuration changes:** No changes to build scripts, CI, or packaging.
- **Unrelated documentation:** This task does not cover general maddy documentation (IMAP, SMTP setup, storage backends, authentication, etc.). Only the address transformation tracking mechanism is in scope.
- **Sender address tracking:** While `MsgMetadata.OriginalFrom` exists for sender tracing, the user's question is specifically about recipient tracking via `OriginalRcpts`.
- **DKIM signing modifier:** The `internal/modify/dkim/` package is out of scope as it does not participate in recipient address rewriting.

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Test execution command (for evidence collection):**
  - `go test ./internal/msgpipeline/ -run "TestMsgPipeline_RcptModifier_OriginalRcpt" -v -count=1`
  - `go test ./internal/msgpipeline/ -run "TestMsgPipeline_BodyNonAtomic_ModifiedRcpt" -v -count=1`
  - `go test ./internal/target/queue/ -run "TestQueueDSN_RcptRewrite" -v -count=1`
- **Default format:** Markdown with Mermaid diagrams
- **Citation requirement:** Every section must reference source files with exact paths and line numbers
- **Style guide:** Investigative Q&A format, following SWE-AtlasQnA-Repo rule structure — provide thinking/rationale behind answers, base answers on code as truth
- **Output location:** `blitzy/documentation/maddy.md`
- **No source modifications:** Per implementation rule, do not modify any existing files in the source repository

## 0.10 Rules for Documentation

The following rules are derived from the user-specified implementation rule **SWE-AtlasQnA-Repo** and the nature of the documentation task:

- **Create a new markdown document named `maddy.md`** that comprehensively answers the questions posed in the prompt
- **Provide thinking / rationale behind the answers** — each claim must be preceded by the reasoning that led to it, grounded in specific code inspection
- **Do not make assumptions, base answers on the code as the truth** — every factual statement must cite a specific file path and, where relevant, line numbers. No speculation about intent beyond what comments and code explicitly show
- **Do not modify any existing files in the source repository** — the deliverable is exclusively the new `blitzy/documentation/maddy.md` file
- **Place the generated document in the `blitzy/documentation` directory** in the destination repository
- **Address all four of the colleague's claims explicitly** — the document must not leave any of the user's questions unanswered
- **Include test execution results as evidence** — run the relevant tests and report their outcomes to prove the documented behavior is accurate
- **Use Mermaid diagrams for complex flows** — the AddRcpt pipeline and DSN emission flows should be visualized
- **Cite exact line numbers** — for all key code references to enable the reader to verify claims independently

## 0.11 References

### 0.11.1 Source Files and Folders Searched

The following files and folders were directly retrieved and inspected during context gathering to derive the conclusions in this Agent Action Plan:

**Root-level files:**

| File | Purpose in Analysis |
|------|-------------------|
| `go.mod` | Identified Go version (1.13), module path, and all dependency versions |
| `HACKING.md` | Developer guide describing module architecture, modifier patterns, and design goals |
| `internal/README.md` | Package directory structure guide mapping subsystem boundaries |
| `.mkdocs.yml` | Identified existing documentation infrastructure (MkDocs with readthedocs theme) |
| `README.md` | Project overview (not directly relevant but examined for documentation gaps) |

**Core data structure and interface files:**

| File | Key Lines | Finding |
|------|-----------|---------|
| `internal/module/msgmetadata.go` | Lines 80–89, 112 | `OriginalRcpts map[string]string` field definition; direction comment ("mapping from the final recipient to the recipient that was presented by the client"); `DeepCopy` method |
| `internal/module/modifier.go` | Lines 26–63, esp. 49–50 | `Modifier` and `ModifierState` interfaces; critical comment: "MsgPipeline will take care of populating MsgMeta.OriginalRcpts. RewriteRcpt doesn't do it." |
| `internal/module/delivery_target.go` | Lines 1–71 | `DeliveryTarget` and `Delivery` interfaces defining the delivery lifecycle |
| `internal/module/partial_delivery.go` | Lines 1–41 | `StatusCollector` interface with comment that `rcptTo` should match the value passed to `AddRcpt` |

**Pipeline orchestration files:**

| File | Key Lines | Finding |
|------|-----------|---------|
| `internal/msgpipeline/msgpipeline.go` | Lines 79–100 (Start), 227–305 (AddRcpt), 338–404 (statusCollector, BodyNonAtomic) | Complete population logic for `OriginalRcpts`; `statusCollector` reverse-translation wrapper; lazy map initialization |

**Modifier implementation files (where user searched):**

| File | Key Lines | Finding |
|------|-----------|---------|
| `internal/modify/group.go` | Lines 1–81 | `Group` serial composition; `RewriteRcpt` chains modifiers but does not touch `OriginalRcpts` |
| `internal/modify/alias_file.go` | Lines 1–308, esp. 257–296 | `alias_file` modifier; pure rewriter with no reference to `OriginalRcpts` |
| `internal/modify/replace_addr.go` | Lines 1–178, esp. 137–142 | `replace_sender`/`replace_rcpt` modifier; pure rewriter with no reference to `OriginalRcpts` |

**DSN and queue files:**

| File | Key Lines | Finding |
|------|-----------|---------|
| `internal/dsn/dsn.go` | Lines 1–276 | Pure DSN serializer; receives `RecipientInfo.FinalRecipient` already translated; does not access `OriginalRcpts` |
| `internal/target/queue/queue.go` | Lines 849–950, esp. 882–897 | `emitDSN()` method consuming `OriginalRcpts` to reverse-translate addresses in bounce recipient info |

**Test files:**

| File | Key Lines | Finding |
|------|-----------|---------|
| `internal/msgpipeline/modifier_test.go` | Lines 253–352 | `TestMsgPipeline_RcptModifier_OriginalRcpt` and `_Multiple` — verify single and chained modifier mapping |
| `internal/msgpipeline/bodynonatomic_test.go` | Lines 50–90 | `TestMsgPipeline_BodyNonAtomic_ModifiedRcpt` — verify LMTP status reverse-translation |
| `internal/target/queue/queue_test.go` | Lines 746–818 | `TestQueueDSN_RcptRewrite` — verify DSN uses original addresses, not aliases; confirms alias privacy |

**Test support files:**

| File | Key Lines | Finding |
|------|-----------|---------|
| `internal/testutils/modifier.go` | Lines 1–105 | Mock modifier with `RcptTo map[string]string` for configurable rewriting |
| `internal/testutils/target.go` | (full file) | Mock delivery target with `Msg` capture and partial delivery support |

**Documentation files examined:**

| File | Finding |
|------|---------|
| `docs/tutorials/alias-to-remote.md` | Discusses alias forwarding configuration and bounce handling — does not cover `OriginalRcpts` |
| `docs/internals/sqlite.md` | SQLite backend internals — not relevant |
| `docs/internals/quirks.md` | Protocol quirks — not relevant |

**Folders explored:**

| Folder | Depth | Purpose |
|--------|-------|---------|
| (root) | 0 | Initial repository structure assessment |
| `internal/` | 1 | Identified all subsystem packages |
| `internal/module/` | 2 | Located core interfaces and data structures |
| `internal/msgpipeline/` | 2 | Located pipeline orchestration and tests |
| `internal/modify/` | 2 | Verified modifiers do NOT contain tracking logic |
| `internal/dsn/` | 2 | Verified DSN package is a pure serializer |
| `internal/target/queue/` | 3 | Located DSN emission consuming `OriginalRcpts` |
| `internal/testutils/` | 2 | Located mock modifier and target implementations |
| `docs/` | 1 | Assessed existing documentation coverage |
| `docs/internals/` | 2 | Checked for existing internals documentation |
| `docs/tutorials/` | 2 | Checked for alias/bounce documentation |

**Codebase-wide grep:**
- `grep -rn "OriginalRcpts\|originalRcpts"` across all `.go` files — 18 occurrences across 5 files, confirming the field is used exclusively in `module/msgmetadata.go` (definition), `msgpipeline/msgpipeline.go` (population and consumption), `msgpipeline/modifier_test.go` (test assertions), `target/queue/queue.go` (DSN consumption), and `target/queue/queue_test.go` (DSN test)

### 0.11.2 Test Execution Results

All four test groups were executed against the repository and passed:

| Test Command | Result | Duration |
|-------------|--------|----------|
| `go test ./internal/msgpipeline/ -run "TestMsgPipeline_RcptModifier_OriginalRcpt" -v -count=1` | PASS (2 tests) | 0.005s |
| `go test ./internal/msgpipeline/ -run "TestMsgPipeline_BodyNonAtomic_ModifiedRcpt" -v -count=1` | PASS | 0.005s |
| `go test ./internal/msgpipeline/ -run "TestMsgPipeline_RcptModifier" -v -count=1` | PASS (8 tests) | 0.006s |
| `go test ./internal/target/queue/ -run "DSN" -v -count=1` | PASS (4 tests) | 1.009s |

### 0.11.3 Attachments

No attachments were provided by the user for this task. No Figma screens or external design assets are applicable.

