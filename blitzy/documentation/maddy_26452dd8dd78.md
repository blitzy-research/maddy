# Maddy Queue Delivery Timing Analysis

## Introduction

This document provides an authoritative, calculation-backed analysis of the maddy mail server's delivery queue timing and throughput characteristics. It is written in response to end-user complaints about long email bounce times and aims to explain — with full source code citations — exactly how the queue processes messages, how retry delays accumulate, and how long users should expect to wait before receiving a bounce notification (DSN — Delivery Status Notification) when delivery permanently fails.

**All values in this document are extracted directly from the source code on branch `maddy_26452dd8dd78`.** No values are estimated or assumed. Every configuration constant, formula, and behavioral claim cites the exact source file and line number where it is defined.

**Scope:** This analysis covers the queue module at `internal/target/queue/` and its interaction with the production configuration at `maddy.conf`. It does not cover the remote delivery target internals, authentication, storage, or IMAP modules.

---

## 1. Queue Configuration Defaults

The queue's behavior is governed by a small set of configuration parameters. The following table lists each parameter, its value, and the exact source code location where it is defined.

| Parameter | Value | Source |
|-----------|-------|--------|
| `initialRetryTime` | `15 * time.Minute` (15 minutes) | `internal/target/queue/queue.go:185` — set in `NewQueue()` |
| `retryTimeScale` | `2` | `internal/target/queue/queue.go:186` — set in `NewQueue()` |
| `postInitDelay` | `10 * time.Second` (10 seconds) | `internal/target/queue/queue.go:187` — set in `NewQueue()` |
| `maxTries` | `8` (default via `cfg.Int`) | `internal/target/queue/queue.go:204` — set in `Init()` |
| `maxParallelism` | `16` (default via `cfg.Int`) | `internal/target/queue/queue.go:205` — set in `Init()` |
| `deliverySemaphore` | `chan struct{}` buffered to `maxParallelism` | `internal/target/queue/queue.go:242` — created in `start()` |

### Production Configuration Confirmation

The production configuration file `maddy.conf` confirms these defaults are used in the deployed system:

- `max_tries 8` — Source: `maddy.conf:125`
- `max_parallelism 16` — Source: `maddy.conf:128`

### Documentation Discrepancy

> **Warning:** The man page `docs/man/maddy-targets.5.scd` at line 65 documents the default value of `max_tries` as `4`. However, the **actual** code default is `8` (at `queue.go:204`) and the production configuration also uses `8` (at `maddy.conf:125`). The example configuration block in the man page at line 23 also shows `max_tries 4`. This is a documentation-vs-code inconsistency — the code is the source of truth, and the correct default is **8**.

---

## 2. Parallel Processing Model

This section explains how the queue processes multiple messages concurrently. The key concept is a **channel-based semaphore** — a Go buffered channel used to limit how many delivery attempts can run at the same time.

### 2.1 The Queue Struct

The `Queue` struct (Source: `internal/target/queue/queue.go:112-147`) contains the concurrency primitive:

```go
// Buffered channel used to restrict count of deliveries attempted
// in parallel.
deliverySemaphore chan struct{}
```

Source: `queue.go:144-146`

The `deliverySemaphore` field is a buffered channel of empty structs. Its buffer size equals `maxParallelism` (default: 16). A goroutine "acquires" a slot by sending a value into the channel; if the channel buffer is full (16 values already in it), the send blocks until another goroutine "releases" a slot by receiving from the channel.

### 2.2 Semaphore Initialization

In the `start()` function, the semaphore is created with the configured parallelism limit:

```go
q.deliverySemaphore = make(chan struct{}, maxParallelism)
```

Source: `queue.go:242`

With the default `maxParallelism=16`, this creates a channel with buffer size 16, allowing up to 16 concurrent delivery attempts.

### 2.3 The dispatch() Function

The `dispatch()` function (Source: `queue.go:275-323`) is the callback passed to the `TimeWheel` scheduler (explained below). When the TimeWheel determines it is time to attempt delivery of a message, it calls `dispatch()`. Here is what happens:

1. **Line 280:** `q.deliveryWg.Add(1)` — Adds to a `sync.WaitGroup` that tracks in-flight deliveries (used during graceful shutdown).
2. **Line 281:** `go func() { ... }()` — Launches a new goroutine for this delivery attempt.
3. **Line 283:** `q.deliverySemaphore <- struct{}{}` — The goroutine attempts to send into the semaphore channel. If 16 goroutines are already holding slots (channel buffer full), this **blocks** until one finishes.
4. **Line 285:** `<-q.deliverySemaphore` (deferred) — When the delivery completes (or panics), the slot is released by receiving from the channel.
5. **Line 321:** `q.tryDelivery(meta, hdr, body)` — The actual delivery attempt is performed.

### 2.4 The TimeWheel Scheduler

The `TimeWheel` (Source: `internal/target/queue/timewheel.go:15-25`) is a concurrent scheduler that manages pending deliveries ordered by their target delivery time. It uses a linked list of `TimeSlot` entries, each containing a target time and associated data.

**Key operations:**

- **`Add(target time.Time, value interface{})`** (Source: `timewheel.go:38-53`): Adds a new slot to the linked list and notifies the background goroutine via `updateNotify` channel.
- **`tick()`** (Source: `timewheel.go:71-128`): The background goroutine that runs continuously:
  - Scans the linked list for the slot with the earliest deadline (lines 78-84)
  - Creates a `time.NewTimer` for that deadline (line 99)
  - When the timer fires, removes the slot and calls `tw.dispatch(closestSlot)` (line 109)
  - If a new slot arrives with an earlier deadline (via `updateNotify`), the timer is stopped and the loop restarts to pick the new earliest slot (lines 112-121)

### 2.5 End-to-End Flow

The complete flow from message submission to delivery attempt is:

1. `Commit()` is called on a new message delivery
2. `TimeWheel.Add(time.Time{}, slot)` schedules the message for immediate dispatch (Source: `queue.go:578`). The zero-value `time.Time{}` represents the Go time epoch (year 0001), which is always in the distant past, so the TimeWheel's timer fires immediately.
3. The `tick()` goroutine in TimeWheel fires immediately (the target time is the zero-value epoch, which is always in the past)
4. `dispatch()` is called, which launches a new goroutine
5. The goroutine blocks on the semaphore channel if ≥16 deliveries are already in progress
6. Once a semaphore slot is acquired, `tryDelivery()` executes the actual delivery

**Key insight:** The model is **not** a fixed thread pool. Every message gets its own goroutine, but the channel-based semaphore limits how many can execute `tryDelivery()` concurrently. Goroutines beyond the limit block on the channel send at line 283, waiting for a slot to open. This means the system can have hundreds of goroutines waiting, but only 16 are actively delivering at any time.

---

## 3. Initial Delivery Pass Throughput Calculation

This section calculates the wall-clock time to complete the first delivery attempt for a batch of 500 messages, assuming each delivery takes approximately 2 seconds to a single destination server.

### 3.1 Given Parameters

| Parameter | Value | Source |
|-----------|-------|--------|
| Number of messages | 500 | User scenario |
| Time per delivery attempt | ~2 seconds | User scenario |
| `max_parallelism` | 16 | `queue.go:205` |

### 3.2 Calculation

The semaphore allows 16 concurrent delivery attempts. In the simplest model, we can think of the deliveries proceeding in "batches" of 16:

1. **Concurrent slots:** 16 messages can be delivered simultaneously
2. **Number of batches:** `⌈500 / 16⌉ = ⌈31.25⌉ = 32` batches
3. **Last batch size:** `500 - (31 × 16) = 500 - 496 = 4` messages (only 4 slots used)
4. **Wall-clock time:** `32 batches × 2 seconds/batch = 64 seconds`

### 3.3 Result

| Metric | Value |
|--------|-------|
| Wall-clock time (ceiling estimate) | **~64 seconds** |
| Effective throughput | **~7.8 messages/second** |

### 3.4 Important Caveat

The 64-second figure is a **ceiling estimate**. In practice, the behavior is closer to a pipeline: as soon as one delivery finishes and its goroutine releases the semaphore slot (Source: `queue.go:285`), the next waiting goroutine immediately proceeds. This means there is no synchronization barrier between "batches" — the semaphore operates as a sliding window, not a batch gate. The actual throughput may be slightly better than 64 seconds due to this pipelining effect, but 64 seconds is a safe upper bound.

---

## 4. Retry Delay Accumulation

This is the most critical section for understanding bounce timing. It explains the retry formula, traces the delivery attempt lifecycle, and computes the complete schedule from first attempt through final bounce.

### 4.1 The Retry Formula

The delay before the next retry attempt is calculated using exponential backoff:

```
delay = initialRetryTime × retryTimeScale ^ (TriesCount - 1)
```

Source (comment): `internal/target/queue/queue.go:121-122`
Source (implementation): `internal/target/queue/queue.go:414`

The actual Go code at line 414:

```go
nextTryTime = nextTryTime.Add(q.initialRetryTime * time.Duration(math.Pow(q.retryTimeScale, float64(meta.TriesCount-1))))
```

With the default values:
- `initialRetryTime` = 15 minutes (Source: `queue.go:185`)
- `retryTimeScale` = 2 (Source: `queue.go:186`)

This produces classic exponential backoff: 15 min, 30 min, 1 hour, 2 hours, 4 hours, 8 hours, 16 hours, 32 hours.

### 4.2 The Delivery Attempt Lifecycle (Off-by-One Analysis)

To understand how many attempts actually occur, we must trace the code path in `tryDelivery()` (Source: `queue.go:365-429`):

1. A new message starts with `TriesCount = 0` — this is the default zero value for the `int` field in the `QueueMetadata` struct (Source: `queue.go:166`).

2. **Termination check at line 390:**
   ```go
   if meta.TriesCount == q.maxTries || len(partialErr.TemporaryFailed) == 0 {
   ```
   This check happens **before** the increment. With `maxTries=8`, it compares the current `TriesCount` against 8.

3. **Increment at line 407:**
   ```go
   meta.TriesCount++
   ```
   The increment happens **after** the termination check, only if delivery will be retried.

4. **What this means:**
   - Entry with TriesCount=0: check `0 == 8` → false → increment to 1 → schedule retry
   - Entry with TriesCount=1: check `1 == 8` → false → increment to 2 → schedule retry
   - Entry with TriesCount=2: check `2 == 8` → false → increment to 3 → schedule retry
   - ...
   - Entry with TriesCount=7: check `7 == 8` → false → increment to 8 → schedule retry
   - Entry with TriesCount=8: check `8 == 8` → **true** → generate bounce (DSN)

**Result: `max_tries=8` produces 9 total delivery attempts (one initial attempt + 8 retries), not 8 as the configuration name might suggest.**

### 4.3 Complete Retry Schedule

The following table shows every delivery attempt, the state of `TriesCount` at each stage, and the computed retry delay. The delay formula uses the **post-increment** value of `TriesCount` (because the increment at line 407 happens before the delay calculation at line 414).

| Attempt | TriesCount at Entry | Check `TriesCount == 8`? | After Increment | Retry Delay Formula | Delay | Cumulative Wait |
|---------|--------------------|--------------------------|-----------------|--------------------|-------|-----------------|
| 1 (initial) | 0 | 0 ≠ 8 → continue | 1 | 15min × 2^(1−1) = 15 × 1 | 15 min | 15 min |
| 2 | 1 | 1 ≠ 8 → continue | 2 | 15min × 2^(2−1) = 15 × 2 | 30 min | 45 min |
| 3 | 2 | 2 ≠ 8 → continue | 3 | 15min × 2^(3−1) = 15 × 4 | 60 min (1h) | 1h 45min |
| 4 | 3 | 3 ≠ 8 → continue | 4 | 15min × 2^(4−1) = 15 × 8 | 120 min (2h) | 3h 45min |
| 5 | 4 | 4 ≠ 8 → continue | 5 | 15min × 2^(5−1) = 15 × 16 | 240 min (4h) | 7h 45min |
| 6 | 5 | 5 ≠ 8 → continue | 6 | 15min × 2^(6−1) = 15 × 32 | 480 min (8h) | 15h 45min |
| 7 | 6 | 6 ≠ 8 → continue | 7 | 15min × 2^(7−1) = 15 × 64 | 960 min (16h) | 31h 45min |
| 8 | 7 | 7 ≠ 8 → continue | 8 | 15min × 2^(8−1) = 15 × 128 | 1920 min (32h) | 63h 45min |
| 9 (final) | 8 | 8 == 8 → **BOUNCE** | — | — | — | **~63h 45min** |

### 4.4 Total Wait Time

The sum of all retry delays:

```
15 + 30 + 60 + 120 + 240 + 480 + 960 + 1920 = 3825 minutes
```

Converting:
- **3825 minutes = 63 hours 45 minutes = 63.75 hours ≈ 2 days 15 hours 45 minutes**

**The bounce notification (DSN) is generated approximately 63 hours and 45 minutes after the first delivery attempt**, assuming every single retry attempt fails with a temporary error. This is the worst-case scenario.

The actual delivery attempt durations themselves (a few seconds each) are negligible compared to the multi-hour retry delays and are not included in this total.

### 4.5 Why the Last Retry Delay Dominates

Note that the final retry delay (attempt 8 → attempt 9) alone is **1920 minutes (32 hours)**. This single delay accounts for more than half of the total 3825-minute wait time. The exponential nature of the backoff means the later retries are vastly longer:

| Retry Delays (sorted) | Cumulative % of Total |
|------------------------|-----------------------|
| First 4 delays (15+30+60+120 = 225 min) | 5.9% |
| Next 2 delays (240+480 = 720 min) | 24.7% |
| Last 2 delays (960+1920 = 2880 min) | 100% |

---

## 5. TriesCount=0 Edge Case Analysis

This section analyzes a mathematical edge case in the retry delay formula when `TriesCount` is zero.

### 5.1 The Code Path

When the maddy server restarts, the function `readDiskQueue()` (Source: `queue.go:622-688`) loads previously queued messages from disk and reschedules them. The retry delay for each loaded message is computed at lines 669-670:

```go
nextTryTime := meta.LastAttempt
nextTryTime = nextTryTime.Add(q.initialRetryTime * time.Duration(math.Pow(q.retryTimeScale, float64(meta.TriesCount-1))))
```

Source: `queue.go:669-670`

### 5.2 When Does TriesCount=0 Occur?

A message that was committed to disk but **never had its first delivery attempt** (e.g., the server crashed between `Commit()` and the first `dispatch()`) will have `TriesCount=0` in its persisted metadata. This is because `TriesCount` is an `int` field in the `QueueMetadata` struct (Source: `queue.go:166`), and Go initializes `int` fields to zero by default.

### 5.3 Mathematical Analysis

When `TriesCount=0`, the formula evaluates to:

```
delay = initialRetryTime × retryTimeScale ^ (TriesCount - 1)
      = 15min × 2 ^ (0 - 1)
      = 15min × 2 ^ (-1)
      = 15min × 0.5
      = 7.5 minutes
```

**This is NOT a crash.** Go's `math.Pow(2, -1)` correctly returns `0.5` per IEEE 754 floating-point arithmetic. The result is a valid, positive delay of 7.5 minutes — which is **shorter** than the expected first-retry delay of 15 minutes.

### 5.4 The postInitDelay Safety Mechanism

Immediately after the delay calculation, a safety check applies (Source: `queue.go:672-674`):

```go
if time.Until(nextTryTime) < q.postInitDelay {
    nextTryTime = time.Now().Add(q.postInitDelay)
}
```

- `postInitDelay` = 10 seconds (Source: `queue.go:187`)
- If the computed `nextTryTime` is less than 10 seconds in the future — for example, if the server was down long enough that `LastAttempt + 7.5min` is already in the past — then the message is rescheduled for `now + 10 seconds`.
- This safety mechanism prevents all disk-loaded messages from firing their delivery attempts simultaneously on startup, which could overwhelm the downstream mail server.

### 5.5 Practical Impact

The 7.5-minute delay is a **mathematical quirk**, not a bug. The practical behavior depends on how long the server was offline:

| Server Downtime | Computed nextTryTime | postInitDelay Applies? | Actual Behavior |
|----------------|---------------------|------------------------|-----------------|
| < 7.5 minutes | `LastAttempt + 7.5min` (in the future) | No | Message retries at `LastAttempt + 7.5min` — slightly ahead of a normal 15-min first retry |
| ≥ 7.5 minutes | `LastAttempt + 7.5min` (in the past) | Yes | Message retries at `now + 10 seconds` — the postInitDelay floor kicks in |

**Key takeaways:**
- No crash occurs — `math.Pow` handles negative exponents correctly
- No infinite loop or negative delay is produced
- The postInitDelay safety net ensures messages are not all dispatched at time zero
- The 7.5-minute delay for TriesCount=0 messages is shorter than the normal 15-minute first retry, but this only affects messages that were queued but never attempted before a server restart — a relatively rare scenario

---

## 6. Test Suite Verification

To confirm the behavioral analysis above, the existing queue test suite was executed.

### 6.1 Test Command and Result

```
go test ./internal/target/queue/... -v -count=1 -timeout 120s
```

**Result: All tests PASS in 1.507s**

```
ok  github.com/foxcpp/maddy/internal/target/queue    1.507s
```

### 6.2 Test Results Detail

| Test Name | Status | What It Validates |
|-----------|--------|-------------------|
| `TestQueueDelivery` | PASS | Basic successful delivery and disk cleanup |
| `TestQueueDelivery_PermanentFail_NonPartial` | PASS | Permanent failure → no retry, immediate bounce |
| `TestQueueDelivery_PermanentFail_Partial` | PASS | Partial permanent failure via `PartialDelivery` interface |
| `TestQueueDelivery_TemporaryFail` | PASS | Temporary failure → automatic retry succeeds on next attempt |
| `TestQueueDelivery_TemporaryFail_Partial` | PASS | Partial temporary failure → selective retry for failed recipients only |
| `TestQueueDelivery_MultipleAttempts` | PASS | Multi-attempt delivery with mixed permanent/temporary failures |
| `TestQueueDelivery_PermanentRcptReject` | PASS | Permanent recipient rejection at `AddRcpt` stage |
| `TestQueueDelivery_TemporaryRcptReject` | PASS | Temporary recipient rejection → retry for rejected recipient |
| `TestQueueDelivery_SerializationRoundtrip` | PASS | Disk persistence and restart recovery (queue restart loads from disk) |
| `TestQueueDelivery_DeserlizationCleanUp/NoMeta` | SKIP | Not implemented (skipped via `t.Skip` at `queue_test.go:547`) |
| `TestQueueDelivery_DeserlizationCleanUp/NoBody` | PASS | Cleanup when body file is missing from disk |
| `TestQueueDelivery_DeserlizationCleanUp/NoHeader` | PASS | Cleanup when header file is missing from disk |
| `TestQueueDelivery_AbortIfNoRecipients` | PASS | Abort delivery when all recipients are rejected |
| `TestQueueDelivery_AbortNoDangling` | PASS | No dangling files left on disk after abort |
| `TestQueueDSN` | PASS | DSN (bounce) message generation for failed deliveries |
| `TestQueueDSN_FromEmptyAddr` | PASS | No DSN generated for null-sender (bounce) messages |
| `TestQueueDSN_NoDSNforDSN` | PASS | No infinite bounce loops (DSN of a DSN is suppressed) |
| `TestQueueDSN_RcptRewrite` | PASS | DSN uses original recipient addresses, not rewritten ones |
| `TestTimeWheelAdd` | PASS | Basic TimeWheel slot addition and dispatch |
| `TestTimeWheelAdd_Ordering` | PASS | TimeWheel dispatches in correct chronological order |
| `TestTimeWheelAdd_Restart` | PASS | TimeWheel handles slot updates and timer restarts |
| `TestTimeWheelAdd_MissingGotoBug` | PASS | Regression test for a previous scheduling bug |
| `TestTimeWheelAdd_EmptyUpdWait` | PASS | TimeWheel correctly waits when queue is empty |

### 6.3 Test Configuration Note

The test helper `newTestQueue()` (Source: `queue_test.go:29-68`) overrides the production defaults for fast test execution:

| Parameter | Test Value | Production Value |
|-----------|------------|------------------|
| `initialRetryTime` | `0` (line 50) | `15 * time.Minute` |
| `retryTimeScale` | `1` (line 51) | `2` |
| `postInitDelay` | `0` (line 52) | `10 * time.Second` |
| `maxTries` | `5` (line 53) | `8` |
| `maxParallelism` | `1` (line 63, via `start(1)`) | `16` |

This means the tests validate the retry **mechanism** (temporary failures are retried, permanent failures generate bounces, disk persistence works) but do not test the production timing values. The timing analysis in this document is derived from the code's default values, not from test execution timing.

---

## 7. Summary and Key Takeaways

The following table summarizes the actionable timing numbers derived from this analysis:

| Question | Answer |
|----------|--------|
| How does the queue process messages in parallel? | Channel-based semaphore (`chan struct{}` with buffer size 16) limits concurrent `tryDelivery()` calls. Each message gets a goroutine, but only 16 execute simultaneously. Source: `queue.go:242,283` |
| How long does the first delivery pass take for 500 messages at 2s/attempt? | **~64 seconds** (`⌈500/16⌉ × 2s = 32 × 2s`). Source: `queue.go:205` for max_parallelism=16 |
| How many delivery attempts does `max_tries=8` produce? | **9 total attempts** (initial + 8 retries). The termination check at line 390 fires when TriesCount reaches 8, but TriesCount starts at 0 and is incremented after the check (line 407). |
| How long from first attempt to bounce if every retry fails? | **~63 hours 45 minutes (~2 days 15 hours 45 minutes)**. Sum of delays: 15+30+60+120+240+480+960+1920 = 3825 minutes. |
| What is the single largest retry delay? | **32 hours** (the 8th retry delay: 15min × 2^7 = 1920 min). This alone is more than half the total wait. |
| What happens when TriesCount=0 in the retry formula? | `15min × 2^(-1) = 7.5 minutes`. Valid float64 math, no crash. Occurs in `readDiskQueue()` for messages that were never attempted before a server restart. Source: `queue.go:670` |
| Does the man page match the code? | **No.** Man page shows `max_tries` default as `4` (`maddy-targets.5.scd:65`), but code default is `8` (`queue.go:204`) and production config uses `8` (`maddy.conf:125`). |

### For End-Users

If you sent a message and the destination server is temporarily refusing delivery, **your bounce notification will arrive approximately 2 days, 15 hours, and 45 minutes (63 hours 45 minutes) after the first delivery attempt** in the worst case. This is because the queue makes 9 delivery attempts with exponentially increasing delays (15 min, 30 min, 1 hour, 2 hours, 4 hours, 8 hours, 16 hours, 32 hours) before giving up and generating a bounce.

The majority of this wait time is concentrated in the final retries — the last retry delay alone is 32 hours. If delivery is going to succeed, it most likely will during the earlier attempts (within the first few hours).
