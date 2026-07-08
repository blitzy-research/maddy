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

**Queue inspection is filesystem-based.** The `maddyctl` companion binary has **no queue subcommand**, so the Q3/Q4 evidence is gathered by directly listing and parsing the queue directory's `.header`/`.body`/`.meta` files. Proven by enumerating its subcommands and grepping for a queue command:

```
$ /tmp/maddywork/maddyctl --help
NAME:
   maddyctl - maddy mail server administration utility

USAGE:
   maddyctl [global options] command [command options] [arguments...]

VERSION:
   unknown (built from source tree)

COMMANDS:
   users        User accounts management
   imap-mboxes  IMAP mailboxes (folders) management
   imap-msgs    IMAP messages management
   help, h      Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --config value  Configuration file to use (default: "/etc/maddy/maddy.conf") [$MADDY_CONFIG]
   --help, -h      show help
   --version, -v   print the version
$ /tmp/maddywork/maddyctl --help 2>&1 | grep -i queue ; echo "exit=$?"
exit=1
```

The complete `COMMANDS` list contains only `users`, `imap-mboxes`, `imap-msgs`, and `help` — there is no `queue` command — and grepping the help output for `queue` matches nothing (`grep` exit status `1`). Hence queue inspection for Q3/Q4 is necessarily filesystem-based.

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

**Evidence — full ordered `-debug` trace.** A message `sender@src.local → user@dest.local` was submitted over SMTP to the loopback endpoint (instance A, submission port `15525`); the downstream returns `451` at DATA. Exact commands (the maddy run whose `-debug` stderr is shown, and the submission):

```
$ /tmp/maddywork/maddy -config /tmp/maddyrun/maddyA.conf -debug     # process under test; its stderr is shown below
$ python3 /tmp/maddyrun/submit.py 127.0.0.1 15525 sender@src.local user@dest.local
SUBMIT wall=05:36:31.521889 epoch=1783488991.521888 -> 127.0.0.1:15525 sender@src.local -> user@dest.local
EHLO -> 250
sendmail returned (empty=accepted): {}
DONE epoch=1783488991.569109
```

Captured maddy `-debug` output (message `aff956e1`; these are maddy's own stderr lines — at this verbosity maddy prints no per-line timestamp — shown from ingress through the pipeline hand-off and the first queue delivery attempt; the two retry log lines that follow this span are shown under Q2(c)):

```
smtp: incoming message	{"msg_id":"aff956e1","sender":"sender@src.local","src_host":"client.local","src_ip":"127.0.0.1:35116"}
[debug] smtp/pipeline: sender sender@src.local matched by default rule	{"msg_id":"aff956e1"}
[debug] smtp/pipeline: global rcpt modifiers: user@dest.local => user@dest.local	{"msg_id":"aff956e1"}
[debug] smtp/pipeline: per-source rcpt modifiers: user@dest.local => user@dest.local	{"msg_id":"aff956e1"}
[debug] smtp/pipeline: recipient user@dest.local matched by default rule (clean = user@dest.local)	{"msg_id":"aff956e1"}
[debug] smtp/pipeline: per-rcpt modifiers: user@dest.local => user@dest.local	{"msg_id":"aff956e1"}
[debug] smtp/pipeline: tgt.Start(sender@src.local) ok, target = queue:q_a	{"msg_id":"aff956e1"}
smtp: RCPT ok	{"msg_id":"aff956e1","rcpt":"user@dest.local"}
[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"aff956e1"}
smtp: accepted	{"msg_id":"aff956e1"}
[debug] smtp: reset	
[debug] queue: starting delivery for aff956e1	
[debug] queue: waiting on delivery semaphore for aff956e1	
[debug] queue: delivery semaphore acquired for aff956e1	
[debug] queue: delivery attempt #1	{"msg_id":"aff956e1"}
[debug] queue: using message ID = aff956e1-1	{"msg_id":"aff956e1"}
[debug] smtp_downstream: connected	{"downstream_server":"127.0.0.1","msg_id":"aff956e1-1"}
[debug] smtp_downstream: connected	{"msg_id":"aff956e1-1","remote_server":"127.0.0.1"}
[debug] queue: target.Start OK	{"msg_id":"aff956e1"}
[debug] queue: delivery.AddRcpt user@dest.local OK	{"msg_id":"aff956e1"}
[debug] queue: delivery.Body OK	{"msg_id":"aff956e1"}
[debug] queue: delivery.Commit failed: temporary failure, try again later	{"msg_id":"aff956e1"}
[debug] queue: delivery.Commit OK	{"msg_id":"aff956e1"}
[debug] queue: failures: permanently: [], temporary: [user@dest.local], errors: map[user@dest.local:temporary failure, try again later]	{"msg_id":"aff956e1"}
```


**Cause → effect.**

- The `msg_id` is `aff956e1` — an **8-character lowercase hex** string, exactly as produced by `GenerateMsgID()` [msgid.go:L12] (4 random bytes → 8 hex digits).
- The ordered lines map 1:1 onto the code path: `incoming message` (ingress, [smtp.go:L128]) → the four `smtp/pipeline:` lines (source rule, global/per-source rcpt modifiers, recipient rule, per-rcpt modifiers — i.e., `srcBlockForAddr()` [msgpipeline.go:L155] and `rcptBlockForAddr()` [msgpipeline.go:L446] plus the modifier chains) → `tgt.Start(...) ok, target = queue:q_a` (routing resolved the target to the queue) → `RCPT ok` [smtp.go:L243] → `accepted` [smtp.go:L334] → `smtp: reset` (the endpoint resets the SMTP transaction after handing the message off) → the `queue:` lines: `waiting on delivery semaphore` / `delivery semaphore acquired` (the semaphore send `q.deliverySemaphore <- struct{}{}` [queue.go:L283]) and `delivery attempt #1` [queue.go:L367].
- The per-attempt delivery ID **`aff956e1-1`** (`using message ID = aff956e1-1`) is `<msg_id>-<TriesCount+1>`; on the first attempt `TriesCount = 0`, so the suffix is `-1`. This matches the construction at [queue.go:L439] and its log at [queue.go:L440].
- The two consecutive `delivery.Commit failed: …` and `delivery.Commit OK` lines are both real and are emitted by design: the queue logs the commit error at [queue.go:L526] and then **unconditionally** logs `delivery.Commit OK` at [queue.go:L529] regardless of the error. The authoritative failure signal is the following `failures: … temporary: [user@dest.local]` line; the message is therefore kept and retried (see Q2).

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
| Connection refused | `smtp_downstream` → closed port `127.0.0.1:15599` | **~12 ms** (near-instant) |
| Black-hole (SYN dropped) | `smtp_downstream` → `192.0.2.1:25` (TEST-NET-1) | **133.2546 s / 133.2540 s** across two runs (spread 0.0006 s) |
| Accept-but-silent (no banner) | `smtp_downstream` → server that accepts TCP but never sends `220` | **Indefinite hang** (no failure after 522 s — see below) |

**Connection refused** (message `13a8ce17`; `smtp_downstream` → closed port `127.0.0.1:15599`; submission port `15555`). Commands (submission through the real SMTP endpoint, then the queue-directory listing that proves the message was *not* persisted):

```
$ python3 /tmp/maddyrun/submit.py 127.0.0.1 15555 sender@src.local refused@dest.local
$ ls -la /tmp/maddyrun/stateRef/q_r/ ; echo "file-count=$(ls -1 /tmp/maddyrun/stateRef/q_r/ | wc -l)"
total 8
drwxr-xr-x 2 root root 4096 Jul  8 05:40 .
drwxr-xr-x 3 root root 4096 Jul  8 05:40 ..
file-count=0
```

Captured maddy output (epoch-prefixed by the wrapper; note the near-instant delta `…245.369888100` → `…245.381770239`, ≈ **12 ms**):

```
1783489245.369888100 [debug] queue: delivery attempt #1	{"msg_id":"13a8ce17"}
1783489245.378809284 [debug] queue: failures: permanently: [refused@dest.local], temporary: [], errors: map[refused@dest.local:dial tcp 127.0.0.1:15599: connect: connection refused]	{"msg_id":"13a8ce17"}
1783489245.381770239 queue: delivery attempt failed	{"io_op":"dial","msg_id":"13a8ce17","rcpt":"refused@dest.local","reason":"dial tcp 127.0.0.1:15599: connect: connection refused","remote_addr":"127.0.0.1:15599","smtp_code":450,"smtp_enchcode":"4.4.2","smtp_msg":"Network I/O error","target":"smtp_downstream"}
1783489245.384828084 queue: not delivered, permanent error	{"msg_id":"13a8ce17","rcpt":"refused@dest.local"}
```

A refused connection returns immediately (the kernel replies with a TCP RST), so the attempt is near-instant — no timeout is involved. (This run also demonstrates the stage-dependent classification discussed below: through `smtp_downstream` a connection failure is permanent, hence `not delivered, permanent error`.)

**Black-hole (SYN silently dropped)** — two runs submitted simultaneously (message `a9717efa` on submission port `15605`, message `18a8dbdf` on `15615`; both `smtp_downstream` → `192.0.2.1:25`), reason `connection timed out`. Command (both instances launched detached with the epoch-timestamp wrapper; then one message submitted to each):

```
$ python3 /tmp/maddyrun/submit.py 127.0.0.1 15605 sender@src.local bh1@dest.local   # BH1
$ python3 /tmp/maddyrun/submit.py 127.0.0.1 15615 sender@src.local bh2@dest.local   # BH2
```

Captured output (complete, both runs):

```
# run 1 (BH1) — message a9717efa
1783489804.430267452 [debug] queue: delivery attempt #1	{"msg_id":"a9717efa"}
1783489937.684827785 queue: delivery attempt failed	{"io_op":"dial","msg_id":"a9717efa","rcpt":"bh1@dest.local","reason":"dial tcp 192.0.2.1:25: connect: connection timed out","remote_addr":"192.0.2.1:25","smtp_code":450,"smtp_enchcode":"4.4.2","smtp_msg":"Network I/O error","target":"smtp_downstream"}
# run 2 (BH2) — message 18a8dbdf
1783489804.430850132 [debug] queue: delivery attempt #1	{"msg_id":"18a8dbdf"}
1783489937.684858306 queue: delivery attempt failed	{"io_op":"dial","msg_id":"18a8dbdf","rcpt":"bh2@dest.local","reason":"dial tcp 192.0.2.1:25: connect: connection timed out","remote_addr":"192.0.2.1:25","smtp_code":450,"smtp_enchcode":"4.4.2","smtp_msg":"Network I/O error","target":"smtp_downstream"}
```

Durations: run 1 = 1783489937.684827785 − 1783489804.430267452 = **133.254560 s**; run 2 = 1783489937.684858306 − 1783489804.430850132 = **133.254008 s**; the spread is **0.000552 s** → **stable across 2 runs**. Mechanism (concrete, not environmental hand-waving): the host's SYN-retransmit budget was observed as

```
$ cat /proc/sys/net/ipv4/tcp_syn_retries
6
```

so for a SYN that is never answered the kernel sends the initial SYN plus 6 retransmits with exponentially increasing gaps (≈ 1 + 2 + 4 + 8 + 16 + 32 + 64 ≈ 127 s) plus a final RTO before returning `ETIMEDOUT` — ≈ 133 s here. Because maddy imposes no dial timeout, this kernel-governed value *is* the per-attempt duration. (This matches the ≈ 133 s figure referenced in the AAP, and is tied directly to `tcp_syn_retries = 6` on this host.)

**Accept-but-silent (no banner)** — the destination completes the TCP handshake but never sends the `220` greeting (message `3b3fae59`; submission port `15701`, silent downstream port `15700`). Command:

```
$ python3 /tmp/maddyrun/submit.py 127.0.0.1 15701 sender@src.local silent2@dest.local
```

After submit, exactly these two `-debug` lines appear (message accepted into the queue, attempt #1 begins) and then **nothing further**:

```
[debug] queue: delivery attempt #1	{"msg_id":"3b3fae59"}
[debug] queue: using message ID = 3b3fae59-1	{"msg_id":"3b3fae59"}
```

The fake server logged exactly one accepted connection (and received no SMTP commands); its complete log:

```
$ cat /tmp/maddyrun/logs/fakeSilentF.log
[fake_smtp mode=silent port=15700] mono=1315731.942816 wall=06:07:32.085714 LISTENING
[fake_smtp mode=silent port=15700] mono=1315735.005424 wall=06:07:35.148320 ACCEPTED connection #1 from ('127.0.0.1', 49888)
```

Confirming the hang was open-ended, after **522 s** (submit at wall `06:07:35`, re-checked at `06:16:17`) there was still **no** `delivery attempt failed` and **no** `will retry` line — the process (`maddy -config /tmp/maddyrun/maddySilentF.conf -debug`) was still blocked:

```
$ python3 -c "print('%.0f s' % ($(date +%s.%N) - 1783490855.081750123))"
522 s
$ grep -c -E 'delivery attempt failed|will retry|not delivered' /tmp/maddyrun/logs/maddySilentF.log
0
```

TCP connect succeeded, but maddy then blocks inside `smtp.NewClient` reading the `220` greeting that never arrives [smtpconn.go:L165]; with no read deadline set anywhere, the attempt hangs forever. Tellingly, the `[debug] smtp_downstream: connected` line that appears for a working downstream (seen in the Q1 trace, emitted at [smtp_downstream.go:L170]) is **absent** here — confirming the block is *before* client creation completes, i.e., in the greeting read.

### (c) The retry log entries (all fields)

A failed **temporary** attempt emits two default-level lines (they appear even without `-debug`). From the `451` reproduction — the **same instance-A run as Q1**, message `aff956e1`, produced by the submission shown in Q1 (`python3 /tmp/maddyrun/submit.py 127.0.0.1 15525 sender@src.local user@dest.local`), the downstream returning `451 4.3.0` at DATA:

```
queue: delivery attempt failed	{"msg_id":"aff956e1","rcpt":"user@dest.local","reason":"temporary failure, try again later","remote_server":"127.0.0.1","smtp_code":451,"smtp_enchcode":"4.3.0","smtp_msg":"temporary failure, try again later","target":"smtp_downstream"}
queue: will retry	{"attempts_count":1,"msg_id":"aff956e1","next_try_delay":"14m59.999999279s","rcpts":["user@dest.local"]}
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

- **`smtp_downstream` dials eagerly inside `Start()`** — `Start()` [smtp_downstream.go:L130] invokes `d.connect(ctx)` at [smtp_downstream.go:L139] (`connect()` is defined at [smtp_downstream.go:L150]). So a connection failure is a `Start()` failure → **permanent → not retried**. **Observed:** the connection-refused run (message `13a8ce17`, in (b) above) logged `queue: not delivered, permanent error` and left the queue directory **empty** (`file-count=0`, shown in (b)), *even though* the wrapped error carried an SMTP `smtp_code:450` — proving the outcome is decided by the **stage** (a `Start()` failure), not by the SMTP code. (Black-hole through `smtp_downstream` is likewise a `Start()` failure and therefore permanent.)
- **`remote` connects lazily inside `AddRcpt()`** — via `connectionForDomain()` [remote.go:L195,L230 → connect.go:L113]. This is proven by two runs against the `remote` target (submission port `15565`; both left the queue **empty** — `find /tmp/maddyrun/stateRem -name '*.meta' | wc -l` = `0`). Commands:

  ```
  $ python3 /tmp/maddyrun/submit.py 127.0.0.1 15565 sender@src.local test@localhost
  $ python3 /tmp/maddyrun/submit.py 127.0.0.1 15565 sender@src.local user@nxdomain-zzz-test.invalid
  ```

  **Observed** (message `387c62d4`, recipient `test@localhost`):

  ```
[debug] queue: target.Start OK	{"msg_id":"387c62d4"}
[debug] queue: delivery.AddRcpt test@localhost failed: dial tcp 127.0.0.1:25: connect: connection refused	{"msg_id":"387c62d4"}
[debug] queue: failures: permanently: [test@localhost], temporary: [], errors: map[test@localhost:dial tcp 127.0.0.1:25: connect: connection refused]	{"msg_id":"387c62d4"}
queue: delivery attempt failed	{"domain":"localhost","io_op":"dial","msg_id":"387c62d4","rcpt":"test@localhost","reason":"dial tcp 127.0.0.1:25: connect: connection refused","remote_addr":"127.0.0.1:25","smtp_code":550,"smtp_enchcode":"5.4.0","smtp_msg":"No usable MXs, last err: dial tcp 127.0.0.1:25: connect: connection refused","target":"remote"}
queue: not delivered, permanent error	{"msg_id":"387c62d4","rcpt":"test@localhost"}
  ```

  and (message `9b0323c2`, recipient `user@nxdomain-zzz-test.invalid`):

  ```
[debug] queue: delivery.AddRcpt user@nxdomain-zzz-test.invalid failed: dial tcp: lookup nxdomain-zzz-test.invalid on 34.118.224.10:53: no such host	{"msg_id":"9b0323c2"}
queue: delivery attempt failed	{"domain":"nxdomain-zzz-test.invalid","io_op":"dial","msg_id":"9b0323c2","rcpt":"user@nxdomain-zzz-test.invalid","reason":"dial tcp: lookup nxdomain-zzz-test.invalid on 34.118.224.10:53: no such host","remote_server":null,"smtp_code":550,"smtp_enchcode":"5.4.0","smtp_msg":"No usable MXs, last err: dial tcp: lookup nxdomain-zzz-test.invalid on 34.118.224.10:53: no such host","target":"remote"}
  ```

  Both traces prove `remote.Start()` returns OK **without** connecting (`queue: target.Start OK` precedes the failure), and that the resolve/connect work happens lazily in `AddRcpt` — the opposite of `smtp_downstream`. In **both** runs the connect loop exhausted all candidate MXs and `connectionForDomain()` wrapped the last error as an `SMTPError` with `Code: exterrors.SMTPCode(err, 451, 550)` and message `"No usable MXs, last err: …"` [connect.go:L202-L213] — observed as `smtp_code:550`, `smtp_enchcode:5.4.0`. Because `SMTPCode()` uses `IsTemporary()` (permanent-by-default) [exterrors/smtp.go:L112-L118] and neither `connection refused` nor `no such host` reports `Temporary() == true`, the code resolved to **`550`**; and `SMTPError.Temporary()` is `Code/100 == 4` [exterrors/smtp.go:L95-L97], so a `550` is **permanent** → `IsTemporaryOrUnspec()` returned `false` → **not retried** (queue emptied). So through `remote`, a *connection-refused* or *no-such-host* failure is also permanent — reached via `AddRcpt`/the `"No usable MXs"` wrapper rather than `Start()` — the observable difference between the targets being **where** the connection is attempted (`Start` vs `AddRcpt`), proven by the `target.Start OK` line preceding the `remote` failure.
  - **(inferred)** The distinct **`554` / `5.4.4` / `"MX lookup error"`** wrapper [connect.go:L250-L258] fires **only** when `rd.rt.resolver.LookupMX()` itself returns an error. In both runs above it did **not** fire: `localhost` has an `A` record, so `LookupMX` returned empty-without-error and the A/AAAA fallback [connect.go:L266-L271] supplied the host; and for `nxdomain-zzz-test.invalid` the failure surfaced later, during the dial's own address lookup (reason prefixed `dial tcp: lookup …: no such host`), again not from `LookupMX`. This one `LookupMX`-error path is therefore established by code reading rather than direct observation.

Because of this stage dependence, the reproductions drive the retry/queue-file path with a temporary **`451` DATA-stage rejection** (a `Body`/`Commit`-stage failure → temporary → retried).

### Attempt #2 physically observed after the full ~15-minute backoff (timing stable across 2 runs)

This wait is mandatory and cannot be simulated. Two independent instances were run with `max_tries 2` (instance A: message `aff956e1`; instance B: message `9b21a24e`); attempt #2 was witnessed in both. The fake server's per-connection timestamps give the true inter-attempt gap. Commands (the two fake-server logs; each line is the fake downstream's own `mono`/`wall` stamp per accepted connection):

```
$ grep ACCEPTED /tmp/maddyrun/logs/fakeA.log
$ grep ACCEPTED /tmp/maddyrun/logs/fakeB.log
```

```
# Instance A fake server (downstream port 15526)
[fake_smtp mode=451 port=15526] mono=1313871.426352 wall=05:36:31.569248 ACCEPTED connection #1 from ('127.0.0.1', 55136)
[fake_smtp mode=451 port=15526] mono=1314771.430281 wall=05:51:31.573176 ACCEPTED connection #2 from ('127.0.0.1', 50720)
# gap #1->#2 = 1314771.430281 - 1313871.426352 = 900.003929 s
# Instance B fake server (downstream port 15536)
[fake_smtp mode=451 port=15536] mono=1313936.337405 wall=05:37:36.480301 ACCEPTED connection #1 from ('127.0.0.1', 38818)
[fake_smtp mode=451 port=15536] mono=1314836.341808 wall=05:52:36.484704 ACCEPTED connection #2 from ('127.0.0.1', 50920)
# gap #1->#2 = 1314836.341808 - 1313936.337405 = 900.004403 s
```

And the `next_try_delay` doubling across the two attempts (instance A). Command:

```
$ grep -E 'delivery attempt #|will retry' /tmp/maddyrun/logs/maddyA.log
```

```
[debug] queue: delivery attempt #1	{"msg_id":"aff956e1"}
queue: will retry	{"attempts_count":1,"msg_id":"aff956e1","next_try_delay":"14m59.999999279s","rcpts":["user@dest.local"]}
[debug] queue: delivery attempt #2	{"msg_id":"aff956e1"}
queue: will retry	{"attempts_count":2,"msg_id":"aff956e1","next_try_delay":"29m59.999998996s","rcpts":["user@dest.local"]}
```

Instance B shows the same doubling for stability: `next_try_delay` `14m59.999999423s` → `29m59.999999194s`. Conclusion: the first backoff is **exactly 15 minutes** (gaps `900.003929 s` and `900.004403 s`; spread `0.000474 s`), and `next_try_delay` **doubles** (15 min → 30 min), matching `initialRetryTime = 15 min` and `retryTimeScale = 2` [queue.go:L185-L186,L414]. Attempts #1 and #2 were physically observed for both instances, and after attempt #2 each message remained on disk at `TriesCount:2` scheduled for attempt #3 (see Q4). The **N + 1** schedule then follows from the source: the terminal check `meta.TriesCount == q.maxTries` [queue.go:L390] is evaluated *before* the counter is incremented, so with `max_tries = 2` it fires on attempt #3 (`TriesCount == 2`), triggering `emitDSN()` [queue.go:L849] and removal from disk — i.e., attempts #1, #2, #3 then a bounce.

---

## Q3 — Where the Queue Stores Pending Messages on Disk

**Direct answer.** The queue stores pending messages in a directory computed as `filepath.Join(StateDirectory, queueName)` [queue.go:L229]. With the default `StateDirectory = "/var/lib/maddy"` [maddy.go:L59] and the shipped queue named `remote_queue` [maddy.conf:L122], the canonical location is **`/var/lib/maddy/remote_queue`**. An explicit `location` directive (or the queue block's first positional argument) overrides this.

**Evidence — default resolution proven at runtime.** With **no** `location` directive, `state /tmp/maddyrun/stateQ3`, and a queue named `remote_queue`, maddy auto-created the queue directory at `<state>/remote_queue` (a `451` message, `4701aa33`, was queued to populate it). Commands:

```
$ python3 /tmp/maddyrun/submit.py 127.0.0.1 15585 sender@src.local user@dest.local
$ find /tmp/maddyrun/stateQ3 -maxdepth 1 -type d
/tmp/maddyrun/stateQ3
/tmp/maddyrun/stateQ3/remote_queue
$ ls -la /tmp/maddyrun/stateQ3/remote_queue/
total 20
drwxr-xr-x 2 root root 4096 Jul  8 05:47 .
drwx------ 3 root root 4096 Jul  8 05:47 ..
-rw-r--r-- 1 root root   22 Jul  8 05:47 4701aa33.body
-rw-r--r-- 1 root root  337 Jul  8 05:47 4701aa33.header
-rw-r--r-- 1 root root  520 Jul  8 05:47 4701aa33.meta
```

The directory name is exactly the queue's name (`remote_queue`) joined onto the state directory, which is precisely `filepath.Join(config.StateDirectory, q.name)` [queue.go:L229]. Substituting the production default `StateDirectory = /var/lib/maddy` [maddy.go:L59] for the test `state` directory yields the canonical `/var/lib/maddy/remote_queue`. The state directory is created and validated at maddy startup, and because `maddyctl` has no queue subcommand, listing this directory is the canonical way to inspect pending messages. (The retry reproductions elsewhere in this document set an explicit `location` under `/tmp/maddyrun` — a non-canonical test path — but the default `filepath.Join` behavior is what is proven here.)

---

## Q4 — On-Disk State Between Retries

**Direct answer.** Between attempts, each queued message is exactly **three files** named by an 8-character lowercase hex ID: **`<id>.header`** (RFC 822 headers, including maddy's own `Received:`), **`<id>.body`** (the raw body), and **`<id>.meta`** (a compact single-line JSON metadata object). The `.meta` records the retry count in its **`TriesCount`** field (`0` while the first attempt is in flight, `1` after the first temporary failure, `2` after the second, and so on), along with the recipients still to retry (`To`), the per-recipient errors (`RcptErrs`), and the `FirstAttempt` / `LastAttempt` timestamps.

### (a) The files that exist

Listing captured immediately after the first temporary failure — the exact Q4 target state (message `aff956e1`, instance A). Command and complete listing:

```
$ ls -la /tmp/maddyrun/stateA/q_a/
total 20
drwxr-xr-x 2 root root 4096 Jul  8 05:36 .
drwxr-xr-x 3 root root 4096 Jul  8 05:36 ..
-rw-r--r-- 1 root root   22 Jul  8 05:36 aff956e1.body
-rw-r--r-- 1 root root  335 Jul  8 05:36 aff956e1.header
-rw-r--r-- 1 root root  516 Jul  8 05:36 aff956e1.meta
```

Three files per message, matching the source: the `.header` path is built at [queue.go:L607], `.body` at [queue.go:L611], and `.meta` at [queue.go:L615]. The `.header` and `.body` contents:

```
$ cat /tmp/maddyrun/stateA/q_a/aff956e1.header
Received: from client.local (localhost [127.0.0.1]) by test.local
 (envelope-sender <sender@src.local>) with ESMTP id aff956e1; Wed, 08 Jul
 2026 05:36:31 +0000
From: sender@src.local
To: user@dest.local
Subject: investigation test
Content-Type: text/plain; charset="utf-8"
Content-Transfer-Encoding: 7bit
MIME-Version: 1.0

$ cat /tmp/maddyrun/stateA/q_a/aff956e1.body
body sent at 05:36:31
```

Note (observed + `file:line`): the `.header` contains maddy's own `Received:` header embedding the message ID `aff956e1`, and it has **no `for` clause** — matching the documented quirk that "`for` field is never included in the `Received` header field" [docs/internals/quirks.md:L9].

### (b) Naming patterns

`<id>` is an **8-character lowercase hex** string — examples observed across the reproductions include `aff956e1`, `9b21a24e`, `3b3fae59`, and `4701aa33` — produced by `GenerateMsgID()` [msgid.go:L12] (4 random bytes hex-encoded). The metadata file is written atomically via a `.meta.new` temporary file plus rename; if a metadata file cannot be parsed, it is quarantined by renaming it to `<id>.meta_broken` [queue.go:L268].

### (c) Retry-count metadata — the full state transition (before / during / after)

The `.meta` is stored as compact single-line JSON. All three states were observed and are shown **complete** (each captured with `cat` on the live `.meta` file). Because a normally-failing message's `TriesCount:0` window lasts only milliseconds, the in-flight (`TriesCount:0`) snapshot is taken from a message held mid-attempt against the **silent** downstream (message `3b3fae59`), while the `TriesCount:1 → 2` progression is from the `451`-flow instance-A message (`aff956e1`); the queue writes an identical metadata shape in every case.

**DURING the first attempt** (in-flight; captured against the silent/hung downstream so the attempt does not complete), message `3b3fae59` — `TriesCount:0`, `RcptErrs:{}`, `FirstAttempt ≈ LastAttempt` (they differ by ~52 ns — set by two separate `time.Now()` calls at enqueue [queue.go:L594-L595]):

```
$ cat /tmp/maddyrun/stateSilentF/q_sf/3b3fae59.meta
{"MsgMeta":{"ID":"3b3fae59","OriginalFrom":"sender@src.local","DontTraceSender":false,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":197,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"sender@src.local","To":["silent2@dest.local"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{},"TriesCount":0,"FirstAttempt":"2026-07-08T06:07:35.122613573Z","LastAttempt":"2026-07-08T06:07:35.122613625Z"}
```

**AFTER the first temporary failure** (the Q4 target state), message `aff956e1` — `TriesCount:1`, `RcptErrs` populated, `LastAttempt` advanced, `FirstAttempt` fixed:

```
$ cat /tmp/maddyrun/stateA/q_a/aff956e1.meta
{"MsgMeta":{"ID":"aff956e1","OriginalFrom":"sender@src.local","DontTraceSender":false,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":194,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"sender@src.local","To":["user@dest.local"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"user@dest.local":{"Code":451,"EnhancedCode":[4,0,0],"Message":"temporary failure, try again later"}},"TriesCount":1,"FirstAttempt":"2026-07-08T05:36:31.524387989Z","LastAttempt":"2026-07-08T05:36:31.570973656Z"}
```

**AFTER the second attempt**, message `aff956e1` — `TriesCount:2`, `FirstAttempt` still fixed at the enqueue time, `LastAttempt` advanced to the second attempt's time (15 minutes later):

```
$ cat /tmp/maddyrun/stateA/q_a/aff956e1.meta
{"MsgMeta":{"ID":"aff956e1","OriginalFrom":"sender@src.local","DontTraceSender":false,"Quarantine":false,"OriginalRcpts":{},"SMTPOpts":{"Size":194,"RequireTLS":false,"UTF8":false},"Conn":null},"From":"sender@src.local","To":["user@dest.local"],"FailedRcpts":null,"TemporaryFailedRcpts":null,"RcptErrs":{"user@dest.local":{"Code":451,"EnhancedCode":[4,0,0],"Message":"temporary failure, try again later"}},"TriesCount":2,"FirstAttempt":"2026-07-08T05:36:31.524387989Z","LastAttempt":"2026-07-08T05:51:31.574759613Z"}
```

The transition is thus **`TriesCount` 0 → 1 → 2**; `FirstAttempt` is written once at enqueue (`2026-07-08T05:36:31.524387989Z`) and never changes, while `LastAttempt` advances on every attempt (`…05:36:31.570973656Z` after attempt #1 → `…05:51:31.574759613Z` after attempt #2, a 15-minute step); `RcptErrs` is populated from the first failure onward.

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

Demonstrated with `max_parallelism 1`, a silent (hung) downstream, and two messages to **different** destinations (message A = `d24f5376` → `msgA_starve@desta.local`; message B = `45bf4182` → `msgB_starve@destb.local`). Setup and submissions (submission port `15595`; the single delivery target for both is the silent downstream on `15596`, so the shared global slot is the only gate):

```
$ python3 /tmp/maddyrun/submit.py 127.0.0.1 15595 sender@src.local msgA_starve@desta.local
$ python3 /tmp/maddyrun/submit.py 127.0.0.1 15595 sender@src.local msgB_starve@destb.local
$ ls -1 /tmp/maddyrun/stateStv/q_stv/*.meta
/tmp/maddyrun/stateStv/q_stv/45bf4182.meta
/tmp/maddyrun/stateStv/q_stv/d24f5376.meta
```

Both messages were persisted, but the downstream accepted **exactly one** TCP connection (`grep ACCEPTED /tmp/maddyrun/logs/fakeStv.log` — the complete log):

```
[fake_smtp mode=silent port=15596] mono=1314543.460142 wall=05:47:43.603040 LISTENING
[fake_smtp mode=silent port=15596] mono=1314546.542646 wall=05:47:46.685542 ACCEPTED connection #1 from ('127.0.0.1', 50610)
```

The `-debug` log shows `d24f5376` acquiring the sole slot while `45bf4182` blocks with **no** `acquired` line (`grep -E 'starting delivery|semaphore' /tmp/maddyrun/logs/maddyStv.log`):

```
[debug] queue: starting delivery for d24f5376	
[debug] queue: waiting on delivery semaphore for d24f5376	
[debug] queue: delivery semaphore acquired for d24f5376	
[debug] queue: delivery attempt #1	{"msg_id":"d24f5376"}
[debug] queue: using message ID = d24f5376-1	{"msg_id":"d24f5376"}
[debug] queue: starting delivery for 45bf4182	
[debug] queue: waiting on delivery semaphore for 45bf4182	
```

Then the silent server was killed by its numeric PID to free the slot; message A failed (its hung greeting read returned `EOF`) and message B **immediately** acquired the freed slot and finally ran (failing `connection refused`, since the server was now gone):

```
$ kill "$STV_FAKE_PID"      # numeric PID of the silent fake server on :15596
1783489692.743667922 queue: delivery attempt failed	{"msg_id":"d24f5376","rcpt":"msgA_starve@desta.local","reason":"EOF","remote_server":"127.0.0.1","target":"smtp_downstream"}
1783489692.753503682 [debug] queue: delivery semaphore acquired for 45bf4182	
1783489692.756874313 [debug] queue: delivery attempt #1	{"msg_id":"45bf4182"}
1783489692.769295773 queue: delivery attempt failed	{"io_op":"dial","msg_id":"45bf4182","rcpt":"msgB_starve@destb.local","reason":"dial tcp 127.0.0.1:15596: connect: connection refused","remote_addr":"127.0.0.1:15596","smtp_code":450,"smtp_enchcode":"4.4.2","smtp_msg":"Network I/O error","target":"smtp_downstream"}
```

Explanation: message A held the sole slot for `1783489692.743667922 − 1783489666.720682002 =` **26.022986 s** (the entire time the silent server was up); message B's first delivery attempt did not run until A released the slot (B acquired it **≈ 9.8 ms** later — `1783489692.753503682 − 1783489692.743667922`). So B's *scheduled* time was independent of A (both were placed on the time wheel immediately), but B's *actual execution* was gated on A's slot. The log ordering — A acquired and failed **before** B acquired — confirms the gating. This ties directly to the single global `q.deliverySemaphore` [queue.go:L283] sized by `max_parallelism` and to the fairness-free wheel selection [timewheel.go:L80]; note the semaphore is destination-agnostic, so even though A and B target different recipients they share the one slot.

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
