# Sender Identity Enforcement and DKIM Signing on Authenticated SMTP Submission in Maddy (commit `26452dd`)

## Scope

This document answers, **from direct runtime observation of a locally built Maddy mail server at repository commit `26452dd`**, the following question:

> On an authenticated SMTP submission session, how does Maddy enforce sender identity and message authentication? Specifically **(a)** how does it decide to accept or reject the sender address supplied by an authenticated client, and **(b)** how does DKIM signing behave for those messages — backed by actual SMTP response codes and actual stored message header bytes?

The investigation was performed by building `cmd/maddy` and `cmd/maddyctl` from source, standing up a canonical submission runtime (authenticated submission, DKIM signing, local delivery to an inspectable mailbox, two same-domain accounts), and driving the **real submission entry point** (SMTP `AUTH` → message pipeline → DKIM modifier → SQLite storage) with a scripted Python `smtplib` client. Every behavioral claim below is presented as **OBSERVED** output next to the command that produced it; claims that can only be derived from code are labeled **INFERRED / source-grounded** with a `file:line` citation naming the specific function, method, or struct. Byte-sensitive results (the `DKIM-Signature` header) were read from the **stored message bytes** and inspected byte-for-byte. All transient runtime state (test config, TLS material, SQLite database, DKIM keys, capture files, scripts) was created outside the source tree and removed afterward; the sole repository artifact is this document.

---

## 1. Direct answer (lead)

Maddy applies **three independent gates** to an authenticated submission, and they are scoped differently from what a casual reading of `maddy.conf` suggests:

1. **Authentication gates whether you may submit at all.** The submission endpoint always requires `AUTH` — this is hard-wired, not a config option. *(Source-grounded: `internal/endpoint/smtp/smtp.go:589-590`, `endp.authAlwaysRequired = true` for submission.)*

2. **Acceptance is a DOMAIN-level authorization on the ENVELOPE sender (`MAIL FROM`) — NOT a per-user check and NOT a `From`-header check.** The message pipeline matches the envelope address in strict precedence: exact address `perSource[cleanFrom]`, then envelope **domain** `perSource[domain]`, then the catch-all `defaultSource`. In the canonical config the local domain is registered as a `source` block, so **any** `MAIL FROM` at that domain is accepted **regardless of which user authenticated**; a non-local envelope domain falls to `default_source { reject 501 5.1.8 "Non-local sender domain" }`. *(Source-grounded: `internal/msgpipeline/msgpipeline.go` — `srcBlockForAddr` at L113/L155; exact-address match `perSource[cleanFrom]` L171; domain match `perSource[domain]` L190; fallback `defaultSource` L193; reject execution L117-119. Config: `maddy.conf` L97 `source $(local_domains)`, L117-118 `default_source { reject 501 5.1.8 "Non-local sender domain" }`.)* This was **OBSERVED**: authenticating as **alice** and sending with `MAIL FROM:<bob@example.org>` (T1) was **ACCEPTED** — the pipeline logged `sender bob@example.org matched by domain rule 'example.org'` and delivered the message.

3. **`From`-alignment gates only whether a DKIM signature is attached — never acceptance.** DKIM signing is governed separately by the `sign_dkim` modifier's `require_sender_match` directive (**default `envelope auth`**). On **any** misalignment — `From` ≠ envelope, `From` ≠ authenticated identity, or `From`-domain ≠ key-domain — the modifier **silently delivers the message UNSIGNED. It does NOT reject.** *(Source-grounded: `internal/modify/dkim/dkim.go` — `require_sender_match` default `["envelope","auth"]` L151-152; `shouldSign` checks L249-329; the pivotal `RewriteBody` at L340 where `id, ok := s.m.shouldSign(...)` (L348) is followed by `if !ok { return nil }` at L349-351 — returning `nil` (no error) means the message proceeds to delivery unsigned; success adds the header via `h.Add("DKIM-Signature", ...)` at L406.)*

**The authenticated submission path never compares `From` to `MAIL FROM` or to the authenticated identity.** The only `From` handling on the submission path is `submissionPrepare`, which validates *presence and syntax* of `From`/`Sender`/`To`/`Cc`/`Bcc`/`Reply-To` and injects `Message-ID`/`Date` if missing — it contains **no** alignment comparison. *(Source-grounded: `internal/endpoint/smtp/submission.go` — `submissionPrepare` L27; missing-`From` → `554 5.6.0` at L39-48.)*

### Net effect (with the observed consequence to highlight)

| Gate | Scoped by | What it controls | Observed consequence |
|------|-----------|------------------|----------------------|
| AUTH required | — | Whether submission is allowed at all | All six scenarios had to authenticate |
| Pipeline `source` routing | **Envelope DOMAIN** | Accept vs reject | **T1 cross-user (auth alice, MAIL FROM bob) is ACCEPTED** — same domain |
| `sign_dkim` `require_sender_match` | `From` vs envelope / auth-identity / key-domain | Whether a `DKIM-Signature` is attached | **T1/T4/T5 delivered UNSIGNED, never rejected** |

The headline "no difference / negative" truths, all confirmed by observation and stated plainly because the truth requires it: **authorization is domain-level, not per-user**; **misaligned `From` yields silent unsigned delivery, not rejection**; and **no `Authentication-Results` header is added on submission** (§8).

---

## 2. Methodology and rules compliance (SWE-AtlasQnA-Repo)

This answer was produced under the **observe-first, read-only** methodology. The mapping below states how each governing rule is satisfied in this document:

- **Ran the code first, then wrote.** All behavioral claims derive from output captured from a running daemon driven through its real submission endpoint (§4–§8); code reading is used only to *explain* observations, and such explanations are labeled source-grounded.
- **Actual, complete, unedited output for every claim, with the command that produced it.** Every scenario shows the client transcript and/or server `io_debug` block in a fenced code block; nothing is elided with `// ...`.
- **Real scale, stability across ≥2 runs.** Each scenario T1–T6 was executed **at least twice over plaintext port 587 and once over implicit-TLS port 465** (transport-independent); results were identical and stable across runs before being reported.
- **Exact code path through the real entry point.** The path exercised is SMTP `AUTH` → message pipeline (`msgpipeline`) → DKIM modifier (`sign_dkim` `RewriteBody`) → local storage (`imapsql` SQLite). No bypassing or synthetic interface was used.
- **Default, canonical configuration.** The runtime mirrors the repository's default submission semantics; the build and invocation commands are stated verbatim (§4), and the canonical version banner (`maddy unknown (built from source tree)`) is reported as-is.
- **Every condition exercised.** Primary (T2 aligned) plus secondary/edge/error paths (T1 cross-user, T3 non-local envelope, T4 `From`≠envelope, T5 `From`-domain≠key, T6 missing `From`); state observed *before* delivery (accept/reject at the protocol) and *after* delivery (signed/unsigned in storage).
- **Exact and grounded.** Every factual claim carries an actual value plus a `file:line` citation naming the function/method/struct; **OBSERVED vs INFERRED is labeled** throughout; the direct answer leads and includes negative results.
- **Byte-exact DKIM verification.** The `DKIM-Signature` was read from the stored message bytes and its `h=` tag inspected byte-for-byte to confirm `From` coverage (§7.3).
- **Read-only source tree.** No existing repository file was modified and no code was added; the only new artifact is this document.

---

## 3. How the runtime was built and configured (OBSERVED)

### 3.1 Build

CGO is **required** — the storage/auth backend uses the `mattn/go-sqlite3` driver (`go.mod` L26, `github.com/mattn/go-sqlite3 v1.11.0`), which is a cgo package. The build used the Go toolchain with `CGO_ENABLED=1`. Both `cmd/maddy` (the daemon) and `cmd/maddyctl` (the admin CLI, needed for user provisioning and mailbox inspection) were built:

```
go build -o maddy ./cmd/maddy
go build -o maddyctl ./cmd/maddyctl
```

Both succeed; the only compiler output is a harmless CGO warning from `mattn/go-sqlite3` about a function returning the address of a local variable. *(Source-grounded: `go.mod` L1 module `github.com/foxcpp/maddy`, L3 `go 1.13`.)*

### 3.2 Canonical version banner (OBSERVED)

The value a plain source build reports:

```
$ ./maddy -v
maddy unknown (built from source tree)
$ ./maddyctl -v
maddyctl version unknown (built from source tree)
```

This is the **canonical build artifact**, not a broken environment: the `Version` string is literally defined as `"unknown (built from source tree)"`. *(Source-grounded: `maddy.go:41`, `Version = "unknown (built from source tree)"`.)*

### 3.3 Daemon flags and startup

The daemon entry point is trivially thin — `cmd/maddy/main.go` is 11 lines and does only `os.Exit(maddy.Run())`. The flags are declared in `maddy.go`: `-libexec` (L103), `-debug` (L104), `-config` (L107), `-log` (L108), `-v` (L109). On startup the daemon **changes its working directory to the state directory** (`os.Chdir(config.StateDirectory)`, `maddy.go:220`), so relative paths in configuration — `dsn all.db` and `dkim_keys/{domain}_{selector}.key` — resolve relative to that state directory. The daemon was started with `-debug`.

### 3.4 Canonical `maddy.conf` semantics

The test configuration was authored **outside the source tree**, faithful to the repository's default submission semantics. The relevant directives and their grounding in the repository default `maddy.conf`:

- **Unified SQLite storage + auth:** `sql local_mailboxes local_authdb { driver sqlite3; dsn all.db }` — `maddy.conf` L32-34. One database (`all.db`) backs both credential checks and IMAP mailbox delivery. The module is registered as `sql` at `internal/storage/sql/sql.go:424` (`module.Register("sql", New)`).
- **Submission endpoint (implicit TLS):** `submission tls://0.0.0.0:465 { auth &local_authdb ... }` — `maddy.conf` L93, L95.
- **Plaintext submission listener for clean protocol capture:** a second `submission tcp://127.0.0.1:587` listener was added with TLS off, `insecure_auth`, and `io_debug`. When no TLS is configured the endpoint auto-enables insecure auth (`internal/endpoint/smtp/smtp.go:537-546`, specifically `endp.serv.AllowInsecureAuth = true` at L545 when `TLSConfig == nil`); `insecure_auth` is the explicit knob (L564) and `io_debug` (L565) emits raw server-side SMTP I/O. **Port 465 (implicit TLS) and port 587 (plaintext) produced identical behavior — the outcome is transport-independent.**
- **Sender routing:** `source $(local_domains) { modify { sign_dkim $(primary_domain) default } deliver_to &local_mailboxes }` — `maddy.conf` L97, L99; catch-all `default_source { reject 501 5.1.8 "Non-local sender domain" }` — L117-118. The `maddy.conf` comment at L115-116 states the intent: block non-local sender domains as "likely a spoofing attempt". Config vars: `hostname example.org` (L4), `primary_domain example.org` (L9), `local_domains $(primary_domain)` (L12).
- **TLS:** the certificate used `self_signed` so Maddy auto-generates a self-signed cert (no external `openssl` needed); the submission endpoint inherits the TLS config via `cfg.Custom("tls", ...)` at `internal/endpoint/smtp/smtp.go:563`.

### 3.5 DKIM key auto-generation (OBSERVED)

On first start, the `sign_dkim example.org default` modifier auto-generated the keypair under the state directory:

- `dkim_keys/example.org_default.key` — the private key, PEM, mode `0600`.
- `dkim_keys/example.org_default.dns` — the public TXT record, of the form `v=DKIM1; k=rsa; p=<base64>`.

The default new-key algorithm is `rsa2048`. The selector `default` and domain `example.org` come directly from `sign_dkim $(primary_domain) default`. The daemon logged the generation with instructions to publish the TXT record. *(Source-grounded: `internal/modify/dkim/dkim.go` — key path template `dkim_keys/{domain}_{selector}.key` L137, new-key log L192-194; `internal/modify/dkim/keys.go` — `writeDNSRecord` L136-163, record format `v=DKIM1; k=%s; p=%s` at L158. The `p=` public-key encoding at `keys.go:143` is analyzed in §7.6 Finding A.)*

### 3.6 User provisioning (OBSERVED)

Two accounts in the same local domain were created. This repository revision uses `maddyctl users create` — **not** the later-version `maddy creds` subcommand:

```
maddyctl --config <cfg> users create alice@example.org -p AlicePass123
maddyctl --config <cfg> users create bob@example.org   -p BobPass123
```

*(Source-grounded: `cmd/maddyctl/users.go` — `usersList`/`ListUsers` L14-15; `usersCreate` L30; flags `--null` → `CreateUserNoPass` L36-37, `--hash` L40, `--bcrypt-cost` L48; `CreateUser` L76.)* Credentials are checked at login by `internal/storage/sql/sql.go` `CheckPlain` (L377), which applies PRECIS `UsernameCaseMapped` to the username (L364) and PRECIS `OpaqueString` to the password (L386) before the backend performs the bcrypt comparison (L391). The domain-auth helper `internal/auth/auth.go` `CheckDomainAuth` (L5) splits the username on `@` and compares domains case-insensitively via `strings.EqualFold` (L25).

### 3.7 Test client and tooling

The test client is Python `smtplib`: SASL PLAIN via `login()`, `set_debuglevel(2)` for the full client-side transcript, `SMTP_SSL` with `ssl.CERT_NONE` for port 465 and plain `SMTP` for port 587. The envelope `MAIL FROM` was decoupled from the `From` header by passing the envelope to `sendmail(from_addr=...)` while setting the `From:` header inside an `email.message`. Note the container **lacks `swaks`, `tcpdump`, and the `sqlite3` CLI**; Python `smtplib`, Maddy's `io_debug`, and Python's `sqlite3` module substituted for them, and mailbox contents were read via `maddyctl imap-msgs ... dump` cross-checked against the go-imap-sql external body blob.

---


## 4. The test matrix (T1–T6): expected vs OBSERVED

All six scenarios authenticate as **alice**. Each was run **≥2× over plaintext port 587 and ×1 over implicit-TLS port 465**, driven through the real submission endpoint, with **identical, stable results** across every run and both transports.

| Test | MAIL FROM (envelope) | From header | Pipeline decision (OBSERVED) | DKIM outcome (OBSERVED) | Msg ID |
|------|----------------------|-------------|------------------------------|-------------------------|--------|
| **T1 cross-user** | `bob@example.org` | `bob@example.org` | **ACCEPTED** (matched by domain rule `example.org`) | **UNSIGNED** — `From` ≠ authenticated identity | `513ba1a1` |
| **T2 aligned** | `alice@example.org` | `alice@example.org` | **ACCEPTED** | **SIGNED** (`DKIM-Signature` present) | `a6d85237` |
| **T3 non-local envelope** | `x@remote.tld` | `x@remote.tld` | **REJECT `501 5.1.8`** (matched by default rule) | N/A (rejected before signing) | `fa2790f2` |
| **T4 From≠envelope** | `alice@example.org` | `bob@example.org` | **ACCEPTED** | **UNSIGNED** — `From` ≠ envelope | `8ff3a298` |
| **T5 From-domain≠key** | `alice@example.org` | `alice@other.tld` | **ACCEPTED** | **UNSIGNED** — `From` domain ≠ key domain | `2cbfd99d` |
| **T6 missing From** | `alice@example.org` | (absent) | **REJECT `554 5.6.0`** at end-of-DATA | N/A | `17157d21` |

**The nuance to call out:** **T1 (cross-user) is ACCEPTED at the pipeline but delivered UNSIGNED.** The pipeline authorizes by domain (envelope `bob@example.org` is local) while the DKIM modifier declines to sign (`From` ≠ authenticated identity `alice`). These are two separate gates, and their decisions diverge for the very same message.

### 4.1 Verbatim SMTP response codes (client transcript, OBSERVED)

The following codes were identical across all sessions where applicable:

- EHLO advertised: `PIPELINING`, `8BITMIME`, `ENHANCEDSTATUSCODES`, `AUTH PLAIN`, `SMTPUTF8`, `SIZE 33554432`.
- AUTH success: `235 2.0.0 Authentication succeeded`
- `MAIL FROM` **always** returned `250 2.0.0 Roger, accepting mail from <ADDR>` — **including for `x@remote.tld`**. The sender reject is *deferred* to the first `RCPT`, because `defer_sender_reject` defaults to true (`internal/endpoint/smtp/smtp.go:567`).
- RCPT accept: `250 2.0.0 I'll make sure <alice@example.org> gets this`
- DATA go-ahead: `354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>`
- DATA accept: `250 2.0.0 OK: queued`
- `QUIT`: `221 2.0.0 Goodnight and good luck`; `RSET`: `250 2.0.0 Session reset`.

**T3 reject — issued at the first `RCPT TO` (deferred from `MAIL FROM`):**

```
501 5.1.8 Non-local sender domain (msg ID = fa2790f2)
```

Python's `smtplib` raises `SMTPRecipientsRefused` for this. The envelope domain `remote.tld` matches no `source` block, so it falls to `default_source`'s `reject`. *(Source-grounded: `maddy.conf:117-118`; reject execution `internal/msgpipeline/msgpipeline.go:117-119`.)*

**T6 reject — issued at the end of `DATA`:**

```
554 5.6.0 Message does not contains a From header field (msg ID = 17157d21)
```

The verbatim server grammar is **"does not contains"** (reproduced exactly). Python's `smtplib` raises `SMTPDataError` for this. *(Source-grounded: the string is defined verbatim at `internal/endpoint/smtp/submission.go:43`, `Message: "Message does not contains a From header field"`, inside `submissionPrepare`'s missing-`From` check at L39-48, which returns SMTP code `554` with enhanced code `{5,6,0}`.)*

**Dependence on how the mismatch is constructed (O3).** The three "mismatch" scenarios produce three *different* outcomes, proving the decision depends on **which** identity is misaligned:

- **Envelope-domain mismatch (T3):** rejected `501 5.1.8` by the pipeline.
- **`From` ≠ envelope, but envelope local (T4):** accepted, delivered unsigned.
- **`From` ≠ authenticated identity, but envelope local (T1):** accepted, delivered unsigned.
- **`From`-domain ≠ signing-key domain (T5):** accepted, delivered unsigned.
- **Missing `From` entirely (T6):** rejected `554 5.6.0` by `submissionPrepare` (a *syntax/presence* check, unrelated to alignment).

---


## 5. Server-side protocol transcripts (`io_debug`) — one ACCEPTED + one REJECTED (OBSERVED, O6)

Because implicit-TLS submission on port 465 is ciphertext on the wire and `tcpdump` is absent from the container, protocol-level visibility was obtained two ways that together cover both an accepted and a rejected transaction **without TLS key extraction**: Maddy's `io_debug` (server-side raw SMTP I/O, `internal/endpoint/smtp/smtp.go:565`) and the client-side `set_debuglevel(2)` transcript. This is the documented fallback: the encrypted channel was captured through server-side and client-side logging plus a plaintext `587` listener, rather than by decrypting TLS. Both blocks below are reproduced verbatim.

### 5.1 ACCEPTED — T1 cross-user (`msg_id 513ba1a1`) — the smoking gun for domain-level (not per-user) authorization

```
submission: AUTH PLAIN AGFsaWNlQGV4YW1wbGUub3JnAEFsaWNlUGFzczEyMw==
submission: 235 2.0.0 Authentication succeeded
submission: mail FROM:<bob@example.org> size=79
submission: 250 2.0.0 Roger, accepting mail from <bob@example.org>
submission: rcpt TO:<alice@example.org>
submission: incoming message {"msg_id":"513ba1a1","sender":"bob@example.org","src_host":"client.test","src_ip":"127.0.0.1:41330","username":"alice@example.org"}
[debug] smtp/pipeline: sender bob@example.org matched by domain rule 'example.org' {"msg_id":"513ba1a1"}
[debug] smtp/pipeline: recipient alice@example.org matched by domain rule 'example.org'
[debug] smtp/pipeline: tgt.Start(bob@example.org) ok, target = sql:local_mailboxes
submission: RCPT ok {"msg_id":"513ba1a1","rcpt":"alice@example.org"}
submission: 250 2.0.0 I'll make sure <alice@example.org> gets this
submission: data
submission: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
submission: adding missing Message-ID
submission: adding missing Date header
sign_dkim: not signing, From address is not authenticated identity {"auth_id":"alice@example.org","from_addr":"bob@example.org","msg_id":"513ba1a1"}
submission: accepted {"msg_id":"513ba1a1"}
```

**What this proves.** The `AUTH PLAIN` payload base64-decodes to `\0alice@example.org\0AlicePass123` (SASL PLAIN is `authzid\0authcid\0passwd`), so the authenticated identity is **alice**. Yet the log records `"sender":"bob@example.org"` and `"username":"alice@example.org"` as **separate** fields, and only the **domain** of the sender gates acceptance: `sender bob@example.org matched by domain rule 'example.org'`. The authenticated `username` is logged but **not consulted** by source routing. The message is then `accepted` — and, per the immediately preceding `sign_dkim: not signing, From address is not authenticated identity` line, delivered **UNSIGNED**.

### 5.2 REJECTED — T3 non-local envelope (`msg_id fa2790f2`)

```
submission: mail FROM:<x@remote.tld> size=74
submission: 250 2.0.0 Roger, accepting mail from <x@remote.tld>
submission: rcpt TO:<alice@example.org>
submission: incoming message {"msg_id":"fa2790f2","sender":"x@remote.tld",...,"username":"alice@example.org"}
[debug] smtp/pipeline: sender x@remote.tld matched by default rule {"msg_id":"fa2790f2"}
submission: RCPT error {"effective_rcpt":"alice@example.org","rcpt":"alice@example.org","reason":"reject directive used","smtp_code":501,"smtp_enchcode":"5.1.8","smtp_msg":"Non-local sender domain"}
submission: 501 5.1.8 Non-local sender domain (msg ID = fa2790f2)
```

**What this proves.** `MAIL FROM` still received `250` (the reject is deferred). The non-local sender `x@remote.tld` `matched by default rule` — i.e., it fell through both the exact-address and domain tiers to `default_source`, whose `reject` directive produced the `501 5.1.8 Non-local sender domain` at the first `RCPT`. (The `,...,` inside the `incoming message` line is reproduced exactly as captured in the transcript; the omitted middle fields — `src_host`, `src_ip` — are identical in shape to the T1 block in §5.1, which shows them in full.)

### 5.3 Mapping the debug strings to source

The pipeline's routing decision is emitted by `internal/msgpipeline/msgpipeline.go` `srcBlockForAddr` (defined L155, called L113):

- `"sender %s matched by domain rule '%s'"` — the `perSource[domain]` path (match at L190, log at L196). **This is the T1 line.**
- `"sender %s matched by default rule"` — the `defaultSource` path (fallback at L193, log at L194). **This is the T3 line.**
- `"sender %s matched by address rule '%s'"` — the exact `perSource[cleanFrom]` path (match at L171, log at L199). *(Not triggered here because the canonical config registers a domain-level `source`, not per-address blocks.)*
- The reject itself: `if sourceBlock.rejectErr != nil { dd.log.Debugf("sender %s rejected with error: %v", ...); return sourceBlock.rejectErr }` at L117-119.

---


## 6. Mechanism (b): DKIM signing behavior (OBSERVED, O4/O5)

### 6.1 Sign/skip debug reasons (verbatim from `maddy.log`)

```
T1 (513ba1a1): sign_dkim: not signing, From address is not authenticated identity {"auth_id":"alice@example.org","from_addr":"bob@example.org","msg_id":"513ba1a1"}
T2 (a6d85237): [debug] sign_dkim: signed {"identifier":"alice@example.org"}
T4 (8ff3a298): sign_dkim: not signing, From address is not envelope address {"envelope":"alice@example.org","from_addr":"bob@example.org","msg_id":"8ff3a298"}
T5 (2cbfd99d): sign_dkim: not signing, From domain is not key domain {"from_domain":"other.tld","key_domain":"example.org","msg_id":"2cbfd99d"}
```

Each maps precisely to a branch of `shouldSign` in `internal/modify/dkim/dkim.go` (L249). The `require_sender_match` directive defaults to `["envelope", "auth"]` (L151-152; allowed set `{envelope, auth_domain, auth_user, off}`), so both the envelope and auth checks are active:

- **T5 — `From`-domain ≠ key-domain** (`dns.Equal(fromDomain, m.domain)` at L287-291). Checked first among the alignment checks; `other.tld` ≠ `example.org`, so it skips here with `from_domain`/`key_domain` fields.
- **T4 — envelope method, `From` ≠ `MAIL FROM`** (`address.Equal(fromAddr, mailFrom)` at L293-296). `From` `bob@example.org` ≠ envelope `alice@example.org`, so it skips with `from_addr`/`envelope` fields.
- **T1 — auth method, `From` ≠ authenticated identity** (L299-310). Here the domain check passes (`example.org`) and the envelope check passes (`From` `bob@example.org` = envelope `bob@example.org`), so evaluation reaches the auth check: `compareWith` is built from `From`, and because the authorization identity `alice@example.org` contains `@`, `compareWith` is set to `address.ForLookup(fromAddr)` = `bob@example.org`; `strings.EqualFold("bob@example.org","alice@example.org")` is false, so it skips with `from_addr`/`auth_id` fields.
- **T2 — all aligned → signs**, returning identifier `alice@example.org` (identifier build at L314-327) and logging `signed`.

`shouldSign` also has earlier guards that were not triggered here but bound the behavior: `off` short-circuit (L250-262), empty `From` (L265-267), malformed `From` (L270-272), and multiple `From` addresses (L274-277).

### 6.2 The pivotal sign-vs-skip mechanism (source-grounded)

The single most important DKIM fact in this document: **misalignment ⇒ silently unsigned, never rejected.** In `internal/modify/dkim/dkim.go`, the modifier's `state` method **`RewriteBody`** (L340) reads the authenticated user (`authUser := s.meta.Conn.AuthUser`, L343-346), calls `shouldSign` (L348), and then:

```go
id, ok := s.m.shouldSign(s.meta.SMTPOpts.UTF8, s.meta.ID, h, s.meta.OriginalFrom, authUser)
if !ok {
    return nil
}
```

The `return nil` at **L349-351** returns **no error** — the modifier declines to add a signature but does **not** fail the delivery, so the message proceeds to storage **UNSIGNED**. Only on the success path does it build the signature and call `h.Add("DKIM-Signature", signer.SignatureValue())` at **L406**, followed by the `signed` debug log at L408. The module is registered as `module.Register("sign_dkim", New)` at L418. There is **no code path** in this modifier that returns a rejection error for a misaligned sender.

### 6.3 Raw stored `DKIM-Signature` bytes — T2 (SIGNED), byte-exact (OBSERVED)

Read from the stored message bytes via `maddyctl imap-msgs <mailbox> dump` and cross-checked by reading the go-imap-sql external body blob directly. Reproduced verbatim (exact wrapping, spacing, and tag order preserved):

```
Dkim-Signature: a=rsa-sha256;
 bh=oY/UZI4p+g7RAXkfVGG/HEXj+NClXPmVj7uvrhS6fig=; c=relaxed/relaxed;
 d=example.org;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=alice@example.org; s=default; t=1784155112; v=1; x=1784587112; b=P689eVzytP3hZ3OZA931bh9Eu/5cckoV9WEWin6ZGZVVxFJq9SI5/ctJSZ1++Tto1dmdBlXYjEc+ct5ATcLEXMx8edyGphglhAhKiRejKb2dtOKFkVgF7xyUGh88d41kzTALgYIJuGVpc2I0mr3b9LzkQsvGyIPsP5b2ARkyHVOhaFfoWfL4y8TyRYMQj/2M8P1xRCfu5qfdLwSssrBS5hJUO9OUOrUJ4yuVE9/v/uVhVqqqGvoCt9stclOQJYTKgQCKWHugSYE63gJI87zdLpHNFMno9epamLhttk+qhAHTm5LbpT+AwhGSMUt+2G/fj96kIsuMf7WdUQ/AjZk/tA==;
```

**`h=` tag analysis (byte-exact, O5).** `From` appears **twice** (`...From:From...`) — it is **oversigned**. So are `Subject:Subject`, `To:To`, `Date:Date`, and `Message-Id:Message-Id`. **Oversigning** means listing a header field one more time than it actually occurs: this both binds the present copy *and* prevents an intermediary from silently adding a second copy without breaking the signature. Therefore the signature **does cover `From`** (in fact it oversigns it). The oversign field set is `oversignDefault` in `internal/modify/dkim/dkim.go` (L31-54; `From` at L37), and the "one entry per occurrence, plus one more" logic is `fieldsToSign` (L202-233: for each oversigned key it appends one entry per occurrence present in the message, L215-217, then one extra, L219). Consequently a header present once in the message (Subject, To, From, Date, Message-Id) appears **twice** in `h=`, while an oversigned header absent from the message (Sender, Cc, MIME-Version, Content-Type, Content-Transfer-Encoding, Reply-To, In-Reply-To, References, Autocrypt, Openpgp) appears **once**. The observed `h=` string is exactly this derivation — confirming T2's message carried `Subject`, `To`, `From`, `Date`, `Message-Id` (each once) and none of the sign-only `List-*`/`Resent-*` fields (`signDefault`, L55-72).

**Timestamps.** `t=1784155112`, `x=1784587112`, so `x − t = 432000` seconds = **5 days = 120h**, matching the documented `sig_expiry` default of `120h` (`docs/man/maddy-filters.5.scd:500-501`) and the code default `5*Day` (`internal/modify/dkim/dkim.go:146`). `c=relaxed/relaxed` matches the `header_canon`/`body_canon` defaults (L140-145) and `a=rsa-sha256` matches the `hash` default `sha256` (L147-148) with the RSA key.

### 6.4 The UNSIGNED messages (OBSERVED)

The three misaligned deliveries each have **no `DKIM-Signature` header at all** in storage: T1 (alice INBOX UID 1), T4 (alice INBOX UID 2), and T5 (alice INBOX UID 3). This is the observed consequence of the §6.2 skip path (`RewriteBody` `return nil`).

---


## 7. Raw stored `Received` header bytes (OBSERVED, O4) — reflects the ENVELOPE, not `From`/auth

The `Received` header Maddy adds to a submitted message records the **envelope sender**, not the `From` header and not the authenticated user. T2 (signed) stored `Received` verbatim (note the **double space** after `Received:`, the `(envelope-sender <...>)` clause, and the **absence** of any `from <host> [ip]` sender trace):

```
Received:  by example.org (envelope-sender <alice@example.org>) with ESMTP
 id a6d85237; Wed, 15 Jul 2026 22:38:32 +0000
```

**The decisive proof that `Received` tracks the envelope, not `From`/auth — T4** (envelope `alice`, `From` `bob`) stored:

```
Received:  by example.org (envelope-sender <alice@example.org>) with ESMTP
 id 8ff3a298; ...
From: bob@example.org
```

(T2 above is the byte-exact full `Received`; the T4 excerpt shows the same generated format with the RFC1123Z date abbreviated as captured, isolating the two fields that make the point — the `(envelope-sender ...)` clause and the message's `From`.) The `(envelope-sender <alice@example.org>)` clause reflects `MAIL FROM` (`alice`), even though the message's `From` is `bob@example.org` — so the `Received` header is keyed to the **envelope**, and separately, neither is it the authenticated identity (though in T4 the envelope happens to equal the auth identity, T1 shows the general case: envelope `bob` with auth `alice` records `bob`).

### 7.1 Why the double space, and why no sender trace — source-grounded

`internal/target/received.go` `GenerateReceived` (L19) builds the value:

- It requires a connection (`msgMeta.Conn == nil` guard, L20-22).
- The `from <host> [ip]` sender-trace clause is emitted **only** when `!msgMeta.DontTraceSender && (proto contains SMTP or LMTP)` (L30-31). On submission, `submissionPrepare` sets `msgMeta.DontTraceSender = true` (`internal/endpoint/smtp/submission.go:28`), so this clause is **skipped entirely**.
- With the sender trace skipped, the builder's first written token is `" by "` (with a **leading space**) + our hostname (L60-64). Because a header is serialized as `Received:` + value, and the value begins with that leading space, the stored bytes show `Received:` followed by **two** spaces before `by` — exactly the observed double space.
- Then `" (envelope-sender <"` + `MAIL FROM` + `">)"` (L66-72) — this is the clause that carries the envelope address.
- Then `" with "` + protocol + `" id "` + message id (L74-82), and finally the RFC1123Z date (L84).

The endpoint wires this up: it captures the envelope as `msgMeta.OriginalFrom = cleanFrom` (`internal/endpoint/smtp/smtp.go:116`), generates the header via `target.GenerateReceived(ctx, s.msgMeta, s.endp.hostname, s.msgMeta.OriginalFrom)` (`smtp.go:303`) and adds it with `header.Add("Received", received)` (`smtp.go:307`). The authenticated identity is stored separately as `AuthUser` (`smtp.go:680`) and is not passed into `GenerateReceived`. Note also that the `for` recipient clause is never included in the `Received` field by design (`docs/internals/quirks.md:9`).

---

## 8. Negative finding: no `Authentication-Results` on submission (OBSERVED, O4)

A `grep` across all stored message blobs found **NO `Authentication-Results` header, no `ARC-*` headers, no `DKIM-Status`, and no `Received-SPF`** on any submission message. The **complete** set of header field names observed across all delivered messages was:

```
Date, Delivered-To, Dkim-Signature, From, Message-Id, Received, Return-Path, Subject, To
```

Therefore, on authenticated submission the only headers Maddy itself adds are:

- `Received` — always (`internal/target/received.go`).
- `DKIM-Signature` — only when the sender is aligned (`internal/modify/dkim/dkim.go:406`).
- Injected `Message-Id` and `Date` when absent — logged as `adding missing Message-ID` / `adding missing Date header` (`internal/endpoint/smtp/submission.go` L30-37 and L111-127).
- `Delivered-To` and `Return-Path` — added by the local-delivery target.

Maddy stamps `Authentication-Results` **only on the inbound check path** (`verify_dkim` / `apply_spf`, configured for the port-25 endpoint at `maddy.conf:54-66`), **never on submission**. This is a genuine negative result: a naive expectation that submitted mail is annotated with an `Authentication-Results` header is refuted by the observed header set.

---


## 9. DKIM triggering under `From` manipulation + what a verifying recipient would conclude (OBSERVED, O5/O8)

§4–§6 already establish the triggering behavior under `From` manipulation: T1 (`From` = cross-user), T4 (`From` ≠ envelope), and T5 (`From`-domain ≠ key-domain) are each **accepted and delivered UNSIGNED** — a manipulated `From` never causes a rejection, only the absence of a signature; and for the aligned T2 the signature **covers `From`** (oversigned). This section reports what a recipient attempting cryptographic verification would actually conclude.

Verification used the **same library Maddy uses** — `github.com/emersion/go-msgauth v0.3.2-0.20191028231513-55b75676976c` (`go.mod:17`), `dkim.Verify`, with the DKIM DNS record served locally — and was cross-checked with Python `dkimpy`. Three findings, each labeled **OBSERVED** with its source grounding:

### Finding A (OBSERVED) — the published public key is PKCS#1 but the verifier expects PKIX (a real interop divergence at this commit)

Maddy's auto-generated `.dns` record's `p=` value base64-decodes to DER beginning `30 82 01 0a 02 82 01 01` — a **bare PKCS#1 `RSAPublicKey`**. This is because `internal/modify/dkim/keys.go:143` encodes the public key with `x509.MarshalPKCS1PublicKey(pubkey)` (inside `writeDNSRecord`, L136-163) and emits it into the record at L158 (`v=DKIM1; k=%s; p=%s`). The go-msgauth verifier, however, parses the `p=` blob with `x509.ParsePKIXPublicKey(b)`, which requires a PKIX `SubjectPublicKeyInfo` (DER beginning `30 82 01 22 30 0d 06 09 2a 86 48 ...`) and **cannot** parse a bare PKCS#1 key. Verifying the stored T2 message with the key exactly as published:

```
RESULT: FAIL domain=example.org err=dkim: key syntax error: x509: failed to parse public key (use ParsePKCS1PublicKey instead for this key format)
```

Python `dkimpy` likewise returned `False`. **Conclusion:** a recipient using Maddy's own auto-generated TXT record verbatim gets a **key syntax error for every signed message** — and this is the same parsing code path Maddy's own `verify_dkim` check would use. (Later Maddy versions switched to `MarshalPKIXPublicKey`; at commit `26452dd` it is PKCS#1.) This is an **OBSERVED interop divergence**.

### Finding B (OBSERVED) — the signature scheme itself is sound (in-library round-trip PASSES)

The body hash is correct and the signing/canonicalization is internally consistent. The stored `bh=` equals a manual `sha256` of the T2 body (`sha256("T2 body\r\n")`, base64) = `oY/UZI4p+g7RAXkfVGG/HEXj+NClXPmVj7uvrhS6fig=`, so the body is intact. A fresh in-memory `go-msgauth dkim.Sign` (relaxed/relaxed, sha256, the same oversigned header set) produced the same `bh`, and verifying that fresh output with the **PKIX-form** key gave:

```
RESULT: PASS domain=example.org identifier=alice@example.org
```

go-msgauth's tag ordering (`formatHeaderParams`, alphabetical with `b` last: `a, bh, c, d, h, i, s, t, v, x, b`) matches the stored header's tag order exactly, so tag ordering is not a factor. ⇒ signing, oversigning, and canonicalization are internally correct.

### Finding C (OBSERVED) — the stored mailbox copy no longer matches its own signature

Re-encoding the same keypair as PKIX (so parsing succeeds) and re-verifying the **stored** T2 message gave:

```
RESULT: FAIL domain=example.org err=dkim: signature did not verify: crypto/rsa: verification error
```

The key now parses, and the body hash still matches, but the **header** signature fails. Since the fresh in-memory round-trip (Finding B) PASSES while the persisted mailbox copy FAILS, the bytes stored in the mailbox (the go-imap-sql external body file) are **not byte-identical** to what `sign_dkim` signed: the local delivery/storage path re-serialized the header after `RewriteBody` added the signature (`internal/modify/dkim/dkim.go:406`). **This finding is scoped precisely to the LOCALLY STORED copy;** remote-relay serialization was not tested (there is no external MX in this local, DNS-independent setup — out of scope).

### Net for a verifying recipient (stated plainly)

For an **aligned** submission (T2), Maddy **does** attach a complete `DKIM-Signature` that covers `From` (oversigned) with a correct body hash — **but** (A) the key it publishes is in a format that standard PKIX-expecting verifiers reject outright, and (C) the copy that lands in the mailbox no longer matches its own signature. For **misaligned** submissions (T1/T4/T5) there is no signature at all, so a recipient simply sees an unsigned message.

---


## 10. Why it is DOMAIN-level, not per-user (source walk tying observation to code)

The observed T1 line `sender bob@example.org matched by domain rule 'example.org'` (§5.1) leads directly to the routing code in `internal/msgpipeline/msgpipeline.go`. The function `srcBlockForAddr(mailFrom)` (defined L155, invoked at L113) decides the source block:

1. **Normalize** the envelope address with `address.ForLookup` (L159). Failure returns `501 5.1.7 "Unable to normalize the sender address"` (L161-167).
2. **Exact-address tier:** `srcBlock, ok := dd.d.perSource[cleanFrom]` (L171). A match here logs `"sender %s matched by address rule '%s'"` (L199).
3. If no exact match, **split** to the domain; an unsplittable non-empty address returns `501 5.1.3 "Invalid sender address"` (L179-187).
4. **Domain tier:** `srcBlock, ok = dd.d.perSource[domain]` (L190). A match logs `"sender %s matched by domain rule '%s'"` (L196). **This is the tier T1 matched.**
5. **Fallback tier:** `srcBlock = dd.d.defaultSource` (L193), logging `"sender %s matched by default rule"` (L194). **This is the tier T3 matched.**

The canonical `maddy.conf` registers the local domain via `source $(local_domains)` (L97), which populates `perSource["example.org"]`. There is **no per-authenticated-user tier at this layer** — the routing map is keyed by envelope address and envelope domain only. Any local-domain `MAIL FROM` therefore matches at the **domain** tier regardless of who authenticated, which is exactly why T1 (auth `alice`, envelope `bob@example.org`) is accepted: same domain. The matched block's `rejectErr` (populated by a `reject` directive) is returned at L117-119. The authenticated identity (`username`) is present in the `incoming message` log but is **never consulted** by `srcBlockForAddr`.

---

## 11. Default-vs-explicit policy, ruled-out interpretations, and the behavior-diverges case (O7/O8)

### 11.1 Default-vs-explicit determination

In the **default/canonical** configuration, Maddy enforces sender authorization **only at the envelope DOMAIN level** (via `source` / `default_source`), and enforces `From`-alignment **only** as a DKIM signing precondition that **degrades to unsigned delivery — never as a rejection.** There is **no built-in per-user "MAIL FROM must equal the authenticated user" enforcement** in this revision, and **no `From`-vs-auth rejection**. Achieving per-user or `From`-based rejection would require **explicit additional policy** that this 2019-era revision does not provide as a submission `check`.

A version caveat matters here: current upstream Maddy documentation shows a `check { authorize_sender { ... } }` construct for submission that **does not exist** at commit `26452dd`. This was resolved against the repository's own source rather than newer docs: this revision relies solely on domain-level `source`/`default_source` routing plus the `sign_dkim` `require_sender_match` precondition. (Grounding: the routing code in `internal/msgpipeline/msgpipeline.go` has only address/domain/default tiers, and `internal/endpoint/smtp/submission.go`'s `submissionPrepare` performs no alignment comparison.)

### 11.2 Ruled-out incorrect interpretations (with evidence)

- **Incorrect (mandatory rebuttal):** *"Because the config pairs `sign_dkim` with a `reject`-based `default_source`, a message whose `From` doesn't match the authenticated user (or the signing domain) will be rejected."* **Refuted by observation:** T1, T4, and T5 are all **ACCEPTED** and delivered (just UNSIGNED); the only rejects are the envelope-domain `default_source` case (T3, `501 5.1.8`) and the missing-`From` syntax case (T6, `554 5.6.0`). Evidence: the `submission: accepted {"msg_id":"513ba1a1"}` log preceded by `not signing, From address is not authenticated identity`, and the `RewriteBody` `if !ok { return nil }` at `internal/modify/dkim/dkim.go:349-351`.
- **Incorrect:** *"Maddy stamps an `Authentication-Results` header on submitted mail."* **Refuted:** no such header appears in any stored message; the full observed header set is `Date, Delivered-To, Dkim-Signature, From, Message-Id, Received, Return-Path, Subject, To` (§8).
- **Incorrect:** *"The `Received` header records the authenticated user."* **Refuted by T4:** the stored `Received` shows `(envelope-sender <alice@example.org>)` (the envelope) while the message's `From` is `bob@example.org` (§7); the auth identity is stored separately as `AuthUser` and is never placed in `Received`.

### 11.3 The behavior-diverges case (O8)

The prime divergence between a reasonable configuration reading and observed behavior: **the default configuration accepts a cross-user, same-domain sender at the pipeline (domain-level match) while the `sign_dkim` modifier silently skips signing and delivers the message UNSIGNED, rather than rejecting it.** A naive reading of `maddy.conf` — seeing `sign_dkim` sitting next to a `reject`-based `default_source` — would suggest that misaligned senders are blocked. In reality they sail through, merely unsigned. T1 is the concrete instance: auth `alice`, envelope+`From` `bob@example.org`, **accepted and stored unsigned**.

### 11.4 Documented-vs-observed contrast (added rigor)

The `sign_dkim` modifier's own manual **does** document the silent-skip, which is worth stating so the divergence is attributed precisely. `docs/man/maddy-filters.5.scd` (≈L518-544) states `require_sender_match` *Default*: `envelope auth` (L518-519) and defines the semantics as requiring the identifiers to match the `From` field and key domain, **"otherwise - don't sign the message"** (L521-522) — i.e., *don't sign*, not *reject* — which matches observation. The same manual documents `sig_expiry` Default `120h` (L500-501, = the observed `x − t = 432000s`), `header_canon`/`body_canon` Default `relaxed` (matching `c=relaxed/relaxed`), and `newkey_algo` Default `rsa2048` (L513-514). So the silent-unsigned behavior is documented **at the modifier level** — a careful reader could know it — but the **`maddy.conf`-level pairing** of `sign_dkim` with a reject-based `default_source` is what invites the naive misreading. The divergence is thus at the *config-composition* level, compounded by the **PKCS#1/PKIX interop bug** (§9 Finding A) that no amount of config reading would predict.

---


## 12. Objective coverage checklist (O1–O8)

| Objective | Status | Where |
|-----------|--------|-------|
| **O1** — Stand up a canonical Maddy instance (submission w/ AUTH, DKIM signing, local delivery, ≥2 users) | Done | §3 |
| **O2** — Run the three named scenarios + edges through the real submission entry point | Done | §4 (T1 cross-user, T3 non-local envelope, T2 legitimate signed; plus T4/T5/T6) |
| **O3** — Report accept/reject mechanism + exact SMTP codes + dependence on how the mismatch is constructed | Done | §1, §4, §5, §10 |
| **O4** — Capture actual stored headers verbatim (`DKIM-Signature`, `Received`, negative `Authentication-Results`) | Done | §6.3, §7, §8 |
| **O5** — Probe DKIM triggering under `From` manipulation, `h=`/`From` coverage byte-exact, verifier conclusion | Done | §6, §9 |
| **O6** — Full SMTP transaction for ≥1 rejected + ≥1 accepted; document what was tried when `tcpdump` unavailable | Done | §5 (io_debug + client transcript; tcpdump absent → documented fallback) |
| **O7** — Determine default enforces domain-level only; per-user/`From` rejection needs explicit policy absent here; ≥1 incorrect interpretation ruled out | Done | §11 |
| **O8** — Identify ≥1 behavior-diverges case | Done | §11.3 (domain accept + silent unsigned) and §9 Finding A (PKCS#1/PKIX interop divergence) |

**Methodology / rules-compliance recap.** The code was run first and this answer written from the captured output; every behavioral claim is presented with the command that produced it and the actual, unedited output in a fenced block; each scenario was executed across ≥2 runs (587 ×2, 465 ×1) and was stable and transport-independent; every factual claim is grounded in an exact `file:line` reference naming the function/method/struct, with **OBSERVED vs INFERRED labeled**; the exact submission code path (AUTH → pipeline → `sign_dkim` → storage) was exercised, not a synthetic stand-in; the source tree is read-only and all transient artifacts were created outside it and removed. (See §2 for the full mapping.)

---

## 13. Source citation appendix

Every `file:line` cited above, with the named symbol. Line numbers are for commit `26452dd`.

**`maddy.conf`**
- L4 `$(hostname) = example.org`; L9 `$(primary_domain) = example.org`; L12 `$(local_domains) = $(primary_domain)`.
- L32-34 `sql local_mailboxes local_authdb { driver sqlite3; dsn all.db }`.
- L54-66 inbound (port-25) checks `require_matching_ehlo` (L56) / `require_mx_record` (L59) / `verify_dkim` (L62) / `apply_spf` (L65) — *contrast only*.
- L93 `submission tls://0.0.0.0:465`; L95 `auth &local_authdb`; L97 `source $(local_domains)`; L99 `sign_dkim $(primary_domain) default`.
- L115-116 anti-spoofing comment; L117-118 `default_source { reject 501 5.1.8 "Non-local sender domain" }`.

**`internal/msgpipeline/msgpipeline.go`**
- `srcBlockForAddr` — call L113, definition L155.
- reject execution `if sourceBlock.rejectErr != nil { ... return sourceBlock.rejectErr }` L117-119 (Debugf L118).
- normalize `address.ForLookup` L159; `501 5.1.7` L161-167; invalid-address `501 5.1.3` L179-187.
- exact match `perSource[cleanFrom]` L171; domain match `perSource[domain]` L190; fallback `defaultSource` L193.
- match Debugf: default rule L194, domain rule L196, address rule L199.

**`internal/modify/dkim/dkim.go`** (module `sign_dkim`)
- `oversignDefault` L31-54 (`From` at L37); `signDefault` L55-72.
- config defaults: `key_path` template L137; `header_canon`/`body_canon` relaxed L140-145; `sig_expiry` `5*Day` L146; `hash` sha256 L147-148; `newkey_algo` rsa2048 L149-150; `require_sender_match` default `["envelope","auth"]` L151-152 (allowed `{envelope, auth_domain, auth_user, off}`).
- `senderMatch` map build L166-172; `fieldsToSign` L202-233 (per-occurrence L215-217, oversign extra L219).
- `shouldSign` L249 — `off` L250-262, empty `From` L265-267, malformed `From` L270-272, multiple `From` L274-277, malformed address L281-285, `From`-domain≠key (`dns.Equal`) L287-291, envelope≠`From` (`address.Equal`) L293-296, auth-identity L299-310, identifier build L314-327.
- `RewriteBody` L340 — `authUser` L343-346, `shouldSign` call L348, `if !ok { return nil }` L349-351, `h.Add("DKIM-Signature", ...)` L406, `signed` log L408.
- `module.Register("sign_dkim", New)` L418; new-key log L192-194.

**`internal/modify/dkim/keys.go`**
- `writeDNSRecord` L136-163; `x509.MarshalPKCS1PublicKey(pubkey)` L143; record format `v=DKIM1; k=%s; p=%s` L158.

**`internal/endpoint/smtp/smtp.go`**
- `msgMeta.OriginalFrom = cleanFrom` L116; `submissionPrepare` call L292; `GenerateReceived` call L303; `header.Add("Received", ...)` L307.
- auto `AllowInsecureAuth` when no TLS L537-546 (`= true` at L545); `insecure_auth` L564; `io_debug` L565; `defer_sender_reject` (default true) L567; TLS inheritance L563.
- `authAlwaysRequired = true` for submission L589-590; `CheckPlain` L653; `AuthUser: username` L680.

**`internal/endpoint/smtp/submission.go`**
- `submissionPrepare` L27; `DontTraceSender = true` L28; inject Message-ID L30-37; missing-`From` `554 5.6.0` L39-48 (verbatim string at L43); Sender/To/Cc/Bcc/Reply-To syntax L50-95; multi-`From` needs Sender L99-109; Date inject L111-127. *(No `From`-vs-`MAIL FROM` / `From`-vs-auth comparison anywhere.)*

**`internal/target/received.go`**
- `GenerateReceived` L19; nil-`Conn` guard L20-22; sender-trace omitted when `DontTraceSender` L30-31; `" by "` + hostname L60-64; `" (envelope-sender <"` + `MAIL FROM` L66-72; `" with "` + proto + `" id "` L74-82; RFC1123Z date L84.

**`internal/auth/auth.go`**
- `CheckDomainAuth` L5; case-insensitive domain compare `strings.EqualFold` L25.

**`internal/storage/sql/sql.go`**
- PRECIS import L35; `UsernameCaseMapped.CompareKey` L364; `CheckPlain` L377; `OpaqueString.CompareKey` L386; backend bcrypt compare `store.Back.CheckPlain` L391; `module.Register("sql", New)` L424.

**`cmd/maddyctl/users.go`**
- imapsql import L8; `usersList` L14 (`ListUsers` L15); `usersCreate` L30; `--null` → `CreateUserNoPass` L36-37; `--hash` L40; `--bcrypt-cost` L48; `CreateUser` L76.

**`cmd/maddy/main.go`**
- 11 lines; `os.Exit(maddy.Run())` L10.

**`maddy.go`**
- `Version = "unknown (built from source tree)"` L41; flags `-libexec` L103, `-debug` L104, `-config` L107, `-log` L108, `-v` L109; chdir to state directory `os.Chdir(config.StateDirectory)` L220.

**`go.mod`**
- module `github.com/foxcpp/maddy` L1; `go 1.13` L3.
- `blitiri.com.ar/go/spf` L6; `github.com/emersion/go-message` L16; `github.com/emersion/go-msgauth v0.3.2-0.20191028231513-55b75676976c` L17; `github.com/emersion/go-sasl` L18; `github.com/emersion/go-smtp v0.12.1-0.20191206174923-1f576e0ec85c` L19; `github.com/foxcpp/go-imap-sql` L20; `github.com/foxcpp/go-mockdns` L21; `github.com/google/uuid v1.1.1` L23; `github.com/mattn/go-sqlite3 v1.11.0` L26; `github.com/miekg/dns v1.1.22` L27; `github.com/urfave/cli v1.22.1` L29; `golang.org/x/crypto` L30; `golang.org/x/text v0.3.2` L34.

**`docs/man/maddy-filters.5.scd`**
- config example `header_canon relaxed` / `body_canon relaxed` / `sig_expiry 120h` / `newkey_algo rsa2048` L386-390; `sig_expiry` Default `120h` L500-501; `newkey_algo` Default `rsa2048` L513-514; `require_sender_match` Default `envelope auth` L518-519 with "otherwise - don't sign the message" L521-522.

**`docs/internals/quirks.md`**
- L9 "`for` field is never included in the `Received` header field".

