# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively answers a series of interconnected technical questions about Maddy mail server's module system, with a focus on the lifecycle from configuration parsing through runtime message processing.

**Documentation Type:** Technical deep-dive / Architecture explainer (onboarding reference document)

**Category:** Create new documentation

The user is onboarding to the Maddy mail server repository and seeks a single, authoritative document that traces the module system's behavior across four dimensions:

- **Configuration-time behavior:** How the declarative configuration file is parsed, what the parser actually produces, and how macros, snippets, and environment variables are resolved before any module sees the config. Specifically, what gets registered immediately upon parsing and what stays unresolved until later.
- **Lazy initialization and the ampersand (`&`) reference syntax:** How `ModuleFromNode` in `internal/config/module/modconfig.go` distinguishes between inline module creation and `&`-prefixed references to pre-registered instances, and how `module.GetInstance()` in `internal/module/instances.go` implements at-most-once lazy initialization with circular dependency protection.
- **Endpoint vs. regular module lifecycle divergence:** Why endpoint modules (SMTP, IMAP, LMTP) are eagerly initialized while regular modules (storage, auth, queue, checks, modifiers) are lazily initialized on first reference, and the architectural consequences of this split.
- **Message-flow-time coordination of checks and modifiers:** How the `checkRunner` in `internal/msgpipeline/check_runner.go` lazily creates per-check states, replays prior SMTP phases for late-initialized checks, runs checks in parallel goroutines with mutex-protected result merging, and how modifiers apply serially through the `modify.Group` wrapper. What runtime evidence (debug logs, `Initialized` map, unused-module errors) confirms that the module graph has settled.

### 0.1.2 Special Instructions and Constraints

- **CRITICAL — No repository modifications:** The user explicitly states "the repository itself should remain unchanged and anything temporary should be cleaned up once we are done." This aligns with the implementation rule: "Do not modify any existing files in the source repository."
- **Output location:** Per the implementation rule `SWE-AtlasQnA-Repo`, the generated document must be a new Markdown file placed in the `blitzy/documentation` directory.
- **Evidence-based reasoning:** The user requests answers grounded in the code: "Do not make assumptions, base your answers on the code as the truth." Every claim in the documentation must cite specific source files and line ranges.
- **Temporary scripts allowed:** "If needed we can use temporary scripts for observations, but the repository itself should remain unchanged." Any temporary scripts created during investigation must be cleaned up.
- **Thinking/rationale required:** "Provide thinking / rationale behind the answers."

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document configuration parsing**, we will create a section that traces the exact code path from `parser.Read()` in `pkg/cfgparser/parse.go` through snippet expansion (`imports.go`), macro expansion (`imports.go:expandMacros`), environment substitution (`env.go`), and into the `moduleMain` → `instancesFromConfig` flow in `maddy.go` lines 243–375.
- To **explain lazy initialization and `&`-syntax**, we will create a section analyzing `ModuleFromNode` in `internal/config/module/modconfig.go` (lines 54–93), the `instances` map and `GetInstance` function in `internal/module/instances.go` (lines 48–75), and the `Initialized` map that breaks circular dependencies.
- To **distinguish endpoint and regular module lifecycles**, we will create a section contrasting `FuncNewEndpoint` vs. `FuncNewModule` in `internal/module/module.go`, the separate registration maps in `registry.go`, and the two-phase loop in `instancesFromConfig` (lines 300–365) that processes endpoints eagerly and regular modules lazily.
- To **document check and modifier coordination**, we will create a section tracing the `checkRunner` in `internal/msgpipeline/check_runner.go` (lazy state creation, parallel goroutine execution, phase replay, mutex-protected merging) and the `modify.Group` in `internal/modify/group.go` (serial chain execution).
- To **identify runtime evidence of graph settlement**, we will document the unused-module error in `instancesFromConfig` (lines 358–365), the `Initialized` map state, and debug log lines emitted by `ModuleFromNode` and the check runner.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs were identified:

- **The `config.Map.AllowUnknown()` / `Process()` mechanism** requires explanation because it is the bridge between endpoint configuration parsing and pipeline construction — the SMTP endpoint's `setConfig` method calls `AllowUnknown()` so that `source`, `destination`, `check`, `modify`, `deliver_to`, and other pipeline directives are collected as "unknown" nodes and passed to `msgpipeline.New()` for pipeline configuration parsing (`internal/endpoint/smtp/smtp.go`, lines 575–586).
- **The `Future` concurrency primitive** (`internal/future/future.go`) deserves mention because it enables the reverse-DNS lookup to proceed asynchronously while the SMTP session progresses, exemplifying how module coordination occurs without explicit wiring.
- **The side-effect import pattern** in `maddy.go` lines 20–38 must be explained because it is the mechanism by which all module `init()` functions execute at process start, populating the registries before any configuration is read.
- **The snippet and macro system** in the configuration parser needs documentation because it is what enables the `$(hostname)`, `$(primary_domain)`, and `(local_delivery_actions)` patterns visible in `maddy.conf`, which the user may perceive as "declarative and order-independent" but actually involves a multi-pass expansion pipeline.


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **MkDocs-based documentation framework** with moderate coverage of user-facing guides but limited coverage of internal architecture and module system behavior.

**Current documentation framework:** MkDocs (configured via `.mkdocs.yml` at the repository root)
- **Site name:** "maddy documentation"
- **Theme:** readthedocs
- **Markdown extensions:** `codehilite` (with `guess_lang: false`)
- **Documentation generator configuration location:** `.mkdocs.yml`
- **API documentation tools in use:** None detected (no JSDoc, Godoc, or Sphinx configuration)
- **Diagram tools detected:** None explicitly configured (Mermaid will be used in the new document)
- **Documentation hosting:** `https://foxcpp.dev/maddy/` (referenced in `README.md`)

**MkDocs navigation structure** (from `.mkdocs.yml`):
```
nav:
  - README.md
  - Tutorials (setting-up, manual-installation, alias-to-remote)
  - get.sh-script.md
  - Manual pages (9 generated man pages)
  - Internals (quirks.md, sqlite.md)
```

**Existing documentation files discovered:**

| Path | Content | Coverage Status |
|------|---------|-----------------|
| `README.md` | Project overview, features, links | High-level only |
| `HACKING.md` | Developer design guide, module patterns, error handling | Architecture summary, limited detail |
| `docs/README.md` | Landing page for documentation site | Navigation only |
| `docs/tutorials/setting-up.md` | End-to-end deployment walkthrough | User-facing setup |
| `docs/tutorials/manual-installation.md` | Source/release installation | User-facing setup |
| `docs/tutorials/alias-to-remote.md` | Routing mail via reroute | Specific use case |
| `docs/get.sh-script.md` | Bootstrap installer documentation | Installation variables |
| `docs/internals/quirks.md` | SMTP/IMAP implementation quirks | Brief, focused |
| `docs/internals/sqlite.md` | SQLite backend behavior | Operational notes |
| `docs/man/README.md` | Manpage authoring workflow | Tooling guide |
| `internal/README.md` | Architectural index of internal packages | Package-level overview |
| `pkg/README.md` | Integration libraries overview | API surface documentation |
| `cmd/README.md` | Command binary index | Binary descriptions |

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used for code analysis relevant to the user's questions:

- **Module system core:** `internal/module/*.go` — All 11 files defining interfaces, registry, instances, metadata
- **Configuration parser:** `pkg/cfgparser/parse.go`, `imports.go`, `env.go` — Full parsing pipeline
- **Configuration mapping:** `internal/config/map.go`, `internal/config/module/modconfig.go` — Directive mapping and module resolution
- **Endpoint implementations:** `internal/endpoint/smtp/smtp.go`, `internal/endpoint/imap/imap.go` — Endpoint lifecycle
- **Message pipeline:** `internal/msgpipeline/msgpipeline.go`, `config.go`, `check_runner.go` — Pipeline orchestration
- **Check infrastructure:** `internal/check/stateless_check.go`, `action.go`, and all check subpackages — Check registration and execution
- **Modifier infrastructure:** `internal/modify/group.go`, `replace_addr.go`, `alias_file.go` — Modifier composition
- **Bootstrap entry point:** `maddy.go` — Server startup and module instantiation
- **Default configuration:** `maddy.conf` — Reference configuration demonstrating all patterns
- **Concurrency primitives:** `internal/future/future.go` — Async value propagation

Key directories examined: root (`maddy.go`, `maddy.conf`), `internal/module/`, `internal/config/`, `internal/config/module/`, `internal/endpoint/smtp/`, `internal/endpoint/imap/`, `internal/msgpipeline/`, `internal/check/`, `internal/modify/`, `pkg/cfgparser/`, `docs/`, `docs/internals/`

**Related documentation found:** `HACKING.md` provides the closest existing coverage to the user's questions, offering a concise design summary. However, it only describes the module system at a conceptual level ("modules are represented by objects implementing the module.Module interface") and does not trace the actual code paths for initialization, lazy resolution, or runtime coordination that the user is asking about.

### 0.2.3 Web Search Research Conducted

No external web search is required for this documentation task. The user's questions are entirely answerable from the codebase itself, and the implementation rule explicitly instructs: "Do not make assumptions, base your answers on the code as the truth." All documentation will be derived from direct code analysis of the repository.


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require documentation to answer the user's questions comprehensively:

- **Module: `maddy.go` (root bootstrap)**
  - Public APIs: `Run()`, `moduleMain()`, `instancesFromConfig()`, `InitDirs()`
  - Current documentation: `HACKING.md` provides a 3-paragraph summary; no line-level tracing exists
  - Documentation needed: Step-by-step walkthrough of the startup sequence, annotated with source lines, explaining the two-phase module loop (endpoints eager, regular modules lazy)

- **Module: `internal/module/registry.go`**
  - Public APIs: `Register()`, `Get()`, `RegisterEndpoint()`, `GetEndpoint()`
  - Current documentation: GoDoc comments only
  - Documentation needed: Explanation of the dual-registry pattern (separate maps for regular and endpoint modules), concurrency safety via `sync.RWMutex`, and the panic-on-duplicate invariant

- **Module: `internal/module/instances.go`**
  - Public APIs: `RegisterInstance()`, `RegisterAlias()`, `HasInstance()`, `GetInstance()`
  - Current documentation: GoDoc comments only
  - Documentation needed: Detailed explanation of lazy initialization, the `Initialized` map's role in circular dependency breaking (setting `true` before calling `Init`), and alias resolution

- **Module: `internal/module/module.go`**
  - Public APIs: `Module` interface, `FuncNewModule`, `FuncNewEndpoint`
  - Current documentation: GoDoc comments with design rationale
  - Documentation needed: Contrast between `FuncNewModule` and `FuncNewEndpoint` signatures and their implications for lifecycle, naming, and registry participation

- **Module: `pkg/cfgparser/parse.go`, `imports.go`, `env.go`**
  - Public APIs: `Read()`, `Node` struct, snippet/macro expansion
  - Current documentation: GoDoc comments only
  - Documentation needed: Multi-pass parsing pipeline explanation (lex → parse → expand imports/snippets → expand macros → expand environment), annotated with how `maddy.conf` patterns like `$(hostname)` and `(local_delivery_actions)` are processed

- **Module: `internal/config/module/modconfig.go`**
  - Public APIs: `ModuleFromNode()`
  - Current documentation: GoDoc comments only
  - Documentation needed: Core explanation of the `&`-prefix reference detection (line 59), the branch between inline module creation and instance lookup, reflection-based interface verification, and how inline modules are initialized immediately vs referenced modules being lazily initialized

- **Module: `internal/endpoint/smtp/smtp.go`**
  - Public APIs: `Endpoint`, `Session`, `Init()`, `setConfig()`
  - Current documentation: None beyond GoDoc
  - Documentation needed: The `AllowUnknown()` + `Process()` pattern that collects pipeline directives as unknown nodes, then passes them to `msgpipeline.New()`, and how the endpoint is eagerly initialized

- **Module: `internal/endpoint/imap/imap.go`**
  - Public APIs: `Endpoint`, `Init()`
  - Current documentation: None beyond GoDoc
  - Documentation needed: How the IMAP endpoint references `&local_authdb` and `&local_mailboxes` via `modconfig.AuthDirective` and `modconfig.StorageDirective`, triggering lazy initialization of those modules

- **Module: `internal/msgpipeline/check_runner.go`**
  - Public APIs: `checkRunner`, `checkStates()`, `runAndMergeResults()`, `applyResults()`
  - Current documentation: None
  - Documentation needed: Lazy check state creation, phase replay for late-created checks, parallel goroutine execution with `sync.WaitGroup`, mutex-protected result merging, and DMARC integration

- **Module: `internal/msgpipeline/config.go`**
  - Public APIs: `parseMsgPipelineRootCfg()`, `parseMsgPipelineSrcCfg()`, `parseMsgPipelineRcptCfg()`
  - Current documentation: None
  - Documentation needed: How check and modifier modules are resolved from pipeline configuration via `modconfig.MessageCheck()` and `modconfig.MsgModifier()` — this is where inline checks (like `require_matching_ehlo`) are created and initialized within the pipeline

- **Module: `internal/modify/group.go`**
  - Public APIs: `Group`, `ModStateForMsg()`, `RewriteSender()`, `RewriteRcpt()`, `RewriteBody()`
  - Current documentation: None
  - Documentation needed: Serial modifier chain execution pattern and its contrast with parallel check execution

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented initialization flow:** No existing document traces the complete path from `parser.Read()` through `instancesFromConfig()` with annotated code references. `HACKING.md` provides only a conceptual overview.
- **Undocumented `&`-syntax mechanics:** `HACKING.md` mentions "if configuration uses &-syntax to reference existing configuration block, ModuleFromNode simply looks it up in the global instances registry" but does not explain the lazy initialization implications or how circular dependencies are handled.
- **Undocumented endpoint lifecycle divergence:** `HACKING.md` notes "'smtp' and 'imap' modules follow a special initialization path, so they are always initialized directly" but does not explain why or what "special" means in terms of registry participation, naming, or configuration processing.
- **Undocumented check runner coordination:** The parallel execution model, phase replay mechanism, and result merging strategy in `check_runner.go` have no documentation whatsoever.
- **Undocumented pipeline configuration parsing:** The mechanism by which `AllowUnknown()` collects pipeline directives and passes them to `msgpipeline.New()` is undocumented.
- **Missing runtime observability guide:** No documentation describes what debug logs, error messages, or internal state changes indicate that the module graph has fully settled.


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The new document will be a single comprehensive Markdown file organized as follows:

```
blitzy/documentation/
└── maddy.md
    ├── Introduction & Context
    ├── 1. Configuration Parsing: From Text to Node Tree
    │   ├── The Multi-Pass Parsing Pipeline
    │   ├── Snippets, Macros, and Environment Expansion
    │   └── What the Parser Produces (the Node Tree)
    ├── 2. Module Registration: What Happens at Import Time
    │   ├── Side-Effect Imports and init() Functions
    │   ├── The Dual-Registry Pattern
    │   └── Complete Module Inventory
    ├── 3. From Config to Running Modules: instancesFromConfig
    │   ├── Phase 1: Classify and Instantiate
    │   ├── Phase 2: Eager Endpoint Initialization
    │   ├── Phase 3: Unused Module Verification
    │   └── The AllowUnknown Pipeline Bridge
    ├── 4. Lazy Initialization and the & Reference Syntax
    │   ├── How ModuleFromNode Resolves References
    │   ├── GetInstance and At-Most-Once Initialization
    │   ├── Circular Dependency Breaking
    │   └── Tracing a Concrete Reference Chain
    ├── 5. Endpoint vs. Regular Module Lifecycle
    │   ├── FuncNewEndpoint vs. FuncNewModule
    │   ├── Registry Participation and Naming
    │   ├── Configuration Processing Differences
    │   └── Why the Divergence Exists
    ├── 6. Message Flow: Check and Modifier Coordination
    │   ├── The checkRunner: Lazy States and Phase Replay
    │   ├── Parallel Check Execution and Result Merging
    │   ├── The Modifier Chain: Serial Group Execution
    │   ├── DMARC Integration into the Check Pipeline
    │   └── Two-Level Source/Destination Routing
    ├── 7. Runtime Evidence of Module Graph Settlement
    │   ├── The Initialized Map
    │   ├── Unused Module Error
    │   ├── Debug Logging Evidence
    │   └── systemd Readiness Notification
    └── Conclusion
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract the startup sequence from `maddy.go` lines 99–375, annotating each phase with source citations
- Extract the parsing pipeline from `pkg/cfgparser/parse.go` (`Read` function), `imports.go` (`expandImports`, `expandMacros`), and `env.go` (`expandEnvironment`)
- Extract the module resolution logic from `internal/config/module/modconfig.go` lines 54–93
- Extract lazy initialization mechanics from `internal/module/instances.go` lines 48–75
- Extract check runner coordination from `internal/msgpipeline/check_runner.go` lines 50–317
- Extract modifier chain behavior from `internal/modify/group.go` lines 22–80
- Generate concrete examples by tracing the default `maddy.conf` through the code paths

**Documentation Standards:**
- Markdown formatting with proper header hierarchy (`#`, `##`, `###`)
- Mermaid diagrams for initialization flow, module resolution, and message pipeline coordination
- Code citations as inline references: `Source: /path/to/file.go:LineNumber`
- Short code snippets (2–3 lines) for clarity at critical decision points
- Tables for module inventories, lifecycle comparisons, and check results

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create within the documentation:

- **Flowchart:** Configuration parsing multi-pass pipeline (lex → parse → expand → normalize)
- **Flowchart:** `instancesFromConfig` two-phase loop (endpoint eager, regular lazy)
- **Sequence diagram:** Lazy initialization chain triggered by `&local_mailboxes` reference in IMAP endpoint Init
- **Flowchart:** `ModuleFromNode` decision tree (ampersand detection → instance lookup vs. inline creation)
- **Sequence diagram:** `checkRunner` lazy state creation with phase replay during message processing
- **Flowchart:** Two-level source/destination routing in the message pipeline
- **State diagram:** Module lifecycle states (Registered → Instantiated → Initialized → Running → Closed)


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/maddy.md` | CREATE | `maddy.go`, `internal/module/registry.go`, `internal/module/instances.go`, `internal/module/module.go`, `pkg/cfgparser/parse.go`, `pkg/cfgparser/imports.go`, `pkg/cfgparser/env.go`, `internal/config/module/modconfig.go`, `internal/config/map.go`, `internal/endpoint/smtp/smtp.go`, `internal/endpoint/imap/imap.go`, `internal/msgpipeline/msgpipeline.go`, `internal/msgpipeline/config.go`, `internal/msgpipeline/check_runner.go`, `internal/check/stateless_check.go`, `internal/modify/group.go`, `internal/future/future.go`, `maddy.conf`, `HACKING.md` | Complete technical deep-dive answering all user questions about the module system lifecycle, configuration parsing, lazy initialization, endpoint vs. regular module divergence, check/modifier coordination during message flow, and runtime evidence of graph settlement |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/maddy.md
Type: Technical deep-dive / Architecture explainer
Source Code:
  - maddy.go (bootstrap, instancesFromConfig)
  - internal/module/registry.go (dual-registry pattern)
  - internal/module/instances.go (lazy initialization, GetInstance)
  - internal/module/module.go (Module interface, FuncNewModule vs FuncNewEndpoint)
  - pkg/cfgparser/parse.go (Node struct, Read function, parsing pipeline)
  - pkg/cfgparser/imports.go (snippet/import expansion, macro expansion)
  - pkg/cfgparser/env.go (environment variable substitution)
  - internal/config/module/modconfig.go (ModuleFromNode, & reference detection)
  - internal/config/map.go (Map, AllowUnknown, Process)
  - internal/endpoint/smtp/smtp.go (SMTP endpoint Init, setConfig, pipeline bridge)
  - internal/endpoint/imap/imap.go (IMAP endpoint Init, auth/storage references)
  - internal/msgpipeline/msgpipeline.go (MsgPipeline, delivery lifecycle)
  - internal/msgpipeline/config.go (pipeline config parsing, check/modifier resolution)
  - internal/msgpipeline/check_runner.go (checkRunner, phase replay, parallel execution)
  - internal/check/stateless_check.go (RegisterStatelessCheck, stateless adapter)
  - internal/check/dns/dns.go (concrete check registration example)
  - internal/modify/group.go (Group, serial modifier chain)
  - internal/modify/replace_addr.go (concrete modifier registration)
  - internal/future/future.go (Future concurrency primitive)
  - maddy.conf (reference configuration demonstrating all patterns)
  - HACKING.md (existing design guide for context)
Sections:
  - Introduction & Context (purpose, user questions restated)
  - Configuration Parsing (multi-pass pipeline, snippets, macros, environment)
  - Module Registration (side-effect imports, init(), dual registry)
  - instancesFromConfig (two-phase loop, eager vs. lazy, AllowUnknown bridge)
  - Lazy Initialization and & Syntax (ModuleFromNode, GetInstance, circular deps)
  - Endpoint vs. Regular Module Lifecycle (naming, registry, Init timing)
  - Message Flow Coordination (checkRunner, parallel checks, serial modifiers, DMARC)
  - Runtime Evidence of Graph Settlement (Initialized map, unused errors, debug logs, systemd)
  - Conclusion
Diagrams:
  - Flowchart: Configuration parsing multi-pass pipeline
  - Flowchart: instancesFromConfig two-phase loop
  - Sequence diagram: Lazy initialization triggered by & reference
  - Flowchart: ModuleFromNode decision tree
  - Sequence diagram: checkRunner phase replay
  - Flowchart: Two-level message pipeline routing
Key Citations:
  - maddy.go:20-38 (side-effect imports)
  - maddy.go:243-287 (moduleMain)
  - maddy.go:294-375 (instancesFromConfig)
  - internal/module/registry.go:7-65 (dual registry)
  - internal/module/instances.go:48-75 (GetInstance lazy init)
  - internal/module/module.go:29-71 (Module interface, constructor types)
  - pkg/cfgparser/parse.go:18-43 (Node struct)
  - pkg/cfgparser/imports.go:10-156 (import/macro expansion)
  - internal/config/module/modconfig.go:54-93 (ModuleFromNode)
  - internal/config/map.go:69-73 (AllowUnknown)
  - internal/config/map.go:559-597 (Process, unknown collection)
  - internal/endpoint/smtp/smtp.go:500-609 (Init, setConfig, pipeline bridge)
  - internal/endpoint/smtp/smtp.go:714-720 (init registration)
  - internal/endpoint/imap/imap.go:53-119 (Init, auth/storage references)
  - internal/msgpipeline/check_runner.go:50-140 (checkStates, phase replay)
  - internal/msgpipeline/check_runner.go:142-209 (runAndMergeResults, parallel execution)
  - internal/msgpipeline/config.go:25-129 (parseMsgPipelineRootCfg)
  - internal/msgpipeline/config.go:350-374 (parseChecksGroup, parseModifiersGroup)
  - internal/modify/group.go:22-80 (Group, serial chain)
```

### 0.5.3 Documentation Files to Update Detail

No existing documentation files will be updated. Per the implementation rule: "Do not modify any existing files in the source repository."

### 0.5.4 Documentation Configuration Updates

No documentation configuration files will be modified. The new document is placed in `blitzy/documentation/` which is an output directory managed by the Blitzy platform, outside the MkDocs navigation tree.

### 0.5.5 Cross-Documentation Dependencies

- **Shared context from `HACKING.md`:** The new document builds upon and deepens the design summary in `HACKING.md` lines 32–62. The new document will reference `HACKING.md` for background context but will not duplicate its content — instead, it will trace the actual code paths that `HACKING.md` only describes conceptually.
- **Relationship to `docs/internals/`:** The new document fills a gap between the user-facing tutorials in `docs/tutorials/` and the implementation-specific notes in `docs/internals/` (which cover SQLite behavior and protocol quirks but not module system architecture).
- **No navigation or index updates needed:** Since the output goes to `blitzy/documentation/`, no changes to `.mkdocs.yml` or any other navigation configuration are required.


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No documentation build tools or external packages are required for this task. The deliverable is a standalone Markdown file (`blitzy/documentation/maddy.md`) that uses only standard Markdown syntax and Mermaid diagram notation. Mermaid diagrams are embedded inline using fenced code blocks and are renderable by any Mermaid-compatible Markdown viewer (GitHub, GitLab, VS Code, MkDocs with the `mermaid2` plugin, etc.).

For reference, the project's existing documentation dependencies are:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| pip | mkdocs | Configured in `.mkdocs.yml` | Documentation site generator (existing project infra) |
| system | scdoc | Referenced in `docs/man/README.md` | Manpage generation from scdoc format |
| python3 | docs/man/prepare_md.py | Custom script | Converts scdoc manpages to Markdown |

The project itself is a Go module with the following key dependencies relevant to the documentation content (these are the libraries that the documented code uses and references, extracted from `go.mod`):

| Registry | Package Name | Version | Relevance to Documentation |
|----------|--------------|---------|---------------------------|
| Go module | `github.com/foxcpp/maddy` | v0.2.2 (module path) | The project itself |
| Go module | `github.com/emersion/go-smtp` | v0.12.1 | SMTP endpoint (documented lifecycle) |
| Go module | `github.com/emersion/go-imap` | v1.0.1 | IMAP endpoint (documented lifecycle) |
| Go module | `github.com/foxcpp/go-imap-sql` | v0.3.2 | SQL storage backend (documented module) |
| Go module | `github.com/miekg/dns` | v1.1.22 | DNS resolver (used by checks) |
| Go module | `github.com/emersion/go-msgauth` | v0.4.0 | DKIM/DMARC (documented check integration) |

### 0.6.2 Documentation Reference Updates

Not applicable. No existing documentation links need updating since this is a new standalone document in a separate directory (`blitzy/documentation/`), and no existing repository files are being modified.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis of the user's specific questions:**

| Question Area | Covered By Existing Docs | Coverage Level |
|---------------|--------------------------|----------------|
| Configuration parsing pipeline | None | 0% — No existing doc traces the parsing pipeline |
| What gets registered immediately vs. lazily | `HACKING.md` lines 60–62 | ~10% — One sentence mentions lazy init |
| Lazy initialization and `&` syntax | `HACKING.md` lines 54–58 | ~15% — Conceptual overview, no code-level detail |
| Endpoint vs. regular module lifecycle | `HACKING.md` lines 60–62 | ~5% — One sentence, no elaboration |
| Check/modifier coordination during message flow | None | 0% — No existing documentation |
| Runtime evidence of module graph settlement | None | 0% — No existing documentation |

**Target coverage:** 100% of the user's stated questions, with every claim traced to specific source file and line references.

**Coverage gaps to address:**
- Configuration parsing: Currently 0%, target 100% — Full pipeline walkthrough from `parser.Read()` through all expansion passes
- Module registration: Currently ~10%, target 100% — Side-effect imports, dual registry, complete inventory
- Lazy initialization: Currently ~15%, target 100% — `GetInstance`, `Initialized` map, circular dependency breaking, concrete trace through `maddy.conf`
- Endpoint divergence: Currently ~5%, target 100% — Contrasting `FuncNewModule` vs. `FuncNewEndpoint`, registry participation, Init timing, naming
- Check/modifier coordination: Currently 0%, target 100% — `checkRunner`, phase replay, parallel execution, `modify.Group` serial chain
- Runtime observability: Currently 0%, target 100% — `Initialized` map, unused module error, debug logs, systemd notification

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every user question must receive a direct, code-cited answer
- Each section must include at least one Mermaid diagram illustrating the described behavior
- All source citations must include file path and line number(s)
- The document must be self-contained — a reader should not need to read other documentation to understand the answers

**Accuracy validation:**
- Every code citation must be verified against the actual file contents retrieved during analysis
- Mermaid diagrams must accurately reflect the control flow as implemented in the source code
- No assumptions or "typical patterns" — only behavior observed in the repository

**Clarity standards:**
- Technical accuracy with accessible language suitable for an onboarding developer
- Progressive disclosure: start with high-level overview, then drill into implementation detail
- Consistent terminology: use "module" (not "component" or "plugin"), "endpoint" (not "listener" or "server"), "check" (not "filter" or "validator"), "modifier" (not "rewriter" or "transformer") throughout
- Each section answers one primary question from the user's prompt

**Maintainability:**
- Source citations with file paths and line ranges for traceability
- Mermaid diagrams for visual comprehension
- Section structure mirrors the temporal flow (parsing → registration → instantiation → initialization → message processing → evidence) for logical navigation

### 0.7.3 Example and Diagram Requirements

- **Minimum code-traced examples:** At least one concrete trace through `maddy.conf` for each major concept (lazy init, `&`-reference, endpoint Init, check coordination)
- **Diagram types required:** Flowcharts (parsing pipeline, instancesFromConfig, ModuleFromNode), Sequence diagrams (lazy init chain, checkRunner phase replay), State diagram (module lifecycle)
- **Code example testing:** All code snippets are extracted verbatim from the repository and cross-referenced with retrieved file contents; no synthetic examples
- **Thinking/rationale:** Each answer will include explicit reasoning explaining why the code works the way it does, per the implementation rule


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/maddy.md` — The sole deliverable: a comprehensive Markdown document answering all user questions about the module system lifecycle

**Source code files analyzed for documentation content (read-only — no modifications):**
- `maddy.go` — Bootstrap, `Run()`, `moduleMain()`, `instancesFromConfig()`
- `maddy.conf` — Default configuration demonstrating all patterns (`&` syntax, snippets, macros, mixed endpoints)
- `HACKING.md` — Design guide, referenced for contextual framing
- `internal/module/registry.go` — Dual module/endpoint registry with `sync.RWMutex`
- `internal/module/instances.go` — Instance management, `GetInstance()`, lazy init, `Initialized` map
- `internal/module/module.go` — `Module` interface, `FuncNewModule`, `FuncNewEndpoint` constructor types
- `internal/module/check.go` — `Check`, `EarlyCheck`, `CheckState`, `CheckResult` interfaces
- `internal/module/modifier.go` — `Modifier`, `ModifierState` interfaces
- `internal/module/delivery_target.go` — `DeliveryTarget`, `Delivery` interfaces
- `internal/module/dummy.go` — Example of `init()` registration pattern
- `pkg/cfgparser/parse.go` — `Node` struct, `Read()` function, recursive parsing
- `pkg/cfgparser/imports.go` — `expandImports()`, `resolveImport()`, `expandMacros()`
- `pkg/cfgparser/env.go` — `expandEnvironment()`, environment variable substitution
- `internal/config/map.go` — `Map` struct, `AllowUnknown()`, `Process()`, unknown node collection
- `internal/config/module/modconfig.go` — `ModuleFromNode()`, `&`-prefix detection, inline vs. reference resolution
- `internal/config/module/check.go` — `MessageCheck()` adapter
- `internal/config/module/delivery.go` — `DeliveryTarget()` adapter
- `internal/config/module/storage.go` — `StorageDirective()` adapter
- `internal/config/module/auth.go` — `AuthDirective()` adapter
- `internal/config/module/modifier.go` — `MsgModifier()` adapter
- `internal/endpoint/smtp/smtp.go` — SMTP `Endpoint`, `Init()`, `setConfig()`, `init()` registration, `AllowUnknown`→pipeline bridge
- `internal/endpoint/imap/imap.go` — IMAP `Endpoint`, `Init()`, `auth`/`storage` directive processing
- `internal/msgpipeline/msgpipeline.go` — `MsgPipeline`, `Start()`, `AddRcpt()`, `Body()`, delivery lifecycle
- `internal/msgpipeline/config.go` — `parseMsgPipelineRootCfg()`, `parseChecksGroup()`, `parseModifiersGroup()`
- `internal/msgpipeline/check_runner.go` — `checkRunner`, `checkStates()`, `runAndMergeResults()`, `applyResults()`
- `internal/check/stateless_check.go` — `RegisterStatelessCheck()`, `statelessCheck` adapter
- `internal/check/action.go` — `FailAction`, `Apply()`
- `internal/check/dns/dns.go` — Concrete check registrations (`require_matching_ehlo`, `require_mx_record`, `require_matching_rdns`)
- `internal/check/dkim/dkim.go` — `verify_dkim` registration
- `internal/check/spf/spf.go` — `apply_spf` registration
- `internal/modify/group.go` — `Group`, `ModStateForMsg()`, serial chain execution
- `internal/modify/replace_addr.go` — `replace_sender`/`replace_rcpt` registration
- `internal/modify/alias_file.go` — `alias_file` registration
- `internal/future/future.go` — `Future` concurrency primitive
- `internal/storage/sql/sql.go` — `sql` module registration (dual AuthProvider + Storage)

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No existing repository files will be created, updated, or deleted. This is enforced by both the user's instructions ("the repository itself should remain unchanged") and the implementation rule ("Do not modify any existing files in the source repository").
- **Test file modifications:** No test files will be changed.
- **Feature additions or code refactoring:** Not applicable — this is a documentation-only task.
- **Deployment configuration changes:** No systemd units, Fail2ban rules, or other dist/ assets will be modified.
- **MkDocs configuration changes:** `.mkdocs.yml` will not be modified since the output lives in `blitzy/documentation/`, not in the project's `docs/` tree.
- **Existing documentation updates:** `README.md`, `HACKING.md`, `docs/internals/*.md`, and all other existing documentation files will remain untouched.
- **Unrelated module internals:** Detailed documentation of specific check algorithms (e.g., SPF evaluation logic, DKIM signature verification math), storage SQL schema details, SMTP protocol compliance details, and IMAP extension negotiation are out of scope unless they directly bear on the module lifecycle questions.
- **Build/CI pipeline documentation:** Documentation of `.build.yml`, `.golangci.yml`, or CI/CD processes is out of scope.
- **User-facing tutorial creation:** No deployment guides, installation tutorials, or administrator how-tos are in scope.


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the deliverable is a standalone Markdown file that does not require a build step. It is renderable by any Markdown viewer that supports Mermaid fenced code blocks.
- **Documentation preview command:** Not applicable.
- **Diagram generation command:** Mermaid diagrams are embedded inline in the Markdown using mermaid-fenced code blocks. No separate generation step is needed.
- **Documentation deployment command:** Not applicable — the file is placed directly in `blitzy/documentation/maddy.md`.
- **Default format:** Markdown with Mermaid diagrams.
- **Citation requirement:** Every technical claim must reference a specific source file path and line number or range (e.g., `Source: internal/module/instances.go:65-70`).
- **Style guide:** Follow the conventions observed in the repository's existing documentation (`HACKING.md`, `docs/internals/`): direct, technical prose; short paragraphs; code-adjacent explanations; avoid marketing language.
- **Documentation validation:** Manual review of all source citations against retrieved file contents. All Mermaid diagrams validated for syntax correctness.
- **Naming convention:** The output file must be named `maddy.md` per the implementation rule: "Create a new markdown document named `<project name>.md`."
- **Output directory:** `blitzy/documentation/` per the implementation rule.


## 0.10 Rules for Documentation

### 0.10.1 User-Specified Rules

The following rules are derived from the user's prompt and the attached implementation rules:

- **"Do not modify any existing files in the source repository."** — No file in the repository may be created, edited, or deleted. Only the new `blitzy/documentation/maddy.md` file is produced.
- **"Do not make assumptions, base your answers on the code as the truth."** — Every statement in the document must be traceable to a specific source file and line range. Speculation, inference from naming conventions, or "typical Go patterns" must not be used as evidence.
- **"Provide thinking / rationale behind the answers."** — Each major answer section must include a rationale paragraph explaining why the code works the way it does, not just what it does. For example, when explaining lazy initialization, the document must explain why the `Initialized` flag is set before `Init()` is called (to break circular dependencies).
- **"Create a new markdown document named `<project name>.md`"** — The output file must be exactly `maddy.md`, placed in the `blitzy/documentation` directory.
- **"the repository itself should remain unchanged and anything temporary should be cleaned up once we are done"** — Any temporary scripts or files created during investigation must be removed before completion. The final state of the repository must be identical to its initial state, with only the new `blitzy/documentation/maddy.md` added in the designated output directory.
- **"If needed we can use temporary scripts for observations"** — Temporary diagnostic scripts are permitted during the analysis phase but must not persist.

### 0.10.2 Documentation Content Rules

- **Source citations for all technical details:** Every code behavior described must cite the specific file path and line number(s).
- **Consistent terminology from the codebase:** Use the exact terms from the code — "module" (not "plugin"), "endpoint" (not "listener"), "check" (not "filter"), "modifier" (not "transformer"), "instance" (not "object"), "factory" (not "constructor" unless quoting the code), "Init" (capitalized, matching the method name).
- **Include Mermaid diagrams for all major flows:** Each section answering a primary user question must include at least one Mermaid diagram.
- **Code snippets must be short and focused:** Maximum 2–3 lines per snippet, showing only the critical decision point or call site.
- **Progressive disclosure:** Start each section with a one-paragraph summary answer, then drill into the implementation detail.
- **No redundancy with HACKING.md:** The document should reference `HACKING.md` for high-level context but must go significantly deeper into code-level mechanics.


## 0.11 References

### 0.11.1 Codebase Files and Folders Searched

The following files and folders were comprehensively searched and analyzed to derive the conclusions in this Agent Action Plan:

**Root-level files:**

| File | Purpose in Analysis |
|------|---------------------|
| `maddy.go` | Server bootstrap, `Run()`, `moduleMain()`, `instancesFromConfig()` — the central startup flow and two-phase module loop |
| `maddy.conf` | Default configuration — demonstrates `&`-reference syntax, snippets, macros, mixed SMTP/IMAP endpoints |
| `HACKING.md` | Existing developer design guide — assessed for existing coverage of module system architecture |
| `config.go` | Logging configuration glue — assessed for relevance to module lifecycle |
| `go.mod` | Go module identity and dependency versions |
| `.mkdocs.yml` | Documentation framework configuration — assessed existing documentation infrastructure |
| `README.md` | Project overview — assessed for documentation coverage |

**Module system core (`internal/module/`):**

| File | Purpose in Analysis |
|------|---------------------|
| `internal/module/registry.go` | Dual-registry implementation (`modules` and `endpoints` maps), `Register()`, `RegisterEndpoint()`, `Get()`, `GetEndpoint()` |
| `internal/module/instances.go` | Instance management, `RegisterInstance()`, `RegisterAlias()`, `GetInstance()` with lazy init and circular dependency breaking |
| `internal/module/module.go` | `Module` interface, `FuncNewModule` and `FuncNewEndpoint` constructor type definitions |
| `internal/module/check.go` | `Check`, `EarlyCheck`, `CheckState`, `CheckResult` interfaces — check lifecycle contract |
| `internal/module/modifier.go` | `Modifier`, `ModifierState` interfaces — modifier lifecycle contract |
| `internal/module/delivery_target.go` | `DeliveryTarget`, `Delivery` interfaces — delivery lifecycle contract |
| `internal/module/dummy.go` | `Dummy` module — example of `init()` registration pattern |
| `internal/module/msgmetadata.go` | `MsgMetadata`, `ConnState` — message metadata model |
| `internal/module/partial_delivery.go` | `PartialDelivery`, `StatusCollector` — LMTP partial delivery contract |
| `internal/module/storage.go` | `Storage` interface — storage backend contract |
| `internal/module/auth.go` | `AuthProvider` interface — authentication contract |

**Configuration system (`pkg/cfgparser/`, `internal/config/`):**

| File | Purpose in Analysis |
|------|---------------------|
| `pkg/cfgparser/parse.go` | `Node` struct, `Read()`, `readNode()`, `readNodes()` — parsing pipeline |
| `pkg/cfgparser/imports.go` | `expandImports()`, `resolveImport()`, `expandMacros()`, `expandSingleValueMacro()` — snippet/macro expansion |
| `pkg/cfgparser/env.go` | `expandEnvironment()` — environment variable substitution |
| `internal/config/map.go` | `Map` struct, `NewMap()`, `AllowUnknown()`, `Process()`, `ProcessWith()` — directive mapping and unknown collection |
| `internal/config/config.go` | `Node` type alias, `NodeErr()` helper |
| `internal/config/module/modconfig.go` | `ModuleFromNode()`, `createInlineModule()`, `initInlineModule()` — module resolution from config |
| `internal/config/module/check.go` | `MessageCheck()` — check resolution adapter |
| `internal/config/module/delivery.go` | `DeliveryDirective()`, `DeliveryTarget()` — delivery target resolution |
| `internal/config/module/storage.go` | `StorageDirective()` — storage resolution adapter |
| `internal/config/module/auth.go` | `AuthDirective()` — auth resolution adapter |
| `internal/config/module/modifier.go` | `MsgModifier()` — modifier resolution adapter |

**Endpoint implementations (`internal/endpoint/`):**

| File | Purpose in Analysis |
|------|---------------------|
| `internal/endpoint/smtp/smtp.go` | SMTP `Endpoint`, `Session`, `New()`, `Init()`, `setConfig()`, `init()` registration, `AllowUnknown`→pipeline bridge |
| `internal/endpoint/imap/imap.go` | IMAP `Endpoint`, `New()`, `Init()`, auth/storage directive processing |

**Message pipeline (`internal/msgpipeline/`):**

| File | Purpose in Analysis |
|------|---------------------|
| `internal/msgpipeline/msgpipeline.go` | `MsgPipeline`, `Start()`, `AddRcpt()`, `Body()`, `Commit()`, `Abort()`, routing logic |
| `internal/msgpipeline/config.go` | `parseMsgPipelineRootCfg()`, `parseMsgPipelineSrcCfg()`, `parseMsgPipelineRcptCfg()`, `parseChecksGroup()`, `parseModifiersGroup()` |
| `internal/msgpipeline/check_runner.go` | `checkRunner`, `checkStates()`, `runAndMergeResults()`, `checkConnSender()`, `checkRcpt()`, `checkBody()`, `applyResults()`, `close()` |

**Check and modifier implementations:**

| File | Purpose in Analysis |
|------|---------------------|
| `internal/check/stateless_check.go` | `RegisterStatelessCheck()`, `statelessCheck` adapter, `statelessCheckState` |
| `internal/check/action.go` | `FailAction`, `FailActionDirective`, `Apply()` |
| `internal/check/dns/dns.go` | `require_matching_ehlo`, `require_mx_record`, `require_matching_rdns` registrations |
| `internal/check/dkim/dkim.go` | `verify_dkim` registration |
| `internal/check/spf/spf.go` | `apply_spf` registration |
| `internal/check/requiretls/requiretls.go` | `require_tls` registration |
| `internal/modify/group.go` | `Group`, `ModStateForMsg()`, serial modifier chain |
| `internal/modify/replace_addr.go` | `replace_sender`, `replace_rcpt` registrations |
| `internal/modify/alias_file.go` | `alias_file` registration |

**Support libraries:**

| File | Purpose in Analysis |
|------|---------------------|
| `internal/future/future.go` | `Future` concurrency primitive — async value propagation pattern |
| `internal/storage/sql/sql.go` | `sql` module registration (dual `AuthProvider` + `Storage`) |

**Documentation files:**

| File | Purpose in Analysis |
|------|---------------------|
| `docs/README.md` | Documentation landing page — assessed for existing coverage |
| `docs/internals/quirks.md` | Implementation quirks — assessed for relevant content |
| `docs/internals/sqlite.md` | SQLite behavior — assessed for relevant content |
| `docs/man/README.md` | Manpage workflow — assessed documentation infrastructure |
| `internal/README.md` | Internal package index — assessed for module system overview |
| `pkg/README.md` | Public API documentation — assessed for cfg parser coverage |
| `cmd/README.md` | Command binary index — assessed for completeness |

**Folders explored:**

| Folder | Depth | Purpose |
|--------|-------|---------|
| Root (`""`) | Level 0 | Repository structure overview |
| `internal/` | Level 1 | Internal packages inventory |
| `internal/module/` | Level 2 | Module system core (11 files) |
| `internal/config/` | Level 2 | Configuration backbone |
| `internal/config/module/` | Level 3 | Module resolution adapters |
| `internal/endpoint/smtp/` | Level 3 | SMTP endpoint implementation |
| `internal/endpoint/imap/` | Level 3 | IMAP endpoint implementation |
| `internal/msgpipeline/` | Level 2 | Message pipeline (13 files) |
| `internal/check/` | Level 2 | Check infrastructure and subpackages |
| `internal/modify/` | Level 2 | Modifier infrastructure |
| `pkg/cfgparser/` | Level 2 | Configuration parser |
| `docs/` | Level 1 | Documentation hub |
| `docs/internals/` | Level 2 | Internal documentation |
| `cmd/` | Level 1 | Command binaries |
| `pkg/` | Level 1 | Public integration libraries |

### 0.11.2 Attachments

No attachments were provided by the user. No Figma screens or external design artifacts were referenced.

### 0.11.3 External References

No external URLs or resources are required for this documentation task. All content is derived from the repository codebase itself.

**Tech spec sections consulted for background context:**
- Section 1.1 "Executive Summary" — Project overview and design goals
- Section 4.1 "High-Level System Workflow" — Server startup lifecycle and message processing flow
- Section 5.2 "Component Details" — Module system foundation, registry architecture, endpoint details, check engine, message pipeline


