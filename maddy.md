# DKIM Signing Investigation: foxcpp/maddy Mail Server

This document is a self-contained technical investigation walkthrough of the [foxcpp/maddy](https://github.com/foxcpp/maddy) mail server's DKIM signing execution path, Message-ID generation logic, and build verification. Every output value reproduced below was obtained by compiling and running Go code against the same library versions used by the maddy codebase — no external references or manual construction were used.

The investigation covers five areas:

1. **Build Verification** — building the maddy binary from source and capturing its version string.
2. **DKIM Key Generation** — generating RSA-2048 and Ed25519 signing keys and examining their DNS TXT records.
3. **Sample Message** — constructing a multi-header test email for DKIM field analysis.
4. **Fields-to-Sign Analysis** — tracing the `fieldsToSign` algorithm to understand oversigning vs. signing behavior.
5. **Message-ID Observation** — exercising the UUID-based Message-ID generation path.

---

## 1. Build Verification

### Build Environment

| Attribute | Value |
|---|---|
| Go toolchain | go1.22.2 linux/amd64 (compatible with `go 1.13` directive in `go.mod` line 3) |
| CGO requirement | `CGO_ENABLED=1` — required for `github.com/mattn/go-sqlite3` v1.11.0 compilation |
| C compiler | gcc 13.3.0 (`build-essential`) |
| Build entrypoint | `cmd/maddy/main.go` — a minimal launcher that calls `maddy.Run()` |
| Version variable | `maddy.go:41` — `Version = "unknown (built from source tree)"` |
| Version function | `maddy.go:89-97` — `BuildInfo()` returns `Version` when `debug.ReadBuildInfo().Main.Version == "(devel)"` |

### Build Command

```bash
CGO_ENABLED=1 go build -o /tmp/maddy-bin/maddy ./cmd/maddy/
```

The binary was placed at `/tmp/maddy-bin/maddy` (outside the repository tree, per constraint).

### Verbatim Version Output

```
$ /tmp/maddy-bin/maddy -v
maddy unknown (built from source tree)
```

This output originates from `maddy.go:125-127`:

```go
if *printVersion {
    fmt.Println("maddy", BuildInfo())
    return 0
}
```

When built from a local source tree (not via `go install` with a module version), `debug.ReadBuildInfo()` reports `Main.Version` as `"(devel)"`, causing `BuildInfo()` (`maddy.go:89-97`) to return the `Version` variable, which defaults to `"unknown (built from source tree)"` (`maddy.go:41`).

---

## 2. DKIM Key Generation

### Method

Key generation replicates the logic in `internal/modify/dkim/keys.go:77-134` (`generateAndWrite`) and `internal/modify/dkim/keys.go:136-163` (`writeDNSRecord`).

- **RSA-2048** is the codebase default `newkey_algo` (per `internal/modify/dkim/dkim.go:150`: `"rsa2048"`). The key is generated via `rsa.GenerateKey(rand.Reader, 2048)` (`keys.go:95`). The public key is serialized using `x509.MarshalPKCS1PublicKey()` (`keys.go:143`), producing a PKCS#1 DER-encoded blob (ASN.1 `SEQUENCE { INTEGER modulus, INTEGER exponent }`).
- **Ed25519** keys are generated via `ed25519.GenerateKey(rand.Reader)` (`keys.go:97`). The public key is the raw 32-byte value (`keys.go:145`: `keyBlob = pubkey`).

The DNS TXT record format (`keys.go:158`) is:

```
v=DKIM1; k=<algorithm>; p=<base64-encoded-public-key>
```

The `.dns` file path is derived from the key path: if the key path ends in `.key`, the extension is replaced with `.dns`; otherwise `.dns` is appended (`keys.go:150-153`).

### Selector and Domain

| Parameter | Value |
|---|---|
| Selector | `default` |
| Domain | `example.com` |

### RSA-2048 DNS TXT Record

**Absolute `.dns` file path:** `/tmp/dkim-keys/example.com_default_rsa.dns`

```
v=DKIM1; k=rsa; p=MIIBCgKCAQEAtFIXN0TMUhaVGIrXY7w0XMbR+UmpbThYX6AOflHKvV6kdJ4KZ6kwXsj8XJeMSlqvmDM60QVFhr6yOZ2x4FWSbKCK+9UD9ORJtC0rTxmIJw7TGjqgrav0ugeqj+thq8hgk/GEuetD3oM2xbUXHZDlWBFf3NVmIt6MrbRU4O3m1GiivF7uLKpGi7fXa4PjQt6Zk8MkzibJGB8DmF/cGgUE/QmH6lxWDzOo1W5rKUpDPhLnpzSdBRZTkVCAUJ/UvUMcuzladW/ZVqdSMEJuX0TN35DwsfU466nnT9R3biOqXZJ338j0Xy8UmLttO1IzRJ61lQTAs5R6RchFgrQwUDgdOwIDAQAB
```

### Ed25519 DNS TXT Record

**Absolute `.dns` file path:** `/tmp/dkim-keys/example.com_default_ed25519.dns`

```
v=DKIM1; k=ed25519; p=wE+XOg/RhbrbwHMGa3SkkYg8XRs835xp43YbisjEt7c=
```

### Comparison Table

| Property | RSA-2048 | Ed25519 |
|---|---|---|
| Key generation function | `rsa.GenerateKey(rand.Reader, 2048)` | `ed25519.GenerateKey(rand.Reader)` |
| Public key serialization | `x509.MarshalPKCS1PublicKey()` (PKCS#1 DER) | Raw public key bytes |
| DKIM algorithm name (`k=`) | `rsa` | `ed25519` |
| Raw public key byte length | 270 | 32 |
| Base64 string length (`p=` value) | 360 | 44 |
| Base64 ends with `=` padding | No | Yes |
| `.dns` file path | `/tmp/dkim-keys/example.com_default_rsa.dns` | `/tmp/dkim-keys/example.com_default_ed25519.dns` |

### Base64 Padding Analysis

Base64 encoding maps every 3 input bytes to 4 output characters. Padding depends on `byte_length mod 3`:

- **RSA-2048:** 270 bytes. `270 mod 3 = 0` → no padding required. Base64 length = `270 × 4 / 3 = 360` characters. Confirmed: the base64 string does **not** end with `=`.
- **Ed25519:** 32 bytes. `32 mod 3 = 2` → single `=` padding. Base64 length = `⌈32 / 3⌉ × 4 = 44` characters. Confirmed: the base64 string **does** end with `=`.

The RSA-2048 raw byte count of 270 arises from the PKCS#1 DER encoding: the 2048-bit modulus occupies 256 bytes, plus the public exponent (typically 65537 = 3 bytes) and ASN.1 structure overhead (SEQUENCE tags, length fields, INTEGER tags with leading zero bytes for sign extension), totaling ~270 bytes. The exact count varies slightly depending on the leading bytes of the modulus.

---

## 3. Sample Message

The following sample email was constructed for the fields-to-sign analysis. It contains exactly **3 `From` headers**, exactly **3 `List-Id` headers**, **no other headers**, and a single non-empty body line. The header object was built using the `github.com/emersion/go-message/textproto` library (`go.mod:17`: version `v0.10.9-0.20191116124005-65fd0119e899`), the same library used by the DKIM signing module (`internal/modify/dkim/dkim.go:14`).

```
From: alice@example.com
From: bob@example.com
From: charlie@example.com
List-Id: list1.example.com
List-Id: list2.example.com
List-Id: list3.example.com

Hello, this is a test message body for DKIM signing investigation.
```

### Header Count Verification

| Header Name | Count |
|---|---|
| `From` | 3 |
| `List-Id` | 3 |
| All other headers | 0 |
| **Total header fields** | **6** |

---

## 4. Fields-to-Sign Analysis

### The `fieldsToSign` Algorithm

The `fieldsToSign` function (`internal/modify/dkim/dkim.go:202-233`) computes the list of header field names to include in the DKIM signature's `h=` tag. It operates on two configuration lists:

- **`oversignDefault`** (`dkim.go:31-54`) — 15 headers that are both signed and **oversigned**:

  ```
  Subject, Sender, To, Cc, From, Date, MIME-Version, Content-Type,
  Content-Transfer-Encoding, Reply-To, In-Reply-To, Message-Id,
  References, Autocrypt, Openpgp
  ```

- **`signDefault`** (`dkim.go:55-72`) — 12 headers that are signed but **not oversigned**:

  ```
  List-Id, List-Help, List-Unsubscribe, List-Post, List-Owner, List-Archive,
  Resent-To, Resent-Sender, Resent-Message-Id, Resent-Date, Resent-From, Resent-Cc
  ```

The algorithm works as follows (`dkim.go:202-233`):

1. A `seen` map prevents duplicate processing of the same header name (case-insensitive).
2. For each header in `oversignDefault`:
   - Skip if already seen.
   - Add the header name **once for each occurrence** in the message (via `h.FieldsByKey(key)`).
   - Add the header name **once more** — this is the "oversign" entry.
3. For each header in `signDefault`:
   - Skip if already seen.
   - Add the header name **once for each occurrence** in the message.
   - **No extra oversign entry.**

### Verbatim Fields-to-Sign List

Applied to the sample message (3 `From`, 3 `List-Id`), the function produces the following list, printed one header per line:

```
Subject
Sender
To
Cc
From
From
From
From
Date
MIME-Version
Content-Type
Content-Transfer-Encoding
Reply-To
In-Reply-To
Message-Id
References
Autocrypt
Openpgp
List-Id
List-Id
List-Id
```

### Counts

| Metric | Value |
|---|---|
| Total entries | 21 |
| Unique header names | 16 |

### Breakdown by Header

**Oversigned headers (from `oversignDefault`):**

| Header | Occurrences in message | + Oversign | = Total entries |
|---|---|---|---|
| `Subject` | 0 | +1 | 1 |
| `Sender` | 0 | +1 | 1 |
| `To` | 0 | +1 | 1 |
| `Cc` | 0 | +1 | 1 |
| **`From`** | **3** | **+1** | **4** |
| `Date` | 0 | +1 | 1 |
| `MIME-Version` | 0 | +1 | 1 |
| `Content-Type` | 0 | +1 | 1 |
| `Content-Transfer-Encoding` | 0 | +1 | 1 |
| `Reply-To` | 0 | +1 | 1 |
| `In-Reply-To` | 0 | +1 | 1 |
| `Message-Id` | 0 | +1 | 1 |
| `References` | 0 | +1 | 1 |
| `Autocrypt` | 0 | +1 | 1 |
| `Openpgp` | 0 | +1 | 1 |
| **Subtotal** | | | **18** |

**Signed-only headers (from `signDefault`):**

| Header | Occurrences in message | + Oversign | = Total entries |
|---|---|---|---|
| **`List-Id`** | **3** | **+0** | **3** |
| `List-Help` | 0 | +0 | 0 |
| `List-Unsubscribe` | 0 | +0 | 0 |
| `List-Post` | 0 | +0 | 0 |
| `List-Owner` | 0 | +0 | 0 |
| `List-Archive` | 0 | +0 | 0 |
| `Resent-To` | 0 | +0 | 0 |
| `Resent-Sender` | 0 | +0 | 0 |
| `Resent-Message-Id` | 0 | +0 | 0 |
| `Resent-Date` | 0 | +0 | 0 |
| `Resent-From` | 0 | +0 | 0 |
| `Resent-Cc` | 0 | +0 | 0 |
| **Subtotal** | | | **3** |

**Grand total: 18 + 3 = 21 entries across 16 unique header names.**

### Oversigned vs. Signed-Only: The Key Difference

The numeric difference between `From` (4 entries) and `List-Id` (3 entries) illustrates the core distinction:

- **`From` (oversigned):** The message contains 3 `From` headers. The algorithm adds all 3, then adds **one more** "phantom" entry. This extra entry tells the DKIM verifier: "if a `From` header appears in the message that I didn't sign, the signature is invalid." This protects against a post-signature attacker injecting a new `From` header — a critical security property for the header that identifies the message sender.

- **`List-Id` (signed-only):** The message contains 3 `List-Id` headers. The algorithm adds all 3, but does **not** add an extra entry. This means a downstream mailing list manager (MLM) could legitimately prepend a new `List-Id` header without breaking the DKIM signature. The code comment at `dkim.go:56-57` explains this design choice: *"Mailing list information. Not oversigned to prevent signature breakage by aliasing MLMs."*

In general, headers that are critical to security or user-visible identity (`From`, `Subject`, `Date`, etc.) are oversigned to prevent tampering. Headers that intermediate relays may legitimately modify (`List-Id`, `Resent-*`) are signed-only to preserve signature validity through the delivery chain.

### Cross-Reference with Unit Tests

The `TestFieldsToSign` test (`internal/modify/dkim/dkim_test.go:16-36`) validates this exact behavior using a simplified scenario: with `oversignHeader=["A", "B"]` and `signHeader=["C"]`, where the header contains A×2, C×2, B×1, the expected sorted result is `["A", "A", "A", "B", "B", "C", "C"]` — demonstrating that `A` gets `2+1=3` entries (oversigned), `B` gets `1+1=2` entries (oversigned), and `C` gets `2+0=2` entries (signed-only).

---

## 5. Message-ID Observation

### Method

The Message-ID generation logic is defined in `internal/endpoint/smtp/submission.go:16-22`:

```go
msgIDField = func() (string, error) {
    id, err := uuid.NewRandom()
    if err != nil {
        return "", err
    }
    return id.String(), nil
}
```

This uses `github.com/google/uuid` v1.1.1 (`go.mod:23`). `uuid.NewRandom()` generates a Version 4 (random) UUID.

In `submissionPrepare` (`submission.go:30-37`), the Message-ID header is formatted as:

```go
header.Set("Message-ID", "<"+msgId+"@"+s.endp.serv.Domain+">")
```

### Five Real Message-ID Outputs

The following values were generated by calling `uuid.NewRandom()` five times and formatting each as `<UUID@example.com>`:

```
<bc98fd02-5d4b-4cc3-bf82-92646e6a3371@example.com>
<96e31028-0571-4c6c-8ae5-12e01b21f29f@example.com>
<15e93008-98b7-49a0-a904-df77d498ee62@example.com>
<d237e48b-4969-48dc-bf98-1f25665273dc@example.com>
<b0bb1ad7-4319-4cd8-8b8c-3edf538339c1@example.com>
```

### Format and Length Analysis

| Property | Value |
|---|---|
| UUID format | `xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx` (8-4-4-4-12 hex digits with hyphens) |
| UUID version nibble | `4` (Version 4: randomly generated) |
| UUID variant bits | `10xx` (RFC 4122 variant) |
| UUID string length | 36 characters |
| Message-ID format | `<UUIDv4@domain>` |
| Message-ID length (with `example.com`) | 50 characters (`1 + 36 + 1 + 11 + 1`) |

Each UUIDv4 contains 122 bits of randomness (128 bits minus 4 version bits and 2 variant bits), providing effectively unique Message-ID values for each email processed by the server.

---

## 6. Source Code References

All source files consulted during this investigation, with their purpose and key line references:

| File | Purpose | Key Lines |
|---|---|---|
| `internal/modify/dkim/dkim.go` | DKIM signing module: `oversignDefault` and `signDefault` header lists, `fieldsToSign` function, module initialization | Lines 31–54 (oversignDefault), 55–72 (signDefault), 202–233 (fieldsToSign), 126–200 (Init), 150 (newkey_algo default) |
| `internal/modify/dkim/keys.go` | DKIM key lifecycle: key generation, PKCS#8 private key writing, DNS TXT record writing | Lines 19–75 (loadOrGenerateKey), 77–134 (generateAndWrite), 136–163 (writeDNSRecord) |
| `internal/modify/dkim/dkim_test.go` | Unit tests for `fieldsToSign` and `shouldSign` behavior | Lines 16–36 (TestFieldsToSign) |
| `internal/modify/dkim/keys_test.go` | Unit tests for Ed25519 key generation and PKCS#8/PKCS#1 key loading | Lines 16–58 (TestKeyLoad_new) |
| `internal/endpoint/smtp/submission.go` | Message-ID generation via UUIDv4, submission preparation pipeline | Lines 16–22 (msgIDField), 27–37 (submissionPrepare Message-ID insertion) |
| `cmd/maddy/main.go` | Binary build entrypoint — minimal launcher calling `maddy.Run()` | Lines 1–11 (entire file) |
| `maddy.go` | Server bootstrap, `Version` variable, `BuildInfo()` function, `Run()` entry point | Line 41 (Version), 89–97 (BuildInfo), 102–164 (Run), 125–127 (version print) |
| `go.mod` | Go module identity (`github.com/foxcpp/maddy`), minimum Go version (`go 1.13`), dependency versions | Line 3 (go 1.13), 17 (go-message), 18 (go-msgauth), 23 (google/uuid v1.1.1), 30 (x/crypto) |
