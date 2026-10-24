# GhostEraser — PLAN.md

**GhostEraser** is our deletion audit agent for HackerRank Orchestrate October 2026, "Ghost Data Still Exists".

> Status: **plan for review. No implementation yet.**
> Companion files: [`ARCHITECTURE.md`](ARCHITECTURE.md) (modules, schemas, tools, invariants), [`EVALUATION_STRATEGY.md`](EVALUATION_STRATEGY.md) (metrics, custom cases, ablations, usage report), [`RESEARCH_NOTES.md`](RESEARCH_NOTES.md) (official rules, winner evidence, domain research, with links and CONFIRMED / HYPOTHESIS / UNKNOWN labels).

GhostEraser's approach is a **deterministic deletion-state engine wrapped in a real but tightly bounded tool-calling agent**. Ship the engine early. Check it against all 32 labelled sample rows before writing any LLM code. Spend the rest of the time on harness-measured accuracy and on being able to defend the build. **Do not trade correctness for sample score late in the day.**

---

## TL;DR

- **What is scored (CONFIRMED, Help Center):** four signals, each scored 0–100, combined as a weighted sum: Code ZIP 30%, Output file 30%, AI interview 30%, Chat transcript 10%. In the May edition every Top‑10 finisher was top-quartile on all four signals. Balance across the four is the strategy.
- **What to build:** a typed, deterministic core that does all the deciding:
  identity graph → evidence ingestion → per-resource timeline as of `audit_at` → `deletion_rules.csv` engine → one `CaseDecision` → canonical renderer → validator that **fails the run**.
  An LLM handles only the bounded, uncertain parts: multilingual intent classification, schema-enforced extraction from PDFs/images, and a step-capped investigator loop over closed typed tools. Every LLM role is cached, traced, and has a `--no-llm` fallback.
- **Completely free tier:** no billing enabled anywhere. Free-tier API keys only (Gemini + Groq), local open-source OCR and parsing first, every LLM call batched, cached and counted against a hard daily quota budget (§3.1). Free quotas are small, so the design treats LLM calls as scarce.
- **What fixes September:**
  - Reverse-engineer every convention from `sample_output.csv` first (Phase 1).
  - Build the end-to-end harness **before** the engine (Phase 2).
  - Measure rare operations (e.g. `REPORT_RESTORE_RISK`) only on rows where they occur.
  - Never weaken an invariant to raise a score; feature freeze at the end of Phase 7.
  - In the interview, quote measured numbers for named modules.

---

## 1. Situation and timeline

| Item | Status | Detail |
|---|---|---|
| Start | CONFIRMED | Challenge email arrives Oct 24, 2026, 12:30 UTC (18:00 IST) — Help Center |
| End | **CONFIRMED** | **Oct 25, 2026, 18:00 IST (12:30 UTC)**, per the challenge email. Record it in `log.txt` in Phase 0. Submission target: **17:30 IST** |
| AI interview | CONFIRMED | 30‑minute voice interview, camera + mic, opens after submission |
| Live evaluation / results | CONFIRMED | Oct 26–27 / Oct 27 |
| File limits | CONFIRMED | ≤ 50 MB per upload; exclude venvs, build artifacts and the provided dataset |

**Feasibility:** at 23:36 IST on Oct 24 about 18 h remain and no code exists, so the phases in §4 are budgeted at ~17.5 h with IST checkpoints. That leaves almost no slack; the cut order in §4.3 is part of the plan, not an emergency measure. Start at **Phase 0 → Phase 1** and skip rereading the research.

---

## 2. Problem analysis

### 2.1 Decomposition (per case, 518 cases)

1. **Intent gate.** Does `request_text` authorize deletion? Requests may be multilingual. Access, correction, export, marketing opt-out or any non-deletion action → `NOT_APPLICABLE,[]`, even if other fields look like a deletion case (AGENTS.md).
2. **Identity resolution.** The request email may be current or historical. Build the full identifier set from `user_identities.csv` (user_id, name, email, phone, email hash) and `user_audit_log.csv` (historical changes): every value valid for this user at any time ≤ `audit_at`.
3. **Evidence linking.** `evidence_files.csv` maps `file_id` → path/origin/capture time but does **not** say which user a file concerns. Link through identifiers, hashes and resource IDs found inside each file.
4. **State reconstruction as of `audit_at`.** For each relevant `(system_id, resource_id)` decide PRESENT / ABSENT / REDACTED / ANONYMIZED; for flows and backups decide ACTIVE / DISABLED / BLOCKED. Ignore anything captured or occurring after `audit_at`. `reported_status` and activity results are **claims**.
5. **Policy application.** Look up the required treatment in `deletion_rules.csv` by data category and condition: DELETE, REDACT, ANONYMIZE, RETAIN, or ACCOUNT_TOMBSTONE (keep stable `user_id`, remove name/email/phone/email hash).
6. **Status.** COMPLETE only if all four spec conditions hold (every copy treated; no active sync/import/restore path; retained records hold only permitted identifiers; required records still exist). Otherwise INCOMPLETE.
7. **Render steps.**
   - COMPLETE → verification steps only, each citing evidence.
   - INCOMPLETE → findings that cite evidence, then remediation and verification with `evidence_id=""`.

### 2.2 Output contract (CONFIRMED: problem_statement.md, AGENTS.md)

- Header `case_id,deletion_status,steps_json`; exactly **518** rows, sorted by `case_id`, at repo-root `output.csv`.
- Step keys exactly: `step_no, phase, system_id, resource_id, operation, state, evidence_id`. `step_no` is 1..n, consecutive.
- 15 allowed (operation, state) pairs; no invented phases, operations or states.
- Order: phase (FINDING → REMEDIATION → VERIFICATION) → operation order in the spec table → `system_id` asc → `resource_id` asc.

**Open points to resolve from the samples in Phase 1:**

| Open point | Why it matters | How to resolve |
|---|---|---|
| "Disable flows / block restores before changing dependent records; process downstream copies before their source when the dependency requires it" (AGENTS.md §6.5) vs. plain lexical sort | Could change order inside one operation | Implement plain lexical sort, diff against 32 samples; add a dependency tie-break only if a sample needs it |
| Which evidence ID to cite when several prove the same fact (earliest? latest ≤ `audit_at`? file over activity row?) | Affects `evidence_id` on every finding and COMPLETE verification | Derive from samples; write the rule with sample case IDs |
| Evaluation report path: README says `code/evaluation/`, AGENTS.md and spec say `evaluation/` | Missing deliverable risk | Generate identical content at **both** paths from one script |
| AGENTS.md submission URL has a typo (`wwww.`) | Wrong submission page | Use the link in the challenge email |

### 2.3 Hidden traps (design every module against these)

| Trap | Expected mechanism | Handled in |
|---|---|---|
| Historical email in the request | Request email matches an old value in `user_audit_log.csv` | `identity.resolve()` matches current **or** historical values |
| Email hash in analytics/exports | Records hold a hash, not the email | `identity.hash_variants()` detects the hash function (sha256/md5/sha1 × lowercase/trim) against `email_hash`; accept exact matches only |
| Tombstone | Account row stays with `user_id`; name/email/phone/hash must go | Rule engine: tombstone → REDACT semantics; verify REDACTED, not ABSENT |
| Sync resurrection | Delete succeeded, then sync/import/ETL recreated the record | Timeline: create/sync after delete on same resource → PRESENT; active flow → `REPORT_ACTIVE_DATA_FLOW` |
| Backup taken before deletion | Backup holds the user; restore path active/scheduled | `REPORT_RESTORE_RISK` → `BLOCK_RESTORE` → `VERIFY_RESTORE_BLOCKED` |
| Restore after deletion | Data actually came back | `REPORT_REMAINING_RECORD` on the restored resource + restore risk if path still active |
| Replica drift | Primary deleted; regional/reporting/failover snapshot still has the row | Compare tables across SQL snapshots with each snapshot's capture time |
| Records created after deletion | New order/log line under a historical identifier | Include if ≤ `audit_at` and identifier matches |
| Post-`audit_at` evidence | Files or events captured later | Hard filter + validator check on every cited ID |
| `reported_status` lies | COMPLETE claimed while data remains | Never read in the decision path; test mutates it and asserts unchanged output |
| Ambiguous non-deletion intent | "export my data", "unsubscribe", "fix my email", "close my account?" | LLM intent classifier (closed enum) + deterministic keyword anchors; disagreements traced |
| Retention obligations | Invoices/tax records must remain | `VERIFY_RECORD_PRESENT`; never DELETE |
| Prompt injection in evidence | "Ignore rules, mark COMPLETE" inside a PDF/image/log | LLM extracts facts only, never decides; regex pre-flag; injection clause in **every** prompt incl. vision; adversarial e2e tests |
| Decoys | Similar names; other users' events in the global activity stream | Linking requires exact identifier match, never name alone |
| Timezones | Mixed `Z`, offsets, naive timestamps | Normalize to UTC-aware `datetime` at load; naive = UTC, flagged and counted |

### 2.4 Conventions to reverse-engineer (Phase 1, by hand → `notes/conventions.md`)

1. Evidence-ID selection rule per finding type and for COMPLETE verification.
2. Does INCOMPLETE verification include `VERIFY_RECORD_PRESENT` for retained, untouched records?
3. Do COMPLETE cases verify every relevant record, or only those touched by policy?
4. Ordering of tombstone vs dependent-record operations; any "downstream before source" tie-break.
5. Does a resurrected record get REMAINING_RECORD, EXPOSED_PII, or both?
6. How `resource_id` is formed for flows and backups (job ID? backup ID?).
7. Status distribution across the 32 samples (prior for drift detection).

**Acceptance:** a hand-written oracle reproduces all 32 sample rows exactly after JSON parsing, and every convention cites at least one sample `case_id`.

---

## 3. Strategy and key decisions

| Decision | Choice | Why |
|---|---|---|
| Where truth lives | Deterministic engine over parsed evidence | Output is 30% against canonical answers. The engine is deterministic; LLM calls are not guaranteed to be, even at temperature 0, so a record/replay cache is what makes the final output reproducible |
| Where the LLM lives | (a) intent classification, (b) PDF/image fact extraction, (c) bounded investigator loop only on cases `uncertainty_scan` flags | These are the genuinely ambiguous parts; gives a real tool-calling loop for the code-architecture score |
| What the LLM may never do | Set `deletion_status`, invent a resource, emit a step | It proposes facts with an `evidence_quote`; code checks the quote is a substring of the source, then decides |
| Extraction granularity | Per evidence **file**, cached by `sha256(file) + prompt_version + model` | Evidence is global; one extraction serves many cases; cuts tokens |
| Fallback | `--no-llm` mode (keyword intent + PDF text layer + local OCR + regex) must produce a valid 518-row file | Rate limits/outages cannot stop the run; accuracy delta reported as an ablation |
| Models | Free tier only. **Proposed** routing: one free vision-capable model for images (Gemini Flash-Lite class) and one free text model for intent + agent (Groq `gpt-oss-120b` class). Exact model IDs, free-tier eligibility, quotas and features must be verified in Phase 0. A fallback only if it passes the golden gate | September's fallback model silently changed semantics; free daily caps differ per provider, so roles are split to spread load |
| Cost | $0: no billing enabled, no paid features (e.g. search grounding), open-source local tools | User requirement; also removes cost-spike risk |

Detailed module layout, schemas, tool contracts and invariants: **[`ARCHITECTURE.md`](ARCHITECTURE.md)**.

### 3.1 Free-tier constraints and budget

Free quotas change often and some are not published, so the figures below are **planning assumptions to verify in Phase 0** (sources in [`RESEARCH_NOTES.md`](RESEARCH_NOTES.md) §4).

| Provider (free tier) | Used for | Reported limits (verify) | Notes |
|---|---|---|---|
| Google Gemini API, Flash-Lite class | Vision extraction (images, text-less PDF pages) | ~500 requests/day per project for 3.x Flash-Lite; ~20/day for 3.x Flash (developer reports, Sep 2026). Google publishes no fixed free limits; AI Studio shows your project's real limits | Pro models are paid-only. Free-tier prompts may be used by Google to improve products, which is acceptable here because the dataset is fictional |
| Groq, `openai/gpt-oss-20b` (intent) and `openai/gpt-oss-120b` (agent) | Batched intent classification; agent investigator loop | Per model: ~30 requests/min, ~1,000 requests/day; ~200K tokens/day reported for `gpt-oss-120b` (Jun–Aug 2026 reports). Limits appear to be per model, which would give each role its own quota — **verify** | The **tokens/day** cap binds first, so the budget below is in tokens, not just calls. Daily reset time is unknown — **verify** |
| OpenRouter `:free` models | Emergency fallback only | 20 requests/min; 50 requests/day without purchased credits | Too small for main use; golden gate still applies |
| Local, open-source | PDF text (pypdf/pdfplumber), OCR (Tesseract via pytesseract), SQLite, JSON/log parsing | No quota | Always tried before any LLM call |

**Quota rules (built into code, not just intent):**
1. **Local first.** PDF text layer → Tesseract OCR → vision LLM only if OCR finds no usable identifiers.
2. **Dedupe, then batch.** Normalize and deduplicate request texts first (templated requests repeat), then send ~20–25 unique texts per call. Batch membership and order are fixed (sorted by text hash) and results are cached **per text**, so re-batching never changes a cached answer.
3. **Extract once.** Evidence extraction is per file, cached on disk; reruns cost zero calls.
4. **Hard budget.** A `QuotaGuard` tracks requests and tokens per provider and model per day (persisted on disk, so restarts can't reset it; reconciled with the provider's remaining-quota response headers where they exist) and stops LLM roles **before** the cap; affected work falls back to the deterministic path and is counted. Exhaustion behaviour and its accuracy cost are tested per provider (`EVALUATION_STRATEGY.md` §5a).
5. **Spend the scarce calls on the final run.** Development iterations use the 32 samples plus cached results; ablations run on samples only.
6. **Quota reset.** Gemini daily quotas reset at midnight Pacific time = **12:30 IST on Oct 25** (PDT is still in effect). The deadline is 18:00 IST, so: development before 12:30 IST uses the current quota day, and the **final full-dataset run happens after 12:30 IST** on a fresh Gemini quota. Groq's reset time must be checked in Phase 0; if it is a rolling 24 h window, budget the final run against what remains.
7. **No quota evasion.** One project and one key per provider; no rotating extra accounts to dodge limits.

**Planned budget for one full run** (assumptions; replace with measured per-call tokens in Phase 4):

| Role | Calls | Tokens (est.) | Provider / model |
|---|---|---|---|
| Intent (deduped, batched) | ≤ 25 | ≤ 25 × 5K ≈ 125K | Groq `gpt-oss-20b` |
| Vision extraction | 1 per image/text-less PDF that OCR cannot read; target ≤ 150 | measure in Phase 4 | Gemini Flash-Lite |
| Agent loop | ≤ 10% of cases × ≤ 4 steps ≈ ≤ 210 | ≤ 140K total (≈ 650 per call **average**, so tool results must be trimmed hard); `QuotaGuard` stops the loop first | Groq `gpt-oss-120b` |
| Retries | ≤ 5% headroom | ≤ 5% | Same as role |

If intent and agent end up on the **same** Groq model (per-model quotas not confirmed), their combined tokens must fit one ~200K/day cap: then cut agent coverage first (§4.3). The earlier "210 calls" figure is only safe if tokens per call are measured and fit.

---

## 4. Implementation phases

You move to the next phase when its **gate passes**, not when the clock says so. Hours are estimates (≈17.5 h total, starting ~23:45 IST Oct 24). The checkpoint column is the **latest** time each gate should pass so that submission at 17:30 IST stays safe; if a checkpoint slips by more than 1 h, apply the cut order in §4.3 immediately.

### 4.1 Phase overview

| Phase | Name | Goal | Est. effort | Checkpoint (latest, IST) | Exit gate |
|---|---|---|---|---|---|
| 0 | Setup | Working repo, valid logging, free-tier keys verified | ~0.5 h | Oct 25 00:15 | First valid `log.txt` entry; `notes/quotas.md` filled |
| 1 | Understand & reverse-engineer | Know every output convention before writing engine code | ~1.5 h | 01:45 | **Gate A**: conventions approved, oracle 32/32 |
| 2 | Harness & validator first | Be able to measure and reject wrong output | ~1.5 h | 03:15 | **Gate B**: harness and validator proven |
| 3 | Deterministic core | End-to-end run with no LLM | ~5 h | 08:15 | **Gate C**: valid 518-row file, tag `core-v1` |
| 4 | LLM roles | Intent + evidence extraction, schema-enforced and cached (samples only) | ~2.5 h | 10:45 | **Gate D**: intent and extraction targets met |
| 5 | Harness-driven accuracy | Close failure buckets with tests | ~2 h | 12:45 | **Gate E**: sample accuracy targets met |
| 6 | Bounded agent & resilience | Real, capped tool-calling loop; graceful failure | ~1.5 h | 14:15 | **Gate F**: agent ablation done, replay and exhaustion tests pass |
| 7 | Docs & feature freeze | README, reports generated, scope locked | ~0.75 h | 15:00 | **Feature freeze** (3 h before deadline) |
| 8 | Hardening | Final full run (after the 12:30 IST Gemini reset), test-backed fixes only | ~1.25 h | 16:15 | Validator + all tests green on the final run |
| 9 | Package & submit | Ship 30 min early | ~0.75 h | **17:30 submit** | Submitted; interview follows |

### 4.2 Phase details

**Phase 0 — Setup (~0.5 h)**
- Tasks: clone repo; launch the AI tool from repo root; confirm `log.txt` entries carry exact `tool=`/`model=`; log the confirmed deadline (18:00 IST) and submission target (17:30 IST); create `.env` and `.gitignore`.
- Free-tier setup: create a Gemini API key in AI Studio and a Groq key **without enabling billing**; install Tesseract and Python libraries locally; record each project's and model's real limits (requests/min, requests/day, tokens/day, reset time, whether limits are per model) in `notes/quotas.md`; make one test call per provider confirming image input, JSON-schema output and (Groq) tool calling work on the free tier.
- Acceptance: first log entry valid; `pytest` runs; `notes/quotas.md` filled; both test calls return schema-valid JSON.
- Stop point: review the log entry before continuing.

**Phase 1 — Understand & reverse-engineer (~1.5 h)**
- Tasks: read spec, rules, manifest and **all 32 samples by hand**. Write `notes/conventions.md` (§2.4) and `notes/data_profile.md` (row counts, ID formats, evidence types, timezone formats, hash function). Hand-write an oracle for the samples.
- Acceptance: every convention cites ≥1 sample `case_id`; hash function matches 100% of identity rows; oracle reproduces 32/32 sample rows.
- Exit: **Gate A** — you approve the conventions. No engine code before this.

**Phase 2 — Harness & validator first (~1.5 h)**
- Tasks: `evaluation/harness.py` with the metrics in `EVALUATION_STRATEGY.md`; naive baselines (all INCOMPLETE with no steps; all NOT_APPLICABLE); `validate.py`; canonical sorter.
- Acceptance: gold-vs-gold scores 100%; sorter restores 32/32 shuffled sample arrays; the validator's **output layer** (ARCHITECTURE §8) accepts all 32 gold rows and rejects ≥ 14 hand-made bad rows, including findings with an empty, unknown, post-`audit_at`, or unrelated `evidence_id`. If a gold row fails a rule, the rule is wrong: fix the rule, not the gold. The **decision layer** is tested in Phase 3 once `CaseDecision` exists.
- Exit: **Gate B**.

**Phase 3 — Deterministic core (~5 h)**
- Tasks: `tables`, `sql_snapshots`, `identity`, `linking`, `timeline`, `rules`, `decision`, `render`; keyword intent stub; PDF text layer only.
- Suggested order: ingestion → identity → linking → timeline → rules → decision/render, running the harness after each module lands.
- Acceptance: `--no-llm` writes a valid 518-row file; sample status ≥ 85%; exact sequence ≥ 50%.
- Exit: **Gate C** — commit tagged `core-v1` (your safe fallback submission from here on).

**Phase 4 — LLM roles (~2.5 h)**
- Tasks: Tesseract OCR path; `intent.py` (schema enum, multilingual, batched); `extract_llm.py` (vision only where OCR fails, schema-enforced, cached, quote-grounded); `UsageLedger` + `QuotaGuard`. Batch pre-extract all evidence files once.
- Acceptance: 100% of sample NOT_APPLICABLE cases correct; extraction schema-failure < 2% of files; 0 ungrounded quotes accepted; projected full-run calls **and tokens** (from measured per-call usage on the samples) ≤ 70% of each provider/model daily cap, leaving 30% for retries and cache misses (a full rerun is served from cache).
- Exit: **Gate D**.

**Phase 5 — Harness-driven accuracy (~2 h)**
- Tasks: work through harness failure buckets, largest first; every fix starts with a failing regression test; keep a change log.
- Acceptance: sample status ≥ 95%; step F1 ≥ 0.90; exact sequence ≥ 75%; no slice regresses > 2 pp.
- Exit: **Gate E**. The change log feeds the evaluation report.

**Phase 6 — Bounded agent & resilience (~1.5 h)**
- Tasks: `uncertainty.scan` + `agent/loop.py` (budget 4 steps, compact tool results, traces) on the free Groq model; retries with backoff that respect `retry-after`; `QuotaGuard` stop before caps; `--no-llm` fallback confirmed.
- Acceptance: agent fires on ≤ 10% of cases; agent ON ≥ agent OFF on samples (otherwise ship OFF by default and say so); warm-cache replay reproduces the output hash; quota-exhaustion tests (`EVALUATION_STRATEGY.md` §5a) pass for each provider — zero requests after exhaustion, valid output — and the accuracy loss per scenario is measured and recorded.
- Exit: **Gate F**.

**Phase 7 — Docs & feature freeze (~0.75 h)**
- Tasks: own README; generators for `evaluation_report.md` (both paths) and `usage_report.md`; team-created cases finalized.
- Exit: **Feature freeze.** After this, only fixes that come with a failing test.

**Phase 8 — Hardening (~1.25 h)**
- Tasks: final full-dataset run **after the 12:30 IST Gemini quota reset**; replay it from the warm cache and confirm the same output hash; test-backed fixes only (re-runs are served from cache); regenerate reports from this exact run.
- Acceptance: validator passes; all tests green; usage report from this run's ledger.

**Phase 9 — Package & submit (~0.75 h)**
- Tasks: build zip; smoke-test in a clean venv (with an empty `.env`, to prove the bundled cache replays with no keys); **submit by 17:30 IST**; then rehearse the cheat sheet (§6) and take the interview.
- Acceptance: zip unpacks and runs with `python3 code/main.py`.

### 4.3 If a phase overruns

If a phase runs more than ~50% over its estimate, cut scope in this order:
1. Team-created cases beyond the required 3.
2. Fallback-model golden gate (run a single model per role only).
3. Agent scope narrowed to one flag type (keep the loop real, traced and capped).
4. Vision extraction limited to files OCR cannot read.

If **free quota** runs short: shrink agent coverage first, then vision extraction; never spend quota on full-dataset dev runs.

If **Gate C is not reached by ~09:30 IST**, freeze the scope at: deterministic core + LLM intent + a minimal agent loop on one flag type; skip further accuracy work beyond test-backed fixes.

**Never cut:** Phase 1 conventions, the harness, the validator, or the feature freeze.

**Rule for the whole build:** if a change raises the sample score but breaks an invariant or a team-created adversarial case, revert it and record the revert in the evaluation report.

**Prompt template for every AI-tool turn** (addresses the September transcript feedback):
1. Phase name and its estimated effort (from §4.1).
2. Files and function signatures to touch (paste from `ARCHITECTURE.md`).
3. Input/output schema (paste the Pydantic model).
4. Acceptance criterion with a number.
5. Explicit stop point ("Stop after X. Do not start Y.").

---

## 5. Risks and mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| Misread convention (citation choice, ordering) silently costs every row | High | Gate A + 32/32 oracle before engine code |
| Not enough time: ~18 h left, no slack for rest | **High** | IST checkpoints (§4.1); slip > 1 h triggers the cut order; `core-v1` is a valid fallback submission from ~08:15 IST |
| Over-strict validator rejects correct output (a rule that gold itself would fail) | Medium | Every validator rule must pass all 32 gold rows before it is enforced; rules about citation choice come from Phase 1 conventions, not guesses |
| Groq tokens/day exhausted by intent + agent together | Medium | Token budget (§3.1); intent and agent on separate models if per-model quotas are confirmed; text dedupe; trimmed tool results |
| Late tuning breaks safety (September C1) | Medium | Feature freeze after Phase 7; invariants raise; adversarial cases in pytest |
| Vision extraction noisy or slow | Medium | Text layer first; per-file cache; one batch pre-extraction in Phase 4 |
| Rate limits / outage | Medium | Exponential backoff (3 retries, honours `retry-after`); concurrency cap; always-valid `--no-llm` |
| Free daily quota exhausted mid-build or mid-run | **High** | `QuotaGuard` stops before caps; disk cache makes reruns free; dev on samples only; final run after the 12:30 IST reset; quota-exhausted run still valid |
| Free tier lacks a needed feature (image input or schema mode) | Low–Medium | Verified by test calls in Phase 0; fallback is local OCR + Pydantic validation with one retry |
| Free quotas change during the event | Low | Real limits recorded in `notes/quotas.md`; budget recomputed from them |
| Timezone ambiguity | Medium | UTC normalization; naive = UTC and counted; naive timestamps within ±14 h of `audit_at` flag the case for the agent, decision recorded |
| Hash function mismatch | Low–Medium | Empirical detection in Phase 1; fail loudly if < 100% |
| A phase overruns its estimate | Medium | Cut order in §4.3; `core-v1` tag is always a valid fallback submission |
| Agent loop lowers accuracy | Medium | Runs only on flagged cases; ship only if ablation shows no regression, otherwise default-off and documented honestly |
| Transcript reads as tool narration | Medium | Prompt template above: file, function, schema, threshold, stop point |

---

## 6. Interview preparation

**Module → input → output** (rehearse pointing at the file):

| Module | Input | Output |
|---|---|---|
| `identity.py` | request email, `audit_at` | `IdentifierSet` with sources |
| `linking.py` | identifiers | `EvidenceHit` list |
| `timeline.py` | hits + activity | `ResourceState` as of `audit_at` |
| `rules.py` | `ResourceState` | `RuleDecision` |
| `decision.py` | all of the above | `CaseDecision` |
| `render.py` | `CaseDecision` | canonically sorted steps |
| `validate.py` | output layer: CSV + evidence index; decision layer: + `CaseDecision`s | pass or `InvariantError` (incl. evidence validity on every finding) |
| `agent/loop.py` | uncertainty flags | proposed facts (accepted by code, never by the model) |

**Answers to prepare (fill in from your own harness):**
- "On the 32 samples, status accuracy is X%, step F1 is Y, exact sequence is Z%. On the N restore-risk rows, recall is …"
- "The LLM cannot mark a case COMPLETE. Status is computed in `decision.py` from rule compliance. The model only proposes facts, and each proposal must quote the source verbatim."
- "We reverted change N: it raised step F1 by 2 pp but failed the tombstone adversarial case."
- "LLM calls aren't guaranteed deterministic even at temperature 0, so reproducibility comes from a record/replay cache keyed per item; replaying the final run with no keys gives the same output hash. Across two cold runs on the samples, intent agreement was X%."
- "On free tiers the scarce resource is tokens per day, so `QuotaGuard` stops a role before the cap and the deterministic path takes over. With Gemini exhausted, status accuracy on the samples dropped by N cases (X pp)."
- Quote sample metrics as **counts** ("31 of 32") as well as percentages: with 32 samples, one case is ~3 pp.

**Monitoring signals → threshold → action:**

| Signal | Threshold | Action |
|---|---|---|
| Extraction schema-failure rate | > 2% of files per run | Alert; pin prompt version; reroute failing files to text-layer only |
| Unlinked evidence files | +5 pp vs baseline | Check hash / identifier normalization drift |
| Agent trigger rate | > 10% of cases | Investigate new evidence formats; widen deterministic parsers |
| NOT_APPLICABLE share | Moves > 10 pp from baseline | Audit intent classifier on a 50-case sample |
| Injection flags | Any new pattern | Manual review; add to adversarial suite |
| Free-quota consumption per run | > 70% of any provider's daily cap | `QuotaGuard` stops that role; switch it to the deterministic path; investigate cache misses |
| Cache hit rate on reruns | < 95% | Check cache keys (content hash, prompt version, model ID) |
| Workflow says COMPLETE, audit says INCOMPLETE | Rising trend | Business KPI: escalate to the owning system team |

**Calibration statement:** "The 10% agent-trigger threshold and the 4-step budget were set from the samples and the free-tier daily token cap. What would change them: an ablation showing the agent fixes errors on flagged cases outside the samples, or a larger confirmed quota that lets more flagged cases be investigated without risking the final run."

---

## 7. Cross-check: September feedback and post-mortem → October plan

| September item | October response |
|---|---|
| C1: invariant weakened 58 min before deadline | Invariants raise (ARCHITECTURE §8); feature freeze after Phase 7; reverts documented |
| C2: `main.py` overwrote values to silence a validator | Validator fails the run; status and steps come from one `CaseDecision` with a round-trip test |
| C3: rare field measured over all rows; target-ID convention never reverse-engineered | Slice metrics on non-trivial rows only (EVALUATION §2); Gate A conventions |
| H1: errors cascaded downstream | Typed stage outputs; per-stage unit tests; harness separates discovery errors from citation errors |
| H2: explanation from a different code path | Single renderer from `CaseDecision` |
| H3: no schema enforcement; fallback changed semantics; temperature dropped; cache path lost a field | Native JSON schema + Pydantic `extra="forbid"`; golden gate for fallback; temperature in config and in the call; cache stores the full validated object |
| H4: harness in scratch; empty evaluation `main.py` | Harness built in Phase 2, in pytest, shipped |
| M1: fixed pipeline, no bounded agent | `agent/loop.py` with closed tools, budget, traces, ablation |
| M2: dataset-overfit regex | No case-family shortcuts (also forbidden by AGENTS.md §6.7); rules read from `deletion_rules.csv` |
| M3: dead code; docstrings contradict code | Phase 8 removes unused code; judges count only observable behaviour |
| M4: coding compressed into final hours | Phases with Gates A–F and numeric acceptance; overrun cut order (§4.3) |
| L1: starter README; mixed/mislabelled usage | Own README; ledger-only usage with exact model IDs |
| L2: abstract monitoring answers | §6 table with thresholds and actions |
| Feedback: interfaces/schemas in prompts + tolerances | Prompt template (§4) pastes ARCHITECTURE blocks with an acceptance line and stop point |
| Feedback: spell out resilience | §3.1 quota rules and §5 rows on rate limits, quota exhaustion, timezone |
| Feedback: exact outputs, recompute downstream, final consistency check | Validator invariants 3–7 |

---

## 8. Final submission checklist

- [ ] `output.csv` at repo root: exact header, 518 rows, sorted, every `steps_json` parses, validator exits 0.
- [ ] `code.zip` contains: code, `config.yaml`, prompts as files, own `README.md` (setup, env vars, `python3 code/main.py`, `--no-llm`), pinned `requirements.txt`, `tests/`.
- [ ] `evaluation/` **and** `code/evaluation/` contain `evaluation_report.md` with: status accuracy + confusion matrix, step and field metrics, sequence metrics, slices, ≥3 custom cases with expected outputs, failures-and-fixes log, ablations.
- [ ] `usage_report.md`: providers, exact model IDs, calls, input/output tokens, total and average per request, per-model and overall totals — all measured. Actual cost shown as **$0 (free tier)**, plus a clearly labelled "equivalent paid-tier estimate" so the cost section is still complete.
- [ ] No billing enabled on any provider; README states the solution runs entirely on free tiers and local open-source tools.
- [ ] Bundled cache (if < 50 MB) so the run reproduces offline; README says so.
- [ ] No `dataset/`, `.venv`, `__pycache__`, `.env`, keys or nested zips. Run `grep -ri "sk-\|api_key="` over the zip contents.
- [ ] `log.txt` uploaded as `chat_transcript`: valid `tool=`/`model=` on every entry, no secrets, nothing rewritten.
- [ ] Submitted by 17:30 IST (deadline 18:00 IST). Camera and mic tested. Cheat sheet and harness numbers open during the interview.

---

## 9. Caveats

- Code, transcript and interview **sub-weights** come from HackerRank's write-up of the **May** edition; the Help Center says weights can vary by edition. Treat them as strong hints, not October rules.
- The October grader's per-dimension weights and partial-credit formula for step order are **not published**. Do not tune toward guesses.
- No write-ups exist yet for this deletion-audit problem; architectural lessons are transferred from the May and June editions.
- Dataset column names, the hash function and evidence-selection conventions were not verified before the event. Gate A exists to settle them.
