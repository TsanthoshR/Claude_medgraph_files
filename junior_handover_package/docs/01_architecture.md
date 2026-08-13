# 01 — System Architecture (Diagrams)

All diagrams are Mermaid — they render directly on GitHub. This document is the visual companion to the blueprint (`02`) and the milestone plan (`03`).

---

## 1. Full architecture — every process on one page

```mermaid
flowchart TB
    subgraph USERS["👥 Users & Surfaces"]
        REV["Reviewers / Traders / Ops"]
        SURF["Chat surface / internal web page (M5)\nCLI (M0+)  ·  adk web dev UI (M2+)"]
    end

    subgraph AGENTS["🤖 Agent Layer (Google ADK, M2→M5)"]
        ROUTER["Root Router Agent\nFlash · constrained classification:\nstructured / policy / review-draft / mixed / ESCALATE"]
        SDA["StructuredDataAgent (M2)\nparameterized SQL FunctionTools"]
        GRA["GraphRAGAgent (M3)\nroute → draft → refine + citations"]
        RDA["ReviewDraftAgent (M5)\ndeterministic orchestration in code"]
        ROUTER --> SDA
        ROUTER --> GRA
        ROUTER --> RDA
        RDA -.calls.-> SDA
        RDA -.calls.-> GRA
    end

    subgraph CORE["⚙️ Core Services (M0)"]
        LLM["LLMClient — the ONLY Gemini gateway\ncomplete(schema) · embed · retries · cost"]
        AUD[("Audit Log\nevery LLM call + every SQL")]
        CFG["Settings (pydantic)\nmodels · thresholds · allow-lists"]
        EVAL["Eval Harness + Golden Set\naccuracy · citation P/R · routing · cost"]
    end

    subgraph STORES["🗄️ Data Layer"]
        ORA[("Oracle (system of record)\ntables · FKs · procs · scheduler jobs\nREAD-ONLY named TNS connection")]
        GS[("GraphStore (contract)\nInMemory now → Neo4j later")]
    end

    subgraph GRAPH["Three-tier graph inside GraphStore"]
        T1["Tier 1 — Operational (mutable)\nschema graph · breaches · reviews\ncomments · attachments"]
        T2["Tier 2 — Source-of-truth docs (versioned)\nregulations · policies · SOPs · Confluence"]
        T3["Tier 3 — Glossary\ndata dictionary · defined terms"]
        T1 -- "REFERENCE_OF / GOVERNED_BY" --> T2
        T2 -- "DEFINITION_OF" --> T3
        T1 -- "DEFINED_AS" --> T3
    end

    subgraph INGEST["📥 Ingest Pipelines (batch)"]
        M1P["M1 Schema sync\nOracle catalog → Tier-1 scaffold\n(no LLM for structure)"]
        M3P["M3 Document ingest\nload → chunk → extract → embed\n→ tag → link"]
        M4P["M4 Nightly delta\nbreach rows · comments · attachments\n→ per-breach mini-graphs"]
    end

    REV --> SURF --> ROUTER
    SDA --> ORA
    GRA --> GS
    AGENTS --> LLM
    INGEST --> LLM
    LLM --> AUD
    SDA --> AUD
    ORA --> M1P & M4P --> GS
    DOCS["📄 Policies · Regulations\nConfluence (labeled pages only)"] --> M3P --> GS
    GS === GRAPH
    EVAL -.judges.-> AGENTS
    CFG -.configures.-> CORE & AGENTS & INGEST
```

**Cross-cutting invariants:** every model call goes through `LLMClient` → audit log (agents included, via ADK callbacks); Oracle is SELECT-only over allow-listed views; all graph access uses the typed `GraphStore` contract; every answer that cites does so with dereferenceable `{doc, version, section}` citations.

---

## 2. The three-tier graph model

```mermaid
erDiagram
    BREACH ||--o{ REVIEW : "HAS_REVIEW (stage 1..2)"
    BREACH }o--|| BENCHMARK : "ON_BENCHMARK"
    BREACH ||--o{ COMMENT : "HAS_COMMENT"
    BREACH ||--o{ ATTACHMENT : "HAS_ATTACHMENT"
    BENCHMARK ||--|| THRESHOLD_CONFIG : "HAS_THRESHOLD"
    JOB ||--o{ PROCEDURE : "RUNS"
    PROCEDURE ||--o{ TABLE : "READS/WRITES"
    TABLE ||--o{ COLUMN : "HAS_COLUMN"
    TABLE ||--o{ TABLE : "FK_TO"

    THRESHOLD_CONFIG }o--o{ POLICY_CLAUSE : "GOVERNED_BY (T1→T2)"
    BENCHMARK }o--o{ METHODOLOGY_DOC : "DEFINED_IN (T1→T2)"
    REVIEW }o--o{ OBLIGATION : "REQUIRED_BY (T1→T2)"
    POLICY_CLAUSE }o--o{ DEFINED_TERM : "DEFINITION_OF (T2→T3)"
    COLUMN }o--o{ DEFINED_TERM : "DEFINED_AS (T1→T3)"
```

Tier 1 nodes carry `gid` (subgraph id) and live keys from Oracle (`breach_id`, table names). Tier 2 nodes carry `{doc_id, version, effective_date, superseded_by}`. Tier 3 nodes carry `{term, definition, source}`. Cross-tier edges are created by the linker (embedding k-NN above threshold) and, for the critical `GOVERNED_BY` set, **human-approved once** (M4).

---

## 3. Ingest pipeline (M3 documents; M4 reuses the same shape)

```mermaid
sequenceDiagram
    participant U as Junior (CLI) / Nightly job
    participant P as Ingest Pipeline
    participant L as LLMClient (Flash)
    participant E as LLMClient (embed)
    participant G as GraphStore
    participant A as Audit Log

    U->>P: ingest --source policies/ --tier 2
    P->>P: load & clean (pdf/docx/Confluence API)<br/>capture doc_id, version, effective_date
    P->>P: chunk (StaticChunker ~1500 tok)
    loop per chunk
        P->>L: extract entities+relations (typed schema)
        L->>A: record call + cost
        P->>E: embed entities & chunk
        E->>A: record
        P->>G: upsert nodes/edges (key = doc_id+chunk_hash → idempotent)
    end
    P->>L: tag summary per document
    P->>G: upsert Summary node
    P->>P: linker: k-NN vs other tiers → REFERENCE_OF / DEFINITION_OF
    P->>G: upsert cross-tier edges
    P-->>U: ingest report: chunks · entities · links · $ · failures
    Note over P,G: New version of a doc? old chunks marked superseded_by — never deleted
```

---

## 4. Query pipeline — route → draft → refine (GraphRAGAgent)

```mermaid
sequenceDiagram
    participant Q as User question
    participant R as Router (tag query)
    participant G as GraphStore
    participant P as LLMClient (Pro)
    participant C as Citation assembler

    Q->>R: "What does policy require at second review?"
    R->>G: vector search over Summary embeddings → top-3 gids
    R->>P: optional re-rank of shortlist (1 call)
    P->>G: pull top-N entities + 2-hop neighbors from routed subgraph
    P->>P: DRAFT answer from serialized triples only (1 call)
    P->>G: fetch REFERENCE_OF / DEFINITION_OF context for used entities
    P->>P: REFINE draft, inject [n] citations (1 call)
    P->>C: map [n] → node ids
    C-->>Q: {answer, citations:[{doc, version, section, snippet}]}
    Note over R,P: ≤ 4 LLM calls per query, O(1) in corpus size — hard budget
```

---

## 5. Agent topology and routing (M5 end-state)

```mermaid
flowchart TD
    Q([User question]) --> RT{Router · Flash\nconstrained output}
    RT -- structured_data --> SDA[StructuredDataAgent]
    RT -- policy_docs --> GRA[GraphRAGAgent]
    RT -- review_draft --> RDA[ReviewDraftAgent]
    RT -- mixed --> RDA
    RT -- cannot answer --> ESC([Honest refusal +\nescalate to human])

    SDA --> TOOLS["8–12 parameterized SQL tools\nmax rows · timeout · SELECT-only tripwire"]
    TOOLS --> ORA[(Oracle RO)]

    RDA -->|"fixed order, in code"| SDA
    RDA -->|"fixed order, in code"| GRA
    RDA --> PACK["Draft review pack:\nnumbers · likely cause · policy clause+citation\nsimilar breaches · open questions"]
    PACK --> HUM([Human reviewer edits & submits\nAI never touches review-state columns])

    style ESC fill:#fdd
    style HUM fill:#dfd
```

Roster is capped at these three + router. New capabilities enter as *tools*, not agents, unless they beat the incumbent on the golden set. Router accuracy is a harness metric (≥95%).

---

## 6. Review-workflow state machine (model this in M1 — it's where wrong answers hide)

```mermaid
stateDiagram-v2
    [*] --> DETECTED: threshold breach (existing proc/cron)
    DETECTED --> PENDING_R1: sent for review
    PENDING_R1 --> R1_DONE: first-stage review complete
    PENDING_R1 --> ESCALATED: SLA exceeded
    R1_DONE --> PENDING_R2
    PENDING_R2 --> CLOSED: second-stage approved
    PENDING_R2 --> RETURNED: sent back with comments
    RETURNED --> PENDING_R1
    ESCALATED --> PENDING_R1
    CLOSED --> [*]
```

The real state names live in a `REF_STATUS`-style table with history tables behind it — discover and encode the *actual* machine in the schema graph (M1-T7); do not trust this diagram over the database.

---

## 7. Milestone map

```mermaid
flowchart LR
    M0["M0\nFoundations\ngateway · audit\ngolden set · mocks"] --> M1["M1\nSchema graph\nOracle catalog → Tier-1"]
    M1 --> M2["M2\nStructuredDataAgent\nADK + SQL tools"]
    M2 --> M3["M3\nDocument graph\nTier 2/3 + citations"]
    M3 --> M4["M4\nThe bridge\nbreaches↔policy\ncomments · attachments"]
    M4 --> M5["M5\nReviewDraftAgent\nmulti-agent + pilot"]
    M5 --> M6["M6\nHardening\nNL2SQL expt · hierarchy\ncompliance review"]
    style M5 stroke-width:3px
```

Each arrow is a **gate**: artifacts → evaluator gate prompt → tech-lead sign-off. M5 (bold) is the demo that sells the project.

## 8. Swap plan: in-memory graph → Neo4j

```mermaid
flowchart LR
    A["GraphStore Protocol\n(typed methods, no raw Cypher)"] --- B["InMemoryGraphStore\n(now: full impl + JSON snapshot)"]
    A --- C["Neo4jGraphStore\n(stub: docstrings lock Cypher mapping)"]
    T["Contract test suite\nparameterized over backends"] -->|validates| B
    T -.->|"add one param when infra lands"| C
    F["make_graph_store(settings)"] -->|backend=memory| B
    F -.->|backend=neo4j| C
```

The mock is fine through M1 and demo-scale M2; if Neo4j is still unapproved at M3's corpus ingest, escalate — don't scale the mock.
