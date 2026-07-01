# maddy SMTP-Security Investigation — branch `maddy_26452dd8dd78`

This document empirically answers two SMTP-security questions about the **maddy** mail server (`github.com/foxcpp/maddy`, commit `26452dd8dd787dc455278b0fdd296f4a5432c768`):

1. **DATA boundary / "lone dot":** what happens when a message body contains normal content, then a line with only a dot (`.`), then *more* data before the final terminator.
2. **Authentication identity confusion across `RSET`:** what happens when a client authenticates as user A, issues `MAIL FROM`, `RSET`s, then issues `MAIL FROM` as a *different* user B without re-authenticating.

Per the governing rules, the answers were **derived by running the code first, then writing** — the server was actually **built with Go 1.13.15 and `CGO_ENABLED=1`** and **run**, and the two scenarios were driven against it over raw SMTP sockets. The captured output is quoted **verbatim** below, and every system claim is grounded in an exact `file:line` reference. The behaviors reported are what the server *actually did*, not what it "would" do.

> Reproducibility note: if the run is regenerated, the `msg_id`s, timestamps, and DKIM key bytes will differ, but the **response codes, log field names, header shapes, `Conn:null`, and signed/not-signed outcomes are stable** and match the source verbatim.

---

## How this was observed

All temporary assets lived **outside the source tree** and were removed afterward; the repository was left byte-for-byte unchanged.

- Built `maddy` + `maddyctl` with **Go 1.13.15** and `CGO_ENABLED=1` (gcc) for the SQLite driver `github.com/mattn/go-sqlite3` v1.11.0 [`go.mod:L26`]; the module targets `go 1.13` [`go.mod:L3`].
- **Throwaway config** (outside the repo): `hostname mx.test.local`, `tls off`, an `sql` module for both auth and storage (SQLite); an **unauthenticated** `smtp tcp://127.0.0.1:2525` endpoint (no checks); an **authenticated** `submission tcp://127.0.0.1:5587` endpoint with `insecure_auth` [`internal/endpoint/smtp/smtp.go:L564`] + `io_debug` [`internal/endpoint/smtp/smtp.go:L565`] and `modify { sign_dkim test.local default }`; delivery to a `queue` whose downstream is a blackhole SMTP listener returning a **temporary `451`** so the `.meta`/`.header`/`.body` files persist for inspection.
- Two users seeded via `maddyctl users create`: `usera@test.local` and `userb@test.local`.
- Raw-socket SMTP client scripts drove the exact byte sequences shown in each question.

**Topology note (587 vs 465/25 — the role is the invariant).** The prompt names port **587** for submission, but maddy's bundled `maddy.conf` binds the `submission` listener to **465** [`maddy.conf:L93`] and the `smtp` listener to **25** [`maddy.conf:L53`]. This is not a conflict: the behavior under investigation derives from the **endpoint role** (unauthenticated `smtp` versus authenticated `submission`) implemented by the endpoint module, **not** from the numeric port. The investigation therefore used a throwaway config binding `smtp` on `127.0.0.1:2525` and `submission` on `127.0.0.1:5587`; the endpoint-module behavior is identical regardless of the port number.

**Why a `451` blackhole (so queue artifacts persist).** A downstream that merely *refuses the connection* is classified by the queue as a **permanent** error → `queue: not delivered, permanent error` → `q.removeFromDisk(...)` [`internal/target/queue/queue.go:L403`], which **deletes** the on-disk files. A downstream returning a real SMTP **`451`** is **temporary** — `exterrors.IsTemporaryOrUnspec(err)` is true [`internal/target/queue/queue.go:L96`] → `queue: will retry` → the three files (`.meta`/`.header`/`.body`) **persist**. That is why a `451` blackhole was used as the queue downstream, making the queue an ideal sink for inspecting both "what was stored" (R1c) and "which identity was recorded" (R2b).

---

## Question 1 — DATA boundary / "lone dot"

> **How observed (Q1):** the payload below was sent after the `354` prompt to the **unauthenticated** `smtp` endpoint on `127.0.0.1:2525`.

**Command / payload** (the exact bytes written after `DATA`):

```
b'From: attacker@external.example\r\nSubject: Q1 boundary test\r\n\r\nnormal content line one\r\n.\r\nMORE DATA AFTER DOT\r\n.\r\n'
```

**Client-side SMTP responses (verbatim):**

```
354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
250 2.0.0 OK: queued
500 5.5.2 Syntax error, MORE command unrecognized
501 5.5.2 Bad command
```

**Server log (verbatim; `msg_id` from this run = `87b7d2f9`):**

```
smtp: incoming message  {"msg_id":"87b7d2f9","sender":"attacker@external.example","src_host":"client.test","src_ip":"127.0.0.1:43360"}
smtp: accepted          {"msg_id":"87b7d2f9"}
smtp: 250 2.0.0 OK: queued
[debug] smtp: reset
smtp: 500 5.5.2 Syntax error, MORE command unrecognized
smtp: 501 5.5.2 Bad command
```

**What was stored** (persisted queue files; downstream forced to a temporary `451` so `.meta`/`.header`/`.body` survive):

```
87b7d2f9.body (24 bytes):  normal content line one          <-- "MORE DATA AFTER DOT" is ABSENT
87b7d2f9.header:
  Received: from client.test (localhost [127.0.0.1]) by mx.test.local
   (envelope-sender <attacker@external.example>) with ESMTP id 87b7d2f9; Wed, 01 Jul 2026 03:10:04 +0000
  From: attacker@external.example
  Subject: Q1 boundary test
87b7d2f9.meta (JSON):  "OriginalFrom":"attacker@external.example","From":"attacker@external.example","To":["victim@remote.example"],"Conn":null
```

### R1a — Does maddy stop at the first dot, keep consuming, or end in an unexpected state?

**maddy STOPS at the first lone dot.** This is fail-safe and **RFC 5321-compliant**, not an unexpected state.

**Mechanism.** maddy's `Session.Data(r io.Reader)` [`internal/endpoint/smtp/smtp.go:L312`] reads the header and body from the reader supplied by the library **to EOF only** — maddy contains **no lone-dot logic of its own**. Inside `prepareBody` [`internal/endpoint/smtp/smtp.go:L283`], the header is parsed with `textproto.ReadHeader` [`internal/endpoint/smtp/smtp.go:L285`] and the body is buffered with `buffer.BufferInMemory` [`internal/endpoint/smtp/smtp.go:L298`], both of which simply consume `r` until it signals EOF. The reader is a `net/textproto.DotReader` created by the `github.com/emersion/go-smtp` dependency's `newDataReader` (`r: c.text.DotReader()`) [`github.com/emersion/go-smtp/data.go:L51-L62`], and it is the `DotReader` that **signals EOF at the first line containing only `.`** and performs dot-unstuffing. The dependency is pinned at `github.com/emersion/go-smtp v0.12.1-0.20191206174923-1f576e0ec85c` [`go.mod:L19`], so this end-of-data behavior is **owned by the library**, not by maddy-authored code.

**Why this is spec-compliant and fail-safe.** Under **RFC 5321** end-of-mail-data transparency, a line containing only `.` terminates the message text, and a client body line that legitimately begins with `.` must be **dot-stuffed** (sent with an extra leading dot, which the server strips). The payload here injected an **un-stuffed** lone dot mid-body, so treating it as the end of data — and stopping there — is exactly what the specification requires. The bytes after the terminator are therefore correctly *no longer message body*; they are re-interpreted as new protocol input (see R1b).

### R1b — What runtime signs reveal which path was taken?

Two distinct, observable signs:

1. **The pre-dot message is accepted**, returning `250 2.0.0 OK: queued`.
2. **The trailing bytes are re-parsed as SMTP commands**, producing `500 5.5.2 Syntax error, MORE command unrecognized` (from the line `MORE DATA AFTER DOT`, whose first token `MORE` is treated as a command verb) and then `501 5.5.2 Bad command` (from the second, final `.`).

**Mechanism.** After `Session.Data` returns, the dependency's `handleData` drains the reader with `io.Copy(ioutil.Discard, r)` [`github.com/emersion/go-smtp/conn.go:L521`] and its `defer c.reset()` [`github.com/emersion/go-smtp/conn.go:L512`] returns the connection to command mode [`github.com/emersion/go-smtp/conn.go:L498-L523`]. The command loop then emits `500 {5,5,2} "Syntax error, %v command unrecognized"` for the unknown verb [`github.com/emersion/go-smtp/conn.go:L78`] and `501 {5,5,2} "Bad command"` when the line fails to parse [`github.com/emersion/go-smtp/server.go:L145`].

**The server log corroborates the exact ordering:** `incoming message` → `accepted` → `[debug] smtp: reset` → the two command errors. The `[debug] smtp: reset` line is maddy's own `s.endp.Log.DebugMsg("reset")` [`internal/endpoint/smtp/smtp.go:L64`] emitted from `Session.Reset()` [`internal/endpoint/smtp/smtp.go:L60`] — its position **between** the `accepted` message and the two `500`/`501` errors is the durable runtime marker that the transaction closed at the dot and the connection reverted to command mode before the trailing lines arrived.

### R1c — What actually ends up stored/queued?

**Only the pre-dot content is stored/queued.** The stored `.body` is **24 bytes** containing exactly `normal content line one`; **`MORE DATA AFTER DOT` is absent** from it. The trailing data never entered the message at all — it was consumed as (failed) commands.

The stored `.header` carries the injected `Received:` header, which maddy builds via `target.GenerateReceived(ctx, s.msgMeta, s.endp.hostname, s.msgMeta.OriginalFrom)` [`internal/endpoint/smtp/smtp.go:L303`]. That header embeds the **envelope sender** — `(envelope-sender <attacker@external.example>)` — because `GenerateReceived` writes `" (envelope-sender <"` + the `mailFrom` argument + `">)"` [`internal/target/received.go:L67-L72`]. The `.meta` records `"OriginalFrom":"attacker@external.example","From":"attacker@external.example","To":["victim@remote.example"],"Conn":null`.


---

## Question 2 — Authentication identity confusion across `RSET`

> **How observed (Q2):** driven against the **authenticated** `submission` endpoint on `127.0.0.1:5587`. Scenario A is the identity-confusion sequence `AUTH PLAIN (usera) → MAIL FROM:<usera@test.local> → RSET → MAIL FROM:<userb@test.local> → RCPT TO:<remote@elsewhere.example> → DATA (From: userb@test.local) → .`. An aligned-identity **control** (`From == auth == usera`) was run to isolate the DKIM decision.

**Scenario A (identity confusion) — client responses (verbatim):**

```
235 2.0.0 Authentication succeeded
250 2.0.0 Roger, accepting mail from <usera@test.local>
250 2.0.0 Session reset
250 2.0.0 Roger, accepting mail from <userb@test.local>
250 2.0.0 OK: queued
```

**Server log (verbatim; `msg_id`s from this run):**

```
submission: incoming message  {"msg_id":"f5b1ee2d","sender":"usera@test.local",...,"username":"usera@test.local"}
submission: 250 2.0.0 Session reset
submission: incoming message  {"msg_id":"0f6ff775","sender":"userb@test.local","src_ip":"127.0.0.1:42936","username":"usera@test.local"}
sign_dkim: not signing, From address is not authenticated identity  {"auth_id":"usera@test.local","from_addr":"userb@test.local","msg_id":"0f6ff775"}
submission: accepted  {"msg_id":"0f6ff775"}
submission: 250 2.0.0 OK: queued
```

**Persisted queue for the confused message `0f6ff775` (`From` userb, authed as usera):**

```
0f6ff775.meta (JSON):  "OriginalFrom":"userb@test.local","From":"userb@test.local","To":["remote@elsewhere.example"],"Conn":null
0f6ff775.header:
  Received:  by mx.test.local (envelope-sender <userb@test.local>) with ESMTP id 0f6ff775; ...
  From: userb@test.local
  (NO DKIM-Signature header present)
```

**Aligned-identity control `52075f04` (`From` usera == auth usera) — opposite DKIM outcome:**

```
52075f04.header:
  Dkim-Signature: a=rsa-sha256; bh=...; c=relaxed/relaxed; d=test.local; h=...; i=usera@test.local; s=default; t=...; v=1; x=...; b=...;
  Received:  by mx.test.local (envelope-sender <usera@test.local>) with ESMTP id 52075f04; ...
```

### R2a — Does maddy reject the command, tie it back to the original identity, or allow it to proceed?

**maddy ALLOWS the second `MAIL FROM` as a different user** — there is no rejection and no rebinding to the original authenticated identity. The server answered the post-`RSET` `MAIL FROM:<userb@test.local>` with `250 2.0.0 Roger, accepting mail from <userb@test.local>` and ultimately `250 2.0.0 OK: queued`, even though the client authenticated as `usera` and never re-authenticated.

**Root-cause chain (all in `internal/endpoint/smtp/smtp.go`):**

- `Session.connState` is **connection-scoped**. `Reset()` [`internal/endpoint/smtp/smtp.go:L60`] calls `abort()` [`internal/endpoint/smtp/smtp.go:L67-L81`], which clears **only message-scoped fields** — `mailFrom`, `opts`, `msgMeta`, `delivery`, `deliveryErr`, `msgCtx` — and **never touches `connState.AuthUser`**. The authentication therefore survives `RSET`.
- `Mail()` [`internal/endpoint/smtp/smtp.go:L162-L179`] performs **no `From == AuthUser` check**. It calls `startDelivery` and then stores `s.mailFrom = from` **unconditionally**, so any sender is accepted at this layer.
- Routing is keyed on the **envelope** `MAIL FROM`, not on the authenticated user: delivery calls `dd.srcBlockForAddr(mailFrom)` [`internal/msgpipeline/msgpipeline.go:L113`], defined at [`internal/msgpipeline/msgpipeline.go:L155`], which matches `perSource[cleanFrom]` [`internal/msgpipeline/msgpipeline.go:L171`] → `perSource[domain]` [`internal/msgpipeline/msgpipeline.go:L190`] → `defaultSource` [`internal/msgpipeline/msgpipeline.go:L193`]. The authenticated identity plays no part in source selection, so a different post-`RSET` sender is accepted subject only to domain-based source blocks.

### R2b — What evidence shows which identity was trusted for headers, queue metadata, and enforcement checks?

The authenticated identity (`usera`) **persists at connection scope** but surfaces only **transiently**; **every persisted artifact reflects the envelope sender (`userb`)**. Taking each sink the question names:

- **Logs (transient).** `startDelivery` logs both `sender` (the envelope `from`) and `username` (`s.connState.AuthUser`) [`internal/endpoint/smtp/smtp.go:L127-L142`, specifically `"sender", from` at `L131` and `"username", s.connState.AuthUser` at `L133`]. The captured log shows `"sender":"userb@test.local"` alongside `"username":"usera@test.local"` — the mismatch between envelope and authenticated identity is visible **only here**, in the live log.
- **Enforcement checks.** The authenticated identity is held in memory via `msgMeta.Conn = &s.connState` [`internal/endpoint/smtp/smtp.go:L86`] (`ConnState.AuthUser` is defined at [`internal/module/msgmetadata.go:L37`]) and is consulted by exactly **one** acceptance-relevant place: the DKIM modifier's `shouldSign` [`internal/modify/dkim/dkim.go:L249`]. With the default `require_sender_match {envelope, auth}` [`internal/modify/dkim/dkim.go:L151-L152`], the `auth` branch compares the header `From` to the authenticated identity and, on mismatch, logs `not signing, From address is not authenticated identity` with fields `from_addr`, `auth_id`, `msg_id` then `return "", false` [`internal/modify/dkim/dkim.go:L299-L310`]. Crucially, this **withholds the signature; it does NOT reject the message** — the log shows `sign_dkim: not signing, ...` immediately followed by `submission: accepted` and `250 2.0.0 OK: queued`.
- **Headers.** The injected `Received:` embeds the **envelope sender** — `(envelope-sender <userb@test.local>)` — not the authenticated user [`internal/target/received.go:L67-L72`]. The confused message `0f6ff775` has **no `DKIM-Signature`** header, whereas the aligned control `52075f04` **is** signed with `i=usera@test.local` (the DKIM identity is bound to the authenticated user). This contrast is the durable, on-disk proof of which identity was trusted for signing.
- **Queue metadata.** The `.meta` records `"OriginalFrom":"userb@test.local","From":"userb@test.local"` and `"Conn":null`. The connection state — and therefore the authenticated identity — is **stripped before serialization**: `updateMetadataOnDisk` [`internal/target/queue/queue.go:L742`] sets `metaCopy.MsgMeta.Conn = nil` [`internal/target/queue/queue.go:L752`] before `json.NewEncoder(file).Encode(metaCopy)` [`internal/target/queue/queue.go:L754`]. The struct is "serialized to the disk by the queue module using JSON" [`internal/module/msgmetadata.go:L52-L54`] and `Conn` is explicitly nullable [`internal/module/msgmetadata.go:L99-L104`]. So the authenticated identity is **absent from on-disk metadata**.
- **Other `AuthUser` consumers (context, not accept-time enforcement).** The `{auth_user}` placeholder in the `command` check returns `s.msgMeta.Conn.AuthUser` [`internal/check/command/command.go:L145-L149`]; downstream SASL forwarding uses `Conn.AuthUser`/`Conn.AuthPassword` [`internal/target/smtp_downstream/sasl.go:L31`, `internal/target/smtp_downstream/sasl.go:L41`]. Neither compares the envelope sender to the authenticated user at accept time.

**Decisive negative finding.** A sweep of `internal/` found **no accept-time enforcement** that compares the envelope `MAIL FROM` to the authenticated user. The built-in check modules are `command`, `dkim`, `dns`, `dnsbl`, `requiretls`, and `spf` — **none** enforces `MAIL FROM == AuthUser` at accept time. The only comparison anywhere is DKIM's `shouldSign` [`internal/modify/dkim/dkim.go:L249`], whose sole consequence is a **withheld signature**.

**Conclusion (R2a + R2b).** Accountability is **"soft"**: the authenticated identity is never rebound onto the envelope and never blocks the mismatched sender. It persists only at connection scope and appears durably nowhere on disk (`"Conn":null`). The **missing `DKIM-Signature`** is the only durable on-disk trace that the header `From` differed from the authenticated user — proven by the aligned control, which **is** signed with `i=usera@test.local`.

---

## Coverage pass

Every sub-part of both questions is answered explicitly above:

- [x] **R1a** — *stop / keep consuming / unexpected?* → **Stops at the first lone dot** (fail-safe, RFC 5321-compliant; the `net/textproto.DotReader` via go-smtp ends at the first `.`). Answered under **R1a**.
- [x] **R1b** — *runtime signs?* → `250 2.0.0 OK: queued` for the pre-dot message, then `500 5.5.2 Syntax error, MORE command unrecognized` and `501 5.5.2 Bad command` for the trailing lines, with `[debug] smtp: reset` logged in between. Answered under **R1b**.
- [x] **R1c** — *what is stored/queued?* → Only the pre-dot body (`24 bytes`, `normal content line one`; `MORE DATA AFTER DOT` absent), plus the injected `Received:` header carrying the envelope sender. Answered under **R1c**.
- [x] **R2a** — *reject / rebind / allow?* → **Allows** the second `MAIL FROM` as a different user; no rejection, no rebinding (auth is connection-scoped and survives `RSET`; `Mail()` does no sender-vs-auth check; routing is keyed on the envelope sender). Answered under **R2a**.
- [x] **R2b** — *which identity is trusted for headers, queue metadata, enforcement checks?* → The **envelope sender** is trusted for headers (`Received:` envelope-sender) and queue metadata (`.meta` `From`/`OriginalFrom`, `"Conn":null`); the **authenticated identity** surfaces only transiently in logs and in the single DKIM `shouldSign` enforcement point, where a mismatch merely **withholds the signature**. The missing `DKIM-Signature` is the only durable on-disk trace. Answered under **R2b** (with the decisive negative finding).

**Topology note** (587 vs bundled 465/25; the endpoint role is the invariant) and the **negative finding** (no accept-time `MAIL FROM == AuthUser` enforcement anywhere in `internal/`) are both stated above.

---

## Read-only scope confirmation

No maddy source file was modified, added, or deleted. The only new file introduced is `blitzy/documentation/maddy_26452dd8dd78.md`; the source `git status` is clean apart from this one file, and all temporary build/run artifacts (throwaway config, SQLite database, client scripts, captured logs, queue state) lived **outside** the source tree and were removed.

