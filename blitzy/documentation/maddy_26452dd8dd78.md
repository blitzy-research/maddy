# maddy Outbound Delivery Under Destination‑Failure Conditions — An Empirical, Code‑Grounded Analysis

This document explains, with precision, **what the [maddy](https://github.com/foxcpp/maddy) mail server actually does** when an outbound message cannot be delivered because the destination SMTP server is non‑responsive (it times out) or otherwise fails transiently. It answers eight specific questions about the connection‑attempt sequence, real timeout durations, retry log entries, on‑disk queue persistence, scheduler prioritization, head‑of‑line blocking, and queue starvation.

Every answer is grounded in the source code (cited as `path:Lstart-Lend`) **and** corroborated by runtime observation. Where the project's own documentation disagrees with the code, **the code is treated as authoritative** and the discrepancy is flagged in the final section.

---

## Subject Under Investigation

| Property | Value |
|----------|-------|
| Module | `github.com/foxcpp/maddy` (`go.mod:L1`) |
| Commit | `26452dd8dd787dc455278b0fdd296f4a5432c768` |
| HEAD subject | *"target/remote: Rewrite connection part to allow more concurrency"* |
| Language floor | `go 1.13` (`go.mod:L3`); build/test verified with Go **1.21.6** |
| Pinned SMTP client | `github.com/emersion/go-smtp v0.12.1-0.20191206174923-1f576e0ec85c` (`go.mod:L19`, `go.sum:L60-L61`) |

The most recent commit rewrote the remote target's connection logic specifically to *allow more concurrency*; that change is directly relevant to the head‑of‑line‑blocking and starvation questions (Q7, Q8).

---

## Methodology & Environment

Answers were produced by combining two techniques:

1. **Static code tracing.** Every behavioral claim is anchored to an exact `file:line` citation that was re‑verified against the source at this commit.
2. **Runtime observation.** Claims are corroborated by actually building and running maddy's code.

**Why the existing in‑package tests are the primary empirical vehicle.** maddy's production retry intervals are *hardcoded* to roughly 15 minutes and are **not** configurable (only `max_tries` and `max_parallelism` are exposed — see below). Waiting out the real back‑off schedule is therefore impractical. The fastest **no‑modify** way to observe the full retry sequence, on‑disk persistence, and crash‑recovery is to run the existing queue tests, because the test harness `newTestQueueDir` (`internal/target/queue/queue_test.go:L47-L54`) *overrides* the otherwise‑hardcoded timing by direct struct assignment — `initialRetryTime = 0` (`L50`), `retryTimeScale = 1` (`L51`), `postInitDelay = 0` (`L52`), `maxTries = 5` (`L53`) — and starts the queue with parallelism `1` (`q.start(1)`, `L63`). The fake delivery target `unreliableTarget` (`internal/target/queue/queue_test.go:L72-L85`) injects deterministic temporary/permanent failures, so the entire retry life‑cycle runs in well under a second without touching any committed file.

**Commands actually executed** (against an out‑of‑tree throwaway copy of the repository under `/tmp`, never against the committed tree, and never modifying any repository file):

```text
# Real queue behavior, verbose, with production-style log formatting:
go test ./internal/target/queue/ -run TestQueueDelivery_TemporaryFail        -v -test.debuglog -test.directlog
go test ./internal/target/queue/ -run TestQueueDelivery_MultipleAttempts     -v -test.debuglog -test.directlog
go test ./internal/target/queue/ -run TestQueueDelivery_SerializationRoundtrip -v -test.debuglog -test.directlog
go test ./internal/target/queue/ -run TestQueueDelivery_DeserlizationCleanUp   -v -test.debuglog -test.directlog   # note the misspelling in the real test name
go test ./internal/target/queue/ -race -cover                                                                     # => PASS, 74.7% coverage, 0 data races
```

Two **throwaway** test files (`blitzy_adhoc_offbyone_test.go`, `blitzy_adhoc_ondisk_test.go`) and a standalone dial‑measurement program were created **outside** the repository tree to capture the off‑by‑one attempt count, the on‑disk file set, and the real TCP timeout durations. None of this scaffolding is part of, or committed to, the repository.

**No application‑level timeout exists at this commit** (proven in Q2). Consequently, the Q2 durations are **OS/TCP‑governed**, and were *measured* with a dialer that exactly replicates maddy's construction, then framed with operating‑system and RFC context. The production retry back‑off schedule (≈15 min base, doubling) is **computed and documented**, not waited out in real time.

> **Note on log visibility in tests.** maddy's debug lines (e.g. `delivery attempt #N`, the semaphore waits) are suppressed unless `-test.debuglog` is passed, and maddy's ISO‑8601 timestamp is stripped in the default test routing (which goes through `t.Log`) but appears verbatim when `-test.directlog` routes output through the production `WriterOutput`. The captured samples below were taken with both flags so they reflect the **production** line shape.

---

## Defaults & Key Constants (quick reference)

| Concept | Value | Citation |
|---------|-------|----------|
| `max_tries` (config directive, default) | **8** | `internal/target/queue/queue.go:L204`; `maddy.conf:L125` |
| `max_parallelism` (config directive, default) | **16** | `internal/target/queue/queue.go:L205`; `maddy.conf:L128` |
| `location` (config directive, default) | `StateDirectory/<queue-name>` | `internal/target/queue/queue.go:L206,L228-L229` |
| `initialRetryTime` (hardcoded) | `15 * time.Minute` | `internal/target/queue/queue.go:L185` |
| `retryTimeScale` (hardcoded) | `2` | `internal/target/queue/queue.go:L186` |
| `postInitDelay` (hardcoded) | `10 * time.Second` | `internal/target/queue/queue.go:L187` |
| `StateDirectory` (default) | `/var/lib/maddy` | `maddy.go:L59,L189-L190` |
| Outbound SMTP port | `25` | `internal/target/remote/remote.go:L41` |
| Production `msg_id` length | 8 hex chars | `internal/msgpipeline/msgid.go:L12-L16` |
| Test‑harness `msg_id` length | 40 hex chars | `internal/testutils/target.go:L184-L185,L192` |

**Crucial off‑by‑one (detailed in Q1):** a configured `max_tries = N` produces **N + 1** actual delivery attempts. Thus the production default of `8` yields **9** attempts, and the test value of `5` yields **6** (empirically proven below).

---

## Question 1 — Connection attempt sequence.

> With a small `max_tries` pointed at a non-responsive SMTP destination that times out, what is the exact sequence of connection attempts?

### Answer

There are **two nested layers** of "attempt," and conflating them is the most common source of confusion:

1. **Message‑level *delivery* attempts (the outer loop).** The queue's `tryDelivery` runs one `deliver()` call per pass (`internal/target/queue/queue.go:L365-L429`). Each pass is *one delivery attempt*. After the attempt, a terminal check decides whether to give up or reschedule the whole message for a later retry via the time wheel.
2. **Per‑attempt MX *connection* attempts (the inner loop).** Within a single delivery attempt, the remote target's `connectionForDomain` (`internal/target/remote/connect.go:L113-L199`) iterates the destination domain's MX records **in ascending preference order** (the records are sorted at `connect.go:L262`), calling `conn.Connect` once per MX (`connect.go:L164`), emitting a `"trying"` debug log per MX (`connect.go:L163`), and `continue`‑ing to the next MX on failure (`connect.go:L188`). A *null MX* (`"."`) short‑circuits to a `556` error (`connect.go:L155-L160`). If the domain publishes no MX records, RFC 5321 §5.1 mandates an *implicit MX* (the domain's own A/AAAA at preference 0).

So for a non‑responsive destination the observable sequence is:

```
delivery attempt #1
   └─ dial MX[0] (preference order) → blocks for the full OS TCP timeout → fails
   └─ dial MX[1] → blocks for the full OS TCP timeout → fails
   └─ … (one dial per MX until all are exhausted)
   → all temporary → log "will retry" → reschedule (back-off)
delivery attempt #2
   └─ … same MX walk …
   …
delivery attempt #(N+1)        ← N = configured max_tries
   → terminal: give up
```

### The off‑by‑one: `max_tries = N` ⇒ **N + 1** delivery attempts

A freshly committed message starts with `TriesCount = 0`: `Start()` builds the `QueueMetadata` (`internal/target/queue/queue.go:L590-L596`) without ever assigning `TriesCount`, so it takes the zero value. Inside `tryDelivery` the ordering is decisive:

- `partialErr := q.deliver(...)` runs **first** (`internal/target/queue/queue.go:L369`) — the attempt is *already performed*.
- The **terminal check** is evaluated next: `if meta.TriesCount == q.maxTries || len(partialErr.TemporaryFailed) == 0` (`queue.go:L390`).
- **Only if the message is *not* terminal** does the counter increment run: `meta.TriesCount++` (`queue.go:L407`).

Because the `== maxTries` comparison happens **before** the increment, the loop performs `deliver()` for `TriesCount = 0, 1, 2, …, maxTries` — that is **`maxTries + 1`** attempts in total.

**Empirical proof (test harness `maxTries = 5` ⇒ exactly 6 attempts).** A throwaway test (run outside the repo tree) drove an always‑temporary‑failing target and counted the actual `deliver()` invocations. Captured production‑formatted output:

```text
2026-06-30T15:51:36.535Z [debug] queue: delivery attempt #1	{"msg_id":"44e9a8a49dabf3f7cc9fe7404b3f9194861af21a"}
2026-06-30T15:51:36.562Z queue: will retry	{"attempts_count":1,"msg_id":"…","next_try_delay":"-738ns","rcpts":["tester1@example.org"]}
2026-06-30T15:51:36.562Z [debug] queue: delivery attempt #2	{…}
2026-06-30T15:51:36.564Z queue: will retry	{"attempts_count":2,…}
2026-06-30T15:51:36.564Z [debug] queue: delivery attempt #3	{…}
2026-06-30T15:51:36.566Z queue: will retry	{"attempts_count":3,…}
2026-06-30T15:51:36.566Z [debug] queue: delivery attempt #4	{…}
2026-06-30T15:51:36.567Z queue: will retry	{"attempts_count":4,…}
2026-06-30T15:51:36.567Z [debug] queue: delivery attempt #5	{…}
2026-06-30T15:51:36.569Z queue: will retry	{"attempts_count":5,…}
2026-06-30T15:51:36.569Z [debug] queue: delivery attempt #6	{…}
    OBSERVED delivery attempts = 6 (maxTries=5, expected maxTries+1=6)
--- PASS
```

There are exactly **6** `delivery attempt #` lines (`#1`…`#6`) and **5** intervening `will retry` reschedules (`attempts_count` 1→5). This matches the `deliver()`@`L369` → check@`L390` → `++`@`L407` ordering precisely. The production default `max_tries = 8` (`queue.go:L204`) therefore yields **9** attempts.

> The negative `next_try_delay` (e.g. `-738ns`) is an artifact of the test harness zeroing `initialRetryTime`/`postInitDelay`; in production the delay follows the back‑off table below.

### How a connection timeout is classified (why it keeps retrying)

A connection timeout or refusal carries **no SMTP status code**. maddy classifies such errors as *temporary/unspecified* via `exterrors.IsTemporaryOrUnspec` (`internal/target/queue/queue.go:L96`, and again at `L336` where it is mapped to a `451`). The recipient therefore lands in the `TemporaryFailed` set, `len(partialErr.TemporaryFailed) != 0`, and the terminal check at `L390` is **not** satisfied until `TriesCount == maxTries`. So a timing‑out destination is retried for the full `N + 1` attempts.

### Production back‑off schedule (computed)

The reschedule delay uses `nextTryTime = initialRetryTime * retryTimeScale^(TriesCount-1)` (`internal/target/queue/queue.go:L121-L122,L413-L414`) with production constants `initialRetryTime = 15m`, `retryTimeScale = 2`, `max_tries = 8`:

| After attempt # | Next attempt # | Delay | Cumulative |
|-----------------|----------------|-------|------------|
| 1 | 2 | 15 m  | 15 m (0.25 h) |
| 2 | 3 | 30 m  | 45 m (0.75 h) |
| 3 | 4 | 60 m  | 105 m (1.75 h) |
| 4 | 5 | 120 m | 225 m (3.75 h) |
| 5 | 6 | 240 m | 465 m (7.75 h) |
| 6 | 7 | 480 m | 945 m (15.75 h) |
| 7 | 8 | 960 m | 1905 m (31.75 h) |
| 8 | 9 | 1920 m | **3825 m (63.75 h ≈ 2.66 days)** |

After attempt **#9** (`TriesCount == 8 == maxTries`) the message gives up. Total wall‑clock to final give‑up ≈ **3825 minutes ≈ 63.75 hours ≈ 2.66 days**. RFC 5321 recommends retrying transient (4xx) failures for **~4–5 days**; maddy's default abandons the message roughly **1.5×–1.9× sooner** than that window. (This is reported as a behavioral observation, not a defect to fix.)

### Code citations

- Outer retry loop & terminal ordering: `internal/target/queue/queue.go:L365-L429` (especially `L369`, `L390`, `L407`).
- Fresh message starts at `TriesCount = 0`: `internal/target/queue/queue.go:L590-L596`.
- Back‑off formula: `internal/target/queue/queue.go:L121-L122,L413-L414`.
- Temporary/unspecified classification: `internal/target/queue/queue.go:L96,L336`.
- Inner MX iteration & per‑MX dial: `internal/target/remote/connect.go:L113-L199` (`L154` loop, `L163` "trying" log, `L164` `Connect`, `L188` continue, `L262` sort, `L155-L160` null MX).
- Default `max_tries` value: `internal/target/queue/queue.go:L204`; `maddy.conf:L125`.

### Rationale/Thinking

Within a single delivery attempt, the *connection* sequence is "one dial per MX host, in ascending preference order, until one succeeds or all are exhausted." Across attempts, the *delivery* sequence is the `N + 1` message‑level retries scheduled by the time wheel (Q6). The off‑by‑one is not a quirk of interpretation — it is a direct, provable consequence of evaluating `== maxTries` before the increment, and the captured 6‑attempt run makes it concrete. For an operator setting a *small* `max_tries` to fail fast against a dead destination, the practical takeaway is: you will see one more attempt than the number you configured, and each attempt re‑walks the full MX list, each unanswered MX costing a full OS‑level TCP timeout (quantified in Q2).

> **Empirically discovered edge case (see Reconciliations §c).** When *all* recipients fail *temporarily* right up to exhaustion, the give‑up branch produces **no** per‑recipient "not delivered" log line and emits **no** DSN/bounce — the message is silently removed from disk. This is because the give‑up code reads `meta.TemporaryFailedRcpts`, a field that is never populated. The sequence above is accurate; the *terminal* handling of a purely‑temporary exhaustion is the surprising part.

---


## Question 2 — Timeout durations.

> How long does each timeout actually take in real terms?

### Answer

**maddy configures *no* application‑level dial or SMTP‑command timeout at this commit.** The real durations are therefore dictated entirely by the **operating system's TCP stack**, not by maddy. Two qualitatively different durations result, depending on *how* the destination is unreachable:

| Destination condition | Kernel behavior | Measured result | What maddy sees |
|-----------------------|-----------------|-----------------|-----------------|
| **Closed port** (nothing listening) | Immediate `RST` | **146.472 µs** | `connect: connection refused` (`ECONNREFUSED`) — effectively instant |
| **Silently dropped** (SYN gets no reply, e.g. firewall `DROP`) | SYN retransmits with exponential back‑off, then gives up | **2m14.02s (≈134 s)** measured; **~127 s** theoretical | `connect: connection timed out` (`ETIMEDOUT`) |

Both numbers were *measured* with a standalone program that replicates maddy's dialer **exactly** — `(&net.Dialer{}).DialContext(context.Background(), "tcp", addr)` — i.e. no `Timeout` field and a deadline‑free context. The analysis host has `/proc/sys/net/ipv4/tcp_syn_retries = 6`.

```text
addr=127.0.0.1:65000 elapsed=146.472µs       err=dial tcp 127.0.0.1:65000: connect: connection refused
addr=192.0.2.1:25    elapsed=2m14.020843878s err=dial tcp 192.0.2.1:25: connect: connection timed out
```

(`192.0.2.1` is `TEST-NET-1`, RFC 5737 documentation space that is routed to nowhere — a reliable silent‑drop destination.)

The classic derivation of the ~127 s figure for `tcp_syn_retries = 6`: the initial SYN goes out at t=0, then is retransmitted with doubling RTO at roughly t = 1, 3, 7, 15, 31, 63 s, after which the kernel waits one final ~64 s RTO before returning `ETIMEDOUT` — summing to ≈127 s. The measured 134 s is consistent with this (the exact value depends on the kernel's initial RTO and rounding); the document reports **134 s as measured, ~127 s as the theoretical default**.

### There is no application timeout — the proof

- **The remote target's dialer is bare.** `internal/target/remote/remote.go:L81` constructs `(&net.Dialer{}).DialContext` with **no** `Timeout` field set.
- **The `smtpconn` wrapper's default dialer is also bare.** `internal/smtpconn/smtpconn.go:L59` uses `(&net.Dialer{}).DialContext`, and a search of the entire `internal/smtpconn/` package finds **no** `Timeout`, `Deadline`, or `SetDeadline` usage anywhere.
- **The delivery context carries no deadline.** The queue invokes delivery with `context.Background()` (`internal/target/queue/queue.go:L442`) — there is no `context.WithTimeout`/`WithDeadline` on the outbound path.
- **The pinned SMTP client has no timeout fields.** `github.com/emersion/go-smtp v0.12.1-0.20191206174923-1f576e0ec85c` exposes a `Client` struct with **no** `Timeout`, `Deadline`, or `CommandTimeout` fields (only the *server* side of that library has `ReadTimeout`/`WriteTimeout`). So there is no protocol‑level read timeout available to maddy at this version. Once a TCP connection *is* established, a server that accepts the connection but then stalls mid‑dialog would block maddy indefinitely.

### The multiplier: sequential MX dials

Because each MX host is dialed **sequentially** and each unresponsive MX blocks for the full OS timeout before `continue`‑ing to the next (`internal/target/remote/connect.go:L154-L188`), a domain with *k* silently‑dropping MX hosts costs roughly **`k × (OS timeout)`** per *delivery attempt*. With the default `tcp_syn_retries = 6`, a 3‑MX dead domain can therefore block a single delivery attempt for ≈ `3 × 134 s ≈ 6.7 minutes`. This directly amplifies the head‑of‑line and starvation behavior discussed in Q7 and Q8.

### Important clarification — the inbound 5 s timeout is unrelated

`internal/endpoint/smtp/smtp.go:L118` does contain a `context.WithTimeout(ctx, 5*time.Second)`, but this is the **inbound** rate‑limiter/semaphore acquisition timeout (it guards `ratelimit.TakeContext` at `L120` and `semaphore.TakeContext` at `L123` on the *receiving* side). It is **not** an outbound dial timeout and must not be conflated with the durations in this section.

### Code citations

- Bare dialer (no `Timeout`): `internal/target/remote/remote.go:L81`.
- Bare `smtpconn` dialer; no timeout/deadline in package: `internal/smtpconn/smtpconn.go:L59`.
- Deadline‑free delivery context: `internal/target/queue/queue.go:L442`.
- Sequential per‑MX dialing: `internal/target/remote/connect.go:L154-L188`.
- go‑smtp pin: `go.mod:L19`, `go.sum:L60-L61`.
- Inbound‑only 5 s timeout (do not conflate): `internal/endpoint/smtp/smtp.go:L118,L120,L123`.

### Rationale/Thinking

The key insight is that maddy delegates *all* connect/command timing to the kernel at this commit. That means the answer to "how long does a timeout take?" is not a maddy constant but an OS property: instantaneous for a refused connection (RST), and `tcp_syn_retries`‑bounded (≈127 s on default Linux, measured here at 134 s) for a silently dropped one. RFC 5321 specifies *minimum* client protocol timeouts (e.g. 5 minutes for the initial server greeting and for `MAIL`/`RCPT` replies) that maddy does **not** implement here, simply because the pinned go‑smtp `Client` exposes no fields to set them. The practical consequence — that a single blocked dial can hold a delivery worker for over two minutes with no way to abort sooner — is what makes the concurrency questions (Q7, Q8) matter.

---


## Question 3 — Retry log entries.

> What specific log entries mark each retry attempt (fields, timestamp format, error detail)?

### Answer

A single delivery attempt and its reschedule emit a well‑defined set of lines, all produced inside `tryDelivery` (`internal/target/queue/queue.go:L367-L418`) through a `DeliveryLogger`:

| Log message | Level | Key fields | Citation |
|-------------|-------|------------|----------|
| `delivery attempt #N` | debug | `msg_id` | `queue.go:L367` (`dl.Debugf("delivery attempt #%d", meta.TriesCount+1)`) |
| `delivered` | info | `rcpt`, `attempt`, `msg_id` | `queue.go:L378` |
| `delivery attempt failed` | error | `rcpt`, `reason` (+ structured error fields), `msg_id` | `queue.go:L384` |
| **`will retry`** (the retry marker) | info | `attempts_count`, `next_try_delay`, `rcpts`, `msg_id` | `queue.go:L415-L418` |
| `not delivered, temporary error` | info | `rcpt`, `msg_id` | `queue.go:L394` |
| `not delivered, permanent error` | info | `rcpt`, `msg_id` | `queue.go:L398` |

The **key "this is a retry" line** is `will retry`, emitted with `dl.Msg("will retry", "attempts_count", meta.TriesCount, "next_try_delay", time.Until(nextTryTime), "rcpts", meta.To)`. Note that `attempts_count` is the *already‑incremented* `TriesCount`, so the first reschedule logs `attempts_count: 1`.

**Every** delivery/queue line carries a `msg_id` field. This is injected by `DeliveryLogger`, which sets `fields["msg_id"] = msgMeta.ID` (`internal/target/delivery.go:L8-L16`, specifically `L13`) on every line it emits.

**Error detail.** The failure line's error is attached via `Logger.Error` (`internal/log/log.go:L89-L103`). That method merges the structured fields produced by `exterrors.Fields(err)` (`log.go:L90`) and, if a `"reason"` key is not already present, sets `allFields["reason"] = err.Error()` (`log.go:L98-L99`). So the human‑readable failure reason appears under `reason`, alongside any structured error fields the error type carries (e.g. `remote_server`, `smtp_code` when the error originated from an SMTP reply).

### Line shape and timestamp format

The full production line shape is:

```
<timestamp> [debug] <logger-name>: <message>\t<ordered-JSON-fields>
```

- The `<message>` then a literal **tab** then the ordered‑JSON field object is assembled in `Logger.formatMsg` (`internal/log/log.go:L138-L148`: `WriteString(msg)` at `L138`, `WriteRune('\t')` at `L139`, `marshalOrderedJSON(...)` at `L148`).
- The `<logger-name>: ` prefix (e.g. `queue: `) is prepended in `Logger.log` (`internal/log/log.go:L181-L182`: `if l.Name != "" { s = l.Name + ": " + s }`).
- The leading **timestamp** is formatted as **UTC ISO‑8601 with millisecond precision** — `stamp.UTC().Format("2006-01-02T15:04:05.000Z ")` (`internal/log/writer.go:L19`). Debug lines additionally receive a `[debug] ` prefix (`internal/log/writer.go:L22`), and each record ends with a newline (`writer.go:L25`).

### Real captured sample

Captured from the committed `TestQueueDelivery_TemporaryFail` test with production‑style formatting (`go test ./internal/target/queue/ -run TestQueueDelivery_TemporaryFail -v -test.debuglog -test.directlog`). That message is addressed to two recipients that both fail *temporarily* on attempt #1, so the capture shows the full retry‑marking vocabulary for one message — `delivery attempt #1`, a per‑recipient `delivery attempt failed`, and the `will retry` reschedule:

```text
2026-06-30T16:33:33.146Z [debug] queue: delivery attempt #1	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1"}
2026-06-30T16:33:33.146Z queue: delivery attempt failed	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester1@example.org","reason":"you shall not pass"}
2026-06-30T16:33:33.146Z queue: delivery attempt failed	{"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","rcpt":"tester2@example.org","reason":"you shall not pass"}
2026-06-30T16:33:33.148Z queue: will retry	{"attempts_count":1,"msg_id":"af8090c7eb39f761862b1f027b4f2b0bb1ce86d1","next_try_delay":"-748ns","rcpts":["tester1@example.org","tester2@example.org"]}
```

Observe: the timestamp `2026-06-30T16:33:33.146Z` exactly matches the `2006-01-02T15:04:05.000Z` layout; the debug line carries `[debug]`; the `queue: ` logger‑name prefix is present; each message is followed by a tab then the JSON field object; `msg_id` is present on every line; and each `delivery attempt failed` line carries the per‑recipient `rcpt` plus the human‑readable `reason` (here `"you shall not pass"`, the temporary error injected by `unreliableTarget`).

> **Production vs test `msg_id` length.** The `msg_id` in the sample above is **40 hex chars** because the test harness derives the ID from `sha1(t.Name())` (see Reconciliations §b). In *production*, `msg_id` is **8 hex chars** (`internal/msgpipeline/msgid.go:L12-L16`). Readers comparing test logs to production logs should expect this difference and not mistake it for a configuration change.

### Code citations

- Attempt/success/failure/retry/give‑up emitters: `internal/target/queue/queue.go:L367,L378,L384,L394,L398,L415-L418`.
- `msg_id` injection: `internal/target/delivery.go:L13`.
- Error → `reason` field: `internal/log/log.go:L89-L103` (`L90`, `L98-L99`).
- Line assembly (msg + tab + JSON): `internal/log/log.go:L138-L148`; name prefix `L181-L182`.
- Timestamp format & `[debug]` prefix: `internal/log/writer.go:L19,L22,L25`.

### Rationale/Thinking

The retry story is fully reconstructable from the logs alone: count `delivery attempt #N` lines to see how many attempts ran; read `delivery attempt failed` for the per‑recipient `reason`; and read `will retry` for the schedule (`attempts_count` and `next_try_delay`). The structured, single‑line `<ts> [debug] <name>: <msg>\t<json>` format is machine‑parseable (split on the tab, parse the JSON), which is why the empirical captures above could be filtered so cleanly. The one caveat for operators — surfaced empirically — is that on a *pure‑temporary exhaustion* the expected final `not delivered, temporary error` line does **not** appear (Reconciliations §c); the last thing you see for such a message is its final `delivery attempt #N`, after which it is silently removed.

---


## Question 4 — Queue storage location.

> Where does the queue store pending messages awaiting retry?

### Answer

Pending messages are stored **on disk**, in a directory given by the queue's `location` directive. When `location` is left empty (the usual case), it **defaults** to `filepath.Join(config.StateDirectory, q.name)`.

The full default resolution chain:

1. `location` is a configurable string directive (`internal/target/queue/queue.go:L206`).
2. If empty, the queue sets `q.location = filepath.Join(config.StateDirectory, q.name)` (`internal/target/queue/queue.go:L228-L229`).
3. `config.StateDirectory` defaults to `DefaultStateDirectory = "/var/lib/maddy"` (`maddy.go:L59`; assigned as the fallback at `maddy.go:L189-L190`; configurable via the global `state` directive at `maddy.go:L245`).
4. The directory is created (if missing) with `os.MkdirAll` during initialization (`internal/target/queue/queue.go:L233`).

For the queue block that ships in the default configuration — `queue remote_queue { … }` (`maddy.conf:L122`) — the instance name is `remote_queue`, so the default on‑disk location is:

```
/var/lib/maddy/remote_queue
```

i.e. `/var/lib/maddy/<queue-instance-name>` in general.

> The in‑package tests do **not** use `/var/lib/maddy`; `newTestQueueDir` points `location` at a fresh temporary directory (`internal/target/queue/queue_test.go:L54`), which is why all on‑disk observation in this document was possible without root or a system state directory.

### Code citations

- `location` directive & empty‑default: `internal/target/queue/queue.go:L206,L228-L229`.
- Directory creation: `internal/target/queue/queue.go:L233`.
- `StateDirectory` default `/var/lib/maddy`: `maddy.go:L59,L189-L190,L245`.
- Shipped queue instance name `remote_queue`: `maddy.conf:L122`.

### Rationale/Thinking

The queue is *durable*: messages awaiting retry survive a process restart precisely because they live on disk under `location` rather than in memory (the crash‑recovery path in Q5 repopulates the scheduler from these files at startup). The default lands under the OS‑level state directory `/var/lib/maddy`, consistent with maddy's convention of keeping mutable runtime state there. Operators who want the queue elsewhere set the `location` directive explicitly; everything downstream (the three‑file set in Q5) is written relative to this directory.

---


## Question 5 — On-disk artifacts and retry metadata.

> After a first failure but before the second retry, what files exist, what is their naming pattern, and what metadata records retry counts?

### Answer

Each queued message is persisted as a **three‑file set**, all under `q.location` (Q4) and named by the message's delivery ID:

| File | Written by | Citation |
|------|-----------|----------|
| `<id>.header` | `os.Create` in `storeNewMessage` | `internal/target/queue/queue.go:L693-L694` (removal symmetry at `L607`) |
| `<id>.body`   | `os.Create` in `storeNewMessage` | `internal/target/queue/queue.go:L712-L713` (removal symmetry at `L611`) |
| `<id>.meta`   | `updateMetadataOnDisk` (called from `storeNewMessage` at `L725`) | `internal/target/queue/queue.go:L615` |

During an atomic metadata update a transient `<id>.meta.new` exists *briefly* before being renamed over `<id>.meta` (see Atomicity below). So **between a first failure and the second retry, exactly three files exist per message**: `<id>.header`, `<id>.body`, `<id>.meta`.

**Empirical capture** (out‑of‑tree throwaway test; first recipient fails temporarily so the message persists for retry):

```text
QUEUE_DIR     = /tmp/maddy-tests-queue1794331394
DELIVERY_ID   = 3a2feb271a5a00a12a66ec87ef2a26154178b047 (len=40 hex chars)
ON_DISK_FILES = [3a2feb271a5a00a12a66ec87ef2a26154178b047.body
                 3a2feb271a5a00a12a66ec87ef2a26154178b047.header
                 3a2feb271a5a00a12a66ec87ef2a26154178b047.meta]
```

### Retry‑count metadata: the `.meta` file

`<id>.meta` is the JSON serialization of the `QueueMetadata` struct (`internal/target/queue/queue.go:L149-L170`, encoded by `json.NewEncoder(file).Encode(metaCopy)` at `L754`). Its schema:

```go
type QueueMetadata struct {
	MsgMeta *module.MsgMetadata
	From    string

	// Recipients that should be tried next.
	// May or may not be equal to partialError.TemporaryFailed.
	To []string                              // L155

	// Information about previous failures (for the bounce message).
	FailedRcpts          []string            // L159
	TemporaryFailedRcpts []string            // L160
	RcptErrs map[string]*smtp.SMTPError      // L163

	// Amount of times delivery *already tried*.
	TriesCount int                           // L166  ← the retry counter

	FirstAttempt time.Time                   // L168
	LastAttempt  time.Time                   // L169
}
```

**`TriesCount` is the retry counter** (`internal/target/queue/queue.go:L166`, commented *"Amount of times delivery already tried"*). After the **first** failed attempt it has been incremented to **`1`**, because the non‑terminal path runs `meta.TriesCount++` (`queue.go:L407`) and then persists the metadata (`updateMetadataOnDisk` at `L409`).

**Empirical capture of the real `.meta` JSON** (after the first temporary failure, before the second retry):

```json
{
  "MsgMeta": {
    "ID": "3a2feb271a5a00a12a66ec87ef2a26154178b047",
    "OriginalFrom": "tester@example.com",
    "DontTraceSender": true,
    "Quarantine": false,
    "OriginalRcpts": null,
    "SMTPOpts": { "Size": 0, "RequireTLS": false, "UTF8": false },
    "Conn": null
  },
  "From": "tester@example.com",
  "To": ["tester1@example.org"],
  "FailedRcpts": null,
  "TemporaryFailedRcpts": null,
  "RcptErrs": {
    "tester1@example.org": { "Code": 451, "EnhancedCode": [4, 0, 0], "Message": "Internal server error" }
  },
  "TriesCount": 1,
  "FirstAttempt": "2026-06-30T15:53:46.696212292Z",
  "LastAttempt":  "2026-06-30T15:53:46.74250161Z"
}
```

Reading this metadata directly answers "what records retry counts": **`"TriesCount": 1`**. Also note:
- **`To`** has been *narrowed* to just the recipient that should be tried next (`["tester1@example.org"]`) — the originally‑addressed `tester2@example.org` succeeded on attempt #1 and was dropped from the retry set.
- **`RcptErrs`** records the per‑recipient SMTP error to be reused in an eventual bounce — here a `451` / enhanced `4.0.0` / `"Internal server error"` (the `4xx` enhanced class confirms it is *temporary*).
- **`TemporaryFailedRcpts": null`** — this field is *never populated* by the queue (see Reconciliations §c), so it serializes as `null` even though the recipient did fail temporarily.

### Atomicity

Metadata writes are crash‑safe via **write‑to‑`.meta.new`‑then‑rename** (`internal/target/queue/queue.go:L742-L767`): create `metaPath + ".new"` (`L744`), encode the metadata (`L754`), `file.Sync()` (`L758`), then `os.Rename(.new → .meta)` (`L762`). A rename on the same filesystem is atomic, so a crash mid‑write can never leave a torn `.meta`.

### Crash recovery and dangling files

On startup, `readDiskQueue` (`internal/target/queue/queue.go` from ~`L621`) scans the location for `.meta` files (`L634`), extracts the ID (`L637`), and stat‑checks that the matching `.header` (`L646`) and `.body` (`L658`) exist. If one is missing, the incomplete set is pruned via `tryRemoveDanglingFile` (`L649-L650` for a missing header, `L661-L662` for a missing body), which removes the file and logs `removed dangling file <name>` (`queue.go:L798-L803`). A `.meta` that **cannot be parsed** is handled differently from a dangling file: `readMessageMeta` returns the JSON decode error (`queue.go:L786-L787`), and `readDiskQueue` simply logs `failed to read meta-data, skipping` and `continue`s to the next entry (`queue.go:L639-L642`) — the unparsable `.meta` is **left in place, neither deleted nor renamed**, during startup scanning. The `.meta_broken` rename belongs to an entirely **different** code path: it occurs only when a `panic` happens while delivering a message, where the `dispatch` deferred `recover()` (`queue.go:L292`) calls `discardBroken` (`queue.go:L295`), which renames `<id>.meta` → `<id>.meta_broken` (`queue.go:L267-L268`; the function's own doc comment notes it "is called from panic handler"). The dangling file pruning above was observed directly:

```text
queue: removed dangling file d2a3db819622e354ce399f7193234e8266b43894.meta
queue: removed dangling file d2a3db819622e354ce399f7193234e8266b43894.header
…
queue: removed dangling file c224cc0254b5e7ee69a27089c5ff8da698a797e4.meta
queue: removed dangling file c224cc0254b5e7ee69a27089c5ff8da698a797e4.body
```

### Code citations

- Three‑file create: `internal/target/queue/queue.go:L693-L694` (header), `L712-L713` (body), `L725` (meta via `updateMetadataOnDisk`).
- Naming symmetry on removal: `internal/target/queue/queue.go:L607,L611,L615`.
- `QueueMetadata` schema & `TriesCount`: `internal/target/queue/queue.go:L149-L170` (counter at `L166`).
- Increment then persist after a non‑terminal attempt: `internal/target/queue/queue.go:L407,L409`.
- Atomic update: `internal/target/queue/queue.go:L742-L767` (`L744`, `L754`, `L758`, `L762`).
- Recovery & dangling pruning: `internal/target/queue/queue.go:L634-L662,L798-L803`. Unparsable `.meta` is skipped (not renamed) on startup: `L639-L642,L786-L787`. The `.meta_broken` rename happens only on dispatch panic recovery: `L292,L295,L267-L268`.

### Rationale/Thinking

The split into `.header` / `.body` / `.meta` separates the immutable message content (header, body) from the mutable delivery state (`.meta`). Only the small `.meta` file changes between retries, and it is updated atomically — so the expensive message bytes are written once while the retry counter and recipient set are rewritten cheaply and safely on every attempt. Inspecting `.meta` between the first failure and the second retry is the most direct way to "see" the retry counter, and the capture above shows `TriesCount` advancing to `1`, `To` narrowing to the still‑failing recipient, and `RcptErrs` preserving the `451` for a future bounce. The recovery logic guarantees that a half‑written set (e.g. process killed between writing `.header` and `.body`) is cleaned up rather than resurrected as a corrupt message.

---


## Question 6 — Scheduler prioritization.

> With multiple messages queued simultaneously to different destinations, how does the scheduler prioritize them?

### Answer

Scheduling is performed by a **time wheel** (`internal/target/queue/timewheel.go`), and the **only** prioritization criterion is **earliest scheduled time** — there is **no** per‑destination fairness, priority, or round‑robin.

Mechanics:

- A single background goroutine `tick()` (`internal/target/queue/timewheel.go:L71`, launched as `go tw.tick()` at `L34`) holds all pending slots in a `container/list` linked list.
- On each cycle it scans the entire list to find the slot whose scheduled `Time` is **closest to now**: `for e := tw.slots.Front(); e != nil; e = e.Next()` (`L78-L84`), with the comparison `slot.Time.Sub(now) < closestSlot.Time.Sub(now) || closestSlot.Value == nil` (`L80`).
- It then arms a **single** timer for just that one slot — `time.NewTimer(closestSlot.Time.Sub(now))` (`L99`) — and when the timer fires, removes the slot (`tw.slots.Remove(closestEl)`, `L106`) and dispatches it (`tw.dispatch(closestSlot)`, `L109`).

Because the comparison is purely on `Time`, **destination identity is irrelevant to the scheduler.** Messages to different destinations are ordered solely by *when their next attempt is due*. When several messages are committed *simultaneously*, they all receive (near‑)identical initial schedule times, so they become due together and are dispatched in immediate succession (subject to the parallelism limit in Q7/Q8).

### Code citations

- Single `tick()` scheduler goroutine: `internal/target/queue/timewheel.go:L34,L71`.
- Earliest‑time selection scan & comparison: `internal/target/queue/timewheel.go:L78-L84` (comparison at `L80`).
- Single timer, remove, dispatch on fire: `internal/target/queue/timewheel.go:L99,L106,L109`.

### Rationale/Thinking

The scheduler is a *deadline* scheduler, not a *fairness* scheduler. Its job is simply "run the next thing that is due, when it is due," and it implements that with an O(n) scan for the minimum‑time slot plus a single armed timer. There is deliberately no notion of destination, tenant, or priority class — two messages to two different domains compete only on their `nextTryTime`. The practical implication, developed in Q7 and Q8, is that *prioritization* is decoupled from *execution capacity*: the wheel will happily declare many messages "due at once," and what happens next is governed entirely by the `max_parallelism` semaphore rather than by any scheduling policy.

---


## Question 7 — Head-of-line blocking.

> What happens to message A's retry timing when message B (to a different destination) blocks on a slow timeout?

### Answer

Message A's **scheduling** is *not* blocked by B, but A's **execution** can be — and whether it is depends entirely on the `max_parallelism` semaphore.

- **The scheduler never blocks.** When a message becomes due, the time wheel's `dispatch` (`internal/target/queue/queue.go:L275`) spawns a **new goroutine per message** (`go func() { … }()` at `L281`) and returns immediately. So the `tick()` loop continues selecting and firing the next due slot regardless of how long B's delivery takes — A becomes "due" and is dispatched **on time**.
- **But every dispatched goroutine must acquire the delivery semaphore before doing real work.** Each goroutine blocks on `q.deliverySemaphore <- struct{}{}` (`internal/target/queue/queue.go:L283`) and releases its slot with `defer func() { <-q.deliverySemaphore }()` (`L285`). The semaphore's capacity is `max_parallelism` (allocated as `make(chan struct{}, maxParallelism)` at `L242`, default **16**).

So there are two regimes:

| Condition | Effect on A |
|-----------|-------------|
| Fewer than `max_parallelism` deliveries in flight | A acquires a semaphore slot immediately and runs **concurrently** with B — B's slow timeout does **not** delay A. |
| All `max_parallelism` slots occupied by goroutines blocked on slow timeouts | A's goroutine **waits at `L283`** until a slot frees, delaying A's *actual delivery* beyond its scheduled time — even though A was dispatched on time. |

Crucially (tie‑in to Q2): because there is **no application dial timeout**, a goroutine blocked on a silently‑dropping destination holds its semaphore slot for the *full OS TCP timeout* (~127 s, measured 134 s), with no way to release it sooner. And per Q1/Q2, a single message to a *k*‑MX dead domain dials each MX sequentially, so it can hold one slot for `k × OS-timeout`. A burst of such messages is exactly what exhausts the semaphore and converts "B blocks" into "A is delayed."

### Observability

The wait is visible in the debug logs as the gap between:

- `waiting on delivery semaphore for <id>` (`internal/target/queue/queue.go:L282`), emitted *before* the goroutine blocks on the semaphore, and
- `delivery semaphore acquired for <id>` (`internal/target/queue/queue.go:L299`), emitted once the slot is granted.

A large time delta between these two lines for message A is the direct signature that A waited on the semaphore (i.e. was delayed by other in‑flight deliveries such as B).

### Code citations

- Goroutine‑per‑message dispatch (scheduler never blocks): `internal/target/queue/queue.go:L275,L281`.
- Semaphore acquire/release: `internal/target/queue/queue.go:L283,L285`.
- Semaphore capacity = `max_parallelism` (default 16): `internal/target/queue/queue.go:L205,L242`.
- Wait/acquire debug logs: `internal/target/queue/queue.go:L282,L299`.
- No timeout (slots held for full OS timeout): `internal/target/remote/remote.go:L81`; `internal/target/queue/queue.go:L442` (cross‑ref Q2).

### Rationale/Thinking

The goroutine‑per‑message design (reinforced by this commit's "allow more concurrency" rewrite of the remote target) means the *scheduler* is immune to head‑of‑line blocking: A's *due time* is honored no matter what B is doing. The head‑of‑line blocking that *can* occur is one level down, at the shared `max_parallelism` semaphore — a global concurrency cap with no per‑destination partitioning. Below the cap, A and B are fully independent; at the cap, the absence of any dial timeout (Q2) means slow destinations hog slots for minutes, and A's *execution* is pushed back until a slot frees. So the precise answer is: **A's scheduled time is never moved by B, but A's actual delivery can be delayed by B if and only if the semaphore is saturated by slow in‑flight deliveries.** This is the bridge to Q8 (starvation).

---


## Question 8 — Queue starvation.

> Is queue starvation observable in logs or connection patterns?

### Answer

**Yes.** And the **sole contention point** at which starvation can occur is the `max_parallelism` semaphore — there is no per‑destination queue, partition, or fairness mechanism anywhere else.

- The only place deliveries serialize against one another is `deliverySemaphore`, whose capacity is the `max_parallelism` directive (`internal/target/queue/queue.go:L205`, allocated at `L242`, default 16). Every delivery goroutine must pass through it (Q7).
- Because there is no fairness, a *burst of slow (timing‑out) deliveries* can occupy **all** semaphore slots and **starve** other due messages — which were dispatched on time but cannot acquire a slot.

**Observable in logs.** With `debug` enabled, starvation manifests as a growing population of messages stuck between the two semaphore log lines:

- many messages logging `waiting on delivery semaphore for <id>` (`internal/target/queue/queue.go:L282`), while
- few logging the corresponding `delivery semaphore acquired for <id>` (`internal/target/queue/queue.go:L299`).

The widening gap (in time, and in count of "waiting" vs "acquired") is the log signature of starvation. The starved messages' own `delivery attempt #N` lines are correspondingly delayed.

**Observable in connection patterns.** The number of concurrent outbound TCP connections **plateaus at `max_parallelism`** (default 16). New dials do not begin until an in‑flight delivery completes or times out and releases its slot. Because there is **no application dial timeout** (Q2), a blocked slot is held for the *full OS TCP timeout* (~127 s, measured 134 s), so under many‑dead‑destination load the plateau is sticky and starvation is pronounced: from the outside you would see exactly 16 simultaneous connection attempts hanging, with no new attempts starting until those time out.

### Code citations

- Sole contention point — `max_parallelism` semaphore: `internal/target/queue/queue.go:L205,L242` (acquire/release `L283,L285`).
- Starvation log signatures: `internal/target/queue/queue.go:L282` (`waiting on delivery semaphore for`), `L299` (`delivery semaphore acquired for`).
- No timeout ⇒ slots held for full OS timeout (cross‑ref Q2): `internal/target/remote/remote.go:L81`; `internal/target/queue/queue.go:L442`.

### Rationale/Thinking

Starvation here is a direct composition of three earlier findings: the scheduler imposes no fairness and simply fires everything that is due (Q6); execution is bounded by a single global semaphore with no per‑destination partition (Q7); and a blocked dial cannot be aborted early because no timeout exists (Q2). Put together, *k* ≥ `max_parallelism` slow destinations will pin every slot for minutes at a time, and any other message — however unrelated its destination — is starved until a slot frees. It is observable both ways: in the logs as a backlog of `waiting on delivery semaphore for …` lines without matching `acquired` lines, and in connection patterns as a hard plateau of exactly `max_parallelism` concurrent, long‑hanging connections. The mitigation levers an operator actually has are `max_parallelism` (raise the cap) — the retry intervals and the (absent) dial timeout are *not* configurable at this commit.

---


## Documentation-vs-Code Reconciliations

Where maddy's own documentation or naming could mislead a reader, the **code is authoritative**. Three points were reconciled during this investigation; each is documented as a *finding*, not a change to make.

### (a) `max_tries` default: man page says 4, code says 8

The man page `docs/man/maddy-targets.5.scd` documents `*Default*: 4` for `max_tries` (`L65`, and the example at `L23` also uses `max_tries 4`). This **contradicts** the code, which registers the directive with a default of **8** (`internal/target/queue/queue.go:L204`), matching the shipped configuration `max_tries 8` (`maddy.conf:L125`).

- **Authoritative value: `8`.** The man‑page value of `4` is a **documentation discrepancy**.
- Combined with the off‑by‑one (Q1), the production default therefore yields **9** real delivery attempts (not 8, and certainly not the 4/5 a reader of the man page might infer).

### (b) Message‑ID length: 8 hex (production) vs 40 hex (tests)

Readers comparing test logs to production logs will notice the `msg_id` field differs in length. Both are correct for their context:

- **Production: 8 hex characters.** `GenerateMsgID` reads **4** random bytes via `crypto/rand` and hex‑encodes them (`internal/msgpipeline/msgid.go:L12-L16`); the ID is assigned at SMTP ingress (`internal/endpoint/smtp/smtp.go:L112`).
- **Test harness: 40 hex characters.** The test utilities derive the ID from `sha1.Sum([]byte(t.Name()))` (20 bytes) and hex‑encode it (`internal/testutils/target.go:L184-L185`), assigning it at `L192`. SHA‑1 is 20 bytes → 40 hex chars. This is why every captured sample in this document shows a 40‑char `msg_id` (e.g. `3a2feb271a5a00a12a66ec87ef2a26154178b047`) whereas a production deployment shows 8 (e.g. `1a2b3c4d`).

**Per‑attempt suffix.** During each delivery attempt the queue appends `-<n>` to the ID where `n = TriesCount + 1`: `msgMeta.ID = msgMeta.ID + "-" + strconv.Itoa(meta.TriesCount+1)` (`internal/target/queue/queue.go:L439`). This is why the in‑package test assertions check IDs like `…-1`, `…-2`, `…-3` for successive attempts, and why attempt #6 in the Q1 capture used a `…-6` suffix internally.

### (c) Finding: `TemporaryFailedRcpts` is never populated — silent drop on pure‑temporary exhaustion

This is an empirical discovery that refines the *theoretical* description of the give‑up path. In `tryDelivery`, the terminal give‑up branch (reached when `meta.TriesCount == q.maxTries`) does two things that both depend on `meta.TemporaryFailedRcpts`:

1. it logs a per‑recipient give‑up line by iterating `for _, rcpt := range meta.TemporaryFailedRcpts` (`internal/target/queue/queue.go:L393-L394`), and
2. it gates the bounce/DSN on `if len(meta.FailedRcpts)+len(meta.TemporaryFailedRcpts) != 0` before calling `q.emitDSN(...)` (`internal/target/queue/queue.go:L400-L401`).

However, an exhaustive search of the queue package shows `meta.TemporaryFailedRcpts` is **only declared** (`queue.go:L160`) and **read** (`L393`, `L400`) — it is **never assigned** anywhere. (`meta.To` is the field that actually carries the recipients to retry; `TemporaryFailedRcpts` stays at its `nil` zero value, as the captured `.meta` JSON in Q5 confirms: `"TemporaryFailedRcpts": null`.)

**Observed consequence** for a message whose recipients fail *temporarily* every time until exhaustion:

- the `L393` loop iterates over an empty slice ⇒ **no `not delivered, temporary error` line is logged**;
- the `L400` gate is `0 + 0 != 0` ⇒ **`emitDSN` is skipped — no bounce is generated**;
- `q.removeFromDisk(meta.MsgMeta)` (`L403`) still runs ⇒ the message is **silently deleted from disk**.

This was confirmed empirically: the off‑by‑one exhaustion run (Q1) produced **zero** `not delivered` / DSN lines — after the final `delivery attempt #6`, the only subsequent action was the message's removal from disk.

- **Authoritative behavior: the code as written** — on a purely‑temporary exhaustion, maddy at this commit gives up *silently* (no give‑up log line, no DSN), which differs from the intuitive expectation (and from the theoretical description of the give‑up path) that a bounce would be emitted. Documented here strictly as a finding; per the project constraints, **no fix is proposed or made.**

---

## Summary

| # | Question | One‑line empirical answer |
|---|----------|---------------------------|
| 1 | Connection attempt sequence | Outer: `max_tries = N` ⇒ **N+1** delivery attempts (proven: 5 ⇒ 6); inner: one dial per MX in preference order. |
| 2 | Timeout durations | No app‑level timeout; **closed port ≈ 146 µs RST**, **silent‑drop ≈ 134 s** (theoretical ~127 s, `tcp_syn_retries=6`). |
| 3 | Retry log entries | `delivery attempt #N`, `delivery attempt failed` (+`reason`), `will retry` (`attempts_count`,`next_try_delay`,`rcpts`); `<ts> [debug] queue: <msg>\t<json>`, ts `2006-01-02T15:04:05.000Z`; `msg_id` on every line. |
| 4 | Storage location | On disk under `location`, default **`/var/lib/maddy/<queue-name>`** (e.g. `/var/lib/maddy/remote_queue`). |
| 5 | On‑disk artifacts | Three files `<id>.header`/`.body`/`.meta`; `.meta` JSON holds **`TriesCount`** (=1 after first failure); atomic `.meta.new`→rename. |
| 6 | Scheduler prioritization | Time wheel selects the **globally earliest‑due** slot; **no per‑destination fairness**. |
| 7 | Head‑of‑line blocking | A's *schedule* never blocked (goroutine‑per‑message); A's *execution* delayed by B only if the `max_parallelism` semaphore is saturated. |
| 8 | Queue starvation | Yes — at the `max_parallelism` semaphore; visible as `waiting…`≫`acquired…` logs and a hard plateau of `max_parallelism` hanging connections. |

**Constraints honored.** This document is the only artifact produced; no repository file was modified, added, or deleted. All runtime observation used the existing in‑package tests and out‑of‑tree throwaway scaffolding under `/tmp`, none of which is committed. All findings are specific to commit `26452dd8dd787dc455278b0fdd296f4a5432c768` and the pinned `github.com/emersion/go-smtp v0.12.1-0.20191206174923-1f576e0ec85c`.

