
# Maddy sender-identity & DKIM enforcement on authenticated submission — a runtime investigation

**Subject under test:** `github.com/foxcpp/maddy`, branch `maddy_26452dd8dd78`, HEAD commit **`26452dd8dd787dc455278b0fdd296f4a5432c768`**.

**Nature of this document:** This is a **run-first** investigation. Every factual claim below is backed by (a) **actual, unedited runtime output** — real `swaks` SMTP transcripts, `maddy -debug` log lines, raw stored message headers dumped from the SQLite store, and DKIM-verifier results — captured by building and running the server, **and** (b) a **`file:line`** reference naming the specific function/struct that performs the work. Statements that were *reasoned from reading source but not directly observed* are explicitly labelled **(inferred)**. All runtime output in this document comes from a **single coherent run** (container hostname `692756bb4fcc`, DKIM key fingerprint `ee58954e…2e486f`, message timestamps `2026-07-07 00:20–00:30 UTC`) so that every transcript, log line, stored header, and verifier verdict is mutually consistent.

**Read-only compliance:** No repository file was modified. The server was built and run entirely from an isolated state directory under `/tmp/maddy-run`; all configs, TLS certs, the SQLite store, `swaks` scripts, DKIM verifiers, logs and captures are **working artifacts** created outside the checkout and deleted afterward. The only committed change from this task is this single document. The `git status --porcelain` proof is in §10.

---

## 1. TL;DR — direct answers

1. **How does Maddy decide to accept or reject a sender from an authenticated session?** Acceptance is decided **solely by the envelope `MAIL FROM` domain** through source-routing in the message pipeline — `srcBlockForAddr` (`internal/msgpipeline/msgpipeline.go:L155`) normalizes the cleaned sender and selects `perSource[full-addr]` → `perSource[domain]` → `defaultSource`. The message **`From:` header is never consulted** for the accept/reject decision, and neither is the authenticated username.

2. **Does the default config bind the sender to the authenticated identity, or is explicit policy required?** The default configuration enforces alignment **only at the domain level**. It does **not** bind `MAIL FROM` or `From:` to the authenticated **username**. There is **no `authorize_sender` check** in this version (the `internal/check/` directory contains only `command`, `dkim`, `dns`, `dnsbl`, `requiretls`, `spf`), and `submissionPrepare` performs **no** sender-vs-auth comparison. Per-user binding would require explicit configuration that the default does not include — and in this version there is no built-in check module that could express it.

3. **A plausible-but-wrong interpretation, ruled out:** "auth-required submission + `require_sender_match envelope auth` means an authenticated user can only send as themselves" is **false**. Scenario **A** (auth `alice`, send as `bob`) and Scenario **D** (auth `alice`, `From: stranger@other.example`) are **both accepted and delivered** (§7.1, §7.4).

4. **`require_sender_match` gates SIGNING, not ACCEPTANCE** (divergence #1). Despite reading like an anti-spoofing acceptance control, `require_sender_match` (default `[envelope auth]`, `internal/modify/dkim/dkim.go:L151-L152`) only decides **whether to DKIM-sign**. A mismatched message is still **accepted and delivered — unsigned** (Scenario D).

5. **The `501 5.1.8` sender rejection surfaces at `RCPT TO`, not `MAIL FROM`** (divergence #2), because `defer_sender_reject` defaults to **true** (`internal/endpoint/smtp/smtp.go:L567`). `MAIL FROM` of a foreign domain returns `250`; the first `RCPT TO` returns `501 5.1.8 "Non-local sender domain"` (§7.2).

6. **Maddy's own DKIM signatures do NOT verify at a standards-compliant recipient** (divergence #3 — the most consequential; it contradicts the naive expectation that enabling `sign_dkim` yields verifiable mail, and it contradicts the plan's *predicted* Scenario C "verifier PASS"). This is a **genuine defect of the pinned build**, not a methodology artifact: it was reproduced against the **exact stored bytes** with maddy's *own* `go-msgauth` library and its *own* generated key, corroborated by an independent verifier (`dkimpy`), and isolated to two independent root causes (§6.3, §8.4):
   - The public key is published as **PKCS#1** rather than the RFC-6376-required **SubjectPublicKeyInfo/PKIX** (`internal/modify/dkim/keys.go:L143`), so a verifier cannot even parse the record.
   - Even after correcting the key to PKIX, the signature **still fails** (`crypto/rsa: verification error`) because the `DKIM-Signature` header is **hashed with one folding algorithm and emitted with another** (`internal/modify/dkim/dkim.go:L406`). The body hash `bh=` is provably correct; the header/RSA check fails. §6.3 gives the reconciliation with the plan's predicted PASS.

7. **`Authentication-Results` is absent on submitted mail.** That header is produced only by the **inbound** checks (`verify_dkim`/`apply_spf`/`dmarc`) on the port-25 receive path, not by the submission/signing path — confirmed absent on every submitted message (§8.5).

8. **Corrections to the plan, confirmed at runtime:** account creation is `maddyctl users create` (there is **no** `creds` command in this version, §9.1), and the `Received` protocol token is **`ESMTPS`** (there is no `ESMTPSA`, §9.2).

All outcomes were **deterministic** across the two runs of each scenario (§7.5).

---

## 2. Environment & exact commands

| Item | Value (observed) |
|------|------------------|
| Canonical image | `andrewparkscaleai/coding-agent:foxcpp__maddy__26452dd8dd787dc455278b0fdd296f4a5432c768` (from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_foxcpp_maddy_1.0`) |
| Commit under test | `26452dd8dd787dc455278b0fdd296f4a5432c768` |
| Go toolchain | `go version go1.18.10 linux/amd64` |
| C toolchain | gcc 10.2.1 (CGO required by `mattn/go-sqlite3 v1.11.0`, see `go.mod:L26`) |
| SMTP test client | `swaks version 20201014.0` |
| TLS tooling | `OpenSSL 1.1.1n` |
| Store inspection | `sqlite3 3.34.1` + `maddyctl imap-msgs` |
| DKIM verifiers | maddy's **own** `go-msgauth` (`dkim.Verify`), and independent `python3 3.9.2` + `dkimpy 1.1.8` |

**Commit confirmation (the binaries are built from the working-tree checkout, whose only difference from `26452dd` is this document — so the compiled Go code is byte-identical to the commit under test):**

```console
$ git -C /src rev-parse 26452dd^{commit}                        # the commit under test (pinned by hash)
26452dd8dd787dc455278b0fdd296f4a5432c768
$ git -C /src merge-base --is-ancestor 26452dd HEAD && echo "26452dd is an ancestor of HEAD"
26452dd is an ancestor of HEAD
$ git -C /src diff --name-status 26452dd..HEAD                  # the only net difference from the code under test
A	blitzy/documentation/maddy_26452dd8dd78.md
```

The single added file is this documentation; **no `.go`, `maddy.conf`, `go.mod`, or `go.sum` differs from `26452dd`**, so the built server is exactly the code under test. (Citations in §11 are cross-checked against a pristine `26452dd` checkout.)

**Build (verbatim commands and complete output):**

```console
$ cd /src
$ CGO_ENABLED=1 go build -o /tmp/maddy-run/bin/maddy ./cmd/maddy
# github.com/mattn/go-sqlite3
sqlite3-binding.c: In function 'sqlite3SelectNew':
sqlite3-binding.c:125322:10: warning: function may return address of local variable [-Wreturn-local-addr]
125322 |   return pNew;
       |          ^~~~
sqlite3-binding.c:125282:10: note: declared here
125282 |   Select standin;
       |          ^~~~~~~
                                          # exit status 0
$ CGO_ENABLED=1 go build -o /tmp/maddy-run/bin/maddyctl ./cmd/maddyctl
# github.com/mattn/go-sqlite3
sqlite3-binding.c: In function 'sqlite3SelectNew':
sqlite3-binding.c:125322:10: warning: function may return address of local variable [-Wreturn-local-addr]
125322 |   return pNew;
       |          ^~~~
sqlite3-binding.c:125282:10: note: declared here
125282 |   Select standin;
       |          ^~~~~~~
                                          # exit status 0
$ ls -l /tmp/maddy-run/bin/
-rwxr-xr-x 1 root root 19307784 Jul  7 00:17 maddy
-rwxr-xr-x 1 root root 19530896 Jul  7 00:17 maddyctl
```

Both compiled successfully (exit 0). The **only** build output is the benign upstream `mattn/go-sqlite3` cgo notice shown above (a C-compiler warning inside the amalgamated SQLite source, not a maddy error).

**Run command (verbatim):**

```console
$ /tmp/maddy-run/bin/maddy -config /tmp/maddy-run/test-maddy.conf -debug
```

`-config` and `-debug` are real flags dispatched by `Run` at `maddy.go:L102`.

### 2.1 Documented deviations from the canonical config (and why)

The run is a faithful copy of the shipped `maddy.conf` with exactly three environment-driven deviations, each explicitly noted. None affects the observed sender/DKIM behavior; the full config is reproduced verbatim in §3.1.

1. **Relocated state.** Global `state /tmp/maddy-run/state` and `runtime /tmp/maddy-run/runtime` directives. Maddy performs `os.Chdir(config.StateDirectory)` at startup (`maddy.go:L220`; default `"/var/lib/maddy"` at `maddy.go:L59`), and the default `dsn all.db` is **relative**, so it resolves *inside* the state dir. Relocating state keeps `all.db`, `dkim_keys/`, `queue/` and `mtasts-cache/` **out of the checkout**. (`.gitignore` excludes compiled binaries, `*.pem`/`*.crt`/`*.key`, and `cmd/maddy/*queue`/`*mtasts-cache`, but **not** `all.db` — which is exactly why relocation is required for byte-for-byte cleanliness.)
2. **Self-signed TLS.** The submission port is implicit-TLS `:465` and requires a certificate. The canonical config points `tls` at `/etc/maddy/certs/$(hostname)/fullchain.pem` (`maddy.conf:L16-L17`), which does not exist in the sandbox, so `tls` is pointed at a self-signed `example.org` pair generated with `openssl req -x509 -newkey rsa:2048 -nodes -keyout /tmp/maddy-run/certs/example.org.key -out /tmp/maddy-run/certs/example.org.crt -subj "/CN=example.org" -days 2 -addext "subjectAltName=DNS:example.org"`.
3. **Test-only plaintext port.** The single submission block binds an additional plaintext listener `tcp://0.0.0.0:1587` alongside the canonical implicit-TLS `tls://0.0.0.0:465`, with `insecure_auth`, **solely** so a human-readable packet capture is possible (`:465` is ciphertext on the wire). This is explicitly a **test-only deviation**; **all four scenarios below were driven over the real implicit-TLS `:465` port** (`swaks --tls-on-connect` to `127.0.0.1:465`), and the primary transaction evidence is the `maddy -debug` log plus the `swaks` transcript. Startup confirms both listeners and warns about the insecure port:

```
submission: listening on tls://0.0.0.0:465
submission: listening on tcp://0.0.0.0:1587
submission: authentication over unencrypted connections is allowed, this is insecure configuration and should be used only for testing!
```

All other pipeline directives are kept faithful to the canonical `maddy.conf` so that observations reflect **default** behavior.

---

## 3. Setup

### 3.1 `test-maddy.conf` — the exact, complete, runnable configuration

This is the **verbatim** config the running server used (byte-for-byte; the three deviations above are the header comments and the marked lines). Critically — and unlike an abbreviated summary — it contains the `driver sqlite3` line (`L27`) that makes the `sql` module runnable, and the submission `source` block delivers to `destination $(local_domains)` (`L77`), matching the canonical `maddy.conf`:

```
# test-maddy.conf — derived from the shipped maddy.conf (commit 26452dd).
# DEVIATIONS FROM CANONICAL (all documented in the answer):
#   1. tls  -> self-signed cert for example.org (canonical points at /etc/maddy/certs/...)
#   2. state/runtime -> relocated under /tmp/maddy-run so the repo tree stays clean
#   3. an extra PLAINTEXT submission listener tcp://0.0.0.0:1587 (+insecure_auth) is added
#      purely as a test-only deviation to allow a human-readable packet capture; the
#      canonical implicit-TLS submission on :465 is preserved unchanged.
# Everything else is byte-faithful to the canonical maddy.conf.

$(hostname) = example.org
$(primary_domain) = example.org
$(local_domains) = $(primary_domain)

# DEVIATION 1: self-signed TLS material for example.org.
tls /tmp/maddy-run/certs/example.org.crt /tmp/maddy-run/certs/example.org.key

# DEVIATION 2: isolate runtime state outside the repository checkout.
state /tmp/maddy-run/state
runtime /tmp/maddy-run/runtime

hostname $(hostname)
autogenerated_msg_domain $(primary_domain)

# Create and initialize sql module, it provides simple authentication and
# storage backend using one database for everything.
sql local_mailboxes local_authdb {
    driver sqlite3
    dsn all.db
}

(local_delivery_actions) {
    modify {
        replace_rcpt postmaster postmaster@$(primary_domain)
        replace_rcpt /(.+)\+(.+)@(.+)/ $1@$3
        alias_file /etc/maddy/aliases
    }
}

smtp tcp://0.0.0.0:25 {
    check {
        require_matching_ehlo
        require_mx_record
        verify_dkim
        apply_spf
    }

    dmarc yes

    source $(local_domains) {
        reject 501 5.1.8 "Use Submission for outgoing SMTP"
    }

    default_source {
        destination postmaster $(local_domains) {
            import local_delivery_actions
            deliver_to &local_mailboxes
        }

        default_destination {
            reject 550 5.1.1 "User not local"
        }
    }
}

submission tls://0.0.0.0:465 tcp://0.0.0.0:1587 {
    # Use sql module for authentication.
    auth &local_authdb

    # DEVIATION 3: allow SASL over the plaintext test port :1587 for packet capture.
    insecure_auth

    source $(local_domains) {
        modify {
            sign_dkim $(primary_domain) default
        }

        destination $(local_domains) {
            import local_delivery_actions
            deliver_to &local_mailboxes
        }

        default_destination {
            deliver_to &remote_queue
        }
    }

    # Prevent local senders from using non-local sender addresses since this is
    # likely a spoofing attempt.
    default_source {
        reject 501 5.1.8 "Non-local sender domain"
    }
}

queue remote_queue {
    max_tries 8
    max_parallelism 16

    target remote {
        authenticate_mx mtasts dnssec
    }

    bounce {
        destination $(local_domains) {
            deliver_to &local_mailboxes
        }
        default_destination {
            reject 550 5.0.0 "Refusing to send DSNs to non-local addresses"
        }
    }
}

imap tls://0.0.0.0:993 {
    auth &local_authdb
    storage &local_mailboxes
}
```

This exhibits the four required properties simultaneously: (1) an **auth-required** submission endpoint on `:465` (`auth &local_authdb`; authentication is *always* required on submission — set at `internal/endpoint/smtp/smtp.go:L590` and enforced at `L662`); (2) **DKIM signing** via `sign_dkim $(primary_domain) default` (canonical `maddy.conf:L99`); (3) **local delivery** to `&local_mailboxes` backed by the `sql`/`imapsql` store with `driver sqlite3` + `dsn all.db` (canonical `maddy.conf:L33-L34`); (4) two accounts, created below. `hostname`, `primary_domain` and `local_domains` all default to `example.org`.

### 3.2 DKIM key auto-generation (first start)

On first start the `sign_dkim` modifier auto-generates the key pair. Observed startup log (`maddy -debug`, verbatim):

```
[debug] /tmp/maddy-run/test-maddy.conf:74: new module sign_dkim [example.org default]
sign_dkim: generating a new rsa2048 keypair...
sign_dkim: generated a new rsa2048 keypair, private key is in dkim_keys/example.org_default.key, TXT record with public key is in dkim_keys/example.org_default.dns,
put its contents into TXT record for default._domainkey.example.org to make signing and verification work
```

Grounding: `internal/modify/dkim/keys.go` — `generateAndWrite` (L77), log line `"generating a new %s keypair..."` (L82), `rsa.GenerateKey(…, 2048)` (L95), `x509.MarshalPKCS8PrivateKey` (L105), key file written with perms `0600` and PEM type `"PRIVATE KEY"` (L121-L131), `.dns` record via `writeDNSRecord` (L136).

Observed key material:

```console
$ head -1 /tmp/maddy-run/state/dkim_keys/example.org_default.key ; stat -c '%a %s bytes' /tmp/maddy-run/state/dkim_keys/example.org_default.key
-----BEGIN PRIVATE KEY-----
600 1704 bytes
```

The public TXT record actually written (this exact `p=` is what a verifier consumes):

```console
$ cat /tmp/maddy-run/state/dkim_keys/example.org_default.dns
v=DKIM1; k=rsa; p=MIIBCgKCAQEAvAev2HVUP8XfQsiJPMzZxogczI3HORJx1SZTUnqIlcBkIx7km4J2mIJ2QFgjK4XtjRh7lp5AiTenlAoFaekgrNX4dD9W59J0aogdXVTK4jlP6N6mS/k+/NqVq9gbz3a8ZKta+wEa7iTzYBs4kUCf5SJTKztNjQH4OPqFtNt89uRG9smPfRtHbS7EdppP3vVIDka2Y+/XzHgN1UNa1S9HivOLxkUKeJXMVlkfCPEX8xfdYzGz3HULjlcKk5+u3TRiVMoRCv/hs0afzEk1I2IoFuQVAyLbA+Bkm767JkDCEukR4T/ThSsz1ZRoSgaJ+szrcNlAbnnMxVC3QYKTRG1MmQIDAQAB
```

> **Important observed detail (foundational to §6.3):** the `p=` value begins `MIIBCgKCAQEA…`, which is the base64 of a bare **PKCS#1 `RSAPublicKey`**. RFC 6376 requires a **SubjectPublicKeyInfo (PKIX)** key, whose base64 begins `MIIBIjANBgkqhkiG9w0BAQEF…`. This is `x509.MarshalPKCS1PublicKey` at `internal/modify/dkim/keys.go:L143` — analyzed in §6.3 and §8.4.

Observed DKIM defaults (from `internal/modify/dkim/dkim.go`): `hash sha256` (L147-L148), `newkey_algo rsa2048` (L149-L150), `key_path dkim_keys/{domain}_{selector}.key` (L137), header/body canonicalization `relaxed`/`relaxed` (L140-L145), default selector `default` (from the config directive).

### 3.3 Accounts and mailboxes

Accounts were created with the **real** subcommand for this version (`users create`; password from `-p` or stdin — `cmd/maddyctl/main.go:L47`/`L71`). Complete captured output:

```console
$ maddyctl --config /tmp/maddy-run/test-maddy.conf users create alice@example.org -p AlicePass123    # exit 0
$ maddyctl --config /tmp/maddy-run/test-maddy.conf users create bob@example.org   -p BobPass123       # exit 0
$ maddyctl --config /tmp/maddy-run/test-maddy.conf users list
alice@example.org
bob@example.org
$ maddyctl --config /tmp/maddy-run/test-maddy.conf imap-mboxes list alice@example.org
INBOX
$ maddyctl --config /tmp/maddy-run/test-maddy.conf imap-mboxes list bob@example.org
INBOX
```

(In this version `local_authdb` and `local_mailboxes` are the **same** `sql`/`imapsql` module, so `go-imap-sql` unifies authentication and storage and an INBOX exists immediately after `users create` — confirmed empirically above via `imap-mboxes list`. `imap-mboxes` is `cmd/maddyctl/main.go:L203`/`L207`.)

---

## 4. How acceptance is decided (envelope-domain source routing)

When authentication succeeds, the submission session (`internal/endpoint/smtp/smtp.go`) accepts `MAIL FROM` syntactically and — because `defer_sender_reject` defaults to **true** (`smtp.go:L567`) — defers the routing decision to the first `RCPT TO`. `Session.Mail` (`smtp.go:L162`) with defer enabled (`if !s.endp.deferServerReject` at `L163`) just stores the sender and returns `250`; the first `Session.Rcpt` (`smtp.go:L208`) invokes the deferred `startDelivery` (`smtp.go:L220`), which runs the message pipeline. A deferred `MAIL FROM` error would be logged `MAIL FROM error (deferred)` (`L223`) and wrapped/returned at `RCPT` (`L225-L226`).

The pipeline decides accept/reject in `srcBlockForAddr` (`internal/msgpipeline/msgpipeline.go:L155`): it computes `cleanFrom = address.ForLookup(mailFrom)` (`L159`), then selects a source block by trying, in order, `perSource[cleanFrom]` (exact address, `L171`) → `perSource[domain]` after `address.Split` (`L174`, `L190`) → `defaultSource` (`L193`). `start` (`msgpipeline.go:L102`) returns the matched block's `rejectErr` if the block is a `reject` directive (`L117-L119`, debug `sender %s rejected with error` at `L118`).

**Decisive point:** this decision is a function of the **envelope `MAIL FROM`** only. The message `From:` header is not read here, and neither is the authenticated username. This is directly visible in the observed debug lines — e.g. Scenario A (`MAIL FROM bob@example.org`) logs `sender bob@example.org matched by domain rule 'example.org'` and is accepted, while Scenario B (`MAIL FROM x@notlocal.example`) logs `sender x@notlocal.example matched by default rule` and is rejected. (Full evidence in §7.)

```mermaid
flowchart TD
    A[SASL PLAIN AUTH on submission :465] --> B{Authenticated?}
    B -->|No| R0[Reject: auth required — smtp.go:L662]
    B -->|Yes| C[MAIL FROM accepted syntactically -> 250]
    C -->|defer_sender_reject = true, smtp.go:L567| D[First RCPT TO triggers startDelivery, smtp.go:L220]
    D --> E{srcBlockForAddr: MAIL FROM domain in local_domains? msgpipeline.go:L155}
    E -->|No| R1[Reject 501 5.1.8 Non-local sender domain at RCPT TO]
    E -->|Yes| F[DATA + submissionPrepare, submission.go:L27]
    F --> G{shouldSign? dkim.go:L249}
    G -->|From domain != key domain, L288| H[Deliver UNSIGNED]
    G -->|From != envelope MAIL FROM, L294| H
    G -->|From != authenticated identity, L306| H
    G -->|all conditions match| I[Add DKIM-Signature, deliver SIGNED — L406]
    H --> S[(Stored in all.db mailbox)]
    I --> S
```

---

## 5. What `submissionPrepare` does — and does NOT do

`submissionPrepare` (`internal/endpoint/smtp/submission.go:L27`) is the only submission-specific header processing, invoked as a message modifier before delivery. Reading the entire function end-to-end (`L27-L130`), it performs exactly the following, in order — a complete enumeration of its address-syntax and header validations:

| Step | Lines | Behavior | Failure response |
|------|-------|----------|------------------|
| 1 | `L28` | sets `msgMeta.DontTraceSender = true` (suppresses the leading `from <host>` clause of `Received`, see §7.6) | — |
| 2 | `L30-L36` | inserts a `Message-ID` if absent (`msgIDField()`; logs `adding missing Message-ID` at `L35`) | `Message-ID generation failed` on RNG error |
| 3 | `L39-L48` | **requires a `From:` header** | `554 5.6.0 "Message does not contains a From header field"` |
| 4 | `L50-L65` | if `Sender:` present, validates it with `mail.ParseAddress` (`L52`) | `554 5.6.0 "Invalid address in Sender"` |
| 5 | `L66-L81` | for each of `To`, `Cc`, `Bcc`, `Reply-To` present, validates with `mail.ParseAddressList` (`L68`) | `554 5.6.0 "Invalid address in <hdr>"` |
| 6 | `L83-L95` | parses `From:` with `mail.ParseAddressList` (`L83`) | `554 5.6.0 "Invalid address in From"` |
| 7 | `L99-L109` | if `From:` has multiple addresses, requires a `Sender:` (RFC 5322 §3.6.2) | `554 5.6.0 "Missing Sender header field"` |
| 8 | `L111-L123` | if `Date:` present, validates with `parseMessageDateTime` | `554 "Malformed Date header"` |
| 9 | `L124-L129` | if `Date:` absent, adds one (logs `adding missing Date header` at `L125`) | — |

Every rejection carries `"modifier": "submission_prepare"` in its structured error (and the offending value in `"addr"`/`"date"`), which is exactly how the edge-case observations in §7.7 are attributed to this function.

**It performs no comparison of the `From:`/`MAIL FROM` sender against the authenticated identity.** Reading the whole function, the only identity-related processing is the **presence** of `From:`, the **uniqueness** rule tying multiple `From:` to a `Sender:`, and **address syntax**. `AuthUser` is never read here, and no sender field is compared to the authenticated user. This is the code-level reason the default configuration cannot enforce per-user sender binding (§8.2).

---

## 6. How signing is decided (`shouldSign` gate order)

`RewriteBody` (`internal/modify/dkim/dkim.go:L340`) reads the authenticated user (`authUser = s.meta.Conn.AuthUser`, `L345`) and calls `shouldSign(...)` (`L348`, passing `s.meta.OriginalFrom`). If it returns `false`, the function returns immediately and the message is delivered **unsigned**; otherwise the signature is added via `h.Add("DKIM-Signature", signer.SignatureValue())` (`L406`) followed by the debug line `signed` (`L408`).

### 6.1 Gate order (evaluated top to bottom), `shouldSign` at `dkim.go:L249`

1. modifier `off` → skip;
2. empty `From` (`L266`);
3. malformed `From` (`L271`);
4. multiple `From` (`L275`);
5. **`From` domain ≠ key domain** (`L287-L288`) → debug `"not signing, From domain is not key domain"` — **Scenario D fails here**;
6. **envelope `MAIL FROM` mismatch** (`L293-L294`, active because `require_sender_match` includes `envelope`) → debug `"not signing, From address is not envelope address"`;
7. **`From` ≠ authenticated identity** (`L299-L306`) → debug `"not signing, From address is not authenticated identity"` — **Scenario A fails here**;
8. otherwise → sign.

Because D fails at gate 5 and A fails at gate 7, they emit **different** debug lines — confirmed verbatim in §7.

### 6.2 Defaults and the oversigned header set

`require_sender_match` default is `[envelope auth]` (`dkim.go:L151-L152`; the valid tokens are `[envelope, auth_domain, auth_user, off]`). The default **oversigned** header list (`oversignDefault`, `dkim.go:L31-L54`) is: `Subject, Sender, To, Cc, From, Date, MIME-Version, Content-Type, Content-Transfer-Encoding, Reply-To, In-Reply-To, Message-Id, References, Autocrypt, Openpgp`. `fieldsToSign` (`dkim.go:L202`) lists each header `(occurrences + 1)` times, which is why the observed `h=` tag doubles the headers that are present (e.g. `Subject:Subject`, `From:From`) and names once those that are absent (e.g. `Sender`, `Cc`, `MIME-Version`). The subtlety at gate 7: because the authenticated name contains `@`, the comparison uses the full address `alice@example.org` (not just the local part; `strings.EqualFold` at `L305`), as seen in Scenario A's debug (`auth_id:alice@example.org`, `from_addr:bob@example.org`). The manual documents this: `docs/man/maddy-filters.5.scd:L535-L538` describes the `auth` token as matching the full address when the username contains `@`, otherwise the local part.

### 6.3 Signatures are produced — but they do not verify (root cause + reconciliation with the plan's predicted PASS)

**Direct statement of the observed result.** Scenario C **is signed** (§7.3) — the server logs `sign_dkim: signed` and stamps a syntactically complete `Dkim-Signature` whose `h=` covers `From`. Yet that signature **fails verification at every RFC-6376 verifier tried, including maddy's own `go-msgauth`**. This directly **contradicts the plan's *predicted* Scenario C "verifier PASS."** The plan framed that outcome as a hypothesis to confirm at runtime (AAP §0.5 states each expected outcome is "to confirm," and §0.5.3 explicitly anticipates that a negative verification result must be judged **against the emitted bytes**). Run-first observation **disconfirms** the predicted PASS and instead surfaces a genuine, reproducible defect of commit `26452dd`. This is precisely the AAP §0.1.1 / §0.5.4 deliverable "a case where the security behavior differs from what a reasonable reading of the configuration would suggest," and per AAP §0.3.2 *fixing* the defect is out of scope — so the resolution is to **report and reconcile the divergence with airtight evidence**, not to fabricate a PASS. The evidence below is captured against the **exact bytes maddy stored** (not a re-serialized copy), meeting the §0.5.3 standard, and reproduced with maddy's *own* libraries (identical `go.mod` pins: `go-message v0.10.9-0.20191116124005-65fd0119e899`, `go-msgauth v0.3.2-0.20191028231513-55b75676976c`, `x/crypto v0.0.0-20191108234033-bd318be0434a`).

**The exact stored bytes.** The signature was verified against the message as it lives in the store, dumped byte-for-byte via `maddyctl imap-msgs dump` (the full stored header block is shown in §7.3). The stored `Dkim-Signature` (CRLF shown as they exist on disk) carries `a=rsa-sha256; c=relaxed/relaxed; d=example.org; s=default; i=alice@example.org; bh=kaTAjM6spWb4aNQuvIfFtkJZ5MvWz8Za5ernh4ELPSY=; …; b=fmC9PkVle…==`.

**Defect 1 — public key published as PKCS#1, not PKIX.** `writeDNSRecord` uses `x509.MarshalPKCS1PublicKey(pubkey)` at `internal/modify/dkim/keys.go:L143`, so the `.dns` `p=` is a bare PKCS#1 `RSAPublicKey`. RFC 6376 requires SubjectPublicKeyInfo. Fed the **as-published** record via a local DNS server, maddy's own verifier cannot even parse the key:

```console
$ go run verify_gomsgauth.go scenarioC_stored.eml     # local DNS serving the AS-PUBLISHED PKCS#1 p=
scenarioC_stored.eml -> domain=example.org identifier=alice@example.org valid=false err=dkim: key syntax error: x509: failed to parse public key (use ParsePKCS1PublicKey instead for this key format)
```

The key *material* is correct — only the ASN.1 wrapper is wrong. Deriving the RFC-correct PKIX encoding from maddy's **own private key** yields a `p=` whose base64 **embeds the published PKCS#1 body verbatim as its suffix** (PKIX = a fixed SubjectPublicKeyInfo prefix + the identical PKCS#1 `RSAPublicKey`):

```console
$ openssl pkey -in /tmp/maddy-run/state/dkim_keys/example.org_default.key -pubout -outform DER | base64 -w0
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A MIIBCgKCAQEAvAev2HVUP8Xf…QYKTRG1MmQIDAQAB
                                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^  <- byte-identical to the .dns p=
$ openssl pkey -in /tmp/maddy-run/state/dkim_keys/example.org_default.key -pubout -outform DER | openssl dgst -sha256
ee58954e424b5d2f649c09ae2ab2241db3de33000922798719b7dd32562e486f      # SPKI fingerprint of the exact signing key
```

(The space in the base64 above is inserted only to mark the PKIX prefix; the real value is unbroken. Same modulus+exponent as the `.dns` record — only the encoding differs.)

**Defect 2 — `DKIM-Signature` header hashed with one folder, emitted with another.** After correcting the served record to that PKIX key (`p=MIIBIjANBgkqhkiG9w0BAQEF…`), the signature **still** fails — now with an RSA/header-hash mismatch, while the **body hash matches**:

```console
$ go run verify_gomsgauth.go scenarioC_stored.eml     # local DNS serving the CORRECTED PKIX p=
scenarioC_stored.eml -> domain=example.org identifier=alice@example.org valid=false err=dkim: signature did not verify: crypto/rsa: verification error

$ python3 verify_dkim.py scenarioC_stored.eml example.org_default.pkix.dns   # independent verifier, PKIX key
verify() returned: False
  -> body hash MATCHED but RSA signature (header hash) FAILED  [dkimpy returns False, not an exception, when only b= fails]
```

**Body hash is provably correct (isolating the failure to the header/RSA hash).** Independently recomputing the relaxed body hash over the exact stored bytes equals the signature's `bh=`, and a tamper control proves dkimpy *would* raise on a genuine body mismatch — so the plain `False` above means only the header hash failed:

```console
$ python3 bodyhash_isolation.py scenarioC_stored.eml example.org_default.pkix.dns
stored bh= tag                    : kaTAjM6spWb4aNQuvIfFtkJZ5MvWz8Za5ernh4ELPSY=
recomputed relaxed bh (CRLF-norm) : kaTAjM6spWb4aNQuvIfFtkJZ5MvWz8Za5ernh4ELPSY=
BODY HASH MATCHES                 : True

tampered-body verify() raised   : "body hash mismatch (got b'M8gnF22UuaPmiGuDU5y9a/UBCAqSIvgG06XIUERHf9g=', expected b'kaTAjM6spWb4aNQuvIfFtkJZ5MvWz8Za5ernh4ELPSY=')"
```

**Isolation via a controlled round-trip** (maddy's real private key, maddy's exact `SignOptions`, identical libraries — a small Go harness `rttest/main.go` emitting three variants, then verified with the same `go-msgauth` `dkim.Verify`):

```console
$ go run main.go
wrote rt_maddy.eml, rt_allinone.eml, rt_fix.eml

# emit the DKIM-Signature exactly as maddy's RewriteBody does:
#   NewSigner + textproto.WriteHeader(signer,*h) + h.Add("DKIM-Signature", signer.SignatureValue()) + go-message serialize
$ go run verify_gomsgauth.go rt_maddy.eml       # PKIX key served
rt_maddy.eml    -> domain=example.org identifier=alice@example.org valid=false err=dkim: signature did not verify: crypto/rsa: verification error
# streaming all-in-one dkim.Sign(...) — one folder used for both hashing and emission:
$ go run verify_gomsgauth.go rt_allinone.eml
rt_allinone.eml -> domain=example.org identifier=alice@example.org valid=true  err=<nil>
# maddy's exact signed headers+body, but emit the sig via signer.Signature() (the folder the signer self-hashed):
$ go run verify_gomsgauth.go rt_fix.eml
rt_fix.eml      -> domain=example.org identifier=alice@example.org valid=true  err=<nil>
```

The `rt_maddy` path (maddy's exact integration) reproduces the **same** `crypto/rsa: verification error` seen on the real stored message; the two variants that use a *single consistent folder* verify. The difference is visible in the raw emitted bytes — the all-in-one/`fix` folders hard-wrap the `bh=`/`h=` tokens mid-value every 75 bytes, whereas maddy's emission keeps them intact:

```text
rt_maddy.eml    (maddy path):  "Dkim-Signature: a=rsa-sha256;\r\n bh=/bj8Q/4NefqpNEbb1CQpx83CKD2ljER1qCkG2hacRag=; c=relaxed/relaxed;\r\n d=example.org;\r\n h=Subject:Subject:Sender:To:To:Cc:From:Fr…"   <- bh= INTACT
rt_allinone.eml (dkim.Sign):   "DKIM-Signature: a=rsa-sha256; bh=/bj8Q/4NefqpNEbb1CQpx83CKD2ljER1qCkG2hacRa\r\n g=; c=relaxed/relaxed; d=example.org; h=Subject:Subject:Sender:To:To:Cc:Fro\r\n m:Fr…"   <- bh= and Cc:From SPLIT mid-token
```

**Mechanism.** Maddy signs by calling `dkim.NewSigner`, writing the current headers to the signer with `textproto.WriteHeader(signer, *h)`, then adding the result with `h.Add("DKIM-Signature", signer.SignatureValue())` (`dkim.go:L406`). Internally the go-msgauth signer computes the `DKIM-Signature` self-hash over its header folded by **`foldHeaderField`** (go-msgauth `header.go`), which hard-wraps **every 75 bytes regardless of token boundaries**. But maddy **emits** the header through go-message's `textproto.WriteHeader`, which folds only at real whitespace. Under **relaxed** header canonicalization the fold's CRLF is removed but a continuation space is kept, so the signer's mid-token wrap injects spaces *inside* tokens that the emitted-then-canonicalized header does not contain. The two byte strings therefore differ → the header hash differs → `crypto/rsa: verification error`. The `fix` round-trip proves it: re-emitting the identical signed headers via `signer.Signature()` (the same folder the signer hashed) makes the message verify; the `allinone` path verifies because `dkim.Sign` uses the *same* folder for hashing and emission.

**Scope of the defect (partly inferred).** Both defects live in shared code — the key-writing `writeDNSRecord` (`keys.go:L143`) and the signature emission in `RewriteBody` (`dkim.go:L406`) — not in anything scenario-specific, so **(inferred)** every DKIM signature emitted by this commit is affected, across all delivery paths (local `imapsql` storage and `smtp_downstream` relay alike). It is **directly observed** for the submission→local-delivery path exercised here. Maddy's own `dkim_test.go` contains only `TestFieldsToSign` and `TestShouldSign` — there is **no round-trip sign→verify test** — which is why this defect is uncaught by upstream CI **(inferred)**.

**Reconciliation, stated plainly.** The correct, honest resolution of the plan's predicted "PASS" is: the prediction is **disconfirmed by observation**. Scenario C satisfies three of the four sub-requirements — it is **accepted**, **delivered**, and **DKIM-signed with `From` covered** — but the signature is **not verifiable** at a standards-compliant recipient. Reporting a PASS would require modifying the product (editing `keys.go:L143` and the `dkim.go:L406` emission), which the read-only scope forbids (AAP §0.3.2, §0.8.2). The divergence is therefore reported as the AAP-required security-behavior finding, with the airtight evidence above.

---


## 7. Per-scenario evidence (A, B, C, D)

Every scenario authenticates as **`alice@example.org`** over implicit TLS on `:465` (`swaks --tls-on-connect`, `--ehlo test.local`). `swaks` sets the envelope `MAIL FROM` with `--from` and the message `From:` header independently with `--header "From: …"`, exactly the envelope-vs-header split the scenarios require. The base64 `AUTH PLAIN` token `AGFsaWNlQGV4YW1wbGUub3JnAEFsaWNlUGFzczEyMw==` decodes to `\0alice@example.org\0AlicePass123`. Each scenario was run **twice**; §7.5 tabulates the two runs and shows the outcomes are deterministic.

### 7.1 Scenario (A) — *"Authenticate as one user but attempt `MAIL FROM` with a different user's address."*

**Command:**
```console
$ swaks --server 127.0.0.1:465 --tls-on-connect --ehlo test.local \
        --auth PLAIN --auth-user alice@example.org --auth-password AlicePass123 \
        --from bob@example.org --to alice@example.org \
        --header "From: bob@example.org" --header "Subject: Scenario A run1" \
        --body "scenario A - authenticated as alice, envelope and header both bob"
```

**Full unedited `swaks` transcript (run 1):**
```
=== Trying 127.0.0.1:465...
=== Connected to 127.0.0.1.
=== TLS started with cipher TLSv1.3:TLS_AES_128_GCM_SHA256:128
=== TLS no local certificate set
=== TLS peer DN="/CN=example.org"
<~  220 example.org ESMTP Service Ready
 ~> EHLO test.local
<~  250-Hello test.local
<~  250-PIPELINING
<~  250-8BITMIME
<~  250-ENHANCEDSTATUSCODES
<~  250-AUTH PLAIN
<~  250-SMTPUTF8
<~  250 SIZE 33554432
 ~> AUTH PLAIN AGFsaWNlQGV4YW1wbGUub3JnAEFsaWNlUGFzczEyMw==
<~  235 2.0.0 Authentication succeeded
 ~> MAIL FROM:<bob@example.org>
<~  250 2.0.0 Roger, accepting mail from <bob@example.org>
 ~> RCPT TO:<alice@example.org>
<~  250 2.0.0 I'll make sure <alice@example.org> gets this
 ~> DATA
<~  354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
 ~> Date: Tue, 07 Jul 2026 00:20:12 +0000
 ~> To: alice@example.org
 ~> From: bob@example.org
 ~> Subject: Scenario A run1
 ~> Message-Id: <20260707002012.002450@692756bb4fcc>
 ~> X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/
 ~> 
 ~> scenario A - authenticated as alice, envelope and header both bob
 ~> 
 ~> 
 ~> .
<~  250 2.0.0 OK: queued
 ~> QUIT
<~  221 2.0.0 Goodnight and good luck
=== Connection closed with remote host.
SWAKS_EXIT=0
```

**`maddy -debug` window (run 1, verbatim):**
```
submission: incoming message	{"msg_id":"de7f4215","sender":"bob@example.org","src_host":"test.local","src_ip":"127.0.0.1:37320","username":"alice@example.org"}
[debug] smtp/pipeline: sender bob@example.org matched by domain rule 'example.org'	{"msg_id":"de7f4215"}
[debug] smtp/pipeline: global rcpt modifiers: alice@example.org => alice@example.org	{"msg_id":"de7f4215"}
[debug] smtp/pipeline: per-source rcpt modifiers: alice@example.org => alice@example.org	{"msg_id":"de7f4215"}
[debug] smtp/pipeline: recipient alice@example.org matched by domain rule 'example.org'	{"msg_id":"de7f4215"}
[debug] smtp/pipeline: per-rcpt modifiers: alice@example.org => alice@example.org	{"msg_id":"de7f4215"}
[debug] smtp/pipeline: tgt.Start(bob@example.org) ok, target = sql:local_mailboxes	{"msg_id":"de7f4215"}
submission: RCPT ok	{"msg_id":"de7f4215","rcpt":"alice@example.org"}
sign_dkim: not signing, From address is not authenticated identity	{"auth_id":"alice@example.org","from_addr":"bob@example.org","msg_id":"de7f4215"}
[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"de7f4215"}
submission: accepted	{"msg_id":"de7f4215"}
[debug] submission: reset	
```

**Raw stored headers — dumped byte-for-byte from the store:**
```console
$ maddyctl --config /tmp/maddy-run/test-maddy.conf imap-msgs dump alice@example.org INBOX 1
Delivered-To: alice@example.org
Return-Path: <bob@example.org>
Received:  by example.org (envelope-sender <bob@example.org>) with ESMTPS id
 de7f4215; Tue, 07 Jul 2026 00:20:12 +0000
Date: Tue, 07 Jul 2026 00:20:12 +0000
To: alice@example.org
From: bob@example.org
Subject: Scenario A run1
Message-Id: <20260707002012.002450@692756bb4fcc>
X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/

scenario A - authenticated as alice, envelope and header both bob
```

**Recipient-side verification of the delivered message (no DNS lookup needed — the message is unsigned):**
```console
$ go run verify_gomsgauth.go scenarioA_stored.eml         # maddy's own go-msgauth
scenarioA_stored.eml -> NO DKIM-Signature found
$ python3 verify_dkim.py scenarioA_stored.eml example.org_default.pkix.dns   # independent dkimpy
verify() returned: False
$ grep -ci '^DKIM-Signature\|^Authentication-Results' scenarioA_stored.eml   # header presence
0
```

**Outcome:** **Accepted (`250`) and delivered.** The envelope domain `example.org` matched the source block by **domain** rule (`srcBlockForAddr`, `msgpipeline.go:L190`) — the fact that the sender is a *different user* (`bob`) is never checked. The message is **NOT DKIM-signed** because the `From:` (`bob@example.org`) is not the authenticated identity (`alice@example.org`) — gate 7 of `shouldSign` (`dkim.go:L299-L306`). The stored message has **no `Dkim-Signature`** and **no `Authentication-Results`** (verifier: `NO DKIM-Signature found`; `dkimpy` `False`; header count `0`). This scenario **disproves** the "auth means send-as-self" interpretation.

### 7.2 Scenario (B) — *"Send with a `MAIL FROM` domain that doesn't match any configured signing domain."*

**Command:**
```console
$ swaks --server 127.0.0.1:465 --tls-on-connect --ehlo test.local \
        --auth PLAIN --auth-user alice@example.org --auth-password AlicePass123 \
        --from x@notlocal.example --to alice@example.org \
        --header "From: x@notlocal.example" --body "scenario B"
```

**Full unedited `swaks` transcript (run 1):**
```
=== Trying 127.0.0.1:465...
=== Connected to 127.0.0.1.
=== TLS started with cipher TLSv1.3:TLS_AES_128_GCM_SHA256:128
=== TLS no local certificate set
=== TLS peer DN="/CN=example.org"
<~  220 example.org ESMTP Service Ready
 ~> EHLO test.local
<~  250-Hello test.local
<~  250-PIPELINING
<~  250-8BITMIME
<~  250-ENHANCEDSTATUSCODES
<~  250-AUTH PLAIN
<~  250-SMTPUTF8
<~  250 SIZE 33554432
 ~> AUTH PLAIN AGFsaWNlQGV4YW1wbGUub3JnAEFsaWNlUGFzczEyMw==
<~  235 2.0.0 Authentication succeeded
 ~> MAIL FROM:<x@notlocal.example>
<~  250 2.0.0 Roger, accepting mail from <x@notlocal.example>
 ~> RCPT TO:<alice@example.org>
<~* 501 5.1.8 Non-local sender domain (msg ID = d35022c9)
 ~> QUIT
<~  221 2.0.0 Goodnight and good luck
=== Connection closed with remote host.
SWAKS_EXIT=24
```

**`maddy -debug` window (run 1, verbatim):**
```
submission: incoming message	{"msg_id":"d35022c9","sender":"x@notlocal.example","src_host":"test.local","src_ip":"127.0.0.1:55624","username":"alice@example.org"}
[debug] smtp/pipeline: sender x@notlocal.example matched by default rule	{"msg_id":"d35022c9"}
[debug] smtp/pipeline: global rcpt modifiers: alice@example.org => alice@example.org	{"msg_id":"d35022c9"}
[debug] smtp/pipeline: per-source rcpt modifiers: alice@example.org => alice@example.org	{"msg_id":"d35022c9"}
[debug] smtp/pipeline: recipient alice@example.org matched by default rule (clean = alice@example.org)	{"msg_id":"d35022c9"}
submission: RCPT error	{"effective_rcpt":"alice@example.org","rcpt":"alice@example.org","reason":"reject directive used","smtp_code":501,"smtp_enchcode":"5.1.8","smtp_msg":"Non-local sender domain"}
submission: aborted	{"msg_id":"d35022c9"}
```

**Outcome:** **Rejected `501 5.1.8 "Non-local sender domain"`.** Critically, `MAIL FROM:<x@notlocal.example>` returns **`250`**, and the rejection surfaces at the first **`RCPT TO`**. This is the deferred-reject behavior: `defer_sender_reject` defaults to **true** (`smtp.go:L567`), so the foreign sender matched `default_source { reject 501 5.1.8 … }` (canonical `maddy.conf:L117-L119`) only when `startDelivery` ran at `RCPT` (`smtp.go:L220`); the debug shows `matched by default rule` (`msgpipeline.go:L193-L194`) and `reason:"reject directive used"`. Nothing is delivered (`submission: aborted`). This is **divergence #2** (§8.4). (`swaks` exit `24` is its "transaction rejected" code.)

### 7.3 Scenario (C) — *"Send a legitimate message that should be properly signed."*

**Command:**
```console
$ swaks --server 127.0.0.1:465 --tls-on-connect --ehlo test.local \
        --auth PLAIN --auth-user alice@example.org --auth-password AlicePass123 \
        --from alice@example.org --to bob@example.org \
        --header "From: alice@example.org" --header "Subject: Scenario C run" \
        --body "scenario C - a legitimate aligned message."
```

**Full unedited `swaks` transcript (run 1):**
```
=== Trying 127.0.0.1:465...
=== Connected to 127.0.0.1.
=== TLS started with cipher TLSv1.3:TLS_AES_128_GCM_SHA256:128
=== TLS no local certificate set
=== TLS peer DN="/CN=example.org"
<~  220 example.org ESMTP Service Ready
 ~> EHLO test.local
<~  250-Hello test.local
<~  250-PIPELINING
<~  250-8BITMIME
<~  250-ENHANCEDSTATUSCODES
<~  250-AUTH PLAIN
<~  250-SMTPUTF8
<~  250 SIZE 33554432
 ~> AUTH PLAIN AGFsaWNlQGV4YW1wbGUub3JnAEFsaWNlUGFzczEyMw==
<~  235 2.0.0 Authentication succeeded
 ~> MAIL FROM:<alice@example.org>
<~  250 2.0.0 Roger, accepting mail from <alice@example.org>
 ~> RCPT TO:<bob@example.org>
<~  250 2.0.0 I'll make sure <bob@example.org> gets this
 ~> DATA
<~  354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
 ~> Date: Tue, 07 Jul 2026 00:20:52 +0000
 ~> To: bob@example.org
 ~> From: alice@example.org
 ~> Subject: Scenario C run
 ~> Message-Id: <20260707002052.002495@692756bb4fcc>
 ~> X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/
 ~> 
 ~> scenario C - a legitimate aligned message.
 ~> 
 ~> 
 ~> .
<~  250 2.0.0 OK: queued
 ~> QUIT
<~  221 2.0.0 Goodnight and good luck
=== Connection closed with remote host.
SWAKS_EXIT=0
```

**`maddy -debug` window (run 1, verbatim):**
```
submission: incoming message	{"msg_id":"dafcb55f","sender":"alice@example.org","src_host":"test.local","src_ip":"127.0.0.1:36490","username":"alice@example.org"}
[debug] smtp/pipeline: sender alice@example.org matched by domain rule 'example.org'	{"msg_id":"dafcb55f"}
[debug] smtp/pipeline: global rcpt modifiers: bob@example.org => bob@example.org	{"msg_id":"dafcb55f"}
[debug] smtp/pipeline: per-source rcpt modifiers: bob@example.org => bob@example.org	{"msg_id":"dafcb55f"}
[debug] smtp/pipeline: recipient bob@example.org matched by domain rule 'example.org'	{"msg_id":"dafcb55f"}
[debug] smtp/pipeline: per-rcpt modifiers: bob@example.org => bob@example.org	{"msg_id":"dafcb55f"}
[debug] smtp/pipeline: tgt.Start(alice@example.org) ok, target = sql:local_mailboxes	{"msg_id":"dafcb55f"}
submission: RCPT ok	{"msg_id":"dafcb55f","rcpt":"bob@example.org"}
[debug] sign_dkim: signed	{"identifier":"alice@example.org"}
[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"dafcb55f"}
submission: accepted	{"msg_id":"dafcb55f"}
[debug] submission: reset	
```

**Raw stored headers — dumped byte-for-byte from the store (this is the exact input to the §6.3 verification; the `b=`/`bh=` are the real, complete values):**
```console
$ maddyctl --config /tmp/maddy-run/test-maddy.conf imap-msgs dump bob@example.org INBOX 1
Delivered-To: bob@example.org
Return-Path: <alice@example.org>
Dkim-Signature: a=rsa-sha256;
 bh=kaTAjM6spWb4aNQuvIfFtkJZ5MvWz8Za5ernh4ELPSY=; c=relaxed/relaxed;
 d=example.org;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=alice@example.org; s=default; t=1783383653; v=1; x=1783815653; b=fmC9PkVle5ukRC6q/55vb594Hpx2kt369513k3pRcqNwVZWdgM9iUnkKe+NP30/zmDIoVgf8NzaSorb3L5UvF4fy9TVz0NceuilL3a4p4gLabEzSHwh5PtIB59vP90TuDJmK/E7FOFwKDRWfj3hgm78RMfopUfifFRwGRYLUwh8H5Hdrc9Qi+cS0WgBoshb2MVPn1QsTr3/mF0AUK8gqev0dcZfocQapTwHMlncdW05FQ2K82UsGA0iFsE5QApoArHKGoQ5YR2oVH7xuE6Efz3qmWWD4hNanaqH/KXgV77kpFbHvtu5sdvzO14qlv29tlA71deVhC9uZwgaFh0+4nQ==;
Received:  by example.org (envelope-sender <alice@example.org>) with ESMTPS
 id dafcb55f; Tue, 07 Jul 2026 00:20:53 +0000
Date: Tue, 07 Jul 2026 00:20:52 +0000
To: bob@example.org
From: alice@example.org
Subject: Scenario C run
Message-Id: <20260707002052.002495@692756bb4fcc>
X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/

scenario C - a legitimate aligned message.
```

**Recipient-side verification (offline, fed maddy's own generated key via a local DNS server) — full breakdown in §6.3:**
```console
$ go run verify_gomsgauth.go scenarioC_stored.eml    # DNS serving the AS-PUBLISHED PKCS#1 p=
scenarioC_stored.eml -> valid=false err=dkim: key syntax error: x509: failed to parse public key (use ParsePKCS1PublicKey instead for this key format)
$ go run verify_gomsgauth.go scenarioC_stored.eml    # DNS serving the CORRECTED PKIX p=
scenarioC_stored.eml -> valid=false err=dkim: signature did not verify: crypto/rsa: verification error
$ python3 verify_dkim.py scenarioC_stored.eml example.org_default.pkix.dns
verify() returned: False
$ grep -ci '^Authentication-Results' scenarioC_stored.eml
0
```

**Outcome:** **Accepted (`250`), delivered, and DKIM-signed** (`sign_dkim: signed`, `dkim.go:L406-L408`). The `DKIM-Signature` `h=` tag **covers `From`** (indeed oversigned: `From:From`), along with `Subject`, `To`, `Date`, `Message-Id`, etc. **However, the signature does NOT verify** — the body hash `bh=` is correct but the header/RSA check fails at both maddy's own `go-msgauth` and independent `dkimpy` (§6.3). This is the observed **contradiction of the plan's *predicted* "verifier PASS"**: run-first observation shows the signature is present and structurally complete, body hash correct, yet unverifiable — a genuine two-defect bug of the pinned build, reconciled in §6.3 and §8.4. No `Authentication-Results` header is present.

### 7.4 Scenario (D) — *"Send a message that should trigger DKIM signing but use a `From` header that doesn't match the authenticated user or signing domain — observe whether a signature is still applied, whether it covers `From`, and what a verifying recipient sees."*

**Command:**
```console
$ swaks --server 127.0.0.1:465 --tls-on-connect --ehlo test.local \
        --auth PLAIN --auth-user alice@example.org --auth-password AlicePass123 \
        --from alice@example.org --to bob@example.org \
        --header "From: stranger@other.example" --header "Subject: Scenario D run" \
        --body "scenario D - DKIM-triggering send with foreign From header"
```

**Full unedited `swaks` transcript (run 1):**
```
=== Trying 127.0.0.1:465...
=== Connected to 127.0.0.1.
=== TLS started with cipher TLSv1.3:TLS_AES_128_GCM_SHA256:128
=== TLS no local certificate set
=== TLS peer DN="/CN=example.org"
<~  220 example.org ESMTP Service Ready
 ~> EHLO test.local
<~  250-Hello test.local
<~  250-PIPELINING
<~  250-8BITMIME
<~  250-ENHANCEDSTATUSCODES
<~  250-AUTH PLAIN
<~  250-SMTPUTF8
<~  250 SIZE 33554432
 ~> AUTH PLAIN AGFsaWNlQGV4YW1wbGUub3JnAEFsaWNlUGFzczEyMw==
<~  235 2.0.0 Authentication succeeded
 ~> MAIL FROM:<alice@example.org>
<~  250 2.0.0 Roger, accepting mail from <alice@example.org>
 ~> RCPT TO:<bob@example.org>
<~  250 2.0.0 I'll make sure <bob@example.org> gets this
 ~> DATA
<~  354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
 ~> Date: Tue, 07 Jul 2026 00:26:46 +0000
 ~> To: bob@example.org
 ~> From: stranger@other.example
 ~> Subject: Scenario D run
 ~> Message-Id: <20260707002646.003851@692756bb4fcc>
 ~> X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/
 ~> 
 ~> scenario D - DKIM-triggering send with foreign From header
 ~> 
 ~> 
 ~> .
<~  250 2.0.0 OK: queued
 ~> QUIT
<~  221 2.0.0 Goodnight and good luck
=== Connection closed with remote host.
SWAKS_EXIT=0
```

**`maddy -debug` window (run 1, verbatim):**
```
submission: incoming message	{"msg_id":"352444f8","sender":"alice@example.org","src_host":"test.local","src_ip":"127.0.0.1:55188","username":"alice@example.org"}
[debug] smtp/pipeline: sender alice@example.org matched by domain rule 'example.org'	{"msg_id":"352444f8"}
[debug] smtp/pipeline: recipient bob@example.org matched by domain rule 'example.org'	{"msg_id":"352444f8"}
submission: RCPT ok	{"msg_id":"352444f8","rcpt":"bob@example.org"}
sign_dkim: not signing, From domain is not key domain	{"from_domain":"other.example","key_domain":"example.org","msg_id":"352444f8"}
submission: accepted	{"msg_id":"352444f8"}
```

**Raw stored headers — dumped byte-for-byte from the store:**
```console
$ maddyctl --config /tmp/maddy-run/test-maddy.conf imap-msgs dump bob@example.org INBOX 3
Delivered-To: bob@example.org
Return-Path: <alice@example.org>
Received:  by example.org (envelope-sender <alice@example.org>) with ESMTPS
 id 352444f8; Tue, 07 Jul 2026 00:26:46 +0000
Date: Tue, 07 Jul 2026 00:26:46 +0000
To: bob@example.org
From: stranger@other.example
Subject: Scenario D run
Message-Id: <20260707002646.003851@692756bb4fcc>
X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/

scenario D - DKIM-triggering send with foreign From header
```

**Recipient-side verification (unsigned message):**
```console
$ go run verify_gomsgauth.go scenarioD_stored.eml
scenarioD_stored.eml -> NO DKIM-Signature found
$ python3 verify_dkim.py scenarioD_stored.eml example.org_default.pkix.dns
verify() returned: False
$ grep -ci '^DKIM-Signature\|^Authentication-Results' scenarioD_stored.eml
0
```

**Outcome (answering each part of the scenario):**
- *Is a signature still applied?* **No.** The `From:` domain `other.example` ≠ the key domain `example.org`, so `shouldSign` returns false at the **first** domain gate (`dkim.go:L287-L288`) — debug `not signing, From domain is not key domain {from_domain:other.example, key_domain:example.org}`. Note this is an **earlier** gate than Scenario A (which fails at the auth-identity gate `L306`).
- *Does it cover `From`?* Not applicable — there is **no** `Dkim-Signature` header in the stored message (verifier `NO DKIM-Signature found`; header count `0`). (For contrast, when a signature *is* produced — Scenario C — the `h=` tag *does* cover `From`, oversigned as `From:From`.)
- *What does a verifying recipient see?* **No signature at all** — the verifier reports `NO DKIM-Signature found`.
- *Was it delivered?* **Yes**, accepted and stored — and the true envelope sender is still exposed in the `Received` header: `(envelope-sender <alice@example.org>)`, even though `From: stranger@other.example`. So a foreign `From:` sails through acceptance untouched (the pipeline only checked the *envelope* domain), and the only "protection" — a DKIM signature — is silently withheld.

### 7.5 Determinism (2 runs per scenario)

Each scenario was executed twice. The table below records, per run, the SMTP response codes, the sign decision, and the verifier verdict; the raw run-2 debug deltas follow. Only volatile fields (`msg_id`, ephemeral source port, `Message-Id`, timestamps) differ between runs — every server *behavior* is identical.

| Scenario | Run | `msg_id` | MAIL FROM | RCPT TO | DATA | sign decision (debug) | verifier |
|----------|-----|----------|-----------|---------|------|-----------------------|----------|
| A | 1 | `de7f4215` | `250` | `250` | `250 OK: queued` | not signing (auth identity, `L306`) | unsigned |
| A | 2 | `d929920b` | `250` | `250` | `250 OK: queued` | not signing (auth identity, `L306`) | unsigned |
| B | 1 | `d35022c9` | `250` | `501 5.1.8` | — (aborted) | n/a (rejected) | n/a |
| B | 2 | `1c5d029c` | `250` | `501 5.1.8` | — (aborted) | n/a (rejected) | n/a |
| C | 1 | `dafcb55f` | `250` | `250` | `250 OK: queued` | **signed** (`L408`) | signed but FAILS (`crypto/rsa`) |
| C | 2 | `8b1232cc` | `250` | `250` | `250 OK: queued` | **signed** (`L408`) | signed but FAILS (`crypto/rsa`) |
| D | 1 | `352444f8` | `250` | `250` | `250 OK: queued` | not signing (key domain, `L288`) | unsigned |
| D | 2 | `ceb80cdb` | `250` | `250` | `250 OK: queued` | not signing (key domain, `L288`) | unsigned |

**Raw run-2 `maddy -debug` deltas (verbatim):**
```
# A run 2
submission: incoming message	{"msg_id":"d929920b","sender":"bob@example.org","src_host":"test.local","src_ip":"127.0.0.1:38194","username":"alice@example.org"}
[debug] smtp/pipeline: sender bob@example.org matched by domain rule 'example.org'	{"msg_id":"d929920b"}
[debug] smtp/pipeline: recipient alice@example.org matched by domain rule 'example.org'	{"msg_id":"d929920b"}
sign_dkim: not signing, From address is not authenticated identity	{"auth_id":"alice@example.org","from_addr":"bob@example.org","msg_id":"d929920b"}
submission: accepted	{"msg_id":"d929920b"}

# B run 2
submission: incoming message	{"msg_id":"1c5d029c","sender":"x@notlocal.example","src_host":"test.local","src_ip":"127.0.0.1:55634","username":"alice@example.org"}
[debug] smtp/pipeline: sender x@notlocal.example matched by default rule	{"msg_id":"1c5d029c"}
[debug] smtp/pipeline: recipient alice@example.org matched by default rule (clean = alice@example.org)	{"msg_id":"1c5d029c"}
submission: RCPT error	{"effective_rcpt":"alice@example.org","rcpt":"alice@example.org","reason":"reject directive used","smtp_code":501,"smtp_enchcode":"5.1.8","smtp_msg":"Non-local sender domain"}
submission: aborted	{"msg_id":"1c5d029c"}

# C run 2
submission: incoming message	{"msg_id":"8b1232cc","sender":"alice@example.org","src_host":"test.local","src_ip":"127.0.0.1:36494","username":"alice@example.org"}
[debug] smtp/pipeline: sender alice@example.org matched by domain rule 'example.org'	{"msg_id":"8b1232cc"}
[debug] smtp/pipeline: recipient bob@example.org matched by domain rule 'example.org'	{"msg_id":"8b1232cc"}
submission: RCPT ok	{"msg_id":"8b1232cc","rcpt":"bob@example.org"}
[debug] sign_dkim: signed	{"identifier":"alice@example.org"}
submission: accepted	{"msg_id":"8b1232cc"}

# D run 2
submission: incoming message	{"msg_id":"ceb80cdb","sender":"alice@example.org","src_host":"test.local","src_ip":"127.0.0.1:55204","username":"alice@example.org"}
[debug] smtp/pipeline: sender alice@example.org matched by domain rule 'example.org'	{"msg_id":"ceb80cdb"}
[debug] smtp/pipeline: recipient bob@example.org matched by domain rule 'example.org'	{"msg_id":"ceb80cdb"}
submission: RCPT ok	{"msg_id":"ceb80cdb","rcpt":"bob@example.org"}
sign_dkim: not signing, From domain is not key domain	{"from_domain":"other.example","key_domain":"example.org","msg_id":"ceb80cdb"}
submission: accepted	{"msg_id":"ceb80cdb"}
```

The two Scenario C runs both produced a stored `Dkim-Signature` (`bob` INBOX UID 1 at `00:20:52`, UID 2 at `00:20:53` — confirmed by `imap-msgs list bob@example.org INBOX`), and both fail verification identically. **No run-to-run variance in observed server behavior.**

### 7.6 The `Received` header and the envelope sender

Every delivered message's `Received` header (grounded in `GenerateReceived`, `internal/target/received.go:L19`) has the form:

```
Received:  by example.org (envelope-sender <MAIL FROM>) with ESMTPS id <id>; <date>
```

Observations tied to the code:
- The **`(envelope-sender <…>)`** clause (`received.go:L69-L71`) always embeds the true envelope `MAIL FROM` — so a delivered message reveals the real envelope sender even when `From:` differs (vivid in Scenario A: `envelope-sender <bob@example.org>`, and Scenario D: `envelope-sender <alice@example.org>` with `From: stranger@other.example`).
- The leading **`from <host> (<rdns> [<ip>])`** clause (`received.go:L30-L31`) is **absent** because `submissionPrepare` sets `DontTraceSender = true` (`submission.go:L28`).
- The protocol token is **`ESMTPS`** (`smtp.go:L692`), confirming the plan's Correction #2 (there is no `ESMTPSA`, §9.2).

### 7.7 Edge / error paths (submission `DATA`-phase validation)

Beyond the four happy/adversarial scenarios, the `submissionPrepare` validations from §5 were exercised directly. All three surface at the **`DATA`** phase (after `MAIL FROM`/`RCPT TO` both return `250`), carry `"modifier":"submission_prepare"`, and abort the transaction — confirming the §5 enumeration with live output.

**Edge 1 — missing `From:` header** (`submission.go:L39-L48`). Full transcript tail + debug:
```
 ~> MAIL FROM:<alice@example.org>
<~  250 2.0.0 Roger, accepting mail from <alice@example.org>
 ~> RCPT TO:<bob@example.org>
<~  250 2.0.0 I'll make sure <bob@example.org> gets this
 ~> DATA
<~  354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
 ~> To: bob@example.org
 ~> Subject: edge case - missing From header
 ~> Date: Tue, 07 Jul 2026 00:30:00 +0000
 ~> Message-Id: <nofrom-test@example.org>
 ~> 
 ~> This message intentionally has no From header.
 ~> .
<~* 554 5.6.0 Message does not contains a From header field (msg ID = f5037673)
SWAKS_EXIT=26
--- maddy -debug ---
submission: DATA error	{"modifier":"submission_prepare","msg_id":"f5037673","reason":"Message does not contains a From header field","smtp_code":554,"smtp_enchcode":"5.6.0","smtp_msg":"Message does not contains a From header field"}
submission: aborted	{"msg_id":"f5037673"}
```

**Edge 2 — invalid `From:` address syntax** (`submission.go:L83-L95`, `mail.ParseAddressList`):
```
 ~> DATA
<~* 554 5.6.0 Invalid address in From (msg ID = 9eae9fbb)
SWAKS_EXIT=26
--- maddy -debug ---
submission: DATA error	{"addr":"not-an-email","modifier":"submission_prepare","msg_id":"9eae9fbb","reason":"mail: missing '@' or angle-addr","smtp_code":554,"smtp_enchcode":"5.6.0","smtp_msg":"Invalid address in From"}
submission: aborted	{"msg_id":"9eae9fbb"}
```

**Edge 3 — invalid `To:` address syntax** (`submission.go:L66-L81`, the `To/Cc/Bcc/Reply-To` loop):
```
 ~> DATA
<~* 554 5.6.0 Invalid address in To (msg ID = 8cfcb5c0)
SWAKS_EXIT=26
--- maddy -debug ---
submission: DATA error	{"addr":"@@@notvalid","modifier":"submission_prepare","msg_id":"8cfcb5c0","reason":"mail: no angle-addr","smtp_code":554,"smtp_enchcode":"5.6.0","smtp_msg":"Invalid address in To"}
submission: aborted	{"msg_id":"8cfcb5c0"}
```

These confirm that submission-time validation is **syntactic/structural** (presence of `From`, address parseability, `Sender` uniqueness, `Date` well-formedness) and never an identity check — reinforcing §8.2. (Auth is also mandatory: with no `AUTH`, the endpoint rejects at `MAIL FROM`/`RCPT` because `authAlwaysRequired = true`, `smtp.go:L590`/`L662`.)

---


## 8. Analysis — the five analytical questions

This section leads with the direct answer to each question, then layers the observed evidence and `file:line` grounding. Every factual claim is backed by a runtime observation from §7 or a code reference; statements reasoned from reading rather than observed are explicitly marked **(inferred)**. The five questions are: (8.1) how accept/reject is decided; (8.2) default-vs-explicit sender binding; (8.3) ruling out a plausible-but-wrong interpretation; (8.4) behaviors that differ from a reasonable config reading; and (8.5) `Authentication-Results` provenance.

### 8.1 How does Maddy decide to accept or reject a sender address from an authenticated session?

**Direct answer:** Acceptance is decided **solely by the domain of the envelope `MAIL FROM`**, via the message pipeline's *source routing*. The message `From:` header is **never consulted** for the accept/reject decision, and the **authenticated identity is never compared** against the sender at accept time.

**Mechanism (grounded).** The submission endpoint hands the cleaned envelope sender to the pipeline's `start` (`internal/msgpipeline/msgpipeline.go:L102`), which calls `srcBlockForAddr` (`msgpipeline.go:L155`). That function normalizes the address (`address.ForLookup`, `L159`) and selects a source block by trying, in order:

1. an **exact address** match `perSource[cleanFrom]` (`msgpipeline.go:L171`) — debug `matched by address rule` (`L199`);
2. a **domain** match `perSource[domain]` after `address.Split` (`msgpipeline.go:L174`, `L190`) — debug `matched by domain rule '%s'` (`L196`);
3. the **`defaultSource`** fallback (`msgpipeline.go:L193`) — debug `matched by default rule` (`L194`).

The matched block's rejection (if any) is returned by `start` (`msgpipeline.go:L117-L119`). In the canonical config there is **one** `source $(local_domains) { … }` block (`maddy.conf:L97`) plus `default_source { reject 501 5.1.8 "Non-local sender domain" }` (`maddy.conf:L117-L119`). So the decision reduces to: *is the `MAIL FROM` domain in `local_domains` (= `example.org`)?* If yes → accept; if no → `501 5.1.8`.

**Observed confirmation (from §7):**
- Scenario A (`MAIL FROM bob@example.org`) and Scenario C (`MAIL FROM alice@example.org`) both matched the domain rule (`sender … matched by domain rule 'example.org'`) and were **accepted** — even though A's sender is a *different user* than the authenticated `alice`.
- Scenario B (`MAIL FROM x@notlocal.example`) matched the default rule and was **rejected** `501 5.1.8 "Non-local sender domain"`.
- Scenario D (`MAIL FROM alice@example.org`, `From: stranger@other.example`) was **accepted** — proving the `From:` header plays no part in acceptance; only the envelope domain mattered.

The decision is therefore **domain-scoped**, not user-scoped and not header-scoped.

### 8.2 Does the DEFAULT config enforce sender alignment with the authenticated identity, or is explicit policy required?

**Direct answer:** The default configuration enforces alignment **only at the domain level**. It does **not** bind the envelope `MAIL FROM` or the message `From:` to the authenticated **username**. Per-user sender enforcement is **not available in this version at all** — there is no check module that could express it — so it cannot be enabled even with explicit config short of writing a `check.command` shim.

**Evidence 1 — no `authorize_sender` check exists at this commit.** A directory listing of `internal/check/` at HEAD `26452dd` contains only:

```console
$ ls internal/check/
action.go  command  dkim  dns  dnsbl  requiretls  skeleton.go  spf  stateless_check.go
```

The subdirectories `command`, `dkim`, `dns`, `dnsbl`, `requiretls`, `spf` are the only check modules; `action.go`, `skeleton.go`, `stateless_check.go` are infrastructure, not checks. **There is no `authorize_sender`.** (Modern Maddy ships `check.authorize_sender`, which maps the authenticated user to permitted sender addresses via a table — that module is a later addition; see §9.4.)

**Evidence 2 — the submission preparation step performs no sender-vs-auth comparison.** `submissionPrepare` (`internal/endpoint/smtp/submission.go:L27`) is the only submission-specific message hook. As fully enumerated in §5, it does only presence/uniqueness/syntax validation of headers: `DontTraceSender = true` (`L28`); `Message-Id` if absent (`L30-L36`); require `From:` else `554 5.6.0` (`L39-L48`); syntax-check `Sender` (`L50-L65`), `To/Cc/Bcc/Reply-To` (`L66-L81`), and `From` (`L83-L95`); require `Sender:` for multiple `From:` (`L99-L109`); and `Date` handling (`L111-L129`). **It never reads `AuthUser` and never compares any sender field to the authenticated identity.**

**Evidence 3 — the authenticated identity is consumed in only four non-test places, none of which is an accept-time sender check.** A grep for `AuthUser` across `internal/` (excluding tests) resolves to: session state/logging in `smtp.go` (`AuthUser: username`, `L680`); relay credentials in `smtp_downstream/sasl.go`; the `{auth_user}` placeholder in `check/command/command.go`; and the **DKIM signing-identity** comparison in `modify/dkim/dkim.go:L345`. The last is a *signing* decision (§8.4), not an acceptance control.

**Conclusion.** With the default config, an authenticated user can send with **any** envelope `MAIL FROM` and **any** `From:` header whose *domain* is local; the *username* is irrelevant to acceptance. Per-user binding would require explicit configuration — and in this version, the only way to express it would be an out-of-the-box-absent module or a hand-rolled `check.command` matching `{auth_user}` against `{sender}`. The default ships neither.

### 8.3 Ruling out a plausible-but-incorrect interpretation

**The tempting (wrong) interpretation:** *"Because submission requires authentication and the DKIM modifier is configured with `require_sender_match envelope auth`, an authenticated user can only send as themselves."*

**This is false, and Scenarios A and D disprove it directly:**
- **Scenario A** — authenticated as `alice`, sent with `MAIL FROM: bob@example.org` and `From: bob@example.org`. Result: **accepted and delivered** (stored in `alice`'s INBOX, dumped in §7.1). The server never objected that `bob ≠ alice`. The *only* consequence of the mismatch was that the message went out **unsigned** — delivery itself was unaffected.
- **Scenario D** — authenticated as `alice`, `From: stranger@other.example`. Result: **accepted and delivered** (stored as `bob` INBOX UID 3, §7.4) despite the `From:` matching neither the authenticated user nor any local domain.

Both scenarios show acceptance succeeding while the sender does not match the authenticated user — so `require_sender_match` is **not** an acceptance gate, and no other mechanism enforces sender==auth. The interpretation is ruled out by observation.

**External corroboration (context, not primary evidence).** foxcpp/maddy **Issue #268** ("A check to only allow MAIL FROM same as the authenticated user's username"), opened **2020-08-28**, documents precisely this default behavior — that with the default configuration an authenticated user can send from any same-domain address — and requests per-user enforcement as a **new feature**. The pinned commit `26452dd` is dated **2019-12-13**, roughly **eight months before** that issue was filed, so the per-user check demonstrably did not exist here. The primary evidence remains the observed A/D runs and the `internal/check/` listing; the issue is cited only to corroborate that this was recognized upstream as a *missing* capability at the time.

### 8.4 Behaviors that differ from a reasonable reading of the configuration

Three observed divergences between "what the config looks like it does" and "what the runtime actually does":

**Divergence #1 — `require_sender_match` gates SIGNING, not ACCEPTANCE.** The directive name and its default value `{envelope, auth}` (`internal/modify/dkim/dkim.go:L151-L152`) read like an authorization control that would *reject* a mismatched sender. In reality it only decides **whether to apply a DKIM signature**; a message that fails the match is still **accepted and delivered — just unsigned** (Scenario D, and the auth-mismatch branch in Scenario A). The code makes this explicit: the mismatch branches in `shouldSign` (`dkim.go:L288`, `L294`, `L306`) merely `return false` (skip signing); they never produce an SMTP error. The manual corroborates the intent — `docs/man/maddy-filters.5.scd:L518` gives the syntax, `L519` states `Default: envelope auth`, and `L521-L522` describe requiring the identifiers to match, "otherwise — don't sign the message." **(This is a signing policy masquerading, by name, as an acceptance policy.)**

**Divergence #2 — the `501 5.1.8` sender rejection surfaces at `RCPT TO`, not at `MAIL FROM`.** A naive reading of `default_source { reject 501 5.1.8 … }` would expect the rejection the moment a foreign `MAIL FROM` is issued. Instead (Scenario B) the server returns `250` to `MAIL FROM` and defers the `501` to the **first `RCPT TO`**. This is because `defer_sender_reject` defaults to **true** (`internal/endpoint/smtp/smtp.go:L567`): `Session.Mail` (`L162`, guarded by `if !s.endp.deferServerReject` at `L163`) simply records the sender and returns `250`; the deferred `startDelivery` runs from the first `Session.Rcpt` (`L208`, `L220`), and on error is wrapped and returned there (`L225-L226`, debug `MAIL FROM error (deferred)` at `L223`). The observed transcript in §7.2 shows exactly this `250`→`501` timing shift.

**Divergence #3 (the most consequential) — DKIM signatures produced by this build do NOT verify at any RFC-6376 verifier.** A reasonable operator reading `sign_dkim` in the config would assume that a "signed" message (Scenario C) verifies as authentic at the recipient — and the plan itself *predicted* a verifier PASS. It does not, and the prediction is disconfirmed by observation. As proven in §6.3 with maddy's own key and libraries against the exact stored bytes, Scenario C's signature fails for **two independent reasons**:

1. **The published public key is PKCS#1, not PKIX/SubjectPublicKeyInfo.** `writeDNSRecord` emits `x509.MarshalPKCS1PublicKey(pubkey)` (`internal/modify/dkim/keys.go:L143`), producing a `p=` value with the PKCS#1 prefix `MIIBCgKCAQEA…`. RFC 6376 requires the SubjectPublicKeyInfo (PKIX) encoding (`MIIBIjANBgkqhkiG9w0BAQEF…`). go-msgauth rejects the record outright: `dkim: key syntax error: x509: failed to parse public key (use ParsePKCS1PublicKey instead for this key format)`.
2. **Even after hand-correcting the record to PKIX, the RSA header check still fails** (`crypto/rsa: verification error`) while the body hash (`bh=`) matches exactly. Root cause: go-msgauth's `foldHeaderField` hard-wraps the emitted `DKIM-Signature` every 75 bytes **regardless of token boundaries**, but maddy's integration (`RewriteBody`, `dkim.go:L406`, `h.Add("DKIM-Signature", signer.SignatureValue())`) hashes the self-field via `NewSigner`/`textproto.WriteHeader` and then re-emits it through a *different* folder — so the bytes the signer hashed differ from the bytes a verifier reconstructs after relaxed-canonicalization. The controlled round-trip in §6.3 (`rt_maddy` → invalid; `rt_allinone` and `rt_fix` → valid, same key/options/libs) isolates this precisely.

This divergence is a genuine security-relevant finding: the server reports success and stamps a `Dkim-Signature`, but a real recipient sees either a malformed key record or a hard verification failure. **(inferred)** that all DKIM signatures emitted by this commit are affected, since the two defects are in the shared key-writing (`keys.go:L143`) and `RewriteBody` (`dkim.go:L406`) code paths, not in anything scenario-specific; **directly observed** for the submission→local-delivery path here. Per AAP §0.3.2 the fix is out of scope; this document reports and reconciles the divergence rather than modifying the product (§6.3).

### 8.5 `Authentication-Results` provenance

**Direct answer:** No `Authentication-Results` header appears on any message submitted in Scenarios A/C/D — and that is expected. That header is written by the **inbound** verification checks (`verify_dkim`, `apply_spf`, `dmarc yes`) that are attached to the port-25 receive path in the canonical config (`maddy.conf:L62`, `L65`, `L70`), **not** by the submission/signing path. Messages injected through the authenticated `submission` endpoint never traverse those inbound checks, so no `Authentication-Results` is added.

**Observed confirmation:** the raw stored headers for A, C, and D (§7.1, §7.3, §7.4) contain `Delivered-To`, `Return-Path`, `Received`, and (for C) `Dkim-Signature` — but **no** `Authentication-Results` header in any of them (explicit `grep -ci '^Authentication-Results'` returned `0` for all three). This matches the code path: the submission pipeline's only header mutations are `submissionPrepare` (§5) plus the DKIM `RewriteBody`, neither of which emits `Authentication-Results`. **(The claim that the *inbound* path would add it is inferred from the config wiring and was not exercised here, since the investigation drove the submission path exclusively.)**

---

## 9. Corrections & caveats

### 9.1 Correction — account creation is `users create`, NOT `creds create`

The upstream plan referenced `maddyctl creds create`. At commit `26452dd` that command **does not exist**. The CLI defines a `users` command (`cmd/maddyctl/main.go:L47`) with a `create` subcommand (`L71`) that reads the password from `--password/-p` or stdin (also `--null/-n`, `--hash` default `bcrypt`, `--bcrypt-cost`, `--cfg-block` default `local_authdb`). There is **no `creds` command** in this version — the document uses the verified `users create` throughout (§3.3). `creds create` is *modern-Maddy* phrasing (modern Maddy separates `creds create` + `imap-acct create`); this version unifies auth and storage through the shared `imapsql` module.

### 9.2 Correction — the `Received` protocol token is `ESMTPS`, NOT `ESMTPSA`

The plan's illustrative `Received` line used `with ESMTPSA`. At this commit the protocol token is computed in `internal/endpoint/smtp/smtp.go` (`L686-L695`) as one of `"LMTP"` (`L687`), `"ESMTPS"` (`L692`, when the TLS handshake completed), or `"ESMTP"` (`L694`). **There is no `ESMTPSA` token in this version.** On the implicit-TLS `:465` submission port the observed header is `with ESMTPS` (see the raw stored headers in §7.1/§7.3/§7.4 and the `Received` analysis in §7.6).

### 9.3 Statements labeled "(inferred)"

For full transparency, the following statements in this document are **inferred from reading code/config** rather than directly observed by running, and are labeled as such at their point of use:

- That the `Authentication-Results` header would be **added on the inbound port-25 path** by `verify_dkim`/`apply_spf`/`dmarc` (§8.5). Only its **absence on the submission path** was directly observed; the inbound path was not exercised.
- That **all** DKIM signatures emitted by this commit (not merely the submission→local-delivery path) are affected by the two verification defects (§6.3, §8.4). The defect was directly observed for the exercised path and reasoned to be general because it lives in shared key-writing (`keys.go:L143`) and `RewriteBody` (`dkim.go:L406`) code.
- That maddy's upstream CI does not catch the DKIM defect (§6.3). This is reasoned from `dkim_test.go` containing only `TestFieldsToSign`/`TestShouldSign` (no round-trip sign→verify test); the CI run itself was not reproduced.

Everything else in this document is backed by observed runtime output (SMTP transcripts, `maddy -debug` lines, raw stored headers, verifier verdicts) together with a `file:line` reference to the code that produces it.

### 9.4 Version-fidelity caveat — modern `authorize_sender` is context only

Modern Maddy ships a `check.authorize_sender` module that binds the authenticated user to a set of permitted sender addresses (via a `user_to_email` table/chain), and modern default configs place it inside the submission `source` block. **That module does not exist at commit `26452dd`** (§8.2, Evidence 1) and was **not** back-ported, enabled, or simulated for this investigation. It is reported here purely as historical context to frame the "default-vs-explicit" question; every observed behavior in this document reflects the pinned version **as-is**. All conclusions are scoped to commit `26452dd8dd787dc455278b0fdd296f4a5432c768`.

### 9.5 Documented environment deviations from the canonical config

These are the only deviations from the shipped `maddy.conf`, each an environment necessity (detailed in §2.1, and visible verbatim in the §3.1 config), and none affecting the observed sender/DKIM behavior:

1. **Relocated state** — a global `state /tmp/maddy-run/state` directive so `os.Chdir(config.StateDirectory)` (`maddy.go:L220`) moves `all.db`, `dkim_keys/`, `queue/`, and `mtasts-cache/` outside the checkout (`.gitignore` does **not** list `all.db`, so relocation is required for byte-for-byte cleanliness).
2. **Self-signed TLS** — the canonical `tls file /etc/maddy/certs/$(hostname)/…` (`maddy.conf:L16-L17`) points at paths absent in the sandbox; a self-signed cert/key for `example.org` was substituted so the implicit-TLS `:465` port can start.
3. **Test-only plaintext port** — the submission block additionally binds `tcp://0.0.0.0:1587` with `insecure_auth`, used only to allow a byte-faithful packet capture; explicitly labeled test-only, and **not used by any of the four scenarios** (all driven over `:465`). The primary evidence is `maddy -debug` + the `swaks` transcript over the real `:465` port.

---


## 10. Cleanup & read-only proof

The entire experiment was conducted with every working artifact confined to `/tmp/maddy-run/` (outside the checkout). No existing repository file was modified; the only repository change is the single new file `blitzy/documentation/maddy_26452dd8dd78.md`.

### 10.1 Read-only proof (relative to the code under test)

The investigation branch differs from the commit under test (`26452dd`) by exactly this one added document; no tracked source, config, or dependency file changed. Actual output:

```console
$ git rev-parse --abbrev-ref HEAD
blitzy-f454e013-443c-41bf-a239-e2757361b5fc
$ git rev-parse 26452dd^{commit}                    # the commit under test (pinned by hash)
26452dd8dd787dc455278b0fdd296f4a5432c768
$ git merge-base --is-ancestor 26452dd HEAD && echo "26452dd is an ancestor of HEAD"
26452dd is an ancestor of HEAD
$ git diff --name-status 26452dd..HEAD              # what changed vs the code under test
A	blitzy/documentation/maddy_26452dd8dd78.md
$ git diff --stat 26452dd -- go.mod go.sum          # dependency manifests
                                                     # (empty output: go.mod / go.sum unchanged)
$ git log 26452dd..HEAD --format='%ae' | sort -u    # authors of every commit since the code under test
agent@blitzy.com
```

`git diff --name-status 26452dd..HEAD` shows a single `A` (added) entry — the documentation — and **no** ` M`/` D`/`R` entry for any tracked file. The empty `git diff` for `go.mod`/`go.sum` confirms **no dependency change**. Every commit layered on top of the code under test is an `agent@blitzy.com` documentation commit (the branch carries one or more such commits that only ever touch this one file, so the net `26452dd..HEAD` difference remains exactly the single added document). The runtime state (`/tmp/maddy-run/`) is a sibling of the checkout and therefore never appears in git status.

### 10.2 Cleanup commands (teardown)

After all evidence was captured, the running processes were stopped and the entire working directory removed so the environment is left clean:

```console
# Stop the running maddy server and the offline DNS helper used for verification
$ kill "$(cat /tmp/maddy-run/maddy.pid)"     # the specific PID recorded at launch
$ pkill -f dnsserver.py 2>/dev/null || true  # local UDP DNS resolver (verification helper)

# Remove the entire isolated runtime state, all scripts, transcripts, keys, certs, and the DB
$ rm -rf /tmp/maddy-run
```

Removing `/tmp/maddy-run` deletes: the test config; the self-signed `certs/`; the SQLite store `state/all.db`; the auto-generated `state/dkim_keys/example.org_default.{key,dns,pkix.dns}`; `state/{queue,mtasts-cache}/`; the compiled `bin/{maddy,maddyctl}`; the `swaks` scenario scripts and `*.log`/`*.eml` transcripts; the Go verifier/round-trip harness (`rttest/`); and the Python DKIM/DNS helper scripts (`verify_dkim.py`, `bodyhash_isolation.py`, `dnsserver.py`). Nothing from the experiment persists.

### 10.3 Final read-only proof (post-cleanup)

After stopping all processes and removing `/tmp/maddy-run`, the checkout shows the new answer document as the sole change and **zero** modifications to any tracked file:

```console
$ git status --porcelain                          # working tree state (empty output below = clean; the document is committed)
$ git diff --name-status 26452dd..HEAD            # the sole net change vs the code under test
A	blitzy/documentation/maddy_26452dd8dd78.md
$ git diff --stat 26452dd..HEAD -- ':!blitzy/documentation'   # any pre-existing tracked file changed?
                                                  # (empty output: none — no source/config/go.mod/go.sum change)
```

`/tmp/maddy-run` was a sibling of the checkout, so its removal leaves the working tree byte-for-byte unchanged apart from adding this document. The compiled `maddy`/`maddyctl` binaries lived under `/tmp/maddy-run/bin` (outside the tree); even had they been built in-tree, `.gitignore` already excludes `cmd/maddy/maddy` and `cmd/maddyctl/maddyctl`. No dependency files (`go.mod`/`go.sum`) were touched, and the `authorize_sender` check was **not** added, enabled, or back-ported.

---

## 11. Reference index (`file:line` → function/struct)

All references verified against a pristine checkout at HEAD `26452dd8dd787dc455278b0fdd296f4a5432c768`. Each row names the specific function/struct/directive the citation anchors; every line number is exact.

### 11.1 `maddy.conf` (canonical config)

| Anchor | What it is |
|---|---|
| `maddy.conf:L16-L17` | `tls file /etc/maddy/certs/$(hostname)/fullchain.pem privkey.pem` (overridden to self-signed in the test config) |
| `maddy.conf:L33` | `driver sqlite3` (inside `storage.imapsql local_mailboxes { … }`) |
| `maddy.conf:L34` | `dsn all.db` (relative → resolves inside the state dir) |
| `maddy.conf:L53` | inbound `smtp tcp://0.0.0.0:25` |
| `maddy.conf:L62,L65,L70` | inbound checks `verify_dkim` (L62), `apply_spf` (L65), `dmarc yes` (L70) — source of `Authentication-Results` on the inbound path |
| `maddy.conf:L74-L76` | inbound `source $(local_domains) { reject … "Use Submission for outgoing SMTP" }` |
| `maddy.conf:L93` | **`submission tls://0.0.0.0:465`** |
| `maddy.conf:L95` | `auth &local_authdb` |
| `maddy.conf:L97` | `source $(local_domains) { … }` (the accept branch) |
| `maddy.conf:L99` | `modify { sign_dkim $(primary_domain) default }` |
| `maddy.conf:L104` | submission `destination $(local_domains) { … deliver_to &local_mailboxes }` |
| `maddy.conf:L117-L119` | **`default_source { reject 501 5.1.8 "Non-local sender domain" }`** |
| `maddy.conf:L149` | `imap tls://0.0.0.0:993` |

### 11.2 `internal/endpoint/smtp/smtp.go` (submission session)

| Anchor | Function/struct |
|---|---|
| `L83` | `startDelivery` — begins delivery for a session |
| `L162-L163` | `Session.Mail` — `MAIL FROM`; with defer=true (`if !s.endp.deferServerReject`, `L163`) stores sender, returns `250` |
| `L208` | `Session.Rcpt` — first `RCPT` runs deferred `startDelivery` |
| `L220` | deferred `startDelivery` invocation from `Rcpt` |
| `L223` | deferred error logged `MAIL FROM error (deferred)` |
| `L225-L226` | deferred error wrapped/returned at `RCPT` |
| `L292` | `submissionPrepare` call site (`s.submissionPrepare(s.msgMeta, &header)`) |
| `L469` / `L472` | struct fields `authAlwaysRequired` / `deferServerReject` |
| `L567` | **`defer_sender_reject` default `true`** (`cfg.Bool("defer_sender_reject", false, true, …)`) |
| `L590` | submission endpoint sets `authAlwaysRequired = true` |
| `L662` | auth requirement enforced |
| `L680` | `AuthUser: username` written into connection/session state |
| `L686-L695` | Proto computation → `"LMTP"` (L687) / `"ESMTPS"` (L692) / `"ESMTP"` (L694) — **no `ESMTPSA`** |

### 11.3 `internal/endpoint/smtp/submission.go` (`submissionPrepare`)

| Anchor | What it does |
|---|---|
| `L27` | `submissionPrepare` function |
| `L28` | **`msgMeta.DontTraceSender = true`** (suppresses the `from …` clause in `Received`) |
| `L30-L36` | generate `Message-Id` if absent (log `adding missing Message-ID` at `L35`) |
| `L39-L48` | require `From:` → `554 5.6.0 "Message does not contains a From header field"` if missing |
| `L50-L65` | validate `Sender:` syntax with `mail.ParseAddress` (`L52`) → `554 5.6.0 "Invalid address in Sender"` |
| `L66-L81` | validate `To`/`Cc`/`Bcc`/`Reply-To` with `mail.ParseAddressList` (`L68`) → `554 5.6.0 "Invalid address in <hdr>"` |
| `L83-L95` | parse `From:` with `mail.ParseAddressList` (`L83`) → `554 5.6.0 "Invalid address in From"` |
| `L99-L109` | require `Sender:` if multiple `From:` addresses → `554 5.6.0 "Missing Sender header field"` |
| `L111-L123` | validate `Date:` (`parseMessageDateTime`) → `554 "Malformed Date header"` |
| `L124-L129` | add missing `Date:` (log `adding missing Date header` at `L125`) |
| — | **no sender-vs-auth comparison anywhere in `L27-L130`** |

### 11.4 `internal/msgpipeline/msgpipeline.go` (source routing)

| Anchor | Function/struct |
|---|---|
| `L102` | `start` — pipeline entry for a transaction |
| `L117-L119` | return matched block's `rejectErr` (debug `sender %s rejected with error` at `L118`) |
| `L155` | **`srcBlockForAddr`** — selects the source block for the cleaned sender |
| `L159` | `address.ForLookup` normalization |
| `L171` | `perSource[cleanFrom]` (exact-address match) |
| `L174` | `address.Split` (extract domain) |
| `L190` | `perSource[domain]` (domain match) |
| `L193` | `defaultSource` fallback |
| `L194` | debug `matched by default rule` |
| `L196` | debug `matched by domain rule '%s'` |
| `L199` | debug `matched by address rule '%s'` |

### 11.5 `internal/modify/dkim/dkim.go` (signing decision)

| Anchor | What it is |
|---|---|
| `L31-L54` | `oversignDefault` — 15 oversigned headers (Subject, Sender, To, Cc, From, Date, MIME-Version, Content-Type, Content-Transfer-Encoding, Reply-To, In-Reply-To, Message-Id, References, Autocrypt, Openpgp) |
| `L55-L72` | `signDefault` |
| `L137` | `key_path` default `dkim_keys/{domain}_{selector}.key` |
| `L140-L142` / `L143-L145` | header / body canonicalization defaults `relaxed` / `relaxed` |
| `L147-L148` | `hash sha256` default |
| `L149-L150` | `newkey_algo rsa2048` default |
| `L151-L152` | **`require_sender_match` default `{envelope, auth}`** |
| `L202` | `fieldsToSign` — lists each header (occurrences+1) times |
| `L249` | **`shouldSign`** — the signing gate |
| `L266` / `L271` / `L275` | gates: empty From / malformed From / multiple From |
| `L287-L288` | **From-domain ≠ key-domain gate (Scenario D fails here)** — debug `not signing, From domain is not key domain` |
| `L293-L297` | envelope-mismatch gate — debug `not signing, From address is not envelope address` (`L294`; not triggered by any of the four scenarios) |
| `L299-L306` | **auth-identity gate (Scenario A fails at L306)** — `strings.EqualFold` at `L305`; debug `not signing, From address is not authenticated identity` at `L306` |
| `L340` | `RewriteBody` |
| `L345` | `authUser = s.meta.Conn.AuthUser` |
| `L348` | `shouldSign` call (passes `OriginalFrom`) |
| `L406` | **`h.Add("DKIM-Signature", signer.SignatureValue())` (Scenario C signs here; the emission-fold defect)** |
| `L408` | `signed` debug line |

### 11.6 `internal/modify/dkim/keys.go` (key auto-generation)

| Anchor | What it is |
|---|---|
| `L19` | `loadOrGenerateKey` |
| `L77` | `generateAndWrite` |
| `L82` | log `generating a new rsa2048 keypair...` |
| `L95` | `rsa.GenerateKey(…, 2048)` |
| `L105` | `x509.MarshalPKCS8PrivateKey` (private key = PKCS#8) |
| `L121-L131` | PEM write, `0600`, block type `"PRIVATE KEY"` |
| `L136` | `writeDNSRecord` |
| `L143` | **`x509.MarshalPKCS1PublicKey(pubkey)` — the PKCS#1 (non-PKIX) public-key defect** |
| `L150-L152` | `.dns` path: `keyPath + ".dns"` (L150), replaced by `keyPath[:len-4] + ".dns"` when `keyPath` ends in `.key` (guard at L151, strip at L152) → yields `example.org_default.dns` |
| `L158` | record string `v=DKIM1; k=rsa; p=<base64>` |

### 11.7 `internal/target/received.go` (`Received` header)

| Anchor | What it is |
|---|---|
| `L19` | `GenerateReceived` |
| `L30-L31` | `from <host> (<rdns> [<ip>])` clause guarded by `!DontTraceSender` (suppressed on submission) |
| `L62-L63` | `by <hostname>` |
| `L69-L71` | **`(envelope-sender <MAIL FROM>)`** — always embeds the true envelope sender |
| `L74-L80` | `with <Proto>` |
| `L81-L84` | `id <msgID>; <date>` |

### 11.8 Other references

| Anchor | What it is |
|---|---|
| `internal/check/` (dir) | contains only `command`, `dkim`, `dns`, `dnsbl`, `requiretls`, `spf` (+ infra `action.go`, `skeleton.go`, `stateless_check.go`) — **no `authorize_sender`** |
| `cmd/maddyctl/main.go:L47` | `users` command |
| `cmd/maddyctl/main.go:L71` | `users create` subcommand (password via `--password/-p` or stdin; `--null/-n`, `--hash` default `bcrypt`) — **no `creds` command** |
| `cmd/maddyctl/main.go:L203,L207` | `imap-mboxes` / `imap-mboxes list` |
| `cmd/maddyctl/main.go:L307,L504,L534` | `imap-msgs` (L307) / `imap-msgs list` (L504) / `imap-msgs dump` (L534) |
| `maddy.go:L59` | `DefaultStateDirectory = "/var/lib/maddy"` |
| `maddy.go:L102` | `Run` (flags `-config`, `-debug`, `-log`) |
| `maddy.go:L220` | **`os.Chdir(config.StateDirectory)`** |
| `maddy.go:L245-L246` | `state` / `runtime` global directives |
| `examples/multitentant-dkim.conf:L23,L26,L46-L47` | domain/key-scoped `source example.org { sign_dkim … }` (L23/L26) + `default_source { reject }` (L46-L47) — corroborates domain-scoped handling |
| `docs/man/maddy-filters.5.scd:L518,L519,L521-L522,L535-L538` | `require_sender_match` syntax (L518), `Default: envelope auth` (L519), "otherwise — don't sign the message" (L521-L522), `auth` token semantics (L535-L538) |
| `go.mod:L1,L3` | module `github.com/foxcpp/maddy`, `go 1.13` |
| `go.mod:L16,L17,L26,L30` | `emersion/go-message v0.10.9-0.20191116124005-65fd0119e899` (L16); `emersion/go-msgauth v0.3.2-0.20191028231513-55b75676976c` (L17); `mattn/go-sqlite3 v1.11.0` — CGO (L26); `golang.org/x/crypto v0.0.0-20191108234033-bd318be0434a` (L30) |

---

## 12. Coverage pass

A final decomposition confirming every named item — each AAP setup property, each of the five analytical questions, each of the four scenarios, both plan corrections, and each user-specified Rule — is addressed with observed evidence.

### 12.1 AAP setup properties

| Property (AAP §0.1.1) | Where satisfied | Evidence |
|---|---|---|
| Auth-required submission on `:465` | §3.1, §7.7 | `auth &local_authdb`; `authAlwaysRequired` (`smtp.go:L590`/`L662`); AUTH PLAIN `235` in every transcript |
| DKIM signing enabled for a domain | §3.2, §7.3 | `sign_dkim $(primary_domain) default`; key auto-gen log; `sign_dkim: signed` |
| Local delivery to an inspectable mailbox | §3.1, §3.3, §7 | `deliver_to &local_mailboxes`; `imap-msgs dump` output for A/C/D |
| ≥ 2 accounts via `maddyctl` | §3.3 | `users create alice@…` / `bob@…`; `users list` |

### 12.2 The five analytical questions

| Question | Section | Direct answer |
|---|---|---|
| How is a sender accepted/rejected from an authenticated session? | §8.1 | Envelope-`MAIL FROM`-domain source routing only (`srcBlockForAddr`) |
| Default vs explicit sender binding? | §8.2 | Domain-level only by default; per-user impossible (no `authorize_sender` at this commit) |
| Rule out a plausible-but-wrong interpretation | §8.3 | "auth ⇒ send-as-self" is false — A and D accepted+delivered |
| A behavior differing from a reasonable config reading | §8.4 | Three divergences: signing-not-acceptance; deferred `RCPT` reject; **DKIM signatures don't verify** |
| `Authentication-Results` provenance | §8.5 | Absent on submitted mail (inbound-only header); confirmed by `grep`=0 |

### 12.3 The four user scenarios (preserved verbatim in §7)

| Scenario | Section | Observed outcome |
|---|---|---|
| (A) auth alice, `MAIL FROM` bob | §7.1 | Accepted + delivered; **unsigned** (auth-identity gate `L306`) |
| (B) `MAIL FROM` foreign domain | §7.2 | `250` at MAIL FROM, **`501 5.1.8` at RCPT TO**; not delivered |
| (C) legitimate aligned | §7.3, §6.3 | Accepted + delivered + **signed**, but signature **does not verify** (two defects) |
| (D) DKIM-triggering, foreign `From` | §7.4 | Accepted + delivered; **unsigned** (key-domain gate `L288`); no signature ⇒ `From` not covered; envelope sender exposed in `Received` |

### 12.4 Plan corrections and required rulings

| Item | Section | Ruling |
|---|---|---|
| `users create` not `creds create` | §9.1 | Confirmed: no `creds` command in this version |
| `Received` token `ESMTPS` not `ESMTPSA` | §9.2, §7.6 | Confirmed: `smtp.go:L692` |
| Predicted Scenario C "verifier PASS" | §6.3, §8.4 | Disconfirmed by observation; reconciled as the AAP-required security divergence (no product change) |

### 12.5 User-specified Rules (AAP §0.7, "SWE-AtlasQnA-Repo")

| Rule | Where honored |
|---|---|
| Deliverable at `blitzy/documentation/maddy_26452dd8dd78.md` | this file |
| Investigate by running the code first | all of §2–§7; every claim has captured output |
| Match magnitude/frequency to real observation; confirm stable across ≥2 runs | §7.5 determinism table + raw run-2 deltas |
| Exercise the real entry point (no bypass) | all sends via `swaks` over the real `:465` submission session |
| Default, canonical build/config; state exact commands | §2 build/run commands; §3.1 verbatim config; three deviations disclosed |
| Exercise every condition, not just the happy path | four scenarios (§7.1–§7.4) + edge/error paths (§7.7) |
| Include actual, complete, unedited output with the producing command | verbatim transcripts, debug windows, `imap-msgs dump`, verifier pipelines throughout |
| Honor directional wording (demonstrate; if it doesn't reproduce, report honestly) | §6.3 reports the DKIM non-verification honestly with reproduction |
| Show observed output for every claim; label inferred | `(inferred)` labels in §6.3/§8.4/§8.5, consolidated in §9.3 |
| Answer every part and every named item; finish with a coverage pass | §8 answers all five questions; this §12 is the coverage pass |
| Be exact and grounded (`file:line` + function/struct) | §11 exact reference index; inline citations throughout |
| Lead with the direct answer, then nuance | §1 TL;DR, and each §8 subsection leads with the direct answer |
| Read-only scope; remove temporary scripts afterward | §10 read-only proof + teardown |

---

*End of investigation. All observations were produced by building and running maddy at commit `26452dd8dd787dc455278b0fdd296f4a5432c768` inside the canonical container; every working artifact was removed afterward, leaving the repository byte-for-byte unchanged except for this document.*

