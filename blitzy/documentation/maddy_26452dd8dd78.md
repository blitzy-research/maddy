# How Maddy's Module System Assembles Itself — Startup and Message Flow

> An evidence-grounded answer to a developer onboarding question about the
> [Maddy](https://github.com/foxcpp/maddy) mail server, **scoped to configurations
> that mix SMTP and IMAP endpoints**.

## Introduction

This document answers a multi-part question about how Maddy's *module system*
assembles itself — first when the server reads its configuration at startup, and
then as messages flow through the pipeline. Every claim below is grounded in the
source at **HEAD `26452dd`** (full commit `26452dd8dd787dc455278b0fdd296f4a5432c768`,
*"target/remote: Rewrite connection part to allow more concurrency"*) and in
**real output captured by building and running the server** — not from reading alone.

In one sentence, the lifecycle is: **register-on-init → lazy-init → eager-endpoint →
settle-guard.** Constructors register themselves into global maps at package-init
(before any config is read); regular top-level blocks are turned into *instances* but
initialized only *lazily* on first reference; endpoints are initialized *eagerly and
directly*; and a final *settle guard* aborts startup if any configured block was never
used.

### ⚠️ Version-fidelity caveat (read this first)

This commit predates Maddy's newer *namespaced* configuration syntax. Every literal in
this document is taken from **this** commit, which uses the **old non-namespaced module
names**: `sql`, `smtp`, `submission`, `lmtp`, `imap`, `queue`, `remote`. The default
config confirms this — for example `sql local_mailboxes local_authdb` and
`smtp tcp://...` blocks in `maddy.conf`. This document deliberately uses **only** those
old names; it does **not** use the newer upstream *namespaced* configuration syntax (the
dotted module names and dedicated configuration blocks introduced in later Maddy releases),
because that syntax does not exist at HEAD `26452dd`. If you compare against the live
upstream website, expect this version skew.

### The exact question, decomposed into five sub-parts

The five sub-questions are reproduced verbatim and answered one-by-one, followed by a
coverage-pass checklist:

- **Q1 — Immediate vs. deferred:** When the server reads the config, what gets registered
  *immediately* versus what stays *unresolved* until later?
- **Q2 — Lazy initialization and ampersand (`&`) syntax:** How does lazy initialization
  work — what happens during *parsing* that lets modules reference each other via the `&`
  syntax even when declared out of order?
- **Q3 — Endpoint divergence:** Where do *endpoint* modules' lifecycles diverge from
  *regular* modules, and why?
- **Q4 — Check/modifier coordination:** As messages flow, how do *check* and *modifier*
  modules coordinate without explicit wiring in the config?
- **Q5 — Settle evidence:** What runtime evidence shows the module graph has fully
  *settled* into place?

---

## How the investigation was run (evidence provenance)

The rule for this task is **run-first**: build and run the relevant code paths and capture
the real output *before* writing. All build artifacts and observation configs live **outside
the repository working tree**, so the repository is left unchanged (`git status --porcelain`
empty at the end).

### Toolchain

The verbatim evidence quoted throughout this document was captured in **this environment**
with **Go 1.18.10** (`go version go1.18.10 linux/amd64`), which satisfies the module's declared
floor `go 1.13` [`go.mod:L3`]. **CGO was enabled** (`CGO_ENABLED=1`) with a C compiler present,
because the default `sql`/SQLite3 storage backend is gated behind a build constraint (see the
Reproducibility note). Every log line and exit code quoted below was captured from this build;
the Go point release affects only the `go version` string itself, not the observed runtime
output, which derives from the source at HEAD `26452dd`.

### Build

```
$ go version
go version go1.18.10 linux/amd64         # satisfies go.mod floor "go 1.13" (go.mod:L3)
$ CGO_ENABLED=1 GOPATH=/tmp/gopath GOCACHE=/tmp/gocache \
    go build -o /tmp/obs/maddy ./cmd/maddy      # exit 0; ~19 MB binary (sqlite3 compiled via CGO)
```

> **Why `./cmd/maddy` and not `.`?** The repository root is `package maddy` — a *library*
> (`maddy.go:L1`). The real executable is `cmd/maddy/main.go`, whose `main()` is just
> `os.Exit(maddy.Run())` [`cmd/maddy/main.go:L10`]. Build the `./cmd/maddy` sub-package.

### Observation config #1 — VALID mixed SMTP+IMAP (`/tmp/obs/valid.conf`, outside the repo)

The `sql` block is declared **last** (lines 28–31) even though the endpoints above it reference
it — this deliberately demonstrates out-of-order `&`-resolution. Endpoints use `tcp://` with a
global `tls off` and a `hostname` so no TLS certificate is needed (`tls://` would otherwise
require TLS config — `internal/endpoint/smtp/smtp.go:L623` / `internal/endpoint/imap/imap.go:L134`):

```
 5  state /tmp/obs/state
 6  runtime /tmp/obs/runtime
 8  hostname mx.observe.test
 9  tls off
11  smtp tcp://127.0.0.1:2525 {
12      deliver_to &local_mailboxes
13  }
15  submission tcp://127.0.0.1:2587 {
16      insecure_auth
17      auth &local_authdb
18      deliver_to &local_mailboxes
19  }
21  imap tcp://127.0.0.1:2143 {
22      insecure_auth
23      auth &local_authdb
24      storage &local_mailboxes
25  }
28  sql local_mailboxes local_authdb {
29      driver sqlite3
30      dsn all.db
31  }
```

Command, exit code, and the captured `stderr` shown **verbatim** via `cat -A`. Every maddy log
line ends with a literal trailing **TAB** — the field separator emitted by `formatMsg`
(`formatted.WriteRune('\t')` [`internal/log/log.go:L139`]) immediately before the newline written
at [`internal/log/writer.go:L25`]. In the `cat -A` renderings below, that trailing TAB shows as
`^I` and the line-ending newline as `$`, so every byte is accounted for. **This `^I`/`$` convention
is used for every captured-log block throughout this document.**

```
$ timeout -s TERM 8 /tmp/obs/maddy -debug -config /tmp/obs/valid.conf 2>/tmp/obs/valid.stderr ; echo "EXIT=$?"
EXIT=124
$ cat -A /tmp/obs/valid.stderr
[debug] sql: go-imap-sql version 0.4.0^I$
[debug] /tmp/obs/valid.conf:12: reference &local_mailboxes^I$
smtp: listening on tcp://127.0.0.1:2525^I$
[debug] /tmp/obs/valid.conf:17: reference &local_authdb^I$
[debug] /tmp/obs/valid.conf:18: reference &local_mailboxes^I$
[debug] submission: authentication provider: sql local_mailboxes^I$
submission: listening on tcp://127.0.0.1:2587^I$
[debug] /tmp/obs/valid.conf:23: reference &local_authdb^I$
[debug] /tmp/obs/valid.conf:24: reference &local_mailboxes^I$
imap: listening on tcp://127.0.0.1:2143^I$
imap: authentication over unencrypted connections is allowed, this is insecure configuration and should be used only for testing!^I$
imap: TLS is disabled, this is insecure configuration and should be used only for testing!^I$
signal received (terminated), next signal will force immediate shutdown.^I$
```

### Observation config #2 — ORPHANED block (`/tmp/obs/orphan.conf`, outside the repo)

The **same endpoint set as the valid config** — `smtp`, `submission`, and `imap` — plus the
referenced `sql local_mailboxes local_authdb` block, **plus** an unreferenced `sql orphan_storage`
block (header at line 34) that no `&` points to anywhere:

```
 5  state /tmp/obs/state
 6  runtime /tmp/obs/runtime
 8  hostname mx.observe.test
 9  tls off
11  smtp tcp://127.0.0.1:2525 {
12      deliver_to &local_mailboxes
13  }
15  submission tcp://127.0.0.1:2587 {
16      insecure_auth
17      auth &local_authdb
18      deliver_to &local_mailboxes
19  }
21  imap tcp://127.0.0.1:2143 {
22      insecure_auth
23      auth &local_authdb
24      storage &local_mailboxes
25  }
28  sql local_mailboxes local_authdb {
29      driver sqlite3
30      dsn all.db
31  }
34  sql orphan_storage {
35      driver sqlite3
36      dsn orphan.db
37  }
```

Command, exit code, and captured `stderr` shown **verbatim** via `cat -A` (same `^I`/`$` convention:
trailing `^I` is the field-separator TAB, `$` is the newline):

```
$ timeout -s TERM 8 /tmp/obs/maddy -debug -config /tmp/obs/orphan.conf 2>/tmp/obs/orphan.stderr ; echo "EXIT=$?"
EXIT=2
$ cat -A /tmp/obs/orphan.stderr
[debug] sql: go-imap-sql version 0.4.0^I$
[debug] /tmp/obs/orphan.conf:12: reference &local_mailboxes^I$
smtp: listening on tcp://127.0.0.1:2525^I$
[debug] /tmp/obs/orphan.conf:17: reference &local_authdb^I$
[debug] /tmp/obs/orphan.conf:18: reference &local_mailboxes^I$
[debug] submission: authentication provider: sql local_mailboxes^I$
submission: listening on tcp://127.0.0.1:2587^I$
[debug] /tmp/obs/orphan.conf:23: reference &local_authdb^I$
[debug] /tmp/obs/orphan.conf:24: reference &local_mailboxes^I$
imap: listening on tcp://127.0.0.1:2143^I$
imap: authentication over unencrypted connections is allowed, this is insecure configuration and should be used only for testing!^I$
imap: TLS is disabled, this is insecure configuration and should be used only for testing!^I$
Unused configuration block at /tmp/obs/orphan.conf:34 - orphan_storage (sql)^I$
```

These two runs — a clean settle (`EXIT=124`) and an aborted settle (`EXIT=2`) — are the
backbone of the evidence below. Both were captured in this environment on Go 1.18.10.

---

## Q1 — Immediate vs. deferred

> *When the server reads the config, what gets registered immediately versus what stays
> unresolved until later?*

There are **three distinct moments**. Two of them happen "immediately" (at different times),
and two things are explicitly *deferred*.

### (a) Immediate — at package init, before any config is read

The very first thing that happens — before `Run()` even parses a config file — is
**constructor registration**. The main package has a block of **17 blank/side-effect imports**
whose only purpose is to trigger each imported package's `init()`:

```go
// maddy.go:L20  — "// Import packages for side-effect of module registration."
// maddy.go:L21-L37  — the 17 `_ "..."` imports; maddy.go:L38 — the closing ')'
_ "github.com/foxcpp/maddy/internal/auth/external"
_ "github.com/foxcpp/maddy/internal/auth/pam"
_ "github.com/foxcpp/maddy/internal/auth/shadow"
_ "github.com/foxcpp/maddy/internal/check/command"
_ "github.com/foxcpp/maddy/internal/check/dkim"
_ "github.com/foxcpp/maddy/internal/check/dns"
_ "github.com/foxcpp/maddy/internal/check/dnsbl"
_ "github.com/foxcpp/maddy/internal/check/requiretls"
_ "github.com/foxcpp/maddy/internal/check/spf"
_ "github.com/foxcpp/maddy/internal/endpoint/imap"
_ "github.com/foxcpp/maddy/internal/endpoint/smtp"
_ "github.com/foxcpp/maddy/internal/modify"
_ "github.com/foxcpp/maddy/internal/modify/dkim"
_ "github.com/foxcpp/maddy/internal/storage/sql"
_ "github.com/foxcpp/maddy/internal/target/queue"
_ "github.com/foxcpp/maddy/internal/target/remote"
_ "github.com/foxcpp/maddy/internal/target/smtp_downstream"
```

Each package's `init()` calls `module.Register(name, constructor)` (regular modules) or
`module.RegisterEndpoint(name, constructor)` (endpoints), for example:

- `module.Register("sql", New)` [`internal/storage/sql/sql.go:L424`]
- `module.RegisterEndpoint("smtp", New)`, `("submission", New)`, `("lmtp", New)`
  [`internal/endpoint/smtp/smtp.go:L715-L717`]
- `module.RegisterEndpoint("imap", New)` [`internal/endpoint/imap/imap.go:L221`]

These populate **two global maps** guarded by a single lock:

```go
// internal/module/registry.go:L7-L11
var (
	modules     = make(map[string]FuncNewModule)
	endpoints   = make(map[string]FuncNewEndpoint)
	modulesLock sync.RWMutex
)
```

Duplicate names are a hard error: `Register` panics with
`"Register: module with specified name is already registered: " + name`
[`internal/module/registry.go:L24`], and `RegisterEndpoint` has the identical panic string
[`internal/module/registry.go:L61`]. This is exactly Go's canonical *register-on-init*
pattern (the same one `database/sql` uses for drivers), which is why duplicate registration is
disallowed. **What is registered immediately, then, is constructors — keyed by name — not
instances.**

### (b) Immediate — while reading config (Loop 1): instances are *created and registered but not initialized*

`Run()` [`maddy.go:L102`] parses the config and calls `moduleMain(cfg)` [`maddy.go:L157`],
which in turn calls `instancesFromConfig(globals.Values, unknown)` [`maddy.go:L267`].
`instancesFromConfig` [`maddy.go:L294`] runs **three sequential loops**. Loop 1
[`maddy.go:L300-L346`] walks every top-level block and creates an **instance object** for it —
but crucially does **not** initialize it:

- For a **regular** module block, the instance is put into the instances registry via
  `module.RegisterInstance(inst, config.NewMap(globals, &block))` [`maddy.go:L338`] (and any
  aliases via `module.RegisterAlias(alias, instName)` [`maddy.go:L343`]). **`Init` is not
  called here.**
- For an **endpoint** block (detected by `endpFactory := module.GetEndpoint(modName)` at
  [`maddy.go:L312`]), the instance is created, appended to a queue via
  `endpoints = append(endpoints, modInfo{instance: inst, cfg: block})` [`maddy.go:L319`],
  and then the loop does `continue` [`maddy.go:L320`] — so an endpoint is **not even
  registered** in the instances registry (see Q3).

### (c) Deferred / unresolved until later

Two things remain unresolved after the config is read:

1. **Actual `Init()` of regular modules.** A regular module is initialized *lazily*, only when
   something first references it — see Q2. Nothing in Loop 1 calls its `Init`.
2. **All `&name` references.** At this point `&local_mailboxes` and `&local_authdb` are still
   just plain string tokens sitting in a `Node.Args` slice (see Q2). They are not resolved to
   module instances until an endpoint's initialization walks its config.

### Observed evidence + rationale

The valid-run log is the proof. The **first** side-effect of *initializing* the `sql` module is
its version line, emitted from within `sql`'s `Init` at
`store.Log.Debugln("go-imap-sql version", imapsql.VersionStr)` [`internal/storage/sql/sql.go:L268`]
(same `cat -A` convention — trailing `^I` = TAB, `$` = newline):

```
[debug] sql: go-imap-sql version 0.4.0^I$
[debug] /tmp/obs/valid.conf:12: reference &local_mailboxes^I$
```

That `go-imap-sql version 0.4.0` line does **not** appear at "config read" time — it appears
only when the *smtp* endpoint (config line 12) first resolves `&local_mailboxes`. That is the
observable signature of deferral: the `sql` block was created in Loop 1, but its `Init` did not
run until the endpoint referenced it. **Rationale:** separating "register the constructor" from
"create the instance" from "initialize the instance" is precisely what makes configuration
ordering irrelevant — the `Init` doc says so directly:
*"...so all module instances are registered at time of initialization, thus initialization does
not depends on ordering of configuration blocks and modules can reference each other without any
problems."* [`internal/module/module.go:L30-L38`].

---

## Q2 — Lazy initialization and the `&` (ampersand) syntax

> *How does lazy initialization work — what happens during parsing that lets modules reference
> each other via the `&` syntax even when declared out of order?*

### What happens during *parsing*: `&name` is just a string token

The config parser builds a tree of `Node` values. A `Node` stores the directive name, its
arguments, and its children — nothing about references:

```go
// pkg/cfgparser/parse.go:L18-L43 (selected fields)
type Node struct {
	Name     string   // L20
	Args     []string // L22
	Children []Node   // L25
	Snippet  bool     // L30
	File     string   // L38
	Line     int      // L42
}
```

When the parser reads a directive's arguments it appends **each argument verbatim** — including
a token like `&local_mailboxes` — as a plain string:

```go
// pkg/cfgparser/parse.go:L97
node.Args = append(node.Args, ctx.Val())
```

**The parser never interprets the leading `&`.** To the parser, `&local_mailboxes` is an ordinary
string in `Node.Args`. This is exactly *why* declaration order does not matter: parsing produces
no cross-block links at all, so a block can reference another block that is textually defined
later (as in the valid config, where `sql` is declared last at lines 28–31 but referenced first
at line 12).

> **Don't confuse `&` with `import`/snippets.** There *is* a parse-time mechanism that stitches
> text together — snippet expansion via the `import` directive
> (`expandImports` [`pkg/cfgparser/imports.go:L10`], `resolveImport` [`pkg/cfgparser/imports.go:L60`],
> error `"unknown import: "+name` [`pkg/cfgparser/imports.go:L72`]). But that is a **textual**
> expansion done during parsing — a snippet `(name){...}` referenced via `import` is copied
> inline (the user manual demonstrates this in *"Snippets & imports"*,
> `docs/man/maddy-config.5.scd:L86-L142`). The `&` syntax is a completely different thing: a
> **module-instance reference** that is resolved much later, at initialization time.

### Where `&` is resolved: `ModuleFromNode`

Resolution happens in `ModuleFromNode` [`internal/config/module/modconfig.go:L54`], which detects
the prefix and branches:

```go
// internal/config/module/modconfig.go
referenceExisting := strings.HasPrefix(args[0], "&")   // L59
...
modObj, err = module.GetInstance(args[0][1:])          // L67  (strips the leading '&')
log.Debugf("%s:%d: reference %s", inlineCfg.File, inlineCfg.Line, args[0])  // L68
```

Note the **ordering**: `GetInstance` (L67) runs **before** the `reference %s` debug line (L68).
So if resolving a reference triggers a module's lazy `Init`, that module's *own* init logs print
**before** the `reference &name` line. (The non-reference path instead constructs an inline module:
`log.Debugf("%s:%d: new module %s %v", ...)` [`internal/config/module/modconfig.go:L70`], `createInlineModule(args[0], args[1:])` [`internal/config/module/modconfig.go:L71`],
and inline modules are initialized in place via `initInlineModule` under
`if !referenceExisting { ... }` [`internal/config/module/modconfig.go:L86-L90`].)

### Lazy, at-most-once initialization with cycle-breaking

`GetInstance` [`internal/module/instances.go:L53-L75`] is where lazy init lives:

```go
// internal/module/instances.go
// (unknown name) -> fmt.Errorf("unknown config block: %s", name)   // L61
// Break circular dependencies.                                     // L64
if Initialized[name] {                                              // L65
	return mod.mod, nil                                             // L66-L67 (short-circuit)
}
Initialized[name] = true                                            // L69  (set BEFORE Init)
if err := mod.mod.Init(mod.cfg); err != nil {                       // L70
	...
}
```

The shared `Initialized` map is declared at `Initialized = make(map[string]bool)`
[`internal/module/instances.go:L16`]. Two properties matter:

1. **At-most-once:** if `Initialized[name]` is already `true`, `GetInstance` returns the existing
   instance without re-initializing (L65–L67).
2. **Cycle-breaking:** `Initialized[name] = true` is set at **L69, before** `Init` runs at **L70**.
   So if module A's `Init` references A (directly or transitively), the second lookup short-circuits
   instead of recursing forever.

### Observed evidence + rationale

The valid run demonstrates both order-independence and at-most-once init:

```
[debug] sql: go-imap-sql version 0.4.0^I$
[debug] /tmp/obs/valid.conf:12: reference &local_mailboxes^I$
...
[debug] /tmp/obs/valid.conf:17: reference &local_authdb^I$
[debug] /tmp/obs/valid.conf:18: reference &local_mailboxes^I$
...
[debug] /tmp/obs/valid.conf:23: reference &local_authdb^I$
[debug] /tmp/obs/valid.conf:24: reference &local_mailboxes^I$
```

- **Out-of-order works:** `sql local_mailboxes local_authdb` is declared **last** (config lines
  28–31), yet it is resolved at **line 12** (the first `&local_mailboxes` in the `smtp` block). The
  parser stored `&local_mailboxes` as a bare token, so the later-declared block is found at init
  time regardless.
- **At-most-once:** `&local_mailboxes`/`&local_authdb` are referenced **five** times (lines
  12, 17, 18, 23, 24), but `sql: go-imap-sql version 0.4.0` prints **exactly once** — at the very
  first resolution (line 12). This is `Initialized[name]` short-circuiting on every subsequent
  reference (`internal/module/instances.go:L65`), after the first pass set it `true` (`internal/module/instances.go:L69`).
- **Ordering of logs:** the `go-imap-sql version 0.4.0` line appears **before** the
  `reference &local_mailboxes` line, exactly because `GetInstance` (`internal/config/module/modconfig.go:L67`) runs before
  the `reference` debug line (`internal/config/module/modconfig.go:L68`).

**Rationale:** lazy resolution keeps parsing dumb (no link resolution) and defers the expensive
work (`Init`) to first use, deduplicated via `Initialized`. The authors describe this design
directly: *"if configuration uses &-syntax to reference existing configuration block, ModuleFromNode
simply looks it up in the global instances registry. All modules defined the configuration as a
separate top-level blocks are created before main initialization..."* and *"Top-level defined module
instances are initialized (Init method) lazily as they are required by other modules."*
[`HACKING.md:L54-L62`].

---

## Q3 — Endpoint lifecycle divergence

> *Where do endpoint modules' lifecycles diverge from regular modules, and why?*

### The contract: `FuncNewEndpoint`

Endpoints are a deliberately different kind of module. The contract is documented on the endpoint
constructor type:

```go
// internal/module/module.go:L59-L71
// FuncNewEndpoint is a function that creates new instance of endpoint module.
//
// Compared to regular modules, endpoint module instances are:
// - Not registered in the global registry.            (L63)
// - Can't be defined inline.                           (L64)
// - Don't have an unique name                          (L65)
// - All config arguments are always passed as an 'addrs' slice and not used as
//   names.                                             (L66-L67)
//
// As a consequence of having no per-instance name, InstanceName of the module
// object always returns the same value as Name.        (L69-L70)
type FuncNewEndpoint func(modName string, addrs []string) (Module, error)  // L71
```

Contrast this with the regular constructor
`type FuncNewModule func(modName, instName string, aliases, inlineArgs []string) (Module, error)`
[`internal/module/module.go:L57`], which *does* carry an instance name and inline args.

### Divergence #1 — Loop 1 skips registration

As noted in Q1, when Loop 1 hits an endpoint block it creates the instance, appends it to the
`endpoints` queue [`maddy.go:L319`], and immediately `continue`s [`maddy.go:L320`]. It therefore
**skips** the `module.RegisterInstance(...)` call [`maddy.go:L338`] that regular modules go through.
So an endpoint never enters the instances registry and can never be the target of an `&` reference.

### Divergence #2 — Loop 2 initializes endpoints *eagerly and directly*

After a guard that requires at least one endpoint —
`if len(endpoints) == 0 { return nil, fmt.Errorf("at least one endpoint should be configured") }`
[`maddy.go:L348-L350`] — Loop 2 initializes each queued endpoint **directly**:

```go
// maddy.go:L352-L356
for _, endp := range endpoints {
	if err := endp.instance.Init(config.NewMap(globals, &endp.cfg)); err != nil {
		return nil, err
	}
}
```

This is the opposite of regular modules, which are initialized *lazily* (Q2). The authors state it
plainly: *"'smtp' and 'imap' modules follow a special initialization path, so they are always
initialized directly."* [`HACKING.md:L60-L62`].

### Why

Endpoints are the **roots of the dependency graph**. Nothing references an endpoint (they are not in
the instances registry, and `&endpoint` is meaningless), so a purely lazy scheme would *never*
initialize them — no reference would ever trigger them. They must be initialized eagerly. And it is
each endpoint's `Init` — opening listeners and reading its own `deliver_to`/`auth`/`storage`
directives — that *drives* the `&` resolutions of the regular modules underneath it (which is how the
lazy `sql` init in Q1/Q2 gets triggered in the first place).

### Observed evidence + rationale

Each endpoint's `Init` opens its listener and logs it via `endp.Log.Printf("listening on %v", addr)`
(`internal/endpoint/smtp/smtp.go:L619`, `internal/endpoint/imap/imap.go:L130`). From the valid run
(`cat -A`; trailing `^I` = TAB, `$` = newline):

```
smtp: listening on tcp://127.0.0.1:2525^I$
submission: listening on tcp://127.0.0.1:2587^I$
imap: listening on tcp://127.0.0.1:2143^I$
```

The three endpoints initialize in **config order** (`smtp` → `submission` → `imap`), all inside
Loop 2's eager loop [`maddy.go:L352-L356`], binding the exact addresses from the config
(`tcp://127.0.0.1:2525`, `tcp://127.0.0.1:2587`, `tcp://127.0.0.1:2143`). Interleaved with them are
the `reference &...` lines — direct evidence that endpoint `Init` is what drives the downstream `&`
resolutions. The `submission` endpoint also logs its resolved auth provider,
`[debug] submission: authentication provider: sql local_mailboxes`
(`internal/endpoint/smtp/smtp.go:L510`), confirming the endpoint wired itself to the lazily-initialized
`sql` instance named `local_mailboxes`. **Rationale:** eager, direct endpoint init is required
precisely *because* endpoints are unreferenced roots that must actively open listeners; everything
below them is pulled in lazily on demand.

---

## Q4 — Check / modifier coordination (no explicit wiring)

> *As messages flow, how do check and modifier modules coordinate without explicit wiring in the
> config?*

> **⚠️ Verifiability caveat (stated up front).** The message-flow behaviors below (parallel check
> execution, phase replay, the modifier chain order, two-level routing) are grounded in **code reading
> with exact `file:line` citations**, *not* in captured message-delivery logs. The observation runs in
> this investigation exercised **startup only** (no test message was delivered), so no runtime
> message-flow log line is quoted for Q4. Where the answer is code-only, it is labeled as such rather
> than asserted from output that was not captured.

### There is no cross-wiring because a pipeline *owns* its checks and modifiers

Checks and modifiers are not top-level blocks that reference each other. They are declared **inside**
an endpoint's message-pipeline `check { }` / `modify { }` blocks (visible in the default `maddy.conf`,
e.g. a `check { ... }` and `modify { ... }` nested within the `smtp`/`submission` blocks). The
`MsgPipeline` (`internal/msgpipeline/msgpipeline.go`) **owns** those checks and modifiers for its
block and runs them itself. There is therefore no config directive that "connects" a specific check to
a specific modifier — coordination is *structural* (by which block they live in), not by explicit
reference. This is the categorical difference from the `&` wiring of Q2: checks/modifiers are composed
positionally inside a pipeline, not looked up by name.

The subsystem layout confirms these are distinct module families:
*`check/` — modules — message checkers (module.Check)* and
*`modify/` — modules — message modifiers (module.Modifier)* [`internal/README.md:L8-L24`].

### Checks: parallel goroutines, merged under locks, with phase replay

`runAndMergeResults` [`internal/msgpipeline/check_runner.go:L142-L209`] runs each check's state on its
**own goroutine**, coordinated by a `sync.WaitGroup`:

```go
// internal/msgpipeline/check_runner.go (selected)
authResLock      sync.Mutex   // L144
headerLock       sync.Mutex   // L145
setQuarantineErr sync.Once    // L149
setRejectErr     sync.Once    // L153
wg               sync.WaitGroup // L155
...
data.wg.Add(1)   // L160
go func() {      // L161  -> parallel execution
	...
}()
...
data.wg.Wait()   // L198
```

Results are merged safely: authentication results and headers are combined under `authResLock`/
`headerLock`, while quarantine and reject verdicts are captured once via `sync.Once`
(`data.setQuarantineErr.Do(...)` at L180, `data.setRejectErr.Do(...)` at L184). The final verdict has
**reject-over-quarantine precedence** — reject is checked first:

```go
// internal/msgpipeline/check_runner.go:L199-L206
if data.rejectErr != nil {
	return data.rejectErr
}
if data.quarantineErr != nil {
	cr.log.Error("quarantined", data.quarantineErr)
	cr.mergedRes.Quarantine = true
}
```

Coordination across the message phases is handled by **replay**. When a check's state is created lazily
(logged as `cr.log.Debugf("initializing state for %v (%p)", objectName(check), check)`
[`internal/msgpipeline/check_runner.go:L67`]), any earlier phases are replayed for it. The code comment says exactly why:
*"Here we replay previous CheckConnection/CheckSender/CheckRcpt calls for any newly initialized checks so
they all get change to see all these things."* [`internal/msgpipeline/check_runner.go:L82-L83`] — replaying `CheckConnection`
[`internal/msgpipeline/check_runner.go:L89`], `CheckSender` [`internal/msgpipeline/check_runner.go:L97`], and `CheckRcpt` [`internal/msgpipeline/check_runner.go:L123`]. So every check observes the same
connection/sender/recipient events even though states are created at different times. The documented
execution order is *"CheckConnection, CheckSender, CheckRcpt, CheckBody"* [`HACKING.md:L117-L118`].

### Modifiers: a strict three-layer chain

Modifiers run as an ordered chain of three layers, each an independent module state:

1. **Global** modifiers — `dd.d.globalModifiers.ModStateForMsg(...)` inside `initRunGlobalModifiers`
   [`internal/msgpipeline/msgpipeline.go:L141-L153`], invoked at [`internal/msgpipeline/msgpipeline.go:L109`].
2. **Source-block** modifiers — `sourceBlock.modifiers.ModStateForMsg(...)`
   [`internal/msgpipeline/msgpipeline.go:L127`].
3. **Per-recipient** modifiers — `rcptBlock.modifiers.ModStateForMsg(...)` inside `getRcptModifiers`
   [`internal/msgpipeline/msgpipeline.go:L488-L494`].

At this commit modifiers are **header-only**: *"currently this is not possible to modify the body
contents, only header can be modified."* [`HACKING.md:L120-L128`].

Two recipient-related mechanisms are easy to conflate, so it is worth stating them **independently** —
the source supports each on its own, and there is **no data flow from the first into the second**:

- **`OriginalRcpts` — a final→original recipient mapping used for status/DSN reporting.** The pipeline
  initializes it once per delivery (`if msgMeta.OriginalRcpts == nil { msgMeta.OriginalRcpts =
  map[string]string{}` [`internal/msgpipeline/msgpipeline.go:L90-L91`]); it is declared as
  `OriginalRcpts map[string]string` [`internal/module/msgmetadata.go:L89`] and documented as *"the
  mapping from the final recipient to the recipient that was presented by the client"* that *"should be
  used when reporting information back to client (via DSN, for example) to prevent disclosing information
  about aliases which is usually unwanted"* [`internal/module/msgmetadata.go:L80-L88`]. It is populated
  **only when a recipient modifier actually rewrites the address** — `if originalTo != to {
  dd.msgMeta.OriginalRcpts[to] = originalTo }` [`internal/msgpipeline/msgpipeline.go:L288-L289`] — and its
  **only** consumer is the pipeline's own `statusCollector.SetStatus`, which maps the effective recipient
  back to the original before status is reported (`original, ok := sc.originalRcpts[rcptTo]` … `rcptTo =
  original` [`internal/msgpipeline/msgpipeline.go:L351-L356`]).
- **`Delivered-To` — added by `sql` storage from the *effective* account, independently of
  `OriginalRcpts`.** The pipeline calls `delivery.AddRcpt(ctx, to)` with the **effective** recipient `to`
  [`internal/msgpipeline/msgpipeline.go:L298`]. The storage's `AddRcpt` method
  [`internal/storage/sql/sql.go:L71`] receives that as its `rcptTo` argument, derives `accountName, err :=
  prepareUsername(rcptTo)` [`internal/storage/sql/sql.go:L74`], and adds `userHeader.Add("Delivered-To",
  accountName)` [`internal/storage/sql/sql.go:L94`]. The `Delivered-To` value therefore comes from the
  **effective account name**; the source does **not** read `OriginalRcpts` when building this header.

### Routing selects *which* block's checks/modifiers apply

Which source block and recipient block apply is decided by **two-level routing**, full-address →
domain → default:

- **Source** (`srcBlockForAddr`, [`internal/msgpipeline/msgpipeline.go:L155-L202`]): full address
  `dd.d.perSource[cleanFrom]` [`internal/msgpipeline/msgpipeline.go:L171`] → domain `dd.d.perSource[domain]` [`internal/msgpipeline/msgpipeline.go:L190`] →
  `dd.d.defaultSource` [`internal/msgpipeline/msgpipeline.go:L193`].
- **Destination** (`rcptBlockForAddr`, [`internal/msgpipeline/msgpipeline.go:L446-L486`]): full
  `dd.sourceBlock.perRcpt[cleanRcpt]` [`internal/msgpipeline/msgpipeline.go:L458`] → domain `perRcpt[domain]` [`internal/msgpipeline/msgpipeline.go:L474`] →
  `dd.sourceBlock.defaultRcpt` [`internal/msgpipeline/msgpipeline.go:L477`].

### Rationale

Checks and modifiers never name each other because the **pipeline** is the coordinator: it selects the
applicable source/recipient blocks by routing, runs that block's checks in parallel (with replay so
they share phase context), merges their verdicts with a fixed precedence, and threads the message
through the fixed global→source→recipient modifier chain. Coordination is entirely structural — a
property of *where* a check/modifier is declared inside a pipeline — which is exactly why no explicit
wiring appears in the config.

---

## Q5 — Runtime evidence the module graph has settled

> *What runtime evidence shows the module graph has fully settled into place?*

### The settle mechanism: Loop 3

After endpoints are initialized (Loop 2), `instancesFromConfig` runs a final **settle guard**,
Loop 3 [`maddy.go:L358-L365`]:

```go
// maddy.go:L358-L365
for _, inst := range mods {
	if module.Initialized[inst.instance.InstanceName()] {
		continue
	}
	return nil, fmt.Errorf("Unused configuration block at %s:%d - %s (%s)",
		inst.cfg.File, inst.cfg.Line, inst.instance.InstanceName(), inst.instance.Name())
}
```

Every regular block created in Loop 1 must, by now, appear in the shared `module.Initialized` map
[`internal/module/instances.go:L16`] — i.e. it must have been reached by *some* `&` reference and
lazily initialized. If any block was never referenced, it is absent from `Initialized`, and startup
**aborts** with the format string `"Unused configuration block at %s:%d - %s (%s)"`
[`maddy.go:L363-L364`]. "Settled" therefore means: *every configured regular block was actually used,
and every endpoint is initialized and listening.*

### Positive evidence — the valid run settles and stays up

In the valid run, all three endpoints reach their `listening on` lines and the process is still alive
when `timeout` sends `SIGTERM` after 8 seconds (`cat -A`; trailing `^I` = TAB, `$` = newline; the
final `EXIT=124` is the shell echo, not a maddy log line, so it carries no tab):

```
imap: listening on tcp://127.0.0.1:2143^I$
imap: authentication over unencrypted connections is allowed, this is insecure configuration and should be used only for testing!^I$
imap: TLS is disabled, this is insecure configuration and should be used only for testing!^I$
signal received (terminated), next signal will force immediate shutdown.^I$
EXIT=124
```

`EXIT=124` is `timeout`'s exit code meaning the process was **still running** when the signal was sent —
the graph settled and the server stayed up serving. (The `signal received (terminated)...` line is the
graceful-shutdown handler, `log.Printf("signal received (%v), next signal will force immediate shutdown.", s)`
[`signal.go:L35`].) For comparison, a normal `Run()` return would exit `0` [`maddy.go:L163`]; the
module/config error path returns `2` [`maddy.go:L160`].

### Negative / confirming evidence — the orphan run aborts

The orphan run uses the **same endpoint set as the valid config** (`smtp`, `submission`, `imap`) and
adds an unreferenced `sql orphan_storage` block (header at config line 34). It is created in Loop 1 but
never `&`-referenced, so it never reaches `GetInstance` and is absent from `Initialized`. All three
endpoints reach their `listening on` lines first, and only then does Loop 3 catch the orphan and abort.
The tail below (`cat -A`; trailing `^I` = TAB, `$` = newline; the final `EXIT=2` is the shell echo, not a
maddy log line, so it carries no tab) shows the `submission` and `imap` listeners settling immediately
before the abort:

```
submission: listening on tcp://127.0.0.1:2587^I$
[debug] /tmp/obs/orphan.conf:23: reference &local_authdb^I$
[debug] /tmp/obs/orphan.conf:24: reference &local_mailboxes^I$
imap: listening on tcp://127.0.0.1:2143^I$
imap: authentication over unencrypted connections is allowed, this is insecure configuration and should be used only for testing!^I$
imap: TLS is disabled, this is insecure configuration and should be used only for testing!^I$
Unused configuration block at /tmp/obs/orphan.conf:34 - orphan_storage (sql)^I$
EXIT=2
```

The final line matches the format string exactly, with `File=/tmp/obs/orphan.conf`, `Line=34`
(the orphan block's header line), `InstanceName=orphan_storage`, and `Name=sql`. `EXIT=2` comes from the
top-level error path returning `2` [`maddy.go:L160`], which `os.Exit`s in
`cmd/maddy/main.go:L10`.

### The ordering is itself evidence

Note that in the orphan run all three endpoints (`smtp`, `submission`, `imap`) log `listening on`
**before** the `Unused configuration block` abort. That ordering is observable proof that **Loop 2
(eager endpoint init + listeners) runs before
Loop 3 (settle guard)** — the unused-block check is the *last* startup step, the final gate after the
graph is otherwise wired up. The two exit codes make the settle state unambiguous:

| Run | Final observable | Exit code | Meaning |
|-----|------------------|-----------|---------|
| Valid | all `listening on` lines, then `signal received (terminated)...` | `EXIT=124` | Settled; still running when SIGTERM arrived (`timeout`) |
| Orphan | `Unused configuration block at /tmp/obs/orphan.conf:34 - orphan_storage (sql)` | `EXIT=2` | Did **not** settle; aborted by Loop 3 (`maddy.go:L160`,`maddy.go:L363-L364`) |

---

## Reproducibility note

- **Exact commands.** Build (out-of-tree output): `CGO_ENABLED=1 ... go build -o /tmp/obs/maddy ./cmd/maddy`.
  Run: `timeout -s TERM 8 /tmp/obs/maddy -debug -config <cfg>`. The `-debug` flag corresponds to
  `flag.BoolVar(&log.DefaultLogger.Debug, "debug", ...)` [`maddy.go:L104`] and is what makes the
  `[debug]`-prefixed lines (e.g. `reference &...`) appear.
- **Toolchain.** Evidence was captured in this environment with **Go 1.18.10**
  (`go version go1.18.10 linux/amd64`); the module's declared floor is `go 1.13` [`go.mod:L3`], which
  Go 1.18.10 satisfies. Only the `go version` string is toolchain-specific; the log lines and exit
  codes derive from the source at HEAD `26452dd`.
- **Out-of-tree, working tree clean.** All configs, state/runtime directories, databases, and the compiled
  binaries lived under `/tmp/obs` (outside the repository) and were removed afterward, so
  `git status --porcelain` on the repository is empty.
- **CGO / SQLite3 prerequisite.** Observing the default `sql`/SQLite3 storage requires `CGO_ENABLED=1`
  plus a C compiler, because the SQLite3 driver is gated by the build constraint `// +build !nosqlite3,cgo`
  [`internal/storage/sql/sqlite3.go:L1`] (which then does `import _ "github.com/mattn/go-sqlite3"` at
  [`internal/storage/sql/sqlite3.go:L5`]). Building **without** CGO still compiles, but the SQLite3 driver
  is absent and `sql` init fails at runtime. Observed with a `CGO_ENABLED=0` build (same `cat -A`
  convention — trailing `^I` = TAB, `$` = newline):

  ```
  $ CGO_ENABLED=0 go build -o /tmp/obs/maddy_nocgo ./cmd/maddy       # still builds (exit 0)
  $ timeout -s TERM 8 /tmp/obs/maddy_nocgo -debug -config /tmp/obs/valid.conf 2>/tmp/obs/nocgo.stderr ; echo "EXIT=$?"
  EXIT=2
  $ cat -A /tmp/obs/nocgo.stderr
  [debug] /tmp/obs/valid.conf:12: reference &local_mailboxes^I$
  sql: NewBackend (open): sql: unknown driver "sqlite3" (forgotten import?)^I$
  ```

  The stdlib error `sql: unknown driver "sqlite3" (forgotten import?)` (from `database/sql`) is wrapped by
  go-imap-sql's `NewBackend (open):` prefix and then by maddy's `fmt.Errorf("sql: %s", err)`
  [`internal/storage/sql/sql.go:L265`] (the failure occurs inside `imapsql.New(...)`
  [`internal/storage/sql/sql.go:L263`], *before* the version log at
  [`internal/storage/sql/sql.go:L268`] — which is why no `go-imap-sql version` line appears in the no-CGO
  run). The `reference &local_mailboxes` line still prints first because `internal/config/module/modconfig.go:L68` runs before the
  error is returned at `internal/config/module/modconfig.go:L73`.
- **Version string distinction.** The runtime prints `go-imap-sql version 0.4.0` — this is the dependency's
  compiled-in constant `const VersionStr = "0.4.0"`
  (`github.com/foxcpp/go-imap-sql@v0.3.2-0.20191208094750-8b4ec6b19a78/version.go:L7`). It is **distinct**
  from the `go.mod` module pin `github.com/foxcpp/go-imap-sql v0.3.2-0.20191208094750-8b4ec6b19a78`
  [`go.mod:L20`]. Do not conflate the library's self-reported version with the module pseudo-version.
- **Version-fidelity reminder.** All names above are the non-namespaced names at HEAD `26452dd`
  (`sql`, `smtp`, `submission`, `lmtp`, `imap`, `queue`, `remote`), **not** the newer upstream namespaced
  syntax.

---

## Coverage-pass checklist

Re-reading the original question, every sub-part — including the specific nuances — is addressed:

| # | Sub-question (and nuance) | Answered in | Status |
|---|---------------------------|-------------|--------|
| Q1 | Immediate vs. deferred: what is registered *immediately* vs. left *unresolved* | [§ Q1](#q1--immediate-vs-deferred) — constructors register at package-init (`maddy.go:L20-L38`, `internal/module/registry.go:L7-L11`); Loop 1 creates+registers instances but does **not** `Init` (`maddy.go:L338`); `Init` and `&name` resolution are deferred | ✅ |
| Q2 | Lazy init & `&`; what happens during *parsing*; **even when declared out of order** | [§ Q2](#q2--lazy-initialization-and-the--ampersand-syntax) — parser stores `&name` as a plain `Node.Args` token (`pkg/cfgparser/parse.go:L97`); `ModuleFromNode` resolves later (`internal/config/module/modconfig.go:L59,L67-L68`); `GetInstance` is lazy + at-most-once with cycle-break (`internal/module/instances.go:L65,L69-L70`); proven by `sql` declared last/resolved first and version line printing once | ✅ |
| Q3 | Endpoint divergence, **and why** | [§ Q3](#q3--endpoint-lifecycle-divergence) — `FuncNewEndpoint` contract (`internal/module/module.go:L59-L71`); Loop 1 skips registration (`maddy.go:L319-L320`); Loop 2 initializes eagerly/directly (`maddy.go:L352-L356`); *why* = endpoints are unreferenced roots that must open listeners (`HACKING.md:L60-L62`); evidenced by the three `listening on` lines | ✅ |
| Q4 | Check/modifier coordination **without explicit wiring** | [§ Q4](#q4--check--modifier-coordination-no-explicit-wiring) — pipeline owns checks/modifiers per block (no cross-reference); parallel checks + `WaitGroup` + replay (`internal/msgpipeline/check_runner.go:L82-L83,L155-L161,L199-L206`); three-layer modifier chain (`internal/msgpipeline/msgpipeline.go:L109,L127,L488-L494`); routing by full→domain→default; **code-grounded, startup-only observation (caveat stated)** | ✅ |
| Q5 | Runtime evidence the graph *settled* | [§ Q5](#q5--runtime-evidence-the-module-graph-has-settled) — Loop 3 settle guard (`maddy.go:L358-L365`); positive: valid run `listening on` + `EXIT=124`; negative: orphan run `Unused configuration block at /tmp/obs/orphan.conf:34 - orphan_storage (sql)` + `EXIT=2`; listeners log before the abort → Loop 2 precedes Loop 3 | ✅ |

All five sub-questions are answered explicitly, each pairing exact `file:line` citations with observed
output (or, for Q4's message-flow internals, explicit code-only grounding with the verifiability caveat)
and a short rationale.

