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

**Build prerequisite — PAM development headers.** A full `go build ./...` compiles
`cmd/maddy-pam-helper`, a CGO program that links against the system PAM library and
includes its header. It declares `#cgo LDFLAGS: -lpam`
(`cmd/maddy-pam-helper/main.go` L4) and its C source does
`#include <security/pam_appl.h>` (`cmd/maddy-pam-helper/pam.c` L4). The project's CI
definition installs PAM accordingly — `.build.yml` lists `pam` among its ArchLinux
`packages` (`.build.yml` L2-L5). On Debian/Ubuntu the equivalent is the
**`libpam0g-dev`** package, which provides `/usr/include/security/pam_appl.h`.
**Without this dev package the full build fails** to link the PAM helper. In this
environment the package was **pre-provisioned** (it was installed during environment
setup, not by this task, which is read-only); the following commands confirm the
prerequisite is satisfied:

**Command:** `dpkg -l | grep libpam0g-dev && test -f /usr/include/security/pam_appl.h && echo present && CGO_ENABLED=1 go build -mod=readonly -o /dev/null ./cmd/maddy-pam-helper/ && echo "pam-helper build OK"`

```text
ii  libpam0g-dev:amd64                   1.7.0-5ubuntu2                    amd64        Development files for PAM
present
pam-helper build OK
```

(The verification builds the PAM helper to `-o /dev/null` so no binary artifact is
written into the read-only repository.)

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

### 1.4 Exact-output fidelity and the terminal-TAB exception

This task's rules require quoting the **actual, complete, unedited** output for every
claim. maddy's `formatMsg` writes the message and then **always** emits a single TAB
before the JSON tail — `formatted.WriteRune('\t')` runs unconditionally
(`internal/log/log.go` L139), *before* the check for whether any fields exist
(`internal/log/log.go` L141). Consequently, a **no-field record** (for example
`smtp: listening on …`, or the debug records `queue: starting delivery for …`) is
emitted as `<module>: <message>\t` with **nothing after the TAB** — i.e. the line
ends with a literal, trailing TAB byte.

To stay byte-faithful, these trailing TABs are **preserved verbatim** in the verbatim
output blocks below and are deliberately **not** stripped. This has one visible
consequence worth calling out explicitly: `git diff --check` treats a trailing TAB as
trailing whitespace. Run against the baseline commit —
`git diff --check 26452dd8dd787dc455278b0fdd296f4a5432c768 HEAD` — it reports each such
line and exits non-zero (exit code `2`) for this document. (The bare `git diff --check`
compares the worktree against the index, so once this document is committed it exits
`0`; reproducing the exit-`2` whitespace report requires the baseline-relative form
above, or an unstaged edit that reintroduces the trailing TABs.) That is expected and
intentional — every flagged line is a no-field
record inside a fenced `text` code block (specifically the `smtp: listening on …` lines
of the SMTP transcripts and the field-less `queue:` debug records — `delivery target`,
`starting delivery for …`, `waiting on delivery semaphore for …`, and
`delivery semaphore acquired for …`), and there is **no** trailing whitespace anywhere
outside the verbatim runtime-output blocks. In other words, the non-zero
`git diff --check 26452dd8dd787dc455278b0fdd296f4a5432c768 HEAD` result is a direct
artifact of honoring the exact-output rule, not a formatting defect.

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
— `Log: log.Logger{Name: modName}` (`internal/endpoint/smtp/smtp.go` L495) — and each
session copies that logger with `log: endp.Log`
(`internal/endpoint/smtp/smtp.go` L677). So for an endpoint configured as
`smtp`, the module name in the log prefix for SMTP operations is **`smtp`**. (It
would read `submission` or `lmtp` for those endpoint types; the default SMTP
delivery endpoint tested here is `smtp`.) The tests confirm the literal `smtp:`
prefix on every record below.

### 2.2 Successful delivery — four consecutive identical runs

**Command (executed 4× consecutively):** `go test -mod=readonly -v -count=1 -run '^TestSMTPDelivery$' ./internal/endpoint/smtp/`
(driver: `TestSMTPDelivery`, `internal/endpoint/smtp/smtp_test.go` L122)

The complete, unedited output of all four runs is reproduced below (each run's
command echo, records, `PASS`, and package-result line are shown). Three classes
of token vary run-to-run and are therefore **not** part of the stable log
content: the random `msg_id` (see Q1e); the **ephemeral TCP ports** the test's
in-process listener and client bind to — the `listening on tcp://127.0.0.1:<port>`
port (`16289`, `34986`, `33839`, `42631` in the four runs below) and the `src_ip`
port (`51462`, `36376`, `45196`, `60644`); and the trailing `go test`
package-result time (`0.005s`/`0.006s` here). Once those documented ephemeral
values are normalized, every maddy **log field** — message text, JSON keys, and
their values — is identical across the four runs. These four runs are the sole
evidence source for the `msg_id` stability table in §2.5 (Q1e).

```text
$ go test -mod=readonly -v -count=1 -run '^TestSMTPDelivery$' ./internal/endpoint/smtp/
=== RUN   TestSMTPDelivery
    output.go:41: smtp: listening on tcp://127.0.0.1:16289	
    output.go:41: smtp: incoming message	{"msg_id":"c8f9d807","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:51462"}
    output.go:41: smtp: RCPT ok	{"msg_id":"c8f9d807","rcpt":"rcpt1@example.com"}
    output.go:41: smtp: RCPT ok	{"msg_id":"c8f9d807","rcpt":"rcpt2@example.com"}
    output.go:41: smtp: accepted	{"msg_id":"c8f9d807"}
--- PASS: TestSMTPDelivery (0.00s)
PASS
ok  	github.com/foxcpp/maddy/internal/endpoint/smtp	0.005s

$ go test -mod=readonly -v -count=1 -run '^TestSMTPDelivery$' ./internal/endpoint/smtp/
=== RUN   TestSMTPDelivery
    output.go:41: smtp: listening on tcp://127.0.0.1:34986	
    output.go:41: smtp: incoming message	{"msg_id":"e1629489","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:36376"}
    output.go:41: smtp: RCPT ok	{"msg_id":"e1629489","rcpt":"rcpt1@example.com"}
    output.go:41: smtp: RCPT ok	{"msg_id":"e1629489","rcpt":"rcpt2@example.com"}
    output.go:41: smtp: accepted	{"msg_id":"e1629489"}
--- PASS: TestSMTPDelivery (0.00s)
PASS
ok  	github.com/foxcpp/maddy/internal/endpoint/smtp	0.005s

$ go test -mod=readonly -v -count=1 -run '^TestSMTPDelivery$' ./internal/endpoint/smtp/
=== RUN   TestSMTPDelivery
    output.go:41: smtp: listening on tcp://127.0.0.1:33839	
    output.go:41: smtp: incoming message	{"msg_id":"2b38bee0","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:45196"}
    output.go:41: smtp: RCPT ok	{"msg_id":"2b38bee0","rcpt":"rcpt1@example.com"}
    output.go:41: smtp: RCPT ok	{"msg_id":"2b38bee0","rcpt":"rcpt2@example.com"}
    output.go:41: smtp: accepted	{"msg_id":"2b38bee0"}
--- PASS: TestSMTPDelivery (0.00s)
PASS
ok  	github.com/foxcpp/maddy/internal/endpoint/smtp	0.006s

$ go test -mod=readonly -v -count=1 -run '^TestSMTPDelivery$' ./internal/endpoint/smtp/
=== RUN   TestSMTPDelivery
    output.go:41: smtp: listening on tcp://127.0.0.1:42631	
    output.go:41: smtp: incoming message	{"msg_id":"5779cf57","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:60644"}
    output.go:41: smtp: RCPT ok	{"msg_id":"5779cf57","rcpt":"rcpt1@example.com"}
    output.go:41: smtp: RCPT ok	{"msg_id":"5779cf57","rcpt":"rcpt2@example.com"}
    output.go:41: smtp: accepted	{"msg_id":"5779cf57"}
--- PASS: TestSMTPDelivery (0.00s)
PASS
ok  	github.com/foxcpp/maddy/internal/endpoint/smtp	0.005s
```

### 2.3 Abort mid-transaction (mid-DATA)

**Command:** `go test -mod=readonly -v -count=1 -run '^TestSMTPDelivery_AbortData$' ./internal/endpoint/smtp/`
(driver: `TestSMTPDelivery_AbortData`, `internal/endpoint/smtp/smtp_test.go` L360)

```text
=== RUN   TestSMTPDelivery_AbortData
    output.go:41: smtp: listening on tcp://127.0.0.1:61055	
    output.go:41: smtp: incoming message	{"msg_id":"0d0cfa1e","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:41292"}
    output.go:41: smtp: RCPT ok	{"msg_id":"0d0cfa1e","rcpt":"test@example.com"}
    output.go:41: smtp: DATA error	{"msg_id":"0d0cfa1e","reason":"unexpected EOF"}
    output.go:41: smtp: aborted	{"msg_id":"0d0cfa1e"}
--- PASS: TestSMTPDelivery_AbortData (0.25s)
PASS
ok  	github.com/foxcpp/maddy/internal/endpoint/smtp	0.260s
```

### 2.4 Abort at logout (control case)

**Command:** `go test -mod=readonly -v -count=1 -run '^TestSMTPDelivery_AbortLogout$' ./internal/endpoint/smtp/`
(driver: `TestSMTPDelivery_AbortLogout`, `internal/endpoint/smtp/smtp_test.go` L398)

```text
=== RUN   TestSMTPDelivery_AbortLogout
    output.go:41: smtp: listening on tcp://127.0.0.1:57374	
    output.go:41: smtp: incoming message	{"msg_id":"d57d4353","sender":"sender@example.org","src_host":"mx.example.org","src_ip":"127.0.0.1:35542"}
    output.go:41: smtp: RCPT ok	{"msg_id":"d57d4353","rcpt":"test@example.com"}
    output.go:41: smtp: aborted	{"msg_id":"d57d4353"}
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
  (`internal/endpoint/smtp/smtp.go` L334). Observed (run 1 of §2.2):
  `smtp: accepted\t{"msg_id":"c8f9d807"}`.
- **Abort** ends with `smtp: aborted\t{"msg_id":"…"}`, emitted by
  `s.log.Msg("aborted", "msg_id", s.msgMeta.ID)` inside `Session.abort`
  (`internal/endpoint/smtp/smtp.go` L72).

The two abort variants further clarify the comparison (Q1c):

- **Abort mid-DATA** inserts an error record `smtp: DATA error\t{…,"reason":…}`
  *before* `aborted`. It is emitted by `s.log.Error("DATA error", err, "msg_id", …)`
  (`internal/endpoint/smtp/smtp.go` L317). The observed `reason` is `"unexpected EOF"`, which is
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
The complete log line is `smtp: RCPT ok\t{"msg_id":"c8f9d807","rcpt":"rcpt1@example.com"}`
(run 1 of §2.2).
It contains **exactly two JSON fields** — `msg_id` and `rcpt` — and is emitted by
`s.endp.Log.Msg("RCPT ok", "rcpt", to, "msg_id", s.msgMeta.ID)` at
`internal/endpoint/smtp/smtp.go` L243. Note that the source passes `rcpt` **first**
and `msg_id` second, yet the output shows `msg_id` before `rcpt`: this is the
alphabetical key sort from `marshalOrderedJSON` (`internal/log/orderedjson.go`
L23) in action. The successful transaction emits one `RCPT ok` per accepted
recipient (two in `TestSMTPDelivery`).

**Q1d — module name in the log prefix.** **`smtp`** (see §2.1;
`internal/endpoint/smtp/smtp.go` L495 and L715-L717).

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

The value is **random per message**. The four consecutive runs of the exact same
command reproduced verbatim in §2.2 produced four different values, each still
matching the format (these values are read directly from the `incoming message`
records of the four transcripts shown above):

| Run (from §2.2) | `msg_id` | Matches `^[0-9a-f]{8}$` |
|-----|----------|--------------------------|
| 1 | `c8f9d807` | yes |
| 2 | `e1629489` | yes |
| 3 | `2b38bee0` | yes |
| 4 | `5779cf57` | yes |

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
`hex.EncodeToString(IDRaw[:])`. `TestQueueDelivery_TemporaryFail` reaches this code
through `testutils.DoTestDelivery` → `DoTestDeliveryMeta` → `DoTestDeliveryErrMeta`,
so the emitting lines are the pair inside `DoTestDeliveryErrMeta` at
`internal/testutils/target.go` L239-L240. (A byte-identical pair at
`internal/testutils/target.go` L184-L185 lives in the unrelated, unused
`DoTestDeliveryNonAtomic` and is **not** on this test's path.) For
`TestQueueDelivery_TemporaryFail` the ID is the 40-hex-character constant
`af8090c7eb39f761862b1f027b4f2b0bb1ce86d1`, **identical across every run** (see the
eight runs in §3.2). Each retry attempt appends an attempt suffix (`…-1`, `…-2`):
the queue rewrites the copied metadata ID with
`msgMeta.ID = msgMeta.ID + "-" + strconv.Itoa(meta.TriesCount+1)`
(`internal/target/queue/queue.go` L439) and logs it via
`dl.Debugf("using message ID = %s", msgMeta.ID)`
(`internal/target/queue/queue.go` L440), visible in the `using message ID = …`
debug records of §3.3. This is a test-harness artifact and is called out here to
avoid conflating it with the canonical random endpoint `msg_id` of Q1e.

### 3.2 Non-debug run — eight consecutive identical runs

**Command (executed 8× consecutively):** `go test -mod=readonly -v -count=1 -run '^TestQueueDelivery_TemporaryFail$' ./internal/target/queue/`
(driver: `TestQueueDelivery_TemporaryFail`, `internal/target/queue/queue_test.go` L297)

The complete, unedited output of all eight runs is reproduced below (each run's
command echo, records, `PASS`, and package-result line are shown). The SHA-1
`msg_id` is identical every run (see §3.1). Two things vary within maddy's own
`queue:` log fields: `next_try_delay`, a near-zero negative test-configuration
artifact (explained in the Q4b caveat, §3.4); and the **relative order of the two
`delivery attempt failed` records**, which is nondeterministic because the queue
ranges over a Go map to emit them (`internal/target/queue/queue.go` L383 — see the
Q2a map-order caveat, §3.4). All eight runs shown below happen to print `tester1`
before `tester2`, but that order is not guaranteed. Separately, the `go test`
per-test and package-result elapsed times (`0.01s` to `0.05s` and `0.012s` to
`0.054s` here) are run-to-run variable and are not part of the log content. These
eight runs are the sole evidence source for the `next_try_delay` distribution
table in §3.4 (Q4b).

```text
$ go test -mod=readonly -v -count=1 -run '^TestQueueDelivery_TemporaryFail$' ./internal/target/queue/
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
    output.go:41: queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-426ns","rcpts":["tester1@example.org","tester2@example.org"]}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org"}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org"}
    queue_test.go:38: --- queue.Close
--- PASS: TestQueueDelivery_TemporaryFail (0.05s)
PASS
ok  	github.com/foxcpp/maddy/internal/target/queue	0.052s

$ go test -mod=readonly -v -count=1 -run '^TestQueueDelivery_TemporaryFail$' ./internal/target/queue/
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
    output.go:41: queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-519ns","rcpts":["tester1@example.org","tester2@example.org"]}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org"}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org"}
    queue_test.go:38: --- queue.Close
--- PASS: TestQueueDelivery_TemporaryFail (0.03s)
PASS
ok  	github.com/foxcpp/maddy/internal/target/queue	0.031s

$ go test -mod=readonly -v -count=1 -run '^TestQueueDelivery_TemporaryFail$' ./internal/target/queue/
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
    output.go:41: queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-447ns","rcpts":["tester1@example.org","tester2@example.org"]}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org"}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org"}
    queue_test.go:38: --- queue.Close
--- PASS: TestQueueDelivery_TemporaryFail (0.05s)
PASS
ok  	github.com/foxcpp/maddy/internal/target/queue	0.054s

$ go test -mod=readonly -v -count=1 -run '^TestQueueDelivery_TemporaryFail$' ./internal/target/queue/
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
    output.go:41: queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-546ns","rcpts":["tester1@example.org","tester2@example.org"]}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org"}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org"}
    queue_test.go:38: --- queue.Close
--- PASS: TestQueueDelivery_TemporaryFail (0.05s)
PASS
ok  	github.com/foxcpp/maddy/internal/target/queue	0.052s

$ go test -mod=readonly -v -count=1 -run '^TestQueueDelivery_TemporaryFail$' ./internal/target/queue/
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
    output.go:41: queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-565ns","rcpts":["tester1@example.org","tester2@example.org"]}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org"}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org"}
    queue_test.go:38: --- queue.Close
--- PASS: TestQueueDelivery_TemporaryFail (0.01s)
PASS
ok  	github.com/foxcpp/maddy/internal/target/queue	0.012s

$ go test -mod=readonly -v -count=1 -run '^TestQueueDelivery_TemporaryFail$' ./internal/target/queue/
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
    output.go:41: queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-364ns","rcpts":["tester1@example.org","tester2@example.org"]}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org"}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org"}
    queue_test.go:38: --- queue.Close
--- PASS: TestQueueDelivery_TemporaryFail (0.03s)
PASS
ok  	github.com/foxcpp/maddy/internal/target/queue	0.031s

$ go test -mod=readonly -v -count=1 -run '^TestQueueDelivery_TemporaryFail$' ./internal/target/queue/
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
    output.go:41: queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-473ns","rcpts":["tester1@example.org","tester2@example.org"]}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org"}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org"}
    queue_test.go:38: --- queue.Close
--- PASS: TestQueueDelivery_TemporaryFail (0.05s)
PASS
ok  	github.com/foxcpp/maddy/internal/target/queue	0.051s

$ go test -mod=readonly -v -count=1 -run '^TestQueueDelivery_TemporaryFail$' ./internal/target/queue/
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
    output.go:41: queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-409ns","rcpts":["tester1@example.org","tester2@example.org"]}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org"}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org"}
    queue_test.go:38: --- queue.Close
--- PASS: TestQueueDelivery_TemporaryFail (0.05s)
PASS
ok  	github.com/foxcpp/maddy/internal/target/queue	0.051s
```

### 3.3 Full `[debug]` sequence — answers Q2a

**Command:** `go test -mod=readonly -v -count=1 -run '^TestQueueDelivery_TemporaryFail$' ./internal/target/queue/ -args -test.debuglog`

The complete, unedited `[debug]` transcript of one run is reproduced below. This
run scheduled `next_try_delay` = `-609ns`, consistent with the §3.2
distribution. The interleaved `target.go:166` lines are the *test harness*
(`testutils.DoTestDelivery`) driving the queue's own inbound
`module.DeliveryTarget` API — i.e. initial acceptance of the message into the
queue (see the Q2a note in §3.4) — not the queue's own `queue:` records and
not the downstream mock target's logs.

```text
$ go test -mod=readonly -v -count=1 -run '^TestQueueDelivery_TemporaryFail$' ./internal/target/queue/ -args -test.debuglog
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
    output.go:41: queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-609ns","rcpts":["tester1@example.org","tester2@example.org"]}
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
--- PASS: TestQueueDelivery_TemporaryFail (0.05s)
PASS
ok  	github.com/foxcpp/maddy/internal/target/queue	0.052s
```

### 3.4 Answers

**Q2a — the complete sequence from initial acceptance through retry.**
The `[debug]` capture in §3.3 is read top-to-bottom below. Two distinct log
*sources* are interleaved there and must not be confused:

- Lines tagged `target.go:166` are the **test harness** — `testutils.DoTestDelivery`
  (`internal/testutils/target.go` L165, which calls `DoTestDeliveryMeta` at L166) →
  `DoTestDeliveryErrMeta` (`internal/testutils/target.go` L236). These are the test
  driving the **queue's own inbound `module.DeliveryTarget` interface**
  (`tgt.Start` L246, `delivery.AddRcpt` L253, `delivery.Body` L263, `delivery.Commit`
  L275) — i.e. the test submitting the message **into** the queue: this is *initial
  acceptance*, **not** the queue's downstream delivery. They are attributed to
  `target.go:166` because both helper frames call `t.Helper()`
  (`internal/testutils/target.go` L172 and L237), so the Go testing framework
  credits the non-helper call site `DoTestDelivery` at L166.
- Lines tagged `output.go:41` are the maddy **`queue:` records** emitted while the
  queue delivers downstream to the mock `unreliableTarget`.

The queue's own `queue:` records appear in this order (every line below is emitted
from `internal/target/queue/queue.go`):

1. `[debug] queue: delivery target: *queue.unreliableTarget` — `q.Log.Debugf("delivery target: %T", q.Target)` (L248)
2. `[debug] queue: starting delivery for af8090c7…` — `q.Log.Debugln("starting delivery for", slot.ID)` (L278)
3. `[debug] queue: waiting on delivery semaphore for af8090c7…` — `q.Log.Debugln("waiting on delivery semaphore for", slot.ID)` (L282)
4. `[debug] queue: delivery semaphore acquired for af8090c7…` — `q.Log.Debugln("delivery semaphore acquired for", slot.ID)` (L299)
5. `[debug] queue: delivery attempt #1` — `dl.Debugf("delivery attempt #%d", meta.TriesCount+1)` (L367)
6. `[debug] queue: using message ID = af8090c7…-1` — `dl.Debugf("using message ID = %s", msgMeta.ID)` (L440; the `-1` suffix is appended at L439)
7. `[debug] queue: target.Start OK` — `dl.Debugf("target.Start OK")` (L456)
8. `[debug] queue: delivery.AddRcpt tester1@example.org OK` — `dl.Debugf("delivery.AddRcpt %s OK", rcpt)` (L470)
9. `[debug] queue: delivery.AddRcpt tester2@example.org OK` — same emitter (L470)
10. `[debug] queue: delivery.Body failed: you shall not pass` — `dl.Debugf("delivery.Body failed: %v", err)` (L504)
11. `[debug] queue: delivery.Body OK` — `dl.Debugf("delivery.Body OK")` (L507, unconditional — see the note below)
12. `[debug] queue: delivery.Abort (all recipients failed)` — `dl.Debugf("delivery.Abort (all recipients failed)")` (L518)
13. `[debug] queue: failures: permanently: [], temporary: [tester1@example.org tester2@example.org], errors: map[…]` — `dl.Debugf("failures: …")` (L370)
14. `queue: delivery attempt failed` — once per recipient, **non-debug** level; `dl.Error("delivery attempt failed", rcptErr, "rcpt", rcpt)` (L384)
15. `queue: will retry` — the retry-scheduling record; `dl.Msg("will retry", …)` (L415-L418)
16. *(attempt #2 repeats the connect steps)* `starting delivery` (L278) → `waiting on delivery semaphore` (L282) → `delivery semaphore acquired` (L299) → `delivery attempt #2` (L367) → `using message ID = …-2` (L440) → `target.Start OK` (L456) → `delivery.AddRcpt … OK` ×2 (L470)
17. `[debug] queue: delivery.Body OK` — `dl.Debugf("delivery.Body OK")` (L507)
18. `[debug] queue: delivery.Commit OK` — `dl.Debugf("delivery.Commit OK")` (L529)
19. `[debug] queue: failures: permanently: [], temporary: [], errors: map[]` (empty — success) — `dl.Debugf("failures: …")` (L370)
20. `queue: delivered` — once per recipient, with `"attempt":2`; `dl.Msg("delivered", "rcpt", rcpt, "attempt", meta.TriesCount+1)` (L378)
21. `[debug] queue: removed message from disk` — `dl.Debugf("removed message from disk")` (L619)

**Recipient-order caveat (reported exactly as observed).** The *phase* ordering
above is stable, but one detail within step 14 is **nondeterministic**: the
**relative order of the two `queue: delivery attempt failed` records** varies from
run to run. The queue emits them by ranging over the `partialErr.Errs` **map**
(`internal/target/queue/queue.go` L383; the field is declared
`Errs map[string]error` at L80), and Go randomizes map-iteration order. Everything
else stays in a fixed order: the `will retry` record's `rcpts` array and the two
`queue: delivered` records are built from the `TemporaryFailed`/`meta.To` **slice**,
appended in recipient-acceptance order by `expandToPartialErr`
(`internal/target/queue/queue.go` L484-L493) and assigned at L387, so they always
read `tester1@example.org` then `tester2@example.org`.

Re-running the non-debug command (§3.2) 50 consecutive times captured **both**
orders — `tester1`-first in 45 runs and `tester2`-first in 5 (~90/10); the eight
runs quoted in §3.2 all happen to be `tester1`-first. One captured `tester2`-first
run is reproduced verbatim below: the two failure records are reversed, while the
`rcpts` array and the `delivered` records remain in their stable slice order,
confirming that only the map-ranged failure lines reorder:

```text
    output.go:41: queue: delivery attempt failed	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org","reason":"you shall not pass"}
    output.go:41: queue: delivery attempt failed	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org","reason":"you shall not pass"}
    output.go:41: queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-559ns","rcpts":["tester1@example.org","tester2@example.org"]}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org"}
    output.go:41: queue: delivered	{"attempt":2,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org"}
```

A behavioral detail, reported exactly as observed: in attempt #1 both a
`delivery.Body failed: you shall not pass` **and** a `delivery.Body OK` debug line
appear. This is **not** two body calls. The queue invokes `delivery.Body` exactly
**once** per attempt at `internal/target/queue/queue.go` L503; when that call
returns an error the queue logs `delivery.Body failed` inside the error branch
(L504) and then logs `delivery.Body OK` **unconditionally** on the next line,
outside that branch (L507). So the `OK` line is emitted regardless of the call's
outcome — a source-level logging quirk, not a successful follow-up delivery. The
genuinely successful body call occurs on **attempt #2**: the mock
`unreliableTargetDelivery.Body` returns `bodyFailures[passedMessages]`
(`internal/target/queue/queue_test.go` L116-L117); `passedMessages` is `0` on
attempt #1 (so `Body` returns `"you shall not pass"`) and is incremented to `1`
when the queue aborts the all-failed attempt via the mock's `Abort`, whose body
runs `utd.ut.passedMessages++` (`internal/target/queue/queue_test.go` L134-L135),
so on attempt #2 `Body` returns
`nil` and only a single `delivery.Body OK` (with no preceding `failed`) is logged
before `delivery.Commit OK`.

**Q2b — the exact attempt / failure / retry lines.** The three lines below are
quoted verbatim from the single `[debug]` run in §3.3 (so the `delivery attempt #1`
debug line and the retry line share one run's `next_try_delay` value, `-609ns`);
the eight non-debug runs in §3.2 emit the same `delivery attempt failed` and
`will retry` records, differing only in the tabulated `next_try_delay` value below
and — for the two `delivery attempt failed` lines — in their run-to-run relative
order (the Q2a map-order caveat).

- **Delivery attempt** (debug-level), from `dl.Debugf("delivery attempt #%d", meta.TriesCount+1)` (`internal/target/queue/queue.go` L367):
  `[debug] queue: delivery attempt #1\t{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}`
- **Delivery failure**, from `dl.Error("delivery attempt failed", rcptErr, "rcpt", rcpt)` (`internal/target/queue/queue.go` L384). Its `reason` is the recipient error text, injected by `Logger.Error` (`internal/log/log.go` L98-L99):
  `queue: delivery attempt failed\t{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org","reason":"you shall not pass"}`
- **Retry scheduling**, from `dl.Msg("will retry", "attempts_count", meta.TriesCount, "next_try_delay", time.Until(nextTryTime), "rcpts", meta.To)` (`internal/target/queue/queue.go` L415-L418):
  `queue: will retry\t{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-609ns","rcpts":["tester1@example.org","tester2@example.org"]}`

(For completeness, the eventual success line is `queue: delivered\t{"attempt":2,…}`,
emitted by `dl.Msg("delivered", "rcpt", rcpt, "attempt", meta.TriesCount+1)`,
`internal/target/queue/queue.go` L378.)

**Q4a — the retry-scheduling line.** It is the `queue: will retry {…}` record
quoted just above (`internal/target/queue/queue.go` L415-L418).

**Q4b — the JSON field name carrying the retry delay.** It is **`next_try_delay`**
(`internal/target/queue/queue.go` L417). The value is `time.Until(nextTryTime)`, a
Go `time.Duration`, which the logger renders through `Duration.String()`
(`internal/log/orderedjson.go` L43-L44).

**Canonical vs. test-configuration (mandatory caveat).** The observed
`next_try_delay` is a **small negative value** and is a **non-canonical
test-configuration artifact**. The test overrides the retry schedule with
`q.initialRetryTime = 0` and `q.retryTimeScale = 1`
(`internal/target/queue/queue_test.go` L50-L51), so the computed
`nextTryTime = time.Now()` (`internal/target/queue/queue.go` L413) has already
passed by the time `time.Until(nextTryTime)` (L417) runs — yielding a slightly
negative duration. Across the **eight identical non-debug runs of §3.2** the value
varied while the **field name and its format never changed** (values read directly
from the §3.2 transcript):

| Run (from §3.2) | `next_try_delay` |
|-----------------|------------------|
| 1 | `-426ns` |
| 2 | `-519ns` |
| 3 | `-447ns` |
| 4 | `-546ns` |
| 5 | `-565ns` |
| 6 | `-364ns` |
| 7 | `-473ns` |
| 8 | `-409ns` |

(The `[debug]` run in §3.3 independently produced `-609ns`, consistent with this
distribution.) All nine values are negative and in the low hundreds of nanoseconds.

In the **default/canonical** configuration the defaults are
`initialRetryTime: 15 * time.Minute` and `retryTimeScale: 2`
(`internal/target/queue/queue.go` L185-L186), and the delay follows
`nextTryTime.Add(q.initialRetryTime * time.Duration(math.Pow(q.retryTimeScale, float64(meta.TriesCount-1))))`
(`internal/target/queue/queue.go` L413-L414) — i.e. approximately `15m0s`, then
`30m0s`, then `1h0m0s`, … for successive attempts, rendered as those
`Duration.String()` forms. **The field name `next_try_delay` is identical in every
configuration; only the value differs.**

---

## 4. Q3 — Remote delivery: MX-authentication failure & TLS fallback

**Package:** `internal/target/remote` · **Module prefix:** `remote`

The remote target logger's `Name` is `"remote"` (`internal/target/remote/remote.go`
L83, `Log: log.Logger{Name: "remote"}`).

### 4.1 MX-authenticity failure — answers Q3a / Q3b

**Command (executed 2× consecutively):** `go test -mod=readonly -v -count=1 -run '^TestRemoteDelivery_AuthMX_Fail$' ./internal/target/remote/`
(driver: `TestRemoteDelivery_AuthMX_Fail`, `internal/target/remote/mxauth_test.go` L16)

The complete, unedited output of **both consecutive runs** is reproduced below
(each run's command echo, records, `PASS`, and package-result line are shown). The
MX-authenticity **error and reply artifact** — the `-- ... delivery.AddRcpt`
line with its `map[…]` field dump — is fully deterministic and **byte-identical**
across runs (no random `msg_id`, and no map-ordered records on this path); the
only run-to-run variation is the trailing `go test` package-result time (`0.005s`
in both runs below, which over 40 repeated runs was observed as `0.005s` in 34,
`0.006s` in 3, `0.007s` in 2, and `0.016s` once). This is the evidence for the
stability statement at the end of this subsection.

```text
$ go test -mod=readonly -v -count=1 -run '^TestRemoteDelivery_AuthMX_Fail$' ./internal/target/remote/
=== RUN   TestRemoteDelivery_AuthMX_Fail
    target.go:233: -- tgt.Start test@example.com
    target.go:233: -- delivery.AddRcpt test@example.invalid
    target.go:233: -- ... delivery.AddRcpt test@example.invalid Failed to estabilish the MX record (mx.example.invalid.) authenticity map[domain:example.invalid reason:Failed to estabilish the MX record (mx.example.invalid.) authenticity smtp_code:550 smtp_enchcode:[5 4 0] smtp_msg:No usable MXs, last err: Failed to estabilish the MX record (mx.example.invalid.) authenticity target:remote]
    target.go:233: -- delivery.Abort
--- PASS: TestRemoteDelivery_AuthMX_Fail (0.00s)
PASS
ok  	github.com/foxcpp/maddy/internal/target/remote	0.005s

$ go test -mod=readonly -v -count=1 -run '^TestRemoteDelivery_AuthMX_Fail$' ./internal/target/remote/
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
path, `"Failed to estabilish the MX record authenticity (MTA-STS)"`
(`internal/target/remote/connect.go` L66-L67), also with `{5, 7, 0}`.

**Q3b — SMTP enhanced status code (X.Y.Z) and the complete reply text.** Two
distinct enhanced codes exist on this path, and both are reported here because the
error is wrapped:

- **Inner** — the MX-authenticity check itself returns enhanced code **`5.7.0`**
  with message *"Failed to estabilish the MX record (mx.example.invalid.)
  authenticity"* (`internal/target/remote/connect.go` L96-L98).
- **Surfaced** — once the single MX is exhausted, the delivery error actually
  returned to the caller is wrapped as **"No usable MXs, last err: …"** with
  enhanced code **`5.4.0`** and `smtp_code` **`550`**, `target` **`remote`**
  (`internal/target/remote/connect.go` L204-L208). This is the code the client
  would receive.

The SMTP-related fields extracted from that harness dump (which additionally
carries `domain` and `reason`, shown in full in the capture above) are:

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
  `exterrors.SMTPEnchCode` (`internal/target/remote/connect.go` L206). That helper
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
  (`internal/exterrors/smtp.go` L99-L107) — which is why the harness line begins
  with the inner string while `smtp_msg` carries the outer reply text.
  `SMTPError.Temporary()` maps 4xx codes to retryable via `se.Code/100 == 4`
  (`internal/exterrors/smtp.go` L95-L96); here the code is 550, i.e. permanent.

This result is **stable across runs**: the MX-authenticity error string, the
`smtp_code`/`smtp_enchcode`/`smtp_msg` fields, and the complete `map[…]` artifact
are byte-identical on every run (deterministic — no random `msg_id`, and no
map-ordered records on this path). The only run-to-run variation in the full
`go test` transcript is the trailing package-result elapsed time (`0.005s` in the
two runs above; across 40 repeated runs it was `0.005s` in 34, `0.006s` in 3,
`0.007s` in 2, and `0.016s` once), which is timing-sensitive and is not part of
the application artifact.

### 4.2 TLS-to-plaintext fallback — answers Q3c

**Command:** `go test -mod=readonly -v -count=1 -run '^TestRemoteDelivery_TLSErrFallback$' ./internal/target/remote/`
(driver: `TestRemoteDelivery_TLSErrFallback`, `internal/target/remote/remote_test.go` L852)

```text
$ go test -mod=readonly -v -count=1 -run '^TestRemoteDelivery_TLSErrFallback$' ./internal/target/remote/
=== RUN   TestRemoteDelivery_TLSErrFallback
    target.go:166: -- tgt.Start test@example.com
    target.go:166: -- delivery.AddRcpt test@example.invalid
    output.go:41: remote: TLS error, falling back to plaintext	{"domain":"example.invalid","msg_id":"2176ec5872ed2b87d832b4070e88232bd94ac7d3","mx":"mx.example.invalid.","reason":"smtpconn: x509: certificate signed by unknown authority"}
    target.go:166: -- delivery.Body
    target.go:166: -- delivery.Commit
--- PASS: TestRemoteDelivery_TLSErrFallback (0.02s)
PASS
ok  	github.com/foxcpp/maddy/internal/target/remote	0.021s
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
`internal/target/remote/connect.go` L175. The `msg_id` is the deterministic 40-hex
SHA-1 of the test name (§3.1), not the 8-hex endpoint value of Q1e.

To show the before/after state around the fallback, the same test with
`-test.debuglog` reveals the plaintext reconnect succeeding immediately after:

**Command:** `go test -mod=readonly -v -count=1 -run '^TestRemoteDelivery_TLSErrFallback$' ./internal/target/remote/ -args -test.debuglog`

```text
$ go test -mod=readonly -v -count=1 -run '^TestRemoteDelivery_TLSErrFallback$' ./internal/target/remote/ -args -test.debuglog
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

Observed exactly as printed, each record resolved to its emitter:

- `[debug] remote: trying` precedes the fallback —
  `rd.Log.DebugMsg("trying", "mx", record.Host, "domain", domain)`
  (`internal/target/remote/connect.go` L163).
- After the `TLS error, falling back to plaintext` record, **two**
  `[debug] remote: connected` records appear. The first is from the remote target
  and carries `domain`/`msg_id`/`mx` —
  `rd.Log.DebugMsg("connected", "mx", conn.ServerName(), "domain", domain)`
  (`internal/target/remote/connect.go` L216). The second is from the underlying
  `smtpconn` connection and carries `msg_id`/`remote_server` —
  `c.Log.DebugMsg("connected", "remote_server", c.serverName)`
  (`internal/smtpconn/smtpconn.go` L248). Together they confirm the plaintext
  reconnect succeeded.
- Delivery then proceeds through `delivery.Body`/`delivery.Commit`, ending with
  `[debug] remote: disconnected from mx.example.invalid.` —
  `rd.Log.Debugf("disconnected from %s", conn.ServerName())`
  (`internal/target/remote/remote.go` L454).

---

## 5. Coverage pass

Every named sub-question is answered above. Summary:

| Item | Answer (short) | Evidence `file:line` |
|------|----------------|----------------------|
| **Q1a** success vs. abort messages | Common `incoming message`; terminal `accepted` (success) vs. `aborted` (abort); mid-DATA abort adds `DATA error` | `internal/endpoint/smtp/smtp.go` L334 / L72 / L317 |
| **Q1b** `RCPT ok` format | `smtp: RCPT ok\t{"msg_id":…,"rcpt":…}` — exactly 2 fields (alpha order) | `internal/endpoint/smtp/smtp.go` L243; `internal/log/orderedjson.go` L23 |
| **Q1c** abort comparison | Terminal record differs; mid-DATA abort inserts `DATA error {reason:"unexpected EOF"}`, logout abort does not; success never emits `aborted` | `internal/endpoint/smtp/smtp.go` L72, L317, L334 |
| **Q1d** module name in prefix | **`smtp`** | `internal/endpoint/smtp/smtp.go` L495, L715-L717 |
| **Q1e** `msg_id` format | 8 lowercase hex `^[0-9a-f]{8}$`, random per message (`c8f9d807`,`e1629489`,`2b38bee0`,`5779cf57` — the four §2.2 runs) | `internal/msgpipeline/msgid.go` L12-L16 |
| **Q2a** full sequence | Stable phase order: acceptance → attempt #1 → failure → will retry → attempt #2 → delivered → removed; the two per-recipient `delivery attempt failed` lines reorder nondeterministically (Go map range, L383) | `internal/target/queue/queue.go` L367, L378, L383-L384, L415-L418 |
| **Q2b** attempt / failure / retry lines | `delivery attempt #1` / `delivery attempt failed` / `will retry` | `internal/target/queue/queue.go` L367 / L384 / L415-L418 |
| **Q3a** MX-auth error string | `Failed to estabilish the MX record (mx.example.invalid.) authenticity` (misspelling verbatim) | `internal/target/remote/connect.go` L98 |
| **Q3b** enhanced code + reply text | Inner `5.7.0`; **surfaced `5.4.0`**, `smtp_code 550`; reply `No usable MXs, last err: …` | `internal/target/remote/connect.go` L96-L98, L204-L208; `internal/exterrors/smtp.go` L11-L13, L122-L128 |
| **Q3c** TLS fallback record | `remote: TLS error, falling back to plaintext` — fields `domain,msg_id,mx,reason` | `internal/target/remote/connect.go` L176-L177 |
| **Q4a** retry-scheduling line | `queue: will retry {…}` | `internal/target/queue/queue.go` L415-L418 |
| **Q4b** retry-delay JSON field | **`next_try_delay`** (a `time.Duration`) | `internal/target/queue/queue.go` L417; `internal/log/orderedjson.go` L43-L44 |

### Reproduction commands (per question)

The exact commands used are consolidated here (each is also shown inline with its
captured output in the referenced section). All use `-mod=readonly` so `go.mod`/
`go.sum` are never rewritten:

- **Build (all questions):** `CGO_ENABLED=1 go build -mod=readonly ./...` — §1.1
- **Q1 SMTP, success:** `go test -mod=readonly -v -count=1 -run '^TestSMTPDelivery$' ./internal/endpoint/smtp/` — §2.2 (run 4× for the `msg_id` stability table §2.5)
- **Q1 SMTP, abort mid-DATA:** `go test -mod=readonly -v -count=1 -run '^TestSMTPDelivery_AbortData$' ./internal/endpoint/smtp/` — §2.3
- **Q1 SMTP, abort at logout:** `go test -mod=readonly -v -count=1 -run '^TestSMTPDelivery_AbortLogout$' ./internal/endpoint/smtp/` — §2.4
- **Q2 / Q4 queue trace + retry:** `go test -mod=readonly -v -count=1 -run '^TestQueueDelivery_TemporaryFail$' ./internal/target/queue/` (run 8× for the `next_try_delay` distribution, §3.2); append `-args -test.debuglog` for the full `[debug]` sequence — §3.3
- **Q3 MX-authentication failure:** `go test -mod=readonly -v -count=1 -run '^TestRemoteDelivery_AuthMX_Fail$' ./internal/target/remote/` — §4.1 (run 2× for stability)
- **Q3 TLS-to-plaintext fallback:** `go test -mod=readonly -v -count=1 -run '^TestRemoteDelivery_TLSErrFallback$' ./internal/target/remote/`; append `-args -test.debuglog` for the reconnect records — §4.2

### Canonical vs. test-configuration

- **Random by design (canonical):** the endpoint `msg_id` (Q1e) is genuinely
  random (8 hex chars) in every configuration; only the format is fixed.
- **Deterministic test artifact:** the 40-hex SHA-1 `msg_id` in the queue/remote
  tests comes from the mock harness `DoTestDeliveryErrMeta`
  (`internal/testutils/target.go` L239-L240), not from production code.
- **Non-canonical value, canonical field:** `next_try_delay` (Q4b) shows a small
  negative nanosecond value **only because** the queue test sets
  `initialRetryTime=0`/`retryTimeScale=1` (`internal/target/queue/queue_test.go`
  L50-L51). The canonical defaults (`15m`, scale `2`;
  `internal/target/queue/queue.go` L185-L186) produce `~15m0s`, `30m0s`, `1h0m0s`,
  … . The field name is identical in all configurations.
- **Verbosity flags are canonical controls, not bypasses:** `-v` surfaces
  `t.Log()` records and `-test.debuglog` enables `[debug]` records
  (`internal/testutils/logger.go` L13-L41); neither changes the behavior being
  logged.

### Repository left unchanged

The investigation is strictly read-only. Every `go` command used `-mod=readonly`,
and no existing repository file — source, test, configuration, `go.mod`, or
`go.sum` — was modified; the only addition to the tree is this document. The
verification commands and their **complete, unedited output** follow (baseline =
the checkout commit `26452dd8dd787dc455278b0fdd296f4a5432c768`):

```text
$ git status --porcelain
(no output above = clean tracked worktree)

$ git diff --name-status 26452dd8dd787dc455278b0fdd296f4a5432c768 -- .
A	blitzy/documentation/maddy_26452dd8dd78.md
(only the added doc above; no M/D entry = no other tracked file differs from baseline in the worktree)

$ git diff --name-status 26452dd8dd787dc455278b0fdd296f4a5432c768 HEAD
A	blitzy/documentation/maddy_26452dd8dd78.md

$ git diff --stat 26452dd8dd787dc455278b0fdd296f4a5432c768 HEAD -- go.mod go.sum
(no output = go.mod/go.sum byte-for-byte unchanged from baseline)

$ git ls-files --others --exclude-standard
(no output = no untracked/temporary files in the repo tree)
```

Two complementary facts are shown:

- **Clean tracked worktree** — `git status --porcelain` prints nothing: there
  are no uncommitted modifications, deletions, or staged changes to any tracked
  file (this document is committed).
- **Sole addition since baseline** — `git diff --name-status <baseline> HEAD`
  prints exactly one line, `A blitzy/documentation/maddy_26452dd8dd78.md`. The `A`
  (added) status on that single path, with **no** `M` (modified) or `D` (deleted)
  entry for any other file, is positive proof that the whole change set is this one
  new document. The `git diff --name-status <baseline> -- .` form confirms the same
  for the working tree; the `go.mod`/`go.sum` diff is empty (manifests byte-for-byte
  unchanged); and `git ls-files --others --exclude-standard` is empty, so no
  untracked or temporary file remains in the repository tree (temporary observation
  scripts were kept outside the repository, under `/tmp`, and removed afterward).

One intentional exception: `git diff --check 26452dd8dd787dc455278b0fdd296f4a5432c768 HEAD`
(the baseline-relative form) exits non-zero (status 2) because the no-field log records
preserve the logger's terminal literal TAB, as explained in §1.4. (The bare
`git diff --check` exits `0` here, since this document is committed and that form
inspects only unstaged worktree changes.) This is deliberate exact-output fidelity, not
a stray-whitespace defect, and it flags only lines inside this document's fenced
transcripts — never any source file.
