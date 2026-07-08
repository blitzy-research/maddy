# maddy Message Pipeline & Delivery Queue — Runtime Behavioral Investigation

This document answers, **from actual observed runtime behavior (not code reading alone)**, how the [maddy](https://github.com/foxcpp/maddy) mail server processes a message from ingress to final delivery, and how its disk-backed delivery queue behaves under repeated delivery failure, connection timeout, and concurrent multi-destination load. The subject is the maddy source tree — Go module `github.com/foxcpp/maddy`, branch `maddy_26452dd8dd78`, HEAD `26452dd` ("target/remote: Rewrite connection part to allow more concurrency"). maddy was **built from source with the pinned toolchain, run through its real SMTP entry point**, and its logs (at default and `-debug` verbosity), per-attempt durations, and on-disk queue state were captured directly. Every behavioral claim below leads with the direct answer, is backed by the complete unedited output that produced it inside a fenced block, and is anchored to a specific source location by `file:line`. The single statement established by code reading rather than direct observation is explicitly labelled as inferred (in Q2).

The five questions answered are:

- **Q1** — How does the message-processing pipeline operate end-to-end, from the moment a message enters until it reaches its final destination?
- **Q2** — With a small `max_tries` and delivery pointed at a non-responsive destination: (a) the exact sequence of connection attempts, (b) how long each timeout actually takes, and (c) the specific log entries that mark each retry.
- **Q3** — Where does the queue store pending messages on disk while they wait to be retried?
- **Q4** — Inspecting the filesystem *after the first failure but before the second retry*: (a) what files exist, (b) their naming patterns, and (c) what retry-count metadata they contain.
- **Q5** — With multiple messages queued simultaneously to *different* destinations: (a) how the scheduler prioritizes them, (b) what happens to message A's retry timing when message B blocks on a slow timeout, and (c) whether queue starvation is observable.

---

## Investigation Setup (Canonical Build, Run, and Method)

**Subject & provenance.** The code under test is the maddy source tree, Go module `github.com/foxcpp/maddy`, with `go 1.13` declared [go.mod:L3]. The branch is `maddy_26452dd8dd78` and HEAD is `26452dd`, whose commit subject is *"target/remote: Rewrite connection part to allow more concurrency."* Because that commit split the connection logic out of the remote target, the low-level connect/MX code for the `remote` target lives in `internal/target/remote/connect.go` (e.g., `connectionForDomain()` [connect.go:L113]) rather than in `remote.go`.

**Toolchain.** maddy was built with the pinned toolchain **Go 1.13.3** — `get.sh` fetches `go1.13.3.linux-amd64.tar.gz` [get.sh:L74], and the manual states the requirement as "Go toolchain (1.13 or newer)" [docs/tutorials/manual-installation.md:L5]. **CGO must be enabled** because `github.com/mattn/go-sqlite3 v1.11.0` [go.mod:L26] is a CGO package, which requires a C compiler (`gcc`); the manual notes the C compiler is otherwise optional (`CGO_ENABLED=0` disables it) [docs/tutorials/manual-installation.md:L15].

**Exact build commands.** The canonical binary was produced as a normal user with:

```
export CGO_ENABLED=1
go build -o /tmp/maddywork/maddy    ./cmd/maddy
go build -o /tmp/maddywork/maddyctl ./cmd/maddyctl
```

Both builds succeeded (exit 0); the resulting `maddy` binary was **21,812,056 bytes**. The only compiler output was a single benign CGO warning from the SQLite binding:

```
sqlite3-binding.c:125322:10: warning: function may return address of local variable [-Wreturn-local-addr]
```

**Version banner (default configuration, normal user).**

```
$ /tmp/maddywork/maddy -v
maddy unknown (built from source tree)
```

A plain source build carries no VCS-stamp `ldflags`, so the compiled `Version` value is the source default `"unknown (built from source tree)"` [maddy.go:L41]; that exact string is what the banner prints.

**Run invocations.** maddy was run as `maddy -config <path>` for default verbosity, and `maddy -config <path> -debug` wherever the `[debug]` pipeline/queue traces are shown below. Note there is **no `-statedir` CLI flag**: the state directory is set with the global `state <path>` directive [maddy.go:L245] (and the runtime directory with `runtime <path>` [maddy.go:L246]).

**Ephemeral test rig (entirely outside the repository, under `/tmp/maddyrun`).** The reproductions used a Caddyfile-style config with a loopback SMTP submission endpoint feeding a `queue` whose delivery target is a `smtp_downstream` relay pointed at a controllable Python fake-SMTP server (which can return a temporary `451`, refuse the connection, black-hole the SYN, or accept-but-stay-silent). Observed config-syntax constraints: booleans are `yes`/`no` (not `true`/`false`); `tls off` disables TLS on an endpoint; the loopback endpoint uses `insecure_auth`; and module instances are referenced with the `&name` syntax (e.g., `deliver_to &remote_queue` in the shipped config [maddy.conf:L111]). Representative config skeleton:

```
state /tmp/maddyrun/stateA
runtime /tmp/maddyrun/runA
hostname test.local
tls off
log stderr

smtp tcp://127.0.0.1:15525 {
    insecure_auth
    defer_sender_reject no
    deliver_to &q_a
}

queue q_a {
    max_tries 2
    max_parallelism 16
    target smtp_downstream {
        targets tcp://127.0.0.1:15526
        attempt_starttls no
        require_tls no
        hostname test.local
    }
    location /tmp/maddyrun/stateA/q_a
}
```

**Why a temporary `451` rejection drives the retry path (pivotal; reused in Q2 and Q5).** maddy's failure classification is *stage-dependent* — whether a failure is retried depends on which delivery stage surfaced it, not merely on the error type (full mechanism and proof in Q2). The fake downstream therefore returns a temporary `451` at the **DATA** stage, which maddy classifies as temporary and consequently **retries** — reliably producing the queue files and retry logs the questions ask about.

**Queue inspection is filesystem-based.** The `maddyctl` companion binary has **no queue subcommand**, so the Q3/Q4 evidence is gathered by directly listing and parsing the queue directory's `.header`/`.body`/`.meta` files.

**Read-only guarantee.** After all reproductions, `git status --porcelain` returned empty (0 lines) on branch `maddy_26452dd8dd78`: the source tree was left byte-for-byte unchanged. Every binary, config, fake-server script, queue directory, and log capture lived under `/tmp/maddywork` and `/tmp/maddyrun`, never inside the repository.

---

## Q1 — End-to-End Message Pipeline (Ingress → Final Delivery)

**Direct answer.** A message enters through the **SMTP endpoint**, is assigned an **8-character hex message ID**, is routed by the **two-level message pipeline** (Level-1 by source address, Level-2 by recipient address, each with modifier chains), and is handed to the target that routing resolves — here the disk-backed **`queue`** — which **acquires a delivery-concurrency semaphore** and then invokes the **delivery target** (`smtp_downstream` in the reproduction, or the default `remote` MX-based target) to attempt final delivery. In one line, as observed:

```
SMTP ingress → assign msg_id → L1 source routing → L2 rcpt routing (+modifiers) → target = queue → semaphore → delivery attempt → downstream/remote target
```

The named code path:

- **Ingress** — `internal/endpoint/smtp/smtp.go`. The message ID is assigned by `msgMeta.ID, err = msgpipeline.GenerateMsgID()` [smtp.go:L112]; the pipeline is invoked via `s.endp.pipeline.Start(...)` [smtp.go:L148]; `incoming message` is logged at [smtp.go:L128] and `RCPT ok` at [smtp.go:L243].
- **Message-ID origin** — `internal/msgpipeline/msgid.go`. `GenerateMsgID()` [msgid.go:L12] reads 4 random bytes (`make([]byte, 4)`) and hex-encodes them, yielding an **8-character lowercase hex** string.
- **Two-level routing** — `internal/msgpipeline/msgpipeline.go`. The Level-1 source block is selected by `srcBlockForAddr()` [msgpipeline.go:L155]; the Level-2 recipient block by `rcptBlockForAddr()` [msgpipeline.go:L446]; global, per-source, and per-recipient modifier chains run in between.
- **Into the queue** — `internal/target/queue/queue.go`. Delivery is scheduled through `dispatch()` [queue.go:L275]; the delivery goroutine acquires the concurrency semaphore with `q.deliverySemaphore <- struct{}{}` [queue.go:L283]; the attempt is performed by `tryDelivery()` [queue.go:L365] / `deliver()` [queue.go:L431]. The per-attempt debug line `dl.Debugf("delivery attempt #%d", meta.TriesCount+1)` [queue.go:L367] uses `TriesCount+1`, and the per-attempt delivery ID is built as `msgMeta.ID = msgMeta.ID + "-" + strconv.Itoa(meta.TriesCount+1)` [queue.go:L439] and logged at [queue.go:L440] — hence the `-<TriesCount+1>` suffix.
- **Out through a delivery target** — the default `internal/target/remote/remote.go` (MX resolution + connect: `AddRcpt()` [remote.go:L195] → `connectionForDomain()` [remote.go:L230 → connect.go:L113], SMTP port `"25"` [remote.go:L41]); or `internal/target/smtp_downstream/smtp_downstream.go` in the reproduction.

**Evidence — full ordered `-debug` trace.** A message `sender@src.local → user@dest.local` was submitted over SMTP to the loopback endpoint; the downstream returns `451` at DATA. Command:

```
python3 submit.py 127.0.0.1 15525 sender@src.local user@dest.local   # sends via smtplib to the maddy SMTP endpoint
```

Captured output (message `9f32a40e`):

```
smtp: incoming message	{"msg_id":"9f32a40e","sender":"sender@src.local","src_host":"client.local","src_ip":"127.0.0.1:32810"}
[debug] smtp/pipeline: sender sender@src.local matched by default rule	{"msg_id":"9f32a40e"}
[debug] smtp/pipeline: global rcpt modifiers: user@dest.local => user@dest.local	{"msg_id":"9f32a40e"}
[debug] smtp/pipeline: per-source rcpt modifiers: user@dest.local => user@dest.local	{"msg_id":"9f32a40e"}
[debug] smtp/pipeline: recipient user@dest.local matched by default rule (clean = user@dest.local)	{"msg_id":"9f32a40e"}
[debug] smtp/pipeline: per-rcpt modifiers: user@dest.local => user@dest.local	{"msg_id":"9f32a40e"}
[debug] smtp/pipeline: tgt.Start(sender@src.local) ok, target = queue:q_a	{"msg_id":"9f32a40e"}
smtp: RCPT ok	{"msg_id":"9f32a40e","rcpt":"user@dest.local"}
[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"9f32a40e"}
smtp: accepted	{"msg_id":"9f32a40e"}
[debug] queue: starting delivery for 9f32a40e	
[debug] queue: waiting on delivery semaphore for 9f32a40e	
[debug] queue: delivery semaphore acquired for 9f32a40e	
[debug] queue: delivery attempt #1	{"msg_id":"9f32a40e"}
[debug] queue: using message ID = 9f32a40e-1	{"msg_id":"9f32a40e"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"9f32a40e-1"}
[debug] smtp_downstream: connected	{"msg_id":"9f32a40e-1","remote_server":"127.0.0.1"}
[debug] queue: target.Start OK	{"msg_id":"9f32a40e"}
[debug] queue: delivery.AddRcpt user@dest.local OK	{"msg_id":"9f32a40e"}
[debug] queue: delivery.Body OK	{"msg_id":"9f32a40e"}
[debug] queue: delivery.Commit failed: temporary failure, try again later	{"msg_id":"9f32a40e"}
[debug] queue: failures: permanently: [], temporary: [user@dest.local], errors: map[user@dest.local:temporary failure, try again later]	{"msg_id":"9f32a40e"}
```

**Cause → effect.**

- The `msg_id` is `9f32a40e` — an **8-character lowercase hex** string, exactly as produced by `GenerateMsgID()` [msgid.go:L12] (4 random bytes → 8 hex digits).
- The ordered lines map 1:1 onto the code path: `incoming message` (ingress, [smtp.go:L128]) → the four `smtp/pipeline:` lines (source rule, global/per-source rcpt modifiers, recipient rule, per-rcpt modifiers — i.e., `srcBlockForAddr()` [msgpipeline.go:L155] and `rcptBlockForAddr()` [msgpipeline.go:L446] plus the modifier chains) → `tgt.Start(...) ok, target = queue:q_a` (routing resolved the target to the queue) → `RCPT ok` [smtp.go:L243] → `accepted` [smtp.go:L334] → the `queue:` lines: `waiting on delivery semaphore` / `delivery semaphore acquired` (the semaphore send `q.deliverySemaphore <- struct{}{}` [queue.go:L283]) and `delivery attempt #1` [queue.go:L367].
- The per-attempt delivery ID **`9f32a40e-1`** (`using message ID = 9f32a40e-1`) is `<msg_id>-<TriesCount+1>`; on the first attempt `TriesCount = 0`, so the suffix is `-1`. This matches the construction at [queue.go:L439] and its log at [queue.go:L440].

---

## Q2 — Retry & Timeout Behavior

**Direct answer.** With a small `max_tries` and delivery pointed at a non-responsive destination, maddy makes **one connection attempt per scheduled try**; **each attempt's duration is dictated entirely by the OS TCP stack and the destination's behavior — maddy sets NO application-level dial or read timeout**; and successive tries are spaced by a **hardcoded exponential backoff starting at 15 minutes and doubling each time**. Each failed attempt emits **two** default-level log lines: `queue: delivery attempt failed` (one per recipient, carrying the error detail) and `queue: will retry` (carrying the attempt count and the next delay). With `max_tries = N` the schedule is **N + 1 total attempts** before the terminal bounce.

### (a) Exact sequence of connection attempts

On each scheduled try, the queue's delivery goroutine acquires the concurrency semaphore [queue.go:L283], logs `delivery attempt #<TriesCount+1>` [queue.go:L367], and the target opens exactly **one** TCP connection to the destination. On failure, if the attempt is not terminal, `meta.TriesCount++` [queue.go:L407] and the message is rescheduled on the time wheel at the computed `nextTryTime` via `q.wheel.Add(nextTryTime, ...)` [queue.go:L414,L420]; the in-memory body/headers are dropped and reloaded from disk on the next attempt. The **terminal condition** is `if meta.TriesCount == q.maxTries || len(partialErr.TemporaryFailed) == 0` [queue.go:L390]; when it holds, the message is bounced via `emitDSN()` [queue.go:L849, called at L401] and removed from disk via `removeFromDisk()` [queue.go:L600, called at L403]. Concretely, the two physically observed attempts (detailed below) form the sequence: **attempt #1 immediately on submit, attempt #2 exactly 15 minutes later.**

### (b) Measured per-attempt timeout durations

The governing mechanism, stated up front: **maddy sets no application-level dial or read timeout anywhere in the connect/greeting path.** The dialer is a bare `(&net.Dialer{}).DialContext` in both the remote target [internal/target/remote/remote.go:L81] and the low-level connection wrapper [internal/smtpconn/smtpconn.go:L59], and the SMTP greeting is read by `smtp.NewClient` with no deadline [internal/smtpconn/smtpconn.go:L165] (invoked from `attemptConnect()` [smtpconn.go:L152]). A bare `net.Dialer{}` has a zero `Timeout` field, which Go documents as "no timeout"; therefore per-attempt duration is governed entirely by the OS/TCP stack and by how the destination misbehaves.

Because maddy prints no per-line timestamp at default verbosity, each maddy log line below was prefixed with an epoch timestamp by piping stderr through a wrapper; durations are computed from those timestamps (and cross-checked against the fake server's own per-connection timestamps). Capture command:

```
/tmp/maddywork/maddy -config /tmp/maddyrun/maddy.conf 2>&1 | while read line; do printf '%s %s\n' "$(date +%s.%N)" "$line"; done
```

Three failure modes were measured:

| Failure mode | Reproduction | Observed per-attempt duration |
|---|---|---|
| Connection refused | `smtp_downstream` → closed port `127.0.0.1:15546` | **~4 ms** (near-instant) |
| Black-hole (SYN dropped) | `smtp_downstream` → `192.0.2.1:25` (TEST-NET-1) | **136.00 s / 135.96 s** across two runs |
| Accept-but-silent (no banner) | `smtp_downstream` → server that accepts TCP but never sends `220` | **Indefinite hang** (no failure after 91 s) |

**Connection refused** (message `d69f4329`; note the near-instant delta `...246.351518381` → `...246.355522913`, ≈ 4 ms):

```
1783480246.351518381 [debug] queue: delivery attempt #1	{"msg_id":"d69f4329"}
1783480246.354563155 [debug] queue: failures: permanently: [refused@dest.local], temporary: [], errors: map[refused@dest.local:dial tcp 127.0.0.1:15546: connect: connection refused]	{"msg_id":"d69f4329"}
1783480246.355522913 queue: delivery attempt failed	{"io_op":"dial","msg_id":"d69f4329","rcpt":"refused@dest.local","reason":"dial tcp 127.0.0.1:15546: connect: connection refused","remote_addr":"127.0.0.1:15546","smtp_code":450,"smtp_enchcode":"4.4.2","smtp_msg":"Network I/O error","target":"smtp_downstream"}
1783480246.356508851 queue: not delivered, permanent error	{"msg_id":"d69f4329","rcpt":"refused@dest.local"}
```

A refused connection returns immediately (the kernel replies with a TCP RST), so the attempt is near-instant — no timeout is involved. (This run also demonstrates the stage-dependent classification discussed below: through `smtp_downstream` a connection failure is permanent, hence `not delivered, permanent error`.)

**Black-hole (SYN silently dropped)** — two runs (messages `86bf7e7b` and `94dabce0`), reason `connection timed out`:

```
# run 1 (BH1)
1783480229.410185709 [debug] queue: delivery attempt #1	{"msg_id":"86bf7e7b"}
1783480365.414867731 queue: delivery attempt failed	{"io_op":"dial","msg_id":"86bf7e7b","rcpt":"bh1@dest.local","reason":"dial tcp 192.0.2.1:25: connect: connection timed out","remote_addr":"192.0.2.1:25","smtp_code":450,"smtp_enchcode":"4.4.2","smtp_msg":"Network I/O error","target":"smtp_downstream"}
# run 2 (BH2)
1783480229.458046716 [debug] queue: delivery attempt #1	{"msg_id":"94dabce0"}
1783480365.414814558 queue: delivery attempt failed	{"io_op":"dial","msg_id":"94dabce0","rcpt":"bh2@dest.local",...,"reason":"dial tcp 192.0.2.1:25: connect: connection timed out",...}
```

Durations: run 1 = 1783480365.414867731 − 1783480229.410185709 = **136.004682 s**; run 2 = 1783480365.414814558 − 1783480229.458046716 = **135.956768 s**; the spread is 0.0479 s → **stable across 2 runs**. Mechanism (concrete, not environmental hand-waving): the host's SYN-retransmit budget was observed as

```
$ cat /proc/sys/net/ipv4/tcp_syn_retries
6
```

so for a SYN that is never answered the kernel sends the initial SYN plus 6 retransmits with exponentially increasing gaps (≈ 1 + 2 + 4 + 8 + 16 + 32 + 64 ≈ 127 s) plus a final RTO before returning `ETIMEDOUT` — ≈ 136 s here. Because maddy imposes no dial timeout, this kernel-governed value *is* the per-attempt duration. (The AAP referenced ≈ 133 s on a different host; the value reported here is the one observed on this host, ≈ 136 s, tied to `tcp_syn_retries = 6`.)

**Accept-but-silent (no banner)** — the destination completes the TCP handshake but never sends the `220` greeting (message `038a825e`). After submit, exactly one line appears and then nothing:

```
1783480279.534689114 [debug] queue: delivery attempt #1	{"msg_id":"038a825e"}
```

After **91.3 s** there was **no** `delivery attempt failed` and **no** `will retry` line — an indefinite hang. The fake server logged exactly one accepted connection and received no commands:

```
[fake_smtp mode=silent port=15566] ... ACCEPTED connection #1 from ('127.0.0.1', 40050)
```

TCP connect succeeded, but maddy then blocks inside `smtp.NewClient` reading the `220` greeting that never arrives [smtpconn.go:L165]; with no read deadline set anywhere, the attempt hangs forever. Tellingly, the `[debug] smtp_downstream: connected` line that appears for a working downstream (seen in the Q1 trace, emitted at [smtp_downstream.go:L170]) is **absent** here — confirming the block is *before* client creation completes, i.e., in the greeting read.

### (c) The retry log entries (all fields)

A failed **temporary** attempt emits two default-level lines. From the `451` reproduction (message `9f32a40e`; submit as in Q1, downstream returns `451 4.3.0` at DATA):

```
queue: delivery attempt failed	{"msg_id":"9f32a40e","rcpt":"user@dest.local","reason":"temporary failure, try again later","remote_server":"127.0.0.1","smtp_code":451,"smtp_enchcode":"4.3.0","smtp_msg":"temporary failure, try again later","target":"smtp_downstream"}
queue: will retry	{"attempts_count":1,"msg_id":"9f32a40e","next_try_delay":"14m59.99999921s","rcpts":["user@dest.local"]}
```

Fields, enumerated explicitly:

- **`queue: delivery attempt failed`** carries `msg_id`, `rcpt`, `reason`, `remote_server`, `smtp_code` (`451`), `smtp_enchcode` (`4.3.0`), `smtp_msg`, and `target` (`smtp_downstream`).
- **`queue: will retry`** carries `attempts_count`, `msg_id`, `next_try_delay`, and `rcpts`.

Provenance of the fields: `msg_id` is auto-added by the queue's per-delivery logger; the SMTP-derived fields (`smtp_code`, `smtp_enchcode`, `smtp_msg`, `reason`, `remote_server`) come from the wrapped `exterrors.SMTPError`; the `delivery attempt failed` line is emitted **per recipient** at [queue.go:L384], and `will retry` is emitted once at [queue.go:L415-L418]. For a *connection-level* failure (the refused/black-hole blocks in (b) above) the field set differs: it adds `io_op` (`dial`) and `remote_addr`, and the wrapper reports `smtp_code` `450`, `smtp_enchcode` `4.4.2`, `smtp_msg` `"Network I/O error"`.

### Mechanism — the backoff is hardcoded (not configurable)

The inter-attempt delay is `nextTryTime = LastAttempt + initialRetryTime × retryTimeScale^(TriesCount−1)`, computed at [queue.go:L414], with `initialRetryTime = 15 * time.Minute` [queue.go:L185] and `retryTimeScale = 2` [queue.go:L186] set in `NewQueue()`. **These two values are not config directives** — they are fixed in code. The only queue directives are `max_tries` (default `8` [queue.go:L204]; `maddy.conf:L125`) and `max_parallelism` (default `16` [queue.go:L205]; `maddy.conf:L128`). The retry counter is advanced by `meta.TriesCount++` [queue.go:L407].

### Mechanism — failure classification is stage-dependent (pivotal; explains `remote` vs `smtp_downstream`)

This is the crux of Q2/Q5 and is documented explicitly. Errors are **temporary-by-default**: `exterrors.IsTemporaryOrUnspec()` returns `true` when the error has no `Temporary()` method [exterrors/temporary.go:L15-L21], whereas `exterrors.IsTemporary()` returns `false` by default [exterrors/temporary.go:L25-L31]. But inside `queue.deliver()` the classification depends on the delivery **stage**:

- A failure of `q.Target.Start()` appends **all** recipients to the permanent-failure set with **no** temporariness check: `perr.Failed = append(perr.Failed, meta.To...)` [queue.go:L446,L450].
- Failures at `AddRcpt` [queue.go:L462] and at `Body`/`Commit` [queue.go:L484-L488] **are** classified via `IsTemporaryOrUnspec()`.

Consequence, with observed proof:

- **`smtp_downstream` dials eagerly inside `Start()`** — `Start()` [smtp_downstream.go:L130] invokes `d.connect(ctx)` at [smtp_downstream.go:L139] (`connect()` is defined at [smtp_downstream.go:L150]). So a connection failure is a `Start()` failure → **permanent → not retried**. **Observed:** the connection-refused run (message `d69f4329`, in (b) above) logged `queue: not delivered, permanent error` and left the queue directory **empty**, *even though* the wrapped error carried a temporary `smtp_code:450` — proving the outcome is decided by the **stage**, not by the SMTP code. (Black-hole through `smtp_downstream` is likewise permanent.)
- **`remote` connects lazily inside `AddRcpt()`** — via `connectionForDomain()` [remote.go:L195,L230 → connect.go:L113]. **Observed** (remote target; submit `test@localhost`; message `baee2bd1`):

  ```
  [debug] queue: target.Start OK	{"msg_id":"baee2bd1"}
  [debug] queue: delivery.AddRcpt test@localhost failed: no such host	{"msg_id":"baee2bd1"}
  [debug] queue: failures: permanently: [test@localhost], temporary: [], errors: map[test@localhost:no such host]	{"msg_id":"baee2bd1"}
  queue: delivery attempt failed	{"msg_id":"baee2bd1","rcpt":"test@localhost","reason":"no such host","smtp_code":554,"smtp_enchcode":"5.4.4","smtp_msg":"MX lookup error","target":"remote"}
  ```

  This proves `remote.Start()` returns OK **without** connecting, and that the resolve/connect work happens in `AddRcpt` (the opposite of `smtp_downstream`). Here the error was an NXDOMAIN MX-lookup failure wrapped as an `SMTPError` with `Code: 554` / enhanced `5.4.4` / `"MX lookup error"` via the lookup-error path in `connect.go` [connect.go:L247,L250-L258]; that wrapper has `Temporary() == false`, so `IsTemporaryOrUnspec()` returned `false` → **permanent**.
  - **(inferred)** A *bare* dial failure (connection refused or timeout) surfacing at `AddRcpt` carries no `Temporary() == false` SMTP wrapper, so `IsTemporaryOrUnspec()` [queue.go:L462, exterrors/temporary.go:L15] would return `true` → **temporary → retried** via the `remote` target. This specific outcome was not driven at runtime because, offline, the `remote` path's system-resolver MX lookup returns NXDOMAIN for made-up/localhost domains **before** it ever reaches the connect stage, and delivering to a real external MX on port 25 is out of scope; the stage-dependent mechanism itself is fully observed (the two traces above), and only this one bare-connection-refused-via-`remote` result is established by code reading.

Because of this stage dependence, the reproductions drive the retry/queue-file path with a temporary **`451` DATA-stage rejection** (a `Body`/`Commit`-stage failure → temporary → retried).

### Attempt #2 physically observed after the full ~15-minute backoff (timing stable across 2 runs)

This wait is mandatory and cannot be simulated. Two independent instances were run with `max_tries 2` (instance A: message `9f32a40e`; instance B: message `c2059df9`); attempt #2 was witnessed in both. The fake server's per-connection timestamps give the true inter-attempt gap:

```
# Instance A fake server
... mono=119298.339360 wall=03:07:26 ACCEPTED connection #1 ...
... mono=120198.342179 wall=03:22:26 ACCEPTED connection #2 ...      # gap = 900.002819 s
# Instance B fake server
... mono=119401.520462 wall=03:09:09 ACCEPTED connection #1 ...
... mono=120301.523539 wall=03:24:09 ACCEPTED connection #2 ...      # gap = 900.003077 s
```

And the `next_try_delay` doubling across the two attempts (instance A):

```
[debug] queue: delivery attempt #1	{"msg_id":"9f32a40e"}
queue: will retry	{"attempts_count":1,"msg_id":"9f32a40e","next_try_delay":"14m59.99999921s","rcpts":["user@dest.local"]}
[debug] queue: delivery attempt #2	{"msg_id":"9f32a40e"}
queue: will retry	{"attempts_count":2,"msg_id":"9f32a40e","next_try_delay":"29m59.999998895s","rcpts":["user@dest.local"]}
```

Instance B shows the same doubling for stability: `next_try_delay` `14m59.999999082s` → `29m59.999999282s`. Conclusion: the first backoff is **exactly 15 minutes** (gaps `900.002819 s` and `900.003077 s`; spread `0.00026 s`), and `next_try_delay` **doubles** (15 min → 30 min), matching `initialRetryTime = 15 min` and `retryTimeScale = 2` [queue.go:L185-L186,L414]. The **N + 1** schedule follows directly: with `max_tries = 2`, attempts #1, #2, and #3 run, and the terminal check `meta.TriesCount == q.maxTries` [queue.go:L390] fires on attempt #3, triggering `emitDSN()` [queue.go:L849] and removal from disk.

---

## Q3 — Where the Queue Stores Pending Messages on Disk

**Direct answer.** The queue stores pending messages in a directory computed as `filepath.Join(StateDirectory, queueName)` [queue.go:L229]. With the default `StateDirectory = "/var/lib/maddy"` [maddy.go:L59] and the shipped queue named `remote_queue` [maddy.conf:L122], the canonical location is **`/var/lib/maddy/remote_queue`**. An explicit `location` directive (or the queue block's first positional argument) overrides this.

**Evidence — default resolution proven at runtime.** With **no** `location` directive, `state /tmp/maddyrun/stateQ3`, and a queue named `remote_queue`, maddy auto-created the queue directory at `<state>/remote_queue`:

```
$ find /tmp/maddyrun/stateQ3 -maxdepth 1 -type d
/tmp/maddyrun/stateQ3
/tmp/maddyrun/stateQ3/remote_queue
$ ls -la /tmp/maddyrun/stateQ3/remote_queue/
-rw-r--r-- 1 root root  22 ... 00bcc3b1.body
-rw-r--r-- 1 root root 242 ... 00bcc3b1.header
-rw-r--r-- 1 root root 418 ... 00bcc3b1.meta
```

The directory name is exactly the queue's name (`remote_queue`) joined onto the state directory, which is precisely `filepath.Join(config.StateDirectory, q.name)` [queue.go:L229]. Substituting the production default `StateDirectory = /var/lib/maddy` [maddy.go:L59] for the test `state` directory yields the canonical `/var/lib/maddy/remote_queue`. The state directory is created and validated at maddy startup, and because `maddyctl` has no queue subcommand, listing this directory is the canonical way to inspect pending messages. (The retry reproductions elsewhere in this document set an explicit `location` under `/tmp/maddyrun` — a non-canonical test path — but the default `filepath.Join` behavior is what is proven here.)

---

## Q4 — On-Disk State Between Retries

**Direct answer.** Between attempts, each queued message is exactly **three files** named by an 8-character lowercase hex ID: **`<id>.header`** (RFC 822 headers, including maddy's own `Received:`), **`<id>.body`** (the raw body), and **`<id>.meta`** (a compact single-line JSON metadata object). The `.meta` records the retry count in its **`TriesCount`** field (`0` while the first attempt is in flight, `1` after the first temporary failure, `2` after the second, and so on), along with the recipients still to retry (`To`), the per-recipient errors (`RcptErrs`), and the `FirstAttempt` / `LastAttempt` timestamps.

### (a) The files that exist

Listing captured immediately after the first temporary failure — the exact Q4 target state (message `9f32a40e`):

```
$ ls -la /tmp/maddyrun/stateA/q_a/
-rw-r--r-- 1 root root  22 ... 9f32a40e.body
-rw-r--r-- 1 root root 240 ... 9f32a40e.header
-rw-r--r-- 1 root root 515 ... 9f32a40e.meta
```

Three files per message, matching the source: the `.header` path is built at [queue.go:L607], `.body` at [queue.go:L611], and `.meta` at [queue.go:L615]. The `.header` and `.body` contents:

```
$ cat 9f32a40e.header
Received: from client.local (localhost [127.0.0.1]) by test.local
 (envelope-sender <sender@src.local>) with ESMTP id 9f32a40e; Wed, 08 Jul
 2026 03:07:26 +0000
From: sender@src.local
To: user@dest.local
Subject: investigation test

$ cat 9f32a40e.body
body sent at 03:07:26
```

Note (observed + `file:line`): the `.header` contains maddy's own `Received:` header embedding the message ID `9f32a40e`, and it has **no `for` clause** — matching the documented quirk that "`for` field is never included in the `Received` header field" [docs/internals/quirks.md:L9].

### (b) Naming patterns

`<id>` is an **8-character lowercase hex** string — examples observed across the reproductions include `9f32a40e`, `c2059df9`, `038a825e`, and `00bcc3b1` — produced by `GenerateMsgID()` [msgid.go:L12] (4 random bytes hex-encoded). The metadata file is written atomically via a `.meta.new` temporary file plus rename; if a metadata file cannot be parsed, it is quarantined by renaming it to `<id>.meta_broken` [queue.go:L268].

### (c) Retry-count metadata — the full state transition (before / during / after)

The `.meta` is stored as compact single-line JSON. All three observed states follow.

**DURING the first attempt** (in-flight; captured with the silent/hung downstream so the attempt does not complete), message `038a825e` — `TriesCount:0`, `RcptErrs:{}`, `FirstAttempt == LastAttempt`:

```
{"MsgMeta":{"ID":"038a825e","OriginalFrom":"sender@src.local",...},"From":"sender@src.local","To":["silent@dest.local"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{},"TriesCount":0,"FirstAttempt":"2026-07-08T03:11:19.483540469Z","LastAttempt":"2026-07-08T03:11:19.48354052Z"}
```

**AFTER the first temporary failure** (the Q4 target state), message `9f32a40e` — `TriesCount:1`, `RcptErrs` populated, `LastAttempt` advanced, `FirstAttempt` fixed:

```
{"MsgMeta":{"ID":"9f32a40e","OriginalFrom":"sender@src.local","DontTraceSender":false,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":99,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"sender@src.local","To":["user@dest.local"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"user@dest.local":{"Code":451,"EnhancedCode":[4,0,0],"Message":"temporary failure, try again later"}},"TriesCount":1,"FirstAttempt":"2026-07-08T03:07:26.685827802Z","LastAttempt":"2026-07-08T03:07:26.731833665Z"}
```

**AFTER the second attempt**, message `9f32a40e` — `TriesCount:2`, `FirstAttempt` still fixed, `LastAttempt` advanced to the second attempt's time:

```
TriesCount = 2 ; FirstAttempt = 2026-07-08T03:07:26.685827802Z ; LastAttempt = 2026-07-08T03:22:26.734693671Z
```

The metadata fields, by name and meaning:

- **`TriesCount`** — the retry counter, incremented at [queue.go:L407]; its transition here is **0 → 1 → 2** across in-flight → after-first-failure → after-second-attempt.
- **`To`** — the recipients still to retry.
- **`RcptErrs`** — a map of recipient → `SMTPError`, each with `Code`, `EnhancedCode` (a 3-integer array, e.g. `[4,0,0]`), and `Message`.
- **`FirstAttempt`** — fixed at enqueue time; **`LastAttempt`** — advances on each attempt.
- **`FailedRcpts`** / **`TemporaryFailedRcpts`** — the permanently- and temporarily-failed recipient sets (`null` here).
- **`MsgMeta`** — the embedded message metadata (`ID`, `OriginalFrom`, `SMTPOpts.Size`, `Quarantine`, etc.).

---

## Q5 — Concurrency & Starvation

**Direct answer.** Scheduling is done by a **single-goroutine time wheel** that always selects the slot with the **soonest scheduled `Time`**; there is **no per-destination priority or fairness**. Concurrency is bounded by a **delivery semaphore sized by `max_parallelism`**. Consequently a message's *scheduled* retry time is independent of other messages, but its *actual execution* is delayed whenever all `max_parallelism` slots are occupied by slow/hung attempts — so a message to a fast destination can be **starved** behind messages stuck on a slow/hung destination. The observable starvation signature is queue files accumulating on disk with no delivery log lines.

### (a) How the scheduler prioritizes queued messages

The scheduler is the time wheel in `internal/target/queue/timewheel.go`: a single goroutine (`tick()` [timewheel.go:L71]) scans the slots and picks the one with the soonest `Time`. The selection loop compares `slot.Time.Sub(now) < closestSlot.Time.Sub(now)` [timewheel.go:L80] (inside the `for` loop at [timewheel.go:L78-L84]), waits until that time, and then dispatches via `tw.dispatch(closestSlot)` [timewheel.go:L109]. New or rescheduled work is announced on the `updateNotify` channel [timewheel.go:L21] by the `Add()` method [timewheel.go:L38], whose send is `tw.updateNotify <- target` [timewheel.go:L52]. There is **no destination-based ordering** — priority is purely soonest-scheduled-time. Concurrency is then gated by the delivery semaphore acquired at the top of each delivery goroutine, `q.deliverySemaphore <- struct{}{}` [queue.go:L283], whose capacity equals `max_parallelism`.

### (b) What happens to message A's retry timing when message B (different destination) blocks on a slow timeout

Demonstrated with `max_parallelism 1`, a silent (hung) downstream, and two messages to **different** recipients (message A = `ab1b9324` → `msgA_starve@dest.local`; message B = `8aad9688` → `msgB_starve@dest.local`). Both were persisted to disk, but the downstream accepted **exactly one** TCP connection:

```
$ ls -1 /tmp/maddyrun/stateStv/q/*.meta
/tmp/maddyrun/stateStv/q/8aad9688.meta
/tmp/maddyrun/stateStv/q/ab1b9324.meta

# fake (silent) downstream — only ONE accept:
[fake_smtp mode=silent port=15576] ... ACCEPTED connection #1 from ('127.0.0.1', 55480)

# maddy -debug: ab1b9324 acquires the sole slot; 8aad9688 blocks with NO 'acquired' line:
[debug] queue: starting delivery for ab1b9324
[debug] queue: waiting on delivery semaphore for ab1b9324
[debug] queue: delivery semaphore acquired for ab1b9324
[debug] queue: delivery attempt #1	{"msg_id":"ab1b9324"}
[debug] queue: using message ID = ab1b9324-1	{"msg_id":"ab1b9324"}
[debug] queue: starting delivery for 8aad9688
[debug] queue: waiting on delivery semaphore for 8aad9688
```

Then the silent server was killed to free the slot; message A failed (its hung greeting read returned `EOF`) and message B immediately acquired the freed slot and finally ran (failing `connection refused`, since the server was now gone):

```
1783480468.744158088 [debug] queue: delivery semaphore acquired for ab1b9324
1783480494.602544858 queue: delivery attempt failed	{"msg_id":"ab1b9324","rcpt":"msgA_starve@dest.local","reason":"EOF","remote_server":"127.0.0.1","target":"smtp_downstream"}
1783480494.605430293 [debug] queue: delivery semaphore acquired for 8aad9688
1783480494.610141238 queue: delivery attempt failed	{"io_op":"dial","msg_id":"8aad9688","rcpt":"msgB_starve@dest.local","reason":"dial tcp 127.0.0.1:15576: connect: connection refused","remote_addr":"127.0.0.1:15576","smtp_code":450,"smtp_enchcode":"4.4.2","smtp_msg":"Network I/O error","target":"smtp_downstream"}
```

Explanation: message A held the sole slot for `1783480494.602544858 − 1783480468.744158088 =` **25.858 s** (the entire time the silent server was up); message B's first delivery attempt did not run until A released the slot (B acquired it ≈ 2.9 ms later). So B's *scheduled* time was independent of A (both were scheduled immediately on the time wheel), but B's *actual execution* was gated on A's slot. The log ordering — A acquired and failed **before** B acquired — confirms the gating. This ties directly to the single global `q.deliverySemaphore` [queue.go:L283] sized by `max_parallelism` and to the fairness-free wheel selection [timewheel.go:L80].

### (c) Observable starvation

Yes — starvation is observable. The signature is queue `.meta` files accumulating on disk (here, two) while **no** `delivery attempt` / `delivery attempt failed` log line appears for the starved message until a slot frees (evident in (b): message B produced no attempt line until A released the slot). This matches a real-world report (maddy GitHub Discussion #610), in which — under a stall — the file count in `/var/lib/maddy/remote_queue` climbed with no `queue:` / `remote:` log lines, followed by a burst of errors on recovery. The general rule, grounded in the code: whenever all `max_parallelism` slots are occupied by slow/hung attempts, every other queued message — **regardless of destination** — waits, because the wheel has no per-destination fairness [timewheel.go:L80] and the semaphore is a single global gate [queue.go:L283].

---

## Appendix — Complete `file:line` Anchor Set

Every anchor cited above, grouped by file. Line numbers are for branch `maddy_26452dd8dd78` @ HEAD `26452dd`.

**`internal/endpoint/smtp/smtp.go`**
- L112 — `msgMeta.ID, err = msgpipeline.GenerateMsgID()` (message-ID assignment)
- L128 — `s.log.Msg("incoming message", ...)` (ingress log line)
- L148 — `delivery, err := s.endp.pipeline.Start(mailCtx, msgMeta, cleanFrom)` (pipeline invocation)
- L243 — `s.endp.Log.Msg("RCPT ok", "rcpt", to, "msg_id", s.msgMeta.ID)`
- L334 — `s.log.Msg("accepted", "msg_id", s.msgMeta.ID)`

**`internal/msgpipeline/msgpipeline.go`**
- L155 — `func (dd *msgpipelineDelivery) srcBlockForAddr(...)` (Level-1 source routing)
- L446 — `func (dd *msgpipelineDelivery) rcptBlockForAddr(...)` (Level-2 recipient routing)

**`internal/msgpipeline/msgid.go`**
- L12 — `func GenerateMsgID() (string, error)` — 4 random bytes hex-encoded → 8-char lowercase hex

**`internal/target/queue/queue.go`**
- L185 — `initialRetryTime: 15 * time.Minute`
- L186 — `retryTimeScale:   2`
- L187 — `postInitDelay:    10 * time.Second`
- L204 — `cfg.Int("max_tries", false, false, 8, &q.maxTries)` (default 8)
- L205 — `cfg.Int("max_parallelism", false, false, 16, &maxParallelism)` (default 16)
- L229 — `q.location = filepath.Join(config.StateDirectory, q.name)`
- L268 — rename to `<id>.meta_broken` (quarantine of unparseable metadata)
- L275 — `func (q *Queue) dispatch(value TimeSlot)`
- L278 — `q.Log.Debugln("starting delivery for", slot.ID)`
- L282 — `q.Log.Debugln("waiting on delivery semaphore for", slot.ID)`
- L283 — `q.deliverySemaphore <- struct{}{}` (concurrency gate; capacity = `max_parallelism`)
- L299 — `q.Log.Debugln("delivery semaphore acquired for", slot.ID)`
- L365 — `func (q *Queue) tryDelivery(...)`
- L367 — `dl.Debugf("delivery attempt #%d", meta.TriesCount+1)`
- L384 — `dl.Error("delivery attempt failed", rcptErr, "rcpt", rcpt)`
- L390 — terminal check: `if meta.TriesCount == q.maxTries || len(partialErr.TemporaryFailed) == 0`
- L398 — `dl.Msg("not delivered, permanent error", "rcpt", rcpt)`
- L401 — `q.emitDSN(meta, header)` (terminal bounce call)
- L403 — `q.removeFromDisk(meta.MsgMeta)` (terminal removal call)
- L407 — `meta.TriesCount++`
- L414 — `nextTryTime = nextTryTime.Add(q.initialRetryTime * time.Duration(math.Pow(q.retryTimeScale, float64(meta.TriesCount-1))))` (backoff)
- L415-L418 — `dl.Msg("will retry", "attempts_count", ..., "next_try_delay", ..., "rcpts", ...)`
- L420 — `q.wheel.Add(nextTryTime, queueSlot{...})`
- L431 — `func (q *Queue) deliver(...) partialError`
- L439 — `msgMeta.ID = msgMeta.ID + "-" + strconv.Itoa(meta.TriesCount+1)` (per-attempt delivery ID suffix)
- L440 — `dl.Debugf("using message ID = %s", msgMeta.ID)`
- L446, L450 — `q.Target.Start()` failure → `perr.Failed = append(perr.Failed, meta.To...)` (Start-stage = permanent, no temporariness check)
- L462 — `if exterrors.IsTemporaryOrUnspec(err)` (AddRcpt-stage classification)
- L484-L488 — Body/Commit-stage classification via `IsTemporaryOrUnspec()`
- L600 — `func (q *Queue) removeFromDisk(...)`
- L607 — `headerPath := filepath.Join(q.location, id+".header")`
- L611 — `bodyPath := filepath.Join(q.location, id+".body")`
- L615 — `metaPath := filepath.Join(q.location, id+".meta")`
- L849 — `func (q *Queue) emitDSN(...)`

**`internal/target/queue/timewheel.go`**
- L21 — `updateNotify chan time.Time`
- L38 — `func (tw *TimeWheel) Add(target time.Time, value interface{})`
- L52 — `tw.updateNotify <- target` (Add's notify send)
- L71 — `func (tw *TimeWheel) tick()` (single scheduler goroutine)
- L78-L84 — slot-scan `for` loop
- L80 — `if slot.Time.Sub(now) < closestSlot.Time.Sub(now) || closestSlot.Value == nil` (soonest-time selection; no per-destination fairness)
- L109 — `tw.dispatch(closestSlot)`

**`internal/target/remote/remote.go`**
- L41 — `var smtpPort = "25"`
- L81 — `dialer: (&net.Dialer{}).DialContext` (no application-level dial timeout)
- L195 — `func (rd *remoteDelivery) AddRcpt(...)` (lazy connect happens here)
- L230 — `conn, err := rd.connectionForDomain(ctx, domain)`

**`internal/target/remote/connect.go`**
- L113 — `func (rd *remoteDelivery) connectionForDomain(...)`
- L247 — `records, err = rd.rt.resolver.LookupMX(ctx, domain)`
- L250-L258 — MX-lookup error wrapped as `SMTPError{Code: SMTPCode(err,451,554), EnhancedCode: {0,4,4}, Message: "MX lookup error", TargetName: "remote"}`
- L266-L271 — fallback to A/AAAA record when no MX records are present

**`internal/target/smtp_downstream/smtp_downstream.go`**
- L130 — `func (u *Downstream) Start(...)`
- L139 — `if err := d.connect(ctx); err != nil` (eager connect from `Start()`)
- L150 — `func (d *delivery) connect(ctx context.Context) error`
- L170 — `d.log.DebugMsg("connected", ...)`

**`internal/smtpconn/smtpconn.go`**
- L59 — `Dialer: (&net.Dialer{}).DialContext` (no application-level dial timeout)
- L152 — `func (c *C) attemptConnect(...)`
- L165 — `cl, err = smtp.NewClient(conn, endp.Host)` (greeting read; no read deadline)

**`internal/exterrors/temporary.go`**
- L15-L21 — `func IsTemporaryOrUnspec(err error) bool` — returns `true` by default (L20) when no `Temporary()` method
- L25-L31 — `func IsTemporary(err error) bool` — returns `false` by default (L30)

**`maddy.go`**
- L41 — `Version = "unknown (built from source tree)"`
- L59 — `DefaultStateDirectory = "/var/lib/maddy"`
- L245 — `globals.String("state", ..., DefaultStateDirectory, &config.StateDirectory)`
- L246 — `globals.String("runtime", ..., DefaultRuntimeDirectory, &config.RuntimeDirectory)`

**`maddy.conf`**
- L111 — `deliver_to &remote_queue`
- L122 — `queue remote_queue {`
- L125 — `max_tries 8`
- L128 — `max_parallelism 16`

**`go.mod`**
- L3 — `go 1.13`
- L19 — `github.com/emersion/go-smtp v0.12.1-0.20191206174923-1f576e0ec85c`
- L26 — `github.com/mattn/go-sqlite3 v1.11.0` (CGO)
- L27 — `github.com/miekg/dns v1.1.22`

**`get.sh`**
- L74 — `wget -q 'https://dl.google.com/go/go1.13.3.linux-amd64.tar.gz'` (pinned Go 1.13.3)

**`docs/tutorials/manual-installation.md`**
- L5 — "Go toolchain (1.13 or newer)"
- L15 — "C compiler (optional, set CGO_ENABLED env. variable to 0 to disable)"

**`docs/internals/quirks.md`**
- L9 — "`for` field is never included in the `Received` header field."
