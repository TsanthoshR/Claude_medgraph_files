# Benchmark-Surveillance AI Assistant — Planning Docs

Planning and execution documents for building a TriGraphRAG-based AI assistant
for a regulated finance benchmark-surveillance workflow (Oracle + Gemini + Google ADK).

## Reading order

1. **[docs/01_trigraphrag_framework_blueprint.md](docs/01_trigraphrag_framework_blueprint.md)** —
   the domain-agnostic framework design, derived from the MedGraphRAG paper
   (arXiv:2408.04187) and a code audit of its reference implementation.
2. **[docs/02_benchmark_ai_milestone_plan.md](docs/02_benchmark_ai_milestone_plan.md)** —
   the blueprint adapted to the Oracle-heavy regulated finance context; milestones M0–M6.
3. **[docs/03_milestone_0_execution_pack.md](docs/03_milestone_0_execution_pack.md)** —
   the current working document: M0 objectives, human-only checklist (H1–H9),
   builder-LLM prompts (T1–T8), gate metrics, and evaluator-LLM prompts.

## Status

- [ ] M0 in progress — see the execution pack, §7 for the definition of done
- Graph backend: in-memory mock (Neo4j pending infra approval)
- Oracle access: named TNS alias, read-only account

Milestone execution packs for M1+ are issued only after the previous gate review passes.
