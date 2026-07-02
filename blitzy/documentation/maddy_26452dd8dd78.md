# How maddy enforces sender identity and message authentication at runtime

**Repository:** `github.com/foxcpp/maddy` @ commit `26452dd8dd787dc455278b0fdd296f4a5432c768`
**Method:** built and run from source; every behavioral claim below is paired with the exact line observed on the wire, in the stored message, or in maddy's own logs. All runtime values (message IDs, DKIM timestamps, signature bytes) come from a single reproducible run and are quoted verbatim.

## TL;DR — the two-layer sender model
maddy does **not** have a single "sender identity" check. Two independent layers act on two different addresses:
1. **Envelope acceptance** — the `MAIL FROM` address. Decided by **domain-level message-pipeline source routing** + the config's `default_source`. There is **no per-account `MAIL FROM == authenticated-user` comparison** on the accept path.
2. **Author signing** — the `From:` header. Decided per-message by the DKIM `shouldSign` gate. A "do not sign" decision is **not** a rejection: the message is delivered **unsigned**.

## Reproduction harness (all under /tmp, repo untouched)
- cgo build (SQLite `sql` module): `CGO_ENABLED=1 GOFLAGS=-mod=mod go build ./cmd/maddy ./cmd/maddyctl` (`go.mod:3` declares `go 1.13`; built in compat mode).
- `/tmp/maddyrun` instance, `tls off`, `-debug`, combined `sql local_mailboxes local_authdb { driver sqlite3 ; dsn all.db }`.
- Two `submission` endpoints, both `insecure_auth yes` **solely to observe the wire** (a /tmp harness choice, not a change to maddy defaults): `:2525` default `sign_dkim example.org default`; `:2526` `sign_dkim example.org default { require_sender_match auth_user }`.
- Accounts `alice@example.org`, `bob@example.org` via `maddyctl users create -p`. First start auto-generated the DKIM key:
```
sign_dkim: generated a new rsa2048 keypair, private key is in dkim_keys/example.org_default.key, TXT record with public key is in dkim_keys/example.org_default.dns,
put its contents into TXT record for default._domainkey.example.org to make signing and verification work
```
confirming default new-key algorithm **rsa2048** (`internal/modify/dkim/dkim.go:150`).

| ID | Endpoint | Auth | MAIL FROM | From | Rcpt | Observed |
|----|----------|------|-----------|------|------|----------|
| A | :2525 | alice | alice@example.org | alice@example.org | bob | accepted + delivered + **signed** |
| B | :2525 | alice | bob@example.org | bob@example.org | alice | accepted + delivered + **NOT signed** |
| C | :2525 | alice | eve@notlocal.com | — | alice | `MAIL FROM` `250`, then **`RCPT` `501 5.1.8`** |
| D | :2525 | alice | alice@example.org | attacker@evil.com | bob | accepted + delivered + **NOT signed** |
| E | :2526 | alice | bob@example.org | bob@example.org | bob | accepted + delivered + **signed `i=bob@example.org`** |

## O1 — How maddy accepts/rejects a sender, and at what granularity
Acceptance is decided by **pipeline source routing at the domain level**, not by comparing `MAIL FROM` to the authenticated user. `startDelivery` records the cleaned sender as `msgMeta.OriginalFrom` and starts the pipeline with no From/AuthUser compare (`internal/endpoint/smtp/smtp.go:83-160`, assignment `:116`); the authenticated user is only *logged*:
```
submission: incoming message	{"msg_id":"7e3f7780","sender":"bob@example.org","src_host":"probe.local","src_ip":"127.0.0.1:46828","username":"alice@example.org"}
```
(Scenario B: `sender`=bob but `username`=alice; delivery proceeds.) The pipeline matches **full address → domain → default_source** (`internal/msgpipeline/msgpipeline.go:155-193`; `perSource[cleanFrom]` `:171` → `perSource[domain]` `:190` → `defaultSource` `:193`):
```
[debug] smtp/pipeline: sender alice@example.org matched by domain rule 'example.org'	{"msg_id":"46abfa3a"}
```
```
[debug] smtp/pipeline: sender eve@notlocal.com matched by default rule	{"msg_id":"3cd6abd7"}
```
The only envelope-level enforcement is **domain membership** via `default_source { reject 501 5.1.8 "Non-local sender domain" }` (`maddy.conf:117-119`). **Granularity: per-domain, not per-account.**

## O2 — Actual SMTP response codes/messages (verbatim) + full protocol dialogs
Both transactions below are the **complete** raw-socket dialogs (every client `C:` command and server `S:` response, from the `220` greeting through `QUIT`/`221`). The `AUTH PLAIN` base64 SASL token is redacted (it carries the password) and the message `DATA` payload is shown as a one-line summary (its stored form is dumped verbatim in **O3**); every SMTP command and every server response line is verbatim.

**Accepted (Scenario A, :2525) — greeting → EHLO → AUTH → MAIL → RCPT → DATA → QUIT:**
```
S: 220 example.org ESMTP Service Ready
C: EHLO probe.local
S: 250-Hello probe.local
S: 250-PIPELINING
S: 250-8BITMIME
S: 250-ENHANCEDSTATUSCODES
S: 250-AUTH PLAIN
S: 250-SMTPUTF8
S: 250 SIZE 33554432
C: AUTH PLAIN <base64(\0alice@example.org\0****)>
S: 235 2.0.0 Authentication succeeded
C: MAIL FROM:<alice@example.org>
S: 250 2.0.0 Roger, accepting mail from <alice@example.org>
C: RCPT TO:<bob@example.org>
S: 250 2.0.0 I'll make sure <bob@example.org> gets this
C: DATA
S: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
C: <DATA: From: alice@example.org; To: bob@example.org; Subject: Scenario-A-legit-signed; + body>
C: .
S: 250 2.0.0 OK: queued
C: QUIT
S: 221 2.0.0 Goodnight and good luck
```
**Rejected (Scenario C, :2525) — greeting → EHLO → AUTH → MAIL (250) → RCPT (501) → QUIT:**
```
S: 220 example.org ESMTP Service Ready
C: EHLO probe.local
S: 250-Hello probe.local
S: 250-PIPELINING
S: 250-8BITMIME
S: 250-ENHANCEDSTATUSCODES
S: 250-AUTH PLAIN
S: 250-SMTPUTF8
S: 250 SIZE 33554432
C: AUTH PLAIN <base64(\0alice@example.org\0****)>
S: 235 2.0.0 Authentication succeeded
C: MAIL FROM:<eve@notlocal.com>
S: 250 2.0.0 Roger, accepting mail from <eve@notlocal.com>
C: RCPT TO:<alice@example.org>
S: 501 5.1.8 Non-local sender domain (msg ID = 3cd6abd7)
C: QUIT
S: 221 2.0.0 Goodnight and good luck
```
- The accept strings `Roger, accepting mail from <…>` / `I'll make sure <…> gets this` / `OK: queued`, the greeting `Goodnight and good luck`, and the `354 … Go ahead` line originate in the **go-smtp dependency** (`go.mod:19`, `v0.12.1-0.20191206174923-1f576e0ec85c`), not maddy source.
- `Non-local sender domain` / `501 5.1.8` come from config `default_source` (`maddy.conf:118`); the ` (msg ID = 3cd6abd7)` suffix is appended by `internal/endpoint/smtp/smtp.go:437`.
- The multiline `250-…` EHLO capabilities (`PIPELINING`, `8BITMIME`, `ENHANCEDSTATUSCODES`, `AUTH PLAIN`, `SMTPUTF8`, `SIZE 33554432`) are advertised by the endpoint before authentication.

## O3 — Raw stored headers (verbatim) + a demonstrated absence
`DKIM-Signature` (Scenario A → bob), via `maddyctl imap-msgs dump bob@example.org INBOX 1` (reproduced exactly as stored, including header folding and the complete `b=` signature bytes):
```
Dkim-Signature: a=rsa-sha256; bh
 =BRbNLh8fl39kILyZlU0lQ5uq6kkwvLNaDjtXnOQNTco=; c=relaxed/relaxed;
 d=example.org;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=alice@example.org; s=default; t=1782954108; v=1; x=1783386108; b=OmTdAZeRBXHGNYYpwRiTAMZrJNw6QJxmQrU9dPsmfKn6ny+WZkI8kjYtj8p+J8vVrMpDFatd8/035HTWubcoXXwoTlLgc22qTp3OghWGsZD4UDolLs0VOe0zolxrUHYuwwqGHvLbYQsmgiROFQNo6lbLkxiqqyRkF0pl1Zjsq+8CBOm2Yd+DA4dp7hoU7q4FtFkWKCVVXeprJJhPOu+3nMjaVgZ3CwEiL3gPgrCMU391bDAEIeewZ4VC/L/x5UIqduc6CNq7jEu5MUbUdZLS3if7VcItbL2t6wPTLxPEPqOoDHtJZQnwCDHj2+paGBeG7B6n6J6NV+DqDLwRIM1vKw==;
```
- Emitted by `h.Add("DKIM-Signature", signer.SignatureValue())` (`internal/modify/dkim/dkim.go:406`). `d=example.org` (SDID), `i=alice@example.org` (AUID), `a=rsa-sha256` (default hash `sha256`, `dkim.go:148`), `c=relaxed/relaxed`.
- **Expiry:** `x − t = 1783386108 − 1782954108 = 432000 s = 5 days`, matching `sig_expiry` default `5*Day` (`dkim.go:146`).
- **Covers `From`:** the `h=` list contains `From:From` → `From` is **oversigned**, so the signature covers it.

`Received` (Scenario A):
```
Received:  by example.org (envelope-sender <alice@example.org>) with ESMTP
 id 46abfa3a; Thu, 02 Jul 2026 01:01:48 +0000
```
Built by `GenerateReceived` (`internal/target/received.go:19-87`). Note the **double space** after `Received:` and the **missing leading `from <host> ([ip])` clause**: submission sets `DontTraceSender` (`internal/endpoint/smtp/submission.go:28`) and the `from …` clause is gated on `!DontTraceSender` (`received.go:30`); the `by …(envelope-sender <…>) with ESMTP id …; <date>` shape matches `received.go:62,69`. The trace `id` (`46abfa3a`) equals the message's `msg_id`.

`Authentication-Results` — **absent**. The four delivered messages were dumped with `maddyctl imap-msgs dump` into `A_bob_uid1.eml` (A→bob INBOX 1), `B_alice_uid1.eml` (B→alice INBOX 1), `D_bob_uid2.eml` (D→bob INBOX 2) and `E_bob_uid3.eml` (E→bob INBOX 3), so the counts below run against the actual stored headers. Counting the header across **all four delivered messages** returns **0**:
```
$ grep -c -i "^Authentication-Results:" A_bob_uid1.eml B_alice_uid1.eml D_bob_uid2.eml E_bob_uid3.eml
A_bob_uid1.eml:0
B_alice_uid1.eml:0
D_bob_uid2.eml:0
E_bob_uid3.eml:0
$ cat A_bob_uid1.eml B_alice_uid1.eml D_bob_uid2.eml E_bob_uid3.eml | grep -c -i "^Authentication-Results:"
0
```
maddy adds it **only when auth results exist** — `if len(cr.mergedRes.AuthResult) != 0 { header.Add("Authentication-Results", …) }` (`internal/msgpipeline/check_runner.go:301-302`) — and the submission path runs no such checks (SPF/DKIM-verify/DMARC are inbound-only, `maddy.conf:54-70`).

Evidence `submissionPrepare` ran (added missing author headers):
```
submission: adding missing Message-ID
submission: adding missing Date header
```
(`submission.go:30-36` Message-ID, `:111-126` Date). A message with **no** `From` would be refused `554 5.6.0 "Message does not contains a From header field"` (`submission.go:39-47`).

## O4 — DKIM signing under identity/domain mismatch + recipient consequence
`shouldSign` (`internal/modify/dkim/dkim.go:249-330`) was driven through both mismatch branches; in every non-signing case the message is **delivered unsigned, never rejected** — `RewriteBody` returns `nil` (`dkim.go:349-350`).

(a) **`From` domain ≠ key domain** (Scenario D, `From: attacker@evil.com`). Decision log:
```
sign_dkim: not signing, From domain is not key domain	{"from_domain":"evil.com","key_domain":"example.org","msg_id":"0e2f47af"}
```
(gate `dkim.go:287-291`). The transaction was still accepted (`250 … OK: queued`, see D in the table) and the **stored message carries no `DKIM-Signature`** — the dumped headers and a header count confirm it:
```
Delivered-To: bob@example.org
Return-Path: <alice@example.org>
Received:  by example.org (envelope-sender <alice@example.org>) with ESMTP
 id 0e2f47af; Thu, 02 Jul 2026 01:02:48 +0000
Date: Thu, 2 Jul 2026 01:02:48 +0000
Message-Id: <39d0645f-892a-4ac5-a11d-23b7fb789ba7@example.org>
From: attacker@evil.com
To: bob@example.org
Subject: Scenario-D-spoofed
```
```
$ grep -c -i "^Dkim-Signature:" D_bob_uid2.eml
0
```
(The trace `id 0e2f47af` matches the decision-log `msg_id`; `From:` is `attacker@evil.com` while `Return-Path` is the envelope `alice@example.org`.)

(b) **`From` ≠ authenticated identity** under default `{envelope, auth}` (Scenario B, auth alice / `From: bob`). Decision log:
```
sign_dkim: not signing, From address is not authenticated identity	{"auth_id":"alice@example.org","from_addr":"bob@example.org","msg_id":"7e3f7780"}
```
(gate `dkim.go:299-310`). Delivered to alice, and the **stored message carries no `DKIM-Signature`**:
```
Delivered-To: alice@example.org
Return-Path: <bob@example.org>
Received:  by example.org (envelope-sender <bob@example.org>) with ESMTP id
 7e3f7780; Thu, 02 Jul 2026 01:02:17 +0000
Date: Thu, 2 Jul 2026 01:02:17 +0000
Message-Id: <c07af33d-b281-4d59-8b2b-0767838ac6d0@example.org>
From: bob@example.org
To: alice@example.org
Subject: Scenario-B-crossuser
```
```
$ grep -c -i "^Dkim-Signature:" B_alice_uid1.eml
0
```

**What a verifying recipient sees:** maddy **never emits a `d=example.org` signature over a `From: attacker@evil.com` message**, so it never produces a DMARC-*misaligned* signature. The recipient of Scenario D sees an **unsigned message** (no DKIM `pass` to rely on), not a valid-but-misaligned one. This enforces the DKIM AUID/SDID relationship (RFC 6376: `i=` domain must equal/subdomain `d=`).

## O5 — Protocol-level capture
The **complete** client/server transcripts for one accepted (A) and one rejected (C) transaction — from the `220` greeting and `EHLO`/`AUTH` through `QUIT` (and, for C, the `501` abort) — are shown in full in **O2**, captured with a temporary raw-socket SMTP client (`/tmp/smtp_probe.py`, removed after). They are cross-referenced with maddy's structured JSON logs (`incoming message`, `matched by … rule`, `sign_dkim: …`, the `501 5.1.8` reject, `signed`). The debug-level `sign_dkim: signed` line is visible because the harness passed `-debug`; the `not signing, …` lines are normal-level. Both channels agree (e.g., the reject's ` (msg ID = 3cd6abd7)` on the wire equals the `incoming message` `msg_id` in the log).

## O6 — Does the default config align MAIL FROM/From to the login identity?
**No.** Default `require_sender_match` = `{envelope, auth}` (`internal/modify/dkim/dkim.go:151-152`) gates **signing**, not **acceptance**; there is no per-account `MAIL FROM` enforcement (O1). Author-domain alignment is a receiver-side DMARC concern (RFC 7489); maddy runs DMARC inbound-only (`maddy.conf:70`).
**Ruled-out misinterpretation:** *"authenticated submission pins `MAIL FROM` to the login user."* **Scenario B disproves it** — authenticated as alice, `MAIL FROM:<bob@example.org>` → `250 2.0.0 Roger, accepting mail from <bob@example.org>`, delivered (`OK: queued`); the mismatch only caused non-signing. The one envelope rule that fires is domain membership (Scenario C → `501 5.1.8`).

## O7 — Config-vs-behavior discrepancies (runtime-supported)
(a) **`require_sender_match auth_user` is a silent no-op.** The enum accepts `auth_user` (`dkim.go:151-152` `{envelope, auth_domain, auth_user, off}`) but `shouldSign` only tests keys `off`/`envelope`/`auth` (`dkim.go:250,293,299`); the man page documents only `off`/`envelope`/`auth`, `*Default*: envelope auth` (`docs/man/maddy-filters.5.scd:518-519,531-535`). Scenario E (endpoint :2526, auth alice, `From: bob`) was **signed**:
```
[debug] sign_dkim: signed	{"identifier":"bob@example.org"}
```
and the stored `DKIM-Signature` (via `maddyctl imap-msgs dump bob@example.org INBOX 3`) carries `d=example.org` with `i=bob@example.org`:
```
Dkim-Signature: a=rsa-sha256;
 bh=yfMCG+QNY7ifaFedCoOf2Qwa4pulUlhArEIaXEqciF8=; c=relaxed/relaxed;
 d=example.org;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=bob@example.org; s=default; t=1782954183; v=1; x=1783386183; b=x0pi5armiM6a0XiX9jB8h/Uw66qF5rLDfOlLel5/Ne98xI/2e0CgupN32LQDDqsZ8zTvqforibuW24b1iHLe7opoEwf2JKrmwvsBEHbQbkmzhdbBFknD6IYRZhck1/H0LtlAScaFQiOmWxyWTdqiGIWhShQEl5DMCTIL6DrIIdh4spVJiXjxGpEzUsPxLdU+Y5gC4IfLSdDDixZ0Nd/G2Q6fCpSpEm9h+iTPvPUtaS5zndBad435bpUAyKDDM+VdC9b3J9JLr4/RMufo1yUDXqjs2f1BXqcM+SAWWpw8HPSwsg6MQyZ/l/rmcE0JDt+QiYsFb+PaaZue+RgsEME4YA==;
```
The **same inputs** were **not** signed under the default endpoint (Scenario B) but **are** signed here — `auth_user` is **less strict** than the default, which no reasonable config reading predicts.
(b) **`default_source` reject looks like a `MAIL FROM` rejection but fires at `RCPT`.** Config `default_source { reject 501 5.1.8 "Non-local sender domain" }` (`maddy.conf:117-119`), but `defer_sender_reject` defaults `true` (`internal/endpoint/smtp/smtp.go:567`), so `MAIL FROM` gets `250` and the `501 5.1.8` is emitted on first `RCPT` — see Scenario C's full dialog in O2 (`MAIL FROM:<eve@notlocal.com>` → `250`, then `RCPT TO:<alice@example.org>` → `501 5.1.8`).

## O8 — Deliverable & hygiene
This markdown file is the only repository addition. `/tmp` artifacts created: `/tmp/maddybin/{maddy,maddyctl}`, `/tmp/maddyrun/**` (config, `all.db`, DKIM keys, mailboxes, `maddy.log`), `/tmp/smtp_probe.py`, `/tmp/maddyrun/dialog_*.txt`, `/tmp/maddyrun/msgs/*.eml`. Temp script + captured messages/dialogs removed and the run/state dir (DB + generated keys) deleted after capture; repo left byte-for-byte unchanged (`.mkdocs.yml` untouched, no nav/index entry added).

## Appendix — `shouldSign` decision flow (`internal/modify/dkim/dkim.go:249-330`)
1. `off` present → sign unconditionally (`:250`). 2. `From` must be present/parseable/single else no-sign. 3. `From` domain must equal key domain else no-sign (`:287-291`, D). 4. `envelope` set & `From`≠`MAIL FROM` → no-sign (`:293`). 5. `auth` set & `From`≠auth identity → no-sign (`:299-310`, B). 6. else sign: `h.Add` (`:406`), debug `signed` (`:408`). Every no-sign path returns `nil` → delivered unsigned (`:349-350`). Pipeline adds `Authentication-Results` (when present) **before** modifiers so a signature can cover it (`internal/msgpipeline/msgpipeline.go:316-320`). Defaults: `relaxed/relaxed`, `sha256` (`:148`), 5-day expiry `432000 s` (`:146`), `rsa2048` (`:150`).

## Standards framing (code is primary truth)
- **RFC 6376 (DKIM):** `d=`=SDID, `i=`=AUID whose domain must be SDID or a subdomain; maddy signs `d=example.org`/`i=alice@example.org` and refuses unrelated `From` domains.
- **RFC 7489 (DMARC):** identifier alignment needs `RFC5322.From` to align with a DKIM(`d=`)/SPF-validated domain; a valid signature alone ≠ author authenticity; alignment is receiver-side; maddy runs DMARC inbound-only.
- **RFC 6409 (Submission):** authenticates the client but is not required to force `MAIL FROM` to equal the login — maddy's per-user non-enforcement is a defensible default.

## Coverage pass — every named item addressed
- **`MAIL FROM`** — O1 (domain routing, not per-account), O2/O6 (B: `250` while auth≠sender), O7b (deferred RCPT reject). ✔
- **`From`** — O4 (signing gated on it), O3 (oversigned `From:From`), O7a (`i=bob@example.org`). ✔
- **`DKIM-Signature`** — O3 (raw, complete `b=`/`bh=`/`d=`/`i=`/`t=`/`x=`/`h=` for A), O4 (demonstrated absent under mismatch, header count `0` on B and D), O7a (full stored sig for E). ✔
- **`Received`** — O3 (verbatim; omitted `from` clause via `DontTraceSender`). ✔
- **`Authentication-Results`** — O3 (demonstrated absence, `grep -c` = `0` across all four delivered messages, code reason). ✔
- **SMTP response codes** — O2 (full dialogs: `220/250-…/235/250/250/354/250/221`; and `250`→`501 5.1.8`). ✔
- **Default-config alignment** — O6 (default `{envelope,auth}` gates signing not acceptance; misinterpretation ruled out via B). ✔
- **Mismatch constructions** — cross-user same-domain (B), non-local domain (C), spoofed `From` (D), config no-op (E). ✔
