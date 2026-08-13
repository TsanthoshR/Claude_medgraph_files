# Milestone 5 — Execution Pack
### ReviewDraftAgent & the Router: multi-agent composition + human pilot (Weeks 9–11)

**Open only after the M4 gate passes.** Uses the M0 context header plus:

> **M5 CONTEXT ADDITION:** This milestone composes the existing pieces into the product. Agent roster is CAPPED: Router + StructuredDataAgent (M2) + GraphRAGAgent (M3 responder wrapped as an agent) + ReviewDraftAgent (orchestration in code around M4's `explain_breach`). New capabilities are tools/functions, not new agents. The Router is a classifier with constrained output, not an open-ended delegator; `mixed` questions route to deterministic orchestration. A fourth route, ESCALATE (honest refusal + handoff), is first-class. Every draft output is a DRAFT: it lands in a staging location, a named human edits and submits through the existing workflow, and the AI never writes to review-state columns. All agent model calls flow through the M2 audit callbacks.

## 1. Objectives

1. **O1 — Router live and measured.** Constrained classification into {structured_data, policy_docs, review_draft, mixed, escalate}; ≥ 95% accuracy on the golden set's category labels.
2. **O2 — ReviewDraftAgent produces the draft pack.** Per breach: what happened (numbers), likely cause (history + thread, hypothesis-framed), governing policy with citations, similar past breaches with outcomes, open questions for the reviewer.
3. **O3 — Similar-breach retrieval.** Embedding k-NN over historical breach summaries with outcome labels.
4. **O4 — Pilot executed.** 2–3 reviewers, two weeks, real breaches; drafts staged, edited, submitted by humans; measured.
5. **O5 — Full-loop audit.** One `session_id` traces a user question through router → agents → tools → answer in the audit log.

## 2. User input needed

| # | Item | Who |
|---|---|---|
| H1 | Pick the pilot surface (simple internal web page / existing chat tool / CLI for power users) — smallest thing reviewers will actually touch | Tech lead + reviewers |
| H2 | Nominate 2–3 pilot reviewers and get their managers' time commitment; baseline their current minutes-per-review for a week BEFORE the pilot | Management |
| H3 | Approve the staging location for drafts (table or doc store) and confirm zero write-path to workflow tables | DBA + compliance |
| H4 | Define outcome labels for historical breaches (closed-data-error, closed-genuine, escalated, …) and confirm they're derivable from status history | Business |
| H5 | Pilot comms: one paragraph to reviewers setting expectations (drafts may be wrong; you own the submission; feedback via 👍/👎+comment) | Tech lead |
| H6 | Sign-off that the draft pack template wording (incl. reviewer attribution per M4 open question 5) is acceptable | Compliance |

## 3. Build tasks with LLM prompts

### T1 — Router

> [CONTEXT HEADER + M5 ADDITION]
> Task: `app/agents/router.py`.
> One Flash call, schema `{route: Literal["structured_data","policy_docs","review_draft","mixed","escalate"], confidence: float, breach_id: int|null, rationale: str(≤20 words)}`. Prompt (versioned): route definitions with 3 examples each, drawn from golden categories; instructions: extract breach_id when present; below `settings.router.min_confidence` (0.7) → escalate; NEVER answer the question itself. Dispatch map: structured_data→M2 agent, policy_docs→M3 responder, review_draft (with breach_id)→T2, mixed→T3, escalate→T4. `route_and_answer(question) -> AnswerResult` with route recorded in audit (session-correlated).
> Tests: mocked model; dispatch correctness; missing breach_id on review_draft → asks one clarifying question rather than guessing.

### T2 — ReviewDraftAgent (deterministic orchestration)

> [CONTEXT HEADER + M5 ADDITION]
> Task: `app/agents/review_draft.py`.
> `draft_review(breach_id) -> DraftPack`: IN CODE, fixed order: M4 `explain_breach` context → T5 similar breaches → assemble `DraftPack{sections: what_happened, likely_cause(hypothesis-framed, from thread evidence only), policy_requirements(with citations), similar_breaches(id, similarity, outcome, one-liner), open_questions, data_freshness, generated_at, model_versions}` — one Pro call per narrative section OR one structured call for all (choose, justify, stay ≤ 4 calls). Render to markdown with a mandatory header: "AI-GENERATED DRAFT — review, edit, and submit under your own name. Citations link to source versions." Write to the H3 staging location; NEVER to workflow tables (no such connection string may even exist in this module — assert in test).
> Tests: section completeness on mocked context; header always present; call budget.

### T3 — Mixed-question orchestrator & T4 — Escalation

> [CONTEXT HEADER + M5 ADDITION]
> Task: `app/agents/mixed.py` + `app/agents/escalate.py`.
> Mixed: decompose IN CODE by simple rule — if breach_id present use breach context + policy responder; else run structured agent and doc responder sequentially and compose with one Pro call that must keep citations intact and attribute which numbers came from which source. Escalate: fixed template — what the system understood, why it can't answer (out of scope / low confidence / missing data), 2 suggested rephrasings, and the human contact from `settings.escalation.contact`; log route=escalate with reason. No model creativity in escalation wording.
> Tests: citation survival through composition; escalate template exact.

### T5 — Similar-breach retrieval

> [CONTEXT HEADER + M5 ADDITION]
> Task: `app/bridge/similar.py`.
> Build: for each historical breach with a thread summary (M4), embed `summary + benchmark + breach magnitude bucket`; store on the Breach node. Outcome label from status history per H4 mapping. `find_similar(breach_id, k=5) -> [{breach_id, score, outcome, business_date, one_liner}]` excluding the breach itself and same-day siblings (they share cause trivially). Backfill CLI. Tests: exclusion rules; empty-history behavior (returns [], draft pack says "no comparable history" honestly).

### T6 — Pilot surface + feedback capture

> [CONTEXT HEADER + M5 ADDITION]
> Task: minimal surface per H1 (default: single-page internal web app — one input box, answer with citations rendered as expandable panels, a "Draft review for breach #" button, 👍/👎 + comment per response persisted to the audit db with session_id). Auth: reuse whatever internal SSO/header the org standard is (ask; else IP-restricted + username header). NO admin features, NO settings UI. Also `eval pilot-report`: weekly aggregation — usage, routes, thumbs, cost, latency, escalation rate.
> Tests: feedback rows persisted; report math.

### T7 — Full-loop trace check

Manual: run 5 real questions end-to-end; for each, pull the audit rows by session_id and confirm the complete chain (router verdict → agent calls → tool calls/SQL → answer hash). Paste one full trace into the PR as evidence.

## 4. Gate metrics

| Metric | Target |
|---|---|
| Router accuracy vs golden categories | ≥ 95%; confusion matrix reported |
| Escalation behavior on 10 out-of-scope questions | 10/10 template, 0 fabrications |
| Draft pack completeness (all sections, header, citations resolve) | 100% of pilot drafts |
| Pilot: reviewer usefulness rating | Mean ≥ 4/5 |
| Pilot: minutes-per-review vs H2 baseline | Reported (target: meaningful reduction; honesty over spin) |
| Pilot: drafts materially wrong (reviewer-flagged) | Each triaged to root cause; zero uncited wrong claims |
| Full-loop audit trace | 5/5 complete chains |
| Cost per draft pack / per question | Reported; within budget |

## 5. Evaluator-LLM prompts

**Code review:** M0 §5.1 plus: "(7) prove no write-path to workflow tables exists in any module (search connection usage); (8) router constrained output — can any path bypass the Literal schema?; (9) does escalation ever leak a partial guess?"

**Draft-pack quality rubric (new):**
> You are a senior benchmark-surveillance reviewer scoring an AI draft review pack. Input: the pack + the underlying raw context (same bundle the M4 walkthrough audit used) + the policy sections cited.
> Score 1–5 each with one-sentence justification: factual accuracy (every number/state correct), cause framing (hypotheses framed as hypotheses, evidence-linked), policy relevance (cited clauses genuinely govern this case, current versions), similar-breach usefulness (would these help a reviewer?), open questions (sharp, non-generic), overall would-I-start-from-this-draft.
> Hard fails regardless of scores: any uncited factual claim; any instruction-like text telling the reviewer what to decide (drafts inform, never decide); missing AI-draft header. Output: score table, hard-fail check, verdict USABLE / NEEDS WORK / UNSAFE.

**Router audit (new):**
> Given the router prompt, 40 questions with golden categories, and the router's outputs: produce the confusion matrix; for each error, diagnose (ambiguous question / prompt gap / missing example) and propose the minimal prompt fix; identify any question where 'escalate' would have been more correct than the golden label itself. Verdict PASS ≥95% / FAIL with fix list.

**Gate review:** M0 §5.3 structure + pilot report + rubric verdicts + trace evidence. The gate question that matters most here: **would the pilot reviewers keep using it if we stopped asking them to?** Get their answer verbatim.

## 6. Traps & open questions

1. **The pilot is a social exercise wearing a technical costume.** A mediocre draft delivered reliably inside the reviewer's existing habit beats a brilliant one behind a new login. H1's answer matters more than any prompt.
2. **Don't tune during the pilot week.** Freeze prompts/models for each pilot week; changes between weeks only, logged. Otherwise the measurement means nothing.
3. **Similar-breach cold start:** if M4 backfill was short, similarity will be thin — the honest "no comparable history" line protects trust; resist padding with weak matches.
4. *Question:* should draft packs auto-generate for every new breach (push) or on-demand (pull)? Start pull; push is an M6 digest decision.
5. *Question:* reviewer edits of drafts are gold for improvement — is capturing the diff (staged draft vs submitted review) permissible? Ask compliance now for M6.

**Exit:** gates green → open `16_milestone_6_execution_pack.md`.
