# Maddy sender-identity & DKIM enforcement on authenticated submission — a runtime investigation

**Subject under test:** `github.com/foxcpp/maddy`, branch `maddy_26452dd8dd78`, HEAD commit **`26452dd8dd787dc455278b0fdd296f4a5432c768`**.

**Nature of this document:** This is a **run‑first** investigation. Every factual claim below is backed by (a) **actual, unedited runtime output** — real `swaks` SMTP transcripts, `maddy -debug` log lines, raw stored message headers, and DKIM‑verifier results — captured by building and running the server, **and** (b) a **`file:line`** reference naming the specific function/struct that performs the work. Statements that were *reasoned from reading source but not directly observed* are explicitly labelled **(inferred)**.

**Read‑only compliance:** No repository file was modified. The server was built and run entirely from an isolated state directory under `/tmp/maddy-run`; all configs, TLS certs, the SQLite store, `swaks` scripts, DKIM verifiers, logs and packet captures are **working artifacts** created outside the checkout and deleted afterward. The only committed change from this task is this single document. The final `git status --porcelain` proof is in §10.

---

## 1. TL;DR — direct answers

1. **How does Maddy decide to accept or reject a sender from an authenticated session?** Acceptance is decided **solely by the envelope `MAIL FROM` domain** through source‑routing in the message pipeline — `srcBlockForAddr` (`internal/msgpipeline/msgpipeline.go:L155`) normalizes the cleaned sender and selects `perSource[full-addr]` → `perSource[domain]` → `defaultSource`. The message **`From:` header is never consulted** for the accept/reject decision, and neither is the authenticated username.

2. **Does the default config bind the sender to the authenticated identity, or is explicit policy required?** The default configuration enforces alignment **only at the domain level**. It does **not** bind `MAIL FROM` or `From:` to the authenticated **username**. There is **no `authorize_sender` check** in this version (the `internal/check/` directory contains only `command`, `dkim`, `dns`, `dnsbl`, `requiretls`, `spf`), and `submissionPrepare` performs **no** sender‑vs‑auth comparison. Per‑user binding would require explicit configuration that the default does not include.

3. **A plausible‑but‑wrong interpretation, ruled out:** "auth‑required submission + `require_sender_match envelope auth` means an authenticated user can only send as themselves" is **false**. Scenario **A** (auth `alice`, send as `bob`) and Scenario **D** (auth `alice`, `From: stranger@other.example`) are **both accepted and delivered**.

4. **`require_sender_match` gates SIGNING, not ACCEPTANCE** (divergence #1). Despite reading like an anti‑spoofing acceptance control, `require_sender_match` (default `[envelope auth]`, `internal/modify/dkim/dkim.go:L151-152`) only decides **whether to DKIM‑sign**. A mismatched message is still **accepted and delivered — unsigned** (Scenario D).

5. **The `501 5.1.8` sender rejection surfaces at `RCPT TO`, not `MAIL FROM`** (divergence #2), because `defer_sender_reject` defaults to **true** (`internal/endpoint/smtp/smtp.go:L567`). `MAIL FROM` of a foreign domain returns `250`; the first `RCPT TO` returns `501 5.1.8 "Non-local sender domain"`.

6. **Maddy's own DKIM signatures do NOT verify at a standards‑compliant recipient** (divergence #3 — the most consequential, and it contradicts the naive expectation that enabling `sign_dkim` yields verifiable mail). Two independent defects, both observed and reproduced with maddy's *own* libraries and key:
   - The public key is published as **PKCS#1** rather than the RFC‑6376‑required **SubjectPublicKeyInfo/PKIX** (`internal/modify/dkim/keys.go:L143`), so verifiers cannot even parse it.
   - Even after correcting the key to PKIX, the signature **still fails** (`crypto/rsa: verification error`) because the `DKIM-Signature` header is **hashed with one folding algorithm and emitted with another** (see §6.3). The body hash `bh=` is correct; the header/RSA check fails.

7. **`Authentication-Results` is absent on submitted mail.** That header is produced only by the **inbound** checks (`verify_dkim`/`apply_spf`/`dmarc`) on the port‑25 receive path, not by the submission/signing path — confirmed absent on every submitted message.

8. **Corrections to the plan, confirmed at runtime:** account creation is `maddyctl users create` (there is **no** `creds` command in this version), and the `Received` protocol token is **`ESMTPS`** (there is no `ESMTPSA`).

All outcomes were **deterministic** across ≥2 runs (§7.5).

---

## 2. Environment & exact commands

| Item | Value (observed) |
|------|------------------|
| Canonical image | `andrewparkscaleai/coding-agent:foxcpp__maddy__26452dd8dd787dc455278b0fdd296f4a5432c768` (from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_foxcpp_maddy_1.0`) |
| Commit under test | `26452dd8dd787dc455278b0fdd296f4a5432c768` |
| Go toolchain | `go version go1.18.10 linux/amd64` |
| C toolchain | gcc 10.2.1 (CGO required by `mattn/go-sqlite3 v1.11.0`, see `go.mod`) |

**Commit confirmation:**

```console
$ git -C <checkout> rev-parse HEAD
26452dd8dd787dc455278b0fdd296f4a5432c768
```

**Build (verbatim commands):**

```console
$ cd <checkout>
$ CGO_ENABLED=1 go build -o /tmp/maddy-run/bin/maddy    ./cmd/maddy
$ CGO_ENABLED=1 go build -o /tmp/maddy-run/bin/maddyctl ./cmd/maddyctl
```

Both compiled successfully (exit 0). The only build output is the benign upstream `mattn/go-sqlite3` cgo notice (`sqlite3-binding.c:...: warning: 'Select standin' may be used uninitialized`), which is a library artifact, not a maddy error.

**Run command (verbatim):**

```console
$ /tmp/maddy-run/bin/maddy -config /tmp/maddy-run/test-maddy.conf -debug
```

`-config` and `-debug` are real flags dispatched by `Run` at `maddy.go:L102`.

**Observation tooling (OS/container level — NOT project dependencies, NOT committed):** `swaks 20201014.0`, `OpenSSL 1.1.1n`, `sqlite3 3.34.1`, `python3 3.9.2` + `dkimpy 1.1.8` (`dkimverify`), and a purpose‑built Go verifier using maddy's *own* `go-msgauth`.

### 2.1 Documented deviations from the canonical config (and why)

The run is a faithful copy of the shipped `maddy.conf` with exactly three environment‑driven deviations, each explicitly noted:

1. **Relocated state.** A global `state /tmp/maddy-run/state` (and `runtime /tmp/maddy-run/runtime`) directive. Maddy performs `os.Chdir(config.StateDirectory)` at startup (`maddy.go:L220`; default `"/var/lib/maddy"` at `maddy.go:L59`), and the default `dsn all.db` is **relative**, so it resolves *inside* the state dir. Relocating state keeps `all.db`, `dkim_keys/`, `queue/` and `mtasts-cache/` **out of the checkout**. (`.gitignore` excludes compiled binaries, `*.pem`/`*.crt`/`*.key`, and `cmd/maddy/*queue`/`*mtasts-cache`, but **not** `all.db` — which is exactly why relocation is required for byte‑for‑byte cleanliness.)
2. **Self‑signed TLS.** The submission port is implicit‑TLS `:465` and requires a certificate. The canonical config points `tls` at `/etc/maddy/certs/$(hostname)/fullchain.pem` (`maddy.conf:L16-17`), which does not exist in the sandbox, so `tls` is pointed at a self‑signed `example.org` pair generated with `openssl req -x509 -newkey rsa:2048 -nodes -keyout .../example.org.key -out .../example.org.crt -subj "/CN=example.org" -days 2 -addext "subjectAltName=DNS:example.org"`.
3. **Test‑only plaintext port.** An additional `submission tcp://127.0.0.1:1587` was bound **solely** so a human‑readable packet capture is possible (port `:465` is ciphertext on the wire). This is explicitly a **test‑only deviation**; the primary transaction capture is the `maddy -debug` log plus the `swaks` transcript. (Runtime note: EHLO on `:1587` advertises `STARTTLS` but the SASL `AUTH` capability is only offered after TLS, so plaintext credentials are never accepted — see §7.6.)

All other pipeline directives are kept faithful to the canonical `maddy.conf` so that observations reflect **default** behavior.

---

## 3. Setup

### 3.1 `test-maddy.conf` — the relevant blocks (verbatim line anchors)

```
state /tmp/maddy-run/state
runtime /tmp/maddy-run/runtime
tls /tmp/maddy-run/certs/example.org.crt /tmp/maddy-run/certs/example.org.key

# sql module: one SQLite DB for BOTH auth (local_authdb) and storage (local_mailboxes).
# dsn all.db is RELATIVE -> resolves inside the relocated state dir. (canonical maddy.conf:L32-35)
sql local_mailboxes local_authdb {
    dsn all.db
}

submission tls://0.0.0.0:465 {
    auth &local_authdb
    source $(local_domains) {
        modify {
            sign_dkim $(primary_domain) default
        }
        destination postmaster $(local_domains) {
            deliver_to &local_mailboxes
        }
        default_destination {
            deliver_to &remote_queue
        }
    }
    default_source {
        reject 501 5.1.8 "Non-local sender domain"
    }
}

submission tcp://127.0.0.1:1587 { ...identical pipeline... }   # TEST-ONLY plaintext port
imap tls://0.0.0.0:993 { auth &local_authdb; storage &local_mailboxes }
```

This exhibits the four required properties simultaneously: (1) an **auth‑required** submission endpoint on `:465` (`auth &local_authdb`; authentication is *always* required on submission — set at `internal/endpoint/smtp/smtp.go:L590` and enforced at `L662`); (2) **DKIM signing** via `sign_dkim $(primary_domain) default` (canonical `maddy.conf:L99`); (3) **local delivery** to `&local_mailboxes` backed by the `imapsql`/`sql` store with `dsn all.db` (canonical `maddy.conf:L34`); (4) two accounts, created below. `hostname`, `primary_domain` and `local_domains` all default to `example.org`.

### 3.2 DKIM key auto‑generation (first start)

On first start the `sign_dkim` modifier auto‑generates the key pair. Observed startup log (`maddy -debug`):

```
sign_dkim: generating a new rsa2048 keypair...
sign_dkim: generated a new rsa2048 keypair, private key is in dkim_keys/example.org_default.key, TXT record with public key is in dkim_keys/example.org_default.dns,
```

Grounding: `internal/modify/dkim/keys.go` — `generateAndWrite` (L77), log line `"generating a new %s keypair..."` (L82), `rsa.GenerateKey(…, 2048)` (L95), `x509.MarshalPKCS8PrivateKey` (L105), key file written with perms `0600` and PEM type `"PRIVATE KEY"` (L121‑L131), `.dns` record via `writeDNSRecord` (L136).

Observed key material:

```console
$ head -1 /tmp/maddy-run/state/dkim_keys/example.org_default.key ; stat -c '%a %s bytes' ...key
-----BEGIN PRIVATE KEY-----
600 1704 bytes
```

The public TXT record actually written (this exact `p=` is what a verifier consumes):

```console
$ cat /tmp/maddy-run/state/dkim_keys/example.org_default.dns
v=DKIM1; k=rsa; p=MIIBCgKCAQEAxEl4ES79Rhoo63xj0Y8SEpvxBYkd9R0f4c5u2+18vkkyPs4ETREQRbd4TrLyHYrZlhLtPgFHM6QNgTuvRqFD6TgBnNds+orxR78mceUilek0micpZ8gPOl74GyH3FPNikCBGHdYWj5XQ+GBs/U+F2/a4khpVBhtCBIqIQ+o/ehw28V8FTCugKWYydoPCiep5VZsfPQ2F8NYhxOPFU5WlXd5voRMxbBeiA3QbfFy1DMvhAwJDqFhHusKG5cPZwOiS8uZu7twSbqupQqDLi719ssLaPYkcEiYC2n6NxEPx4r3dobVqWscBa6Ujo2TEs3qjXshNPcgZwIBvf4YGKi/vnwIDAQAB
```

> **Important observed detail (foundational to §6.3):** the `p=` value begins `MIIBCgKCAQEA…`, which is the base64 of a bare **PKCS#1 `RSAPublicKey`**. RFC 6376 requires a **SubjectPublicKeyInfo (PKIX)** key, whose base64 begins `MIIBIjANBgkqhkiG9w0BAQEF…`. This is `x509.MarshalPKCS1PublicKey` at `internal/modify/dkim/keys.go:L143` — analyzed in §6.3 and §8.4.

Observed DKIM defaults (from `internal/modify/dkim/dkim.go`): `hash sha256` (L147‑148), `newkey_algo rsa2048` (L149‑150), `key_path dkim_keys/{domain}_{selector}.key` (L137), header/body canonicalization `relaxed`/`relaxed` (L141‑145), default selector `default` (from the config directive).

### 3.3 Accounts and mailboxes

Accounts were created with the **real** subcommand for this version (`users create`; password from `-p` or stdin — `cmd/maddyctl/main.go:L47`/`L71`):

```console
$ maddyctl --config /tmp/maddy-run/test-maddy.conf users create alice@example.org -p alicepass   # exit 0
$ maddyctl --config /tmp/maddy-run/test-maddy.conf users create bob@example.org   -p bobpass      # exit 0
```

Verification that accounts and delivery mailboxes exist (`imap-mboxes`, `cmd/maddyctl/main.go:L203`/`L207`):

```console
$ maddyctl --config .../test-maddy.conf users list
alice@example.org
bob@example.org

$ maddyctl --config .../test-maddy.conf imap-mboxes list alice@example.org
INBOX
$ maddyctl --config .../test-maddy.conf imap-mboxes list bob@example.org
INBOX
```

(In this version `local_authdb` and `local_mailboxes` are the **same** `imapsql`/`sql` module, so `go-imap-sql` unifies authentication and storage and an INBOX exists immediately after `users create` — confirmed empirically above. The benign `Failed to initialize update pipe … connection refused` notice printed by `maddyctl` merely means it will not notify IMAP clients that have the mailbox open; direct DB reads succeed.)

---

## 4. How acceptance is decided (envelope‑domain source routing)

When authentication succeeds, the submission session (`internal/endpoint/smtp/smtp.go`) accepts `MAIL FROM` syntactically and — because `defer_sender_reject` defaults to **true** (`smtp.go:L567`) — defers the routing decision to the first `RCPT TO`. `Mail` (`smtp.go:L162`) with defer enabled just stores the sender and returns `250`; the first `Rcpt` (`smtp.go:L208`) invokes the deferred `startDelivery` (`smtp.go:L220`), which runs the message pipeline.

The pipeline decides accept/reject in `srcBlockForAddr` (`internal/msgpipeline/msgpipeline.go:L155`): it computes `cleanFrom = address.ForLookup(mailFrom)` (L156/L159), then selects a source block by trying, in order, `perSource[cleanFrom]` (exact address, L171) → `perSource[domain]` (domain, L190) → `defaultSource` (L193). `start` (`msgpipeline.go:L102`) returns the matched block's `rejectErr` if the block is a `reject` directive (L117‑L119).

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

`submissionPrepare` (`internal/endpoint/smtp/submission.go:L27`) is the only submission‑specific header processing. It:

- sets `msgMeta.DontTraceSender = true` (`L28`) — which suppresses the leading `from <host>` clause of the `Received` header (see §7.4);
- inserts a `Message-ID` if absent (~`L30-36`);
- **requires a `From:` header**, returning `554 5.6.0 "Message does not contains a From header field"` if absent (`L39-48`);
- requires a `Sender:` header if there are multiple `From:` addresses (`L98-103`);
- adds a missing `Date:` header (`L111`, `L125`).

**It performs no comparison of the `From:`/`MAIL FROM` sender against the authenticated identity.** (Verified by reading the entire function; the only identity‑related processing is the presence/uniqueness of `From:`.) This is the code‑level reason the default configuration cannot enforce per‑user sender binding.

---

## 6. How signing is decided (`shouldSign` gate order)

`RewriteBody` (`internal/modify/dkim/dkim.go:L340`) reads the authenticated user (`authUser = s.meta.Conn.AuthUser`, `L345`) and calls `shouldSign(...)` (`L348`, passing `s.meta.OriginalFrom`). If it returns `false`, the function returns immediately and the message is delivered **unsigned**; otherwise the signature is added via `h.Add("DKIM-Signature", signer.SignatureValue())` (`L406`) followed by the debug line `signed` (`L408`).

### 6.1 Gate order (evaluated top to bottom), `shouldSign` at `dkim.go:L249`

1. modifier `off` → skip;
2. empty `From` (`L266`);
3. malformed `From` (`L271`);
4. multiple `From` (`L275`);
5. **`From` domain ≠ key domain** (`L287-288`) → debug `"not signing, From domain is not key domain"` — **Scenario D fails here**;
6. **envelope `MAIL FROM` mismatch** (`L293-294`, active because `require_sender_match` includes `envelope`);
7. **`From` ≠ authenticated identity** (`L299-306`) → debug `"not signing, From address is not authenticated identity"` — **Scenario A fails here**;
8. otherwise → sign.

Because D fails at gate 5 and A fails at gate 7, they emit **different** debug lines — confirmed verbatim in §7.

### 6.2 Defaults and the oversigned header set

`require_sender_match` default is `[envelope auth]` (`dkim.go:L151-152`; options are `[envelope, auth_domain, auth_user, off]`). The default **oversigned** header list (`oversignDefault`, `dkim.go:L31-52`) is: `Subject, Sender, To, Cc, From, Date, MIME-Version, Content-Type, Content-Transfer-Encoding, Reply-To, In-Reply-To, Message-Id, References, Autocrypt, Openpgp`. `fieldsToSign` (`dkim.go:L202`) lists each header `(occurrences + 1)` times, which is why the observed `h=` tag doubles the headers that are present (e.g. `Subject:Subject`, `From:From`) and names once those that are absent (e.g. `Sender`, `Cc`, `MIME-Version`). The subtlety at gate 7: because the authenticated name contains `@`, the comparison uses the full address `alice@example.org` (not just the local part), as seen in Scenario A's debug (`auth_id:alice@example.org`, `from_addr:bob@example.org`).

### 6.3 Signatures are produced — but they do not verify (root cause)

Scenario C **is signed** (§7.3), yet the signature **fails verification at any RFC‑6376 verifier**. This was established with byte‑faithful wire captures and reproduced with maddy's **own** libraries (identical `go.mod` pins: `go-message v0.10.9-0.20191116124005-65fd0119e899`, `go-msgauth v0.3.2-0.20191028231513-55b75676976c`, `x/crypto v0.0.0-20191108234033-bd318be0434a`). There are **two independent defects**:

**Defect 1 — public key published as PKCS#1, not PKIX.** `writeDNSRecord` uses `x509.MarshalPKCS1PublicKey(pubkey)` at `internal/modify/dkim/keys.go:L143`, so the `.dns` `p=` is a bare PKCS#1 `RSAPublicKey`. RFC 6376 requires SubjectPublicKeyInfo. Fed the as‑published record, maddy's own verifier reports a **parse** failure:

```console
$ cat capture_C.wire.eml | ./dkimverify-go        # DNS serving the as-published PKCS#1 p=
domain=example.org identifier=alice@example.org valid=false err=dkim: key syntax error: x509: failed to parse public key (use ParsePKCS1PublicKey instead for this key format)
```

The key *material* is correct — the fingerprint of the public key derived from the private `.key` equals the fingerprint of the published `.dns` key (`dd1322dc…818b95`); only the *encoding* is wrong.

**Defect 2 — `DKIM-Signature` header hashed with one folder, emitted with another.** After correcting the record to PKIX (`p=MIIBIjANBgkqhkiG9w0BAQEF…`) and serving it, the signature **still** fails — but now with an RSA/hash mismatch, and the **body hash matches**:

```console
$ cat capture_C.wire.eml | ./dkimverify-go        # DNS serving the CORRECTED PKIX p=
domain=example.org identifier=alice@example.org valid=false err=dkim: signature did not verify: crypto/rsa: verification error

$ python3 verify.py capture_C.wire.eml             # dkimpy, PKIX key
bh: b'p7QrzafR7oC6tUDqTof/6nhDVWINwUf3YwS8WdcaqCk='   # <- computed body hash EQUALS the signature's bh=
b'Dkim-Signature' valid: False                        # <- header/RSA check fails
RESULT: False
```

**Isolation via a controlled round‑trip** (maddy's real private key, maddy's exact `SignOptions`, identical libraries):

```console
$ ./rt maddy       # NewSigner + textproto.WriteHeader(signer,*h) + h.Add(...) + emit via go-message  (== maddy's RewriteBody)
[maddy] domain=example.org id=alice@example.org valid=false err=dkim: signature did not verify: crypto/rsa: verification error
$ ./rt allinone    # dkim.Sign(...) streaming all-in-one
[allinone] domain=example.org id=alice@example.org valid=true  err=<nil>
$ ./rt fix         # maddy's exact signed headers+body, but emit the sig via signer.Signature() (go-msgauth folding)
[fix] valid=true err=<nil>
```

**Mechanism.** Maddy signs by calling `dkim.NewSigner`, writing the current headers to the signer with `textproto.WriteHeader(signer, *h)`, then adding the result with `h.Add("DKIM-Signature", signer.SignatureValue())` (`dkim.go:L406`). Internally the go‑msgauth signer computes the `DKIM-Signature` self‑hash over its header folded by **`foldHeaderField`** (go‑msgauth `header.go`), which hard‑wraps **every 75 bytes regardless of token boundaries**:

```go
func foldHeaderField(kv string) string {
	buf := bytes.NewBufferString(kv + crlf)
	line := make([]byte, 75) // 78 - len("\r\n\s")
	first := true
	var fold strings.Builder
	for len, err := buf.Read(line); err != io.EOF; len, err = buf.Read(line) {
		if first { first = false } else { fold.WriteString("\r\n ") }
		fold.Write(line[:len])
	}
	return fold.String()
}
```

Under **relaxed** header canonicalization the fold's `CRLF` is removed but the continuation **space is kept**, so this hard‑wrap injects a space *inside* tokens. Maddy, however, **emits** the header through go‑message's `textproto.WriteHeader`, which folds only at real whitespace — so no spurious space. The two byte strings therefore differ. The canonicalized `DKIM-Signature` diff makes it explicit (spaces marked by context):

```text
all-in-one (VALID, go-msgauth fold):  ...WdcaqC k=; ... h=Subject:Subject:...Fro m:From... Reply- To... i=alice@ example.org ...
maddy-style (FAILS, go-message fold):  ...WdcaqCk=;  ... h=Subject:Subject:...From:From...  Reply-To...  i=alice@example.org ...
```

The signer hashed the top string (with mid‑token spaces); a verifier reconstructs the bottom string from the emitted header → the header hash differs → `crypto/rsa: verification error`. The `fix` round‑trip proves it: re‑emitting the identical signed headers via `signer.Signature()` (the same folder the signer hashed) makes the message verify. The `allinone` path verifies because `dkim.Sign` uses the *same* folder for hashing and emission.

**Scope of the defect.** This is intrinsic to `RewriteBody` (`dkim.go:L406`) and therefore affects **every** delivery path (local `imapsql` storage and `smtp_downstream` relay alike), since the signature is always added via `h.Add` and then serialized by go‑message. Analysis of the mechanism is grounded in the code above and the reproductions; the conclusion that *all* maddy‑emitted signatures at this commit are affected is **(inferred)** from that shared code path (directly observed for the submission→delivery path used here).

---


## 7. Per‑scenario evidence (A, B, C, D)

Every scenario authenticates as **`alice@example.org`** over implicit TLS on `:465`. `swaks` sets the envelope `MAIL FROM` with `--from` and the message `From:` header independently with `--header "From: …"`, exactly the envelope‑vs‑header split the scenarios require. Each scenario was run **twice**; §7.5 shows the outcomes are deterministic.

### 7.1 Scenario (A) — *"Authenticate as one user but attempt `MAIL FROM` with a different user's address."*

**Command:**
```console
$ swaks --server 127.0.0.1:465 --tls-on-connect \
        --auth PLAIN --auth-user alice@example.org --auth-password alicepass \
        --from bob@example.org --to alice@example.org \
        --header "From: bob@example.org" --header "Subject: scenario A" --body "scenario A"
```

**Full unedited `swaks` transcript:**
```
=== Trying 127.0.0.1:465...
=== Connected to 127.0.0.1.
=== TLS started with cipher TLSv1.3:TLS_AES_128_GCM_SHA256:128
=== TLS no local certificate set
=== TLS peer DN="/CN=example.org"
<~  220 example.org ESMTP Service Ready
 ~> EHLO 692756bb4fcc
<~  250-Hello 692756bb4fcc
<~  250-PIPELINING
<~  250-8BITMIME
<~  250-ENHANCEDSTATUSCODES
<~  250-AUTH PLAIN
<~  250-SMTPUTF8
<~  250 SIZE 33554432
 ~> AUTH PLAIN AGFsaWNlQGV4YW1wbGUub3JnAGFsaWNlcGFzcw==
<~  235 2.0.0 Authentication succeeded
 ~> MAIL FROM:<bob@example.org>
<~  250 2.0.0 Roger, accepting mail from <bob@example.org>
 ~> RCPT TO:<alice@example.org>
<~  250 2.0.0 I'll make sure <alice@example.org> gets this
 ~> DATA
<~  354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
 ~> Date: Mon, 06 Jul 2026 22:26:42 +0000
 ~> To: alice@example.org
 ~> From: bob@example.org
 ~> Subject: scenario A
 ~> Message-Id: <20260706222642.022913@692756bb4fcc>
 ~> X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/
 ~> 
 ~> scenario A
 ~> 
 ~> 
 ~> .
<~  250 2.0.0 OK: queued
 ~> QUIT
<~  221 2.0.0 Goodnight and good luck
=== Connection closed with remote host.
```

**`maddy -debug` window:**
```
submission: incoming message	{"msg_id":"bd16dc0f","sender":"bob@example.org","src_host":"692756bb4fcc","src_ip":"127.0.0.1:57762","username":"alice@example.org"}
[debug] smtp/pipeline: sender bob@example.org matched by domain rule 'example.org'	{"msg_id":"bd16dc0f"}
[debug] smtp/pipeline: tgt.Start(bob@example.org) ok, target = sql:local_mailboxes	{"msg_id":"bd16dc0f"}
submission: RCPT ok	{"msg_id":"bd16dc0f","rcpt":"alice@example.org"}
sign_dkim: not signing, From address is not authenticated identity	{"auth_id":"alice@example.org","from_addr":"bob@example.org","msg_id":"bd16dc0f"}
[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"bd16dc0f"}
submission: accepted	{"msg_id":"bd16dc0f"}
```

**Raw stored headers (delivered to `alice`'s INBOX):**
```
Delivered-To: alice@example.org
Return-Path: <bob@example.org>
Received:  by example.org (envelope-sender <bob@example.org>) with ESMTPS id
 bd16dc0f; Mon, 06 Jul 2026 22:26:42 +0000
Date: Mon, 06 Jul 2026 22:26:42 +0000
To: alice@example.org
From: bob@example.org
Subject: scenario A
Message-Id: <20260706222642.022913@692756bb4fcc>
X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/

scenario A
```

**Outcome:** **Accepted (`250`) and delivered.** The envelope domain `example.org` matched the source block by **domain** rule (`srcBlockForAddr`, `msgpipeline.go:L190`) — the fact that the sender is a *different user* (`bob`) is never checked. The message is **NOT DKIM‑signed** because the `From:` (`bob@example.org`) is not the authenticated identity (`alice@example.org`) — gate 7 of `shouldSign` (`dkim.go:L299-306`). There is **no `Dkim-Signature`** and **no `Authentication-Results`** in the stored message. This scenario **disproves** the "auth means send‑as‑self" interpretation.

### 7.2 Scenario (B) — *"Send with a `MAIL FROM` domain that doesn't match any configured signing domain."*

**Command:**
```console
$ swaks --server 127.0.0.1:465 --tls-on-connect \
        --auth PLAIN --auth-user alice@example.org --auth-password alicepass \
        --from x@notlocal.example --to alice@example.org \
        --header "From: x@notlocal.example" --body "scenario B"
```

**Full unedited `swaks` transcript:**
```
=== Trying 127.0.0.1:465...
=== Connected to 127.0.0.1.
=== TLS started with cipher TLSv1.3:TLS_AES_128_GCM_SHA256:128
=== TLS no local certificate set
=== TLS peer DN="/CN=example.org"
<~  220 example.org ESMTP Service Ready
 ~> EHLO 692756bb4fcc
<~  250-Hello 692756bb4fcc
<~  250-PIPELINING
<~  250-8BITMIME
<~  250-ENHANCEDSTATUSCODES
<~  250-AUTH PLAIN
<~  250-SMTPUTF8
<~  250 SIZE 33554432
 ~> AUTH PLAIN AGFsaWNlQGV4YW1wbGUub3JnAGFsaWNlcGFzcw==
<~  235 2.0.0 Authentication succeeded
 ~> MAIL FROM:<x@notlocal.example>
<~  250 2.0.0 Roger, accepting mail from <x@notlocal.example>
 ~> RCPT TO:<alice@example.org>
<~* 501 5.1.8 Non-local sender domain (msg ID = 161fd486)
 ~> QUIT
<~  221 2.0.0 Goodnight and good luck
=== Connection closed with remote host.
```

**`maddy -debug` window:**
```
submission: incoming message	{"msg_id":"161fd486","sender":"x@notlocal.example","src_host":"692756bb4fcc","src_ip":"127.0.0.1:58018","username":"alice@example.org"}
[debug] smtp/pipeline: sender x@notlocal.example matched by default rule	{"msg_id":"161fd486"}
submission: RCPT error	{"effective_rcpt":"alice@example.org","rcpt":"alice@example.org","reason":"reject directive used","smtp_code":501,"smtp_enchcode":"5.1.8","smtp_msg":"Non-local sender domain"}
submission: aborted	{"msg_id":"161fd486"}
```

**Outcome:** **Rejected `501 5.1.8 "Non-local sender domain"`.** Critically, `MAIL FROM:<x@notlocal.example>` returns **`250`**, and the rejection surfaces at the first **`RCPT TO`**. This is the deferred‑reject behavior: `defer_sender_reject` defaults to **true** (`smtp.go:L567`), so the foreign sender matched `default_source { reject 501 5.1.8 … }` (canonical `maddy.conf:L117-119`) only when `startDelivery` ran at `RCPT` (`smtp.go:L220`); the debug shows `matched by default rule` (`msgpipeline.go:L193-194`) and `reason:"reject directive used"`. Nothing is delivered (`submission: aborted`). This is **divergence #2** (§8.4).

### 7.3 Scenario (C) — *"Send a legitimate message that should be properly signed."*

**Command:**
```console
$ swaks --server 127.0.0.1:465 --tls-on-connect \
        --auth PLAIN --auth-user alice@example.org --auth-password alicepass \
        --from alice@example.org --to bob@example.org \
        --header "From: alice@example.org" --header "Subject: scenario C" --body "scenario C"
```

**Full unedited `swaks` transcript:**
```
=== Trying 127.0.0.1:465...
=== Connected to 127.0.0.1.
=== TLS started with cipher TLSv1.3:TLS_AES_128_GCM_SHA256:128
=== TLS no local certificate set
=== TLS peer DN="/CN=example.org"
<~  220 example.org ESMTP Service Ready
 ~> EHLO 692756bb4fcc
<~  250-Hello 692756bb4fcc
<~  250-PIPELINING
<~  250-8BITMIME
<~  250-ENHANCEDSTATUSCODES
<~  250-AUTH PLAIN
<~  250-SMTPUTF8
<~  250 SIZE 33554432
 ~> AUTH PLAIN AGFsaWNlQGV4YW1wbGUub3JnAGFsaWNlcGFzcw==
<~  235 2.0.0 Authentication succeeded
 ~> MAIL FROM:<alice@example.org>
<~  250 2.0.0 Roger, accepting mail from <alice@example.org>
 ~> RCPT TO:<bob@example.org>
<~  250 2.0.0 I'll make sure <bob@example.org> gets this
 ~> DATA
<~  354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
 ~> Date: Mon, 06 Jul 2026 22:27:29 +0000
 ~> To: bob@example.org
 ~> From: alice@example.org
 ~> Subject: scenario C
 ~> Message-Id: <20260706222729.022938@692756bb4fcc>
 ~> X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/
 ~> 
 ~> scenario C
 ~> 
 ~> 
 ~> .
<~  250 2.0.0 OK: queued
 ~> QUIT
<~  221 2.0.0 Goodnight and good luck
=== Connection closed with remote host.
```

**`maddy -debug` window:**
```
submission: incoming message	{"msg_id":"1f953e28","sender":"alice@example.org","src_host":"692756bb4fcc","src_ip":"127.0.0.1:58736","username":"alice@example.org"}
[debug] smtp/pipeline: sender alice@example.org matched by domain rule 'example.org'	{"msg_id":"1f953e28"}
[debug] smtp/pipeline: tgt.Start(alice@example.org) ok, target = sql:local_mailboxes	{"msg_id":"1f953e28"}
submission: RCPT ok	{"msg_id":"1f953e28","rcpt":"bob@example.org"}
[debug] sign_dkim: signed	{"identifier":"alice@example.org"}
[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"1f953e28"}
submission: accepted	{"msg_id":"1f953e28"}
```

**Raw stored headers (delivered to `bob`'s INBOX) — signature body truncated with an explicit marker; all tags intact:**
```
Delivered-To: bob@example.org
Return-Path: <alice@example.org>
Dkim-Signature: a=rsa-sha256;
 bh=p7QrzafR7oC6tUDqTof/6nhDVWINwUf3YwS8WdcaqCk=; c=relaxed/relaxed;
 d=example.org;
 h=Subject:Subject:Sender:To:To:Cc:From:From:Date:Date:MIME-Version:Content-Type:Content-Transfer-Encoding:Reply-To:In-Reply-To:Message-Id:Message-Id:References:Autocrypt:Openpgp; i=alice@example.org; s=default; t=1783376849; v=1; x=1783808849; b=pb1rabqPaToEBSkdMvRKth10zGlHIjTN4Z3euptHzKt5SNxvl2wmxN7ZY3SIYusLUxw/CAf2qAS…[truncated]…OfTDBw==;
Received:  by example.org (envelope-sender <alice@example.org>) with ESMTPS
 id 1f953e28; Mon, 06 Jul 2026 22:27:29 +0000
Date: Mon, 06 Jul 2026 22:27:29 +0000
To: bob@example.org
From: alice@example.org
Subject: scenario C
Message-Id: <20260706222729.022938@692756bb4fcc>
X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/

scenario C
```

**Outcome:** **Accepted (`250`), delivered, and DKIM‑signed** (`sign_dkim: signed`, `dkim.go:L406-408`). The `DKIM-Signature` `h=` tag **covers `From`** (indeed oversigned: `From:From`), along with `Subject`, `To`, `Date`, `Message-Id`, etc. **However, the signature does NOT verify** — see the verifier output and root‑cause analysis in §6.3. This is the observed contradiction of the AAP's *predicted* "verifier PASS": run‑first observation shows the signature is present and structurally complete, body hash correct, but the header/RSA check fails at any RFC‑6376 verifier (both maddy's own `go-msgauth` and `dkimpy`). No `Authentication-Results` header is present.

**Recipient‑side verification (offline, fed the generated key):**
```console
$ cat <bytes maddy emitted for scenario C> | ./dkimverify-go     # PKIX-corrected key served via local DNS
domain=example.org identifier=alice@example.org valid=false err=dkim: signature did not verify: crypto/rsa: verification error
```
(Full two‑defect breakdown, including the as‑published PKCS#1 parse failure and the controlled round‑trip proving the folding root cause, is in §6.3.)

### 7.4 Scenario (D) — *"Send a message that should trigger DKIM signing but use a `From` header that doesn't match the authenticated user or signing domain — observe whether a signature is still applied, whether it covers `From`, and what a verifying recipient sees."*

**Command:**
```console
$ swaks --server 127.0.0.1:465 --tls-on-connect \
        --auth PLAIN --auth-user alice@example.org --auth-password alicepass \
        --from alice@example.org --to bob@example.org \
        --header "From: stranger@other.example" --header "Subject: scenario D" --body "scenario D"
```

**Full unedited `swaks` transcript:**
```
=== Trying 127.0.0.1:465...
=== Connected to 127.0.0.1.
=== TLS started with cipher TLSv1.3:TLS_AES_128_GCM_SHA256:128
=== TLS no local certificate set
=== TLS peer DN="/CN=example.org"
<~  220 example.org ESMTP Service Ready
 ~> EHLO 692756bb4fcc
<~  250-Hello 692756bb4fcc
<~  250-PIPELINING
<~  250-8BITMIME
<~  250-ENHANCEDSTATUSCODES
<~  250-AUTH PLAIN
<~  250-SMTPUTF8
<~  250 SIZE 33554432
 ~> AUTH PLAIN AGFsaWNlQGV4YW1wbGUub3JnAGFsaWNlcGFzcw==
<~  235 2.0.0 Authentication succeeded
 ~> MAIL FROM:<alice@example.org>
<~  250 2.0.0 Roger, accepting mail from <alice@example.org>
 ~> RCPT TO:<bob@example.org>
<~  250 2.0.0 I'll make sure <bob@example.org> gets this
 ~> DATA
<~  354 2.0.0 Go ahead. End your data with <CR><LF>.<CR><LF>
 ~> Date: Mon, 06 Jul 2026 22:27:46 +0000
 ~> To: bob@example.org
 ~> From: stranger@other.example
 ~> Subject: scenario D
 ~> Message-Id: <20260706222746.022950@692756bb4fcc>
 ~> X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/
 ~> 
 ~> scenario D
 ~> 
 ~> 
 ~> .
<~  250 2.0.0 OK: queued
 ~> QUIT
<~  221 2.0.0 Goodnight and good luck
=== Connection closed with remote host.
```

**`maddy -debug` window:**
```
submission: incoming message	{"msg_id":"d78d9b2d","sender":"alice@example.org","src_host":"692756bb4fcc","src_ip":"127.0.0.1:47736","username":"alice@example.org"}
[debug] smtp/pipeline: sender alice@example.org matched by domain rule 'example.org'	{"msg_id":"d78d9b2d"}
[debug] smtp/pipeline: tgt.Start(alice@example.org) ok, target = sql:local_mailboxes	{"msg_id":"d78d9b2d"}
submission: RCPT ok	{"msg_id":"d78d9b2d","rcpt":"bob@example.org"}
sign_dkim: not signing, From domain is not key domain	{"from_domain":"other.example","key_domain":"example.org","msg_id":"d78d9b2d"}
[debug] smtp/pipeline: delivery.Body ok, Delivery object = *msgpipeline.delivery	{"msg_id":"d78d9b2d"}
submission: accepted	{"msg_id":"d78d9b2d"}
```

**Raw stored headers (delivered to `bob`'s INBOX):**
```
Delivered-To: bob@example.org
Return-Path: <alice@example.org>
Received:  by example.org (envelope-sender <alice@example.org>) with ESMTPS
 id d78d9b2d; Mon, 06 Jul 2026 22:27:46 +0000
Date: Mon, 06 Jul 2026 22:27:46 +0000
To: bob@example.org
From: stranger@other.example
Subject: scenario D
Message-Id: <20260706222746.022950@692756bb4fcc>
X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/

scenario D
```

**Outcome (answering each part of the scenario):**
- *Is a signature still applied?* **No.** The `From:` domain `other.example` ≠ the key domain `example.org`, so `shouldSign` returns false at the **first** domain gate (`dkim.go:L287-288`) — debug `not signing, From domain is not key domain {from_domain:other.example, key_domain:example.org}`. Note this is an **earlier** gate than Scenario A (which fails at the auth‑identity gate `L306`).
- *Does it cover `From`?* Not applicable — there is **no** `Dkim-Signature` header in the stored message.
- *What does a verifying recipient see?* **No signature at all.** maddy's own verifier:
  ```console
  $ cat <bytes maddy emitted for scenario D> | ./dkimverify-go
  NO SIGNATURES FOUND
  ```
- *Was it delivered?* **Yes**, accepted and stored — and the true envelope sender is still exposed in the `Received` header: `(envelope-sender <alice@example.org>)`. So a foreign `From:` sails through acceptance untouched (the pipeline only checked the *envelope* domain), and the only "protection" — a DKIM signature — is silently withheld.

### 7.5 Determinism (≥2 runs)

Each scenario was executed twice. Normalizing only volatile fields (`msg_id`, ephemeral source port, timestamps):

- **`maddy -debug` decision lines: byte‑identical** across runs for **all four** scenarios — i.e. routing (`matched by domain rule 'example.org'` / `matched by default rule`), the sign decision (`signed` / `not signing, …`), `RCPT ok`/`RCPT error`, and `accepted`/`aborted` are fully deterministic.
- **`swaks` transcripts: identical SMTP response codes** across runs (A/C/D: `250`/`250`/`250 queued`; B: `250` then `501 5.1.8` at RCPT). The only difference is the **client‑generated `Message-Id`** (a `swaks` per‑send wall‑clock value, e.g. `…222642.022913` vs `…222643.022919`) — a client artifact, not a server behavior.
- DKIM verdicts reproduce: C fails (RSA header check, `bh` matching); A and D are unsigned.

No run‑to‑run variance in observed server behavior.

### 7.6 The `Received` header and the envelope sender

Every delivered message's `Received` header (grounded in `GenerateReceived`, `internal/target/received.go:L19`) has the form:

```
Received:  by example.org (envelope-sender <MAIL FROM>) with ESMTPS id <id>; <date>
```

Observations tied to the code:
- The **`(envelope-sender <…>)`** clause (`received.go:L69-71`) always embeds the true envelope `MAIL FROM` — so a delivered message reveals the real envelope sender even when `From:` differs (vivid in Scenario A: `envelope-sender <bob@example.org>`, and Scenario D: `envelope-sender <alice@example.org>` with `From: stranger@other.example`).
- The leading **`from <host> (<rdns> [<ip>])`** clause (`received.go:L30`) is **absent** because `submissionPrepare` sets `DontTraceSender = true` (`submission.go:L28`).
- The protocol token is **`ESMTPS`** (`smtp.go:L692`), confirming the plan's Correction #2 (there is no `ESMTPSA`).

---

## 8. Analysis — the four analytical questions

This section leads with the direct answer to each question, then layers the observed evidence and `file:line` grounding. Every factual claim is backed by a runtime observation from §7 or a code reference; statements that were reasoned from reading rather than observed are explicitly marked **(inferred)**.

### 8.1 How does Maddy decide to accept or reject a sender address from an authenticated session?

**Direct answer:** Acceptance is decided **solely by the domain of the envelope `MAIL FROM`**, via the message pipeline's *source routing*. The message `From:` header is **never consulted** for the accept/reject decision, and the **authenticated identity is never compared** against the sender at accept time.

**Mechanism (grounded).** The submission endpoint hands the cleaned envelope sender to the pipeline's `start` (`internal/msgpipeline/msgpipeline.go:L102`), which calls `srcBlockForAddr` (`msgpipeline.go:L155`). That function normalizes the address (`address.ForLookup`, `L159`) and selects a source block by trying, in order:

1. an **exact address** match `perSource[cleanFrom]` (`msgpipeline.go:L171`) — debug `matched by address rule` (`L199`);
2. a **domain** match `perSource[domain]` after `address.Split` (`msgpipeline.go:L174`, `L190`) — debug `matched by domain rule '%s'` (`L196`);
3. the **`defaultSource`** fallback (`msgpipeline.go:L193`) — debug `matched by default rule` (`L194`).

The matched block's rejection (if any) is returned by `start` (`msgpipeline.go:L117-119`). In the canonical config there is **one** `source $(local_domains) { … }` block (`maddy.conf:L97`) plus `default_source { reject 501 5.1.8 "Non-local sender domain" }` (`maddy.conf:L117-119`). So the decision reduces to: *is the `MAIL FROM` domain in `local_domains` (= `example.org`)?* If yes → accept; if no → `501 5.1.8`.

**Observed confirmation (from §7):**
- Scenario A (`MAIL FROM bob@example.org`) and Scenario C (`MAIL FROM alice@example.org`) both matched the domain rule (`sender … matched by domain rule 'example.org'`) and were **accepted** — even though A's sender is a *different user* than the authenticated `alice`.
- Scenario B (`MAIL FROM x@notlocal.example`) matched the default rule and was **rejected** `501 5.1.8 "Non-local sender domain"`.
- Scenario D (`MAIL FROM alice@example.org`, `From: stranger@other.example`) was **accepted** — proving the `From:` header plays no part in acceptance; only the envelope domain mattered.

The decision is therefore **domain‑scoped**, not user‑scoped and not header‑scoped.

### 8.2 Does the DEFAULT config enforce sender alignment with the authenticated identity, or is explicit policy required?

**Direct answer:** The default configuration enforces alignment **only at the domain level**. It does **not** bind the envelope `MAIL FROM` or the message `From:` to the authenticated **username**. Per‑user sender enforcement is **not available in this version at all** — there is no check module that could express it — so it cannot be enabled even with explicit config short of writing a `check.command` shim.

**Evidence 1 — no `authorize_sender` check exists at this commit.** A directory listing of `internal/check/` at HEAD `26452dd` contains only:

```console
$ ls internal/check/
action.go  command  dkim  dns  dnsbl  requiretls  skeleton.go  spf  stateless_check.go
```

The subdirectories `command`, `dkim`, `dns`, `dnsbl`, `requiretls`, `spf` are the only check modules; `action.go`, `skeleton.go`, `stateless_check.go` are infrastructure, not checks. **There is no `authorize_sender`.** (Modern Maddy ships `check.authorize_sender`, which maps the authenticated user to permitted sender addresses via a table — that module is a later addition; see §9.4.)

**Evidence 2 — the submission preparation step performs no sender‑vs‑auth comparison.** `submissionPrepare` (`internal/endpoint/smtp/submission.go:L27`) is the only submission‑specific message hook. Reading it end‑to‑end, it does exactly four things: sets `DontTraceSender = true` (`L28`); generates a `Message-Id` if absent (`~L30-36`); requires a `From:` header, returning `554 5.6.0 "Message does not contains a From header field"` if missing (`L39-48`) and requiring a `Sender:` header if there are multiple `From:` addresses (`L98-103`); and adds a missing `Date:` (`L125`). **It never reads `AuthUser` and never compares any sender field to the authenticated identity.**

**Evidence 3 — the authenticated identity is consumed in only four non‑test places, none of which is an accept‑time sender check.** A grep for `AuthUser` across `internal/` (excluding tests) resolves to: session state/logging in `smtp.go` (`AuthUser: username`, `L680`); relay credentials in `smtp_downstream/sasl.go`; the `{auth_user}` placeholder in `check/command/command.go`; and the **DKIM signing‑identity** comparison in `modify/dkim/dkim.go:L345`. The last is a *signing* decision (§8.4), not an acceptance control.

**Conclusion.** With the default config, an authenticated user can send with **any** envelope `MAIL FROM` and **any** `From:` header whose *domain* is local; the *username* is irrelevant to acceptance. Per‑user binding would require explicit configuration — and in this version, the only way to express it would be an out‑of‑the‑box‑absent module or a hand‑rolled `check.command` matching `{auth_user}` against `{sender}`. The default ships neither.

### 8.3 Ruling out a plausible‑but‑incorrect interpretation

**The tempting (wrong) interpretation:** *"Because submission requires authentication and the DKIM modifier is configured with `require_sender_match envelope auth`, an authenticated user can only send as themselves."*

**This is false, and Scenarios A and D disprove it directly:**
- **Scenario A** — authenticated as `alice`, sent with `MAIL FROM: bob@example.org` and `From: bob@example.org`. Result: **accepted and delivered** to the mailbox (stored as `alice` INBOX UID 1). The server never objected that `bob ≠ alice`. The *only* consequence of the mismatch was that the message went out **unsigned** — delivery itself was unaffected.
- **Scenario D** — authenticated as `alice`, `From: stranger@other.example`. Result: **accepted and delivered** (stored as `bob` INBOX UID 3) despite the `From:` matching neither the authenticated user nor any local domain.

Both scenarios show acceptance succeeding while the sender does not match the authenticated user — so `require_sender_match` is **not** an acceptance gate, and no other mechanism enforces sender==auth. The interpretation is ruled out by observation.

**External corroboration (context, not primary evidence).** foxcpp/maddy **Issue #268** ("A check to only allow MAIL FROM same as the authenticated user's username"), opened **2020‑08‑28**, documents precisely this default behavior — that with the default configuration an authenticated user can send from any same‑domain address (e.g. `user@example.org` sending as `admin@example.org`) — and requests per‑user enforcement as a **new feature** (milestone 0.5). The pinned commit `26452dd` is dated **2019‑12‑13**, roughly **eight months before** that issue was filed, so the per‑user check demonstrably did not exist here. The primary evidence remains the observed A/D runs and the `internal/check/` listing; the issue is cited only to corroborate that this was recognized upstream as a *missing* capability at the time.

### 8.4 Behaviors that differ from a reasonable reading of the configuration

Three observed divergences between "what the config looks like it does" and "what the runtime actually does":

**Divergence #1 — `require_sender_match` gates SIGNING, not ACCEPTANCE.** The directive name and its default value `{envelope, auth}` (`internal/modify/dkim/dkim.go:L151-152`) read like an authorization control that would *reject* a mismatched sender. In reality it only decides **whether to apply a DKIM signature**; a message that fails the match is still **accepted and delivered — just unsigned** (Scenario D, and the auth‑mismatch branch in Scenario A). The code makes this explicit: the mismatch branches in `shouldSign` (`dkim.go:L288`, `L294`, `L306`) merely `return false` (skip signing); they never produce an SMTP error. The manual corroborates the intent — `docs/man/maddy-filters.5.scd` (`~L518-522`, `~L544`) describes `require_sender_match` as requiring identifiers to match "otherwise — don't sign the message" (paraphrased). **(This is a signing policy masquerading, by name, as an acceptance policy.)**

**Divergence #2 — the `501 5.1.8` sender rejection surfaces at `RCPT TO`, not at `MAIL FROM`.** A naive reading of `default_source { reject 501 5.1.8 … }` would expect the rejection the moment a foreign `MAIL FROM` is issued. Instead (Scenario B) the server returns `250` to `MAIL FROM` and defers the `501` to the **first `RCPT TO`**. This is because `defer_sender_reject` defaults to **true** (`internal/endpoint/smtp/smtp.go:L567`): `Mail` (`L162`) with defer=true simply records the sender and returns `250`; the deferred `startDelivery` runs from the first `Rcpt` (`L208`, `L220`), and on error is wrapped and returned there (`L225-226`, debug `MAIL FROM error (deferred)` at `L223`). The observed transcript in §7.2 shows exactly this `250`→`501` timing shift.

**Divergence #3 (the most consequential) — DKIM signatures produced by this build do NOT verify at any RFC‑6376 verifier.** A reasonable operator reading `sign_dkim` in the config would assume that a "signed" message (Scenario C) verifies as authentic at the recipient. It does not. As proven in §6.3 with maddy's own key and libraries, Scenario C's signature fails for **two independent reasons**:

1. **The published public key is PKCS#1, not PKIX/SubjectPublicKeyInfo.** `writeDNSRecord` emits `x509.MarshalPKCS1PublicKey(pubkey)` (`internal/modify/dkim/keys.go:L143`), producing a `p=` value with the PKCS#1 prefix `MIIBCgKCAQEA…`. RFC 6376 requires the SubjectPublicKeyInfo (PKIX) encoding (`MIIBIjANBgkqhkiG9w0BAQEF…`). go‑msgauth rejects the record outright:
   ```
   err=dkim: key syntax error: x509: failed to parse public key (use ParsePKCS1PublicKey instead for this key format)
   ```
2. **Even after hand‑correcting the record to PKIX, the RSA header check still fails** (`crypto/rsa: verification error`) while the body hash (`bh=`) matches exactly. Root cause: go‑msgauth's `foldHeaderField` hard‑wraps the emitted `DKIM-Signature` every 75 bytes **regardless of token boundaries**, but maddy's integration (`RewriteBody`, `dkim.go:L406`, `h.Add`) hashes the self‑field via `NewSigner`/`textproto.WriteHeader` and then re‑emits it through a *different* folder — so the bytes the signer hashed differ from the bytes a verifier reconstructs after relaxed‑canonicalization. The controlled round‑trip in §6.3 (`./rt maddy` → invalid; `./rt allinone` and `./rt fix` → valid, same key/options/libs) isolates this precisely.

This divergence is a genuine security‑relevant finding: the server reports success and stamps a `Dkim-Signature`, but a real recipient would see either a malformed key record or a hard verification failure. **(inferred)** that all DKIM signatures emitted by this commit are affected, since the two defects are in the shared key‑writing and `RewriteBody` code paths, not in anything scenario‑specific; **directly observed** for the submission→local‑delivery path here.

### 8.5 `Authentication-Results` provenance

**Direct answer:** No `Authentication-Results` header appears on any message submitted in Scenarios A/C/D — and that is expected. That header is written by the **inbound** verification checks (`verify_dkim`, `apply_spf`, `dmarc yes`) that are attached to the port‑25 receive path in the canonical config (`maddy.conf:L62`, `L65`, `L70`), **not** by the submission/signing path. Messages injected through the authenticated `submission` endpoint never traverse those inbound checks, so no `Authentication-Results` is added.

**Observed confirmation:** the raw stored headers for A, C, and D (§7.1, §7.3, §7.4) contain `Delivered-To`, `Return-Path`, `Received`, and (for C) `Dkim-Signature` — but **no** `Authentication-Results` header in any of them. This matches the code path: the submission pipeline's only header mutations are `submissionPrepare` (§5) plus the DKIM `RewriteBody`, neither of which emits `Authentication-Results`. **(The claim that the *inbound* path would add it is inferred from the config wiring and was not exercised here, since the investigation drove the submission path exclusively.)**

---

## 9. Corrections & caveats

### 9.1 Correction — account creation is `users create`, NOT `creds create`

The upstream plan referenced `maddyctl creds create`. At commit `26452dd` that command **does not exist**. The CLI defines a `users` command (`cmd/maddyctl/main.go:L47`) with a `create` subcommand (`L71`) that reads the password from stdin (flags `--password/-p`, `--null/-n`, `--hash` default `bcrypt`, `--bcrypt-cost`, `--cfg-block` default `local_authdb`). There is **no `creds` command** in this version. Observed:

```console
$ /tmp/maddy-run/bin/maddyctl --config /tmp/maddy-run/test-maddy.conf creds create alice@example.org
Incorrect Usage.
...
$ /tmp/maddy-run/bin/maddyctl --config /tmp/maddy-run/test-maddy.conf users create alice@example.org
Enter password for new user:
<created>
```

`creds create` is *modern‑Maddy* phrasing (modern Maddy separates `creds create` + `imap-acct create`); this version unifies auth and storage through the shared `imapsql` module and uses `users create`. The document uses the verified command throughout (§3.3).

### 9.2 Correction — the `Received` protocol token is `ESMTPS`, NOT `ESMTPSA`

The plan's illustrative `Received` line used `with ESMTPSA`. At this commit the protocol token is computed in `internal/endpoint/smtp/smtp.go` (`~L686-695`) as one of `"LMTP"` (`L687`), `"ESMTPS"` (`L692`, when the TLS handshake completed), or `"ESMTP"` (`L694`). **There is no `ESMTPSA` token in this version.** On the implicit‑TLS `:465` submission port the observed header is `with ESMTPS` (see the raw stored headers in §7.1/§7.3/§7.4 and the `Received` analysis in §7.6).

### 9.3 Statements labeled "(inferred)"

For full transparency, the following statements in this document are **inferred from reading code/config** rather than directly observed by running, and are labeled as such at their point of use:

- That the `Authentication-Results` header would be **added on the inbound port‑25 path** by `verify_dkim`/`apply_spf`/`dmarc` (§8.5). Only its **absence on the submission path** was directly observed; the inbound path was not exercised.
- That **all** DKIM signatures emitted by this commit (not merely the submission→local‑delivery path) are affected by the two verification defects (§8.4, Divergence #3). The defect was directly observed for the exercised path and reasoned to be general because it lives in shared key‑writing (`keys.go:L143`) and `RewriteBody` (`dkim.go:L406`) code.

Everything else in this document is backed by observed runtime output (SMTP transcripts, `maddy -debug` lines, raw stored headers, verifier verdicts) together with a `file:line` reference to the code that produces it.

### 9.4 Version‑fidelity caveat — modern `authorize_sender` is context only

Modern Maddy ships a `check.authorize_sender` module that binds the authenticated user to a set of permitted sender addresses (via a `user_to_email` table/chain), and modern default configs place it inside the submission `source` block. **That module does not exist at commit `26452dd`** (§8.2, Evidence 1) and was **not** back‑ported, enabled, or simulated for this investigation. It is reported here purely as historical context to frame the "default‑vs‑explicit" question; every observed behavior in this document reflects the pinned version **as‑is**. All conclusions are scoped to commit `26452dd8dd787dc455278b0fdd296f4a5432c768`.

### 9.5 Documented environment deviations from the canonical config

These are the only deviations from the shipped `maddy.conf`, each an environment necessity (detailed in §2.1) and none affecting the observed sender/DKIM behavior:

1. **Relocated state** — a global `state /tmp/maddy-run/state` directive so `os.Chdir(config.StateDirectory)` (`maddy.go:L220`) moves `all.db`, `dkim_keys/`, `queue/`, and `mtasts-cache/` outside the checkout (`.gitignore` does **not** list `all.db`, so relocation is required for byte‑for‑byte cleanliness).
2. **Self‑signed TLS** — the canonical `tls file /etc/maddy/certs/$(hostname)/…` (`maddy.conf:L16-17`) points at paths absent in the sandbox; a self‑signed cert/key for `example.org` was substituted so the implicit‑TLS `:465` port can start.
3. **Optional plaintext test port / raw sink** — used only to obtain byte‑faithful captures for the DKIM root‑cause analysis (§6.3), explicitly labeled test‑only; the primary evidence is `maddy -debug` + the `swaks` transcript over the real `:465` port.

---

## 10. Cleanup & read‑only proof

The entire experiment was conducted with the repository checkout mounted **read‑only** and every working artifact confined to `/tmp/maddy-run/` (outside the checkout). No existing repository file was modified; the only repository change is the single new file `blitzy/documentation/maddy_26452dd8dd78.md`.

### 10.1 Read‑only proof (during investigation)

At all times during the investigation, `git status --porcelain` inside the checkout `/tmp/blitzy/maddy/…` reported only the new documentation path as untracked — never any modified source, config, `go.mod`, or `go.sum`:

```console
$ git -C /tmp/blitzy/maddy/… rev-parse HEAD
26452dd8dd787dc455278b0fdd296f4a5432c768

$ git -C /tmp/blitzy/maddy/… status --porcelain
?? blitzy/
```

`?? blitzy/` is git's porcelain collapse of an entirely‑untracked directory whose sole contents are `blitzy/documentation/maddy_26452dd8dd78.md`. No tracked file appears (no ` M`, ` A`, ` D`, or `R` entries), confirming zero edits to existing repository files. The runtime state (`/tmp/maddy-run/`) is outside the checkout and therefore never appears in git status.

### 10.2 Cleanup commands

After all evidence was captured, the running processes were stopped and the entire working directory removed so the environment is left clean:

```console
# Stop the running maddy server and helper listeners (raw sink + offline DNS resolver)
$ pkill -f 'maddy -config' ; pkill -f rawsink.py ; pkill -f dnssrv

# Remove the entire isolated runtime state, all scripts, transcripts, keys, certs, and the DB
$ rm -rf /tmp/maddy-run
```

Removing `/tmp/maddy-run` deletes: the test config(s); the self‑signed `certs/`; the SQLite store `state/all.db`; the auto‑generated `state/dkim_keys/example.org_default.{key,dns,PKIX.dns}`; `state/{queue,mtasts-cache}/`; the `swaks` scenario scripts and `*.log`/`*.eml`/`*.wire.eml` transcripts; the temporary Go verifier (`dkimv/`) and round‑trip harness (`rt/`); and the temporary Python DKIM/DNS helper scripts. Nothing from the experiment persists.

### 10.3 Final read‑only proof (post‑cleanup)

After stopping all processes and removing `/tmp/maddy-run`, the checkout shows only the new answer document as the sole change, `HEAD` is unchanged, and there are **zero** modifications to any tracked file:

```console
$ git rev-parse HEAD
26452dd8dd787dc455278b0fdd296f4a5432c768

$ git status --porcelain
?? blitzy/

$ git status --porcelain -uall          # expand the untracked directory to the exact path
?? blitzy/documentation/maddy_26452dd8dd78.md

$ git diff --stat HEAD                    # edits to tracked files — none
$                                         # (empty output: no source/config/go.mod/go.sum changes)
```

`git status --porcelain` collapses the entirely‑untracked `blitzy/` directory to a single `?? blitzy/` line; `-uall` expands it to show the one and only file it contains. The empty `git diff --stat HEAD` confirms **no existing repository file was modified** — the task is byte‑for‑byte read‑only apart from adding this document. (Note: `/tmp/maddy-run` is a harness‑managed bind‑mount point; its *contents* — DB, keys, certs, scripts, logs, binaries — were all deleted, leaving the mount point empty and outside the checkout regardless.)

> The compiled `maddy`/`maddyctl` binaries were built and kept under `/tmp/maddy-run/bin` (outside the tree); even had they been built in‑tree, `.gitignore` already excludes `cmd/maddy/maddy` and `cmd/maddyctl/maddyctl`. No dependency files (`go.mod`/`go.sum`) were touched, and the `authorize_sender` check was **not** added, enabled, or back‑ported.

---

## 11. Reference index (`file:line` → function/struct)

All references verified at HEAD `26452dd8dd787dc455278b0fdd296f4a5432c768`. Each row names the specific function/struct/directive the citation anchors.

### 11.1 `maddy.conf` (canonical config)

| Anchor | What it is |
|---|---|
| `maddy.conf:L16-17` | `tls file /etc/maddy/certs/$(hostname)/fullchain.pem privkey.pem` (overridden to self‑signed in the test config) |
| `maddy.conf:L32-35` | `storage.imapsql local_mailboxes { … dsn all.db }` (`dsn all.db` at L34; relative → resolves inside state dir) |
| `maddy.conf:L53,L62,L65,L70` | inbound `smtp tcp://0.0.0.0:25` with `verify_dkim` (L62), `apply_spf` (L65), `dmarc yes` (L70) — source of `Authentication-Results` on the inbound path |
| `maddy.conf:L74-76` | inbound `source $(local_domains) { reject … "Use Submission for outgoing SMTP" }` |
| `maddy.conf:L93` | **`submission tls://0.0.0.0:465`** |
| `maddy.conf:L95` | `auth &local_authdb` |
| `maddy.conf:L97` | `source $(local_domains) { … }` (the accept branch) |
| `maddy.conf:L99` | `modify { sign_dkim $(primary_domain) default }` |
| `maddy.conf:L104-107` | `destination … deliver_to &local_mailboxes` |
| `maddy.conf:L117-119` | **`default_source { reject 501 5.1.8 "Non-local sender domain" }`** |
| `maddy.conf:L149` | `imap tls://0.0.0.0:993` |

### 11.2 `internal/endpoint/smtp/smtp.go` (submission session)

| Anchor | Function/struct |
|---|---|
| `L83` | `startDelivery` — begins delivery for a session |
| `L162` | `Session.Mail` — `MAIL FROM`; with defer=true stores sender, returns `250` (`if !s.endp.deferServerReject` at `L163`) |
| `L208` | `Session.Rcpt` — first `RCPT` runs deferred `startDelivery` (`L220`); deferred error logged `MAIL FROM error (deferred)` (`L223`), wrapped/returned (`L225-226`) |
| `L469` / `L472` | struct fields `authAlwaysRequired` / `deferServerReject` |
| `L567` | **`defer_sender_reject` default `true`** (`cfg.Bool("defer_sender_reject", false, true, …)`) |
| `L588-590` | submission endpoint sets `authAlwaysRequired = true` (`L590`) |
| `L662` | auth requirement enforced |
| `L680` | `AuthUser: username` written into connection/session state |
| `L686-695` | Proto computation → `"LMTP"` (L687) / `"ESMTPS"` (L692) / `"ESMTP"` (L694) — **no `ESMTPSA`** |
| `L290-292` | `submissionPrepare` call site |

### 11.3 `internal/endpoint/smtp/submission.go` (`submissionPrepare`)

| Anchor | What it does |
|---|---|
| `L27` | `submissionPrepare` function |
| `L28` | **`msgMeta.DontTraceSender = true`** (suppresses the `from …` clause in `Received`) |
| `L30-36` | generate `Message-Id` if absent |
| `L39-48` | require `From:` → `554 5.6.0 "Message does not contains a From header field"` if missing |
| `L98-103` | require `Sender:` if multiple `From:` addresses |
| `L111,L125` | parse `Date:`; add missing `Date:` (log `adding missing Date header`) |
| — | **no sender‑vs‑auth comparison anywhere** |

### 11.4 `internal/msgpipeline/msgpipeline.go` (source routing)

| Anchor | Function/struct |
|---|---|
| `L102` | `start` — pipeline entry for a transaction |
| `L117-119` | return matched block's `rejectErr` (debug `sender %s rejected with error` at `L118`) |
| `L155` | **`srcBlockForAddr`** — selects the source block for the cleaned sender |
| `L159` | `address.ForLookup` normalization |
| `L171` | `perSource[cleanFrom]` (exact‑address match) |
| `L174` | `address.Split` (extract domain) |
| `L190` | `perSource[domain]` (domain match; debug `matched by domain rule '%s'` ~`L196`) |
| `L193` | `defaultSource` fallback (debug `matched by default rule` ~`L194`) |
| `L199` | debug `matched by address rule` |

### 11.5 `internal/modify/dkim/dkim.go` (signing decision)

| Anchor | What it is |
|---|---|
| `L31-52` | `oversignDefault` — 15 oversigned headers (Subject, Sender, To, Cc, From, Date, MIME‑Version, Content‑Type, Content‑Transfer‑Encoding, Reply‑To, In‑Reply‑To, Message‑Id, References, Autocrypt, Openpgp) |
| `L55-72` | `signDefault` |
| `L137` | `key_path` default `dkim_keys/{domain}_{selector}.key` |
| `L142` / `L145` | header/body canonicalization defaults `relaxed`/`relaxed` |
| `L147-148` | `hash sha256` default |
| `L149-150` | `newkey_algo rsa2048` default |
| `L151-152` | **`require_sender_match` default `{envelope, auth}`** |
| `L202` | `fieldsToSign` — lists each header (occurrences+1) times |
| `L249` | **`shouldSign`** — the signing gate |
| `L266` / `L271` / `L275` | gates: empty From / malformed From / multiple From |
| `L287-288` | **From‑domain ≠ key‑domain gate (Scenario D fails here)** — debug `not signing, From domain is not key domain` |
| `L293-294` | envelope‑mismatch gate — debug `not signing, From address is not envelope address` (not triggered by any of the four scenarios; quoted from source) |
| `L299-306` | **auth‑identity gate (Scenario A fails at L306)** — debug `not signing, From address is not authenticated identity` |
| `L340` | `RewriteBody` |
| `L345` | `authUser = s.meta.Conn.AuthUser` |
| `L348` | `shouldSign` call (passes `OriginalFrom`) |
| `L406` | **`h.Add("DKIM-Signature", signer.SignatureValue())` (Scenario C signs here)** |
| `L408` | `signed` debug line |

### 11.6 `internal/modify/dkim/keys.go` (key auto‑generation)

| Anchor | What it is |
|---|---|
| `L19` | `loadOrGenerateKey` |
| `L77` | `generateAndWrite` |
| `L82` | log `generating a new rsa2048 keypair...` |
| `L95` | `rsa.GenerateKey(…, 2048)` |
| `L105` | `x509.MarshalPKCS8PrivateKey` (private key = PKCS#8) |
| `L121-131` | PEM write, `0600`, block type `"PRIVATE KEY"` |
| `L136` | `writeDNSRecord` |
| `L143` | **`x509.MarshalPKCS1PublicKey(pubkey)` — the PKCS#1 (non‑PKIX) public‑key defect** |
| `L150` | `.dns` path = `keyPath[:-4]+".dns"` |
| `L158` | record string `v=DKIM1; k=rsa; p=<base64>` |

### 11.7 `internal/target/received.go` (`Received` header)

| Anchor | What it is |
|---|---|
| `L19` | `GenerateReceived` |
| `L30-31` | `from <host> (<rdns> [<ip>])` clause guarded by `!DontTraceSender` (suppressed on submission) |
| `L62-63` | `by <hostname>` |
| `L69-71` | **`(envelope-sender <MAIL FROM>)`** — always embeds the true envelope sender |
| `L74-79` | `with <Proto>` |
| `L81-84` | `id <msgID>; <date>` |

### 11.8 Other references

| Anchor | What it is |
|---|---|
| `internal/check/` (dir) | contains only `command`, `dkim`, `dns`, `dnsbl`, `requiretls`, `spf` (+ infra `action.go`, `skeleton.go`, `stateless_check.go`) — **no `authorize_sender`** |
| `cmd/maddyctl/main.go:L47` | `users` command |
| `cmd/maddyctl/main.go:L71` | `users create` subcommand (stdin password; `--password/-p`, `--null/-n`, `--hash` default `bcrypt`) — **no `creds` command** |
| `cmd/maddyctl/main.go:L203,L207` | `imap-mboxes` / `imap-mboxes list` |
| `cmd/maddyctl/main.go:L307,L534` | `imap-msgs` / `imap-msgs dump` |
| `maddy.go:L59` | `DefaultStateDirectory = "/var/lib/maddy"` |
| `maddy.go:L102` | `Run` (flags `-config`, `-debug`, `-log`) |
| `maddy.go:L220` | **`os.Chdir(config.StateDirectory)`** |
| `maddy.go:L245-246` | `state` / `runtime` global directives |
| `examples/multitentant-dkim.conf` | domain/key‑scoped `source <domain> { sign_dkim … }` + `default_source { reject }` (corroborates domain‑scoped handling) |
| `docs/man/maddy-filters.5.scd:~L518-522,~L544` | documented `require_sender_match` — "otherwise — don't sign the message" |
| `go.mod:L1,L3` | module `github.com/foxcpp/maddy`, `go 1.13`; `mattn/go-sqlite3 v1.11.0` (CGO); `emersion/go-msgauth v0.3.2-0.20191028231513-55b75676976c`; `emersion/go-message v0.10.9-0.20191116124005-65fd0119e899`; `golang.org/x/crypto v0.0.0-20191108234033-bd318be0434a` |

---

*End of investigation. All observations were produced by building and running maddy at commit `26452dd8dd787dc455278b0fdd296f4a5432c768` inside the canonical container; every working artifact was removed afterward, leaving the repository byte‑for‑byte unchanged except for this document.*

