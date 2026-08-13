# Milestone 3 — Execution Pack
### Document Graph: Tier 2 sources + Tier 3 glossary, with citations (Weeks 5–7)

**Open only after the M2 gate passes.** Uses the M0 context header plus:

> **M3 CONTEXT ADDITION:** This milestone implements the TriGraphRAG document core (blueprint §3). New code in `app/ingest/` and `app/query/`. Tier-2 documents are versioned and authoritative: `{doc_id, version, effective_date, superseded_by, source_system, owner}` on every doc node; superseded content is marked, never deleted; retrieval filters to current versions by default. All extraction uses the **finance domain pack** in `config/domain_packs/finance/` — a directory of `{entity_types.yaml, tag_categories.yaml, prompt overrides}`; code must load any pack by name with zero code changes. Confluence ingestion pulls ONLY pages labeled `ai-source-of-truth`. Query path budget: ≤ 4 LLM calls, O(1) in corpus size.

## 1. Objectives

1. **O1 — Corpus in, versioned.** 20–50 authoritative documents (policies, methodology, SOPs, labeled Confluence pages) ingested with full metadata; re-ingest is a no-op; new versions supersede.
2. **O2 — Typed extraction via domain pack.** Entities/relations extracted per the finance pack; pack swap requires no code change (proven with a second toy pack).
3. **O3 — Tier 3 stands.** Glossary from compliance terms + the M1 data dictionary; `DEFINITION_OF` links from Tier-2 entities.
4. **O4 — Route→draft→refine answers with dereferenceable citations.** Policy questions answered with `{doc, version, section, snippet}` citations that open to real content.
5. **O5 — Measured.** Routing hit-rate ≥ 80% top-3, citation precision ≥ 0.8 on the `policy_docs` golden category; cost per document and per query reported.

## 2. User input needed

| # | Item | Who |
|---|---|---|
| H1 | Collect the authoritative corpus WITH version/date/owner; confirm each is current-in-force; deliver as files + a manifest CSV | Compliance + tech lead |
| H2 | In Confluence: create/apply the `ai-source-of-truth` label to designated pages; provide a scoped read account/token for the API (not a personal token); name the spaces in scope | Confluence admin + business owner |
| H3 | Provide the business/regulatory glossary (or bless building Tier 3 from data dictionary + terms defined inside the policies themselves) | Compliance |
| H4 | Review the finance domain pack lists (entity types, tag categories) in a 30-min session — these encode the domain | Senior dev + business |
| H5 | Sign off that document content may go to Gemini under the H3(M0) classification (documents may differ from DB data!) | Compliance |

## 3. Build tasks with LLM prompts

### T1 — Loaders & cleaning

> [CONTEXT HEADER + M3 ADDITION]
> Task: `app/ingest/loaders.py`. Loaders for pdf (pypdf/pdfplumber text extraction; if a page yields <50 chars mark `needs_ocr:true` and skip content honestly), docx (python-docx), html/markdown, and Confluence (REST API: search by label `ai-source-of-truth` in `settings.confluence.spaces`, fetch storage-format body + `version.number`, `history.lastUpdated`, ancestors; clean storage HTML: strip macros to their text content, tables to markdown, resolve page-links to `confluence:{page_id}` refs). Common output: `LoadedDoc{doc_id (stable: source_system+native_id), title, version, effective_date, owner, text, structure: list[Section{heading, level, char_span}], hierarchy_parent (confluence ancestors), load_warnings}`. Manifest CSV loader for H1 metadata merge.
> Tests: canned files per format; Confluence client fully mocked; a page WITHOUT the label is never fetched (assert).

### T2 — Chunker

> [CONTEXT HEADER + M3 ADDITION]
> Task: `app/ingest/chunker.py`. `Chunker` protocol + `StaticChunker`: pack paragraphs to `settings.ingest.max_chunk_tokens` (default 1500, token count via a cheap estimator documented in code) with 1-paragraph overlap; never split mid-paragraph; carry `Section` info onto each `Chunk{chunk_id = doc_id + content_hash, text, section_path, char_span}` so citations can say "§4.2". Property tests: reassembly, stable ids, section attribution.

### T3 — Extractor + domain pack

> [CONTEXT HEADER + M3 ADDITION]
> Task: `app/ingest/extractor.py` + `config/domain_packs/finance/`.
> Pack files: `entity_types.yaml` (benchmark, threshold_rule, obligation, control, review_stage, deadline, role, sanction, defined_term, procedure_step — each with a one-line definition the prompt will include), `tag_categories.yaml` (APPLIES_TO, OBLIGATION, DEADLINE, ESCALATION, ROLE, PENALTY, SCOPE), optional `extract_prompt_override.j2`.
> Extractor: per chunk, ONE Flash call → `ExtractionResult{entities:[{name,type∈pack,context}], relations:[{source,target,description}]}` (schema-constrained, one retry); reject entities with out-of-pack types (log, don't crash). Embed entities (name+type+context) and chunk text. Include a second toy pack `config/domain_packs/generic/` and a test proving pack swap needs no code edits.

### T4 — Tagger, Tier-3 builder, Linker

> [CONTEXT HEADER + M3 ADDITION]
> Task: three modules.
> `tagger.py`: per document, one Flash call → structured summary per tag_categories; store `Summary{gid=doc_id, content, embedding}`.
> `tier3.py`: build glossary nodes from (a) H3 glossary file, (b) M1 data dictionary (reuse those nodes — link, don't duplicate), (c) `defined_term` entities extracted from policies ("'Breach' means…" patterns) flagged `mined:true` for review; key = normalized term.
> `linker.py`: for each Tier-2 entity, `vector_search` over Tier-3 → `DEFINITION_OF` when cosine ≥ `settings.linking.def_threshold` (0.6); (Tier-1→Tier-2 linking is M4). Batch, resumable, report links created/skipped.
> Tests on InMemory store with synthetic embeddings.

### T5 — Ingest pipeline + versioning

> [CONTEXT HEADER + M3 ADDITION]
> Task: `app/ingest/pipeline.py` + CLI `ingest --source <path|confluence> --tier 2 --pack finance`.
> Flow: load → (compare `version` to stored: same → skip; higher → ingest new chunks, mark ALL prior-version chunk/entity/summary nodes `superseded_by:<new version>`; lower → refuse loudly) → chunk → extract → embed → upsert → tag → link → `IngestReport{docs, chunks, entities, links, skipped, superseded, cost_usd, warnings}` printed and saved. `ingest sync-confluence` = delta by version number. Everything resumable (crash mid-doc → rerun completes without duplicates — content-hash keys make this true; test it).

### T6 — Route → draft → refine responder

> [CONTEXT HEADER + M3 ADDITION]
> Task: `app/query/router.py` + `app/query/responder.py` + prompts.
> Router: embed query → vector_search Summaries (current versions only) → top-3 gids; optional Flash re-rank (settings flag).
> Responder pass 1 (DRAFT, Pro): serialize routed docs' top-`settings.query.top_entities` (default 40) entities by query-similarity + their 2-hop neighbors as triples + the source chunk texts for the top entities; instruction: answer ONLY from this material; say "insufficient context" when true.
> Pass 2 (REFINE, Pro): fetch `DEFINITION_OF` neighborhoods for entities used; numbered reference list; revise draft injecting `[n]` markers.
> Citation assembly: map `[n]` → `{doc_id, version, section_path, snippet(≤200 chars), node_id}`; drop any marker that doesn't map (log warning "hallucinated marker") — never invent a citation. Return `AnswerResult` (harness-compatible). CLI `ask-docs "<question>"`. Budget assert: ≤ 4 LLM calls per query (tested).
> Wire into the M2 harness as the `policy_docs` answer_fn.

## 4. Gate metrics

| Metric | Target |
|---|---|
| Re-ingest of unchanged corpus | 0 new nodes, 0 LLM calls |
| Version supersede test (edit one doc, bump version) | old marked, new retrieved, old citations still resolve |
| Routing hit-rate (`policy_docs` golden, expected doc in top-3) | ≥ 80% |
| Citation precision (cited docs ∈ expected_sources) / recall | ≥ 0.8 / report |
| Hallucinated citation markers | 0 in golden run |
| "Insufficient context" honesty (5 questions with answers NOT in corpus) | 5/5 refuse, 0 fabrications |
| Pack swap without code change | Test green |
| Cost per document (ingest) and per query | Reported; within budget |
| Confluence: only labeled pages ingested | Assert + spot check |

## 5. Evaluator-LLM prompts

**Code review:** M0 §5.1 plus: "(7) Version logic — construct a scenario where a superseded chunk could be retrieved as current; (8) citation faithfulness — can a snippet be assembled that doesn't exist verbatim in the source chunk?; (9) query budget — count worst-case LLM calls."

**Citation-faithfulness audit (new):**
> You audit an AI system's citations for a compliance context. Input: 15 Q&A pairs with citations `{doc, version, section, snippet}` and the full text of each cited section.
> For each citation judge: SUPPORTS (section genuinely supports the claim it's attached to) / PARTIAL (related, doesn't establish the claim) / MISLEADING (contradicts or irrelevant) / STALE (a newer version of the doc exists in the provided manifest). Quote the exact sentence that supports or fails each.
> Then judge each ANSWER overall: is any material claim uncited? Output: per-citation table, per-answer verdict, precision estimate, PASS (precision ≥ 0.8, zero MISLEADING) / FAIL.

**Domain-pack review (new):**
> You are a regulatory-domain reviewer. Given the entity_types and tag_categories YAML for a benchmark-surveillance corpus and 20 sample extracted entities with source sentences: 1) flag type definitions that are ambiguous or overlapping (would two annotators disagree?); 2) flag sample entities that are mis-typed, with the correct type; 3) list up to 5 missing entity types the samples suggest; 4) verdict ACCEPT/REVISE.

**Gate review:** M0 §5.3 structure with M3 metrics; artifacts = ingest reports, eval report, faithfulness audit, pack review, H1–H5 evidence.

## 6. Traps & open questions

1. **PDF extraction quality decides everything downstream.** Eyeball the extracted text of your 5 most important documents before ingesting the rest. Scanned pages → `needs_ocr` list → human decision (retype the key sections, or OCR later).
2. **Don't ingest all of Confluence "because it's easy."** The label is the safety mechanism; three stale pages describing an old workflow will poison citations worse than an empty Tier 2.
3. **Chunk size vs section granularity:** if citations keep saying "§ whole-document", sections aren't being captured — fix the loader's structure detection before blaming retrieval.
4. *Question:* documents in languages other than English? (Changes embedding evaluation and judge rubric.)
5. *Question:* is there a policy review calendar (annual re-approval)? If yes, `effective_date`+review cycle could drive a staleness warning in answers — note for M6.

**Exit:** gates green → open `14_milestone_4_execution_pack.md`.
