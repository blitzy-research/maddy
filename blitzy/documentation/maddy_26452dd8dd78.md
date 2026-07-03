# How Maddy Enforces Sender Identity and Message Authentication for Authenticated SMTP Submission

**A runtime-grounded investigation.** This document answers, from *live runtime evidence*, how Maddy decides whether to accept or reject a sender address in an authenticated SMTP submission session, and how it applies (or skips) DKIM message signing. Every behavioral claim is paired with the exact observed output line that demonstrates it and with the `file:line` source anchor that implements it.

## Investigation methodology and provenance

The investigation was conducted *run-first*: a real `maddy` server was built from this repository and run in an out-of-tree scratch harness (`/tmp/maddy_scratch`, with its state directory outside the repository tree). Every value, SMTP transcript, stored header, DKIM artifact, and debug-log line quoted below was **captured verbatim from the running binary** over real SMTP (implicit TLS, port 465) and real IMAP (implicit TLS, port 993). After evidence capture the harness was torn down so the repository stays byte-for-byte unchanged (see the *Cleanup note*).

Provenance labels used throughout:

- **[observed]** — captured at runtime from the real running binary (SMTP response, stored header, debug log, console output).
- **[inferred]** — derived from reading the source; the exact path was not separately exercised. Always cited to `file:line`.
- **[non-canonical transport]** — an auxiliary transport that was stubbed because the canonical one is unavailable in the sandbox (specifically, DKIM public-key DNS for the un-registrable domain `test.example`). This is distinguished from the **canonical signing computation**, whose bytes are exactly what Maddy produced.

**Two enforcement layers must be kept rigorously separate**, and this document never conflates them:

- **Layer 1 — accept/reject of the envelope sender.** Decided purely by *source routing* on the `MAIL FROM` address in `srcBlockForAddr` [`internal/msgpipeline/msgpipeline.go:L155-L201`]. The authenticated identity plays **no** part in this decision.
- **Layer 2 — whether the message is DKIM-signed.** Decided independently by the signing modifier's `shouldSign()` under `require_sender_match` [`internal/modify/dkim/dkim.go:L151-L152, L249-L310`]. A failed check **skips signing but still delivers** — it never rejects.

---

## Section 1 — Environment and build

**Canonical environment [observed].** The build and run were performed in the canonical Docker image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_foxcpp_maddy_1.0`. The module is `github.com/foxcpp/maddy` and the module directive is `go 1.13`, cited to `go.mod`:

```
go 1.13
```

The repository is at branch `maddy_26452dd8dd78` (HEAD `26452dd8dd787dc455278b0fdd296f4a5432c768`).

**Build commands [observed].** The two binaries were built **out-of-tree** so the repository working tree is not dirtied. Note that `.gitignore` ignores `cmd/maddy/maddy` and `cmd/maddyctl/maddyctl` but **not** a root `./maddy`, so writing the binaries to `/tmp/maddy_scratch` is the safe choice:

```
go build -o /tmp/maddy_scratch/maddy ./cmd/maddy
go build -o /tmp/maddy_scratch/maddyctl ./cmd/maddyctl
```

Both builds exited `0`. The Go toolchain used was go1.20.14 (installed at `/usr/local/go`); gcc 13.3.0 was present for the `github.com/mattn/go-sqlite3` cgo build. The only compiler warning was a harmless `sqlite3-binding.c` `-Wreturn-local-addr` originating from `go-sqlite3 v1.11.0` — it does not affect the binary.

**Version banner [observed, verbatim].** Running `./maddy -v` prints:

```
maddy unknown (built from source tree)
```

This is the **canonical default-build value**: the version string defaults to `Version = "unknown (built from source tree)"` [`maddy.go:L41`] because no VCS build information is embedded in a plain `go build` from the source tree. `maddyctl --version` prints the same string. The `-v` flag itself is declared as `printVersion = flag.Bool("v", false, "print version and exit")` [`maddy.go:L109`].

---

## Section 2 — Configuration used, and why the substitutions are non-behavioral

The test configuration reproduces the **default `maddy.conf` pipeline structure verbatim** and substitutes **only** four non-behavioral values:

1. a controllable test domain, `test.example`, in place of the packaged placeholder;
2. a self-signed TLS certificate/key pair (required because both submission :465 and IMAP :993 are *implicit* TLS);
3. loopback listen addresses; and
4. an out-of-tree state directory (so `all.db` and the generated `dkim_keys/` never land in the repository).

None of these four substitutions touches the enforcement *logic* — source routing, `require_sender_match`, the reject codes, and the delivery target are all unchanged — so the observations below are canonical for Maddy's default policy.

**Preserved default structure, with source anchors.** The following directives are taken verbatim from the packaged `maddy.conf`:

- Authenticated submission endpoint on implicit TLS :465 — `submission tls://0.0.0.0:465 { auth &local_authdb ... }` [`maddy.conf:L93-L95`].
- Local `source` block with the DKIM modifier and local delivery — `source $(local_domains) { modify { sign_dkim $(primary_domain) default } destination $(local_domains) { ... deliver_to &local_mailboxes } }` [`maddy.conf:L97-L107`].
- The catch-all reject for non-local senders — `default_source { reject 501 5.1.8 "Non-local sender domain" }` [`maddy.conf:L117-L119`].
- SQLite-backed credential store + mailbox storage — `sql local_mailboxes local_authdb { driver sqlite3 ... dsn all.db }` [`maddy.conf:L32-L35`].
- The IMAP inspection endpoint on implicit TLS :993 — `imap tls://0.0.0.0:993 { auth &local_authdb; storage &local_mailboxes }` [`maddy.conf:L149-L152`].

The default configuration path is `/etc/maddy/maddy.conf`, formed by `filepath.Join(ConfigDirectory, "maddy.conf")` with `ConfigDirectory = "/etc/maddy"` [`maddy.go:L107`, `maddy.go:L48`]. The test config was supplied via the `-config` flag and the server was run with `-debug`.

**Startup evidence [observed, verbatim, from the debug log].** On first startup Maddy loaded the SQL storage, instantiated the `sign_dkim` module, and — because no key existed yet — auto-generated a fresh RSA-2048 keypair and wrote both the private key and the public-key DNS record file, then bound the two listeners:

```
[debug] sql: go-imap-sql version 0.4.0
[debug] /tmp/maddy_scratch/maddy_test.conf:27: new module sign_dkim [test.example default]
sign_dkim: generating a new rsa2048 keypair...
sign_dkim: generated a new rsa2048 keypair, private key is in dkim_keys/test.example_default.key, TXT record with public key is in dkim_keys/test.example_default.dns,
submission: listening on tls://0.0.0.0:465
imap: listening on tls://0.0.0.0:993
```

The keypair-generation line is produced by `m.log.Printf("generating a new %s keypair...", newKeyAlgo)` [`internal/modify/dkim/keys.go:L82`], and `rsa2048` is the default algorithm from the `newkey_algo` enum (`"rsa2048"` default) [`internal/modify/dkim/dkim.go:L149-L150`]. The private-key/DNS-record filenames come from the `.key`→`.dns` path derivation [`internal/modify/dkim/keys.go:L150-L152`].

---

## Section 3 — Account and DKIM-key setup

**Accounts were provisioned through the real CLI, not by direct database writes.** The `maddyctl` command surface [observed, verbatim] is:

```
COMMANDS:
   users        User accounts management
   imap-mboxes  IMAP mailboxes (folders) management
   imap-msgs    IMAP messages management
   help, h      Shows a list of commands or help for one command
GLOBAL OPTIONS:
   --config value  Configuration file to use (default: "/etc/maddy/maddy.conf") [$MADDY_CONFIG]
```

Note there is **no `imap-acct`** command in this version of `maddyctl`. The two users were created with `maddyctl --config <cfg> users create userA@test.example` (password supplied via `--password`), and likewise for `userB`. The `users` command is registered at `cmd/maddyctl/main.go:L47` and its `create` subcommand at `cmd/maddyctl/main.go:L71`; the account is persisted through the real `Storage.CreateUser` path at `internal/storage/sql/maddyctl.go:L17`.

**Two accounts, each with an auto-created INBOX [observed, verbatim]:**

```
$ maddyctl --config maddy_test.conf users list
usera@test.example
userb@test.example
$ maddyctl --config maddy_test.conf imap-mboxes list usera@test.example
INBOX
$ maddyctl --config maddy_test.conf imap-mboxes list userb@test.example
INBOX
```

The addresses were normalized to lowercase (`usera` / `userb`), and `INBOX` was auto-created by the go-imap-sql `CreateUser` path. In the boundary tests, **`userA` is the authenticating user** and **`userB` is the impersonated sender**.

**The DKIM public-key DNS record that a verifying recipient would fetch [observed, verbatim]**, captured from `state/dkim_keys/test.example_default.dns` — this is literally the value that would be published at `default._domainkey.test.example`:

```
v=DKIM1; k=rsa; p=MIIBCgKCAQEAotQXaMP7AMGaSVk7Gjry6+ovw79bnyv3Fcdb8o/9pdIyXwRZs4X7iCMqsGVU/1hrc0XyT3o+bO1XBk24bjUC30ZsZp5cT6clmR67NixXRtpoC2CU6VBvpZcKI3+RkfpwaC0Nzyr42hVnswgkxOXymQ0XW8Xiq+gKkHxP4vKIttISqX/hS+dq3tvNvkH2vnM+F2H9kxmz7ls8R86e9jhzDHDkYKJoZJhBbCKgJZnv9LIisF7+C7SCXi0xFfYOGTd7KEBUX8Jobk1dW7SzmYFqtCiWq7sKxfw7ZqYJaP6LkBEE5qVnRAbV2UF3br48UnY68/DtCROcyL/2d/65yNlcTwIDAQAB
```

The record is assembled by `fmt.Sprintf("v=DKIM1; k=%s; p=%s", ...)` [`internal/modify/dkim/keys.go:L158`]. The precise significance of how the `p=` bytes are encoded is analyzed in Section 6 (it is the primary config-vs-behavior divergence).

---

## Section 4 — Boundary tests with verbatim SMTP transcripts

**Client [observed].** The sending client was Python's `smtplib.SMTP_SSL` with `set_debuglevel(1)`, connecting over implicit TLS to :465 (self-signed certificate, verification disabled). This is a **TLS-terminating verbose client that drives the real submission endpoint** — it is not a bypassing interface; it exercises the exact code path a normal MUA would.

**Session capabilities [observed].** The endpoint advertised the following on `EHLO`: `PIPELINING`, `8BITMIME`, `ENHANCEDSTATUSCODES`, `AUTH PLAIN`, `SMTPUTF8`, and `SIZE 33554432`. Successful authentication is answered `235 2.0.0 Authentication succeeded`.

Six tests were run. **T3 is the required ≥1 accepted transaction; T2 is the required ≥1 rejected transaction.** T1a/T1b/Tm are the cross-user and mismatched-`From` cases; Tn is the missing-`From` case.

### Enumeration of every observed/derived SMTP response code

| Code | Enhanced | Meaning | Where |
|------|----------|---------|-------|
| `235` | `2.0.0` | Authentication succeeded | [observed] auth reply (T3) |
| `250` | `2.0.0` | MAIL FROM / RCPT TO / message accepted (`OK: queued`) | [observed] T3, T1a, T1b, Tm |
| `354` | `2.0.0` | Data go-ahead | [observed] T3 |
| `221` | `2.0.0` | Connection close on `QUIT` | [observed] T3 |
| `501` | `5.1.8` | `Non-local sender domain` (envelope domain not local) | [observed] T2; `maddy.conf:L117-L119` |
| `501` | `5.1.7` | `Unable to normalize the sender address` | [inferred] `internal/msgpipeline/msgpipeline.go:L162-L164` |
| `501` | `5.1.3` | `Invalid sender address` (reason "Can't extract local-part and host-part") | [inferred] `internal/msgpipeline/msgpipeline.go:L181-L185` |
| `502` | `5.5.1` | `Missing RCPT TO command.` (DATA after a rejected RCPT) | [observed] T2 |
| `554` | `5.6.0` | Missing/invalid `From`/`Sender`/recipient header family | [observed] Tn; `internal/endpoint/smtp/submission.go:L41-L103` |

### T3 — legitimate aligned message → ACCEPTED + SIGNED

Authenticate as `usera`, `MAIL FROM:<usera@test.example>`, `From: usera@test.example`. Full client transcript [observed, verbatim]:

```
send: 'ehlo client.test.example\r\n'
reply: b'250-Hello client.test.example\r\n' ... b'250-AUTH PLAIN\r\n' ... b'250 SIZE 33554432\r\n'
send: 'AUTH PLAIN AHVzZXJhQHRlc3QuZXhhbXBsZQBwYXNzd29yZEE=\r\n'
reply: b'235 2.0.0 Authentication succeeded\r\n'
send: 'mail FROM:<usera@test.example>\r\n'
reply: b'250 2.0.0 Roger, accepting mail from <usera@test.example>\r\n'
send: 'rcpt TO:<userb@test.example>\r\n'
reply: b"250 2.0.0 I'll make sure <userb@test.example> gets this\r\n"
send: 'data\r\n'
reply: b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
send: b'From: usera@test.example\r\nTo: userb@test.example\r\nSubject: T3-aligned-legit\r\n\r\nBody of T3-aligned-legit (...).\r\n.\r\n'
reply: b'250 2.0.0 OK: queued\r\n'
send: 'quit\r\n'
reply: b'221 2.0.0 Goodnight and good luck\r\n'
```

Matching Maddy `-debug` decision lines [observed, verbatim]:

```
[debug] smtp/pipeline: sender usera@test.example matched by domain rule 'test.example'   {"msg_id":"a3414c27"}
submission: adding missing Message-ID
submission: adding missing Date header
[debug] sign_dkim: signed   {"identifier":"usera@test.example"}
submission: accepted   {"msg_id":"a3414c27"}
```

- **Claim: the envelope was routed to the local `source` block by its domain** — evidence: `sender usera@test.example matched by domain rule 'test.example'`. This is the domain match in `srcBlockForAddr` [`internal/msgpipeline/msgpipeline.go:L190`] logged at [`msgpipeline.go:L196`].
- **Claim: the message was DKIM-signed** — evidence: `sign_dkim: signed   {"identifier":"usera@test.example"}`, emitted by `s.m.log.DebugMsg("signed", "identifier", id)` after `h.Add("DKIM-Signature", signer.SignatureValue())` [`internal/modify/dkim/dkim.go:L406-L408`].
- **Claim: the message was accepted and queued** — evidence: the SMTP reply `250 2.0.0 OK: queued` and the log line `submission: accepted   {"msg_id":"a3414c27"}`.

### T2 — non-local sender domain → REJECTED `501 5.1.8` (deferred to RCPT)

Authenticate as `usera`, `MAIL FROM:<spoof@nonlocal.tld>`. Full client transcript [observed, verbatim]:

```
send: 'mail FROM:<spoof@nonlocal.tld>\r\n'
reply: b'250 2.0.0 Roger, accepting mail from <spoof@nonlocal.tld>\r\n'
send: 'rcpt TO:<userb@test.example>\r\n'
reply: b'501 5.1.8 Non-local sender domain (msg ID = 9f53b0a5)\r\n'
send: 'data\r\n'
reply: b'502 5.5.1 Missing RCPT TO command.\r\n'
```

Matching Maddy `-debug` decision lines [observed, verbatim]:

```
[debug] smtp/pipeline: sender spoof@nonlocal.tld matched by default rule   {"msg_id":"9f53b0a5"}
submission: RCPT error   {"effective_rcpt":"userb@test.example","rcpt":"userb@test.example","reason":"reject directive used","smtp_code":501,"smtp_enchcode":"5.1.8","smtp_msg":"Non-local sender domain"}
submission: aborted   {"msg_id":"9f53b0a5"}
```

- **Claim: a non-local envelope domain falls through to `default_source`** — evidence: `sender spoof@nonlocal.tld matched by default rule`. This is the default-rule fallback in `srcBlockForAddr` [`internal/msgpipeline/msgpipeline.go:L190` no match → default] logged at [`msgpipeline.go:L194`].
- **Claim: the rejection renders the configured `501 5.1.8 "Non-local sender domain"`** — evidence: `smtp_code":501,"smtp_enchcode":"5.1.8","smtp_msg":"Non-local sender domain"`. The literal is the `reject` directive at [`maddy.conf:L117-L119`], rendered by `parseRejectDirective` [`internal/msgpipeline/config.go:L292`].
- **Claim (critical): `MAIL FROM` is answered `250` and the rejection surfaces only at `RCPT`, because sender rejection is deferred** — evidence: the transcript shows `mail FROM:<spoof@nonlocal.tld>` → `250 2.0.0 Roger, accepting mail from <spoof@nonlocal.tld>` yet `rcpt TO` → `501 5.1.8 Non-local sender domain`. This is the `defer_sender_reject` default, which is `true`: `cfg.Bool("defer_sender_reject", false, true, &endp.deferServerReject)` [`internal/endpoint/smtp/smtp.go:L567`], taking the deferral branch `if !s.endp.deferServerReject` [`internal/endpoint/smtp/smtp.go:L163`]. **A reader must not misread the `250` at `MAIL` as acceptance** — the transaction was rejected at `RCPT`. The subsequent `DATA` then returns `502 5.5.1 Missing RCPT TO command.` because no recipient was ever accepted.

### T1a — same-domain cross-user (envelope + `From` = userB, auth = userA) → ACCEPTED + DKIM SKIPPED

Authenticate as `usera`; `MAIL FROM:<userb@test.example>`; `From: userb@test.example`. Transcript key lines [observed]: `MAIL FROM:<userb@test.example>` → `250 2.0.0 Roger, accepting mail from <userb@test.example>`; `RCPT` → `250`; `DATA` → `250 2.0.0 OK: queued`. Matching Maddy `-debug` lines [observed, verbatim]:

```
[debug] smtp/pipeline: sender userb@test.example matched by domain rule 'test.example'   {"msg_id":"16a76681"}
sign_dkim: not signing, From address is not authenticated identity   {"auth_id":"usera@test.example","from_addr":"userb@test.example","msg_id":"16a76681"}
submission: accepted   {"msg_id":"16a76681"}
```

- **Claim: `userA`, authenticated, successfully sent a message bearing `userB`'s address — the message was accepted** — evidence: `submission: accepted   {"msg_id":"16a76681"}` (and `250 2.0.0 OK: queued`). This is the primary falsification evidence used in Section 8.
- **Claim: the only consequence of the auth/`From` mismatch was that signing was skipped (not that the message was rejected)** — evidence: `sign_dkim: not signing, From address is not authenticated identity`. This is the `auth` check in `shouldSign()` [`internal/modify/dkim/dkim.go:L299-L308`].

### T1b — same-domain cross-user (MAIL FROM = userB, `From` = userA, auth = userA) → ACCEPTED + DKIM SKIPPED

Matching Maddy `-debug` lines [observed, verbatim]:

```
[debug] smtp/pipeline: sender userb@test.example matched by domain rule 'test.example'   {"msg_id":"970eb97b"}
sign_dkim: not signing, From address is not envelope address   {"envelope":"userb@test.example","from_addr":"usera@test.example","msg_id":"970eb97b"}
submission: accepted   {"msg_id":"970eb97b"}
```

- **Claim: the message was accepted despite `From` (userA) not matching the envelope (userB)** — evidence: `submission: accepted   {"msg_id":"970eb97b"}`.
- **Claim: signing was skipped by the `envelope` check, which fires *before* the `auth` check** — evidence: `sign_dkim: not signing, From address is not envelope address`. This is the `envelope` check [`internal/modify/dkim/dkim.go:L293-L296`], ordered ahead of the `auth` check [`dkim.go:L299`].

### Tm — mismatched `From`, foreign domain (auth = userA, MAIL FROM = userA, `From: stranger@other.invalid`) → ACCEPTED + DKIM SKIPPED

This is the user's explicit *"From matches neither the authenticated user nor the signing domain"* case. Matching Maddy `-debug` lines [observed, verbatim]:

```
[debug] smtp/pipeline: sender usera@test.example matched by domain rule 'test.example'   {"msg_id":"f8877787"}
sign_dkim: not signing, From domain is not key domain   {"from_domain":"other.invalid","key_domain":"test.example","msg_id":"f8877787"}
submission: accepted   {"msg_id":"f8877787"}
```

- **Claim: routing accepted the message because the *envelope* (`usera@test.example`) is local — the `From` header plays no part in Layer 1** — evidence: `sender usera@test.example matched by domain rule 'test.example'` and `submission: accepted   {"msg_id":"f8877787"}`.
- **Claim: signing was skipped because the `From` domain (`other.invalid`) is not the key domain (`test.example`)** — evidence: `sign_dkim: not signing, From domain is not key domain   {"from_domain":"other.invalid","key_domain":"test.example",...}`. This is the From-domain check [`internal/modify/dkim/dkim.go:L287-L290`]. The message was therefore **delivered unsigned** (analyzed in Section 6).

### Tn — missing `From` header entirely → REJECTED `554 5.6.0` at DATA

`DATA` reply [observed, verbatim]:

```
554 5.6.0 Message does not contains a From header field (msg ID = 6647b5cf)
```

Matching Maddy `-debug` lines [observed, verbatim]:

```
submission: DATA error   {"modifier":"submission_prepare","msg_id":"6647b5cf","reason":"Message does not contains a From header field","smtp_code":554,"smtp_enchcode":"5.6.0","smtp_msg":"Message does not contains a From header field"}
submission: aborted   {"msg_id":"6647b5cf"}
```

- **Claim: a submission with no `From` header is rejected `554 5.6.0` at DATA with the message "Message does not contains a From header field"** — evidence: the `DATA` reply above. Note the source-code grammatical typo **"contains"** is reproduced verbatim; the literal is at [`internal/endpoint/smtp/submission.go:L41-L43`].

**The rest of the missing/invalid-`From` family [inferred from source; not each variant was separately exercised]:** the same `554 5.6.0` class also covers `"Invalid address in <Sender/To/Cc/Bcc/Reply-To>"` [`internal/endpoint/smtp/submission.go:L54-L56, L70-L72`], `"Invalid address in From"` [`submission.go:L86-L88`], and `"Missing Sender header field"` when there are multiple `From` addresses without a `Sender` header [`submission.go:L101-L103`]. The submission handler also sets `msgMeta.DontTraceSender = true` [`submission.go:L28`], which shapes the `Received` header in Section 5.


---

## Section 5 — Verbatim stored headers (read back through the real IMAP endpoint)

**Inspection method [observed].** The delivered messages were read back through the real IMAP endpoint using Python's `imaplib.IMAP4_SSL` to :993, `SELECT INBOX`, and `FETCH (BODY.PEEK[HEADER])` for `userb@test.example`. (The IMAP/storage read is the sanctioned inspection path — it reads the message exactly as Maddy stored it.) Four messages were stored in `userb`'s INBOX (T3, T1a, T1b, Tm); the three misaligned messages are unsigned.

**T3 (signed) full stored header block [observed, verbatim].** Reproduced exactly as stored, including the header folding, the two-space `Received:  by`, and the lowercase `Dkim-Signature` / `Message-Id` rendering produced by go-imap-sql:

```
Delivered-To: userb@test.example
Return-Path: <usera@test.example>
Dkim-Signature: a=rsa-sha256;
 bh=iT9Qop7GXJAOgqBBZyj0IZvaW+ehSyasvZhaVisaQY4=; c=relaxed/relaxed;
 d=test.example;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=usera@test.example; s=default; t=1783029517; v=1; x=1783461517; b=n5Eh0zfTbkoqpOg57JiT5vs0WSYasQ/udwh3+SI2jHKC8JhKfQuNqXTcv8YY4/mBAfLnyuWpXchIHnWt9QY4tji1MdGhchj/D32xr8sxc3wpFoMHVwykdtO+YXjM+z5eVxeKjJuSZNQNtJLg+Ng8B9GLVsi3gyUSvNxTKKz7aaKLyhYqEJABi2ks31YCJ2SUh44n9gm85i3IWvGbyhyS8hz0sNtOzJgBwxWcVEtNC++BeukGtpqnl5iDg9JWY+WI9UGuBEOYED+wKt2R2n0zAmuiEeJoUJlxmNr/m3pM18TGHM624y7s6waLQt/KzFOT3r4JWyMtlZLEiIQ31aTcZg==;
Received:  by test.example (envelope-sender <usera@test.example>) with
 ESMTPS id a3414c27; Thu, 02 Jul 2026 21:58:37 +0000
Date: Thu, 2 Jul 2026 21:58:37 +0000
Message-Id: <ecf1ce83-4049-4d23-8075-be13e230b0d8@test.example>
From: usera@test.example
To: userb@test.example
Subject: T3-aligned-legit
```

- **Claim: the `DKIM-Signature` covers the `From` header — in fact it *oversigns* it.** Evidence: the `h=` tag contains `From` **twice** (`...Cc:From:From:Date...`). Oversigning adds the header name a second time so that a maliciously *added* second `From` would break verification. `From` is in Maddy's oversign default list [`internal/modify/dkim/dkim.go:L31-L38`, `From` at `L37`]. The full tag set observed is: `v=1`, `a=rsa-sha256`, `c=relaxed/relaxed`, `d=test.example`, `s=default`, `i=usera@test.example`, `bh=iT9Qop7GXJAOgqBBZyj0IZvaW+ehSyasvZhaVisaQY4=`, `t=1783029517`, `x=1783461517`, `b=n5Eh0zfT...` (truncated for readability but reproduced in full in the block above).
- **Claim: the signature is *DMARC-alignable in principle*, because `d=` equals the `From` domain.** Evidence: `d=test.example` and `From: usera@test.example` share the domain `test.example`. Whether it *validates* is a separate question answered in Section 6 (it does not).
- **Claim: the signature carries a 5-day expiry.** Evidence: `t=1783029517` and `x=1783461517`; `1783461517 − 1783029517 = 432000` seconds = 5 days.
- **Claim: the `Received` header Maddy added for submission has *no* leading `from <host> [ip]` client-trace clause; it begins directly with `by`.** Evidence: `Received:  by test.example (envelope-sender <usera@test.example>) with ESMTPS id a3414c27; ...`. The client-trace clause is gated on `!DontTraceSender` [`internal/target/received.go:L30`], and submission sets `DontTraceSender = true` [`internal/endpoint/smtp/submission.go:L28`], so the builder skips straight to the ` by ` clause [`internal/target/received.go:L62`], then ` (envelope-sender <` [`received.go:L69`], ` with ` + protocol (`ESMTPS`) [`received.go:L75-L79`], and ` id ` + the message ID [`received.go:L81-L82`]. The two spaces in `Received:  by` are the header-separator space plus the value's own leading space before `by`. Header generation is `GenerateReceived` [`internal/target/received.go:L19`].
- **Claim: there is *no* `Authentication-Results` header on any stored message.** Evidence: the header is absent from every fetched header block. The header is only added when checks emit results — `if len(cr.mergedRes.AuthResult) != 0 { header.Add("Authentication-Results", ...) }` [`internal/msgpipeline/check_runner.go:L301-L302`]. The default submission block defines no checks, so no `Authentication-Results` is produced. (This is confirmed **absent** at runtime, not merely expected.)

**T1a / T1b / Tm (unsigned) [observed].** None of the three misaligned messages carries a `Dkim-Signature` header; each carries `Delivered-To`, `Return-Path`, and a submission-shape `Received` (same `Received:  by ...` form, no from-clause). For example, the Tm stored message shows `From: stranger@other.invalid` with `Return-Path: <usera@test.example>` and no signature — confirming the skip-not-reject behavior at the storage layer.

---

## Section 6 — DKIM under a mismatched `From`, and what a verifying recipient actually sees

This is the deepest sub-question. It has three parts: (a) how sign-vs-skip is decided and the full enumeration of skip reasons; (b) the specific mismatched-`From` case; and (c) whether a recipient can actually verify the one signed message.

### (a) Sign-vs-skip is decided by `shouldSign()`, and a failed check *skips signing but still delivers*

Signing is gated by `require_sender_match`, whose default is `[envelope, auth]` [`internal/modify/dkim/dkim.go:L151-L152`]. When any check fails, `RewriteBody()` returns `nil` (no error) instead of raising an SMTP error:

```
id, ok := s.m.shouldSign(...)
if !ok {
    return nil
}
```

This **skip-not-reject** behavior is at [`internal/modify/dkim/dkim.go:L348-L350`] and was observed at runtime in T1a, T1b, and Tm — all three were `submission: accepted` and **delivered unsigned**. It is essential not to confuse this with a routing reject.

**Every `require_sender_match` / `shouldSign()` skip reason, with its exact log message and `file:line`:**

| Skip reason | Exact log message | Source |
|-------------|-------------------|--------|
| `off` token present | (signing disabled entirely; no per-message log) | `dkim.go:L250` |
| empty `From` | `not signing, empty From` | `dkim.go:L266` |
| malformed `From` field | `not signing, malformed From field` | `dkim.go:L271` |
| multiple addresses in `From` | `not signing, multiple addresses in From` | `dkim.go:L275` |
| malformed address in `From` | `not signing, malformed address in From` | `dkim.go:L282` |
| `From` domain ≠ key domain | `not signing, From domain is not key domain` | `dkim.go:L288` **[observed: Tm]** |
| `envelope` check (`From` ≠ MAIL FROM) | `not signing, From address is not envelope address` | `dkim.go:L294` **[observed: T1b]** |
| `auth` check (`From` ≠ authenticated identity) | `not signing, From address is not authenticated identity` | `dkim.go:L306` **[observed: T1a]** |

The `From`-domain check unconditionally requires `From` domain to equal the key domain [`dkim.go:L287-L290`]; the `envelope` check is guarded by the `envelope` token [`dkim.go:L293`]; and the `auth` check is guarded by the `auth` token [`dkim.go:L299`].

### (b) The mismatched-`From` case (the user's follow-on question)

With `From: stranger@other.invalid` (matching **neither** the authenticated user `usera@test.example` **nor** the signing/key domain `test.example`), the message is **not signed** and is delivered unsigned. Evidence [observed, Tm]: `sign_dkim: not signing, From domain is not key domain   {"from_domain":"other.invalid","key_domain":"test.example",...}`.

Answering the three sub-parts precisely:

- **Is the signature still applied?** No — evidence: the Tm debug line above and the absence of any `Dkim-Signature` header on Tm's stored message (Section 5). Because the From-domain check fails first, `shouldSign()` returns `("", false)` and `RewriteBody()` returns `nil` without adding a signature.
- **Does it cover `From`?** There is no signature at all, so there is nothing that covers `From`.
- **What would a verifying recipient see?** An ordinary **unsigned** message (with `From: stranger@other.invalid` and `Return-Path: <usera@test.example>`), i.e. no DKIM protection whatsoever.

### (c) What a verifying recipient actually sees for the one signed message (T3) — DKIM verification deep-dive

- **The signature covers `From` (oversigned, Section 5) and the body hash is correct [observed].** An independent recomputation of the body hash under *both* the relaxed and the simple canonicalizations matched the signature's `bh=iT9Qop7GXJAOgqBBZyj0IZvaW+ehSyasvZhaVisaQY4=`. So the message *body* is intact end-to-end.

- **PRIMARY divergence — the published public key is PKCS#1, not the RFC-6376 SubjectPublicKeyInfo (SPKI) [observed + source-confirmed].** The root cause is that Maddy marshals the RSA public key with `keyBlob = x509.MarshalPKCS1PublicKey(pubkey)` [`internal/modify/dkim/keys.go:L143`], which emits a bare PKCS#1 `RSAPublicKey`, whereas DKIM verifiers expect a DER-encoded SPKI. Runtime proof: `openssl asn1parse` of the captured `p=` decodes to `SEQUENCE { INTEGER (modulus), INTEGER (010001) }` — a bare PKCS#1 `RSAPublicKey` (an SPKI key would instead wrap an `AlgorithmIdentifier` plus a `BIT STRING`). Consistently, the captured `p=` base64 begins `MIIBCgKCAQEA...`, whereas an RFC-correct SPKI key begins `MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A...`. When the real published record is handed to a standard verifier — **go-msgauth, the very library Maddy uses both to sign and to verify (its `verify_dkim` check), version `v0.3.2-0.20191028231513-55b75676976c`** per `go.mod` — it fails to parse the key [observed, verbatim]:

  ```
  dkim: key syntax error: x509: failed to parse public key (use ParsePKCS1PublicKey instead for this key format)
  ```

  **Therefore a recipient using Maddy's own DKIM stack cannot even parse the published key**, so the signature can never be validated. *Provenance:* the DNS lookup was **stubbed / non-canonical** — `test.example` cannot be published in real DNS, so a local resolver returned the captured record — but the **key bytes and the signing computation are exactly what Maddy produced**, so this is a canonical statement about Maddy's output, not about the stubbed transport.

- **SECONDARY observation — even a format-corrected key fails to verify the delivered copy [observed; root byte not isolated].** After re-encoding the *same* key into correct SPKI (`openssl rsa -RSAPublicKey_in -pubout`), go-msgauth parses it but then reports:

  ```
  dkim: signature did not verify: crypto/rsa: verification error
  ```

  on the copy obtained from storage (IMAP) and from an SMTP relay capture, even though the body hash matches; the independent `dkimpy` verifier (which is lenient enough to parse the PKCS#1 key directly) *also* fails the header signature. **Control proving the harness is correct:** a go-msgauth sign→verify round-trip with the *same* key succeeds (`Err=<nil>`) across all four canonicalization/oversign variants — simple/relaxed × no-oversign/oversign — including the relaxed + oversign combination that matches Maddy's configuration. Since go-msgauth verifies its own oversigned + relaxed output, the delivered-message failure indicates that the **stored/relayed header bytes differ from the exact bytes Maddy signed** — consistent with the header set being re-serialized after the `DKIM-Signature` is added. The precise mutated byte is **not isolated** and this re-serialization is **[inferred]**; it was observed only on stored/relayed copies (a real outbound `target.remote` path was not separately tested).

- **Net answer [observed]:** aligned mail *is* signed and the signature *does* cover `From`, but a verifying recipient **cannot validate it** — primarily because the published key record is PKCS#1 rather than SPKI (a standard verifier rejects the key outright), and secondarily because the delivered copy's signature does not validate under test even with a format-corrected key (while the body hash does match).

### Protocol-capture method and the fallback that was required

Per the repo-integrity directive, the capture approach and its fallback are stated explicitly:

- **Plaintext `tcpdump` is insufficient** because :465 and :993 are *implicit* TLS — a raw packet capture yields ciphertext. In this environment `tcpdump` was additionally **not even installed** [observed]: `tcpdump: command not found`. A plaintext `EHLO` to :465 likewise returned nothing usable [observed]: the connection closed with no SMTP banner (the server expects a TLS handshake first).
- **The sanctioned alternatives that reveal the real exchange were used:** (1) Maddy's own `-debug` log (the decision lines quoted throughout Section 4); and (2) a **TLS-terminating verbose client**. For example, `openssl s_client -connect 127.0.0.1:465 -quiet` returned the real banner [observed] `220 test.example ESMTP Service Ready` followed by `221 2.0.0 Goodnight and good luck`; and `smtplib.SMTP_SSL` with `set_debuglevel(1)` produced the full dialogues in Section 4. A TLS-terminating client that observes the true dialogue is **not** a bypassing interface — it speaks the real SMTP protocol to the real endpoint.


---

## Section 7 — Conclusion on default-config sender alignment

**Accept/reject is decided *only* by the envelope `MAIL FROM` domain (Layer 1); the authenticated identity is never consulted for acceptance.** Evidence: `srcBlockForAddr` operates on `var cleanFrom = mailFrom` [`internal/msgpipeline/msgpipeline.go:L156`] and matches `dd.d.perSource[domain]` [`msgpipeline.go:L190`] — the authenticated user does not appear in this function at all. At runtime, T1a and T1b (cross-user, same *local* domain) were both **accepted** (`submission: accepted`), whereas T2 (a *non-local* domain) was **rejected** `501 5.1.8`. The dividing line is the *envelope domain's locality*, not the sender's identity.

**DKIM signing (Layer 2) is a separate decision and is skip-not-reject.** Evidence: misaligned mail (T1a, T1b, Tm) was delivered **unsigned**, not rejected, because `RewriteBody()` returns `nil` on a failed check [`internal/modify/dkim/dkim.go:L348-L350`].

**Therefore, Maddy's default configuration does *not* bind `MAIL FROM` or `From` to the logged-in user.** Per-user sender-alignment enforcement is simply not part of the default pipeline; it would require explicit policy. The two layers must be held rigorously apart: acceptance is a Layer-1 (envelope-domain) decision, and signing is a Layer-2 (`shouldSign`) decision — a DKIM skip must never be described as a routing reject, nor vice-versa.

---

## Section 8 — Ruled-out incorrect interpretation (falsification)

**Hypothesis ruled out: "authenticated submission binds `MAIL FROM` / `From` to the logged-in user by default."**

If that hypothesis were true, T1a (authenticate as `usera`, but `MAIL FROM:<userb@test.example>`) would have been **rejected**. Instead it was **accepted** end to end [observed]:

```
mail FROM:<userb@test.example>
250 2.0.0 Roger, accepting mail from <userb@test.example>
rcpt TO:<userb@test.example>
250 ...
data → 250 2.0.0 OK: queued
submission: accepted   {"msg_id":"16a76681"}
```

The *only* consequence of the identity mismatch was that signing was skipped [observed]: `sign_dkim: not signing, From address is not authenticated identity` — a **skip, not a reject**. This directly falsifies the binding hypothesis: the authenticated identity gates *signing* (Layer 2), it does **not** gate *acceptance* (Layer 1).

---

## Section 9 — Config-vs-behavior divergence

**Primary divergence.** The presence of `sign_dkim $(primary_domain) default` in the default `source` block [`maddy.conf:L99`] invites the reasonable reading that *authenticated mail is signed with a verifiable signature*. Runtime observation contradicts this on two counts:

- **(a) Misaligned-`From` mail is silently delivered *unsigned* (skip-not-reject).** Evidence [observed, T1a/T1b/Tm]: e.g. `sign_dkim: not signing, From address is not authenticated identity` followed by `submission: accepted`. Source: `if !ok { return nil }` [`internal/modify/dkim/dkim.go:L348-L350`]. A reader expecting a rejection (or at least a warning to the sender) instead gets a delivered, unsigned message and no SMTP-level signal.
- **(b) Even the aligned, signed message cannot be verified by a standard verifier, because Maddy publishes the key in PKCS#1 rather than SPKI.** Evidence [observed, verbatim]:

  ```
  dkim: key syntax error: x509: failed to parse public key (use ParsePKCS1PublicKey instead for this key format)
  ```

  Source root cause: `x509.MarshalPKCS1PublicKey(pubkey)` [`internal/modify/dkim/keys.go:L143`]. So the config that appears to provide verifiable DKIM signing produces, in the canonical default build, a published key that a standard verifier (including Maddy's own go-msgauth) rejects.

**Secondary divergences (each runtime-supported):**

- **The `require_sender_match` default token `auth` is not among the documented valid enum values.** The enum declares valid values `["envelope", "auth_domain", "auth_user", "off"]` yet sets the default to `["envelope", "auth"]` [`internal/modify/dkim/dkim.go:L151-L152`], and `shouldSign()` really does test `m.senderMatch["auth"]` [`dkim.go:L299`]. So the *default* uses a token (`auth`) that the *configuration surface* does not list as selectable — a config-surface-vs-runtime-default discrepancy. This was exercised at runtime: T1a's skip reason `From address is not authenticated identity` is exactly the `auth`-token branch firing under the default.
- **Sender rejection surfaces at `RCPT`, not at `MAIL FROM`.** A reader of `default_source { reject ... }` might expect `MAIL FROM:<spoof@nonlocal.tld>` to be rejected immediately, but it is answered `250` and the `501 5.1.8` appears only at `RCPT` because `defer_sender_reject` defaults `true` [`internal/endpoint/smtp/smtp.go:L567`]. Evidence [observed, T2]: `mail FROM ... 250 2.0.0 Roger, accepting mail from <spoof@nonlocal.tld>` then `rcpt TO ... 501 5.1.8 Non-local sender domain`.

---

## Section 10 — Standards grounding (context; does not replace runtime evidence)

The following situates the observed behavior within the relevant RFCs. This is *standards context*, not runtime evidence.

- **RFC 6409 §6.1 (Message Submission).** Enforcing that `MAIL FROM` matches the authenticated identity is an **optional** MSA action — an MSA *MAY* reject with `550 5.7.1` when the sender lacks submission rights, but it is not required to. Domain-level-only locality (exactly what Maddy's default does) is therefore standards-conformant. Maddy's `501` invalid-address path likewise aligns with RFC 6409's guidance to reject syntactically improper addresses with `501`.
- **RFC 6376 (DKIM).** The DKIM signing identity is deliberately decoupled from header addresses, and verifiers fetch the public key from a TXT record at `{selector}._domainkey.<domain>` — here `default._domainkey.test.example`. The `p=` value is expected to be a DER-encoded **SubjectPublicKeyInfo**; this is precisely why the PKCS#1 encoding observed in Sections 6 and 9 is a defect. RFC 6376's oversigning rationale (hashing a header name more times than it appears, so an added duplicate breaks verification) explains why Maddy oversigns `From`.
- **RFC 7489 (DMARC).** DKIM *alignment* — the signature's `d=` domain aligning with the `From` header domain — is a DMARC concern, not a base-DKIM one. This is why `shouldSign()` refuses to sign a `From`-misaligned message: signing it could never yield DMARC alignment, so Maddy declines rather than emit a signature that is useless for alignment.

The synthesis: a reasonable reading of the default `maddy.conf` (`auth &local_authdb` next to `sign_dkim`) might suggest that authenticated submission binds the sender to the logged-in user and always signs their mail. The standards show that per-user enforcement is *optional* (RFC 6409 §6.1) and that DKIM declines to sign misaligned mail to protect DMARC alignment — which is exactly the behavior confirmed at runtime: Maddy's default accepts same-domain cross-user senders and silently skips signing on misalignment rather than rejecting.

---

## Cleanup note (repository integrity)

**Created out-of-tree in `/tmp/maddy_scratch` (never inside the repository):** the two build binaries (`maddy`, `maddyctl`); a structure-preserving test configuration; a self-signed TLS certificate and private key for `test.example`; the SQLite `all.db` (users + delivered messages); the auto-generated `dkim_keys/` directory (private key + `.dns` public-key record); and temporary observation scripts (the `smtplib.SMTP_SSL` send client, the `imaplib.IMAP4_SSL` fetch client, a raw SMTP sink used for on-the-wire capture, a local DNS stub for the DKIM key lookup, and out-of-tree Go DKIM verify / round-trip helper modules).

**Removed after evidence capture:** all of the above artifacts were deleted and the Maddy / sink / DNS-stub processes were stopped. Nothing dirtied the repository: `.gitignore` already ignores `*.pem` / `*.crt` / `*.key`; the SQLite database and `dkim_keys/` lived outside the tree; and the build binaries were written out-of-tree (a root `./maddy` is *not* ignored, which is why `-o /tmp/maddy_scratch/...` was used).

**Confirmation:** `git status --porcelain` on the source repository is **empty** (the tree is byte-for-byte unchanged; the baseline was verified clean before any work began). The only permanent addition anywhere is this document, `blitzy/documentation/maddy_26452dd8dd78.md`.

---

## Final coverage confirmation

- **Three named sends** — cross-user `MAIL FROM` (T1a/T1b), non-local `MAIL FROM` domain (T2), legitimate signed (T3) — all present, plus the mismatched-`From` case (Tm) and the missing-`From` case (Tn).
- **≥1 rejected transcript** (T2, `501 5.1.8`) and **≥1 accepted transcript** (T3, `250 2.0.0 OK: queued`), both verbatim.
- **Every SMTP code enumerated** with literal and cause (Section 4 table): `235`, `250`, `354`, `221`, `501 5.1.8`, `501 5.1.7` [inferred], `501 5.1.3` [inferred], `502 5.5.1`, `554 5.6.0` family.
- **Every `require_sender_match` / `shouldSign()` skip variant** enumerated with its exact log message and `file:line` (Section 6 table).
- **Verbatim `DKIM-Signature`** (with `From` in `h=`), **`Received`** (no from-clause), and the **confirmed-absent `Authentication-Results`** (Section 5).
- **Conclusion** (Layer-1 domain-only accept; Layer-2 skip-not-reject), **falsification** (T1a acceptance), **divergence** (PKCS#1 key + skip-not-reject + `auth` enum + deferred `RCPT`), and **standards grounding** (RFC 6409 / 6376 / 7489).
- **Provenance labels** applied throughout: **[observed]** vs **[inferred]**, and the stubbed DKIM DNS is marked **[non-canonical transport]** distinct from the canonical signing computation.
- **Cleanup note** included, with the statement that the repository is byte-for-byte unchanged and this document is the only permanent addition. No source file was modified; the document is self-contained.

