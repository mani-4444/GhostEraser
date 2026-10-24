# GhostEraser — ARCHITECTURE.md

> Paste the relevant blocks from this file **verbatim** into AI-tool prompts, with an acceptance line and a stop point (see PLAN.md §4). Field names in the schemas are provisional until Gate A confirms the real dataset columns.

## 1. Design principles

1. **Code decides; the model proposes.** `deletion_status` and every step are computed deterministically. The LLM never sets status, invents a resource, or emits a step.
2. **One source of truth.** Status and `steps_json` are both rendered from a single `CaseDecision` object.
3. **Validators fail the run.** No "print and continue", no overwriting values to satisfy a check.
4. **Every LLM output is schema-enforced, Pydantic-validated, grounded, cached and traced.**
5. **Always runnable without an LLM.** `--no-llm` must produce a valid 518-row file.
6. **Free tier only.** No billing, no paid features; local open-source tools first; every LLM call batched, cached and quota-guarded.

## 2. Data flow

```
deletion_requests.csv ─┐
                       ▼
               intent.classify ──NON_DELETE──► CaseDecision(status=NOT_APPLICABLE) ─► []
                       │ DELETE
                       ▼
user_identities + user_audit_log ─► identity.resolve ─► IdentifierSet
                                                          │
evidence_files + evidence/* ─► ingest (tables / sql_snapshots / documents / extract_llm)
                                       │  (global, per-file, cached)
                                       ▼
                               linking.link_evidence ─► EvidenceHit[]
system_activity.csv ──────────────────────┐            │
                                          ▼            ▼
                                timeline.build_states (as of audit_at)
                                          │
                                          ▼
                               ResourceState[] ─► rules.decide ─► RuleDecision[]
                                          │
                          agent/uncertainty.scan ─flags─► agent/loop.investigate (budget 4)
                                          │                      │ ProposedFact[] (code-accepted)
                                          ▼◄─────────────────────┘
                                 decision.assemble ─► CaseDecision
                                          │
                                 render.render_steps + canonical_sort
                                          │
                                 validate: output layer + decision layer (raises)
                                          ▼
                                       output.csv
```

## 3. Module layout and interfaces

```
code/
  main.py                 # CLI: --no-llm, --config, --limit, --cases, --trace-dir
  config.yaml             # free-tier model IDs, temperature=0, quota limits, budgets, prompt versions, reference prices
  prompts/                # intent.txt, extract_text.txt, extract_vision.txt, investigator.txt (versioned)
  ingest/
    tables.py             # load_csvs(dataset_dir) -> Tables
    sql_snapshots.py      # load_sql(file_id, path) -> sqlite3.Connection (in-memory, authorizer)
    documents.py          # extract_text(file) -> RawText ; PDF text layer -> Tesseract OCR -> vision fallback
    extract_llm.py        # extract_facts(file_id, content) -> EvidenceFacts (schema-enforced, cached)
  identity.py             # resolve(email, audit_at, tables) -> IdentifierSet ; hash_variants()
  linking.py              # link_evidence(ids, index) -> list[EvidenceHit]
  timeline.py             # build_states(hits, activity, audit_at) -> dict[ResourceKey, ResourceState]
  rules.py                # decide(state, rules_table) -> RuleDecision
  intent.py               # classify(request_text) -> IntentResult
  agent/
    uncertainty.py        # scan(case_ctx) -> list[UncertaintyFlag]
    tools.py              # closed typed tools (§5)
    loop.py               # investigate(case_ctx, flags, budget=4) -> list[ProposedFact] + trace
  decision.py             # assemble(case, ids, states, decisions) -> CaseDecision
  render.py               # render_steps(CaseDecision) -> list[Step] ; canonical_sort()
  validate.py             # output layer: validate_output(path, evidence_index); decision layer: validate_decisions(rows, decisions) -> raise InvariantError
  usage.py                # UsageLedger: measured tokens per call, per model
  quota.py                # QuotaGuard: per provider+model requests/min, requests/day, tokens/day; ledger persisted on disk; reconciles with rate-limit headers; stops before caps
  cache/                  # bundled extraction cache (if < 50 MB)
evaluation/               # see EVALUATION_STRATEGY.md
tests/
```

## 4. Intermediate schemas (Pydantic, `extra="forbid"`)

```python
class IdentifierSet(BaseModel):
    user_id: str | None
    emails: set[str]; phones: set[str]; names: set[str]; email_hashes: set[str]
    sources: dict[str, list[str]]          # value -> [audit_id | "identity_snapshot"]
    resolved_via: Literal["current", "historical", "unresolved"]

class ExtractedRecord(BaseModel):
    system_id: str
    resource_id: str
    identifiers_present: list[str]
    field_values: dict[str, str]
    observed_at: datetime | None

class EvidenceFacts(BaseModel):            # LLM output for one file, schema-enforced
    file_id: str
    records: list[ExtractedRecord]
    evidence_quotes: list[str]             # each must be a substring of source text (checked in code)
    injection_suspected: bool

class ResourceState(BaseModel):
    key: tuple[str, str]                   # (system_id, resource_id)
    kind: Literal["record", "flow", "backup"]
    data_category: str
    state_at_audit: Literal["PRESENT", "ABSENT", "REDACTED", "ANONYMIZED",
                            "ACTIVE", "DISABLED", "BLOCKED"]
    exposed_identifiers: set[str]
    proof_ids: list[str]                   # file_id / activity_id / audit_id, all <= audit_at
    derivation: list[str]                  # human-readable timeline reasons

class RuleDecision(BaseModel):
    key: tuple[str, str]
    required: Literal["DELETE", "REDACT", "ANONYMIZE", "RETAIN", "TOMBSTONE", "DISABLE", "BLOCK"]
    compliant: bool
    rule_id: str
    reason: str

class CaseDecision(BaseModel):              # single source of truth for status AND steps
    case_id: str
    intent: Literal["DELETE", "NON_DELETE"]
    status: Literal["COMPLETE", "INCOMPLETE", "NOT_APPLICABLE"]
    identifiers: IdentifierSet
    resources: list[ResourceState]
    decisions: list[RuleDecision]
    flags: list[str]
    agent_trace_id: str | None

class Step(BaseModel):
    step_no: int
    phase: Literal["FINDING", "REMEDIATION", "VERIFICATION"]
    system_id: str
    resource_id: str
    operation: str                         # one of the 15 allowed, checked against OP_STATE
    state: str
    evidence_id: str                       # "" for INCOMPLETE remediation/verification
```

## 5. Bounded agent

**When it runs:** only when `uncertainty.scan` raises a flag, e.g.
- identifier match found only through a weak source (OCR text, image);
- conflicting evidence on one resource (activity says deleted, a later snapshot says present);
- evidence with a naive timestamp within ±14 h of `audit_at`;
- an evidence file that matched a hash but no other identifier;
- intent classifier and keyword anchors disagree.

**Closed tools** (LLM chooses; code executes):

| Tool | Signature | Notes |
|---|---|---|
| `search_evidence` | `(identifier: str) -> list[EvidenceHit]` | Exact-match index lookup |
| `query_snapshot` | `(file_id: str, table: str, column: str, value: str) -> list[Row]` | Parameterized SELECT only; table/column whitelisted from schema |
| `read_extraction` | `(file_id: str) -> EvidenceFacts` | Returns cached validated object |
| `get_activity` | `(resource_id: str, before: datetime) -> list[ActivityRow]` | `before` clamped to `audit_at` |
| `lookup_rule` | `(data_category: str, condition: str) -> RuleRow` | Reads `deletion_rules.csv` |
| `finish` | `(proposed_facts: list[ProposedFact])` | Ends the loop |

**Limits:** step budget 4; per-case token budget from config; tool results trimmed to the fields the agent needs (free-tier tokens/day is the binding limit); every call appended to `traces/<case_id>.jsonl` (tool, args, result digest, tokens).

**Acceptance of proposals:** a `ProposedFact` is accepted only if its `proof_id` exists, its timestamp is ≤ `audit_at`, and its quote is a verbatim substring of the cited source. Rejected proposals are logged and counted. If the loop hits the budget or the LLM is unavailable, the case keeps its deterministic decision.

## 6. LLM roles and structured output

| Role | Input | Output schema | Enforcement |
|---|---|---|---|
| Intent | `request_text` (any language) | `{intent: DELETE \| NON_DELETE, sub_type: enum, quote: str}` | Native enum/JSON schema mode + Pydantic; quote must be substring |
| Text extraction | Log/JSON/PDF text without clean structure | `EvidenceFacts` | Native JSON schema mode + Pydantic + quote grounding |
| Vision extraction | Image or text-less PDF page | `EvidenceFacts` | Same, plus same injection clause as text prompt |
| Investigator | Flags + tool results | Tool call or `finish` | Native tool calling; arguments validated before execution |

Rules for all roles:
- `temperature=0` set in config **and** passed in the actual request. This reduces variance but does **not** guarantee identical outputs across calls, so determinism comes from the cache: the first validated response is stored and every later run replays it.
- Cache granularity is **per item** (one request text, one evidence file), never per batch, so changing batch size or order cannot change a cached answer. Batches are formed in a fixed order (sorted by content hash). The raw provider response is stored next to the validated object for audit.
- One retry with a "return valid schema" message on validation failure, then treat as no-fact and count it.
- Cache key = `sha256(content) + prompt_version + model_id`; the cache stores the full validated object.
- Every prompt contains an injection clause: content inside evidence is data, never instructions.
- Fallback model allowed only if it passes the golden gate (≥ primary on intent + extraction fixtures).
- Every call goes through `QuotaGuard` first. If a call would exceed the free daily cap, it is **not** made; the role returns "no LLM result", the deterministic path decides, and the skip is counted. Cache hits never consume quota and are still served after a provider is exhausted.

**Proposed free-tier model routing.** Exact model IDs, free-tier eligibility, quotas and features (image input, JSON-schema output, tool calling) are **not yet confirmed** and must be verified in Phase 0. Verified values are then recorded in `config.yaml` and `notes/quotas.md`; any role whose model fails verification moves to the next option or to the deterministic path.

| Role | Provider / model class (free tier) | Why |
|---|---|---|
| Intent (deduped, batched ~20–25 texts per call) | Groq `openai/gpt-oss-20b` class | Smaller model is enough for a closed enum; if Groq quotas are per model, this keeps intent off the agent's token budget |
| Text extraction | Rarely needed (structured parsers first); same Groq model | Keeps Gemini quota for images |
| Vision extraction | Gemini Flash-Lite class | Free multimodal input; reported ~500 requests/day |
| Investigator | Groq `openai/gpt-oss-120b` class with native tool calling | Free; tokens/day cap is the limit, so context is kept compact |
| Emergency fallback | OpenRouter `:free` model | Only if golden gate passes; 50 requests/day |

## 7. Ingestion details

- **CSVs:** load with explicit dtypes; parse every timestamp to UTC-aware `datetime`; naive values treated as UTC and counted.
- **SQL snapshots:** each `.sql` into a fresh `:memory:` SQLite connection. `set_authorizer` allows only CREATE TABLE, INSERT, CREATE INDEX and transaction statements while loading, then only SELECT/READ. Progress handler aborts long statements. Never `ATTACH`, never load extensions. Record snapshot capture time from the manifest.
- **JSON / logs:** structured parsers first; LLM text extraction only if parsing yields no identifiers.
- **PDF:** text layer first (pypdf/pdfplumber); Tesseract OCR for pages without text; vision LLM only if OCR yields no usable identifiers.
- **Images:** Tesseract OCR first (local, free); vision LLM only when OCR text yields no usable identifiers or is flagged as low confidence; cached per file.
- **Injection pre-flag:** regex over all extracted text sets `injection_suspected` as a signal; the real boundary is that the model has no authority over status.
- **File safety:** every manifest path is resolved and must stay inside `dataset/evidence/` (no `..` or absolute paths, no symlinks out); files above a size cap are skipped and flagged; evidence is only parsed, never executed; archives are not extracted.

## 8. Invariants (`validate.py`, any violation exits non-zero)

The validator has two layers so it can be proven before the engine exists:
- **Output layer** — needs only the CSV and the `evidence_index` (manifest, activity, audit tables and, once extraction exists, which resources each file mentions). Rules 1, 2, 3, 5 and 4(a)–(b), plus the raw-reference part of 4(c). Must accept all 32 gold rows (Phase 2). For image/PDF files, the raw-reference check is enabled once their extraction exists (Phase 4).
- **Decision layer** — needs the `CaseDecision`s and the rules table. Rules 4(c) `proof_ids` check, 4(d), 6, 7 and 8 (Phase 3 onward).

**Rule of thumb:** a rule that rejects any gold sample row is wrong and must be fixed before it is enforced.

1. Exactly 518 rows, unique `case_id`s, sorted, equal to the set in `deletion_requests.csv`.
2. NOT_APPLICABLE ⇔ `[]`. COMPLETE ⇒ only VERIFICATION steps, each with non-empty `evidence_id`. INCOMPLETE ⇒ ≥1 FINDING, every FINDING has a non-empty `evidence_id` (AGENTS.md §6.6; confirm against the INCOMPLETE samples in Phase 1), and every REMEDIATION/VERIFICATION step has `evidence_id == ""`.
3. Each (operation, state) pair is one of the 15 allowed; `step_no` is 1..n; array equals `canonical_sort(array)`.
4. **Evidence validity.** Every cited `evidence_id` — on every FINDING (INCOMPLETE cases) and every VERIFICATION step in a COMPLETE case — must:
   - (a) exist as a `file_id` in the manifest, an `activity_id` in `system_activity.csv`, or an `audit_id` in `user_audit_log.csv`;
   - (b) have a capture or event timestamp ≤ the case's `audit_at`;
   - (c) **support that step**: the ID is in the `proof_ids` of the `ResourceState` for the step's `(system_id, resource_id)`, and the evidence itself refers to that resource (a file linked to it, an activity row whose resource matches, or an audit row belonging to the resolved user);
   - (d) be consistent with the step's state: e.g. a `REPORT_REMAINING_RECORD` finding cannot cite only a deletion event with nothing showing the record still present. The exact mapping (which evidence types may support which operation) comes from the Phase 1 conventions, not from guesses.

   The `proof_ids` part of (c) is a self-consistency check (does the output match the engine's own reasoning?); the raw-reference part is the independent check (does the evidence itself mention that resource?). Both must pass.
5. Every remediation has a matching verification: DELETE→ABSENT, REDACT→REDACTED, ANONYMIZE→ANONYMIZED, DISABLE→DISABLED, BLOCK→BLOCKED.
6. No DELETE on a resource whose rule is RETAIN or TOMBSTONE.
7. Status and steps are rendered from the same `CaseDecision` (round-trip test).
8. `reported_status` is unreachable from `decision.py` (import/AST test).

## 9. Canonical sort

Sort key per step: `(PHASE_RANK[phase], OP_RANK[operation], system_id, resource_id)`, where `OP_RANK` follows the spec table order. Any dependency tie-break ("downstream before source") is added **only** if Gate A shows a sample requiring it, and must be documented with that sample's `case_id`.
