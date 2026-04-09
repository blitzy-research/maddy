# DKIM Signing Investigation: foxcpp/maddy Mail Server

> **Purpose**: This document walks through one concrete DKIM signing execution path inside the `foxcpp/maddy` mail server codebase. Every finding is grounded in the actual source code — built, run, and observed empirically. No external documentation or assumptions are used; the code is the sole source of truth.

---

## 1. Building Maddy from Source

### Build Entrypoint

The build entrypoint is `cmd/maddy/main.go`. The entire file is minimal — it declares `package main`, imports `os` and `github.com/foxcpp/maddy`, and calls `os.Exit(maddy.Run())`:

```go
// Source: cmd/maddy/main.go:1-11
package main

import (
    "os"

    "github.com/foxcpp/maddy"
)

func main() {
    os.Exit(maddy.Run())
}
```

This means `maddy.Run()` (defined in `maddy.go`) is the true entry point for all server logic, including module registration. The DKIM signing module (`internal/modify/dkim`) is registered via a blank import at `maddy.go:33`:

```go
_ "github.com/foxcpp/maddy/internal/modify/dkim"
```

### Build Command

The binary is compiled with the standard Go toolchain and placed **outside** the repository tree (to satisfy the constraint of not modifying any repository files):

```bash
go build -o /tmp/maddy-build/maddy ./cmd/maddy/
```

**Build notes**:
- Go **1.22.2** was used for compilation; the `go.mod` file (line 3) specifies a minimum of **Go 1.13** (`go.mod:3`).
- **CGO is required** because the project depends on `mattn/go-sqlite3` (v1.11.0, listed in `go.mod:26`), which is a C-based SQLite3 driver that requires a C compiler (GCC 13.3.0 was used).

### Version Output

Running the compiled binary with the version flag produces:

```
maddy unknown (built from source tree)
```

**Why this output appears**: The `Version` variable in `maddy.go` (line 41) is hard-coded to:

```go
// Source: maddy.go:41
Version = "unknown (built from source tree)"
```

The `BuildInfo()` function (`maddy.go:89-97`) is responsible for determining what version string to display. It calls `debug.ReadBuildInfo()` from the Go standard library. When the module version is `"(devel)"` — which is exactly what happens when building from a local source tree rather than from a tagged release — the function returns the `Version` variable directly:

```go
// Source: maddy.go:89-97
func BuildInfo() string {
    if info, ok := debug.ReadBuildInfo(); ok {
        if info.Main.Version == "(devel)" {
            return Version
        }
        return info.Main.Version + " " + info.Main.Sum
    }
    return Version + " (GOPATH build)"
}
```

Since we built from the source tree, `info.Main.Version` is `"(devel)"`, so the function returns `Version` which is `"unknown (built from source tree)"`. The final output format is `maddy <BuildInfo()>`, yielding `maddy unknown (built from source tree)`.

---

## 2. DKIM Key Generation

The DKIM key lifecycle begins when the `Modifier.Init()` method (`internal/modify/dkim/dkim.go:126`) processes the module configuration. It resolves the key file path using a template (default: `dkim_keys/{domain}_{selector}.key` per `dkim.go:137`) and calls `loadOrGenerateKey(keyPath, newKeyAlgo)` (`keys.go:19`). If the key file does not exist on disk, `generateAndWrite` (`keys.go:77`) is invoked to create a new keypair and its companion DNS TXT record file.

### 2.1 RSA-2048 (Default Algorithm)

**RSA-2048 is the default** because the `newkey_algo` configuration option defaults to `"rsa2048"`:

```go
// Source: internal/modify/dkim/dkim.go:149-150
cfg.Enum("newkey_algo", false, false,
    []string{"rsa4096", "rsa2048", "ed25519"}, "rsa2048", &newKeyAlgo)
```

#### Key Generation Path

1. **`generateAndWrite`** (`keys.go:77`) is called with `newKeyAlgo = "rsa2048"`.
2. The `switch` statement at `keys.go:89` matches the `"rsa2048"` case:
   - Sets `dkimName = "rsa"` (`keys.go:94`) — this becomes the `k=` tag value in the DNS record.
   - Calls `rsa.GenerateKey(rand.Reader, 2048)` (`keys.go:95`) to generate a 2048-bit RSA keypair.
3. The private key is marshaled to PKCS#8 DER format via `x509.MarshalPKCS8PrivateKey(pkey)` (`keys.go:105`).
4. `writeDNSRecord(keyPath, dkimName, pkey)` is called (`keys.go:116`) to create the DNS TXT record file.
5. The private key is written as a PEM-encoded `"PRIVATE KEY"` block to the key file with `0600` permissions (`keys.go:121-131`).

#### DNS Record Generation (`writeDNSRecord`)

The `writeDNSRecord` function (`keys.go:136-163`) performs the following:

1. Extracts the **public key** from the signer. For RSA, it type-asserts to `*rsa.PublicKey` and calls `x509.MarshalPKCS1PublicKey(pubkey)` (`keys.go:143`), which produces the DER-encoded PKCS#1 representation of the RSA public key.
2. Determines the `.dns` file path: if the key path ends with `.key`, the extension is replaced with `.dns`; otherwise `.dns` is appended (`keys.go:150-153`).
3. Writes the DNS TXT record in the format `v=DKIM1; k=%s; p=%s` (`keys.go:158`), where `%s` values are the DKIM algorithm name and the base64-encoded public key blob.

#### Verbatim RSA-2048 DNS TXT Record

From `/tmp/dkim-keys/example.com_test2025.dns`:

```
v=DKIM1; k=rsa; p=MIIBCgKCAQEAmGNE+coBVMhWx4kC0Vw4k4pbnwcR3ZoRRULERsN7mXI+18VHnxgW3cD/SPyQYDUUWsW8Fi3I7mFzoZuYMxzoETxFBweLuV2jvKKwSnlh9OqPaR7elud9VJtlinngdZHJSMbDuN2k/5tP64nroCLOvY8x6Ltp9pMCo4loXH4quOkvbF8jwjep/pyspV4xNe9rvVsq+04E8Nk6PL0cuSGlW0Dg63T7U5+FwVF9BlLg7puYLQ12RLM3anllJGeoLSWMLJAiBbxRvNZizGtHARWhqvL0JsyoXcwpLr0boz8Ln7swqTYWG8ZLBzTmY9Eb2gG2KxjYagF4viyRaW4XmmC3uwIDAQAB
```

#### RSA-2048 Key Analysis

| Property | Value |
|----------|-------|
| **Base64 length** | 360 characters |
| **Raw public key byte length** | 270 bytes (PKCS#1 DER encoding via `x509.MarshalPKCS1PublicKey`) |
| **Base64 `=` padding** | None — the base64 string does not end with `=` |
| **Absolute `.dns` file path** | `/tmp/dkim-keys/example.com_test2025.dns` |
| **Selector** | `test2025` |
| **Domain** | `example.com` |

The absence of base64 padding is expected: 270 bytes is evenly divisible by 3 (270 ÷ 3 = 90), so standard base64 encoding produces exactly 360 characters (90 × 4) with no remainder requiring `=` padding.

### 2.2 Ed25519

For the `"ed25519"` algorithm, the key generation path differs in two important ways:

1. **Key generation**: `generateAndWrite` calls `ed25519.GenerateKey(rand.Reader)` (`keys.go:97`) instead of `rsa.GenerateKey`. The `dkimName` variable retains its initial value of `"ed25519"` (set at `keys.go:86` as `dkimName = newKeyAlgo`, which is `"ed25519"`).

2. **Public key encoding in DNS record**: In `writeDNSRecord`, the ed25519 case at `keys.go:144-145` does NOT use any DER or PKCS encoding. Instead, it assigns the raw public key bytes directly:

```go
// Source: internal/modify/dkim/keys.go:144-145
case ed25519.PublicKey:
    keyBlob = pubkey
```

This means the DNS TXT record contains the base64 encoding of the **raw 32-byte ed25519 public key** — no ASN.1 wrapper, no PKCS structure, just the 32 bytes of the public key itself.

The private key, however, is still marshaled via `x509.MarshalPKCS8PrivateKey` and written as a PEM `"PRIVATE KEY"` block, the same as RSA (`keys.go:105-131`).

#### Verbatim Ed25519 DNS TXT Record

From `/tmp/dkim-keys/example.com_test2025_ed25519.dns`:

```
v=DKIM1; k=ed25519; p=ZSA+bjhlMSpGCkeuwPcQ/n7LmfN6q9UfoSxe/c3BIAM=
```

#### Ed25519 Key Analysis

| Property | Value |
|----------|-------|
| **Base64 length** | 44 characters |
| **Raw public key byte length** | 32 bytes (raw ed25519 public key bytes) |
| **Base64 `=` padding** | Ends with `=` (one padding character) |
| **Absolute `.dns` file path** | `/tmp/dkim-keys/example.com_test2025_ed25519.dns` |
| **Selector** | `test2025` (with `_ed25519` suffix in filename only) |
| **Domain** | `example.com` |

The single `=` padding character is expected: 32 bytes produces ⌈32/3⌉ = 11 groups with a 2-byte remainder, yielding 44 base64 characters where the last character is `=` padding.

### 2.3 Comparative Analysis

| Property | RSA-2048 | Ed25519 |
|----------|----------|---------|
| Algorithm config value | `rsa2048` | `ed25519` |
| DKIM `k=` tag | `rsa` | `ed25519` |
| Public key encoding | `x509.MarshalPKCS1PublicKey` (DER) | Raw bytes |
| Raw public key bytes | 270 | 32 |
| Base64 length | 360 chars | 44 chars |
| Base64 `=` padding | None | Yes (1 char) |
| Private key format | PKCS#8 PEM | PKCS#8 PEM |

**Key observation**: The ed25519 DNS TXT record is approximately **8× shorter** than the RSA-2048 record (44 vs. 360 base64 characters). This is significant for DNS TXT record size constraints — a single UDP DNS response has a 512-byte practical limit without EDNS, and the RSA-2048 public key alone consumes 360 characters of that budget. Ed25519 fits comfortably with room to spare.

Both algorithms share the same private key storage format (PKCS#8 PEM), which means existing key management workflows work identically regardless of algorithm choice. The difference is entirely in the DNS record size and the public key encoding path within `writeDNSRecord` (`keys.go:141-148`).

---

## 3. Sample Message Construction

To investigate how the `fieldsToSign` algorithm treats different header types, we construct a sample message specifically designed to expose the behavioral difference between **oversigned** headers and **sign-only** headers.

### The Sample Message

```
From: alice@example.com
From: bob@example.com
From: carol@example.com
List-Id: list1.example.com
List-Id: list2.example.com
List-Id: list3.example.com

This is the body.
```

### Design Rationale

This message is deliberately crafted with two key properties:

1. **Three `From` headers**: `From` appears in the `oversignDefault` list (`dkim.go:37`). The `fieldsToSign` algorithm will iterate all three instances AND add one extra "oversign" entry, producing **4 total entries** for `From`.

2. **Three `List-Id` headers**: `List-Id` appears in the `signDefault` list (`dkim.go:58`). The `fieldsToSign` algorithm will iterate all three instances but will NOT add an extra entry, producing **3 total entries** for `List-Id`.

3. **No other headers**: By omitting all other headers, we can clearly see that headers in `oversignDefault` that are absent from the message still get exactly **1 entry** (the oversign), while headers in `signDefault` that are absent get **0 entries**.

4. **Non-empty body**: The body line `"This is the body."` ensures the message is a complete, well-formed email with a header/body separator (the blank line) and content. While the body is not directly relevant to the `fieldsToSign` algorithm (which operates only on headers), a complete message is needed for end-to-end DKIM signing.

The two header types — `From` (oversigned) and `List-Id` (sign-only) — are set to the same count (3 instances each) so the behavioral difference becomes immediately apparent: same input count, different output count.

---

## 4. Fields-to-Sign Analysis

The `fieldsToSign` method (`internal/modify/dkim/dkim.go:202-233`) determines which header fields are included in the DKIM signature's `h=` tag. This is the core algorithm that differentiates oversigning from regular signing.

### 4.1 Verbatim Fields-to-Sign List

Running the `fieldsToSign` algorithm with the sample message above, using the default `oversignDefault` (15 headers, `dkim.go:31-54`) and `signDefault` (12 headers, `dkim.go:55-72`) lists, produces exactly **21 entries**:

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

### 4.2 Oversign vs. Sign-Only Explanation

The `fieldsToSign` method processes headers in two distinct passes. A `seen` map (`dkim.go:205`) provides case-insensitive deduplication via `strings.ToLower(key)` (`dkim.go:209-210, 222-223`) to ensure each header name is processed only once, even if it appears in both lists.

#### Pass 1: Oversign Loop (`dkim.go:208-220`)

For each header name in `m.oversignHeader` (defaults to `oversignDefault`):

1. Skip if already processed (checked via the `seen` map).
2. Iterate all instances of that header in the message using `h.FieldsByKey(key)`, appending **one entry per instance found** (`dkim.go:215-216`).
3. Append **one additional entry** — the "oversign" (`dkim.go:219`). This extra entry tells the DKIM verifier to reject any additional instance of that header that might be added after signing.

**Examples from our sample message**:

- **`From`** (3 instances in message): The loop at `dkim.go:215-216` appends `From` three times (once per instance). Then `dkim.go:219` appends one more `From` (the oversign). Result: **4 entries** for `From`.
- **`Subject`** (0 instances in message): The `FieldsByKey` iterator produces nothing, so no instance entries are added. But the oversign entry at `dkim.go:219` is still appended. Result: **1 entry** for `Subject`. This means the DKIM signature will invalidate if someone inserts a `Subject` header after signing.
- The same pattern applies to all other `oversignDefault` headers not present in the message (`Sender`, `To`, `Cc`, `Date`, `MIME-Version`, `Content-Type`, `Content-Transfer-Encoding`, `Reply-To`, `In-Reply-To`, `Message-Id`, `References`, `Autocrypt`, `Openpgp`): each contributes exactly **1 entry** (the oversign).

#### Pass 2: Sign-Only Loop (`dkim.go:221-231`)

For each header name in `m.signHeader` (defaults to `signDefault`):

1. Skip if already processed (checked via the `seen` map).
2. Iterate all instances of that header in the message using `h.FieldsByKey(key)`, appending **one entry per instance found** (`dkim.go:228-229`).
3. **No additional "oversign" entry is appended.** This is the critical difference.

The code comment at `dkim.go:56-57` explains the rationale:

> *"Not oversigned to prevent signature breakage by aliasing MLMs."*

Mailing List Managers (MLMs) may add or modify `List-*` and `Resent-*` headers during message redistribution. Oversigning these headers would cause DKIM verification failures when the MLM adds new instances, breaking the signature unnecessarily.

**Examples from our sample message**:

- **`List-Id`** (3 instances in message): The loop appends `List-Id` three times (once per instance). No oversign entry is added. Result: **3 entries** for `List-Id`.
- **`List-Help`**, **`List-Unsubscribe`**, and all other `signDefault` headers (0 instances): No entries are appended at all. Result: **0 entries** for each.

#### The Key Behavioral Difference

| Header | List Membership | Instances in Message | Entries in Fields-to-Sign | Calculation |
|--------|----------------|---------------------|--------------------------|-------------|
| `From` | `oversignDefault` (`dkim.go:37`) | 3 | **4** | 3 instances + 1 oversign |
| `List-Id` | `signDefault` (`dkim.go:58`) | 3 | **3** | 3 instances + 0 oversign |
| `Subject` | `oversignDefault` (`dkim.go:33`) | 0 | **1** | 0 instances + 1 oversign |
| `List-Help` | `signDefault` (`dkim.go:59`) | 0 | **0** | 0 instances + 0 oversign |

The extra entry for `From` means: if anyone adds a 4th `From` header to the message after signing, the DKIM signature will **invalidate**. For `List-Id`, adding a 4th instance after signing will NOT invalidate the signature — which is exactly the desired behavior for mailing list headers.

### 4.3 Entry Counts

- **Total entries**: 21
- **Unique header names**: 16

#### Detailed Breakdown

**From `oversignDefault`** (15 headers processed):

| Header | Instances Found | Oversign Entry | Total Entries |
|--------|----------------|----------------|---------------|
| Subject | 0 | +1 | 1 |
| Sender | 0 | +1 | 1 |
| To | 0 | +1 | 1 |
| Cc | 0 | +1 | 1 |
| From | 3 | +1 | 4 |
| Date | 0 | +1 | 1 |
| MIME-Version | 0 | +1 | 1 |
| Content-Type | 0 | +1 | 1 |
| Content-Transfer-Encoding | 0 | +1 | 1 |
| Reply-To | 0 | +1 | 1 |
| In-Reply-To | 0 | +1 | 1 |
| Message-Id | 0 | +1 | 1 |
| References | 0 | +1 | 1 |
| Autocrypt | 0 | +1 | 1 |
| Openpgp | 0 | +1 | 1 |
| **Subtotal** | **3** | **+15** | **18** |

**From `signDefault`** (12 headers processed):

| Header | Instances Found | Oversign Entry | Total Entries |
|--------|----------------|----------------|---------------|
| List-Id | 3 | +0 | 3 |
| List-Help | 0 | +0 | 0 |
| List-Unsubscribe | 0 | +0 | 0 |
| List-Post | 0 | +0 | 0 |
| List-Owner | 0 | +0 | 0 |
| List-Archive | 0 | +0 | 0 |
| Resent-To | 0 | +0 | 0 |
| Resent-Sender | 0 | +0 | 0 |
| Resent-Message-Id | 0 | +0 | 0 |
| Resent-Date | 0 | +0 | 0 |
| Resent-From | 0 | +0 | 0 |
| Resent-Cc | 0 | +0 | 0 |
| **Subtotal** | **3** | **+0** | **3** |

**Grand total**: 18 + 3 = **21 entries**

**Unique header names**: The 21 entries contain only **16 distinct header names** because `From` appears 4 times and `List-Id` appears 3 times. The 16 unique names are: Subject, Sender, To, Cc, From, Date, MIME-Version, Content-Type, Content-Transfer-Encoding, Reply-To, In-Reply-To, Message-Id, References, Autocrypt, Openpgp, List-Id.

---

## 5. Message-ID Generation

The maddy codebase contains two distinct Message-ID generation mechanisms that serve entirely different purposes. This section focuses on the internal tracking ID generator and contrasts it with the email header generator.

### 5.1 GenerateMsgID() Outputs

The `GenerateMsgID()` function in `internal/msgpipeline/msgid.go` (lines 12-16) was invoked seven times, producing the following values:

```
c7214967
a5763bb5
f1f309c5
5dba1fa2
89db56ad
5929e20e
38cfe569
```

### 5.2 Format and Length Description

#### Implementation

The `GenerateMsgID` function (`internal/msgpipeline/msgid.go:12-16`) is concise:

```go
// Source: internal/msgpipeline/msgid.go:12-16
func GenerateMsgID() (string, error) {
    rawID := make([]byte, 4)
    _, err := rand.Read(rawID)
    return hex.EncodeToString(rawID), err
}
```

The algorithm is straightforward:

1. **Allocates a 4-byte slice**: `rawID := make([]byte, 4)` (`msgid.go:13`)
2. **Fills it with cryptographic random bytes**: `rand.Read(rawID)` from `crypto/rand` (`msgid.go:14`)
3. **Hex-encodes it**: `hex.EncodeToString(rawID)` (`msgid.go:15`)

#### Observed Format

- **Length**: Always **8 characters** (4 random bytes × 2 hex characters per byte = 8 characters)
- **Character set**: Lowercase hexadecimal `[0-9a-f]`
- **Entropy**: 32 bits (4 bytes × 8 bits) — sufficient for internal pipeline tracking but not globally unique
- **Purpose**: This is the **internal tracking Message-ID**, used in `module.MsgMeta` for correlating log entries and pipeline processing within a single maddy instance. It is NOT the email `Message-ID` header that appears in the delivered message.

#### Contrast with the Email `Message-ID` Header

A separate function in `internal/endpoint/smtp/submission.go` (lines 16-22) generates the RFC 5322-compliant `Message-ID` header for outgoing emails:

```go
// Source: internal/endpoint/smtp/submission.go:16-22
msgIDField = func() (string, error) {
    id, err := uuid.NewRandom()
    if err != nil {
        return "", err
    }
    return id.String(), nil
}
```

This function uses `uuid.NewRandom()` from `github.com/google/uuid` (v1.1.1, per `go.mod:23`) to generate a UUID v4 (128-bit random identifier). The UUID is then formatted as `<uuid@domain>` for the email's `Message-ID` header (`submission.go:36`):

```go
// Source: internal/endpoint/smtp/submission.go:36
header.Set("Message-ID", "<"+msgId+"@"+s.endp.serv.Domain+">")
```

| Property | `GenerateMsgID()` | `msgIDField()` |
|----------|-------------------|----------------|
| **Location** | `internal/msgpipeline/msgid.go:12-16` | `internal/endpoint/smtp/submission.go:16-22` |
| **Output format** | 8-char hex string (e.g., `c7214967`) | UUID v4 string (e.g., `550e8400-e29b-41d4-a716-446655440000`) |
| **Entropy** | 32 bits (4 bytes) | 122 bits (UUID v4) |
| **Purpose** | Internal pipeline tracking (`MsgMeta`) | RFC 5322 email `Message-ID` header |
| **Global uniqueness** | Not guaranteed | Practically guaranteed |
| **Library** | Go stdlib (`crypto/rand`, `encoding/hex`) | `github.com/google/uuid` |

---

## 6. Source File References

The following table lists every source file analyzed during this investigation, the specific line ranges examined, and what was documented from each:

| Source File | Key Lines | Content Documented |
|------------|-----------|-------------------|
| `cmd/maddy/main.go` | 1–11 | Build entrypoint; imports `os` and `github.com/foxcpp/maddy`, calls `os.Exit(maddy.Run())` |
| `maddy.go` | 41 | `Version = "unknown (built from source tree)"` — the hard-coded version string |
| `maddy.go` | 89–97 | `BuildInfo()` function — returns `Version` when `info.Main.Version == "(devel)"` |
| `maddy.go` | 33 | DKIM module registration: `_ "github.com/foxcpp/maddy/internal/modify/dkim"` |
| `internal/modify/dkim/dkim.go` | 31–54 | `oversignDefault` list — 15 header names that are oversigned by default |
| `internal/modify/dkim/dkim.go` | 55–72 | `signDefault` list — 12 header names that are signed (not oversigned) by default |
| `internal/modify/dkim/dkim.go` | 56–57 | Code comment explaining why `signDefault` headers are not oversigned (MLM aliasing) |
| `internal/modify/dkim/dkim.go` | 126 | `Modifier.Init()` — entry point for DKIM module configuration |
| `internal/modify/dkim/dkim.go` | 137 | Default `key_path` template: `dkim_keys/{domain}_{selector}.key` |
| `internal/modify/dkim/dkim.go` | 149–150 | `newkey_algo` default: `rsa2048` (options: `rsa4096`, `rsa2048`, `ed25519`) |
| `internal/modify/dkim/dkim.go` | 202–233 | `fieldsToSign` algorithm — oversign loop (208–220) and sign-only loop (221–231) |
| `internal/modify/dkim/keys.go` | 19 | `loadOrGenerateKey` — loads existing key or generates new one |
| `internal/modify/dkim/keys.go` | 77–133 | `generateAndWrite` — generates keypair, writes DNS record and PEM private key |
| `internal/modify/dkim/keys.go` | 86 | `dkimName = newKeyAlgo` — initial DKIM algorithm name assignment |
| `internal/modify/dkim/keys.go` | 94–95 | RSA-2048: `dkimName = "rsa"`, `rsa.GenerateKey(rand.Reader, 2048)` |
| `internal/modify/dkim/keys.go` | 97 | Ed25519: `ed25519.GenerateKey(rand.Reader)` |
| `internal/modify/dkim/keys.go` | 105 | Private key marshaling: `x509.MarshalPKCS8PrivateKey(pkey)` |
| `internal/modify/dkim/keys.go` | 121–131 | PEM file creation: `"PRIVATE KEY"` block with `0600` permissions |
| `internal/modify/dkim/keys.go` | 136–163 | `writeDNSRecord` — creates DNS TXT record file from public key |
| `internal/modify/dkim/keys.go` | 143 | RSA public key encoding: `x509.MarshalPKCS1PublicKey(pubkey)` |
| `internal/modify/dkim/keys.go` | 145 | Ed25519 public key encoding: `keyBlob = pubkey` (raw 32 bytes) |
| `internal/modify/dkim/keys.go` | 150–153 | DNS file path logic: `.key` → `.dns` extension replacement |
| `internal/modify/dkim/keys.go` | 158 | DNS record format: `v=DKIM1; k=%s; p=%s` |
| `internal/msgpipeline/msgid.go` | 12–16 | `GenerateMsgID()` — 4 random bytes hex-encoded to 8-char string |
| `internal/endpoint/smtp/submission.go` | 16–22 | `msgIDField` — UUID v4 generation for email `Message-ID` header |
| `internal/endpoint/smtp/submission.go` | 36 | `Message-ID` header format: `<uuid@domain>` |
| `go.mod` | 1–3 | Module path: `github.com/foxcpp/maddy`, Go version: `1.13` |
| `go.mod` | 17 | `github.com/emersion/go-msgauth v0.3.2-0.20191028231513` |
| `go.mod` | 16 | `github.com/emersion/go-message v0.10.9-0.20191116124005` |
| `go.mod` | 23 | `github.com/google/uuid v1.1.1` |
| `go.mod` | 26 | `github.com/mattn/go-sqlite3 v1.11.0` |
| `go.mod` | 30 | `golang.org/x/crypto v0.0.0-20191108234033` |

---

*This investigation was conducted by building and running the foxcpp/maddy source code. No external documentation or web sources were consulted — all findings are grounded exclusively in the codebase.*
