# Milestone 4 — Execution Pack
### The Bridge: breaches ↔ policy, and mining comments & attachments (Weeks 7–9)

**Open only after the M3 gate passes.** Uses the M0 context header plus:

> **M4 CONTEXT ADDITION:** This milestone connects the tiers and ingests the unstructured islands. New code in `app/bridge/` and `app/ingest/breach_ingest.py`. Rules: Tier-1 business nodes (breaches, reviews) are synced FROM Oracle read-only views — Oracle remains the system of record; the graph is a queryable mirror keyed by natural ids (`breach_id`) so re-sync is idempotent. The critical `GOVERNED_BY`/`DEFINED_IN` links between live data and policy are **seeded by similarity but activated only by one-time human approval** — an approved-links file in git is curated data. Comment/attachment extraction uses Flash with the finance pack; extraction failures are recorded honestly (`extraction_failed:true`), never silently skipped. The nightly delta job mirrors existing cron patterns.

## 1. Objectives

1. **O1 — Live Tier-1 mirror.** Breaches, reviews, comments-metadata, attachments-metadata for a configured date window synced into the graph, idempotently, linked to the M1 schema/state nodes.
2. **O2 — Approved cross-tier links.** ThresholdConfig→PolicyClause, Benchmark→MethodologyDoc, ReviewStage→Obligation links proposed by the linker, human-approved, loaded, and versioned in git.
3. **O3 — Comment & attachment mini-graphs.** Per-breach unstructured content extracted into typed entities + one summary per breach thread; failures honest.
4. **O4 — Nightly delta.** One scheduled command updates everything incrementally; re-runs safe.
5. **O5 — Stitched answers.** Questions combining live data + comments + policy ("for breach #123, what does policy require at second review and what did the first reviewer conclude?") answered with citations; 10 historical breaches validated by a business reviewer.

## 2. User input needed

| # | Item | Who |
|---|---|---|
| H1 | Allow-list the breach/review/comment/attachment views for sync; confirm columns pass classification (comments may contain names — decide masking) | DBA + compliance |
| H2 | **Link-approval session (~1h):** review the proposed cross-tier links CSV; approve/reject/correct each | Senior business + compliance |
| H3 | Choose the 10 historical breaches for validation (mix: data error, genuine breach, late review, attachment-heavy) | Business reviewer |
| H4 | Attachment reality check: list actual formats in the system (pdf? xlsx? msg? scans?) and pick the top-2 to support first | Ops |
| H5 | Approve the nightly job's schedule and host (align with existing cron windows) | Infra + DBA |

## 3. Build tasks with LLM prompts

### T1 — Breach/review sync

> [CONTEXT HEADER + M4 ADDITION]
> Task: `app/ingest/breach_ingest.py` + CLI `sync-breaches --since <date>`.
> Read allow-listed views (H1) via existing tools/OracleReadOnly; upsert `Breach{breach_id, business_date, benchmark, value, threshold, status_code, ...}`, `Review{review_id, breach_id, stage, reviewer_ref, status, completed_at}`, `CommentMeta{comment_id, breach_id, author_ref, created_at}` (text handled in T3), `AttachmentMeta{attachment_id, breach_id, filename, mime, size}`. Edges: `ON_BENCHMARK`→ M1 Benchmark-related nodes, `HAS_REVIEW`, `HAS_COMMENT`, `HAS_ATTACHMENT`, `IN_STATUS`→ M1 Status nodes (via H2(M2) mapping). Water-mark incremental: persist last-synced timestamp in a graph meta node; `--since` overrides. Deletions/corrections upstream → properties updated on re-sync (row hash compare), `last_synced` stamped.
> Tests: idempotent double-run; status edge correctness; watermark honored.

### T2 — Cross-tier link seeding + approval workflow

> [CONTEXT HEADER + M4 ADDITION]
> Task: `app/bridge/links.py`.
> 1) `propose()`: for each ThresholdConfig/Benchmark/ReviewStage node, vector_search against Tier-2 entities of compatible types (config map, e.g. threshold_rule↔ThresholdConfig) → candidates ≥ `settings.linking.bridge_threshold` (0.55) → write `proposed_links.csv` {src_key, dst_doc, dst_section, score, snippet, verdict:""}.
> 2) Human fills verdict ∈ {approve, reject, redirect:<node>} (H2 session).
> 3) `apply(csv)`: create `GOVERNED_BY`/`DEFINED_IN`/`REQUIRED_BY` edges ONLY for approved rows, with props `{approved_by, approved_on, score}`; the reviewed CSV is committed to `config/approved_links/` — it is versioned curated data. Re-proposing after new documents ingested only surfaces NEW candidates (dedupe against decided ones).
> Tests: only approved become edges; rejected never re-proposed; redirect honored.

### T3 — Comment & attachment mining

> [CONTEXT HEADER + M4 ADDITION]
> Task: `app/ingest/thread_ingest.py`.
> Comments: fetch text for CommentMeta nodes (allow-listed view), apply H1 masking, per comment one Flash extraction (finance pack + extra types: root_cause_hypothesis, action_taken, request, decision) into a per-breach mini-graph (`gid = breach:{id}`); then one summary per breach thread (`Summary{gid}` — same shape as M3 so the M3 responder can route to breach threads too).
> Attachments: dispatch by mime (H4 top-2 first): pdf → M3 loader; xlsx → openpyxl extract sheet names + used-range as markdown per sheet (cap cells, note truncation). Unknown/failed → node property `extraction_failed:true, reason` and a line in the report — NEVER silent. Extracted text → same per-breach mini-graph.
> Tests: masking applied before any prompt (assert); failed attachment honesty; mini-graph keys stable.

### T4 — Nightly delta job

> [CONTEXT HEADER + M4 ADDITION]
> Task: `app/cli.py` command `nightly` = sync-breaches (watermark) → thread_ingest for new/changed → linker delta → graph snapshot → emit `NightlyReport` (counts, cost, failures) to a file and stdout; non-zero exit on hard failure so the scheduler alerts. Include a crontab/DBMS_SCHEDULER example and a runbook section (what to do when it fails: it is safe to just re-run). Lock file to prevent overlapping runs.
> Tests: overlapping-run guard; partial failure of one breach doesn't abort the batch (per-item try/except, reported).

### T5 — Stitched answering

> [CONTEXT HEADER + M4 ADDITION]
> Task: `app/query/breach_context.py`: `explain_breach(breach_id) -> BreachContext` assembling IN CODE (deterministic, no LLM): live fields via M2 tools, review states + history, thread summary + key extracted entities, `GOVERNED_BY` policy clauses (current versions) with citation payloads, threshold config. Then ONE Pro call turns BreachContext into a concise narrative with `[n]` citations (reuse M3 citation assembly). Register as answer_fn for the `mixed` golden category; CLI `explain-breach <id>`.
> Tests: assembly with mocked stores; citation mapping; budget ≤ 2 LLM calls.

## 4. Gate metrics

| Metric | Target |
|---|---|
| Double-run `sync-breaches` and `nightly` | 0 duplicates, watermark advances |
| Link approval coverage: proposed links decided | 100% decided; approved edges match CSV exactly |
| Comment extraction: masking verified on sample of 20 | 0 leaks |
| Attachment honesty: failures visible in report | 100% of failures surfaced |
| 10 historical breach walkthroughs (H3) with business reviewer | ≥ 9/10 "correct and useful"; all citations open correctly |
| `mixed` golden category accuracy | ≥ 75% (first pass on hardest category) |
| Nightly runtime + cost | Within window/budget agreed at H5 |

## 5. Evaluator-LLM prompts

**Code review:** M0 §5.1 plus: "(7) Is Oracle still the sole system of record — find any path where graph state could be treated as authoritative over the DB; (8) masking before model — prove order of operations; (9) idempotency keys — construct a duplicate-node scenario."

**Breach-walkthrough audit (new):**
> You audit an AI-assembled breach explanation for a regulated review workflow. Input per case: the raw data (breach row, review rows, comment texts post-masking, policy sections) and the AI narrative with citations.
> Check: 1) every factual claim traceable to the raw data (quote both sides); 2) the cited policy clauses actually govern this breach type (or is the link over-broad?); 3) reviewer conclusions represented faithfully, not amplified or editorialized; 4) no causal claims beyond what comments state (hypothesis stated as hypothesis?); 5) masking holes.
> Output per case: claim-by-claim table, verdict FAITHFUL / MINOR ISSUES / MISREPRESENTS. Overall PASS requires zero MISREPRESENTS.

**Gate review:** M0 §5.3 structure with M4 metrics; artifacts = nightly reports ×3 consecutive days, walkthrough notes, link CSV history, audit verdicts.

## 6. Traps & open questions

1. **Comments contain people.** Masking (H1) is the milestone's compliance hot spot — test it before anything else in T3, and re-test after any prompt change.
2. **Link over-breadth:** a policy clause approved as governing "all benchmarks" will get cited for everything; prefer approving the narrowest correct clause. Put this instruction in the H2 session agenda.
3. **Attachment scope discipline:** H4's top-2 formats only; a junior can burn a week on .msg parsing. The `extraction_failed` path IS the feature for the rest.
4. *Question:* how far back to backfill (`--since`)? 3–6 months is usually enough for similar-breach retrieval in M5; full history is cost without benefit.
5. *Question:* do comment authors need attribution in summaries ("first reviewer concluded…") and does that conflict with masking? Get a ruling now — it shapes M5's draft pack wording.

**Exit:** gates green → open `15_milestone_5_execution_pack.md`.
