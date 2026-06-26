# maddy Submission Sender-Identity & DKIM-Signing Investigation

> **Scope of this document.** This is an investigative, evidence-first Q&A report that answers — from **runtime behavior corroborated by code-as-truth** — *how the [maddy](https://github.com/foxcpp/maddy) mail server enforces sender identity and message authentication during authenticated message submission*. Every claim below is backed by either a verbatim runtime artifact (SMTP transcript, stored header, debug-log line, DNS record, or verifier error) reproduced exactly in a fenced code block, and/or a precise `[path:Lstart-Lend]` citation into the source tree at HEAD `26452dd8dd787dc455278b0fdd296f4a5432c768`. No behavior is asserted that cannot be cited.
>
> **Repository integrity.** The maddy source tree was treated as **read-only**. No source file was created, modified, or deleted. The single committed artifact is this document. The closing attestation in [Section 6](#section-6--artifacts--cleanup-attestation) records the unchanged-source proof.

---

## Section 1 — Summary & Environment

### 1.1 Objective

The goal was to determine **empirically** — not from documentation or assumption — whether and how maddy binds the sender of an authenticated submission session (the envelope `MAIL FROM` and the `From` header) to the authenticated identity, and how its `sign_dkim` modifier behaves across aligned and misaligned sender constructions. The investigation built maddy from this repository, ran a faithful submission server, drove it with four controlled sender-boundary experiments (plus a mandatory fallback capture), inspected the delivered mail byte-for-byte, attempted DKIM verification, probed configuration-load behavior, and then cleaned up — all while leaving the source tree untouched.

### 1.2 Build

maddy was compiled directly from this repository. The default storage/auth backend is SQLite via the cgo driver `github.com/mattn/go-sqlite3` [go.mod], so `CGO_ENABLED=1` is mandatory — without a C compiler the build fails.

- **Toolchain:** go1.20.14 linux/amd64; gcc 13.3.0; openssl 3.0.13; python3 3.12.3.
- **Module:** `github.com/foxcpp/maddy`; floor `go 1.13` [go.mod].
- **Build commands** (output to an out-of-repo bin directory):

```
export GO111MODULE=on CGO_ENABLED=1
go build -o /tmp/maddy-investigation/bin/maddy    ./cmd/maddy
go build -o /tmp/maddy-investigation/bin/maddyctl ./cmd/maddyctl
```

Both binaries compiled with **EXIT 0**. `maddy -v` reported `maddy unknown (built from source tree)`.

### 1.3 Throwaway runtime configuration (authored OUTSIDE the repo)

A throwaway `maddy.conf` was authored under `/tmp/maddy-investigation/` (never inside the repo tree) that mirrors the shipped semantics of the repository's `maddy.conf` so the findings transfer to the default deployment. The shipped config uses `example.org`; the test config used the local test domain `maddytest.local` — a faithful mirror.

- `$(hostname)=mail.maddytest.local`; `$(primary_domain)=$(local_domains)=maddytest.local`.
- `sql local_mailboxes local_authdb { driver sqlite3; dsn /tmp/maddy-investigation/run/all.db }` — one SQLite database serves as **both** the mailbox store and the credentials store, mirroring `maddy.conf` [maddy.conf:L32-L35].
- **Authenticated submission listener** `submission tls://0.0.0.0:465` with `auth &local_authdb`, a `source $(local_domains)` block whose `modify{ sign_dkim $(primary_domain) default }` signs locally and delivers to `&local_mailboxes`, and `default_source { reject 501 5.1.8 "Non-local sender domain" }`. **This block carries no `check{}`/`dmarc`** — exactly as the shipped submission block [maddy.conf:L93-L120].
- **Contrast port-25 listener** `smtp tcp://0.0.0.0:25` with `check{ require_matching_ehlo; require_mx_record; verify_dkim; apply_spf }` and `dmarc yes`, mirroring [maddy.conf:L53-L91]. This block exists only so the presence/absence of the `Authentication-Results` header can be contrasted against the checkless submission block.
- `imap tls://0.0.0.0:993` for mailbox inspection; a throwaway self-signed TLS certificate at `certs/{cert,key}.pem`.

### 1.4 Accounts and DKIM key

Two accounts were provisioned in the one local domain via `maddyctl` (`internal/storage/sql/maddyctl.go` — `CreateUser` [internal/storage/sql/maddyctl.go:L17]; CLI entry `cmd/maddyctl/users.go` — `usersCreate` [cmd/maddyctl/users.go:L30]). `maddyctl ... creds list` confirmed:

```
usera@maddytest.local
userb@maddytest.local
```

Passwords were userA=`passwordA123`, userB=`passwordB456`. Usernames are normalized to lowercase/case-folded by `prepareUsername` [internal/storage/sql/sql.go:L351] (used by `CheckPlain` [internal/storage/sql/sql.go:L377]), which is why `userA` is stored and looked up as `usera@maddytest.local`.

On first start, maddy auto-generated the signing key `run/dkim_keys/maddytest.local_default.key` (PEM `-----BEGIN PRIVATE KEY-----`, i.e. PKCS#8) and the public-key TXT record `…_default.dns` (selector `default`, `d=maddytest.local`). The `.dns` record, **verbatim**:

```
v=DKIM1; k=rsa; p=MIIBCgKCAQEAzSJomPtFsw8lWXfK483I9hXpOTji+JxoTFEkzcvHgIvES0lpUzmeZ9om+wlP1uUpXoz7cIrQ+MHRH2Kv6rPwDw1haXqFeLphrjQB/4pMIwMHCti51DiTK1PzDQCHGf12rS64BxJMqpafmidYXfXH+GMnhXGRBA25FwF8QgFOwvw8p94670ylhDFNs5CZfnFR7ZK9yzi9iFW0bNm899RlXrw4JvUMW/m+BDQK12JOGyPhMiHXQ/rAU979/qaI1wOX5aKRUAs21346R0lV+3sZ5HtbQK89Azzaaui2nPi8LHNKM759p67o2FQ4sm8npEMmg3yfR6wvATeDTuT36WHI+QIDAQAB
```

The `p=` value begins `MIIBCgKCAQEA…`, which is the DER prefix of a **PKCS#1 `RSAPublicKey`** — *not* a PKIX/SubjectPublicKeyInfo key (which would begin `MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A…`). This is generated by `writeDNSRecord`, which marshals RSA public keys with `x509.MarshalPKCS1PublicKey` [internal/modify/dkim/keys.go:L143] and base64-encodes them into `p=` [internal/modify/dkim/keys.go:L158]. It is central to Finding **b-2** in [Section 4, Q4](#q4--for-the-spoofed-from-case-e4-is-a-signature-applied-does-it-cover-from-what-does-a-verifier-see-and-for-the-signed-case-e3).

### 1.5 Where artifacts lived

**Every** runtime artifact — built binaries, the test config, the self-signed certificate, the SQLite database, the generated DKIM key and `.dns` record, captured transcripts, debug logs, `.eml` header dumps, and the client/verifier scripts — was created **outside the repository** under `/tmp/maddy-investigation/`, and was deleted after evidence capture (enumerated in [Section 6](#section-6--artifacts--cleanup-attestation)).

---

## Section 2 — Decision-Path Overview

The single most important structural finding is that, on the authenticated submission path, **acceptance and signing are two independent decisions**, and the authenticated identity (`AuthUser` [internal/endpoint/smtp/smtp.go:L680]) is **not an input to acceptance at all** in the default configuration — it influences only the DKIM signing decision.

1. **Acceptance is gated by the envelope `MAIL FROM`.** The pipeline selects a `source` block purely from the envelope address via `srcBlockForAddr` [internal/msgpipeline/msgpipeline.go:L155-L201]: it tries the full normalized address (`perSource[cleanFrom]`), then the domain (`perSource[domain]`), then falls back to `defaultSource`. A local-domain envelope matches the `source $(local_domains)` block and is accepted; a non-local envelope falls through to `default_source { reject 501 5.1.8 "Non-local sender domain" }` [maddy.conf:L117-L119]. The authenticated username is **never** consulted here.
2. **Header validation is presence/syntax-only.** The `submissionPrepare` modifier [internal/endpoint/smtp/submission.go:L27-L130] adds a `Message-ID` if missing, requires a `From` header to exist (else `554 5.6.0`), validates `Sender`/`To`/`Cc`/`Bcc`/`Reply-To` syntax, requires a `Sender` header if `From` has multiple addresses, and validates/adds `Date`. It **never** compares `From` to the authenticated user or to the envelope.
3. **Signing is an independent modifier decision.** `shouldSign` [internal/modify/dkim/dkim.go:L249-L330] decides whether to attach a `DKIM-Signature`. On any mismatch it returns `("", false)` and the message is delivered **unsigned — not rejected**.

```mermaid
flowchart TD
    A[Client connects to submission listener :465] --> B{SASL authenticated?}
    B -- No --> R0[Reject: AUTH required, authAlwaysRequired=true smtp.go:L589-L590]
    B -- Yes, AuthUser set smtp.go:L680 --> C[MAIL FROM captured as OriginalFrom smtp.go:L116]
    C --> E{srcBlockForAddr by envelope MAIL FROM msgpipeline.go:L155-L201}
    E -- local-domain envelope --> F[source local_domains block: accept]
    E -- non-local envelope --> R2[default_source: reject 501 5.1.8 maddy.conf:L117-L119]
    F --> D[submissionPrepare: validate From/Sender presence+syntax only submission.go:L27-L130]
    D -- From missing/invalid --> R1[Reject 554 5.6.0 after DATA]
    D -- From present and valid --> G{sign_dkim shouldSign? dkim.go:L249-L330}
    G -- From-domain == key-domain AND require_sender_match satisfied --> H[Add DKIM-Signature]
    G -- mismatch --> I[Skip signing: deliver UNSIGNED, not rejected]
    H --> J[deliver_to local mailbox]
    I --> J
```

**Headline finding:** maddy's submission path is **AUTH-required but sender-authorization-optional**. Authentication is mandatory (`authAlwaysRequired=true` [internal/endpoint/smtp/smtp.go:L589-L590]; an auth provider is required, else the endpoint refuses to start [internal/endpoint/smtp/smtp.go:L592-L593]), but **no built-in check binds the sender to the authenticated user** in the default configuration. The result is a **fail-open-for-delivery / fail-closed-for-signing** posture: a misaligned message is delivered, just without a signature.

---

## Section 3 — Per-Experiment Results (E1–E4, plus the E5 fallback)

All four experiments were driven by a scripted SMTP client **authenticated as userA** on the submission listener (`:465`), independently varying the envelope `MAIL FROM` and the `From` header. Reply codes were captured from the session transcripts; the governing `sign_dkim` decision was captured from maddy's debug log; and the delivered message headers were extracted byte-for-byte from `run/messages/`.

### E1 — cross-user `MAIL FROM` mismatch → **ACCEPTED**

- **Client action:** auth = userA; `MAIL FROM:<userB@maddytest.local>`; `From: userB@maddytest.local`; `RCPT TO:<userA@maddytest.local>`.
- **Transcript (verbatim):**

```
S: 235 2.0.0 Authentication succeeded
C: MAIL FROM:<userB@maddytest.local>
S: 250 2.0.0 Roger, accepting mail from <userB@maddytest.local>
C: RCPT TO:<userA@maddytest.local>
S: 250 2.0.0 I'll make sure <userA@maddytest.local> gets this
C: DATA
S: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
... (From: userB@maddytest.local) ...
C: .
S: 250 2.0.0 OK: queued
==== E1 SUMMARY: MAIL=250 RCPT=250 DATA=354 FINAL=250 ====
```

- **Reply codes:** AUTH `235`; MAIL `250`; RCPT `250`; DATA `354`; final `250 2.0.0 OK: queued`.
- **Governing debug-log line (verbatim):**

```
sign_dkim: not signing, From address is not authenticated identity {"auth_id":"userA@maddytest.local","from_addr":"userB@maddytest.local","msg_id":"1201412b"}
```

- **Stored result:** **delivered, UNSIGNED.** Stored headers (verbatim):

```
Delivered-To: usera@maddytest.local
Return-Path: <userB@maddytest.local>
Received:  by mail.maddytest.local (envelope-sender <userB@maddytest.local>)
 with ESMTPS id 1201412b; Fri, 26 Jun 2026 19:50:12 +0000
From: userB@maddytest.local
To: userA@maddytest.local
Subject: E1 cross-user MAIL FROM mismatch
Date: Mon, 01 Jan 2024 00:00:00 +0000
Message-ID: <E1.1782503412@client.test>
```

- **Why:** the envelope `MAIL FROM` is a local domain, so `srcBlockForAddr` selects the local `source` block and the message is accepted regardless of who authenticated [internal/msgpipeline/msgpipeline.go:L155-L201]. The default `require_sender_match` includes `auth`, so `shouldSign` refuses to sign because the `From` local-part (`userB`) does not equal the authenticated identity (`userA`) [internal/modify/dkim/dkim.go:L299-L310] — but refusal means *no signature*, not rejection.

### E2 — non-local envelope domain → **REJECTED (at `RCPT TO`)**

- **Client action:** auth = userA; `MAIL FROM:<spoofer@nonlocal.tld>`; `RCPT TO:<userA@maddytest.local>`.
- **Transcript (verbatim):**

```
C: MAIL FROM:<spoofer@nonlocal.tld>
S: 250 2.0.0 Roger, accepting mail from <spoofer@nonlocal.tld>
C: RCPT TO:<userA@maddytest.local>
S: 501 5.1.8 Non-local sender domain (msg ID = 2d8d49b6)
C: DATA
S: 502 5.5.1 Missing RCPT TO command.
==== E2 SUMMARY: MAIL=250 RCPT=501 DATA=502 FINAL=None ====
```

- **Reply codes:** MAIL `250`; RCPT `501 5.1.8 Non-local sender domain`; DATA `502 5.5.1 Missing RCPT TO command.`
- **Governing debug-log line (verbatim):**

```
submission: RCPT error {"reason":"reject directive used","smtp_code":501,"smtp_enchcode":"5.1.8","smtp_msg":"Non-local sender domain"}
```

- **Stored result:** **nothing delivered** (rejected before `DATA`).
- **Why:** the envelope `MAIL FROM` domain (`nonlocal.tld`) matches no `source` block, so `srcBlockForAddr` falls back to `defaultSource` [internal/msgpipeline/msgpipeline.go:L155-L201], which is the `default_source { reject 501 5.1.8 "Non-local sender domain" }` directive [maddy.conf:L117-L119]. Note the rejection fires at **`RCPT TO`**, not at `MAIL FROM` (which returned `250`): the source block is resolved at the first recipient, not when the envelope sender is received. This timing is one of the runtime-vs-reasonable-reading discrepancies in [Q7](#q7--at-least-one-case-where-runtime-behavior-differs-from-a-reasonable-reading-of-the-configuration).

### E3 — happy path, fully aligned → **ACCEPTED + SIGNED**

- **Client action:** auth = userA; `MAIL FROM:<userA@maddytest.local>`; `From: userA@maddytest.local`; `RCPT TO:<userA@maddytest.local>`.
- **Transcript (verbatim):**

```
C: MAIL FROM:<userA@maddytest.local>   S: 250 ...
C: RCPT TO:<userA@maddytest.local>     S: 250 ...
C: DATA  S: 354 ...   (From: userA@maddytest.local)   C: .   S: 250 2.0.0 OK: queued
==== E3 SUMMARY: MAIL=250 RCPT=250 DATA=354 FINAL=250 ====
```

- **Reply codes:** MAIL `250`; RCPT `250`; DATA `354`; final `250 2.0.0 OK: queued`.
- **Governing debug-log line (verbatim):**

```
sign_dkim: signed {"identifier":"userA@maddytest.local"}
```

- **Stored result:** **delivered, SIGNED.** Stored headers (verbatim — note `go-message` stored the field title-cased as `Dkim-Signature`):

```
Delivered-To: usera@maddytest.local
Return-Path: <userA@maddytest.local>
Dkim-Signature: a=rsa-sha256;
 bh=+6VbdMGWEdZKKlGvcAgQFO8gEJUPLvrDoPLxhqxjEoQ=; c=relaxed/relaxed;
 d=maddytest.local;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=userA@maddytest.local; s=default; t=1782503436; v=1; x=1782935436; b=SJpwHNFfAJiZ51rRgZ1iqLBrX7SXuCvKvN7aVyVmYpfBJ4EZDmBgn0GCL8bEEX5fQlWHnIUxqg1/antynRv9xrcgQA7ZkrlXHEBWNKXyYp4c+3t2ATD+1RZqYYDVDmZmIo8UDN0DKhfbAViKV9sojCfaEfprQ5K4N28Y73403HKH3CMee9ZTi8Z13gjSmJAxsr5h8r80iynwUPiLxRfCGRW4+38w11ixVD3VBzCYxdqZVzNgPo9w7RxBt/4VmiuOhdyOkwfHlUoWW+UKtV5qSn/lNCkUePI1jTsSoOJWRvPe1NZYDubgjw7+N2tAkxBF1Cx2xCsYei3xyQueuNSWDQ==;
Received:  by mail.maddytest.local (envelope-sender <userA@maddytest.local>)
 with ESMTPS id 61b19bef; Fri, 26 Jun 2026 19:50:36 +0000
From: userA@maddytest.local
To: userA@maddytest.local
Subject: E3 happy path aligned
Date: Mon, 01 Jan 2024 00:00:00 +0000
Message-ID: <E3.1782503436@client.test>
```

- **Why:** every alignment check in `shouldSign` passes — `From`-domain equals the key domain, `From` equals the envelope `MAIL FROM`, and the `From` local-part equals the authenticated identity — so the message is signed [internal/modify/dkim/dkim.go:L249-L330]. The `h=` tag lists `From` (in fact `From:From`, oversigned), so the signature genuinely covers the `From` header; `d=maddytest.local`, `i=userA@maddytest.local`, `s=default`.

### E4 — spoofed `From` header → **ACCEPTED + UNSIGNED (not rejected)**

- **Client action:** auth = userA; `MAIL FROM:<userA@maddytest.local>` (local, aligned envelope); `From: someone@other-domain.example` (spoofed header); `RCPT TO:<userA@maddytest.local>`.
- **Transcript (verbatim):**

```
C: MAIL FROM:<userA@maddytest.local>   S: 250 ...
C: RCPT TO:<userA@maddytest.local>     S: 250 ...
C: DATA  S: 354 ...   (From: someone@other-domain.example)   C: .   S: 250 2.0.0 OK: queued
==== E4 SUMMARY: MAIL=250 RCPT=250 DATA=354 FINAL=250 ====
```

- **Reply codes:** MAIL `250`; RCPT `250`; DATA `354`; final `250 2.0.0 OK: queued`.
- **Governing debug-log line (verbatim):**

```
sign_dkim: not signing, From domain is not key domain {"from_domain":"other-domain.example","key_domain":"maddytest.local","msg_id":"43650d4c"}
```

- **Stored result:** **delivered, UNSIGNED.** Stored headers (verbatim):

```
Delivered-To: usera@maddytest.local
Return-Path: <userA@maddytest.local>
Received:  by mail.maddytest.local (envelope-sender <userA@maddytest.local>)
 with ESMTPS id 43650d4c; Fri, 26 Jun 2026 19:50:43 +0000
From: someone@other-domain.example
To: userA@maddytest.local
Subject: E4 spoofed From header
Date: Mon, 01 Jan 2024 00:00:00 +0000
Message-ID: <E4.1782503443@client.test>
```

- **Why:** the envelope is local and aligned, so the message is accepted and delivered; but the `From`-header domain (`other-domain.example`) differs from the signing key's domain (`maddytest.local`), so `shouldSign` skips signing at the first alignment gate [internal/modify/dkim/dkim.go:L287-L291]. The message is delivered with **no** `DKIM-Signature`.

### E5 — mandatory fallback capture (`openssl s_client`)

Per the methodological requirement, in addition to maddy debug logging + raw stored-message extraction (the primary capture), the documented fallback transcript was also captured with `openssl s_client -connect 127.0.0.1:465`, for an accepted userA→userB submission (verbatim):

```
220 mail.maddytest.local ESMTP Service Ready
235 2.0.0 Authentication succeeded
250 2.0.0 Roger, accepting mail from <userA@maddytest.local>
250 2.0.0 I'll make sure <userB@maddytest.local> gets this
354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
250 2.0.0 OK: queued
221 2.0.0 Goodnight and good luck
```

This satisfies the **mandatory-fallback** requirement: the SMTP transaction is captured independently of maddy's own logging, directly from the TLS wire via `openssl s_client`.

---

## Section 4 — Question-by-Question Answers (Q1–Q7)

Each question is answered with an explicit **Answer**, the **Rationale (thinking)** that connects evidence to conclusion, and the **Evidence** (code citation and/or verbatim artifact).

### Q1 — Does maddy bind the authenticated session's sender address, and what are the actual reply codes/text?

**Answer.** **No — by default maddy does not bind the sender to the authenticated identity.** Authentication is required to submit at all, but neither the envelope `MAIL FROM` nor the `From` header is compared to `AuthUser`. The submission layer validates only the *presence and syntax* of `From`/`Sender`; acceptance is decided by envelope-domain routing.

**Rationale (thinking).** Three independent code paths together determine the outcome, and none of them compares the sender to the authenticated user. (1) `submissionPrepare` [internal/endpoint/smtp/submission.go:L27-L130] only checks that a `From` header exists and that `From`/`Sender`/recipient fields are syntactically valid; if `From` is absent it returns `554 5.6.0 "Message does not contains a From header field"` [internal/endpoint/smtp/submission.go:L39-L48], but it never inspects *who* `From` is. (2) Acceptance is decided by `srcBlockForAddr` [internal/msgpipeline/msgpipeline.go:L155-L201], which routes purely on the envelope `MAIL FROM` (full address → domain → default). (3) The authenticated identity is carried as `AuthUser` [internal/endpoint/smtp/smtp.go:L680] and the envelope as `OriginalFrom`/`cleanFrom` [internal/endpoint/smtp/smtp.go:L116], but `AuthUser` is consumed **only** by the optional `sign_dkim` matching logic, never by acceptance. E1 proves this empirically: authenticated as userA with `MAIL FROM:<userB@…>`, the message was accepted and delivered.

**Evidence.** Reply codes observed across the experiments: `235` (auth success); `250` for `MAIL`/`RCPT` on a local envelope (E1, E3, E4, E5); `501 5.1.8 Non-local sender domain` at `RCPT` for a non-local envelope (E2); `250 2.0.0 OK: queued` on accept. Unauthenticated submission is impossible because `authAlwaysRequired=true` for submission endpoints [internal/endpoint/smtp/smtp.go:L589-L590], and the endpoint refuses to start without an auth provider [internal/endpoint/smtp/smtp.go:L592-L593]. See the E1 transcript and stored headers in [Section 3](#e1--cross-user-mail-from-mismatch--accepted).

### Q2 — Does behavior differ by *how* the mismatch is constructed (envelope vs. header; cross-user vs. cross-domain)?

**Answer.** **Yes — strongly.** The construction of the mismatch determines which subsystem reacts and whether the message is rejected, accepted-and-signed, or accepted-and-unsigned:

| Mismatch construction | Subsystem that reacts | Outcome |
|---|---|---|
| Envelope `MAIL FROM` in a **non-local** domain (E2) | Acceptance routing | **Rejected** `501 5.1.8` at `RCPT` |
| Envelope `MAIL FROM` = **another local user** (E1) | Acceptance routing accepts; signing refuses | **Accepted, unsigned** |
| `From` **header** in a non-key domain (E4) | Signing only | **Accepted, unsigned** |
| Fully aligned (E3) | All checks pass | **Accepted, signed** |

**Rationale (thinking).** The envelope domain governs *acceptance* (cross-domain → reject, same-domain → accept), while the `From` header and the authenticated identity govern only *signing*. So a *cross-domain envelope* mismatch (E2) is rejected, but a *cross-user same-domain* mismatch (E1) and a *spoofed-header* mismatch (E4) are both accepted and merely left unsigned. The "how" matters because acceptance and signing are decided by different code with different inputs.

**Evidence.** E1 (same-domain cross-user → accepted), E2 (cross-domain envelope → `501 5.1.8`), E4 (spoofed `From` header → accepted, unsigned), all verbatim in [Section 3]; routing via `srcBlockForAddr` [internal/msgpipeline/msgpipeline.go:L155-L201]; the E4 signing-skip at the key-domain gate [internal/modify/dkim/dkim.go:L287-L291].

### Q3 — Verbatim stored headers (raw `DKIM-Signature`, maddy-added `Received`, `Authentication-Results`/other security headers)

**Answer.** maddy adds `Delivered-To`, `Return-Path`, and `Received` to every delivered message; it adds a `DKIM-Signature` only when `shouldSign` accepts (E3, not E1/E4); and it adds **no `Authentication-Results` header at all** to mail delivered via the submission listener. The verbatim stored headers for E1, E3, and E4 are reproduced in [Section 3](#e1--cross-user-mail-from-mismatch--accepted) and are not duplicated here.

**Rationale (thinking).** The absence of `Authentication-Results` on submission mail is not an accident — it is a direct consequence of where checks live. `Authentication-Results` is emitted by `applyResults` **only when** merged check/DMARC results exist:

> `applyResults` adds the header inside `if len(cr.mergedRes.AuthResult) != 0 { header.Add("Authentication-Results", …) }` [internal/msgpipeline/check_runner.go:L301-L302].

The shipped submission block carries **no `check{}` and no `dmarc`** [maddy.conf:L93-L120], so `AuthResult` is empty and the header is never written. By contrast, the port-25 `smtp` block does carry checks + `dmarc yes` [maddy.conf:L53-L91], so inbound mail there *would* receive an `Authentication-Results` header. The `Received` header is built by `GenerateReceived` [internal/target/received.go:L19-L87] and added by the endpoint via `header.Add("Received", received)` [internal/endpoint/smtp/smtp.go:L303-L307]; its form is `by <hostname> (envelope-sender <MAIL FROM>) with <Proto> id <ID>; <date>`, with the leading `from <client>` trace omitted because submission sets `DontTraceSender` [internal/endpoint/smtp/submission.go:L28] (checked in `GenerateReceived` [internal/target/received.go:L30]). This matches the stored `Received` lines exactly.

**Evidence.** The E1/E3/E4 stored-header blocks in [Section 3] — none carries `Authentication-Results`; E3 carries the full raw `Dkim-Signature`; all three carry the `(envelope-sender <…>)`-form `Received`. Code: [internal/msgpipeline/check_runner.go:L262-L303], [internal/target/received.go:L19-L87], [internal/endpoint/smtp/smtp.go:L303-L307].

### Q4 — For the spoofed-`From` case (E4): is a signature applied, does it cover `From`, what does a verifier see? And for the signed case (E3)?

**Answer (E4).** **No signature is applied.** `shouldSign` skips signing because the `From`-header domain (`other-domain.example`) is not the key domain (`maddytest.local`) [internal/modify/dkim/dkim.go:L287-L291]; the verbatim debug line is shown in [Section 3](#e4--spoofed-from-header--accepted--unsigned-not-rejected). A verifier therefore sees `dkim=none` — there is simply no signature to evaluate. maddy refuses to emit a signature that would not align with the visible `From`.

**Answer (E3).** **A signature is applied and it does cover `From`** — the `h=` tag lists `From` twice (`…From:From…`, i.e. *oversigned*; maddy's oversign default set is From, Subject, Sender, To, Cc, Date), which both signs the present `From` and guards against a second `From` being added. **However**, a verifier that fetches maddy's *published* public key cannot use it, because the published key is in the wrong format (Finding **b-2** below).

**Rationale (thinking) — two distinct findings, kept separate:**

- **Finding b-1 — the signing math is correct.** The relaxed body hash of E3 computed independently equals the signature's `bh=`, and a pure `Sign → Verify` round-trip using maddy's *actual private key* together with a PKIX-re-encoded public key **passes** (`err=<nil>`). So maddy computes a genuine, `From`-covering signature.
- **Finding b-2 — the published public key is in the wrong format, so no standard verifier can use it (a real bug).** `writeDNSRecord` marshals the RSA public key with `x509.MarshalPKCS1PublicKey` (PKCS#1) [internal/modify/dkim/keys.go:L143] and base64-encodes it into `p=` [internal/modify/dkim/keys.go:L158]. But the verifier in maddy's own DKIM library, `github.com/emersion/go-msgauth` [go.mod] (`dkim/query.go`), parses `p=` with `x509.ParsePKIXPublicKey`, which expects PKIX/SPKI. Since go-msgauth backs **both** `sign_dkim` (the modifier) and `verify_dkim` (the port-25 check [internal/check/dkim/dkim.go:L157-L215]), maddy's own verifier cannot parse maddy's own published key. Feeding the published `.dns` to go-msgauth yields, verbatim:

```
Verifier sees: FAIL  d=maddytest.local
err=dkim: key syntax error: x509: failed to parse public key (use ParsePKCS1PublicKey instead for this key format)
```

The published-key prefix confirms the format: the `p=` value begins `MIIBCgKCAQEA…` (PKCS#1 `RSAPublicKey`), whereas a PKIX key would begin `MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A…` (see the `.dns` record in [Section 1.4](#14-accounts-and-dkim-key)). `dkimpy` also returns `verify=False` on the published artifacts as corroboration; note that modern python-`cryptography` can auto-parse PKCS#1, so dkimpy's failure is not *solely* attributable to the key — the code-attributable smoking gun is the go-msgauth error above.

- **Evidence-handling caveat (stated precisely so it is not mis-stated).** The signature does **not** byte-verify against the *stored* mailbox copy (or an `smtp_downstream`-relayed copy) even with a format-corrected key, because `go-message` re-serializes the message *after* signing. The verbatim error returned by this stored-copy verification attempt is:

```
crypto/rsa: verification error
```

The re-serialization canonicalizes, for example, header field-name case (`DKIM-Signature` → `Dkim-Signature`, `Message-ID` → `Message-Id`, both visible in the E3 stored headers). This is an **inspection artifact, not an additional signing bug**: the pure round-trip in b-1 proves the signature itself is valid. To verify a maddy DKIM signature one needs the exact as-signed bytes **and** a PKIX-corrected key.

**Net.** maddy's default `sign_dkim` output is **unverifiable by standard verifiers and by maddy's own `verify_dkim`**, despite the signature being cryptographically sound, because the *published key* is encoded in PKCS#1 where PKIX is expected.

### Q5 — A captured SMTP transaction for at least one rejected and at least one accepted message

**Answer.** **Satisfied.** A **rejected** transaction is captured verbatim in **E2** (`501 5.1.8 Non-local sender domain`). **Accepted** transactions are captured verbatim in **E1**, **E3**, **E4**, and the **E5** `openssl s_client` fallback (`250 2.0.0 OK: queued`). All transcripts appear in [Section 3](#section-3--per-experiment-results-e1e4-plus-the-e5-fallback).

**Rationale (thinking).** The requirement is explicitly a coverage requirement (≥1 reject, ≥1 accept); E2 covers reject and E1/E3/E4/E5 cover accept, with both maddy-log-derived transcripts (E1–E4) and an independent wire transcript (E5).

**Evidence.** E2 (rejected) and E1/E3/E4/E5 (accepted) verbatim transcripts in [Section 3].

### Q6 — Does the **default** config enforce sender↔authenticated-identity alignment, or is explicit policy required?

**Answer.** **The default configuration does NOT enforce sender↔authenticated-identity alignment.** Binding the envelope sender (or `From`) to the authenticated identity is an *optional* MSA behavior; maddy ships without it, and — critically — **this version has no `authorize_sender`-style check module**, so per-user envelope enforcement is not even available as a built-in check to turn on. Enforcement would require external/custom policy.

**Rationale (thinking).** Two plausible-but-incorrect interpretations are ruled out with direct evidence:

- **Wrong interpretation (i): "Because AUTH is required, `MAIL FROM`/`From` must equal the authenticated user."** **FALSE.** In E1, authenticated as userA, `MAIL FROM:<userB@maddytest.local>` (and `From: userB@…`) was **accepted and delivered** (`250 2.0.0 OK: queued`; message stored, see E1 headers). Acceptance is by envelope-domain routing [internal/msgpipeline/msgpipeline.go:L155-L201], which never reads `AuthUser`. Requiring AUTH gates *whether you may submit*, not *whose identity you may claim*.
- **Wrong interpretation (ii): "`require_sender_match {envelope, auth}` (the default) means a `From`≠auth message is rejected."** **FALSE.** `require_sender_match` governs only the *signing* decision in `shouldSign`; on a mismatch it returns `("", false)` and the message is delivered **unsigned, not rejected** [internal/modify/dkim/dkim.go:L249-L330]. E1 (cross-user) and E4 (spoofed `From`) were both accepted and merely left unsigned — the fail-open-for-delivery / fail-closed-for-signing posture.

This is consistent with the message-submission standard, which makes per-identity submission-rights enforcement optional rather than mandatory (see [Section 5](#section-5--standards-framing)).

**Evidence.** E1 and E4 accepted+delivered (verbatim transcripts and stored headers, [Section 3]); `srcBlockForAddr` [internal/msgpipeline/msgpipeline.go:L155-L201]; `shouldSign` returns `("", false)` on mismatch [internal/modify/dkim/dkim.go:L249-L330]. A repository-wide read found no `authorize_sender`-style check module in this version.

### Q7 — At least one case where runtime behavior differs from a reasonable reading of the configuration

**Answer.** **Three** such discrepancies were observed.

**Rationale (thinking).** Each discrepancy is a place where an operator's reasonable reading of the config (the manual's value list, "signing means recipients can verify," "reject non-local senders at the envelope") diverges from what the code actually does at runtime. The three are presented below, each paired with the verbatim runtime artifact that exposes the divergence and the code path that explains it.

**(a) `require_sender_match` documentation-vs-code, plus a silent-skip that inverts the apparent guarantee.** A reasonable operator reading the manual would configure `require_sender_match auth` to bind `From` to the authenticated identity. The manual `docs/man/maddy-filters.5.scd` documents `*Default*: envelope auth` and lists valid values as `off | envelope | auth` [docs/man/maddy-filters.5.scd:L518-L544]. But the code's enum accepts a **different** value set — `{envelope, auth_domain, auth_user, off}` — with a hardcoded default of `{envelope, auth}` [internal/modify/dkim/dkim.go:L151-L152], and `shouldSign` reads **only** the map keys `"off"`, `"envelope"`, and `"auth"` [internal/modify/dkim/dkim.go:L250,L293,L299] — it never reads `"auth_user"` or `"auth_domain"`. The runtime confirms both halves:

- Setting `require_sender_match auth` (the *documented* value) is **rejected at config load**, verbatim:

```
invalid argument, valid values are: [envelope auth_domain auth_user off]
```

  (This message is produced by `EnumList` [internal/config/map.go:L81] via `MatchErr("invalid argument, valid values are: %v", allowed)` [internal/config/map.go:L100,L131].) Setting `auth_user`, `auth_domain`, `envelope`, or `off` loads OK.
- **Silent skip (the dangerous part):** with `require_sender_match auth_user` set, re-running the exact E1 cross-user case (auth=userA, `From: userB`) produced:

```
sign_dkim: signed {"identifier":"userB@maddytest.local"}
```

  i.e. a setting that *looks* like it tightens per-user `From`↔auth binding actually **signs** a message attributing it to userB even though userA authenticated — because `"auth_user"` is never read by `shouldSign`, and setting it *replaces* the default `{envelope, auth}`, so even the envelope check is dropped. The apparent guarantee is inverted: the message is now signed as userB.

**(b) The published DKIM key is unverifiable — even by maddy's own `verify_dkim`.** A reasonable reading of "enable `sign_dkim` and publish the generated `.dns`" is "recipients can verify our mail." In reality the published `p=` is PKCS#1 [internal/modify/dkim/keys.go:L143,L158] while go-msgauth's verifier expects PKIX, and go-msgauth backs both `sign_dkim` and `verify_dkim` [internal/check/dkim/dkim.go:L157-L215], so the output is unverifiable everywhere (Finding b-2, [Q4](#q4--for-the-spoofed-from-case-e4-is-a-signature-applied-does-it-cover-from-what-does-a-verifier-see-and-for-the-signed-case-e3); verbatim verifier error reproduced there).

**(c) The non-local-sender rejection fires at `RCPT TO`, not `MAIL FROM`.** A reasonable reading of "reject non-local senders" is "reject at `MAIL FROM`." In E2, `MAIL FROM:<spoofer@nonlocal.tld>` returned `250`, and the `501 5.1.8` rejection arrived only at `RCPT TO` — because the source block is resolved at the first recipient via `srcBlockForAddr` [internal/msgpipeline/msgpipeline.go:L155-L201], not when the envelope sender is received. Verbatim transcript and log in [Section 3](#e2--non-local-envelope-domain--rejected-at-rcpt-to).

**Evidence.** Config-load error and silent-skip log (verbatim above); [internal/modify/dkim/dkim.go:L151-L152, L249-L330]; [internal/config/map.go:L81,L100,L131]; [docs/man/maddy-filters.5.scd:L518-L544]; [internal/modify/dkim/keys.go:L143,L158]; E2 transcript in [Section 3].


---

## Section 5 — Standards Framing

The standards below are paraphrased in our own words to frame the runtime findings; no normative text is quoted.

- **RFC 6409 (Message Submission for Mail).** This standard establishes that a Message Submission Agent must, by default, refuse a `MAIL` command on an unauthenticated session (the canonical reply being `530`). Crucially, *enforcing submission rights* — rejecting a `MAIL FROM` that is not authorized for the authenticated identity, typically with a `550`/`5.7.1` reply — is described as an **optional** capability ("MAY"), not a requirement. It also recommends rejecting illegal envelope-address syntax with `501` and post-`DATA` header-address problems with `554`. **Implication:** maddy's lack of a per-user `MAIL FROM`↔auth binding in the default configuration is standards-compliant, because that binding is an *optional* MSA feature. This directly frames the [Q6](#q6--does-the-default-config-enforce-senderauthenticated-identity-alignment-or-is-explicit-policy-required) determination and explains why E1 is accepted rather than rejected.
- **RFC 6376 (DKIM Signatures).** DKIM does not require the signing domain (`d=`) to equal the visible `From`-header domain; a cryptographically valid signature can, in principle, exist over a spoofed `From`. **Implication:** a server *could* sign E4 regardless of the `From` domain and still produce a "valid" DKIM signature. maddy instead **refuses** to sign on a `From`/key-domain mismatch [internal/modify/dkim/dkim.go:L287-L291], so the delivered E4 message carries *no* `DKIM-Signature` rather than one whose `d=` is misaligned with `From`.
- **RFC 7489 (DMARC).** It is DMARC *identifier alignment* — not DKIM itself — that requires the `d=` domain to align with the `From` domain. **Implication:** a misaligned-but-valid signature would pass bare DKIM verification yet fail DMARC alignment. maddy's default behavior avoids emitting such a misaligned signature by declining to sign when `From` and key domains differ, delivering the message unsigned instead. (Separately, the published-key format bug in Finding b-2 means even the *aligned* E3 signature is unverifiable by standard verifiers; that is a maddy implementation defect, not a standards question.)

---

## Section 6 — Artifacts & Cleanup Attestation

### 6.1 Throwaway artifacts (all created OUTSIDE the repository, under `/tmp/maddy-investigation/`)

Every artifact below was created outside the maddy source tree and **deleted after evidence capture**. None was ever committed; none is an UPDATE/DELETE on any repository file.

- Built binaries: `bin/maddy`, `bin/maddyctl`.
- The throwaway `maddy.conf` (plus the relay/sink contrast configs used for the `Authentication-Results` comparison and the `require_sender_match` config-load tests).
- The self-signed TLS material: `certs/cert.pem`, `certs/key.pem`.
- The SQLite storage/auth database: `run/all.db`.
- The auto-generated DKIM key and public-key record: `run/dkim_keys/maddytest.local_default.key` and `run/dkim_keys/maddytest.local_default.dns`.
- Captured SMTP transcripts (E1–E5), maddy debug logs, and `.eml`/stored-message header dumps extracted from `run/messages/`.
- The client and DKIM-verifier scripts used to drive E1–E5 and to perform the go-msgauth/dkimpy verification.

### 6.2 Cleanup

All of the above were removed after the evidence in this document was captured. No test database or runtime state remains.

### 6.3 Closing attestation — source repository left byte-for-byte unchanged

The maddy source repository was treated as strictly read-only throughout. After the investigation, the working tree was clean and the HEAD commit was unchanged:

- `git status --porcelain` produced **no output** (empty — no modified, added, or deleted tracked files in the source tree).
- HEAD = `26452dd8dd787dc455278b0fdd296f4a5432c768`.

The **only** committed change for this task is this single document, `blitzy/documentation/maddy_26452dd8dd78.md`, in the destination repository. No existing file in the maddy source tree was created, modified, or deleted. The `/app` tooling tree was never inspected, and no `.blitzyignore` files exist anywhere in the repository.

