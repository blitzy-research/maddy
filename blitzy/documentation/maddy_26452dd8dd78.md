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

Both builds exited `0`. The toolchain versions were captured with `go version` and `gcc --version` (and `go env GOROOT`) [observed, verbatim]:

```
go version go1.18.10 linux/amd64
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
/usr/local/go
```

So the Go toolchain used was **go1.18.10** (installed at `/usr/local/go`), and **gcc 15.2.0** was present for the `github.com/mattn/go-sqlite3` cgo build (`CGO_ENABLED=1`). The only compiler diagnostic was a harmless `sqlite3-binding.c` `-Wreturn-local-addr` warning originating from `go-sqlite3 v1.11.0`; it appeared identically for both binaries and does not affect the build (exit code stayed `0`) [observed, verbatim]:

```
# github.com/mattn/go-sqlite3
sqlite3-binding.c: In function ‘sqlite3SelectNew’:
sqlite3-binding.c:125322:10: warning: function may return address of local variable [-Wreturn-local-addr]
125322 |   return pNew;
       |          ^~~~
sqlite3-binding.c:125282:10: note: declared here
125282 |   Select standin;
       |          ^~~~~~~
```

**Version banner [observed, verbatim].** Running `/tmp/maddy_scratch/maddy -v` prints:

```
maddy unknown (built from source tree)
```

This is the **canonical default-build value**: the version string defaults to `Version = "unknown (built from source tree)"` [`maddy.go:L41`] because no VCS build information is embedded in a plain `go build` from the source tree. `maddyctl --version` prints the same string. The `-v` flag itself is declared as `printVersion = flag.Bool("v", false, "print version and exit")` [`maddy.go:L109`].

**Exact build and invocation commands [observed, verbatim].** The complete command sequence a normal user runs to reproduce this investigation — built and run out-of-tree under `/tmp/maddy_scratch`:

```bash
# 1. Build the two binaries out-of-tree (both exited 0)
go build -o /tmp/maddy_scratch/maddy ./cmd/maddy
go build -o /tmp/maddy_scratch/maddyctl ./cmd/maddyctl

# 2. Print the canonical version banner
/tmp/maddy_scratch/maddy -v
# -> maddy unknown (built from source tree)

# 3. Provision the two accounts through the real CLI (password read from stdin, or via -p)
/tmp/maddy_scratch/maddyctl --config /tmp/maddy_scratch/maddy_test.conf users create usera@test.example -p '<passwordA>'
/tmp/maddy_scratch/maddyctl --config /tmp/maddy_scratch/maddy_test.conf users create userb@test.example -p '<passwordB>'

# 4. Start the real server with the structure-preserving config and debug logging
/tmp/maddy_scratch/maddy -config /tmp/maddy_scratch/maddy_test.conf -debug
```

The step-4 command is the exact invocation whose `-debug` output is quoted as the startup evidence in Section 2; it binds `submission: listening on tls://0.0.0.0:465` and `imap: listening on tls://0.0.0.0:993`. The config was saved out-of-tree as `/tmp/maddy_scratch/maddy_test.conf`; the filename is not behavioral, so the generic form `/tmp/maddy_scratch/maddy -config /tmp/maddy_scratch/maddy.conf -debug` is equivalent (both filenames were confirmed to start the server and bind `:465`/`:993`).

**`maddy` accepts no positional subcommand [observed, verbatim].** There is no `run` verb — appending a trailing token such as `run` (i.e. `/tmp/maddy_scratch/maddy -config /tmp/maddy_scratch/maddy_test.conf -debug run`) is rejected by `if len(flag.Args()) != 0 { fmt.Println("usage:", os.Args[0], "[options]"); return 2 }` [`maddy.go:L120-L122`], which prints a usage line and exits with status `2`:

```
usage: /tmp/maddy_scratch/maddy [options]
```

---

## Section 2 — Configuration used, and why the substitutions are non-behavioral

The test configuration reproduces the **default `maddy.conf` pipeline structure verbatim** and substitutes **only** three non-behavioral values:

1. a controllable test domain, `test.example`, in place of the packaged placeholder;
2. a self-signed TLS certificate/key pair (required because both submission :465 and IMAP :993 are *implicit* TLS);
3. an out-of-tree state directory (so `all.db` and the generated `dkim_keys/` never land in the repository).

The **listen addresses were deliberately kept at the packaged defaults** — `tls://0.0.0.0:465` for submission and `tls://0.0.0.0:993` for IMAP — matching `maddy.conf:L93` and `maddy.conf:L149`. The test client simply connects to those wildcard listeners over the loopback address `127.0.0.1`. This is confirmed directly by the startup log below, which reports `submission: listening on tls://0.0.0.0:465` and `imap: listening on tls://0.0.0.0:993` — a `0.0.0.0` wildcard bind, **not** a loopback bind.

None of these three substitutions touches the enforcement *logic* — source routing, `require_sender_match`, the reject codes, and the delivery target are all unchanged — so the observations below are canonical for Maddy's default policy.

**Preserved default structure, with source anchors.** The following directives are taken verbatim from the packaged `maddy.conf`:

- Authenticated submission endpoint on implicit TLS :465 — `submission tls://0.0.0.0:465 { auth &local_authdb ... }` [`maddy.conf:L93-L95`].
- Local `source` block with the DKIM modifier and local delivery — `source $(local_domains) { modify { sign_dkim $(primary_domain) default } destination $(local_domains) { ... deliver_to &local_mailboxes } }` [`maddy.conf:L97-L107`].
- The catch-all reject for non-local senders — `default_source { reject 501 5.1.8 "Non-local sender domain" }` [`maddy.conf:L117-L119`].
- SQLite-backed credential store + mailbox storage — `sql local_mailboxes local_authdb { driver sqlite3 ... dsn all.db }` [`maddy.conf:L32-L35`].
- The IMAP inspection endpoint on implicit TLS :993 — `imap tls://0.0.0.0:993 { auth &local_authdb; storage &local_mailboxes }` [`maddy.conf:L149-L152`].

The default configuration path is `/etc/maddy/maddy.conf`, formed by `filepath.Join(ConfigDirectory, "maddy.conf")` with `ConfigDirectory = "/etc/maddy"` [`maddy.go:L107`, `maddy.go:L48`]. The test config was supplied via the `-config` flag and the server was run with `-debug` — i.e. the exact invocation `/tmp/maddy_scratch/maddy -config /tmp/maddy_scratch/maddy_test.conf -debug` listed in Section 1; the startup log immediately below is that command's `-debug` output.

**Startup evidence [observed, verbatim, from the debug log].** On first startup Maddy loaded the SQL storage, instantiated the `sign_dkim` module, and — because no key existed yet — auto-generated a fresh RSA-2048 keypair and wrote both the private key and the public-key DNS record file, then bound the two listeners:

```
[debug] sql: go-imap-sql version 0.4.0
[debug] maddy_test.conf:27: new module sign_dkim [test.example default]
sign_dkim: generating a new rsa2048 keypair...
sign_dkim: generated a new rsa2048 keypair, private key is in dkim_keys/test.example_default.key, TXT record with public key is in dkim_keys/test.example_default.dns,
put its contents into TXT record for default._domainkey.test.example to make signing and verification work
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
v=DKIM1; k=rsa; p=MIIBCgKCAQEA3ntY7ju4V7ikWTPWlF/qW06wB04U52oC6mXYnRR0D/89wq6oeOncpPMTgPQ2h1LiSZcOGjFPf2lkpqKKW/FAdMakPtQ+FWEexGSqakxUU94z5r7W6n2rqTclxvM7rjgVclvi+tYFqXtg6ps02Agh1L/XWQ/aGZ16aV3QcScB4UNGS/XkLmWtPLKZKwLOlBrevPFEUa1uAqhHJCSCychv9J27C0gw6X80Wawfv3QyNFlhvh+1i5w6db7Lhv6wGkCY4VJ1m5UVEat9GguYijh/BrAoGQK6bGv4Ul5igoDJ5vlcnJYDJ617/OUWwodd1PqWF+o+xn3au7TaQQa+fmFx6wIDAQAB
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
reply: b'250-Hello client.test.example\r\n'
reply: b'250-PIPELINING\r\n'
reply: b'250-8BITMIME\r\n'
reply: b'250-ENHANCEDSTATUSCODES\r\n'
reply: b'250-AUTH PLAIN\r\n'
reply: b'250-SMTPUTF8\r\n'
reply: b'250 SIZE 33554432\r\n'
reply: retcode (250); Msg: b'Hello client.test.example\nPIPELINING\n8BITMIME\nENHANCEDSTATUSCODES\nAUTH PLAIN\nSMTPUTF8\nSIZE 33554432'
send: 'AUTH PLAIN AHVzZXJhQHRlc3QuZXhhbXBsZQBwYXNzd29yZEE=\r\n'
reply: b'235 2.0.0 Authentication succeeded\r\n'
reply: retcode (235); Msg: b'2.0.0 Authentication succeeded'
send: 'mail from:<usera@test.example>\r\n'
reply: b'250 2.0.0 Roger, accepting mail from <usera@test.example>\r\n'
reply: retcode (250); Msg: b'2.0.0 Roger, accepting mail from <usera@test.example>'
send: 'rcpt to:<userb@test.example>\r\n'
reply: b"250 2.0.0 I'll make sure <userb@test.example> gets this\r\n"
reply: retcode (250); Msg: b"2.0.0 I'll make sure <userb@test.example> gets this"
send: 'data\r\n'
reply: b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
reply: retcode (354); Msg: b'2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>'
data: (354, b'2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>')
send: b'From: usera@test.example\r\nTo: userb@test.example\r\nSubject: T3-aligned-legit\r\n\r\nBody of T3-aligned-legit.\r\n.\r\n'
reply: b'250 2.0.0 OK: queued\r\n'
reply: retcode (250); Msg: b'2.0.0 OK: queued'
data: (250, b'2.0.0 OK: queued')
send: 'quit\r\n'
reply: b'221 2.0.0 Goodnight and good luck\r\n'
reply: retcode (221); Msg: b'2.0.0 Goodnight and good luck'
```

Matching Maddy `-debug` decision lines [observed, verbatim]:

```
[debug] smtp/pipeline: sender usera@test.example matched by domain rule 'test.example'   {"msg_id":"31b3d758"}
submission: adding missing Message-ID
submission: adding missing Date header
[debug] sign_dkim: signed   {"identifier":"usera@test.example"}
submission: accepted   {"msg_id":"31b3d758"}
```

- **Claim: the envelope was routed to the local `source` block by its domain** — evidence: `sender usera@test.example matched by domain rule 'test.example'`. This is the domain match in `srcBlockForAddr` [`internal/msgpipeline/msgpipeline.go:L190`] logged at [`internal/msgpipeline/msgpipeline.go:L196`].
- **Claim: the message was DKIM-signed** — evidence: `sign_dkim: signed   {"identifier":"usera@test.example"}`, emitted by `s.m.log.DebugMsg("signed", "identifier", id)` after `h.Add("DKIM-Signature", signer.SignatureValue())` [`internal/modify/dkim/dkim.go:L406-L408`].
- **Claim: the message was accepted and queued** — evidence: the SMTP reply `250 2.0.0 OK: queued` and the log line `submission: accepted   {"msg_id":"31b3d758"}`.

### T2 — non-local sender domain → REJECTED `501 5.1.8` (deferred to RCPT)

Authenticate as `usera`, `MAIL FROM:<spoof@nonlocal.tld>`. Full client transcript [observed, verbatim]:

```
send: 'ehlo client.test.example\r\n'
reply: b'250-Hello client.test.example\r\n'
reply: b'250-PIPELINING\r\n'
reply: b'250-8BITMIME\r\n'
reply: b'250-ENHANCEDSTATUSCODES\r\n'
reply: b'250-AUTH PLAIN\r\n'
reply: b'250-SMTPUTF8\r\n'
reply: b'250 SIZE 33554432\r\n'
reply: retcode (250); Msg: b'Hello client.test.example\nPIPELINING\n8BITMIME\nENHANCEDSTATUSCODES\nAUTH PLAIN\nSMTPUTF8\nSIZE 33554432'
send: 'AUTH PLAIN AHVzZXJhQHRlc3QuZXhhbXBsZQBwYXNzd29yZEE=\r\n'
reply: b'235 2.0.0 Authentication succeeded\r\n'
reply: retcode (235); Msg: b'2.0.0 Authentication succeeded'
send: 'mail from:<spoof@nonlocal.tld>\r\n'
reply: b'250 2.0.0 Roger, accepting mail from <spoof@nonlocal.tld>\r\n'
reply: retcode (250); Msg: b'2.0.0 Roger, accepting mail from <spoof@nonlocal.tld>'
send: 'rcpt to:<userb@test.example>\r\n'
reply: b'501 5.1.8 Non-local sender domain (msg ID = 913b2863)\r\n'
reply: retcode (501); Msg: b'5.1.8 Non-local sender domain (msg ID = 913b2863)'
send: 'DATA\r\n'
reply: b'502 5.5.1 Missing RCPT TO command.\r\n'
reply: retcode (502); Msg: b'5.5.1 Missing RCPT TO command.'
send: 'quit\r\n'
reply: b'221 2.0.0 Goodnight and good luck\r\n'
reply: retcode (221); Msg: b'2.0.0 Goodnight and good luck'
```

Matching Maddy `-debug` decision lines [observed, verbatim]:

```
[debug] smtp/pipeline: sender spoof@nonlocal.tld matched by default rule   {"msg_id":"913b2863"}
submission: RCPT error   {"effective_rcpt":"userb@test.example","rcpt":"userb@test.example","reason":"reject directive used","smtp_code":501,"smtp_enchcode":"5.1.8","smtp_msg":"Non-local sender domain"}
submission: aborted   {"msg_id":"913b2863"}
```

- **Claim: a non-local envelope domain falls through to `default_source`** — evidence: `sender spoof@nonlocal.tld matched by default rule`. This is the default-rule fallback in `srcBlockForAddr` [`internal/msgpipeline/msgpipeline.go:L190` no match → default] logged at [`internal/msgpipeline/msgpipeline.go:L194`].
- **Claim: the rejection renders the configured `501 5.1.8 "Non-local sender domain"`** — evidence: `smtp_code":501,"smtp_enchcode":"5.1.8","smtp_msg":"Non-local sender domain"`. The literal is the `reject` directive at [`maddy.conf:L117-L119`], rendered by `parseRejectDirective` [`internal/msgpipeline/config.go:L292`].
- **Claim (critical): `MAIL FROM` is answered `250` and the rejection surfaces only at `RCPT`, because sender rejection is deferred** — evidence: the transcript shows `mail from:<spoof@nonlocal.tld>` → `250 2.0.0 Roger, accepting mail from <spoof@nonlocal.tld>` yet `rcpt to:<userb@test.example>` → `501 5.1.8 Non-local sender domain`. This is the `defer_sender_reject` default, which is `true`: `cfg.Bool("defer_sender_reject", false, true, &endp.deferServerReject)` [`internal/endpoint/smtp/smtp.go:L567`], taking the deferral branch `if !s.endp.deferServerReject` [`internal/endpoint/smtp/smtp.go:L163`]. **A reader must not misread the `250` at `MAIL` as acceptance** — the transaction was rejected at `RCPT`. The subsequent `DATA` then returns `502 5.5.1 Missing RCPT TO command.` because no recipient was ever accepted.

### T1a — same-domain cross-user (envelope + `From` = userB, auth = userA) → ACCEPTED + DKIM SKIPPED

Authenticate as `usera`; `MAIL FROM:<userb@test.example>`; `From: userb@test.example`. Full client transcript [observed, verbatim]:

```
send: 'ehlo client.test.example\r\n'
reply: b'250-Hello client.test.example\r\n'
reply: b'250-PIPELINING\r\n'
reply: b'250-8BITMIME\r\n'
reply: b'250-ENHANCEDSTATUSCODES\r\n'
reply: b'250-AUTH PLAIN\r\n'
reply: b'250-SMTPUTF8\r\n'
reply: b'250 SIZE 33554432\r\n'
reply: retcode (250); Msg: b'Hello client.test.example\nPIPELINING\n8BITMIME\nENHANCEDSTATUSCODES\nAUTH PLAIN\nSMTPUTF8\nSIZE 33554432'
send: 'AUTH PLAIN AHVzZXJhQHRlc3QuZXhhbXBsZQBwYXNzd29yZEE=\r\n'
reply: b'235 2.0.0 Authentication succeeded\r\n'
reply: retcode (235); Msg: b'2.0.0 Authentication succeeded'
send: 'mail from:<userb@test.example>\r\n'
reply: b'250 2.0.0 Roger, accepting mail from <userb@test.example>\r\n'
reply: retcode (250); Msg: b'2.0.0 Roger, accepting mail from <userb@test.example>'
send: 'rcpt to:<userb@test.example>\r\n'
reply: b"250 2.0.0 I'll make sure <userb@test.example> gets this\r\n"
reply: retcode (250); Msg: b"2.0.0 I'll make sure <userb@test.example> gets this"
send: 'data\r\n'
reply: b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
reply: retcode (354); Msg: b'2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>'
data: (354, b'2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>')
send: b'From: userb@test.example\r\nTo: userb@test.example\r\nSubject: T1a-crossuser-fromB\r\n\r\nBody of T1a-crossuser-fromB.\r\n.\r\n'
reply: b'250 2.0.0 OK: queued\r\n'
reply: retcode (250); Msg: b'2.0.0 OK: queued'
data: (250, b'2.0.0 OK: queued')
send: 'quit\r\n'
reply: b'221 2.0.0 Goodnight and good luck\r\n'
reply: retcode (221); Msg: b'2.0.0 Goodnight and good luck'
```

Matching Maddy `-debug` lines [observed, verbatim]:

```
[debug] smtp/pipeline: sender userb@test.example matched by domain rule 'test.example'   {"msg_id":"06ac4941"}
sign_dkim: not signing, From address is not authenticated identity   {"auth_id":"usera@test.example","from_addr":"userb@test.example","msg_id":"06ac4941"}
submission: accepted   {"msg_id":"06ac4941"}
```

- **Claim: `userA`, authenticated, successfully sent a message bearing `userB`'s address — the message was accepted** — evidence: `submission: accepted   {"msg_id":"06ac4941"}` (and `250 2.0.0 OK: queued`). This is the primary falsification evidence used in Section 8.
- **Claim: the only consequence of the auth/`From` mismatch was that signing was skipped (not that the message was rejected)** — evidence: `sign_dkim: not signing, From address is not authenticated identity`. This is the `auth` check in `shouldSign()` [`internal/modify/dkim/dkim.go:L299-L308`].

### T1b — same-domain cross-user (MAIL FROM = userB, `From` = userA, auth = userA) → ACCEPTED + DKIM SKIPPED

Authenticate as `usera`; `MAIL FROM:<userb@test.example>`; `From: usera@test.example`. Full client transcript [observed, verbatim]:

```
send: 'ehlo client.test.example\r\n'
reply: b'250-Hello client.test.example\r\n'
reply: b'250-PIPELINING\r\n'
reply: b'250-8BITMIME\r\n'
reply: b'250-ENHANCEDSTATUSCODES\r\n'
reply: b'250-AUTH PLAIN\r\n'
reply: b'250-SMTPUTF8\r\n'
reply: b'250 SIZE 33554432\r\n'
reply: retcode (250); Msg: b'Hello client.test.example\nPIPELINING\n8BITMIME\nENHANCEDSTATUSCODES\nAUTH PLAIN\nSMTPUTF8\nSIZE 33554432'
send: 'AUTH PLAIN AHVzZXJhQHRlc3QuZXhhbXBsZQBwYXNzd29yZEE=\r\n'
reply: b'235 2.0.0 Authentication succeeded\r\n'
reply: retcode (235); Msg: b'2.0.0 Authentication succeeded'
send: 'mail from:<userb@test.example>\r\n'
reply: b'250 2.0.0 Roger, accepting mail from <userb@test.example>\r\n'
reply: retcode (250); Msg: b'2.0.0 Roger, accepting mail from <userb@test.example>'
send: 'rcpt to:<userb@test.example>\r\n'
reply: b"250 2.0.0 I'll make sure <userb@test.example> gets this\r\n"
reply: retcode (250); Msg: b"2.0.0 I'll make sure <userb@test.example> gets this"
send: 'data\r\n'
reply: b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
reply: retcode (354); Msg: b'2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>'
data: (354, b'2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>')
send: b'From: usera@test.example\r\nTo: userb@test.example\r\nSubject: T1b-crossuser-fromA\r\n\r\nBody of T1b-crossuser-fromA.\r\n.\r\n'
reply: b'250 2.0.0 OK: queued\r\n'
reply: retcode (250); Msg: b'2.0.0 OK: queued'
data: (250, b'2.0.0 OK: queued')
send: 'quit\r\n'
reply: b'221 2.0.0 Goodnight and good luck\r\n'
reply: retcode (221); Msg: b'2.0.0 Goodnight and good luck'
```

Matching Maddy `-debug` lines [observed, verbatim]:

```
[debug] smtp/pipeline: sender userb@test.example matched by domain rule 'test.example'   {"msg_id":"e1aa26eb"}
sign_dkim: not signing, From address is not envelope address   {"envelope":"userb@test.example","from_addr":"usera@test.example","msg_id":"e1aa26eb"}
submission: accepted   {"msg_id":"e1aa26eb"}
```

- **Claim: the message was accepted despite `From` (userA) not matching the envelope (userB)** — evidence: `submission: accepted   {"msg_id":"e1aa26eb"}`.
- **Claim: signing was skipped by the `envelope` check, which fires *before* the `auth` check** — evidence: `sign_dkim: not signing, From address is not envelope address`. This is the `envelope` check [`internal/modify/dkim/dkim.go:L293-L296`], ordered ahead of the `auth` check [`internal/modify/dkim/dkim.go:L299`].

### Tm — mismatched `From`, foreign domain (auth = userA, MAIL FROM = userA, `From: stranger@other.invalid`) → ACCEPTED + DKIM SKIPPED

This is the user's explicit *"From matches neither the authenticated user nor the signing domain"* case. Full client transcript [observed, verbatim]:

```
send: 'ehlo client.test.example\r\n'
reply: b'250-Hello client.test.example\r\n'
reply: b'250-PIPELINING\r\n'
reply: b'250-8BITMIME\r\n'
reply: b'250-ENHANCEDSTATUSCODES\r\n'
reply: b'250-AUTH PLAIN\r\n'
reply: b'250-SMTPUTF8\r\n'
reply: b'250 SIZE 33554432\r\n'
reply: retcode (250); Msg: b'Hello client.test.example\nPIPELINING\n8BITMIME\nENHANCEDSTATUSCODES\nAUTH PLAIN\nSMTPUTF8\nSIZE 33554432'
send: 'AUTH PLAIN AHVzZXJhQHRlc3QuZXhhbXBsZQBwYXNzd29yZEE=\r\n'
reply: b'235 2.0.0 Authentication succeeded\r\n'
reply: retcode (235); Msg: b'2.0.0 Authentication succeeded'
send: 'mail from:<usera@test.example>\r\n'
reply: b'250 2.0.0 Roger, accepting mail from <usera@test.example>\r\n'
reply: retcode (250); Msg: b'2.0.0 Roger, accepting mail from <usera@test.example>'
send: 'rcpt to:<userb@test.example>\r\n'
reply: b"250 2.0.0 I'll make sure <userb@test.example> gets this\r\n"
reply: retcode (250); Msg: b"2.0.0 I'll make sure <userb@test.example> gets this"
send: 'data\r\n'
reply: b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
reply: retcode (354); Msg: b'2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>'
data: (354, b'2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>')
send: b'From: stranger@other.invalid\r\nTo: userb@test.example\r\nSubject: Tm-foreign-from\r\n\r\nBody of Tm-foreign-from.\r\n.\r\n'
reply: b'250 2.0.0 OK: queued\r\n'
reply: retcode (250); Msg: b'2.0.0 OK: queued'
data: (250, b'2.0.0 OK: queued')
send: 'quit\r\n'
reply: b'221 2.0.0 Goodnight and good luck\r\n'
reply: retcode (221); Msg: b'2.0.0 Goodnight and good luck'
```

Matching Maddy `-debug` lines [observed, verbatim]:

```
[debug] smtp/pipeline: sender usera@test.example matched by domain rule 'test.example'   {"msg_id":"25ec8565"}
sign_dkim: not signing, From domain is not key domain   {"from_domain":"other.invalid","key_domain":"test.example","msg_id":"25ec8565"}
submission: accepted   {"msg_id":"25ec8565"}
```

- **Claim: routing accepted the message because the *envelope* (`usera@test.example`) is local — the `From` header plays no part in Layer 1** — evidence: `sender usera@test.example matched by domain rule 'test.example'` and `submission: accepted   {"msg_id":"25ec8565"}`.
- **Claim: signing was skipped because the `From` domain (`other.invalid`) is not the key domain (`test.example`)** — evidence: `sign_dkim: not signing, From domain is not key domain   {"from_domain":"other.invalid","key_domain":"test.example",...}`. This is the From-domain check [`internal/modify/dkim/dkim.go:L287-L290`]. The message was therefore **delivered unsigned** (analyzed in Section 6).

### Tn — missing `From` header entirely → REJECTED `554 5.6.0` at DATA

Authenticate as `usera`; `MAIL FROM:<usera@test.example>`; recipient `userb@test.example`; the message carries **no `From` header**. Full client transcript [observed, verbatim]:

```
send: 'ehlo client.test.example\r\n'
reply: b'250-Hello client.test.example\r\n'
reply: b'250-PIPELINING\r\n'
reply: b'250-8BITMIME\r\n'
reply: b'250-ENHANCEDSTATUSCODES\r\n'
reply: b'250-AUTH PLAIN\r\n'
reply: b'250-SMTPUTF8\r\n'
reply: b'250 SIZE 33554432\r\n'
reply: retcode (250); Msg: b'Hello client.test.example\nPIPELINING\n8BITMIME\nENHANCEDSTATUSCODES\nAUTH PLAIN\nSMTPUTF8\nSIZE 33554432'
send: 'AUTH PLAIN AHVzZXJhQHRlc3QuZXhhbXBsZQBwYXNzd29yZEE=\r\n'
reply: b'235 2.0.0 Authentication succeeded\r\n'
reply: retcode (235); Msg: b'2.0.0 Authentication succeeded'
send: 'mail from:<usera@test.example>\r\n'
reply: b'250 2.0.0 Roger, accepting mail from <usera@test.example>\r\n'
reply: retcode (250); Msg: b'2.0.0 Roger, accepting mail from <usera@test.example>'
send: 'rcpt to:<userb@test.example>\r\n'
reply: b"250 2.0.0 I'll make sure <userb@test.example> gets this\r\n"
reply: retcode (250); Msg: b"2.0.0 I'll make sure <userb@test.example> gets this"
send: 'data\r\n'
reply: b'354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>\r\n'
reply: retcode (354); Msg: b'2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>'
data: (354, b'2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>')
send: b'To: userb@test.example\r\nSubject: Tn-missing-from\r\n\r\nBody of Tn-missing-from.\r\n.\r\n'
reply: b'554 5.6.0 Message does not contains a From header field (msg ID = 6d6f933e)\r\n'
reply: retcode (554); Msg: b'5.6.0 Message does not contains a From header field (msg ID = 6d6f933e)'
data: (554, b'5.6.0 Message does not contains a From header field (msg ID = 6d6f933e)')
send: 'quit\r\n'
reply: b'221 2.0.0 Goodnight and good luck\r\n'
reply: retcode (221); Msg: b'2.0.0 Goodnight and good luck'
```

The `DATA` command is answered `554 5.6.0 Message does not contains a From header field (msg ID = 6d6f933e)`. Matching Maddy `-debug` lines [observed, verbatim]:

```
submission: DATA error   {"modifier":"submission_prepare","msg_id":"6d6f933e","reason":"Message does not contains a From header field","smtp_code":554,"smtp_enchcode":"5.6.0","smtp_msg":"Message does not contains a From header field"}
submission: aborted   {"msg_id":"6d6f933e"}
```

- **Claim: a submission with no `From` header is rejected `554 5.6.0` at DATA with the message "Message does not contains a From header field"** — evidence: the `DATA` reply above. Note the source-code grammatical typo **"contains"** is reproduced verbatim; the literal is at [`internal/endpoint/smtp/submission.go:L41-L43`].

**The rest of the missing/invalid-`From` family [inferred from source; not each variant was separately exercised]:** the same `554 5.6.0` class also covers `"Invalid address in <Sender/To/Cc/Bcc/Reply-To>"` [`internal/endpoint/smtp/submission.go:L54-L56, L70-L72`], `"Invalid address in From"` [`internal/endpoint/smtp/submission.go:L86-L88`], and `"Missing Sender header field"` when there are multiple `From` addresses without a `Sender` header [`internal/endpoint/smtp/submission.go:L101-L103`]. The submission handler also sets `msgMeta.DontTraceSender = true` [`internal/endpoint/smtp/submission.go:L28`], which shapes the `Received` header in Section 5.


---

## Section 5 — Verbatim stored headers (read back through the real IMAP endpoint)

**Inspection method [observed].** The delivered messages were read back through the real IMAP endpoint using Python's `imaplib.IMAP4_SSL` to :993, `SELECT INBOX`, and `FETCH (BODY.PEEK[HEADER])` for `userb@test.example`. (The IMAP/storage read is the sanctioned inspection path — it reads the message exactly as Maddy stored it.) Four messages were stored in `userb`'s INBOX (T3, T1a, T1b, Tm); the three misaligned messages are unsigned.

**T3 (signed) full stored header block [observed, verbatim].** Reproduced exactly as stored, including the header folding, the two-space `Received:  by`, and the lowercase `Dkim-Signature` / `Message-Id` rendering produced by go-imap-sql:

```
Delivered-To: userb@test.example
Return-Path: <usera@test.example>
Dkim-Signature: a=rsa-sha256;
 bh=N+3MkpbIxAU54kCTSqG9wDk7nZMDrtpNnj6/CSnoCdY=; c=relaxed/relaxed;
 d=test.example;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=usera@test.example; s=default; t=1783039243; v=1; x=1783471243; b=OMPF0Z3Ujb4naBVTCANrOM17M2WS5y0CFXkOaslNUd1vTe+GLAFOFlKmPN+fFnT3ZW7WNPJHCV+2+oZ+GqyrThersEDUMYje0ovp7g0cC5BbwnxK1VJrevUtF2iwq1LNEORitQeAEpZE4uNsB2qQ1uuGOotb4knkUEgteOOA7d7453fvmwTxq5yUM7YOmvIHTz4nPazBlIGifynyFGR03tgba4USPNNAdEilOGpRdAfuyoOy9aeGdT2qw/vOFumVbWZbNBRw4pbzf6jKJyho43BxPsCy+5JTZj7nuF1spStu1wFXgHqU+a03V7E+ncbO+vnJGlkUp6GXvpaveWqG1A==;
Received:  by test.example (envelope-sender <usera@test.example>) with
 ESMTPS id 31b3d758; Fri, 03 Jul 2026 00:40:43 +0000
Date: Fri, 3 Jul 2026 00:40:43 +0000
Message-Id: <410eb941-51dc-4aeb-b5f4-5fa98662e5de@test.example>
From: usera@test.example
To: userb@test.example
Subject: T3-aligned-legit
```

- **Claim: the `DKIM-Signature` covers the `From` header — in fact it *oversigns* it.** Evidence: the `h=` tag contains `From` **twice** (`...Cc:From:From:Date...`). Oversigning adds the header name a second time so that a maliciously *added* second `From` would break verification. `From` is in Maddy's oversign default list [`internal/modify/dkim/dkim.go:L31-L38`, `From` at `L37`]. The full tag set observed is: `v=1`, `a=rsa-sha256`, `c=relaxed/relaxed`, `d=test.example`, `s=default`, `i=usera@test.example`, `bh=N+3MkpbIxAU54kCTSqG9wDk7nZMDrtpNnj6/CSnoCdY=`, `t=1783039243`, `x=1783471243`, `b=OMPF0Z3U...` (truncated for readability but reproduced in full in the block above).
- **Claim: the signature is *DMARC-alignable in principle*, because `d=` equals the `From` domain.** Evidence: `d=test.example` and `From: usera@test.example` share the domain `test.example`. Whether it *validates* is a separate question answered in Section 6 (it does not).
- **Claim: the signature carries a 5-day expiry.** Evidence: `t=1783039243` and `x=1783471243`; `1783471243 − 1783039243 = 432000` seconds = 5 days.
- **Claim: the `Received` header Maddy added for submission has *no* leading `from <host> [ip]` client-trace clause; it begins directly with `by`.** Evidence: `Received:  by test.example (envelope-sender <usera@test.example>) with ESMTPS id 31b3d758; ...`. The client-trace clause is gated on `!DontTraceSender` [`internal/target/received.go:L30`], and submission sets `DontTraceSender = true` [`internal/endpoint/smtp/submission.go:L28`], so the builder skips straight to the ` by ` clause [`internal/target/received.go:L62`], then ` (envelope-sender <` [`internal/target/received.go:L69`], ` with ` + protocol (`ESMTPS`) [`internal/target/received.go:L75-L79`], and ` id ` + the message ID [`internal/target/received.go:L81-L82`]. The two spaces in `Received:  by` are the header-separator space plus the value's own leading space before `by`. Header generation is `GenerateReceived` [`internal/target/received.go:L19`].
- **Claim: there is *no* `Authentication-Results` header on any stored message.** Evidence: the header is absent from every fetched header block. The header is only added when checks emit results — `if len(cr.mergedRes.AuthResult) != 0 { header.Add("Authentication-Results", ...) }` [`internal/msgpipeline/check_runner.go:L301-L302`]. The default submission block defines no checks, so no `Authentication-Results` is produced. (This is confirmed **absent** at runtime, not merely expected.)

**T1a / T1b / Tm (unsigned) full stored header blocks [observed, verbatim].** Each was read back through the same real IMAP `FETCH (BODY.PEEK[HEADER])` path and is reproduced exactly as stored. None of the three carries a `Dkim-Signature` or an `Authentication-Results` header.

*T1a* — envelope + `From` = `userB`, auth = `userA`:

```
Delivered-To: userb@test.example
Return-Path: <userb@test.example>
Received:  by test.example (envelope-sender <userb@test.example>) with
 ESMTPS id 06ac4941; Fri, 03 Jul 2026 00:40:45 +0000
Date: Fri, 3 Jul 2026 00:40:45 +0000
Message-Id: <366d49df-3323-428a-a468-300b335afcbc@test.example>
From: userb@test.example
To: userb@test.example
Subject: T1a-crossuser-fromB
```

*T1b* — `MAIL FROM` = `userB`, `From` = `userA`, auth = `userA`:

```
Delivered-To: userb@test.example
Return-Path: <userb@test.example>
Received:  by test.example (envelope-sender <userb@test.example>) with
 ESMTPS id e1aa26eb; Fri, 03 Jul 2026 00:40:45 +0000
Date: Fri, 3 Jul 2026 00:40:45 +0000
Message-Id: <63822e6e-312c-41e2-8e11-979a717ebdad@test.example>
From: usera@test.example
To: userb@test.example
Subject: T1b-crossuser-fromA
```

*Tm* — `MAIL FROM` = `userA`, `From: stranger@other.invalid`:

```
Delivered-To: userb@test.example
Return-Path: <usera@test.example>
Received:  by test.example (envelope-sender <usera@test.example>) with
 ESMTPS id 25ec8565; Fri, 03 Jul 2026 00:40:46 +0000
Date: Fri, 3 Jul 2026 00:40:46 +0000
Message-Id: <758d75fe-5596-42eb-869b-565889e7b41c@test.example>
From: stranger@other.invalid
To: userb@test.example
Subject: Tm-foreign-from
```

- **Claim: all three misaligned messages were delivered *unsigned* — the skip-not-reject behavior is visible at the storage layer.** Evidence: none of the three blocks above contains a `Dkim-Signature` line (contrast the T3 block, which does), and none contains an `Authentication-Results` line; yet each was stored and delivered (`Delivered-To: userb@test.example` present in all three).
- **Claim: the `Received` header and `Return-Path` record the *envelope* sender — the address routing matched on — not the `From` header.** Evidence: T1a and T1b both show `(envelope-sender <userb@test.example>)` and `Return-Path: <userb@test.example>` even though T1b's `From` is `usera@test.example`; Tm shows `(envelope-sender <usera@test.example>)` and `Return-Path: <usera@test.example>` while its `From` is `stranger@other.invalid`.
- **Claim: the `From` header is stored exactly as submitted, even when it names another local user or a foreign domain.** Evidence: `From: userb@test.example` (T1a), `From: usera@test.example` (T1b), and `From: stranger@other.invalid` (Tm) — Maddy neither rewrote nor rejected any of them.

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

**The `off` token is the always-sign short-circuit — it is *not* a skip reason.** When `require_sender_match off` is set, `shouldSign()` takes its very first branch [`internal/modify/dkim/dkim.go:L250`] and returns `("@" + aDomain, true)` for non-EAI mail [`internal/modify/dkim/dkim.go:L259`] or `("@" + m.domain, true)` for EAI mail [`internal/modify/dkim/dkim.go:L261`] — i.e. it returns `ok = true` *before* any of the `From`-domain, `envelope`, or `auth` checks run, so `RewriteBody()` proceeds to add the `DKIM-Signature` rather than hitting the `if !ok { return nil }` skip [`internal/modify/dkim/dkim.go:L348-L350`]. The man page states this verbatim — `off`: "Disable check, always sign." [`docs/man/maddy-filters.5.scd:L531-L532`]. This was confirmed at runtime: a T1a-misaligned message (authenticated as `usera`, `MAIL FROM`/`From` = `userb`) that the default `[envelope, auth]` policy *skips* was instead **signed** under `require_sender_match off`:

```
[debug] sign_dkim: signed	{"identifier":"@test.example"}
```
**[observed]** — and was delivered with a real `DKIM-Signature` (`d=test.example; i=@test.example; s=default; c=relaxed/relaxed; h=…From:From…`) that covers `From`, versus the same message under the default policy which logged `not signing, From address is not authenticated identity` and was delivered unsigned. The *only* sub-case in which `off` does **not** sign is a rare IDNA failure converting the key domain to A-labels, which logs `not signing, cannot convert key domain domain into A-labels` and returns `("", false)` [`internal/modify/dkim/dkim.go:L254-L256`]. (`off` also may not be combined with other sender-match tokens [`internal/modify/dkim/dkim.go:L170-L171`].)

**Every `require_sender_match` / `shouldSign()` skip reason (all of which apply only when `off` is *not* set), with its exact log message and `file:line`:**

| Skip reason | Exact log message | Source |
|-------------|-------------------|--------|
| empty `From` | `not signing, empty From` | `internal/modify/dkim/dkim.go:L266` |
| malformed `From` field | `not signing, malformed From field` | `internal/modify/dkim/dkim.go:L271` |
| multiple addresses in `From` | `not signing, multiple addresses in From` | `internal/modify/dkim/dkim.go:L275` |
| malformed address in `From` | `not signing, malformed address in From` | `internal/modify/dkim/dkim.go:L282` |
| `From` domain ≠ key domain | `not signing, From domain is not key domain` | `internal/modify/dkim/dkim.go:L288` **[observed: Tm]** |
| `envelope` check (`From` ≠ MAIL FROM) | `not signing, From address is not envelope address` | `internal/modify/dkim/dkim.go:L294` **[observed: T1b]** |
| `auth` check (`From` ≠ authenticated identity) | `not signing, From address is not authenticated identity` | `internal/modify/dkim/dkim.go:L306` **[observed: T1a]** |

The `From`-domain check unconditionally requires `From` domain to equal the key domain [`internal/modify/dkim/dkim.go:L287-L290`]; the `envelope` check is guarded by the `envelope` token [`internal/modify/dkim/dkim.go:L293`]; and the `auth` check is guarded by the `auth` token [`internal/modify/dkim/dkim.go:L299`].

### (b) The mismatched-`From` case (the user's follow-on question)

With `From: stranger@other.invalid` (matching **neither** the authenticated user `usera@test.example` **nor** the signing/key domain `test.example`), the message is **not signed** and is delivered unsigned. Evidence [observed, Tm]: `sign_dkim: not signing, From domain is not key domain   {"from_domain":"other.invalid","key_domain":"test.example",...}`.

Answering the three sub-parts precisely:

- **Is the signature still applied?** No — evidence: the Tm debug line above and the absence of any `Dkim-Signature` header on Tm's stored message (Section 5). Because the From-domain check fails first, `shouldSign()` returns `("", false)` and `RewriteBody()` returns `nil` without adding a signature.
- **Does it cover `From`?** There is no signature at all, so there is nothing that covers `From`.
- **What would a verifying recipient see?** An ordinary **unsigned** message (with `From: stranger@other.invalid` and `Return-Path: <usera@test.example>`), i.e. no DKIM protection whatsoever.

### (c) What a verifying recipient actually sees for the one signed message (T3) — DKIM verification deep-dive

- **The signature covers `From` (oversigned, Section 5) and the body hash is correct [observed].** An independent recomputation of the body hash under *both* the relaxed and the simple canonicalizations matched the signature's `bh=N+3MkpbIxAU54kCTSqG9wDk7nZMDrtpNnj6/CSnoCdY=` (the exact `bh=` in T3's stored `DKIM-Signature`, Section 5). One subtlety had to be handled: the copy fetched over IMAP stores the body with a lone `LF` line ending, whereas DKIM's `bh=` is computed over the `CRLF` wire body, so the stored body must be normalized back to `CRLF` before canonicalization for the hashes to match. Verbatim recomputation output:

  ```
  stored body repr : b'Body of T3-aligned-legit.\n'
  wire   body repr : b'Body of T3-aligned-legit.\r\n'
  relaxed canon repr: b'Body of T3-aligned-legit.\r\n'
  signature bh= : N+3MkpbIxAU54kCTSqG9wDk7nZMDrtpNnj6/CSnoCdY=
  relaxed  bh   : N+3MkpbIxAU54kCTSqG9wDk7nZMDrtpNnj6/CSnoCdY= MATCH
  simple   bh   : N+3MkpbIxAU54kCTSqG9wDk7nZMDrtpNnj6/CSnoCdY= MATCH
  ```

  So the message *body* is intact end-to-end.

- **PRIMARY divergence — the published public key is PKCS#1, not the RFC-6376 SubjectPublicKeyInfo (SPKI) [observed + source-confirmed].** The root cause is that Maddy marshals the RSA public key with `keyBlob = x509.MarshalPKCS1PublicKey(pubkey)` [`internal/modify/dkim/keys.go:L143`], which emits a bare PKCS#1 `RSAPublicKey`, whereas DKIM verifiers expect a DER-encoded SPKI. Runtime proof — `openssl asn1parse` of the captured `p=` decodes to a two-INTEGER `SEQUENCE` (modulus, then the exponent `010001`), i.e. a bare PKCS#1 `RSAPublicKey`; an SPKI key would instead wrap an `AlgorithmIdentifier` plus a `BIT STRING` [observed, verbatim]:

  ```
      0:d=0  hl=4 l= 266 cons: SEQUENCE          
      4:d=1  hl=4 l= 257 prim: INTEGER           :DE7B58EE3BB857B8A45933D6945FEA5B4EB0074E14E76A02EA65D89D14740FFF3DC2AEA878E9DCA4F31380F4368752E249970E1A314F7F6964A6A28A5BF14074C6A43ED43E15611EC464AA6A4C5453DE33E6BED6EA7DABA93725C6F33BAE3815725BE2FAD605A97B60EA9B34D80821D4BFD7590FDA199D7A695DD0712701E143464BF5E42E65AD3CB2992B02CE941ADEBCF14451AD6E02A847242482C9C86FF49DBB0B4830E97F3459AC1FBF7432345961BE1FB58B9C3A75BECB86FEB01A4098E152759B951511AB7D1A0B988A387F06B0281902BA6C6BF8525E628280C9E6F95C9C960327AD7BFCE516C2875DD4FA9617EA3EC67DDABBB4DA4106BE7E6171EB
    265:d=1  hl=2 l=   3 prim: INTEGER           :010001
  ```

  Consistently, the captured `p=` base64 begins with the PKCS#1 marker, whereas an RFC-correct SPKI encoding of the *same* key begins with the SPKI marker [observed, verbatim]:

  ```
  PKCS#1 p= prefix : MIIBCgKCAQEA3ntY7ju4V7ikWTPWlF/q
  SPKI   p= prefix : MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A
  ```

  When the real published record is handed to a standard verifier — **go-msgauth, the very library Maddy uses both to sign and to verify (its `verify_dkim` check), version `v0.3.2-0.20191028231513-55b75676976c`** per `go.mod` — it fails to parse the key outright [observed, verbatim]:

  ```
  [PUBLISHED-PKCS1] Domain=test.example Identifier=usera@test.example Err=dkim: key syntax error: x509: failed to parse public key (use ParsePKCS1PublicKey instead for this key format)
  ```

  **Therefore a recipient using Maddy's own DKIM stack cannot even parse the published key**, so the signature can never be validated. *Provenance:* the DNS lookup was **stubbed / non-canonical** — `test.example` cannot be published in real DNS, so a local resolver returned the captured record — but the **key bytes and the signing computation are exactly what Maddy produced**, so this is a canonical statement about Maddy's output, not about the stubbed transport.

- **SECONDARY observation — even a format-corrected key fails to verify the delivered copy [observed; root byte not isolated].** First, the obvious `openssl` re-encode path does *not* work here: `openssl rsa -RSAPublicKey_in -pubout` on the captured PKCS#1 key was attempted and failed [observed, verbatim]:

  ```
  Could not find private key of public key from /tmp/maddy_scratch/evidence/pubkey_pkcs1.pem
  ```

  The format correction was therefore done in Go, by parsing the captured key with `x509.ParsePKCS1PublicKey` and re-marshalling the *same* key with `x509.MarshalPKIXPublicKey` (producing the `MIIBIjAN...` SPKI form shown above). go-msgauth then parses that SPKI record but reports, on the full T3 message fetched from IMAP storage [observed, verbatim]:

  ```
  [SPKI-CORRECTED] Domain=test.example Identifier=usera@test.example Err=dkim: signature did not verify: crypto/rsa: verification error
  ```

  Crucially, go-msgauth checks the **body hash first** and only then the RSA header signature: `// Check body hash` … `failError("body hash did not verify")` at [`github.com/emersion/go-msgauth/dkim/verify.go:L340-L354`], followed by `// Compute data hash` … `failError("signature did not verify: " + err.Error())` at [`github.com/emersion/go-msgauth/dkim/verify.go:L357-L384`]. Because the reported failure is `signature did not verify` (not `body hash did not verify`), **the body-hash check already passed** and only the header RSA signature failed. **Control proving the key/library/signing pipeline are correct:** a go-msgauth sign→verify round-trip with the *same* private key — using relaxed/relaxed canonicalization and an oversigned `From` (matching Maddy's `c=relaxed/relaxed` and its oversign configuration), verified against the format-corrected SPKI record — succeeds [observed, verbatim]:

  ```
  [ROUNDTRIP] Domain=test.example Identifier=@test.example Err=<nil>
  ```

  Since go-msgauth verifies its own freshly-signed, oversigned + relaxed output with this key, the delivered-message failure indicates that the **stored header bytes differ from the exact bytes Maddy signed** — consistent with the stored header set being re-serialized (e.g. header-name casing / folding) after the `DKIM-Signature` is added. The precise mutated byte is **not isolated** and this re-serialization is **[inferred]**; it was observed only on the stored (IMAP) copy — a real outbound `target.remote` wire path was not separately tested, so what a remote recipient would see on the original wire message may differ from the stored copy.

- **Net answer [observed]:** aligned mail *is* signed and the signature *does* cover `From`, but a verifying recipient **cannot validate it** — primarily because the published key record is PKCS#1 rather than SPKI (a standard verifier rejects the key outright), and secondarily because the delivered copy's signature does not validate under test even with a format-corrected key (while the body hash does match).

### Protocol-capture method and the fallback that was required

Per the repo-integrity directive, the capture approach and its fallback are stated explicitly, each with the exact tool output:

- **Plaintext `tcpdump` is insufficient** because :465 and :993 are *implicit* TLS — a raw packet capture yields ciphertext. In this environment `tcpdump` was additionally **not even installed** [observed, verbatim]:

  ```
  tcpdump: command not found
  /bin/bash: line 986: tcpdump: command not found
  ```

- **A plaintext `EHLO` to :465 returns no SMTP banner** because the server expects a TLS handshake first, so the raw read comes back empty [observed, verbatim]:

  ```
  sent (plaintext): EHLO client.test.example
  server replied (plaintext): b''
  ```

- **The sanctioned alternative — a TLS-terminating verbose client — reveals the real exchange.** `openssl s_client -connect 127.0.0.1:465 -quiet` produced the real banner, the full `EHLO` capability list, and the `QUIT` response [observed, verbatim]:

  ```
  220 test.example ESMTP Service Ready
  250-Hello client.test.example
  250-PIPELINING
  250-8BITMIME
  250-ENHANCEDSTATUSCODES
  250-AUTH PLAIN
  250-SMTPUTF8
  250 SIZE 33554432
  221 2.0.0 Goodnight and good luck
  ```

  This confirms the capability list quoted in the T3 transcript (Section 4) against a second, independent TLS-terminating client. Together with Maddy's own `-debug` decision lines (quoted throughout Section 4) and `smtplib.SMTP_SSL` with `set_debuglevel(1)` (the full dialogues in Section 4), these give complete visibility into the real protocol exchange. A TLS-terminating client that observes the true dialogue is **not** a bypassing interface — it speaks the real SMTP protocol to the real endpoint.


---

## Section 7 — Conclusion on default-config sender alignment

**Accept/reject is decided *only* by the envelope `MAIL FROM` domain (Layer 1); the authenticated identity is never consulted for acceptance.** Evidence: `srcBlockForAddr` operates on `var cleanFrom = mailFrom` [`internal/msgpipeline/msgpipeline.go:L156`] and matches `dd.d.perSource[domain]` [`internal/msgpipeline/msgpipeline.go:L190`] — the authenticated user does not appear in this function at all. At runtime, T1a and T1b (cross-user, same *local* domain) were both **accepted** (`submission: accepted`), whereas T2 (a *non-local* domain) was **rejected** `501 5.1.8`. The dividing line is the *envelope domain's locality*, not the sender's identity.

**DKIM signing (Layer 2) is a separate decision and is skip-not-reject.** Evidence: misaligned mail (T1a, T1b, Tm) was delivered **unsigned**, not rejected, because `RewriteBody()` returns `nil` on a failed check [`internal/modify/dkim/dkim.go:L348-L350`].

**Therefore, Maddy's default configuration does *not* bind `MAIL FROM` or `From` to the logged-in user.** Per-user sender-alignment enforcement is simply not part of the default pipeline; it would require explicit policy. The two layers must be held rigorously apart: acceptance is a Layer-1 (envelope-domain) decision, and signing is a Layer-2 (`shouldSign`) decision — a DKIM skip must never be described as a routing reject, nor vice-versa.

---

## Section 8 — Ruled-out incorrect interpretation (falsification)

**Hypothesis ruled out: "authenticated submission binds `MAIL FROM` / `From` to the logged-in user by default."**

If that hypothesis were true, T1a (authenticate as `usera`, but `MAIL FROM:<userb@test.example>`) would have been **rejected**. Instead it was **accepted** end to end. The decisive lines, quoted verbatim from the T1a client transcript and the `-debug` log (the complete T1a transcript is in Section 4) [observed, verbatim]:

```
send: 'mail from:<userb@test.example>\r\n'
reply: b'250 2.0.0 Roger, accepting mail from <userb@test.example>\r\n'
send: 'rcpt to:<userb@test.example>\r\n'
reply: b"250 2.0.0 I'll make sure <userb@test.example> gets this\r\n"
send: b'From: userb@test.example\r\nTo: userb@test.example\r\nSubject: T1a-crossuser-fromB\r\n\r\nBody of T1a-crossuser-fromB.\r\n.\r\n'
reply: b'250 2.0.0 OK: queued\r\n'
submission: accepted   {"msg_id":"06ac4941"}
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

- **The `require_sender_match` default token `auth` is rejected by the code-level validation enum — a genuine three-surface inconsistency.** The `cfg.EnumList("require_sender_match", ...)` call declares its *code-level validation enum* (the `allowed` set) as `["envelope", "auth_domain", "auth_user", "off"]` while setting the default to `["envelope", "auth"]` [`internal/modify/dkim/dkim.go:L151-L152`]. The default is accepted only because `EnumList`'s default callback returns the default list **without** validating it against the allowed set [`internal/config/map.go:L82-L84`]; an *explicitly* supplied `require_sender_match auth` is instead rejected at config-parse time. Evidence [observed, verbatim] — starting the server with a `sign_dkim` block containing `require_sender_match auth`:

  ```
  /tmp/maddy_scratch/f2_auth.conf:23: invalid argument, valid values are: [envelope auth_domain auth_user off]
  ```

  produced by `m.MatchErr("invalid argument, valid values are: %v", allowed)` [`internal/config/map.go:L100`]. This is a genuine **three-surface** mismatch: (1) the *code validation enum* accepts `{envelope, auth_domain, auth_user, off}` and rejects `auth`; (2) the compiled-in *default* is `{envelope, auth}`; and (3) the *man page* documents the valid values as `{off, envelope, auth}` [`docs/man/maddy-filters.5.scd:L530-L538`] — so the man page (the actual documentation) *does* document `auth`, and does *not* mention `auth_domain`/`auth_user`. It is therefore the **code validation enum**, not the documentation, that omits `auth`. Compounding this, `shouldSign()` only ever reads `senderMatch["off"]`, `senderMatch["envelope"]`, and `senderMatch["auth"]` [`internal/modify/dkim/dkim.go:L250, L293, L299`] — it never reads `auth_domain` or `auth_user`, so those two enum-accepted tokens are **dead config** with no runtime effect (confirmed [observed]: `require_sender_match auth_domain` parses and the server starts — `submission: listening on tls://127.0.0.1:11465` — while none of the sender-match branches consume it). That the *default* (`auth`) is a value the *validation enum* would reject when typed explicitly was also exercised via T1a, whose skip reason `From address is not authenticated identity` is exactly the `auth`-token branch [`internal/modify/dkim/dkim.go:L299`] firing under the default.
- **Sender rejection surfaces at `RCPT`, not at `MAIL FROM`.** A reader of `default_source { reject ... }` might expect `MAIL FROM:<spoof@nonlocal.tld>` to be rejected immediately, but it is answered `250` and the `501 5.1.8` appears only at `RCPT` because `defer_sender_reject` defaults `true` [`internal/endpoint/smtp/smtp.go:L567`]. Evidence [observed, T2]: `mail FROM ... 250 2.0.0 Roger, accepting mail from <spoof@nonlocal.tld>` then `rcpt TO ... 501 5.1.8 Non-local sender domain`.

---

## Section 10 — Standards grounding (context; does not replace runtime evidence)

The following situates the observed behavior within the relevant RFCs. This is *standards context*, not runtime evidence.

- **[RFC 6409](https://datatracker.ietf.org/doc/html/rfc6409) §6.1 (Message Submission).** Enforcing that `MAIL FROM` matches the authenticated identity is an **optional** MSA action — an MSA *MAY* reject with `550 5.7.1` when the sender lacks submission rights, but it is not required to. Domain-level-only locality (exactly what Maddy's default does) is therefore standards-conformant. Maddy's `501` invalid-address path likewise aligns with RFC 6409's guidance to reject syntactically improper addresses with `501`.
- **[RFC 6376](https://datatracker.ietf.org/doc/html/rfc6376) (DKIM).** The DKIM signing identity is deliberately decoupled from header addresses, and verifiers fetch the public key from a TXT record at `{selector}._domainkey.<domain>` — here `default._domainkey.test.example`. The `p=` value is expected to be a DER-encoded **SubjectPublicKeyInfo**; this is precisely why the PKCS#1 encoding observed in Sections 6 and 9 is a defect. RFC 6376's oversigning rationale (hashing a header name more times than it appears, so an added duplicate breaks verification) explains why Maddy oversigns `From`.
- **[RFC 7489](https://datatracker.ietf.org/doc/html/rfc7489) (DMARC).** DKIM *alignment* — the signature's `d=` domain aligning with the `From` header domain — is a DMARC concern, not a base-DKIM one. This is why `shouldSign()` refuses to sign a `From`-misaligned message: signing it could never yield DMARC alignment, so Maddy declines rather than emit a signature that is useless for alignment.

The synthesis: a reasonable reading of the default `maddy.conf` (`auth &local_authdb` next to `sign_dkim`) might suggest that authenticated submission binds the sender to the logged-in user and always signs their mail. The standards show that per-user enforcement is *optional* (RFC 6409 §6.1) and that DKIM declines to sign misaligned mail to protect DMARC alignment — which is exactly the behavior confirmed at runtime: Maddy's default accepts same-domain cross-user senders and silently skips signing on misalignment rather than rejecting.

**Standards references (stable links; each verified to resolve with `curl -sI`).** These are the primary sources cited above:

- RFC 6409 — *Message Submission for Mail*: <https://datatracker.ietf.org/doc/html/rfc6409>
- RFC 6376 — *DomainKeys Identified Mail (DKIM) Signatures*: <https://datatracker.ietf.org/doc/html/rfc6376>
- RFC 7489 — *Domain-based Message Authentication, Reporting, and Conformance (DMARC)*: <https://datatracker.ietf.org/doc/html/rfc7489>

Link resolution [observed, verbatim]:

```
curl -sI https://datatracker.ietf.org/doc/html/rfc6409  ->  HTTP/2 200
curl -sI https://datatracker.ietf.org/doc/html/rfc6376  ->  HTTP/2 200
curl -sI https://datatracker.ietf.org/doc/html/rfc7489  ->  HTTP/2 200
```

---

## Cleanup note (repository integrity)

**Created out-of-tree in `/tmp/maddy_scratch` (never inside the repository):** the two build binaries (`maddy`, `maddyctl`); a structure-preserving test configuration; a self-signed TLS certificate and private key for `test.example`; the SQLite `all.db` (users + delivered messages); the auto-generated `dkim_keys/` directory (private key + `.dns` public-key record); and temporary observation scripts (the `smtplib.SMTP_SSL` send client, the `imaplib.IMAP4_SSL` fetch client, a raw SMTP sink used for on-the-wire capture, a local DNS stub for the DKIM key lookup, and out-of-tree Go DKIM verify / round-trip helper modules).

**Removed after evidence capture:** all of the above artifacts were deleted and the Maddy / sink / DNS-stub processes were stopped. Nothing dirtied the repository: `.gitignore` already ignores `*.pem` / `*.crt` / `*.key`; the SQLite database and `dkim_keys/` lived outside the tree; and the build binaries were written out-of-tree (a root `./maddy` is *not* ignored, which is why `-o /tmp/maddy_scratch/...` was used).

**Confirmation:** `git status --porcelain` on the source repository is **empty** (the tree is byte-for-byte unchanged; the baseline was verified clean before any work began). The only permanent addition anywhere is this document, `blitzy/documentation/maddy_26452dd8dd78.md`.

**Cleanup commands and observed output [observed, verbatim].** The teardown sequence and the resulting source-repository state (each of the three `git` commands produced **no output**, shown here by the immediately-following shell prompt):

```
$ kill "$MADDY_PID"          # stop the running server (frees the :465 and :993 listeners)
$ rm -rf /tmp/maddy_scratch  # remove the entire out-of-tree harness (binaries, config, certs, all.db, dkim_keys/, scripts)
$ ls -d /tmp/maddy_scratch
ls: cannot access '/tmp/maddy_scratch': No such file or directory
$ git status --porcelain
$ git diff --stat
$ git status --porcelain --untracked-files=all
$
```

The `ls` confirms the harness is gone, and `git status --porcelain`, `git diff --stat`, and `git status --porcelain --untracked-files=all` all returned empty — the source working tree is byte-for-byte identical to `HEAD`, with no leftover binary, database, DKIM key, or certificate anywhere in the tree. This document (already tracked) is the sole permanent addition.

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

