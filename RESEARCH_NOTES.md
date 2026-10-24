# GhostEraser — RESEARCH_NOTES.md

Official rules, winner evidence and domain research behind GhostEraser's design.

Labels: **CONFIRMED** = stated in an official source for the cited edition. **HYPOTHESIS** = reasonable inference, not stated for October. **UNKNOWN** = not published; do not tune toward it.

## 1. Official HackerRank rules and judging

| Claim | Label | Source |
|---|---|---|
| Four signals, each 0–100; final score = weighted sum: Code 30%, Output 30%, Interview 30%, Transcript 10%. Weights and rules can vary by edition. | CONFIRMED | [HackerRank Help Center — HackerRank Orchestrate](https://help.hackerrank.com/articles/3728112689-hackerrank-orchestrate) |
| Start Oct 24, 2026 12:30 UTC; live evaluation Oct 26–27; results Oct 27; 30-min voice AI interview with camera + mic after submission; ≤ 50 MB per file. | CONFIRMED | Help Center; [October contest page](https://www.hackerrank.com/hackerrank-orchestrate-october26) |
| End time: Oct 25, 2026, 18:00 IST (12:30 UTC) | CONFIRMED (challenge email, as confirmed by the team) | Matches the earlier unofficial listing; [AGENTS.md](https://github.com/interviewstreet/hackerrank-orchestrate-october26/blob/main/AGENTS.md) leaves it unconfigured |
| Code rubric sub-weights: Agent architecture 30% ("is this an agent, or a hardcoded workflow with LLM calls?"), Prompt & tool craft 30%, Agent robustness 25% (guardrails, retries, iteration caps, output validation), Engineering rigor 15% | CONFIRMED for May 2026; HYPOTHESIS for October | [Behind the Scenes of HackerRank Orchestrate](https://www.hackerrank.com/blog/behind-the-scenes-of-hackerrank-orchestrate/) |
| Judges reward only what is observable in source code; README claims and intent comments do not count | CONFIRMED (May) | Same |
| Output: per-row accuracy vs golden set, plus a submission-level safety score for adversarial inputs | CONFIRMED (May); Help Center mentions adversarial safety generally | Same; Help Center |
| October output scoring considers status, identity resolution, discovered records/risks, cited evidence, policy decisions, completeness and order of `steps_json` | CONFIRMED (dimensions only; weights UNKNOWN) | [problem_statement.md](https://github.com/interviewstreet/hackerrank-orchestrate-october26/blob/main/problem_statement.md) |
| Transcript rubric: architecture ownership 35%, technical specificity 25%, iteration & verification 25%, safety/edge cases 15% | CONFIRMED (May) | Behind the Scenes blog |
| Interview rubric: technical depth & ownership 40%, problem understanding 25%, communication 20%, honesty 15% | CONFIRMED (May) | Behind the Scenes blog; Help Center lists the same topics |
| Tie-break: Interview → Code → Output → Transcript | CONFIRMED (May) | Behind the Scenes blog |
| Chat transcript = root `log.txt` written by the AI tool; each entry needs exact `tool=` and `model=` | CONFIRMED | [AGENTS.md](https://github.com/interviewstreet/hackerrank-orchestrate-october26/blob/main/AGENTS.md) §2, §5 |
| No case-family shortcuts / hard-coding | CONFIRMED | AGENTS.md §6.7 |
| Hidden grader formulas (partial credit for order, per-dimension weights) | UNKNOWN | — |

## 2. What the evidence says about winners (May/June editions)

Source for all bullets unless noted: [Behind the Scenes of HackerRank Orchestrate](https://www.hackerrank.com/blog/behind-the-scenes-of-hackerrank-orchestrate/) and [Getting better at Orchestrate](https://www.hackerrank.com/blog/getting-better-at-orchestrate/).

- **Architecture mattered.** "Single agent with tools or RAG" produced rank 1 and most of the Top 10. "Deterministic rule pipeline / no explicit agent" had no Top‑10 entries. "Single prompt / single model call" had no Top‑50 entries. → A deterministic engine alone is penalized on the code signal; the agent loop must be real and visible.
- **Balance mattered.** Sorting by output accuracy alone would have recovered only a minority of the actual Top 50, and no pair of signals was strongly rank-correlated. → Don't over-invest in output at the expense of interview and transcript.
- **Interview behaviour.** Longer, specific answers that name a mechanism scored higher; very short answers and answers narrating the AI tool scored lower.
- **May #1 approach (as described by HackerRank):** one agent, one read-only search/read tool; injection check before the model; a citation only counts if the agent actually opened that file; cheap rules for trivial cases, expensive model only when needed.
- **"Getting better at Orchestrate" advice:** trace one input through the system; validate inputs before the model call and outputs after it; reject unsupported labels; retry malformed responses within a limit; run on the full dataset, not only cases that worked; in the transcript, name files, functions, schemas and thresholds.
- **Gap:** no write-ups exist yet for the October deletion-audit problem. These lessons are transferred, not proven for this task.

## 3. Domain research (informs design; `deletion_rules.csv` is authoritative)

| Topic | Finding | Design consequence | Source |
|---|---|---|---|
| Erasure from backups | UK ICO guidance: if backup data cannot be erased immediately, put it "beyond use" and ensure it is not restored into live systems | `REPORT_RESTORE_RISK` → `BLOCK_RESTORE` → `VERIFY_RESTORE_BLOCKED` | [DPN — Right to erasure](https://dpnetwork.org.uk/right-to-erasure/); [myEDPO — RTBF from backups](https://www.myedpo.com/post/2018/08/21/right-to-erasure-rtbf-from-backups) |
| Regulator focus on Art. 17 | EDPB's coordinated enforcement on the right to erasure flagged backup deletion and weak anonymisation as recurring problems | Good interview framing for why restore risk and anonymisation checks matter | Reported in research output; **primary EDPB link not captured — verify before quoting** |
| Hashes are still personal data | Pseudonymised data remains personal data; FTC: hashing does not make data anonymous | Treat `email_hash` as a direct identifier (as the spec also does) | [EDPB pseudonymisation guidelines consultation](https://www.edpb.europa.eu/system/files/documents/webform/public_consultation_reply/20250228-edpb-guidelines-on-pseudonymisation-mdt-public-consultation.pdf); [FTC — No, hashing still doesn't make your data anonymous](https://www.ftc.gov/policy/advocacy-research/tech-at-ftc/2024/07/no-hashing-still-doesnt-make-your-data-anonymous) |
| Prompt injection | OWASP LLM01: segregate and label untrusted content; enforce privileges outside the model; pattern filters do not reliably catch indirect injection | Regex is a signal, not a boundary; the LLM has no authority over status | [OWASP LLM01:2025](https://genai.owasp.org/llmrisk/llm01-prompt-injection/); [OWASP cheat sheet](https://cheatsheetseries.owasp.org/cheatsheets/LLM_Prompt_Injection_Prevention_Cheat_Sheet.html) |
| Untrusted SQL | SQLite recommends an authorizer to restrict statements and a progress handler to abort long-running ones | `set_authorizer` + progress handler on `:memory:` connections; no ATTACH, no extensions | [SQLite — Defense Against the Dark Arts](https://www.sqlite.org/security.html); [Python `set_authorizer` example](https://www.zetcode.com/python/sqlite3-connection-set-authorizer/) |
| Structured outputs | OpenAI strict `json_schema` guarantees schema adherence; Gemini supports `response_mime_type=application/json` with a response schema and an enum mode for classification | Use native schema mode; still validate values with Pydantic | [OpenAI Structured Outputs](https://developers.openai.com/api/docs/guides/structured-outputs); [Firebase AI Logic — structured output](https://firebase.google.com/docs/ai-logic/generate-structured-output); [Google Cloud — control generated output](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/capabilities/control-generated-output) |

## 4. Free-tier LLM options (checked Oct 24, 2026; verify in Phase 0)

Free quotas are often unpublished or change without notice. All figures below are **HYPOTHESIS until confirmed** in each provider's console during Phase 0.

| Provider | Finding | Label | Source |
|---|---|---|---|
| Google Gemini API | Free tier covers Flash and Flash-Lite classes only; Pro models are paid-only. Google publishes no fixed free-tier limits; developers reported ~20 requests/day for 3.x Flash and ~500/day for 3.x Flash-Lite in Sep 2026. Quotas are per project and reset at midnight Pacific time | Reported (Sep 2026) | [agentdeals — Gemini API pricing changes](https://agentdeals.dev/gemini-api-pricing-changes); [agentdeals — Gemini API pricing 2026](https://agentdeals.dev/gemini-api-pricing-2026) |
| Google Gemini API | An earlier (May 2026) write-up cited far higher Flash limits (~1,500/day), showing how fast these numbers move | Reported (May 2026), likely outdated | [TinkerLLM — Gemini free tier limits](https://tinkerllm.com/blog/gemini-api-free-tier-limits-rate-quotas/) |
| Google Gemini API | Free-tier content may be used to improve Google products and reviewed by humans; Google advises not sending sensitive data on the free tier | Reported | [agentdeals — Gemini API pricing changes](https://agentdeals.dev/gemini-api-pricing-changes) |
| Groq | Free plan lists `openai/gpt-oss-120b`, `gpt-oss-20b`, Llama and Qwen models at ~30 requests/min and ~1,000 requests/day; limits apply per organization | Reported (Jun 2026) | [eesel — Groq pricing 2026](https://eesel.ai/blog/groq-pricing) |
| Groq | `gpt-oss-120b`: 30 requests/min, 1,000/day, ~200K tokens/day | Reported (Aug 2026) | [Novita — Free LLM API comparison 2026](https://blogs.novita.ai/free-llm-api-comparison-2026/) |
| Groq | Limits are listed separately for each model, which suggests each model has its own quota; daily reset time not stated | HYPOTHESIS — verify in the Groq console in Phase 0 | [eesel — Groq pricing 2026](https://eesel.ai/blog/groq-pricing) |
| OpenRouter | `:free` models: 20 requests/min; 50 requests/day with < 10 credits purchased, 1,000/day with ≥ 10 credits | CONFIRMED (official docs) | [OpenRouter — API limits](https://openrouter.ai/docs/api/reference/limits) |
| Image input / JSON-schema output on free tiers | Not covered by any source found | UNKNOWN — verify with a test call in Phase 0 | — |

Design consequence: local OCR and parsers first, batched intent, per-file cached extraction, a `QuotaGuard` that stops before caps, and only one full-dataset LLM run.

## 5. Starter-repo discrepancies to remember

- Evaluation report path: README → `code/evaluation/`; AGENTS.md and spec → `evaluation/`. Ship both.
- AGENTS.md submission URL typo (`wwww.`). Use the link from the challenge email.
- Repo: [interviewstreet/hackerrank-orchestrate-october26](https://github.com/interviewstreet/hackerrank-orchestrate-october26)
