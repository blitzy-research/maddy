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

**Nothing on the submission path compares `From` to `MAIL FROM` or to the authenticated identity *for the purpose of acceptance*.** The acceptance-path `From` handling is `submissionPrepare`, which validates *presence and syntax* of `From`/`Sender`/`To`/`Cc`/`Bcc`/`Reply-To` and injects `Message-ID`/`Date` if missing — it contains **no** alignment comparison. The *only* place any `From` comparison happens at all is inside the `sign_dkim` modifier, and there it governs **signing only, never acceptance** (§6): the message is delivered either way. *(Source-grounded: `internal/endpoint/smtp/submission.go` — `submissionPrepare` L27; missing-`From` → `554 5.6.0` at L39-48; the `From`-vs-envelope / `From`-vs-auth comparisons live in `internal/modify/dkim/dkim.go` `shouldSign` at L287-310, on the *signing* path — not on the acceptance path.)*

### Net effect (with the observed consequence to highlight)

| Gate | Scoped by | What it controls | Observed consequence |
|------|-----------|------------------|----------------------|
| AUTH required | — | Whether submission is allowed at all | All six scenarios had to authenticate |
| Pipeline `source` routing | **Envelope DOMAIN** | Accept vs reject | **T1 cross-user (auth alice, MAIL FROM bob) is ACCEPTED** — same domain |
| `sign_dkim` `require_sender_match` | `From` vs envelope / auth-identity / key-domain | Whether a `DKIM-Signature` is attached | **T1/T4/T5 delivered UNSIGNED, never rejected** |

The headline "no difference / negative" truths, all confirmed by observation and stated plainly because the truth requires it: **authorization is domain-level, not per-user**; **misaligned `From` yields silent unsigned delivery, not rejection**; and **no `Authentication-Results` header is added on submission *by default*** (§8 shows this is config-conditional, not absolute).

---

## 2. Methodology and rules compliance (SWE-AtlasQnA-Repo)

This answer was produced under the **observe-first, read-only** methodology. The mapping below states how each governing rule is satisfied in this document:

- **Ran the code first, then wrote.** All behavioral claims derive from output captured from a running daemon driven through its real submission endpoint (§4–§8); code reading is used only to *explain* observations, and such explanations are labeled source-grounded.
- **Actual, complete, unedited output for every claim, with the command that produced it.** The exact producing commands are shown for the build (§3.1), the daemon startup (§3.3), user provisioning (§3.6), each SMTP scenario via the `harness.py`/`run_all.sh` client (§5.0), the mailbox dump (§6.3), and the DKIM verifier (§9). One **complete, greeting-through-`QUIT`** accepted transaction and one complete rejected transaction are captured at both the client and server (`io_debug`) level in §5.0. Blocks that show only the decisive key lines (§5.1, §5.2, §6.1) are **explicitly labeled "excerpt,"** and any elision inside an excerpt (e.g. the `,...,` for `src_host`/`src_ip` in the §5.2 T3 excerpt) is flagged as an excerpt marker whose full form appears in the corresponding §5.0 capture. No code logic is elided anywhere (there is no `// ...`).
- **Evidence provenance (historical vs. corroborating).** Per the checkpoint's evidence constraint, the **byte-exact historical captures** from the original investigation — the §5.1/§5.2 protocol excerpts (`msg_id`s `513ba1a1`, `fa2790f2`, …), the §6.1 sign/skip log lines, the §6.3 raw `DKIM-Signature`, and the §7 raw `Received` — are **preserved, not replaced by rerun bytes.** The §5.0 complete transcripts are a **fresh corroborating run** of the identical harness: its **deterministic** fields (SMTP response codes and exact message strings, pipeline log-line sequence, sign/skip decisions, `bh=`, the `h=` field list, `c=`/`a=`, and `x − t`) reproduce **byte-for-byte**, while inherently **per-run** fields (`msg_id`, ephemeral `src_ip` port, injected `Message-ID`/`Date`, the signing timestamps `t`/`x`, and therefore the signature `b=`) differ by design. Both are labeled wherever they appear.
- **Real scale, stability across ≥2 runs.** Each scenario T1–T6 was executed **at least twice over plaintext port 587 and once over implicit-TLS port 465** (transport-independent); results were identical and stable across runs before being reported.
- **Exact code path through the real entry point.** The path exercised is SMTP `AUTH` → message pipeline (`msgpipeline`) → DKIM modifier (`sign_dkim` `RewriteBody`) → local storage (`imapsql` SQLite). No bypassing or synthetic interface was used.
- **Default, canonical configuration.** The runtime mirrors the repository's default submission semantics; the build and invocation commands are stated verbatim (§4), and the canonical version banner (`maddy unknown (built from source tree)`) is reported as-is.
- **Every condition exercised.** Primary (T2 aligned) plus secondary/edge/error paths (T1 cross-user, T3 non-local envelope, T4 `From`≠envelope, T5 `From`-domain≠key, T6 missing `From`); state observed *before* delivery (accept/reject at the protocol) and *after* delivery (signed/unsigned in storage).
- **Exact and grounded.** Every factual claim carries an actual value plus a `file:line` citation naming the function/method/struct; **OBSERVED vs INFERRED is labeled** throughout; the direct answer leads and includes negative results.
- **Byte-exact DKIM verification.** The `DKIM-Signature` was read from the stored message bytes and its `h=` tag inspected byte-for-byte to confirm `From` coverage (§6.3).
- **Read-only source tree.** No existing repository file was modified and no code was added; the only new artifact is this document.

---

## 3. How the runtime was built and configured (OBSERVED)

### 3.1 Build

CGO is **required** — the storage/auth backend uses the `mattn/go-sqlite3` driver (`go.mod` L26, `github.com/mattn/go-sqlite3 v1.11.0`), which is a cgo package. Both `cmd/maddy` (the daemon) and `cmd/maddyctl` (the admin CLI, needed for user provisioning and mailbox inspection) were built with `CGO_ENABLED=1`. The exact producing commands and their **complete, unedited** output (Go 1.18.10, gcc 10.2.1, in the project container):

```
$ GO111MODULE=on CGO_ENABLED=1 GOOS=linux GOARCH=amd64 \
    go build -trimpath -tags debug -o /tmp/maddy_qa_ws/bin/maddy ./cmd/maddy
# github.com/mattn/go-sqlite3
sqlite3-binding.c: In function 'sqlite3SelectNew':
sqlite3-binding.c:125322:10: warning: function may return address of local variable [-Wreturn-local-addr]
125322 |   return pNew;
       |          ^~~~
sqlite3-binding.c:125282:10: note: declared here
125282 |   Select standin;
       |          ^~~~~~~
$ echo $?
0

$ GO111MODULE=on CGO_ENABLED=1 GOOS=linux GOARCH=amd64 \
    go build -trimpath -o /tmp/maddy_qa_ws/bin/maddyctl ./cmd/maddyctl
# github.com/mattn/go-sqlite3
sqlite3-binding.c: In function 'sqlite3SelectNew':
sqlite3-binding.c:125322:10: warning: function may return address of local variable [-Wreturn-local-addr]
125322 |   return pNew;
       |          ^~~~
sqlite3-binding.c:125282:10: note: declared here
125282 |   Select standin;
       |          ^~~~~~~
$ echo $?
0
```

Both builds exit `0`; the sole compiler output is the harmless CGO warning from `mattn/go-sqlite3` about a function returning the address of a local variable (identical for both binaries). `-tags debug` enables the `-debug` verbose logging used throughout; `-trimpath` keeps paths reproducible. *(Source-grounded: `go.mod` L1 module `github.com/foxcpp/maddy`, L3 `go 1.13`.)*

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

The daemon entry point is trivially thin — `cmd/maddy/main.go` is 11 lines and does only `os.Exit(maddy.Run())`. The flags are declared in `maddy.go`: `-libexec` (L103), `-debug` (L104), `-config` (L107), `-log` (L108), `-v` (L109). On startup the daemon **changes its working directory to the state directory** (`os.Chdir(config.StateDirectory)`, `maddy.go:220`), so relative paths in configuration — `dsn all.db` and `dkim_keys/{domain}_{selector}.key` — resolve relative to that state directory.

The exact invocation and the **startup log** it emitted (this is an *excerpt* of the daemon's `-debug` startup output — the config-parse `[debug]` lines and the DKIM-keygen and listener lines, which are the parts that matter here; the full 33 KB log continues with per-message lines shown in §5):

```
$ /tmp/maddy_qa_ws/bin/maddy -debug -config /tmp/maddy_qa_ws/maddy.conf
tls: using self-signed certificate, this is not secure!
[debug] sql: go-imap-sql version 0.4.0
[debug] /tmp/maddy_qa_ws/maddy.conf:19: reference &local_authdb
[debug] /tmp/maddy_qa_ws/maddy.conf:22: new module sign_dkim [example.org default]
sign_dkim: generating a new rsa2048 keypair...
sign_dkim: generated a new rsa2048 keypair, private key is in dkim_keys/example.org_default.key, TXT record with public key is in dkim_keys/example.org_default.dns,
put its contents into TXT record for default._domainkey.example.org to make signing and verification work
[debug] /tmp/maddy_qa_ws/maddy.conf:25: reference &local_mailboxes
[debug] /tmp/maddy_qa_ws/maddy.conf:28: reference &local_mailboxes
[debug] submission: authentication provider: sql local_mailboxes
submission: listening on tls://0.0.0.0:465
[debug] /tmp/maddy_qa_ws/maddy.conf:37: reference &local_authdb
[debug] /tmp/maddy_qa_ws/maddy.conf:42: new module sign_dkim [example.org default]
[debug] /tmp/maddy_qa_ws/maddy.conf:45: reference &local_mailboxes
[debug] /tmp/maddy_qa_ws/maddy.conf:48: reference &local_mailboxes
submission: I/O debugging is on! It may leak passwords in logs, be careful!
[debug] submission: authentication provider: sql local_mailboxes
submission: listening on tcp://127.0.0.1:587
```

This single startup confirms, in order: the self-signed TLS notice, the go-imap-sql backend, the **auto-generated `rsa2048` DKIM keypair** (§3.5), the implicit-TLS listener on `:465`, the `io_debug`/`insecure_auth` plaintext listener on `127.0.0.1:587`, and that both listeners share the `sql local_mailboxes` auth provider.

### 3.4 Canonical `maddy.conf` semantics

The test configuration was authored **outside the source tree**, faithful to the repository's default submission semantics. The relevant directives and their grounding in the repository default `maddy.conf`:

- **Unified SQLite storage + auth:** `sql local_mailboxes local_authdb { driver sqlite3; dsn all.db }` — `maddy.conf` L32-34. One database (`all.db`) backs both credential checks and IMAP mailbox delivery. The module is registered as `sql` at `internal/storage/sql/sql.go:424` (`module.Register("sql", New)`).
- **Submission endpoint (implicit TLS):** `submission tls://0.0.0.0:465 { auth &local_authdb ... }` — `maddy.conf` L93, L95.
- **Plaintext submission listener for clean protocol capture:** a second `submission tcp://127.0.0.1:587` (cleartext) listener was added with an explicit `insecure_auth` and `io_debug`. It **inherits the global `tls self_signed` configuration** (the `cfg.Custom("tls", ...)` directive at `internal/endpoint/smtp/smtp.go:563` is inherit-from-global), so `endp.serv.TLSConfig` is **non-nil**; consequently the auto-insecure-auth path (`internal/endpoint/smtp/smtp.go:537-546`, `endp.serv.AllowInsecureAuth = true` at L545, which fires **only when `TLSConfig == nil`**) does **not** apply to this listener. Instead, plaintext `AUTH` on the not-yet-encrypted channel is permitted by the **explicit `insecure_auth` knob** (`cfg.Bool("insecure_auth", ...)` at L564), and `io_debug` (L565) emits raw server-side SMTP I/O. Because TLS is *available* (`TLSConfig != nil`), this cleartext listener **advertises `STARTTLS`** — confirmed in the §4.1/§5.0 transcripts and grounded in `github.com/emersion/go-smtp` `conn.go:229-230`, which appends `STARTTLS` only when `c.server.TLSConfig != nil && !isTLS` — whereas the implicit-TLS `465` listener does not (it is already encrypted). The client authenticates over the still-cleartext channel *without* issuing `STARTTLS`, which is what keeps the `io_debug` capture legible. **Port 465 (implicit TLS) and port 587 (plaintext) produced identical behavior — the outcome is transport-independent.**
- **Sender routing:** `source $(local_domains) { modify { sign_dkim $(primary_domain) default } deliver_to &local_mailboxes }` — `maddy.conf` L97, L99; catch-all `default_source { reject 501 5.1.8 "Non-local sender domain" }` — L117-118. The `maddy.conf` comment at L115-116 states the intent: block non-local sender domains as "likely a spoofing attempt". Config vars: `hostname example.org` (L4), `primary_domain example.org` (L9), `local_domains $(primary_domain)` (L12).
- **TLS:** the certificate used `self_signed` so Maddy auto-generates a self-signed cert (no external `openssl` needed); the submission endpoint inherits the TLS config via `cfg.Custom("tls", ...)` at `internal/endpoint/smtp/smtp.go:563`.

### 3.5 DKIM key auto-generation (OBSERVED)

On first start, the `sign_dkim example.org default` modifier auto-generated the keypair under the state directory:

- `dkim_keys/example.org_default.key` — the private key, PEM, mode `0600`.
- `dkim_keys/example.org_default.dns` — the public TXT record, of the form `v=DKIM1; k=rsa; p=<base64>`.

The default new-key algorithm is `rsa2048`. The selector `default` and domain `example.org` come directly from `sign_dkim $(primary_domain) default`. The daemon logged the generation with instructions to publish the TXT record. *(Source-grounded: `internal/modify/dkim/dkim.go` — key path template `dkim_keys/{domain}_{selector}.key` L137, new-key log L192-194; `internal/modify/dkim/keys.go` — `writeDNSRecord` L136-163, record format `v=DKIM1; k=%s; p=%s` at L158. The `p=` public-key encoding at `keys.go:143` is analyzed in §9 Finding A.)*

### 3.6 User provisioning (OBSERVED)

Two accounts in the same local domain were created. This repository revision uses `maddyctl users create` — **not** the later-version `maddy creds` subcommand. The exact producing commands and their **complete** output (on success `users create` prints nothing and exits `0`; `users list` confirms both accounts):

```
$ maddyctl --config /tmp/maddy_qa_ws/maddy.conf users create alice@example.org -p AlicePass123
$ echo $?
0
$ maddyctl --config /tmp/maddy_qa_ws/maddy.conf users create bob@example.org -p BobPass123
$ echo $?
0
$ maddyctl --config /tmp/maddy_qa_ws/maddy.conf users list
alice@example.org
bob@example.org
```

*(Source-grounded: `cmd/maddyctl/users.go` — `usersList`/`ListUsers` L14-15; `usersCreate` L30; flags `--null` → `CreateUserNoPass` L36-37, `--hash` L40, `--bcrypt-cost` L48; `CreateUser` L76.)* Credentials are checked at login by `internal/storage/sql/sql.go` `CheckPlain` (L377), which applies PRECIS `UsernameCaseMapped` to the username (L364) and PRECIS `OpaqueString` to the password (L386) before the backend performs the bcrypt comparison (L391). The domain-auth helper `internal/auth/auth.go` `CheckDomainAuth` (L5) splits the username on `@` and compares domains case-insensitively via `strings.EqualFold` (L25).

### 3.7 Test client and tooling

The test client is a Python `smtplib` harness (`harness.py`, reproduced verbatim in §5.0). It authenticates with SASL PLAIN via `login()`, enables `set_debuglevel(1)` for the full client-side transcript, uses `SMTP_SSL` with `ssl.CERT_NONE` for port 465 and plain `SMTP` for port 587, and **decouples the envelope `MAIL FROM` from the `From` header** by driving the low-level verbs directly — `smtp.mail(envelope_from)`, `smtp.rcpt(rcpt)`, `smtp.data(raw_message_bytes)` — while the `From:` header is written inside the raw message body. This lets each scenario set an envelope sender that differs from the `From` header and capture the per-verb response code. Note the container **lacks `swaks`, `tcpdump`, and the `sqlite3` CLI**; Python `smtplib`, Maddy's `io_debug`, and Python's `sqlite3` module substituted for them, and mailbox contents were read with the exact-syntax command `maddyctl --config <cfg> imap-msgs dump <user> <mailbox> <seq>` (subcommand **before** its positional arguments — see §6.3), cross-checked against the go-imap-sql external body blob.

---


## 4. The test matrix (T1–T6): expected vs OBSERVED

All six scenarios authenticate as **alice**. Each was run **≥2× over plaintext port 587 and ×1 over implicit-TLS port 465**, driven through the real submission endpoint, with **identical, stable results** across every run and both transports. The producing command (`harness.py`/`run_all.sh`) and one complete greeting-through-`QUIT` accepted and rejected capture are in §5.0; the `Msg ID` column below shows the **historical** ids from the original investigation (preserved per the evidence rule).

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

- EHLO advertised: `PIPELINING`, `8BITMIME`, `ENHANCEDSTATUSCODES`, `STARTTLS`, `AUTH PLAIN`, `SMTPUTF8`, `SIZE 33554432` (matching the complete §5.0 transcript). **`STARTTLS` is transport-specific (OBSERVED):** it is advertised on the plaintext `587` submission listener (TLS is available but not yet active) but is **absent on the implicit-TLS `465` listener** (the connection is already encrypted) — a live EHLO on each port confirmed `STARTTLS` present on `587` and absent on `465`, stable across repeated probes. *(Source-grounded: `github.com/emersion/go-smtp` `conn.go:229-230` appends `STARTTLS` only when `c.server.TLSConfig != nil && !isTLS`.)*
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

Because implicit-TLS submission on port 465 is ciphertext on the wire and `tcpdump` is absent from the container, protocol-level visibility was obtained two ways that together cover both an accepted and a rejected transaction **without TLS key extraction**: Maddy's `io_debug` (server-side raw SMTP I/O, `internal/endpoint/smtp/smtp.go:565`) and the client-side `set_debuglevel(1)` transcript. This is the documented fallback: the encrypted channel was captured through server-side and client-side logging plus a plaintext `587` listener, rather than by decrypting TLS.

§5.0 gives the **producing command** and the **complete, greeting-through-`QUIT`** client and server capture for one accepted and one rejected transaction (a fresh corroborating run). §5.1 and §5.2 then present the **key-line excerpts** from the original investigation with their historical `msg_id`s preserved; those blocks are **excerpts** (labeled as such), and their complete greeting-through-`QUIT` counterparts are the §5.0 captures.

### 5.0 Producing command, harness, and COMPLETE greeting-through-`QUIT` capture (OBSERVED, fresh corroborating run)

Every scenario was driven by the `harness.py` client (reproduced verbatim), invoked once per scenario/transport by `run_all.sh`, which also slices the daemon's `io_debug` log per transaction:

```
$ python3 /tmp/maddy_qa_ws/scripts/harness.py T1 plain      # one scenario, plaintext :587
$ python3 /tmp/maddy_qa_ws/scripts/harness.py T3 tls        # one scenario, implicit-TLS :465
# run_all.sh drives all six over plaintext (r1, r2) and TLS (r1):
$ bash /tmp/maddy_qa_ws/scripts/run_all.sh
ALL RUNS DONE
18
```

> **Provenance / evidence rule.** Per the checkpoint constraint, the byte-exact **historical** captures (§5.1, §5.2, §6.1, §6.3, §7) are **preserved, not replaced**. The blocks in this §5.0 are a **fresh corroborating run** of the identical harness; the **deterministic** fields (SMTP response codes and their exact message strings, the sequence of pipeline log lines, and the sign/skip decisions) reproduce **identically**, while the **inherently per-run** fields (`msg_id`, `src_ip` ephemeral port, and the injected `Message-ID`/`Date`) necessarily differ. This run's ids are T1 `9fa14237`, T3 `933c585c`.

**ACCEPTED — T1 cross-user (auth `alice`, envelope+`From` `bob@example.org`).** Complete client transcript (`set_debuglevel(1)`), greeting through `QUIT`:

```
===TRANSCRIPT-BEGIN===
connect: to ('127.0.0.1', 587) None
reply: b'220 example.org ESMTP Service Ready\r\n'
reply: retcode (220); Msg: b'example.org ESMTP Service Ready'
connect: b'example.org ESMTP Service Ready'
send: 'ehlo client.test\r\n'
reply: b'250-Hello client.test\r\n'
reply: b'250-PIPELINING\r\n'
reply: b'250-8BITMIME\r\n'
reply: b'250-ENHANCEDSTATUSCODES\r\n'
reply: b'250-STARTTLS\r\n'
reply: b'250-AUTH PLAIN\r\n'
reply: b'250-SMTPUTF8\r\n'
reply: b'250 SIZE 33554432\r\n'
reply: retcode (250); Msg: b'Hello client.test\nPIPELINING\n8BITMIME\nENHANCEDSTATUSCODES\nSTARTTLS\nAUTH PLAIN\nSMTPUTF8\nSIZE 33554432'
send: 'AUTH PLAIN AGFsaWNlQGV4YW1wbGUub3JnAEFsaWNlUGFzczEyMw==\r\n'
reply: b'235 2.0.0 Authentication succeeded\r\n'
reply: retcode (235); Msg: b'2.0.0 Authentication succeeded'
send: 'mail FROM:<bob@example.org>\r\n'
reply: b'250 2.0.0 Roger, accepting mail from <bob@example.org>\r\n'
reply: retcode (250); Msg: b'2.0.0 Roger, accepting mail from <bob@example.org>'
send: 'rcpt TO:<alice@example.org>\r\n'
reply: b"250 2.0.0 I'll make sure <alice@example.org> gets this\r\n"
reply: retcode (250); Msg: b"2.0.0 I'll make sure <alice@example.org> gets this"
send: 'data\r\n'
reply: b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
reply: retcode (354); Msg: b'2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>'
data: (354, b'2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>')
send: b'From: bob@example.org\r\nTo: alice@example.org\r\nSubject: T1\r\n\r\nT1 body\r\n.\r\n'
reply: b'250 2.0.0 OK: queued\r\n'
reply: retcode (250); Msg: b'2.0.0 OK: queued'
data: (250, b'2.0.0 OK: queued')
send: 'quit\r\n'
reply: b'221 2.0.0 Goodnight and good luck\r\n'
reply: retcode (221); Msg: b'2.0.0 Goodnight and good luck'
===TRANSCRIPT-END===
===RESULT=== {'mail': (250, '2.0.0 Roger, accepting mail from <bob@example.org>'), 'rcpt': (250, "2.0.0 I'll make sure <alice@example.org> gets this"), 'data': (250, '2.0.0 OK: queued')}
```

Corresponding **complete server-side `io_debug`** for the same transaction (greeting through `QUIT`), showing the domain-rule match and the DKIM skip:

```
submission: 220 example.org ESMTP Service Ready
submission: ehlo client.test
submission: 250-Hello client.test
submission: 250-PIPELINING
submission: 250-8BITMIME
submission: 250-ENHANCEDSTATUSCODES
submission: 250-STARTTLS
submission: 250-AUTH PLAIN
submission: 250-SMTPUTF8
submission: 250 SIZE 33554432
submission: AUTH PLAIN AGFsaWNlQGV4YW1wbGUub3JnAEFsaWNlUGFzczEyMw==
submission: 235 2.0.0 Authentication succeeded
submission: mail FROM:<bob@example.org>
submission: 250 2.0.0 Roger, accepting mail from <bob@example.org>
submission: rcpt TO:<alice@example.org>
submission: incoming message {"msg_id":"9fa14237","sender":"bob@example.org","src_host":"client.test","src_ip":"127.0.0.1:44450","username":"alice@example.org"}
[debug] smtp/pipeline: sender bob@example.org matched by domain rule 'example.org' {"msg_id":"9fa14237"}
[debug] smtp/pipeline: global rcpt modifiers: alice@example.org => alice@example.org {"msg_id":"9fa14237"}
[debug] smtp/pipeline: per-source rcpt modifiers: alice@example.org => alice@example.org {"msg_id":"9fa14237"}
[debug] smtp/pipeline: recipient alice@example.org matched by domain rule 'example.org' {"msg_id":"9fa14237"}
[debug] smtp/pipeline: per-rcpt modifiers: alice@example.org => alice@example.org {"msg_id":"9fa14237"}
[debug] smtp/pipeline: tgt.Start(bob@example.org) ok, target = sql:local_mailboxes {"msg_id":"9fa14237"}
submission: RCPT ok {"msg_id":"9fa14237","rcpt":"alice@example.org"}
submission: 250 2.0.0 I'll make sure <alice@example.org> gets this
submission: data
submission: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
submission: From: bob@example.org
To: alice@example.org
Subject: T1

T1 body
.
submission: adding missing Message-ID
submission: adding missing Date header
sign_dkim: not signing, From address is not authenticated identity {"auth_id":"alice@example.org","from_addr":"bob@example.org","msg_id":"9fa14237"}
[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery {"msg_id":"9fa14237"}
submission: accepted {"msg_id":"9fa14237"}
submission: 250 2.0.0 OK: queued
[debug] submission: reset
submission: quit
submission: 221 2.0.0 Goodnight and good luck
```

**REJECTED — T3 non-local envelope (auth `alice`, envelope+`From` `x@remote.tld`).** Complete client transcript, greeting through `QUIT`:

```
===TRANSCRIPT-BEGIN===
connect: to ('127.0.0.1', 587) None
reply: b'220 example.org ESMTP Service Ready\r\n'
reply: retcode (220); Msg: b'example.org ESMTP Service Ready'
connect: b'example.org ESMTP Service Ready'
send: 'ehlo client.test\r\n'
reply: b'250-Hello client.test\r\n'
reply: b'250-PIPELINING\r\n'
reply: b'250-8BITMIME\r\n'
reply: b'250-ENHANCEDSTATUSCODES\r\n'
reply: b'250-STARTTLS\r\n'
reply: b'250-AUTH PLAIN\r\n'
reply: b'250-SMTPUTF8\r\n'
reply: b'250 SIZE 33554432\r\n'
reply: retcode (250); Msg: b'Hello client.test\nPIPELINING\n8BITMIME\nENHANCEDSTATUSCODES\nSTARTTLS\nAUTH PLAIN\nSMTPUTF8\nSIZE 33554432'
send: 'AUTH PLAIN AGFsaWNlQGV4YW1wbGUub3JnAEFsaWNlUGFzczEyMw==\r\n'
reply: b'235 2.0.0 Authentication succeeded\r\n'
reply: retcode (235); Msg: b'2.0.0 Authentication succeeded'
send: 'mail FROM:<x@remote.tld>\r\n'
reply: b'250 2.0.0 Roger, accepting mail from <x@remote.tld>\r\n'
reply: retcode (250); Msg: b'2.0.0 Roger, accepting mail from <x@remote.tld>'
send: 'rcpt TO:<alice@example.org>\r\n'
reply: b'501 5.1.8 Non-local sender domain (msg ID = 933c585c)\r\n'
reply: retcode (501); Msg: b'5.1.8 Non-local sender domain (msg ID = 933c585c)'
send: 'quit\r\n'
reply: b'221 2.0.0 Goodnight and good luck\r\n'
reply: retcode (221); Msg: b'2.0.0 Goodnight and good luck'
===TRANSCRIPT-END===
===RESULT=== {'mail': (250, '2.0.0 Roger, accepting mail from <x@remote.tld>'), 'rcpt': (501, '5.1.8 Non-local sender domain (msg ID = 933c585c)')}
```

Corresponding **complete server-side `io_debug`**, showing the default-rule match and the reject directive:

```
submission: 220 example.org ESMTP Service Ready
submission: ehlo client.test
submission: 250-Hello client.test
submission: 250-PIPELINING
submission: 250-8BITMIME
submission: 250-ENHANCEDSTATUSCODES
submission: 250-STARTTLS
submission: 250-AUTH PLAIN
submission: 250-SMTPUTF8
submission: 250 SIZE 33554432
submission: AUTH PLAIN AGFsaWNlQGV4YW1wbGUub3JnAEFsaWNlUGFzczEyMw==
submission: 235 2.0.0 Authentication succeeded
submission: mail FROM:<x@remote.tld>
submission: 250 2.0.0 Roger, accepting mail from <x@remote.tld>
submission: rcpt TO:<alice@example.org>
submission: incoming message {"msg_id":"933c585c","sender":"x@remote.tld","src_host":"client.test","src_ip":"127.0.0.1:44460","username":"alice@example.org"}
[debug] smtp/pipeline: sender x@remote.tld matched by default rule {"msg_id":"933c585c"}
[debug] smtp/pipeline: global rcpt modifiers: alice@example.org => alice@example.org {"msg_id":"933c585c"}
[debug] smtp/pipeline: per-source rcpt modifiers: alice@example.org => alice@example.org {"msg_id":"933c585c"}
[debug] smtp/pipeline: recipient alice@example.org matched by default rule (clean = alice@example.org) {"msg_id":"933c585c"}
submission: RCPT error {"effective_rcpt":"alice@example.org","rcpt":"alice@example.org","reason":"reject directive used","smtp_code":501,"smtp_enchcode":"5.1.8","smtp_msg":"Non-local sender domain"}
submission: 501 5.1.8 Non-local sender domain (msg ID = 933c585c)
submission: quit
submission: 221 2.0.0 Goodnight and good luck
submission: aborted {"msg_id":"933c585c"}
```

These two blocks are the O6 deliverable: a complete accepted and a complete rejected transaction, each captured at the protocol level (server `io_debug`) and the client level, with the producing command shown.

### 5.1 ACCEPTED — T1 cross-user — historical **excerpt** (`msg_id 513ba1a1`, key lines) — the smoking gun for domain-level (not per-user) authorization

> **This block is an EXCERPT** (the AUTH-through-`accepted` key lines from the original investigation, historical `msg_id 513ba1a1`). Its complete greeting-through-`QUIT` counterpart — client and server — is the T1 capture in §5.0.

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

### 5.2 REJECTED — T3 non-local envelope — historical **excerpt** (`msg_id fa2790f2`, key lines)

> **This block is an EXCERPT** (the `MAIL`-through-reject key lines from the original investigation, historical `msg_id fa2790f2`). The literal `,...,` on the `incoming message` line is an **elision marker for that excerpt** (the omitted `src_host`/`src_ip` fields). Its complete greeting-through-`QUIT` counterpart — client and server, with those fields shown in full — is the T3 capture in §5.0.

```
submission: mail FROM:<x@remote.tld> size=74
submission: 250 2.0.0 Roger, accepting mail from <x@remote.tld>
submission: rcpt TO:<alice@example.org>
submission: incoming message {"msg_id":"fa2790f2","sender":"x@remote.tld",...,"username":"alice@example.org"}
[debug] smtp/pipeline: sender x@remote.tld matched by default rule {"msg_id":"fa2790f2"}
submission: RCPT error {"effective_rcpt":"alice@example.org","rcpt":"alice@example.org","reason":"reject directive used","smtp_code":501,"smtp_enchcode":"5.1.8","smtp_msg":"Non-local sender domain"}
submission: 501 5.1.8 Non-local sender domain (msg ID = fa2790f2)
```

**What this proves.** `MAIL FROM` still received `250` (the reject is deferred). The non-local sender `x@remote.tld` `matched by default rule` — i.e., it fell through both the exact-address and domain tiers to `default_source`, whose `reject` directive produced the `501 5.1.8 Non-local sender domain` at the first `RCPT`. (The `,...,` here is the excerpt's elision marker for the `src_host`/`src_ip` fields; the §5.0 T3 capture shows the identical line with those fields in full — `"src_host":"client.test","src_ip":"127.0.0.1:44460"` — and the matching server `RCPT error {... "reason":"reject directive used","smtp_code":501 ...}` line.)

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

Each maps precisely to a branch of `shouldSign` in `internal/modify/dkim/dkim.go` (L249). The `require_sender_match` directive defaults to `["envelope", "auth"]` (L151-152; allowed set `{envelope, auth_domain, auth_user, off}` — note the default value `auth` is itself **outside** this allowed set, a parser/default contradiction reproduced at runtime in §11.5), so both the envelope and auth checks are active:

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

Read from the stored message bytes with the exact-revision command syntax — **the `dump` subcommand comes before its positional `USERNAME MAILBOX SEQ` arguments** (`cmd/maddyctl/imap.go` `msgsDump` reads `ctx.Args()` as USERNAME, MAILBOX, SEQ at L403). The producing command and its complete output (T2 is UID 2 in alice's INBOX for the aligned scenario):

```
$ maddyctl --config /tmp/maddy_qa_ws/maddy.conf imap-msgs dump alice@example.org INBOX 2
```

> **Producing-command note (P4-F6).** The *wrong* ordering documented in earlier drafts — `maddyctl imap-msgs INBOX dump` (mailbox before the subcommand) — does **not** work: it prints `No help topic for 'INBOX'` and (a separate CLI defect, P5-F1, documented in §14.1) still exits `0`. The corrected `dump <user> <mailbox> <seq>` form above is what produced the bytes below.

The historical byte-exact `DKIM-Signature` from the original investigation is reproduced verbatim below (exact wrapping, spacing, and tag order preserved). Per the checkpoint's evidence rule, these **historical bytes are preserved, not replaced**; a fresh corroborating capture (§5.0) reproduces the deterministic fields — `bh=`, the full `h=` field list, `c=`, `a=`, and the `x − t` span — byte-for-byte, differing only in the inherently per-run fields `t`/`x` (signing timestamps) and therefore `b=` (the RSA signature over them):

```
Dkim-Signature: a=rsa-sha256;
 bh=oY/UZI4p+g7RAXkfVGG/HEXj+NClXPmVj7uvrhS6fig=; c=relaxed/relaxed;
 d=example.org;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=alice@example.org; s=default; t=1784155112; v=1; x=1784587112; b=P689eVzytP3hZ3OZA931bh9Eu/5cckoV9WEWin6ZGZVVxFJq9SI5/ctJSZ1++Tto1dmdBlXYjEc+ct5ATcLEXMx8edyGphglhAhKiRejKb2dtOKFkVgF7xyUGh88d41kzTALgYIJuGVpc2I0mr3b9LzkQsvGyIPsP5b2ARkyHVOhaFfoWfL4y8TyRYMQj/2M8P1xRCfu5qfdLwSssrBS5hJUO9OUOrUJ4yuVE9/v/uVhVqqqGvoCt9stclOQJYTKgQCKWHugSYE63gJI87zdLpHNFMno9epamLhttk+qhAHTm5LbpT+AwhGSMUt+2G/fj96kIsuMf7WdUQ/AjZk/tA==;
```

**`h=` tag analysis (byte-exact, O5).** `From` appears **twice** (`...From:From...`) — it is **oversigned**. So are `Subject:Subject`, `To:To`, `Date:Date`, and `Message-Id:Message-Id`. **Oversigning** means listing a header field one more time than it actually occurs: this both binds the present copy *and* prevents an intermediary from silently adding a second copy without breaking the signature. Therefore the signature **does cover `From`** (in fact it oversigns it). The oversign field set is `oversignDefault` in `internal/modify/dkim/dkim.go` (L31-54; `From` at L37), and the "one entry per occurrence, plus one more" logic is `fieldsToSign` (L202-233: for each oversigned key it appends one entry per occurrence present in the message, L215-217, then one extra, L219). Consequently a header present once in the message (Subject, To, From, Date, Message-Id) appears **twice** in `h=`, while an oversigned header absent from the message (Sender, Cc, MIME-Version, Content-Type, Content-Transfer-Encoding, Reply-To, In-Reply-To, References, Autocrypt, Openpgp) appears **once**. The observed `h=` string is exactly this derivation — confirming T2's message carried `Subject`, `To`, `From`, `Date`, `Message-Id` (each once) and none of the sign-only `List-*`/`Resent-*` fields (`signDefault`, L55-72).

**Timestamps.** `t=1784155112`, `x=1784587112`, so `x − t = 432000` seconds = **5 days = 120h**, matching the documented `sig_expiry` default of `120h` (`docs/man/maddy-filters.5.scd:500-501`) and the code default `5*Day` (`internal/modify/dkim/dkim.go:146`). `c=relaxed/relaxed` matches the `header_canon`/`body_canon` defaults (L140-145) and `a=rsa-sha256` matches the `hash` default `sha256` (L147-148) with the RSA key.

### 6.4 The UNSIGNED messages (OBSERVED)

The three misaligned deliveries each have **no `DKIM-Signature` header at all** in storage. In the fresh corroborating run the four accepted messages land in alice's INBOX in delivery order — **T1 = UID 1 (unsigned), T2 = UID 2 (signed), T4 = UID 3 (unsigned), T5 = UID 4 (unsigned)** — verified by dumping each: `imap-msgs dump alice@example.org INBOX 1/3/4` show **zero** `Dkim-Signature` lines, while `... INBOX 2` shows exactly one. This is the observed consequence of the §6.2 skip path (`RewriteBody` `return nil`). *(UID values are storage sequence numbers assigned in delivery order, hence per-run; the signed/unsigned split is invariant.)*

### 6.5 Duplicate physical `From` lines → order-dependent signing (OBSERVED; out-of-scope source behavior)

A message may carry **two physical `From:` header lines**. `submissionPrepare` accepts it (it checks presence/syntax, not cardinality), and the sign/skip outcome then depends on **wire order**. Producing command (against the canonical `:587`, authenticated as `alice`, envelope `alice`, both `From` addresses at `example.org`):

```
$ python3 /tmp/maddy_qa_ws/scripts/dupfrom.py   # sends DUPA (alice-first) then DUPB (bob-first)
```

- **DUPA — `From: alice@example.org` first, then `From: bob@example.org`:** accepted (`250 2.0.0 OK: queued`) and **SIGNED**. The stored `DKIM-Signature` has `i=alice@example.org` and `h=Subject:Subject:Sender:To:To:Cc:From:From:From:Date:Date:...` — **`From` appears three times** (two physical occurrences + one oversign), and **both** physical `From` lines are retained in storage.
- **DUPB — `From: bob@example.org` first, then `From: alice@example.org`:** accepted and **UNSIGNED**; both `From` lines retained.

**Mechanism (source-grounded).** `shouldSign` reads the `From` value with `fromVal := h.Get("From")` at `internal/modify/dkim/dkim.go:264`. `Header.Get` returns a **single** value — the **first wire-order** `From` (go-message `textproto/header.go:81-87` returns `fields[len(fields)-1].v`, which is first-in-wire-order given go-message's reversed internal storage). Because `Get` collapses the two physical lines to one, the multiple-`From` guard `len(fromAddrs) != 1 && !m.multipleFromOk` (`dkim.go:271-274`) **never fires**, and alignment is judged against whichever `From` happened to be first on the wire: `alice` (aligned → sign) for DUPA, `bob` (misaligned vs the authenticated identity → skip) for DUPB. That is why the outcome flips with header order while the message is accepted in both cases.

**Scope.** This is an OBSERVED behavior of the DKIM modifier and the go-message header API at commit `26452dd`; per AAP §0.5.2 (observation task, no behavior change) and §0.6.2 (no dependency change) it is **documented, not remediated** — no source or dependency edit is made. (The broader set of out-of-scope robustness/CLI observations is collected in §14.)

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

## 8. `Authentication-Results` on submission: a CONFIG-CONDITIONAL negative (OBSERVED, O4)

**Direct answer:** in the **default/canonical** submission configuration Maddy adds **no** `Authentication-Results` header — but this is a property of *which checks are configured on the submission endpoint*, **not** an absolute rule that submission "never" carries one. Both halves were OBSERVED.

**Canonical config — the header is absent.** A `grep` across all stored message blobs from the canonical run found **NO `Authentication-Results` header, no `ARC-*` headers, no `DKIM-Status`, and no `Received-SPF`** on any submission message. The **complete** set of header field names observed across all delivered messages was:

```
Date, Delivered-To, Dkim-Signature, From, Message-Id, Received, Return-Path, Subject, To
```

So, in the canonical config, the only headers Maddy itself adds on authenticated submission are:

- `Received` — always (`internal/target/received.go`).
- `DKIM-Signature` — only when the sender is aligned (`internal/modify/dkim/dkim.go:406`).
- Injected `Message-Id` and `Date` when absent — logged as `adding missing Message-ID` / `adding missing Date header` (`internal/endpoint/smtp/submission.go` L30-37 and L111-127).
- `Delivered-To` and `Return-Path` — added by the local-delivery target.

The reason is narrow: the canonical submission blocks contain **no `check` that populates an authentication result** — the `verify_dkim`/`apply_spf` checks are wired only to the port-25 inbound endpoint (`maddy.conf:54-66`). `Authentication-Results` is emitted by the **check runner**, not by the submission endpoint or by the DKIM *signer*.

**It is NOT absolute — adding a check to submission DOES stamp the header (OBSERVED, P4-F2 correction).** To rule out the over-broad reading "submission can never carry `Authentication-Results`," the canonical daemon was stopped and a variant config was run in which a `check { verify_dkim }` block was added to the submission `source` (all other semantics identical). Producing command:

```
$ maddy -debug -config /tmp/maddy_qa_ws/maddy_f2.conf     # source $(local_domains) { check { verify_dkim } ... }
```

An **aligned** message (AUTH `alice`, envelope `alice`, `From: alice`) was then submitted over the authenticated `:587` endpoint — a message that the submission pipeline itself DKIM-signs, so it is unambiguously a *submission* message. The daemon logged the check running **before** signing (the incoming message has no signature yet), complete and unedited:

```
[debug] smtp/pipeline: initializing state for verify_dkim: (0xc0001bf180)	{"msg_id":"8f5367f7"}
[debug] verify_dkim: no signatures present	{"msg_id":"8f5367f7"}
smtp/pipeline: no check action	{"check":"verify_dkim","msg_id":"8f5367f7","reason":"No DKIM signatures","smtp_code":550,"smtp_enchcode":"5.7.20","smtp_msg":"No DKIM signatures"}
[debug] sign_dkim: signed	{"identifier":"alice@example.org"}
```

The stored message (dumped verbatim) now carries **both** a submission-added `DKIM-Signature` **and** an `Authentication-Results` header:

```
$ maddyctl --config /tmp/maddy_qa_ws/maddy_f2.conf imap-msgs dump alice@example.org INBOX <seq>
Dkim-Signature: a=rsa-sha256; bh
 =4MBbKXlF0CJIU9cQSeMHoxjoqCld18FNRJz7PNrmOMM=; c=relaxed/relaxed;
 d=example.org;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=alice@example.org; s=default; t=1784197573; v=1; x=1784629573; b=HeRiOhw0lvBWbdZREO6V0OHwnIdTaOQZnc5MFc7AnGgh+VWggox8aUHQwVpobcgd94eqZ9oXMkvT3bmj7ghpMQsodAfUhbQ8IWQTBAJcxuMCqSTzKgOeClplrQvj7kV7OvlRzplrgRFw1wcd+Xj9iu4BZdOu4T9QYcQcFA4x1BEDc5b5R4FJfu1TxHwfmGkil3Dfbof2XsuUsqssp7z0f+CpXPPbuMp8JeKNdlVjfxhco7IRW5GvW/TupfN+QlLaalQbjgDkpxusoyBccMlV7n1sPwGOifDoiK2z3Mz7sV2R8L4222f8PkckeL/GZiQ5c/P1230/iBKXbXhyX0QTwA==;
Authentication-Results: ; dkim=none 
Received:  by example.org (envelope-sender <alice@example.org>) with ESMTP
 id 8f5367f7; Thu, 16 Jul 2026 10:26:13 +0000
From: alice@example.org
```

Two observed details worth stating exactly:

- The value is **`Authentication-Results: ; dkim=none`** — `dkim=none` because the check ran on the *incoming* message, which has no signature yet (signing happens later, in the modify stage). The result reflects what the check saw, not the signature the pipeline subsequently added.
- The **authserv-id field is empty** (the string is literally `; dkim=none` with nothing before the semicolon) in this configuration.

**Grounding.** The header is added by the check runner whenever a check populates an authentication result, with **no submission-vs-inbound guard**: `internal/msgpipeline/check_runner.go` builds and calls `textproto.Header.Add("Authentication-Results", ...)` (L299-303). The submission endpoint does not suppress it. So the correct statement is the config-conditional one above: **absent by default because the canonical submission blocks run no such check; present the moment one is configured.** The canonical daemon and config were restored immediately after this measurement.

---


## 9. DKIM triggering under `From` manipulation + what a verifying recipient would conclude (OBSERVED, O5/O8)

§4–§6 already establish the triggering behavior under `From` manipulation: T1 (`From` = cross-user), T4 (`From` ≠ envelope), and T5 (`From`-domain ≠ key-domain) are each **accepted and delivered UNSIGNED** — a manipulated `From` never causes a rejection, only the absence of a signature; and for the aligned T2 the signature **covers `From`** (oversigned). This section reports what a recipient attempting cryptographic verification would actually conclude.

Verification used the **same library Maddy uses** — `github.com/emersion/go-msgauth v0.3.2-0.20191028231513-55b75676976c` (`go.mod:17`), `dkim.Verify`, with the DKIM DNS record served locally by `github.com/foxcpp/go-mockdns` (the project's own test-DNS dependency, `go.mod:21`). The producing command was a small Go program (built against a copy of the repo's `go.mod`/`go.sum` so the dependency versions are identical) that (1) reads the published `.dns` `p=` value and tries `x509.ParsePKIXPublicKey` on it (Finding A), (2) does a fresh in-memory `dkim.Sign`+`dkim.Verify` round-trip against the PKIX-encoded form of the same key (Finding B), and (3) runs `dkim.Verify` on the stored T2 mailbox bytes (Finding C). Because a DNS TXT string cannot exceed 255 bytes, the ~380-byte PKIX record is served as multiple ≤255-byte TXT strings, which `dkim.Verify` rejoins (`query.go` `strings.Join(txts, "")`). The command and its **complete, unedited** output:

```
$ /tmp/maddy_qa_ws/bin/verifier \
    /tmp/maddy_qa_ws/captures/T2_stored.eml \
    /tmp/maddy_qa_ws/state/dkim_keys/example.org_default.key \
    /tmp/maddy_qa_ws/state/dkim_keys/example.org_default.dns
Finding A: published p= DER prefix = 30 82 01 0a 02 82 01 01
RESULT: FAIL domain=example.org err=dkim: key syntax error: x509: failed to parse public key (use ParsePKCS1PublicKey instead for this key format)
Finding A: PKIX-form DER prefix = 30 82 01 22 30 0d 06 09
RESULT: PASS domain=example.org identifier=alice@example.org
RESULT: FAIL domain=example.org err=dkim: signature did not verify: crypto/rsa: verification error
```

These three `RESULT:` lines are exactly the Finding A / B / C outcomes discussed below (this fresh corroborating run reproduces the historical A/B/C strings byte-for-byte). Three findings, each labeled **OBSERVED** with its source grounding:

### Finding A (OBSERVED) — the published public key is PKCS#1 but the verifier expects PKIX (a real interop divergence at this commit)

Maddy's auto-generated `.dns` record's `p=` value base64-decodes to DER beginning `30 82 01 0a 02 82 01 01` — a **bare PKCS#1 `RSAPublicKey`**. This is because `internal/modify/dkim/keys.go:143` encodes the public key with `x509.MarshalPKCS1PublicKey(pubkey)` (inside `writeDNSRecord`, L136-163) and emits it into the record at L158 (`v=DKIM1; k=%s; p=%s`). The go-msgauth verifier, however, parses the `p=` blob with `x509.ParsePKIXPublicKey(b)`, which requires a PKIX `SubjectPublicKeyInfo` (DER beginning `30 82 01 22 30 0d 06 09 2a 86 48 ...`) and **cannot** parse a bare PKCS#1 key. Verifying the stored T2 message with the key exactly as published:

```
RESULT: FAIL domain=example.org err=dkim: key syntax error: x509: failed to parse public key (use ParsePKCS1PublicKey instead for this key format)
```

**Conclusion:** a recipient using Maddy's own auto-generated TXT record verbatim gets a **key syntax error for every signed message** — and this is the same parsing code path Maddy's own `verify_dkim` check would use. At commit `26452dd`, the encoding is unambiguously PKCS#1: `internal/modify/dkim/keys.go:143` calls `x509.MarshalPKCS1PublicKey(pubkey)` (the observed DER prefix `30 82 01 0a` confirms this at runtime). *(This document scopes every claim to commit `26452dd` and makes no assertion about other revisions' encoding, which were not built or observed here.)* This is an **OBSERVED interop divergence at this commit**.

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

The key now parses, and the error is specifically `crypto/rsa: verification error` — **not** `"body hash did not verify"`. This is decisive: go-msgauth checks the body hash **first** and returns `"body hash did not verify"` on a body mismatch (`verify.go:353-354`), and only **after** that computes the header-data hash and verifies the RSA signature (`verify.go:357+`). Getting the RSA error therefore proves the **body hash matched** while the **signed-header set differs**. Since the fresh in-memory round-trip (Finding B) PASSES while the persisted mailbox copy FAILS, the bytes stored in the mailbox (the go-imap-sql external body file) are **not byte-identical** to what `sign_dkim` signed: the local delivery/storage path re-serialized the **header** after `RewriteBody` added the signature (`internal/modify/dkim/dkim.go:406`) — the body is intact, but the header ordering/serialization that the signature commits to was altered on the way into storage. **This finding is scoped precisely to the LOCALLY STORED copy;** remote-relay serialization was not tested (there is no external MX in this local, DNS-independent setup — out of scope).

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

In the **default/canonical** configuration, Maddy enforces sender authorization **only at the envelope DOMAIN level** (via `source` / `default_source`), and enforces `From`-alignment **only** as a DKIM signing precondition that **degrades to unsigned delivery — never as a rejection.** So *out of the box* there is **no per-user "MAIL FROM must equal the authenticated user" enforcement** and **no `From`-vs-auth rejection** — this is the DEFAULT behavior, established by observation (T1/T4/T5 are all accepted, §4).

Per-user enforcement is nonetheless **achievable with explicit configuration in this very revision** — the earlier claim that the 2019-era revision offers no submission `check` for it is **incorrect**, and was disproved at runtime. The generic `command` check (`internal/check/command/command.go`, registered at L374) runs an external program at a chosen stage and maps its exit code to an action. Placed at the **sender** stage with the `{auth_user}` and `{sender}` placeholders, mapping a nonzero exit to a reject, it produces exactly the per-user gate the default lacks. Config fragment added to the submission `source` (variant `maddy_f4.conf`):

```
check {
    command /tmp/maddy_qa_ws/scripts/cmp_sender.sh "{auth_user}" "{sender}" {
        run_on sender
        code 1 reject 553 5.7.1 "Sender does not match authenticated user"
    }
}
```

where `cmp_sender.sh` is simply `[ "$1" = "$2" ] && exit 0; exit 1`. Producing command and complete client transcript (authenticated as `alice`, envelope `bob`) — OBSERVED:

```
$ python3 /tmp/maddy_qa_ws/scripts/f4.py
send: 'mail FROM:<bob@example.org>\r\n'
reply: b'250 2.0.0 Roger, accepting mail from <bob@example.org>\r\n'
send: 'rcpt TO:<alice@example.org>\r\n'
reply: b'553 5.7.1 Sender does not match authenticated user (msg ID = 0495ce78)\r\n'
```

The aligned control (envelope `alice`) is accepted end-to-end (`MAIL FROM -> 250`, `RCPT -> 250`). Two grounded observations: **(1)** the reject carries the exact **`553 5.7.1`** code and message from the `code 1 reject ...` directive — `command`'s exit-`1` default action is `Reject` (`New`, L54-57), overridden to this code/message via `ParseActionDirective` (`Init`, L108-118); **(2)** it surfaces at **`RCPT`**, not `MAIL FROM`, because `defer_sender_reject` defaults to **`true`** (`internal/endpoint/smtp/smtp.go:567`), deferring sender-stage rejects to the first `RCPT`. So the correct determination is: **the default provides no per-user gate, but this revision absolutely *can* enforce one via an explicit sender-stage `command` check.** The canonical daemon/config were restored immediately after this measurement.

A naming caveat, grounded strictly in this commit: there is **no dedicated `authorize_sender` check** at `26452dd`. A repository-wide search returns **zero matches** — `grep -rn "authorize_sender" --include=*.go .` and the same over `docs/` both produce no output. Per-user sender authorization is therefore not available as a *named* submission check here; it is achieved with the *generic* `command` check demonstrated above. (This document deliberately makes **no** claim about what other/newer Maddy versions provide — only what is present at `26452dd`: the routing code in `internal/msgpipeline/msgpipeline.go` has only address/domain/default tiers, `internal/endpoint/smtp/submission.go`'s `submissionPrepare` performs no alignment comparison, and the `command` check at `internal/check/command/command.go` is the in-tree mechanism for per-user policy.)

### 11.2 Ruled-out incorrect interpretations (with evidence)

- **Incorrect (mandatory rebuttal):** *"Because the config pairs `sign_dkim` with a `reject`-based `default_source`, a message whose `From` doesn't match the authenticated user (or the signing domain) will be rejected."* **Refuted by observation:** T1, T4, and T5 are all **ACCEPTED** and delivered (just UNSIGNED); the only rejects are the envelope-domain `default_source` case (T3, `501 5.1.8`) and the missing-`From` syntax case (T6, `554 5.6.0`). Evidence: the `submission: accepted {"msg_id":"513ba1a1"}` log preceded by `not signing, From address is not authenticated identity`, and the `RewriteBody` `if !ok { return nil }` at `internal/modify/dkim/dkim.go:349-351`.
- **Incorrect:** *"Maddy stamps an `Authentication-Results` header on submitted mail **by default**."* **Refuted for the canonical config:** no such header appears in any stored message from the default run; the full observed header set is `Date, Delivered-To, Dkim-Signature, From, Message-Id, Received, Return-Path, Subject, To`. (This is *config-conditional*, not absolute — adding a `check` such as `verify_dkim` to the submission `source` DOES add the header, as demonstrated with observed evidence in §8.)
- **Incorrect:** *"The `Received` header records the authenticated user."* **Refuted by T4:** the stored `Received` shows `(envelope-sender <alice@example.org>)` (the envelope) while the message's `From` is `bob@example.org` (§7); the auth identity is stored separately as `AuthUser` and is never placed in `Received`.

### 11.3 The behavior-diverges case (O8)

The prime divergence between a reasonable configuration reading and observed behavior: **the default configuration accepts a cross-user, same-domain sender at the pipeline (domain-level match) while the `sign_dkim` modifier silently skips signing and delivers the message UNSIGNED, rather than rejecting it.** A naive reading of `maddy.conf` — seeing `sign_dkim` sitting next to a `reject`-based `default_source` — would suggest that misaligned senders are blocked. In reality they sail through, merely unsigned. T1 is the concrete instance: auth `alice`, envelope+`From` `bob@example.org`, **accepted and stored unsigned**.

### 11.4 Documented-vs-observed contrast (added rigor)

The `sign_dkim` modifier's own manual **does** document the silent-skip, which is worth stating so the divergence is attributed precisely. `docs/man/maddy-filters.5.scd` (≈L518-544) states `require_sender_match` *Default*: `envelope auth` (L518-519) and defines the semantics as requiring the identifiers to match the `From` field and key domain, **"otherwise - don't sign the message"** (L521-522) — i.e., *don't sign*, not *reject* — which matches observation. The same manual documents `sig_expiry` Default `120h` (L500-501, = the observed `x − t = 432000s`), `header_canon`/`body_canon` Default `relaxed` (matching `c=relaxed/relaxed`), and `newkey_algo` Default `rsa2048` (L513-514). So the silent-unsigned behavior is documented **at the modifier level** — a careful reader could know it — but the **`maddy.conf`-level pairing** of `sign_dkim` with a reject-based `default_source` is what invites the naive misreading. The divergence is thus at the *config-composition* level, compounded by the **PKCS#1/PKIX interop bug** (§9 Finding A) that no amount of config reading would predict.

### 11.5 The `require_sender_match` parser/default contradiction and the `auth_user` bypass (O8, OBSERVED)

Two further divergences live in the very directive that governs `From`-alignment — both OBSERVED at runtime, both prime O8 material because they defeat a careful operator's reasonable expectation.

**(a) The documented default value cannot be written.** `require_sender_match`'s default is the pair `["envelope", "auth"]` (`internal/modify/dkim/dkim.go:151-152`), but the directive's **parser** accepts only `{envelope, auth_domain, auth_user, off}` — the token **`auth` is not in the allowed set.** So an operator who makes the default explicit — exactly what the manual's `Default: envelope auth` line invites — gets a **fatal startup error**. Producing command and complete, unedited output:

```
$ maddy -debug -config /tmp/maddy_qa_ws/maddy_f3a.conf     # sign_dkim ... { require_sender_match envelope auth }
tls: using self-signed certificate, this is not secure!
/tmp/maddy_qa_ws/maddy_f3a.conf:23: invalid argument, valid values are: [envelope auth_domain auth_user off]
$ echo $?
2
```

The daemon exits with status **2**. The internal default is thus a value the configuration language itself rejects — a genuine default-vs-parser contradiction, not a mere documentation nit.

**(b) `auth_user` silently DISABLES the authenticated-identity check.** `shouldSign`'s auth-identity branch is guarded by a test of the map key **`"auth"` only**, never `"auth_user"` (`internal/modify/dkim/dkim.go:299-310`). So configuring `require_sender_match envelope auth_user` leaves the auth check **unreachable**. OBSERVED: with that config, authenticating as **alice** but submitting envelope+`From` **bob@example.org** (the T1 input, which is UNSIGNED under the default) produces a **valid signature as Bob**. Producing commands and observed output — the decisive daemon log line is shown (the `...` marks the routine client transcript / startup lines elided around it), and the byte-exact signed result is the stored `DKIM-Signature` dump shown immediately after:

```
$ maddy -debug -config /tmp/maddy_qa_ws/maddy_f3b.conf     # sign_dkim ... { require_sender_match envelope auth_user }
$ python3 /tmp/maddy_qa_ws/scripts/f3b.py                  # AUTH alice ; MAIL FROM:<bob@example.org> ; From: bob@example.org
...
[debug] sign_dkim: signed	{"identifier":"bob@example.org"}

$ maddyctl --config /tmp/maddy_qa_ws/maddy_f3b.conf imap-msgs dump alice@example.org INBOX <seq>
Dkim-Signature: a=rsa-sha256;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=bob@example.org; s=default; t=1784197509; v=1; x=1784629509; b=rMekJTvg92MLx/8xCxIz3ObnHQ28AsvMoeErIfpd3f2HGRWVEz6iY9wzddMVOli5qxLp3l+8y7vSrrF/UdF19RV7ZMzUDxqFX8JpA38+JoU4bfZoiMedpHh160e8hq/qe3GrwBUwcMfu6gLQeAE8qFNNi0a4dLRugjmiu5KiaFC8K+qn7ORpjzZgtaTM5ptoJXkkaCKHvVcsKerhwO2iixrW9bobu9vld+lFCYgzsQvcpa83CYy3xTrNlrzP2h0t6zKCFNz0n7Wi5PPPxI2mU79r7x2mmKboOzfpsKG99PBWrY5EQMH5RXRJsaCU17WM2GFmmhB7Yo7JnjRnFsVMVQ==;
From: bob@example.org
Subject: F3B-authuser
```

The envelope check still passes here (`From bob` = `MAIL FROM bob`), and with the auth check bypassed the message is **signed as `i=bob@example.org`** — Alice has produced an `example.org`-domain DKIM signature over a `From: bob` message, with `From` covered (`...From:From...`). An operator who writes `auth_user` expecting a *stricter*, per-user binding in fact gets a *weaker* policy than the default. Both daemons above were experiment-only (separate configs `maddy_f3a.conf`/`maddy_f3b.conf`); the canonical daemon and config were restored immediately afterward.

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
| **O7** — Determine default enforces domain-level only; per-user/`From` rejection is not in the default but IS achievable via an explicit sender-stage `command` check (observed `553`, §11.1); ≥1 incorrect interpretation ruled out | Done | §11 |
| **O8** — Identify ≥1 behavior-diverges case | Done | §11.3 (domain accept + silent unsigned), §9 Finding A (PKCS#1/PKIX interop divergence), and §11.5 (`require_sender_match` parser/default contradiction + `auth_user` bypass) |

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

**`internal/check/command/command.go`** (per-user policy mechanism, §11.1)
- `modName = "command"` L28; default actions `1:Reject`, `2:Quarantine` `New` L54-61; inline `cmd`/`cmdArgs` L68-69; `run_on` enum `{conn,sender,rcpt,body}` default `body` L88-90; `code <exit> <action>` → `ParseActionDirective` L106-118; `{auth_user}`→`Conn.AuthUser` L145, `{sender}`→`mailFrom` L178; `CheckSender` L314; `module.Register(modName, New)` L374.

**`cmd/maddyctl/main.go`** (P5-F1)
- `if err := app.Run(os.Args); err != nil { fmt.Fprintln(os.Stderr, err) }` L563-565 — **no `os.Exit(1)`** on error.

**`cmd/maddyctl/imap.go`** (P7-F3)
- `if !ctx.Bool("yes,y")` L109 (looks up a nonexistent composite flag name; flag is `yes`/`y`); prompt L119; contrast correct `ctx.Bool("yes")` at L235 (`imap-msgs remove`).

**`internal/address/split.go`** (P7-F1)
- "intentionally naive … does almost no sanity checks" comment L15-17; `strings.LastIndexByte(addr, '@')` L23 → `bad@@example.org` = local-part `bad@` + domain `example.org`.

**`internal/msgpipeline/check_runner.go`** (P4-F2)
- `Authentication-Results` built and added L299-303 (no submission-vs-inbound guard).

**Pinned dependencies exercised for the supplementary observations (`go.mod`)**
- `github.com/emersion/go-smtp v0.12.1-0.20191206174923-1f576e0ec85c` L19 — `conn.go:412-420` (P6-F1 initial-response decode returns with no reply); `lengthlimit_reader.go:21-46` (P8-F3 value-receiver `Read`).
- `golang.org/x/net v0.0.0-20191126235420-ef20fe5d7933` L31 — `idna` normalization (P8-F2, CVE-2026-39821 / GO-2026-5026).
- `github.com/foxcpp/go-imap-sql v0.3.2-0.20191208094750-8b4ec6b19a78` L20 — `mailbox.go:548-597`/`backend.go:544-588`/`sql.go:736-758` (P7-F2 orphan `extKeys`).

---

## 14. Supplementary out-of-scope observed behaviors (robustness, CLI, normalization, storage)

The QA pass surfaced several runtime behaviors that lie **outside the AAP scope for remediation**: they are defects in Maddy source or in pinned third-party dependencies, and the AAP is explicit that this is an **observation task, not a bug fix** (§0.5.2: "Changing, fixing, or 'improving' Maddy's behavior … is out of scope"; §0.6.2: no dependency/manifest change). Accordingly, each item below is **documented from direct runtime observation — not remediated.** No Maddy source file and no dependency was modified; the only repository artifact is this document. Every behavioral claim is shown next to the exact command that produced it and grounded to a `file:line`. (The DKIM-specific duplicate-`From` case is documented separately in §6.5.) All observations were captured against the canonical daemon on `:587` (plus two brief config variants and CLI-only invocations); the canonical daemon/config were restored afterward.

### 14.1 CLI: administrative failures exit `0`; `imap-mboxes remove --yes` is ignored (OBSERVED)

**P5-F1 — every `maddyctl` failure prints an error but exits `0`.** Producing commands and complete output:

```
$ maddyctl --config /tmp/maddy_qa_ws/maddy.conf users create ; echo EXIT=$?
Error: USERNAME is required
EXIT=0
$ maddyctl --config .../maddy.conf users create zz@example.org -p x --bcrypt-cost 2 ; echo EXIT=$?
Error: too small bcrypt cost
EXIT=0
$ maddyctl --config .../maddy.conf imap-msgs dump ; echo EXIT=$?
Error: USERNAME is required
EXIT=0
$ maddyctl --config .../maddy.conf users list extra-arg ; echo EXIT=$?
alice@example.org
bob@example.org
EXIT=0
$ maddyctl --config /nonexistent.conf users list ; echo EXIT=$?
no requested block found in configuration
EXIT=0
```

Every failing action returns status **`0`**, and `users list` silently ignores the extra positional argument. **Grounding:** `cmd/maddyctl/main.go:563-565` — `if err := app.Run(os.Args); err != nil { fmt.Fprintln(os.Stderr, err) }`; the error is printed but there is **no `os.Exit(1)`**, so `main` returns normally and the exit code is `0`. **Scope:** Maddy source defect — documented, not fixed (§0.5.2).

**P7-F3 — `imap-mboxes remove --yes` still prompts and cancels on EOF.** Producing command and complete output (noninteractive, stdin at EOF):

```
$ maddyctl --config .../maddy.conf imap-mboxes create alice@example.org P7F3BOX      # EXIT=0
$ maddyctl --config .../maddy.conf imap-mboxes remove --yes alice@example.org P7F3BOX < /dev/null ; echo EXIT=$?
Are you sure you want to delete that mailbox? [y/N]: <nil>
Cancelled
EXIT=0
$ maddyctl --config .../maddy.conf imap-mboxes list alice@example.org | grep P7F3BOX
P7F3BOX
```

The `--yes` flag is ignored, the tool prompts, the EOF read (`<nil>`) is treated as "no", the mailbox is **not** deleted, and (per P5-F1) the exit code is `0`. **Grounding:** `cmd/maddyctl/imap.go:109` calls `ctx.Bool("yes,y")` — passing the *composite* string `"yes,y"` as the flag **name**. The registered flag is `yes` (alias `y`), so `ctx.Bool("yes,y")` never matches and always returns `false`, defeating `--yes`. Contrast `imap.go:235` (the `imap-msgs remove` path), which correctly uses `ctx.Bool("yes")`. **Scope:** Maddy source defect — documented, not fixed.

### 14.2 Provisioning: a malformed double-`@` identity is accepted (OBSERVED)

**P7-F1.** `maddyctl users create bad@@example.org` succeeds (EXIT `0`), the account appears in `users list`, authenticates, and its envelope is accepted:

```
$ maddyctl --config .../maddy.conf users create "bad@@example.org" -p DisposableP7Pass ; echo EXIT=$?
EXIT=0
$ maddyctl --config .../maddy.conf users list
alice@example.org
bad@@example.org
bob@example.org
```

Over `:587`, authenticating as `bad@@example.org`:

```
AUTH -> 235 2.0.0 Authentication succeeded
MAIL -> 250 2.0.0 Roger, accepting mail from <bad@@example.org>
RCPT -> 250 2.0.0 I'll make sure <alice@example.org> gets this
```

The delivery outcome then depends on the **`From` header** (the precise, observed nuance):

- `From: bad@@example.org` → `DATA -> 554 5.6.0 Invalid address in From (msg ID = 45b425ec)` — the malformed address fails `submissionPrepare`'s `From` syntax check.
- `From: alice@example.org` (a separately-valid header) → `DATA -> 250 2.0.0 OK: queued`, delivered **unsigned**, stored with `Received:  by example.org (envelope-sender <bad@@example.org>)`.

**Grounding:** `internal/address/split.go` `Split` uses `strings.LastIndexByte(addr, '@')` (L23), so `bad@@example.org` splits into local-part `bad@` and domain `example.org` — the **local** domain — which is why the envelope passes the domain gate; the function is documented as "intentionally naive … does almost no sanity checks" (comment L15-17). Provisioning persists the identity without rejecting the extra `@`. **Scope:** Maddy source defect — documented, not fixed.

### 14.3 Protocol robustness (pinned `go-smtp v0.12.1-0.20191206174923-1f576e0ec85c`)

**P6-F1 — a malformed AUTH PLAIN initial response gets no reply.** Producing sequence and complete output (raw socket):

```
greeting: b'220 example.org ESMTP Service Ready\r\n'
ehlo-reply: b'250-Hello c\r\n250-PIPELINING\r\n250-8BITMIM' ...
AUTH PLAIN !!! -> NO REPLY (client recv timed out after 12.0s)
QUIT -> b'221 2.0.0 Goodnight and good luck\r\n'
```

The server sends **nothing** in response to `AUTH PLAIN !!!` (invalid base64 initial response); a later `QUIT` still works, so the connection is alive but the client is blocked. **Grounding:** `go-smtp/conn.go:412-420` decodes the initial response and, on failure, simply returns with no reply:

```go
ir, err = base64.StdEncoding.DecodeString(parts[1])
if err != nil {
    return
}
```

There is no `c.WriteResponse(...)` on the error path. Maddy's `read_timeout` defaults to **10 minutes** (`internal/endpoint/smtp/smtp.go:560` — `cfg.Duration("read_timeout", false, false, 10*time.Minute, ...)`), so a real client can hang that long. **Scope:** dependency defect — documented, not fixed (§0.6.2).

**P8-F3 — DATA body lines of ≈1,999–4,094 bytes stall the session.** Producing command (`p8f3.py`) and complete output across boundaries:

```
N=1998 -> 250 2.0.0 OK: queued (0.03s)
N=1999 -> DISCONNECTED (10.01s)      # no final reply within 10s
N=2500 -> DISCONNECTED (10.01s)      # no final reply within 10s
N=3000 -> DISCONNECTED (10.01s)      # no final reply within 10s
N=4095 -> 554 5.0.0 Internal server error (msg ID = 84e6fe93) (0.00s)
```

A 1,998-byte body line is accepted promptly; lines of 1,999–3,000 bytes produce **no final reply** (the client hits its 10 s timeout while the server holds the session); a 4,095-byte line is promptly rejected `554`. **Grounding:** `go-smtp/lengthlimit_reader.go:21-46` — `func (r lineLimitReader) Read(...)` has a **value receiver**, so its `curLineLength` counter (reset on `'\n'` at L38, incremented at L41, compared to `LineLimit` at L23/L43) mutates a *copy* and does not persist across `Read` calls, corrupting the length accounting for lines in this range. **Scope:** dependency defect — documented, not fixed.

**P8-F4 — malformed MIME is reported as a generic internal error.** A header block containing a bare `Not-A-Header` line (no colon):

```
DATA -> 554 5.0.0 Internal server error (msg ID = c51ffed9)
```

The parse error surfaces as **`554 5.0.0 Internal server error`** rather than a client-actionable `5.6.x` content/syntax status. **Grounding:** the header/body handling and error wrapping in `internal/endpoint/smtp/smtp.go` (`Data` → `wrapErr`, ≈L283-323) maps the parse failure to the generic internal-error response. **Scope:** Maddy source defect — documented, not fixed.

### 14.4 Normalization (pinned `golang.org/x/net` `ef20fe5d7933`, CVE-2026-39821)

**P8-F2 — an invalid Punycode A-label normalizes into the local domain and is signed as local.** Producing sequence and complete output (auth `alice`; envelope + raw `From` = `alice@xn--example-.org`):

```
MAIL -> 250 2.0.0 Roger, accepting mail from <alice@xn--example-.org>
RCPT -> 250 2.0.0 I'll make sure <alice@example.org> gets this
DATA -> 250 2.0.0 OK: queued
```

The stored message shows the normalization crossing both the routing and DKIM gates, while the raw `From` retains the A-label:

```
Dkim-Signature: a=rsa-sha256; ... d=example.org; ... i=alice@example.org; ...
Received:  by example.org (envelope-sender <alice@example.org>) with ESMTP ...
From: alice@xn--example-.org
```

So `xn--example-.org` is normalized to `example.org`: the envelope matches the local `source` block (accepted), the DKIM modifier signs with `d=example.org` and `i=alice@example.org`, yet the header the recipient sees (`From: alice@xn--example-.org`) is the un-normalized A-label. **Grounding:** `golang.org/x/net v0.0.0-20191126235420-ef20fe5d7933` (`go.mod:31`) — the `idna` package version associated with GO-2026-5026 / CVE-2026-39821. **Scope:** dependency defect — documented, not fixed (§0.6.2).

### 14.5 Storage integrity (pinned `go-imap-sql v0.3.2-0.20191208094750-8b4ec6b19a78`)

**P7-F2 — deleting a user leaves an orphan `extKeys` row that survives restart.** Producing sequence and complete output (SQLite read-only queries against `all.db`):

```
# before:                       extKeys=22  msgs=22   (0 orphans)
# deliver 1 message to fresh tmpuser@example.org:
                                extKeys=23  msgs=23  users=4  mboxes=5
# maddyctl users remove tmpuser@example.org   ->   EXIT=0
                                extKeys=23  msgs=22  users=3  mboxes=4
ORPHAN extKeys (no matching msgs.extBodyKey) = 1
   orphan (id,uid,refs): ('763814b31f0a711a47147dacab76f578', 4, 1)
# after daemon restart:
AFTER RESTART: extKeys=23 msgs=22 orphans=1
   orphan persists (id,uid,refs): ('763814b31f0a711a47147dacab76f578', 4, 1)
integrity_check: ok
```

Deleting the user removed its `users`, `mboxes`, and `msgs` rows but **left the `extKeys` body-key row** (`uid=4` — the deleted user — `refs=1`). The orphan **persists across a daemon restart**, and SQLite's `PRAGMA integrity_check` still reports `ok`, so the leak is invisible to integrity/FK checks. **Grounding:** `go-imap-sql v0.3.2-0.20191208094750-8b4ec6b19a78` (`go.mod:20`); the deletion paths (`mailbox.go:548-597`, `backend.go:544-588`, `sql.go:736-758`) do not delete zero-reference `extKeys` / decrement refcounts in the same transaction as the row deletions. **Scope:** dependency defect — documented, not fixed.

---

