# TriGraphRAG — Internal Framework Blueprint

*A reusable, domain-agnostic implementation of the MedGraphRAG architecture (Wu et al., 2024, arXiv:2408.04187), designed from the paper's ideas and the lessons (good and bad) of its reference implementation.*

**Audience:** junior developers implementing this step by step under tech-lead review.
**Status:** design document / implementation plan.

---

## 1. Why we are building this (and not forking the paper's repo)

The published repo (`MedicineToken/Medical-Graph-RAG`) is a research prototype. Our code audit found:

| Paper claim | What the code does | Verdict |
|---|---|---|
| Sliding-window semantic chunking | LLM proposition extraction + agentic chunker (1 LLM call per paragraph **and** per proposition) | Replace with cheaper deterministic-first chunking |
| UMLS-typed entity extraction | Generic "any noun phrase" CAMEL prompt, called **twice** per chunk (one result discarded) | Rebuild with typed, schema-driven extraction, single call |
| 12-layer hierarchical tag clustering | One flat `Summary` node per subgraph, no hierarchy | Build a real (but shallow) hierarchy, or vector-index summaries |
| U-Retrieval top-down descent | **Linear scan**: one GPT-4 call per stored summary per query, verbal 5-point rating | Replace with embedding search; LLM only for re-ranking |
| Bottom-up multi-layer refinement | Exactly 2 calls: draft answer → 1 citation-refinement pass | Keep the 2-pass core; make layer count configurable |
| Trinity cross-layer linking | Cypher GDS cosine query exists (`ref_link`) but the CLI entrypoint calls the wrong function (`link_context`) — links never created | Rebuild with tests |
| — | Hardcoded model names, hardcoded DB password, node-merge query merges only one pair per run | Config-driven everything |

**What we keep from the paper — the two ideas that are actually valuable:**

1. **Three-tier graph with typed cross-tier edges.** Tier 1 = mutable tenant/project data. Tier 2 = curated authoritative sources ("the reference of"). Tier 3 = controlled vocabulary / glossary ("the definition of"). Every Tier-1 entity resolves to a citable source and a canonical definition. This is what makes answers *traceable* instead of plausible.
2. **Route-then-refine retrieval.** Route the query to one (or few) subgraphs via summaries; draft an answer from the subgraph's internal edges; refine the draft with cross-tier reference context and force inline citations. Their ablation shows the graph construction contributes most of the gain; retrieval refinement adds a smaller bump — so we invest accordingly.

**Generalization rule:** everywhere the paper says "medical," we read "domain with (a) an authoritative corpus and (b) a controlled vocabulary." Legal (statutes + Black's Law-style glossary), finance (filings + GAAP/IFRS terms), internal engineering (ADRs + internal glossary) all fit. If a project has no controlled vocabulary, Tier 3 collapses gracefully into Tier 2 — the framework must support 2-tier and 3-tier modes.

---

## 2. Architecture overview

```
                        ┌─────────────────────────────────────────┐
                        │              QUERY PIPELINE             │
                        │  tag(query) → route → draft → refine    │
                        └───────────────▲─────────────────────────┘
                                        │ reads
┌───────────────────────────────────────┴─────────────────────────┐
│                        GRAPH STORE (Neo4j)                      │
│                                                                 │
│  Tier 1: Project graphs (one Meta-Graph per doc/chunk-group)    │
│    (Entity {gid, name, type, context, embedding})               │
│    (Entity)-[:REL {desc}]->(Entity)        within a gid         │
│    (Summary {gid, tags, embedding})-[:SUMMARIZES]->(Entity)     │
│                                                                 │
│  Tier 2: Source graph (papers/books/docs)   — immutable         │
│  Tier 3: Vocabulary graph (ontology terms)  — immutable         │
│                                                                 │
│  (t1:Entity)-[:REFERENCE_OF]->(t2:Entity)   cross-tier          │
│  (t2:Entity)-[:DEFINITION_OF]->(t3:Entity)  cross-tier          │
└───────────────────────────────────────▲─────────────────────────┘
                                        │ writes
                        ┌───────────────┴─────────────────────────┐
                        │            INGEST PIPELINE              │
                        │ chunk → extract → embed → link → tag    │
                        └─────────────────────────────────────────┘
```

Two pipelines, one store. **Ingest is batch/offline; query is online.** They must never share mutable state other than the graph store.

### 2.1 Package layout

```
trigraphrag/
  config.py            # pydantic Settings: models, thresholds, tiers, db
  models/
    entities.py        # Entity, Relation, MetaGraph, TagSummary (pydantic)
  llm/
    client.py          # ONE thin LLM interface: complete(), embed()
    prompts/           # versioned prompt files, per-domain overridable
      extract.j2
      relate.j2
      tag_summary.j2
      answer_draft.j2
      answer_refine.j2
  ingest/
    chunker.py         # Chunker protocol + Static, Semantic impls
    extractor.py       # schema-driven entity extraction
    linker.py          # cross-tier similarity linking
    tagger.py          # tag-summary generation + hierarchy build
    pipeline.py        # orchestration, idempotency, checkpointing
  store/
    graph.py           # GraphStore protocol + Neo4j impl (ALL Cypher here)
    vector.py          # summary/entity vector index (Neo4j native or external)
  query/
    router.py          # query → gid(s)
    responder.py       # draft + refine + citation assembly
  eval/
    harness.py         # golden-set QA runner, citation precision/recall
  cli.py               # ingest / link / query / eval commands
```

**Hard rules for contributors**

- No Cypher outside `store/graph.py`. No prompt strings outside `llm/prompts/`.
- Every LLM output that we parse must be **JSON constrained by a pydantic schema**, with one retry-on-parse-failure. Never parse free text with `if "similar" in response` (this is exactly the fragile pattern in the reference repo).
- All thresholds, model names, and layer counts come from `config.py`. Nothing hardcoded (the repo hardcodes model names in five places and a DB password in one).
- Everything keyed by `gid` (subgraph id) and `tier`. Tier 2/3 nodes carry `tier` labels so retrieval queries can filter cheaply.

---

## 3. Component specifications

### 3.1 Chunker (`ingest/chunker.py`)

```python
class Chunker(Protocol):
    def chunk(self, doc: Document) -> list[Chunk]: ...
```

- **Phase-1 impl: `StaticChunker`** — split on blank lines, pack paragraphs into chunks up to `max_tokens` (default 1,500) with 1-paragraph overlap. Deterministic, free, testable.
- **Phase-4 impl: `SemanticChunker`** — the paper's actual algorithm (not the repo's): keep a sliding window of the last *w*=5 paragraphs; one **cheap-model** LLM call decides "does paragraph *j* continue the current topic? yes/no"; token cap as hard boundary. Cost: 1 small call per paragraph, not per proposition.
- Acceptance: chunk boundaries stable across runs on the same input; property test that concatenated chunks (minus overlap) reconstruct the doc.

### 3.2 Extractor (`ingest/extractor.py`)

One LLM call per chunk (the repo does two; the first result is discarded — don't repeat this). Output schema:

```python
class Entity(BaseModel):
    name: str          # surface form or canonical derivative
    type: str          # MUST be from settings.entity_types (domain enum)
    context: str       # 1–3 sentences situating the entity in this doc

class Relation(BaseModel):
    source: str; target: str
    description: str   # short verb phrase

class ExtractionResult(BaseModel):
    entities: list[Entity]; relations: list[Relation]
```

- `entity_types` is a **config list**, not a prompt hardcode. Medical = UMLS semantic types; legal = {party, obligation, statute, definition, deadline…}; each project supplies its own. This is the single most important domain-adaptation point.
- Entity embedding = `embed(f"name: {name}; type: {type}; context: {context}")` — the paper's `C_e` recipe. Store on the node.
- Relations are only created *within* a chunk's entity set (Meta-Graph = per-chunk directed graph, per the paper).

### 3.3 Linker (`ingest/linker.py`)

- For each Tier-1 entity, k-NN search against Tier-2 entity embeddings (vector index, **not** the O(N²) Cypher cross product in the repo's `ref_link`); create `REFERENCE_OF` edge when cosine ≥ `δ_r` (default 0.5, per config). Same for Tier-2 → Tier-3 with `DEFINITION_OF`, δ default 0.6.
- Optional guard: require type compatibility (a config map, e.g. medication→drug), replacing the repo's brittle "identical Neo4j labels" requirement.
- Same-tier dedup (`merge_similar_nodes` in the repo, which only merges one pair per invocation): implement as batched union-find on similarity pairs above threshold, merge all clusters, keep provenance list of merged names. Ship behind a flag; off by default until Phase 5 — premature merging destroys citations.

### 3.4 Tagger (`ingest/tagger.py`)

- Per Meta-Graph: one LLM call producing a **structured tag summary** against `settings.tag_categories` (the repo's medical category prompt in `summerize.py` is a good template — generalize the category list). Store as `Summary{gid, content, embedding}` with `SUMMARIZES` edges.
- Hierarchy: Phase 1 = **no hierarchy** — just a vector index over summaries (this already beats the repo's linear LLM scan by orders of magnitude in cost and latency). Phase 5 = optional agglomerative clustering of summaries into parent summaries, depth from config (paper uses up to 12; start with 2–3; you hit diminishing returns fast at small corpus sizes).

### 3.5 Router (`query/router.py`)

1. Tag the query with the same tag-summary prompt (1 call).
2. Vector search over Summary embeddings → top-`m` candidate gids (m=3 default).
3. Optional LLM re-rank of the top-m (1 call) — this is the *correct* use of the repo's rating idea: re-rank a shortlist, never scan the corpus.

### 3.6 Responder (`query/responder.py`)

The repo's 2-pass core is worth keeping, hardened:

1. **Draft pass.** Pull the routed subgraph: top-`N` entities by query-embedding similarity (paper: 60) + their neighbors within `k` hops (implement as k=2 default; the paper's "16-hop" setting is almost certainly k-NN count, not path length — a 16-hop traversal would pull the whole graph). Serialize as `(name)-[rel]->(name)` triples. One call: answer from GRAPH only.
2. **Refine pass.** Fetch `REFERENCE_OF` / `DEFINITION_OF` neighborhoods for the entities actually used; number them; one call with the repo's `sys_prompt_two` pattern: *revise the draft, add inline citations [1][3], or answer plainly if references don't apply.*
3. **Citation assembly (new — the repo doesn't do this):** map [n] markers back to Tier-2/3 node ids and return `{answer, citations: [{marker, tier, node_id, source_doc, snippet}]}`. A citation the caller can't dereference is decoration, not evidence.
4. Optional bottom-up passes over parent summaries (Phase 5), capped by `settings.refine_layers` (paper uses 4–6).

### 3.7 Evaluation harness (`eval/`)

Non-negotiable before any tuning. Per project: 30–50 golden Q/A pairs with expected source docs. Metrics: answer accuracy (LLM-judged vs gold), **citation precision/recall** (did cited node ids come from the expected docs), routing hit-rate (right gid in top-m), latency, $-per-query and $-per-1k-tokens-ingested. Every PR that touches prompts or thresholds runs the harness.

---

## 4. Implementation plan (phased, junior-friendly)

Each phase is independently shippable and reviewed before the next starts. Estimated effort assumes one junior dev per phase with tech-lead review.

**Phase 0 — Skeleton & plumbing (2–3 days).**
Repo scaffold, `config.py`, LLM client wrapper with retry/JSON-mode/cost-logging, Neo4j docker-compose, `GraphStore` protocol with `upsert_entities / upsert_relations / query_subgraph / vector_search`, CI with lint+tests.
*Accept:* `cli.py ping` round-trips to Neo4j and the LLM; unit tests green.

**Phase 1 — Minimal ingest + naive query (1 week).**
StaticChunker → Extractor → embed → store, one Summary per doc, vector routing, draft-pass-only responder.
*Accept:* ingest 20 sample docs; 10 golden questions answered from the right doc ≥ 8/10; total ingest cost logged and < agreed budget.

**Phase 2 — Two-tier linking + citations (1 week).**
Load a Tier-2 source corpus, Linker with vector k-NN, refine pass, citation assembly.
*Accept:* eval harness reports citation precision ≥ 0.8 on golden set; every citation dereferences to a real Tier-2 node.

**Phase 3 — Third tier + domain packs (3–4 days).**
Vocabulary tier, `DEFINITION_OF` links, "domain pack" = {entity_types, tag_categories, prompt overrides} as a directory; ship two packs (e.g., medical-lite, internal-docs).
*Accept:* switching domain packs requires zero code changes.

**Phase 4 — Semantic chunking + routing re-rank (3–4 days).**
SemanticChunker behind a flag; A/B on eval harness vs StaticChunker; LLM re-rank of top-m routes.
*Accept:* semantic chunking only enabled by default if it beats static on the harness by an agreed margin at acceptable cost.

**Phase 5 — Scale & polish (1–2 weeks, as needed).**
Summary hierarchy + bottom-up refinement, batched node dedup, incremental re-ingest (delete-by-gid + re-add; gid-per-doc makes this safe), parallel ingest workers, cost dashboards.

**Explicitly deferred / out of scope v1:** 12-layer hierarchies, cross-graph node merging across tenants, streaming updates, the paper's 5-shot response ensembling (eval-time trick, inflates cost 5×), fine-tuning anything.

---

## 5. Cost & performance guardrails (learned from the audit)

- **Ingest cost model:** ~(1 chunker call if semantic) + 1 extract call + E embeddings per chunk + 1 tag call per doc. Log per-doc cost; alert if a doc exceeds budget.
- **Query cost model:** 1 tag + 1 optional re-rank + 1 draft + 1 refine = **≤ 4 LLM calls, O(1) in corpus size.** The reference repo is O(N) LLM calls per query — this is the single biggest thing we're fixing.
- Cheap model for chunking/tagging/re-rank; strong model only for draft/refine. Model choice per role in config.
- Embedding model changes invalidate all stored vectors — store `embedding_model` on every node; refuse to mix.

## 6. Risks & mitigations

| Risk | Mitigation |
|---|---|
| Wrong route → confidently wrong answer | top-m routing (not top-1) + "insufficient context" instruction in draft prompt + routing hit-rate metric |
| Hallucinated citations | citations only from dereferenceable node ids; harness checks precision |
| Threshold fragility (0.5 everywhere in repo/paper) | thresholds in config; harness sweep script in `eval/` |
| No controlled vocabulary in some domains | 2-tier mode is first-class, not a hack |
| Extraction schema drift across domains | domain packs versioned; harness per pack |

## 7. Reference material for the team

- Paper: arXiv 2408.04187 (read §2 Method + Table 3 ablation; skim the rest).
- Reference repo: `github.com/MedicineToken/Medical-Graph-RAG` — read `utils.py` (`get_response`, `ref_link`, `sys_prompt_two`) and `summerize.py` (tag prompt) as *inspiration*; treat everything else as a cautionary tale.
- Microsoft GraphRAG (`microsoft/graphrag`) — the community-detection approach we deliberately avoid; useful contrast for design discussions.
