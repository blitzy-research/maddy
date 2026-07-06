# maddy Runtime Logging & Delivery Behavior — Onboarding Q&A

This document answers seven onboarding questions (Q1–Q7) about the **runtime logging and delivery behavior** of the [maddy](https://github.com/foxcpp/maddy) mail server across three subsystems: the **SMTP endpoint** (`internal/endpoint/smtp`), the **delivery queue** (`internal/target/queue`), and the **remote (outbound) delivery target** (`internal/target/remote`).

- **Module:** `github.com/foxcpp/maddy` [go.mod:L1], language directive `go 1.13` [go.mod:L3].
- **Branch / commit:** `maddy_26452dd8dd78` — HEAD `26452dd` ("target/remote: Rewrite connection part to allow more concurrency").

Every answer is written **run-first**: the relevant code path was actually built and exercised through its real test entry point, the emitted output was captured verbatim, and only then was the answer written. Each answer block gives **(1)** the exact command that produced the output, **(2)** the verbatim observed output, **(3)** the emitter `file:line` naming the responsible function/struct, and **(4)** a short causal explanation. Values that vary run-to-run (the SMTP `msg_id`, the `next_try_delay`) or that come from a test stand-in (the 40-hex message ID) are explicitly **labeled as such**.

---

## How the output was captured

**Toolchain & isolation.** All builds and tests were run with **Go 1.21.13** (which satisfies the `go 1.13` directive in `go.mod` [go.mod:L3]). The Go toolchain lives at `GOROOT=/usr/local/go`; the module cache and build cache were pinned **outside the repository tree** via `GOPATH=/tmp/gopath` and `GOCACHE=/tmp/gocache`. Because nothing writes inside the checkout, the repository is never mutated by a build or test.

**Read-only guarantee.** Default Go module mode can rewrite `go.sum`. To prevent any mutation, every command was run with **`-mod=readonly`** (the committed `go.sum` is sufficient). The repository was confirmed unchanged at the end via an **empty `git status --porcelain`** (see the closing note).

**Canonical log-line shape.** maddy uses its own minimal logging library in `internal/log` (not logrus/zap). A log line has the shape:

```text
<name>: <msg><TAB><orderedJSON>
```

- The `<msg>` and the `<TAB>` are written by `Logger.formatMsg`, which always appends a tab after the message and then the JSON body **only if** there are fields to render [internal/log/log.go:L135-L152]. A line with no fields therefore ends in a bare trailing tab.
- The JSON keys are **sorted alphabetically** by `marshalOrderedJSON` using `sort.Strings` [internal/log/orderedjson.go:L16-L23]; `time.Duration` values are rendered via `.String()` [internal/log/orderedjson.go:L43-L44] and `LogFormatter` values (such as an enhanced status code) via `FormatLog()` [internal/log/orderedjson.go:L45-L46].
- The `<name>: ` prefix is prepended by `Logger.log`: `if l.Name != "" { s = l.Name + ": " + s }` [internal/log/log.go:L180-L182].
- Every **delivery-scoped** line carries a `msg_id` field because `DeliveryLogger` copies the base logger and injects `fields["msg_id"] = msgMeta.ID` [internal/target/delivery.go:L8-L16] (injection at [internal/target/delivery.go:L13]). Lines emitted on a plain (non-delivery) logger do **not** carry a `msg_id` field.

**Test-harness presentation nuance.** The test harness routes log output two ways [internal/testutils/logger.go:L17-L40]:

- By default, output goes through `t.Log`, which the Go test runner prints with a source tag such as `output.go:41:` or `target.go:233:`; the harness also prepends `[debug] ` to debug messages [internal/testutils/logger.go:L28-L39].
- With **`-test.directlog`**, output goes to stderr through `wcOutput.Write`, which prepends a UTC timestamp (e.g. `2026-07-06T22:05:02.567Z `) and, for debug messages, the `[debug] ` marker [internal/log/writer.go:L15-L28].

In the quoted blocks below, the **canonical content** is preserved: the harness `output.go:NN:` / `target.go:NN:` source tag and the run-specific timestamp are stripped, but the `[debug] ` marker is **kept** on debug lines exactly as observed, and the literal TAB between the message and the JSON is preserved.

**Debug flag requirement.** The queue's `starting delivery`, `delivery attempt #N`, and `using message ID` lines are `Debug*` calls; they are suppressed unless **`-test.debuglog`** is passed [internal/testutils/logger.go:L14]. That flag is therefore required to observe the complete Q4 sequence.

**Exact commands used (one per subsystem):**

```text
# Q1–Q3 (SMTP endpoint)
go test -mod=readonly -v -count=1 -run 'TestSMTPDelivery$|TestSMTPDelivery_AbortData$' ./internal/endpoint/smtp/

# Q4 & Q7 (delivery queue)
go test -mod=readonly -v -count=1 -run 'TestQueueDelivery_TemporaryFail$|TestQueueDelivery_MultipleAttempts$' ./internal/target/queue/ -test.debuglog -test.directlog

# Q5 & Q6 (remote delivery)
go test -mod=readonly -v -count=1 -run 'TestRemoteDelivery_AuthMX_Fail$|TestRemoteDelivery_TLSErrFallback$' ./internal/target/remote/ -test.debuglog
```

**Test entry points executed:** `TestSMTPDelivery` [internal/endpoint/smtp/smtp_test.go:L122], `TestSMTPDelivery_AbortData` [internal/endpoint/smtp/smtp_test.go:L360], `TestQueueDelivery_TemporaryFail` [internal/target/queue/queue_test.go:L297], `TestQueueDelivery_MultipleAttempts` [internal/target/queue/queue_test.go:L357], `TestRemoteDelivery_AuthMX_Fail` [internal/target/remote/mxauth_test.go:L16], `TestRemoteDelivery_TLSErrFallback` [internal/target/remote/remote_test.go:L852].

**Stability.** Each command was run at least twice. Field names, log-line formats, the module-name prefixes, the enhanced status codes, and the deterministic 40-hex test IDs were identical across runs. Only the SMTP `msg_id` (random per run) and the `next_try_delay` (a timing value) differed run-to-run, as noted in the relevant answers.

---

## Q1 — SMTP delivery log: success (`RCPT ok`) vs. abort

**Command:**

```text
go test -mod=readonly -v -count=1 -run 'TestSMTPDelivery$|TestSMTPDelivery_AbortData$' ./internal/endpoint/smtp/
```

### Q1a — Recipient successfully added (`RCPT ok`)

Observed (from `TestSMTPDelivery`; canonical content, harness tag `output.go:41:` stripped):

```text
smtp: incoming message	{"msg_id":"59d93eb4","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:39530"}
smtp: RCPT ok	{"msg_id":"59d93eb4","rcpt":"rcpt1@example.com"}
smtp: RCPT ok	{"msg_id":"59d93eb4","rcpt":"rcpt2@example.com"}
smtp: accepted	{"msg_id":"59d93eb4"}
```

- **`RCPT ok` emitter:** `s.endp.Log.Msg("RCPT ok", "rcpt", to, "msg_id", s.msgMeta.ID)` [internal/endpoint/smtp/smtp.go:L243]. The fields are passed in the order (`rcpt`, `msg_id`) but are rendered **alphabetically** → `msg_id`, `rcpt` (per `marshalOrderedJSON` [internal/log/orderedjson.go:L16-L23]). One `RCPT ok` line is emitted per accepted recipient.
- **`incoming message` emitter (no-auth variant):** `s.log.Msg("incoming message", ...)` [internal/endpoint/smtp/smtp.go:L136]; its fields (`src_host`, `src_ip`, `sender`, `msg_id`) render alphabetically as `msg_id`, `sender`, `src_host`, `src_ip`.
- **`accepted` emitter:** `s.log.Msg("accepted", "msg_id", s.msgMeta.ID)` [internal/endpoint/smtp/smtp.go:L334] (and the equivalent at [internal/endpoint/smtp/smtp.go:L377]). The success path terminates with `smtp: accepted` carrying **only** `msg_id`.

### Q1b — Delivery aborted mid-transaction (`aborted`)

Observed (from `TestSMTPDelivery_AbortData`):

```text
smtp: incoming message	{"msg_id":"969509ba","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:39536"}
smtp: RCPT ok	{"msg_id":"969509ba","rcpt":"test@example.com"}
smtp: DATA error	{"msg_id":"969509ba","reason":"unexpected EOF"}
smtp: aborted	{"msg_id":"969509ba"}
```

- **`aborted` emitter:** `s.log.Msg("aborted", "msg_id", s.msgMeta.ID)` [internal/endpoint/smtp/smtp.go:L72]; it carries **only** `msg_id`.
- **`DATA error` emitter (preceding the abort here):** `s.log.Error("DATA error", err, "msg_id", s.msgMeta.ID)` [internal/endpoint/smtp/smtp.go:L317]. `Logger.Error` folds the error's text into a `reason` field [internal/log/log.go:L89-L104], which is why this line additionally carries `reason` (rendered alphabetically as `msg_id`, `reason`).

### Contrast (the core of Q1)

The **success** path ends with `smtp: accepted` carrying only `msg_id`. The **abort** path emits `smtp: aborted` (also carrying only `msg_id`), optionally preceded by a `smtp: DATA error` line that additionally carries a `reason` field. Note the different `msg_id` values (`59d93eb4` vs. `969509ba`) — each message is assigned its own ID (see Q3).

> **Run-to-run note:** the 8-hex `msg_id` is **random per run**. The values above (`59d93eb4`, `969509ba`) and other observed values (`26205c4b`, `ec068264`) are all equally valid; only the *format* is stable.

---

## Q2 — Module name in the log prefix = `smtp`

Every SMTP line in Q1 is prefixed **`smtp: `**. The prefix is the logger's `Name`, prepended by `Logger.log`: `if l.Name != "" { s = l.Name + ": " + s }` [internal/log/log.go:L180-L182].

The SMTP endpoint sets that name at construction: `Log: log.Logger{Name: modName}` [internal/endpoint/smtp/smtp.go:L495], where `modName` is the module name the endpoint was registered under — one of `smtp` / `submission` / `lmtp` [internal/endpoint/smtp/smtp.go:L715-L717]. The tests exercise the endpoint registered as `smtp`, so the observed prefix is `smtp`.

For comparison, the other two subsystems in this document set their own logger names: the queue logger's `Name` is `queue` [internal/target/queue/queue.go:L188] and the remote logger's `Name` is `remote` [internal/target/remote/remote.go:L83]. There is no per-file logger; the prefix is purely the per-subsystem logger `Name`.

---

## Q3 — Exact `msg_id` format = 8 lowercase hexadecimal characters

The `msg_id` is generated by `GenerateMsgID()`, which reads **4 random bytes** and hex-encodes them → **8 lowercase hex characters** [internal/msgpipeline/msgid.go:L12-L16]:

```go
func GenerateMsgID() (string, error) {
	rawID := make([]byte, 4)
	_, err := rand.Read(rawID)
	return hex.EncodeToString(rawID), err
}
```

It uses the Go standard library (`crypto/rand` + `encoding/hex`) — **not** the `google/uuid` dependency — so the observed IDs are 8 hex characters, not a 36-character UUID. The SMTP endpoint assigns it per message at [internal/endpoint/smtp/smtp.go:L112] (`msgMeta.ID, err = msgpipeline.GenerateMsgID()`).

Observed 8-hex values (random per run, stable at 8 hex characters): `59d93eb4`, `969509ba`, `26205c4b`, `ec068264`.

> **Non-canonical caveat (labeled).** The queue and remote **tests** do **not** use `GenerateMsgID`. They inject a synthetic **40-hex** ID via the test harness: `IDRaw := sha1.Sum([]byte(t.Name()))` → 20 bytes → `hex.EncodeToString` [internal/testutils/target.go:L239-L245]. This is a **deterministic test artifact** (a SHA-1 of the test's name), e.g. `af8090c7eb39f761862b1f027b4f2b0bb1ce86d1` for `TestQueueDelivery_TemporaryFail`. Because it is derived from the test name, it is **identical across runs** — but it is **not** the real 8-hex `GenerateMsgID` production format. The 40-hex IDs appearing in the Q4/Q6/Q7 blocks are this test artifact.

---

## Q4 — Complete queue delivery sequence (accept → attempt → fail → retry)

**Command:**

```text
go test -mod=readonly -v -count=1 -run 'TestQueueDelivery_TemporaryFail$|TestQueueDelivery_MultipleAttempts$' ./internal/target/queue/ -test.debuglog -test.directlog
```

Observed for `TestQueueDelivery_TemporaryFail` (deterministic 40-hex test ID `af8090c7eb39f761862b1f027b4f2b0bb1ce86d1`; canonical content, per-line UTC timestamps stripped, `[debug] ` markers kept). This is the complete sequence from the start of delivery, through the first attempt and its failure, the retry scheduling, and into the second attempt (a preceding `[debug] queue: delivery target: *queue.unreliableTarget` line records the selected target):

```text
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
queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-693ns","rcpts":["tester1@example.org","tester2@example.org"]}
[debug] queue: starting delivery for af8090c7eb39f761862b1f027b4f2b0bb1ce86d1	
[debug] queue: waiting on delivery semaphore for af8090c7eb39f761862b1f027b4f2b0bb1ce86d1	
[debug] queue: delivery semaphore acquired for af8090c7eb39f761862b1f027b4f2b0bb1ce86d1	
[debug] queue: delivery attempt #2	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
[debug] queue: using message ID = af8090c7eb39f761862b1f027b4f2b0bb1ce86d1-2	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
```

In this test the second attempt succeeds; the run continues (and ends) with the per-recipient success and cleanup lines:

```text
queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org"}
queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org"}
[debug] queue: removed message from disk	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
```

**Emitters (naming the responsible function for each):**

- **`starting delivery for` / `waiting on delivery semaphore for` / `delivery semaphore acquired for`** = `q.Log.Debugln(...)` [internal/target/queue/queue.go:L278, L282, L299]. These are emitted on `q.Log` directly (logger name `queue`), **not** on a `DeliveryLogger`. `Debugln` passes `nil` fields to `formatMsg` [internal/log/log.go:L43-L48], so the message text ends with a bare **trailing tab** and there is **no `msg_id` JSON field** — the slot ID appears **inline in the message text** instead.
- **`delivery attempt #N`** = `dl.Debugf("delivery attempt #%d", meta.TriesCount+1)` [internal/target/queue/queue.go:L367]. Here `dl` is the `DeliveryLogger` [internal/target/queue/queue.go:L366], whose `Fields` already contains `msg_id` [internal/target/delivery.go:L13]; that is why this line — unlike the three bootstrap lines above — carries a `{"msg_id":...}` body even though `Debugf` itself passes `nil` fields [internal/log/log.go:L36-L41].
- **`using message ID = <id>-N`** = `dl.Debugf("using message ID = %s", msgMeta.ID)` [internal/target/queue/queue.go:L440]; the per-try `-N` suffix is built as `msgMeta.ID + "-" + strconv.Itoa(meta.TriesCount+1)` [internal/target/queue/queue.go:L439], so attempt #1 uses `...-1`, attempt #2 uses `...-2`, etc.
- **`delivery attempt failed`** = `dl.Error("delivery attempt failed", rcptErr, "rcpt", rcpt)` [internal/target/queue/queue.go:L384]; one line per failed recipient. `Logger.Error` folds the error text into `reason` [internal/log/log.go:L89-L104], so the fields render alphabetically as `msg_id`, `rcpt`, `reason`.
- **`will retry`** = `dl.Msg("will retry", ...)` [internal/target/queue/queue.go:L415-L418]; see Q7 for its fields and the delay computation.

**Partial-success variant.** In `TestQueueDelivery_MultipleAttempts` (deterministic test ID `cab2c2f3f939862a8eaea9f844143e36c0f2c1b5`), a recipient that succeeds on a given attempt additionally logs a `delivered` line — emitter `dl.Msg("delivered", "rcpt", rcpt, "attempt", meta.TriesCount+1)` [internal/target/queue/queue.go:L378] (fields render alphabetically as `attempt`, `msg_id`, `rcpt`):

```text
queue: delivered	{"attempt":1,"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","rcpt":"tester3@example.org"}
queue: delivered	{"attempt":3,"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","rcpt":"tester2@example.org"}
```

**State transition.** Attempt #1 runs, both recipients fail temporarily (`delivery attempt failed`), the queue schedules a retry (`will retry`), and attempt #2 runs — where, in `TestQueueDelivery_TemporaryFail`, delivery then succeeds.

> **Non-canonical caveat (labeled).** The `af8090c7…` / `cab2c2f3…` IDs are the 40-hex `sha1(t.Name())` test artifact (see Q3), **not** the production 8-hex `msg_id`.

---

## Q5 — MX authentication failure (error string, enhanced code, reply text)

**Command:**

```text
go test -mod=readonly -v -count=1 -run 'TestRemoteDelivery_AuthMX_Fail$|TestRemoteDelivery_TLSErrFallback$' ./internal/target/remote/ -test.debuglog
```

Observed top-level propagated error from `TestRemoteDelivery_AuthMX_Fail` (canonical content, harness tag `target.go:233:` stripped; **identical across runs**):

```text
delivery.AddRcpt test@example.invalid Failed to estabilish the MX record (mx.example.invalid.) authenticity map[domain:example.invalid reason:Failed to estabilish the MX record (mx.example.invalid.) authenticity smtp_code:550 smtp_enchcode:[5 4 0] smtp_msg:No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity target:remote]
```

**Client-facing values to report:**

- **Error message string:** `Failed to estabilish the MX record (mx.example.invalid.) authenticity` (the inner per-MX message), surfaced as the reply text `No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity` (the outer aggregated message).
- **SMTP code:** `smtp_code:550`.
- **SMTP enhanced status code (in `X.Y.Z` form):** **`5.4.0`**. The observed `smtp_enchcode:[5 4 0]` is Go's `%v` rendering of the same `EnhancedCode` value (a 3-element array printed with spaces and square brackets); the dotted `X.Y.Z` form is what `EnhancedCode.FormatLog()` produces (see below).

> **Observed source defect, reported as-is (not corrected):** the word **"estabilish"** is a misspelling of "establish" present in the maddy source [internal/target/remote/connect.go:L98]. It is reported here exactly as observed and is intentionally **not** fixed, because this is a read-only investigation.

**Two enhanced codes participate — both are reported:**

- **Inner (per-MX authenticity check).** When MX authentication is required and the MX is not authenticated, the code returns an `SMTPError` with `Code: 550`, `EnhancedCode{5, 7, 0}`, and message `fmt.Sprintf("Failed to estabilish the MX record (%s) authenticity", mx)` [internal/target/remote/connect.go:L96-L99]. (There is also an MTA-STS variant with `EnhancedCode{5, 7, 0}` and message `"Failed to estabilish the MX record authenticity (MTA-STS)"` [internal/target/remote/connect.go:L64-L68].) In `X.Y.Z` form this inner code is **`5.7.0`**.
- **Outer (aggregation when no MX is usable).** When every MX has been rejected, the connect loop bails out with a wrapper `SMTPError`: `Code: exterrors.SMTPCode(err, 451, 550)`, `EnhancedCode: exterrors.SMTPEnchCode(err, exterrors.EnhancedCode{0, 4, 0})`, and message `"No usable MXs, last err: " + lastErr.Error()` [internal/target/remote/connect.go:L204-L214]. This is the code/message actually propagated to the caller, observed as `smtp_code:550` and `smtp_enchcode:[5 4 0]` → **`5.4.0`**.

**Why `{0,4,0}` renders as `5.4.0` (`[5 4 0]`):**

- The structured fields `smtp_code` / `smtp_enchcode` / `smtp_msg` are produced by `SMTPError.Fields()` [internal/exterrors/smtp.go:L72-L79].
- `SMTPEnchCode` sets the first (class) digit of the enhanced code. Its body is [internal/exterrors/smtp.go:L122-L128]:

  ```go
  func SMTPEnchCode(err error, code EnhancedCode) EnhancedCode {
  	if IsTemporary(err) {
  		code[0] = 4
  	}
  	code[0] = 5
  	return code
  }
  ```

  In this observed path the class digit ends up **`5`**, so the default `EnhancedCode{0, 4, 0}` becomes `{5, 4, 0}`.
- `EnhancedCode.FormatLog()` renders that as the dotted string `fmt.Sprintf("%d.%d.%d", ec[0], ec[1], ec[2])` = **`5.4.0`** [internal/exterrors/smtp.go:L11-L13]. The bracketed `[5 4 0]` seen in the captured line is instead the Go `%v` rendering of the same underlying value (the test helper prints the error's field map with `%v`).

> **RFC 3463 background (context only, not required for the answer).** Enhanced status codes have the structure `class.subject.detail`, where a leading `5` denotes a permanent failure and a leading `4` a persistent-transient (retryable) one. `X.7.0` maps to "Other or undefined security status" (the inner `5.7.0`, a security/policy condition), and `X.4.0` maps to "Other or undefined network/routing status" (the aggregated `5.4.0`, a routing condition).

---

## Q6 — TLS→plaintext fallback log line

Observed from `TestRemoteDelivery_TLSErrFallback` (same command as Q5; canonical content, harness tag `output.go:41:` stripped; **identical across runs**):

```text
remote: TLS error, falling back to plaintext	{"domain":"example.invalid","msg_id":"2176ec5872ed2b87d832b4070e88232bd94ac7d3","mx":"mx.example.invalid.","reason":"smtpconn: tls: failed to verify certificate: x509: certificate signed by unknown authority"}
```

- **Emitter:** `rd.Log.Error("TLS error, falling back to plaintext", err, "mx", record.Host, "domain", domain)` [internal/target/remote/connect.go:L176-L177]. It is reached when the connection error is a `smtpconn.TLSError` **and** there is no auth error (`authErr == nil`) [internal/target/remote/connect.go:L175].
- **Fields (alphabetically sorted):** `domain`, `msg_id`, `mx`, `reason`. The `msg_id` is present because this is a delivery-scoped logger [internal/target/delivery.go:L13]; the `reason` field is the TLS error text folded in by `Logger.Error` [internal/log/log.go:L89-L104].
- **Prefix:** the `remote: ` prefix comes from the remote logger `Name: "remote"` [internal/target/remote/remote.go:L83].

> **Non-canonical caveat (labeled).** The 40-hex `msg_id` `2176ec58…` is the deterministic `sha1(t.Name())` test artifact (see Q3), not a production 8-hex `msg_id`. Because it is derived from the test name it is identical across runs.

---

## Q7 — Retry-delay JSON field name = `next_try_delay`

This is drawn from the same queue run as Q4 (command: `go test -mod=readonly -v -count=1 -run 'TestQueueDelivery_TemporaryFail$|TestQueueDelivery_MultipleAttempts$' ./internal/target/queue/ -test.debuglog -test.directlog`). The retry-scheduling line is `queue: will retry`, and the JSON field that holds the retry delay is **`next_try_delay`**:

```text
queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-693ns","rcpts":["tester1@example.org","tester2@example.org"]}
```

- **Emitter:** `dl.Msg("will retry", "attempts_count", meta.TriesCount, "next_try_delay", time.Until(nextTryTime), "rcpts", meta.To)` [internal/target/queue/queue.go:L415-L418]. The passed fields, plus the injected `msg_id`, render alphabetically as `attempts_count`, `msg_id`, `next_try_delay`, `rcpts`.
- **Value type & rendering:** `next_try_delay` is a Go `time.Duration` (the result of `time.Until(nextTryTime)`), rendered by `marshalOrderedJSON` via `.String()` [internal/log/orderedjson.go:L43-L44] — hence the `"-693ns"` string form.
- **Computation:** `nextTryTime := time.Now()` then `nextTryTime = nextTryTime.Add(q.initialRetryTime * time.Duration(math.Pow(q.retryTimeScale, float64(meta.TriesCount-1))))` [internal/target/queue/queue.go:L413-L414], with `initialRetryTime = 15 * time.Minute` and `retryTimeScale = 2` [internal/target/queue/queue.go:L185-L186]. The logged value is `time.Until(nextTryTime)`.
- **Sibling fields on the same line:** `attempts_count` and `rcpts`.

> **Test-timing artifact (labeled), not real backoff.** In these tests the retry backoff parameters are overridden so the scheduled time is essentially "now"; evaluating `time.Until(nextTryTime)` a moment later therefore yields a tiny **negative** duration. Observed values varied run-to-run — `-693ns`, `-697ns`, `-554ns`, `-339ns`, `-460ns`, `-446ns` — always a small negative nanosecond value. In production the backoff is **not** negative: it starts at `initialRetryTime = 15 * time.Minute` and scales by `retryTimeScale = 2` per attempt [internal/target/queue/queue.go:L185-L186]. The field *name* (`next_try_delay`) and its rendering are stable; only the tiny negative *value* is a test-timing artifact.

---

## Coverage summary

| Question | Answer (grounded in observed output) |
|----------|--------------------------------------|
| **Q1** | Success ends with `smtp: accepted` (only `msg_id`); each recipient logs `smtp: RCPT ok` (`msg_id`, `rcpt`) [internal/endpoint/smtp/smtp.go:L243, L334/L377]. Abort logs `smtp: aborted` (only `msg_id`) [internal/endpoint/smtp/smtp.go:L72], here preceded by `smtp: DATA error` (`msg_id`, `reason`) [internal/endpoint/smtp/smtp.go:L317]. |
| **Q2** | Log prefix module name = **`smtp`** — the logger `Name` set at [internal/endpoint/smtp/smtp.go:L495] and prepended at [internal/log/log.go:L180-L182]. |
| **Q3** | `msg_id` = **8 lowercase hex chars** from `GenerateMsgID` [internal/msgpipeline/msgid.go:L12-L16]. The 40-hex `sha1(t.Name())` value is a labeled test artifact [internal/testutils/target.go:L239-L245]. |
| **Q4** | Full sequence: `starting delivery` → `waiting/acquired semaphore` (bootstrap, no `msg_id` field) → `delivery attempt #1` → `using message ID …-1` → `delivery attempt failed` (per recipient) → `will retry` → `delivery attempt #2` [internal/target/queue/queue.go:L278/L282/L299/L367/L440/L384/L415-L418], plus the partial-success `delivered` line [internal/target/queue/queue.go:L378]. |
| **Q5** | Inner code **`5.7.0`** / `550`, message `Failed to estabilish the MX record (mx.example.invalid.) authenticity` [internal/target/remote/connect.go:L96-L99]; aggregated/propagated code **`5.4.0`** (`smtp_enchcode:[5 4 0]`) / `550`, reply text `No usable MXs, last err: …` [internal/target/remote/connect.go:L204-L214]. |
| **Q6** | `remote: TLS error, falling back to plaintext` with fields `domain`, `msg_id`, `mx`, `reason` [internal/target/remote/connect.go:L176-L177]. |
| **Q7** | Retry-delay field name = **`next_try_delay`** (a `time.Duration`) [internal/target/queue/queue.go:L415-L418]; near-zero negative value in tests is a labeled timing artifact. |

---

## Closing note — repository left unchanged

This was a strictly **read-only** investigation. The only file written is this document (`blitzy/documentation/maddy_26452dd8dd78.md`); no existing repository file was modified, and no observed source defect (such as the "estabilish" misspelling [internal/target/remote/connect.go:L98]) was corrected.

- The Go module cache and build cache were isolated under `/tmp` (`GOPATH=/tmp/gopath`, `GOCACHE=/tmp/gocache`), and every build/test command used **`-mod=readonly`**, so no build or test could mutate the checkout (not even `go.sum`).
- After all builds and test runs, `git status --porcelain` reported **no changes** to any tracked source file (the only untracked entry is this new document).
- No temporary observation scripts were created; the investigation ran through `go test` directly.
- **Run-to-run variance recap:** the SMTP `msg_id` (8-hex) and the queue `next_try_delay` differ each run; the deterministic 40-hex `sha1(t.Name())` IDs, the module-name prefixes (`smtp` / `queue` / `remote`), the enhanced status codes (`5.7.0`, `5.4.0` / `[5 4 0]`), and all log-line formats and JSON field names were stable across the runs performed.

