# maddy Runtime Behavior — Evidence-Grounded Q&A

This document answers a four-part runtime-behavior question set about the
[`maddy`](../../README.md) mail server, captured by **actually running the project's
own tests** and quoting their real output. It targets the repository at commit
`26452dd` ("target/remote: Rewrite connection part to allow more concurrency",
branch `maddy_26452dd8dd78`). Every quoted log line, status code, error string,
JSON field name, and identifier below was observed in an executed test run, and
every factual claim is paired with a `file:line` citation into the source tree.

> **Scope note.** This is a strictly read-only investigation. No existing
> repository file was modified; the only artifact produced is this document.
> Two source quirks are reported **exactly as observed and deliberately left
> uncorrected**: the misspelling `estabilish` in the MX-authenticity message
> [internal/target/remote/connect.go:L98], and the unconditional `code[0] = 5`
> class-forcing in `SMTPEnchCode` [internal/exterrors/smtp.go:L126].

## 1. Methodology and toolchain

**Toolchain actually used:** `go version go1.26.4 linux/amd64` (GOROOT
`/tmp/go1264/go`, installed outside the repository working tree). All Go
build/module/temp caches were likewise kept **outside** the repository
(`GOCACHE=/tmp/go-cache-1264`, `GOTMPDIR=/tmp`, `GOMODCACHE=/root/go/pkg/mod`,
`GOFLAGS=-mod=readonly`, `GOTOOLCHAIN=local`) so the working tree stays
byte-for-byte unchanged. Every value quoted below was observed in a test run
executed under this toolchain.

**Verified invocation pattern.** maddy's test logger routes messages to Go's
`t.Log`, which the `go test` runner only prints in verbose mode; debug-level
lines require maddy's custom `-test.debuglog` flag. The flag is registered by
the test-utility package, so it **must appear _after_ the package path** —
placing it before the package path makes `go test` treat it as a flag for the
root module invocation and report `[no test files]`:

```
go test -v -count=1 -run <Name> ./<pkg>/ -test.debuglog
```

- `-v` surfaces `t.Log` output (the test logger writes through `t.Log`)
  [internal/testutils/logger.go:L34].
- `-test.debuglog` turns on debug-level lines such as `delivery attempt #N`
  [internal/testutils/logger.go:L14,L18-L41].
- `-count=1` disables test result caching; every run is non-interactive.

**Exact commands used per question group:**

```
# Q1 — SMTP endpoint (success + both abort variants)
go test -v -count=1 -run 'TestSMTPDelivery' ./internal/endpoint/smtp/ -test.debuglog

# Q2 / Q4 — queue delivery lifecycle and retry scheduling
go test -v -count=1 -run '^TestQueueDelivery_TemporaryFail$' ./internal/target/queue/ -test.debuglog
go test -v -count=1 -run 'TestQueueDelivery_MultipleAttempts' ./internal/target/queue/ -test.debuglog

# Q3 — remote MX-authentication failure + TLS-to-plaintext fallback
go test -v -count=1 -run 'TestRemoteDelivery_AuthMX_Fail|TestRemoteDelivery_TLSErrFallback' ./internal/target/remote/ -test.debuglog
```

All three target test packages **compiled and PASSED**
(`internal/endpoint/smtp`, `internal/target/queue`, `internal/target/remote`).

**Reading the raw output.** Because the test logger writes through `t.Log`, the
Go test runner prepends its own source location and indentation to each line,
e.g. `    output.go:41: `. That prefix is **an artifact of the test runner, not
part of maddy's log format**. Throughout this document the log lines are quoted
*without* the `output.go:41:` prefix so that the clean maddy log line is shown.
Within each maddy log line the message and its `{json}` object are separated by
a **literal TAB character** (shown here as an actual tab inside the fenced
blocks; verified with `cat -A`, which renders it as `^I`).

## 2. Log-format primer

maddy structured log lines have the shape `<name>: <msg><TAB>{ordered-json}`.
The relevant mechanics, all in the logging package:

- **Name prefix.** The logger prepends `Name + ": "` to every message
  [internal/log/log.go:L182]. The canonical format is documented in the `Msg`
  doc-comment as `name: msg\t{"key":"value","key2":"value2"}`
  [internal/log/log.go:L60] (the `\t` there is the source comment's escape for
  the TAB that appears literally in the rendered output).
- **Alphabetical field ordering.** JSON fields are rendered as an object whose
  keys are collected and `sort.Strings`-ed before serialization
  [internal/log/orderedjson.go:L23]. This is why fields that are *coded* in one
  order (e.g. `"rcpt", to, "msg_id", …`) appear **alphabetized** in the output
  (`{"msg_id":…,"rcpt":…}`).
- **Automatic `reason` field.** `Logger.Error` adds a `reason` field equal to
  the underlying error's text when one is not already present
  [internal/log/log.go:L89-L104] (specifically the assignment at
  [internal/log/log.go:L98-L99]).
- **Injected `msg_id`.** `DeliveryLogger` attaches a persistent `msg_id` field
  to every queue/remote delivery log line by copying it from the message
  metadata [internal/target/delivery.go:L8-L16].
- **Two distinct `msg_id` formats — this matters for the answers below:**
  - **Endpoint-originated** messages receive **8 lowercase hexadecimal
    characters** from `GenerateMsgID()`, which reads 4 random bytes and
    hex-encodes them [internal/msgpipeline/msgid.go:L12-L16]. This value is
    **random per run**.
  - Tests that **drive a delivery target directly** (via the `testutils`
    harness) receive a **deterministic 40-hex-character SHA-1 of the test
    name**: `hex(sha1(t.Name()))` [internal/testutils/target.go:L184-L185].
    This value is **reproducible across runs**.

## 3. Q1 — SMTP endpoint behavior (`internal/endpoint/smtp`)

**Tests exercised:** `TestSMTPDelivery`
[internal/endpoint/smtp/smtp_test.go:L122] (successful delivery),
`TestSMTPDelivery_AbortData` [internal/endpoint/smtp/smtp_test.go:L360] (abort
mid-DATA), and `TestSMTPDelivery_AbortLogout`
[internal/endpoint/smtp/smtp_test.go:L398] (abort by disconnecting before DATA).
The endpoint logger name is set to `"smtp"`
[internal/endpoint/smtp/smtp_test.go:L49]; the message pipeline uses the
sub-logger `"smtp/pipeline"` [internal/endpoint/smtp/smtp_test.go:L85].

**Verbatim — successful delivery (`TestSMTPDelivery`):**

```text
smtp: incoming message	{"msg_id":"d3250afc","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:60004"}
smtp: RCPT ok	{"msg_id":"d3250afc","rcpt":"rcpt1@example.com"}
smtp: RCPT ok	{"msg_id":"d3250afc","rcpt":"rcpt2@example.com"}
smtp: accepted	{"msg_id":"d3250afc"}
```

**Verbatim — aborted mid-DATA (`TestSMTPDelivery_AbortData`):**

```text
smtp: incoming message	{"msg_id":"0464e3a9","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:33234"}
smtp: RCPT ok	{"msg_id":"0464e3a9","rcpt":"test@example.com"}
smtp: DATA error	{"msg_id":"0464e3a9","reason":"unexpected EOF"}
smtp: aborted	{"msg_id":"0464e3a9"}
```

**Verbatim — aborted via logout (`TestSMTPDelivery_AbortLogout`):**

```text
smtp: incoming message	{"msg_id":"ae472f73","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:41442"}
smtp: RCPT ok	{"msg_id":"ae472f73","rcpt":"test@example.com"}
smtp: aborted	{"msg_id":"ae472f73"}
```

Note the logout variant ends `RCPT ok → aborted` with **no** `DATA error` line,
because the client disconnected before entering the DATA phase.

### Answers

**(Q1a) Successful delivery vs. aborted delivery — the log messages.**
The two paths diverge at the **terminal** message:

- A **successful** delivery ends with **`smtp: accepted`**, carrying only
  `msg_id` [internal/endpoint/smtp/smtp.go:L334].
- An **aborted** delivery ends with **`smtp: aborted`**, carrying only `msg_id`
  [internal/endpoint/smtp/smtp.go:L72].
- When the abort happens **during DATA**, an additional **`smtp: DATA error`**
  line is emitted first, carrying `{msg_id, reason}`; it is produced by
  `s.log.Error("DATA error", err, "msg_id", s.msgMeta.ID)`
  [internal/endpoint/smtp/smtp.go:L317], and the automatic `reason` field
  (added by `Logger.Error` [internal/log/log.go:L98-L99]) was observed as
  **`"unexpected EOF"`**.

**(Q1b) The "recipient successfully added" line and all its JSON fields.**
The line is **`smtp: RCPT ok`** and it carries exactly **two** JSON fields —
**`msg_id`** and **`rcpt`** [internal/endpoint/smtp/smtp.go:L243]. In the source
it is written as `Msg("RCPT ok", "rcpt", to, "msg_id", s.msgMeta.ID)` (i.e.
`rcpt` first), but the ordered-JSON renderer alphabetizes the keys, so the
observed object is `{"msg_id":…,"rcpt":…}` (see the alphabetical-ordering rule
[internal/log/orderedjson.go:L23]).

**(Q1c) Comparison of the two sequences.**

- **Success:** `incoming message` → `RCPT ok` (once per recipient) → `accepted`.
- **Abort mid-DATA:** `incoming message` → `RCPT ok` → `DATA error` → `aborted`.
- **Abort via logout:** `incoming message` → `RCPT ok` → `aborted`.

The distinguishing signals are (1) the **terminal line** — `accepted` for
success versus `aborted` for an abort — and (2) the presence of a **`DATA
error`** line, which appears only when the abort occurs during the DATA phase.

**(Q1d) Module name in the log prefix for SMTP operations = `smtp`.**
It is set by `endp.Log = testutils.Logger(t, "smtp")`
[internal/endpoint/smtp/smtp_test.go:L49], and the logger prepends `Name + ": "`
to every message [internal/log/log.go:L182]. The message pipeline emits under
the related sub-logger prefix **`smtp/pipeline`**
[internal/endpoint/smtp/smtp_test.go:L85].

**(Q1e) `msg_id` format = 8 lowercase hexadecimal characters.**
Endpoint-originated messages get their ID from `GenerateMsgID()`, which reads 4
random bytes and hex-encodes them (4 bytes → 8 hex chars)
[internal/msgpipeline/msgid.go:L12-L16].
**⚠ Non-deterministic:** the value is random per run. The values shown above —
`d3250afc`, `0464e3a9`, `ae472f73` — are illustrative examples from **this**
run; a different run yields different 8-hex IDs. The **format** (8 lowercase hex
characters) is the stable answer.


## 4. Q2 — Queue delivery lifecycle (`internal/target/queue`)

**Test exercised:** `TestQueueDelivery_TemporaryFail`
[internal/target/queue/queue_test.go:L297] (and, for a multi-retry variant,
`TestQueueDelivery_MultipleAttempts` [internal/target/queue/queue_test.go:L357]).
The queue logger name is `"queue"` [internal/target/queue/queue_test.go:L58].
This test drives the queue target **directly** through the `testutils` harness,
so its `msg_id` is the **deterministic 40-hex SHA-1** of the test name
[internal/testutils/target.go:L184-L185]. Independently confirmed:
`printf '%s' 'TestQueueDelivery_TemporaryFail' | sha1sum` →
`af8090c7eb39f761862b1f027b4f2b0bb1ce86d1`, which matches the observed IDs
exactly.

**Verbatim single-message lifecycle** (captured from the isolated run
`-run '^TestQueueDelivery_TemporaryFail$'`; the message has two recipients,
`tester1@example.org` and `tester2@example.org`):

```text
[debug] queue: delivery attempt #1	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
queue: delivery attempt failed	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org","reason":"you shall not pass"}
queue: delivery attempt failed	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org","reason":"you shall not pass"}
queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-457ns","rcpts":["tester1@example.org","tester2@example.org"]}
[debug] queue: delivery attempt #2	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org"}
queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org"}
```

The block above shows the **delivery-lifecycle** lines that answer the question.
Between them, maddy also emits finer-grained `[debug]` trace lines (e.g.
`starting delivery for <id>`, `delivery semaphore acquired`, `target.Start OK`,
`delivery.AddRcpt <rcpt> OK`, `delivery.Body failed: you shall not pass`,
`removed message from disk`); those are elided here to keep the focus on the
acceptance → failure → retry → delivery sequence, and each quoted line above is
reproduced verbatim from the captured output.

### Answer — the complete sequence from acceptance through retry

Reading the trace in order, the lifecycle is:

1. **`queue: delivery attempt #N`** — a **debug-level** line (visible only with
   `-test.debuglog`), emitted by `dl.Debugf("delivery attempt #%d",
   meta.TriesCount+1)` [internal/target/queue/queue.go:L367]. It carries only
   `{msg_id}`.
2. **`queue: delivery attempt failed`** — fields `{msg_id, rcpt, reason}`,
   emitted by `dl.Error("delivery attempt failed", rcptErr, "rcpt", rcpt)`
   [internal/target/queue/queue.go:L384]; the `reason` was observed as
   **`"you shall not pass"`** (added automatically by `Logger.Error`
   [internal/log/log.go:L98-L99]).
3. **`queue: will retry`** — fields `{attempts_count, msg_id, next_try_delay,
   rcpts}` [internal/target/queue/queue.go:L415-L418]. This is the
   retry-scheduling line dissected in Q4 below.
4. **`queue: delivery attempt #N+1`** — the next debug-level attempt line
   [internal/target/queue/queue.go:L367].
5. **`queue: delivered`** — fields `{attempt, msg_id, rcpt}`, emitted by
   `dl.Msg("delivered", "rcpt", rcpt, "attempt", meta.TriesCount+1)`
   [internal/target/queue/queue.go:L378].

**Per-recipient repetition (fidelity detail).** Because the test message has
**two** recipients, the recipient-scoped lines appear **once per recipient**:
two `delivery attempt failed` lines on attempt #1 (one for
`tester1@example.org`, one for `tester2@example.org`), and two `delivered`
lines on attempt #2. The `delivery attempt #N` (debug) and `will retry` lines,
by contrast, are emitted **once per attempt** for the whole message; the
`will retry` line lists all pending recipients in its `rcpts` array.

**`msg_id` here = 40-hex SHA-1** (`af8090c7eb39f761862b1f027b4f2b0bb1ce86d1`),
deterministic and reproducible, precisely because the test invokes the queue
target directly via the harness [internal/testutils/target.go:L184-L185] — in
contrast to the random 8-hex endpoint IDs in Q1.


## 5. Q3 — Remote delivery: MX-authentication failure + TLS fallback (`internal/target/remote`)

**Tests exercised:** `TestRemoteDelivery_AuthMX_Fail`
[internal/target/remote/mxauth_test.go:L16] (MX authenticity required but not
satisfied) and `TestRemoteDelivery_TLSErrFallback`
[internal/target/remote/remote_test.go:L852] (TLS handshake fails, delivery
falls back to plaintext). The remote logger name is `"remote"`
[internal/target/remote/mxauth_test.go:L38], and `AuthMX_Fail` sets
`requireMXAuth: true` [internal/target/remote/mxauth_test.go:L37], which selects
the authenticity check discussed below.

### (Q3a) MX authenticity failure — the exact internal error string

When MX authentication is required and the MX is not authenticated, the code
raises an `exterrors.SMTPError` with `Code: 550`
[internal/target/remote/connect.go:L96], `EnhancedCode: {5, 7, 0}`
[internal/target/remote/connect.go:L97], and the message
[internal/target/remote/connect.go:L98]:

```text
Failed to estabilish the MX record (mx.example.invalid.) authenticity
```

The message is produced by `fmt.Sprintf("Failed to estabilish the MX record
(%s) authenticity", mx)` and **contains the source-code misspelling
`estabilish`** (for "establish"). It is quoted **exactly as observed**, typo
included, and is deliberately **not** corrected.

### (Q3b) The surfaced error — enhanced status code and reply text

The error that the test observes at the delivery boundary is the **re-wrapped**
form. The test harness dumps the error and its fields verbatim (the leading
`-- ... delivery.AddRcpt test@example.invalid` is the harness's own annotation):

```text
Failed to estabilish the MX record (mx.example.invalid.) authenticity map[domain:example.invalid reason:Failed to estabilish the MX record (mx.example.invalid.) authenticity smtp_code:550 smtp_enchcode:[5 4 0] smtp_msg:No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity target:remote]
```

From this dump, the exact values are:

- **Enhanced status code (X.Y.Z form) = `5.4.0`** — rendered in the dump as
  `smtp_enchcode:[5 4 0]`.
- **`smtp_code` = `550`**.
- **Reply text (`smtp_msg`) = `No usable MXs, last err: Failed to estabilish the
  MX record (mx.example.invalid.) authenticity`**.

The `smtp_code` / `smtp_enchcode` / `smtp_msg` keys are emitted by
`SMTPError.Fields()` [internal/exterrors/smtp.go:L72-L79].

**Rationale — why the surfaced enhanced code is `5.4.0`, not the `5.7.0` raised
internally.** The internal authenticity check uses enhanced code `{5, 7, 0}`
[internal/target/remote/connect.go:L97]. However, that error is re-wrapped by
the "No usable MXs" path [internal/target/remote/connect.go:L204-L214], which
constructs a new `SMTPError` whose enhanced code is computed by
`exterrors.SMTPEnchCode(err, exterrors.EnhancedCode{0, 4, 0})`
[internal/target/remote/connect.go:L206]. `SMTPEnchCode` is intended to set the
first digit (the class) based on whether the error is temporary, but it
**unconditionally forces the class digit to `5`**: it sets `code[0] = 4` for a
temporary error [internal/exterrors/smtp.go:L124] and then executes
`code[0] = 5` on the very next line [internal/exterrors/smtp.go:L126], which
overrides the temporary case. Applied to the `{0, 4, 0}` default, this yields
`{5, 4, 0}` = **`5.4.0`**. The second (`4`) and third (`0`) digits are carried
through from the `{0, 4, 0}` default, so the specific `7` from the internal
`{5, 7, 0}` does not survive the re-wrap. This `code[0] = 5` behavior is
reported **as observed** and, per scope, is **not** corrected.

### (Q3c) TLS-to-plaintext fallback — the exact log line

When the TLS handshake fails and MX-authentication policy permits plaintext, the
remote target logs a fallback message and reconnects without TLS. Verbatim:

```text
remote: TLS error, falling back to plaintext	{"domain":"example.invalid","msg_id":"2176ec5872ed2b87d832b4070e88232bd94ac7d3","mx":"mx.example.invalid.","reason":"smtpconn: tls: failed to verify certificate: x509: certificate signed by unknown authority"}
```

- The message is **`remote: TLS error, falling back to plaintext`** with exactly
  **four** JSON fields — **`domain`**, **`msg_id`**, **`mx`**, **`reason`**
  (alphabetical) [internal/target/remote/connect.go:L176-L177].
- **`mx`** (`mx.example.invalid.`) and **`domain`** (`example.invalid`) are
  passed explicitly at the call site
  [internal/target/remote/connect.go:L177].
- **`msg_id`** (`2176ec5872ed2b87d832b4070e88232bd94ac7d3`) is injected by
  `DeliveryLogger` [internal/target/delivery.go:L8-L16]; it is the deterministic
  40-hex SHA-1 of the test name — independently confirmed via
  `printf '%s' 'TestRemoteDelivery_TLSErrFallback' | sha1sum` →
  `2176ec5872ed2b87d832b4070e88232bd94ac7d3`.
- **`reason`** is added automatically by `Logger.Error` from the underlying
  error's text [internal/log/log.go:L89-L104]. Under the `go1.26.4` toolchain
  used for this investigation it was observed as **`smtpconn: tls: failed to
  verify certificate: x509: certificate signed by unknown authority`**. The
  `smtpconn: ` prefix is maddy's own: `smtpconn.TLSError.Error()` returns
  `"smtpconn: " + err.Err.Error()` [internal/smtpconn/smtpconn.go:L144-L146],
  wrapping the Go standard-library TLS verification error that follows.

**Rationale — when the fallback fires.** The reconnect-without-TLS branch is
guarded so that it triggers only when the connection error is an
`smtpconn.TLSError` **and** MX authentication did not object to plaintext:
`if _, ok := err.(smtpconn.TLSError); ok && authErr == nil`
[internal/target/remote/connect.go:L175]. In `TestRemoteDelivery_TLSErrFallback`
the server presents a certificate that fails verification, producing exactly
that `smtpconn.TLSError`, and because MX auth is not enforced, plaintext is
permitted and the fallback proceeds.


## 6. Q4 — Queue retry-scheduling field (`internal/target/queue`)

**The retry-scheduling line is `queue: will retry`**
[internal/target/queue/queue.go:L415].

**The JSON field that carries the retry delay value is `next_try_delay`**
[internal/target/queue/queue.go:L417] — quoted exactly as `next_try_delay`
(not "a delay field"). The complete, alphabetically-ordered field set on that
line is `{attempts_count, msg_id, next_try_delay, rcpts}` (with `msg_id`
injected by `DeliveryLogger`).

**Value and rationale.** `next_try_delay` is a Go `time.Duration` rendered as a
string, computed as `time.Until(nextTryTime)`
[internal/target/queue/queue.go:L417]. In the tests it is a near-zero /
negative-nanosecond value — observed as **`"-457ns"`** in the isolated
`TestQueueDelivery_TemporaryFail` run above (and, e.g., `"-623ns"` / `"-287ns"`
in `TestQueueDelivery_MultipleAttempts`). The reason it is essentially zero is
that the test helper `cleanQueue` overrides the retry parameters to
`initialRetryTime = 0` [internal/target/queue/queue_test.go:L50] and
`retryTimeScale = 1` [internal/target/queue/queue_test.go:L51], so the computed
"next try" time is effectively *now*; the tiny **negative** magnitude is just
the scheduling overhead that elapsed between computing `nextTryTime` and
evaluating `time.Until`.
**⚠ Non-deterministic magnitude** — the field **name** `next_try_delay` is the
stable answer; the numeric value is an illustrative artifact that varies per
run.

**Production behavior (rationale).** Outside the test overrides, the next-try
time is computed as `initialRetryTime × retryTimeScale^(TriesCount-1)`
[internal/target/queue/queue.go:L414], with production defaults
`initialRetryTime = 15 * time.Minute`
[internal/target/queue/queue.go:L185] and `retryTimeScale = 2`
[internal/target/queue/queue.go:L186] — i.e. the delay grows as
**`15m × 2^(n-1)`** (15 minutes, then 30 minutes, then 1 hour, …). The default
maximum number of attempts is **`max_tries = 8`**
[internal/target/queue/queue.go:L204]; the test's `cleanQueue` lowers this to
`maxTries = 5` [internal/target/queue/queue_test.go:L53], but `8` is the
production default.

## 7. Coverage table

| Sub-part | One-line answer | Section |
|----------|-----------------|---------|
| **Q1a** — success vs. abort log messages | Success terminates with `smtp: accepted`; abort terminates with `smtp: aborted`; a mid-DATA abort adds `smtp: DATA error` (`reason` observed `"unexpected EOF"`) | §3 |
| **Q1b** — "recipient added" line + all JSON fields | `smtp: RCPT ok` with exactly two fields, `msg_id` and `rcpt` (alphabetized) | §3 |
| **Q1c** — comparison of sequences | Success: `incoming message → RCPT ok(×N) → accepted`; abort: `… → RCPT ok → [DATA error →] aborted`; distinguished by terminal `accepted` vs `aborted` and the presence of `DATA error` | §3 |
| **Q1d** — module name in the log prefix | `smtp` (pipeline sub-logger prefix is `smtp/pipeline`) | §3 |
| **Q1e** — `msg_id` format | 8 lowercase hexadecimal characters from `GenerateMsgID()`; random per run | §3 |
| **Q2** — complete queue lifecycle (accept → retry) | `delivery attempt #N` (debug) → `delivery attempt failed` (`msg_id,rcpt,reason`) → `will retry` (`attempts_count,msg_id,next_try_delay,rcpts`) → `delivery attempt #N+1` (debug) → `delivered` (`attempt,msg_id,rcpt`); recipient-scoped lines repeat once per recipient | §4 |
| **Q3a** — MX authenticity error string | `Failed to estabilish the MX record (mx.example.invalid.) authenticity` (typo `estabilish` reported as-is) | §5 |
| **Q3b** — enhanced status code + reply text | Enhanced code `5.4.0` (`smtp_enchcode:[5 4 0]`), `smtp_code` `550`, reply text `No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity`; surfaced `5.4.0` (not internal `5.7.0`) due to `SMTPEnchCode` forcing `code[0]=5` | §5 |
| **Q3c** — TLS→plaintext fallback log line + all JSON fields | `remote: TLS error, falling back to plaintext` with four fields `domain`, `msg_id`, `mx`, `reason`; observed `reason` = `smtpconn: tls: failed to verify certificate: x509: certificate signed by unknown authority` | §5 |
| **Q4** — retry-delay JSON field name | `next_try_delay` on the `queue: will retry` line | §6 |

All ten sub-parts (Q1a–e, Q2, Q3a–c, Q4) are explicitly addressed above.

## 8. Determinism note

To distinguish stable answers from run-varying artifacts:

**Deterministic — identical across runs (and toolchain-independent):**

- All maddy log **message strings**: `incoming message`, `RCPT ok`, `accepted`,
  `aborted`, `DATA error`, `delivery attempt #N`, `delivery attempt failed`,
  `will retry`, `delivered`, `TLS error, falling back to plaintext`.
- All **JSON field names**: `msg_id`, `rcpt`, `sender`, `src_host`, `src_ip`,
  `reason`, `attempt`, `attempts_count`, `next_try_delay`, `rcpts`, `domain`,
  `mx`, and the error keys `smtp_code`, `smtp_enchcode`, `smtp_msg`.
- All **module prefixes**: `smtp`, `smtp/pipeline`, `queue`, `remote`.
- **Status codes and reply text**: `smtp_code` `550`, surfaced enhanced code
  `5.4.0`, internally-raised `5.7.0`, and the `No usable MXs, last err: …`
  reply text — including the `estabilish` typo.
- The **40-hex-character SHA-1 `msg_id`s** for target-driven tests, e.g.
  `af8090c7eb39f761862b1f027b4f2b0bb1ce86d1` (queue) and
  `2176ec5872ed2b87d832b4070e88232bd94ac7d3` (remote), because they are
  `hex(sha1(t.Name()))`.

**Non-deterministic — varies per run:**

- The **endpoint 8-hex `msg_id`** (e.g. `d3250afc`, `0464e3a9`, `ae472f73`),
  because it is 4 random bytes hex-encoded.
- The **magnitude** of `next_try_delay` (e.g. `-457ns`), a sub-microsecond
  scheduling artifact under the tests' zeroed retry parameters.

