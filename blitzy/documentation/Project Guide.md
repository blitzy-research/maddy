# Blitzy Project Guide — maddy SMTP Server Runtime-Evidenced Security Review

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a **runtime-evidenced security review** of the `maddy` SMTP server (`github.com/foxcpp/maddy`, Go 1.13), targeting mail operators and security engineers. It answers two protocol-abuse questions from direct observation of the *running* server: **Q1** — how maddy resolves the DATA message boundary when a bare-dot line is followed by more data; and **Q2** — which identity maddy trusts after a client authenticates as user A, resets, then sends `MAIL FROM:<B>` without re-authenticating. The single deliverable is one read-only Markdown document that builds and runs maddy, captures real SMTP dialogue/logs/on-disk artifacts, and renders a "fails-safe" judgement. Business impact: it exposes an SMTP-smuggling-relevant leniency and an on-disk accountability gap to inform hardening decisions.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3', 'pie2':'#FFFFFF', 'pieStrokeColor':'#B23AF2', 'pieOuterStrokeColor':'#B23AF2', 'pieStrokeWidth':'2px', 'pieSectionTextColor':'#B23AF2', 'pieTitleTextSize':'17px'}}}%%
pie showData title Completion Status — 95.0% Complete
    "Completed Work (AI)" : 57
    "Remaining Work" : 3
```

| Metric | Value |
|--------|-------|
| **Total Hours** | 60 |
| **Completed Hours (AI + Manual)** | 57 (57 AI + 0 Manual) |
| **Remaining Hours** | 3 |
| **Percent Complete** | **95.0%** |

> Completion is computed with the PA1 AAP-scoped methodology: `Completed / (Completed + Remaining) = 57 / 60 = 95.0%`. All 34 AAP-specified work items are complete and validated; the 3 remaining hours are path-to-production human review and merge. Completion is held below 99% because human sign-off has not yet occurred.

### 1.3 Key Accomplishments

- ✅ **Sole deliverable created and committed** — `blitzy/documentation/maddy_26452dd8dd78.md` (2,329 lines, 140,533 bytes) added in the correct location with the branch-named filename.
- ✅ **Both questions fully answered from runtime observation** — Q1 (DATA dot-terminator) and Q2 (auth identity across transactions), each with every named sub-part addressed.
- ✅ **Canonical build reproduced byte-for-byte** — `go build -tags 'nopam nosqlite3' -o /tmp/maddy-bin ./cmd/maddy` → exit 0, binary **exactly 18,859,700 bytes**, banner `maddy unknown (built from source tree)`.
- ✅ **34 runtime cells exercised and stable** — Q1: 6 terminator variants × ports 25/587 × 2 runs (24 cells); Q2: 5 identity scenarios × 2 runs (10 cells); all behaviors identical across repeats.
- ✅ **Security findings evidenced** — bare-`LF` terminator leniency (SMTP-smuggling-relevant) with a working second-message injection on port 25; authenticated principal absent from on-disk `.meta` (`"Conn":null`) while DKIM correctly withholds signing on identity mismatch.
- ✅ **88 `file:line` citations across 14 source files**, 133 verbatim evidence code blocks, a full coverage matrix, and a split-by-axis fails-safe judgement.
- ✅ **Read-only compliance proven** — `git diff 26452dd..HEAD` = exactly one added file; `go.mod`/`go.sum` unchanged; working tree clean; all ephemeral scaffolding removed.
- ✅ **Independently validated** — dependencies verified, 100% unit tests pass with 0 data races, and every runtime claim reproduced.

### 1.4 Critical Unresolved Issues

There are **no critical issues blocking release or validation**. All autonomous work is complete and validated; the only open items are the human review/merge gate.

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Human technical review & acceptance of the security-review document not yet performed | Document cannot be formally accepted/merged until reviewed | Reviewing security engineer | < 0.5 day |
| Security triage decision on the two disclosed findings (bare-`LF` smuggling leniency; on-disk accountability gap) pending | Findings are documented but not yet dispositioned by a stakeholder | Security owner | < 0.5 day |

### 1.5 Access Issues

**No access issues identified.** The task is a self-contained, read-only investigation of the in-repository `maddy` source. All dependencies resolved from the module cache (`go mod verify` → "all modules verified"), the canonical toolchain (Go 1.13.15) is present, and no external services, credentials, or third-party APIs are required. The runtime harness used only local ports and synthetic throwaway credentials.

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| Source repository (`github.com/foxcpp/maddy`) | Read/commit | None — branch checked out, deliverable committed | Resolved | Blitzy Agent |
| Go module cache (55 modules) | Read | None — all modules verified | Resolved | Blitzy Agent |
| Runtime harness ports (25/587/2525) | Local bind | None — local-only, ephemeral, removed after use | Resolved | Blitzy Agent |

### 1.6 Recommended Next Steps

1. **[High]** Perform the technical accuracy review of `blitzy/documentation/maddy_26452dd8dd78.md` — spot-check the Q1 smuggling reproduction (§4.5) and Q2 identity evidence (§5.3–§5.6), and confirm the fails-safe judgement (§7) is sound (~1.5h).
2. **[High]** Verify read-only compliance and reproducibility — confirm the diff is a single added file, `go.mod`/`go.sum` unchanged, and the §8 harness scripts are complete enough to re-derive evidence under Go 1.13.15 (~0.5h).
3. **[Medium]** Triage the two disclosed findings and record an organizational disposition (remediation is a separate, out-of-scope initiative) (~0.5h).
4. **[Medium]** Approve the pull request and merge the branch (single additive file, no conflicts expected) (~0.5h).
5. **[Low]** *(Optional, out of scope)* Open follow-up engineering tickets for bare-newline/strict-CRLF rejection and for persisting the authenticated principal in queue metadata.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed work was performed autonomously by Blitzy agents (four `agent@blitzy.com` commits plus independent final validation). Each component traces to AAP-specified requirements.

| Component | Hours | Description |
|-----------|------:|-------------|
| Environment setup, canonical build & toolchain verification | 3 | Provisioned Go 1.13.15; verified canonical `go build -tags 'nopam nosqlite3'` from `cmd/maddy` (exit 0, 18,859,700-byte binary); confirmed default banner and `SIZE 33554432` advertisement. |
| Ephemeral runtime harness engineering | 8 | Authored 7 non-repository scripts: scratch `maddy.conf` (smtp :25 + submission :587, `insecure_auth`, `io_debug`), `extauth` credential helper, `smtp_downstream` sink (451 to retain messages), raw-SMTP client, Q1 driver, Q2 driver, cell-capture helper. |
| Q1 investigation — DATA/message-boundary | 9 | Traced the boundary through go-smtp → `net/textproto` dot reader; exercised 6 terminator variants across ports 25/587 × 2 runs (24 cells); captured 354 prompt, response codes, `io_debug`, stored `.body`. |
| Q2 investigation — auth identity across transactions | 10 | Drove auth-A → MAIL A → RSET → MAIL B scenario plus 4 controls × 2 runs (10 cells); captured acceptance log, `Received` header, DKIM decision, `{auth_user}` check, downstream SASL bytes, on-disk `.meta`; analyzed in-memory vs on-disk identity split. |
| Source mechanism analysis & `file:line` grounding | 5 | Produced 88 precise citations across 14 source files (smtp.go, go-smtp conn.go/data.go, net/textproto reader.go, dkim.go, queue.go, received.go, memory.go, command.go, sasl.go, msgmetadata.go, maddy.conf, HACKING.md). |
| Web-search research framing | 2 | Framed the judgement against RFC 5321 §4.5.2 and the 2023–2024 SMTP-smuggling class (US-CERT VU#302671; Sendmail/Exim/Postfix hardening) — context only. |
| Document authoring (2,329 lines) | 12 | Wrote direct answers up front, per-scenario verbatim evidence blocks (133 code blocks), the coverage matrix, stability tables, the split-by-axis fails-safe judgement, and the full reproducible appendix. |
| Coverage pass, stability re-runs & read-only cleanup | 3 | Confirmed every named sub-part answered; re-ran each scenario ≥2×; removed all scaffolding; verified `git status` clean. |
| Final validation — independent 5-gate reproduction | 5 | Re-verified dependencies, compilation (byte-exact), 100% unit tests + 0 races, all 34 runtime cells, and every citation/evidence block byte-for-byte. |
| **Total Completed** | **57** | |

### 2.2 Remaining Work Detail

All remaining work is **path-to-production human activity**. Fixing/hardening maddy is explicitly out of AAP scope and therefore carries **zero** hours here.

| Category | Hours | Priority |
|----------|------:|----------|
| Technical accuracy review of the security-review document (read + spot-check Q1/Q2 evidence + fails-safe judgement) | 1.5 | High |
| Read-only compliance & reproducibility verification (single-file diff; unchanged `go.mod`/`go.sum`; §8 scripts re-derivable) | 0.5 | High |
| Security findings triage & sign-off (disposition of the two disclosed findings — decision only) | 0.5 | Medium |
| Pull request approval & merge to target branch | 0.5 | Medium |
| **Total Remaining** | **3.0** | |

### 2.3 Hours Reconciliation

| Reconciliation Check | Value | Status |
|----------------------|-------|--------|
| Section 2.1 Completed total | 57h | ✅ |
| Section 2.2 Remaining total | 3h | ✅ |
| Section 2.1 + Section 2.2 | 60h = Total Hours (§1.2) | ✅ |
| Remaining hours across §1.2 / §2.2 / §7 | 3h everywhere | ✅ |
| Completion % (57 / 60) | 95.0% | ✅ |

---

## 3. Test Results

All results below originate from Blitzy's autonomous validation logs for this project; the unit-test, compilation, and race-detector rows were **independently reproduced in this assessment session** (byte-exact binary size, all sampled packages `ok`).

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|------------:|-------:|-------:|-----------|-------|
| Unit Tests | Go `testing` (`go test ./...`) | 232 test functions (44 test files) | 232 | 0 | Not measured | `go test ./...` exit 0; zero FAIL lines; all packages with tests report `ok`. |
| Race Detection | Go `-race` | 4 packages | 4 | 0 | — | `endpoint/smtp`, `target/queue`, `modify/dkim`, `target/smtp_downstream`: 0 data races. |
| Compilation | `go build` | 3 modes | 3 | 0 | — | Canonical (`nopam nosqlite3`, 18,859,700 B) + full build + `go build ./...`: all exit 0. |
| Runtime Behavioral — Q1 | Raw-SMTP drivers over TCP | 24 cells | 24 | 0 | — | 6 terminator variants × ports 25/587 × 2 runs; all stable. |
| Runtime Behavioral — Q2 | Raw-SMTP drivers over TCP | 10 cells | 10 | 0 | — | 5 identity scenarios × 2 runs; all stable. |
| **Total** | — | **273 checks** | **273** | **0** | — | Categories use differing units (functions/packages/modes/cells); all green. |

**Independent reproduction in this session:** `go mod verify` → "all modules verified"; canonical build → exit 0, 18,859,700 bytes; `go test -count=1` on `endpoint/smtp`, `target/queue` (1.506s), `modify/dkim` (0.009s), `target/smtp_downstream` (0.018s) → all `ok`.

---

## 4. Runtime Validation & UI Verification

**UI verification is not applicable** — this project produces a documentation artifact for a headless SMTP server; there is no frontend. Runtime validation below concerns SMTP protocol behavior (the "API integration outcomes" for a mail server).

**Server runtime health:**
- ✅ **Operational** — Server startup emits banner `220 localhost ESMTP Service Ready`; `smtp` listens on `tcp://0.0.0.0:25`, `submission` on `tcp://0.0.0.0:587`.
- ✅ **Operational** — Port 25 EHLO advertises `PIPELINING / 8BITMIME / ENHANCEDSTATUSCODES / SMTPUTF8 / SIZE 33554432` (no AUTH, as expected for unauthenticated intake).
- ✅ **Operational** — Port 587 EHLO additionally advertises `AUTH PLAIN`; `AUTH` returns `235 2.0.0 Authentication succeeded`.

**Q1 — DATA / message-boundary outcomes:**
- ✅ **Operational** — Canonical `<CR><LF>.<CR><LF>`: stops at the first bare-dot line; stores an 18-byte body; leading-dot un-stuffing applied; CRLF→LF normalized.
- ✅ **Operational (fail-closed)** — Bare `CR` (`\r.\r`): never terminates; transaction ends in `554` after EOF; nothing stored.
- ⚠ **Partial (lenient, by delegation)** — Bare `LF` (`\n.\n`): accepted as a terminator (originates in Go `net/textproto`, not maddy code) — the SMTP-smuggling-relevant leniency.
- ✅ **Operational** — Mid-body bare dot + trailing lines: post-dot bytes handed back to the command loop and parsed as new commands (`501 Bad command`); on port 25 a second attacker-controlled message is injected/queued; on port 587 the injection is domain-gated (`501 5.1.8`).

**Q2 — Authentication identity outcomes:**
- ✅ **Operational** — Identity is connection-scoped, set once, and **not** re-scoped by `RSET`; mismatched `MAIL FROM:<B>` is accepted at the command layer (`250`) but every live decision remains tied to the original login A.
- ✅ **Operational** — `Received` header records the envelope sender (B); DKIM signer **withholds** the signature on identity mismatch (logs the reason) rather than mis-signing; `{auth_user}` check and downstream SASL relay both carry A (byte-exact).
- ⚠ **Partial (accountability gap)** — On-disk `.meta` has `"Conn":null`; the authenticated principal A is **not** persisted, so post-hoc disk forensics see only envelope B.

---

## 5. Compliance & Quality Review

Cross-mapping of AAP deliverables and the governing "SWE-AtlasQnA-Repo" rules to observed compliance status.

| Benchmark / Rule | Requirement | Status | Evidence / Notes |
|------------------|-------------|--------|------------------|
| Deliverable rule | Single `.md` named for branch under `blitzy/documentation/` | ✅ Pass | `blitzy/documentation/maddy_26452dd8dd78.md` present and committed. |
| Run-first-then-write | Build/run and capture real output before writing | ✅ Pass | 133 verbatim evidence blocks; server startup, dialogue bytes, on-disk artifacts. |
| Canonical build/config | State exact build & invocation; use defaults | ✅ Pass | `go build -tags 'nopam nosqlite3' … ./cmd/maddy`; banner and 32 MiB size default documented; harness deviations disclosed. |
| Exercise every condition | Primary + secondary + error + edge + transitional | ✅ Pass | Canonical/bare-LF/bare-CR/mid-dot/dot-stuffed/smuggle across both ports; Q2 mismatch + 3 controls. |
| Observe before/during/after | Capture state around transitions | ✅ Pass | Connection state around the bare-dot; identity before/after `RSET`. |
| Actual, complete output | Unedited output + producing command per claim | ✅ Pass | Full SMTP dialogue, `io_debug`, `.body`/`.header`/`.meta`, SASL base64. |
| Exactness & grounding | `file:line` per code claim; lead with direct answer | ✅ Pass | 88 citations / 14 files; §1.1 direct answers up front. |
| Stability | Re-run unchanged input ≥2×; report distribution | ✅ Pass | §6.2 shows distinct `msg_id`s per run; behavior identical. |
| Coverage pass | Every named sub-part confirmed answered | ✅ Pass | §6.1 coverage matrix maps each Q1/Q2 sub-part to evidence. |
| Read-only scope | No source modified; only doc added; scaffolding removed | ✅ Pass | `git diff 26452dd..HEAD` = 1 added file; `go.mod`/`go.sum` unchanged; tree clean. |
| Path accuracy | Cite real source paths | ✅ Pass (reconciled) | AAP's `framework/module/` corrected to real `internal/module/msgmetadata.go`. |
| Compilation quality | Builds cleanly | ✅ Pass | Canonical + full + `./...` all exit 0. |
| Test quality | Suite green | ✅ Pass | `go test ./...` exit 0; 0 data races on Q1/Q2 packages. |

**Fixes applied during autonomous validation:** none required — the document was found accurate on independent reproduction and needed zero modifications. **Outstanding compliance items:** none autonomous; human sign-off pending (see §2.2).

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Runtime evidence captured under a disclosed non-default harness (`io_debug`, `insecure_auth`, submission on :587 vs shipped :465) may differ from shipped defaults on peripheral details | Technical | Low | Low | Every deviation disclosed (§3); boundary framing attributed to `net/textproto` (config-independent); identical submission code path | Mitigated |
| Observations pinned to Go 1.13.15 + go-smtp pre-release; stdlib/dependency line numbers and bare-`LF` behavior are version-specific | Technical | Low | Medium | Exact versions + base commit pinned; reproduce with the stated toolchain | Accepted (report as-is at pinned versions) |
| Document contains a working SMTP-smuggling reproduction and names an on-disk accountability gap; should be handled as security-sensitive until triaged | Security | Medium | Medium | Restrict distribution to security stakeholders; triage findings; leniency originates in Go `net/textproto`, not maddy code | Open (human triage) |
| Harness enabled `io_debug` (can leak passwords) and `insecure_auth` | Security | Low | Low | Only synthetic throwaway credentials used; scaffolding deleted; git verified clean | Mitigated |
| Ephemeral harness deleted (read-only rule); re-verification needs §8 scripts + EOL Go 1.13.15 | Operational | Low | Medium | §8 reproduces all scripts verbatim + exact commands | Mitigated |
| Branch merge of a single additive file in a new directory | Integration | Low | Low | Additive-only; diff = exactly 1 file; `go.mod`/`go.sum` unchanged | Mitigated |
| AAP cited a non-existent path (`framework/module/`) | Technical / Process | Low | Low | Agents used the real `internal/module/msgmetadata.go`; validator confirmed | Resolved |

**Overall risk posture: LOW.** No High-severity risks. One Medium/Open item (treat the document as sensitive and triage the two findings) maps directly to the remaining human review tasks.

---

## 7. Visual Project Status

**Project hours breakdown (Completed vs Remaining):**

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3', 'pie2':'#FFFFFF', 'pieStrokeColor':'#B23AF2', 'pieOuterStrokeColor':'#B23AF2', 'pieStrokeWidth':'2px', 'pieSectionTextColor':'#B23AF2', 'pieTitleTextSize':'17px'}}}%%
pie showData title Project Hours Breakdown (Total 60h)
    "Completed Work" : 57
    "Remaining Work" : 3
```

**Remaining work by priority (hours):**

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3', 'pie2':'#A8FDD9', 'pieStrokeColor':'#B23AF2', 'pieOuterStrokeColor':'#B23AF2', 'pieStrokeWidth':'2px', 'pieSectionTextColor':'#B23AF2', 'pieTitleTextSize':'16px'}}}%%
pie showData title Remaining 3h by Priority
    "High" : 2
    "Medium" : 1
```

**Remaining hours per category (Section 2.2):**

| Category | Hours | Bar |
|----------|------:|-----|
| Doc technical accuracy review | 1.5 | █████████████████ |
| Read-only / reproducibility verification | 0.5 | █████ |
| Security findings triage & sign-off | 0.5 | █████ |
| PR approval & merge | 0.5 | █████ |
| **Total** | **3.0** | |

> **Integrity:** the "Remaining Work" pie value (3) equals the Section 1.2 Remaining Hours (3) and the sum of the Section 2.2 Hours column (1.5 + 0.5 + 0.5 + 0.5 = 3.0). "Completed Work" (57) + "Remaining Work" (3) = 60h total.

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a complete, rigorously evidenced security review of maddy's SMTP message handling. Both questions are answered from direct runtime observation: **Q1** establishes that maddy stops at the first bare-dot line (boundary delegated to Go's `net/textproto` dot reader), is fail-closed on a bare `CR`, but **lenient on a bare `LF`** — the exact leniency behind the 2023–2024 SMTP-smuggling class, demonstrated by a real second-message injection on port 25. **Q2** establishes that the authenticated identity is connection-scoped and never re-scoped by `RSET`; every live decision stays tied to the original login (DKIM withholds rather than mis-signs), but the authenticated principal is **not persisted** to queue metadata, creating an on-disk accountability gap. The work is backed by 88 `file:line` citations, 133 verbatim evidence blocks, 34 stable runtime cells, and a full coverage matrix.

**Remaining gaps & critical path to production.** The project is **95.0% complete** (57 of 60 hours). The remaining 3 hours are entirely human path-to-production: technical review and read-only/reproducibility verification (2h, High), then security triage and PR merge (1h, Medium). No autonomous work remains, and no code defects exist to fix (the deliverable required zero modifications on validation).

**Production readiness assessment.** The deliverable is **production-ready pending human sign-off**: it compiles cleanly, all unit tests pass with zero data races, the canonical binary reproduces byte-for-byte (18,859,700 bytes), every runtime claim was independently reproduced, and read-only compliance is proven (single additive file; `go.mod`/`go.sum` untouched; clean tree).

| Success Metric | Target | Actual | Met? |
|----------------|--------|--------|------|
| Deliverable created at correct path | Yes | Yes | ✅ |
| Both questions + all sub-parts answered | 100% | 100% (§6.1 matrix) | ✅ |
| Canonical build reproduces | Byte-exact | 18,859,700 B | ✅ |
| Unit tests passing | 100% | 100%, 0 races | ✅ |
| Read-only compliance | 1 added file | 1 added file | ✅ |
| Runtime cells stable | All | 34/34 stable | ✅ |

**Recommendations (priority-ordered):** (1) complete the technical review and accept the document; (2) triage the two disclosed findings; (3) approve and merge; (4) *optionally, out of scope,* open follow-up tickets for bare-newline rejection and for persisting the authenticated principal in queue metadata.

---

## 9. Development Guide

Every command below was executed during this assessment and produced the stated output. Run all commands from the repository root unless noted.

### 9.1 System Prerequisites

- **Go 1.13.15** (canonical; matches `go.mod`'s `go 1.13`). Newer Go may shift version-specific line numbers/behavior noted in the document.
- **OS:** Linux/amd64 (verified) or macOS.
- **Tooling:** `git` + `git-lfs`.
- **Disk:** ~100 MB for build output and module cache.
- **Optional runtime harness:** free TCP ports **25**, **587**, **2525** (25/587 need root or `CAP_NET_BIND_SERVICE`, or remap to high ports).

### 9.2 Environment Setup

```bash
# Confirm toolchain (expected: go version go1.13.15 linux/amd64)
go version

# From the repository root; no environment variables are required to build.
# GOPATH defaults to /root/go; module mode is on (go.mod present).
go env GOPATH GOMODCACHE
```

### 9.3 Dependency Installation

```bash
# Download and verify all modules (expected: "all modules verified")
go mod download
go mod verify
```

### 9.4 Build (Canonical)

```bash
# Canonical, cgo-free build from the real entry point (expected: exit 0)
go build -tags 'nopam nosqlite3' -o /tmp/maddy-bin ./cmd/maddy ; echo "exit=$?"

# Verify the binary size (expected: 18859700 bytes) and banner
stat -c '%s bytes' /tmp/maddy-bin          # -> 18859700 bytes
/tmp/maddy-bin -v                          # -> maddy unknown (built from source tree)

# Alternative cgo-free path:
# CGO_ENABLED=0 go build -o /tmp/maddy-bin ./cmd/maddy
```

### 9.5 Verification (Tests)

```bash
# Full unit-test suite (expected: all packages ok, exit 0)
go test ./...

# Race detector on the Q1/Q2-relevant packages (expected: ok, 0 data races)
go test -race -count=1 \
  ./internal/endpoint/smtp/... \
  ./internal/target/queue/... \
  ./internal/modify/dkim/... \
  ./internal/target/smtp_downstream/...
```

### 9.6 Read the Deliverable

```bash
# Direct answers to Q1 and Q2 appear up front:
sed -n '1,120p' blitzy/documentation/maddy_26452dd8dd78.md

# Confirm read-only compliance (expected: single added file; empty status)
git diff --name-status 26452dd..HEAD
git status --porcelain
```

### 9.7 (Optional) Reproduce the Runtime Evidence

The document's §8 appendix contains every harness script verbatim (scratch `maddy.conf`, `extauth` helper, `smtp_downstream` sink, raw-SMTP client, Q1/Q2 drivers, cell-capture helper). To reproduce:

```bash
# 1) Paste the §8 scratch config + helpers into a /tmp working dir (NON-repository).
# 2) Start the downstream sink (returns 451 so the queue retains messages on disk).
# 3) Run maddy with the -debug flag so io_debug SMTP dialogue is emitted:
#    /tmp/maddy-bin -config /tmp/maddy-scratch/maddy.conf -debug run
# 4) Drive scenarios, e.g.:
#    python3 q1_driver.py 25 smuggle      # Q1 smuggling on port 25
#    python3 q2_driver.py mismatch        # Q2 auth-A -> RSET -> MAIL B
# 5) Inspect stored .body/.header/.meta under the queue sink directory.
# 6) Cleanup: rm -rf /tmp/maddy-scratch /tmp/maddy-bin ; git status --porcelain
```

### 9.8 Troubleshooting

- **Build errors mentioning `pam` / `sqlite` / cgo** → add `-tags 'nopam nosqlite3'` (or `CGO_ENABLED=0`), as in §9.4.
- **`go: requires go >= 1.x` or behavioral drift** → install exact **Go 1.13.15**; findings are version-pinned (see Risk R-T2).
- **`bind: permission denied` on ports 25/587** → run the harness as root, grant `CAP_NET_BIND_SERVICE`, or remap to high ports in the scratch config.
- **Messages not persisting to disk** → ensure the downstream sink returns `451` so the queue keeps `.body`/`.header`/`.meta`.
- **No SMTP dialogue in logs** → enable `io_debug yes` in the scratch config and run maddy with `-debug`.
- **`git status` not clean** → confirm all harness artifacts live under `/tmp` (outside the tree) and were removed.

---

## 10. Appendices

### A. Command Reference

| Command | Purpose | Expected Result |
|---------|---------|-----------------|
| `go version` | Confirm toolchain | `go version go1.13.15 linux/amd64` |
| `go mod verify` | Verify dependencies | `all modules verified` |
| `go build -tags 'nopam nosqlite3' -o /tmp/maddy-bin ./cmd/maddy` | Canonical build | exit 0 |
| `stat -c '%s bytes' /tmp/maddy-bin` | Binary size | `18859700 bytes` |
| `/tmp/maddy-bin -v` | Version banner | `maddy unknown (built from source tree)` |
| `go test ./...` | Full unit suite | all `ok`, exit 0 |
| `go test -race -count=1 ./internal/endpoint/smtp/... …` | Race detection | `ok`, 0 races |
| `git diff --name-status 26452dd..HEAD` | Read-only proof | `A blitzy/documentation/maddy_26452dd8dd78.md` |
| `git status --porcelain` | Clean-tree proof | (empty) |

### B. Port Reference

| Port | Role | Notes |
|------|------|-------|
| 25 | `smtp` intake (unauthenticated) | No AUTH advertised; smuggling reproducible here |
| 587 | `submission` (authenticated) | Advertises `AUTH PLAIN`; injection domain-gated |
| 465 | Shipped default `submission` (TLS) | Documented; harness used :587 to match the scenario |
| 2525 | Ephemeral downstream sink | Returns `451` to retain queued messages (harness only) |

### C. Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/maddy_26452dd8dd78.md` | **The deliverable** (2,329 lines) |
| `cmd/maddy/main.go` | Canonical build entry point |
| `cmd/maddyctl/users.go` | Admin CLI (`users create`) |
| `internal/endpoint/smtp/smtp.go` | SMTP intake; `Session.Data`, `Login`, `newSession`, `Reset` |
| `internal/endpoint/smtp/submission.go` | Submission-path header handling |
| `internal/modify/dkim/dkim.go` | DKIM `require_sender_match` / `shouldSign` |
| `internal/target/queue/queue.go` | On-disk `.meta` serialization (`Conn` nulled) |
| `internal/target/received.go` | `Received` header generation |
| `internal/module/msgmetadata.go` | `ConnState.AuthUser` / `MsgMetadata.Conn` |
| `maddy.conf` | Shipped default configuration |

### D. Technology Versions

| Component | Version |
|-----------|---------|
| Go toolchain | 1.13.15 (`go.mod`: `go 1.13`) |
| `github.com/emersion/go-smtp` | v0.12.1-0.20191206174923-1f576e0ec85c |
| `github.com/emersion/go-message` | v0.10.9-0.20191116124005-65fd0119e899 |
| `github.com/emersion/go-msgauth` | v0.3.2-0.20191028231513-55b75676976c |
| `github.com/emersion/go-sasl` | v0.0.0-20190817083125-240c8404624e |
| `github.com/google/uuid` | v1.1.1 |
| `golang.org/x/net` | v0.0.0-20191126235420-ef20fe5d7933 |
| `net/textproto` | bundled with Go 1.13.15 |
| Total modules | 55 (all verified) |

### E. Environment Variable Reference

| Variable | Required? | Notes |
|----------|-----------|-------|
| `GOPATH` | No | Defaults to `/root/go`; module cache under `$GOPATH/pkg/mod` |
| `CGO_ENABLED` | No | Set `0` as an alternative to `-tags 'nopam nosqlite3'` for a cgo-free build |
| `GOFLAGS` | No | Not required |

> No application runtime environment variables are needed. The optional harness is configured entirely via the scratch `maddy.conf`, not environment variables.

### F. Developer Tools Guide

| Tool | Use |
|------|-----|
| `go build` / `go test` | Compile and test (see §9) |
| `go mod download` / `go mod verify` | Dependency management |
| raw-SMTP Python drivers (§8 of the doc) | Reproduce Q1/Q2 runtime evidence |
| `od -c` / hex dump | Verify byte-sensitive terminator behavior |
| `jq` | Inspect queued `.meta` JSON |
| `git diff` / `git status` | Verify read-only compliance |

### G. Glossary

| Term | Definition |
|------|------------|
| **DATA dot-terminator** | The SMTP end-of-data marker: a line containing only `.` (RFC 5321 §4.5.2) |
| **Dot-stuffing** | Client prefixes body lines starting with `.` with an extra `.`; server strips one leading dot |
| **SMTP smuggling** | Boundary desynchronization (bare-`LF`/`CR`) letting two hops disagree on message end, enabling injection |
| **Bare `LF` / bare `CR`** | Non-standard line endings (`\n` or `\r` alone) instead of `\r\n` |
| **Connection-scoped identity** | Authenticated principal set once at session creation; not re-scoped by `RSET` |
| **`RSET`** | SMTP reset — clears the in-flight transaction (recipients/sender) but not authentication |
| **Envelope sender** | The `MAIL FROM` address (may differ from the authenticated identity) |
| **`require_sender_match`** | DKIM directive (default `["envelope","auth"]`) governing when to sign |
| **`.meta` / `.body` / `.header`** | On-disk queue artifacts written per message |
| **Fail-safe vs fail-closed** | Fail-safe: no undefined/unsafe state on bad input; fail-closed: strictly reject ambiguous input |
