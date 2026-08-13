# Benchmark-Surveillance AI Assistant — Milestone Plan
### Adapting the TriGraphRAG blueprint to a regulated finance stack (Oracle + Gemini + Google ADK)

**Audience:** junior developers, guided milestone by milestone.
**Stack constraint:** Gemini models (Flash + Pro + `gemini-embedding` / `text-embedding-004`), Google Agent Development Kit (ADK), Oracle DB, Python (`oracledb` driver).
**Domain:** daily benchmark calculation → threshold-breach detection → two-stage review workflow with comments and attachments; heavy use of tables, FKs, stored procedures, and scheduled (cron/DBMS_SCHEDULER) jobs.

---

## 0. The one big adaptation from the original blueprint

The original blueprint assumed the knowledge lives in *documents* and must be extracted by an LLM. **Here, most Tier-1 knowledge already lives in Oracle as structured rows with foreign keys — which is already a graph.** So:

| Tier | Original blueprint | This project |
|---|---|---|
| **Tier 1 (mutable)** | Entities extracted from user docs by LLM | **Built mechanically from Oracle**: breaches, benchmarks, reviews, reviewers, comments, attachments — rows become nodes, FKs become edges. *No LLM extraction, no hallucination risk, near-zero cost.* LLM extraction is used **only** for the unstructured islands: comment text and attachment contents. |
| **Tier 2 (sources)** | Medical papers | Regulations (e.g., BMR/IOSCO-style texts), internal policies, benchmark methodology docs, SOPs, review procedures |
| **Tier 3 (vocabulary)** | UMLS | Internal **data dictionary** + financial/regulatory glossary: benchmark definitions, threshold definitions, review-state definitions, column/term meanings |

Everything else from the blueprint carries over: typed cross-tier edges (`REFERENCE_OF`, `DEFINITION_OF`), route-then-refine answering, citations that dereference to real nodes, config-driven domain packs, an eval harness before tuning, and the ≤ 4-LLM-calls-per-query guardrail.

**Gemini reality check (good news):** nothing in the blueprint needs a frontier model. Extraction, tagging, routing re-rank → Gemini Flash. Draft/refine answering → Gemini Pro. Embeddings → `gemini-embedding`. Gemini's long context is actually an advantage for stuffing serialized subgraphs into the refine pass. ADK gives you the agent/tool scaffolding for free, replacing the custom `query/` orchestration layer.

---

## 1. Non-negotiable guardrails (set these before writing any feature code)

Because this is a regulated environment, these are requirements, not suggestions:

1. **Read-only DB principal.** The AI stack connects with an Oracle account that has `SELECT` on an allow-listed set of views only. It can never call stored procedures that mutate state, never `INSERT/UPDATE/DELETE`. Reviews are *drafted* by AI, *submitted* by humans.
2. **No free-form SQL from the LLM in v1.** The agent calls **parameterized query tools** (you write the SQL; the model picks the tool and fills typed parameters). Free NL2SQL is a Milestone-6 experiment behind a validator, not a foundation.
3. **Full audit log.** Every prompt, tool call, SQL executed, and model response is persisted with timestamp, user, and model version. In a regulated shop, assume this log will be read by compliance one day.
4. **Data classification sign-off first.** Get written approval for which tables/columns may be sent to Gemini. Mask trader identifiers and any client data at the view layer if required (e.g., expose `TRADER_REF` surrogate keys, not names). Prefer Vertex AI endpoints with your org's data-governance controls over consumer API keys.
5. **Human-in-the-loop for anything that enters the review workflow.** AI output is a *draft with citations*; a named human owns the submission.
6. **Determinism where possible.** Threshold breach math stays in the existing stored procedures/cron jobs. The AI never recomputes breaches; it *explains* and *summarizes* them. Keep the system of record untouched.

---

## 2. Target architecture

```
                 ┌──────────────────────── ADK Root Agent (router) ───────────────────────┐
                 │                                                                        │
    ┌────────────▼───────────┐   ┌───────────────▼──────────────┐   ┌────────────────────▼─────────────┐
    │  StructuredDataAgent   │   │   GraphRAGAgent              │   │  ReviewDraftAgent                │
    │  (ADK tools = param-   │   │   (route → draft → refine    │   │  (breach evidence pack →         │
    │   eterized SQL views)  │   │    over doc graph, citations)│   │   draft summary w/ citations)    │
    └────────────┬───────────┘   └───────────────┬──────────────┘   └────────────────────┬─────────────┘
                 │ SELECT (read-only)            │ Cypher/vector                          │ both
        ┌────────▼─────────┐          ┌──────────▼─────────────────────────────────┐
        │  Oracle (system  │  sync    │  Graph store                               │
        │  of record)      ├─────────►│  Tier1: breaches/reviews/comments (from    │
        │  tables, FKs,    │  batch   │         Oracle metadata + rows)            │
        │  procs, cron     │          │  Tier2: regulations/policies/SOPs          │
        └──────────────────┘          │  Tier3: data dictionary + glossary         │
                                      └────────────────────────────────────────────┘
```

**Graph store choice:** Neo4j Community in a container is the default (matches the blueprint, best tooling for juniors). If infra won't approve new components, Oracle's own property-graph / vector features (23ai) are an acceptable fallback — keep `store/graph.py` as the only place that knows which one you chose, exactly as the blueprint's `GraphStore` protocol prescribes.

---

## 3. Milestones

Each milestone ends with a demo to the tech lead and a written acceptance check. Estimates assume 1–2 juniors part-time with weekly review.

---

### M0 — Foundations, access, and the golden set (Week 1)
*Goal: everything boring that blocks everything else.*

- Provision: read-only Oracle account + allow-listed views; Gemini/Vertex access; ADK project scaffold; Neo4j docker-compose; git repo using the blueprint's package layout (`config.py`, `llm/client.py`, `store/graph.py`, `eval/`).
- Wrap Gemini in the blueprint's single LLM client: `complete()` (JSON-mode via response schemas), `embed()`, retries, **per-call cost + token logging**.
- **Build the golden set (do this now, not later):** collect 30–50 real questions your users actually ask, e.g. *"Which breaches from yesterday are still pending second-stage review?"*, *"Why did benchmark X breach its threshold on 2026-08-05?"*, *"What does the policy say about late second reviews?"*, *"Which cron job populates BREACH_DAILY and when did it last run?"* — with expected answers/sources.
- Audit-log table + middleware from day one.

**Accept:** `cli.py ping` hits Oracle (read-only), Gemini, and the graph store; golden set reviewed by a business user; data-classification sign-off obtained.

---

### M1 — Schema graph: turn Oracle metadata into Tier-1 scaffolding (Weeks 2–3)
*Goal: the database explains itself. No LLM needed for most of this — great first task for a junior because it's deterministic and testable.*

- Extract from Oracle catalog views: `ALL_TABLES`, `ALL_TAB_COLUMNS`, `ALL_CONSTRAINTS`/`ALL_CONS_COLUMNS` (PK/FK), `ALL_VIEWS`, `ALL_PROCEDURES`/`ALL_ARGUMENTS`, `ALL_DEPENDENCIES` (proc→table lineage), `ALL_SCHEDULER_JOBS` or the cron inventory.
- Load as a graph: `(:Table)-[:HAS_COLUMN]->(:Column)`, `(:Table)-[:FK_TO]->(:Table)`, `(:Procedure)-[:READS|WRITES]->(:Table)`, `(:Job)-[:RUNS]->(:Procedure)` with schedules.
- Enrich each table/column node with a 1-line description: pull from `ALL_TAB_COMMENTS`/`ALL_COL_COMMENTS` where present; where absent, generate with Gemini Flash from name + sample column names, **flag as `ai_generated: true`**, and have a DBA/senior review the descriptions for the ~30 core tables. Embed descriptions → this becomes the seed of the Tier-3 data dictionary.

**Accept:** graph answers, via plain Cypher, questions like "what feeds table X," "which job writes breaches," "impact of dropping column Y" — validated against the DBA's knowledge on 10 spot checks. This deliverable is useful on its own even if the project stopped here.

---

### M2 — StructuredDataAgent: conversational answers over live Oracle data (Weeks 3–5)
*Goal: the first thing traders/reviewers can actually use.*

- Design 8–12 **parameterized SQL tools** as ADK `FunctionTool`s over the allow-listed views, covering the workflow: `get_breaches(date, benchmark?, status?)`, `get_review_status(breach_id)`, `get_pending_reviews(stage, reviewer?)`, `get_breach_history(benchmark, window)`, `get_comments(breach_id)`, `list_attachments(breach_id)`, `get_job_runs(job, window)`, `get_threshold_config(benchmark)`.
- Each tool: typed pydantic params, bounded result size, returns rows + column metadata. ADK agent (Gemini Flash for tool selection; Pro for final synthesis) composes tool calls and answers in prose with the underlying numbers.
- Add the M1 schema graph as a lookup tool so the agent can resolve "second review" → the actual status codes/columns (juniors will discover the real enum values live in some `REF_STATUS` table — model that).
- Guardrails: max rows per tool, query timeout, refusal message when no tool fits (never guess numbers).

**Accept:** ≥ 80% of the structured-data questions in the golden set answered correctly end-to-end; zero non-SELECT statements possible (verified by DB grants, not by prompt); latency < 10 s typical.

---

### M3 — Document graph: Tier 2 + Tier 3 per the blueprint (Weeks 5–7)
*Goal: the GraphRAG core, on the easy 80%.*

- Corpus: regulations, internal policies, benchmark methodology docs, review SOPs (start with 20–50 documents; get the authoritative versions from compliance, with version/date metadata on every node — regulated docs change, and citing a superseded policy is worse than citing nothing).
- Implement blueprint Phases 1–3 with the **finance domain pack**: entity types like `{benchmark, threshold_rule, obligation, control, review_stage, deadline, role, sanction, defined_term}`; tag categories like `{APPLIES_TO, OBLIGATION, DEADLINE, ESCALATION, ROLE, PENALTY}`.
- Static chunking first (skip semantic chunking entirely until the eval harness demands it). Gemini Flash extraction into the pydantic schema; `gemini-embedding` for vectors; Tier-3 glossary from the compliance glossary + M1 data dictionary; `REFERENCE_OF` / `DEFINITION_OF` linking via vector k-NN with config thresholds.
- Route → draft → refine responder with dereferenceable citations `{doc, version, section, snippet}`.

**Accept:** policy/regulation questions from the golden set: routing hit-rate ≥ 80% top-3, citation precision ≥ 0.8; every citation resolves to a real doc section a reviewer can open.

---

### M4 — The bridge: link breaches to policy, and mine comments/attachments (Weeks 7–9)
*Goal: the differentiating feature — evidence-based breach context. This is the paper's "triple" idea paying off.*

- **Cross-tier linking:** connect Tier-1 nodes to Tier-2/3 — `(:ThresholdConfig)-[:GOVERNED_BY]->(:PolicyClause)`, `(:Benchmark)-[:DEFINED_IN]->(:MethodologyDoc)`, `(:ReviewStage)-[:REQUIRED_BY]->(:Obligation)`. Seed the obvious links by embedding similarity (blueprint's linker), then have a senior/compliance person **approve links once**; approved links are curated data, not per-query LLM output.
- **Unstructured islands:** nightly batch (align with existing cron patterns) that ingests new review comments and attachments (PDF/Excel extraction) into per-breach mini-graphs — entities like `{root_cause, action, counterparty_ref, date}` extracted by Flash, `SUMMARIZES` summary per breach thread.
- New golden questions now answerable: *"For breach #123, what does policy require at second review and what did the first reviewer conclude?"* — answer stitches Oracle facts (M2 tools) + comment summary (this milestone) + policy citation (M3).

**Accept:** 10 real historical breaches walked end-to-end with a business reviewer; they confirm the linked policy clauses and comment summaries are correct; incremental nightly sync is idempotent (re-runs don't duplicate nodes — key everything by breach id + hash).

---

### M5 — ReviewDraftAgent: human-in-the-loop review assistant (Weeks 9–11)
*Goal: measurable time savings in the two-stage review workflow.*

- ADK multi-agent composition: root agent orchestrates StructuredDataAgent + GraphRAGAgent, then a drafting step (Gemini Pro) produces a **review draft pack** per breach: what happened (numbers from Oracle), why it likely breached (history + methodology), governing policy with citations, prior similar breaches and their outcomes, open questions for the reviewer.
- Output lands as a *draft* in a staging table or document — the human reviewer edits and submits through the existing workflow. The AI never touches review-state columns.
- Similar-breach retrieval: embed breach summaries; k-NN over historical breaches ("this looks like the March 12 breach on the same benchmark, closed as data-vendor error").
- UX: start with wherever reviewers already live — even a simple internal web page or chat surface is fine; do not build a big UI yet.

**Accept:** pilot with 2–3 reviewers for two weeks; measure minutes-per-review before/after and reviewer-rated draft usefulness ≥ 4/5; 100% of drafts carry dereferenceable citations; audit log captures every draft.

---

### M6 — Hardening, evaluation discipline, and careful expansion (Weeks 11+)
*Goal: production posture; only now consider the risky/fancy features.*

- Eval harness runs on every prompt/threshold change (blueprint §3.7 metrics + cost-per-query dashboards).
- Load/failure drills: Oracle unavailable, Gemini quota exhausted, graph store down → agent degrades gracefully with honest error messages.
- **Now** experiment, each behind a flag and judged by the harness: constrained NL2SQL (generated SQL passes an `EXPLAIN`-plan validator + allow-listed tables + auto-LIMIT before execution); semantic chunking; summary hierarchy for a growing corpus; proactive daily digest ("3 breaches pending second review > 48h, policy deadline is 72h") — read-only notifications first.
- Documentation pack: runbook, data-flow diagram for compliance, model-version pinning policy, prompt changelog.

**Accept:** compliance/security review passed; on-call runbook exists; golden-set scores tracked over ≥ 4 weeks without regression.

---

## 4. Milestone map to the original blueprint

| Blueprint phase | This plan |
|---|---|
| Phase 0 skeleton | M0 |
| Phase 1 minimal ingest+query | M2 (structured) + M3 (documents) — split because structured data is the faster win here |
| Phase 2 two-tier linking + citations | M3–M4 |
| Phase 3 domain packs | M3 (finance pack) |
| Phase 4 semantic chunking / re-rank | M6 (deferred until eval demands it) |
| Phase 5 scale & polish | M5–M6 |

## 5. Traps to warn the juniors about explicitly

1. **Don't let the LLM do arithmetic or breach logic.** Numbers come from SQL tools; the model narrates them. If the model states a number, it must appear verbatim in a tool result.
2. **Status codes lie in wait.** "Sent for review," "first review done," etc. are enum values in some reference table with history tables behind them — model the state machine in the schema graph early (M1), or every agent answer about workflow will be subtly wrong.
3. **Version everything from compliance.** A citation to policy v3 when v4 is in force is a real incident in this domain. Doc nodes carry `version, effective_date, superseded_by`.
4. **Attachments are messy** (scanned PDFs, Excel with merged cells). Budget real time in M4; extract what you can, and store "extraction failed, human must open" as an honest node property rather than silently skipping.
5. **Cost creep hides in ingest.** Log $ per document and per breach thread from M0; Gemini Flash for everything except the final draft/refine keeps this small.
6. **Keep the ≤ 4-LLM-calls-per-query rule** from the blueprint for the GraphRAG path; tool-calling paths get a max-tool-iterations cap (e.g., 6) so the agent can't loop.
7. **The demo that sells the project internally is M5's review draft pack** — keep M1–M4 scoped tightly so you get there.
