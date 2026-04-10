# SPF/DMARC Interaction in Maddy: Technical Investigation Report

## Summary

Three SMTP sessions were reproduced with controlled DNS records against the maddy `MsgPipeline` configured with `apply_spf` (default configuration) and `doDMARC=true`, but **without** the `verify_dkim` module. All three domains publish `v=spf1 -all` (guaranteed SPF Fail) but produce different internal outcomes:

- **Case A** (subdomain `sub.maddy-dmarc.com`, inherits DMARC `sp=none`): SPF enforcement applies normally — the message is **quarantined** (SMTP `250`, but `MsgMeta.Quarantine=true`).
- **Case B** (organizational domain `maddy-dmarc.com`, `p=reject`): SPF enforcement is **deferred** to DMARC, but DMARC cannot evaluate without DKIM results — the message is **accepted with no enforcement** (SMTP `250`, `MsgMeta.Quarantine=false`). This is the counterintuitive case.
- **Case C** (domain `no-dmarc.com`, no DMARC record): SPF enforcement applies normally — the message is **quarantined** (SMTP `250`, but `MsgMeta.Quarantine=true`), same as Case A.

The root cause is a deferral gap: `relyOnDMARC` in `spf.go:212-235` clears SPF enforcement when a non-none DMARC policy exists, but `EvaluateAlignment` in `evaluate.go:148` returns `ResultNone` when DKIM results are absent, which maps to `PolicyNone` in `verifier.go:142-143`, causing DMARC to take no action either.

The default `failAction` for SPF Fail is `check.FailAction{Quarantine: true}` (spf.go:70-73). This means SPF Fail with default configuration results in **quarantine** (250 response), not rejection (550). Both Cases A and C follow the exact same code path after `relyOnDMARC` returns `false` — they call `s.spfResult(res.res, res.err)` at spf.go:364, which invokes `s.c.failAction.Apply(...)` with the default quarantine action. Therefore, Cases A and C both produce `250` with `MsgMeta.Quarantine=true`. Case B is the outlier: it defers to DMARC (clears Quarantine and Reject at spf.go:359-360), and DMARC returns `PolicyNone` due to the `!dkimPresent` guard, so the message passes with no quarantine and no rejection.

## Controlled DNS Configuration

| Domain | Record Type | Value | Purpose |
|---|---|---|---|
| `_dmarc.maddy-dmarc.com` | TXT | `v=DMARC1; p=reject; sp=none` | Organizational DMARC: reject for org domain, none for subdomains |
| `maddy-dmarc.com` | TXT | `v=spf1 -all` | Guarantees SPF Fail for any sender |
| `sub.maddy-dmarc.com` | TXT | `v=spf1 -all` | Guarantees SPF Fail for any sender |
| `no-dmarc.com` | TXT | `v=spf1 -all` | Guarantees SPF Fail for any sender |
| `_dmarc.no-dmarc.com` | — | (no record) | No DMARC record exists for this domain |
| `_dmarc.sub.maddy-dmarc.com` | — | (no record) | No subdomain DMARC; inherits from org domain |

DNS interception was performed using `go-mockdns` (version pinned in `go.sum` as `v0.0.0-20191123143003`) with `PatchNet` to intercept `net.DefaultResolver`. This ensures both the SPF library (`blitiri.com.ar/go/spf`) and DMARC `FetchRecord` (evaluate.go:25,41) see these controlled DNS records during test execution.

## Case A — Subdomain with Inherited DMARC (sp=none)

**Test setup**: MAIL FROM `<sender@sub.maddy-dmarc.com>`, RFC5322 From header `sender@sub.maddy-dmarc.com`. Pipeline configuration: `apply_spf` check (default config) + `doDMARC=true`, **no** `verify_dkim` module.

### (1) SMTP Response After End-of-DATA

```
250 2.0.0 OK: queued
```

The message receives a `250` success response on the SMTP wire. However, it is **quarantined internally** — `MsgMeta.Quarantine` is set to `true` (check_runner.go:263-264). Quarantine does not produce a 4xx or 5xx SMTP error; it silently sets the quarantine flag and the message is delivered to a quarantine/junk destination.

### (2) Outcome Stage

**End of DATA.** The `apply_spf` check runs SPF evaluation asynchronously during `CheckConnection` (spf.go:303-317). The result is consumed in `CheckBody` (spf.go:338) which is invoked at end of DATA by `checkBody` in check_runner.go:245-259.

### (3) Relevant Server Log Lines

Maddy uses structured JSON logging (internal/log/log.go `formatMsg` → `marshalOrderedJSON`). Each log line has the format `<logger_name>: <message>\t{<ordered_json_fields>}`. The `msg_id` field is added by `target.DeliveryLogger` (internal/target/delivery.go:13) and appears in all per-message log entries. Variable IDs are replaced with `<id>` below.

```
apply_spf: result: fail (<nil>)	{"msg_id":"<id>"}
pipeline: quarantined	{"check":"apply_spf","msg_id":"<id>","reason":"SPF authentication failed","smtp_code":550,"smtp_enchcode":"5.7.23","smtp_msg":"SPF authentication failed"}
```

- **First line**: SPF evaluation result from the async goroutine (spf.go:315, `Debugf` level — appears only when debug is enabled). The `msg_id` comes from the logger's `Fields` map.
- **Second line**: From `runAndMergeResults` (check_runner.go:204) via `Logger.Error`, which extracts structured fields from the `SMTPError` (exterrors/smtp.go:72-91) and merges them with the logger's `msg_id` field. The `smtp_enchcode` is formatted via `EnhancedCode.FormatLog()` (exterrors/smtp.go:11-13).
- The `"deferring action due to a DMARC policy"` log line is **not** emitted for Case A because `relyOnDMARC` returns `false`.

### Rationale

1. **SPF evaluation** (spf.go:314): `spf.CheckHostWithSender` returns `Fail` because `v=spf1 -all` rejects all senders.

2. **`relyOnDMARC` call** (spf.go:351): `CheckBody` calls `relyOnDMARC(ctx, header)`.

3. **DMARC record fetch** (spf.go:219 → evaluate.go:21-68):
   - `FetchRecord` queries `_dmarc.sub.maddy-dmarc.com` → no record found.
   - Falls back to organizational domain via `publicsuffix.EffectiveTLDPlusOne("sub.maddy-dmarc.com")` → `maddy-dmarc.com` (evaluate.go:34).
   - Queries `_dmarc.maddy-dmarc.com` → finds `v=DMARC1; p=reject; sp=none`.
   - Returns `policyDomain="maddy-dmarc.com"`, record with `Policy=reject`, `SubdomainPolicy=none`.

4. **Subdomain policy substitution** (spf.go:228-232):
   - `policy = record.Policy` → `reject` (line 228).
   - `!dns.Equal("maddy-dmarc.com", "sub.maddy-dmarc.com")` → `true` AND `record.SubdomainPolicy != ""` → `true` (line 230).
   - `policy = record.SubdomainPolicy` → `none` (line 231).

5. **`relyOnDMARC` returns `false`** (spf.go:234): `none == PolicyNone` → `return false`.

6. **Normal SPF enforcement** (spf.go:364): `s.spfResult(res.res, res.err)` is called.

7. **`failAction.Apply`** (spf.go:155-165 → action.go:81-98): Default `failAction = {Quarantine: true}` (spf.go:70-73). The `Apply` method ORs `Quarantine=true` into the result (action.go:96). Result: `{Reason: SMTPError{550, 5.7.23, "SPF authentication failed"}, Quarantine: true, Reject: false}`.

8. **Pipeline quarantine** (check_runner.go:179-182, 203-206): `runAndMergeResults` detects `Quarantine=true`, logs `"quarantined"`, sets `cr.mergedRes.Quarantine = true`.

9. **DMARC evaluation** (check_runner.go:267-268): `Apply` is called. `EvaluateAlignment` (evaluate.go:148) finds `!dkimPresent` (no `verify_dkim` module), returns `{Authres: {Value: ResultNone, Reason: "Not enough information (required checks are disabled)"}}`. In `Apply` (verifier.go:142): `ResultNone` → returns `PolicyNone`. No DMARC rejection.

10. **`applyResults`** (check_runner.go:263-264): Sets `cr.msgMeta.Quarantine = true`. Returns nil. Message delivered with quarantine flag.

## Case B — Organizational Domain with p=reject

**Test setup**: MAIL FROM `<sender@maddy-dmarc.com>`, RFC5322 From header `sender@maddy-dmarc.com`. Same pipeline: `apply_spf` (default) + `doDMARC=true`, **no** `verify_dkim`.

### (1) SMTP Response After End-of-DATA

```
250 2.0.0 OK: queued
```

The message is **accepted with no quarantine and no rejection**. This is the counterintuitive case — despite `v=spf1 -all` causing SPF Fail AND `p=reject` in the DMARC record, the message passes through completely clean. `MsgMeta.Quarantine` remains `false`.

### (2) Outcome Stage

**End of DATA.** Same timing as Case A — SPF evaluation is async during `CheckConnection`, result consumed in `CheckBody` at end of DATA.

### (3) Relevant Server Log Lines

```
apply_spf: result: fail (<nil>)	{"msg_id":"<id>"}
apply_spf: deferring action due to a DMARC policy	{"msg_id":"<id>"}
pipeline: no check action	{"check":"apply_spf","msg_id":"<id>","reason":"SPF authentication failed","smtp_code":550,"smtp_enchcode":"5.7.23","smtp_msg":"SPF authentication failed"}
```

- **First line**: SPF evaluation result (spf.go:315, `Debugf` level — appears only when debug is enabled).
- **Second line**: From `CheckBody` (spf.go:353, `Printf` level — always logged because `res.res != spf.Pass`). This is the **key observable signal** unique to Case B.
- **Third line**: From `runAndMergeResults` (check_runner.go:191) via `Logger.Error`. Since `Quarantine=false` and `Reject=false` but `Reason != nil`, the code enters the `"no check action"` branch and logs the structured error fields for deployment testing purposes.

### Rationale

1. **SPF evaluation** (spf.go:314): Returns `Fail` (same as all cases, `v=spf1 -all`).

2. **`relyOnDMARC` call** (spf.go:351): `CheckBody` calls `relyOnDMARC(ctx, header)`.

3. **DMARC record fetch** (spf.go:219 → evaluate.go:21-68):
   - `FetchRecord` queries `_dmarc.maddy-dmarc.com` → finds `v=DMARC1; p=reject; sp=none` directly (no fallback needed).
   - Returns `policyDomain="maddy-dmarc.com"`, record with `Policy=reject`, `SubdomainPolicy=none`.

4. **Subdomain check** (spf.go:230): `!dns.Equal("maddy-dmarc.com", "maddy-dmarc.com")` → `false`. The subdomain branch is **not** taken. `policy` stays as `record.Policy` = `reject`.

5. **`relyOnDMARC` returns `true`** (spf.go:234): `reject != PolicyNone` → `return true`.

6. **Deferral logging** (spf.go:352-353): Since `res.res` is `Fail` (not `Pass`), `Printf` is used: `"deferring action due to a DMARC policy"`.

7. **SPF result computed then cleared** (spf.go:358-361):
   - `checkRes = s.spfResult(res.res, res.err)` → would normally have `{Quarantine: true}` from default `failAction`.
   - **Line 359**: `checkRes.Quarantine = false` — quarantine flag cleared.
   - **Line 360**: `checkRes.Reject = false` — reject flag cleared.
   - The returned `CheckResult` has `Reason` set (the SMTPError) but `Quarantine=false` and `Reject=false`. Only the `AuthResult` (SPF fail authentication result) is preserved.

8. **"No check action" log** (check_runner.go:187-191): In `runAndMergeResults`, since both `Quarantine` and `Reject` are false but `Reason` is non-nil, the code falls into the `else if subCheckRes.Reason != nil` branch (check_runner.go:187) and logs `"no check action"` with the error (check_runner.go:191). This is the "action ignore" case for deployment testing.

9. **DMARC evaluation** (check_runner.go:267-268): Same as Case A — `EvaluateAlignment` (evaluate.go:148) finds `!dkimPresent`, returns `ResultNone`. `Apply` (verifier.go:142) maps `ResultNone` to `PolicyNone`. **No DMARC rejection occurs.**

10. **The DMARC evaluation gap**: `apply_spf` deferred enforcement to DMARC (cleared its own Quarantine/Reject). But DMARC cannot make a decision without DKIM results (the `!dkimPresent` guard at evaluate.go:148), so it returns `PolicyNone`. The message falls through with **no enforcement from either SPF or DMARC**. Both `MsgMeta.Quarantine` and any rejection error remain unset.

## Case C — Domain with No DMARC Record

**Test setup**: MAIL FROM `<sender@no-dmarc.com>`, RFC5322 From header `sender@no-dmarc.com`. Same pipeline: `apply_spf` (default) + `doDMARC=true`, **no** `verify_dkim`.

### (1) SMTP Response After End-of-DATA

```
250 2.0.0 OK: queued
```

The message receives a `250` success response, identical to Cases A and B on the wire. Internally, however, it is **quarantined** — `MsgMeta.Quarantine=true`, same as Case A. This is because `relyOnDMARC` returns `false` (no DMARC record) and the default SPF `failAction` is `{Quarantine: true}`.

Cases A and C follow the **exact same code path** after `relyOnDMARC` returns `false`. Both call `s.spfResult(res.res, res.err)` at spf.go:364, which applies the default quarantine action. The behavioral contrast that matters is between Cases A/C (quarantined) and Case B (clean acceptance with no enforcement).

### (2) Outcome Stage

**End of DATA.** Same timing as Cases A and B.

### (3) Relevant Server Log Lines

```
apply_spf: result: fail (<nil>)	{"msg_id":"<id>"}
pipeline: quarantined	{"check":"apply_spf","msg_id":"<id>","reason":"SPF authentication failed","smtp_code":550,"smtp_enchcode":"5.7.23","smtp_msg":"SPF authentication failed"}
```

Same log output as Case A (same structured JSON format, same fields). The `"deferring action due to a DMARC policy"` line is **not** emitted because `relyOnDMARC` returns `false`.

### Rationale

1. **SPF evaluation**: Returns `Fail` (`v=spf1 -all`).

2. **`relyOnDMARC` call** (spf.go:351).

3. **DMARC record fetch** (evaluate.go:21-68):
   - Queries `_dmarc.no-dmarc.com` → no record found (NXDOMAIN).
   - Org domain fallback: `publicsuffix.EffectiveTLDPlusOne("no-dmarc.com")` → `no-dmarc.com` (same domain) (evaluate.go:34).
   - Queries `_dmarc.no-dmarc.com` again → still no record.
   - Returns `"", nil, nil` (evaluate.go:49-51 — `record == nil`).

4. **`relyOnDMARC` returns `false`** (spf.go:224-225): `record == nil` → `return false`.

5. **Normal SPF enforcement** (spf.go:364): Same as Case A. `failAction = {Quarantine: true}` → `{Quarantine: true, Reject: false}`.

6. **Pipeline quarantine** (check_runner.go:179-182, 203-206): Quarantine path triggered, logged.

7. **DMARC evaluation** (check_runner.go:267-268): `Apply` (verifier.go:111) receives the fetched data. Since `data.record == nil` (no DMARC record was found), `Apply` returns early at verifier.go:132-137 with `{Authres: {Value: ResultNone, From: "no-dmarc.com"}, PolicyNone}` — `EvaluateAlignment` is never called. Unlike Cases A and B, where the `!dkimPresent` guard in `EvaluateAlignment` (evaluate.go:148) produces the `ResultNone`, Case C's `ResultNone` comes from the nil-record short-circuit. The outcome is the same (`PolicyNone`), but the code path and the `Reason` field differ: Case C's `DMARCResult` has an empty `Reason`, while Cases A/B carry `"Not enough information (required checks are disabled)"`.

8. **Result**: Message quarantined internally. SMTP responds `250`.

## Comparison Summary

| Dimension | Case A (`sub.maddy-dmarc.com`) | Case B (`maddy-dmarc.com`) | Case C (`no-dmarc.com`) |
|---|---|---|---|
| SPF Record | `v=spf1 -all` | `v=spf1 -all` | `v=spf1 -all` |
| SPF Result | Fail | Fail | Fail |
| DMARC Record Found | Yes (inherited from org) | Yes (direct match) | No |
| Effective DMARC Policy | `sp=none` (subdomain) | `p=reject` (org domain) | N/A |
| `relyOnDMARC` Result | `false` | **`true`** | `false` |
| SPF Enforcement | **Normal (quarantine)** | **Cleared** | **Normal (quarantine)** |
| "Deferring" Log Emitted | No | **Yes** | No |
| DMARC `EvaluateAlignment` | `ResultNone` (!dkimPresent) | `ResultNone` (!dkimPresent) | `ResultNone` (no record — not called) |
| DMARC Policy Applied | `PolicyNone` | `PolicyNone` | `PolicyNone` |
| SMTP Response | `250 2.0.0 OK: queued` | `250 2.0.0 OK: queued` | `250 2.0.0 OK: queued` |
| `MsgMeta.Quarantine` | `true` | **`false`** | `true` |
| Auth-Results: spf | `fail` | `fail` | `fail` |
| Auth-Results: dmarc | `none` | `none` | `none` |

**The paradox**: The domain with the strongest DMARC policy (`p=reject`) receives the weakest enforcement (none), while domains with weaker or no DMARC policies receive stronger enforcement (quarantine).

## Why Case B Is Accepted Despite SPF -all

### The Deferral Mechanism

The `relyOnDMARC` function (spf.go:212-235) is the entry point for the deferral decision:

1. It extracts the RFC5322 From domain via `maddydmarc.ExtractFromDomain(hdr)` (spf.go:213).
2. It fetches the DMARC record via `maddydmarc.FetchRecord(ctx, dns.DefaultResolver(), fromDomain)` (spf.go:219).
3. It determines the effective policy — using `record.SubdomainPolicy` when `policyDomain != fromDomain` (spf.go:230-231), otherwise using `record.Policy` (spf.go:228).
4. If the effective policy is **not** `none`, it returns `true`, signaling that SPF enforcement should defer to DMARC (spf.go:234).

For `maddy-dmarc.com`, the record is found directly at `_dmarc.maddy-dmarc.com` with `p=reject`. Since `policyDomain == fromDomain` (both `maddy-dmarc.com`), the subdomain branch at spf.go:230 is not taken. Policy remains `reject` → `relyOnDMARC` returns `true`.

The intent is documented in the comment at spf.go:298-301: if a DMARC policy exists, the SPF check should not independently reject or quarantine — instead, DMARC alignment evaluation should make the final call.

### The Clearing of SPF Enforcement

When `relyOnDMARC` returns `true`, `CheckBody` (spf.go:358-361) computes the SPF result and then **explicitly clears** both enforcement flags:

```go
checkRes := s.spfResult(res.res, res.err)
checkRes.Quarantine = false
checkRes.Reject = false
return checkRes
```

Only the `AuthResult` (the SPF authentication result for the Authentication-Results header) is preserved. The `Reason` field (the SMTPError) is also preserved, but with both `Quarantine` and `Reject` set to `false`, it carries no enforcement weight. This means the SPF check passes through only informational data, not enforcement action.

### The DMARC Evaluation Gap

This is the root cause of the counterintuitive behavior:

1. **`applyResults`** (check_runner.go:267-268) calls `cr.dmarcVerify.Apply(cr.mergedRes.AuthResult)`.

2. **`Apply`** (verifier.go:111-156) receives the fetched DMARC record and the merged auth results.

3. **`EvaluateAlignment`** (evaluate.go:99-184) is called. It iterates over `authRes` looking for `*authres.DKIMResult` and `*authres.SPFResult` entries.

4. **The `!dkimPresent` guard** (evaluate.go:148): Since the pipeline has no `verify_dkim` module, no `DKIMResult` was ever added to `AuthResult`. Therefore `dkimPresent = false`. The function returns immediately with:

   ```go
   res.Authres = authres.DMARCResult{
       Value:  authres.ResultNone,
       Reason: "Not enough information (required checks are disabled)",
       From:   fromDomain,
   }
   ```

5. **Back in `Apply`** (verifier.go:142): The condition `result.Authres.Value == authres.ResultPass || result.Authres.Value == authres.ResultNone` evaluates to `true` because `ResultNone` matches. It returns `result, dmarc.PolicyNone`.

6. **Back in `applyResults`** (check_runner.go:270): `policy == PolicyNone` → the switch at check_runner.go:270-296 does not match `PolicyReject` or `PolicyQuarantine`. **No DMARC rejection or quarantine is applied.**

The gap: `apply_spf` deferred enforcement to DMARC, but DMARC cannot evaluate alignment without both SPF and DKIM results. The `!dkimPresent` guard is a safety check that avoids making DMARC decisions with incomplete data, but it creates an enforcement vacuum when `verify_dkim` is absent from the pipeline.

### Why Case A Does NOT Defer

The subdomain policy logic in `relyOnDMARC` (spf.go:228-232) determines Case A's behavior:

1. For `sub.maddy-dmarc.com`, `FetchRecord` (evaluate.go:25-51) first queries `_dmarc.sub.maddy-dmarc.com` — no record found. It then falls back to the organizational domain `maddy-dmarc.com` via `publicsuffix.EffectiveTLDPlusOne` (evaluate.go:34).

2. The DMARC record found is `v=DMARC1; p=reject; sp=none`.

3. In `relyOnDMARC` (spf.go:230-231): since `policyDomain` (`maddy-dmarc.com`) ≠ `fromDomain` (`sub.maddy-dmarc.com`) and `record.SubdomainPolicy != ""`, the effective policy becomes `SubdomainPolicy` = `none`.

4. `none == PolicyNone` → `relyOnDMARC` returns `false` (spf.go:234).

5. Because `relyOnDMARC` is false, SPF enforcement is **not** cleared. The default `failAction = {Quarantine: true}` applies normally (spf.go:364 → spf.go:155-165 → action.go:96).

**Key insight**: The `sp=none` tag in the organizational DMARC record causes subdomains to be treated as if there is no actionable DMARC policy, which means SPF enforcement stays active for subdomains.

### The Decision-Making Component

**`relyOnDMARC`** in `internal/check/spf/spf.go` (lines 212-235) is the runtime component that makes the deferral decision. It is a method on the SPF check's per-message `state` object, called during `CheckBody` (line 351). It determines whether the SPF check should defer its enforcement action to the DMARC evaluation pipeline.

The function's decision is binary:

- `true` → Clear SPF enforcement, rely on DMARC (which may fail to enforce due to the evaluation gap)
- `false` → Apply SPF enforcement normally (quarantine or reject per configuration)

## How to Verify Using Only Observable Signals

### SMTP Responses

All three cases produce `250 2.0.0 OK: queued` on the wire, so SMTP response codes alone **cannot** distinguish between them. The SMTP response only differentiates accepted messages from rejected ones — it cannot reveal quarantine status.

If the SPF check were configured with `fail_action reject` instead of the default `quarantine`, Cases A and C would produce:

```
550 5.7.23 SPF authentication failed (msg ID = <id>)
```

while Case B would still produce `250 2.0.0 OK: queued`.

### Server Logs

The distinguishing log lines are:

1. **`"deferring action due to a DMARC policy"`** (spf.go:353): This log line appears **only** for Case B. It is the definitive signal that `relyOnDMARC` returned `true` and SPF enforcement was cleared. It uses `Printf` level (not debug), so it always appears in standard logs.

2. **`"quarantined"`** (check_runner.go:204): Appears for Cases A and C. Indicates SPF enforcement triggered quarantine. Absent for Case B.

3. **`"no check action"`** (check_runner.go:191): Appears **only** for Case B. Indicates the check result had a `Reason` (error) but both `Quarantine` and `Reject` were cleared. This is the "action ignore" case logged for deployment testing.

| Log Signal | Case A | Case B | Case C |
|---|---|---|---|
| `"deferring action due to a DMARC policy"` | ✗ | ✓ | ✗ |
| `"quarantined"` | ✓ | ✗ | ✓ |
| `"no check action"` | ✗ | ✓ | ✗ |

### Authentication-Results Header

All three cases produce `spf=fail` and `dmarc=none` in the Authentication-Results header, but the DMARC portion differs between Cases A/B and Case C due to different code paths producing the `ResultNone`.

**Cases A and B** — `EvaluateAlignment` is called, hits the `!dkimPresent` guard (evaluate.go:148), and returns a `DMARCResult` with `Reason: "Not enough information (required checks are disabled)"` (evaluate.go:151). The `authres` formatter includes this non-empty reason in the header:

```
<hostname>; spf=fail smtp.mailfrom=<domain>; dmarc=none reason="Not enough information (required checks are disabled)" header.from=<domain>
```

**Case C** — `Apply` (verifier.go:132-137) returns early because `data.record == nil` (no DMARC record found). The `DMARCResult` has `Value: ResultNone` and `From: "no-dmarc.com"`, but an empty `Reason` field. The `authres` formatter (authres/format.go:60) skips the `reason` key when the value is empty, so the header omits it:

```
<hostname>; spf=fail smtp.mailfrom=no-dmarc.com; dmarc=none header.from=no-dmarc.com
```

The `dmarc=none` result appears in all three cases, but the presence or absence of the `reason` clause is a distinguishing observable signal: it reveals whether the DMARC verifier reached `EvaluateAlignment` (Cases A/B) or short-circuited on a nil record (Case C).

If `verify_dkim` were present in the pipeline, the Authentication-Results would differ significantly and DMARC would be able to make a real pass/fail decision.

### Quarantine Metadata

`MsgMeta.Quarantine` is set in `applyResults` (check_runner.go:263-264). This flag is propagated to the delivery target:

- **Cases A and C**: `MsgMeta.Quarantine = true` — message routed to junk/quarantine.
- **Case B**: `MsgMeta.Quarantine = false` — message delivered normally.

This flag is observable through:

1. The delivery target's behavior (e.g., IMAP stores to Junk folder).
2. The `"quarantined"` log line (present for A/C, absent for B).
3. Testing with `testutils.Target` which captures `msg.MsgMeta.Quarantine`.

## Root Cause Summary

1. **The `relyOnDMARC` deferral**: When a non-none DMARC policy exists for the From domain, `apply_spf` clears its enforcement action and relies on DMARC evaluation. This is by design — it prevents SPF and DMARC from double-punishing messages (spf.go:298-301 comment).

2. **The `!dkimPresent` evaluation gap**: `EvaluateAlignment` (evaluate.go:148) returns `ResultNone` when no DKIM results are present. This is a correctness guard — DMARC alignment requires both SPF and DKIM data. Without DKIM, the result is indeterminate (`ResultNone`), not a failure.

3. **The `ResultNone → PolicyNone` bypass**: In `Apply` (verifier.go:142), `ResultNone` is treated the same as `ResultPass` — it maps to `PolicyNone` (no enforcement). This means indeterminate results are treated as "no policy to apply," which is the safe default but creates the enforcement gap.

**Net effect**: For Case B, the SPF check defers to DMARC, and DMARC defers back to "no action" because it lacks DKIM data. The message passes through with no enforcement from either system. Adding `verify_dkim` to the pipeline would close this gap because DMARC would then have enough data to evaluate alignment and apply the `p=reject` policy.

## Test Reproduction Details

1. **Test framework**: Go `testing` package with `go-mockdns` for DNS interception.

2. **Pipeline construction**: `MsgPipeline` with `msgpipelineCfg{doDMARC: true, globalChecks: [apply_spf]}`.

3. **DNS setup**: `mockdns.Resolver{Zones: ...}` and `mockdns.PatchNet` to intercept `net.DefaultResolver` for SPF checks.

4. **Message delivery**: `doTestDelivery` pattern from existing `dmarc_test.go` (lines 22-62).

5. **Observable outputs captured**:
   - SMTP response (error or nil from delivery)
   - `MsgMeta.Quarantine` flag
   - `Authentication-Results` header
   - Server log output via `testutils.Logger`

6. **Commands executed**:
   - `go test -v -run TestSPF_DMARC ./internal/msgpipeline/` — Pipeline-level reproduction
   - `go test -v -run TestSMTP_RealSPF_DMARC ./internal/endpoint/smtp/` — SMTP endpoint wire-level reproduction
   - All existing tests verified passing: `go test ./internal/dmarc/ ./internal/msgpipeline/`

7. **Temporary test files** (not tracked):
   - `internal/msgpipeline/spf_dmarc_repro_test.go` — Pipeline-level 3-case reproduction
   - `internal/msgpipeline/spf_dmarc_nodkim_test.go` — With/without DKIM comparison
   - `internal/endpoint/smtp/spf_dmarc_smtp_test.go` — SMTP wire-level test
   - `internal/endpoint/smtp/spf_dmarc_real_test.go` — SMTP test with actual `apply_spf`

## Source Files Examined

| File | Key Elements | Lines Referenced |
|---|---|---|
| `internal/check/spf/spf.go` | `Check` struct, `relyOnDMARC`, `CheckBody`, `spfResult`, default `failAction` | 70-73, 155-165, 212-235, 298-301, 330-365 |
| `internal/dmarc/evaluate.go` | `FetchRecord`, `EvaluateAlignment`, `!dkimPresent` guard | 21-68, 99-184, 148-154 |
| `internal/dmarc/verifier.go` | `Verifier`, `Apply`, `ResultNone → PolicyNone` bypass | 48-53, 66-95, 111-156, 142-143 |
| `internal/dmarc/dmarc.go` | Type aliases: `Record`, `Policy`, `PolicyNone`, `Resolver` | Full file |
| `internal/msgpipeline/check_runner.go` | `runAndMergeResults`, `applyResults`, `checkBody`, `doDMARC` | 142-209, 245-259, 262-309 |
| `internal/msgpipeline/config.go` | `msgpipelineCfg`, `doDMARC` parsing | Full file |
| `internal/msgpipeline/msgpipeline.go` | `MsgPipeline`, `Mock` function | Full file |
| `internal/endpoint/smtp/smtp.go` | `Session.Data`, `wrapErr` | 312-344, 389-450 |
| `internal/check/action.go` | `FailAction`, `Apply` method | 34-39, 81-98 |
| `internal/msgpipeline/dmarc_test.go` | Existing DMARC pipeline tests | 22-62, 86-205 |
| `internal/dmarc/verifier_test.go` | Existing verifier unit tests | Full file |
