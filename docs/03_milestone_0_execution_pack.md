# Milestone 0 — Execution Pack
### Foundations, Access, and the Golden Set (Week 1)

This pack is self-contained: a junior should be able to complete M0 using only this document, an LLM coding assistant (Gemini), and the humans listed in §2. The next milestone's pack will be issued only after the M0 gate review passes.

---

## 1. Objectives

By the end of M0:

1. **O1 — Working skeleton.** A git repo with the agreed package layout, running `python -m cli ping` successfully against all three backends: Oracle (read-only, via the named TNS connection), Gemini, and the graph store (in-memory backend for now; Neo4j swaps in later with zero caller changes).
2. **O2 — One LLM gateway.** Every model call in the project goes through a single client (`llm/client.py`) that provides JSON-schema-constrained completion, embeddings, retries, and per-call cost/token logging. No other file imports the Gemini SDK.
3. **O3 — Audit trail from day one.** Every LLM call and every SQL execution is persisted to an audit log with timestamp, user, inputs, outputs, model version, and cost.
4. **O4 — Golden set v1.** 30–50 real questions with expected answers/sources, collected from business users, stored in a versioned, schema-validated file, loadable by the eval harness skeleton.
5. **O5 — Governance cleared.** Written data-classification sign-off; read-only DB account verified as unable to write; secrets management in place (no credentials in code or git history).

**Explicitly out of scope for M0:** any schema-graph building (M1), any SQL tools (M2), any document ingestion (M3), any agent logic. Resist the urge — M0 is plumbing and governance.

---

## 2. User input needed — humans must do these; an LLM cannot

Hand this checklist to the tech lead / juniors on day 1. Builder prompts in §3 assume these are done or in flight.

| # | Item | Who | Blocking for |
|---|---|---|---|
| H1 | Create read-only Oracle account; grant `SELECT` only on an initial allow-list of views (even 2–3 views is enough for M0). Connection is via a **named TNS alias**: provide the `tnsnames.ora` entry (and wallet files if the connection uses one) and the directory to use as `TNS_ADMIN`. ⚠️ A *SQLcl saved connection* is not the same thing — SQLcl stores those in its own encrypted store which Python cannot read; extract the underlying TNS entry/connect descriptor (`show tns` / the connect string) and put it in a `tnsnames.ora` the app can reach | DBA + junior | T3 |
| H2 | Verify H1 truly cannot write: attempt an `INSERT`/`UPDATE`/`EXECUTE` from that account and capture the ORA- error as evidence | DBA + junior | Gate review |
| H3 | Data-classification sign-off: written list of tables/columns approved to be sent to Gemini; masking rules for trader/client identifiers | Compliance / data owner | T2 usage, M1+ |
| H4 | Provision Gemini access — decide **Vertex AI vs AI Studio key** (Vertex strongly preferred in a regulated org: org policies, data-residency, no data used for training); create project, enable APIs, obtain service account | Cloud admin | T2 |
| H5 | **Deferred — Neo4j is not available yet.** M0 uses an in-memory graph store; no approval needed now. Instead: start the approval process for Neo4j (or decide the Oracle 23ai property-graph fallback) so it lands before the M1/M2 boundary, and record the expected date | Infra/security | M1+ swap |
| H6 | Secrets mechanism decision: env vars via `.env` (dev) + the org's vault for anything shared; confirm what's allowed | Security | T1 |
| H7 | **Golden set collection**: 1-hour session with 2–3 business users (reviewers/traders/ops). Capture 30–50 real questions verbatim, plus for each: the correct answer (or where it lives), and which system/doc is the source of truth. An LLM can *format* this afterwards; only humans can *source* it | Tech lead + business users | T7, all future eval |
| H8 | Python version and internal package-index constraints; proxy/egress rules from the dev host to Google APIs | Infra | T1, T2 |
| H9 | Name a product owner who will re-review the golden set and later score answers | Management | M2+ |

---

## 3. Build tasks with LLM prompts

### 3.0 Context header — prepend to EVERY builder prompt

> **PROJECT CONTEXT (do not deviate):**
> We are building the foundations of an AI assistant for a regulated finance benchmark-surveillance workflow. Stack: Python 3.11, Oracle DB accessed via the `oracledb` package with a READ-ONLY account using a **named TNS alias** (tnsnames.ora under a configured `TNS_ADMIN` directory), Google Gemini via Vertex AI (`google-genai` SDK), a **`GraphStore` abstraction whose only implementation for now is in-memory** (Neo4j arrives in a later milestone; the official `neo4j` driver may be added as an optional dependency but no code may require a running Neo4j), `pydantic` v2 and `pydantic-settings` for all config and schemas, `typer` for the CLI, `pytest` for tests.
> **Hard rules:** (1) No secrets in code — all credentials from environment variables loaded through the central `Settings` class. (2) No SQL outside `store/oracle.py`; no Cypher outside `store/graph.py`; no Gemini SDK imports outside `llm/client.py`. (3) Every LLM call and SQL execution must be recorded through `audit/log.py`. (4) Only `SELECT` statements may ever be sent to Oracle; enforce this in code even though the account is read-only. (5) All LLM outputs we parse must be constrained by a pydantic schema. (6) Every module gets type hints, docstrings, and pytest unit tests with external services mocked.
> Package layout (create only what your task needs):
> `app/config.py · app/llm/client.py · app/store/oracle.py · app/store/graph.py · app/audit/log.py · app/eval/golden.py · app/eval/harness.py · app/cli.py · tests/`
> Output format: complete file contents per file, then a short "how to run" note. Ask me clarifying questions ONLY if a hard rule would otherwise be violated; otherwise choose sensible defaults and state your assumptions at the top.

---

### T1 — Repo scaffold + configuration

**Builder prompt:**
> [CONTEXT HEADER]
> Task: Create the project scaffold and central configuration.
> Deliver: (a) `pyproject.toml` with the dependencies listed in the context and dev deps (pytest, ruff); (b) `app/config.py` containing a `Settings(BaseSettings)` class with nested groups: `oracle` (tns_alias: str, tns_admin: DirectoryPath — the folder containing tnsnames.ora and any wallet, user, password, allowlisted_views: list[str], thick_mode: bool = False with instant-client path if True), `gemini` (project, location, flash_model default "gemini-2.0-flash", pro_model, embed_model, max_output_tokens, temperature), `graph` (backend: Literal["memory","neo4j"] = "memory", persist_path: Path | None for the memory backend's snapshot file, plus optional uri/user/password used only when backend="neo4j"), `audit` (db_path default "./audit.sqlite"), `run` (env: dev/prod, user_tag) — all secrets sourced from env vars with prefix `BSA_`, `.env` supported in dev; (c) `.env.example` with every variable documented and dummy values; (d) `.gitignore` covering `.env`, audit db, caches; (e) `README.md` with 10-line quickstart; (f) `tests/test_config.py` proving env vars override defaults and that instantiating Settings never logs or prints secret values; (g) a `Makefile` or `justfile` with `install`, `lint`, `test`, `ping` targets.
> Acceptance: `pytest` passes with no network access; `ruff check` clean.

### T2 — Gemini LLM client (the single gateway)

**Builder prompt:**
> [CONTEXT HEADER]
> Task: Implement `app/llm/client.py`, the ONLY module allowed to import the Gemini SDK.
> Requirements:
> 1. `class LLMClient` constructed from `Settings`, exposing exactly three methods:
>    `complete(prompt: str, *, system: str | None, schema: type[BaseModel] | None, model_role: Literal["flash","pro"], purpose: str) -> str | BaseModel` — when `schema` is given, use Gemini's JSON/response-schema mode and validate into the pydantic model; on validation failure retry ONCE with the validation error appended to the prompt; then raise `LLMParseError`.
>    `embed(texts: list[str], *, purpose: str) -> list[list[float]]` — batched, preserving order.
>    `ping() -> dict` — a 1-token completion and a 1-text embed; returns models used and latency.
> 2. Retries with exponential backoff + jitter on 429/5xx (max 3), honoring a per-minute request budget from config.
> 3. **Cost & audit:** after every call compute tokens in/out (from the API's usage metadata) and estimated cost from a `PRICES` dict in this file (document that prices must be reviewed monthly), then record via `audit.log.record_llm_call(purpose=..., model=..., tokens_in=..., tokens_out=..., cost_usd=..., latency_ms=..., ok=...)`. Never log prompt contents at INFO level; full prompts/responses go only into the audit store.
> 4. `tests/test_llm_client.py`: mock the SDK; cover schema success, one-retry-then-fail parse, backoff on 429, audit record emitted per call, embed order preservation.
> Do not implement any agent or business logic.

### T3 — Oracle read-only connector

**Builder prompt:**
> [CONTEXT HEADER]
> Task: Implement `app/store/oracle.py`.
> Requirements:
> 1. `class OracleReadOnly` with a connection pool from `Settings`, connecting by **named TNS alias**: `oracledb.create_pool(user=..., password=..., dsn=settings.oracle.tns_alias, config_dir=settings.oracle.tns_admin)` (thin mode reads tnsnames.ora from `config_dir`; also set the `TNS_ADMIN` env var for belt-and-braces). If `thick_mode` is true, call `oracledb.init_oracle_client()` once at startup. On connect failure, the error message must say which alias and which config_dir were used and list the aliases actually found in tnsnames.ora — juniors will lose hours to a mis-set TNS_ADMIN otherwise. Method `select(sql: str, params: dict, *, purpose: str, max_rows: int = 500) -> QueryResult` where `QueryResult` is a pydantic model holding `columns: list[str]`, `rows: list[tuple]`, `truncated: bool`, `elapsed_ms: int`.
> 2. **Defense in depth:** before execution, reject with `UnsafeSQLError` any statement that (a) does not start with SELECT/WITH after stripping comments/whitespace, (b) contains a semicolon, or (c) references any object not in `settings.oracle.allowlisted_views` (parse identifiers naively via regex and document the limitation — the DB grants are the real enforcement; this check is a tripwire).
> 3. Statement timeout (e.g., `call_timeout`) from config; always apply `FETCH FIRST :max_rows ROWS ONLY` if not present.
> 4. Every execution recorded via `audit.log.record_sql(purpose, sql, params, rowcount, elapsed_ms, ok, error)`.
> 5. `ping()` runs `SELECT 1 FROM DUAL` and `SELECT view_name FROM user_views` (or queries one allow-listed view) and returns server version + reachable views.
> 6. Tests with a fake connection object: unsafe-SQL rejection cases (INSERT, `;--` injection, non-allowlisted table), truncation flag, audit emission.

### T4 — Graph store: protocol + in-memory implementation (Neo4j-ready)

*Neo4j is not available in this environment yet. We build the abstraction plus a fully working in-memory backend now; when Neo4j arrives, only a config change and one new class should be needed. The swap guarantee comes from a shared contract test suite, not from good intentions.*

**Builder prompt:**
> [CONTEXT HEADER]
> Task: Implement `app/store/graph.py` with a swappable graph backend.
> Requirements:
> 1. A `GraphStore` Protocol defining the contract all backends must satisfy:
>    `upsert_nodes(label: str, records: list[dict], key: str) -> int` (idempotent by `key`; re-upserting the same key updates properties, never duplicates);
>    `upsert_edges(rel_type: str, pairs: list[tuple[node_key, node_key]], src_label: str, dst_label: str, props: dict | None) -> int` (idempotent; edges to missing nodes raise `NodeNotFound`);
>    `get_node(label, key) -> dict | None`; `neighbors(label, key, rel_type: str | None, direction: Literal["out","in","both"], depth: int = 1) -> list[dict]`;
>    `vector_search(label: str, embedding: list[float], k: int, min_score: float = 0.0) -> list[tuple[dict, float]]` over nodes that carry an `embedding` property (cosine similarity);
>    `count(label=None)`, `clear()`, `ping() -> dict`, `snapshot()/load()` for persistence.
>    NOTE: no `run_read(cypher)` in the protocol — free-form Cypher would make the in-memory backend unswappable. All graph access goes through these typed methods; if a future milestone genuinely needs Cypher, it will be added to the Neo4j class as an explicitly non-portable extension.
> 2. `InMemoryGraphStore` implementing the FULL protocol: dict-of-dicts for nodes keyed by (label, key), adjacency lists for edges, brute-force cosine for `vector_search` (numpy; fine for ≤ ~50k nodes — document this limit), optional JSON-lines snapshot to `settings.graph.persist_path` so a junior's built graph survives restarts (`snapshot()` on demand + atexit; `load()` at startup when the file exists; embeddings included).
> 3. `Neo4jGraphStore` as a stub: same class surface, every method raising `NotImplementedError("arrives with Neo4j infra")`, each with a docstring specifying the exact Cypher pattern it will use (e.g., upsert_nodes → `MERGE` on the key property; vector_search → Neo4j vector index). The docstrings ARE a deliverable — they lock the mapping now.
> 4. `def make_graph_store(settings) -> GraphStore` factory switching on `settings.graph.backend`. Callers never import a concrete class.
> 5. **Contract tests** (`tests/test_graphstore_contract.py`): a pytest test class parameterized over backends, currently running against `InMemoryGraphStore` only, covering idempotent upserts, neighbor traversal both directions, depth-2 traversal, vector_search ordering + min_score, missing-node errors, snapshot round-trip. When Neo4j lands, adding it to the parameter list re-validates the entire contract with zero new test code — state this in a comment.
> 6. `ping()` for the memory backend returns `{"backend": "memory", "nodes": n, "edges": m, "persist_path": ...}`.

### T5 — Audit log

**Builder prompt:**
> [CONTEXT HEADER]
> Task: Implement `app/audit/log.py`.
> Requirements:
> 1. SQLite-backed (path from config) with WAL mode; two tables: `llm_calls(ts, user_tag, purpose, model, prompt, response, tokens_in, tokens_out, cost_usd, latency_ms, ok, error)` and `sql_calls(ts, user_tag, purpose, sql, params_json, rowcount, elapsed_ms, ok, error)`. Auto-migrate on first use.
> 2. Public functions `record_llm_call(...)`, `record_sql(...)`, and `summary(since: datetime) -> AuditSummary` returning call counts, total cost, error counts.
> 3. Writes must never crash the caller: on audit failure, log ERROR and continue (document this trade-off: availability over audit completeness in dev; note that prod should flip to fail-closed and make it a config flag `audit.fail_closed`).
> 4. Tests: records round-trip, summary math, fail-open behavior, concurrent writes (threads) don't corrupt.

### T6 — CLI

**Builder prompt:**
> [CONTEXT HEADER]
> Task: Implement `app/cli.py` with `typer`.
> Commands:
> `ping` — runs Oracle, Gemini, and graph-store pings; prints a table of component / status / detail / latency (graph row shows `backend=memory` until Neo4j lands); exit code 0 only if all pass; `--skip` option per component. Also `graph-snapshot` and `graph-load` commands for the memory backend's persistence file. 
> `audit-summary --since 24h` — prints the audit summary.
> `golden validate <path>` — validates a golden-set file against the schema from T7 and prints counts by category.
> Each command must work when only SOME backends are configured (helpful errors, not stack traces). Include tests using typer's CliRunner with all backends mocked.

### T7 — Golden set schema + eval harness skeleton

**Builder prompt:**
> [CONTEXT HEADER]
> Task: Implement `app/eval/golden.py` and `app/eval/harness.py` (skeleton only — no answering system exists yet).
> Requirements:
> 1. Pydantic models: `GoldenItem(id, question, category: Literal["structured_data","policy_docs","workflow","lineage","mixed"], expected_answer: str, expected_sources: list[str], answer_type: Literal["numeric","list","narrative","yes_no"], provided_by, added_on, notes)` and `GoldenSet(version, items)` with validators: unique ids, non-empty expected_answer, ≥1 expected_source unless category=="mixed" with notes.
> 2. Loader for a YAML file `golden/golden_v1.yaml`; include a filled-in EXAMPLE file with 6 realistic finance-surveillance items (2 per major category) clearly marked as placeholders to be replaced by the human-collected set.
> 3. `harness.py`: `run(golden: GoldenSet, answer_fn: Callable[[str], AnswerResult]) -> EvalReport` where `AnswerResult(answer, citations, cost_usd, latency_ms)`; for M0, wire a `DummyAnswerer` that returns "not implemented" so the report machinery (per-category counts, cost totals, JSON + markdown report writers) can be tested end-to-end now.
> 4. Leave clearly-marked TODO hooks where the LLM-judge scoring will plug in (M2).
> 5. Tests for validators, loader, and report generation.

### T8 — CI

**Builder prompt:**
> [CONTEXT HEADER]
> Task: Add CI (GitHub Actions or the org's equivalent — produce GitHub Actions YAML and note what to change for GitLab).
> Pipeline: ruff lint → pytest (unit only, no integration marks) → `pip-audit` dependency scan → a secret-scan step (gitleaks) → build artifact of the eval report from the DummyAnswerer run as a smoke test. Cache pip. Fail the build on any step failure. Python version from H8 (parameterize).

---

## 4. Evaluation metrics for the M0 gate

M0 is plumbing, so metrics are largely binary checks plus a few numbers:

| Objective | Metric | Target |
|---|---|---|
| O1 | `cli ping` all-green on the dev host | Pass |
| O1 | Unit test pass rate / lint | 100% pass, ruff clean |
| O2 | Grep proof: Gemini SDK imported only in `llm/client.py`; `oracledb` only in `store/oracle.py`; graph access only via the `GraphStore` protocol (no concrete backend class imported outside `store/graph.py`) | 0 violations |
| O2 | Every `complete()`/`embed()` call produces an audit row with cost | Verified by test + manual spot check |
| O3 | Audit summary shows ≥1 LLM call and ≥1 SQL call after ping; rows contain model version and cost | Pass |
| O4 | Golden set size and coverage | ≥30 items; ≥5 per category; validated by `golden validate`; reviewed by product owner (H9) |
| O5 | Write-attempt evidence from read-only account (H2) captured in repo `docs/evidence/` | ORA- error screenshot/log present |
| O5 | Secret scan of full git history | 0 findings |
| Hygiene | Unsafe-SQL tripwire test cases | All rejected |
| Cost | Total Gemini spend during M0 | < agreed budget (should be cents) |

---

## 5. Evaluator-LLM prompts

Use a **separate LLM session** (ideally Gemini Pro with no access to the builder conversation) so the evaluator isn't anchored on the builder's reasoning.

### 5.1 Per-task code review prompt

> You are a senior Python reviewer for a REGULATED finance project. Review the following code against the task specification and hard rules I will paste below. The stakes: this code will handle pre-release financial data and must produce a defensible audit trail.
> Evaluate in this order and output a markdown report with these exact sections:
> 1. **Spec compliance** — table of each numbered requirement → met / partially / missing, with file+line evidence.
> 2. **Hard-rule violations** — secrets in code, SDK imports outside the gateway module, SQL/Cypher outside store modules, LLM calls that bypass the audit log, non-SELECT statements possible. Any violation is an automatic FAIL.
> 3. **Security & data handling** — injection routes, prompt contents leaking into INFO logs, credentials in error messages, PII in audit rows beyond what H3 sign-off allows.
> 4. **Failure behavior** — what happens when Oracle (bad TNS alias / wrong TNS_ADMIN), Gemini, or the graph persistence file is unavailable or corrupt; are errors actionable?
> 5. **Test adequacy** — do tests cover the failure paths, not just happy paths? List untested requirements.
> 6. **Verdict** — PASS / FAIL with a prioritized fix list (max 10 items, each with concrete code-level suggestion).
> Be adversarial: actively try to construct an input that breaks a hard rule (e.g., craft a SQL string that slips past the tripwire) and show it if you find one.
> [PASTE: context header + task prompt + full code]

### 5.2 Golden-set quality review prompt

> You are evaluating a "golden set" of Q&A pairs that will be the permanent yardstick for an AI assistant in a benchmark-surveillance workflow (threshold breaches, two-stage reviews, comments, attachments, Oracle-based data, policy documents).
> For the set below, report:
> 1. **Coverage matrix** — count of items per category (structured_data / policy_docs / workflow / lineage / mixed) and per answer_type; flag any category below 5 items.
> 2. **Answerability** — for each item, is the expected answer verifiable from the stated expected_sources? Flag vague items ("depends", "ask the team") for rewrite.
> 3. **Ambiguity** — items where two reasonable people would give different answers; propose a sharper phrasing.
> 4. **Leakage risk** — items whose expected_answer contains information that would let a model score well without actually retrieving (answer embedded in the question).
> 5. **Missing realistic scenarios** — given the domain, list 10 question archetypes NOT represented (e.g., late second-stage reviews, breach recurrence, job-failure impact, superseded policy versions, attachment-based evidence).
> 6. **Verdict** — ACCEPT / REVISE with the top fixes.
> [PASTE: golden_v1.yaml]

### 5.3 Milestone gate review prompt

> You are the gate reviewer for Milestone 0 of this project. Below are: the M0 objectives and metrics table, the human-input checklist with claimed completion status, the `cli ping` output, the audit summary output, the CI run result, and the per-task review verdicts.
> Determine, with evidence cited for each: (1) every objective O1–O5 met? (2) any hard-rule violation open? (3) any human checklist item (H1–H9) unresolved that blocks Milestone 1 (schema-graph build needs H1 and H3 at minimum; H5/Neo4j is NOT a blocker — M1 builds on the in-memory backend, but confirm the contract tests exist so the later swap is safe)? (4) risks to carry forward as a register.
> Output: GATE PASS or GATE HOLD, a risk register (risk / likelihood / impact / owner), and the three most valuable things to verify manually that automated checks cannot (e.g., that the read-only account evidence is genuine, that golden-set answers were confirmed by a business user, that the Vertex project has training-data-use disabled).
> [PASTE: artifacts]

---

## 6. Gaps in this workflow & questions to settle before/during M0

Things this plan (and the original blueprint) does not yet cover — decide now or consciously defer:

1. **Vertex vs AI Studio (H4) is the biggest fork.** If only AI Studio keys are available, get explicit compliance approval for the data-use terms before ANY real data touches the model — otherwise M0 proceeds with synthetic data only. *Question: which one, and is data-use-for-training disabled in writing?*
2. **Where does this run?** Dev laptop vs internal VM vs cloud. Affects eventual Neo4j approval (H5), egress rules (H8), and later scheduling. Also decide the in-memory backend's ceiling: it is fine through M1 and demo-scale M2; revisit before ingesting the full document corpus (M3) — if Neo4j is still not approved by then, escalate rather than scale the mock. *Question: target runtime environment for M1+?*
3. **Who are the end users and through what surface?** The plan defers UX to M5, but knowing "chat in Teams/Slack" vs "web page" vs "email digest" now prevents rework. *Question: preferred delivery surface?*
4. **Golden-set answer freshness.** Structured-data answers ("yesterday's breaches") change daily. Decide convention: pin questions to fixed historical dates in the golden set. (The T7 schema supports this; the humans writing items must follow it.)
5. **Non-English content?** If policies/comments exist in other languages, embedding and extraction choices in M3/M4 change. *Question: languages in scope?*
6. **Audit log itself contains sensitive prompts.** SQLite on a dev box is fine for M0; decide the prod home (Oracle table? central logging) and retention period now, since compliance may mandate one. *Question: audit retention requirement?*
7. **Model version pinning.** Gemini model aliases move. Config uses explicit versions; decide the upgrade policy (harness must pass on the new version before switching). Documented in T1 README — needs an owner.
8. **Change management.** In regulated shops, even a dev tool reading prod-replica data may need a formal request. *Question: does M0 need a change ticket, and is there a prod-replica vs prod distinction for the read-only account?*
9. **Team capacity.** The plan assumes 1–2 juniors; if it's one junior part-time, expect M0 to take 2 weeks, not 1 — the human checklist (H1–H9) is usually the long pole, so start it on day 1 in parallel with T1.

---

## 7. Definition of done (one line)

**M0 is done when a fresh clone + `.env` + `make install && make test && python -m cli ping` goes all-green on the dev host, the audit summary shows the pings, the golden set passes both schema validation and the §5.2 review, the H1–H9 checklist has evidence attached, and the §5.3 gate prompt returns GATE PASS.**
