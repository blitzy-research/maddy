# maddy — Runtime Logging & Delivery Behavior Q&A

**Repository:** `github.com/foxcpp/maddy`
**Commit:** `26452dd8dd787dc455278b0fdd296f4a5432c768`
**Toolchain:** Go 1.13 (verified with `go1.13.15`)

This document answers four clusters of questions (R1–R4) about how `maddy` logs and
behaves at runtime. **Every answer is derived from actual `go test -v` output** captured
by building and running the project's own test suite at the commit above, and is then
**corroborated against the emitting source code**. Each answer is followed by an explicit
**rationale** that reconciles the observed output against the source line that emits it and
against `maddy`'s logging rules.

> **Source of truth.** The questions are about *runtime behavior*, so the primary evidence
> is the test output. The source code is treated as the authoritative explanation for *why*
> each line looks the way it does. Strings are quoted **verbatim** — including the upstream
> misspelling **"estabilish"** (it is reproduced exactly, never "corrected").

---

## 1. Methodology / Reproduction

All evidence was produced inside the project's build environment with the **Go 1.13**
toolchain (per `go.mod`: `go 1.13`). The canonical build/test commands are declared in
`.build.yml`: `go build ./...` [.build.yml:11] and `go test ./... -cover -race`
[.build.yml:14]. The investigation narrows these to one targeted `-run` selector per
cluster.

**Build first** (confirms the module cache is intact and all packages compile):

```
go build ./...
```

**Per-cluster test commands (verified):**

Each command is shown in its own fenced block so the literal `|` regex alternation is
copy-paste-safe (inside a Markdown table the pipes would have to be backslash-escaped, and
those escaped pipes select no tests when copied verbatim from the raw source).

*R1 — SMTP endpoint logging:*

```
go test -v -run 'TestSMTPDelivery$|TestSMTPDelivery_AbortData|TestSMTPDelivery_AbortLogout' ./internal/endpoint/smtp/
```

*R2 / R4 — Queue trace & retry:*

```
go test -v -run 'TestQueueDelivery_TemporaryFail$' ./internal/target/queue/ -test.debuglog
```

*R3 — Remote MX-auth & TLS fallback:*

```
go test -v -run 'TestRemoteDelivery_AuthMX_Fail$|TestRemoteDelivery_TLSErrFallback$' ./internal/target/remote/
```

**Why `-v` is mandatory.** The test logger `testutils.Logger(t, name)` routes every log
line to `t.Log` [internal/testutils/logger.go:28-34]. Go only prints `t.Log` output when
tests run in verbose mode, so **without `-v` the structured log lines are invisible**.

**Why `-test.debuglog` is mandatory for R2.** The `delivery attempt #N` line is emitted via
`Debugf` and is suppressed unless debug logging is enabled
[internal/target/queue/queue.go:367], [internal/testutils/logger.go:14]. The test logger
sets `Debug: *debugLog` [internal/testutils/logger.go:39] from the `-test.debuglog` flag
[internal/testutils/logger.go:14], and prefixes debug lines with `"[debug] "`
[internal/testutils/logger.go:31-32].

> **Reproduction caveat (a real gotcha).** With this Go 1.13 toolchain, placing the custom
> `-test.debuglog` flag **before** the package path
> (e.g. `go test -v -test.debuglog -run X ./internal/target/queue/`) silently routes the run
> to the root module and prints `[no test files]` — producing **zero** log output. The flag
> MUST be placed **after** the package path (`... ./internal/target/queue/ -test.debuglog`),
> or passed via `-args -test.debuglog`, or you can `cd` into the package directory first.

> **Per-run variation.** Four things legitimately differ between runs and are **not** part
> of the stable answer: (1) the SMTP `msg_id` is a fresh random 8-char hex value each run;
> (2) the queue `next_try_delay` is a near-zero, possibly slightly-negative duration (a
> timing artifact of the test's zero retry interval); (3) the precise *interleaving* of the
> concurrency-related `[debug]` semaphore lines (`waiting on delivery semaphore` /
> `delivery semaphore acquired`) relative to surrounding output could in principle shift with
> goroutine scheduling — though their **presence** is deterministic, as is the unconditional
> `failures: …` summary line emitted after every delivery attempt
> [internal/target/queue/queue.go:370-371]; and (4) the **relative order of the two
> `delivery attempt failed` lines**, which are emitted by ranging over a Go map
> [internal/target/queue/queue.go:383] and therefore may swap between runs (see R2). Each
> required answer line's **content and field set is deterministic** — only the relative order
> of those two per-recipient failure lines is not. The verbatim
> blocks below are the captured ground truth; an independent re-run confirmed every required
> line (see the closing "Reproduction confirmation" note).

---

## 2. Logging mechanics primer

Every `maddy` log line renders as:

```
module: message	{json}
```

(a literal **TAB** separates the message from the JSON object). The behavior is governed by
three small pieces of the logging subsystem; all four answers below rely on these rules:

- **Module-name prefix.** `Logger.log` prepends `Name + ": "` to the message when the logger
  has a name [internal/log/log.go:181-182]. So a logger named `smtp` produces lines starting
  with `smtp: `.
- **The TAB is always present.** `Logger.formatMsg` writes the message and then
  unconditionally writes a TAB [internal/log/log.go:135-139]; the JSON object is appended
  **only if there is at least one field**. A line with no fields therefore consists of the
  message followed by that lone TAB and no `{...}` block (this otherwise-invisible trailing
  TAB is trimmed from the transcripts below for cleanliness, per the repository's
  `.editorconfig`).
- **Keys are sorted ALPHABETICALLY.** The ordered-JSON marshaller collects the field keys and
  calls `sort.Strings` before writing them [internal/log/orderedjson.go:18-23]. Every JSON
  block below is shown in alphabetical key order to mirror the real output.
- **Value normalization** [internal/log/orderedjson.go:42-50]: `time.Time` →
  `"2006-01-02T15:04:05.000"`; `time.Duration` → `.String()` (e.g. `"15m0s"`, `"0s"`,
  `"-806ns"`); a `fmt.Stringer` → `.String()`; an `error` → `.Error()`. This is why
  `next_try_delay` (a `time.Duration`) is rendered as a **string**.
- **`Logger.Error(msg, err, ...)`** merges the structured fields carried by the error
  (`exterrors.Fields(err)`), then adds `reason = err.Error()` **iff** a `reason` field is not
  already present, then appends the explicit key/value pairs [internal/log/log.go:89-103].
  This is the source of the `reason` field on `delivery attempt failed` and on the TLS
  fallback line.
- **`DeliveryLogger` injects `msg_id`.** Per-delivery loggers are wrapped by
  `target.DeliveryLogger`, which copies the base fields and sets
  `Fields["msg_id"] = msgMeta.ID` [internal/target/delivery.go:8-16]. Because `formatMsg`
  merges a logger's base `Fields` into every line [internal/log/log.go:144-147], **queue and
  remote per-delivery lines always carry `msg_id`**. By contrast, the **SMTP endpoint logger
  has a `Name` only and no base `Fields`** [internal/endpoint/smtp/smtp.go:495], so its lines
  carry **only** the fields passed explicitly to `Msg(...)`.

**`msg_id` format.** `GenerateMsgID()` reads 4 random bytes and returns
`hex.EncodeToString(...)` [internal/msgpipeline/msgid.go:12-15] — i.e. an **8-character
lowercase hexadecimal** string. That is the production format.

> **8-char vs 40-char `msg_id` nuance.** In the **SMTP endpoint** tests the `msg_id` is the
> real runtime 8-char value (e.g. `f9a9cdba`). In the **queue** and **remote** tests the
> `msg_id` is a **40-character** hex string because the test harness sets
> `msgMeta.ID = hex(sha1(t.Name()))` for determinism — e.g.
> `sha1("TestQueueDelivery_TemporaryFail")` = `af8090c7eb39f761862b1f027b4f2b0bb1ce86d1` and
> `sha1("TestRemoteDelivery_TLSErrFallback")` = `2176ec5872ed2b87d832b4070e88232bd94ac7d3`.
> The **production format remains 8-char hex**; the 40-char values are purely a test-harness
> artifact.

---

## R1 — SMTP endpoint logging

**Questions.** What is the exact log line when a recipient is successfully added (list ALL
JSON fields)? How does it compare to the line logged when a delivery is aborted
mid-transaction? Which module name appears in the SMTP log prefix, and what is the exact
`msg_id` format?

### Captured output (verbatim)

From `TestSMTPDelivery` [internal/endpoint/smtp/smtp_test.go:122] — a successful
two-recipient delivery:

```
smtp: incoming message	{"msg_id":"f9a9cdba","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:53814"}
smtp: RCPT ok	{"msg_id":"f9a9cdba","rcpt":"rcpt1@example.com"}
smtp: RCPT ok	{"msg_id":"f9a9cdba","rcpt":"rcpt2@example.com"}
smtp: accepted	{"msg_id":"f9a9cdba"}
```

From `TestSMTPDelivery_AbortData` [internal/endpoint/smtp/smtp_test.go:360] — delivery
aborted mid-`DATA`:

```
smtp: RCPT ok	{"msg_id":"245f622b","rcpt":"test@example.com"}
smtp: DATA error	{"msg_id":"245f622b","reason":"unexpected EOF"}
smtp: aborted	{"msg_id":"245f622b"}
```

From `TestSMTPDelivery_AbortLogout` [internal/endpoint/smtp/smtp_test.go:398] — abort via
early logout:

```
smtp: RCPT ok	{"msg_id":"32349475","rcpt":"test@example.com"}
smtp: aborted	{"msg_id":"32349475"}
```

### Answers

- **Exact "recipient added" line:**

  ```
  smtp: RCPT ok	{"msg_id":"<8-hex>","rcpt":"<recipient>"}
  ```

  **ALL JSON fields = exactly two: `msg_id` and `rcpt`** (alphabetical). It is emitted by
  `s.endp.Log.Msg("RCPT ok", "rcpt", to, "msg_id", s.msgMeta.ID)`
  [internal/endpoint/smtp/smtp.go:243].

- **Aborted-mid-transaction line:**

  ```
  smtp: aborted	{"msg_id":"<8-hex>"}
  ```

  **Only ONE field: `msg_id`.** It is emitted by
  `s.log.Msg("aborted", "msg_id", s.msgMeta.ID)` [internal/endpoint/smtp/smtp.go:72], and
  `s.log` is the same endpoint logger (`s.log = endp.Log`
  [internal/endpoint/smtp/smtp.go:677]).

- **Comparison.** Both lines share `msg_id`. `RCPT ok` additionally carries `rcpt` — the
  recipient address just accepted — whereas `aborted` carries **no** `rcpt`, because an abort
  applies to the **whole transaction**, not to a single recipient. Neither line carries any
  extra base fields, because the SMTP endpoint logger has a `Name` only and **no** base
  `Fields` [internal/endpoint/smtp/smtp.go:495]. The `accepted` line
  [internal/endpoint/smtp/smtp.go:334] likewise carries only `msg_id`, confirming the same
  endpoint-logger pattern.

- **Module name in the prefix = `smtp`.** The endpoint is registered as
  `module.RegisterEndpoint("smtp", New)` [internal/endpoint/smtp/smtp.go:715], the logger is
  created with `Name: modName` [internal/endpoint/smtp/smtp.go:495], and the test wires it as
  `endp.Log = testutils.Logger(t, "smtp")` [internal/endpoint/smtp/smtp_test.go:49]. Hence
  every line is prefixed `smtp: `.

- **`msg_id` format = 8-character lowercase hexadecimal** — the hex encoding of 4 random
  bytes [internal/msgpipeline/msgid.go:12-15]. Observed example values: `f9a9cdba`,
  `245f622b`, `32349475`.

### Rationale

The field set on each line is exactly the list of key/value pairs passed to `Msg(...)`, and
nothing more: the endpoint logger has no base `Fields`, so there is no `msg_id` injection or
any other automatic field here — `msg_id` appears only because it is passed explicitly. The
keys are printed alphabetically by `marshalOrderedJSON` (`msg_id` sorts before `rcpt`)
[internal/log/orderedjson.go:18-23], which is why even though the source passes `rcpt`
*before* `msg_id`, the rendered line shows `msg_id` first. The `aborted` line proves the
"whole-transaction" semantics: it deliberately omits `rcpt` because at abort time the failure
is not attributable to one recipient.

---

## R2 — Queue delivery trace (acceptance → failure → retry)

**Question.** What is the complete, ordered sequence of log messages from initial acceptance
through retry? Quote the exact lines for the delivery attempt, the failure, and the retry
scheduling.

### Captured output (verbatim)

`go test -v -run 'TestQueueDelivery_TemporaryFail$' ./internal/target/queue/ -test.debuglog`
[internal/target/queue/queue_test.go:297] — the first attempt temporarily fails both
recipients, then the retry succeeds:

```
[debug] queue: delivery target: *queue.unreliableTarget
[debug] queue: starting delivery for af8090c7eb39f761862b1f027b4f2b0bb1ce86d1
[debug] queue: waiting on delivery semaphore for af8090c7eb39f761862b1f027b4f2b0bb1ce86d1
[debug] queue: delivery semaphore acquired for af8090c7eb39f761862b1f027b4f2b0bb1ce86d1
[debug] queue: delivery attempt #1	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
[debug] queue: using message ID = af8090c7eb39f761862b1f027b4f2b0bb1ce86d1-1	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
[debug] queue: target.Start OK	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
[debug] queue: delivery.AddRcpt tester1@example.org OK	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
[debug] queue: delivery.AddRcpt tester2@example.org OK	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
[debug] queue: delivery.Body failed: you shall not pass	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
[debug] queue: delivery.Body OK	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
[debug] queue: delivery.Abort (all recipients failed)	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
[debug] queue: failures: permanently: [], temporary: [tester1@example.org tester2@example.org], errors: map[tester1@example.org:you shall not pass tester2@example.org:you shall not pass]	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
queue: delivery attempt failed	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org","reason":"you shall not pass"}
queue: delivery attempt failed	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org","reason":"you shall not pass"}
queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-1.132µs","rcpts":["tester1@example.org","tester2@example.org"]}
[debug] queue: starting delivery for af8090c7eb39f761862b1f027b4f2b0bb1ce86d1
[debug] queue: waiting on delivery semaphore for af8090c7eb39f761862b1f027b4f2b0bb1ce86d1
[debug] queue: delivery semaphore acquired for af8090c7eb39f761862b1f027b4f2b0bb1ce86d1
[debug] queue: delivery attempt #2	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
[debug] queue: using message ID = af8090c7eb39f761862b1f027b4f2b0bb1ce86d1-2	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
[debug] queue: target.Start OK	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
[debug] queue: delivery.AddRcpt tester1@example.org OK	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
[debug] queue: delivery.AddRcpt tester2@example.org OK	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
[debug] queue: delivery.Body OK	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
[debug] queue: delivery.Commit OK	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
[debug] queue: failures: permanently: [], temporary: [], errors: map[]	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org"}
queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org"}
[debug] queue: removed message from disk	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
```

> **Ordering caveat (the two per-recipient failure lines).** The two `delivery attempt
> failed` lines above are emitted by ranging over a Go **map**
> (`for rcpt, rcptErr := range partialErr.Errs`) [internal/target/queue/queue.go:383], and Go
> randomizes map-iteration order — so their **relative order is not guaranteed** and may swap
> from run to run. The block above is one real captured run (it happens to show
> `tester1@example.org` before `tester2@example.org`); other runs emit the same two lines in
> the opposite order, and both orders were observed across repeated runs at this commit. What
> **is** stable is: (a) the higher-level sequence — `delivery attempt #1` → the `failures: …`
> summary → the two `delivery attempt failed` lines (in *either* order) → `will retry`;
> (b) each failure line's message and field set (`msg_id`, `rcpt`, `reason`); and (c) the
> `failures: …` summary line's own key ordering, which stays alphabetical because Go's `fmt`
> sorts map keys when rendering `%v` [internal/target/queue/queue.go:370-371]. Only the raw
> `range` at [internal/target/queue/queue.go:383] is unordered.

### The three required lines

- **Delivery attempt (debug-gated):**

  ```
  [debug] queue: delivery attempt #1	{"msg_id":"<id>"}
  ```

  Emitted by `dl.Debugf("delivery attempt #%d", meta.TriesCount+1)`
  [internal/target/queue/queue.go:367]. It is **only** visible with `-test.debuglog`; the
  `[debug] ` prefix is added by the test logger [internal/testutils/logger.go:31-32].

- **Failure (logged once per failed recipient):**

  ```
  queue: delivery attempt failed	{"msg_id":"<id>","rcpt":"<addr>","reason":"you shall not pass"}
  ```

  Emitted by `dl.Error("delivery attempt failed", rcptErr, "rcpt", rcpt)`
  [internal/target/queue/queue.go:384]. For this test the error is
  `exterrors.WithTemporary(errors.New("you shall not pass"), true)`, which carries **no**
  structured fields, so the field set is **exactly** `msg_id`, `rcpt`, `reason` — there are no
  error-specific extras (e.g. no `smtp_code`). `reason` is injected by `Error()` from
  `err.Error()` [internal/log/log.go:96-99]. Both recipients fail, so this line appears
  **exactly twice** — once for `tester1@example.org` and once for `tester2@example.org`, with
  an **identical field set** on each. Their **relative order is not deterministic**: the loop
  ranges over a Go map (`for rcpt, rcptErr := range partialErr.Errs`)
  [internal/target/queue/queue.go:383], and Go randomizes map iteration, so the two lines may
  appear in either order from one run to the next (both orders were observed across repeated
  runs at this commit).

- **Retry scheduling:**

  ```
  queue: will retry	{"attempts_count":1,"msg_id":"<id>","next_try_delay":"<duration>","rcpts":["tester1@example.org","tester2@example.org"]}
  ```

  Emitted by
  `dl.Msg("will retry", "attempts_count", meta.TriesCount, "next_try_delay", time.Until(nextTryTime), "rcpts", meta.To)`
  [internal/target/queue/queue.go:415-418]. There are **FOUR** fields (alphabetical):
  `attempts_count`, `msg_id`, `next_try_delay`, `rcpts`. Note that `rcpts` is a **JSON array**
  of recipient strings — it is present and must not be omitted.

### Rationale

The ordered sequence shows the full lifecycle: the message enters delivery, an attempt is
made (`delivery attempt #1`), the synthetic target fails the body for both recipients
(`delivery.Body failed: you shall not pass`), the transaction is aborted, each recipient's
failure is logged (`delivery attempt failed` ×2), and the message is rescheduled
(`will retry`). All the `[debug] ...` lines — including `delivery attempt #1` itself — are
emitted via `Debugf`/`DebugMsg` and are therefore visible **only** under `-test.debuglog`;
the three lines without the `[debug]` prefix (`delivery attempt failed`, `will retry`,
`delivered`) are normal `Msg`/`Error` lines that appear whenever `-v` is on. Every line
carries `msg_id` because the queue uses a `DeliveryLogger`
[internal/target/delivery.go:8-16]. `attempts_count` and (on the success tail) `attempt` are
**numbers**, not strings, because they are passed as integers and `orderedjson` does not
quote numeric values. The successful retry tail logs `delivered` with `attempt: 2`
[internal/target/queue/queue.go:378] and finally `removed message from disk`.

Two debug lines on the success path are **source-deterministic**, not scheduling artifacts, and
are an integral part of the stable ordered sequence above. First, immediately after each
delivery attempt returns, `tryDelivery` **unconditionally** logs a failures summary via
`dl.Debugf("failures: permanently: %v, temporary: %v, errors: %v", …)`
[internal/target/queue/queue.go:370-371]: on the first (failed) attempt it lists both
temporarily-failed recipients and their errors, and on the second (successful) attempt it shows
empty containers — `failures: permanently: [], temporary: [], errors: map[]`. Because it is
emitted on **every** attempt with no guard, this line always appears — right after the attempt
completes and before the per-recipient `delivered` / `delivery attempt failed` lines — whenever
`-test.debuglog` is on. Second, the semaphore lines (`waiting on delivery semaphore`,
`delivery semaphore acquired`) are emitted on every dispatch
[internal/target/queue/queue.go:282, :299], so their **presence** on both attempts is
deterministic too; only their precise *interleaving* with other concurrent goroutine output
could in principle shift under different goroutine scheduling.

One deliberate exception to "identical reproduction" must be called out: the **relative order
of the two `delivery attempt failed` lines is not stable**. They are emitted by ranging over a
Go map (`for rcpt, rcptErr := range partialErr.Errs`)
[internal/target/queue/queue.go:383], and Go randomizes map iteration, so across runs the two
lines may appear as `tester1` → `tester2` or `tester2` → `tester1` (both orders were observed
at this commit). Everything else reproduces identically: each required line's message and
field set, the higher-level sequence (`delivery attempt #1` → `failures: …` summary → the two
`delivery attempt failed` lines, in either order → `will retry`), and — notably — the
`failures: …` summary line's own key ordering, which stays alphabetical because Go's `fmt`
sorts map keys when rendering `%v` [internal/target/queue/queue.go:370-371] (unlike the raw
`range` at [internal/target/queue/queue.go:383]). In short, the *stable required lines and
their field sets* reproduce identically; only the *relative order of the two per-recipient
failure lines* is non-deterministic.

---


## R3 — Remote delivery: MX-authenticity failure & TLS→plaintext fallback

**Questions.** What is the exact error string when MX authenticity checks fail? What is the
precise SMTP enhanced status code (X.Y.Z) and the complete reply text? What is the exact log
message when the system falls back from TLS to plaintext (list ALL JSON fields)?

### Captured output (verbatim)

TLS→plaintext fallback, from `TestRemoteDelivery_TLSErrFallback`
[internal/target/remote/remote_test.go:852]:

```
remote: TLS error, falling back to plaintext	{"domain":"example.invalid","msg_id":"2176ec5872ed2b87d832b4070e88232bd94ac7d3","mx":"mx.example.invalid.","reason":"smtpconn: x509: certificate signed by unknown authority"}
```

MX-authenticity failure, from `TestRemoteDelivery_AuthMX_Fail`
[internal/target/remote/mxauth_test.go:16] — the fields carried by the **surfaced** delivery
error (as dumped by the test harness):

```
domain:example.invalid
reason:Failed to estabilish the MX record (mx.example.invalid.) authenticity
smtp_code:550
smtp_enchcode:[5 4 0]
smtp_msg:No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity
target:remote
```

The returned error is a chain of **two** `SMTPError` layers (the inner one is the actual
MX-authenticity rejection; the outer one is the aggregate that the caller ultimately sees):

```
INNER  (the actual MX-authenticity rejection): Code=550  EnhancedCode=5.7.0  Message="Failed to estabilish the MX record (mx.example.invalid.) authenticity"
OUTER  (the aggregate wrapper that is surfaced): Code=550  EnhancedCode=5.4.0  Message="No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity"
```

### Answers

- **Exact MX-authenticity error string** (reproduced verbatim, including the upstream
  misspelling **"estabilish"**):

  ```
  Failed to estabilish the MX record (mx.example.invalid.) authenticity
  ```

  Source: `fmt.Sprintf("Failed to estabilish the MX record (%s) authenticity", mx)`
  [internal/target/remote/connect.go:98]. With the mock DNS fixture the `%s` expands to
  `mx.example.invalid.` (note the trailing dot — it is a fully-qualified DNS name).

- **SMTP enhanced status code & complete reply text.** The MX-authenticity rejection itself
  is constructed with SMTP code **`550`** [internal/target/remote/connect.go:96] and enhanced
  code **`5.7.0`** [internal/target/remote/connect.go:97]. Its complete reply text is:

  ```
  550 5.7.0 Failed to estabilish the MX record (mx.example.invalid.) authenticity
  ```

  **Important nuance (documented honestly).** That `550 5.7.0` is the **inner** error raised
  at the MX-authenticity check [internal/target/remote/connect.go:94-99]. Because **all**
  candidate MXs fail, the remote target wraps the last error in an aggregate
  `"No usable MXs, last err: ..."` `SMTPError` [internal/target/remote/connect.go:204-209].
  That aggregate is built with `EnhancedCode: exterrors.SMTPEnchCode(err, exterrors.EnhancedCode{0, 4, 0})`
  [internal/target/remote/connect.go:206]; the helper forces the class digit to `5`, yielding
  the **surfaced** enhanced code **`5.4.0`** (visible above as `smtp_enchcode:[5 4 0]`), while
  the original `5.7.0` is preserved deeper in the chain and the "estabilish" text survives as
  the `reason` / `smtp_msg`. In short:
  **the MX-authenticity rejection is `550 5.7.0`; the aggregated delivery failure the caller
  ultimately sees is `550 5.4.0`, carrying the same "estabilish" message as its reason.**

- **TLS→plaintext fallback log:**

  ```
  remote: TLS error, falling back to plaintext	{"domain":"<domain>","msg_id":"<id>","mx":"<mx host>","reason":"<TLS verification error>"}
  ```

  **ALL JSON fields = exactly four: `domain`, `msg_id`, `mx`, `reason`** (alphabetical).
  Source: `rd.Log.Error("TLS error, falling back to plaintext", err, "mx", record.Host, "domain", domain)`
  [internal/target/remote/connect.go:176-177]. Here `msg_id` is the injected base field
  (the remote target uses a `DeliveryLogger` [internal/target/delivery.go:8-16]); `reason` is
  added by `Error()` from the TLS error's text [internal/log/log.go:96-99]; `mx` is
  `record.Host`; and `domain` is the recipient domain. In the captured run the `reason` is
  `smtpconn: x509: certificate signed by unknown authority` — the test connects to a STARTTLS
  server presenting an untrusted certificate. This branch fires **only** when the error is an
  `smtpconn.TLSError` **and** MX authentication is not required (`authErr == nil`)
  [internal/target/remote/connect.go:175].

### RFC 3463 — what the enhanced codes mean

Enhanced status codes use the structure **`class.subject.detail`** (X.Y.Z), defined by
**RFC 3463 — Enhanced Mail System Status Codes**
(https://datatracker.ietf.org/doc/html/rfc3463):

- **Class `5` = permanent failure** (not likely to be resolved by resending as-is). The class
  digit must match the first digit of the basic SMTP code — consistent with the `550` here.
- **Subject `7` = Security or Policy Status.** Detail **`0`** (X.7.0) = *"Other or undefined
  security status"* — something related to security caused the message to be returned and the
  condition cannot be expressed by a more specific code (it may also be used when the
  condition cannot be further described due to security policies in force).

Therefore **`5.7.0` = a permanent rejection on security/policy grounds**, which is exactly
right for a failed MX-authenticity check (DNSSEC / MTA-STS). For contrast, the surfaced
aggregate **`5.4.0`** is class `5` (permanent), subject `4` = *"Network and Routing Status"*,
detail `0` = *"Other or undefined network or routing status"* — a sensible classification for
the "No usable MXs" aggregation that abandons delivery after exhausting all MX candidates.

### Rationale

The two-layer error chain explains the apparent discrepancy between the security code
(`5.7.0`) and the surfaced code (`5.4.0`): the inner error is the precise, per-MX security
verdict, while the outer error is the routing-level summary returned after every MX has been
tried and rejected. Both layers share SMTP code `550` and the same "estabilish" message
(carried forward as the `reason`). For the TLS-fallback line, the field set and alphabetical
ordering follow directly from the single `Error(...)` call plus the injected `msg_id`; the
`reason` field is the `smtpconn.TLSError`'s text, and the line is emitted precisely on the
fallback branch that downgrades to plaintext when TLS fails but MX authentication does not
demand TLS.

---

## R4 — Queue retry: the delay field

**Question.** Quote the log line showing retry scheduling and identify the exact JSON field
name that contains the retry-delay value.

### Answer

- The retry-scheduling line is **`queue: will retry`** (quoted in full under R2):

  ```
  queue: will retry	{"attempts_count":1,"msg_id":"<id>","next_try_delay":"<duration>","rcpts":["tester1@example.org","tester2@example.org"]}
  ```

- The JSON field that contains the retry delay is **`next_try_delay`**
  [internal/target/queue/queue.go:417]. Its companion field is **`attempts_count`** (a
  numeric value) [internal/target/queue/queue.go:416].

- **Type & value.** `next_try_delay` is a Go `time.Duration` rendered as a **string** by the
  log formatter [internal/log/orderedjson.go:42-43] (e.g. `"15m0s"`, `"0s"`). In the captured
  test run the value was **`"-1.132µs"`** (slightly negative). This near-zero/negative value
  is a deliberate **test artifact**, not the canonical production value — see below.

### Rationale (the value is configuration-dependent — there is no single "the" number)

The line logs `time.Until(nextTryTime)`, where
`nextTryTime = time.Now().Add(initialRetryTime × retryTimeScale^(TriesCount-1))`
[internal/target/queue/queue.go:414-417]. The backoff formula is documented in-source as
`initialRetryTime * retryTimeScale ^ (TriesCount - 1)`
[internal/target/queue/queue.go:121-122].

- **Production defaults:** `initialRetryTime = 15 * time.Minute` and `retryTimeScale = 2`
  [internal/target/queue/queue.go:185-186]. So the first retry delay is
  `15m × 2^0 ≈ "15m0s"`, the second `30m`, the third `1h`, and so on (exponential backoff).
- **Test-harness override:** `TestQueueDelivery_TemporaryFail` sets `initialRetryTime = 0` and
  `retryTimeScale = 1` [internal/target/queue/queue_test.go:50-51]. With a zero base delay,
  `nextTryTime ≈ time.Now()`, and `time.Until(...)` — evaluated a few microseconds *later* —
  comes out marginally **negative** (hence the captured `"-1.132µs"`). The test deliberately
  uses a near-zero interval so the retry fires immediately rather than waiting minutes.

So the **field name is fixed (`next_try_delay`)**, but its **value is configuration-dependent**:
a non-trivial positive duration in production (`"15m0s"`, `"30m0s"`, …) and a near-zero,
possibly slightly-negative duration under the test's zero-delay override.

---

## Source references

| Fact | Location |
|------|----------|
| `RCPT ok` line (fields `rcpt`, `msg_id`) | internal/endpoint/smtp/smtp.go:243 |
| `aborted` line (field `msg_id`) | internal/endpoint/smtp/smtp.go:72 |
| `accepted` line (field `msg_id`) | internal/endpoint/smtp/smtp.go:334, :377 |
| Endpoint logger has `Name` only, no base `Fields` | internal/endpoint/smtp/smtp.go:495 |
| `s.log == endp.Log` | internal/endpoint/smtp/smtp.go:677 |
| Module name `smtp` registered | internal/endpoint/smtp/smtp.go:715 |
| Test logger wiring `Logger(t, "smtp")` | internal/endpoint/smtp/smtp_test.go:49 |
| `msg_id` = 8-char lowercase hex (4 random bytes) | internal/msgpipeline/msgid.go:12-15 |
| `delivery attempt #N` (debug-gated) | internal/target/queue/queue.go:367 |
| `delivered` (field `attempt`) | internal/target/queue/queue.go:378 |
| `delivery attempt failed` (via `Error`) | internal/target/queue/queue.go:384 |
| Per-recipient failure lines emitted by `range` over a map (relative order **not** guaranteed) | internal/target/queue/queue.go:383 |
| `will retry` (`attempts_count`, `next_try_delay`, `rcpts`) | internal/target/queue/queue.go:415-418 |
| Backoff formula + defaults (`15m`, `2`) | internal/target/queue/queue.go:121-122, :185-186 |
| Test retry override (`0`, `1`) | internal/target/queue/queue_test.go:50-51 |
| MX-auth rejection `550 5.7.0` + "estabilish" string | internal/target/remote/connect.go:94-99 |
| Aggregate `No usable MXs` → surfaced `5.4.0` | internal/target/remote/connect.go:204-209 |
| TLS→plaintext fallback log line | internal/target/remote/connect.go:175-177 |
| Module name `remote`; `target:remote` field | internal/target/remote/remote.go:83, :45 |
| Module-name prefix `Name + ": "` | internal/log/log.go:181-182 |
| TAB always written after message | internal/log/log.go:135-139 |
| `Error()` injects `reason = err.Error()` | internal/log/log.go:89-103 |
| Alphabetical key sort (`sort.Strings`) | internal/log/orderedjson.go:18-23 |
| `time.Duration` → string normalization | internal/log/orderedjson.go:42-43 |
| `DeliveryLogger` injects `msg_id` | internal/target/delivery.go:8-16 |
| `t.Log` forwarding; `-test.debuglog`; `[debug]` prefix | internal/testutils/logger.go:14, :28-34 |
| Canonical build/test commands | .build.yml:11, :14 |

---

## Reproduction confirmation

The verbatim blocks above are the captured ground truth. An independent re-run at the same
commit (Go 1.13.15) reproduced **every required line** identically, with only the expected
per-run variation:

- **R1** — same line structure and field sets; the random 8-char `msg_id` values differed
  (e.g. `3a5eda68`, `2b4c0bf4`, `7ffcbadb`), confirming the 8-char lowercase-hex format.
- **R2 / R4** — identical deterministic 40-char `msg_id`
  (`af8090c7eb39f761862b1f027b4f2b0bb1ce86d1`); every **required line and field set** above
  reproduced identically, including the second-attempt semaphore lines and the unconditional
  `failures: permanently: [], temporary: [], errors: map[]` summary line
  [internal/target/queue/queue.go:370-371]. Two details are legitimately run-dependent and are
  **not** asserted as a fixed value: (i) the `next_try_delay` timing artifact, which came out
  as a different near-zero negative value (`"-806ns"` vs the captured `"-1.132µs"`); and
  (ii) the **relative order of the two `delivery attempt failed` lines**, which is emitted by
  ranging over a Go map [internal/target/queue/queue.go:383] — across repeated `-count=1`
  re-runs at this commit the pair appeared in **both** orders (`tester1` → `tester2` and
  `tester2` → `tester1`), confirming the order is not a stable contract. The higher-level
  sequence (`delivery attempt #1` → `failures: …` summary → the two failure lines in either
  order → `will retry`) and each line's field set are stable.
- **R3** — the TLS-fallback line and the MX-authenticity error fields (including the verbatim
  "estabilish" message and `smtp_enchcode:[5 4 0]`) matched exactly, with the same
  deterministic 40-char `msg_id` (`2176ec5872ed2b87d832b4070e88232bd94ac7d3`).

No `maddy` source, test, configuration, or build file was modified at any point; the only new
artifact is this document.

