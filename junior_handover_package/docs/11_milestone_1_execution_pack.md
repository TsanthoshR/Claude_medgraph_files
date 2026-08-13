# Milestone 1 — Execution Pack
### Schema Graph: the Oracle catalog becomes Tier-1 scaffolding (Weeks 2–3)

**Open this pack only after the M0 gate returned GATE PASS.**
All builder prompts use the **context header from `10_milestone_0_execution_pack.md` §3.0**, plus this milestone addition:

> **M1 CONTEXT ADDITION:** M0 modules exist and must be reused, never duplicated: `Settings`, `LLMClient`, `OracleReadOnly.select()`, `make_graph_store()`, audit log. The graph backend is the in-memory implementation. New code lives in `app/schema_graph/` plus CLI commands. This milestone builds a graph OF the database's structure (tables, columns, FKs, procedures, jobs, statuses) — it does NOT load business row data (that's M4). Almost everything here is deterministic; the only LLM usage is generating missing descriptions, and every generated description is flagged `ai_generated: true`.

---

## 1. Objectives

1. **O1 — Catalog extracted.** Tables, columns, PK/FK constraints, views, procedures with arguments, proc→table dependencies, and scheduler jobs read from Oracle catalog views into typed pydantic records.
2. **O2 — Graph loaded, idempotently.** The catalog becomes Tier-1 nodes/edges via the GraphStore contract; re-running `schema-sync` on an unchanged DB changes nothing (counts stable).
3. **O3 — Every core object described.** The ~30 core tables and their columns carry descriptions (DB comments where present; Flash-generated + human-reviewed where absent) with embeddings — the seed of the Tier-3 data dictionary.
4. **O4 — Lineage answers work.** "What feeds table X?", "Which job writes breaches?", "Impact of dropping column Y?" answerable via typed traversal functions and CLI, validated by the DBA.
5. **O5 — The review state machine is modeled.** The real status codes, their reference table, and legal transitions exist as graph nodes/edges.

## 2. User input needed (humans only)

| # | Item | Who | Blocking for |
|---|---|---|---|
| H1 | Extend grants: the read-only account needs SELECT on the application schema's objects so `ALL_*` views expose them (or grants on specific `DBA_*` views if the DBA prefers); confirm which schema name(s) to scan | DBA | T1 |
| H2 | Name the **~30 core tables** for the benchmark/breach/review workflow (the catalog will return hundreds; we describe/embed core first) | Senior dev / DBA | T3 |
| H3 | Identify the **status reference tables** and history tables for the two-stage review workflow, and sketch the legal transitions on a whiteboard photo | Senior dev / business | T7 |
| H4 | 30–60 min DBA session to review AI-generated descriptions and spot-check 10 lineage answers | DBA | Gate |
| H5 | Confirm scheduler visibility: `ALL_SCHEDULER_JOBS` vs external cron — if OS cron exists, provide the crontab export | DBA/Infra | T1 |

## 3. Build tasks with LLM prompts

### T1 — Catalog reader

> [CONTEXT HEADER + M1 ADDITION]
> Task: Implement `app/schema_graph/catalog.py`.
> Read-only queries (via `OracleReadOnly.select`, purpose="schema_sync") against: `ALL_TABLES`, `ALL_TAB_COLUMNS`, `ALL_CONSTRAINTS` + `ALL_CONS_COLUMNS` (types P,R,U — resolve FK to referenced table/columns), `ALL_VIEWS`, `ALL_PROCEDURES` + `ALL_ARGUMENTS`, `ALL_DEPENDENCIES` (name→referenced_name for PROCEDURE/PACKAGE/TRIGGER vs TABLE/VIEW), `ALL_TAB_COMMENTS`, `ALL_COL_COMMENTS`, `ALL_SCHEDULER_JOBS` (+ `ALL_SCHEDULER_RUNNING_JOBS` optional). Filter by `settings.oracle.scan_schemas: list[str]` (add to config).
> Output: pydantic records `TableMeta, ColumnMeta, FKMeta, ProcMeta, DependencyMeta, JobMeta` with a `CatalogSnapshot` container and `snapshot_hash` (stable hash of sorted content) for change detection.
> Constraints: paginate large results with the existing max_rows mechanism; no string-formatted SQL — bind variables only; every query's purpose logged.
> Tests: fake `OracleReadOnly` returning canned rows; FK resolution correctness; hash stability under row order changes.

### T2 — Graph builder

> [CONTEXT HEADER + M1 ADDITION]
> Task: Implement `app/schema_graph/builder.py` mapping a `CatalogSnapshot` into the GraphStore.
> Nodes (label / key): `Table`/`schema.table`, `Column`/`schema.table.column`, `Procedure`/`schema.proc`, `Job`/`job_name`, `View`/`schema.view`. Edges: `HAS_COLUMN`, `FK_TO` (props: constraint_name, columns), `READS`/`WRITES` (derive direction: dependencies give reference only — mark all as `DEPENDS_ON` with `access:"unknown"` unless the dependency type or proc source clarifies; do NOT guess), `RUNS` (Job→Procedure via job_action parsing — best effort, flag `parsed:false` when job_action isn't a recognizable proc call), `IMPLEMENTS` (View→Table via dependencies).
> Idempotency: everything through `upsert_nodes/upsert_edges`; deleting obsolete objects = mark `active:false` (soft delete), never remove — history matters here.
> Provide `build(snapshot) -> BuildReport` (counts created/updated/deactivated).
> Tests via InMemoryGraphStore: re-run same snapshot → zero changes; object disappears → deactivated not deleted; FK edge endpoints exist.

### T3 — Descriptions & dictionary seed

> [CONTEXT HEADER + M1 ADDITION]
> Task: Implement `app/schema_graph/describe.py`.
> For nodes in `settings.schema_graph.core_tables` (new config, from H2) and their columns: if a DB comment exists → use it verbatim with `source:"db_comment"`. Else generate with Flash: prompt includes table name, column names/types, 0 rows of data (NEVER sample real rows — H3 data-classification applies), producing ≤ 25-word description; store with `source:"ai", ai_generated:true, reviewed:false`.
> Then `embed()` name+description for every described node and store on the node.
> CLI `schema-describe --review-export out.csv` exports all ai_generated descriptions for the DBA (H4); `--review-import reviewed.csv` applies edits and sets `reviewed:true`.
> Tests: db-comment precedence; no row data in any prompt (assert on prompt content); import round-trip.

### T4 — Lineage & impact queries

> [CONTEXT HEADER + M1 ADDITION]
> Task: Implement `app/schema_graph/lineage.py` with typed functions over the GraphStore (no raw queries):
> `what_feeds(table) -> LineageReport` (procs/jobs writing or depending, upstream tables via FK direction), `what_reads(table)`, `impact_of(column|table) -> ImpactReport` (dependent views/procs/FK children, depth-limited, cycle-safe), `job_chain(job)` (job→proc→tables), `find_object(fuzzy_name)` using description embeddings (vector_search) for "which table holds breach results?" style lookup.
> CLI: `schema lineage <table>`, `schema impact <object>`, `schema find "<question>"` with readable table output.
> Tests: cycle in FK graph doesn't hang; depth limit respected; find_object returns core table for a paraphrased description.

### T5 — schema-sync orchestration

> [CONTEXT HEADER + M1 ADDITION]
> Task: Implement `app/schema_graph/sync.py` + CLI `schema-sync`: read catalog → compare `snapshot_hash` to last stored (persist last hash in graph meta node) → if changed, build → describe (only new/changed objects) → print BuildReport + cost. `--force` bypasses hash check. Wire `graph-snapshot` save at the end. Add a `docs/runbook_m1.md` (generated) describing the weekly sync habit.
> Tests: unchanged hash short-circuits without LLM calls (assert zero llm audit rows).

### T6 — Status/state-machine modeling

> [CONTEXT HEADER + M1 ADDITION]
> Task: Implement `app/schema_graph/states.py`. Input: `settings.schema_graph.status_tables` (from H3: e.g. `{table:"REF_STATUS", code_col:"CODE", name_col:"DESCRIPTION"}` list) and a transitions file `config/state_transitions.yaml` (hand-written from the H3 whiteboard: list of `{from,to,trigger}` — mark file `human_authored: true`).
> Read status rows (these small reference tables ARE approved data per H3 confirmation — assert table is in the allow-list), create `Status` nodes and `TRANSITIONS_TO` edges from the YAML, link `Status` nodes to the `Column` node holding them (`VALUE_OF`).
> Output: `state_machine_report` CLI showing the machine as an ASCII/mermaid diagram to paste into PRs; compare YAML transitions against distinct transitions observed in the history table IF one is allow-listed (report-only: "observed transition PENDING_R2→CLOSED_AUTO not in YAML").
> Tests: report flags observed-but-undeclared transitions; refuses non-allow-listed tables.

## 4. Gate metrics

| Objective | Metric | Target |
|---|---|---|
| O1/O2 | `schema-sync` twice in a row → second run: 0 created / 0 updated / 0 LLM calls | Pass |
| O2 | Node/edge counts vs catalog counts (tables, columns, FKs) | Exact match |
| O3 | Core tables+columns described; ai_generated ones exported, reviewed, re-imported | 100% of H2 list; `reviewed:true` ≥ 90% |
| O4 | DBA spot-check of lineage/impact answers | ≥ 9/10 confirmed |
| O4 | `schema find` on 10 paraphrased descriptions of core tables | ≥ 8/10 top-1 |
| O5 | State machine report matches H3 whiteboard; undeclared observed transitions triaged | Pass |
| Cost | Total Gemini spend for M1 | Within agreed budget (descriptions only — expect small) |
| Hygiene | Zero row-data in prompts (test-asserted); all grep-proof rules from M0 still 0 violations | Pass |

## 5. Evaluator-LLM prompts

**Per-task code review:** reuse M0 §5.1 verbatim, adding to its checklist: "(7) Determinism — is any graph structure derived from an LLM output? Structure must come only from catalog data; LLM contributes descriptions only. (8) Row-data leakage — could any real table row reach a prompt?"

**Schema-graph correctness review (new):**
> You are auditing a graph built from an Oracle catalog for a regulated finance system. Inputs: (a) the raw catalog extracts (CSV per catalog view), (b) node/edge count summary and 30 sampled edges with properties, (c) the state-machine YAML and observed-transitions report.
> Verify: 1) counts reconcile (each FK constraint ↔ one FK_TO edge; each column ↔ one HAS_COLUMN); 2) no edge invents direction the catalog can't support (DEPENDS_ON with access:"unknown" is the honest form — flag any READS/WRITES lacking justification); 3) sampled edges' properties match the raw extract exactly; 4) every ai_generated description in the sample is plausible for its name/type and none exceeds 25 words or leaks data values; 5) the state machine: transitions in YAML but never observed (possible dead paths) and observed but undeclared (modeling gap) — list both.
> Output: reconciliation table, discrepancy list with severity, PASS/FAIL.

**Gate review:** reuse M0 §5.3 structure with M1 objectives/metrics pasted; artifacts = sync report ×2, DBA spot-check notes, review-import evidence, correctness-review verdict.

## 6. Traps & open questions

1. **`ALL_DEPENDENCIES` doesn't give read vs write.** Resist inferring; `DEPENDS_ON access:"unknown"` is correct until someone parses proc source (a possible M6 enhancement). Wrong lineage direction is worse than admitted ignorance.
2. **Hundreds of non-core tables** will flood the graph — load them all as bare nodes (cheap, useful for FK completeness) but describe/embed only core. Decide with the tech lead whether to hide non-core from `schema find` results.
3. **Job actions** are often anonymous PL/SQL blocks — `parsed:false` is fine; the report should count them so H5 follow-up can cover them manually.
4. *Question:* are there multiple environments (DEV/UAT/PROD-replica)? The snapshot should record which; comparing schema drift across environments is a free win to note for M6.
5. *Question:* any tables with row-level security or restricted columns the catalog exposes but H3 sign-off excludes? If yes, add them to a `describe_blocklist`.

**Exit:** gate metrics green + evaluator PASS + tech-lead sign-off → open `12_milestone_2_execution_pack.md`.
