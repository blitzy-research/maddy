# maddy Runtime Behavior Q&A — SMTP Endpoint, Delivery Queue, and Remote Delivery

> **Branch / commit:** `maddy_26452dd8dd78` @ `26452dd8dd787dc455278b0fdd296f4a5432c768`
> **Module:** `github.com/foxcpp/maddy` (go.mod declares `go 1.13` as the minimum — `go.mod:3`)
> **Document type:** observation-grounded onboarding Q&A. Every behavioral claim below is paired with the **verbatim** log line or error string that was observed when the relevant test suite was actually run, plus a `file:line` citation to the source that emits it. Any statement derived from reading source alone (not observed at runtime) is explicitly labeled **(inferred)**.

---

## Preamble — how this document was produced

### Objective

Answer four question groups about maddy's runtime behavior by **running maddy's own tests** and quoting the real output, not by reading source in isolation:

- **Q1 — SMTP endpoint logging** (`internal/endpoint/smtp`): success vs. abort messages, the complete `RCPT ok` log line (all JSON fields), the abort output for comparison, the module name in the log prefix, and the `msg_id` format.
- **Q2 — Queue message trace** (`internal/target/queue`): the complete, ordered sequence of log messages from initial acceptance through retry.
- **Q3 — Remote delivery** (`internal/target/remote`): the MX-authenticity failure error string, the precise SMTP enhanced status code (`X.Y.Z`) and complete reply text, and the TLS→plaintext fallback log line (all JSON fields).
- **Q4 — Queue retry scheduling** (`internal/target/queue`): the retry-scheduling log line and the exact JSON field name that carries the retry delay.

### Toolchain and build (default configuration, canonical commands)

The commands below were executed in this environment; all output quoted in this document comes from these runs.

| Item | Value (observed) |
|------|------------------|
| Go toolchain | `go version go1.22.12 linux/amd64` (satisfies the `go 1.13` minimum at `go.mod:3`) |
| C compiler (cgo) | `gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0` |
| cgo | `CGO_ENABLED=1` (Go default; required by the `mattn/go-sqlite3` driver and the PAM helper) |
| Build command | `CGO_ENABLED=1 go build ./...` → exit `0` (the only stderr is a benign `go-sqlite3` C warning: `warning: function may return address of local variable [-Wreturn-local-addr]`) |
| Canonical build/test (`.build.yml:11`, `.build.yml:14`) | `go build ./...` and `go test ./... -cover -race` |

**Read-only source tree.** No file in the maddy source tree was modified. `git status --porcelain` was **empty (clean)** before the runs, after the build, and after every test run. All captured output was written **outside** the repository and removed afterward. The only artifact this task adds is this document, under `blitzy/documentation/` (outside the maddy source tree).

### The maddy log line format

maddy's structured logger emits each line in the shape:

```
<Name>: <msg>	{ordered-JSON}
```

- The leading `<Name>: ` prefix is applied in `Logger.log` — `s = l.Name + ": " + s` (`internal/log/log.go:182`, inside the `log` method beginning at `internal/log/log.go:180`).
- The message is written, then a **literal TAB**, then the JSON object, in `Logger.formatMsg`: `formatted.WriteString(msg)` (`internal/log/log.go:138`), `formatted.WriteRune('\t')` (`internal/log/log.go:139`), and `marshalOrderedJSON(&formatted, fields)` (`internal/log/log.go:148`). JSON keys are emitted in **alphabetical order** (hence "ordered-JSON"). `formatMsg` spans `internal/log/log.go:135`–`155`.
- `Logger.Msg(msg, fields...)` (`internal/log/log.go:72`) logs at info level; `Logger.Error(msg, err, fields...)` (`internal/log/log.go:89`–`104`) additionally **auto-injects a `reason` field** from the error text: `if allFields["reason"] == nil { allFields["reason"] = err.Error() }` (`internal/log/log.go:98`–`100`).
- Debug-gated lines are additionally prefixed with `[debug] `, and (in the production writer) optionally a UTC timestamp: see `wcOutput.Write` (`internal/log/writer.go:16`–`29`) — timestamp at `internal/log/writer.go:18`–`20`, `builder.WriteString("[debug] ")` at `internal/log/writer.go:21`–`23`.

> **`	` = literal TAB.** In every quoted log line below there is a **real TAB** character between the message and the `{` of the JSON (do not read it as spaces). This is maddy's actual on-the-wire separator (`internal/log/log.go:139`). If your Markdown renderer collapses it, note that the raw bytes contain one TAB there.

> **`output.go:41:` is a test artifact, not maddy's format.** When maddy's tests run, the test logger routes each line through Go's `testing.T.Log`, which prefixes it with indentation and the caller location `    output.go:41: `. That prefix is produced by Go's `testing` package, **not** by maddy. Every quoted line below shows the **canonical** maddy portion, i.e. everything from `<name>:` onward; the `output.go:41:` prefix is dropped. (Wiring: `internal/testutils/logger.go:34` calls `t.Log(str)`.)

### Why `-v -test.debuglog -count=1` is mandatory

The requested JSON fields are only visible when the tests run **verbose** with **debug logging enabled**:

- The test logger defines two flags — `test.debuglog` (`internal/testutils/logger.go:14`) and `test.directlog` (`internal/testutils/logger.go:15`).
- By default it routes every line to `t.Log(str)` (`internal/testutils/logger.go:34`); Go only prints `t.Log` output for a **passing** test when `-v` is set.
- Debug-level lines (`[debug] …`) are gated on the `test.debuglog` flag via the logger's `Debug: *debugLog` setting (`internal/testutils/logger.go:39`); the `[debug] ` prefix is added at `internal/testutils/logger.go:31`–`33`.
- `-count=1` bypasses Go's test cache so the lines are actually re-emitted on each run.

> **Placement matters:** the `-test.debuglog` flag must appear **after** the package path on the `go test` command line; otherwise `go test` consumes the package argument and silently runs the root package (0 tests).

---

## Q1 — SMTP endpoint logging (`internal/endpoint/smtp`)

**Command (run twice, for `msg_id` stability):**

```
go test -count=1 -v -run 'TestSMTPDelivery$|TestSMTPDelivery_AbortData$|TestSMTPDelivery_AbortLogout$' ./internal/endpoint/smtp/ -test.debuglog
```

**Result:** all three sub-tests **PASS**:

```
--- PASS: TestSMTPDelivery (0.00s)
--- PASS: TestSMTPDelivery_AbortData (0.25s)
--- PASS: TestSMTPDelivery_AbortLogout (0.25s)
PASS
ok  	github.com/foxcpp/maddy/internal/endpoint/smtp	0.510s
```

Entry points: `TestSMTPDelivery` (`internal/endpoint/smtp/smtp_test.go:122`), `TestSMTPDelivery_AbortData` (`internal/endpoint/smtp/smtp_test.go:360`), `TestSMTPDelivery_AbortLogout` (`internal/endpoint/smtp/smtp_test.go:398`).

### Q1(d) — Module name in the SMTP log prefix

**Answer: `smtp`.** Every SMTP endpoint log line begins with `smtp: `.

- **Evidence (observed):**
  ```
  smtp: accepted	{"msg_id":"025a2737"}
  ```
- **Source:** the logger name is set from `modName` in the endpoint constructor — `Log: log.Logger{Name: modName}` (`internal/endpoint/smtp/smtp.go:495`). The SMTP-family endpoints are registered under the names `smtp`, `submission`, and `lmtp` (`internal/endpoint/smtp/smtp.go:715`–`717`); this run exercises the `smtp` endpoint, so `smtp` is the canonical prefix observed. The `<Name>: ` prefix itself is applied at `internal/log/log.go:182`.

### Q1(e) — `msg_id` format (canonical)

**Answer: an 8-character lowercase hexadecimal string** (4 random bytes, hex-encoded).

- **Source:** `GenerateMsgID` (`internal/msgpipeline/msgid.go:12`–`16`) builds `rawID := make([]byte, 4)` (`:13`), fills it with `rand.Read(rawID)` (`:14`), and returns `hex.EncodeToString(rawID)` (`:15`) — 4 bytes → 8 hex characters.
- **Evidence + stability (≥2 runs, run scale = 2):** the format is stable across runs; the values are random each run:
  - **Run A** observed `msg_id` values: `025a2737`, `da71ca80`, `1d09a677`
  - **Run B** observed `msg_id` values: `7429a110`, `d538d274`, `faef9411`

  All six are 8 lowercase-hex characters. This document quotes **Run A** values in the log lines below; Run B confirms only the *format* is stable (the *values* differ per run, as expected from `rand.Read`).

### Q1(a) — Successful delivery: the exact log messages

`TestSMTPDelivery` (a successful delivery of one message to two recipients), canonical lines in order (Run A, `msg_id` `025a2737`):

```
smtp: incoming message	{"msg_id":"025a2737","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:56828"}
smtp: RCPT ok	{"msg_id":"025a2737","rcpt":"rcpt1@example.com"}
smtp: RCPT ok	{"msg_id":"025a2737","rcpt":"rcpt2@example.com"}
smtp: accepted	{"msg_id":"025a2737"}
```

- The transaction opens with `smtp: incoming message` — emitted at `internal/endpoint/smtp/smtp.go:136` (the non-authenticated branch used by this test; an authenticated variant with an extra `username` field lives at `internal/endpoint/smtp/smtp.go:128`).
- Each accepted recipient logs `smtp: RCPT ok` (`internal/endpoint/smtp/smtp.go:243`).
- **A successful delivery terminates with `smtp: accepted`** — emitted at `internal/endpoint/smtp/smtp.go:334` (the `Data` path) and `internal/endpoint/smtp/smtp.go:377` (the `LMTPData` path).

### Q1(b) — The complete `RCPT ok` log line, including ALL JSON fields

> User request (verbatim): *"show me the complete log line format when a recipient is successfully added (including all JSON fields)"*

**Answer — the complete line, with a real TAB between the message and the JSON:**

```
smtp: RCPT ok	{"msg_id":"025a2737","rcpt":"rcpt1@example.com"}
```

**The JSON object has exactly two fields, in alphabetical order: `msg_id` and `rcpt`.**

- **Source:** `s.endp.Log.Msg("RCPT ok", "rcpt", to, "msg_id", s.msgMeta.ID)` (`internal/endpoint/smtp/smtp.go:243`). The call passes `rcpt` then `msg_id`, but `marshalOrderedJSON` (`internal/log/log.go:148`) re-orders keys alphabetically, so `msg_id` precedes `rcpt` in the output — matching the observed line above.
- The terminal success line for the whole message is `smtp: accepted	{"msg_id":"025a2737"}` (single field `msg_id`; `internal/endpoint/smtp/smtp.go:334`).

### Q1(c) — Aborted delivery (for comparison)

**`TestSMTPDelivery_AbortData`** — the client drops the connection mid-`DATA` (Run A, `msg_id` `da71ca80`):

```
smtp: RCPT ok	{"msg_id":"da71ca80","rcpt":"test@example.com"}
smtp: DATA error	{"msg_id":"da71ca80","reason":"unexpected EOF"}
smtp: aborted	{"msg_id":"da71ca80"}
```

**`TestSMTPDelivery_AbortLogout`** — the client aborts after recipients but issues no `DATA` (Run A, `msg_id` `1d09a677`). Note there is **no** `DATA error` line here:

```
smtp: RCPT ok	{"msg_id":"1d09a677","rcpt":"test@example.com"}
smtp: aborted	{"msg_id":"1d09a677"}
```

- **`smtp: aborted`** — emitted at `internal/endpoint/smtp/smtp.go:72`: `s.log.Msg("aborted", "msg_id", s.msgMeta.ID)` (single field `msg_id`).
- **`smtp: DATA error`** — emitted at `internal/endpoint/smtp/smtp.go:317` (the `Data` path) and `internal/endpoint/smtp/smtp.go:360` (the `LMTPData` path): `s.log.Error("DATA error", err, "msg_id", s.msgMeta.ID)`. The `reason` field (here `"unexpected EOF"`) is **auto-injected by `Logger.Error`** from the underlying error (`internal/log/log.go:98`–`100`), which is why `reason` appears even though it is not passed explicitly.

### Q1 — Success vs. abort, explicit contrast

- A **successful** delivery ends with **`smtp: accepted`** (`internal/endpoint/smtp/smtp.go:334`), as seen in `TestSMTPDelivery`.
- An **aborted** delivery ends with **`smtp: aborted`** (`internal/endpoint/smtp/smtp.go:72`), **optionally preceded** by `smtp: DATA error	{…,"reason":…}` when the abort happens during `DATA` (present in `TestSMTPDelivery_AbortData`, absent in `TestSMTPDelivery_AbortLogout`).

---
## Q2 — Queue message trace (`internal/target/queue`)

**Command:**

```
go test -count=1 -v -run 'TestQueueDelivery$|TestQueueDelivery_TemporaryFail$|TestQueueDelivery_MultipleAttempts$' ./internal/target/queue/ -test.debuglog
```

**Result:** all three sub-tests **PASS**:

```
--- PASS: TestQueueDelivery (0.00s)
--- PASS: TestQueueDelivery_TemporaryFail (0.00s)
--- PASS: TestQueueDelivery_MultipleAttempts (0.00s)
PASS
ok  	github.com/foxcpp/maddy/internal/target/queue	0.05s
```

Entry points: `TestQueueDelivery` (`internal/target/queue/queue_test.go:225`), `TestQueueDelivery_TemporaryFail` (`internal/target/queue/queue_test.go:297`), `TestQueueDelivery_MultipleAttempts` (`internal/target/queue/queue_test.go:357`).

**Module prefix:** `queue` — set at `Log: log.Logger{Name: "queue"}` (`internal/target/queue/queue.go:188`).

> **`msg_id` in the queue is NON-CANONICAL (a test-harness artifact).** The 40-hex value below (`cab2c2f3…`) is a deterministic **SHA-1 of the test name** produced by the mock target (`sha1.Sum([]byte(t.Name()))` at `internal/testutils/target.go:184`, then `hex.EncodeToString` at `internal/testutils/target.go:185`), injected into every delivery log line by `DeliveryLogger` — `fields["msg_id"] = msgMeta.ID` (`internal/target/delivery.go:13`). The **canonical** production `msg_id` is the SMTP endpoint's 8-hex `GenerateMsgID` value (see Q1(e) and the reconciliations section). Verified: `sha1("TestQueueDelivery_MultipleAttempts")` = `cab2c2f3f939862a8eaea9f844143e36c0f2c1b5`.

### The complete, ordered trace

The richest lifecycle is `TestQueueDelivery_MultipleAttempts`: one message to three recipients where `tester1` fails **permanently**, `tester3` succeeds immediately, and `tester2` fails **temporarily** twice before finally succeeding on the third attempt. The canonical (non-`output.go:41:`) lines, in the exact observed order (TAB before each `{`; `[debug]` lines require `-test.debuglog`):

```
[debug] queue: starting delivery for cab2c2f3f939862a8eaea9f844143e36c0f2c1b5	
[debug] queue: delivery attempt #1	{"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5"}
[debug] queue: failures: permanently: [tester1@example.org], temporary: [tester2@example.org], errors: map[tester1@example.org:you shall not pass 1 tester2@example.org:you shall not pass 2]	{"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5"}
queue: delivered	{"attempt":1,"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","rcpt":"tester3@example.org"}
queue: delivery attempt failed	{"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","rcpt":"tester2@example.org","reason":"you shall not pass 2"}
queue: delivery attempt failed	{"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","rcpt":"tester1@example.org","reason":"you shall not pass 1"}
queue: will retry	{"attempts_count":1,"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","next_try_delay":"-522ns","rcpts":["tester2@example.org"]}
[debug] queue: delivery attempt #2	{"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5"}
queue: delivery attempt failed	{"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","rcpt":"tester2@example.org","reason":"you shall not pass 3"}
queue: will retry	{"attempts_count":2,"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","next_try_delay":"-293ns","rcpts":["tester2@example.org"]}
[debug] queue: delivery attempt #3	{"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5"}
queue: delivered	{"attempt":3,"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","rcpt":"tester2@example.org"}
queue: not delivered, permanent error	{"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","rcpt":"tester1@example.org"}
```

Each distinct line explained (one claim → one evidence):

1. **Initial acceptance** (start of processing) is a debug line with **no JSON object** (bare message + trailing TAB):
   ```
   [debug] queue: starting delivery for cab2c2f3f939862a8eaea9f844143e36c0f2c1b5	
   ```
   Emitted by `q.Log.Debugln("starting delivery for", slot.ID)` (`internal/target/queue/queue.go:278`). Note it uses the *queue* logger `q.Log` (not the per-delivery `dl`), so it carries **no** `msg_id` JSON field. There is **no** separate `"accepted"` string in the queue package (unlike the SMTP endpoint) — verified: `grep -c '"accepted"'` in `queue.go` returns `0`.

2. **Delivery attempt marker** (per attempt), debug:
   ```
   [debug] queue: delivery attempt #1	{"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5"}
   ```
   Emitted by `dl.Debugf("delivery attempt #%d", meta.TriesCount+1)` (`internal/target/queue/queue.go:367`).

3. **Successful recipient**:
   ```
   queue: delivered	{"attempt":1,"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","rcpt":"tester3@example.org"}
   ```
   Emitted by `dl.Msg("delivered", "rcpt", rcpt, "attempt", meta.TriesCount+1)` (`internal/target/queue/queue.go:378`). Fields (alphabetical): `attempt`, `msg_id`, `rcpt`.

4. **Failed recipient** (per failing rcpt on this attempt):
   ```
   queue: delivery attempt failed	{"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","rcpt":"tester2@example.org","reason":"you shall not pass 2"}
   ```
   Emitted by `dl.Error("delivery attempt failed", rcptErr, "rcpt", rcpt)` (`internal/target/queue/queue.go:384`). Fields: `msg_id`, `rcpt`, `reason` (the `reason` is auto-injected by `Logger.Error`, `internal/log/log.go:98`–`100`).

   > **Ordering note (observed).** The *relative* order of the two sibling `delivery attempt failed` lines for different recipients **within a single attempt** is **non-deterministic** — it follows Go's per-recipient goroutine / result-map iteration order, not a fixed sequence. Re-running `TestQueueDelivery_MultipleAttempts` with `-count=1` six times yielded the `tester1`-before-`tester2` order in 4 runs and the reverse in 2; the block above quotes one such run. What *is* stable across runs is the overall lifecycle order (`starting delivery` → `delivery attempt #N` → per-rcpt `delivered` / `delivery attempt failed` → `will retry` → next attempt → terminal `not delivered, …`).

5. **Retry scheduling** (see Q4 for the field-name detail):
   ```
   queue: will retry	{"attempts_count":1,"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","next_try_delay":"-522ns","rcpts":["tester2@example.org"]}
   ```
   Emitted by `dl.Msg("will retry", "attempts_count", meta.TriesCount, "next_try_delay", time.Until(nextTryTime), "rcpts", meta.To)` (`internal/target/queue/queue.go:415`–`418`). Only the **temporarily**-failed recipient (`tester2`) is carried in `rcpts` — the permanently-failed `tester1` is not retried.

6. **Terminal permanent failure** (emitted once the message is finalized, after the last attempt):
   ```
   queue: not delivered, permanent error	{"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","rcpt":"tester1@example.org"}
   ```
   Emitted by `dl.Msg("not delivered, permanent error", "rcpt", rcpt)` (`internal/target/queue/queue.go:398`). Its sibling branch for the temporary-but-exhausted case is `dl.Msg("not delivered, temporary error", "rcpt", rcpt)` (`internal/target/queue/queue.go:394`).

**Cause → effect summary:** the message is *accepted* (`starting delivery for …`), *attempted* (`delivery attempt #1`), and partially succeeds (`delivered` for `tester3`) while `tester1` fails permanently and `tester2` fails temporarily (`delivery attempt failed` ×2). The temporary failure triggers *rescheduling* (`will retry`, carrying only `tester2`), a second attempt fails again, a third attempt finally *delivers* `tester2`, and the permanently-failed `tester1` is reported once at the end (`not delivered, permanent error`).

**Simplest retry, for contrast** — `TestQueueDelivery_TemporaryFail` (`msg_id` `af8090c7…`), where both recipients fail temporarily once then both succeed on attempt 2:

```
queue: delivery attempt failed	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org","reason":"you shall not pass"}
queue: delivery attempt failed	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org","reason":"you shall not pass"}
queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-367ns","rcpts":["tester1@example.org","tester2@example.org"]}
queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org"}
queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org"}
```

> **Note on the queue `msg_id` → test mapping.** By computing the SHA-1 of each test name and cross-checking against the observed `reason` strings, the harness IDs resolve as: `10a443bb…` = `TestQueueDelivery`, `af8090c7…` = `TestQueueDelivery_TemporaryFail`, `cab2c2f3…` = `TestQueueDelivery_MultipleAttempts`. (The plain `TestQueueDelivery` simply logs two `queue: delivered	{"attempt":1,…}` lines and no failures.)

---

## Q3 — Remote delivery: MX-auth failure and TLS fallback (`internal/target/remote`)

**Command:**

```
go test -count=1 -v -run 'TestRemoteDelivery_AuthMX_Fail$|TestRemoteDelivery_MTASTS_SkipNonMatching$|TestRemoteDelivery_TLSErrFallback$' ./internal/target/remote/ -test.debuglog
```

**Result:** all three sub-tests **PASS**:

```
--- PASS: TestRemoteDelivery_AuthMX_Fail (0.00s)
--- PASS: TestRemoteDelivery_MTASTS_SkipNonMatching (0.00s)
--- PASS: TestRemoteDelivery_TLSErrFallback (0.01s)
PASS
ok  	github.com/foxcpp/maddy/internal/target/remote	0.06s
```

Entry points: `TestRemoteDelivery_AuthMX_Fail` (`internal/target/remote/mxauth_test.go:16`), `TestRemoteDelivery_TLSErrFallback` (`internal/target/remote/remote_test.go:852`).

**Module prefix:** `remote` — set at `Log: log.Logger{Name: "remote"}` (`internal/target/remote/remote.go:83`); the per-delivery logger is wrapped by `target.DeliveryLogger(rt.Log, msgMeta)` (`internal/target/remote/remote.go:190`), which injects `msg_id` (`internal/target/delivery.go:13`). The `msg_id` here is again the non-canonical 40-hex SHA-1 harness id.

### Q3(a) — MX-authenticity failure error string

**Answer (observed verbatim, misspelling preserved):**

```
Failed to estabilish the MX record (mx.example.invalid.) authenticity
```

- **Reported exactly as observed** — the source literally misspells "establish" as **`estabilish`**; it is **not** corrected here.
- **Source:** `internal/target/remote/connect.go:94`–`99` builds an `&exterrors.SMTPError{…}` with `Code: 550` (`:96`), `EnhancedCode: exterrors.EnhancedCode{5, 7, 0}` (`:97`), and `Message: fmt.Sprintf("Failed to estabilish the MX record (%s) authenticity", mx)` (`:98`). The intrinsic enhanced code is therefore **`5.7.0`** — see Q3(b) for how the *surfaced* code differs.
- **Related variants (inferred — not emitted at runtime by the tests run here):**
  - **MTA-STS variant:** `Failed to estabilish the MX record authenticity (MTA-STS)`, also enhanced `5.7.0` (`internal/target/remote/connect.go:64`–`68`). *(inferred)* — `TestRemoteDelivery_MTASTS_SkipNonMatching` took the *matching-MX* success branch at runtime (it logged `remote: authenticated MX using MTA-STS` and connected), so this specific failure string was not observed; it is cited from source.
  - **TLS-required variant:** enhanced `5.7.1` for `TLS is required but unsupported or failed …` (`internal/target/remote/connect.go:101`–`107`). *(inferred)* from source.

### Q3(b) — Precise SMTP enhanced status code (X.Y.Z) and complete reply text

> User request (verbatim): *"the precise SMTP enhanced status code (in X.Y.Z format) and the complete reply text"*

The surfaced delivery error is what a client/caller actually sees. It was observed verbatim in the test's error dump. The dump content is produced by `t.Log("-- ... delivery.AddRcpt", rcpt, err, exterrors.Fields(err))` (`internal/testutils/target.go:255`), which prints `exterrors.Fields(err)` as a Go map (keys sorted alphabetically by `fmt`). The leading `target.go:233:` prefix is a Go `testing` harness artifact, **not** part of maddy's log format: because `DoTestDeliveryErrMeta` marks itself `t.Helper()` (`internal/testutils/target.go:237`), `t.Log` attributes the location to that helper's caller — the `return DoTestDeliveryErrMeta(...)` line inside `DoTestDeliveryErr` (`internal/testutils/target.go:233`). The canonical content is everything from `-- ... delivery.AddRcpt` onward:

```
-- ... delivery.AddRcpt test@example.invalid Failed to estabilish the MX record (mx.example.invalid.) authenticity map[domain:example.invalid reason:Failed to estabilish the MX record (mx.example.invalid.) authenticity smtp_code:550 smtp_enchcode:[5 4 0] smtp_msg:No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity target:remote]
```

From this the precise values are:

- **Surfaced SMTP enhanced status code (X.Y.Z): `5.4.0`** — from `smtp_enchcode:[5 4 0]`.
- **SMTP (basic) code: `550`** — from `smtp_code:550`.
- **Complete reply text (`smtp_msg`):**
  ```
  No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity
  ```

**Reconciliation — intrinsic `5.7.0` vs. surfaced `5.4.0` (cause → effect):**

- The *intrinsic* MX-auth error carries enhanced code **`5.7.0`** (`internal/target/remote/connect.go:97`). (This intrinsic triple is not printed as a standalone `X.Y.Z` at runtime; it is cited from source.)
- When no MX can be used, that error is **wrapped** into a `No usable MXs, last err: …` `SMTPError` (`internal/target/remote/connect.go:204`–`213`) whose enhanced code is computed as `EnhancedCode: exterrors.SMTPEnchCode(err, exterrors.EnhancedCode{0, 4, 0})` (`internal/target/remote/connect.go:206`) and whose basic code is `exterrors.SMTPCode(err, 451, 550)` (`internal/target/remote/connect.go:205`).
- `SMTPEnchCode` (`internal/exterrors/smtp.go:122`–`127`) starts from the fallback `{0,4,0}`, conditionally sets `code[0] = 4` when the error is temporary (`internal/exterrors/smtp.go:123`–`125`), **but then unconditionally executes `code[0] = 5`** (`internal/exterrors/smtp.go:126`) — forcing the class digit to `5`. The result is `{5, 4, 0}` = **`5.4.0`**, exactly as observed.

So both values are real and correct for their layer: **`5.7.0`** is the intrinsic MX-auth code (source `connect.go:97`), and **`5.4.0`** is the surfaced/wrapped code the caller sees (observed; produced by `SMTPEnchCode` forcing `code[0]=5`).

### Q3(c) — TLS→plaintext fallback log, including ALL JSON fields

**Answer (observed verbatim, from `TestRemoteDelivery_TLSErrFallback`; TAB before `{`):**

```
remote: TLS error, falling back to plaintext	{"domain":"example.invalid","msg_id":"2176ec5872ed2b87d832b4070e88232bd94ac7d3","mx":"mx.example.invalid.","reason":"smtpconn: tls: failed to verify certificate: x509: certificate signed by unknown authority"}
```

**The JSON object has exactly four fields, in alphabetical order: `domain`, `msg_id`, `mx`, `reason`.**

- **Source:** `rd.Log.Error("TLS error, falling back to plaintext", err, "mx", record.Host, "domain", domain)` (`internal/target/remote/connect.go:176`–`177`). The call passes `mx` then `domain`; `marshalOrderedJSON` (`internal/log/log.go:148`) sorts keys alphabetically. The `reason` field is auto-injected from the TLS error by `Logger.Error` (`internal/log/log.go:98`–`100`), and `msg_id` is injected by `DeliveryLogger` (`internal/target/delivery.go:13`).

---

## Q4 — Queue retry scheduling (`internal/target/queue`)

> User request (verbatim): *"identify the exact JSON field name that contains the retry delay value"*

**Answer: the retry-delay JSON field name is `next_try_delay`.**

- **Retry-scheduling log line (observed verbatim; from `TestQueueDelivery_MultipleAttempts`, `msg_id` `cab2c2f3…`; TAB before `{`):**
  ```
  queue: will retry	{"attempts_count":1,"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","next_try_delay":"-522ns","rcpts":["tester2@example.org"]}
  ```
  The full field set (alphabetical) is `attempts_count`, `msg_id`, **`next_try_delay`**, `rcpts`.
- **Source:** `dl.Msg("will retry", "attempts_count", meta.TriesCount, "next_try_delay", time.Until(nextTryTime), "rcpts", meta.To)` (`internal/target/queue/queue.go:415`–`418`) — the `next_try_delay` key is passed at `internal/target/queue/queue.go:417`.

**Magnitude reconciliation — observed (test) vs. canonical (production):**

- **Observed values are negative / near-zero nanoseconds** (reported exactly as observed, **not** corrected): `next_try_delay":"-522ns"` and `"-293ns"` (`TestQueueDelivery_MultipleAttempts`), and `"-367ns"` (`TestQueueDelivery_TemporaryFail`). These are **non-canonical test artifacts**: the queue test harness sets `initialRetryTime = 0` (`internal/target/queue/queue_test.go:50`), `retryTimeScale = 1` (`:51`), and `maxTries = 5` (`:53`), so the computed "next try" time is essentially "now" and `time.Until(...)` yields a tiny negative duration by the time it is formatted.
- **Canonical production schedule:** the delay is computed as `initialRetryTime * retryTimeScale^(TriesCount-1)` (`internal/target/queue/queue.go:414`, via `math.Pow`). With production defaults `initialRetryTime = 15 * time.Minute` (`internal/target/queue/queue.go:185`) and `retryTimeScale = 2` (`internal/target/queue/queue.go:186`), and `max_tries` defaulting to `8` (`internal/target/queue/queue.go:204`), the real backoff schedule is **15m, 30m, 60m, 120m, …** (i.e. `15m × 2^(n−1)`).

---

## Canonical vs. non-canonical reconciliations (explicit)

Three observed values differ from what a normal production deployment would show; each is labeled below with both forms.

1. **`msg_id`.**
   - **Canonical (production, SMTP entry point):** 8-character lowercase hex from `GenerateMsgID` — `make([]byte, 4)` + `rand.Read` + `hex.EncodeToString` (`internal/msgpipeline/msgid.go:12`–`16`). Observed e.g. `025a2737`.
   - **Non-canonical (queue/remote tests):** 40-character hex = SHA-1 of the test name from the mock target (`internal/testutils/target.go:184`–`185`), injected into delivery logs by `DeliveryLogger` (`internal/target/delivery.go:13`). Observed e.g. `cab2c2f3f939862a8eaea9f844143e36c0f2c1b5`.

2. **Retry delay (`next_try_delay`).**
   - **Non-canonical (observed):** negative/near-zero ns (`-522ns`, `-293ns`, `-367ns`) because the tests set `initialRetryTime = 0` (`internal/target/queue/queue_test.go:50`).
   - **Canonical (production):** `15m × 2^(n−1)` → 15m, 30m, 60m, … from defaults `initialRetryTime = 15m` (`internal/target/queue/queue.go:185`) and `retryTimeScale = 2` (`internal/target/queue/queue.go:186`), formula at `internal/target/queue/queue.go:414`.

3. **Enhanced status code for the MX-auth failure.**
   - **Intrinsic (source):** `5.7.0` at the MX-auth error site (`internal/target/remote/connect.go:97`).
   - **Surfaced (observed):** `5.4.0`, because the `No usable MXs` wrapper (`internal/target/remote/connect.go:204`–`213`) runs the fallback `{0,4,0}` through `SMTPEnchCode`, which unconditionally forces the class digit `code[0] = 5` (`internal/exterrors/smtp.go:126`).

---

## Coverage pass — every named item addressed

- **Q1 — SMTP endpoint logging**
  - [x] (a) success vs. abort messages — success ends `smtp: accepted`; abort ends `smtp: aborted` (± `smtp: DATA error`). *(observed)*
  - [x] (b) complete `RCPT ok` line with **all** JSON fields — exactly `{msg_id, rcpt}` (`internal/endpoint/smtp/smtp.go:243`). *(observed)*
  - [x] (c) aborted-delivery output for comparison — `DATA error` + `aborted` (AbortData) vs. `aborted` only (AbortLogout). *(observed)*
  - [x] (d) module name in the log prefix — **`smtp`** (`internal/endpoint/smtp/smtp.go:495`). *(observed)*
  - [x] (e) `msg_id` format — **8 lowercase hex chars** (`internal/msgpipeline/msgid.go:12`–`16`), format stable across 2 runs, values random per run. *(observed)*
- **Q2 — Queue message trace**
  - [x] complete ordered sequence from initial acceptance (`starting delivery for …`) through attempt → delivered → failure → `will retry` → re-attempt → terminal `not delivered, permanent error`, each with a citation; queue prefix **`queue`** (`internal/target/queue/queue.go:188`). *(observed)*
- **Q3 — Remote delivery**
  - [x] (a) MX-auth failure error string, with the literal misspelling **`estabilish`** reproduced (`internal/target/remote/connect.go:94`–`99`). *(observed)*; MTA-STS and TLS-required variants labeled **(inferred)**.
  - [x] (b) precise enhanced status code + complete reply text — surfaced **`5.4.0`** / code `550` / `No usable MXs, last err: …` (observed), reconciled with intrinsic **`5.7.0`** (`internal/target/remote/connect.go:97`) via `SMTPEnchCode` (`internal/exterrors/smtp.go:126`).
  - [x] (c) TLS→plaintext fallback log with **all** JSON fields — exactly `{domain, msg_id, mx, reason}` (`internal/target/remote/connect.go:176`–`177`). *(observed)*
- **Q4 — Queue retry scheduling**
  - [x] retry-scheduling line `queue: will retry` and the retry-delay field name **`next_try_delay`** (`internal/target/queue/queue.go:415`–`418`, key at `:417`); observed negative ns quoted verbatim; production formula `15m × 2^(n−1)` and defaults stated. *(observed + source)*
- **Reconciliations**
  - [x] All three canonical-vs-non-canonical reconciliations (`msg_id`, retry delay, enhanced code) presented explicitly.

**Completeness statement:** every distinct item named in Q1–Q4 (including each "e.g. / such as / including / like" sub-item) has been answered above, each behavioral claim paired with a single verbatim observed line and a `file:line` citation, with inferred (source-only) statements labeled **(inferred)** and non-canonical test values labeled and reconciled to their canonical production forms.
