# maddy Runtime Logging — Onboarding Q&A

> **Repository:** `github.com/foxcpp/maddy`
> **Commit:** `26452dd8dd787dc455278b0fdd296f4a5432c768`
> **Branch:** `maddy_26452dd8dd78`
> **Toolchain:** Go 1.19.13 (`linux/amd64`), `CGO_ENABLED=1`

This document onboards a new engineer to **maddy's runtime logging behavior** by
*actually building the server and running its existing Go tests with verbose
output*, then transcribing the **real, unedited** log records and resolving every
observed token back to the exact `file:line` that emits it. Every claim below is
grounded in captured runtime output (shown in full, with the command that
produced it) **and** a source citation with the reasoning behind it. Nothing here
is inferred from reading the source alone.

It answers four grouped questions:

- **Q1** — SMTP endpoint logging: success vs. abort, the `RCPT ok` record, the
  module prefix, and the `msg_id` format.
- **Q2** — Queue delivery trace: the complete acceptance-through-retry sequence
  and the attempt / failure / retry lines.
- **Q3** — Remote delivery: the MX-authenticity failure error string, its SMTP
  enhanced status code and reply text, and the TLS-to-plaintext fallback record.
- **Q4** — Queue retry scheduling: the retry-scheduling line and the JSON field
  name carrying the retry delay.

---

## 1. Methodology

### 1.1 Build and invocation commands

The module declares only a floor of `go 1.13` (`go.mod` L3) and its SQLite backend
requires CGO, so the investigation used **Go 1.19.13 with `CGO_ENABLED=1`**. The
canonical build recipe is `go build ./...` and the canonical test recipe from
`.build.yml` is `go test ./... -cover -race`. Every `go` command below passes
**`-mod=readonly`** so `go.mod`/`go.sum` can never be rewritten — this honors the
read-only rule of the task.

Canonical build (exit 0; the single warning is a benign `go-sqlite3` compiler
note and is expected — it comes from the C dependency, not maddy):

**Command:** `CGO_ENABLED=1 go build -mod=readonly ./...`

```text
# github.com/mattn/go-sqlite3
sqlite3-binding.c: In function ‘sqlite3SelectNew’:
sqlite3-binding.c:125322:10: warning: function may return address of local variable [-Wreturn-local-addr]
125322 |   return pNew;
       |          ^~~~
sqlite3-binding.c:125282:10: note: declared here
125282 |   Select standin;
       |          ^~~~~~~
```

The exact per-question invocation commands are listed in each section and repeated
in the coverage pass (§5).

### 1.2 Universal log-record shape

maddy uses a small in-house structured logger. Every record has the shape
`<module>: <message>\t{<JSON>}` — here `\t` denotes a single literal TAB byte:

- The message is written, then a single **TAB** (`'\t'`), then the JSON object —
  see `Logger.formatMsg` in `internal/log/log.go` (L135-L155): it calls
  `formatted.WriteString(msg)` (L138) then `formatted.WriteRune('\t')` (L139) then
  marshals the ordered JSON. When a record has no fields (e.g. `listening on …`),
  the message is still followed by the trailing TAB and an empty JSON tail.
- The `<module>:` prefix is the logger's `Name`, prepended by `Logger.log` via
  `s = l.Name + ": " + s` (`internal/log/log.go` L182).
- The default sink is **stderr**:
  `var DefaultLogger = Logger{Out: WriterOutput(os.Stderr, false)}`
  (`internal/log/log.go` L202).
- **JSON keys are sorted alphabetically.** `marshalOrderedJSON`
  (`internal/log/orderedjson.go` L16) collects the keys and calls
  `sort.Strings(order)` before emitting (L23). This is why the field order in the
  output is always alphabetical even when the calling code passes the fields in a
  different order.
- Value rendering: a `time.Duration` is rendered via its `.String()` method
  (`internal/log/orderedjson.go` L43-L44); a `time.Time` as
  `"2006-01-02T15:04:05.000"` (L41-L42); a `LogFormatter` (e.g. an SMTP enhanced
  code) via its `FormatLog()` (L45-L46).
- `Logger.Error(msg, err, fields…)` (`internal/log/log.go` L89) merges the error's
  structured fields, and if no `reason` field is present it sets `reason` to
  `err.Error()` (L98-L99). This is the origin of the `reason` field seen in
  `DATA error`, `delivery attempt failed`, and the TLS fallback record.

### 1.3 How records surface under `go test -v`

The test logger (`internal/testutils/logger.go` L13-L41) wires records into Go's
`t.Log()` by default via `log.FuncOutput` (L28), prefixing `"[debug] "` to
debug-level records (L32). Two harness flags control visibility:

- `-test.debuglog` (L14) — enable `[debug]`-level records (off by default).
- `-test.directlog` (L15) — log to stderr instead of `t.Log()`.

Because records go through `t.Log()`, `go test -v` prints each one tagged with the
source location of the log call. maddy's own records therefore appear tagged
`output.go:41:` (the `FuncOutput` write site), while the test-harness helper
messages appear tagged `target.go:166:`/`target.go:233:` or `queue_test.go:…`.
**These tags are the `-v` presentation layer, not part of the record itself.** The
`-v` flag and `-test.debuglog` are the software's own visibility controls; they do
not alter the behavior being logged.

---

## 2. Q1 — SMTP endpoint logging

**Package:** `internal/endpoint/smtp` · **Module prefix:** `smtp`

### 2.1 Where the `smtp` prefix comes from (answers Q1d)

The endpoint is registered under three module names in `init()`
(`internal/endpoint/smtp/smtp.go` L715-L717):

```go
module.RegisterEndpoint("smtp", New)
module.RegisterEndpoint("submission", New)
module.RegisterEndpoint("lmtp", New)
```

`New` sets the endpoint logger's `Name` to the module name it was registered under
— `Log: log.Logger{Name: modName}` (`smtp.go` L495) — and each session copies that
logger with `log: endp.Log` (`smtp.go` L675). So for an endpoint configured as
`smtp`, the module name in the log prefix for SMTP operations is **`smtp`**. (It
would read `submission` or `lmtp` for those endpoint types; the default SMTP
delivery endpoint tested here is `smtp`.) The tests confirm the literal `smtp:`
prefix on every record below.

### 2.2 Successful delivery

**Command:** `go test -mod=readonly -v -count=1 -run '^TestSMTPDelivery$' ./internal/endpoint/smtp/`
(driver: `TestSMTPDelivery`, `internal/endpoint/smtp/smtp_test.go` L122)

```text
=== RUN   TestSMTPDelivery
    output.go:41: smtp: listening on tcp://127.0.0.1:19386	
    output.go:41: smtp: incoming message	{"msg_id":"1e451b6a","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:52288"}
    output.go:41: smtp: RCPT ok	{"msg_id":"1e451b6a","rcpt":"rcpt1@example.com"}
    output.go:41: smtp: RCPT ok	{"msg_id":"1e451b6a","rcpt":"rcpt2@example.com"}
    output.go:41: smtp: accepted	{"msg_id":"1e451b6a"}
--- PASS: TestSMTPDelivery (0.00s)
PASS
ok  	github.com/foxcpp/maddy/internal/endpoint/smtp	0.005s
```

### 2.3 Abort mid-transaction (mid-DATA)

**Command:** `go test -mod=readonly -v -count=1 -run '^TestSMTPDelivery_AbortData$' ./internal/endpoint/smtp/`
(driver: `TestSMTPDelivery_AbortData`, `smtp_test.go` L360)

```text
=== RUN   TestSMTPDelivery_AbortData
    output.go:41: smtp: listening on tcp://127.0.0.1:52235	
    output.go:41: smtp: incoming message	{"msg_id":"37d04764","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:43118"}
    output.go:41: smtp: RCPT ok	{"msg_id":"37d04764","rcpt":"test@example.com"}
    output.go:41: smtp: DATA error	{"msg_id":"37d04764","reason":"unexpected EOF"}
    output.go:41: smtp: aborted	{"msg_id":"37d04764"}
--- PASS: TestSMTPDelivery_AbortData (0.25s)
PASS
ok  	github.com/foxcpp/maddy/internal/endpoint/smtp	0.255s
```

### 2.4 Abort at logout (control case)

**Command:** `go test -mod=readonly -v -count=1 -run '^TestSMTPDelivery_AbortLogout$' ./internal/endpoint/smtp/`
(driver: `TestSMTPDelivery_AbortLogout`, `smtp_test.go` L398)

```text
=== RUN   TestSMTPDelivery_AbortLogout
    output.go:41: smtp: listening on tcp://127.0.0.1:64663	
    output.go:41: smtp: incoming message	{"msg_id":"eb8b66bc","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:40372"}
    output.go:41: smtp: RCPT ok	{"msg_id":"eb8b66bc","rcpt":"test@example.com"}
    output.go:41: smtp: aborted	{"msg_id":"eb8b66bc"}
--- PASS: TestSMTPDelivery_AbortLogout (0.25s)
PASS
ok  	github.com/foxcpp/maddy/internal/endpoint/smtp	0.256s
```

### 2.5 Answers

**Q1a / Q1c — success vs. abort log messages.**
Both transactions open identically with `smtp: incoming message`, whose fields are
`msg_id`, `sender`, `src_host`, `src_ip` (and `username` too, when the client
authenticated). This record is emitted at `internal/endpoint/smtp/smtp.go`
L127-L142 — there are two branches, the authenticated one (`s.log.Msg` at L128,
which adds `username`) and the anonymous one (`s.log.Msg` at L136). The
transactions then diverge at the **terminal record**:

- **Success** ends with `smtp: accepted\t{"msg_id":"…"}`, emitted by
  `s.log.Msg("accepted", "msg_id", s.msgMeta.ID)` after the delivery commits
  (`smtp.go` L334). Observed: `smtp: accepted\t{"msg_id":"1e451b6a"}`.
- **Abort** ends with `smtp: aborted\t{"msg_id":"…"}`, emitted by
  `s.log.Msg("aborted", "msg_id", s.msgMeta.ID)` inside `Session.abort`
  (`smtp.go` L72).

The two abort variants further clarify the comparison (Q1c):

- **Abort mid-DATA** inserts an error record `smtp: DATA error\t{…,"reason":…}`
  *before* `aborted`. It is emitted by `s.log.Error("DATA error", err, "msg_id", …)`
  (`smtp.go` L317). The observed `reason` is `"unexpected EOF"`, which is
  `err.Error()` captured by `Logger.Error` (`internal/log/log.go` L98-L99) because
  the underlying error carried no explicit `reason` field.
- **Abort at logout** has **no `DATA error`** record — the sequence is simply
  `incoming message` → `RCPT ok` → `aborted`. This isolates the terminal `aborted`
  record as the true success/abort discriminator; the `DATA error` line appears
  only when the abort is caused by a mid-DATA failure.

In short: `incoming message` is common to both; the discriminator is `accepted`
(success) vs. `aborted` (abort), with an optional preceding `DATA error` when the
abort originates mid-DATA.

**Q1b — the record when a recipient is successfully added.**
The complete log line is `smtp: RCPT ok\t{"msg_id":"1e451b6a","rcpt":"rcpt1@example.com"}`.
It contains **exactly two JSON fields** — `msg_id` and `rcpt` — and is emitted by
`s.endp.Log.Msg("RCPT ok", "rcpt", to, "msg_id", s.msgMeta.ID)` at
`internal/endpoint/smtp/smtp.go` L243. Note that the source passes `rcpt` **first**
and `msg_id` second, yet the output shows `msg_id` before `rcpt`: this is the
alphabetical key sort from `marshalOrderedJSON` (`internal/log/orderedjson.go`
L23) in action. The successful transaction emits one `RCPT ok` per accepted
recipient (two in `TestSMTPDelivery`).

**Q1d — module name in the log prefix.** **`smtp`** (see §2.1; `smtp.go` L495 and
L715-L717).

**Q1e — the `msg_id` format.**
It is **exactly 8 lowercase hexadecimal characters** matching `^[0-9a-f]{8}$`.
It is produced by `GenerateMsgID()` (`internal/msgpipeline/msgid.go` L12-L16),
which reads **4 random bytes** from `crypto/rand` and hex-encodes them
(4 bytes × 2 hex chars = 8 chars):

```go
func GenerateMsgID() (string, error) {
	rawID := make([]byte, 4)
	_, err := rand.Read(rawID)
	return hex.EncodeToString(rawID), err
}
```

The value is **random per message**. Running the exact same command four times
produced four different values, each still matching the format:

| Run | `msg_id` | Matches `^[0-9a-f]{8}$` |
|-----|----------|--------------------------|
| 1 | `1e451b6a` | yes |
| 2 | `7c08d713` | yes |
| 3 | `5a5ca2f7` | yes |
| 4 | `71e45ff4` | yes |

The **format is stable** (always 8 lowercase hex chars); only the **value** varies,
by design. (The `msg_id` seen in the queue and remote tests below is different — a
40-character SHA-1 — because those tests inject the ID through a mock harness; see
§3.1.)

---

## 3. Q2 & Q4 — Queue delivery trace and retry scheduling

**Package:** `internal/target/queue` · **Module prefix:** `queue`

The queue logger's `Name` is set to `"queue"` at `internal/target/queue/queue.go`
L188 (`Log: log.Logger{Name: "queue"}`), so every record below carries the
`queue:` prefix.

### 3.1 The `msg_id` in these tests

In the queue (and remote) tests the message ID is **not** the 8-hex endpoint value
of Q1e. The mock delivery harness derives a deterministic ID as the **SHA-1 of the
test name**: `IDRaw := sha1.Sum([]byte(t.Name()))` then
`hex.EncodeToString(IDRaw[:])` (`internal/testutils/target.go` L184-L185). For
`TestQueueDelivery_TemporaryFail` that is the 40-hex-character constant
`af8090c7eb39f761862b1f027b4f2b0bb1ce86d1`, **identical across every run**. Each
retry attempt appends an attempt suffix (`…-1`, `…-2`), visible in the
`using message ID = …` debug records. This is a test-harness artifact and is
called out here to avoid conflating it with the canonical random endpoint `msg_id`
of Q1e.

### 3.2 Non-debug run

**Command:** `go test -mod=readonly -v -count=1 -run '^TestQueueDelivery_TemporaryFail$' ./internal/target/queue/`
(driver: `TestQueueDelivery_TemporaryFail`, `queue_test.go` L297)

```text
=== RUN   TestQueueDelivery_TemporaryFail
=== PAUSE TestQueueDelivery_TemporaryFail
=== CONT  TestQueueDelivery_TemporaryFail
    target.go:166: -- tgt.Start tester@example.com
    target.go:166: -- delivery.AddRcpt tester1@example.org
    target.go:166: -- delivery.AddRcpt tester2@example.org
    target.go:166: -- delivery.Body
    target.go:166: -- delivery.Commit
    output.go:41: queue: delivery attempt failed	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org","reason":"you shall not pass"}
    output.go:41: queue: delivery attempt failed	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org","reason":"you shall not pass"}
    output.go:41: queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-463ns","rcpts":["tester1@example.org","tester2@example.org"]}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org"}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org"}
    queue_test.go:38: --- queue.Close
--- PASS: TestQueueDelivery_TemporaryFail (0.05s)
PASS
ok  	github.com/foxcpp/maddy/internal/target/queue	0.053s
```

### 3.3 Full `[debug]` sequence — answers Q2a

**Command:** `go test -mod=readonly -v -count=1 -run '^TestQueueDelivery_TemporaryFail$' ./internal/target/queue/ -args -test.debuglog`

```text
=== RUN   TestQueueDelivery_TemporaryFail
=== PAUSE TestQueueDelivery_TemporaryFail
=== CONT  TestQueueDelivery_TemporaryFail
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
    output.go:41: queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-467ns","rcpts":["tester1@example.org","tester2@example.org"]}
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
--- PASS: TestQueueDelivery_TemporaryFail (0.02s)
PASS
ok  	github.com/foxcpp/maddy/internal/target/queue	0.024s
```

### 3.4 Answers

**Q2a — the complete sequence from initial acceptance through retry.**
Reading the `[debug]` capture top-to-bottom, the maddy `queue:` records appear in
this order (the interleaved `-- tgt.Start` / `-- delivery.*` lines tagged
`target.go:166` are the mock target's own harness logs, reproduced verbatim in the
block above):

1. `[debug] queue: delivery target: *queue.unreliableTarget`
2. `[debug] queue: starting delivery for af8090c7…`
3. `[debug] queue: waiting on delivery semaphore for af8090c7…`
4. `[debug] queue: delivery semaphore acquired for af8090c7…`
5. `[debug] queue: delivery attempt #1` — `dl.Debugf("delivery attempt #%d", meta.TriesCount+1)` (`queue.go` L367)
6. `[debug] queue: using message ID = af8090c7…-1`
7. `[debug] queue: target.Start OK`
8. `[debug] queue: delivery.AddRcpt tester1@example.org OK`
9. `[debug] queue: delivery.AddRcpt tester2@example.org OK`
10. `[debug] queue: delivery.Body failed: you shall not pass`
11. `[debug] queue: delivery.Body OK`
12. `[debug] queue: delivery.Abort (all recipients failed)`
13. `[debug] queue: failures: permanently: [], temporary: [tester1@example.org tester2@example.org], errors: map[…]`
14. `queue: delivery attempt failed` — once per recipient, non-debug level (`queue.go` L384)
15. `queue: will retry` — the retry-scheduling record (`queue.go` L415-L418)
16. *(attempt #2 repeats the connect steps)* `starting delivery` → `waiting on delivery semaphore` → `delivery semaphore acquired` → `delivery attempt #2` → `using message ID = …-2` → `target.Start OK` → `delivery.AddRcpt … OK` (×2)
17. `[debug] queue: delivery.Body OK`
18. `[debug] queue: delivery.Commit OK`
19. `[debug] queue: failures: permanently: [], temporary: [], errors: map[]` (empty — success)
20. `queue: delivered` — once per recipient, with `"attempt":2` (`queue.go` L378)
21. `[debug] queue: removed message from disk`

A behavioral note reported exactly as observed: in attempt #1 both a
`delivery.Body failed: you shall not pass` **and** a subsequent `delivery.Body OK`
debug line appear. This is a property of the test's mock `unreliableTarget`, which
fails the first body attempt for the recipients and then succeeds; the queue
records both the failure and the follow-up call verbatim.

**Q2b — the exact attempt / failure / retry lines.**

- **Delivery attempt** (debug-level), from `dl.Debugf("delivery attempt #%d", meta.TriesCount+1)` (`queue.go` L367):
  `[debug] queue: delivery attempt #1\t{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}`
- **Delivery failure**, from `dl.Error("delivery attempt failed", rcptErr, "rcpt", rcpt)` (`queue.go` L384). Its `reason` is the recipient error text, injected by `Logger.Error` (`log.go` L98-L99):
  `queue: delivery attempt failed\t{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org","reason":"you shall not pass"}`
- **Retry scheduling**, from `dl.Msg("will retry", "attempts_count", meta.TriesCount, "next_try_delay", time.Until(nextTryTime), "rcpts", meta.To)` (`queue.go` L415-L418):
  `queue: will retry\t{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-463ns","rcpts":["tester1@example.org","tester2@example.org"]}`

(For completeness, the eventual success line is `queue: delivered\t{"attempt":2,…}`,
emitted by `dl.Msg("delivered", "rcpt", rcpt, "attempt", meta.TriesCount+1)`,
`queue.go` L378.)

**Q4a — the retry-scheduling line.** It is the `queue: will retry {…}` record
quoted just above (`queue.go` L415-L418).

**Q4b — the JSON field name carrying the retry delay.** It is **`next_try_delay`**
(`queue.go` L417). The value is `time.Until(nextTryTime)`, a Go `time.Duration`,
which the logger renders through `Duration.String()` (`internal/log/orderedjson.go`
L43-L44).

**Canonical vs. test-configuration (mandatory caveat).** The observed
`next_try_delay` is a **small negative value** and is a **non-canonical
test-configuration artifact**. The test overrides the retry schedule with
`q.initialRetryTime = 0` and `q.retryTimeScale = 1` (`queue_test.go` L50-L51), so
the computed `nextTryTime = time.Now()` (`queue.go` L413-L414) has already passed
by the time `time.Until(nextTryTime)` runs — yielding a slightly negative
duration. Across 8 identical runs the value varied while the **field name and its
format never changed**:

| Run | `next_try_delay` |
|-----|------------------|
| 1 | `-463ns` |
| 2 | `-552ns` |
| 3 | `-547ns` |
| 4 | `-547ns` |
| 5 | `-588ns` |
| 6 | `-582ns` |
| 7 | `-807ns` |
| 8 | `-507ns` |

(The `[debug]` run in §3.3 independently produced `-467ns`, consistent with this
distribution.) All values are in the tens-to-hundreds of nanoseconds, negative.

In the **default/canonical** configuration the defaults are
`initialRetryTime: 15 * time.Minute` and `retryTimeScale: 2` (`queue.go`
L185-L186), and the delay follows
`nextTryTime.Add(q.initialRetryTime * time.Duration(math.Pow(q.retryTimeScale, float64(meta.TriesCount-1))))`
(`queue.go` L413-L414) — i.e. approximately `15m0s`, then `30m0s`, then `1h0m0s`,
… for successive attempts, rendered as those `Duration.String()` forms. **The
field name `next_try_delay` is identical in every configuration; only the value
differs.**

---

## 4. Q3 — Remote delivery: MX-authentication failure & TLS fallback

**Package:** `internal/target/remote` · **Module prefix:** `remote`

The remote target logger's `Name` is `"remote"` (`internal/target/remote/remote.go`
L83, `Log: log.Logger{Name: "remote"}`).

### 4.1 MX-authenticity failure — answers Q3a / Q3b

**Command:** `go test -mod=readonly -v -count=1 -run '^TestRemoteDelivery_AuthMX_Fail$' ./internal/target/remote/`
(driver: `TestRemoteDelivery_AuthMX_Fail`, `internal/target/remote/mxauth_test.go` L16)

```text
=== RUN   TestRemoteDelivery_AuthMX_Fail
    target.go:233: -- tgt.Start test@example.com
    target.go:233: -- delivery.AddRcpt test@example.invalid
    target.go:233: -- ... delivery.AddRcpt test@example.invalid Failed to estabilish the MX record (mx.example.invalid.) authenticity map[domain:example.invalid reason:Failed to estabilish the MX record (mx.example.invalid.) authenticity smtp_code:550 smtp_enchcode:[5 4 0] smtp_msg:No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity target:remote]
    target.go:233: -- delivery.Abort
--- PASS: TestRemoteDelivery_AuthMX_Fail (0.00s)
PASS
ok  	github.com/foxcpp/maddy/internal/target/remote	0.005s
```

The `map[…]` dump on the `-- ... delivery.AddRcpt` line is the delivery error's
structured fields, printed by the test harness via
`t.Log("-- ... delivery.AddRcpt", rcpt, err, exterrors.Fields(err))`
(`internal/testutils/target.go`, tag `target.go:233`). The raw `smtp_enchcode`
prints as the Go array literal `[5 4 0]` there because it is dumped with `%v`; in
maddy's own structured log output it renders as `5.4.0` (see below).

**Q3a — the exact MX-authenticity error string.** Reproduced verbatim (including
the source's misspelling **"estabilish"**, which is emitted as-is and is *not*
corrected here):

```text
Failed to estabilish the MX record (mx.example.invalid.) authenticity
```

It is produced by
`fmt.Sprintf("Failed to estabilish the MX record (%s) authenticity", mx)` with
`Code: 550` and `EnhancedCode: {5, 7, 0}` at `internal/target/remote/connect.go`
L94-L99 (the format string is at L98; `%s` is the MX host, here
`mx.example.invalid.`). A sibling MTA-STS non-match variant exists on the same code
path, `"Failed to estabilish the MX record authenticity (MTA-STS)"` (`connect.go`
L66-L67), also with `{5, 7, 0}`.

**Q3b — SMTP enhanced status code (X.Y.Z) and the complete reply text.** Two
distinct enhanced codes exist on this path, and both are reported here because the
error is wrapped:

- **Inner** — the MX-authenticity check itself returns enhanced code **`5.7.0`**
  with message *"Failed to estabilish the MX record (mx.example.invalid.)
  authenticity"* (`connect.go` L96-L98).
- **Surfaced** — once the single MX is exhausted, the delivery error actually
  returned to the caller is wrapped as **"No usable MXs, last err: …"** with
  enhanced code **`5.4.0`** and `smtp_code` **`550`**, `target` **`remote`**
  (`connect.go` L204-L214). This is the code the client would receive.

The observed fields dump therefore reads (extracted from the capture above):

```text
smtp_code:550   smtp_enchcode:[5 4 0]   smtp_msg:No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity   target:remote
```

So the **precise enhanced status code in X.Y.Z form is `5.4.0`** (surfaced), and
the **complete reply text** (`smtp_msg`) is:

```text
No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity
```

Why `5.4.0` and not the inner `5.7.0`, and how X.Y.Z is formed:

- **Rendering.** An `EnhancedCode` is a `LogFormatter`; its `FormatLog()` returns
  `fmt.Sprintf("%d.%d.%d", ec[0], ec[1], ec[2])` (`internal/exterrors/smtp.go`
  L11-L13). So `{5,4,0}` renders as `5.4.0`. The raw `[5 4 0]` in the test dump is
  just the un-formatted `%v` of the array value.
- **Leading digit.** The surfaced code is seeded `{0, 4, 0}` and passed through
  `exterrors.SMTPEnchCode` (`connect.go` L206). That helper
  (`internal/exterrors/smtp.go` L122-L128) assigns `code[0] = 4` in the
  `if IsTemporary(err)` branch (L124) but then unconditionally sets `code[0] = 5`
  on the next line (L126), so the first digit becomes `5`, turning `{0,4,0}` into
  `{5,4,0}` (a permanent failure).
- **Precedence (inner vs. surfaced).** `exterrors.Fields` (`internal/exterrors/fields.go`
  L28) walks the error chain from the outer error inward and keeps the **first**
  value seen for each key — its own comment states *"Outer errors override fields
  of the inner ones."* (L35, with the `continue` guard at L38). Because the outer
  "No usable MXs" wrapper is visited first, its `smtp_enchcode` (`5.4.0`) wins over
  the inner check's `5.7.0`. The fields themselves come from `SMTPError.Fields()`,
  which exposes `smtp_code` (L77), `smtp_enchcode` (L78), `smtp_msg` (L79), plus
  `target` (L84) and `reason` (L86-L89) (`internal/exterrors/smtp.go` L72-L92).
  Separately, `err.Error()` prints the **inner** text ("Failed to estabilish…")
  because `SMTPError.Error()` returns the inner error when `Reason` is empty
  (`exterrors/smtp.go` L99-L107) — which is why the harness line begins with the
  inner string while `smtp_msg` carries the outer reply text.
  `SMTPError.Temporary()` maps 4xx codes to retryable via `se.Code/100 == 4`
  (`exterrors/smtp.go` L95-L96); here the code is 550, i.e. permanent.

This result was stable across ≥2 runs (identical output each time).

### 4.2 TLS-to-plaintext fallback — answers Q3c

**Command:** `go test -mod=readonly -v -count=1 -run '^TestRemoteDelivery_TLSErrFallback$' ./internal/target/remote/`
(driver: `TestRemoteDelivery_TLSErrFallback`, `internal/target/remote/remote_test.go` L852)

```text
=== RUN   TestRemoteDelivery_TLSErrFallback
    target.go:166: -- tgt.Start test@example.com
    target.go:166: -- delivery.AddRcpt test@example.invalid
    output.go:41: remote: TLS error, falling back to plaintext	{"domain":"example.invalid","msg_id":"2176ec5872ed2b87d832b4070e88232bd94ac7d3","mx":"mx.example.invalid.","reason":"smtpconn: x509: certificate signed by unknown authority"}
    target.go:166: -- delivery.Body
    target.go:166: -- delivery.Commit
--- PASS: TestRemoteDelivery_TLSErrFallback (0.01s)
PASS
ok  	github.com/foxcpp/maddy/internal/target/remote	0.018s
```

**Q3c — the exact TLS→plaintext fallback log, with all JSON fields.**

```text
remote: TLS error, falling back to plaintext	{"domain":"example.invalid","msg_id":"2176ec5872ed2b87d832b4070e88232bd94ac7d3","mx":"mx.example.invalid.","reason":"smtpconn: x509: certificate signed by unknown authority"}
```

Its four alphabetically-sorted JSON fields are **`domain`, `msg_id`, `mx`,
`reason`**. It is emitted by
`rd.Log.Error("TLS error, falling back to plaintext", err, "mx", record.Host, "domain", domain)`
at `internal/target/remote/connect.go` L176-L177. The `mx` and `domain` are passed
explicitly; `msg_id` is contributed by the delivery logger; and `reason` is the
underlying TLS error text pulled from the error object by `Logger.Error`
(`internal/log/log.go` L89, L98-L99) — here
`"smtpconn: x509: certificate signed by unknown authority"`. The fallback fires
**only** when the connection error is a `smtpconn.TLSError` **and** MX
authentication does not object to plaintext (`authErr == nil`), guarded at
`connect.go` L175. The `msg_id` is the deterministic 40-hex SHA-1 of the test name
(§3.1), not the 8-hex endpoint value of Q1e.

To show the before/after state around the fallback, the same test with
`-test.debuglog` reveals the plaintext reconnect succeeding immediately after:

**Command:** `go test -mod=readonly -v -count=1 -run '^TestRemoteDelivery_TLSErrFallback$' ./internal/target/remote/ -args -test.debuglog`

```text
=== RUN   TestRemoteDelivery_TLSErrFallback
    target.go:166: -- tgt.Start test@example.com
    target.go:166: -- delivery.AddRcpt test@example.invalid
    output.go:41: [debug] remote: trying	{"domain":"example.invalid","msg_id":"2176ec5872ed2b87d832b4070e88232bd94ac7d3","mx":"mx.example.invalid."}
    output.go:41: remote: TLS error, falling back to plaintext	{"domain":"example.invalid","msg_id":"2176ec5872ed2b87d832b4070e88232bd94ac7d3","mx":"mx.example.invalid.","reason":"smtpconn: x509: certificate signed by unknown authority"}
    output.go:41: [debug] remote: connected	{"domain":"example.invalid","msg_id":"2176ec5872ed2b87d832b4070e88232bd94ac7d3","mx":"mx.example.invalid."}
    output.go:41: [debug] remote: connected	{"msg_id":"2176ec5872ed2b87d832b4070e88232bd94ac7d3","remote_server":"mx.example.invalid."}
    target.go:166: -- delivery.Body
    target.go:166: -- delivery.Commit
    output.go:41: [debug] remote: disconnected from mx.example.invalid.	{"msg_id":"2176ec5872ed2b87d832b4070e88232bd94ac7d3"}
--- PASS: TestRemoteDelivery_TLSErrFallback (0.01s)
PASS
ok  	github.com/foxcpp/maddy/internal/target/remote	0.019s
```

Observed exactly as printed: a `[debug] remote: trying` record precedes the
fallback; then, after the `TLS error, falling back to plaintext` record, **two**
`[debug] remote: connected` records appear (one from the remote target carrying
`domain`/`msg_id`/`mx`, one from the underlying `smtpconn` carrying
`msg_id`/`remote_server`), confirming the plaintext reconnect succeeded; delivery
then proceeds through `delivery.Body`/`delivery.Commit`, ending with
`[debug] remote: disconnected from mx.example.invalid.`.

---

## 5. Coverage pass

Every named sub-question is answered above. Summary:

| Item | Answer (short) | Evidence `file:line` |
|------|----------------|----------------------|
| **Q1a** success vs. abort messages | Common `incoming message`; terminal `accepted` (success) vs. `aborted` (abort); mid-DATA abort adds `DATA error` | `smtp.go` L334 / L72 / L317 |
| **Q1b** `RCPT ok` format | `smtp: RCPT ok\t{"msg_id":…,"rcpt":…}` — exactly 2 fields (alpha order) | `smtp.go` L243; `orderedjson.go` L23 |
| **Q1c** abort comparison | Terminal record differs; mid-DATA abort inserts `DATA error {reason:"unexpected EOF"}`, logout abort does not; success never emits `aborted` | `smtp.go` L72, L317, L334 |
| **Q1d** module name in prefix | **`smtp`** | `smtp.go` L495, L715-L717 |
| **Q1e** `msg_id` format | 8 lowercase hex `^[0-9a-f]{8}$`, random per message (`1e451b6a`,`7c08d713`,`5a5ca2f7`,`71e45ff4`) | `msgid.go` L12-L16 |
| **Q2a** full sequence | Ordered records: acceptance → attempt #1 → failure → will retry → attempt #2 → delivered → removed | `queue.go` L367, L378, L384, L415-L418 |
| **Q2b** attempt / failure / retry lines | `delivery attempt #1` / `delivery attempt failed` / `will retry` | `queue.go` L367 / L384 / L415-L418 |
| **Q3a** MX-auth error string | `Failed to estabilish the MX record (mx.example.invalid.) authenticity` (misspelling verbatim) | `connect.go` L98 |
| **Q3b** enhanced code + reply text | Inner `5.7.0`; **surfaced `5.4.0`**, `smtp_code 550`; reply `No usable MXs, last err: …` | `connect.go` L96-L98, L204-L214; `exterrors/smtp.go` L11-L13, L122-L128 |
| **Q3c** TLS fallback record | `remote: TLS error, falling back to plaintext` — fields `domain,msg_id,mx,reason` | `connect.go` L176-L177 |
| **Q4a** retry-scheduling line | `queue: will retry {…}` | `queue.go` L415-L418 |
| **Q4b** retry-delay JSON field | **`next_try_delay`** (a `time.Duration`) | `queue.go` L417; `orderedjson.go` L43-L44 |

### Canonical vs. test-configuration

- **Random by design (canonical):** the endpoint `msg_id` (Q1e) is genuinely
  random (8 hex chars) in every configuration; only the format is fixed.
- **Deterministic test artifact:** the 40-hex SHA-1 `msg_id` in the queue/remote
  tests comes from the mock harness (`testutils/target.go` L184-L185), not from
  production code.
- **Non-canonical value, canonical field:** `next_try_delay` (Q4b) shows a small
  negative nanosecond value **only because** the queue test sets
  `initialRetryTime=0`/`retryTimeScale=1` (`queue_test.go` L50-L51). The canonical
  defaults (`15m`, scale `2`; `queue.go` L185-L186) produce `~15m0s`, `30m0s`,
  `1h0m0s`, … . The field name is identical in all configurations.
- **Verbosity flags are canonical controls, not bypasses:** `-v` surfaces
  `t.Log()` records and `-test.debuglog` enables `[debug]` records
  (`testutils/logger.go` L13-L41); neither changes the behavior being logged.

### Repository left unchanged

The investigation is strictly read-only. All `go` commands used `-mod=readonly`,
and no existing repository file was modified. After all runs,
`git status --porcelain` reported no changes to tracked files; `go.mod`/`go.sum`
are byte-for-byte unchanged. The only addition to the working tree is this
document, `blitzy/documentation/maddy_26452dd8dd78.md`.
