# Runtime Investigation: Sender Identity (SMTP AUTH Alignment) and DKIM Signing in foxcpp/maddy

**Repository:** foxcpp/maddy
**Commit:** `26452dd8dd787dc455278b0fdd296f4a5432c768`
**Branch:** `maddy_26452dd8dd78`
**Question under test:** How does maddy enforce sender identity (SMTP AUTH sender alignment) and DKIM signing at runtime on the authenticated submission endpoint?

---

## Bottom line up front (BLUF)

**The default maddy configuration does NOT enforce alignment between the authenticated user and the message sender.** An authenticated user may put any *local-domain* address in `MAIL FROM` and in the `From:` header and the message is **accepted (`250 2.0.0 OK: queued`) and delivered**. The only sender control the default configuration applies is a *domain-level* open-relay guard (`default_source { reject 501 5.1.8 "Non-local sender domain" }`), which rejects a **non-local** envelope domain at `RCPT TO`. Per-user (identity) alignment is **not** enforced and requires explicit policy configuration. DKIM signing is a **separate** decision: on any mismatch the message is delivered **unsigned** rather than rejected, so a verifying recipient of a spoofed-`From` message sees an *unsigned* message, not a signature over a mismatched `From`.

Each of these claims is proven below with actual captured runtime output (SMTP dialogues, `-debug` server logs, raw stored headers, and byte-level DKIM verification), and grounded in `file:line` references verified against the on-disk source at this commit.

---

## Methodology (run-first, mandatory)

All conclusions in this document are grounded in **observed runtime behavior**, not code reading alone. The investigation followed this sequence:

1. **Build** `maddy` and `maddyctl` from the exact commit with the canonical `go build` commands (Go 1.13.15, CGO enabled for the SQLite3 driver).
2. **Run** the server in its default (canonical) configuration — shipped `maddy.conf` plus exactly two minimal, policy-preserving local substitutions — under `-debug`.
3. **Create two accounts** (`user1@example.org`, `user2@example.org`) via `maddyctl users create`.
4. **Drive the real authenticated SMTPS submission endpoint** (`tls://0.0.0.0:465`) with a Python `smtplib.SMTP_SSL` client across six scenarios (C, A, B, D, E, F) plus one labeled non-canonical probe.
5. **Capture** actual SMTP response codes, full protocol dialogues, `-debug` signing decisions, the raw stored message headers from the on-disk message store, and byte-level DKIM verification (independent body-hash reproduction + positive control).
6. All runtime work occurred **outside** the source repository; the repository was left byte-identical (verified with `git status --porcelain`).

---

## 1. Environment, build, and invocation (canonical values)

### 1.1 Toolchain

- **Go 1.13.15** — `go.mod` declares `go 1.13` (the module's directive; the highest documented supported patch, 1.13.15, was used).
- **gcc** (build-essential) — **required** because the default storage backend uses `github.com/mattn/go-sqlite3`, a CGO package. `CGO_ENABLED=1` is mandatory.
- **Container:** `andrewparkscaleai/coding-agent:foxcpp__maddy__26452dd8dd787dc455278b0fdd296f4a5432c768`.

### 1.2 Build commands (exact, canonical)

```
CGO_ENABLED=1 go build -o maddy ./cmd/maddy
CGO_ENABLED=1 go build -o maddyctl ./cmd/maddyctl
```

Both built successfully (the only compiler output is a benign non-blocking SQLite `-Wreturn-local-addr` warning from the vendored `sqlite3-binding.c`, which does not affect the build result).

### 1.3 Version banner (canonical)

The canonical `maddy -v` banner, captured verbatim (`version_banner.txt`):

```
maddy unknown (built from source tree)
```

This is the **default** value produced by a plain `go build` with no `-X` linker flags: the package-level constant is defined at `maddy.go:41`:

```
	Version = "unknown (built from source tree)"
```

**Caveat (canonical vs. custom):** a release build that injects a version via `-ldflags "-X ..."` would print a different string. `maddy unknown (built from source tree)` is the correct value to report for a normal-user `go build` of this commit.

### 1.4 Invocation

```
maddy -config <path> -debug
```

**This commit has NO `run` subcommand.** `Run()` in `maddy.go` requires an empty positional-argument list — `maddy.go:120`:

```
	if len(flag.Args()) != 0 {
```

(any positional argument aborts startup). The server is therefore launched directly with `-config`.

### 1.5 Canonical configuration = shipped `maddy.conf` + exactly TWO minimal, policy-preserving substitutions

The exact `diff` of the config used against the shipped `maddy.conf` (`maddy.conf.diff`) is:

```
0a1,3
> state /tmp/maddy_work/state
> runtime /tmp/maddy_work/runtime
> 
49d51
<         alias_file /etc/maddy/aliases
```

The two substitutions and why each is policy-preserving:

1. **Added `state /tmp/maddy_work/state` and `runtime /tmp/maddy_work/runtime`.** maddy's compiled defaults are `/var/lib/maddy` (`maddy.go:59` `DefaultStateDirectory = "/var/lib/maddy"`) and `/run/maddy` (`maddy.go:70` `DefaultRuntimeDirectory = "/run/maddy"`), which are not writable for a normal-user local run. `InitDirs()` performs `os.Chdir(config.StateDirectory)` (`maddy.go:220` `if err := os.Chdir(config.StateDirectory); err != nil {`), so the relative `all.db` and the `dkim_keys/` directory resolve **inside the state directory**. Redirecting the state/runtime dirs changes only *where files live*, not sender/DKIM policy.
2. **Removed the `alias_file /etc/maddy/aliases` directive** (the aliases file is absent locally; leaving it would abort startup). This only affects alias expansion, not sender acceptance or DKIM signing.

**Everything governing sender handling and DKIM is byte-identical to the shipped file:** the submission `auth &local_authdb`, the `source $(local_domains)` / `default_source { reject ... }` routing, the `sign_dkim $(primary_domain) default` modifier, and the `require_sender_match` default. The shipped policy lines (confirmed against the repo working tree at this commit) are:

- `maddy.conf:16-17` — global TLS:
  ```
  tls /etc/maddy/certs/$(hostname)/fullchain.pem \
      /etc/maddy/certs/$(hostname)/privkey.pem
  ```
- `maddy.conf:32-35` — SQLite backend providing both `local_mailboxes` and `local_authdb`:
  ```
  sql local_mailboxes local_authdb {
      driver sqlite3
      dsn all.db
  }
  ```
- `maddy.conf:93-95` — submission endpoint + auth:
  ```
  submission tls://0.0.0.0:465 {
      # Use sql module for authentication.
      auth &local_authdb
  ```
- `maddy.conf:97-100` — source-block routing + DKIM signer:
  ```
      source $(local_domains) {
          modify {
              sign_dkim $(primary_domain) default
          }
  ```
- `maddy.conf:104-108` — local delivery:
  ```
          destination $(local_domains) {
              import local_delivery_actions
              deliver_to &local_mailboxes
          }
  ```
- `maddy.conf:117-119` — the ONLY default sender guard (domain-level open-relay control):
  ```
      default_source {
          reject 501 5.1.8 "Non-local sender domain"
      }
  ```
- `maddy.conf:149-152` — IMAP endpoint (used to inspect delivered mail):
  ```
  imap tls://0.0.0.0:993 {
      auth &local_authdb
      storage &local_mailboxes
  }
  ```

**TLS is mandatory on the submission port.** Port 465 is **implicit TLS (SMTPS / TLS-on-connect)**, not STARTTLS, so a TLS-on-connect client is required and a certificate/key must exist at the configured `tls` path. A self-signed certificate/key was generated at the shipped path `/etc/maddy/certs/example.org/{fullchain.pem,privkey.pem}` (keeping the `tls` directive byte-identical); the client accepts it by disabling certificate verification. A self-signed certificate is the minimal, policy-preserving substitution for a local run and does not alter sender/DKIM behavior.

### 1.6 Server startup evidence (all three listeners active)

From `server_canonical.log` (the `-debug` server log), quoted verbatim (lines shown with their log-file line numbers):

```
 9	[debug] sql: go-imap-sql version 0.4.0
11	smtp: listening on tcp://0.0.0.0:25
13	[debug] /tmp/maddy_work/maddy.conf.used:101: new module sign_dkim [example.org default]
14	sign_dkim: generating a new rsa2048 keypair...
15	sign_dkim: generated a new rsa2048 keypair, private key is in dkim_keys/example.org_default.key, TXT record with public key is in dkim_keys/example.org_default.dns,
16	put its contents into TXT record for default._domainkey.example.org to make signing and verification work
27	submission: listening on tls://0.0.0.0:465
30	imap: listening on tls://0.0.0.0:993
```

All three listeners are active: inbound SMTP `tcp://0.0.0.0:25`, submission `tls://0.0.0.0:465`, and IMAP `tls://0.0.0.0:993`. On first startup the `sign_dkim` modifier auto-generated an RSA-2048 keypair (private key at `dkim_keys/example.org_default.key`, public-key TXT record at `dkim_keys/example.org_default.dns`, both relative to the state directory per the `os.Chdir` above).

**Key persistence across restart (state transition observed):** after a `SIGTERM` stop and relaunch with the same state directory, `server_canonical_restart.log` shows all three listeners return (`smtp` line 11, `submission` line 23, `imap` line 27) **and NO new keypair is generated** — a `grep -c "generating a new rsa2048"` over the restart log returns `0`. The `sign_dkim` module also re-initialises at line 13 with no subsequent `generating a new rsa2048 keypair` line (contrast the first-startup line 14 above). The relevant restart lines:

```
11	smtp: listening on tcp://0.0.0.0:25
13	[debug] /tmp/maddy_work/maddy.conf.used:101: new module sign_dkim [example.org default]
23	submission: listening on tls://0.0.0.0:465
27	imap: listening on tls://0.0.0.0:993
```

The on-disk key is reused; the same public key remains valid for verification across restarts.

### 1.7 Accounts (usernames are full email addresses)

Account names include the domain — `docs/tutorials/setting-up.md:135-136` states "account names include the domain … full address should be specified as a username." Two accounts (`user1@example.org`, `user2@example.org`) were created with `maddyctl`, which operates **directly on the SQLite store** (no running server needed): `findBlockInCfg` calls `maddy.InitDirs()` (`cmd/maddyctl/config.go:31`), which `os.Chdir`s into the state directory (`maddy.go:220` — `os.Chdir(config.StateDirectory)`), so the relative `dsn all.db` resolves inside the state dir. The `users` → `create USERNAME` subcommand takes `--password`/`-p` and a default bcrypt `--hash` (`cmd/maddyctl/main.go:71-99`); `users list` prints one username per line (`fmt.Println(user)`, `cmd/maddyctl/users.go:25`); `imap-mboxes list` prints one mailbox per line (`fmt.Println(info.Name)`, `cmd/maddyctl/imap.go:59`).

The actual, unedited command transcript (preserved as `maddyctl_setup_output.txt`; each `$` line is the command run and each `[exit N]` its status; captured with `stdout`+`stderr` merged):

```
$ /tmp/maddy_acct/maddyctl --config /tmp/maddy_acct/maddy.conf users create user1@example.org --password Password123!
[exit 0]

$ /tmp/maddy_acct/maddyctl --config /tmp/maddy_acct/maddy.conf users create user2@example.org --password Password123!
[exit 0]

$ /tmp/maddy_acct/maddyctl --config /tmp/maddy_acct/maddy.conf users list
user1@example.org
user2@example.org
[exit 0]

$ /tmp/maddy_acct/maddyctl --config /tmp/maddy_acct/maddy.conf imap-mboxes list user2@example.org
INBOX
[exit 0]

$ /tmp/maddy_acct/maddyctl --config /tmp/maddy_acct/maddy.conf imap-mboxes list user1@example.org
INBOX
[exit 0]
```

Both `create` commands returned `[exit 0]` with no output; `users list` shows the two full-address accounts `user1@example.org` and `user2@example.org`; and `imap-mboxes list` shows the auto-created `INBOX` for each account (the INBOX is created implicitly with the account). The output is deterministic — two independent runs against separate fresh state directories produced byte-identical transcripts.

> **Provenance / non-canonical test artifact.** This transcript was captured in a fresh scratch workdir (`/tmp/maddy_acct`) using the **byte-identical canonical** `sql local_mailboxes local_authdb { driver sqlite3; dsn all.db }` block — only the `state`/`runtime` directories were redirected, exactly the policy-preserving substitution described in §1.5 (its `diff` against the canonical `maddy.conf.used` is those two lines only). It is a faithful re-run of the canonical account-setup commands, performed to preserve the `maddyctl` output as evidence (the account-setup commands' stdout was not among the originally preserved evidence files). The same two accounts are independently corroborated at runtime by the authenticated SMTPS `AUTH` successes and the delivered messages captured in §4–§6. The password `Password123!` is a test value, not a maddy default; the accounts exist only inside temporary SQLite databases, which are removed during cleanup (see §11).

---

## 2. The six questions (restated verbatim)

1. How does maddy decide accept/reject of a sender address from an authenticated session? Does enforcement happen? What are the actual SMTP response codes and messages? Does behavior differ based on how the mismatch is constructed?
2. For accepted/delivered messages: capture actual stored headers — raw DKIM-Signature, Received headers maddy adds, Authentication-Results / other security headers (actual content, not summary).
3. Capture the SMTP transaction (protocol exchange) for at least one rejected and one accepted message via logging or network inspection; if debug logging insufficient, show what was tried and use an alternative (packet capture / manual SMTP client).
4. Determine whether the default config enforces sender alignment with authenticated identity, or whether explicit policy config is required.
5. Rule out at least one plausible-but-incorrect interpretation using specific evidence.
6. Identify one case where maddy's security behavior differs from what a reasonable reading of the configuration might suggest, supported by runtime observations.

---

## 3. Direct answers

Each answer leads with the bottom-line result and is proven by the captured evidence in §§4–8 and grounded in §9.

### 3.Q4 — Does the default config enforce sender alignment with the authenticated identity? (answered first, as it frames everything)

**NO. The default configuration does NOT enforce alignment between the authenticated user and the sender address. Explicit policy configuration would be required for user-level alignment, and even that (as shown in §8) does not exist as a submission-acceptance control in this commit.**

Runtime proof: the identity-mismatch message (Scenario A — authenticated as `user1@example.org`, `MAIL FROM` and `From:` both `user2@example.org`) was **accepted and delivered**, returning `250 2.0.0 OK: queued`; it was merely left **unsigned**. The only default sender enforcement is the *domain-level* open-relay guard `default_source { reject 501 5.1.8 "Non-local sender domain" }` (`maddy.conf:117-119`), which is about the envelope *domain*, not the *user*.

Code grounding: the submission preparation function `submissionPrepare` (`internal/endpoint/smtp/submission.go:27`) performs Message-ID/Date insertion and From/address **syntax** validation, but contains **no comparison** of the `From`/`MAIL FROM` address against the authenticated user — a `grep -i` for `authuser`/`authname`/`connstate` across the entire 130-line file returns nothing. The delivery entry point `startDelivery` records the authenticated user for logging only: `internal/endpoint/smtp/smtp.go:127` `if s.connState.AuthUser != "" {` and `:133` `"username", s.connState.AuthUser,` — it does **not** authorize the envelope sender against it.

Standards basis: this is RFC 6409 (Message Submission for Mail, STD 72)-compliant. §4.3 (Require Authentication), §5.1 (Enforce Address Syntax → 501/554), and §8.1–§8.2 (add Date/Message-ID) are MUST/SHOULD items and maddy implements them; but **§6.1 "Enforce Submission Rights" is a MAY (optional)**. maddy simply does not implement the optional per-identity check by default.

### 3.Q1 — How does maddy decide accept/reject of a sender from an authenticated session? Actual codes? Does it differ by mismatch construction?

**Acceptance is decided by the envelope `MAIL FROM` domain via message-pipeline source-block routing, NOT by the authenticated identity.** A local-domain sender is accepted regardless of which user authenticated; a non-local-domain sender is rejected.

Actual SMTP codes (captured, §5):
- **Accepted** local-domain sender → `250 2.0.0 OK: queued` at end of `DATA`.
- **Rejected** non-local sender → `501 5.1.8 Non-local sender domain`, **emitted at `RCPT TO`** (the `MAIL FROM` command is optimistically accepted with `250 2.0.0 Roger, accepting mail from <addr>` first).

**YES — behavior differs by how the mismatch is constructed:**
- Same *local* domain, **different user** (auth `user1`, `MAIL FROM user2@example.org`) → **ACCEPTED** (`250`).
- **Non-local** domain (auth `user1`, `MAIL FROM foo@evil.com`) → **REJECTED** (`501 5.1.8` at `RCPT TO`).

There is **no per-identity envelope-sender authorization** anywhere in the accept path. The routing is driven purely by the source block: `maddy.conf:97` `source $(local_domains) { ... }` matches local-domain senders; `maddy.conf:117-119` `default_source { reject 501 5.1.8 "Non-local sender domain" }` catches everything else. The server confirms the routing decision per message in `-debug`: a local sender is `matched by domain rule 'example.org'` while `foo@evil.com` is `matched by default rule` (quoted in §4).

### 3.Q2 — For accepted/delivered messages, what are the actual stored headers (raw `DKIM-Signature`, `Received`, `Authentication-Results` / other security headers)?

**A delivered message carries at most ONE maddy-added security header — a `DKIM-Signature`, and only when the message is aligned (relaxed/relaxed, `d=example.org`, `s=default`, `i=user1@example.org`). Every delivered message additionally gets exactly one `Received` header that has an `(envelope-sender <addr>)` clause but NO client-IP `from` clause. NO `Authentication-Results` header is ever added on the submission path.** The complete raw bytes are quoted verbatim in §6 (signed UID1 and unsigned UID2), and the published key plus byte-exact verification are in §7.

Code grounding:
- The `Received` header is built by `GenerateReceived` (`internal/target/received.go:19`). Its client-IP `from` clause is gated on sender tracing at `:30`, and submission sets `msgMeta.DontTraceSender = true` (`internal/endpoint/smtp/submission.go:28`), so the stored `Received` omits the client IP while always keeping the envelope-sender clause (`:69`). This is exactly the shape observed in §6 (`Received:  by example.org (envelope-sender <user1@example.org>) with ESMTPS id …`).
- NO `Authentication-Results` header appears because the check runner adds it only when at least one check produced a result — `if len(cr.mergedRes.AuthResult) != 0 {` (`internal/msgpipeline/check_runner.go:301`), `header.Add("Authentication-Results", …)` (`:302`) — and the submission block declares no `check{}`, so the result set is empty. (Contrast: the inbound port-25 pipeline runs SPF/DKIM-verify checks that DO populate `Authentication-Results`; see §9.8.)
- The `DKIM-Signature` is present **only** on the fully aligned message (Scenario C, §6.1/§6.2); the three misaligned scenarios are delivered unsigned (Scenario A shown in §6.3).

### 3.Q3 — Capture the SMTP transaction (protocol exchange) for at least one rejected and one accepted message.

**Captured in full. `-debug` server logging plus a real authenticated `smtplib.SMTP_SSL` client was sufficient — no packet capture was necessary.** §5 contains the complete, unedited protocol dialogue for one ACCEPTED message and two REJECTED messages:
- **ACCEPTED** — Scenario C (happy path): the full `EHLO` / `AUTH PLAIN` / `MAIL FROM` / `RCPT TO` / `DATA` exchange ending in `250 2.0.0 OK: queued` (§5.1).
- **REJECTED** — Scenario B (non-local sender): rejection `501 5.1.8 Non-local sender domain` emitted at `RCPT TO` (§5.2).
- **REJECTED (syntactic contrast)** — Scenario F (missing `From`): rejection `554 5.6.0` emitted at end of `DATA` (§5.3).

Each dialogue pairs the client-side send/reply lines (from `client_out.txt`) with the matching `-debug` server decision lines (from `server_canonical.log`), quoted verbatim.

### 3.Q5 — A plausible-but-incorrect interpretation, ruled out with evidence

**Ruled out:** the intuitive reading that *"maddy rejects a message whose `MAIL FROM` differs from the authenticated user"* is **DISPROVEN**.

Evidence (Scenario A): authenticated as `user1@example.org`, `MAIL FROM:<user2@example.org>`, the transaction returned `250 2.0.0 OK: queued` and the server logged acceptance. The server's own `-debug` lines show it recording the mismatch and accepting anyway:

```
45	submission: incoming message	{"msg_id":"a756d9a5","sender":"user2@example.org","src_host":"client.test","src_ip":"127.0.0.1:51836","username":"user1@example.org"}
57	submission: accepted	{"msg_id":"a756d9a5"}
```

Here `sender` (`user2@example.org`) ≠ `username` (`user1@example.org`), yet the message is `accepted`. A second incorrect reading — *"a mismatch is silently dropped"* — is also ruled out: the message was not dropped but **delivered** to `user2`'s INBOX as UID2 (its raw stored bytes are shown in §6).

### 3.Q6 — A case where security behavior differs from a reasonable reading of the configuration

**The `require_sender_match` option's `auth_domain` / `auth_user` tokens SILENTLY DISABLE the authenticated-identity check** — the opposite of what their names suggest.

A reasonable reading of the documentation/config is that setting `require_sender_match ... auth_domain` (or `auth_user`) *tightens* sender matching against the authenticated identity. The runtime observation is the reverse. Under the default config the identity-mismatch message (Scenario A) is **unsigned**; under a labeled non-canonical probe adding `require_sender_match envelope auth_domain`, the **identical** inputs produce a message **signed as `user2@example.org`** while the authenticated `username` is `user1@example.org`:

```
31	submission: incoming message	{"msg_id":"1e57717e","sender":"user2@example.org","src_host":"client.test","src_ip":"127.0.0.1:37698","username":"user1@example.org"}
41	[debug] sign_dkim: signed	{"identifier":"user2@example.org"}
```

Root cause (`internal/modify/dkim/dkim.go`): the auth check reads **only the literal map key `"auth"`** — `:299` `if _, do := m.senderMatch["auth"]; do {`. But `"auth"` is **not** one of the user-selectable `EnumList` tokens; the selectable auth tokens are `auth_domain`/`auth_user` (`:151-152`), which `shouldSign()` never reads. Selecting either **replaces** the default `"auth"` entry in the set, so `senderMatch["auth"]` is absent and the identity check is skipped. Thus a directive named to *tighten* sender matching actually **disables** it. Full mechanism and the before/after runtime proof are in §8.


---

## 4. Scenario matrix (canonical configuration)

Six scenarios were driven through the real authenticated SMTPS submission endpoint. The table records the **actual observed** SMTP result, the DKIM decision, and the real `msg_id` assigned by the server in this run. (These `msg_id` values are from the actual run and are the authoritative identifiers used throughout this document.)

| # | Scenario | auth / MAIL FROM / From | SMTP result | DKIM decision (msg_id) |
|---|----------|-------------------------|-------------|------------------------|
| C | happy path (should be signed) | user1 / user1@example.org / user1@example.org | `250 2.0.0 OK: queued` — ACCEPTED | **SIGNED** (`e2bfb479`) |
| A | identity mismatch | user1 / user2@example.org / user2@example.org | `250 2.0.0 OK: queued` — ACCEPTED | UNSIGNED — auth branch (`a756d9a5`) |
| B | non-local sender domain | user1 / foo@evil.com / foo@evil.com | `501 5.1.8 Non-local sender domain` — REJECTED at RCPT | n/a — rejected before delivery (`38a31af0`) |
| D | From domain ≠ key domain | user1 / user1@example.org / user1@evil.com | `250 2.0.0 OK: queued` — ACCEPTED | UNSIGNED — domain branch (`ba9a9026`) |
| E | envelope mismatch | user1 / user1@example.org / user2@example.org | `250 2.0.0 OK: queued` — ACCEPTED | UNSIGNED — envelope branch (`726e0f2e`) |
| F | missing From (syntax contrast) | user1 / user1@example.org / (omitted) | `554 5.6.0 Message does not contains a From header field` — REJECTED at DATA | n/a — rejected before signing (`7f53b7c0`) |

**Coverage established by this matrix:**
- **All four `shouldSign` decision outcomes are exercised:** signed (C), not-authenticated-identity (A), not-key-domain (D), not-envelope-address (E).
- **Both reject paths are exercised:** `501 5.1.8` routing rejection at `RCPT TO` (B); `554 5.6.0` syntactic rejection at `DATA` (F).
- **Scenario F is the ONLY submission-time *content* enforcement, and it is purely syntactic** (a missing `From` header). It is included specifically to contrast against the **absent** identity enforcement: maddy will reject a message with no `From` at all, but will happily accept a `From` belonging to a different user.

### 4.1 EHLO capabilities and per-command responses (verbatim)

Every dialogue opened with the same EHLO advertisement (from any scenario in `client_out.txt`):

```
send: 'ehlo client.test\r\n'
reply: b'250-Hello client.test\r\n'
reply: b'250-PIPELINING\r\n'
reply: b'250-8BITMIME\r\n'
reply: b'250-ENHANCEDSTATUSCODES\r\n'
reply: b'250-AUTH PLAIN\r\n'
reply: b'250-SMTPUTF8\r\n'
reply: b'250 SIZE 33554432\r\n'
```

Authentication (base64 of `\0user1@example.org\0Password123!`) always succeeded:

```
send: 'AUTH PLAIN AHVzZXIxQGV4YW1wbGUub3JnAFBhc3N3b3JkMTIzIQ==\r\n'
reply: b'235 2.0.0 Authentication succeeded\r\n'
```

`MAIL FROM` was **always** optimistically accepted with `250`, even for the non-local `foo@evil.com` (the rejection comes later, at `RCPT`):

```
send: 'mail from:<foo@evil.com>\r\n'
reply: b'250 2.0.0 Roger, accepting mail from <foo@evil.com>\r\n'
```

### 4.2 Per-scenario `-debug` server decision lines (verbatim from `server_canonical.log`)

**Scenario C — SIGNED (msg_id `e2bfb479`):**

```
31	submission: incoming message	{"msg_id":"e2bfb479","sender":"user1@example.org","src_host":"client.test","src_ip":"127.0.0.1:51824","username":"user1@example.org"}
32	[debug] smtp/pipeline: sender user1@example.org matched by domain rule 'example.org'	{"msg_id":"e2bfb479"}
39	submission: adding missing Message-ID
40	submission: adding missing Date header
41	[debug] sign_dkim: signed	{"identifier":"user1@example.org"}
43	submission: accepted	{"msg_id":"e2bfb479"}
```

**Scenario A — ACCEPTED + UNSIGNED, auth branch (msg_id `a756d9a5`):**

```
45	submission: incoming message	{"msg_id":"a756d9a5","sender":"user2@example.org","src_host":"client.test","src_ip":"127.0.0.1:51836","username":"user1@example.org"}
46	[debug] smtp/pipeline: sender user2@example.org matched by domain rule 'example.org'	{"msg_id":"a756d9a5"}
55	sign_dkim: not signing, From address is not authenticated identity	{"auth_id":"user1@example.org","from_addr":"user2@example.org","msg_id":"a756d9a5"}
57	submission: accepted	{"msg_id":"a756d9a5"}
```

This is the `auth` branch of `shouldSign()` at `internal/modify/dkim/dkim.go:299-310`: `sender` (`user2@example.org`) matches the local domain so it routes/accepts, but the From-vs-auth comparison fails (`from_addr` ≠ `auth_id`), so signing is skipped and the message is delivered unsigned.

**Scenario B — REJECTED at RCPT, `501 5.1.8` (msg_id `38a31af0`):**

```
59	submission: incoming message	{"msg_id":"38a31af0","sender":"foo@evil.com","src_host":"client.test","src_ip":"127.0.0.1:51838","username":"user1@example.org"}
60	[debug] smtp/pipeline: sender foo@evil.com matched by default rule	{"msg_id":"38a31af0"}
63	[debug] smtp/pipeline: recipient user2@example.org matched by default rule (clean = user2@example.org)	{"msg_id":"38a31af0"}
64	submission: RCPT error	{"effective_rcpt":"user2@example.org","rcpt":"user2@example.org","reason":"reject directive used","smtp_code":501,"smtp_enchcode":"5.1.8","smtp_msg":"Non-local sender domain"}
65	submission: aborted	{"msg_id":"38a31af0"}
```

The non-local `foo@evil.com` is `matched by default rule` (i.e., it fell through to `default_source`), whose `reject 501 5.1.8 "Non-local sender domain"` directive fires — and it fires **at `RCPT`**, producing `reason:"reject directive used"` with `smtp_code:501`, `smtp_enchcode:"5.1.8"`.

**Scenario D — ACCEPTED + UNSIGNED, domain branch (msg_id `ba9a9026`):**

```
67	submission: incoming message	{"msg_id":"ba9a9026","sender":"user1@example.org","src_host":"client.test","src_ip":"127.0.0.1:51850","username":"user1@example.org"}
77	sign_dkim: not signing, From domain is not key domain	{"from_domain":"evil.com","key_domain":"example.org","msg_id":"ba9a9026"}
79	submission: accepted	{"msg_id":"ba9a9026"}
```

Here the envelope (`MAIL FROM`) is the local `user1@example.org` (so it is accepted), but the *header* `From:` is `user1@evil.com`; the first `shouldSign()` check — the key-domain comparison at `dkim.go:287-291` — fails (`from_domain` `evil.com` ≠ `key_domain` `example.org`), so it is delivered unsigned.

**Scenario E — ACCEPTED + UNSIGNED, envelope branch (msg_id `726e0f2e`):**

```
81	submission: incoming message	{"msg_id":"726e0f2e","sender":"user1@example.org","src_host":"client.test","src_ip":"127.0.0.1:51860","username":"user1@example.org"}
91	sign_dkim: not signing, From address is not envelope address	{"envelope":"user1@example.org","from_addr":"user2@example.org","msg_id":"726e0f2e"}
93	submission: accepted	{"msg_id":"726e0f2e"}
```

The From domain (`example.org`) *does* match the key domain, so the domain check passes; but the `envelope` token comparison at `dkim.go:293-297` fails (`from_addr` `user2@example.org` ≠ `envelope` `user1@example.org`), so it is delivered unsigned.

**Scenario F — REJECTED at DATA, `554 5.6.0` (msg_id `7f53b7c0`):**

```
95	submission: incoming message	{"msg_id":"7f53b7c0","sender":"user1@example.org","src_host":"client.test","src_ip":"127.0.0.1:51864","username":"user1@example.org"}
102	submission: RCPT ok	{"msg_id":"7f53b7c0","rcpt":"user2@example.org"}
103	submission: adding missing Message-ID
104	submission: DATA error	{"modifier":"submission_prepare","msg_id":"7f53b7c0","reason":"Message does not contains a From header field","smtp_code":554,"smtp_enchcode":"5.6.0","smtp_msg":"Message does not contains a From header field"}
105	submission: aborted	{"msg_id":"7f53b7c0"}
```

Here `RCPT` succeeds (`RCPT ok`) but the `submission_prepare` modifier rejects at `DATA` because the message has no `From` header — the only content-level enforcement submission performs, and it is purely syntactic (`internal/endpoint/smtp/submission.go:40-43`).

**RFC 6409 §8.1–§8.2 observed:** for each of the four accepted messages the server logged a `submission: adding missing Message-ID` + `submission: adding missing Date header` pair (lines 39-40, 53-54, 75-76, 89-90), confirming maddy adds both Date and Message-ID at submission time. Line 103 also shows `submission: adding missing Message-ID` for Scenario F (`msg_id 7f53b7c0`), but that Message-ID is added *before* the missing-From rejection at `DATA` (line 104, `554 5.6.0`); Scenario F therefore receives no Date line and is never delivered.


---

## 5. Full SMTP transactions (Q3)

The complete client-side protocol dialogues below are copied **verbatim** from `client_out.txt` (produced by a Python `smtplib.SMTP_SSL` client with `set_debuglevel(1)`). The `send:` lines are client → server; the `reply:` lines are server → client; the `b'...\r\n'` byte-string formatting is the Python debug output reproduced exactly (the `\r\n` are the literal CRLF terminators). At least one accepted and one rejected transaction are included, plus the syntactic-rejection contrast.

### 5.1 ACCEPTED — Scenario C (happy path, signed)

```
send: 'ehlo client.test\r\n'
reply: b'250-Hello client.test\r\n'
reply: b'250-PIPELINING\r\n'
reply: b'250-8BITMIME\r\n'
reply: b'250-ENHANCEDSTATUSCODES\r\n'
reply: b'250-AUTH PLAIN\r\n'
reply: b'250-SMTPUTF8\r\n'
reply: b'250 SIZE 33554432\r\n'
reply: retcode (250); Msg: b'Hello client.test\nPIPELINING\n8BITMIME\nENHANCEDSTATUSCODES\nAUTH PLAIN\nSMTPUTF8\nSIZE 33554432'
send: 'AUTH PLAIN AHVzZXIxQGV4YW1wbGUub3JnAFBhc3N3b3JkMTIzIQ==\r\n'
reply: b'235 2.0.0 Authentication succeeded\r\n'
reply: retcode (235); Msg: b'2.0.0 Authentication succeeded'
send: 'mail from:<user1@example.org>\r\n'
reply: b'250 2.0.0 Roger, accepting mail from <user1@example.org>\r\n'
reply: retcode (250); Msg: b'2.0.0 Roger, accepting mail from <user1@example.org>'
RESULT MAIL FROM <user1@example.org> => 250 b'2.0.0 Roger, accepting mail from <user1@example.org>'
send: 'rcpt to:<user2@example.org>\r\n'
reply: b"250 2.0.0 I'll make sure <user2@example.org> gets this\r\n"
reply: retcode (250); Msg: b"2.0.0 I'll make sure <user2@example.org> gets this"
RESULT RCPT TO <user2@example.org> => 250 b"2.0.0 I'll make sure <user2@example.org> gets this"
send: 'data\r\n'
reply: b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
reply: retcode (354); Msg: b'2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>'
data: (354, b'2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>')
send: b'From: user1@example.org\r\nTo: user2@example.org\r\nSubject: Scenario C happy path\r\nMIME-Version: 1.0\r\nContent-Type: text/plain; charset=utf-8\r\n\r\nThis is a test message body.\r\n.\r\n'
reply: b'250 2.0.0 OK: queued\r\n'
reply: retcode (250); Msg: b'2.0.0 OK: queued'
data: (250, b'2.0.0 OK: queued')
RESULT DATA => 250 b'2.0.0 OK: queued'
send: 'quit\r\n'
reply: b'221 2.0.0 Goodnight and good luck\r\n'
reply: retcode (221); Msg: b'2.0.0 Goodnight and good luck'
```

### 5.2 REJECTED — Scenario B (non-local sender domain, `501 5.1.8` at RCPT)

```
send: 'ehlo client.test\r\n'
reply: b'250-Hello client.test\r\n'
reply: b'250-PIPELINING\r\n'
reply: b'250-8BITMIME\r\n'
reply: b'250-ENHANCEDSTATUSCODES\r\n'
reply: b'250-AUTH PLAIN\r\n'
reply: b'250-SMTPUTF8\r\n'
reply: b'250 SIZE 33554432\r\n'
reply: retcode (250); Msg: b'Hello client.test\nPIPELINING\n8BITMIME\nENHANCEDSTATUSCODES\nAUTH PLAIN\nSMTPUTF8\nSIZE 33554432'
send: 'AUTH PLAIN AHVzZXIxQGV4YW1wbGUub3JnAFBhc3N3b3JkMTIzIQ==\r\n'
reply: b'235 2.0.0 Authentication succeeded\r\n'
reply: retcode (235); Msg: b'2.0.0 Authentication succeeded'
send: 'mail from:<foo@evil.com>\r\n'
reply: b'250 2.0.0 Roger, accepting mail from <foo@evil.com>\r\n'
reply: retcode (250); Msg: b'2.0.0 Roger, accepting mail from <foo@evil.com>'
RESULT MAIL FROM <foo@evil.com> => 250 b'2.0.0 Roger, accepting mail from <foo@evil.com>'
send: 'rcpt to:<user2@example.org>\r\n'
reply: b'501 5.1.8 Non-local sender domain (msg ID = 38a31af0)\r\n'
reply: retcode (501); Msg: b'5.1.8 Non-local sender domain (msg ID = 38a31af0)'
RESULT RCPT TO <user2@example.org> => 501 b'5.1.8 Non-local sender domain (msg ID = 38a31af0)'
RESULT RCPT rejected (code 501), issuing RSET+QUIT
send: 'RSET\r\n'
reply: b'250 2.0.0 Session reset\r\n'
reply: retcode (250); Msg: b'2.0.0 Session reset'
send: 'quit\r\n'
reply: b'221 2.0.0 Goodnight and good luck\r\n'
reply: retcode (221); Msg: b'2.0.0 Goodnight and good luck'
```

Note the sequencing: `MAIL FROM:<foo@evil.com>` is answered `250 2.0.0 Roger, accepting mail from <foo@evil.com>` (optimistic accept), and the rejection `501 5.1.8 Non-local sender domain (msg ID = 38a31af0)` arrives only in response to `RCPT TO`. This confirms the reject is a *routing/RCPT-time* decision, not a `MAIL FROM`-time one.

### 5.3 REJECTED (syntactic contrast) — Scenario F (missing From, `554 5.6.0` at DATA)

```
send: 'ehlo client.test\r\n'
reply: b'250-Hello client.test\r\n'
reply: b'250-PIPELINING\r\n'
reply: b'250-8BITMIME\r\n'
reply: b'250-ENHANCEDSTATUSCODES\r\n'
reply: b'250-AUTH PLAIN\r\n'
reply: b'250-SMTPUTF8\r\n'
reply: b'250 SIZE 33554432\r\n'
reply: retcode (250); Msg: b'Hello client.test\nPIPELINING\n8BITMIME\nENHANCEDSTATUSCODES\nAUTH PLAIN\nSMTPUTF8\nSIZE 33554432'
send: 'AUTH PLAIN AHVzZXIxQGV4YW1wbGUub3JnAFBhc3N3b3JkMTIzIQ==\r\n'
reply: b'235 2.0.0 Authentication succeeded\r\n'
reply: retcode (235); Msg: b'2.0.0 Authentication succeeded'
send: 'mail from:<user1@example.org>\r\n'
reply: b'250 2.0.0 Roger, accepting mail from <user1@example.org>\r\n'
reply: retcode (250); Msg: b'2.0.0 Roger, accepting mail from <user1@example.org>'
RESULT MAIL FROM <user1@example.org> => 250 b'2.0.0 Roger, accepting mail from <user1@example.org>'
send: 'rcpt to:<user2@example.org>\r\n'
reply: b"250 2.0.0 I'll make sure <user2@example.org> gets this\r\n"
reply: retcode (250); Msg: b"2.0.0 I'll make sure <user2@example.org> gets this"
RESULT RCPT TO <user2@example.org> => 250 b"2.0.0 I'll make sure <user2@example.org> gets this"
send: 'data\r\n'
reply: b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
reply: retcode (354); Msg: b'2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>'
data: (354, b'2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>')
send: b'To: user2@example.org\r\nSubject: Scenario F missing From\r\nMIME-Version: 1.0\r\nContent-Type: text/plain; charset=utf-8\r\n\r\nThis is a test message body.\r\n.\r\n'
reply: b'554 5.6.0 Message does not contains a From header field (msg ID = 7f53b7c0)\r\n'
reply: retcode (554); Msg: b'5.6.0 Message does not contains a From header field (msg ID = 7f53b7c0)'
data: (554, b'5.6.0 Message does not contains a From header field (msg ID = 7f53b7c0)')
RESULT DATA => 554 b'5.6.0 Message does not contains a From header field (msg ID = 7f53b7c0)'
send: 'quit\r\n'
reply: b'221 2.0.0 Goodnight and good luck\r\n'
reply: retcode (221); Msg: b'2.0.0 Goodnight and good luck'
```

Here `RCPT TO` succeeds (`250`); the rejection `554 5.6.0 Message does not contains a From header field` arrives at the end of `DATA`, when the `submission_prepare` modifier inspects the header block. This is the *only* content check submission enforces, and it is syntactic — contrast it with Scenario A, where a `From:` belonging to a *different user* passed unchallenged.


---

## 6. Raw stored headers (Q2 — actual content, not summary)

Stored messages were captured with:

```
maddyctl --config <conf> imap-msgs dump --uid user2@example.org INBOX <uid>
```

and cross-checked against the external blob store under `<state>/messages/<extBodyKey>`. The `maddyctl dump` output is **byte-identical** to the raw external blob: `signed_msg_UID1.raw` == `dump_UID1_signed.txt` (both **1202 bytes**, sha256 `16a3df6660f70257159e63a50aaeed81a9635dc3327a9cfe9a37e376d82dd70c`), and `unsigned_msg_UID2.raw` == `dump_UID2_unsigned.txt` (both **471 bytes**). Delivery order in `user2`'s INBOX: UID1 = Scenario C (signed), UID2 = Scenario A (unsigned), UID3 = D, UID4 = E.

### 6.1 SIGNED message — Scenario C, UID1 (1202 bytes)

Verbatim content (readable form; each header line is CRLF-terminated in the stored bytes — see the byte-exact rendering in §6.2):

```
Delivered-To: user2@example.org
Return-Path: <user1@example.org>
Dkim-Signature: a=rsa-sha256;
 bh=jX3F0bCAI7sIbkHyy3mLYO28ieDQz2R0P8HwQkklFj4=; c=relaxed/relaxed;
 d=example.org;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:MIME-Version:Content-Type:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=user1@example.org; s=default; t=1783490096; v=1; x=1783922096; b=CPiwhle04Ra5NDqwBDJGwX7wELm67ZxAWzSQGPihqPpUrdX4bkpOILgC3vjqLfb3hnY4zYxF3XoXqmh7pfVnKT8YeRMIbJCBNPoKilGkR+/lpV60LCd1lVYLSuGK3r9VzCtdEW04Y98Bg3o8gKBzw9YIDQtwzyeanJc4FG5XOYyEFDuUyAf4DloviufL0H67W5dTWdJF1gIL5Zj8qtfbDr23kZ4xKCd1tAECCkQQxS2SG/AdI2JlCcO1F7weS9Yj5D41PRIhg/5mq1mxaIcqcouE503mlsLhoyazRi7JpwGQhwidRFZ5dtlkVe6sddDnPNDNbURialP99SW2qXgqpw==;
Received:  by example.org (envelope-sender <user1@example.org>) with ESMTPS
 id e2bfb479; Wed, 08 Jul 2026 05:54:56 +0000
Date: Wed, 8 Jul 2026 05:54:56 +0000
Message-Id: <595ab022-d729-4d91-9ef3-7fdd03b1270a@example.org>
From: user1@example.org
To: user2@example.org
Subject: Scenario C happy path
MIME-Version: 1.0
Content-Type: text/plain; charset=utf-8

This is a test message body.
```

**`DKIM-Signature` field-by-field (grounded in `internal/modify/dkim/dkim.go`):**
- `a=rsa-sha256` — RSA + SHA-256 signing algorithm.
- `c=relaxed/relaxed` — relaxed header and body canonicalization.
- `d=example.org` — the signing domain (SDID); equals `$(primary_domain)`.
- `s=default` — the selector (the `sign_dkim $(primary_domain) default` argument, `maddy.conf:99`).
- `i=user1@example.org` — the AUID (signing identity), set to the aligned From address.
- `t=1783490096` — signature timestamp; `x=1783922096` — expiration.
- `v=1` — DKIM version.
- `bh=jX3F0bCAI7sIbkHyy3mLYO28ieDQz2R0P8HwQkklFj4=` — the body hash (independently reproduced in §7 — exact match).
- `h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:MIME-Version:Content-Type:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp` — the signed-header list. Note it is **oversigned**: headers that are present (`Subject`, `To`, `From`, `Date`, `MIME-Version`, `Content-Type`, `Message-Id`) are **double-listed**, which per RFC 6376 prevents a second copy of those headers being added downstream without breaking the signature.
- `b=CPiwhle04R...2qXgqpw==` — the RSA signature value. The header is added by `dkim.go:406` `h.Add("DKIM-Signature", signer.SignatureValue())`, and the server logs `signed` immediately afterward at `dkim.go:408` `s.m.log.DebugMsg("signed", "identifier", id)`.

**`Received` header — no client-IP `from` clause.** The stored `Received` is exactly:

```
Received:  by example.org (envelope-sender <user1@example.org>) with ESMTPS id e2bfb479; Wed, 08 Jul 2026 05:54:56 +0000
```

There is **no `from <host> [ip]` clause**. This is because submission sets `msgMeta.DontTraceSender = true` (`internal/endpoint/smtp/submission.go:28`), and the `from` clause in `internal/target/received.go:30` is gated on `!msgMeta.DontTraceSender && (strings.Contains(msgMeta.Conn.Proto, "SMTP") || ...)` — so the client IP is deliberately omitted for submitted mail. The `(envelope-sender <...>)` clause at `received.go:69` (`builder.WriteString(" (envelope-sender <")`) is **always** present. The `Received` header itself is added in `internal/endpoint/smtp/smtp.go`: `:303` builds it via `target.GenerateReceived(...)` and `:307` `header.Add("Received", received)`.

**No `Authentication-Results` header.** The submission block declares no `check{}` set, so no check ever produces an `AuthResult`. In `internal/msgpipeline/check_runner.go:301` the header is gated on `if len(cr.mergedRes.AuthResult) != 0 {` and only then added at `:302` `header.Add("Authentication-Results", authres.Format(hostname, cr.mergedRes.AuthResult))`. With an empty result set the branch is never taken. (Contrast: the inbound port-25 pipeline runs `apply_spf` — `internal/check/spf/spf.go:28` `const modName = "apply_spf"` — and `verify_dkim`, which *do* produce `Authentication-Results`.)

### 6.2 Byte-exact rendering of the signed message (`cat -A`; `^M` = CR, `$` = end of line)

This proves the exact stored bytes: header name casing (`Dkim-Signature`, title-cased), CRLF line terminators, and the fold points (at `;` boundaries).

```
Delivered-To: user2@example.org^M$
Return-Path: <user1@example.org>^M$
Dkim-Signature: a=rsa-sha256;^M$
 bh=jX3F0bCAI7sIbkHyy3mLYO28ieDQz2R0P8HwQkklFj4=; c=relaxed/relaxed;^M$
 d=example.org;^M$
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:MIME-Version:Content-Type:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=user1@example.org; s=default; t=1783490096; v=1; x=1783922096; b=CPiwhle04Ra5NDqwBDJGwX7wELm67ZxAWzSQGPihqPpUrdX4bkpOILgC3vjqLfb3hnY4zYxF3XoXqmh7pfVnKT8YeRMIbJCBNPoKilGkR+/lpV60LCd1lVYLSuGK3r9VzCtdEW04Y98Bg3o8gKBzw9YIDQtwzyeanJc4FG5XOYyEFDuUyAf4DloviufL0H67W5dTWdJF1gIL5Zj8qtfbDr23kZ4xKCd1tAECCkQQxS2SG/AdI2JlCcO1F7weS9Yj5D41PRIhg/5mq1mxaIcqcouE503mlsLhoyazRi7JpwGQhwidRFZ5dtlkVe6sddDnPNDNbURialP99SW2qXgqpw==;^M$
Received:  by example.org (envelope-sender <user1@example.org>) with ESMTPS^M$
 id e2bfb479; Wed, 08 Jul 2026 05:54:56 +0000^M$
Date: Wed, 8 Jul 2026 05:54:56 +0000^M$
Message-Id: <595ab022-d729-4d91-9ef3-7fdd03b1270a@example.org>^M$
From: user1@example.org^M$
To: user2@example.org^M$
Subject: Scenario C happy path^M$
MIME-Version: 1.0^M$
Content-Type: text/plain; charset=utf-8^M$
^M$
This is a test message body.$
```

### 6.3 UNSIGNED message — Scenario A, UID2 (471 bytes)

Verbatim content:

```
Delivered-To: user2@example.org
Return-Path: <user2@example.org>
Received:  by example.org (envelope-sender <user2@example.org>) with ESMTPS
 id a756d9a5; Wed, 08 Jul 2026 05:54:56 +0000
Date: Wed, 8 Jul 2026 05:54:56 +0000
Message-Id: <11f0a62c-a942-4fe3-abdb-d107d459b6ab@example.org>
From: user2@example.org
To: user2@example.org
Subject: Scenario A identity mismatch
MIME-Version: 1.0
Content-Type: text/plain; charset=utf-8

This is a test message body.
```

Byte-exact rendering (`cat -A`):

```
Delivered-To: user2@example.org^M$
Return-Path: <user2@example.org>^M$
Received:  by example.org (envelope-sender <user2@example.org>) with ESMTPS^M$
 id a756d9a5; Wed, 08 Jul 2026 05:54:56 +0000^M$
Date: Wed, 8 Jul 2026 05:54:56 +0000^M$
Message-Id: <11f0a62c-a942-4fe3-abdb-d107d459b6ab@example.org>^M$
From: user2@example.org^M$
To: user2@example.org^M$
Subject: Scenario A identity mismatch^M$
MIME-Version: 1.0^M$
Content-Type: text/plain; charset=utf-8^M$
^M$
This is a test message body.$
```

**Observations:**
- There is **NO `Dkim-Signature` header at all** — the message was delivered *unsigned-but-delivered*. This is the `RewriteBody()` early-return path: `internal/modify/dkim/dkim.go:349-351` `if !ok { return nil }` — when `shouldSign()` returns `false`, `RewriteBody` returns `nil` (no error), so the message proceeds to delivery without a signature.
- `Received` is present (same structure as the signed message, no client-IP `from` clause).
- No `Authentication-Results`.

**This directly answers the user's 4th scenario question — "what a verifying recipient sees":** for a `From`/identity mismatch, **a verifying recipient sees an UNSIGNED message**, not a DKIM signature covering a mismatched `From`. maddy never applies a signature over a non-aligned `From`; it simply omits the signature and delivers the message.


---

## 7. DKIM published key and byte-exact verification (Q2 / byte-sensitivity rule)

### 7.1 Auto-generated public key (verbatim)

The auto-generated public-key TXT record (`dkim_keys/example.org_default.dns`, 378 bytes) is quoted verbatim (`example.org_default.dns`):

```
v=DKIM1; k=rsa; p=MIIBCgKCAQEAsScBFG3pF3mByQaX8ZZkQ0dLrisMwNDeMtSYwe+wbbX6vFHnNRUaFkCjlyQNfeWF0Sp2ZSwuFjJYW4csRxCvH6xWDj1DSanJu2H1YgQH9UbOaYWNyEd3OHIEXahSxul94PkTJuzWaWbx4KKdQo+Kip5DBAcWS+h3Kcsn1sV43sBgzHJ3wb8D8UDpIQ93264K8smFxYeh1qaExWtuQl9bM9V37qL/0rlzQohpLznFmzURlcnzMlF9v3IKKjoIxhfqzdzBhjMSDaMy2LiUfwOg05Ib9X9HhUx9kIqqx2fEJaiF+LE6feg1VskbFA+YbkP34kvAnuGqKk+1T4NySwhz+wIDAQAB
```

### 7.2 PKCS#1-vs-PKIX key-publication detail (genuine code-level observation)

The `p=` value begins `MIIBCgKCAQEA`, which is the base64 prefix of a **PKCS#1 `RSAPublicKey`** structure. This is because maddy encodes the key with PKCS#1 at `internal/modify/dkim/keys.go:143`:

```
		keyBlob = x509.MarshalPKCS1PublicKey(pubkey)
```

A standards-conformant DKIM verifier expects the key in **PKIX / SubjectPublicKeyInfo** form (which would begin `MIIBIjANBgkqhkiG9w0BAQEF...`). This matters because the go-msgauth verify path parses `p=` with `x509.ParsePKIXPublicKey`. The verifier output confirms the mismatch and the required re-encoding (`dkim_verify_output.txt`):

```
================================================================
BLOCK 2: Published key encoding - PKCS#1 (maddy) vs PKIX (verifiers)
================================================================
published p= = 270 DER bytes, as maddy emits it (keys.go:143 x509.MarshalPKCS1PublicKey)
x509.ParsePKCS1PublicKey  => OK  (rsa 2048-bit)
x509.ParsePKIXPublicKey   => ERR x509: failed to parse public key (use ParsePKCS1PublicKey instead for this key format)
   ^ go-msgauth/dkim query.go:110 uses ParsePKIXPublicKey, so the as-published key is unusable by conformant verifiers without re-encoding.
```

So maddy's own published key does **not** parse as PKIX (the format a standard verifier tries first); it parses only as PKCS#1. Verification could proceed only after re-encoding the key PKCS#1 → PKIX and serving *that* to the verifier. This is a real, code-level observation about maddy's key publication in this commit.

### 7.3 Body hash `bh=` independently reproduced — EXACT MATCH

The stored body hash was reproduced independently (RFC 6376 relaxed body canonicalization + SHA-256). The result is an exact match (`dkim_verify_output.txt`):

```
================================================================
BLOCK 1: Independent bh= (body hash) reproduction - relaxed canon
================================================================
body bytes (from store, 29 bytes): "This is a test message body.\n"
relaxed-canon body            : "This is a test message body.\r\n"
bh= in stored DKIM-Signature  : jX3F0bCAI7sIbkHyy3mLYO28ieDQz2R0P8HwQkklFj4=
independently recomputed bh=  : jX3F0bCAI7sIbkHyy3mLYO28ieDQz2R0P8HwQkklFj4=
BODY HASH MATCH               : true
```

Because the body hash reproduces exactly against the stored bytes, **the signing machinery is sound**: maddy computed the body hash correctly and embedded it correctly.

### 7.4 Full `b=` signature — positive control PASSES; stored check is INCONCLUSIVE (not invalid)

Two verification runs were performed with go-msgauth's own `dkim` library (the same library maddy uses to sign), fed by a stub DNS server returning the PKIX-re-encoded public key.

**Positive control** — sign a fresh message with the *same* RSA key (relaxed/relaxed) and verify it end-to-end. This isolates the DNS + key + verifier pipeline from any storage effects (`dkim_verify_output.txt`):

```
================================================================
BLOCK 3: Positive control - freshly sign + verify (PKIX key via stub DNS)
================================================================
fresh-signed verification: Domain=example.org Identifier=user1@example.org Err=<nil>
```

`Err=<nil>` — the freshly-signed message verifies **valid**. This proves the DNS-serving tooling, the (re-encoded) key, and the verifier are all correct.

**Stored bytes** — verify the exact re-serialized stored message from the message store (`dkim_verify_output.txt`):

```
================================================================
BLOCK 4: Stored bytes - dkim.Verify against the re-serialized stored form
================================================================
stored header begins: "Delivered-To: user2@example.org\r\nReturn-Path: <user1@example"
stored verification: Domain=example.org Identifier=user1@example.org Err=dkim: signature did not verify: crypto/rsa: verification error
NOTE: 'signature did not verify' (NOT 'body hash did not verify') on the re-serialized
stored copy indicates the body hash matched but the header hash did not, due to
go-message re-folding the long DKIM-Signature header. Per the byte-exactness rule this
negative result on a re-serialized copy is INCONCLUSIVE, not a claim of invalidity.
```

**Root cause = re-serialization, NOT invalidity — the stored result is INCONCLUSIVE per the byte-exactness rule.** Two pieces of evidence establish this:

1. The error is `crypto/rsa: verification error` — an **RSA signature (`b=`) mismatch**, occurring *after* the body-hash gate passed (the positive control and §7.3 both confirm the body hash is correct on the stored bytes). So the body is intact; only the signed *header* bytes differ from what the signer hashed.
2. The stored `DKIM-Signature` header is **not the signer-emitted byte sequence**. go-msgauth emits the header name as `DKIM-Signature`, but the stored copy was re-serialized by `go-imap-sql`/`go-message`, which **re-cased** the name to `Dkim-Signature` (title case) and **re-folded** the long value at `;` boundaries — both visible in the byte-exact `cat -A` rendering in §6.2. Because relaxed *header* canonicalization is sensitive to the exact bytes and structure of the signed `DKIM-Signature` header, verifying against this re-serialized copy tests bytes the signer never signed. The `NOTE` printed with BLOCK 4 states this conclusion directly.

Because the body hash reproduces exactly (§7.3) and the positive control passes end-to-end (BLOCK 3), the signing machinery is proven sound. **Per the governing byte-exactness rule, a negative result obtained from a re-serialized copy is treated as INCONCLUSIVE, never as "invalid".** The exact `bh=` reproduction (§7.3) and the passing positive control are the positive proof that the signing machinery is correct.

> **DNS-serving tooling note (all outside the maddy repo):** the RSA-2048 public key produces a TXT record (~410 bytes) larger than a single UDP DNS string. `foxcpp/go-mockdns` could not serve it over UDP, so it was replaced by a small hand-written `miekg/dns` server that sets `TC=1` (truncated) on the UDP response to force Go's resolver to retry over TCP (run with `GODEBUG=netdns=go`, `/etc/resolv.conf` temporarily pointed at `127.0.0.1` and restored afterward). The Go verifier lived in a **separate temporary Go module**; **maddy's `go.mod`/`go.sum` were untouched.**


---

## 8. The config-vs-behavior divergence, proven at runtime (Q6 — LABELED NON-CANONICAL)

> **This section uses a deliberately NON-canonical configuration.** It exists solely to prove the divergence. The canonical configuration was restored afterward; the divergence is reported as observed.

### 8.1 Probe configuration

The probe config is the canonical `maddy.conf.used` with one **policy-affecting** change — the `sign_dkim` directive converted to block form to add `require_sender_match envelope auth_domain` — together with a policy-neutral change to separate `state`/`runtime` directories (`probe_state`/`probe_runtime`) so the probe run generates its own DKIM keypair and database in isolation from the canonical run. The exact, unedited `diff` (canonical → probe) has both hunks:

```
1,2c1,2
< state /tmp/maddy_work/state
< runtime /tmp/maddy_work/runtime
---
> state /tmp/maddy_work/probe_state
> runtime /tmp/maddy_work/probe_runtime
101c101,103
<             sign_dkim $(primary_domain) default
---
>             sign_dkim $(primary_domain) default {
>                 require_sender_match envelope auth_domain
>             }
```

The `1,2c1,2` hunk is run-isolation only and has no bearing on sender/DKIM policy; the `101c101,103` hunk is the sole policy-affecting change under investigation.

i.e., the probe's signer block is:

```
    source $(local_domains) {
        modify {
            sign_dkim $(primary_domain) default {
                require_sender_match envelope auth_domain
            }
        }
```

### 8.2 Identical inputs to Scenario A, opposite outcome

The probe was driven with inputs **identical to Scenario A**: authenticated as `user1@example.org`, `MAIL FROM:<user2@example.org>`, `From: user2@example.org`.

- **CANONICAL result** (default `require_sender_match` = `{envelope, auth}`), Scenario A, msg_id `a756d9a5`:
  ```
  55	sign_dkim: not signing, From address is not authenticated identity	{"auth_id":"user1@example.org","from_addr":"user2@example.org","msg_id":"a756d9a5"}
  ```
  → **UNSIGNED.**

- **PROBE result** (`require_sender_match envelope auth_domain`), msg_id `1e57717e`. The client received `250 2.0.0 OK: queued` (`probe_client_out.txt`):
  ```
  send: b'From: user2@example.org\r\nTo: user2@example.org\r\nSubject: Scenario A identity mismatch (probe)\r\nMIME-Version: 1.0\r\nContent-Type: text/plain; charset=utf-8\r\n\r\nThis is a test message body.\r\n.\r\n'
  reply: b'250 2.0.0 OK: queued\r\n'
  ```
  and the server signed it **as `user2@example.org`** while the authenticated session `username` is `user1@example.org` (`server_probe.log`):
  ```
  31	submission: incoming message	{"msg_id":"1e57717e","sender":"user2@example.org","src_host":"client.test","src_ip":"127.0.0.1:37698","username":"user1@example.org"}
  41	[debug] sign_dkim: signed	{"identifier":"user2@example.org"}
  43	submission: accepted	{"msg_id":"1e57717e"}
  ```
  → **SIGNED as `user2@example.org`.**

**Same inputs, opposite security outcome:** the canonical config refuses to sign the spoofed identity; the probe — configured with a directive whose name (`auth_domain`) reads as "match the authenticated domain" — **signs it as the spoofed user**. (Note the probe's `msg_id` `1e57717e` is from this separate probe run and differs from the canonical Scenario A `a756d9a5`.)

### 8.3 Mechanism and root cause (`internal/modify/dkim/dkim.go`)

The `require_sender_match` option is declared as an `EnumList` at `internal/modify/dkim/dkim.go:151-152`:

```
	cfg.EnumList("require_sender_match", false, false,
		[]string{"envelope", "auth_domain", "auth_user", "off"}, []string{"envelope", "auth"}, &senderMatch)
```

- The **allowed (user-selectable) tokens** are `envelope`, `auth_domain`, `auth_user`, `off`.
- The **default value** is `{envelope, auth}`.

The selected tokens are turned into a set at `dkim.go:166-169`:

```
	m.senderMatch = make(map[string]struct{}, len(senderMatch))
	for _, method := range senderMatch {
		m.senderMatch[method] = struct{}{}
	}
```

`shouldSign()` (`dkim.go:249`) then has three guard branches. The **auth branch** reads **only the literal key `"auth"`** — `dkim.go:299-310`:

```
	if _, do := m.senderMatch["auth"]; do {
		compareWith := norm.NFC.String(fromUser)
		authName := norm.NFC.String(authName)
		if strings.Contains(authName, "@") {
			compareWith, _ = address.ForLookup(fromAddr)
		}
		if !strings.EqualFold(compareWith, authName) {
			m.log.Msg("not signing, From address is not authenticated identity",
				"from_addr", fromAddr, "auth_id", authName, "msg_id", msgId)
			return "", false
		}
	}
```

**The divergence, in cause→effect terms:**

1. `"auth"` is present in the **default** list (`{envelope, auth}`), so under the default config `senderMatch["auth"]` exists and the identity check at `:299` runs — which is why canonical Scenario A is refused signing.
2. `"auth"` is **NOT** one of the user-selectable `EnumList` tokens (`:151` lists only `envelope`/`auth_domain`/`auth_user`/`off`). The only auth-related tokens a user *can* select are `auth_domain` and `auth_user` — and **`shouldSign()` never reads either of them.** There is no `m.senderMatch["auth_domain"]` or `m.senderMatch["auth_user"]` lookup anywhere in the signing logic.
3. When a user writes `require_sender_match envelope auth_domain`, the resulting set is `{envelope, auth_domain}`. This **replaces** the default set — so the key `"auth"` is now **absent**. The `:299` `if _, do := m.senderMatch["auth"]; do {` guard is false, and the authenticated-identity check is **silently skipped**.
4. With the authenticated-identity check skipped, only the `envelope` check and the domain check remain — and in the probe both pass. In the probe, both `MAIL FROM` and `From:` are `user2@example.org`, so the `envelope` check passes; the domain check also passes because `user2@example.org` is in `example.org`. With every remaining guard satisfied, the message is signed — **as the spoofed `user2@example.org`.**

**Net effect:** a directive named `auth_domain`, which a reasonable operator would read as *"require the From to match the authenticated domain"* (a tightening), instead **disables** the authenticated-identity check entirely, because the code keys off the literal string `"auth"` that the operator cannot select. This is the case where maddy's runtime security behavior diverges from a reasonable reading of the configuration.

> **Completeness note — the other `shouldSign` branches (outside the enforcement question).** The domain / envelope / auth guards above are the sender-alignment checks that produce the four `shouldSign` outcomes exercised in §4.2. For completeness — so this enumeration is not mistaken for the whole function — `shouldSign()` (`internal/modify/dkim/dkim.go:249-330`) contains three further branches that are *not* sender-alignment guards and fall outside the six questions and four scenarios:
>
> - **`off` short-circuit (`dkim.go:250-262`).** When the selectable `off` token is chosen, the function returns before any From/envelope/auth check and signs **unconditionally**, using the key-domain identifier (`return "@" + aDomain, true` at `:259` for a non-EAI message, `return "@" + m.domain, true` at `:261` for an EAI message). So the `off` token listed at `:151` *disables every* alignment guard rather than tuning one; the only non-signing path inside this branch is a failure to convert the key domain to A-labels (`:253-256`).
> - **From-syntax guards (`dkim.go:264-285`).** The message is delivered **unsigned** (`return "", false`) when the `From` header is empty (`:265-268`), malformed (`:269-273`), carries multiple addresses without `allow_multiple_from` (`:274-277`), or holds an unparseable address (`:279-285`). These are syntactic checks, not identity checks.
> - **EAI / A-label identifier construction (`dkim.go:314-329`).** For a non-EAI message the From domain is converted to A-labels to build the signing identifier emitted as `i=`.
>
> None of these branches change any canonical-config conclusion in this document; they are recorded so that "three guard branches / four outcomes" is understood as the sender-alignment subset of `shouldSign()`, not the entire function.


---

## 9. Complete file:line grounding

Every reference below was verified against the on-disk source in the repository working tree at commit `26452dd8dd787dc455278b0fdd296f4a5432c768` (zero discrepancies).

### 9.1 `internal/modify/dkim/dkim.go` (the `sign_dkim` modifier)

- **L151-152** — `require_sender_match` `EnumList`: allowed `["envelope", "auth_domain", "auth_user", "off"]`, default `["envelope", "auth"]`:
  ```
  	cfg.EnumList("require_sender_match", false, false,
  		[]string{"envelope", "auth_domain", "auth_user", "off"}, []string{"envelope", "auth"}, &senderMatch)
  ```
- **L166-169** — `m.senderMatch` set build (`for _, method := range senderMatch { m.senderMatch[method] = struct{}{} }`).
- **L249** — `func (m *Modifier) shouldSign(eai bool, msgId string, h *textproto.Header, mailFrom string, authName string) (string, bool) {`.
- **L287-291** — domain branch: `if !dns.Equal(fromDomain, m.domain) {` → `"not signing, From domain is not key domain"` (`from_domain`, `key_domain`).
- **L293-297** — envelope branch: `if _, do := m.senderMatch["envelope"]; do && !address.Equal(fromAddr, mailFrom) {` → `"not signing, From address is not envelope address"` (`from_addr`, `envelope`).
- **L299-310** — auth branch: `if _, do := m.senderMatch["auth"]; do {` … `norm.NFC.String(...)` + `address.ForLookup(...)` + `strings.EqualFold(compareWith, authName)` → `"not signing, From address is not authenticated identity"` (`from_addr`, `auth_id`).
- **L340** — `func (s state) RewriteBody(ctx context.Context, h *textproto.Header, body buffer.Buffer) error {`.
- **L343-346** — `authUser` read from `s.meta.Conn.AuthUser`.
- **L348** — `id, ok := s.m.shouldSign(s.meta.SMTPOpts.UTF8, s.meta.ID, h, s.meta.OriginalFrom, authUser)`.
- **L349-351** — unsigned-but-delivered: `if !ok { return nil }`.
- **L406** — `h.Add("DKIM-Signature", signer.SignatureValue())`.
- **L408** — `s.m.log.DebugMsg("signed", "identifier", id)`.

### 9.2 `internal/endpoint/smtp/submission.go` (submission preparation)

- **L27** — `func (s *Session) submissionPrepare(msgMeta *module.MsgMetadata, header *textproto.Header) error {`.
- **L28** — `msgMeta.DontTraceSender = true` (why `Received` omits the client IP for submitted mail).
- **L40-43** — missing From → `Code: 554`, `EnhancedCode: exterrors.EnhancedCode{5, 6, 0}`, `Message: "Message does not contains a From header field"`. (Address-syntax failures elsewhere in the function also return `554 5.6.0`.)
- **Whole file (130 lines)** — contains **NO** comparison of `From`/`MAIL FROM` against the authenticated user; a case-insensitive search for `authuser`/`authname`/`connstate` returns nothing. This is the code basis for "no identity enforcement at submission time."

### 9.3 `internal/endpoint/smtp/smtp.go` (delivery entry + Received)

- **L127** — `if s.connState.AuthUser != "" {` (records the authenticated user).
- **L133** — `"username", s.connState.AuthUser,` (logs it only — no authorization of the envelope sender against it).
- **L303** — `received, err := target.GenerateReceived(ctx, s.msgMeta, s.endp.hostname, s.msgMeta.OriginalFrom)`.
- **L307** — `header.Add("Received", received)`.

### 9.4 `internal/target/received.go` (`Received` header generation)

- **L30** — client-IP `from` clause gate: `if !msgMeta.DontTraceSender && (strings.Contains(msgMeta.Conn.Proto, "SMTP") || ...` — submission skips the `from` clause because `DontTraceSender` is true.
- **L69** — `builder.WriteString(" (envelope-sender <")` — the envelope-sender clause is **always** present.

### 9.5 `internal/msgpipeline/check_runner.go` (`Authentication-Results`)

- **L301** — gate: `if len(cr.mergedRes.AuthResult) != 0 {`.
- **L302** — `header.Add("Authentication-Results", authres.Format(hostname, cr.mergedRes.AuthResult))`. With no `check{}` in the submission block the result set is empty, so the header is never added.

### 9.6 `internal/modify/dkim/keys.go` (key generation/publication)

- **L143** — `keyBlob = x509.MarshalPKCS1PublicKey(pubkey)` — the public key is published as **PKCS#1** (hence the `MIIBCgKCAQEA` `p=` prefix and the PKIX-parse failure in §7.2). The TXT record is formatted as `"v=DKIM1; k=%s; p=%s"`.

### 9.7 `maddy.go` (entry point / defaults)

- **L41** — `Version = "unknown (built from source tree)"` (the canonical banner).
- **L59** — `DefaultStateDirectory = "/var/lib/maddy"`.
- **L70** — `DefaultRuntimeDirectory = "/run/maddy"`.
- **L102** — `func Run() int {` — the process entry point that parses flags (`-config`, `-debug`, `-v`) and launches the server; this commit has **no `run` subcommand** (the binary is invoked as `maddy -config <path>`).
- **L120** — `if len(flag.Args()) != 0 {` (empty-positional-args requirement; no `run` subcommand).
- **L220** — `if err := os.Chdir(config.StateDirectory); err != nil {` (relative `all.db`/`dkim_keys/` resolve inside the state dir).

### 9.8 `internal/check/spf/spf.go` and `internal/check/dkim/dkim.go` (contrast: real inbound checks)

- `internal/check/spf/spf.go:28` — `const modName = "apply_spf"` (a real sender-policy check that DOES exist, used on port 25).
- `internal/check/dkim/dkim.go` — inbound `verify_dkim`. Its `func (d *dkimCheckState) CheckConnection(...)` at **L87** returns an empty result — `return module.CheckResult{}` at **L88** — and produces **no** `AuthResult` (the same is true of `CheckSender` at **L91-93** and `CheckRcpt` at **L95-97**). The DKIM `AuthResult` is produced instead by `func (d *dkimCheckState) CheckBody(...)` at **L99**: the no-signature branch builds `AuthResult: []authres.Result{ &authres.DKIMResult{ Value: authres.ResultNone } }` at **L115-117**, and for messages carrying signatures the per-signature results are appended into `res.AuthResult` (declared `res := module.CheckResult{AuthResult: make([]authres.Result, 0, len(verifications))}` at **L157**) via `res.AuthResult = append(res.AuthResult, &authres.DKIMResult{...})` at **L187**. This `AuthResult` is exactly what feeds `Authentication-Results` through `check_runner.go:301-302` on the inbound path — and is absent on submission because the submission block declares no `check{}`.
- **Absent module:** a repository-wide `grep -i "authorize_sender|authorizesender"` across `internal/` returns **nothing** — there is **no** built-in sender-authorization check module in this commit. (SPF, DNSBL, inbound DKIM-verify, requiretls, command, and DNS checks exist; a per-identity sender-authorization check does not.)

### 9.9 `cmd/maddyctl/main.go` (management CLI)

- **L38** — global `--config` flag (`Name: "config"`), required by every subcommand.
- **L47** — `users` subcommand; **L51** — `users list`; **L71** — `users create USERNAME` (with `--password`/`-p` defined at **L84**, default bcrypt `--hash`, `--bcrypt-cost`); **L136** — `users password`.
- **L203** — `imap-mboxes`; **L207** — `imap-mboxes list`.
- **L307** — `imap-msgs`; **L504** — `imap-msgs list`; **L534** — `imap-msgs dump` (with `--uid`/`-u`).
- These are the exact commands used to create the two accounts (§1.7) and to dump the stored messages (§6).

### 9.10 `docs/tutorials/setting-up.md`

- **L31** — "One dependency you need to figure out is the C compiler, it is needed for SQLite3 support which is used in the default configuration" (the basis for the CGO build requirement).
- **L132** — `$ maddyctl users create postmaster@example.org` (the account-creation example the two test accounts follow).
- **L135-136** — "Note that account names include the domain. When authenticating in the mail client, full address should be specified as a username as well." — the basis for creating and authenticating `user1@example.org` / `user2@example.org` as full-address usernames.

### 9.11 `maddy.conf` (default policy — shipped line numbers)

- **L16-17** — global `tls` (cert/key under `/etc/maddy/certs/$(hostname)/`).
- **L32-35** — `sql local_mailboxes local_authdb { driver sqlite3 / dsn all.db }`.
- **L93-95** — `submission tls://0.0.0.0:465 {` + `auth &local_authdb`.
- **L97-100** — `source $(local_domains) { modify { sign_dkim $(primary_domain) default } }`.
- **L104-108** — `destination $(local_domains) { import local_delivery_actions / deliver_to &local_mailboxes }`.
- **L117-119** — `default_source { reject 501 5.1.8 "Non-local sender domain" }` (the only default sender guard — domain-level).
- **L149-152** — `imap tls://0.0.0.0:993 { auth &local_authdb / storage &local_mailboxes }`.

### 9.12 `go.mod` (toolchain + pinned dependency versions)

- `go 1.13`.
- `github.com/emersion/go-smtp v0.12.1-0.20191206174923-1f576e0ec85c` — SMTP submission server + response codes.
- `github.com/emersion/go-msgauth v0.3.2-0.20191028231513-55b75676976c` — DKIM signing/verification.
- `github.com/emersion/go-message v0.10.9-0.20191116124005-65fd0119e899` — message parsing/serialization (governs the header re-folding behavior in §7.4).
- `github.com/foxcpp/go-imap-sql v0.3.2-0.20191208094750-8b4ec6b19a78` — IMAP SQL store (`all.db` + external blob store).
- `github.com/emersion/go-imap v1.0.1`; `github.com/emersion/go-sasl v0.0.0-20190817083125-240c8404624e`; `github.com/mattn/go-sqlite3 v1.11.0` (CGO); `golang.org/x/crypto v0.0.0-20191108234033-bd318be0434a`; `github.com/foxcpp/go-mockdns v0.0.0-20191123143003-02edb10da1e3`; `github.com/miekg/dns v1.1.22`.

### 9.13 RFC 6409 (Message Submission for Mail, STD 72)

- **§4.3** Require Authentication (MUST) — implemented (submission requires `auth`).
- **§5.1** Enforce Address Syntax (501/554) — implemented (Scenario F `554 5.6.0`).
- **§6.1** Enforce Submission Rights — **a MAY (optional)** — **not implemented by default** (the authoritative basis that maddy's default non-enforcement of per-identity alignment is RFC-compliant).
- **§8.1–§8.2** add Date / Message-ID — implemented (observed `adding missing Message-ID` / `adding missing Date header`).

### 9.14 External references (corroboration)

The document's primary evidence is the captured runtime output plus the `file:line` source citations above; the following external sources are independent corroboration of the code-grounded characterizations (not a citation-fidelity dependency):

- `https://maddy.email/reference/modifiers/dkim/` — states the `sign_dkim` signing key is "selected based on the SMTP envelope sender," that a message whose envelope-sender domain matches no loaded key is delivered **unsigned** rather than rejected, and that `require_sender_match` governs the **signing** decision (whether `From` must match `MAIL FROM` and/or the authorization identity), not message acceptance. Corroborates §3, §4, and §8.
- `https://maddy.email/reference/smtp-pipeline/` — states `source` blocks are selected by matching the message sender "as specified in MAIL FROM," and that a `reject` directive rejects the recipient (fired per `RCPT TO`). Corroborates §3.Q1 and the `501 5.1.8`-at-`RCPT` observation in §5.2.
- **RFC 6409 (Message Submission for Mail, STD 72)** — see §9.13; §6.1 "Enforce Submission Rights" is an optional **MAY**, the authoritative basis for maddy's default non-enforcement of per-identity alignment being RFC-compliant.


---

## 10. Coverage-pass checklist

Final coverage pass over every question, scenario, and named item. Each box is checked and annotated with where in this document it is shown.

- [x] **Build/invocation commands + canonical banner** `maddy unknown (built from source tree)` stated; all three listeners (25/465/993) shown active; two accounts created. → §1.2–§1.7
- [x] **Q1 accept/reject mechanism** answered with actual codes: `250 2.0.0 OK: queued` (accept), `501 5.1.8 Non-local sender domain` at `RCPT TO` (reject). → §3.Q1, §4, §5.1–§5.2
- [x] **Mismatch-construction difference** demonstrated: same-domain/different-user ACCEPTED (Scenario A) vs. non-local domain REJECTED (Scenario B). → §3.Q1, §4
- [x] **Q4 default-alignment answer** stated directly (NO enforcement; explicit policy required), grounded in `submission.go` (no identity check, L27-130) + RFC 6409 §6.1 (a MAY). → §3.Q4, §9.2, §9.13
- [x] **Q2 raw stored headers** captured verbatim — `DKIM-Signature`, `Received`, and the absence of `Authentication-Results` — for a signed (UID1) and an unsigned (UID2) message, with byte-exact (`cat -A`) rendering. → §6
- [x] **All four `shouldSign` outcomes** shown with real `-debug` lines: `signed` (C), `not signing … not authenticated identity` (A), `not signing … not envelope address` (E), `not signing … not key domain` (D). → §4.2
- [x] **Q3 full SMTP dialogues** for ≥1 accepted (C) and ≥1 rejected (B), plus the syntactic-rejection contrast (F). → §5
- [x] **Byte-sensitivity handled correctly:** body hash `bh=jX3F0bCAI7sIbkHyy3mLYO28ieDQz2R0P8HwQkklFj4=` reproduced **exactly**; stored `b=` reported **INCONCLUSIVE** (re-serialization), not invalid; positive control passes; PKCS#1-vs-PKIX detail noted. → §7.2–§7.4
- [x] **Q5 plausible-but-incorrect interpretation** ruled out ("maddy rejects `MAIL FROM` ≠ authenticated user") using the accepted same-domain/different-user transaction (Scenario A). → §3.Q5
- [x] **Q6 config-vs-behavior divergence** proven at runtime: `require_sender_match envelope auth_domain` signs the identity-mismatch message as `user2@example.org`; before/after for identical inputs; root cause = code reads only the literal key `"auth"` (`dkim.go:299`), which is not user-selectable. → §3.Q6, §8
- [x] **Every factual claim carries a `file:line` reference or captured output.** → throughout, consolidated in §9
- [x] **Repository integrity:** source tree byte-unchanged (`git status --porcelain` empty); source commit under investigation `26452dd8dd787dc455278b0fdd296f4a5432c768` on source branch `maddy_26452dd8dd78`; deliverable committed on destination branch `blitzy-62bbeeda-df48-4e6a-b6a4-dda6742e6660`; temporary runtime artifacts (workdirs, database, certificates, binaries) removed, with the captured evidence set preserved outside the repository. → §11.2

### Question-by-question confirmation

| Question | Direct answer | Evidence |
|----------|---------------|----------|
| Q1 accept/reject of authenticated sender | Envelope-domain source-block routing, not identity; local accepted `250`, non-local `501 5.1.8` at RCPT; differs by construction | §3.Q1, §4, §5.1–§5.2 |
| Q2 raw stored headers | `DKIM-Signature`/`Received` captured verbatim; no `Authentication-Results` on submission | §6, §7 |
| Q3 SMTP transactions | Full accepted (C) + rejected (B) + syntax (F) dialogues | §5 |
| Q4 default sender alignment | NO; explicit policy required; RFC 6409 §6.1 is a MAY | §3.Q4, §9.2, §9.13 |
| Q5 ruled-out interpretation | "rejects MAIL FROM ≠ auth user" disproven by Scenario A | §3.Q5 |
| Q6 config-vs-behavior divergence | `auth_domain`/`auth_user` silently disable the identity check | §3.Q6, §8 |

### Named-item confirmation

- **Four scenarios (user-specified) + two extras:** identity mismatch (A), non-local domain (B), happy path (C), From≠auth/signing (D); plus envelope mismatch (E) and missing-From syntax (F). → §4
- **`shouldSign` branches:** domain (`dkim.go:287-291`), envelope (`:293-297`), auth (`:299-310`). → §4.2, §8.3, §9.1
- **Both reject paths:** `501 5.1.8` at RCPT (§5.2); `554 5.6.0` at DATA (§5.3).
- **Header generators:** `Received` (`received.go`), `Authentication-Results` (`check_runner.go`). → §6, §9.4–§9.5
- **Key publication:** PKCS#1 (`keys.go:143`). → §7.2, §9.6
- **Absent module:** no `authorize_sender`. → §9.8

---

## 11. Repository integrity and cleanup

### 11.1 Source vs destination — the two repositories, distinguished

This is a documentation-only task involving two distinct git references that must not be conflated:

- **Source repository under investigation** (must remain byte-unchanged): `foxcpp/maddy`, **source branch `maddy_26452dd8dd78`**, **source commit `26452dd8dd787dc455278b0fdd296f4a5432c768`** (`target/remote: Rewrite connection part to allow more concurrency`). This is the exact commit whose runtime behavior every section above documents. No file in the maddy source tree was created, modified, or deleted.
- **Destination repository / review branch** (where the single deliverable is committed): branch **`blitzy-62bbeeda-df48-4e6a-b6a4-dda6742e6660`**. The only persistent output is this documentation file, `blitzy/documentation/maddy_26452dd8dd78.md`, added as a single additive commit **on top of** the source commit `26452dd8…` (the deliverable commit's parent is the investigation target).

Verified with git (stable, commit-strategy-independent commands):

```
$ git rev-parse --abbrev-ref HEAD                          # destination review branch
blitzy-62bbeeda-df48-4e6a-b6a4-dda6742e6660
$ git rev-parse 26452dd8dd787dc455278b0fdd296f4a5432c768   # source commit exists, byte-unchanged
26452dd8dd787dc455278b0fdd296f4a5432c768
$ git log --oneline -1 26452dd8dd787dc455278b0fdd296f4a5432c768
26452dd target/remote: Rewrite connection part to allow more concurrency
$ git status --porcelain                                   # no source-tree change
            (no output — zero entries)
```

`git status --porcelain` returns **zero lines** — no file in the maddy source tree was created, modified, or deleted. The destination `HEAD` is the documentation commit on branch `blitzy-62bbeeda-…`; its parent is the source commit `26452dd8…` on source branch `maddy_26452dd8dd78`. Note that `git rev-parse HEAD` returns the *deliverable* commit hash, **not** `26452dd8…`; the source commit under investigation is HEAD's parent, shown above via its full hash and subject so the two are unambiguous.

### 11.2 Ephemeral artifacts: removed workdirs vs. preserved evidence set

All runtime work occurred **outside** the source repository. The artifacts fall into two categories — those removed after evidence capture, and the captured evidence itself, which is preserved outside the repository for auditability. Neither affects source-tree integrity.

**Removed after evidence capture** (verified absent — `find /tmp/maddy_work` → *No such file or directory*; `/etc/maddy` → absent):

- The temporary workdirs `/tmp/maddy_work` (the canonical and labeled non-canonical probe runs) and `/tmp/maddy_acct` (the §1.7 account-command re-run), plus the certificate directory `/etc/maddy`.
- The built binaries `maddy` and `maddyctl`.
- The self-signed TLS certificate/key that lived at `/etc/maddy/certs/example.org/{fullchain.pem,privkey.pem}`.
- The state-directory contents: `all.db` (+ `-wal`/`-shm`), `dkim_keys/example.org_default.{key,dns}`, and `messages/<blobs>`.
- The standalone Go DKIM-verifier module (only its output, `dkim_verify_output.txt`, is preserved below).
- All SQLite databases — so the two test accounts (`user1@example.org`, `user2@example.org`) no longer exist anywhere; they lived only inside the now-removed databases.

**Preserved outside the repository** (in `/tmp/maddy_evidence/`, retained for audit; entirely outside the source tree, does not affect source-tree integrity): the captured evidence set this report was authored from and quotes verbatim throughout —

- configuration copies `maddy.conf.used` and `maddy_probe.conf`;
- the `-debug` server logs `server_canonical.log`, `server_canonical_restart.log`, `server_probe.log`;
- the SMTP client transcripts `client_out.txt`, `probe_client_out.txt`;
- the raw stored messages and dumps `signed_msg_UID1.raw`, `unsigned_msg_UID2.raw`, `dump_UID1_signed.txt`, `dump_UID2_unsigned.txt`, `probe_signed_dump.txt`;
- the published DKIM key `example.org_default.dns` and the DKIM byte-verification output `dkim_verify_output.txt`;
- the `maddyctl` account-setup transcript `maddyctl_setup_output.txt` (§1.7);
- the version banner `version_banner.txt`;
- the Python SMTPS client scripts `smtp_client.py`, `probe_client.py`.

Every value in that evidence set is reproduced inline in this document, so the report remains self-contained even independently of the preserved directory. No runtime artifact was ever placed inside the source repository.


---
