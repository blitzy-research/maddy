# maddy Logging Behaviour Across the SMTP Endpoint, Queue, and Remote Delivery Subsystems

> A runtime-verified onboarding reference for developers new to
> [`github.com/foxcpp/maddy`](https://github.com/foxcpp/maddy).

## Introduction

This document explains **how the maddy email server writes its logs** across three
delivery subsystems, and answers eleven specific runtime-behaviour questions
(grouped **A**–**D**) about those logs:

- **Group A — SMTP submission/reception endpoint** (`internal/endpoint/smtp`): what a
  successful recipient line looks like, how success differs from an abort, the module
  prefix, and the message-ID format.
- **Group B — outbound message queue** (`internal/target/queue`): the complete ordered
  sequence of log messages from acceptance through a retry, and the exact
  attempt/failure/retry lines.
- **Group C — remote (MX) delivery target** (`internal/target/remote`): the MX-authenticity
  failure text, the SMTP enhanced status code(s) and reply text, and the TLS→plaintext
  fallback line.
- **Group D — queue retry logging** (`internal/target/queue`): the retry-scheduling line
  and the JSON field that carries the retry delay.

### Commit under investigation

Everything below was observed and every `file:line` citation was verified against the
checked-out commit:

```
branch: maddy_26452dd8dd78
commit: 26452dd8dd787dc455278b0fdd296f4a5432c768
```

### Methodology: read-only + run-first

This was a **read-only** investigation — no source file was modified. The answers were
produced **run-first**: the relevant Go tests were built and executed, the *real* output
was captured, and each answer was written from what was observed and then traced back to
the emitting call site. Every claim below is paired with the exact command that produced
its evidence and a `file:line` citation. Values that vary between runs (the random
`msg_id`, the retry-delay nanosecond delta) were confirmed across at least two runs and are
explicitly separated from what is stable (field names, ordering, formats, module prefixes).

Where a value comes from a **test-harness stand-in** rather than the production code path
(most importantly the 40-hex `msg_id` used by the queue/remote tests), it is explicitly
labeled **non-canonical**.

---

## Reproduction environment

The toolchain is **Go 1.13.15**, which satisfies the module's `go 1.13` floor
(`go.mod:L3`). gcc is not required because the `smtp`, `queue`, and `remote` packages need
no cgo, so all builds/tests use `CGO_ENABLED=0`.

```
export PATH=$PATH:/usr/local/go/bin
export GOPATH=/root/go                 # outside the repo tree
export GOCACHE=/root/.cache/go-build   # outside the repo tree
export CGO_ENABLED=0                   # smtp/queue/remote packages need no cgo
export GO111MODULE=on
export GOFLAGS=-mod=readonly           # protects go.mod/go.sum from modification
export GOPROXY=https://proxy.golang.org,direct
go version   # -> go version go1.13.15 linux/amd64
```

The canonical per-test observation command shape is:

```
go test -count=1 -v -run '<TestName>$' ./internal/<pkg>/ [-test.debuglog]
```

- `-count=1` disables test caching so each run truly re-executes (needed to confirm
  run-to-run variation).
- `-v` surfaces each test's `t.Log` output (in the default logger mode, log records are
  routed through `t.Log`).
- `-test.debuglog` is a maddy-specific test flag (see the preamble) that additionally emits
  `[debug]`-level lines.

All Go module and build caches live **outside** the repository tree (`/root/...`), and all
temporary evidence scripts were kept outside the repo and deleted afterward, so the working
tree stays git-clean and only this document is added.

---
## Shared log-format preamble

Every group's answer depends on a handful of shared mechanics implemented by maddy's
in-house structured logger in `internal/log`. maddy does **not** use a third-party logging
library. Understand these once and every captured line below becomes self-explanatory.

### 1. Line shape: `<Name>: <msg>` + TAB + ordered JSON

`Logger.Msg` builds a `map[string]interface{}` from the variadic `fields...`, then writes
the message, a **TAB**, and a single-line JSON object of those fields. The emitter is:

- `func (l Logger) Msg(msg string, fields ...interface{})` — `internal/log/log.go:L72`
- `func (l Logger) Error(msg string, err error, fields ...interface{})` —
  `internal/log/log.go:L89`; `Error` additionally merges the error's structured fields via
  `exterrors.Fields(err)` (this is how `reason`, `smtp_code`, etc. appear).

The module **Name** is turned into the `"<Name>: "` prefix inside
`func (l Logger) log(debug bool, s string)` at `internal/log/log.go:L180-L183`:

```go
func (l Logger) log(debug bool, s string) {
	if l.Name != "" {
		s = l.Name + ": " + s
	}
```

### 2. Fields are printed in alphabetical order

`marshalOrderedJSON` collects the keys and calls `sort.Strings(order)` before emitting them
(`internal/log/orderedjson.go:L16-L23`):

```go
func marshalOrderedJSON(output *strings.Builder, m map[string]interface{}) error {
	order := make([]string, 0, len(m))
	for k := range m {
		order = append(order, k)
	}
	sort.Strings(order)
```

This is why, for example, the `RCPT ok` line prints `msg_id` **before** `rcpt` even though
the call site passes `"rcpt", to, "msg_id", ...` — the output order is alphabetical, not
call order.

### 3. Special-type value rendering

Before a value is JSON-encoded, `orderedjson.go:L41-L50` special-cases a few types:

```go
		switch casted := val.(type) {
		case time.Time:
			val = casted.Format("2006-01-02T15:04:05.000")
		case time.Duration:
			val = casted.String()
		case LogFormatter:
			val = casted.FormatLog()
		case fmt.Stringer:
			val = casted.String()
		case error:
			val = casted.Error()
		}
```

Consequences used below:

- `next_try_delay` is a `time.Duration`, so it prints as a duration string (`-597ns`,
  `15m0s`).
- `smtp_enchcode` is an `EnhancedCode`, which implements `LogFormatter.FormatLog()`, so in a
  **real** structured log it renders as `5.4.0` (see Group C for why the *test* prints
  `[5 4 0]` instead).
- an `error` value renders via its `Error()` string (this is the `reason` field).

### 4. Timestamp behaviour (OBSERVED in both modes)

The test logger (`testutils.Logger`) has two output modes, selected by `-test.directlog`
(`internal/testutils/logger.go:L18-L44`):

- **Default mode** routes each record through `t.Log` via a `FuncOutput` whose function
  signature is `func(_ time.Time, debug bool, str string)` — the `time.Time` timestamp
  argument is **ignored** (`internal/testutils/logger.go:L27-L40`). So default `-v` output
  carries **no timestamp**, and each line is additionally prefixed by Go's own testing
  caller annotation `output.go:41:` (that prefix comes from `testing.T.Log`, not from
  maddy). OBSERVED (default mode):

```text
smtp: RCPT ok	{"msg_id":"e242088d","rcpt":"rcpt1@example.com"}
```

  (shown with the `output.go:41:` caller prefix stripped; the raw captured line is
  `    output.go:41: ` + the above).

- **Direct mode** (`-test.directlog`) returns `log.WriterOutput(os.Stderr, true)`
  (`internal/testutils/logger.go:L19-L23`); `wcOutput.Write` then prefixes a UTC timestamp
  `Format("2006-01-02T15:04:05.000Z ")` (`internal/log/writer.go:L16-L25`). Because the
  writer bypasses `t.Log`, there is **no** `output.go:41:` prefix. OBSERVED (direct mode,
  `-test.directlog`):

```text
2026-07-08T04:26:16.769Z smtp: RCPT ok	{"msg_id":"310d2821","rcpt":"rcpt1@example.com"}
```

  The writer source (`internal/log/writer.go:L16-L25`):

```go
func (w wcOutput) Write(stamp time.Time, debug bool, msg string) {
	builder := strings.Builder{}
	if w.timestamps {
		builder.WriteString(stamp.UTC().Format("2006-01-02T15:04:05.000Z "))
	}
	if debug {
		builder.WriteString("[debug] ")
	}
	builder.WriteString(msg)
	builder.WriteRune('\n')
```

### 5. The `[debug] ` prefix and the two test flags

Debug-level lines (emitted by `Debugln`/`Debugf`/`DebugMsg`) are prefixed with `[debug] `
in both modes and are only shown when `-test.debuglog` is set. The two maddy test flags are
declared at `internal/testutils/logger.go:L13-L15`:

```go
debugLog  = flag.Bool("test.debuglog", false, "(maddy) Turn on debug log messages")
directLog = flag.Bool("test.directlog", false, "(maddy) Log to stderr instead of test log")
```

### 6. Per-delivery `msg_id` injection (with an important nuance)

`target.DeliveryLogger` clones a logger's `Fields` map and sets `fields["msg_id"] =
msgMeta.ID` (`internal/target/delivery.go:L8-L15`):

```go
func DeliveryLogger(l log.Logger, msgMeta *module.MsgMetadata) log.Logger {
	fields := make(map[string]interface{}, len(l.Fields)+1)
	for k, v := range l.Fields {
		fields[k] = v
	}
	fields["msg_id"] = msgMeta.ID
	l.Fields = fields
	return l
}
```

So every line emitted **through a delivery logger** carries a `msg_id` field.

**Nuance (OBSERVED + source-confirmed):** some queue lifecycle lines are emitted through the
*raw* queue logger `q.Log` (Name `queue`) rather than the per-delivery logger `dl`. Those
raw lines (`starting delivery for ...`, the semaphore lines) therefore carry **no** `msg_id`
JSON field — the ID appears only inside the message text. This is detailed in Group B.

---
## Group A — SMTP endpoint logging

All Group A output was captured in the default logger mode (no timestamp; each maddy line is
shown by `go test` with a leading `output.go:41:` caller annotation, which is Go's testing
prefix — the maddy log line is everything from `smtp:` onward).

### A1 — the complete `RCPT ok` line, with every JSON field

**Direct answer.** A successful recipient add emits exactly:

```text
smtp: RCPT ok	{"msg_id":"e242088d","rcpt":"rcpt1@example.com"}
```

It carries **two** JSON fields, in this order: `msg_id`, then `rcpt`.

**Command:**

```
go test -count=1 -v -run 'TestSMTPDelivery$' ./internal/endpoint/smtp/
```

**Complete unedited output:**

```text
=== RUN   TestSMTPDelivery
--- PASS: TestSMTPDelivery (0.00s)
    output.go:41: smtp: listening on tcp://127.0.0.1:13172	
    output.go:41: smtp: incoming message	{"msg_id":"e242088d","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:50520"}
    output.go:41: smtp: RCPT ok	{"msg_id":"e242088d","rcpt":"rcpt1@example.com"}
    output.go:41: smtp: RCPT ok	{"msg_id":"e242088d","rcpt":"rcpt2@example.com"}
    output.go:41: smtp: accepted	{"msg_id":"e242088d"}
PASS
ok  	github.com/foxcpp/maddy/internal/endpoint/smtp	0.004s
```

**Source citation.** `internal/endpoint/smtp/smtp.go:L243`, in the RCPT handler:

```go
s.endp.Log.Msg("RCPT ok", "rcpt", to, "msg_id", s.msgMeta.ID)
```

**Causal reasoning.** The call site passes the fields in the order `rcpt`, then `msg_id`,
but the ordered-JSON marshaller sorts keys alphabetically (`orderedjson.go:L16-L23`), and
`msg_id` < `rcpt`, so the emitted order is the reverse of the call order. The `smtp:` prefix
is the logger `Name` (see A3). There is no timestamp because this is default (`t.Log`) mode
(preamble §4).

**Sibling variants.**
- **Second recipient (OBSERVED)** — the same test adds a second recipient, producing an
  identical line with a different `rcpt`:

```text
smtp: RCPT ok	{"msg_id":"e242088d","rcpt":"rcpt2@example.com"}
```

- **`incoming message` (OBSERVED)** — the line that opens every transaction, with four
  fields (`msg_id`, `sender`, `src_host`, `src_ip`), emitted at `smtp.go:L128`:

```text
smtp: incoming message	{"msg_id":"e242088d","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:50520"}
```

- **`accepted` (OBSERVED)** — the terminal success line (see A2), `smtp.go:L334`.

### A2 — success vs. abort

**Direct answer.** A **successful** transaction ends with `smtp: accepted`; an **aborted**
one ends with `smtp: aborted`. When the abort happens during the DATA phase, the `aborted`
line is immediately preceded by a `smtp: DATA error` line carrying a `reason`. Concretely,
the success side ends with:

```text
smtp: accepted	{"msg_id":"e242088d"}
```

and the DATA-abort side ends with:

```text
smtp: DATA error	{"msg_id":"04f1d745","reason":"unexpected EOF"}
smtp: aborted	{"msg_id":"04f1d745"}
```

**Command (abort case):**

```
go test -count=1 -v -run 'TestSMTPDelivery_AbortData$' ./internal/endpoint/smtp/
```

**Complete unedited output (abort case):**

```text
=== RUN   TestSMTPDelivery_AbortData
--- PASS: TestSMTPDelivery_AbortData (0.25s)
    output.go:41: smtp: listening on tcp://127.0.0.1:63826	
    output.go:41: smtp: incoming message	{"msg_id":"04f1d745","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:50042"}
    output.go:41: smtp: RCPT ok	{"msg_id":"04f1d745","rcpt":"test@example.com"}
    output.go:41: smtp: DATA error	{"msg_id":"04f1d745","reason":"unexpected EOF"}
    output.go:41: smtp: aborted	{"msg_id":"04f1d745"}
PASS
ok  	github.com/foxcpp/maddy/internal/endpoint/smtp	0.255s
```

**Source citations.**
- `accepted` — `s.log.Msg("accepted", "msg_id", s.msgMeta.ID)` at
  `internal/endpoint/smtp/smtp.go:L334` (the atomic `Body` path) and the identical call at
  `L377` (the `BodyNonAtomic` path).
- `aborted` — `s.log.Msg("aborted", "msg_id", s.msgMeta.ID)` at
  `internal/endpoint/smtp/smtp.go:L72`.
- `DATA error` — `s.log.Error("DATA error", err, "msg_id", s.msgMeta.ID)` at
  `internal/endpoint/smtp/smtp.go:L317` and the identical call at `L360` (the two `wrapErr`
  closures for the atomic and non-atomic body paths).

**Causal reasoning.** `accepted` and `aborted` each carry only `msg_id` because that is the
sole field passed. `DATA error` is emitted through `Logger.Error`, which merges the wrapped
error's fields; here that yields the `reason` field, whose value `"unexpected EOF"` is the
error's `Error()` string. `msg_id` sorts before `reason` alphabetically.

**Sibling variants.**
- **Abort by logout, NO `DATA error` (OBSERVED)** — `TestSMTPDelivery_AbortLogout` aborts by
  quitting after RCPT, before any DATA failure, so it ends with `aborted` and has **no**
  preceding `DATA error`:

```text
smtp: aborted	{"msg_id":"3bebf265"}
```

  Command + complete unedited output:

```
go test -count=1 -v -run 'TestSMTPDelivery_AbortLogout$' ./internal/endpoint/smtp/
```

```text
=== RUN   TestSMTPDelivery_AbortLogout
--- PASS: TestSMTPDelivery_AbortLogout (0.25s)
    output.go:41: smtp: listening on tcp://127.0.0.1:52555	
    output.go:41: smtp: incoming message	{"msg_id":"3bebf265","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:60270"}
    output.go:41: smtp: RCPT ok	{"msg_id":"3bebf265","rcpt":"test@example.com"}
    output.go:41: smtp: aborted	{"msg_id":"3bebf265"}
PASS
ok  	github.com/foxcpp/maddy/internal/endpoint/smtp	0.254s
```

### A3 — the module prefix

**Direct answer.** The prefix is **`smtp`** (the leading `smtp:` on every line above).

**Command / output.** Visible in every Group A capture (e.g. `smtp: RCPT ok ...`).

**Source citation.** The endpoint constructor sets the logger `Name` to the module name at
`internal/endpoint/smtp/smtp.go:L495`:

```go
Log:        log.Logger{Name: modName},
```

and the test installs a logger with that name at `internal/endpoint/smtp/smtp_test.go:L49`:

```go
endp.Log = testutils.Logger(t, "smtp")
```

**Causal reasoning.** `modName` is the registered module name for the endpoint instance. The
`Name` becomes the `"<Name>: "` prefix via `Logger.log` (preamble §1).

**Sibling variants (SOURCE-DERIVED).** The same constructor builds `submission` and `lmtp`
endpoint instances, distinguished at `internal/endpoint/smtp/smtp.go:L492-L493`:

```go
submission: modName == "submission",
lmtp:       modName == "lmtp",
```

so the prefix is `submission` or `lmtp` for those module instances (they share this
`smtp.go` code and thus the same `RCPT ok` / `accepted` / `aborted` lines).

### A4 — the exact `msg_id` format

**Direct answer.** The canonical production `msg_id` is **8 lowercase hexadecimal
characters** (4 random bytes, hex-encoded).

**Command / output.** OBSERVED across five separate runs, the `msg_id` differs every time
while the format is constant:

- `TestSMTPDelivery` run 1 → `e242088d`
- `TestSMTPDelivery` run 2 → `1f2e1254`
- `TestSMTPDelivery_AbortData` → `04f1d745`
- `TestSMTPDelivery_AbortLogout` → `3bebf265`
- `TestSMTPDelivery` (direct mode) → `310d2821`

**Source citation.** `internal/msgpipeline/msgid.go:L12-L16`:

```go
func GenerateMsgID() (string, error) {
	rawID := make([]byte, 4)
	_, err := rand.Read(rawID)
	return hex.EncodeToString(rawID), err
}
```

It is called from the endpoint at `internal/endpoint/smtp/smtp.go:L112`.

**Causal reasoning.** 4 random bytes hex-encode to exactly 8 lowercase hex characters. The
value is drawn from `crypto/rand` per message, so it is **random/varying per run**; only the
**format** (8 hex chars) is stable.

**Sibling / contrast (labeled).** This 8-hex value is the **canonical** production
`msg_id`. The 40-hex `msg_id` seen in the Group B/C tests
(`af8090c7eb39f761862b1f027b4f2b0bb1ce86d1`, etc.) is a **non-canonical test-harness
artifact** — `hex(sha1(t.Name()))` — not produced by `GenerateMsgID`. See the recap for the
external verification.

---
## Group B — queue delivery tracing

Group B was captured with `-test.debuglog` so the full lifecycle (including `[debug]` lines)
is visible. The `msg_id` here is the **non-canonical** 40-hex harness value
`af8090c7eb39f761862b1f027b4f2b0bb1ce86d1` (= `sha1("TestQueueDelivery_TemporaryFail")`;
see recap).

### B1 — the complete ordered sequence from acceptance through retry to success

**Direct answer.** In `TestQueueDelivery_TemporaryFail` the message flows through **two**
delivery attempts: attempt #1 fails with a temporary error (`you shall not pass`), the queue
logs `will retry`, attempt #2 succeeds, the queue logs `delivered` for each recipient, and
finally the message is removed from disk. The always-emitted (non-debug) milestones are
`delivery attempt failed` → `will retry` → `delivered`; the surrounding `[debug]` lines show
the full lifecycle.

**Command:**

```
go test -count=1 -v -run 'TestQueueDelivery_TemporaryFail$' ./internal/target/queue/ -test.debuglog
```

**Complete unedited output (full sequence, not truncated):**

```text
=== RUN   TestQueueDelivery_TemporaryFail
=== PAUSE TestQueueDelivery_TemporaryFail
=== CONT  TestQueueDelivery_TemporaryFail
--- PASS: TestQueueDelivery_TemporaryFail (0.03s)
    output.go:41: [debug] queue: delivery target: *queue.unreliableTarget	
    target.go:166: -- tgt.Start tester@example.com
    target.go:166: -- delivery.AddRcpt tester1@example.org
    target.go:166: -- delivery.AddRcpt tester2@example.org
    target.go:166: -- delivery.Body
    target.go:166: -- delivery.Commit
    output.go:41: [debug] queue: starting delivery for af8090c7eb39f761862b1f027b4f2b0bb1ce86d1	
    output.go:41: [debug] queue: waiting on delivery semaphore for af8090c7eb39f761862b1f027b4f2b0bb1ce86d1	
    output.go:41: [debug] queue: delivery semaphore acquired for af8090c7eb39f761862b1f027b4f2b0bb1ce86d1	
    output.go:41: [debug] queue: delivery attempt #1	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
    output.go:41: [debug] queue: using message ID = af8090c7eb39f761862b1f027b4f2b0bb1ce86d1-1	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
    output.go:41: [debug] queue: target.Start OK	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
    output.go:41: [debug] queue: delivery.AddRcpt tester1@example.org OK	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
    output.go:41: [debug] queue: delivery.AddRcpt tester2@example.org OK	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
    output.go:41: [debug] queue: delivery.Body failed: you shall not pass	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
    output.go:41: [debug] queue: delivery.Body OK	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
    output.go:41: [debug] queue: delivery.Abort (all recipients failed)	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
    output.go:41: [debug] queue: failures: permanently: [], temporary: [tester1@example.org tester2@example.org], errors: map[tester1@example.org:you shall not pass tester2@example.org:you shall not pass]	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
    output.go:41: queue: delivery attempt failed	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org","reason":"you shall not pass"}
    output.go:41: queue: delivery attempt failed	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org","reason":"you shall not pass"}
    output.go:41: queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-597ns","rcpts":["tester1@example.org","tester2@example.org"]}
    output.go:41: [debug] queue: starting delivery for af8090c7eb39f761862b1f027b4f2b0bb1ce86d1	
    output.go:41: [debug] queue: waiting on delivery semaphore for af8090c7eb39f761862b1f027b4f2b0bb1ce86d1	
    output.go:41: [debug] queue: delivery semaphore acquired for af8090c7eb39f761862b1f027b4f2b0bb1ce86d1	
    output.go:41: [debug] queue: delivery attempt #2	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
    output.go:41: [debug] queue: using message ID = af8090c7eb39f761862b1f027b4f2b0bb1ce86d1-2	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
    output.go:41: [debug] queue: target.Start OK	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
    output.go:41: [debug] queue: delivery.AddRcpt tester1@example.org OK	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
    output.go:41: [debug] queue: delivery.AddRcpt tester2@example.org OK	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
    output.go:41: [debug] queue: delivery.Body OK	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
    output.go:41: [debug] queue: delivery.Commit OK	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
    output.go:41: [debug] queue: failures: permanently: [], temporary: [], errors: map[]	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org"}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org"}
    output.go:41: [debug] queue: removed message from disk	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
    queue_test.go:38: --- queue.Close
PASS
ok  	github.com/foxcpp/maddy/internal/target/queue	0.034s
```

**Source citations (line by line).**
- `starting delivery for <id>` — `q.Log.Debugln("starting delivery for", slot.ID)` at
  `internal/target/queue/queue.go:L278` (emitted through the **raw** `q.Log`).
- `delivery attempt #N` — `dl.Debugf("delivery attempt #%d", meta.TriesCount+1)` at
  `internal/target/queue/queue.go:L367`.
- `delivery attempt failed` — `dl.Error("delivery attempt failed", rcptErr, "rcpt", rcpt)`
  at `internal/target/queue/queue.go:L384`.
- `will retry` — `dl.Msg("will retry", ...)` at `internal/target/queue/queue.go:L415-L418`.
- `delivered` — `dl.Msg("delivered", "rcpt", rcpt, "attempt", meta.TriesCount+1)` at
  `internal/target/queue/queue.go:L378`.

The per-delivery logger is created at `internal/target/queue/queue.go:L366`:
`dl := target.DeliveryLogger(q.Log, meta.MsgMeta)`, over the base queue logger
`Log: log.Logger{Name: "queue"}` set at `internal/target/queue/queue.go:L188`.

**Causal reasoning — the `msg_id` nuance (OBSERVED + source-confirmed).** The AAP's shorthand
"every line carries a `msg_id` field" is **only** true for the delivery-logger (`dl`) lines.
The `starting delivery for ...` line and the two semaphore lines are emitted through the
**raw** `q.Log` (not `dl`), so they carry **no** `msg_id` JSON field — the ID appears only
inside the message *text*. You can see this directly in the capture: the `starting delivery
for af80...` line has no trailing `{...}` object, whereas every line from `delivery attempt
#1` onward (built from `dl`) does. This is because `DeliveryLogger` injects `msg_id` into the
logger's `Fields` (preamble §6), and only `dl` has that field set.

**Debug vs. non-debug.** `starting delivery for` and `delivery attempt #N` are
`Debugln`/`Debugf` (visible only with `-test.debuglog`); `delivery attempt failed` (`Error`),
`will retry` (`Msg`), and `delivered` (`Msg`) are always emitted.

### B2 — the exact attempt / failure / retry lines

**Direct answer.** The delivery-attempt failure and the retry-scheduling lines, quoted
verbatim (one `delivery attempt failed` per still-temporary recipient, then one `will retry`
for the message):

```text
queue: delivery attempt failed	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org","reason":"you shall not pass"}
queue: delivery attempt failed	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org","reason":"you shall not pass"}
queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-597ns","rcpts":["tester1@example.org","tester2@example.org"]}
```

(The delivery *attempt itself* is the `[debug] queue: delivery attempt #1` line shown in the
full B1 capture above.)

**Source citations.** `delivery attempt failed` — `queue.go:L384`; `will retry` —
`queue.go:L415-L418`.

**Causal reasoning.** `delivery attempt failed` is emitted via `dl.Error(...)`, so the wrapped
error's `Error()` string becomes the `reason` field — here `"you shall not pass"`, the error
returned by the test's `unreliableTarget`. Fields sort alphabetically: `msg_id`, `rcpt`,
`reason`. `will retry` carries `attempts_count`, `msg_id`, `next_try_delay`, `rcpts` (again
alphabetical); the `next_try_delay` value is analysed in Group D.

**Sibling variants (queue outcomes)** at `internal/target/queue/queue.go:L378-L398`:
- `delivered` — **OBSERVED** (success, `L378`), shown above.
- `not delivered, temporary error` — **SOURCE-DERIVED** (`L394`): emitted for each
  still-temporarily-failing recipient once the maximum number of tries is exhausted.
- `not delivered, permanent error` — **SOURCE-DERIVED** (`L398`): emitted for each
  permanently-failed recipient.

---
## Group C — remote (MX) delivery logging

The remote target's base logger has `Name: "remote"` (`internal/target/remote/remote.go:L83`),
and each delivery wraps it with a per-delivery logger
(`Log = target.DeliveryLogger(rt.Log, msgMeta)`, `internal/target/remote/remote.go:L190`), so
remote log lines carry the `msg_id` field. The `msg_id` in these tests is again the
**non-canonical** 40-hex harness value (see recap).

### C1 — the exact MX-authenticity failure string

**Direct answer.** When MX-authenticity cannot be established, the inner error string is,
**verbatim** (note the source misspelling **"estabilish"**, reproduced exactly, not
corrected):

```text
Failed to estabilish the MX record (mx.example.invalid.) authenticity
```

**Command:**

```
go test -count=1 -v -run 'TestRemoteDelivery_AuthMX_Fail$' ./internal/target/remote/ -test.debuglog
```

**Complete unedited output:**

```text
=== RUN   TestRemoteDelivery_AuthMX_Fail
--- PASS: TestRemoteDelivery_AuthMX_Fail (0.00s)
    target.go:233: -- tgt.Start test@example.com
    target.go:233: -- delivery.AddRcpt test@example.invalid
    output.go:41: [debug] remote: trying	{"domain":"example.invalid","msg_id":"ac08d9f027f71627267fb3eae96f84d56762fa16","mx":"mx.example.invalid."}
    target.go:233: -- ... delivery.AddRcpt test@example.invalid Failed to estabilish the MX record (mx.example.invalid.) authenticity map[domain:example.invalid reason:Failed to estabilish the MX record (mx.example.invalid.) authenticity smtp_code:550 smtp_enchcode:[5 4 0] smtp_msg:No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity target:remote]
    target.go:233: -- delivery.Abort
PASS
ok  	github.com/foxcpp/maddy/internal/target/remote	0.004s
```

**Source citation.** The `requireMXAuth` branch in `checkPolicies`
(`internal/target/remote/connect.go:L94-L99`), where the `Message` (with the misspelling) is
`L98`:

```go
if rd.rt.requireMXAuth && !authenticated {
	return &exterrors.SMTPError{
		Code:         550,
		EnhancedCode: exterrors.EnhancedCode{5, 7, 0},
		Message:      fmt.Sprintf("Failed to estabilish the MX record (%s) authenticity", mx),
	}
}
```

**Causal reasoning.** `%s` is the MX host, here `mx.example.invalid.` (with the trailing dot,
as it comes from DNS). This error is an `exterrors.SMTPError`; it is not logged as a standalone
maddy line in this test but is surfaced through the mock target and wrapped by the outer
"No usable MXs" error (see C2).

**Sibling variants.**
- **MTA-STS enforce mismatch (SOURCE-DERIVED).** `internal/target/remote/connect.go:L63`
  logs `remote: skipping MX not matching MTA-STS` and returns 550 / `5.7.0` with message
  `"Failed to estabilish the MX record authenticity (MTA-STS)"` (`connect.go:L67`). Note:
  when run, `TestRemoteDelivery_AuthMX_MTASTS_Fail` actually surfaced the **generic**
  `requireMXAuth` error (identical `smtp_enchcode:[5 4 0]` / "No usable MXs" wrapper), so the
  `"(MTA-STS)"` message text is read-from-source, not observed emitting here. Its complete
  unedited output:

```text
=== RUN   TestRemoteDelivery_AuthMX_MTASTS_Fail
--- PASS: TestRemoteDelivery_AuthMX_MTASTS_Fail (0.00s)
    target.go:233: -- tgt.Start test@example.com
    target.go:233: -- delivery.AddRcpt test@example.invalid
    output.go:41: [debug] remote: trying	{"domain":"example.invalid","msg_id":"9c16c132275c01ac3cbf097c300dc342ba3dbfae","mx":"mx.example.invalid."}
    target.go:233: -- ... delivery.AddRcpt test@example.invalid Failed to estabilish the MX record (mx.example.invalid.) authenticity map[domain:example.invalid reason:Failed to estabilish the MX record (mx.example.invalid.) authenticity smtp_code:550 smtp_enchcode:[5 4 0] smtp_msg:No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity target:remote]
    target.go:233: -- delivery.Abort
PASS
ok  	github.com/foxcpp/maddy/internal/target/remote	0.005s
```

- **TLS required but unsupported (SOURCE-DERIVED).** `internal/target/remote/connect.go:L100-L105`
  returns 550 / `5.7.1` with `"TLS is required but unsupported or failed (mx = %s)"`.

### C2 — the enhanced status code(s) and complete reply text (BOTH layers)

**Direct answer.** There are **two distinct enhanced codes at two layers**, and both must be
reported:

- **Inner (the authenticity check itself):** SMTP code **550**, enhanced code **`5.7.0`**
  (Security or Policy Status), message
  `Failed to estabilish the MX record (mx.example.invalid.) authenticity`.
- **Outer (what is actually returned to the client — the "No usable MXs" wrapper):** SMTP
  code **550**, enhanced code **`5.4.0`** (Network and Routing Status), message
  `No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity`.

**Command / output.** Same run as C1. The mock target renders the returned `SMTPError`'s
fields as a Go map (via `%v`), OBSERVED verbatim:

```text
map[domain:example.invalid reason:Failed to estabilish the MX record (mx.example.invalid.) authenticity smtp_code:550 smtp_enchcode:[5 4 0] smtp_msg:No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity target:remote]
```

**Source citations.**
- Inner `EnhancedCode{5, 7, 0}` — `internal/target/remote/connect.go:L97` (hardcoded in the
  `requireMXAuth` branch shown in C1).
- Outer wrapper — `internal/target/remote/connect.go:L201-L214`:

```go
return nil, &exterrors.SMTPError{
	Code:         exterrors.SMTPCode(err, 451, 550),
	EnhancedCode: exterrors.SMTPEnchCode(err, exterrors.EnhancedCode{0, 4, 0}),
	Message:      "No usable MXs, last err: " + lastErr.Error(),
	TargetName:   "remote",
	Err:          lastErr,
	Misc: map[string]interface{}{
		"domain": domain,
	},
}
```

- The field names `smtp_code`, `smtp_enchcode`, `smtp_msg`, and `reason` come from
  `(*SMTPError).Fields()` at `internal/exterrors/smtp.go:L72-L91`:

```go
ctx["smtp_code"] = se.Code
ctx["smtp_enchcode"] = se.EnhancedCode
ctx["smtp_msg"] = se.Message
...
if se.Reason != "" {
	ctx["reason"] = se.Reason
} else if se.Err != nil {
	ctx["reason"] = se.Err.Error()
}
```

**Causal reasoning.**

1. **Why the test prints `[5 4 0]` but a real log prints `5.4.0`.** The mock target prints
   the `EnhancedCode` value with Go's `%v`, which renders the 3-element array as `[5 4 0]`. In
   a **real** structured maddy log, `EnhancedCode` implements `LogFormatter`, so
   `orderedjson.go` calls `FormatLog()` (`internal/exterrors/smtp.go:L11-L13`), which formats
   it as `5.4.0`:

```go
func (ec EnhancedCode) FormatLog() string {
	return fmt.Sprintf("%d.%d.%d", ec[0], ec[1], ec[2])
}
```

2. **Why the outer code becomes `5.4.0` (key finding).** The wrapper builds the enhanced code
   with `SMTPEnchCode(err, EnhancedCode{0, 4, 0})`. `SMTPEnchCode`
   (`internal/exterrors/smtp.go:L122-L128`) is:

```go
func SMTPEnchCode(err error, code EnhancedCode) EnhancedCode {
	if IsTemporary(err) {
		code[0] = 4
	}
	code[0] = 5
	return code
}
```

   The assignment `code[0] = 5` runs **unconditionally** — the `IsTemporary` branch that sets
   `code[0] = 4` is immediately overwritten and is effectively dead code — so the class digit
   is **always forced to `5`**. The subject/detail `.4.0` come from the `{0, 4, 0}` fallback.
   Hence `{0,4,0}` → `{5,4,0}` = `5.4.0`. The `smtp_code` `550` comes from
   `SMTPCode(err, 451, 550)` (`connect.go:L205`), which returns the permanent code `550`
   because the underlying error is permanent.

3. **RFC 3463 semantics (authoritative, cross-checked).** Enhanced codes are
   `class.subject.detail`
   ([RFC 3463](https://datatracker.ietf.org/doc/html/rfc3463)). Class `5` = **permanent
   failure**; subject `.4.` = **Network and Routing Status** (`X.4.0` = "Other or undefined
   network or routing status"); subject `.7.` = **Security or Policy Status** (`X.7.0` =
   "Other or undefined security status"). So the two layers are semantically coherent: the
   outer reply `5.4.0` (permanent, network/routing) generalises an MX-selection failure as a
   routing problem, while the inner check `5.7.0` (permanent, security/policy) states the true
   underlying cause — an MX authenticity/policy failure.

**Sibling variant — null MX (OBSERVED via passing assertion).** `TestRemoteDelivery_NullMX`
asserts (via `testutils.CheckSMTPErr`, `internal/target/remote/remote_test.go:L327`) that a
`.`-host MX yields SMTP **556**, enhanced **`5.1.10`**, message
`Domain does not accept email (null MX)` — the source is `connect.go:L155-L159`. The error is
**checked, not logged**, so the test produces no log line; its complete unedited output:

```text
=== RUN   TestRemoteDelivery_NullMX
--- PASS: TestRemoteDelivery_NullMX (0.00s)
PASS
ok  	github.com/foxcpp/maddy/internal/target/remote	0.003s
```

### C3 — the TLS→plaintext fallback line, with every JSON field

**Direct answer.** When the server falls back from TLS to plaintext it emits (four fields,
alphabetically sorted: `domain`, `msg_id`, `mx`, `reason`):

```text
remote: TLS error, falling back to plaintext	{"domain":"example.invalid","msg_id":"2176ec5872ed2b87d832b4070e88232bd94ac7d3","mx":"mx.example.invalid.","reason":"smtpconn: x509: certificate signed by unknown authority"}
```

**Command:**

```
go test -count=1 -v -run 'TestRemoteDelivery_TLSErrFallback$' ./internal/target/remote/ -test.debuglog
```

**Complete unedited output:**

```text
=== RUN   TestRemoteDelivery_TLSErrFallback
--- PASS: TestRemoteDelivery_TLSErrFallback (0.03s)
    target.go:166: -- tgt.Start test@example.com
    target.go:166: -- delivery.AddRcpt test@example.invalid
    output.go:41: [debug] remote: trying	{"domain":"example.invalid","msg_id":"2176ec5872ed2b87d832b4070e88232bd94ac7d3","mx":"mx.example.invalid."}
    output.go:41: remote: TLS error, falling back to plaintext	{"domain":"example.invalid","msg_id":"2176ec5872ed2b87d832b4070e88232bd94ac7d3","mx":"mx.example.invalid.","reason":"smtpconn: x509: certificate signed by unknown authority"}
    output.go:41: [debug] remote: connected	{"domain":"example.invalid","msg_id":"2176ec5872ed2b87d832b4070e88232bd94ac7d3","mx":"mx.example.invalid."}
    output.go:41: [debug] remote: connected	{"msg_id":"2176ec5872ed2b87d832b4070e88232bd94ac7d3","remote_server":"mx.example.invalid."}
    target.go:166: -- delivery.Body
    target.go:166: -- delivery.Commit
    output.go:41: [debug] remote: disconnected from mx.example.invalid.	{"msg_id":"2176ec5872ed2b87d832b4070e88232bd94ac7d3"}
PASS
ok  	github.com/foxcpp/maddy/internal/target/remote	0.032s
```

**Source citation.** `internal/target/remote/connect.go:L175-L177`, gated on the error being a
`smtpconn.TLSError` and MX-auth not objecting:

```go
if _, ok := err.(smtpconn.TLSError); ok && authErr == nil {
	rd.Log.Error("TLS error, falling back to plaintext", err,
		"mx", record.Host, "domain", domain)
```

**Causal reasoning.** `mx` and `domain` are the explicit call-site fields; `msg_id` is injected
by the per-delivery logger (`rd.Log` = `target.DeliveryLogger(rt.Log, msgMeta)`); `reason` is
the underlying TLS error's `Error()` text (here `"smtpconn: x509: certificate signed by unknown
authority"`), added because the line is emitted through `Logger.Error`. The four keys are
printed alphabetically (`domain`, `msg_id`, `mx`, `reason`). The fallback only happens when the
error is a `smtpconn.TLSError` **and** MX-auth policy did not object (`authErr == nil`) — if
TLS were required by policy, the code would instead return the `5.7.1` error from C1's siblings
rather than falling back.

---
## Group D — queue retry logging

(Same run as Group B: `TestQueueDelivery_TemporaryFail` with `-test.debuglog`.)

### D1 — the retry-scheduling line

**Direct answer.** Retry scheduling is shown by the `will retry` line, quoted verbatim:

```text
queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-597ns","rcpts":["tester1@example.org","tester2@example.org"]}
```

**Command:**

```
go test -count=1 -v -run 'TestQueueDelivery_TemporaryFail$' ./internal/target/queue/ -test.debuglog
```

(Full unedited output is the Group B capture above.)

**Source citation.** `internal/target/queue/queue.go:L415-L418`:

```go
dl.Msg("will retry",
	"attempts_count", meta.TriesCount,
	"next_try_delay", time.Until(nextTryTime),
	"rcpts", meta.To)
```

**Causal reasoning.** Emitted via `dl.Msg`, so it carries the `msg_id` field from the delivery
logger plus the three explicit fields. Alphabetical ordering gives `attempts_count`, `msg_id`,
`next_try_delay`, `rcpts`.

**Sibling variants.** The other terminal queue outcomes (`delivered`, `not delivered,
temporary error`, `not delivered, permanent error`) are enumerated in Group B (B2).

### D2 — the JSON field carrying the retry delay

**Direct answer.** The field is **`next_try_delay`**. It is a `time.Duration`, rendered via
`String()` (preamble §3).

**Command / output — stability across two runs.** The field **name and structure are stable**;
the **value varies** and is a tiny **negative** duration in tests:

- Run 1 → `next_try_delay":"-597ns"`
- Run 2 → `next_try_delay":"-585ns"`

**Source citation.** `internal/target/queue/queue.go:L413-L414` (the delay computation) and
`L417` (the field):

```go
nextTryTime := time.Now()
nextTryTime = nextTryTime.Add(q.initialRetryTime * time.Duration(math.Pow(q.retryTimeScale, float64(meta.TriesCount-1))))
```

**Causal reasoning — why the value is a tiny negative in tests.** The field is
`time.Until(nextTryTime)`, and `nextTryTime = time.Now().Add(initialRetryTime *
retryTimeScale^(tries-1))`. The queue test sets `q.initialRetryTime = 0`
(`internal/target/queue/queue_test.go:L50`), so `nextTryTime ≈ now`; by the time
`time.Until` is evaluated a few nanoseconds later, that elapsed time makes the remaining
duration slightly **negative**, and its exact nanosecond magnitude jitters per run
(`-597ns` vs `-585ns`). Only the value varies — the field name, type, and position are stable.

**Canonical production default (clearly separated).** In the **default** configuration the
first-retry delay would be **`15m0s`**, not a negative nanosecond value, because
`initialRetryTime` defaults to `15 * time.Minute` (`internal/target/queue/queue.go:L185`) and
`retryTimeScale` defaults to `2` (`L186`), with the delay formula
`initialRetryTime × retryTimeScale^(tries-1)` (`L414`). So `15m0s` is the canonical value a
developer will see in production for the first retry; the `-597ns` / `-585ns` values here are
**test-only artifacts** of `initialRetryTime = 0`.

---
## Key distinctions (recap)

### 1. Canonical vs. non-canonical `msg_id`

- **Canonical (production):** 8 lowercase hex characters, random, from `GenerateMsgID()`
  (`internal/msgpipeline/msgid.go:L12-L16`). OBSERVED in Group A: `e242088d`, `1f2e1254`,
  `04f1d745`, `3bebf265`, `310d2821` — a different value every run.
- **Non-canonical (test-harness artifact) — labeled as such wherever it appears:** the 40-hex
  IDs in the queue and remote tests are `hex(sha1(t.Name()))`, produced by the mock target
  `DoTestDeliveryErrMeta` (`internal/testutils/target.go:L239-L245`:
  `IDRaw := sha1.Sum([]byte(t.Name())); hex.EncodeToString(IDRaw[:])`). They are deterministic
  per test name (stable across runs) but do **not** represent production behaviour. This is
  externally verifiable — the observed IDs equal the SHA-1 of the test names:

```text
$ for t in TestQueueDelivery_TemporaryFail TestRemoteDelivery_TLSErrFallback TestRemoteDelivery_AuthMX_Fail; do printf "%s -> " "$t"; printf "%s" "$t" | sha1sum | cut -d" " -f1; done
TestQueueDelivery_TemporaryFail -> af8090c7eb39f761862b1f027b4f2b0bb1ce86d1
TestRemoteDelivery_TLSErrFallback -> 2176ec5872ed2b87d832b4070e88232bd94ac7d3
TestRemoteDelivery_AuthMX_Fail -> ac08d9f027f71627267fb3eae96f84d56762fa16
```

  Each hash matches the `msg_id` OBSERVED in the corresponding test
  (`af8090c7...` in Group B, `2176ec58...` and `ac08d9f0...` in Group C).

### 2. Two enhanced-code layers in Group C

| Layer | SMTP code | Enhanced code | RFC 3463 class/subject | Message |
|-------|-----------|---------------|------------------------|---------|
| Inner authenticity check | 550 | `5.7.0` | permanent / Security or Policy | `Failed to estabilish the MX record (mx.example.invalid.) authenticity` |
| Outer reply ("No usable MXs") | 550 | `5.4.0` | permanent / Network and Routing | `No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity` |

The inner `5.7.0` is hardcoded (`connect.go:L97`); the outer `5.4.0` is produced by
`SMTPEnchCode` forcing the class digit to `5` over the `{0,4,0}` fallback (`exterrors/smtp.go:L122-L128`).
The test prints the enhanced code as the slice `[5 4 0]` (Go `%v`); a real structured log renders it as
`5.4.0` via `EnhancedCode.FormatLog()` (`exterrors/smtp.go:L11-L13`).

### 3. Stable names/structure vs. run-varying values

| Stable (never changes) | Varying (changes per run) |
|------------------------|---------------------------|
| Module prefixes `smtp` / `queue` / `remote` | The specific 8-hex `msg_id` (e.g. `e242088d` → `1f2e1254`) |
| Field **names** and alphabetical ordering | `next_try_delay` nanosecond magnitude/sign (`-597ns` → `-585ns`) |
| `msg_id` **format** (8 hex chars, production) | — |
| `next_try_delay` **field name** and type | — |
| SHA-1-derived harness IDs (deterministic per test name) | — |

Both variable cases were confirmed across **two** runs: the Group A `msg_id` changed between
runs (random), while the Group B/D `next_try_delay` field name/structure stayed identical and
only the nanosecond value changed.

---

## Coverage & reproducibility

Every named item is answered above with the required six parts (direct answer, exact command,
complete unedited output, `file:line` citation, causal reasoning, sibling variants):

| Item | Answer (short) |
|------|----------------|
| **A1** | `smtp: RCPT ok` + `{"msg_id":"<8hex>","rcpt":"..."}` (2 fields, `msg_id` before `rcpt`) — `smtp.go:L243` |
| **A2** | success ends `accepted`; abort ends `aborted` (DATA abort preceded by `DATA error`) — `smtp.go:L334/L377`, `L72`, `L317/L360` |
| **A3** | module prefix `smtp` — `smtp.go:L495`; siblings `submission`/`lmtp` (`L492-L493`) |
| **A4** | 8 lowercase hex chars, random — `msgid.go:L12-L16` (canonical) |
| **B1** | acceptance → attempt#1 fail → `will retry` → attempt#2 → `delivered` → removed from disk — `queue.go:L278/L367/L384/L415/L378` |
| **B2** | `delivery attempt failed` (`L384`) and `will retry` (`L415-L418`) quoted verbatim |
| **C1** | `Failed to estabilish the MX record (mx.example.invalid.) authenticity` (misspelling verbatim) — `connect.go:L98` |
| **C2** | inner 550/`5.7.0` (`connect.go:L97`) and outer 550/`5.4.0` (`connect.go:L206`, `SMTPEnchCode`) — both reported |
| **C3** | `remote: TLS error, falling back to plaintext` + 4 fields (`domain`,`msg_id`,`mx`,`reason`) — `connect.go:L176-L177` |
| **D1** | the `will retry` line — `queue.go:L415-L418` |
| **D2** | field `next_try_delay` (`time.Duration`); test `-597ns`/`-585ns`, canonical default `15m0s` — `queue.go:L185/L414/L417` |

**All referenced tests passed** (each capture above shows `--- PASS`). Commands were run with
`-count=1` (no caching) under Go 1.13.15, `CGO_ENABLED=0`, at commit
`26452dd8dd787dc455278b0fdd296f4a5432c768`.

**Read-only invariant honoured.** No repository source file was modified; only this document
was added under `blitzy/documentation/`. All Go module/build caches live outside the repo tree
and every temporary observation script was kept outside the repo and deleted, so the working
tree remains git-clean apart from this file.

**Reference.** Enhanced status-code semantics per
[RFC 3463 — Enhanced Mail System Status Codes](https://datatracker.ietf.org/doc/html/rfc3463).
