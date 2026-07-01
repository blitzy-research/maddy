# How Maddy Enforces Sender Identity & Message Authentication on an Authenticated SMTP Submission Session — A Runtime Investigation

## Abstract

This document answers, from **observed runtime behavior**, how the Maddy email server (commit `26452dd8dd787dc455278b0fdd296f4a5432c768`, branch `maddy_26452dd8dd78`) enforces sender identity and message authentication on an authenticated SMTP *submission* session. Rather than reasoning from the source alone, the server was actually built and run, driven through the user's boundary scenarios over a live SMTP connection, and its stored output inspected byte-for-byte. The headline finding is that Maddy's **default** configuration authorizes the envelope sender by **domain membership, not by the authenticated user** — so authenticating as one user and sending as a *different same-domain user* is **accepted** — while DKIM "sender match" gates only *signing*, never *acceptance*. Every claim below is backed by either a verbatim captured value or an exact `file:line` citation.

> **User's framing (verbatim):** *"I want to understand how Maddy actually enforces sender identity and message authentication at runtime, without relying on assumptions from reading the source or configuration documentation."*

## Environment at a glance

| Property | Observed value |
|----------|----------------|
| Repository commit | `26452dd8dd787dc455278b0fdd296f4a5432c768` (branch `maddy_26452dd8dd78`) |
| Go toolchain (observed) | `go version go1.18.10 linux/amd64` — see note below |
| C compiler / CGO | `gcc (Debian 10.2.1-6) 10.2.1 20210110`, `CGO_ENABLED=1` (required by `github.com/mattn/go-sqlite3`) |
| Module lower bound | `go 1.13` (`go.mod:L3`) |
| CI baseline | `go build ./...` (`.build.yml:L11`), `go test ./... -cover -race` (`.build.yml:L13`) |
| Binaries | `maddy`, `maddyctl` (prebuilt in the pinned image; `maddy -v` → `maddy unknown (built from source tree)`) |
| Storage/auth backend | one `sql` (SQLite) instance serving BOTH `local_mailboxes` and `local_authdb` |
| Endpoints exercised | `submission tls://0.0.0.0:4650` (implicit TLS), `submission tcp://0.0.0.0:4655` (plaintext, for wire visibility), `imap tls://0.0.0.0:4930` |
| Accounts | `alice@example.org`, `bob@example.org` |

> **Honesty note on the toolchain (Rule 5).** The task blueprint asserted the build used "Go 1.21.6". The **observed** toolchain in the running environment was `go1.18.10` (`/usr/local/go/VERSION` = `go1.18.10`; `go version` → `go version go1.18.10 linux/amd64`). This document reports the **observed** value. Either toolchain satisfies the module's declared lower bound of `go 1.13` (`go.mod:L3`) and neither changes the behavior analyzed here.

> **Methodology note.** All experiment artifacts (test config, self-signed TLS, DKIM key, `all.db`, scripts, transcripts) were created **outside** the repository tree under `/tmp/maddy-exp` and removed afterward. Throughout the experiment `git status --porcelain` reported an empty (pristine) working tree; the only persistent addition to the repository is this document.

---

## R1 — Environment & Setup

### What was built and run

`maddy` and `maddyctl` were built from this repository inside the provided Docker image (`andrewparkscaleai/coding-agent:foxcpp__maddy__26452dd8dd787dc455278b0fdd296f4a5432c768`, from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_foxcpp_maddy_1.0`) with `CGO_ENABLED=1` and `CC=gcc`. CGO is mandatory because `github.com/mattn/go-sqlite3 v1.11.0` is a C-backed driver. The repository's own CI (`.build.yml`) uses the same two commands as its baseline:

```
# .build.yml:L11
- go build ./...
# .build.yml:L13
- go test ./... -cover -race
```

The observed toolchain:

```
$ go version
go version go1.18.10 linux/amd64
$ gcc --version | head -1
gcc (Debian 10.2.1-6) 10.2.1 20210110
```

### Ephemeral test configuration (modeled on the default `maddy.conf`)

The runtime config reproduced the security-relevant parts of the shipped `maddy.conf` exactly, changing only paths and ports so the experiment stayed self-contained:

- An authenticated implicit-TLS submission endpoint — in the default file, `submission tls://0.0.0.0:465` with mandatory `auth &local_authdb` (`maddy.conf:L93-L95`).
- A `source $(local_domains)` block whose `modify { sign_dkim $(primary_domain) default }` enables DKIM signing (`maddy.conf:L97-L99`).
- One `sql` module providing BOTH storage and auth over a single SQLite database — `sql local_mailboxes local_authdb { driver sqlite3 … dsn all.db }` (`maddy.conf:L32-L35`).
- `default_source { reject 501 5.1.8 "Non-local sender domain" }` for any non-local envelope sender (`maddy.conf:L117-L119`).
- Global vars `$(primary_domain) = example.org` (`maddy.conf:L9`) and `$(local_domains) = $(primary_domain)` (`maddy.conf:L12`).

A **plaintext `submission tcp://` endpoint with `insecure_auth`** was added *only* to make protocol bytes observable for R6; self-signed TLS material was generated for the implicit-TLS ports.

### DKIM key auto-generation on first run

The `sign_dkim` modifier's `key_path` default is `dkim_keys/{domain}_{selector}.key` (`internal/modify/dkim/dkim.go:L137`); generation and the emitted log occur at `internal/modify/dkim/dkim.go:L182-L195`. On first start the server logged (verbatim):

```
sign_dkim: generating a new rsa2048 keypair...
sign_dkim: generated a new rsa2048 keypair, private key is in dkim_keys/example.org_default.key, TXT record with public key is in dkim_keys/example.org_default.dns,
put its contents into TXT record for default._domainkey.example.org to make signing and verification work
```

so the key materialized at `dkim_keys/example.org_default.key` (with a companion `.dns` TXT record), exactly as the `key_path` default predicts.

### Two accounts provisioned

Accounts were created with `maddyctl users create` (command tree: `cmd/maddyctl/main.go:L47` `"users"` → `L71` `"create"`; syntax modeled on `docs/tutorials/setting-up.md:L132` `$ maddyctl users create postmaster@example.org`). The command prints nothing on success and exits 0:

```
$ maddyctl --config maddy-test.conf users create alice@example.org --password ...   # exit 0, no output
$ maddyctl --config maddy-test.conf users create bob@example.org   --password ...   # exit 0, no output
$ maddyctl --config maddy-test.conf users list
alice@example.org
bob@example.org
```

### Startup banner (verbatim, `maddy -debug`)

```
[debug] sql: go-imap-sql version 0.4.0
[debug] submission: authentication provider: sql local_mailboxes
submission: listening on tls://0.0.0.0:4650
submission: listening on tcp://0.0.0.0:4655
submission: authentication over unencrypted connections is allowed, this is insecure configuration and should be used only for testing!
imap: listening on tls://0.0.0.0:4930
```

### EHLO capability list (verbatim, plaintext endpoint)

```
220 example.org ESMTP Service Ready
250-Hello client.test
250-PIPELINING
250-8BITMIME
250-ENHANCEDSTATUSCODES
250-STARTTLS
250-AUTH PLAIN
250-SMTPUTF8
250 SIZE 33554432
```

On the implicit-TLS endpoint (`tls://…:4650`) the `250-STARTTLS` line is **absent** (the channel is already encrypted).

### Rationale

A single `sql` instance underlies both `local_mailboxes` and `local_authdb`: the module registers once as `module.Register("sql", New)` (`internal/storage/sql/sql.go:L424`) and exposes credential checking (`CheckPlain`, consumed by the submission endpoint's `Login`) alongside mailbox storage; the auth interfaces live in `internal/auth/auth.go`. This is why **one** backend both authenticates the SMTP session and stores the delivered mail — the authenticated identity and the delivery target are validated against the same user table.

---

## R2 & R3 — Boundary tests and the accept/reject logic (the exact wire responses)

The three sender constructions were driven over a live SMTP session (Python `smtplib`-style raw-socket client with a `send:`/`reply:` trace). The user's constructions are preserved **verbatim** below, each with its **measured** outcome.

1. *"authenticate as one user but attempt to send with a different user's address in MAIL FROM"* → authenticate as `alice@example.org`, `MAIL FROM:<bob@example.org>` → **ACCEPTED** → `250 2.0.0 OK: queued`.
2. *"send with a MAIL FROM domain that doesn't match any configured signing domain"* → `MAIL FROM:<user@notlocal.test>` (a non-local domain) → **REJECTED** → `501 5.1.8 Non-local sender domain`.
3. *"send a legitimate message that should be properly signed"* → authenticate as `alice`, `MAIL FROM:<alice@example.org>`, `From: alice@example.org` → **ACCEPTED** (and DKIM-signed; see R4/R5).

### Accept/reject matrix (verbatim wire responses)

| Scenario | Auth identity | MAIL FROM / condition | Verbatim wire response | Mechanism (citation) |
|----------|---------------|-----------------------|------------------------|----------------------|
| Unauthenticated `MAIL FROM` | *(none)* | `MAIL FROM:<alice@example.org>` before AUTH | `502 5.7.0 Please authenticate first` | Submission forces auth: `authAlwaysRequired = true` (`smtp.go:L589-L593`); `AnonymousLogin` → `smtp.ErrAuthRequired` (`smtp.go:L663`); library text `errors.New("Please authenticate first")` (`go-smtp/backend.go:9`) surfaced as `502` via the non-`SMTPError` fallback (`go-smtp/conn.go:270`) |
| Bad credentials | *(bad login)* | `AUTH PLAIN` with wrong password | `454 4.7.0 Invalid credentials` | `Login` → `if !endp.Auth.CheckPlain(...) { … errors.New("Invalid credentials") }` (`smtp.go:L653-L655`), log `"authentication failed"` (`smtp.go:L654`); surfaced as `454` via `go-smtp/conn.go:438` |
| **#1 same-domain impersonation** | `alice@example.org` | `MAIL FROM:<bob@example.org>` | `250 2.0.0 OK: queued` — **ACCEPTED** | Source block matched at **domain** level; local-part never compared to auth user (`msgpipeline.go:L155-L199`) |
| **#2 non-local domain** | `alice@example.org` | `MAIL FROM:<user@notlocal.test>` | `501 5.1.8 Non-local sender domain` (at `RCPT TO`) | Falls through to `default_source` reject (`maddy.conf:L117-L119`); `rejectErr` returned in the sender phase (`msgpipeline.go:L117-L119`) |
| **#3 legitimate signed** | `alice@example.org` | `MAIL FROM:<alice@example.org>`, `From: alice@example.org` | `250 2.0.0 OK: queued` — **ACCEPTED & signed** | Domain match + `shouldSign` all checks pass (`dkim.go:L264-L330`) |
| Missing `From` header | `alice@example.org` | valid envelope, DATA lacks `From:` | `554 5.6.0 Message does not contains a From header field` *(source-derived, see note)* | `submissionPrepare` (`submission.go:L39-L48`, message text at `:L43`, grammatical quirk "contains" preserved) |

> **Source-derived (not observed) entries.** The missing/invalid/malformed-header rows quote the **exact source string** with its `file:line`, clearly labeled — they were not separately captured in the transcript set. All other rows are **observed** verbatim wire lines. Additional header-fault strings from the same routine: `554 5.6.0 "Invalid address in <hdr>"` for a bad `Sender`/`To`/`Cc`/`Bcc`/`Reply-To` (`submission.go:L50-L81`), `554 5.6.0 "Invalid address in From"` (`submission.go:L83-L95`), `554 5.6.0 "Missing Sender header field"` (`submission.go:L99-L109`), and `554 "Malformed Date header"` **with no enhanced code** (`submission.go:L111-L123`).

### The core answer: acceptance is **domain-scoped**, not user-scoped

On an authenticated session, whether a sender address is accepted is decided by **MAIL FROM *domain* membership**, **not** by the authenticated user. The routing engine's `srcBlockForAddr` matches a source block by envelope sender with priority **full-address → domain → default**:

- full-address lookup `dd.d.perSource[cleanFrom]` (`internal/msgpipeline/msgpipeline.go:L171`),
- then domain lookup `dd.d.perSource[domain]` (`:L190`),
- else `dd.d.defaultSource` (`:L193`) — whole function `internal/msgpipeline/msgpipeline.go:L155-L199`.

The matched block's `rejectErr` is returned in the **MAIL FROM (sender) phase** at `internal/msgpipeline/msgpipeline.go:L117-L119`. Because the default config defines only a **domain-level** `source $(local_domains)` block plus a `default_source` reject — and **never compares the MAIL FROM local-part to the authenticated user** — same-domain impersonation matches the domain block and is accepted, while a non-local domain falls through to `default_source`.

The server's own debug log makes the domain-level match explicit (verbatim):

```
[debug] smtp/pipeline: sender bob@example.org matched by domain rule 'example.org'	{"msg_id":"7a820cc8"}
```

and for the non-local case:

```
[debug] smtp/pipeline: sender user@notlocal.test matched by default rule	{"msg_id":"67e1552f"}
```

### Reply-timing nuance — the `501` is **deferred to `RCPT TO`**

`defer_sender_reject` (default `true`) defers a sender rejection to the `RCPT TO` phase to limit information leakage. The transcript confirms this precisely: the non-local `MAIL FROM` is first **accepted** with a `250`, and the `501` only appears **after `RCPT TO`** (verbatim client/server trace):

```
send: 'MAIL FROM:<user@notlocal.test>\r\n'
reply: '250 2.0.0 Roger, accepting mail from <user@notlocal.test>\r\n'
send: 'RCPT TO:<alice@example.org>\r\n'
reply: '501 5.1.8 Non-local sender domain\r\n'
```

The matching server-side debug log:

```
submission: RCPT error	{"effective_rcpt":"alice@example.org","rcpt":"alice@example.org","reason":"reject directive used","smtp_code":501,"smtp_enchcode":"5.1.8","smtp_msg":"Non-local sender domain"}
submission: aborted	{"msg_id":"67e1552f"}
```

For contrast, the accepted legitimate message ends with the queued acknowledgment (verbatim):

```
send: '.\r\n'
reply: '250 2.0.0 OK: queued\r\n'
```

### Rationale

Authentication is mandatory on the submission endpoint (`smtp.go:L589-L593`), but once the session is authenticated, sender *authorization* is a pure routing decision keyed on the envelope domain. There is no per-user `source` block and no local-part comparison anywhere in `srcBlockForAddr` — so the security boundary the default config actually enforces is "the envelope sender's domain must be local," nothing finer.

---

## R4 — Actual stored headers (quoted verbatim, not summarized)

For an accepted/delivered message the raw RFC822 was fetched from `alice`'s `INBOX` (via IMAP `FETCH` over `imap tls://…:4930`, cross-checked against the on-disk store read with Python's `sqlite3` module — `maddyctl imap-msgs` at `cmd/maddyctl/main.go:L307` is the equivalent CLI path). The stored headers are quoted **exactly** below.

### The legitimate, signed message (Scenario 3) — full stored header block

```
Delivered-To: alice@example.org
Return-Path: <alice@example.org>
Dkim-Signature: a=rsa-sha256;
 bh=PaB//6aGE2gmBdiR2saRJtAdGopzAmohJeW/w1DOtQc=; c=relaxed/relaxed;
 d=example.org;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:MIME-Version:Content-Type:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=alice@example.org; s=default; t=1782883915; v=1; x=1783315915; b=LTbnnqOlgHGaQDfiqcL5wLc8VeRUYUtxoKmR5wkLd+cGymJ8bYwuldlKmglORu3H1VkpBfWYmIYoQp8hPyrmAlqZ8rh03SXDkRW0UVGGsYIvKePJy8qoHJFLN8QEb93VU1DUfDrjNjTiKteNh91d+YojlMeN2UbdkNEtSr5DbAOqHib9dd4ujcHyrlh9JSCoXVF3kN8kIeg4WfEy8pN2RHCwPW3jzbBdEYFRMUOtmozqy2dQD15NdHNNykL2ziO4KbG6fGbmB2HqVeMVjrs7m9BsrxUTDVtONaUnC4DeiaUSpFULLvGkckBjv0IJm5hltHs4rCKlrqJGHOeBQ/YxyA==;
Received:  by example.org (envelope-sender <alice@example.org>) with ESMTP id 098774b5; Wed, 01 Jul 2026 05:31:55 +0000
Date: Wed, 1 Jul 2026 05:31:55 +0000
Message-Id: <46bf47e9-5bc8-4bcc-a36e-48ccc6020814@example.org>
From: alice@example.org
To: alice@example.org
Subject: Scenario3 legitimate signed
MIME-Version: 1.0
Content-Type: text/plain
```

### `DKIM-Signature` — observations

- The full `b=` and `bh=` base64 values above are reproduced **character-for-character** as stored.
- **The header is stored as `Dkim-Signature:`**, not `DKIM-Signature:` — the storage layer (`go-imap-sql`) re-cased the field name on write. This exact re-casing matters for R5 (it is one of the two reasons the signature does not verify).
- Tags appear in **alphabetical order** (`a`, `bh`, `c`, `d`, `h`, `i`, `s`, `t`, `v`, `x`, `b`) — `v=1` is **not** first.
- The `h=` tag contains **`From:From`** (doubled), i.e. `From` is **over-signed**, so a verifier sees `From` covered by the signature. This follows from `From` being present in `oversignDefault` (`internal/modify/dkim/dkim.go:L30-L38`, `From` at `:L37`). (Over-signing lists each name twice so that adding a second instance of that header post-signing breaks the signature.)

### `Received` — client-origin stripping and the double-space quirk

Quoted exactly (note the **double space** after `Received:` and the **absence of any `from <client>` clause**):

```
Received:  by example.org (envelope-sender <alice@example.org>) with ESMTP id 098774b5; Wed, 01 Jul 2026 05:31:55 +0000
```

The shape is `Received:  by <hostname> (envelope-sender <…>) with <PROTO> id <id>; <RFC1123Z date>`. The client origin is stripped because the submission path sets `msgMeta.DontTraceSender = true` (`internal/endpoint/smtp/submission.go:L28`), and `GenerateReceived` gates the `from <client>` clause on `!msgMeta.DontTraceSender && (…SMTP/LMTP…)` (`internal/target/received.go:L30-L31`). With that clause omitted, the assembled value **begins with a leading space** (` by …`, written as `builder.WriteString(" by ")` at `received.go:L62`), which serializes to the visible **double space** after `Received:`.

A companion message submitted over the **implicit-TLS** endpoint (port 4650) shows the protocol token differs — `with ESMTPS` instead of `ESMTP` — while the double-space/stripped-origin shape is identical (verbatim):

```
Received:  by example.org (envelope-sender <alice@example.org>) with ESMTPS id a69c6a16; Wed, 01 Jul 2026 05:47:29 +0000
```

This matches the endpoint setting the session protocol to `ESMTPS` when TLS is active vs `ESMTP` otherwise (`internal/endpoint/smtp/smtp.go`, `newSession`).

### `Message-Id` and `Date` — inserted by the submission modifier

Both were **inserted by Maddy** because the client omitted them. Message-ID insertion is at `internal/endpoint/smtp/submission.go:L30-L37`; Date insertion at `:L124-L127`. The stored values (verbatim):

```
Date: Wed, 1 Jul 2026 05:31:55 +0000
Message-Id: <46bf47e9-5bc8-4bcc-a36e-48ccc6020814@example.org>
```

Note the inserted `Date` uses a **non-zero-padded day** (`Wed, 1 Jul`) — a direct consequence of the Go layout `"Mon, 2 Jan 2006 15:04:05 -0700"` used at `submission.go:L126` — whereas the `Received` header's own date is RFC1123Z with a **zero-padded** day (`Wed, 01 Jul`). The two dates in the same message are formatted differently.

### `Authentication-Results` — explicitly **ABSENT**

There is **no** `Authentication-Results` header on the submitted/locally-delivered mail — confirmed by grepping every stored message (zero matches). This is expected: that header is produced only by **inbound checks** — `internal/check/dkim/dkim.go:L115-L187` builds an `authres.DKIMResult` and `internal/check/spf/spf.go:L117-L156` builds an `authres.SPFResult` — and the **submission path has no `check{}` block** (contrast the port-25 `smtp` pipeline's `check { … }` at `maddy.conf:L53-L66`). Submission signs outbound mail; it does not verify it, so it writes no authentication-results.

### Rationale

Everything Maddy adds on the submission path is either a *signature over* the message (DKIM) or *trace/normalization* metadata (`Received`, and the inserted `Message-Id`/`Date`). It deliberately does **not** record the client's IP in `Received` (privacy for authenticated submitters, via `DontTraceSender`) and does **not** emit `Authentication-Results` (there is nothing to verify on a message the server itself is originating).

---

## R5 — DKIM under a `From` mismatch

The user's DKIM construction is preserved **verbatim**:

> *"send a message that should trigger DKIM signing but use a From header that doesn't match the authenticated user or the signing domain. Observe whether the signature is still applied, whether it covers the From header, and what a recipient verifying this message would actually see."*

This was implemented as Scenario 4: authenticate as `alice@example.org`, `MAIL FROM:<alice@example.org>`, but `From: mallory@otherdomain.test` — a `From` domain that differs from the key domain `example.org`.

### Observed outcome: signing is **silently skipped**, the message is delivered **UNSIGNED — not rejected**

The message was **accepted** (`250 2.0.0 OK: queued`) and stored, but with **no** `DKIM-Signature` header. The server's debug log states exactly why (verbatim):

```
sign_dkim: not signing, From domain is not key domain	{"from_domain":"otherdomain.test","key_domain":"example.org","msg_id":"e68d904d"}
```

This is the branch `if !dns.Equal(fromDomain, m.domain) { … return "", false }` in `shouldSign()` (`internal/modify/dkim/dkim.go:L287-L291`). `RewriteBody` then does `id, ok := s.m.shouldSign(...); if !ok { return nil }` — returning **without** adding a signature (`internal/modify/dkim/dkim.go:L340-L351`, silent-skip at `:L349-L351`). There is **no rejection path here**: a signing failure never becomes an SMTP error.

The stored Scenario 4 message confirms the absence (verbatim header block):

```
Delivered-To: alice@example.org
Return-Path: <alice@example.org>
Received:  by example.org (envelope-sender <alice@example.org>) with ESMTP id e68d904d; Wed, 01 Jul 2026 05:31:56 +0000
Date: Wed, 1 Jul 2026 05:31:56 +0000
Message-Id: <03f5fed1-edd4-4f3c-b962-e6694600f1ce@example.org>
From: mallory@otherdomain.test
To: alice@example.org
Subject: Scenario4 DKIM From-mismatch
MIME-Version: 1.0
Content-Type: text/plain
```

### Both non-signing branches were observed

There are two distinct reasons `shouldSign` can decline, and **both** were captured:

- **`From` domain ≠ key domain** (Scenario 4, above): `"not signing, From domain is not key domain"` (`dkim.go:L287-L291`).
- **`From` ≠ authenticated identity** (Scenario 1: auth `alice`, `From: bob@example.org` — same domain, different user): the message was still **accepted and delivered unsigned**, with the log:

```
sign_dkim: not signing, From address is not authenticated identity	{"auth_id":"alice@example.org","from_addr":"bob@example.org","msg_id":"7a820cc8"}
```

This is the `auth` check at `dkim.go:L299-L310`. It confirms that the **same-domain impersonation** that R2/R3 showed is *accepted at the SMTP layer* is *also not signed* — the two mechanisms are independent.

### When signing DOES occur, `From` is covered (over-signed)

For the legitimate message the log confirms signing succeeded:

```
[debug] sign_dkim: signed	{"identifier":"alice@example.org"}
```

and, as shown in R4, the emitted `h=` tag lists `From:From`. `From` is in `oversignDefault` (`internal/modify/dkim/dkim.go:L30-L38`, `From` at `:L37`), so a verifier sees `From` covered.

### `require_sender_match` gates SIGNING, never ACCEPTANCE

The `require_sender_match` policy — default `["envelope","auth"]` (`internal/modify/dkim/dkim.go:L151-L152`), envelope check at `:L293-L297`, auth check at `:L299-L310` — influences **only** `shouldSign()`. A mismatch suppresses the signature; it **never** rejects the message. The manual describes it the same way: *"Require specified identifiers to match From header field and key domain, otherwise - don't sign the message."* (`docs/man/maddy-filters.5.scd:L521-L522`).

### What a verifying recipient actually sees: **DKIM `FAIL`, for two independent reasons**

The most consequential run-first finding: **even the *legitimate* signed message does not verify.** This was established with the **authoritative** verifier — `github.com/emersion/go-msgauth/dkim`, the *same library Maddy signs with* — built into a tiny containerized checker that injected the generated public key via `go-mockdns` and called `dkim.Verify` on the stored message.

The generated public-key TXT record (`dkim_keys/example.org_default.dns`, verbatim):

```
v=DKIM1; k=rsa; p=MIIBCgKCAQEAvP4z7qigv4UE08+/FkOz9g1WO7Unf/+IdJF6TJmkAwgs8/sf7/bRDhkU+Zzd7d+UWUP998EkAoqKZJWQkFDL4Hj6VqcgxhdkrUB6kpX7M10bCRZAXNMZHH+lzbgWaNsc2GZw/dhgCfsBskBR+MuzN31C276UKtwkCY+ko/SP1+M40W3DkIb3lWnlhjtPwL0ppDxi6DHxtFFNYRSrWROJMwox2UoeO8teikJd2C5XzX3Q+w6DT8XBLSw12mzg1EahZozCXn2shgMV5qmSBbx9QjpYps7JbKjkByqRk1oRq0EFR3kqs91MuhqFT4LrS3ZhzpdaULp0AmBDK99lNlgWfQIDAQAB
```

**Defect A — PKCS#1 vs PKIX public-key encoding.** The `p=` value decodes to a DER blob beginning `3082010a0282010100…` = `SEQUENCE { INTEGER(257 bytes), … }`, i.e. a **bare PKCS#1 `RSAPublicKey`**. A proper DKIM record carries a **SubjectPublicKeyInfo (PKIX/SPKI)** structure, which would begin `MIIBIjANBgkqhkiG9w0BAQEF…`. Maddy writes the key with `x509.MarshalPKCS1PublicKey` (`internal/modify/dkim/keys.go:L143`), but go-msgauth's verifier parses it with `x509.ParsePKIXPublicKey`. The verifier's verdict on Maddy's own record (verbatim, identical for both bare-LF and CRLF message variants):

```
FAIL: dkim: key syntax error: x509: failed to parse public key (use ParsePKCS1PublicKey instead for this key format)
```

**Defect B — post-signing re-serialization.** To prove the failure is not *only* the key encoding, the same key was re-encoded as proper SPKI and the verification retried. It **still failed** — now at the signature check itself (verbatim, both variants):

```
FAIL: dkim: signature did not verify: crypto/rsa: verification error
```

An independent Python reproduction isolated the cause: the **body hash** recomputes and **matches** the stored `bh=` value (`PaB//6aGE2gmBdiR2saRJtAdGopzAmohJeW/w1DOtQc=`, `MATCH=True`), so the body canonicalization is fine — but the **header hash** fails, consistent with the storage layer re-serializing the header set (notably re-casing `DKIM-Signature` → `Dkim-Signature`, as shown in R4) so the delivered header bytes differ from the bytes that were signed.

### Rationale and the bottom line for R5

- **Is the signature still applied under a `From` mismatch?** No — it is silently skipped and the message is delivered **unsigned** (`dkim.go:L287-L291`, `:L340-L351`). No rejection occurs.
- **Does the signature cover `From` when present?** Yes — `From` is over-signed (`dkim.go:L37`), so `From` appears in `h=`.
- **What would a verifying recipient see?** For the legitimate message: **DKIM `FAIL`**, not `pass` — first a key-parse `permerror` against strict verifiers (Defect A), and even with a corrected key a `crypto/rsa: verification error` (Defect B). "Signed" is not the same as "verifiable"; the recipient's verifier is the source of truth, and here it rejects the signature.

---

## R6 — SMTP transaction capture (methods and verbatim transactions)

### Why raw `tcpdump` is not viable for the default endpoint

The default submission endpoint uses **implicit TLS on port 465**, so a packet sniffer sees only encrypted, opaque payload bytes. Byte-level capture of the *plaintext* protocol therefore requires either client-side tracing inside the TLS session or a temporary plaintext endpoint. That the default endpoint is genuinely implicit-TLS was confirmed directly (verbatim):

```
$ echo Q | openssl s_client -connect 127.0.0.1:4650 -quiet
...(TLS handshake)...
220 example.org ESMTP Service Ready
```

The `220` banner arrives **only after** the TLS handshake completes — proof the endpoint speaks TLS immediately, with no `STARTTLS` step.

### Methods actually used

1. **Client-side protocol tracing over the encrypted channel** — a Python raw-socket client wrapped in `ssl` (equivalent to `smtplib.SMTP_SSL(...).set_debuglevel(1)`; `swaks --tls-on-connect` / `openssl s_client -connect host:465` are interchangeable). This prints each `send:`/`reply:` line.
2. **Maddy `debug` logging** to expose pipeline decisions — the msgpipeline "matched by … rule" lines (`internal/msgpipeline/msgpipeline.go:L194,L196,L199`) and the `sign_dkim: not signing …` lines.
3. **The documented fallback** — a temporary **plaintext `submission tcp://` endpoint with `insecure_auth`** in the ephemeral config, which makes the exact protocol bytes observable (and would allow a `tcpdump` byte capture). The startup banner explicitly warned about it: `submission: authentication over unencrypted connections is allowed, this is insecure configuration and should be used only for testing!`

The producing client (abridged to the essentials):

```python
import socket, base64
s = socket.create_connection(("127.0.0.1", 4655))
def cmd(line):
    s.sendall(line + b"\r\n"); print("send:", line); 
    data = s.recv(4096); print("reply:", data)
cmd(b"EHLO client.test")
authz = b"\x00alice@example.org\x00AlicePass123"
cmd(b"AUTH PLAIN " + base64.b64encode(authz))
cmd(b"MAIL FROM:<alice@example.org>")
cmd(b"RCPT TO:<alice@example.org>")
cmd(b"DATA")
s.sendall(b"From: alice@example.org\r\nTo: alice@example.org\r\n"
          b"Subject: Scenario3 legitimate signed\r\n\r\n"
          b"Hello from scenario 3, this should be DKIM signed.\r\n.\r\n")
print("reply:", s.recv(4096))
cmd(b"QUIT")
```

### Verbatim ACCEPTED transaction (Scenario 3 — the legitimate, signed message)

```
send: b'EHLO client.test'
reply: b'250-Hello client.test\r\n250-PIPELINING\r\n250-8BITMIME\r\n250-ENHANCEDSTATUSCODES\r\n250-STARTTLS\r\n250-AUTH PLAIN\r\n250-SMTPUTF8\r\n250 SIZE 33554432\r\n'
send: b'AUTH PLAIN AGFsaWNlQGV4YW1wbGUub3JnAEFsaWNlUGFzczEyMw=='
reply: b'235 2.0.0 Authentication succeeded\r\n'
send: b'MAIL FROM:<alice@example.org>'
reply: b'250 2.0.0 Roger, accepting mail from <alice@example.org>\r\n'
send: b'RCPT TO:<alice@example.org>'
reply: b"250 2.0.0 I'll make sure <alice@example.org> gets this\r\n"
send: b'DATA'
reply: b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
reply: b'250 2.0.0 OK: queued\r\n'
send: b'QUIT'
reply: b'221 2.0.0 Goodnight and good luck\r\n'
```

Matching server-side debug log (verbatim):

```
[debug] smtp/pipeline: sender alice@example.org matched by domain rule 'example.org'	{"msg_id":"098774b5"}
submission: adding missing Message-ID
submission: adding missing Date header
[debug] sign_dkim: signed	{"identifier":"alice@example.org"}
submission: accepted	{"msg_id":"098774b5"}
```

### Verbatim REJECTED transaction (Scenario 2 — the non-local domain)

```
send: b'AUTH PLAIN AGFsaWNlQGV4YW1wbGUub3JnAEFsaWNlUGFzczEyMw=='
reply: b'235 2.0.0 Authentication succeeded\r\n'
send: b'MAIL FROM:<user@notlocal.test>'
reply: b'250 2.0.0 Roger, accepting mail from <user@notlocal.test>\r\n'
send: b'RCPT TO:<alice@example.org>'
reply: b'501 5.1.8 Non-local sender domain\r\n'
send: b'QUIT'
reply: b'221 2.0.0 Goodnight and good luck\r\n'
```

Matching server-side debug log (verbatim):

```
[debug] smtp/pipeline: sender user@notlocal.test matched by default rule	{"msg_id":"67e1552f"}
submission: RCPT error	{"effective_rcpt":"alice@example.org","rcpt":"alice@example.org","reason":"reject directive used","smtp_code":501,"smtp_enchcode":"5.1.8","smtp_msg":"Non-local sender domain"}
submission: aborted	{"msg_id":"67e1552f"}
```

### Rationale

The two transactions above bracket the acceptance logic: the accepted one shows the full `EHLO → AUTH → MAIL → RCPT → DATA → 250 queued` happy path with the server inserting `Message-ID`/`Date` and signing; the rejected one shows the `501 5.1.8` surfacing only **after `RCPT TO`** (the `defer_sender_reject` behavior from R3). The client trace and the server debug log agree line-for-line, which is why client-side tracing plus `debug` logging is a complete substitute for a raw packet capture that TLS would render unreadable anyway.

---

## R7 — Default-policy conclusion (with a ruled-out interpretation)

### Conclusion

**Maddy's default configuration enforces sender alignment only at the DOMAIN level — NOT per authenticated user.** Authentication is mandatory (`internal/endpoint/smtp/smtp.go:L589-L593`), but once the session is authenticated, the envelope sender is authorized purely by **domain membership** via source-block matching (`internal/msgpipeline/msgpipeline.go:L155-L199`). The authenticated username is captured and propagated (`AuthUser: username`, `internal/endpoint/smtp/smtp.go:L680`) but is **never compared** to `MAIL FROM` in the acceptance path. Explicit per-user policy would require adding a per-user `source` block or a check the default config does not contain.

### The plausible-but-incorrect interpretation, ruled out by evidence

A reasonable but **wrong** reading is: *"an authenticated session ⇒ MAIL FROM must equal the authenticated user."* The runtime evidence refutes this directly:

- Scenario 1 authenticated as `alice@example.org` and sent `MAIL FROM:<bob@example.org>` — a **different user, same domain** — and the server **accepted** it: `250 2.0.0 OK: queued`.
- The debug log shows the match was made purely on domain: `sender bob@example.org matched by domain rule 'example.org'`.
- There is **no per-user source block** and **no local-part comparison** anywhere in `srcBlockForAddr` (`msgpipeline.go:L155-L199`).

So the per-user binding is an *assumption*, not something the default config does.

### Standards framing (grounds the conclusion)

This behavior is **standards-permissible**. RFC 6409 ("Message Submission for Mail") §4.3 makes authentication mandatory by default, but §6.1 "Enforce Submission Rights" — rejecting a `MAIL FROM` whose address "has insufficient submission rights or is not authorized with the authentication used" — is an **optional ("MAY")** behavior, to be answered with `550` and an enhanced code such as `5.7.1`. RFC 4954 defines the SASL `AUTH` extension that underpins the authenticated session. A submission server that authenticates the session but does **not** bind `MAIL FROM` to the user is therefore conforming — the per-user binding is an *opt-in policy*, not a protocol requirement. Maddy's default simply does not opt in; it enforces the coarser, still-useful "local-domain-only" boundary instead.

### Rationale

The design cleanly separates three concerns: **authentication** (mandatory, `smtp.go:L589-L593`), **sender authorization** (domain-scoped routing, `msgpipeline.go:L155-L199`), and **outbound authentication of the message** (DKIM signing, gated by `require_sender_match`). None of the three, in the default config, ties the envelope local-part to the logged-in user — which is exactly why Scenario 1 is accepted and why the correct answer to "does the default enforce per-user sender alignment?" is **no**.

---

## R8 — Config-vs-runtime divergences (three, each tied to observation)

### Divergence 1 — the anti-spoofing comment promises more than the code delivers

The default config comments the `default_source` reject as an anti-spoofing measure:

```
# maddy.conf:L115-L116
# Prevent local senders from using non-local sender addresses since
# this is likely a spoofing attempt.
# maddy.conf:L117-L119
default_source {
    reject 501 5.1.8 "Non-local sender domain"
}
```

A reader could infer this "prevents spoofing." Runtime shows it blocks only **non-local domains**; it does **nothing** about same-domain user impersonation. Scenario 1 (auth `alice`, `MAIL FROM:<bob@example.org>`) was **accepted** (`250 2.0.0 OK: queued`). The comment invites a stronger anti-spoofing reading than the code delivers — the actual guarantee is "sender domain must be local," not "sender must be you."

### Divergence 2 — `require_sender_match`: documentation vs. code enum mismatch (verified empirically)

The manual documents the valid values as `off` / `envelope` / `auth` with default `envelope auth`:

```
# docs/man/maddy-filters.5.scd:L518-L519
*Syntax*: require_sender_match _ids..._ ++
*Default*: envelope auth
# docs/man/maddy-filters.5.scd:L531,L533,L535 — Valid values: off / envelope / auth
```

But the code's config enum is a **different set**:

```go
// internal/modify/dkim/dkim.go:L151-L152
cfg.EnumList("require_sender_match", false, false,
    []string{"envelope", "auth_domain", "auth_user", "off"},
    []string{"envelope", "auth"}, &r.senderMatch)
```

This produces two concrete, testable contradictions, both confirmed at runtime:

- **The documented value `auth` FAILS enum validation.** Starting maddy with `require_sender_match auth` in a `sign_dkim` block aborts at load time (verbatim):

```
/tmp/maddy-exp/enum-test.conf:18: invalid argument, valid values are: [envelope auth_domain auth_user off]
```

  Note the irony: `auth` is the **default** (and IS honored by `shouldSign` at `dkim.go:L299`), yet it cannot be set **explicitly** because it is absent from the validator's allowed list.

- **`auth_domain` / `auth_user` PASS validation but are inert.** Starting maddy with `require_sender_match auth_user` loaded successfully (the endpoint came up and listened), yet these tokens are **never consulted** — `shouldSign` honors only `envelope` (`dkim.go:L293`), `auth` (`:L299`), and `off`. They are accepted-but-dead configuration.

### Divergence 3 — DKIM is configured and a signature IS emitted, yet it fails to verify

A reasonable config reading is "`sign_dkim` on ⇒ recipients get `dkim=pass`." Runtime contradicts it. As shown in R5, the legitimate message was signed (`sign_dkim: signed`), a full `DKIM-Signature` was stored, and yet the authoritative go-msgauth verifier returned:

```
FAIL: dkim: key syntax error: x509: failed to parse public key (use ParsePKCS1PublicKey instead for this key format)
```

and, even after correcting the key encoding:

```
FAIL: dkim: signature did not verify: crypto/rsa: verification error
```

So "DKIM configured" does **not** imply "recipients see `dkim=pass`" — the emitted signature is unverifiable for two independent reasons (key-encoding at `keys.go:L143`; post-signing re-serialization by the storage layer).

### Bonus divergence — reply codes: spec-idealized vs. library-actual

A spec-level reading predicts unauthenticated `MAIL` → `530` and bad credentials → `535`. Maddy + go-smtp actually return **`502 5.7.0 Please authenticate first`** and **`454 4.7.0 Invalid credentials`** — these are the go-smtp library's fallback branches (`go-smtp/conn.go:270` for `502`, `:438` for `454`), not the RFC-idealized codes. A reader expecting `530`/`535` from the SMTP standards would be surprised by the observed `502`/`454`.

### Rationale

Each divergence is a place where the *documentation or configuration surface* suggests one behavior while the *running code* does another — precisely the class of finding the investigation targets. The most security-relevant is #1 (the anti-spoofing comment overstates the guarantee); the most operationally surprising is #2 (the documented default value is unsettable); and the most consequential for real mail flow is #3 (a signature that never verifies).

---

## Closing — Artifact inventory & cleanup confirmation

### Ephemeral artifacts created OUTSIDE the committed tree (and removed afterward)

All of the following were created under `/tmp/maddy-exp` (never inside the repository) and deleted after the experiment:

- the ephemeral test config (modeled on `maddy.conf`, adding a plaintext `submission tcp://` endpoint with `insecure_auth`);
- self-signed TLS certificate and private key (for the implicit-TLS ports 4650/4930);
- the auto-generated DKIM key `dkim_keys/example.org_default.key` and its companion `dkim_keys/example.org_default.dns`;
- the SQLite store `all.db` (plus its journal/WAL and the external message blobs);
- observation scripts (Python `smtplib`/`imaplib`/`ssl`/`sqlite3`, the go-msgauth verifier module) and captured transcripts/logs.

### Cleanup confirmation

Throughout the experiment the repository working tree remained pristine — `git status --porcelain` produced **no output** (empty) at every check, and no `all.db`, `dkim_keys/`, or `*.dns` artifact ever leaked into the repository (they resolve against Maddy's state directory, which was redirected into `/tmp/maddy-exp`). After this document is written, the repository contains exactly **one** new file and nothing else:

```
$ git status --porcelain
?? blitzy/documentation/maddy_26452dd8dd78.md
```

No file in the Maddy source tree was modified, added, or deleted.

### Coverage pass — every sub-part answered

| Req | What it asked | Section that answers it |
|-----|---------------|--------------------------|
| **R1** | Build/run with auth submission, DKIM signing, local delivery, ≥2 accounts | **R1 — Environment & Setup** |
| **R2** | Drive the three verbatim sender constructions | **R2 & R3** (constructions preserved verbatim) |
| **R3** | Accept/reject logic from observation — exact codes, does outcome differ by construction | **R2 & R3** (matrix + core answer + timing) |
| **R4** | Actual stored headers — `DKIM-Signature`, `Received`, `Authentication-Results` verbatim | **R4 — Actual stored headers** |
| **R5** | DKIM under `From` mismatch — applied? covers `From`? what a verifier sees | **R5 — DKIM under a `From` mismatch** |
| **R6** | Capture ≥1 rejected and ≥1 accepted SMTP transaction; methods tried | **R6 — SMTP transaction capture** |
| **R7** | Default-policy conclusion + rule out a wrong interpretation | **R7 — Default-policy conclusion** |
| **R8** | ≥1 config-vs-runtime divergence, evidence-backed | **R8** (three + a bonus) |

All eight requirements are explicitly addressed; the user's three sender-boundary constructions and the DKIM `From`-mismatch construction are preserved verbatim in R2/R3 and R5 respectively.

---

## Appendix — Verified citation index

Every `file:line` below was verified against the source at commit `26452dd8dd787dc455278b0fdd296f4a5432c768`.

| Citation | Fact it grounds |
|----------|-----------------|
| `maddy.conf:L9` | `$(primary_domain) = example.org` |
| `maddy.conf:L12` | `$(local_domains) = $(primary_domain)` |
| `maddy.conf:L32-L35` | `sql local_mailboxes local_authdb { driver sqlite3 … dsn all.db }` |
| `maddy.conf:L53-L66` | inbound port-25 `check { … }` (absent on submission) |
| `maddy.conf:L93-L95` | `submission tls://0.0.0.0:465` + `auth &local_authdb` |
| `maddy.conf:L97-L99` | `source $(local_domains)` → `modify` → `sign_dkim $(primary_domain) default` |
| `maddy.conf:L115-L116` | anti-spoofing comment |
| `maddy.conf:L117-L119` | `default_source { reject 501 5.1.8 "Non-local sender domain" }` |
| `internal/endpoint/smtp/submission.go:L28` | `msgMeta.DontTraceSender = true` |
| `internal/endpoint/smtp/submission.go:L30-L37` | Message-ID insertion |
| `internal/endpoint/smtp/submission.go:L39-L48` | missing `From` → `554 5.6.0 "Message does not contains a From header field"` |
| `internal/endpoint/smtp/submission.go:L50-L95` | invalid `Sender`/`To`/…/`From` → `554 5.6.0 "Invalid address in …"` |
| `internal/endpoint/smtp/submission.go:L99-L109` | `554 5.6.0 "Missing Sender header field"` |
| `internal/endpoint/smtp/submission.go:L111-L127` | `554 "Malformed Date header"` (no enhanced code); Date insertion (fmt at `:L126`) |
| `internal/endpoint/smtp/smtp.go:L492` | `submission: modName == "submission"` |
| `internal/endpoint/smtp/smtp.go:L589-L593` | mandatory-auth (`authAlwaysRequired = true`) |
| `internal/endpoint/smtp/smtp.go:L653-L655` | `Login` → `errors.New("Invalid credentials")` |
| `internal/endpoint/smtp/smtp.go:L661-L663` | `AnonymousLogin` → `smtp.ErrAuthRequired` |
| `internal/endpoint/smtp/smtp.go:L680` | `AuthUser: username` (propagated, never compared for acceptance) |
| `go-smtp/backend.go:9` | `ErrAuthRequired = errors.New("Please authenticate first")` |
| `go-smtp/conn.go:270` | `WriteResponse(502, {5,7,0}, …)` |
| `go-smtp/conn.go:438` | `WriteResponse(454, {4,7,0}, …)` |
| `internal/msgpipeline/msgpipeline.go:L117-L119` | `rejectErr` returned in sender phase |
| `internal/msgpipeline/msgpipeline.go:L155-L199` | `srcBlockForAddr` (full-address `:L171` → domain `:L190` → default `:L193`) |
| `internal/target/received.go:L30-L31` | `!DontTraceSender` gate on the `from <client>` clause |
| `internal/target/received.go:L62` | leading-space ` by ` (double-space quirk) |
| `internal/modify/dkim/dkim.go:L30-L38` | `oversignDefault` (incl. `From` at `:L37`) |
| `internal/modify/dkim/dkim.go:L137` | `key_path` default `dkim_keys/{domain}_{selector}.key` |
| `internal/modify/dkim/dkim.go:L151-L152` | `require_sender_match` enum `[envelope,auth_domain,auth_user,off]`, default `[envelope,auth]` |
| `internal/modify/dkim/dkim.go:L182-L195` | key generation + log |
| `internal/modify/dkim/dkim.go:L287-L291` | `shouldSign` — "From domain is not key domain" |
| `internal/modify/dkim/dkim.go:L299-L310` | `shouldSign` — "From address is not authenticated identity" |
| `internal/modify/dkim/dkim.go:L340-L351` | `RewriteBody` silent-skip `if !ok { return nil }` |
| `internal/modify/dkim/keys.go:L143` | `x509.MarshalPKCS1PublicKey` (PKCS#1, not PKIX/SPKI) — DKIM verify defect root cause |
| `internal/check/dkim/dkim.go:L115-L187` | inbound `authres.DKIMResult` (Authentication-Results producer) |
| `internal/check/spf/spf.go:L117-L156` | inbound `authres.SPFResult` (Authentication-Results producer) |
| `internal/storage/sql/sql.go:L424` | `module.Register("sql", New)` |
| `internal/auth/auth.go` | auth interfaces (`CheckPlain`/`CheckDomainAuth`) |
| `cmd/maddyctl/main.go:L47,L71` | `users` / `create` |
| `cmd/maddyctl/main.go:L307` | `imap-msgs` (raw stored-message inspection) |
| `docs/man/maddy-filters.5.scd:L518-L535` | documented `require_sender_match` (default `envelope auth`; values `off`/`envelope`/`auth`) |
| `docs/tutorials/setting-up.md:L132` | `maddyctl users create postmaster@example.org` |
| `examples/multitentant-dkim.conf:L23-L26` | `source … { modify { sign_dkim <domain> default } }` |
| `go.mod:L3` | `go 1.13` (module lower bound) |
| `.build.yml:L11,L13` | `go build ./...` / `go test ./... -cover -race` |

### Key insights (woven throughout)

1. **Authorization is domain-scoped, not user-scoped** — the single fact that makes Scenario 1 "accepted" surprising and grounds R7.
2. **Header binding is syntactic only** — `submissionPrepare` checks existence/parseability of `From`, never comparing it to the authenticated user.
3. **DKIM "sender match" gates SIGNING, never ACCEPTANCE** — a `From`/envelope/auth mismatch yields an *unsigned* delivery, not a rejection.
4. **The observed unauth/bad-creds codes (`502 5.7.0` / `454 4.7.0`) are go-smtp library defaults**, not the RFC-idealized `530`/`535`.
5. **A signature can be present yet unverifiable** — "signed" ≠ "verifiable"; the recipient's verifier is the source of truth.
