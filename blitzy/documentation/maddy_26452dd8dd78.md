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

**The *admission* decision on the authenticated submission path never compares `From` to `MAIL FROM` or to the authenticated identity** — no `From`-alignment check ever gates *acceptance*. This narrower claim is the precise one: the `sign_dkim` modifier **does** compare `From` to the envelope (`address.Equal(fromAddr, mailFrom)` at `dkim.go:293`) and to the authenticated identity (`strings.EqualFold(compareWith, authName)` at `dkim.go:305`), but those comparisons feed **only** the sign/skip decision (gate 3 above) — they never accept or reject the message. On the acceptance path proper, the only `From` handling is `submissionPrepare`, which validates *presence and syntax* of `From`/`Sender`/`To`/`Cc`/`Bcc`/`Reply-To` and injects `Message-ID`/`Date` if missing — it contains **no** alignment comparison. *(Source-grounded: `internal/endpoint/smtp/submission.go` — `submissionPrepare` L27; missing-`From` → `554 5.6.0` at L39-48. The `From`-vs-envelope / `From`-vs-auth comparisons that DO exist live only in `internal/modify/dkim/dkim.go` `shouldSign` at L293 and L305, and feed only sign/skip, confirmed by the T1/T4 UNSIGNED-but-ACCEPTED observations.)*

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

- **Ran the code first, then wrote.** All behavioral claims derive from output captured from a running daemon driven through its real submission endpoint (§4–§9); code reading is used only to *explain* observations, and such explanations are labeled source-grounded.
- **Actual, complete, unedited output for every claim, with the command that produced it.** The build, configuration, provisioning, and startup evidence appears with its originating command in §3; each scenario shows the **complete** client transcript and/or server `io_debug` block in a fenced code block (§5); nothing behavioral is elided. The one literal ellipsis anywhere in captured output is Maddy's own log string `generating a new rsa2048 keypair...` (§3.7), reproduced as emitted.
- **Real scale, stability across ≥2 runs.** Each scenario T1–T6 was executed **twice over plaintext port 587 and once over implicit-TLS port 465** — 18 runs total, all driven through the real endpoint. Results were identical and transport-independent; the complete per-run ledger (run → transport → response → message ID → stored UID) is in §5.4, and outcomes were confirmed stable before being reported.
- **Exact code path through the real entry point.** The path exercised is SMTP `AUTH` → message pipeline (`msgpipeline`) → DKIM modifier (`sign_dkim` `RewriteBody`) → local storage (`imapsql` SQLite). No bypassing or synthetic interface was used. The supplemental plaintext `587` listener is a transport onto the **same** endpoint module and pipeline as the canonical implicit-TLS `465` listener; it is labeled supplemental wherever it is used, and §5.4 shows `465` and `587` produce identical outcomes.
- **Default, canonical configuration.** The runtime mirrors the repository's default submission semantics (§3.5 reproduces the complete config, with its recipient-side DNS-independent deviations called out); the build and invocation commands are stated verbatim (§3.2, §3.6), and the canonical version banner (`maddy unknown (built from source tree)`) is reported as-is (§3.3).
- **Every condition exercised.** Primary (T2 aligned) plus secondary/edge/error paths (T1 cross-user, T3 non-local envelope, T4 `From`≠envelope, T5 `From`-domain≠key, T6 missing `From`); state observed *before* delivery (accept/reject at the protocol) and *after* delivery (signed/unsigned in storage).
- **Exact and grounded.** Every factual claim carries an actual value plus a `file:line` citation naming the function/method/struct; **OBSERVED vs INFERRED is labeled** throughout; the direct answer leads and includes negative results.
- **Byte-exact DKIM verification.** The `DKIM-Signature` was read from the stored message bytes and its `h=` tag inspected byte-for-byte to confirm `From` coverage (§6.3), and the published key was run through the same verifier library Maddy uses (§9).
- **Read-only source tree, transient cleanup, final status.** No existing repository file was modified and no code was added; every transient artifact (config, TLS material, SQLite database, DKIM keys, scripts, captures, logs) was created outside the source tree and removed afterward. The exact isolation, teardown, and `git status` verification commands and output are in §3.10.

---

## 3. How the runtime was built, configured, and provisioned (OBSERVED)

All commands below were run in the canonical build/run container (Go 1.18.10, gcc 10.2.1, Debian 11) with the repository bind-mounted at `/app`. All transient runtime state lives in a working directory **outside the source tree** at `/tmp/maddy_run/` (state at `/tmp/maddy_run/state`, config at `/tmp/maddy_run/maddy.conf`, database `all.db`, DKIM keys, capture files, and scripts), so the source tree is never touched. §3.10 documents the isolation, teardown, and final `git status`.

### 3.0 Test-only safety caveats (READ FIRST)

> **⚠ Every credential, key, and transport choice below is SYNTHETIC, DISPOSABLE, and TEST-ONLY, destroyed at cleanup (§3.10). None is a real secret.** Specifically:
>
> - **Synthetic passwords.** `AlicePass123` / `BobPass123` are throwaway test values, not real credentials. The base64 SASL PLAIN payload shown in transcripts (§5) is trivially decodable *by design* — it is included because the rules require complete, unedited protocol bytes, and it encodes only these disposable passwords.
> - **`-p` exposes passwords in shell history.** Provisioning with `maddyctl users create ... -p <pw>` places the password on the command line. Maddy's own CLI warns about this — `cmd/maddyctl/main.go:84` documents the `password,p` flag (L83) as *"…WARNING: Provided only for debugging convenience. Don't leave your passwords in shell history!"*, and by default (`Description: "Reads password from stdin"`, L73) the password is read from stdin. In production, omit `-p` and let the CLI read from stdin. It is used here only so provisioning is a single reproducible line.
> - **`io_debug` can log passwords.** The supplemental `587` listener enables `io_debug`, which logs raw SMTP I/O — including the AUTH line. Maddy emits the warning `submission: I/O debugging is on! It may leak passwords in logs, be careful!` on startup (observed in §3.6; source `internal/endpoint/smtp/smtp.go:565`). It is enabled here solely to obtain server-side protocol bytes for capture.
> - **Plaintext AUTH is confined to loopback.** The `587` listener sets `tls off` + `insecure_auth`, permitting AUTH without TLS. It binds **only** to `127.0.0.1` and exists only for clean protocol capture; the canonical, primary endpoint is implicit-TLS `465`. Maddy's source has two "insecure configuration … used only for testing!" warnings — `authentication over unencrypted connections is allowed…` (`internal/endpoint/smtp/smtp.go:538`) and `TLS is disabled…` (`:542`) — but **both are guarded by `!allLocal`** (L537, L541). Because this listener is loopback-only, `allLocal` is true, so **those two warnings were suppressed and did not appear in the startup log** (§3.6); only the unconditional `endp.serv.AllowInsecureAuth = true` at `:545` took effect. (The separate `io_debug` warning is unconditional and *did* appear.) This loopback guard is itself the mechanism confining plaintext AUTH to local capture.
> - **`ssl.CERT_NONE` is test-only.** The Python client disables certificate verification for the self-signed `465` endpoint (`ctx.verify_mode = ssl.CERT_NONE`, §3.9). This is acceptable only because the certificate is a throwaway self-signed cert on loopback; never disable verification against real endpoints.

### 3.1 Toolchain and environment (OBSERVED)

CGO is **mandatory** — the storage/auth backend uses the `mattn/go-sqlite3` driver (`go.mod` L26, `github.com/mattn/go-sqlite3 v1.11.0`), a cgo package requiring a C compiler. The exact toolchain and build environment:

```
$ go version
go version go1.18.10 linux/amd64

$ gcc --version
gcc (Debian 10.2.1-6) 10.2.1 20210110

$ echo CGO_ENABLED=$CGO_ENABLED GOOS=$GOOS GOARCH=$GOARCH GO111MODULE=$GO111MODULE
CGO_ENABLED=1 GOOS=linux GOARCH=amd64 GO111MODULE=on

$ python3 --version
Python 3.9.2
```

### 3.2 Build (OBSERVED)

Both `cmd/maddy` (the daemon) and `cmd/maddyctl` (the admin CLI, needed for user provisioning and mailbox inspection) were built from the source tree with `CGO_ENABLED=1`, output to the working directory outside the tree. The only compiler output is a single harmless CGO warning from `mattn/go-sqlite3` (a known upstream `-Wreturn-local-addr` diagnostic in the bundled SQLite amalgamation); both builds exit `0`:

```
$ CGO_ENABLED=1 go build -trimpath -tags debug -o /tmp/maddy_run/maddy ./cmd/maddy
# github.com/mattn/go-sqlite3
sqlite3-binding.c: In function 'sqlite3SelectNew':
sqlite3-binding.c:125322:10: warning: function may return address of local variable [-Wreturn-local-addr]
125322 |   return pNew;
       |          ^~~~
sqlite3-binding.c:125282:10: note: declared here
125282 |   Select standin;
       |          ^~~~~~~
build maddy exit=0

$ CGO_ENABLED=1 go build -trimpath -o /tmp/maddy_run/maddyctl ./cmd/maddyctl
# github.com/mattn/go-sqlite3
sqlite3-binding.c: In function 'sqlite3SelectNew':
sqlite3-binding.c:125322:10: warning: function may return address of local variable [-Wreturn-local-addr]
125322 |   return pNew;
       |          ^~~~
sqlite3-binding.c:125282:10: note: declared here
125282 |   Select standin;
       |          ^~~~~~~
build maddyctl exit=0
```

The `-tags debug` build tag enables the verbose `[debug]` log lines relied on in §5; it does not change behavior. *(Source-grounded: `go.mod` L1 module `github.com/foxcpp/maddy`, L3 `go 1.13`.)*

### 3.3 Canonical version banner (OBSERVED)

The value a plain source build reports:

```
$ /tmp/maddy_run/maddy -v
maddy unknown (built from source tree)
$ /tmp/maddy_run/maddyctl -v
maddyctl version unknown (built from source tree)
```

This is the **canonical build artifact**, not a broken environment: the `Version` string is literally defined as `"unknown (built from source tree)"`. *(Source-grounded: `maddy.go:41`, `Version = "unknown (built from source tree)"`; the `-v` flag prints `"maddy " + BuildInfo()` at `maddy.go:126`.)*

### 3.4 State/runtime directories, daemon flags, and working-directory chdir

The daemon entry point is trivially thin — `cmd/maddy/main.go` does only `os.Exit(maddy.Run())`. The flags are declared in `maddy.go`: `-libexec` (L103), `-debug` (L104), `-config` (L107), `-log` (L108), `-v` (L109). On startup the daemon **changes its working directory to the state directory** (`os.Chdir(config.StateDirectory)`, `maddy.go:220`), so relative paths in configuration — `dsn all.db` and `dkim_keys/{domain}_{selector}.key` — resolve relative to that state directory. The config therefore sets `state /tmp/maddy_run/state` and `runtime /tmp/maddy_run/runtime` explicitly, and the same `state` directory is used as the working directory for `maddyctl` provisioning (§3.8) so its relative `dsn all.db` resolves to the identical database. The daemon was started with `-debug` (§3.6).

### 3.5 Canonical `maddy.conf` (complete, OBSERVED)

The test configuration was authored **outside the source tree** at `/tmp/maddy_run/maddy.conf`, faithful to the repository's default submission semantics (`maddy.conf` L93–L120). It is reproduced here **in full** so the runtime is exactly reproducible. Two recipient-side deviations, both DNS-independent and never exercised by the sender-authorization/DKIM tests (whose recipients are all local), are called out in the header comment: local-domain destinations `deliver_to &local_mailboxes` directly (the default `import local_delivery_actions` alias/plus-addr macro, which needs `/etc/maddy/aliases`, is omitted as irrelevant), and the remote `default_destination` is a `reject` instead of `deliver_to &remote_queue` (there is no external MX in a local run). The supplemental plaintext `587` listener (`tls off`, `insecure_auth`, `io_debug`) exists purely for clean protocol capture (see the §3.0 caveats); the **primary** endpoint remains implicit-TLS submission on `465`.

```
## Canonical test maddy.conf (commit 26452dd) — authored OUTSIDE the source tree.
## Faithful to the repository default submission semantics (maddy.conf L93-L120).
## Deviations, all DNS-independent and on the RECIPIENT side (never exercised by
## the sender-authorization / DKIM tests, all of whose recipients are local):
##   - local-domain destinations deliver_to &local_mailboxes directly (the default
##     "import local_delivery_actions" alias/plus-addr macro is omitted; it needs
##     /etc/maddy/aliases which is irrelevant to sender-auth/DKIM);
##   - remote default_destination is a reject instead of "deliver_to &remote_queue"
##     (no external MX in a local, DNS-independent run).
## A supplemental plaintext submission listener on 127.0.0.1:587 (tls off,
## insecure_auth, io_debug) is added purely for clean protocol capture; the
## primary endpoint remains implicit-TLS submission on :465.

$(hostname) = example.org
$(primary_domain) = example.org
$(local_domains) = $(primary_domain)

state /tmp/maddy_run/state
runtime /tmp/maddy_run/runtime

hostname $(hostname)
autogenerated_msg_domain $(primary_domain)

tls self_signed

sql local_mailboxes local_authdb {
    driver sqlite3
    dsn all.db
}

# Primary submission endpoint: implicit TLS on 465 (repository default).
submission tls://0.0.0.0:465 {
    auth &local_authdb

    source $(local_domains) {
        modify {
            sign_dkim $(primary_domain) default
        }
        destination $(local_domains) {
            deliver_to &local_mailboxes
        }
        default_destination {
            reject 550 5.1.1 "Remote delivery disabled in test config"
        }
    }

    default_source {
        reject 501 5.1.8 "Non-local sender domain"
    }
}

# Supplemental plaintext submission endpoint on 587 for clean protocol capture.
submission tcp://127.0.0.1:587 {
    auth &local_authdb
    tls off
    insecure_auth
    io_debug

    source $(local_domains) {
        modify {
            sign_dkim $(primary_domain) default
        }
        destination $(local_domains) {
            deliver_to &local_mailboxes
        }
        default_destination {
            reject 550 5.1.1 "Remote delivery disabled in test config"
        }
    }

    default_source {
        reject 501 5.1.8 "Non-local sender domain"
    }
}

imap tls://0.0.0.0:993 {
    auth &local_authdb
    storage &local_mailboxes
}
```

The directives that matter for the question, with their grounding in the repository default `maddy.conf`:

- **Unified SQLite storage + auth:** `sql local_mailboxes local_authdb { driver sqlite3; dsn all.db }` — `maddy.conf` L32-34. One database (`all.db`) backs both credential checks and IMAP mailbox delivery. The module is registered as `sql` at `internal/storage/sql/sql.go:424` (`module.Register("sql", New)`).
- **Primary submission endpoint (implicit TLS):** `submission tls://0.0.0.0:465 { auth &local_authdb ... }` — `maddy.conf` L93, L95.
- **Supplemental plaintext submission listener for clean protocol capture:** `submission tcp://127.0.0.1:587` with `tls off`, `insecure_auth`, and `io_debug`. When no TLS is configured the endpoint auto-enables insecure auth (`internal/endpoint/smtp/smtp.go:537-546`, specifically `endp.serv.AllowInsecureAuth = true` at L545 when `TLSConfig == nil`); `insecure_auth` is the explicit knob (L564) and `io_debug` (L565) emits raw server-side SMTP I/O. **This is the same endpoint module and pipeline as `465`, reached over a different transport — §5.4 shows the two produce identical outcomes.**
- **Sender routing (identical in both listeners):** `source $(local_domains) { modify { sign_dkim $(primary_domain) default } destination ... deliver_to &local_mailboxes }` — `maddy.conf` L97, L99; catch-all `default_source { reject 501 5.1.8 "Non-local sender domain" }` — L117-118. The `maddy.conf` comment at L115-116 states the intent: block non-local sender domains as "likely a spoofing attempt". Config vars: `hostname example.org` (L4), `primary_domain example.org` (L9), `local_domains $(primary_domain)` (L12).
- **TLS:** `tls self_signed` so Maddy auto-generates a self-signed cert (no external `openssl` needed); the endpoint inherits the TLS config via `cfg.Custom("tls", ...)` at `internal/endpoint/smtp/smtp.go:563`. The self-signed mode logs `tls: using self-signed certificate, this is not secure!` on startup (observed in §3.6; source `internal/config/tls_server.go:32`).

### 3.6 Daemon invocation and startup log (OBSERVED)

The daemon was started detached, with `-debug`, writing its log to a file outside the source tree; its PID was recorded so that only that process is stopped at teardown (§3.10):

```
$ cd /tmp/maddy_run
$ nohup ./maddy -debug -config /tmp/maddy_run/maddy.conf > /tmp/maddy_run/maddy.log 2>&1 &
$ echo $! > /tmp/maddy_run/maddy.pid
```

The complete startup log (`/tmp/maddy_run/maddy.log`, reproduced verbatim; each line ends with a trailing tab where Maddy's structured-field separator sits) shows the self-signed-TLS warning, the one-time DKIM key generation, the `io_debug` password-leak warning on the `587` listener, and all three listeners coming up:

```
tls: using self-signed certificate, this is not secure!
[debug] sql: go-imap-sql version 0.4.0
[debug] /tmp/maddy_run/maddy.conf:33: reference &local_authdb
[debug] /tmp/maddy_run/maddy.conf:37: new module sign_dkim [example.org default]
sign_dkim: generating a new rsa2048 keypair...
sign_dkim: generated a new rsa2048 keypair, private key is in dkim_keys/example.org_default.key, TXT record with public key is in dkim_keys/example.org_default.dns,
put its contents into TXT record for default._domainkey.example.org to make signing and verification work
[debug] /tmp/maddy_run/maddy.conf:40: reference &local_mailboxes
[debug] submission: authentication provider: sql local_mailboxes
submission: listening on tls://0.0.0.0:465
[debug] /tmp/maddy_run/maddy.conf:54: reference &local_authdb
[debug] /tmp/maddy_run/maddy.conf:61: new module sign_dkim [example.org default]
[debug] /tmp/maddy_run/maddy.conf:64: reference &local_mailboxes
submission: I/O debugging is on! It may leak passwords in logs, be careful!
[debug] submission: authentication provider: sql local_mailboxes
submission: listening on tcp://127.0.0.1:587
[debug] /tmp/maddy_run/maddy.conf:77: reference &local_authdb
[debug] /tmp/maddy_run/maddy.conf:78: reference &local_mailboxes
imap: listening on tls://0.0.0.0:993
```

The `tls: using self-signed certificate, this is not secure!` line confirms the `tls self_signed` mode (§3.5); the `submission: I/O debugging is on! It may leak passwords in logs, be careful!` line is the `io_debug` leakage warning referenced in the §3.0 caveats.

### 3.7 DKIM key auto-generation (OBSERVED)

On first start, the `sign_dkim example.org default` modifier auto-generated the keypair under the state directory. The private key is PKCS#8 PEM at mode `600`; the public record is a bare `v=DKIM1; k=rsa; p=<base64>` TXT record:

```
$ ls -la dkim_keys/
total 16
drwxr-xr-x 2 root root 4096 Jul 16 00:41 .
drwxr-xr-x 4 root root 4096 Jul 16 00:41 ..
-rw-r--r-- 1 root root  378 Jul 16 00:41 example.org_default.dns
-rw------- 1 root root 1704 Jul 16 00:41 example.org_default.key

$ stat -c "%a %n" dkim_keys/example.org_default.key
600 dkim_keys/example.org_default.key

$ cat dkim_keys/example.org_default.dns
v=DKIM1; k=rsa; p=MIIBCgKCAQEAwXh+ejKBmEwhTRbkAtD76eRmkzBiOGJUzctlp4eICRgD57xC08jFt0lemury+MKwoaWO4AyajaB1G1P/wiHuF7iLgEnc+Ckg/cCINCLssz3BdBtgTO73roIYqNtMmjfBEaIcittFrpjxIMBM6kiNaNiRqOivU9/dbhJ6zP+sV00syEhU3fMgUzc6sE0yFJKbm1zdpaYutkvWQV2oKEU0P4RjtWuGLEOrnd0MkleLzmBTuyVz+OqrzpegEQ4a3L9IKEhL1jaNiJsJkWZpaU7MGmfr7tdYyEimtgHP/kVOgVCmQ7APeTT6fxDCKzLryVwkCFaE0cSNkBDujMu3VNV0DQIDAQAB

$ head -1 dkim_keys/example.org_default.key   # PEM type only (private key not otherwise shown)
-----BEGIN PRIVATE KEY-----
```

The default new-key algorithm is `rsa2048`; the selector `default` and domain `example.org` come directly from `sign_dkim $(primary_domain) default`. The base64 `p=` value begins `MIIBCgKC`, which decodes to DER `30 82 01 0a 02 82 01 01 00` — a **bare PKCS#1 `RSAPublicKey`**, not a PKIX `SubjectPublicKeyInfo`. That encoding choice is the subject of §9 Finding A. *(Source-grounded: `internal/modify/dkim/dkim.go` — key path template `dkim_keys/{domain}_{selector}.key` L137, new-key log L192-194; `internal/modify/dkim/keys.go` — `writeDNSRecord` L136-163, `x509.MarshalPKCS1PublicKey(pubkey)` at L143, record format `v=DKIM1; k=%s; p=%s` at L158.)*

### 3.8 User provisioning (OBSERVED)

Two accounts in the same local domain were created. This repository revision uses `maddyctl users create` — **not** the later-version `maddy creds` subcommand. Provisioning was run from the state directory so the CLI's relative `dsn all.db` resolves to the same database the daemon uses (§3.4). **The passwords are synthetic/disposable (§3.0);** `-p` is used only for a single reproducible line and would otherwise be prompted:

```
$ cd /tmp/maddy_run/state   # so relative dsn all.db matches the daemon (maddy.go:220 chdir to state)
$ maddyctl --config /tmp/maddy_run/maddy.conf users create alice@example.org -p AlicePass123
exit=0
$ maddyctl --config /tmp/maddy_run/maddy.conf users create bob@example.org -p BobPass123
exit=0
$ maddyctl --config /tmp/maddy_run/maddy.conf users list
alice@example.org
bob@example.org
exit=0
```

*(Source-grounded: `cmd/maddyctl/users.go` — `usersList`/`ListUsers` L14-15; `usersCreate` L30; flags `--null` → `CreateUserNoPass` L36-37, `--hash` L40, `--bcrypt-cost` L48; `CreateUser` L76.)* Credentials are checked at login by `internal/storage/sql/sql.go` `CheckPlain` (L377), which applies PRECIS `UsernameCaseMapped` to the username (L364) and PRECIS `OpaqueString` to the password (L386) before the backend performs the bcrypt comparison (L391). The domain-auth helper `internal/auth/auth.go` `CheckDomainAuth` (L5) splits the username on `@` and compares domains case-insensitively via `strings.EqualFold` (L25).

### 3.9 Test client and tooling (OBSERVED)

The test client is a **raw-socket** Python 3 script (`smtplib`'s higher-level helpers were bypassed so the exact protocol bytes — including the SASL PLAIN line and the `DATA` terminator — are captured verbatim). It:

- opens a TCP socket (wrapping it in TLS with `ssl.CERT_NONE` for the self-signed `465` endpoint; plain socket for `127.0.0.1:587`);
- logs **every** line as `C:` (client→server) or `S:` (server→client);
- authenticates by sending `AUTH PLAIN <base64(\0authcid\0passwd)>` directly;
- **decouples the envelope from the header** by sending `MAIL FROM:<envelope>` while writing an independent `From:` header (or omitting it, for T6) inside the `DATA` payload.

The scenario definitions (envelope vs `From` per T1–T6) are embedded in the script; the script is invoked as `python3 smtp_client.py <T#> <587|465>` and prints the complete transcript. The full transcripts it produced are in §5, and the script itself is a transient artifact removed at cleanup (§3.10). The container **lacks `swaks`, `tcpdump`, and the `sqlite3` CLI**; this raw-socket client, Maddy's `io_debug` (§5), Python's `sqlite3` module, and `maddyctl imap-msgs dump` (§6.3) substituted for them.

### 3.10 Isolation, cleanup, and final repository status

All transient state was created under `/tmp/maddy_run/` (outside the source tree) plus two host-side scripts (`/tmp/smtp_client.py`, `/tmp/dkim_verify.go`). Teardown stops **only** the daemon we spawned (by the recorded PID) and removes every transient artifact:

```
$ kill "$(cat /tmp/maddy_run/maddy.pid)"                 # stop only our daemon
$ rm -rf /tmp/maddy_run                                  # config, TLS, all.db, DKIM keys, captures, logs, scripts
$ rm -f /tmp/smtp_client.py /tmp/dkim_verify.go          # host-side observation scripts
```

After teardown the repository contains exactly one change — this document — and no whitespace defects:

```
$ git status --porcelain
 M blitzy/documentation/maddy_26452dd8dd78.md
$ git diff --check
$ echo exit=$?
exit=0
```

(The `git status`/`git diff --check` verification is re-confirmed at the delivery gate in §12.)

---


## 4. The test matrix (T1–T6): expected vs OBSERVED

All six scenarios authenticate as **alice**. Each was run **twice over plaintext port 587 and once over implicit-TLS port 465** (18 runs total), driven through the real submission endpoint, with **identical, stable, transport-independent results** across every run. The `Msg ID` column below lists the **Run A (587)** identifier as the representative; because each scenario ran three times, each produced three distinct message IDs — the complete per-run ledger (all 18 runs → transport → response → message ID → stored UID) is in §5.4.

| Test | MAIL FROM (envelope) | From header | Pipeline decision (OBSERVED) | DKIM outcome (OBSERVED) | Msg ID (Run A / §5.4) |
|------|----------------------|-------------|------------------------------|-------------------------|--------|
| **T1 cross-user** | `bob@example.org` | `bob@example.org` | **ACCEPTED** (matched by domain rule `example.org`) | **UNSIGNED** — `From` ≠ authenticated identity | `fddb4efa` |
| **T2 aligned** | `alice@example.org` | `alice@example.org` | **ACCEPTED** | **SIGNED** (`DKIM-Signature` present) | `f072eb77` |
| **T3 non-local envelope** | `x@remote.tld` | `x@remote.tld` | **REJECT `501 5.1.8`** (matched by default rule) | N/A (rejected before signing) | `ced400a6` |
| **T4 From≠envelope** | `alice@example.org` | `bob@example.org` | **ACCEPTED** | **UNSIGNED** — `From` ≠ envelope | `5712eab7` |
| **T5 From-domain≠key** | `alice@example.org` | `alice@other.tld` | **ACCEPTED** | **UNSIGNED** — `From` domain ≠ key domain | `aaa506ae` |
| **T6 missing From** | `alice@example.org` | (absent) | **REJECT `554 5.6.0`** at end-of-DATA | N/A | `7b8821d7` |

**The nuance to call out:** **T1 (cross-user) is ACCEPTED at the pipeline but delivered UNSIGNED.** The pipeline authorizes by domain (envelope `bob@example.org` is local) while the DKIM modifier declines to sign (`From` ≠ authenticated identity `alice`). These are two separate gates, and their decisions diverge for the very same message.

### 4.1 Verbatim SMTP response codes (client transcript, OBSERVED)

The following codes were identical across all sessions where applicable:

- EHLO advertised: `PIPELINING`, `8BITMIME`, `ENHANCEDSTATUSCODES`, `AUTH PLAIN`, `SMTPUTF8`, `SIZE 33554432`.
- AUTH success: `235 2.0.0 Authentication succeeded`
- `MAIL FROM` **always** returned `250 2.0.0 Roger, accepting mail from <ADDR>` — **including for `x@remote.tld`**. The sender reject is *deferred* to the first `RCPT`, because `defer_sender_reject` defaults to true (`internal/endpoint/smtp/smtp.go:567`).
- RCPT accept: `250 2.0.0 I'll make sure <alice@example.org> gets this`
- DATA go-ahead: `354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>`
- DATA accept: `250 2.0.0 OK: queued`
- `QUIT`: `221 2.0.0 Goodnight and good luck`.

(The `RSET` response was not exercised by the test client and is therefore not reported.) The **complete** protocol transactions from which these codes are excerpted appear verbatim in §5.1 (accepted) and §5.2 (rejected).

**T3 reject — issued at the first `RCPT TO` (deferred from `MAIL FROM`), Run A:**

```
501 5.1.8 Non-local sender domain (msg ID = ced400a6)
```

Python's `smtplib`-style clients raise `SMTPRecipientsRefused` for this; the raw-socket client observed it as the `S:` line after `RCPT TO`. The envelope domain `remote.tld` matches no `source` block, so it falls to `default_source`'s `reject`. The complete transaction is in §5.2. *(Source-grounded: `maddy.conf:117-118`; reject execution `internal/msgpipeline/msgpipeline.go:117-119`.)*

**T6 reject — issued at the end of `DATA`, Run A:**

```
554 5.6.0 Message does not contains a From header field (msg ID = 7b8821d7)
```

The verbatim server grammar is **"does not contains"** (reproduced exactly). The complete transaction is in §5.2. *(Source-grounded: the string is defined verbatim at `internal/endpoint/smtp/submission.go:43`, `Message: "Message does not contains a From header field"`, inside `submissionPrepare`'s missing-`From` check at L39-48, which returns SMTP code `554` with enhanced code `{5,6,0}`.)*

**Dependence on how the mismatch is constructed (O3).** The three "mismatch" scenarios produce three *different* outcomes, proving the decision depends on **which** identity is misaligned:

- **Envelope-domain mismatch (T3):** rejected `501 5.1.8` by the pipeline.
- **`From` ≠ envelope, but envelope local (T4):** accepted, delivered unsigned.
- **`From` ≠ authenticated identity, but envelope local (T1):** accepted, delivered unsigned.
- **`From`-domain ≠ signing-key domain (T5):** accepted, delivered unsigned.
- **Missing `From` entirely (T6):** rejected `554 5.6.0` by `submissionPrepare` (a *syntax/presence* check, unrelated to alignment).

---


## 5. Full SMTP transactions and per-run ledger (OBSERVED, O6)

Because implicit-TLS submission on port 465 is ciphertext on the wire and `tcpdump` is absent from the container, protocol-level visibility was obtained two complementary ways, together covering both an accepted and a rejected transaction **without TLS key extraction**: (1) the **raw-socket client transcript** (§3.9) that logs every `C:`/`S:` line, and (2) Maddy's server-side **`io_debug`** (raw server SMTP I/O; `internal/endpoint/smtp/smtp.go:565`), captured on the plaintext `587` listener. This is the documented fallback in the request: the encrypted `465` channel was captured through client-side and server-side logging plus a plaintext `587` listener, not by decrypting TLS. Every block below is reproduced **verbatim and complete** — greeting through `QUIT`.

The capture command for each transaction was `python3 smtp_client.py <T#> <587|465>` (§3.9); the server-side blocks are slices of `/tmp/maddy_run/maddy.log` for the same message ID. In every client block below, the leading `###`-prefixed line is **the client script's own emitted run-header** (the first line the tool writes to its transcript file, recording scenario/transport/auth/envelope/`From`), not a Markdown heading and not an editorial label inserted afterward — it is reproduced because these blocks are the complete, unedited transcript files. Likewise the inline `[base64 of …]`, `[end-of-DATA terminator]`, and `(msg ID = …)` annotations are printed by the client script itself and are therefore part of the captured bytes.

### 5.1 Complete ACCEPTED transaction — T2 aligned (`msg_id f072eb77`, plaintext 587)

Client transcript, greeting → `QUIT`, nothing elided (the `AUTH PLAIN` base64 encodes only the synthetic disposable password — §3.0):

```
### T2  transport=plaintext  127.0.0.1:587  auth=alice@example.org  MAIL FROM=<alice@example.org>  From:=alice@example.org
S: 220 example.org ESMTP Service Ready
C: EHLO test.client
S: 250-Hello test.client
S: 250-PIPELINING
S: 250-8BITMIME
S: 250-ENHANCEDSTATUSCODES
S: 250-AUTH PLAIN
S: 250-SMTPUTF8
S: 250 SIZE 33554432
C: AUTH PLAIN AGFsaWNlQGV4YW1wbGUub3JnAEFsaWNlUGFzczEyMw==   [base64 of \0alice@example.org\0AlicePass123]
S: 235 2.0.0 Authentication succeeded
C: MAIL FROM:<alice@example.org>
S: 250 2.0.0 Roger, accepting mail from <alice@example.org>
C: RCPT TO:<alice@example.org>
S: 250 2.0.0 I'll make sure <alice@example.org> gets this
C: DATA
S: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
C: From: alice@example.org
C: To: alice@example.org
C: Subject: T2 aligned
C: 
C: T2 aligned body
C: 
C: .   [end-of-DATA terminator]
S: 250 2.0.0 OK: queued
C: QUIT
S: 221 2.0.0 Goodnight and good luck
```

This is the accepted-and-**signed** case; its stored `DKIM-Signature` bytes are dissected in §6.3.

### 5.2 Complete REJECTED transactions

**T3 non-local envelope (`msg_id ced400a6`, plaintext 587) — rejected at the first `RCPT TO`.** `MAIL FROM` still gets `250` (the reject is *deferred*; `defer_sender_reject` defaults to true, `internal/endpoint/smtp/smtp.go:567`); the `501` lands at `RCPT`:

```
### T3  transport=plaintext  127.0.0.1:587  auth=alice@example.org  MAIL FROM=<x@remote.tld>  From:=x@remote.tld
S: 220 example.org ESMTP Service Ready
C: EHLO test.client
S: 250-Hello test.client
S: 250-PIPELINING
S: 250-8BITMIME
S: 250-ENHANCEDSTATUSCODES
S: 250-AUTH PLAIN
S: 250-SMTPUTF8
S: 250 SIZE 33554432
C: AUTH PLAIN AGFsaWNlQGV4YW1wbGUub3JnAEFsaWNlUGFzczEyMw==   [base64 of \0alice@example.org\0AlicePass123]
S: 235 2.0.0 Authentication succeeded
C: MAIL FROM:<x@remote.tld>
S: 250 2.0.0 Roger, accepting mail from <x@remote.tld>
C: RCPT TO:<alice@example.org>
S: 501 5.1.8 Non-local sender domain (msg ID = ced400a6)
C: QUIT
S: 221 2.0.0 Goodnight and good luck
```

**T6 missing `From` (`msg_id 7b8821d7`, plaintext 587) — rejected at the end of `DATA`.** Here envelope and recipient are both local, so `MAIL FROM`/`RCPT`/`DATA` all succeed; the `554` is raised by `submissionPrepare` only after the (`From`-less) message body is transmitted:

```
### T6  transport=plaintext  127.0.0.1:587  auth=alice@example.org  MAIL FROM=<alice@example.org>  From:=(absent)
S: 220 example.org ESMTP Service Ready
C: EHLO test.client
S: 250-Hello test.client
S: 250-PIPELINING
S: 250-8BITMIME
S: 250-ENHANCEDSTATUSCODES
S: 250-AUTH PLAIN
S: 250-SMTPUTF8
S: 250 SIZE 33554432
C: AUTH PLAIN AGFsaWNlQGV4YW1wbGUub3JnAEFsaWNlUGFzczEyMw==   [base64 of \0alice@example.org\0AlicePass123]
S: 235 2.0.0 Authentication succeeded
C: MAIL FROM:<alice@example.org>
S: 250 2.0.0 Roger, accepting mail from <alice@example.org>
C: RCPT TO:<alice@example.org>
S: 250 2.0.0 I'll make sure <alice@example.org> gets this
C: DATA
S: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
C: To: alice@example.org
C: Subject: T6 missing-From
C: 
C: T6 missing-from body
C: 
C: .   [end-of-DATA terminator]
S: 554 5.6.0 Message does not contains a From header field (msg ID = 7b8821d7)
C: QUIT
S: 221 2.0.0 Goodnight and good luck
```

The two rejects therefore occur at **different protocol stages** — `RCPT` (T3, envelope-domain authorization) versus end-of-`DATA` (T6, `From` presence/syntax) — a direct O3 observation that the outcome depends on *which* condition fails.

### 5.3 Server-side `io_debug` — the internal pipeline + DKIM decision (OBSERVED)

The client cannot see the pipeline's routing decision or the DKIM sign/skip decision; `io_debug` does. Below is the **complete, unedited** server-side `io_debug` slice for **T1 cross-user** (`msg_id fddb4efa`, 587) — the smoking gun for domain-level (not per-user) authorization. `io_debug` echoes each protocol line prefixed `submission:` and emits a blank line after each echo; structured log fields follow a tab. Reproduced exactly as emitted:

```
submission: 220 example.org ESMTP Service Ready

submission: EHLO test.client

submission: 250-Hello test.client

submission: 250-PIPELINING

submission: 250-8BITMIME

submission: 250-ENHANCEDSTATUSCODES

submission: 250-AUTH PLAIN

submission: 250-SMTPUTF8

submission: 250 SIZE 33554432

submission: AUTH PLAIN AGFsaWNlQGV4YW1wbGUub3JnAEFsaWNlUGFzczEyMw==

submission: 235 2.0.0 Authentication succeeded

submission: MAIL FROM:<bob@example.org>

submission: 250 2.0.0 Roger, accepting mail from <bob@example.org>

submission: RCPT TO:<alice@example.org>

submission: incoming message	{"msg_id":"fddb4efa","sender":"bob@example.org","src_host":"test.client","src_ip":"127.0.0.1:54834","username":"alice@example.org"}
[debug] smtp/pipeline: sender bob@example.org matched by domain rule 'example.org'	{"msg_id":"fddb4efa"}
[debug] smtp/pipeline: global rcpt modifiers: alice@example.org => alice@example.org	{"msg_id":"fddb4efa"}
[debug] smtp/pipeline: per-source rcpt modifiers: alice@example.org => alice@example.org	{"msg_id":"fddb4efa"}
[debug] smtp/pipeline: recipient alice@example.org matched by domain rule 'example.org'	{"msg_id":"fddb4efa"}
[debug] smtp/pipeline: per-rcpt modifiers: alice@example.org => alice@example.org	{"msg_id":"fddb4efa"}
[debug] smtp/pipeline: tgt.Start(bob@example.org) ok, target = sql:local_mailboxes	{"msg_id":"fddb4efa"}
submission: RCPT ok	{"msg_id":"fddb4efa","rcpt":"alice@example.org"}
submission: 250 2.0.0 I'll make sure <alice@example.org> gets this

submission: DATA

submission: 354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>

submission: From: bob@example.org

submission: To: alice@example.org
Subject: T1 cross-user

T1 cross-user body

.

submission: adding missing Message-ID	
submission: adding missing Date header	
sign_dkim: not signing, From address is not authenticated identity	{"auth_id":"alice@example.org","from_addr":"bob@example.org","msg_id":"fddb4efa"}
[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"fddb4efa"}
submission: accepted	{"msg_id":"fddb4efa"}
submission: 250 2.0.0 OK: queued

[debug] submission: reset	
submission: QUIT

submission: 221 2.0.0 Goodnight and good luck

```

**What this proves.** The `AUTH PLAIN` payload base64-decodes to `\0alice@example.org\0AlicePass123` (SASL PLAIN is `authzid\0authcid\0passwd`), so the authenticated identity is **alice**. Yet the `incoming message` line records `"sender":"bob@example.org"` and `"username":"alice@example.org"` as **separate** fields, and only the **domain** of the sender gates acceptance: `sender bob@example.org matched by domain rule 'example.org'`. The authenticated `username` is logged but **not consulted** by source routing. The message is then `accepted` and `250 2.0.0 OK: queued` — and, per the immediately preceding `sign_dkim: not signing, From address is not authenticated identity` line, delivered **UNSIGNED**.

The complete server-side slice for the **T3 reject** (`msg_id ced400a6`) shows the corresponding internal path — `matched by default rule` then the reject directive — again with the full `incoming message` fields (no elision):

```
submission: MAIL FROM:<x@remote.tld>

submission: 250 2.0.0 Roger, accepting mail from <x@remote.tld>

submission: RCPT TO:<alice@example.org>

submission: incoming message	{"msg_id":"ced400a6","sender":"x@remote.tld","src_host":"test.client","src_ip":"127.0.0.1:54838","username":"alice@example.org"}
[debug] smtp/pipeline: sender x@remote.tld matched by default rule	{"msg_id":"ced400a6"}
[debug] smtp/pipeline: global rcpt modifiers: alice@example.org => alice@example.org	{"msg_id":"ced400a6"}
[debug] smtp/pipeline: per-source rcpt modifiers: alice@example.org => alice@example.org	{"msg_id":"ced400a6"}
[debug] smtp/pipeline: recipient alice@example.org matched by default rule (clean = alice@example.org)	{"msg_id":"ced400a6"}
submission: RCPT error	{"effective_rcpt":"alice@example.org","rcpt":"alice@example.org","reason":"reject directive used","smtp_code":501,"smtp_enchcode":"5.1.8","smtp_msg":"Non-local sender domain"}
submission: 501 5.1.8 Non-local sender domain (msg ID = ced400a6)

submission: QUIT

submission: 221 2.0.0 Goodnight and good luck

submission: aborted	{"msg_id":"ced400a6"}
```

**Mapping the debug strings to source.** The pipeline's routing decision is emitted by `internal/msgpipeline/msgpipeline.go` `srcBlockForAddr` (defined L155, called L113):

- `"sender %s matched by domain rule '%s'"` — the `perSource[domain]` path (match at L190, log at L196). **This is the T1 line.**
- `"sender %s matched by default rule"` — the `defaultSource` path (fallback at L193, log at L194). **This is the T3 line.**
- `"sender %s matched by address rule '%s'"` — the exact `perSource[cleanFrom]` path (match at L171, log at L199). *(Not triggered here because the canonical config registers a domain-level `source`, not per-address blocks.)*
- The reject itself, reproduced verbatim from source at L117-119: `if sourceBlock.rejectErr != nil { dd.log.Debugf("sender %s rejected with error: %v", mailFrom, sourceBlock.rejectErr); return sourceBlock.rejectErr }`.

### 5.4 Per-run ledger and reset/isolation (OBSERVED, R4)

**Reset/isolation.** Before the reported runs, the database was reset to a known-clean state for auditability: the daemon was stopped, `all.db` and its `messages` external-blob directory were wiped (the `dkim_keys` directory was **kept** so the signing key stays identical to the published `.dns` in §3.7), the two accounts were re-provisioned (§3.8), and the daemon was restarted. All accepted messages therefore land in a fresh `alice` `INBOX` with **monotonically increasing UIDs in delivery order** (go-imap-sql assigns UIDs sequentially per mailbox), which is what makes the message-ID → UID mapping below unambiguous. Each scenario was then run three times: **Run A** and **Run B** over plaintext `587`, **Run C** over implicit-TLS `465`.

Every one of the 18 runs produced the identical decision and SMTP response for its scenario; only the per-message IDs (and, for signed T2, the `b=`/`t=`/`x=` signature values) differ run-to-run, exactly as expected for freshly generated messages. Full ledger:

| Scenario | Decision + final SMTP response (all runs identical) | Run A (587) msg ID → UID | Run B (587) msg ID → UID | Run C (465) msg ID → UID |
|----------|------------------------------------------------------|--------------------------|--------------------------|--------------------------|
| **T1** cross-user | ACCEPTED, UNSIGNED · `250 2.0.0 OK: queued` | `fddb4efa` → UID 1 | `c2b92945` → UID 5 | `44a93bfa` → UID 9 |
| **T2** aligned | ACCEPTED, **SIGNED** · `250 2.0.0 OK: queued` | `f072eb77` → UID 2 | `bea89484` → UID 6 | `3aab5df1` → UID 10 |
| **T3** non-local | **REJECT** at `RCPT` · `501 5.1.8 Non-local sender domain` | `ced400a6` → (not delivered) | `adeb92e9` → (not delivered) | `6a4f62fb` → (not delivered) |
| **T4** From≠envelope | ACCEPTED, UNSIGNED · `250 2.0.0 OK: queued` | `5712eab7` → UID 3 | `1ac8cf44` → UID 7 | `6b7c9146` → UID 11 |
| **T5** From-domain≠key | ACCEPTED, UNSIGNED · `250 2.0.0 OK: queued` | `aaa506ae` → UID 4 | `76a75aa9` → UID 8 | `b7e6b48c` → UID 12 |
| **T6** missing From | **REJECT** at end-of-`DATA` · `554 5.6.0 Message does not contains a From header field` | `7b8821d7` → (not delivered) | `ecad9445` → (not delivered) | `9c62b482` → (not delivered) |

The 12 accepted messages occupy `alice` `INBOX` UIDs 1–12 in exact delivery order (T1,T2,T4,T5 for Run A → UIDs 1–4; the same four for Run B → 5–8; for Run C → 9–12). The two rejected scenarios (T3, T6) never reach storage, so they consume no UID. Mailbox enumeration confirming this map is in §7; the stored bytes for the Run A representatives (UIDs 1–4) are dissected in §6 and §7. **Result: the accept/reject decision and DKIM sign/skip outcome are stable and transport-independent across all 18 runs.**

---


## 6. Mechanism (b): DKIM signing behavior (OBSERVED, O4/O5)

### 6.1 Sign/skip decision per scenario — raw `io_debug` log lines (OBSERVED)

The `sign_dkim` modifier logs its decision for every message. The four decision lines below are reproduced **exactly as emitted** by the daemon (Run A, sliced from `/tmp/maddy_run/maddy.log`); each scenario label is kept **outside** the fenced output so the fenced bytes are untouched raw log — the tab before the structured JSON fields object and the `[debug]` prefix on the *signed* line are part of those bytes. (The same lines appear in context inside the §5.3 server-side slice.)

**T1 cross-user (`fddb4efa`) — skips, auth-identity method:**

```
sign_dkim: not signing, From address is not authenticated identity	{"auth_id":"alice@example.org","from_addr":"bob@example.org","msg_id":"fddb4efa"}
```

**T2 aligned (`f072eb77`) — signs:**

```
[debug] sign_dkim: signed	{"identifier":"alice@example.org"}
```

**T4 `From`≠envelope (`5712eab7`) — skips, envelope method:**

```
sign_dkim: not signing, From address is not envelope address	{"envelope":"alice@example.org","from_addr":"bob@example.org","msg_id":"5712eab7"}
```

**T5 `From`-domain≠key (`aaa506ae`) — skips, key-domain check:**

```
sign_dkim: not signing, From domain is not key domain	{"from_domain":"other.tld","key_domain":"example.org","msg_id":"aaa506ae"}
```

Each maps precisely to a branch of `shouldSign` in `internal/modify/dkim/dkim.go` (L249). The `require_sender_match` directive defaults to `["envelope", "auth"]` (L151-152; the parser's allowed set is `{envelope, auth_domain, auth_user, off}` — the default-vs-allowed inconsistency is dissected in §6.5), so both the envelope and auth checks are active:

- **T5 — `From`-domain ≠ key-domain** (`if !dns.Equal(fromDomain, m.domain)` at L287, log L288). Checked first among the alignment checks; `other.tld` ≠ `example.org`, so it skips here with `from_domain`/`key_domain` fields.
- **T4 — envelope method, `From` ≠ `MAIL FROM`** (`if _, do := m.senderMatch["envelope"]; do && !address.Equal(fromAddr, mailFrom)` at L293, log L294). `From` `bob@example.org` ≠ envelope `alice@example.org`, so it skips with `from_addr`/`envelope` fields.
- **T1 — auth method, `From` ≠ authenticated identity** (auth block L299-310). Here the domain check passes (`example.org`) and the envelope check passes (`From` `bob@example.org` = envelope `bob@example.org`), so evaluation reaches the auth check: `compareWith` is built from `From`, and because the authorization identity `alice@example.org` contains `@`, `compareWith` is set to `address.ForLookup(fromAddr)` = `bob@example.org` (L302-303); `strings.EqualFold("bob@example.org","alice@example.org")` at L305 is false, so it skips (log L306) with `from_addr`/`auth_id` fields.
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

The stored message is read with `maddyctl imap-msgs dump`. The valid syntax at this commit is `maddyctl imap-msgs dump [command options] USERNAME MAILBOX SEQ` (usage string emitted by the binary; implemented at `cmd/maddyctl/imap.go:403`, wired at `cmd/maddyctl/main.go:534`), with `--uid`/`-u` to interpret `SEQ` as a UID. Because the config's `dsn all.db` is relative to the `state` directory, the command is run from there. Exact command and its **complete, unedited** captured output for T2 (alice `INBOX`, UID 2, `msg_id f072eb77`):

```
$ cd /tmp/maddy_run/state
$ /tmp/maddy_run/maddyctl --config /tmp/maddy_run/maddy.conf imap-msgs dump --uid alice@example.org INBOX 2
Delivered-To: alice@example.org
Return-Path: <alice@example.org>
Dkim-Signature: a=rsa-sha256;
 bh=QqRw2xeTmXHtQwwhf50wtFh4KN3B/NnkSFhmKCsO4AM=; c=relaxed/relaxed;
 d=example.org;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=alice@example.org; s=default; t=1784162779; v=1; x=1784594779; b=TgSjJfFelcwbSkyD4g3OcthLG49gfO4lgcdkJ07Z7d5hrs4p6jMx0vaxJEM0MIteROGTvDz+S4EyKeLWB2rModk0kBdZhrXdWZlQ0T/dkg9BnLxAn7+H8MfYyWk3IfxhAeQYGo3bpEzJ4Mzy7Bt6ooAHRnui6YCfedzh7nEJckkMJyLEJOir40hp7zVmErKQgKJcjalzCAzZMZRr2P5z5yE57LuxyhLF7CRlIgFWQDayo3DewcaMaUCBZBEWa3b6xpNO2zMBMv92rnJMrFYzECiGmjmXoal2N5pNJxvL29T6j9NkXv0apjWh2CkB8OKiv79RJiUXRUV+dJAl7ErntA==;
Received:  by example.org (envelope-sender <alice@example.org>) with ESMTP
 id f072eb77; Thu, 16 Jul 2026 00:46:19 +0000
Date: Thu, 16 Jul 2026 00:46:19 +0000
Message-Id: <89e95e46-1286-4a3e-a508-8ec7a72b2b12@example.org>
From: alice@example.org
To: alice@example.org
Subject: T2 aligned

T2 aligned body
```

This stored blob is **1093 bytes**, `sha256 = af3247ae5c58a9833ebb244c7458c57fd3d6e3049a60ff4d45d2a1eecb5f376e`. Re-running the exact command above a second time produced byte-identical output (same sha256), and the same bytes were cross-checked against the go-imap-sql external body blob read directly from `all.db` (§9 Finding C uses this same 1093-byte blob). The header lines are CRLF-terminated (14 `CR` bytes); the final body line is stored LF-only, exactly as dumped.

**`h=` tag analysis (byte-exact, O5).** `From` appears **twice** — the substring `From:From` is present within the `h=` list shown above — it is **oversigned**. So are `Subject:Subject`, `To:To`, `Date:Date`, and `Message-Id:Message-Id`. **Oversigning** means listing a header field one more time than it actually occurs: this both binds the present copy *and* prevents an intermediary from silently adding a second copy without breaking the signature. Therefore the signature **does cover `From`** (in fact it oversigns it). The oversign field set is `oversignDefault` in `internal/modify/dkim/dkim.go` (L31-54; `From` at L37), and the "one entry per occurrence, plus one more" logic is `fieldsToSign` (L202-233: for each oversigned key it appends one entry per occurrence present in the message, L215-217, then one extra, L219). Consequently a header present once in the message (Subject, To, From, Date, Message-Id) appears **twice** in `h=`, while an oversigned header absent from the message (Sender, Cc, MIME-Version, Content-Type, Content-Transfer-Encoding, Reply-To, In-Reply-To, References, Autocrypt, Openpgp) appears **once**. The observed `h=` string is exactly this derivation — confirming T2's message carried `Subject`, `To`, `From`, `Date`, `Message-Id` (each once) and none of the sign-only `List-*`/`Resent-*` fields (`signDefault`, L55-72).

**Timestamps.** `t=1784162779`, `x=1784594779`, so `x − t = 432000` seconds = **5 days = 120h**, matching the documented `sig_expiry` default of `120h` (`docs/man/maddy-filters.5.scd:500-501`) and the code default `5*Day` (`internal/modify/dkim/dkim.go:146`). `c=relaxed/relaxed` matches the `header_canon`/`body_canon` defaults (L140-145) and `a=rsa-sha256` matches the `hash` default `sha256` (L147-148) with the RSA key. `bh=QqRw2xeTmXHtQwwhf50wtFh4KN3B/NnkSFhmKCsO4AM=` is the body hash; §9 Finding B independently reproduces this exact `bh` from the same body.

### 6.4 The UNSIGNED messages (OBSERVED)

The three misaligned Run-A deliveries each have **no `DKIM-Signature` header at all** in storage: T1 (alice `INBOX` UID 1, `msg_id fddb4efa`), T4 (alice `INBOX` UID 3, `msg_id 5712eab7`), and T5 (alice `INBOX` UID 4, `msg_id aaa506ae`). (Their stored bytes are dumped in full in §7.) T2 (the only signed accepted case) is UID 2. This is the observed consequence of the §6.2 skip path (`RewriteBody` `return nil`), and it is confirmed by the negative grep in §8, whose header-field enumeration shows `Dkim-Signature` present in exactly the signed messages and absent from every skipped one.

---

### 6.5 The `require_sender_match` parser/default/implementation inconsistency (OBSERVED + source-grounded)

`§6.1` forward-referenced this dissection. The `require_sender_match` directive has a **three-way inconsistency** among (a) the values the config parser *accepts*, (b) the built-in *default*, and (c) the values `shouldSign` actually *honors*. All three were pinned to source and the parser behavior was **proven at runtime** (not inferred).

**(a) What the parser accepts.** The directive is parsed by an `EnumList` whose allowed set is `{envelope, auth_domain, auth_user, off}`, with a built-in default of `{envelope, auth}`:

```go
// internal/modify/dkim/dkim.go:151-152
cfg.EnumList("require_sender_match", false, false,
    []string{"envelope", "auth_domain", "auth_user", "off"}, []string{"envelope", "auth"}, &senderMatch)
```

Note the built-in default `[]string{"envelope", "auth"}` (the 5th argument) contains the token **`auth`, which is NOT a member of the allowed set** (the 4th argument). A default supplied programmatically bypasses the `EnumList` validation; a value *typed by an operator* does not.

**(b) What `shouldSign` honors.** `shouldSign` tests the resulting map for two **literal** keys only — `"envelope"` and `"auth"` — never `"auth_domain"` or `"auth_user"`:

```go
// internal/modify/dkim/dkim.go:293  (envelope method)
if _, do := m.senderMatch["envelope"]; do && !address.Equal(fromAddr, mailFrom) {
// internal/modify/dkim/dkim.go:299  (auth method)
if _, do := m.senderMatch["auth"]; do {
```

So `auth_domain` and `auth_user` are **parse-accepted but dead**: they can be written into `maddy.conf` without error, yet no branch of `shouldSign` ever consults them, and they silently disable both alignment checks (the map ends up containing neither `"envelope"` nor `"auth"`).

**(c) Runtime proof (OBSERVED).** Each explicit value was written into a throwaway `sign_dkim $(primary_domain) default { require_sender_match <value> }` block (with `state` pointed at a scratch dir) and fed to the **real daemon**. A parse error surfaces during module init **before** any endpoint binds; a value that *parses* proceeds past DKIM init and then fails only on `bind: address already in use`, because the real daemon from §3.6 still held `:465`. The per-variant command was:

```
timeout 6 /tmp/maddy_run/maddy -config <variant>.conf 2>&1 | grep -iE 'invalid argument|keypair|listen'
```

driven by a loop that prints a `===== require_sender_match <value> =====` header before each. The complete, unedited output follows (two batches — the first covering the multi-token default and the three other allowed tokens, the second added afterward to cover the single tokens `auth` and `envelope`). Each `invalid argument` / `listen` line is emitted with a trailing TAB — Maddy's empty log-fields separator — trimmed for display:

```
===== require_sender_match envelope auth =====
/tmp/rsm_test/rsm_envelope_auth.conf:38: invalid argument, valid values are: [envelope auth_domain auth_user off]
(if 'listen'/'starting' appears => parsed OK; an error line => parse rejected)

===== require_sender_match auth_domain =====
submission: listen tcp 0.0.0.0:465: bind: address already in use
(if 'listen'/'starting' appears => parsed OK; an error line => parse rejected)

===== require_sender_match auth_user =====
submission: listen tcp 0.0.0.0:465: bind: address already in use
(if 'listen'/'starting' appears => parsed OK; an error line => parse rejected)

===== require_sender_match off =====
submission: listen tcp 0.0.0.0:465: bind: address already in use
(if 'listen'/'starting' appears => parsed OK; an error line => parse rejected)
```

```
===== require_sender_match auth =====
/tmp/rsm_test/rsm2_auth.conf:38: invalid argument, valid values are: [envelope auth_domain auth_user off]

===== require_sender_match envelope =====
submission: listen tcp 0.0.0.0:465: bind: address already in use
```

The two parse-**rejected** values are the multi-token default `envelope auth` and the single token `auth` — both fail identically with `invalid argument, valid values are: [envelope auth_domain auth_user off]` because `auth` is not in the allowed set. The four values that reach `bind: address already in use` (`auth_domain`, `auth_user`, `off`, `envelope`) all **parsed successfully**. (On a *cold* scratch state dir the parse-OK values additionally emit `sign_dkim: generating a new rsa2048 keypair...` immediately before the bind error, exactly as in §3.7; the output above is from a *warm* scratch dir where the key already existed, so the daemon proceeds straight to the bind attempt.)

The resulting truth table (all rows OBSERVED for the parse column; the honored column is source-grounded to `dkim.go:293/299` and corroborated by the §6.1 observations):

| Explicit `require_sender_match` value | Parses? (OBSERVED) | Honored by `shouldSign`? |
|---------------------------------------|--------------------|--------------------------|
| `envelope` | **Yes** | **Yes** — tested at `dkim.go:293` |
| `auth` | **No** — `invalid argument, valid values are: [envelope auth_domain auth_user off]` | (token tested at L299, but it is un-typeable) |
| `auth_domain` | **Yes** | **No — dead** (`shouldSign` never tests `"auth_domain"`) |
| `auth_user` | **Yes** | **No — dead** |
| `off` | **Yes** | **Yes** — short-circuits at `dkim.go:250-262` |
| `envelope auth` (== the built-in default) | **No** — enum rejects `auth` | default is honored only because it is set programmatically, bypassing `EnumList` validation |

**Net (the divergence).** The active default `{envelope, auth}` is the pair the implementation honors, and it works **only** because it is injected as the `EnumList` default (which skips validation). The moment an operator tries to *write that same default explicitly*, the daemon **refuses to start** with `invalid argument`. The token the implementation actually keys on for the auth check — `auth` — is un-typeable in config; the two tokens that *are* typeable in its place — `auth_domain`, `auth_user` — are dead no-ops that silently turn the check off. Only `envelope` and `off` are simultaneously typeable and honored. The upstream `sign_dkim` manual compounds the trap by documenting the *Default* as `envelope auth` (`docs/man/maddy-filters.5.scd` L518-519) — i.e., it documents a value that cannot be entered. This is a second, config-parser-level instance of "behavior diverges from a reasonable configuration reading" (companion to the §11.3 sign/skip case and the §9 Finding A crypto case).

---


## 7. Raw stored `Received` header bytes (OBSERVED, O4) — reflects the ENVELOPE, not `From`/auth

The `Received` header Maddy adds to a submitted message records the **envelope sender**, not the `From` header and not the authenticated user. Every stored message below is the **complete, unedited** output of `maddyctl imap-msgs dump --uid alice@example.org INBOX N` (run from `/tmp/maddy_run/state`, syntax per §6.3), where `N` is the concrete UID named in each block's heading (`1` for T1, `3` for T4).

**The decisive case — T1 (alice `INBOX` UID 1, `msg_id fddb4efa`).** This is the only scenario where the envelope sender (`bob`), the `From` header (`bob`), and the **authenticated identity** (`alice`) do not all coincide — so it is the case that can distinguish "tracks envelope" from "tracks authenticated user." Complete stored bytes:

```
Delivered-To: alice@example.org
Return-Path: <bob@example.org>
Received:  by example.org (envelope-sender <bob@example.org>) with ESMTP id
 fddb4efa; Thu, 16 Jul 2026 00:46:18 +0000
Date: Thu, 16 Jul 2026 00:46:18 +0000
Message-Id: <112955e3-f2d5-4599-83e0-f42bab0d3efe@example.org>
From: bob@example.org
To: alice@example.org
Subject: T1 cross-user

T1 cross-user body
```

The client authenticated as **alice** (§5.3 shows `"username":"alice@example.org"` for this exact `msg_id`), yet the `Received` clause is `(envelope-sender <bob@example.org>)` and `Return-Path` is `<bob@example.org>`. The header therefore follows the **envelope `MAIL FROM` (`bob`)**, definitively **not** the authenticated identity (`alice`). This is the raw T1 evidence that isolates envelope-vs-auth; the `smtp.go`/`received.go` source walk in §7.1 explains why.

**T4 (alice `INBOX` UID 3, `msg_id 5712eab7`) — envelope `alice`, `From` `bob`.** Complete stored bytes (full RFC1123Z date, nothing abbreviated):

```
Delivered-To: alice@example.org
Return-Path: <alice@example.org>
Received:  by example.org (envelope-sender <alice@example.org>) with ESMTP
 id 5712eab7; Thu, 16 Jul 2026 00:46:21 +0000
Date: Thu, 16 Jul 2026 00:46:21 +0000
Message-Id: <94bb0089-f6bf-4d10-9204-ab3bc74a94cc@example.org>
From: bob@example.org
To: alice@example.org
Subject: T4 From-not-envelope

T4 from-ne-envelope body
```

Here the `(envelope-sender <alice@example.org>)` clause reflects `MAIL FROM` (`alice`) even though the message's `From` is `bob@example.org` — so `Received` is keyed to the **envelope, not `From`**. T2's signed `Received` (byte-exact in §6.3) and T5's (UID 4, envelope `alice`, `From` `alice@other.tld`) show the identical generated format.

**A byte-level detail worth noting:** T1's `Received` folds *after* `id` (`…with ESMTP id` ⏎ ` fddb4efa;…`) while T2/T4/T5 fold *before* `id` (`…with ESMTP` ⏎ ` id …`). This is purely a function of first-line length — T1's `(envelope-sender <bob@example.org>)` is two characters shorter than the `alice@example.org` variants, so `id` still fits before the wrap. It confirms these are genuinely distinct captured messages, not copies of one blob. In every case, note the **double space** after `Received:`, the `(envelope-sender <…>)` clause, and the **absence** of any `from <host> [ip]` sender trace.

### 7.1 Why the double space, and why no sender trace — source-grounded

`internal/target/received.go` `GenerateReceived` (L19) builds the value:

- It requires a connection (`msgMeta.Conn == nil` guard, L20-22).
- The `from <host> [ip]` sender-trace clause is emitted **only** when `!msgMeta.DontTraceSender && (proto contains SMTP or LMTP)` (L30-31). On submission, `submissionPrepare` sets `msgMeta.DontTraceSender = true` (`internal/endpoint/smtp/submission.go:28`), so this clause is **skipped entirely**.
- With the sender trace skipped, the builder's first written token is `" by "` (with a **leading space**) + our hostname (L60-64). Because a header is serialized as `Received:` + value, and the value begins with that leading space, the stored bytes show `Received:` followed by **two** spaces before `by` — exactly the observed double space.
- Then `" (envelope-sender <"` + `MAIL FROM` + `">)"` (L66-72) — this is the clause that carries the envelope address.
- Then `" with "` + protocol + `" id "` + message id (L74-82), and finally the RFC1123Z date (L84).

The endpoint wires this up: it captures the envelope as `msgMeta.OriginalFrom = cleanFrom` (`internal/endpoint/smtp/smtp.go:116`), generates the header via `target.GenerateReceived(ctx, s.msgMeta, s.endp.hostname, s.msgMeta.OriginalFrom)` (`smtp.go:303`) and adds it with `header.Add("Received", received)` (`smtp.go:307`). The authenticated identity is stored separately as `AuthUser` (`smtp.go:680`) and is not passed into `GenerateReceived`. Note also that the `for` recipient clause is never included in the `Received` field by design (`docs/internals/quirks.md:9`).

---

## 8. Negative finding: no `Authentication-Results` on submission under the tested configuration (OBSERVED, O4)

Exact negative search over all 12 stored messages, run in the evidence directory, with its complete output:

```
$ grep -iE 'Authentication-Results|ARC-|Received-SPF|DKIM-Status' stored_uid*.txt ; echo exit=$?
exit=1
```

`grep` printed **no matching lines** and exited `1` (its documented "no lines selected" status), so **none** of the 12 delivered submission messages carries an `Authentication-Results`, `ARC-*`, `Received-SPF`, or `DKIM-Status` header. The **complete** set of header field names present across all 12 stored messages (start-of-line `Name:` tokens; folded continuation lines excluded) is exactly:

```
$ grep -hE '^[A-Za-z][A-Za-z0-9-]*:' stored_uid*.txt | sed -E 's/^([A-Za-z0-9-]+):.*/\1/' | sort -u | tr '\n' ' '
Date Delivered-To Dkim-Signature From Message-Id Received Return-Path Subject To
```

So under this configuration the only headers Maddy itself adds on submission are:

- `Received` — always (`internal/target/received.go`).
- `DKIM-Signature` — only when the sender is aligned (`internal/modify/dkim/dkim.go:406`); present on the signed messages and absent from the skipped ones (§6.4), consistent with the field enumeration above showing `Dkim-Signature` at all only because the signed T2-type messages are in the set.
- Injected `Message-Id` and `Date` when absent — logged as `adding missing Message-ID` / `adding missing Date header` (`internal/endpoint/smtp/submission.go` L30-37 and L111-127).
- `Delivered-To` and `Return-Path` — added by the local-delivery target.

**Scope of this negative result (correcting an overgeneralization).** The absence of `Authentication-Results` is a property of **this tested, default-like submission configuration**, not an invariant of "submission" as such, and it is **not** produced by an inbound-only code path. The header is added by the **common** message-pipeline delivery path: `msgpipelineDelivery.Body` calls `dd.checkRunner.applyResults(dd.d.Hostname, &header)` at `internal/msgpipeline/msgpipeline.go:316`, and `applyResults` (`internal/msgpipeline/check_runner.go:262`) adds the header at **L302** — `header.Add("Authentication-Results", authres.Format(hostname, cr.mergedRes.AuthResult))` — but **only** when the guard at **L301**, `if len(cr.mergedRes.AuthResult) != 0`, holds. That `AuthResult` slice is populated by configured checks (e.g. `verify_dkim`/`apply_spf`, wired for the port-25 endpoint at `maddy.conf:54-66`). The tested submission block declares **no** result-producing checks, so `AuthResult` is empty and the header is correctly omitted. An administrator who added such checks to the submission block **would** see the header — and, per the comment at `msgpipeline.go:320-321`, modifiers (including `sign_dkim`) run *after* `applyResults`, so a subsequent DKIM signature would cover it. The observed absence is therefore a genuine negative result **for the tested configuration**, and it refutes a naive expectation that submitted mail is always annotated — **without** claiming Maddy can *never* stamp `Authentication-Results` on submission.

---


## 9. DKIM triggering under `From` manipulation + what a verifying recipient would conclude (OBSERVED, O5/O8)

§4–§6 already establish the triggering behavior under `From` manipulation: T1 (`From` = cross-user), T4 (`From` ≠ envelope), and T5 (`From`-domain ≠ key-domain) are each **accepted and delivered UNSIGNED** — a manipulated `From` never causes a rejection, only the absence of a signature; and for the aligned T2 the signature **covers `From`** (oversigned). This section reports what a recipient attempting cryptographic verification would actually conclude.

Verification used the **same library Maddy uses** — `github.com/emersion/go-msgauth v0.3.2-0.20191028231513-55b75676976c` — driving the real `dkim.Verify`, with the DKIM DNS record served locally by `github.com/foxcpp/go-mockdns v0.0.0-20191123143003-02edb10da1e3` (Maddy's own test-DNS dependency). The verifier was a small standalone program (`verify_main.go`) built inside the Maddy module tree so it links the **identical pinned** versions from `go.mod`; it was retained as a publishable synthetic artifact. Its core does exactly this:

```go
// verify_main.go (key operations; full program retained alongside the evidence)
priv := loadPriv(keyPath)                              // PKCS#8 .key that Maddy generated
pkcs1 := x509.MarshalPKCS1PublicKey(&priv.PublicKey)   // the form Maddy PUBLISHES (keys.go:143)
pkix, _ := x509.MarshalPKIXPublicKey(&priv.PublicKey)  // the form go-msgauth EXPECTS (query.go:110)
// FINDING A (direct): the exact parser call go-msgauth makes on the p= blob
_, e1 := x509.ParsePKIXPublicKey(pubDER)               // pubDER = base64-decode of published p=
// FINDING A (end-to-end): real dkim.Verify against the PUBLISHED (PKCS#1) record
report("A", verifyWithRecord(rec, msg))                // msg = stored T2 blob
// FINDING B: fresh in-library Sign then Verify with a PKIX record
report("B", verifyWithRecord(pkixRec, freshlySigned))
// FINDING C: re-verify the STORED mailbox copy with a PKIX record (so parsing succeeds)
report("C", verifyWithRecord(pkixRec, msg))
```

`verifyWithRecord` publishes `record` as the TXT for `default._domainkey.example.org.` via `mockdns.NewServer` + `PatchNet`, then calls `dkim.Verify(bytes.NewReader(msg))` — i.e. the genuine library entry point, not a re-implementation. Running the built binary produced the following **complete, unedited** output (the whole capture, `dkim_verify.txt`, `sha256 = 15316cc5658c3cd418f534a6baed00a105471ce70b51d24af8afa2e5b9f8abe4`):

```
$ ./dkimverify
== dependency versions ==
go-msgauth v0.3.2-0.20191028231513-55b75676976c ; go-mockdns v0.0.0-20191123143003-02edb10da1e3

== key fingerprints (sha256 of DER) ==
PKCS#1 public DER: sha256=4d791f05b029aed4f9e98610a0b99f649d129b005ef01b6e390412aeb59cf1f1 prefix=30 82 01 0a 02 82 01 01 00
PKIX   public DER: sha256=f03b70796d6d8d069b36d8272a501ca95d647982bf01a069dd57cb22b4afd79c prefix=30 82 01 22 30 0d 06 09 2a 86 48 86

== published record p= DER analysis ==
published p= decodes to DER prefix: 30 82 01 0a 02 82 01 01 00
published p= bytes == MarshalPKCS1PublicKey(pub)? true

== FINDING A (direct parser test — the exact call at go-msgauth dkim/query.go:110) ==
x509.ParsePKIXPublicKey(published p=) -> ERROR: x509: failed to parse public key (use ParsePKCS1PublicKey instead for this key format)
x509.ParsePKCS1PublicKey(published p=) -> OK (confirms it IS a bare PKCS#1 key)

stored T2 blob: sha256=af3247ae5c58a9833ebb244c7458c57fd3d6e3049a60ff4d45d2a1eecb5f376e size=1093

== FINDING A (end-to-end — real go-msgauth dkim.Verify, published PKCS#1 record) ==
A: FAIL domain=example.org err=dkim: key syntax error: x509: failed to parse public key (use ParsePKCS1PublicKey instead for this key format)

== FINDING B (scheme soundness — fresh in-library sign/verify round-trip) ==
fresh signature bh   = QqRw2xeTmXHtQwwhf50wtFh4KN3B/NnkSFhmKCsO4AM=
stored T2 signature bh = QqRw2xeTmXHtQwwhf50wtFh4KN3B/NnkSFhmKCsO4AM=
body hashes match (same body -> same bh)? true
B: PASS domain=example.org identifier=alice@example.org

== FINDING C (stored mailbox copy vs its own signature — PKIX record so parsing succeeds) ==
C: FAIL domain=example.org err=dkim: signature did not verify: crypto/rsa: verification error
```

Three findings, each labeled **OBSERVED** with its source grounding, all traceable to specific lines of the output above:

### Finding A (OBSERVED) — the published public key is PKCS#1 but the verifier expects PKIX (a real interop divergence at this commit)

Maddy's auto-generated `.dns` record's `p=` value base64-decodes to DER beginning `30 82 01 0a 02 82 01 01 00` (see "published p= DER analysis" in the output) — a **bare PKCS#1 `RSAPublicKey`**. The verifier confirms this two ways: the published `p=` bytes are **exactly** `MarshalPKCS1PublicKey(pub)` (`published p= bytes == MarshalPKCS1PublicKey(pub)? true`), and its `sha256 = 4d791f05…cf1f1` matches the PKCS#1 fingerprint, not the PKIX one (`sha256 = f03b7079…fd79c`, prefix `30 82 01 22 30 0d 06 09 2a 86 48 86`). This is because `internal/modify/dkim/keys.go:143` encodes the public key with `x509.MarshalPKCS1PublicKey(pubkey)` (inside `writeDNSRecord`, L136-163) and emits it into the record at L158 (`v=DKIM1; k=%s; p=%s`).

The go-msgauth verifier, however, parses the `p=` blob with **`x509.ParsePKIXPublicKey(b)`** — this is the exact call at `github.com/emersion/go-msgauth@v0.3.2-0.20191028231513-55b75676976c/dkim/query.go:110`, inside `case "rsa", "":`, whose error is wrapped as `permFailError("key syntax error: " + err.Error())` at query.go:112. `ParsePKIXPublicKey` requires a PKIX `SubjectPublicKeyInfo` and **cannot** parse a bare PKCS#1 key. In the output above, the isolated parser line reads `x509.ParsePKIXPublicKey(published p=) -> ERROR: x509: failed to parse public key (use ParsePKCS1PublicKey instead for this key format)`, and the real end-to-end run reports the identical wrapped message: `A: FAIL domain=example.org err=dkim: key syntax error: x509: failed to parse public key (use ParsePKCS1PublicKey instead for this key format)`. The companion line `x509.ParsePKCS1PublicKey(published p=) -> OK` confirms the blob *is* a valid PKCS#1 key — it is purely a format/parser mismatch, not a corrupt key. **Conclusion:** a recipient using Maddy's own auto-generated TXT record verbatim gets a **key syntax error for every signed message**, verified against the stored T2 blob (`sha256 = af3247ae…f376e`). This is an **OBSERVED interop divergence** at commit `26452dd` (later Maddy versions switched the publish side to `MarshalPKIXPublicKey`).

### Finding B (OBSERVED) — the signature scheme itself is sound (in-library round-trip PASSES)

The signing scheme itself is sound. A fresh in-memory `go-msgauth dkim.Sign` over the **same body** (`"T2 aligned body\r\n"`, relaxed/relaxed, sha256, header keys `From,To,Subject,Date,Message-Id`) produced a body hash **identical** to the one stored in T2's `DKIM-Signature` (§6.3):

```
fresh signature bh   = QqRw2xeTmXHtQwwhf50wtFh4KN3B/NnkSFhmKCsO4AM=
stored T2 signature bh = QqRw2xeTmXHtQwwhf50wtFh4KN3B/NnkSFhmKCsO4AM=
body hashes match (same body -> same bh)? true
B: PASS domain=example.org identifier=alice@example.org
```

Because the fresh signature was produced and then verified through the genuine library round-trip with the **PKIX-form** key (so parsing succeeds), the `PASS` shows that signing, oversigning, canonicalization, and RSA verification are all internally correct — and the matching `bh` proves the stored message's body is intact and is exactly what was hashed. ⇒ the DKIM **scheme** is sound; the failures in Findings A and C are *not* defects of the signing algorithm.

### Finding C (OBSERVED) — the stored mailbox copy no longer matches its own signature

Re-encoding the same keypair as PKIX (so parsing succeeds, isolating the header-signature question from Finding A's parser issue) and re-verifying the **stored** T2 blob (`sha256 = af3247ae…f376e`) gave:

```
C: FAIL domain=example.org err=dkim: signature did not verify: crypto/rsa: verification error
```

The key now parses and the body hash still matches (Finding B), yet the **header** signature fails. Since the fresh in-memory round-trip (Finding B) PASSES while the persisted mailbox copy FAILS, the bytes stored in the mailbox are **not byte-identical to the header block `sign_dkim` signed**. The cause is the local-storage consumer: go-imap-sql's `mboxDelivery` (`github.com/foxcpp/go-imap-sql@v0.3.2-0.20191208094750-8b4ec6b19a78/delivery.go:229`) takes `header = header.Copy()` (delivery.go:230), **adds the per-recipient fields** `Delivered-To`/`Return-Path` (the `d.perRcptHeader` loop, delivery.go:231-234 — visible as the first two stored lines in every §6.3/§7 dump), and then **re-serializes the whole header** with `textproto.WriteHeader(&headerBlob, header)` (delivery.go:237). That re-serialization re-emits the signed fields' folding/ordering as go-message chooses, which no longer reproduces the exact byte sequence the signer canonicalized, so RSA verification of `b=` fails. **This finding is scoped precisely to the LOCALLY STORED mailbox copy** — it says nothing about the on-the-wire validity of a relayed message; remote-relay serialization was not tested (there is no external MX in this local, DNS-independent setup — out of scope for this investigation, per the §3.5 config note on the reject-based remote `default_destination`). Finding B already establishes that the signature `sign_dkim` produces is itself valid.

### Net for a verifying recipient (stated plainly)

For an **aligned** submission (T2), Maddy **does** attach a complete `DKIM-Signature` that covers `From` (oversigned) with a correct body hash, and the signing scheme is cryptographically sound (**Finding B PASS**) — **but** two independent, separately-scoped problems bite a real verifier: (**A**) the public key Maddy *publishes* is a bare PKCS#1 blob, which the standard PKIX-expecting parser (go-msgauth `query.go:110`, and by extension anyone using the published record verbatim) rejects with a key syntax error; and (**C**) the copy that lands in the *local mailbox* has been re-serialized by go-imap-sql, so it no longer matches its own signature — a storage-representation artifact, not a signing defect. For **misaligned** submissions (T1/T4/T5) there is no signature at all, so a recipient simply sees an unsigned message. In short: aligned mail is genuinely signed by a sound scheme, but at this commit neither the published key format (A) nor the persisted mailbox bytes (C) would let a verifier confirm it as-is.

---


## 10. Why it is DOMAIN-level, not per-user (source walk tying observation to code)

The observed T1 line `sender bob@example.org matched by domain rule 'example.org'` (byte-exact in the §5.3 server-side `io_debug` slice) leads directly to the routing code in `internal/msgpipeline/msgpipeline.go`. The function `srcBlockForAddr(mailFrom)` (defined L155, invoked at L113) decides the source block:

1. **Normalize** the envelope address with `address.ForLookup` (L159). Failure returns `501 5.1.7 "Unable to normalize the sender address"` (L161-167).
2. **Exact-address tier:** `srcBlock, ok := dd.d.perSource[cleanFrom]` (L171). A match here logs `"sender %s matched by address rule '%s'"` (L199).
3. If no exact match, **split** to the domain; an unsplittable non-empty address returns `501 5.1.3 "Invalid sender address"` (L179-187).
4. **Domain tier:** `srcBlock, ok = dd.d.perSource[domain]` (L190). A match logs `"sender %s matched by domain rule '%s'"` (L196). **This is the tier T1 matched.**
5. **Fallback tier:** `srcBlock = dd.d.defaultSource` (L193), logging `"sender %s matched by default rule"` (L194). **This is the tier T3 matched.**

The canonical `maddy.conf` registers the local domain via `source $(local_domains)` (L97), which populates `perSource["example.org"]`. There is **no per-authenticated-user tier at this layer** — the routing map is keyed by envelope address and envelope domain only. Any local-domain `MAIL FROM` therefore matches at the **domain** tier regardless of who authenticated, which is exactly why T1 (auth `alice`, envelope `bob@example.org`) is accepted: same domain. The matched block's `rejectErr` (populated by a `reject` directive) is returned at L117-119. The authenticated identity (`username`) is present in the `incoming message` log but is **never consulted** by `srcBlockForAddr`.

---

## 11. Default-vs-explicit policy, ruled-out interpretations, and the behavior-diverges case (O7/O8)

### 11.1 Default-vs-explicit determination

In the **default/canonical** configuration, Maddy enforces sender authorization **only at the envelope DOMAIN level** (via `source` / `default_source`), and enforces `From`-alignment **only** as a DKIM signing precondition that **degrades to unsigned delivery — never as a rejection.** There is **no built-in per-user "MAIL FROM must equal the authenticated user" enforcement** active by default in this revision, and **no `From`-vs-auth rejection** out of the box.

It is important to state this precisely — the correct claim is **"no dedicated sender-authorization check and no default per-user policy," NOT "no possible per-user policy."** The distinction matters:

- **No dedicated construct.** Current upstream Maddy documentation shows a `check { authorize_sender { ... } }` construct for submission that **does not exist** at commit `26452dd`. This was resolved against the repository's own source rather than newer docs. (Grounding: the routing code in `internal/msgpipeline/msgpipeline.go` has only address/domain/default tiers, and `internal/endpoint/smtp/submission.go`'s `submissionPrepare` performs no alignment comparison.)
- **A generic policy hook DOES exist.** This revision ships a general-purpose external-`command` check (module `command`, `internal/check/command/command.go` — `modName = "command"` L28, registered at L374) that **can run at the sender stage** and can therefore implement a per-user `MAIL FROM == {auth_user}` rejection. `CheckSender` runs the command only when the check's `stage` is `sender` (`if s.c.stage != StageSender { return }` at L317, with `StageSender = "sender"` at L34), and `expandCommand` substitutes both `{auth_user}` (→ `s.msgMeta.Conn.AuthUser`, L145-147) and `{sender}` (→ `s.mailFrom`, L178-179) into the command's arguments. The mapping from the command's exit code to an SMTP action defaults to **exit code 1 ⇒ `Reject: true`** (`actions` map initialised at L54-57), producing a `550 5.7.1` rejection on the reject path (`res.Reject = true`, error built at L288+). So an operator *could* enforce per-user alignment with a two-line shell comparator wired as `check { command ... }` at the sender stage — it is simply **not a dedicated feature, not `From`-aware without extra work (the command sees the envelope `{sender}`, not the `From` header), and not enabled in the canonical config.**

The bottom line for O7: **out of the box** this 2019-era revision authorizes senders at the domain level only and never rejects on `From`/auth mismatch; **per-user rejection is achievable but only via explicit, operator-authored `command`-check policy**, never by default.

### 11.2 Ruled-out incorrect interpretations (with evidence)

- **Incorrect (mandatory rebuttal):** *"Because the config pairs `sign_dkim` with a `reject`-based `default_source`, a message whose `From` doesn't match the authenticated user (or the signing domain) will be rejected."* **Refuted by observation:** T1, T4, and T5 are all **ACCEPTED** and delivered (just UNSIGNED); the only rejects are the envelope-domain `default_source` case (T3, `501 5.1.8`) and the missing-`From` syntax case (T6, `554 5.6.0`). Evidence (byte-exact in the §5.3 server slice and §6.1): for T1, the line `sign_dkim: not signing, From address is not authenticated identity` is immediately followed by `submission: accepted	{"msg_id":"fddb4efa"}` (same `msg_id`), i.e. the skip decision does not stop acceptance; this is the `RewriteBody` `if !ok { return nil }` at `internal/modify/dkim/dkim.go:349-351` returning no error.
- **Incorrect:** *"Maddy stamps an `Authentication-Results` header on submitted mail."* **Refuted:** no such header appears in any stored message; the full observed header set is `Date, Delivered-To, Dkim-Signature, From, Message-Id, Received, Return-Path, Subject, To` (§8).
- **Incorrect:** *"The `Received` header records the authenticated user."* **Refuted by T4:** the stored `Received` shows `(envelope-sender <alice@example.org>)` (the envelope) while the message's `From` is `bob@example.org` (§7); the auth identity is stored separately as `AuthUser` and is never placed in `Received`.

### 11.3 The behavior-diverges case (O8)

The prime divergence between a reasonable configuration reading and observed behavior: **the default configuration accepts a cross-user, same-domain sender at the pipeline (domain-level match) while the `sign_dkim` modifier silently skips signing and delivers the message UNSIGNED, rather than rejecting it.** A naive reading of `maddy.conf` — seeing `sign_dkim` sitting next to a `reject`-based `default_source` — would suggest that misaligned senders are blocked. In reality they sail through, merely unsigned. T1 is the concrete instance: auth `alice`, envelope+`From` `bob@example.org`, **accepted and stored unsigned**.

### 11.4 Documented-vs-observed contrast (added rigor)

The `sign_dkim` modifier's own manual **does** document the silent-skip, which is worth stating so the divergence is attributed precisely. `docs/man/maddy-filters.5.scd` (≈L518-544) states `require_sender_match` *Default*: `envelope auth` (L518-519) and defines the semantics as requiring the identifiers to match the `From` field and key domain, **"otherwise - don't sign the message"** (L521-522) — i.e., *don't sign*, not *reject* — which matches observation. The same manual documents `sig_expiry` Default `120h` (L500-501, = the observed `x − t = 432000s`), `header_canon`/`body_canon` Default `relaxed` (matching `c=relaxed/relaxed`), and `newkey_algo` Default `rsa2048` (L513-514). So the silent-unsigned behavior is documented **at the modifier level** — a careful reader could know it — but the **`maddy.conf`-level pairing** of `sign_dkim` with a reject-based `default_source` is what invites the naive misreading. The divergence is thus at the *config-composition* level, compounded by the **PKCS#1/PKIX interop bug** (§9 Finding A) that no amount of config reading would predict.

---


## 12. Objective coverage checklist (O1–O8)

| Objective | Status | Where / evidence |
|-----------|--------|------------------|
| **O1** — Stand up a canonical Maddy instance (submission w/ AUTH, DKIM signing, local delivery, ≥2 users) | **PASS (OBSERVED)** | §3 — built `cmd/maddy` (`-tags debug`) + `cmd/maddyctl` with `CGO_ENABLED=1`; canonical `maddy.conf` (implicit-TLS :465 + plaintext :587); `alice@`/`bob@example.org` provisioned; startup + DKIM keygen logs captured |
| **O2** — Run the three named scenarios + edges through the real submission entry point | **PASS (OBSERVED)** | §4/§5 — T1 cross-user, T3 non-local envelope, T2 legitimate signed (the three named), plus edges T4/T5/T6; **18 runs total** (6 scenarios × {587 ×2, 465 ×1}), all stable |
| **O3** — Report accept/reject mechanism + exact SMTP codes + dependence on how the mismatch is constructed | **PASS (OBSERVED)** | §1, §4, §5, §10 — verbatim codes (`250 2.0.0`, `501 5.1.8`, `554 5.6.0`); outcome depends on mismatch *construction* (envelope-domain ⇒ reject; `From`-only/cross-user ⇒ accept-unsigned) |
| **O4** — Capture actual stored headers verbatim (`DKIM-Signature`, `Received`, negative `Authentication-Results`) | **PASS (OBSERVED)** | §6.3 (byte-exact `DKIM-Signature`, sha256 pinned), §7 (raw `Received`), §8 (negative grep, exit=1) |
| **O5** — Probe DKIM triggering under `From` manipulation, `h=`/`From` coverage byte-exact, verifier conclusion | **PASS (OBSERVED)** | §6 (sign/skip per scenario + §6.5 parser inconsistency), §9 (`h=` covers `From`; verifier Findings A/B/C) |
| **O6** — Full SMTP transaction for ≥1 rejected + ≥1 accepted; document what was tried when `tcpdump` unavailable | **PASS (OBSERVED), with documented fallback** | §5.1 (complete accepted T2) + §5.2 (complete rejected T3 & T6). **Honest caveat:** `tcpdump`/`swaks` are **absent** from the container (§3.9), so protocol-level bytes come from a full raw-socket **client transcript** + server **`io_debug`** on a plaintext :587 listener — **not an on-the-wire packet capture**. This is the explicitly-permitted alternative; no `.pcap` was produced |
| **O7** — Determine default enforces domain-level only; per-user/`From` rejection needs explicit policy; ≥1 incorrect interpretation ruled out | **PASS (OBSERVED + source-grounded)** | §11.1 (domain-level only by default; per-user rejection achievable **only** via explicit `command`-check policy — not a dedicated feature, not default), §11.2 (three interpretations refuted with evidence) |
| **O8** — Identify ≥1 behavior-diverges case | **PASS (OBSERVED) — three found** | §11.3 (domain accept + silent unsigned), §9 Finding A (PKCS#1/PKIX interop divergence), §6.5 (`require_sender_match` default is un-typeable) |

**Methodology / rules-compliance recap.** The code was run first and this answer written from the captured output; every behavioral claim is presented with the command that produced it and the actual, unedited output in a fenced block; each scenario was executed across **3 transports (587 ×2, 465 ×1) = 18 runs**, and every scenario's outcome (decision, SMTP code, sign/skip, resulting UID) was **stable and transport-independent** across those runs (per-run ledger in §5.4); every factual claim is grounded in an exact `file:line` reference naming the function/method/struct, with **OBSERVED vs INFERRED labeled**; the exact submission code path (AUTH → pipeline → `sign_dkim` → storage) was exercised, not a synthetic stand-in; the source tree is read-only and all transient artifacts were created outside it and removed. The one strict-sense limitation, stated plainly: **O6's protocol visibility is a client-transcript + server-`io_debug` capture, not a wire `tcpdump`,** because the capture tools are absent from the environment (§3.9). (See §2 for the full mapping.)

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
- reject execution L117-119, verbatim: `if sourceBlock.rejectErr != nil {` / `dd.log.Debugf("sender %s rejected with error: %v", mailFrom, sourceBlock.rejectErr)` (L118) / `return sourceBlock.rejectErr }`.
- normalize `address.ForLookup` L159; `501 5.1.7` L161-167; invalid-address `501 5.1.3` L179-187.
- exact match `perSource[cleanFrom]` L171; domain match `perSource[domain]` L190; fallback `defaultSource` L193.
- match Debugf: default rule L194, domain rule L196, address rule L199.
- `applyResults` call L316; comment "Run modifiers after Authentication-Results addition" L320-321.

**`internal/msgpipeline/check_runner.go`** (the common check-result path, used by submission too)
- `applyResults` L262; guard `if len(cr.mergedRes.AuthResult) != 0` L301; `header.Add("Authentication-Results", authres.Format(hostname, cr.mergedRes.AuthResult))` L302. *(In the tested submission config no check contributes an `AuthResult`, so the guard is false and no header is added — §8.)*

**`internal/check/command/command.go`** (module `command` — the generic policy hook, §11.1)
- `modName = "command"` L28; stage consts `StageConnection`/`StageSender`/`StageRcpt`/`StageBody` L33-36 (`StageSender = "sender"` L34); stage `EnumList` default `StageBody` L89.
- default exit-code→action map `{1: Reject, 2: Quarantine}` L54-60; per-code override `c.actions[exitCode] = action` L113.
- `expandCommand` placeholder substitution L143; `{auth_user}` → `s.msgMeta.Conn.AuthUser` L145-147; `{sender}` → `s.mailFrom` L178-179.
- `CheckSender` gate `if s.c.stage != StageSender { return }` L317; reject path `res.Reject = true` L267/L284, `550 5.7.1` SMTPError built L288+.
- `module.Register(modName, New)` L374.

**`internal/modify/dkim/dkim.go`** (module `sign_dkim`)
- `oversignDefault` L31-54 (`From` at L37); `signDefault` L55-72.
- config defaults: `key_path` template L137; `header_canon`/`body_canon` relaxed L140-145; `sig_expiry` `5*Day` L146; `hash` sha256 L147-148; `newkey_algo` rsa2048 L149-150; `require_sender_match` `EnumList` allowed set `{envelope, auth_domain, auth_user, off}` (L151) vs built-in default `["envelope","auth"]` (L152) — **the default token `auth` is NOT in the allowed set**, so the default is un-typeable (runtime-proven parser inconsistency dissected in §6.5).
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
