# Maddy SMTP DATA Boundary-Termination Behavior Investigation

**Investigation of Maddy mail server at commit `26452dd8dd78`**

This document presents a comprehensive, code-grounded analysis of how the Maddy mail server handles SMTP DATA phase termination under various line-ending framings. The investigation covers six dimensions: boundary framing sensitivity, moment-of-decision observability, pipelining safety, back-to-back message consistency, proxy interaction, and residual effects. Every conclusion is traced directly to source code — no assumptions are made.

---

## Table of Contents

1. [DATA Boundary Framing Sensitivity](#1-data-boundary-framing-sensitivity)
2. [Moment-of-Decision Observability](#2-moment-of-decision-observability)
3. [Pipelining Under Pressure](#3-pipelining-under-pressure)
4. [Back-to-Back Message Consistency](#4-back-to-back-message-consistency)
5. [Proxy Interaction](#5-proxy-interaction)
6. [Residual Effects](#6-residual-effects)
7. [Summary and Conclusions](#7-summary-and-conclusions)
8. [Source Code References](#8-source-code-references)

---

## 1. DATA Boundary Framing Sensitivity

**Question:** How does Maddy react to payloads where the only variation is how the DATA termination sequence is framed?

### 1.1 The RFC Standard

Per RFC 5321 §4.1.1.4, the canonical DATA terminator is `<CRLF>.<CRLF>` — that is, the byte sequence `\r\n.\r\n`. RFC 5321 §2.3.8 further specifies that `<CRLF>` (`\r\n`) is the standard line terminator for SMTP. Any deviation from this sequence is, strictly speaking, non-compliant with the protocol specification.

### 1.2 Maddy's Delegation to Go's DotReader

Maddy itself performs **no** DATA boundary detection. The entire boundary decision is delegated through a chain of components that terminates in Go's standard library:

1. **go-smtp `conn.go` `handleData()`** (line 498): After sending the `354` response (`"Go ahead. End your data with <CR><LF>.<CR><LF>"`), creates a `dataReader` via `newDataReader(c)`.
2. **go-smtp `data.go` `newDataReader()`** (line 57): Wraps `c.text.DotReader()` — the Go standard library's `net/textproto.Reader.DotReader()` — with an optional size limit via `MaxMessageBytes`. **No additional boundary logic is added.**
3. **Go stdlib `net/textproto/reader.go` `DotReader()`** (line 300): Creates a `dotReader` struct — a state machine that is the **sole arbiter** of when the DATA phase ends.

The critical insight: **Maddy adds NO additional boundary validation on top of `DotReader`.** The `dataReader` in go-smtp's `data.go` only adds size limiting (checking `MaxMessageBytes`), not any boundary detection or validation logic. The go-smtp `handleData()` function passes this reader directly to `Session.Data(r)` in Maddy's `internal/endpoint/smtp/smtp.go` (line 312).

### 1.3 The DotReader Finite State Machine

The `dotReader` in Go's `net/textproto/reader.go` (lines 311–397) implements a six-state finite state machine (FSM). The states and transitions are:

```
States:
  stateBeginLine (0) — At the start of a line; initial state
  stateDot       (1) — Read '.' at the beginning of a line
  stateDotCR     (2) — Read '.\r' at the beginning of a line
  stateCR        (3) — Read '\r' (possibly at end of line)
  stateData      (4) — Reading data in the middle of a line
  stateEOF       (5) — Reached end-of-data marker; terminal state
```

**Transition table:**

| Current State    | Input Byte | Next State      | Action                                           |
|------------------|------------|-----------------|--------------------------------------------------|
| `stateBeginLine` | `.`        | `stateDot`      | Continue (do not emit the dot yet)               |
| `stateBeginLine` | `\r`       | `stateCR`       | Continue (do not emit `\r` yet)                  |
| `stateBeginLine` | other      | `stateData`     | Emit the byte                                    |
| `stateDot`       | `\r`       | `stateDotCR`    | Continue (dot-CR at line start)                  |
| `stateDot`       | **`\n`**   | **`stateEOF`**  | **Terminate — bare LF accepted!**                |
| `stateDot`       | other      | `stateData`     | Emit the byte (dot-unstuffing: leading dot consumed) |
| `stateDotCR`     | **`\n`**   | **`stateEOF`**  | **Terminate — canonical CRLF accepted**          |
| `stateDotCR`     | other      | `stateData`     | `UnreadByte`; emit saved `\r`                    |
| `stateCR`        | `\n`       | `stateBeginLine`| Emit `\n` (CRLF → newline normalization)         |
| `stateCR`        | other      | `stateData`     | `UnreadByte`; emit saved `\r`                    |
| `stateData`      | `\r`       | `stateCR`       | Continue (possible line ending)                  |
| `stateData`      | `\n`       | `stateBeginLine`| Emit `\n` (**bare LF starts new line!**)         |
| `stateData`      | other      | `stateData`     | Emit the byte                                    |
| `stateEOF`       | (any)      | `stateEOF`      | Return `io.EOF`                                  |

### 1.4 Termination Variant Analysis

The following table enumerates every relevant DATA termination framing and traces it through the DotReader FSM:

| Sequence | Description | FSM Path | Result |
|----------|-------------|----------|--------|
| `\r\n.\r\n` | Canonical (RFC 5321) | `stateBeginLine` →(`.`)→ `stateDot` →(`\r`)→ `stateDotCR` →(`\n`)→ **`stateEOF`** | **ACCEPTED** |
| `\n.\n` | Bare LF only | `stateBeginLine` →(`.`)→ `stateDot` →(`\n`)→ **`stateEOF`** | **ACCEPTED** |
| `\n.\r\n` | LF before dot, CRLF after | `stateBeginLine` →(`.`)→ `stateDot` →(`\r`)→ `stateDotCR` →(`\n`)→ **`stateEOF`** | **ACCEPTED** |
| `\r\n.\n` | CRLF before dot, LF after | `stateBeginLine` →(`.`)→ `stateDot` →(`\n`)→ **`stateEOF`** | **ACCEPTED** |
| `\r\n..\r\n` | Dot-stuffed line | `stateBeginLine` →(`.`)→ `stateDot` →(`.`)→ `stateData` (second dot emitted as body; first dot consumed as dot-unstuffing) | **NOT a terminator** — body line containing `.` |
| `\r\n. \r\n` | Dot followed by space | `stateBeginLine` →(`.`)→ `stateDot` →(` `)→ `stateData` (space emitted; leading dot consumed) | **NOT a terminator** — body line containing ` \r\n` (space + CRLF) |
| `abc.\r\n` | Dot not at column 0 | In `stateData` when `.` is encountered; dot is emitted as regular data | **NOT a terminator** — dot must be at `stateBeginLine` |

**Key finding:** All four combinations of `\n` and `\r\n` around the lone dot are accepted as valid terminators. This is because the DotReader treats bare `\n` as a valid line boundary (transitioning to `stateBeginLine` from `stateData`), and in `stateDot`, a bare `\n` directly reaches `stateEOF` without requiring `\r`.

### 1.5 Line Ending Normalization

The DotReader normalizes all line endings in its output. Specifically:
- `\r\n` is converted to `\n` (the `\r` is consumed in `stateCR`, and only `\n` is emitted when transitioning to `stateBeginLine`)
- Bare `\n` is emitted as-is

This means the body content passed to Maddy's `Session.prepareBody()` (`internal/endpoint/smtp/smtp.go`, line 283) always contains only bare `\n` line endings, regardless of what the client sent. The `bufio.Reader` wrapping at line 284, the header parsing via `textproto.ReadHeader(bufr)` at line 285, and the `buffer.BufferInMemory(bufr)` call at line 298 (`internal/buffer/memory.go`, line 27) all operate on this `\n`-normalized stream.

### 1.6 RFC Compliance Assessment

This behavior is **more lenient** than RFC 5321 §2.3.8, which specifies `<CRLF>` (`\r\n`) as the line terminator. The acceptance of bare `\n` as a line boundary — and consequently as part of the DATA terminator — means that non-compliant clients using bare-LF line endings will have their messages accepted by Maddy, even though a strictly RFC-compliant server should only recognize `\r\n.\r\n`.

The leniency is inherited entirely from the Go standard library's `net/textproto` package and is not a Maddy-specific decision.

---

## 2. Moment-of-Decision Observability

**Question:** What is visible in logs, transcripts, or protocol-level debug output at the precise instant the server concludes the DATA phase and transitions back to command parsing?

### 2.1 The `io_debug` Configuration Flag

The primary mechanism for wire-level observability is the `io_debug` configuration directive:

- **Definition:** `internal/endpoint/smtp/smtp.go` line 565:
  ```go
  cfg.Bool("io_debug", false, false, &ioDebug)
  ```
- **Activation:** `internal/endpoint/smtp/smtp.go` lines 603–606:
  ```go
  if ioDebug {
      endp.serv.Debug = endp.Log.DebugWriter()
      endp.Log.Println("I/O debugging is on! It may leak passwords in logs, be careful!")
  }
  ```
- **Effect on go-smtp:** When `Server.Debug` is set, go-smtp's `conn.go` `init()` method (lines 65–73) wraps the raw connection in `io.TeeReader` and `io.MultiWriter`:
  ```go
  if c.server.Debug != nil {
      rwc = struct {
          io.Reader
          io.Writer
          io.Closer
      }{
          io.TeeReader(rwc.Reader, c.server.Debug),
          io.MultiWriter(rwc.Writer, c.server.Debug),
          rwc.Closer,
      }
  }
  ```
  This mirrors **every byte** on the wire (both client-to-server and server-to-client) to the debug output. When enabled, the exact line endings and the exact dot terminator framing sent by the client are visible in the debug log.

**Important caveat:** The `Logger.DebugWriter()` method (`internal/log/log.go`, lines 172–178) returns `ioutil.Discard` if the Logger's `Debug` flag is false:
```go
func (l Logger) DebugWriter() io.Writer {
    if !l.Debug {
        return ioutil.Discard
    }
    l.Debug = true
    return &l
}
```
This means `io_debug` only produces output when the endpoint's `debug` flag is also enabled (line 566: `cfg.Bool("debug", true, false, &endp.Log.Debug)`). Note that the default config in `maddy.conf` does **not** include `io_debug`.

### 2.2 Structured Log Events During DATA Lifecycle

Maddy emits structured log events at key points in the DATA lifecycle. The log format, defined in `internal/log/log.go` (lines 60–76), is:
```
name: msg\t{"key":"value","key2":"value2"}
```

The complete sequence of log events for a successful DATA transaction:

| Event | Method | Location | Fields |
|-------|--------|----------|--------|
| `"incoming message"` | `Session.startDelivery()` | `smtp.go` lines 128–141 | `src_host`, `src_ip`, `sender`, `msg_id`, optionally `username` |
| `"RCPT ok"` | `Session.Rcpt()` | `smtp.go` line 243 | `rcpt`, `msg_id` |
| `"accepted"` | `Session.Data()` | `smtp.go` line 334 | `msg_id` |
| `"reset"` | `Session.Reset()` | `smtp.go` line 64 | (none — debug-only via `DebugMsg`) |

For failed DATA transactions:

| Event | Method | Location | Fields |
|-------|--------|----------|--------|
| `"DATA error"` | `Session.Data()` via `wrapErr` | `smtp.go` line 317 | `msg_id`, plus `reason` and structured error fields from `exterrors.Fields(err)` |
| `"aborted"` | `Session.abort()` | `smtp.go` line 72 | `msg_id` |

The `Logger.Error()` method (`internal/log/log.go`, lines 89–103) adds a `"reason"` field extracted from `err.Error()` and merges any structured error fields provided by `exterrors.Fields(err)`. These structured fields can include `smtp_code`, `smtp_enchcode`, and `smtp_msg` as defined in `internal/exterrors/`.

### 2.3 What Is NOT Visible

The DotReader FSM state transitions are **entirely internal** to the Go standard library and produce **no log output, no callbacks, and no observable side effects** beyond the eventual `io.EOF` return. There is:

- No log entry when the DotReader transitions to `stateEOF`
- No log entry distinguishing canonical `\r\n.\r\n` from non-canonical `\n.\n` termination
- No warning when bare-LF framing is used
- No metric or counter tracking non-canonical terminations

The **only** way to observe the exact boundary framing used by the client is via the `io_debug` raw wire capture. Without `io_debug`, the server's behavior is identical for all four accepted termination variants — the same `"accepted"` log entry with the same fields is emitted regardless of framing.

---

## 3. Pipelining Under Pressure

**Question:** Does ambiguous boundary framing within a pipelined session cause the server to misinterpret body data as post-DATA commands, or vice versa? Would a stricter peer disagree with Maddy's boundary decision?

### 3.1 The Command Loop and `closeDot()`

go-smtp's `handleConn()` loop in `server.go` (lines 138–170) follows this pattern:
```go
for {
    line, err := c.ReadLine()
    if err == nil {
        cmd, arg, err := parseCmd(line)
        // ...
        c.handle(cmd, arg)
    }
    // error handling...
}
```

The critical safety mechanism is in `textproto.Reader.ReadLine()`. Before reading the next line from the underlying buffered reader, `ReadLine()` calls `closeDot()` (`net/textproto/reader.go`, lines 405–413):
```go
func (r *Reader) closeDot() {
    if r.dot == nil {
        return
    }
    buf := make([]byte, 128)
    for r.dot != nil {
        // When Read reaches EOF or an error,
        // it will set r.dot == nil.
        r.dot.Read(buf)
    }
}
```

This drains the entire DotReader until it reaches `stateEOF` (at which point `r.dot` is set to `nil`). This means: **even if the session handler did not read the entire body, `closeDot()` will consume all remaining body data up to and including the terminator before any new command line is parsed.**

### 3.2 The Defensive Drain in `handleData()`

go-smtp's `handleData()` in `conn.go` (lines 524–525) includes an explicit drain after the session callback returns:
```go
r := newDataReader(c)
code, enhancedCode, msg := toSMTPStatus(c.Session().Data(r))
io.Copy(ioutil.Discard, r) // Make sure all the data has been consumed
c.WriteResponse(code, enhancedCode, msg)
```

This is a "belt-and-suspenders" approach: even if `Session.Data(r)` returns early without fully reading the body, `io.Copy(ioutil.Discard, r)` drains the remaining data. Combined with `closeDot()` in the next `ReadLine()`, this provides two layers of protection against body data leaking into command parsing.

### 3.3 Pipelining Safety Conclusion

**Pipelining is SAFE.** The `closeDot()` + drain pattern ensures that:

1. The DotReader consumes **exactly** the bytes up to and including the terminator (whichever variant was used).
2. Any bytes **after** the terminator remain in the underlying `bufio.Reader` buffer.
3. The next `ReadLine()` call reads from this buffer, correctly parsing the pipelined command.

Even if a client sends pipelined commands immediately after a non-canonical `\n.\n` terminator in the same TCP write, the DotReader will recognize `\n.\n` as the boundary and stop. The pipelined command bytes remain in the buffer for `ReadLine()` to process.

**Additional safety:** go-smtp's `parse.go` `parseCmd()` (line 9) strips trailing `\r\n` from command lines, and `smtp.go` `validateLine()` (line 23) rejects any command string containing `\n` or `\r`. This prevents any body data that accidentally leaked past the DotReader from being accepted as a valid command.

### 3.4 Stricter Peer Disagreement

If a stricter MTA only accepts `\r\n.\r\n` as the DATA terminator:

- It would **not** recognize `\n.\n` as a terminator and would continue reading body data.
- The two servers would **disagree** on where the body ends.
- However, when Maddy **forwards** messages via `internal/smtpconn/smtpconn.go` `Data()` (lines 303–323), it uses go-smtp's client `Data()` method which returns a `DotWriter`.
- The `DotWriter` (`net/textproto/writer.go`, lines 69–99) converts all `\n` to `\r\n` in its output and appends `.\r\n` on `Close()` (lines 103–118).
- Therefore, the downstream peer **always** sees canonical RFC-compliant `\r\n.\r\n` termination, regardless of what the original client sent.
- The boundary mismatch does **NOT propagate** downstream through Maddy when it acts as a relay.

---

## 4. Back-to-Back Message Consistency

**Question:** Is the boundary decision point deterministic across back-to-back messages over a single connection, or does timing, buffering, or state leakage cause inconsistency?

### 4.1 Complete State Cleanup Chain

After each message transaction, the following cleanup sequence executes:

1. **Maddy `Session.Data()` completes** (`internal/endpoint/smtp/smtp.go`, lines 337–341): Sets `s.delivery = nil`, `s.msgCtx = nil`, ends the trace task, and releases the semaphore.

2. **go-smtp `handleData()` drains residual body** (`conn.go`, line 524): `io.Copy(ioutil.Discard, r)` ensures all body data is consumed through the DotReader.

3. **go-smtp `handleData()` sends the response** (`conn.go`, line 525): Sends `250 OK` or the appropriate error response.

4. **go-smtp `handleData()` calls `c.reset()`** (`conn.go`, line 516, deferred at entry): The `reset()` method (lines 694–703) acquires a lock, calls `c.session.Reset()` (which triggers Maddy's `Session.Reset()` — aborting any active delivery and logging `"reset"`), and clears `c.fromReceived = false` and `c.recipients = nil`.

5. **Next `ReadLine()` invokes `closeDot()`**: When the `handleConn()` loop calls `c.ReadLine()` for the next command, `closeDot()` runs as a final safety measure, draining any remaining DotReader data (though the drain in step 2 should have already consumed it).

### 4.2 Fresh DotReader Per Transaction

A **new** `dotReader` instance is created for each DATA phase via `c.text.DotReader()` in `newDataReader()` (`go-smtp/data.go`, line 59). The `DotReader()` method (`net/textproto/reader.go`, line 300) calls `closeDot()` on any previous reader and creates a fresh `dotReader` with `state: 0` (`stateBeginLine`). There is **no state carryover** from a previous DotReader instance.

### 4.3 Test Evidence

The Maddy test suite includes explicit verification of back-to-back message handling:

- **`TestSMTPDelivery_Multi`** (`internal/endpoint/smtp/smtp_test.go`, lines 322–358): Sends two complete messages over a single connection using `submitMsg()`. The first message is from `sender1@example.org` to `rcpt1@example.com` and `rcpt2@example.com`; the second from `sender2@example.org` to `rcpt3@example.com` and `rcpt4@example.com`. The test verifies that both messages are delivered with correct metadata (sender, recipients, Received header with correct msg_id).

- **`TestSMTPDelivery_Reset`** (`internal/endpoint/smtp/smtp_test.go`, lines 429–462): Sends a MAIL FROM + RCPT TO, then issues RSET, then sends a complete message. Verifies that the first (incomplete) transaction has no effect on the second.

### 4.4 Determinism Conclusion

The boundary decision is **fully deterministic** and **timing-independent**:

- The DotReader FSM is purely synchronous — it processes bytes one at a time as they are read from the buffered reader, with no timeouts, no buffering heuristics, and no ambient state.
- The `bufio.Reader` buffer persists across transactions (it is the same TCP connection), but this is by design — it simply means bytes that have arrived from the network are available for the next read operation. This does not introduce non-determinism.
- Each DATA phase gets a fresh `dotReader` instance with `state = stateBeginLine`.
- The cleanup chain (drain → response → reset → closeDot) ensures complete isolation between transactions.

There is **no state leakage**. The same input bytes will always produce the same boundary decision, regardless of timing, load, or the number of preceding transactions on the same connection.

---

## 5. Proxy Interaction

**Question:** How would a TCP proxy that normalizes or blocks ambiguous line-ending framing interact with Maddy?

### 5.1 Scenario A: Normalizing Proxy (Converts All Line Endings to `\r\n`)

If a TCP proxy sits between the client and Maddy and converts all bare `\n` to `\r\n`:

- The proxy would convert `\n.\n` to `\r\n.\r\n` before the bytes reach Maddy.
- Maddy's DotReader would see only canonical framing and terminate the DATA phase via the `stateDot` → `stateDotCR` → `stateEOF` path.
- The message body would contain `\n`-normalized content (since DotReader normalizes `\r\n` to `\n` in output anyway).
- **Result:** No errors, no observable difference in Maddy's behavior compared to the non-proxied case. The delivered message body is identical because DotReader normalizes line endings regardless of input framing.

### 5.2 Scenario B: Strict Proxy (Blocks Ambiguous Framing)

If a TCP proxy enforces strict RFC 5321 compliance and rejects any DATA phase that uses bare `\n` in the terminator:

- The proxy would detect `\n.\n` as a non-compliant terminator and either:
  - Close the connection (Maddy sees a truncated TCP stream)
  - Return an error response to the upstream client (Maddy never sees the message)
- If the proxy closes the connection mid-DATA, the DotReader would return `io.ErrUnexpectedEOF` (because EOF arrives before the terminator). This error propagates through `prepareBody()` → `Data()`, triggering a `"DATA error"` log entry and delivery abort.
- If the proxy rejects before DATA, Maddy never receives the message.
- **Result:** The client receives an error. No message is delivered through Maddy.

### 5.3 Outbound Re-Encoding Path

When Maddy acts as a relay and forwards messages to downstream servers, the outbound path through `internal/smtpconn/smtpconn.go` `Data()` (lines 303–323) uses go-smtp's client:

```go
func (c *C) Data(ctx context.Context, hdr textproto.Header, body io.Reader) error {
    wc, err := c.cl.Data()  // Returns a DotWriter
    // ...
    if err := textproto.WriteHeader(wc, hdr); err != nil { ... }
    if _, err := io.Copy(wc, body); err != nil { ... }
    if err := wc.Close(); err != nil { ... }
    return nil
}
```

The `DotWriter` (`net/textproto/writer.go`, lines 69–118):
- Converts all `\n` to `\r\n` during `Write()` (line 87: if `c == '\n'`, writes `\r` before `\n`)
- Escapes leading dots by doubling them (line 82)
- On `Close()`, ensures the final line ends with `\r\n` and appends `.\r\n` (lines 103–118)

**Therefore:** Downstream peers **always** receive canonical RFC-compliant framing (`\r\n.\r\n`), regardless of what the original client sent to Maddy. The "leniency mismatch" does **not** propagate through Maddy when it acts as a relay.

### 5.4 Body Content Caveat

While the boundary framing is normalized on outbound, the **body content** may differ subtly: if the original body contained intentional bare-LF sequences (e.g., `\n` within a line that the sender intended to preserve as-is), the DotReader treats these as line boundaries and normalizes them to `\n`. When the DotWriter re-encodes for downstream delivery, these become `\r\n`. A receiving MTA that preserves `\r\n` would see different content than the original sender intended — but this is an inherent consequence of the bare-LF leniency, not a Maddy-specific issue.

---

## 6. Residual Effects

**Question:** What observable artifacts persist after a problematic transaction — partial messages, connection state pollution, log entries indicating silent acceptance or rejection — and what expected evidence is absent?

### 6.1 Artifacts Present After Accepted Transactions

When the DATA terminator is accepted (any of the four variants: `\r\n.\r\n`, `\n.\n`, `\n.\r\n`, `\r\n.\n`):

| Artifact | Description | Source |
|----------|-------------|--------|
| `"accepted"` log entry | Emitted with `msg_id` field | `smtp.go` line 334 |
| Message delivered to target | Body stored via `buffer.BufferInMemory()` with `\n`-only line endings | `smtp.go` lines 321–332 |
| `"reset"` debug log | Emitted when go-smtp calls `Session.Reset()` after transaction | `smtp.go` line 64 |
| `io_debug` wire capture | Raw bytes showing exact client framing (if `io_debug` is enabled) | `smtp.go` lines 603–606, `conn.go` lines 65–73 |

### 6.2 Artifacts Present After Failed Transactions

When the DATA phase fails (header parse error, body too large, connection error):

| Artifact | Description | Source |
|----------|-------------|--------|
| `"DATA error"` log entry | Emitted with `msg_id` and structured error fields including `reason`, `smtp_code`, `smtp_enchcode` | `smtp.go` line 317, `log.go` lines 89–103 |
| `"aborted"` log entry | Emitted when delivery is aborted | `smtp.go` line 72 |
| No message delivered | Delivery pipeline is atomic — `delivery.Body()` and `delivery.Commit()` must both succeed | `smtp.go` lines 326–332 |

### 6.3 Artifacts That Are ABSENT

The following expected evidence is **not present**, which is a significant observation:

- **No log entry distinguishing canonical vs. non-canonical boundary framing.** The DotReader treats all four termination variants identically. There is no warning, notice, or debug message when bare-LF framing is used. The `"accepted"` log entry is identical regardless of framing.

- **No `docs/internals/quirks.md` entry for bare-LF acceptance.** The quirks file (lines 1–31) documents only two categories:
  - SMTP: `for` field omission in `Received` header (line 9)
  - IMAP: `\Recent` flag always set, sequence number inconsistency
  
  The bare-LF DATA terminator acceptance is **not documented** as a known quirk, despite being a deviation from RFC 5321 §2.3.8.

- **No metric or counter** tracking non-canonical termination events.

- **No configuration option** to enforce strict `\r\n`-only line endings.

### 6.4 Connection State After Problematic Transactions

**Successful delivery (any framing):**
The connection is clean and ready for the next transaction. The `reset()` call (`go-smtp/conn.go`, lines 694–703) clears all per-message state (`fromReceived = false`, `recipients = nil`, and calls `session.Reset()`). The DotReader has been fully drained via `io.Copy(ioutil.Discard, r)`. The `handleConn()` loop resumes reading the next command.

**Failed delivery (header parse error, body too large, delivery rejection):**
The delivery is aborted via `Session.abort()` (`smtp.go`, lines 67–81), which releases the semaphore, calls `delivery.Abort()`, and clears per-message state. The DotReader is **still** drained by `io.Copy(ioutil.Discard, r)` in `handleData()` (`conn.go`, line 524). The connection remains usable for subsequent transactions.

**Abrupt client disconnect during DATA:**
- Maddy's `Session.Logout()` (`smtp.go`, lines 269–281) aborts any active delivery.
- go-smtp detects the closed connection when the next `ReadLine()` returns `io.EOF` and terminates the `handleConn()` goroutine.
- **No partial messages are delivered.** The test `TestSMTPDelivery_AbortData` (`internal/endpoint/smtp/smtp_test.go`, lines 360–396) explicitly verifies this: it sends a message body without the terminating dot, then closes the connection. After a 250ms wait, it confirms zero messages were delivered.

### 6.5 Atomicity Guarantee

No partial messages are ever delivered. The delivery pipeline is atomic:
1. `Session.prepareBody()` (`smtp.go`, lines 283–310) reads the entire body into a `MemoryBuffer` via `buffer.BufferInMemory(bufr)`.
2. `delivery.Body()` receives the complete headers and buffered body.
3. `delivery.Commit()` finalizes the delivery.
4. Both `Body()` and `Commit()` must succeed for the message to be delivered.

If the body is truncated (e.g., connection drops mid-DATA), the DotReader returns `io.ErrUnexpectedEOF`, which propagates through `ioutil.ReadAll()` in `BufferInMemory()` (`internal/buffer/memory.go`, line 28), through `prepareBody()`, and back to `Data()`, which invokes `wrapErr()` to emit the `"DATA error"` log and returns the error to go-smtp.

---

## 7. Summary and Conclusions

### 7.1 Termination Variant Summary Table

| Sequence | RFC Compliant? | Accepted by Maddy? | FSM Terminal Path |
|----------|---------------|--------------------|--------------------|
| `\r\n.\r\n` | ✅ Yes | ✅ Yes | `stateDot` → `stateDotCR` → `stateEOF` |
| `\n.\n` | ❌ No | ✅ Yes | `stateDot` → `stateEOF` |
| `\n.\r\n` | ❌ No (bare LF before dot) | ✅ Yes | `stateDot` → `stateDotCR` → `stateEOF` |
| `\r\n.\n` | ❌ No (bare LF after dot) | ✅ Yes | `stateDot` → `stateEOF` |
| `\r\n..\r\n` | N/A (dot-stuffed) | ❌ Not a terminator | `stateDot` → `stateData` (unstuffed) |
| `\r\n. \r\n` | N/A (space after dot) | ❌ Not a terminator | `stateDot` → `stateData` |
| Mid-line `.` | N/A (not at line start) | ❌ Not a terminator | Dot in `stateData`, no transition |

### 7.2 Key Findings

1. **Maddy accepts bare-LF DATA terminators.** Through Go's `net/textproto.DotReader`, all four combinations of `\n` and `\r\n` around a lone dot at the beginning of a line are recognized as valid DATA terminators. This makes Maddy more lenient than RFC 5321 §2.3.8 requires.

2. **The leniency is inherited from the Go standard library,** not from Maddy-specific code. Maddy and go-smtp add no additional boundary detection logic. The `dataReader` in `go-smtp/data.go` only adds size limiting.

3. **The leniency is silent.** No log entries, warnings, metrics, or configuration options distinguish canonical from non-canonical termination. The behavior is undocumented in `docs/internals/quirks.md`.

4. **Pipelining is safe.** The `closeDot()` mechanism in `textproto.Reader.ReadLine()` and the `io.Copy(ioutil.Discard, r)` drain in `handleData()` provide two layers of protection against body data leaking into command parsing.

5. **Back-to-back messages are fully isolated.** Fresh `dotReader` instances, explicit state cleanup (`reset()`), and full body drainage ensure deterministic, timing-independent behavior across transactions.

6. **Downstream forwarding normalizes all framing.** The `DotWriter` used in `internal/smtpconn/smtpconn.go` always produces `\r\n` line endings and `.\r\n` termination, preventing the leniency from propagating to downstream MTAs.

7. **No partial messages are ever delivered.** The delivery pipeline is atomic — the entire body is buffered in memory before `delivery.Body()` and `delivery.Commit()` are called. Truncated connections result in error propagation and delivery abort.

8. **This behavior is consistent with the Postel robustness principle** ("be conservative in what you send, be liberal in what you accept") but could cause interoperability issues with stricter MTAs that only recognize `\r\n.\r\n`. Such a disagreement would not propagate downstream through Maddy because the outbound `DotWriter` re-encodes to canonical framing.

### 7.3 The `lineLimitReader` and Bare LF

It is worth noting that the `lineLimitReader` in `go-smtp/lengthlimit_reader.go` (lines 22–47) also treats bare `\n` as a line boundary. Its `Read()` method resets `curLineLength` on `\n` (line 39), not on `\r\n` specifically. This means line-length enforcement is consistent with the DotReader's bare-LF acceptance — a line terminated by bare `\n` correctly resets the line-length counter.

---

## 8. Source Code References

### Maddy Core

| File | Key Lines | Purpose |
|------|-----------|---------|
| `internal/endpoint/smtp/smtp.go` | 35–58 | `Session` struct definition |
| | 60–65 | `Session.Reset()` — logs "reset", aborts delivery if active |
| | 67–81 | `Session.abort()` — releases semaphore, aborts delivery, logs "aborted" |
| | 128–141 | `Session.startDelivery()` — logs "incoming message" |
| | 243 | `Session.Rcpt()` — logs "RCPT ok" |
| | 269–281 | `Session.Logout()` — aborts active delivery on disconnect |
| | 283–310 | `Session.prepareBody()` — bufio wrap, header parse, BufferInMemory |
| | 312–344 | `Session.Data()` — prepareBody → delivery.Body → Commit, logs "accepted" |
| | 317 | `wrapErr` closure — logs "DATA error" |
| | 551–609 | `Endpoint.setConfig()` — `io_debug` at line 565, activation at lines 603–606 |
| `internal/smtpconn/smtpconn.go` | 303–323 | `C.Data()` — outbound DotWriter encoding |
| `internal/buffer/memory.go` | 27–33 | `BufferInMemory()` — `ioutil.ReadAll` into `MemoryBuffer` |
| `internal/log/log.go` | 26–34 | `Logger` struct |
| | 60–76 | `Logger.Msg()` — structured log format |
| | 89–103 | `Logger.Error()` — adds "reason" field |
| | 106–113 | `Logger.DebugMsg()` — debug-gated log |
| | 172–178 | `Logger.DebugWriter()` — returns writer or `ioutil.Discard` |

### Maddy Tests

| File | Key Lines | Purpose |
|------|-----------|---------|
| `internal/endpoint/smtp/smtp_test.go` | 25–28 | `testMsg` constant with CRLF endings |
| | 90–120 | `submitMsg()` / `submitMsgOpts()` helpers |
| | 322–358 | `TestSMTPDelivery_Multi` — back-to-back messages |
| | 360–396 | `TestSMTPDelivery_AbortData` — no delivery on abrupt disconnect |
| | 429–462 | `TestSMTPDelivery_Reset` — state isolation after RSET |
| `internal/testutils/smtp_server.go` | 18–26 | `SMTPMessage` struct — captures Data as `[]byte` |
| | 116–131 | `session.Data()` — `ioutil.ReadAll`, stores bytes |

### go-smtp v0.12.1

| File | Key Lines | Purpose |
|------|-----------|---------|
| `conn.go` | 49–75 | `init()` — `lineLimitReader` + debug `TeeReader`/`MultiWriter` |
| | 498–525 | `handleData()` — 354 response, `newDataReader`, `Session.Data(r)`, drain, response, reset |
| | 694–703 | `reset()` — clears `fromReceived`, `recipients`, calls `session.Reset()` |
| `data.go` | 48–80 | `dataReader` — wraps `DotReader()` with `MaxMessageBytes` limit |
| `server.go` | 82–86 | `NewServer()` — advertises `PIPELINING`, `8BITMIME`, `ENHANCEDSTATUSCODES` |
| | 124–170 | `handleConn()` — command loop: `ReadLine()` → `parseCmd()` → `handle()` |
| `parse.go` | 8–37 | `parseCmd()` — strips trailing `\r\n`, splits command from arguments |
| `lengthlimit_reader.go` | 22–47 | `lineLimitReader` — resets byte counter on `\n` (bare LF) |
| `smtp.go` | 23–27 | `validateLine()` — rejects strings containing `\n` or `\r` |

### Go Standard Library (Go 1.13.15)

| File | Key Lines | Purpose |
|------|-----------|---------|
| `net/textproto/reader.go` | 290–306 | `DotReader()` — creates new `dotReader`, calls `closeDot()` on previous |
| | 311–397 | `dotReader.Read()` — six-state FSM for DATA boundary detection |
| | 405–413 | `closeDot()` — drains active DotReader to EOF |
| `net/textproto/writer.go` | 37–48 | `DotWriter()` — creates new `dotWriter` |
| | 69–99 | `dotWriter.Write()` — converts `\n` to `\r\n`, escapes leading dots |
| | 103–118 | `dotWriter.Close()` — ensures `\r\n` and appends `.\r\n` |

### Documentation

| File | Key Lines | Purpose |
|------|-----------|---------|
| `docs/internals/quirks.md` | 7–11 | SMTP quirks — only `Received` `for` field omission; **no DATA boundary leniency documented** |
| `maddy.conf` | 53–91 | Default SMTP endpoint config on port 25; **no `io_debug` directive** |
| `HACKING.md` | 1–50 | Design goals and developer guide |
