# Maddy Module System: Startup Assembly — Onboarding Q&A

> An onboarding guide for engineers new to [Maddy](https://github.com/foxcpp/maddy)
> (`github.com/foxcpp/maddy` [go.mod:L1]). It explains, **strictly from the source code**,
> how Maddy's module system assembles itself when the server starts — with special
> attention to configurations that **mix SMTP and IMAP endpoints**.

This document answers six questions about how Maddy wires its modules together at boot.
It is grounded entirely in the source at branch `maddy_26452dd8dd78`: **every factual
claim carries an inline `[path:line]` citation** to verified code, and the design
rationale ("why it is built this way") is taken from the source comments themselves.
The runtime evidence in the appendix was produced by compiling the tree with `go1.23.12`
(the repository declares a minimum of `go 1.13` [go.mod:L3]) and running the resulting
binary against throwaway configurations; the captured `-debug` logs are reproduced
verbatim. Code is the single source of truth here — nothing below is generic mail-server
lore.

**The six questions:**

1. **Q1** — How does the module system come together at startup, especially for a config that mixes SMTP and IMAP endpoints?
2. **Q2** — When the config is read, what is registered immediately versus what stays unresolved until later?
3. **Q3** — How does lazy initialization work, and what makes the `&` reference syntax — and out-of-order references — possible?
4. **Q4** — Where do endpoint modules' lifecycles diverge from regular modules, and why?
5. **Q5** — As messages flow, how do `check` and `modify` modules coordinate without explicit wiring?
6. **Q6** — What runtime evidence shows the module graph has fully settled?

---

## Orientation: The Two-Registry Mental Model (read this first)

The single biggest source of confusion when reading Maddy's bootstrap is conflating its
**two distinct registries**. Almost every answer below depends on keeping them separate,
so we establish the model up front.

### Registry #1 — the *constructor* registry (populated EAGERLY, at import time)

Maddy keeps two global maps of **module constructors** in
`internal/module/registry.go`: `modules` for regular modules (type `FuncNewModule`) and
`endpoints` for endpoint modules (type `FuncNewEndpoint`), both guarded by a single
`sync.RWMutex` [internal/module/registry.go:L7-L11]. Constructors enter these maps through
`Register` and `RegisterEndpoint`, each of which **panics on a duplicate name**
[internal/module/registry.go:L19-L28; internal/module/registry.go:L56-L65].

These functions are meant to be called from each module package's `init()` — the comment
on `Register` says so directly:

> "You probably want to call this function from func init() of module package."
> [internal/module/registry.go:L18]

Those `init()` functions actually run because the entrypoint **blank-imports every module
package purely for its side effects**. The comment "Import packages for side-effect of
module registration." sits atop the import block [maddy.go:L20], which pulls in (among
others) the checks [maddy.go:L24-L29], `internal/endpoint/imap` [maddy.go:L30],
`internal/endpoint/smtp` [maddy.go:L31], `internal/modify` [maddy.go:L32], and the
storage/target packages [maddy.go:L34-L37]. This is the canonical Go "register-on-init"
idiom — the same pattern the standard library's `database/sql` uses for driver
registration. Everything else in this document is code-cited; this single sentence is the
one piece of external corroboration.

**The mental hook:** the constructor registry answers *"what types of modules exist?"* It
is fixed at compile/import time, before a single byte of configuration is read.

### Registry #2 — the *instance* registry (populated at parse time, initialized LAZILY)

The second registry lives in `internal/module/instances.go` and holds three global maps:
`instances` (instance name → the constructed module plus its `config.Map`), `aliases`
(alias → instance name), and `Initialized` (instance name → bool)
[internal/module/instances.go:L9-L17]. Top-level configuration blocks become *instances*
here via `RegisterInstance` [internal/module/instances.go:L23-L28]; crucially, they are
**constructed but not yet initialized** — their `Init` method is not called until the
instance is first referenced. The function that performs that first-reference
initialization is `GetInstance`, the at-most-once lazy-init engine
[internal/module/instances.go:L53-L75].

**The mental hook:** the instance registry answers *"which configured blocks exist?"* It is
populated only after the configuration file has been parsed.

| | Constructor registry | Instance registry |
|---|---|---|
| **File** | `internal/module/registry.go` [L7-L11] | `internal/module/instances.go` [L9-L17] |
| **Holds** | module *types* (factory funcs) | configured *blocks* (live objects) |
| **Populated** | eagerly, at package `init()` (import time) [maddy.go:L20-L38] | at config parse time, via `RegisterInstance` [maddy.go:L338] |
| **Keyed by** | module name (`smtp`, `sql`, `dummy`, …) | instance name (`local_authdb`, `remote_queue`, …) |
| **`Init` called?** | n/a (factories, not instances) | lazily, on first `&`-reference [instances.go:L70] |
| **Answers** | "what module types exist?" | "which config blocks exist?" |

Keep these separate and the rest of the bootstrap reads cleanly.

---

## The Startup Sequence (in order)

With the two registries in mind, here is the boot path from process start to "graph
settled," in the exact order it executes.

**Step 0 — constructors self-register (before `main` logic).** As described above, every
imported module package's `init()` runs at program load and calls
`Register`/`RegisterEndpoint`, filling the constructor maps [internal/module/registry.go:L7-L65;
maddy.go:L20-L38]. For example, the SMTP package registers three personalities to one
constructor [internal/endpoint/smtp/smtp.go:L715-L717] and the IMAP package registers one
[internal/endpoint/imap/imap.go:L221].

**Step 1 — the entire config is parsed first.** `parser.Read` returns the whole expanded
configuration `Node` tree *before any module logic runs*: it calls `readTree`, then
`expandEnvironment`, and returns the full `[]Node` [pkg/cfgparser/parse.go:L294-L298]. Each
`Node` carries `Name`, `Args`, `Children`, `File`, and `Line`, plus `Snippet`/`Macro`
flags that are "Always false for all nodes returned from Read because snippets are expanded
before it returns" [pkg/cfgparser/parse.go:L18-L43]. The configuration type used everywhere
else is just a type alias of the parser node — `type ( Node = parser.Node )`
[internal/config/config.go:L9-L11].

> **Key structural fact:** at parse time, `&something` is nothing more than a string sitting
> in `Node.Args`. No module is constructed, referenced, or initialized while parsing. This
> is the foundation that makes out-of-order references possible (see Q3).

**Step 2 — `moduleMain` drives startup.** `Run` opens and parses the config
[maddy.go:L150] and then calls `moduleMain(cfg)` [maddy.go:L157]. `moduleMain`
[maddy.go:L243] processes global directives (state/runtime dirs, hostname, `tls`, `log`,
`debug`, …) and then hands the unknown (non-global) blocks to `instancesFromConfig`
[maddy.go:L267].

**Step 3 — the two-pass bootstrap in `instancesFromConfig`** [maddy.go:L294-L375]:

- **Pass 1 — construct every top-level block** (the loop `for _, block := range nodes`)
  [maddy.go:L300-L346]. The instance name and any aliases are taken from `block.Args`, or
  the block name itself if there are no args [maddy.go:L303-L308]. Then each block goes down
  one of two paths:
  - **Endpoint path** [maddy.go:L312-L321]: if `module.GetEndpoint(modName)` returns a
    non-nil factory, the endpoint is constructed with `endpFactory(modName, block.Args)`,
    appended to a **separate `endpoints` slice**, and the loop `continue`s — **skipping
    `RegisterInstance` entirely**. Endpoints therefore never enter the instance registry and
    cannot be `&`-referenced.
  - **Regular module path** [maddy.go:L323-L345]: `module.Get(modName)` yields the
    constructor (nil → `"unknown module or global directive"` [maddy.go:L325]); a duplicate
    name is rejected via `module.HasInstance` [maddy.go:L328-L330]; the instance is built
    with `factory(modName, instName, modAliases, nil)` [maddy.go:L332]; it is then placed in
    the instance registry with `module.RegisterInstance(inst, config.NewMap(globals, &block))`
    [maddy.go:L338] and each alias is registered [maddy.go:L339-L344]. **`Init` is *not*
    called here.**
- **Reject configs with zero endpoints**: `if len(endpoints) == 0 { … "at least one
  endpoint should be configured" }` [maddy.go:L348-L350].
- **Pass 2 — eager endpoint initialization**: the bootstrap loops over the endpoint slice
  and calls `endp.instance.Init(config.NewMap(globals, &endp.cfg))` for each
  [maddy.go:L352-L356]. An endpoint's `Init` builds its message pipeline (e.g.
  `msgpipeline.New` [internal/endpoint/smtp/smtp.go:L580]), and pipeline construction is
  what resolves `&` references — through `ModuleFromNode` → `GetInstance` — cascading lazy
  initialization into every regular module the endpoint depends on.
- **The graph-settled guard**: after eager endpoint init, the bootstrap walks every regular
  instance and requires `module.Initialized[...] == true`; otherwise it aborts with
  `"Unused configuration block at %s:%d - %s (%s)"` [maddy.go:L358-L365].

The whole flow, documenting *existing* behavior (not a change to make):

```
init() of imported module packages              [maddy.go:L20-L38]
  └─> Register / RegisterEndpoint populate the
      modules & endpoints constructor maps        [registry.go:L7-L65]

parser.Read builds the full Node tree            [parse.go:L294-L298]
  └─> moduleMain ─> instancesFromConfig           [maddy.go:L243, L294]

      Pass 1: construct every top-level block      [maddy.go:L300-L346]
        • endpoints: GetEndpoint + factory,
          appended to slice, NO RegisterInstance   [maddy.go:L312-L321]
        • regular:   factory + RegisterInstance,
          constructed but NOT Init'd               [maddy.go:L323-L345]

      Reject if zero endpoints                     [maddy.go:L348-L350]

      Pass 2: eager endp.Init for every endpoint   [maddy.go:L352-L356]
        └─> endpoint Init builds pipeline,
            resolves & via GetInstance             [modconfig.go:L54-L93]
        └─> GetInstance lazily Init's regular
            modules, at most once, breaks cycles    [instances.go:L53-L75]

      Guard: every regular instance Initialized?
        else "Unused configuration block"          [maddy.go:L358-L365]

  └─> endpoints listening ─> the graph has settled
```

---

## Q1 — Startup assembly with mixed SMTP + IMAP endpoints

**Question.** How does Maddy's module system come together when the server starts,
particularly in configurations that mix SMTP and IMAP endpoints?

**Mechanism.** Startup runs through `moduleMain → instancesFromConfig` and its two passes
[maddy.go:L294-L375]: every top-level block is *constructed* in pass 1, and every endpoint
is *initialized* eagerly in pass 2. The decisive detail for mixed configs is that **SMTP and
IMAP are not special-cased relative to each other** — they register and are driven through
the *same* endpoint machinery:

- SMTP's `init()` registers three personalities — `smtp`, `submission`, and `lmtp` — all to
  the same constructor `New` [internal/endpoint/smtp/smtp.go:L715-L717].
- IMAP's `init()` registers `imap` to its own `New` [internal/endpoint/imap/imap.go:L221].
- Both `New` functions share the identical `FuncNewEndpoint` signature
  `func(modName string, addrs []string) (module.Module, error)`
  [internal/endpoint/smtp/smtp.go:L488; internal/endpoint/imap/imap.go:L44], and each
  endpoint type exposes an `Init` [internal/endpoint/smtp/smtp.go:L500;
  internal/endpoint/imap/imap.go:L53].

Consequently, when pass 1 reaches *any* endpoint block — `smtp`, `submission`, `lmtp`, or
`imap` — it takes the same branch: `module.GetEndpoint(modName)` returns a factory, the
endpoint is constructed and appended to the shared `endpoints` slice, and the loop
`continue`s [maddy.go:L312-L321]. In pass 2, every endpoint in that slice is initialized by
the same eager loop [maddy.go:L352-L356]. A mixed config is therefore assembled
*uniformly*: the bootstrap does not care whether a given endpoint speaks SMTP or IMAP.

The repository's own example config demonstrates exactly this coexistence in one file:
`smtp tcp://0.0.0.0:25` [maddy.conf:L53], `submission tls://0.0.0.0:465` [maddy.conf:L93],
and `imap tls://0.0.0.0:993` [maddy.conf:L149] all live side by side, and the IMAP block
references instances declared far earlier in the file — `auth &local_authdb` and
`storage &local_mailboxes` [maddy.conf:L149-L152] both point at the `sql local_mailboxes
local_authdb` block at [maddy.conf:L32].

**Why it is designed this way.** Treating every endpoint uniformly — one constructor
registry (`endpoints`), one lifecycle (construct-in-pass-1, eager-`Init`-in-pass-2) — keeps
the bootstrap simple and *personality-agnostic*. Adding a new endpoint kind (or a new SMTP
personality) requires no change to `instancesFromConfig`; it only requires another
`RegisterEndpoint` call in some package's `init()`. The mixing of SMTP and IMAP "just works"
because the assembler never branches on the protocol — it branches only on
*endpoint-vs-regular* (Q4).

---

## Q2 — Eager registration vs. deferred resolution

**Question.** When the server reads the config, what gets registered immediately versus what
stays unresolved until later?

**Mechanism — registered immediately (eager).** The *constructors* are the thing registered
immediately. They populate the `modules`/`endpoints` maps at package-`init()` time, which
happens at import — long before any config is read [internal/module/registry.go:L7-L65;
maddy.go:L20-L38]. After import, the binary already knows the full catalog of module *types*
it can build.

**Mechanism — deferred (unresolved until later).** When the config is parsed and pass 1
runs, each top-level *regular* block becomes an *instance* via `RegisterInstance`
[internal/module/instances.go:L23-L28; maddy.go:L338]. At that moment the instance object
exists (its constructor has run), but it is **left uninitialized** — `Init` has not been
called. It stays that way until something references it. That is "what stays unresolved
until later."

It is worth drawing the contrast precisely, because two different verbs are involved:

- **Construction** (the factory call) happens for *every* block in pass 1 — endpoints at
  [maddy.go:L314] and regular modules at [maddy.go:L332].
- **Initialization** (`Init`) is deferred: for *regular modules* it is driven lazily by
  `GetInstance` on first reference [internal/module/instances.go:L70]; for *endpoints* it
  happens eagerly in pass 2 [maddy.go:L352-L356].

So "registered immediately" = the module *types* in the constructor registry; "unresolved
until later" = the per-config *instances*, which are built in pass 1 but not initialized
until referenced (regular) or until the eager pass (endpoints).

**Why it is designed this way.** Separating "the set of available module *types*"
(known at compile/import time) from "the set of configured *instances*" (known only after
parsing) lets the binary be self-contained and statically aware of every capability it
ships, while keeping the actual wiring fully dynamic and driven by the operator's config.
The constructor registry is closed and fixed; the instance graph is open and per-deployment.

---

## Q3 — Lazy initialization and the `&` reference syntax (out-of-order references)

**Question.** How does lazy initialization work, and what happens during parsing that makes
the `&` reference syntax possible, so modules can reference each other even when declared out
of order?

**Mechanism — parsing makes order irrelevant.** Because the entire configuration tree is
parsed before any module work begins [pkg/cfgparser/parse.go:L294-L298], a token like
`&remote_queue` is, at parse time, *merely a string* in some node's `Args`. Nothing is
looked up or constructed during parsing, so the position of a block in the file carries no
weight: a block may be referenced before the line on which it is later defined.

**Mechanism — how `&` is resolved.** Resolution happens later, when a module's config is
processed and it asks for a dependency through `ModuleFromNode` [internal/config/module/modconfig.go:L54].
That function branches on whether the first argument starts with `&`
[internal/config/module/modconfig.go:L59]:

- **Reference path** (`&name`): it calls `module.GetInstance(args[0][1:])` and logs
  `"<file>:<line>: reference <&name>"` [internal/config/module/modconfig.go:L67-L68].
- **Inline path** (no `&`): it logs `"<file>:<line>: new module <name> <args>"` and builds a
  fresh module via `createInlineModule` [internal/config/module/modconfig.go:L70-L71].

**Mechanism — the lazy at-most-once engine.** `GetInstance` is where laziness lives
[internal/module/instances.go:L53-L75]. It first resolves any alias to the real instance name
[internal/module/instances.go:L54-L57]; if the name is unknown it returns
`"unknown config block: %s"` [internal/module/instances.go:L61]. Then comes the critical
ordering: it **sets `Initialized[name] = true` *before* calling `Init`**
[internal/module/instances.go:L69-L70], having already short-circuited with the existing
object if the instance was previously initialized [internal/module/instances.go:L64-L67].
Marking the instance initialized *before* running `Init` is what "breaks circular
dependencies": if A's `Init` references B and B's `Init` references A, the second visit to A
returns the (partially initialized) object instead of recursing forever. The net guarantee
is that each instance is initialized **at most once**, on first reference.

**Why it is designed this way (the canonical rationale).** The `Module` interface comment
states outright that `Init` is deliberately split out of the constructor:

> "It is not done in FuncNewModule so all module instances are registered at time of
> initialization, thus initialization does not depends on ordering of configuration blocks
> and modules can reference each other without any problems."
> [internal/module/module.go:L30-L35]

(The grammatical quirk "does not depends on ordering" is reproduced verbatim from the
source.) In other words, decoupling `Init` from construction is *precisely* the design choice
that makes order-independent, mutually-referencing config blocks safe. `HACKING.md` corroborates
the developer-facing view: modules referenced with `&` are looked up in the global instances
registry, and "Top-level defined module instances are initialized (Init method) lazily as they
are required by other modules." [HACKING.md:L54-L62].

**Worked example.** In the bundled config, the `submission` endpoint contains
`deliver_to &remote_queue` [maddy.conf:L111] — a forward reference to a `queue remote_queue`
block that is defined *later* in the same file [maddy.conf:L122]. This is the exact
out-of-order case the question asks about, and the success-path runtime log in the appendix
shows such a forward reference resolving cleanly.

---


## Q4 — Endpoint vs. regular module lifecycle divergence

**Question.** Where do endpoint modules' lifecycles diverge from regular modules, and why?

**Mechanism — divergence point 1: a separate registry.** Endpoints are not stored alongside
regular modules. They live in their own `endpoints` map, keyed to the `FuncNewEndpoint`
factory type, with their own registration and lookup functions
[internal/module/registry.go:L9; internal/module/registry.go:L45-L65]. A regular module is
fetched with `module.Get`; an endpoint with `module.GetEndpoint`.

**Mechanism — divergence point 2: never placed in the instance registry.** In pass 1, an
endpoint is constructed and appended to the local `endpoints` slice, then the loop
`continue`s *before* reaching `RegisterInstance` [maddy.go:L312-L321]. A regular module, by
contrast, is always registered as an instance [maddy.go:L338]. Because endpoints never enter
the instance registry, **they cannot be the target of an `&` reference** — there is nothing
for `GetInstance` to find.

**Mechanism — divergence point 3: eager vs. lazy `Init`.** Endpoints are initialized
*eagerly* in pass 2, every one of them, unconditionally [maddy.go:L352-L356]. Regular
modules are initialized *lazily*, only when first referenced, via `GetInstance`
[internal/module/instances.go:L53-L75].

**The contract that codifies the divergence.** The `FuncNewEndpoint` documentation spells out
how endpoints differ from regular modules:

> "Compared to regular modules, endpoint module instances are:
> - Not registered in the global registry.
> - Can't be defined inline.
> - Don't have an unique name
> - All config arguments are always passed as an 'addrs' slice and not used as names.
>
> As a consequence of having no per-instance name, InstanceName of the module object always
> returns the same value as Name."
> [internal/module/module.go:L59-L71]

That is why the endpoint factory's signature takes an `addrs []string` rather than the
`(modName, instName, aliases, inlineArgs)` shape of a regular `FuncNewModule`
[internal/module/module.go:L57; internal/module/module.go:L71].

**Why it is designed this way.** Endpoints are the **roots of the module graph** — they are
the listeners that accept connections, and *nothing references them*. If they were left to
the lazy path, nothing would ever trigger their `Init`, and the server would parse a config
and then do nothing. So endpoints must be initialized eagerly. And because *their* eager
`Init` is what builds the message pipeline — `msgpipeline.New`
[internal/endpoint/smtp/smtp.go:L580] — it is the endpoints that *cascade* resolution into
everything they reference (auth providers, storage, queues, targets), pulling the rest of the
graph up through `GetInstance`. `HACKING.md` summarizes this special path: "'smtp' and 'imap'
modules follow a special initialization path, so they are always initialized directly."
[HACKING.md:L60-L62].

---

## Q5 — Implicit check/modifier coordination (no explicit wiring)

**Question.** As messages flow through the system, how do `check` and `modify` modules
coordinate without explicit wiring in the config?

**Mechanism — composition is positional, not by name.** Checks and modifiers are not
`&`-referenced or named; they are *nested blocks* whose order in the config defines their
order in the pipeline. `parseMsgPipelineRootCfg` walks the pipeline's child nodes with a
`for _, node := range nodes { switch node.Name { … } }` and appends each parsed group in file
order [internal/msgpipeline/config.go:L25; internal/msgpipeline/config.go:L31-L32]. A `check`
block is parsed and its checks appended to `cfg.globalChecks`
[internal/msgpipeline/config.go:L33; internal/msgpipeline/config.go:L43]; a `modify` block is
parsed and its modifiers appended to `cfg.globalModifiers`
[internal/msgpipeline/config.go:L44; internal/msgpipeline/config.go:L54]; a `source` block
introduces per-source routing [internal/msgpipeline/config.go:L55]. The pipeline config struct
simply holds `globalChecks []module.Check` and `globalModifiers modify.Group`
[internal/msgpipeline/config.go:L17-L23].

**Mechanism — checks run in parallel, results merged under locks.** At message time, the
check runner executes the configured checks concurrently. `runAndMergeResults`
[internal/msgpipeline/check_runner.go:L142] launches a goroutine per check state
[internal/msgpipeline/check_runner.go:L161], merges each check's `AuthResult`/`Header`
contributions under `sync.Mutex` [internal/msgpipeline/check_runner.go:L144-L145], captures
the first reject/quarantine decision with `sync.Once`
[internal/msgpipeline/check_runner.go:L149; internal/msgpipeline/check_runner.go:L153], and
joins on a `sync.WaitGroup` [internal/msgpipeline/check_runner.go:L198]. `HACKING.md` makes
the parallelism explicit and notes the stable ordering of the *phases*: "Don't share any
state between messages, your code will be executed in parallel." and "You can assume that
order of check functions execution is as follows: CheckConnection, CheckSender, CheckRcpt,
CheckBody." [HACKING.md:L113-L118].

**Mechanism — modifiers run as an ordered serial chain.** Modifiers, unlike checks, are
strictly sequential. The `Group` type "wraps multiple modifiers and runs them serially" over
its `Modifiers []module.Modifier` slice [internal/modify/group.go:L11-L24], iterating them in
order [internal/modify/group.go:L24]. The `Modifier` interface even documents the guarantee:
"Calls on ModifierState are always strictly ordered." [internal/module/modifier.go:L22]. The
relevant interface contracts are `Check`/`CheckState` [internal/module/check.go:L14;
internal/module/check.go:L38], `Modifier`/`ModifierState`
[internal/module/modifier.go:L26; internal/module/modifier.go:L32], and
`DeliveryTarget`/`Delivery` [internal/module/delivery_target.go:L13;
internal/module/delivery_target.go:L22], all flowing through the two-level
source-then-recipient routing in the pipeline [internal/msgpipeline/msgpipeline.go:L34;
internal/msgpipeline/msgpipeline.go:L155].

**Why it is designed this way.** Coordination is **positional within the pipeline block**, so
no `&` wiring is necessary: a check's or modifier's place in message processing is determined
entirely by *where its block sits* in the config, not by a name another block must reference.
This is why the question's "without explicit wiring" holds — the wiring is implicit in block
nesting and order. Checks are parallel because they are read-only inspectors with no ordering
dependency between them (so the runner overlaps their latency and merges results safely under
locks), whereas modifiers are serial because each may depend on the header state left by the
previous one. The bundled config shows the pattern: the inbound `smtp` endpoint nests a
`check{ require_matching_ehlo; require_mx_record; verify_dkim; apply_spf }` block
[maddy.conf:L54-L66], and `submission` nests a `modify{ sign_dkim … }` block
[maddy.conf:L98-L100] — neither names nor references the other.

---

## Q6 — Runtime evidence that the graph settled

**Question.** What runtime evidence shows the module graph has fully settled into place?

There are two complementary kinds of evidence — *success signals* that resolution happened
and the listeners came up, and an *active guard* that fails startup if any block was left
dangling.

**Success signals.**

- **Reference-resolution debug lines.** Every `&`-reference and every inline construction is
  logged by `ModuleFromNode`: `"<file>:<line>: reference <&name>"` on the reference path
  [internal/config/module/modconfig.go:L68] and `"<file>:<line>: new module <name> <args>"`
  on the inline path [internal/config/module/modconfig.go:L70]. Seeing these lines means the
  pipeline actually pulled its dependencies out of (or built them into) the graph.
- **Endpoints listening.** Once an endpoint's eager `Init` finishes binding, it logs
  `listening on …` [internal/endpoint/smtp/smtp.go:L619]. When every configured endpoint has
  logged this, the roots of the graph are live — which, since endpoint `Init` is what
  cascades resolution, means everything they reference initialized successfully too.

**The active guard.** After the eager endpoint-init pass, `instancesFromConfig` verifies that
*every* regular instance was reached — i.e. that `module.Initialized[...]` is true for each.
If any top-level regular block was never referenced, startup is aborted with the formatted
error `"Unused configuration block at %s:%d - %s (%s)"` [maddy.go:L358-L365]. The format
maps to `file:line - InstanceName (Name)`. This is the decisive runtime proof: the server
*refuses to run* unless the graph is fully connected — there can be no silently dead,
unreferenced module instance.

The captured logs in the appendix below show both: the success path (references resolving,
including an out-of-order one, and both endpoints listening) and the failure path (the
unused-block guard aborting startup with a non-zero exit code).

---


## Runtime Evidence (Appendix)

The logs below are **genuine captured output** from a throwaway observation run, included
to back Q6 with real behavior rather than assertion. They were produced as follows, and the
temporary files have since been deleted — nothing was ever written into the Maddy source
tree:

- The server was built to a temp path with CGO disabled:
  `CGO_ENABLED=0 go build -o /tmp/maddy ./cmd/maddy`, using **go1.23.12** (the repository's
  declared minimum is `go 1.13` [go.mod:L3]).
- Because CGO was disabled, the SQLite-backed `sql` storage module was not exercised; the
  CGO-free `dummy` module — which "implements AuthProvider and DeliveryTarget interfaces but
  does nothing" [internal/module/dummy.go:L11-L12] and is registered as `dummy`
  [internal/module/dummy.go:L57] — served as the auth and delivery target.
- `tls off` [internal/config/tls_server.go:L22] avoided certificate setup, and all temp
  configs, state, and runtime directories lived under `/tmp`.
- Since SMTP/submission/LMTP and IMAP share one `RegisterEndpoint` + eager-`Init` code path
  [internal/endpoint/smtp/smtp.go:L715-L717; internal/endpoint/imap/imap.go:L221], this
  SMTP/submission evidence generalizes directly to IMAP.

### Success path — mixed endpoints with an out-of-order reference

The throwaway config declared two endpoints (`smtp` and `submission`), a nested `check{}`
containing `require_mx_record`, and `deliver_to &remote_target` — where the `dummy
remote_target` and `dummy local_authdb` blocks were declared **after** the endpoints that
reference them (the out-of-order case). Run with `maddy -debug -log stderr`:

```
[debug] /tmp/.../success.conf:10: new module require_mx_record []
[debug] /tmp/.../success.conf:12: reference &remote_target
smtp: listening on tcp://127.0.0.1:10025
[debug] /tmp/.../success.conf:18: reference &local_authdb
[debug] /tmp/.../success.conf:20: reference &remote_target
[debug] submission: authentication provider: dummy local_authdb
submission: listening on tcp://127.0.0.1:10587
signal received (terminated), next signal will force immediate shutdown.
```

Mapping each line to verified source:

| Log line | What it proves | Source |
|---|---|---|
| `new module require_mx_record []` | inline construction of a check with no args (the *inline* path) | [internal/config/module/modconfig.go:L70] |
| `reference &remote_target` (config line 12) | an `&`-reference resolving **even though `remote_target` is declared later in the file** — out-of-order forward reference (Q3) | [internal/config/module/modconfig.go:L68] |
| `reference &local_authdb` (config line 18) | resolution of an auth provider also declared after the endpoint | [internal/config/module/modconfig.go:L68] |
| `submission: authentication provider: dummy local_authdb` | the resolved auth provider attached to the endpoint | [internal/endpoint/smtp/smtp.go:L510] |
| `smtp: listening on …` and `submission: listening on …` | **both** endpoints reached their listeners → the graph settled for a mixed-endpoint config (Q1) | [internal/endpoint/smtp/smtp.go:L619] |

The two `listening on` lines, taken together, are the success evidence that a config mixing
two endpoint kinds assembled completely and the forward references resolved.

### Failure path — the unused-block guard

The second throwaway config added a top-level `dummy orphan_block` that **nothing
references**, alongside a valid `smtp` endpoint (so the "at least one endpoint should be
configured" check [maddy.go:L348-L350] passes and execution actually reaches the guard).
Run the same way, `maddy` exited with code **2**:

```
[debug] /tmp/.../unused_block.conf:12: reference &inline_target
smtp: listening on tcp://127.0.0.1:10026
Unused configuration block at /tmp/.../unused_block.conf:7 - orphan_block (dummy)
```

Mapping to source:

| Log line | What it proves | Source |
|---|---|---|
| `reference &inline_target` | the endpoint's referenced target resolved normally | [internal/config/module/modconfig.go:L68] |
| `smtp: listening on …` | the endpoint bound — so the failure is *not* an endpoint error | [internal/endpoint/smtp/smtp.go:L619] |
| `Unused configuration block at …:7 - orphan_block (dummy)` | the guard tripped because `module.Initialized["orphan_block"]` was false; the message is the exact format string `"Unused configuration block at %s:%d - %s (%s)"` rendered as `file:line - InstanceName (Name)` | [maddy.go:L358-L365] |

The non-zero exit reflects `Run` returning `2` whenever `moduleMain` reports an error
[maddy.go:L157-L161]. This is the *active* runtime proof for Q6: Maddy will not start while
any top-level regular block remains unreferenced — the module graph must be fully connected,
or the process aborts.

---

## Summary

- Maddy keeps **two registries**: a *constructor* registry filled eagerly at import time
  [internal/module/registry.go:L7-L65] and an *instance* registry filled at parse time and
  initialized lazily [internal/module/instances.go:L9-L75].
- Startup parses the whole config first [pkg/cfgparser/parse.go:L294-L298], then runs a
  **two-pass bootstrap**: construct every block (pass 1), eagerly `Init` every endpoint
  (pass 2) [maddy.go:L294-L375].
- **Endpoints diverge** from regular modules: separate registry, no instance registration,
  eager init — because they are the graph roots and nothing would otherwise initialize them
  [internal/module/module.go:L59-L71; maddy.go:L312-L321; maddy.go:L352-L356].
- **Lazy, at-most-once `Init`** plus a *parse-everything-first* model make `&` references and
  out-of-order declarations safe — exactly as the `Module.Init` comment intends
  [internal/module/module.go:L30-L35; internal/module/instances.go:L53-L75].
- **Checks and modifiers** coordinate *positionally* inside the pipeline block — parallel
  checks merged under locks, serial modifiers — needing no `&` wiring
  [internal/msgpipeline/config.go:L17-L102; internal/msgpipeline/check_runner.go:L142-L198;
  internal/modify/group.go:L11-L24].
- The graph is proven settled by **reference-resolution logs plus listeners coming up**, and
  guaranteed by the **unused-block guard** that hard-fails startup
  [maddy.go:L358-L365].

