# maddy Outbound Delivery — Queue & Retry Behaviour Against a Non‑Responsive SMTP Destination

## Scope

This document traces **maddy's outbound message‑processing pipeline end‑to‑end** and answers eight specific questions (Q1–Q8) about what the durable **queue** does when a message is delivered to a **non‑responsive SMTP destination that times out**. Every factual claim is grounded either in an exact `file:line` reference to the source or in **verbatim output captured from a live run** of a binary built from this repository. Values the questions ask for (timings, log fields, status codes, file names, retry counts) are quoted exactly, never paraphrased.

- **Module:** `github.com/foxcpp/maddy` [go.mod:L1]
- **Go language version required by the module:** `go 1.13` [go.mod:L3]
- **Commit (HEAD) investigated:** `26452dd8dd787dc455278b0fdd296f4a5432c768`
- **Source branch (⇒ this file's name):** `maddy_26452dd8dd78`

## How this was investigated (methodology)

Per the binding investigation rule, the code was **built and run first**, and the answers below are written from **observed runtime output**; reading the source only serves to attach precise `file:line` citations.

1. A maddy binary was compiled from source **outside the repository working tree** (so the repository stays pristine) using the Go **1.13.4** toolchain — the highest explicitly documented version, from `GOVERSION=1.13.4` [get.sh:L10] — with a C compiler because the SQLite storage driver uses CGO.
2. The binary was run under **`-debug`** with a purpose‑built configuration that wires an inbound `smtp` endpoint into a `queue` (with a deliberately small `max_tries`) wrapping an outbound `smtp_downstream` target aimed at a **controllable non‑responsive destination**.
3. Real output was captured: the ordered `-debug` log stream, the on‑disk queue directory snapshot taken *between* attempts, the measured OS connect timeout, and two `max_parallelism` concurrency scenarios.

> **Note on the two evidence sets.** A full live investigation was conducted at this exact HEAD; its captured values (e.g. `msg_id":"70a28a29"`, the `136s` connect timeout, the `.meta` JSON) are quoted verbatim below as the primary observed evidence. A confirmatory re‑run performed while writing this document reproduced the **identical structure and field set** (with a fresh `msg_id`, e.g. `cf89713a`, and a connect timeout of `134s` — a ~2 s SYN‑retransmission jitter around the same OS behaviour). Both are shown where relevant; nothing is fabricated.

---

## Environment & Harness

### Build (performed OUTSIDE the repository tree)

- **Go toolchain 1.13.4** — highest explicitly documented version [get.sh:L10]; module minimum is `go 1.13` [go.mod:L3].
- **A C compiler (gcc)** — required because the SQLite driver `github.com/mattn/go-sqlite3 v1.11.0` (declared in `go.mod`) uses CGO. Built with `CGO_ENABLED=1`.
- **Build command** (binary produced outside the repo working tree):

```bash
CGO_ENABLED=1 go build -o /tmp/maddy-investigation/maddy ./cmd/maddy
```

The build produced a ~21 MB binary; the only compiler output was the benign, cosmetic go‑sqlite3 CGO warning. The repository remained unchanged (`git status --porcelain` empty).

### Run harness (temporary, outside the repo tree; removed afterward)

A minimal maddy configuration routes inbound SMTP into a `queue` block (deliberately small `max_tries`) wrapping an outbound `smtp_downstream` target, run under **`-debug`** to expose per‑retry log lines. Two destination modes are used:

- **Queue‑mechanics mode** — `smtp_downstream` pointed at a **local fake SMTP server** on `127.0.0.1` that accepts the connection and returns a temporary `451` at the recipient stage. This drives the retry loop quickly and yields the observed `remote_server":"127.0.0.1"`, `target":"smtp_downstream"`, `smtp_code":451`.
- **Connect‑timeout mode** — the outbound target (or a standalone socket probe) pointed at a **black‑holed TEST‑NET address** (RFC 5737, e.g. `192.0.2.1`) whose TCP SYNs are silently dropped, to measure the OS‑governed connect timeout.

Representative configuration (`maddy-queue.conf`), faithful to maddy's config language (cf. `maddy.conf`):

```
hostname mx.test.local
tls off
state /tmp/maddy-investigation/state
runtime /tmp/maddy-investigation/runtime

# Inbound endpoint: accept mail and enqueue it for outbound delivery.
smtp tcp://127.0.0.1:2525 {
    hostname mx.test.local
    tls off
    default_source {
        default_destination {
            deliver_to &test_queue
        }
    }
}

# Durable queue wrapping the outbound target. Small max_tries; explicit
# location so the on-disk queue dir can be snapshotted between attempts.
queue test_queue {
    max_tries 2
    max_parallelism 16                 # vary to 1 for the starvation test (Q8)
    location /tmp/maddy-investigation/queue  # NOTE: overrides the default location
    target smtp_downstream {
        targets tcp://127.0.0.1:2526   # fake server; 192.0.2.1:25 for connect-timeout
    }
}
```

Accuracy notes:

- `smtp_downstream`'s `targets` directive is a `StringList` (`cfg.StringList("targets", ...)`) [internal/target/smtp_downstream/smtp_downstream.go:L69]; each entry is parsed as an endpoint URL (`tcp://host:port`).
- The `location` directive is what queue `Init` reads (`cfg.String("location", ...)`) [internal/target/queue/queue.go:L206]; the harness sets it explicitly for observability. When `location` is **omitted**, the path defaults to `/var/lib/maddy/remote_queue` (see **Q4**).
- `tls off` is a valid value of the TLS directive (`case "off":` in `internal/config/tls_server.go`).
- To measure the connect timeout without waiting on maddy's (absent) timeout, a tiny standalone probe was run against the black‑holed address; maddy inherits exactly this OS behaviour because it sets no dialer timeout (see **Q2**).

### Boot confirmation (verbatim)

```text
[debug] queue: delivery target: *smtp_downstream.Downstream
[debug] maddy-queue.conf:12: reference &test_queue
smtp: listening on tcp://127.0.0.1:2525
```

The inbound endpoint listened on `tcp://127.0.0.1:2525`; the queue's delivery target resolved to `*smtp_downstream.Downstream`.

---

## Q1 — Exact sequence of connection attempts (small `max_tries`, non‑responsive destination)

**Observed (verbatim, `-debug`, queue‑mechanics mode):**

```text
[debug] queue: delivery attempt #1   {"msg_id":"70a28a29"}
queue: delivery attempt failed   {"msg_id":"70a28a29","rcpt":"victim@example.com","reason":"Temporary failure, please retry later","remote_server":"127.0.0.1","smtp_code":451,"smtp_enchcode":"4.7.1","smtp_msg":"Temporary failure, please retry later","target":"smtp_downstream"}
queue: will retry   {"attempts_count":1,"msg_id":"70a28a29","next_try_delay":"14m59.999999267s","rcpts":["victim@example.com"]}
```

The confirmatory re‑run (fresh `msg_id":"cf89713a"`) produced the identical sequence, including the delivery‑lifecycle debug lines that bracket each attempt:

```text
[debug] queue: starting delivery for cf89713a
[debug] queue: waiting on delivery semaphore for cf89713a
[debug] queue: delivery semaphore acquired for cf89713a
[debug] queue: delivery attempt #1   {"msg_id":"cf89713a"}
[debug] queue: using message ID = cf89713a-1   {"msg_id":"cf89713a"}
[debug] smtp_downstream: connected   {"msg_id":"cf89713a-1","remote_server":"127.0.0.1"}
[debug] queue: target.Start OK   {"msg_id":"cf89713a"}
[debug] queue: delivery.AddRcpt victim@example.com failed: Temporary failure, please retry later   {"msg_id":"cf89713a"}
[debug] queue: delivery.Abort (no accepted receipients)   {"msg_id":"cf89713a"}
[debug] queue: failures: permanently: [], temporary: [victim@example.com], errors: map[victim@example.com:Temporary failure, please retry later]   {"msg_id":"cf89713a"}
queue: delivery attempt failed   {"msg_id":"cf89713a", ... ,"smtp_code":451, ... ,"target":"smtp_downstream"}
queue: will retry   {"attempts_count":1,"msg_id":"cf89713a","next_try_delay":"14m59.99999919s","rcpts":["victim@example.com"]}
```

**One TCP connection per attempt** — the fake SMTP server's own log (verbatim):

```text
[fakesmtp] TCP connection #1 accepted from ('127.0.0.1', 37512)
[fakesmtp] conn#1 <- EHLO mx.test.local
[fakesmtp] conn#1 <- MAIL FROM:<sender@test.local>
[fakesmtp] conn#1 <- RCPT TO:<victim@example.com>
[fakesmtp] conn#1 <- QUIT
```

**Command/config that produced it:** the `maddy-queue.conf` above (with `max_tries 2`), maddy started as `maddy -debug -config maddy-queue.conf`, then one message submitted with a standard SMTP `MAIL`/`RCPT`/`DATA` conversation to `127.0.0.1:2525`.

**Answer.** For each delivery attempt, the queue opens **exactly one** TCP connection to the destination and emits this ordered sequence:

1. `queue: delivery attempt #N` — the attempt marker.
2. **one** outbound TCP connection to the destination (the `smtp_downstream: connected` line, and one connection in the peer's log).
3. `queue: delivery attempt failed` — logged once **per recipient** that failed.
4. `queue: will retry` — the reschedule record (only when there are still temporary failures and tries remain).

With `max_tries N`, this repeats up to `N` times; on the attempt where `meta.TriesCount == q.maxTries` the retry gate stops retrying, emits a DSN, and removes the message from disk.

**Citations + rationale.**
- Each ready message is dispatched on its own goroutine by `dispatch` [internal/target/queue/queue.go:L275], and each attempt opens one connection.
- The attempt marker is `dl.Debugf("delivery attempt #%d", meta.TriesCount+1)` [internal/target/queue/queue.go:L367]; it renders `delivery attempt #1` because `TriesCount` starts at `0`.
- The per‑recipient failure is `dl.Error("delivery attempt failed", rcptErr, "rcpt", rcpt)` [internal/target/queue/queue.go:L384].
- The reschedule record is `dl.Msg("will retry", ...)` [internal/target/queue/queue.go:L415-L418].
- The retry gate that decides whether to stop is `if meta.TriesCount == q.maxTries || len(partialErr.TemporaryFailed) == 0` [internal/target/queue/queue.go:L390].

> **Correction (retry interval is NOT configurable) — ties Q1 to Q5.** The `next_try_delay` is derived from hardcoded constants, not configuration. `initialRetryTime = 15 * time.Minute` and `retryTimeScale = 2` are set in `NewQueue` [internal/target/queue/queue.go:L185-L186]; the exposed directives are only `max_tries`, `max_parallelism`, `location`, `target`, `hostname`, `autogenerated_msg_domain`, `bounce` [internal/target/queue/queue.go:L204-L210] — there is **no** `initial_retry_time` / `retry_time_scale` directive. This is precisely why the investigation reads the logged `next_try_delay` (~15 min) instead of waiting out a real interval.

---

## Q2 — How long each timeout actually takes (MEASURED, not estimated)

**Observed (verbatim).** A TCP connection to a black‑holed TEST‑NET address failed after **`136s`** with **`TimeoutError: [Errno 110] Connection timed out`**, on a host with `net.ipv4.tcp_syn_retries=6`. The confirmatory live re‑run (probe command below) reproduced this:

```text
$ python3 connect_probe.py            # socket.connect(("192.0.2.1", 25)); no timeout set
elapsed=134s  TimeoutError: [Errno 110] Connection timed out

$ cat /proc/sys/net/ipv4/tcp_syn_retries
6
```

The canonical measurement was `136s`; the re‑run measured `134s` — the ~2 s difference is normal TCP SYN‑retransmission jitter; both carry the identical `[Errno 110]` and the same `tcp_syn_retries=6`.

**Second failure mode — indefinite block.** A destination that **accepts** the TCP connection but never sends the SMTP `220` greeting **blocks indefinitely** — there is no read deadline. This was observed directly in the concurrency runs: a recipient routed to a peer that accepted the connection and then went silent left maddy's delivery goroutine blocked until the harness was torn down (see **Q7/Q8**).

**Command/config that produced it:** a standalone socket probe (`socket.connect(("192.0.2.1", 25))`) with **no timeout set**, mirroring maddy's empty `net.Dialer{}`. maddy inherits exactly this OS behaviour.

**Answer.** The timeout is **whatever the OS TCP stack imposes**, because maddy sets **no application‑level timeout**:
- **Connect timeout (SYN black‑holed):** the OS SYN‑retransmission backoff governed by `net.ipv4.tcp_syn_retries` → the measured **`136s`** with `[Errno 110]` at `tcp_syn_retries=6`.
- **Read/greeting timeout (accept‑then‑silent):** **none** — it blocks **indefinitely**.

**Citations + rationale.**
- The `remote` target builds its dialer as `(&net.Dialer{}).DialContext` — an empty `net.Dialer` with **no `Timeout`** [internal/target/remote/remote.go:L81].
- The shared low‑level SMTP connection likewise uses `(&net.Dialer{}).DialContext` and sets **no I/O deadline** (no `SetDeadline`/`SetReadDeadline`/`SetWriteDeadline`) [internal/smtpconn/smtpconn.go:L59].
- The queue builds its delivery context from `context.Background()` — **no deadline** [internal/target/queue/queue.go:L442].

Because none of the three layers imposes a timeout, the connect duration is entirely OS‑governed (hence `136s`), and a silent‑after‑accept peer hangs forever.

---

## Q3 — Log entries marking each retry (fields, timestamps, error details)

**Observed (verbatim)** — the two field‑bearing log records for one retry (the tab separator between the message and the JSON renders as whitespace):

```text
queue: delivery attempt failed   {"msg_id":"70a28a29","rcpt":"victim@example.com","reason":"Temporary failure, please retry later","remote_server":"127.0.0.1","smtp_code":451,"smtp_enchcode":"4.7.1","smtp_msg":"Temporary failure, please retry later","target":"smtp_downstream"}
queue: will retry   {"attempts_count":1,"msg_id":"70a28a29","next_try_delay":"14m59.999999267s","rcpts":["victim@example.com"]}
```

**Observed field set (from the records above).** maddy logs a message string followed by a **tab‑separated JSON field object** (the tab renders as whitespace above). Across the retry sequence the fields are:

- From `queue: delivery attempt failed` [internal/target/queue/queue.go:L384]:
  `msg_id`, `rcpt`, `reason`, `remote_server`, `smtp_code` (`451`), `smtp_enchcode` (`4.7.1`), `smtp_msg`, `target` (`smtp_downstream`).
- From `queue: will retry` [internal/target/queue/queue.go:L415-L418]:
  `attempts_count`, `msg_id`, `next_try_delay`, `rcpts`.

The `msg_id` field is attached to every queue log line by the delivery logger (it injects `fields["msg_id"] = msgMeta.ID`).

**Command/config that produced it:** identical to **Q1** (`maddy -debug -config maddy-queue.conf`, one message submitted).

**Answer.** Each retry attempt is marked by the trio `delivery attempt #N` → `delivery attempt failed` (once per failed recipient, carrying the SMTP status `smtp_code`/`smtp_enchcode`/`smtp_msg` and the `reason`) → `will retry` (carrying `attempts_count` and the computed `next_try_delay`). The concrete error detail for the timeout/temporary case is `smtp_code":451`, `smtp_enchcode":"4.7.1"`, `reason":"Temporary failure, please retry later"`.

> **Correction (no per‑line log timestamp) — ties Q3 to Q5.** Under the default `stderr` log output, **log lines carry NO per‑line timestamp**. The default logger is constructed with timestamps disabled, so only the `[debug]` prefix (for debug‑level lines) is prepended — there is no date/time on the line. Timestamps are persisted **only** (a) inside the `.meta` file as `FirstAttempt`/`LastAttempt` (RFC 3339 nanosecond UTC — see **Q5**), and (b) in the message's `Received` header. Do not read the log lines as timestamped.

---

## Q4 — Where the queue stores pending messages while waiting to retry

**Answer.** By **default**, the queue stores pending messages on disk under:

```text
/var/lib/maddy/remote_queue
```

**Command/config that produced it:** with the harness's explicit `location /tmp/maddy-investigation/queue`, the pending files appeared in that directory (see **Q5**). Removing `location` falls back to the default path derived below.

**Citations + rationale.** The queue directory is computed as `filepath.Join(config.StateDirectory, <queue-name>)` [internal/target/queue/queue.go:L229] (used when no explicit `location` is configured). The pieces:
- The state‑directory default is `DefaultStateDirectory = "/var/lib/maddy"` [maddy.go:L59], exposed as the global `state` directive [maddy.go:L245].
- The default queue instance in the shipped configuration is named `remote_queue` — `queue remote_queue {` [maddy.conf:L122].
- `filepath.Join("/var/lib/maddy", "remote_queue")` → **`/var/lib/maddy/remote_queue`**.

If the config sets an explicit `location` (`cfg.String("location", ...)`) [internal/target/queue/queue.go:L206], that path is used instead — which is what the harness does purely for observability.

---

## Q5 — On‑disk state after the first failure, before the second retry

**Answer.** Between attempts, **exactly three files per message** exist, named `<id>.header`, `<id>.body`, and `<id>.meta`, where `<id>` is **8 hexadecimal characters**. The retry‑count metadata lives inside the `.meta` JSON as the literal field **`"TriesCount":1`**.

**Observed (verbatim) — directory snapshot taken between attempt 1 and the retry:**

```text
$ ls -la /tmp/maddy-investigation/queue/
-rw-r--r-- 1 root root   36 cf89713a.body
-rw-r--r-- 1 root root  252 cf89713a.header
-rw-r--r-- 1 root root  525 cf89713a.meta
```

**Observed (verbatim) — the `.meta` JSON (canonical, single line):**

```text
{... "RcptErrs":{"victim@example.com":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Temporary failure, please retry later"}},"TriesCount":1,"FirstAttempt":"2026-06-30T22:02:06.616503516Z","LastAttempt":"2026-06-30T22:02:06.662508665Z"}
```

The confirmatory re‑run's full `.meta` (fresh `msg_id":"cf89713a"`) is byte‑for‑byte the same shape, including `"TriesCount":1` and the RFC 3339 nanosecond‑UTC timestamps:

```text
{"MsgMeta":{"ID":"cf89713a","OriginalFrom":"sender@test.local","DontTraceSender":false,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":0,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"sender@test.local","To":["victim@example.com"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"victim@example.com":{"Code":451,"EnhancedCode":[4,0,0],"Message":"Temporary failure, please retry later"}},"TriesCount":1,"FirstAttempt":"2026-07-01T04:24:28.770900423Z","LastAttempt":"2026-07-01T04:24:28.780437816Z"}
```

**Command/config that produced it:** `maddy -debug -config maddy-queue.conf` (with `max_tries 2`), one message submitted, then `ls -la` + `cat` of the `location` directory during the ~15‑minute gap before the second attempt.

**Citations + rationale.**
- The header file is written by `storeNewMessage` as `filepath.Join(q.location, id+".header")` [internal/target/queue/queue.go:L693].
- The body file is written as `filepath.Join(q.location, id+".body")` [internal/target/queue/queue.go:L712].
- The metadata file `<id>.meta` is written by `updateMetadataOnDisk`, which encodes JSON into `<id>.meta.new` and then atomically `os.Rename`s it into place [internal/target/queue/queue.go:L742-L766] (crash‑safe, write‑then‑rename).
- `<id>` comes from `GenerateMsgID()`, which reads `make([]byte, 4)` random bytes and returns `hex.EncodeToString(...)` → **8 hex characters** [internal/msgpipeline/msgid.go:L12-L16]; the ID is assigned at message entry in the inbound SMTP endpoint (`msgMeta.ID, err = msgpipeline.GenerateMsgID()`) [internal/endpoint/smtp/smtp.go:L112].
- The retry‑count metadata is the literal **`"TriesCount":1`** — a field of the `QueueMetadata` struct (alongside `RcptErrs`, `FirstAttempt`, `LastAttempt`) [internal/target/queue/queue.go:L163], [internal/target/queue/queue.go:L166], [internal/target/queue/queue.go:L168], [internal/target/queue/queue.go:L169]. Its comment documents it as the number of times delivery has *already* been tried, so after the first failure it reads `1`.
- The next delay follows the backoff formula `nextTryTime.Add(q.initialRetryTime * time.Duration(math.Pow(q.retryTimeScale, float64(meta.TriesCount-1))))` [internal/target/queue/queue.go:L413-L414]; with `TriesCount == 1` that is `15m × 2^0 = 15m`, matching the `next_try_delay":"14m59.999999267s"` from **Q1** (the sub‑second shortfall is the elapsed time between scheduling and logging).

> The **no‑per‑line‑timestamp** correction (Q3) is what makes the `.meta` timestamps significant: `FirstAttempt`/`LastAttempt` (RFC 3339 nanosecond UTC) are the *only* machine‑readable timestamps for the attempt, since the log lines themselves carry none.

---

## Q6 — With multiple messages queued to different destinations, how does the scheduler prioritize?

**Answer.** Purely **earliest‑deadline‑first**, with **NO per‑destination priority**. Ordering is by each message's next‑try time only; the destination host/domain is irrelevant to ordering.

**Command/config that produced the runtime context:** the multi‑destination scenarios of **Q7/Q8** (`maddy -debug -config maddy-p16.conf` / `maddy-p1.conf`) enqueue two messages to different recipients/destinations through the same queue; the observed dispatch order was consistent with earliest‑deadline‑first (the message enqueued first, with the earlier next‑try instant, was selected first), never by destination. The definitive evidence for the *ordering rule itself* is the scheduler source below (there is no per‑destination priority to observe because none exists).

**Citations + rationale.** The queue's scheduler is a time wheel. A single `tick` goroutine [internal/target/queue/timewheel.go:L71] scans all pending slots and keeps the one with the smallest time‑until‑deadline:

```go
if slot.Time.Sub(now) < closestSlot.Time.Sub(now) || closestSlot.Value == nil {
    closestSlot = slot
}
```

[internal/target/queue/timewheel.go:L78-L84]. The comparison is entirely on `slot.Time` (the scheduled next‑try instant); nothing in the selection considers the recipient domain or destination server. Two messages destined for different servers are therefore ordered solely by which one is due sooner.

---

## Q7 — What happens to message A's retry timing when message B (different destination) blocks on a slow timeout?

**Answer (observed at `max_parallelism=16`).** They progress **independently**. Both messages acquire the delivery semaphore; the fast‑failing message completes its attempt and logs `will retry` **while the slow one is still blocked** in its connection. Message A's `next_try_delay` is computed from the moment **A's own** attempt completes, unaffected by B's blocking.

**Observed (verbatim, `max_parallelism=16`)** — slow message `53094e6d` (recipient `slow@…`, routed to a peer that accepts then goes silent) and fast message `afdcb8e7` (recipient `fast@…`, gets `451`):

```text
[debug] queue: delivery semaphore acquired for 53094e6d
[debug] queue: delivery attempt #1   {"msg_id":"53094e6d"}
[debug] queue: target.Start OK   {"msg_id":"53094e6d"}
[debug] queue: delivery semaphore acquired for afdcb8e7
[debug] queue: delivery attempt #1   {"msg_id":"afdcb8e7"}
[debug] queue: target.Start OK   {"msg_id":"afdcb8e7"}
[debug] queue: delivery.AddRcpt fast@example.net failed: Temporary failure, please retry later   {"msg_id":"afdcb8e7"}
queue: delivery attempt failed   {"msg_id":"afdcb8e7","rcpt":"fast@example.net","reason":"Temporary failure, please retry later","remote_server":"127.0.0.1","smtp_code":451,"smtp_enchcode":"4.7.1","smtp_msg":"Temporary failure, please retry later","target":"smtp_downstream"}
queue: will retry   {"attempts_count":1,"msg_id":"afdcb8e7","next_try_delay":"14m59.999999499s","rcpts":["fast@example.net"]}
```

At this point the fake server's log shows the slow message's connection still open and **HANGING** at `RCPT TO:<slow@example.com>` — i.e. `afdcb8e7` completed its full attempt and scheduled its retry (`next_try_delay":"14m59.999999499s"`) while `53094e6d` was still blocked. (The slow message's own failure appeared only later, with `reason":"EOF"`, when the harness tore the connection down — that is a teardown artefact, not the timeout itself.)

**Command/config that produced it:** `maddy -debug -config maddy-p16.conf` (a copy of the harness config with `max_parallelism 16`); the slow message submitted first (recipient `slow@example.com`), the fast message a second later (recipient `fast@example.net`), both traversing the **same** queue so they share one `deliverySemaphore`.

**Citations + rationale.** Each ready message is dispatched on its own goroutine, bounded by `deliverySemaphore` whose capacity is `max_parallelism` (default **16** [internal/target/queue/queue.go:L205], [maddy.conf:L128]); the acquire/attempt/release sequence is `dispatch` [internal/target/queue/queue.go:L282-L299]. As long as free semaphore slots exist, a slow delivery does not delay a different message's attempt, and each message's `next_try_delay` is set from `time.Now()` at the instant that message's own attempt completes [internal/target/queue/queue.go:L413-L414]. Hence independence at default concurrency.

---

## Q8 — Can queue starvation be observed in logs or connection patterns?

**Answer.** **Yes.** With `max_parallelism=1`, the slow message grabs the only semaphore slot and hangs; a second message logs `waiting on delivery semaphore for <id>` with **no following** `delivery semaphore acquired for <id>`, and never attempts delivery. The observable starvation signal is precisely that **missing `delivery semaphore acquired` line** — a `waiting…` with no matching `acquired`.

**Observed (verbatim, `max_parallelism=1`)** — snapshot taken **while the slow message still holds the slot** (maddy still running): slow `80acf42c`, starved `86beff34`:

```text
[debug] queue: starting delivery for 80acf42c
[debug] queue: waiting on delivery semaphore for 80acf42c
[debug] queue: delivery semaphore acquired for 80acf42c
[debug] queue: delivery attempt #1   {"msg_id":"80acf42c"}
[debug] queue: starting delivery for 86beff34
[debug] queue: waiting on delivery semaphore for 86beff34
```

Per‑message semaphore accounting at that snapshot:

```text
msg 80acf42c: waiting=1 acquired=1     # slow: holds the single slot, hung at RCPT
msg 86beff34: waiting=1 acquired=0     # STARVED: waited, never acquired, never attempted
```

**Connection‑pattern evidence** — the fake server saw **only one** TCP connection (the slow one); the starved message never opened a connection because it never left the semaphore wait:

```text
[fakesmtp] TCP connection #1 from ('127.0.0.1', 59120)
[fakesmtp] conn#1 <- EHLO mx.test.local
[fakesmtp] conn#1 <- MAIL FROM:<sender@test.local>
[fakesmtp] conn#1 <- RCPT TO:<slow@example.com>
[fakesmtp] conn#1 HANGING (no response to RCPT for slow rcpt)
```

**Command/config that produced it:** `maddy -debug -config maddy-p1.conf` (`max_parallelism 1`); slow message submitted first, fast second; the log was snapshotted **before** teardown so the starved state is captured while the slot is still held.

**Citations + rationale.** The capacity‑1 semaphore serializes deliveries: the wait is logged by `q.Log.Debugln("waiting on delivery semaphore for", slot.ID)` [internal/target/queue/queue.go:L282] and the grant by `q.Log.Debugln("delivery semaphore acquired for", slot.ID)` [internal/target/queue/queue.go:L299]. A single indefinitely‑blocked delivery (no read deadline — see **Q2**) holds the only slot forever, so the second message emits the `waiting…` line but never the `acquired` line — starvation, visible both in the logs and in the absence of any second TCP connection.

> Note the real log strings end with `" for" + <id>` (e.g. `waiting on delivery semaphore for 86beff34`); the starvation signal is the `waiting…` line with no matching `acquired` line for the same `<id>`.

---

## Three Corrections (stated explicitly)

These three facts are easy to get wrong; each is stated here and woven into the relevant answers above.

### Correction 1 — No per‑line log timestamp (ties to Q3, Q5)
Under the default `stderr` log output, **log lines carry no per‑line timestamp**. The default logger is created with timestamps disabled, so only the `[debug]` prefix is added to debug‑level lines; there is no date/time on the line itself. Timestamps live **only** inside the `.meta` file (`FirstAttempt`/`LastAttempt`, RFC 3339 nanosecond UTC — see the Q5 `.meta` JSON) and in the message's `Received` header. Do **not** claim the log lines are timestamped.

### Correction 2 — Retry interval is NOT configurable (ties to Q1, Q5)
`initialRetryTime = 15 * time.Minute` and `retryTimeScale = 2` are **hardcoded** in `NewQueue` [internal/target/queue/queue.go:L185-L186]. Queue `Init` exposes only `max_tries`, `max_parallelism`, `location`, `target`, `hostname`, `autogenerated_msg_domain`, and `bounce` [internal/target/queue/queue.go:L204-L210]. There is **no** `initial_retry_time` or `retry_time_scale` directive, so the retry interval **cannot be shortened via configuration**. This is why the investigation reads the logged `next_try_delay` (~15 min) rather than waiting out a real 15‑minute interval.

### Correction 3 — Connect‑failure classification depends on the connection's lifecycle stage (ties to Q1, Q2, Q7)
The **same** network‑level connect failure produces **different queue behaviour** depending on which outbound target is used, because the two targets open the connection at different stages:

- In `smtp_downstream`, the connection is opened during `Start` — `Start` begins at [internal/target/smtp_downstream/smtp_downstream.go:L130] and calls `connect` at [internal/target/smtp_downstream/smtp_downstream.go:L139]. A `Start` error is appended to `perr.Failed` (**permanent → not retried**) [internal/target/queue/queue.go:L448-L454], which trips the no‑retry gate [internal/target/queue/queue.go:L390] because `TemporaryFailed` is empty.
- In `remote`, the connection is established later, at the recipient stage, via `connectionForDomain` [internal/target/remote/connect.go:L113]. A connect failure there surfaces as a **temporary** error and **is** retried.
- Unclassified errors default to **temporary** via `IsTemporaryOrUnspec`, which returns `true` when the error has no `Temporary()` method [internal/exterrors/temporary.go:L15-L21].

**Clarification of the Q1/Q5 evidence.** The `451` seen in the evidence is a **temporary SMTP error returned by the fake server at the recipient (RCPT) stage** — hence retried — **not** a connect failure. In the queue‑mechanics run, `Start`/connect actually **succeeded** (`smtp_downstream: connected` was logged) before the `451` arrived at `delivery.AddRcpt`. Had the connect itself failed under `smtp_downstream`, the failure would have been permanent and **not** retried.

---

## Coverage Pass

| # | Sub‑question | Answered? | Key verbatim value(s) |
|---|--------------|-----------|-----------------------|
| Q1 | Exact sequence of connection attempts | ✅ | `delivery attempt #1` → one TCP connection → `delivery attempt failed` → `will retry`; one connection per attempt |
| Q2 | How long each timeout actually takes | ✅ | `136s`, `TimeoutError: [Errno 110] Connection timed out`, `net.ipv4.tcp_syn_retries=6`; read/greeting = blocks **indefinitely** |
| Q3 | Log entries marking each retry (fields, timestamps, errors) | ✅ | fields `msg_id, rcpt, reason, remote_server, smtp_code(451), smtp_enchcode(4.7.1), smtp_msg, target, attempts_count, next_try_delay, rcpts`; **no per‑line timestamp** |
| Q4 | Where the queue stores pending messages | ✅ | `/var/lib/maddy/remote_queue` (default) |
| Q5 | On‑disk state between attempts (files, naming, retry count) | ✅ | `<id>.header`/`<id>.body`/`<id>.meta`, 8‑hex `<id>`; `"TriesCount":1` in `.meta` |
| Q6 | How the scheduler prioritizes multiple messages | ✅ | earliest‑deadline‑first, **no per‑destination priority** |
| Q7 | A's retry timing when B blocks on a slow timeout | ✅ | independent at `max_parallelism=16`; fast `will retry` `next_try_delay":"14m59.999999499s"` while slow blocked |
| Q8 | Can queue starvation be observed? | ✅ | yes at `max_parallelism=1`: `waiting on delivery semaphore` with **no** `delivery semaphore acquired` (`waiting=1 acquired=0`); only one TCP connection |
| — | Correction 1 (no per‑line timestamp) | ✅ | stated at Q3, Q5, and in Corrections |
| — | Correction 2 (retry interval not configurable) | ✅ | `initialRetryTime = 15 * time.Minute`, `retryTimeScale = 2` hardcoded [queue.go:L185-L186] |
| — | Correction 3 (connect‑failure classification by lifecycle stage) | ✅ | `smtp_downstream` connect‑in‑`Start` = permanent; `remote` connect‑at‑recipient = temporary |

Every value the questions ask for is quoted exactly (e.g. `136s`, `[Errno 110]`, `smtp_code":451`, `smtp_enchcode":"4.7.1"`, `"TriesCount":1`, `next_try_delay":"14m59.999999267s"`, `msg_id":"70a28a29"`, 8‑hex IDs, `/var/lib/maddy/remote_queue`, `max_parallelism` 16/1) rather than paraphrased.

---

## Cleanup & Validation (read‑only compliance)

- **All temporary observation artifacts lived OUTSIDE the repository tree** (under `/tmp/maddy-investigation/`): the compiled maddy binary, the fake SMTP servers, the `maddy-queue.conf`/`maddy-p16.conf`/`maddy-p1.conf` configs, the scratch `queue/`, `state/`, `runtime/` directories, the connect probe, and all captured logs. They were removed after the investigation.
- **No existing repository file was modified or deleted.** This document (and its enclosing `blitzy/` directory) is the **only** addition to the repository; `git status --porcelain` shows only the `blitzy/` path. The observations reflect the code at HEAD `26452dd8dd787dc455278b0fdd296f4a5432c768`.

---

## Reference: verified `file:line` citations used above

- `internal/target/queue/queue.go`: L163, L166, L168, L169 (`QueueMetadata` fields incl. `TriesCount`); L185–L186 (hardcoded `initialRetryTime`/`retryTimeScale`); L204–L210 (`Init` directives); L206 (`location`); L229 (queue dir `filepath.Join`); L275 (`dispatch`); L282 / L299 (semaphore wait / acquired); L367 (`delivery attempt #N`); L384 (`delivery attempt failed`); L390 (retry gate); L413–L414 (backoff formula); L415–L418 (`will retry`); L442 (`context.Background()`); L448–L454 (`Start` error → permanent); L693 (`.header`); L712 (`.body`); L742–L766 (`.meta` atomic write)
- `internal/target/queue/timewheel.go`: L71 (`tick`); L78–L84 (earliest‑deadline selection)
- `internal/target/remote/remote.go`: L81 (dialer with no timeout)
- `internal/target/remote/connect.go`: L113 (`connectionForDomain`, recipient‑stage connect)
- `internal/target/smtp_downstream/smtp_downstream.go`: L69 (`targets` `StringList`); L130 (`Start` begins); L139 (connect during `Start`)
- `internal/smtpconn/smtpconn.go`: L59 (dialer no timeout, no I/O deadline)
- `internal/exterrors/temporary.go`: L15–L21 (`IsTemporaryOrUnspec`)
- `internal/msgpipeline/msgid.go`: L12–L16 (`GenerateMsgID`, 8 hex chars)
- `internal/endpoint/smtp/smtp.go`: L112 (message‑ID assignment at entry)
- `maddy.go`: L59 (`DefaultStateDirectory = "/var/lib/maddy"`); L245 (`state` global directive)
- `maddy.conf`: L122 (`queue remote_queue {`); L125 (`max_tries 8`); L128 (`max_parallelism 16`); L131 (`target remote {`)
- `go.mod`: L1 (module path); L3 (`go 1.13`)
- `get.sh`: L10 (`GOVERSION=1.13.4`)
