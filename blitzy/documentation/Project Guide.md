# Blitzy Project Guide

**Project:** Maddy Module-System Startup — Developer-Onboarding Q&A Documentation
**Repository:** `github.com/foxcpp/maddy` (source branch `maddy_26452dd8dd78`, HEAD `26452dd`)
**Working branch:** `blitzy-25330606-79c5-4622-9c90-dc77a1a63ce9` (HEAD `1cc08de`)
**Governing rule set:** `SWE-AtlasQnA-Repo` (read-only investigation)

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, comprehensive Markdown document that explains — from direct runtime observation of the canonical Maddy build — how Maddy's module system assembles at server startup, especially in configurations that mix SMTP and IMAP endpoints sharing one backing instance. It answers five developer-onboarding questions (immediate registration vs. deferral; lazy initialization and the `&` reference syntax; endpoint-lifecycle divergence; check/modifier coordination; and the runtime evidence that the module graph has settled). Every behavioral claim is paired with the exact command run, the complete unedited output, and a `file:line` citation. The target audience is engineers onboarding onto Maddy's internals. The task is strictly read-only: the sole repository change is the new documentation file.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Completion Status — 92.5% Complete
    "Completed Work" : 37
    "Remaining Work" : 3
```

| Metric | Value |
|---|---|
| **Total Hours** | 40 |
| **Completed Hours (AI + Manual)** | 37 (37 AI + 0 Manual) |
| **Remaining Hours** | 3 |
| **Percent Complete** | **92.5%** |

Completion is computed with the PA1 AAP-scoped method: `Completed ÷ (Completed + Remaining) = 37 ÷ 40 = 92.5%`. All 18 AAP-specified deliverables are complete and independently validated; the remaining 3 hours are the path-to-production human-review-and-merge gate for a documentation deliverable.

### 1.3 Key Accomplishments

- ✅ Authored the sole in-scope deliverable `blitzy/documentation/maddy_26452dd8dd78.md` — 1,207 lines / 10,632 words answering all five sub-questions.
- ✅ Built the canonical binary from `./cmd/maddy` (cgo/SQLite) and ran it through its **real entry point** with `-debug`; no test doubles or hooks.
- ✅ Captured **8 runtime conditions** (mixed SMTP+IMAP, forward reference, orphan/unused-block, dangling `&`-ref, unknown inline module, driven SMTP transaction, `--network none` variant, systemd datagrams), each **reproduced across two runs**.
- ✅ Grounded every claim with **143 verified `file:line` citations** across ~20 source files and **58 provenance labels** (`[Observed at runtime]` / `[Observed from source]` / `[Inferred]`).
- ✅ Included a **Coverage pass** mapping all five questions and every named item, plus a Condition→command→output map, source-lineage note, and Observed-vs-Inferred summary.
- ✅ Maintained strict **read-only compliance**: zero source/test/config/build files modified; all temporary artifacts removed; working tree clean.
- ✅ Passed autonomous validation: compilation exit 0, **20/20 test packages pass**, all runtime output reproduced **byte-identical**, **zero edits required**.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None (no defects)._ No compilation, test, runtime, citation, or scope defect remains. | No release blockers. | — | — |
| Human SME review not yet performed (quality gate, not a defect) | Merge is gated on human sign-off per standard process | Maddy maintainer / reviewing engineer | 0.5 day |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| Maintainer Docker image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_foxcpp_maddy_1.0` | Container registry pull | Canonical build/run requires this image (Go 1.18.10 + warm OFFLINE module cache); there is **no native Go on the host** | Available — used successfully by autonomous validation; reviewer needs registry pull access to reproduce runtime conditions | Reviewing engineer / platform |
| Maddy source repository | Git write (merge) | Standard merge permission needed to land the PR | Pending — normal human merge step | Repository maintainer |

No blocking access issues were encountered during autonomous work (builds/tests/runs all completed offline in the provided image).

### 1.6 Recommended Next Steps

1. **[High]** Spot-check a representative sample (~15–20) of the 143 `file:line` citations against the checkout at rev `26452dd` to confirm quoted code matches source. *(≈1.0h)*
2. **[High]** Reproduce at least one runtime condition (Condition 1, `mixed.conf`) in the maintainer Docker image to confirm the single-`sql`-init/three-references signature and both endpoints reaching `listening`. *(≈1.0h)*
3. **[High]** Read the document end-to-end for clarity/completeness of onboarding value (five questions, Coverage pass, provenance labels). *(≈0.5h)*
4. **[Medium]** Approve and merge `blitzy/documentation/maddy_26452dd8dd78.md`; confirm `git status --porcelain` remains clean afterward. *(≈0.5h)*

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|---|---:|---|
| Build environment & canonical binary | 3 | Obtain/run the maintainer Docker image; `CGO_ENABLED=1 GOPROXY=off go build -mod=readonly` from `./cmd/maddy`; confirm version banner `maddy unknown (built from source tree)`; establish clean-tree baseline |
| Observation configuration authoring | 4 | Author 6 temporary configs outside the repo: mixed SMTP+IMAP sharing one `sql`, forward-reference, inline check+modify, orphan/unused-block, dangling `&`-ref, unknown inline module (correct `state`/`runtime`/`hostname`/`tls off` directives) |
| Runtime observation & capture | 5 | Run 8 conditions via the real `cmd/maddy -debug` entry point; capture complete unedited output; drive a real SMTP transaction (`smtplib`); capture systemd `READY`/`STOPPING` datagrams; the `--network none` variant |
| Reproducibility verification | 2 | Re-run each condition ≥2 times; normalize documented per-run tokens (scratch path, `msg_id`, `CheckState` pointer, ephemeral port); confirm byte-identical signals |
| Source investigation & citation harvesting | 6 | Read ~20 source files; trace the `Run → moduleMain → instancesFromConfig` call chain, `GetInstance`/`ModuleFromNode`/`check_runner`; identify and verify 143 `file:line` anchors |
| Answer-document authoring | 10 | Write 1,207 lines / 10,632 words: Build&environment, overview, five Question sections, Coverage pass, source-lineage note, Observed-vs-Inferred summary; 52 code blocks, 17 tables, 1 mermaid overview |
| Review-and-fix & QA cycles | 4 | Resolve 13 code-review findings (doc grew 438→1,207 lines) and 1 QA finding; correct two AAP off-by-one anchors (`sql.go` Init L176; `go-sqlite3` `go.mod:L26`) |
| Coverage pass & final citation verification | 2 | Confirm every sub-question and named item (`smtp`/`submission`/`lmtp`/`imap`, `check`/`modify`, `&`-refs, inline modules, before/after states, error paths) is answered; verify 143/143 citations |
| Cleanup & repository-integrity verification | 1 | Delete all temporary observation artifacts; confirm `git status --porcelain` empty and only one file added vs. base |
| **Total** | **37** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|---|---:|---|
| Human SME technical review (citation spot-check + reproduce ≥1 runtime condition + end-to-end readability/completeness) | 2.5 | High |
| PR approval & merge to target branch | 0.5 | Medium |
| **Total** | **3** | |

### 2.3 Basis of Estimate

Estimates derive from the observable evidence of the autonomous work: 3 agent commits (`fa1abd3` → `d48094a` → `1cc08de`) totaling +1,207/−0 lines through a genuine author→review→QA cycle; a 10,632-word, citation-dense deliverable; and 8 runtime conditions each reproduced twice. Confidence is **High** — scope is well-defined and fully validated (143/143 citations accurate, byte-identical reproduction, zero edits required), so **no rework hours** are carried. Remaining hours reflect only the standard human review-and-merge gate for a documentation deliverable.

---

## 3. Test Results

All results below originate from Blitzy's autonomous validation logs for this project (final-validation session, repository mounted read-only, executed in the maintainer Docker image).

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---|---|
| Unit / Package | Go `go test` | 20 packages | 20 | 0 | N/A¹ | `CGO_ENABLED=1 GOPROXY=off go test -mod=readonly ./...` → exit 0. Suite spans 232 `func Test…` across 44 `*_test.go` files in 20 packages; 26 packages have no test files (normal). Confirms the read-only doc change breaks nothing. |
| Compilation | Go build | 3 targets | 3 | 0 | N/A | `go build ./cmd/maddy`, `./cmd/maddyctl`, and `./...` all exit 0. Only diagnostic is the benign SQLite `-Wreturn-local-addr` amalgamation warning (expected, harmless). |
| Runtime Observation | `cmd/maddy` binary + Python `smtplib`/socket drivers | 8 conditions | 8 | 0 | N/A | Conditions C1, C2, C2b, C3, C4, C5, C6a, C6b — each reproduced across two runs, byte-identical after normalizing documented per-run tokens. |

¹ Coverage percentage was not part of the autonomous validation scope for this read-only documentation task (the AAP defines no coverage target and modifies no code), so it is honestly reported as **N/A** rather than fabricated. The pass/fail figures are the authoritative executed results.

---

## 4. Runtime Validation & UI Verification

**UI Verification:** **N/A** — Maddy is a headless SMTP/IMAP daemon with no graphical user interface, and the deliverable is a Markdown document. There is no UI surface to verify. (Reviewer-facing usability is instead validated by the document's navigability — see Section 9.)

**Runtime Validation:** the canonical binary was exercised through its real entry point (`cmd/maddy -debug`) across all documented conditions. Status legend: ✅ Operational | ⚠ Partial | ❌ Failing.

- ✅ **C1 — Mixed SMTP+IMAP (`mixed.conf`):** one `sql: go-imap-sql version 0.4.0` line despite **three** references (lines 12/17/18); both endpoints reach `listening` (`tcp://127.0.0.1:10025`, `:10143`); exit 0. Proves at-most-once lazy initialization.
- ✅ **C5 — Forward reference (`forward.conf`):** `sql` block defined last but referenced earlier; resolves cleanly; exit 0. Proves order-independence.
- ✅ **C3 — Orphan / unused block (`orphan.conf`):** startup aborts with `Unused configuration block … orphan_db (sql)`; exit 2. The settlement guard fires correctly.
- ✅ **C6a — Dangling `&`-reference (`badref.conf`):** `reference &nonexistent` logged **before** `unknown config block: nonexistent`; exit 2. Attempt-vs-settlement ordering.
- ✅ **C6b — Unknown inline module (`badmod.conf`):** `new module no_such_check_xyz []` logged **before** `unknown module`; exit 2. Log-before-construction ordering.
- ✅ **C2 — Driven SMTP transaction (`pipeline.conf` + `smtp_driver.py`):** per-message `initializing state for require_matching_ehlo:`; quarantine `{reason:"no such host", smtp_code:550}`; **three** modifier scopes in order (global → per-source → per-rcpt); recipient rejection 550; exit 0. Daemon 21/21 lines + client output identical across runs.
- ✅ **C2b — C2 under `--network none`:** quarantine reason → `network is unreachable` / `450`; client verdict 550 unchanged. Environment-sensitive value correctly labeled.
- ✅ **C4 — systemd notifications (`systemd_listen.py`):** byte-exact `READY=1\nSTATUS=Listening for incoming connections...` and `STOPPING=1\nSTATUS=Waiting for running transactions to complete...` datagrams; daemon exit 0.
- ✅ **Version banner:** `/work/maddy-bin -v` → `maddy unknown (built from source tree)` (canonical default-build result).
- ✅ **Endpoint-personality coverage:** a config with all four personalities (`smtp`/`submission`/`lmtp`/`imap`) sharing one `sql` instance all reached `listening`.

**Result:** 100% of documented runtime conditions are Operational and reproducible; no partial or failing conditions.

---

## 5. Compliance & Quality Review

Cross-mapping of the AAP deliverables and the `SWE-AtlasQnA-Repo` governing rules to their validated status. Legend: ✅ Pass | ⚠ Partial | ❌ Fail.

| Benchmark / Rule | Requirement | Status | Progress | Evidence |
|---|---|---|---|---|
| Deliverable rule | Create `blitzy/documentation/<branch>.md` answering the questions | ✅ Pass | 100% | `maddy_26452dd8dd78.md`, 1,207 lines |
| Read-only scope | No existing source/test/config/build file modified | ✅ Pass | 100% | `git diff 26452dd..HEAD` = 1 file added only |
| No code additions | No code other than the answer document | ✅ Pass | 100% | Diff shows only the `.md` file |
| No dependency changes | `go.mod`/`go.sum` untouched | ✅ Pass | 100% | Both files unchanged |
| Investigate-by-running-first | Build & run before writing | ✅ Pass | 100% | 8 runtime conditions captured |
| Canonical path only | Real `cmd/maddy` entry point; label non-canonical | ✅ Pass | 100% | No test doubles/hooks used |
| Reproducibility | Stable across ≥2 runs | ✅ Pass | 100% | run1+run2 shown per condition |
| Every-condition coverage | Primary + secondary + error/edge + transitional | ✅ Pass | 100% | C1/C2/C2b/C3/C4/C5/C6a/C6b |
| Complete evidence | Full unedited output + exact command; no elision | ✅ Pass | 100% | 52 code blocks, verbatim output |
| Exactness | `file:line` + named function/struct per claim | ✅ Pass | 100% | 143/143 citations verified |
| Observed-vs-Inferred labeling | Provenance labeled; inferred marked | ✅ Pass | 100% | 58 labels + summary section |
| Coverage pass | Every sub-question & named item answered | ✅ Pass | 100% | Coverage pass section present |
| Cleanup | Temporary artifacts removed; tree clean | ✅ Pass | 100% | `git status --porcelain` empty |
| Compilation quality | Clean build | ✅ Pass | 100% | build exit 0 (benign warning only) |
| Test quality | Existing suite still green | ✅ Pass | 100% | 20/20 packages ok, 0 fail |
| Human SME review | Independent technical sign-off | ⚠ Partial | 0% | Pending — the 3h remaining gate |

**Fixes applied during autonomous validation/authoring:** 13 code-review findings resolved (the document nearly tripled, 438→1,207 lines, adding runtime conditions, evidence, and the coverage pass); 1 QA finding resolved (Condition 2b resolver IP labeled environment-specific); and **two AAP citation off-by-one anchors were corrected** in the deliverable (`sql.go` `Init` is L176 not L177; `go-sqlite3` is `go.mod:L26` not L31) — the document is more accurate than the source AAP.

**Outstanding compliance items:** only the human SME review/sign-off (Section 2.2), which is a process gate rather than a code or content defect.

---

## 6. Risk Assessment

Risk profile is **LOW** across all categories — this is a read-only documentation task with nothing deployed, no dependency changes, no credentials, and no running production service. Risks are reported honestly, including "none" results.

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Citation drift if the doc is later rebased onto a newer Maddy revision (143 anchors pinned to HEAD `26452dd`) | Technical | Low | Low | Explicit "Source lineage and read-only note" pins every anchor to `26452dd`; re-verify only if rebased onto a different revision | Mitigated |
| Runtime-condition reproduction requires the maintainer Docker image (no native Go on host; needs Go 1.18.10 + warm OFFLINE cache) | Technical | Low | Medium | Doc records the exact Docker invocation and `GOPROXY=off` build commands; validator reproduced all 8 conditions byte-identical | Mitigated |
| Build toolchain (Go 1.18.10) newer than declared target `go 1.13` (`go.mod:L3`) | Technical | Low | Low | Intentional maintainer toolchain; doc explicitly makes no max-patch claim and labels the toolchain as container-provided | Accepted / Documented |
| Plaintext `tls off` loopback listeners used in observation configs | Security | Low (informational) | N/A | Testing-only, bound to `127.0.0.1`, transient and deleted; doc carries an explicit caveat that plaintext must never be exposed beyond loopback | Closed |
| Human-review bottleneck — merge depends on SME availability to review a dense 10,632-word technical doc | Operational / Process | Low | Low–Medium | Self-contained doc with per-claim provenance labels, a Coverage pass, and a Condition→command→output map to accelerate review | Open (= the remaining work) |
| No deployed service / no runtime operational footprint (monitoring, health checks, backups) | Operational | None | N/A | Deliverable is a static Markdown file; nothing runs in production | N/A (no exposure) |
| No external integrations; dependencies unchanged | Integration | None | N/A | Read-only task; `go.mod`/`go.sum` untouched; deps used only to build for observation; no API keys/webhooks/services wired | N/A (no exposure) |

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Project Hours Breakdown
    "Completed Work" : 37
    "Remaining Work" : 3
```

**Remaining hours by category** (from Section 2.2; sums to 3h):

```mermaid
xychart-beta
    title "Remaining Hours by Category"
    x-axis ["SME Technical Review", "PR Approval and Merge"]
    y-axis "Hours" 0 --> 3
    bar [2.5, 0.5]
```

**Integrity note:** "Remaining Work" = **3** here equals the Remaining Hours in Section 1.2 and the sum of the Section 2.2 Hours column (2.5 + 0.5 = 3). "Completed Work" = **37** equals Completed Hours in Section 1.2 and the Section 2.1 total.

---

## 8. Summary & Recommendations

**Achievements.** The project is **92.5% complete** (37 of 40 hours). All 18 AAP-specified deliverables are done and independently validated: the canonical binary was built and exercised through its real entry point; eight runtime conditions were captured and reproduced byte-identical across two runs; and the sole in-scope artifact — `blitzy/documentation/maddy_26452dd8dd78.md` (1,207 lines) — answers all five onboarding questions with 143 verified `file:line` citations and per-claim provenance labels. Strict read-only compliance was maintained: exactly one file added, working tree clean.

**Remaining gaps.** The outstanding 3 hours (7.5%) are entirely the path-to-production human gate for a documentation deliverable: an SME technical review (citation spot-check, reproducing at least one runtime condition, and an end-to-end read) followed by PR approval and merge. There are **no** code, test, runtime, or citation defects to fix — autonomous validation required **zero edits**.

**Critical path to production.** Reviewer pulls the maintainer image → reproduces Condition 1 and spot-checks citations → reads the document → approves and merges. Estimated wall-clock: well under one working day.

**Success metrics (all met by autonomous work):** compilation exit 0; 20/20 test packages pass; 8/8 runtime conditions reproduced byte-identical; 143/143 citations verified; all five sub-questions and every named item covered; repository unchanged except the single doc.

| Dimension | Assessment |
|---|---|
| Scope adherence (read-only) | ✅ Exemplary — 1 file added, 0 modified |
| Content completeness | ✅ All 5 questions + coverage pass |
| Evidence rigor | ✅ 143 citations, 8 reproduced conditions |
| Build/test health | ✅ Clean build, 20/20 packages |
| Production readiness | ✅ Ready pending human review/merge |

**Production readiness assessment: READY for human review and merge.** The deliverable is content-complete, accurate, and validated; only the standard human sign-off remains.

---

## 9. Development Guide

This guide covers how to build, run, reproduce the documented observations, and review the deliverable. Commands marked **(host-verified)** were executed successfully on the host during this assessment; commands marked **(Docker image)** require the maintainer image because there is **no native Go on the host** (validated byte-identical by autonomous validation).

### 9.1 System Prerequisites

- **Host (verified present):** `git` 2.51.0, `docker` 28.5.2, `python3` 3.13.7, `gcc` 15.2.0.
- **Go toolchain:** **not on the host.** Use the maintainer Docker image, which provides Go 1.18.10, gcc 10.2.1, python3 3.9.2, and a **warm OFFLINE Go module cache**. The project declares target `go 1.13` (`go.mod:L3`); the container toolchain is the canonical build environment used here.
- **Image:** `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_foxcpp_maddy_1.0` (baked source at `/app`, rev `26452dd`).

### 9.2 Environment Setup

```bash
# (host-verified) Confirm repository identity
cd /path/to/maddy-checkout
git branch --show-current          # -> blitzy-25330606-79c5-4622-9c90-dc77a1a63ce9
git log -1 --pretty='HEAD=%h %s'   # -> HEAD=1cc08de docs(maddy): ... (QA F-1)

# (host-verified) Confirm the deliverable and read-only compliance
test -f blitzy/documentation/maddy_26452dd8dd78.md && wc -l blitzy/documentation/maddy_26452dd8dd78.md
git status --porcelain             # empty = clean working tree
git diff --name-status 26452dd..HEAD   # -> A  blitzy/documentation/maddy_26452dd8dd78.md
```

Run the canonical build/observe environment with the repository mounted **read-only** and all writes directed to a scratch dir **outside** the repo:

```bash
# (Docker image) Canonical reproduction harness
docker run --rm \
  -v /path/to/maddy-checkout:/src:ro \
  -v /tmp/maddy-capture:/work \
  -w /src \
  ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_foxcpp_maddy_1.0 \
  -c 'bash /work/run_default.sh'
```

Scratch discipline used for every temporary artifact (never inside the repo):

```bash
OBS="$(mktemp -d /tmp/maddy-obs.XXXXXXXX)"; chmod 700 "$OBS"
```

### 9.3 Dependency Installation

**None required.** This is a read-only task with a warm offline module cache. Builds and tests run with `GOPROXY=off` and `-mod=readonly`, which forbid network fetches and any change to `go.mod`/`go.sum`. A C compiler (present in the image) is needed because `github.com/mattn/go-sqlite3 v1.11.0` (`go.mod:L26`) is cgo-based.

### 9.4 Build & Run / Observe

```bash
# (Docker image) Build the canonical binary (exit 0; only a benign SQLite -Wreturn-local-addr warning)
CGO_ENABLED=1 GOPROXY=off go build -mod=readonly -o /work/maddy-bin ./cmd/maddy

# (Docker image) Version banner (canonical default build)
/work/maddy-bin -v            # -> maddy unknown (built from source tree)

# (Docker image) Condition 1 — mixed SMTP+IMAP, debug logging on
/work/maddy-bin -debug -config "$OBS/mixed.conf" > "$OBS/c1.out" 2>&1 &
pid=$!
for i in $(seq 1 100); do grep -q "imap: listening" "$OBS/c1.out" && break; kill -0 "$pid" 2>/dev/null || break; sleep 0.1; done
kill -TERM "$pid"; wait "$pid"; echo "exit=$?"

# (Docker image) Condition 2 — driven SMTP transaction (after creating a user)
python3 "$OBS/smtp_driver.py" 10025

# (Docker image) Condition 4 — systemd readiness/stopping datagrams
python3 "$OBS/systemd_listen.py" "$OBS/notify.sock" "$OBS/mixed.conf" /work/maddy-bin
```

Config essentials for the observation files (avoid the common pitfalls in Section 9.7): global `state` and `runtime` set to absolute paths, a global `hostname localhost`, and `tls off` on loopback listeners.

### 9.5 Verification

```bash
# (Docker image) Full test suite still green after the read-only change
CGO_ENABLED=1 GOPROXY=off go test -mod=readonly ./...   # -> exit 0 (20/20 packages ok)

# Expected Condition 1 signature (proves at-most-once lazy init):
#   [debug] sql: go-imap-sql version 0.4.0     <- appears exactly once
#   smtp: listening on tcp://127.0.0.1:10025
#   imap: listening on tcp://127.0.0.1:10143
#   exit=0                                       <- clean SIGTERM shutdown

# (host-verified) Cleanup & integrity
rm -rf "$OBS"
git status --porcelain    # must be empty
```

### 9.6 Example Usage — Reviewing the Deliverable

```bash
# (host-verified) Navigate the five Question sections
grep -nE '^## Question [0-9]' blitzy/documentation/maddy_26452dd8dd78.md
# -> L181 Q1, L260 Q2, L401 Q3, L473 Q4, L875 Q5

# (host-verified) Jump to the coverage / lineage / provenance summary
grep -nE '^## (Coverage pass|Source lineage|Observed vs)' blitzy/documentation/maddy_26452dd8dd78.md

# (host-verified) Spot-check a citation against source (example: registry maps)
sed -n '8,10p' internal/module/registry.go
# -> modules   = make(map[string]FuncNewModule)
#    endpoints = make(map[string]FuncNewEndpoint)   (matches the doc verbatim)
```

### 9.7 Troubleshooting

- **`unknown module or global directive`** → use the global directives `state` and `runtime` (absolute paths), **not** `state_dir`/`runtime_dir` (`maddy.go:L245-L246`).
- **`smtp` fails to start / missing hostname** → add a global `hostname localhost` (`internal/endpoint/smtp/smtp.go:L558`).
- **Relative `dsn` file appears in an unexpected place** → maddy `os.Chdir`s into the state directory (`maddy.go:L220`), so relative paths resolve there.
- **`go: command not found` on the host** → expected; build/run inside the maintainer Docker image.
- **SQLite `-Wreturn-local-addr` warning** → benign amalgamation warning, not an error; build still exits 0.
- **Exit code `2` on startup** → a config/startup failure (`maddy.go:L135/L146/L154/L160`), e.g., an unused block or a bad reference; exit `0` is a clean single-`SIGTERM` shutdown (`maddy.go:L163`).
- **Debug lines missing** → the `reference &…` / `new module …` / `initializing state for …` lines require the `-debug` flag (`maddy.go:L104`).

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---|---|
| Build canonical binary | `CGO_ENABLED=1 GOPROXY=off go build -mod=readonly -o /work/maddy-bin ./cmd/maddy` |
| Build maddyctl | `go build -o /work/maddyctl ./cmd/maddyctl` |
| Build entire module | `CGO_ENABLED=1 GOPROXY=off go build -mod=readonly ./...` |
| Run test suite | `CGO_ENABLED=1 GOPROXY=off go test -mod=readonly ./...` |
| Version banner | `/work/maddy-bin -v` |
| Run daemon (debug) | `/work/maddy-bin -debug -config <conf>` |
| Drive SMTP transaction | `python3 smtp_driver.py 10025` |
| Capture systemd signals | `python3 systemd_listen.py <sock> <conf> <bin>` |
| Read-only compliance check | `git status --porcelain` ; `git diff --name-status 26452dd..HEAD` |
| Navigate deliverable | `grep -nE '^## Question [0-9]' blitzy/documentation/maddy_26452dd8dd78.md` |

### Appendix B — Port Reference (observation configs, loopback only)

| Service | Address | Notes |
|---|---|---|
| SMTP endpoint | `tcp://127.0.0.1:10025` | Temporary observation config; `tls off` (testing-only) |
| IMAP endpoint | `tcp://127.0.0.1:10143` | Temporary observation config; `tls off` (testing-only) |
| `submission`/`lmtp` | loopback (as configured) | Covered via shared-constructor identity; personalities `smtp`/`submission`/`lmtp` → `smtp.New` |
| systemd `NOTIFY_SOCKET` | AF_UNIX/SOCK_DGRAM (scratch path) | Bound by `systemd_listen.py` to capture `READY=1`/`STOPPING=1` |

### Appendix C — Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/maddy_26452dd8dd78.md` | **The deliverable** (1,207 lines) |
| `maddy.go` | Orchestration: side-effect imports, `Run`, `moduleMain`, `instancesFromConfig` |
| `cmd/maddy/main.go` | Canonical entry point → `maddy.Run()` |
| `internal/module/registry.go` | Two factory maps + `Register`/`RegisterEndpoint` |
| `internal/module/instances.go` | Instances registry + `GetInstance` (lazy init) |
| `internal/config/module/modconfig.go` | `ModuleFromNode` — `&` detection + inline branch |
| `internal/msgpipeline/check_runner.go` | Parallel check runner + per-message state line |
| `internal/endpoint/smtp/smtp.go`, `internal/endpoint/imap/imap.go` | Endpoints; register `smtp`/`submission`/`lmtp` and `imap` |
| `internal/storage/sql/sql.go` | Shared `sql` backing instance (`Init` at L176) |
| `go.mod` | Declares `go 1.13`; pins dependency versions |

### Appendix D — Technology Versions

| Component | Version | Source |
|---|---|---|
| Go (declared target) | 1.13 | `go.mod:L3` |
| Go (build toolchain) | 1.18.10 | maintainer Docker image |
| gcc (image) | 10.2.1 | maintainer Docker image |
| Python (drivers) | 3.9.2 (image) / 3.13.7 (host) | observation drivers |
| `emersion/go-smtp` | v0.12.1-0.20191206174923-1f576e0ec85c | `go.mod:L19` |
| `emersion/go-imap` | v1.0.1 | `go.mod:L9` |
| `foxcpp/go-imap-sql` | v0.3.2-0.20191208094750-8b4ec6b19a78 | `go.mod:L20` |
| `mattn/go-sqlite3` (cgo) | v1.11.0 | `go.mod:L26` |
| `urfave/cli` | v1.22.1 | `go.mod:L29` |
| `miekg/dns` | v1.1.22 | `go.mod:L27` |
| `emersion/go-msgauth` | v0.3.2-0.20191028231513-55b75676976c | `go.mod:L17` |
| `blitiri.com.ar/go/spf` | v0.0.0-20191018194539-a683815bdae8 | `go.mod:L6` |
| `google/uuid` | v1.1.1 | `go.mod:L23` |

### Appendix E — Environment Variable Reference

| Variable | Value | Purpose |
|---|---|---|
| `CGO_ENABLED` | `1` | Required for the cgo SQLite driver |
| `GOPROXY` | `off` | Forbid network fetches; use warm offline cache |
| build flag `-mod=readonly` | (set on build/test) | Forbid changes to `go.mod`/`go.sum` |
| `NOTIFY_SOCKET` | scratch AF_UNIX path | Set by `systemd_listen.py` so maddy emits readiness datagrams |

### Appendix F — Developer Tools Guide

- **Docker harness** — mount the checkout `:ro` at `/src`, a scratch dir at `/work`; run non-interactively with `-c`. Ensures the repository cannot be modified.
- **`grep -nE` / `sed -n`** — the fastest way to navigate the 1,207-line deliverable and to spot-check any `file:line` citation against source (host-verified in Section 9.6).
- **`git diff --name-status 26452dd..HEAD`** — one-shot read-only-compliance proof (must show only the single `.md` file).
- **Python `smtplib` driver** — drives `EHLO`/`MAIL FROM`/`RCPT TO` to elicit the pipeline's per-message check-state and modifier-scope debug lines.

### Appendix G — Glossary

| Term | Meaning |
|---|---|
| **Module** | A unit satisfying the `Module` contract (`Init`/`Name`/`InstanceName`); registered by a factory function |
| **Factory registration** | Blank-import side-effect: each package's `init()` calls `Register`/`RegisterEndpoint`, populating the global maps before `main` |
| **Instance** | A configured module object created from a top-level config block; registered via `RegisterInstance` but not initialized until referenced |
| **Lazy initialization** | Deferring a module's `Init` until first use via `GetInstance`, cached in the `Initialized` map (at-most-once) |
| **`&` reference** | A directive argument beginning with `&name` that `ModuleFromNode` resolves to a live, already-or-lazily-initialized instance |
| **Endpoint** | A module in a separate registry, initialized eagerly and directly (the graph's active roots); has no instance name and cannot be inline |
| **Check / Modifier** | Pipeline interfaces invoked by the message pipeline in a fixed order across global/per-source/per-recipient scopes, without explicit config wiring |
| **Unused-block guard** | Startup abort (`Unused configuration block`) if any top-level block was never initialized — evidence the graph settled |
| **Condition (C1–C6b)** | A named, reproducible runtime scenario captured in the deliverable with its exact command and complete output |