# maddy runtime logging & error behavior — onboarding notes (branch `maddy_26452dd8dd78`)

This document answers a set of onboarding questions about the [maddy](https://github.com/foxcpp/maddy) mail server's **runtime logging and error behavior**. Every answer is grounded in **actual test-execution output** that was captured by *running* the relevant Go test suites — not by reading source alone. Each behavioral claim is placed directly next to the exact command that produced it and the verbatim log line it emitted, with a `file:line` citation into the source.

---

## How this was produced (reproducibility)

- **Toolchain:** Go `1.13.15 linux/amd64` (the `go.mod` floor is `go 1.13` [go.mod:L3]) with `gcc` present for cgo.
- **Environment:** `GOFLAGS` **unset** (Go 1.13 rejects the `-mod=mod` flag), `CGO_ENABLED=1` (the tree pulls in `github.com/mattn/go-sqlite3`, which needs cgo), `GOPATH=/root/go`.
- **Observability flags (both mandatory):**
  - `-v` surfaces `t.Log` output from the tests.
  - `-test.debuglog` turns on `[debug]`-level lines. This flag is defined in the test logger as `flag.Bool("test.debuglog", …)` [internal/testutils/logger.go:L14] (its sibling `test.directlog` is at [internal/testutils/logger.go:L15]).
  - Without **both**, the pipeline / acceptance / TLS-fallback sequences are not fully visible.
- **Flag-placement pitfall:** the custom `-test.debuglog` flag **must appear after the package path**. If it is placed before the package path, `go test` silently runs the *root* module package and prints `?   github.com/foxcpp/maddy   [no test files]`. The correct form used throughout is:
  `go test -count=1 -v -run '<regex>' ./internal/<pkg>/ -test.debuglog`.
- **`-count=1`** is used to bypass the test result cache so every quote reflects a fresh run.
- All four suites **passed** on the run whose output is quoted below.

### A note on the exact shape of every quoted line

The test logger routes each maddy log line through Go's `t.Log`; the routing function lives at [internal/testutils/logger.go:L28-40] and prepends `[debug] ` to debug-level lines. Because `t.Log` annotates its output with the caller's `file:line`, the **raw** runner line looks like this (one real example, copied verbatim from `/tmp/smtp_succ.txt`):

```
    output.go:41: smtp: incoming message	{"msg_id":"fe0157be","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:41738"}
```

The `    output.go:41: ` prefix is the Go `testing` package's `t.Log` caller annotation, **not** part of maddy's log format. maddy's own log line is everything after it:

```
smtp: incoming message	{"msg_id":"fe0157be","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:41738"}
```

For readability, **every quote below shows the maddy log content proper** (the part after `output.go:41: `). The separator between the message and the JSON blob is a literal **TAB** character. This format is produced by `Logger.Msg`, which writes `"<Name>: <msg>"` followed by a tab and an ordered-JSON field map [internal/log/log.go:L72-76]; the error variant `Logger.Error` additionally merges `exterrors.Fields(err)` into the JSON and adds a `reason` field [internal/log/log.go:L88-101].

---

## `msg_id` comes in two different shapes — read this first

The questions ask about the observed `msg_id`. There are **two distinct shapes** in this codebase, and confusing them would mislead a reader, so they are distinguished up front. The delivery logger auto-attaches a `msg_id` field to every delivery-related log line via `DeliveryLogger`, which sets `fields["msg_id"] = msgMeta.ID` [internal/target/delivery.go:L8-15].

1. **SMTP endpoint id — 8 hex characters, non-deterministic per run.** The endpoint mints the id in `GenerateMsgID`, which reads **4** bytes from `crypto/rand` and hex-encodes them [internal/msgpipeline/msgid.go:L12-16]:

   ```go
   func GenerateMsgID() (string, error) {
       rawID := make([]byte, 4)
       _, err := rand.Read(rawID)
       return hex.EncodeToString(rawID), err
   }
   ```

   4 random bytes → 8 hex characters. Because the bytes come from `crypto/rand`, **the value changes on every run.** On the run quoted here the endpoint ids were `fe0157be` (success), `b9847559` (abort-in-DATA), and `71bf4a8d` (abort-at-logout). Your run will show different 8-hex values.

2. **Queue / remote id — deterministic 40-hex SHA-1.** The queue and remote tests drive delivery through the `testutils.DoTestDelivery*` helpers, which assign a deterministic `msg_id` computed as the **SHA-1 of the test name** — `IDRaw := sha1.Sum([]byte(t.Name()))` then `hex.EncodeToString(IDRaw[:])`, stored as `msgMeta.ID` [internal/testutils/target.go:L239-L245]. The id therefore comes from the *helper*, not from any particular target: the queue test drives a queue backed by an in-test `unreliableTarget`, and the remote test instantiates a `remote.Target`, yet both receive a name-derived id. Because it is a hash of the fixed test name, the id is a **40-hex-character** value that is **stable across runs** and therefore quotable as a constant: `af8090c7eb39f761862b1f027b4f2b0bb1ce86d1` = `SHA1("TestQueueDelivery_TemporaryFail")` (queue temporary-fail test) and `2176ec5872ed2b87d832b4070e88232bd94ac7d3` = `SHA1("TestRemoteDelivery_TLSErrFallback")` (remote TLS-fallback test).

> Throughout, endpoint `msg_id` values are flagged as **run-varying**, while queue/remote SHA-1 ids are flagged as **deterministic**.

---

## Group 1 — SMTP endpoint: success vs. abort, log format, module name, `msg_id`

All Group-1 output comes from the SMTP-endpoint test package `internal/endpoint/smtp/`.

### 1a — Exact log messages during a **successful** SMTP delivery

Command:

```
go test -count=1 -v -run '^TestSMTPDelivery$' ./internal/endpoint/smtp/ -test.debuglog 2>&1 | tee /tmp/smtp_succ.txt
```

Observed output (`TestSMTPDelivery`, `PASS`; this run's endpoint `msg_id=fe0157be`, run-varying):

```
smtp: listening on tcp://127.0.0.1:35613
smtp: incoming message	{"msg_id":"fe0157be","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:41738"}
smtp: RCPT ok	{"msg_id":"fe0157be","rcpt":"rcpt1@example.com"}
smtp: RCPT ok	{"msg_id":"fe0157be","rcpt":"rcpt2@example.com"}
smtp: accepted	{"msg_id":"fe0157be"}
```

The three success milestones and their call-sites:

- `smtp: incoming message …` — emitted by `s.log.Msg("incoming message", …)` [internal/endpoint/smtp/smtp.go:L128].
- `smtp: RCPT ok …` — emitted once per accepted recipient by `s.endp.Log.Msg("RCPT ok", "rcpt", to, "msg_id", s.msgMeta.ID)` [internal/endpoint/smtp/smtp.go:L243].
- `smtp: accepted …` — emitted when the whole transaction is accepted by `s.log.Msg("accepted", "msg_id", s.msgMeta.ID)` [internal/endpoint/smtp/smtp.go:L334].

A successful delivery therefore logs: **`incoming message` → `RCPT ok` (per recipient) → `accepted`**, all carrying the same `msg_id`.

### 1b — Exact log messages during an **aborted** SMTP delivery (mid-transaction)

Command:

```
go test -count=1 -v -run '^TestSMTPDelivery_Abort' ./internal/endpoint/smtp/ -test.debuglog 2>&1 | tee /tmp/smtp_abort.txt
```

There are two abort shapes. **Abort during DATA** (`TestSMTPDelivery_AbortData`, `PASS`; `msg_id=b9847559`) logs a `DATA error` and then `aborted`:

```
smtp: incoming message	{"msg_id":"b9847559","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:33592"}
smtp: RCPT ok	{"msg_id":"b9847559","rcpt":"test@example.com"}
smtp: DATA error	{"msg_id":"b9847559","reason":"unexpected EOF"}
smtp: aborted	{"msg_id":"b9847559"}
```

**Abort at logout** (`TestSMTPDelivery_AbortLogout`, `PASS`; `msg_id=71bf4a8d`) goes straight to `aborted` with no `DATA error`:

```
smtp: incoming message	{"msg_id":"71bf4a8d","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:33596"}
smtp: RCPT ok	{"msg_id":"71bf4a8d","rcpt":"test@example.com"}
smtp: aborted	{"msg_id":"71bf4a8d"}
```

Call-sites:

- `smtp: DATA error …` — emitted by `s.log.Error("DATA error", err, "msg_id", s.msgMeta.ID)` [internal/endpoint/smtp/smtp.go:L317]. Because it goes through `Logger.Error`, the JSON carries a `reason` field derived from the error text [internal/log/log.go:L88-101]; here the observed `reason` is `"unexpected EOF"`.
- `smtp: aborted …` — emitted by `s.log.Msg("aborted", "msg_id", s.msgMeta.ID)` [internal/endpoint/smtp/smtp.go:L72].

### 1c — Complete `RCPT ok` log line, **including all JSON fields**

The complete recipient-added line has **exactly two** JSON fields — `msg_id` and `rcpt` — in that order:

```
smtp: RCPT ok	{"msg_id":"fe0157be","rcpt":"rcpt1@example.com"}
```

This is produced verbatim by `s.endp.Log.Msg("RCPT ok", "rcpt", to, "msg_id", s.msgMeta.ID)` [internal/endpoint/smtp/smtp.go:L243]. Although the call lists `rcpt` before `msg_id`, the JSON serializer emits fields in a fixed (sorted) order, so `msg_id` appears first in the output. No other fields are present on this line.

### 1d — "Recipient added" line vs. "aborted" line (direct comparison)

Put side by side, from the same transaction the recipient-added line carries `msg_id` **and** `rcpt`, whereas the aborted line carries **only** `msg_id`:

```
smtp: RCPT ok	{"msg_id":"b9847559","rcpt":"test@example.com"}
smtp: aborted	{"msg_id":"b9847559"}
```

Differences, exactly:

- **Fields:** `RCPT ok` has `{msg_id, rcpt}` [internal/endpoint/smtp/smtp.go:L243]; `aborted` has `{msg_id}` only [internal/endpoint/smtp/smtp.go:L72].
- **Same `msg_id`** ties both lines to one transaction (`b9847559` above).
- For a **mid-DATA** abort, the `aborted` line is immediately preceded by a `DATA error` line whose `reason` is `"unexpected EOF"`:

  ```
  smtp: DATA error	{"msg_id":"b9847559","reason":"unexpected EOF"}
  smtp: aborted	{"msg_id":"b9847559"}
  ```

### 1e — Which **module name** appears in the log prefix for SMTP operations

The module name in the log prefix is **`smtp`** — visible as the leading token of every endpoint line above (e.g. `smtp: incoming message …`). The prefix is the logger's `Name`, assigned at endpoint construction from the module name:

```go
Log: log.Logger{Name: modName}
```

[internal/endpoint/smtp/smtp.go:L495]. The line format `"<Name>: <msg>\t{json}"` comes from `Logger.Msg` [internal/log/log.go:L72-76] (and `Logger.Error`, which additionally merges error fields and adds `reason` [internal/log/log.go:L88-101]).

There is also a distinct **pipeline sub-logger named `smtp/pipeline`**, set at [internal/endpoint/smtp/smtp.go:L586] (`Name: "smtp/pipeline"`). It is observable as debug lines in the same run, e.g.:

```
[debug] smtp/pipeline: sender sender@example.org matched by default rule	{"msg_id":"fe0157be"}
```

So: the SMTP endpoint logs under **`smtp`**, while its message pipeline logs under **`smtp/pipeline`**.

### 1f — Full transaction envelope (incoming / accepted / DATA-error)

The three envelope milestones, each with its observed line and call-site:

- **incoming** — `smtp: incoming message	{"msg_id":"fe0157be", …}` [internal/endpoint/smtp/smtp.go:L128].
- **accepted** — `smtp: accepted	{"msg_id":"fe0157be"}` [internal/endpoint/smtp/smtp.go:L334].
- **DATA error** — `smtp: DATA error	{"msg_id":"b9847559","reason":"unexpected EOF"}` [internal/endpoint/smtp/smtp.go:L317].

A clean transaction runs incoming → accepted; a mid-DATA failure runs incoming → DATA error → aborted (see 1a/1b for the full runs).

### 1g — The exact `msg_id` format observed

The endpoint `msg_id` is **8 hexadecimal characters** — for example `fe0157be` on this run (and `b9847559`, `71bf4a8d` for the two abort tests). It is produced by `GenerateMsgID`, which hex-encodes 4 `crypto/rand` bytes [internal/msgpipeline/msgid.go:L12-16], so **it is non-deterministic and changes on every run**. Re-running `TestSMTPDelivery` will show a different 8-hex value in the same positions:

```
smtp: accepted	{"msg_id":"fe0157be"}
```

(Contrast with the queue/remote **40-hex SHA-1** ids in Groups 2–4, which are deterministic — see the "two shapes" note above.)

---


## Group 2 — Queue: trace a message from acceptance through retry to delivery

All Group-2 output comes from the queue test package `internal/target/queue/`. The `msg_id` here is the **deterministic 40-hex SHA-1** `af8090c7eb39f761862b1f027b4f2b0bb1ce86d1`, so these quotes are stable across runs.

### 2a — Complete **sequence** of queue log messages (initial acceptance → retry → delivery)

Command:

```
go test -count=1 -v -run '^TestQueueDelivery_TemporaryFail$' ./internal/target/queue/ -test.debuglog 2>&1 | tee /tmp/queue_tf.txt
```

Observed sequence (`TestQueueDelivery_TemporaryFail`, `PASS`). The first attempt fails for both recipients, the queue schedules a retry, and the second attempt delivers:

```
queue: delivery attempt failed	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org","reason":"you shall not pass"}
queue: delivery attempt failed	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org","reason":"you shall not pass"}
queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-496ns","rcpts":["tester1@example.org","tester2@example.org"]}
queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org"}
queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org"}
```

So the full lifecycle is: **`delivery attempt failed` (per recipient) → `will retry` → `delivered` (per recipient, on `attempt` 2)**. The module prefix throughout is **`queue`**.

Supporting run — `TestQueueDelivery_MultipleAttempts` (`PASS`, `msg_id=cab2c2f3f939862a8eaea9f844143e36c0f2c1b5`) — shows the retry counter progressing across **more than one** retry (`attempts_count` 1 → 2) and delivery on a later attempt:

```
go test -count=1 -v -run '^TestQueueDelivery_MultipleAttempts$' ./internal/target/queue/ -test.debuglog
```

```
queue: delivered	{"attempt":1,"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","rcpt":"tester3@example.org"}
queue: delivery attempt failed	{"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","rcpt":"tester1@example.org","reason":"you shall not pass 1"}
queue: delivery attempt failed	{"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","rcpt":"tester2@example.org","reason":"you shall not pass 2"}
queue: will retry	{"attempts_count":1,"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","next_try_delay":"-412ns","rcpts":["tester2@example.org"]}
queue: delivery attempt failed	{"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","rcpt":"tester2@example.org","reason":"you shall not pass 3"}
queue: will retry	{"attempts_count":2,"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","next_try_delay":"-270ns","rcpts":["tester2@example.org"]}
queue: delivered	{"attempt":3,"msg_id":"cab2c2f3f939862a8eaea9f844143e36c0f2c1b5","rcpt":"tester2@example.org"}
```

### 2b — Exact log lines showing the delivery **attempt** and **failure**

The per-recipient **failure** line, quoted verbatim, includes the failing `reason`:

```
queue: delivery attempt failed	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org","reason":"you shall not pass"}
```

This is emitted by `dl.Error("delivery attempt failed", rcptErr, "rcpt", rcpt)` [internal/target/queue/queue.go:L384]. Because it uses `Logger.Error`, the failing error's text is surfaced as the `reason` field [internal/log/log.go:L88-101]; the test's injected error text is `"you shall not pass"`.

The corresponding **successful attempt** line names the attempt number (`attempt`):

```
queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org"}
```

emitted by `dl.Msg("delivered", "rcpt", rcpt, "attempt", meta.TriesCount+1)` [internal/target/queue/queue.go:L378]. The observed `"attempt":2` confirms delivery succeeded on the second try, after the first attempt failed.

### 2c — Exact log line showing **retry scheduling**

The retry-scheduling line, quoted verbatim:

```
queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-496ns","rcpts":["tester1@example.org","tester2@example.org"]}
```

emitted by `dl.Msg("will retry", "attempts_count", meta.TriesCount, "next_try_delay", time.Until(nextTryTime), "rcpts", meta.To)` [internal/target/queue/queue.go:L415-418]. Its JSON fields are `attempts_count` [queue.go:L416], `msg_id` (auto-attached), `next_try_delay` [queue.go:L417], and `rcpts` [queue.go:L418]. The `next_try_delay` value is discussed in detail under Group 4.

---


## Group 3 — Remote target: MX-authenticity failure and TLS→plaintext fallback

All Group-3 output comes from the remote-delivery test package `internal/target/remote/`.

### About the MX-authenticity observation (why a temporary harness was needed)

The existing test `TestRemoteDelivery_AuthMX_Fail` [internal/target/remote/mxauth_test.go:L16] asserts **only that an error occurred** — it does `if err == nil { t.Fatal("Expected an error, got none") }` and never inspects the error's code, enhanced code, or reply text. To capture the *exact* reply (items 3a–3c), a **temporary** observation test was authored, run once, and then **deleted** (the repository is left byte-for-byte unchanged; see the closing "Repository cleanliness" note). The temporary test was modeled on `TestRemoteDelivery_AuthMX_Fail`, set `requireMXAuth: true`, called `testutils.DoTestDeliveryErr(t, &tgt, "test@example.com", []string{"test@example.invalid"})` [internal/testutils/target.go:L232], and printed `err.Error()` plus `exterrors.Fields(err)` (which exposes `smtp_code`, `smtp_enchcode`, `smtp_msg` [internal/exterrors/smtp.go:L77-79]).

The **exact** temporary test is reproduced below in full so the observed output is reproducible. It was created at `internal/target/remote/zzobserve_test.go`, run once, and then **deleted** — it is **never committed** to the repository. It deliberately prints through `fmt.Printf` (not `t.Logf`) so the `OBSERVE` lines appear on stdout **without** the `t.Log` `file:line` prefix and can therefore be quoted verbatim. Because it is in `package remote`, it can read the unexported `Target` fields directly, and it re-uses the package-level `smtpPort` variable [internal/target/remote/remote.go:L41]:

```go
package remote

import (
	"fmt"
	"net"
	"testing"

	"github.com/foxcpp/go-mockdns"
	"github.com/foxcpp/maddy/internal/exterrors"
	"github.com/foxcpp/maddy/internal/testutils"
)

// TestZZObserve_AuthMXFailReply is a TEMPORARY observation harness (not part of
// the repository). It reproduces the MX-authenticity failure exercised by
// TestRemoteDelivery_AuthMX_Fail and prints the returned error's Error() string
// plus exterrors.Fields(err), so the exact reply (items 3a-3c) can be captured.
// It is created, run once, and then deleted; the working tree is left clean.
func TestZZObserve_AuthMXFailReply(t *testing.T) {
	be, srv := testutils.SMTPServer(t, "127.0.0.1:"+smtpPort)
	defer srv.Close()
	defer testutils.CheckSMTPConnLeak(t, srv)

	zones := map[string]mockdns.Zone{
		"example.invalid.": {
			MX: []net.MX{{Host: "mx.example.invalid.", Pref: 10}},
		},
		"mx.example.invalid.": {
			A: []string{"127.0.0.1"},
		},
	}
	resolver := &mockdns.Resolver{Zones: zones}

	tgt := Target{
		name:          "remote",
		hostname:      "mx.example.com",
		resolver:      &mockdns.Resolver{Zones: zones},
		dialer:        resolver.DialContext,
		extResolver:   nil,
		requireMXAuth: true,
		Log:           testutils.Logger(t, "remote"),
	}

	_, err := testutils.DoTestDeliveryErr(t, &tgt, "test@example.com", []string{"test@example.invalid"})
	if err == nil {
		t.Fatal("Expected an error, got none")
	}
	if be.MailFromCounter != 0 {
		t.Fatal("MAIL FROM issued for server failing authentication")
	}

	f := exterrors.Fields(err)
	ench := f["smtp_enchcode"].(exterrors.EnhancedCode)
	fmt.Printf("OBSERVE err.Error()=%q\n", err.Error())
	fmt.Printf("OBSERVE smtp_code=%v\n", f["smtp_code"])
	fmt.Printf("OBSERVE smtp_enchcode=%v\n", f["smtp_enchcode"])
	fmt.Printf("OBSERVE smtp_msg=%q\n", f["smtp_msg"])
	fmt.Printf("OBSERVE reply_line=%v %s %v\n", f["smtp_code"], ench.FormatLog(), f["smtp_msg"])
}
```

Command used to run the temporary harness (create the file above, run it, then delete it):

```
go test -count=1 -v -run '^TestZZObserve_AuthMXFailReply$' ./internal/target/remote/ -test.debuglog 2>&1 | tee /tmp/remote_mxauth.txt
rm -f internal/target/remote/zzobserve_test.go        # remove the temporary harness immediately after capture
```

Verbatim observed output (`PASS`):

```
OBSERVE err.Error()="Failed to estabilish the MX record (mx.example.invalid.) authenticity"
OBSERVE smtp_code=550
OBSERVE smtp_enchcode=[5 4 0]
OBSERVE smtp_msg="No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity"
OBSERVE reply_line=550 5.4.0 No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity
```

### 3a — Exact **error message string** when MX authenticity checks fail

The observed error string is (reported **exactly as observed**, including the source's own spelling):

```
OBSERVE err.Error()="Failed to estabilish the MX record (mx.example.invalid.) authenticity"
```

i.e. the string is `Failed to estabilish the MX record (mx.example.invalid.) authenticity`. **Note the source misspelling "estabilish"** (not "establish") — this is verbatim from the source and is deliberately **not corrected**. The per-MX error is built at [internal/target/remote/connect.go:L95-99]:

```go
return &exterrors.SMTPError{
    Code:         550,
    EnhancedCode: exterrors.EnhancedCode{5, 7, 0},
    Message:      fmt.Sprintf("Failed to estabilish the MX record (%s) authenticity", mx),
}
```

The message template `"Failed to estabilish the MX record (%s) authenticity"` — with `%s` = `mx.example.invalid.` — is at [internal/target/remote/connect.go:L98]. (`err.Error()` surfaces this inner text because `SMTPError.Error()` returns the wrapped cause's text — see the mechanism under 3b.) A near-identical MTA-STS variant string, `"Failed to estabilish the MX record authenticity (MTA-STS)"`, exists at [internal/target/remote/connect.go:L66-67] but is not the one exercised by this test.

### 3b — Precise SMTP **enhanced status code** (X.Y.Z)

The observed enhanced status code is **`5.4.0`**:

```
OBSERVE smtp_enchcode=[5 4 0]
```

**This is reported exactly as observed. It is `5.4.0`, not the `5.7.0` that a static read of [internal/target/remote/connect.go:L97] would suggest.** The reason is that the returned error is **not** the raw per-MX `SMTPError` (whose `EnhancedCode` is `{5, 7, 0}`); it is the **outer wrapper** built when no MX is usable [internal/target/remote/connect.go:L204-211]:

```go
return nil, &exterrors.SMTPError{
    Code:         exterrors.SMTPCode(err, 451, 550),
    EnhancedCode: exterrors.SMTPEnchCode(err, exterrors.EnhancedCode{0, 4, 0}),
    Message:      "No usable MXs, last err: " + lastErr.Error(),
    TargetName:   "remote",
    Err:          lastErr,
    …
}
```

Tracing the enhanced code, `SMTPEnchCode(err, {0,4,0})` [internal/exterrors/smtp.go:L122-128]:

```go
func SMTPEnchCode(err error, code EnhancedCode) EnhancedCode {
    if IsTemporary(err) {
        code[0] = 4
    }
    code[0] = 5
    return code
}
```

Starting from the default `{0, 4, 0}`, the failure is permanent so the `IsTemporary` branch is skipped, and the **next line unconditionally sets `code[0] = 5`** — yielding `{5, 4, 0}` → **`5.4.0`**. (Even for a temporary error, `code[0] = 4` would be immediately overwritten by the unconditional `code[0] = 5`; the class digit here is effectively always `5`. The middle/last digits `4` and `0` come straight from the `{0,4,0}` default, so the inner authenticity error's `7` is discarded.)

### 3c — Complete **reply text** from the test output

The complete reply line is:

```
OBSERVE reply_line=550 5.4.0 No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity
```

i.e. `550 5.4.0 No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity`. It is assembled from the three observed `SMTPError` fields:

- `smtp_code=550` — the numeric code, from `SMTPCode(err, 451, 550)` [internal/target/remote/connect.go:L205]; since the error is not temporary, `SMTPCode` returns the permanent `550` [internal/exterrors/smtp.go:L112-118]. (`SMTPError.Temporary()` is `Code/100 == 4` [internal/exterrors/smtp.go:L95-97], so `550` is permanent.)
- `smtp_enchcode=[5 4 0]` → `5.4.0` (see 3b).
- `smtp_msg="No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity"` — the outer wrapper's `Message` [internal/target/remote/connect.go:L207], which prefixes `"No usable MXs, last err: "` to the inner error's text.

Note the layering: the **inner** `5.7.0` authenticity `SMTPError` is retained as the wrapped cause (`Err: lastErr` [internal/target/remote/connect.go:L209]) and surfaced through `SMTPError.Error()`, which returns the wrapped error's text [internal/exterrors/smtp.go:L99-107]. That is why `err.Error()` shows the inner "Failed to estabilish…" message (item 3a) while the reply's enhanced code is the **outer** `5.4.0` (item 3b). This matches the HACKING.md "Error handling" convention that errors carry `smtp_code`/`smtp_enchcode`/`smtp_msg` and that the SMTP status code overrides `exterrors.IsTemporary()`.

### 3d — Exact **TLS→plaintext fallback** log message, **including all JSON fields**

Command:

```
go test -count=1 -v -run '^TestRemoteDelivery_TLSErrFallback$' ./internal/target/remote/ -test.debuglog 2>&1 | tee /tmp/remote_tls.txt
```

Observed line (`TestRemoteDelivery_TLSErrFallback`, `PASS`; deterministic `msg_id=2176ec5872ed2b87d832b4070e88232bd94ac7d3`):

```
remote: TLS error, falling back to plaintext	{"domain":"example.invalid","msg_id":"2176ec5872ed2b87d832b4070e88232bd94ac7d3","mx":"mx.example.invalid.","reason":"smtpconn: x509: certificate signed by unknown authority"}
```

The line has **all four** JSON fields: `domain` = `example.invalid`, `msg_id` = `2176ec5872ed2b87d832b4070e88232bd94ac7d3`, `mx` = `mx.example.invalid.`, and `reason` = `smtpconn: x509: certificate signed by unknown authority`. It is emitted by `rd.Log.Error("TLS error, falling back to plaintext", err, …)` [internal/target/remote/connect.go:L176]; because it uses `Logger.Error`, the underlying TLS error text is surfaced as `reason` [internal/log/log.go:L88-101]. The module prefix is **`remote`**.

---


## Group 4 — Queue retry: the scheduling line and the retry-delay field name

Both items re-use the `TestQueueDelivery_TemporaryFail` output from Group 2 (`msg_id=af8090c7eb39f761862b1f027b4f2b0bb1ce86d1`, deterministic).

Command:

```
go test -count=1 -v -run '^TestQueueDelivery_TemporaryFail$' ./internal/target/queue/ -test.debuglog 2>&1 | tee /tmp/queue_tf.txt
```

### 4a — The retry-scheduling log line

The retry-scheduling line is **`queue: will retry`**, quoted verbatim:

```
queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-496ns","rcpts":["tester1@example.org","tester2@example.org"]}
```

emitted by `dl.Msg("will retry", …)` [internal/target/queue/queue.go:L415].

### 4b — The exact JSON field name holding the retry delay

The field name is **`next_try_delay`**. It is set by `"next_try_delay", time.Until(nextTryTime)` [internal/target/queue/queue.go:L417]. In the line above the observed value is:

```
"next_try_delay":"-496ns"
```

Two facts about this value, reported exactly as observed:

- The **field name `next_try_delay` is the stable answer** — it is a literal string in the source [internal/target/queue/queue.go:L417].
- The **value is a small negative duration** (`-496ns` on this run; `-412ns` and `-270ns` were observed in `TestQueueDelivery_MultipleAttempts`). It is computed as `time.Until(nextTryTime)`, and in these accelerated tests the "next try time" has essentially already elapsed by the time it is logged, so the delta is a tiny **negative** nanosecond value that **varies per run**. (The AAP reference run showed `-381ns`.) The magnitude is therefore run-varying/near-zero-negative; the field name is the constant to rely on.

---

## Coverage pass — all 15 named items

| Item | Question | Answer (with evidence above) | Primary citation |
|------|----------|------------------------------|------------------|
| **1a** | Log messages during a **successful** delivery | `incoming message` → `RCPT ok` (×recipients) → `accepted` (see 1a block) | smtp.go:L128, L243, L334 |
| **1b** | Log messages during an **aborted** delivery (mid-transaction) | DATA abort: `DATA error` → `aborted`; logout abort: `aborted` only (see 1b blocks) | smtp.go:L317, L72 |
| **1c** | Complete `RCPT ok` line, **all JSON fields** | Exactly `{msg_id, rcpt}`: `smtp: RCPT ok\t{"msg_id":"…","rcpt":"…"}` | smtp.go:L243 |
| **1d** | "recipient added" vs "aborted" comparison | `RCPT ok` has `{msg_id,rcpt}`; `aborted` has `{msg_id}` only; same id; DATA abort preceded by `DATA error reason:"unexpected EOF"` | smtp.go:L243 vs L72 (+L317) |
| **1e** | **Module name** in the SMTP log prefix | **`smtp`** (endpoint); pipeline sub-logger **`smtp/pipeline`** | smtp.go:L495; L586 |
| **1f** | Full transaction envelope (incoming/accepted/DATA-error) | `incoming message` / `accepted` / `DATA error` lines all shown | smtp.go:L128, L334, L317 |
| **1g** | Exact **`msg_id` format** observed | **8 hex chars** (e.g. `fe0157be`), from 4 `crypto/rand` bytes; **non-deterministic** | msgpipeline/msgid.go:L12-16 |
| **2a** | Complete queue **sequence** acceptance→retry | `delivery attempt failed` → `will retry` → `delivered` (see 2a block) | queue.go:L384, L415, L378 |
| **2b** | Delivery **attempt** + **failure** lines | Failure: `delivery attempt failed … reason:"you shall not pass"`; success: `delivered … "attempt":2` | queue.go:L384; L378 |
| **2c** | **Retry-scheduling** line | `queue: will retry\t{"attempts_count":1,…,"next_try_delay":"…","rcpts":[…]}` | queue.go:L415-418 |
| **3a** | MX-authenticity **error string** | `Failed to estabilish the MX record (mx.example.invalid.) authenticity` (source spelling "estabilish") | connect.go:L98 |
| **3b** | Enhanced **status code** (X.Y.Z) | **`5.4.0`** (observed `smtp_enchcode=[5 4 0]`), NOT `5.7.0` — outer wrapper via `SMTPEnchCode` | connect.go:L204-211; exterrors/smtp.go:L122-128 |
| **3c** | Complete **reply text** | `550 5.4.0 No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity` | connect.go:L205-207; exterrors/smtp.go:L112-118 |
| **3d** | TLS→plaintext fallback line, **all JSON fields** | `remote: TLS error, falling back to plaintext\t{"domain","msg_id","mx","reason"}` | connect.go:L176 |
| **4a** | Retry-scheduling log line | `queue: will retry\t{…}` | queue.go:L415 |
| **4b** | Exact JSON field name for retry delay | **`next_try_delay`** (value a small negative duration, e.g. `-496ns`, run-varying) | queue.go:L417 |

All fifteen items (1a, 1b, 1c, 1d, 1e, 1f, 1g, 2a, 2b, 2c, 3a, 3b, 3c, 3d, 4a/4b) are addressed by name, each with a verbatim observed line, the command that produced it, and a `file:line` citation.

### Quirks reported exactly as observed (not "fixed")

- **"estabilish"** — the source misspelling in the MX-auth message [internal/target/remote/connect.go:L98] is preserved verbatim, never corrected to "establish".
- **`5.4.0`** — the observed enhanced code for the MX-auth failure is `5.4.0`, not the `5.7.0` a static read of [internal/target/remote/connect.go:L97] would imply; the wrapper + `SMTPEnchCode` mechanism [internal/exterrors/smtp.go:L122-128] is explained under 3b.

### `msg_id` shapes (restated)

- **Endpoint:** 8-hex, `crypto/rand`, **non-deterministic** per run [internal/msgpipeline/msgid.go:L12-16].
- **Queue/remote:** 40-hex SHA-1, **deterministic** (`af8090c7…` queue, `2176ec58…` TLS).

### Repository cleanliness

This document is the **only** persistent new file. The temporary observation test (`internal/target/remote/zzobserve_test.go`) used to capture items 3a–3c was deleted immediately after use, so it is **absent** from the repository. In the final state the working tree is **clean**, and the only path that differs between the baseline and `HEAD` is this document — `blitzy/documentation/maddy_26452dd8dd78.md`. No existing source, test, dependency, or build file was modified.

