# InsightGraph: A Self-Correcting Research Assistant
**Step-by-step implementation plan (for VS Code repo)**  
Scope: baseline **without RAG** (web → local index → reasoning graph), but architected to “hot-swap” **Qdrant RAG** later.

---

## How to use this plan
Each block = **High-Level Goal** (what we’re trying to achieve), followed by **Steps**, **Definition of Done (DoD)**, and **Artifacts**. Work top to bottom. Keep runs reproducible and idempotent.

> Project name: `insightgraph` (suggested)  
> Repo layout (suggested):
```
insightgraph/
  README.md
  INSIGHTGRAPH_PLAN.md
  config/
    crawl_allowlist.yaml
    retrieval.yaml
  data/
    raw_html/            # fetched pages
    corpus/              # normalized text
    chunks/              # chunk-level JSONL
    memory/              # persistent memory (JSON/SQLite)
    indexes/             # local BM25/TF-IDF artifacts
  graphs/
    main_graph/
    subgraphs/
  tests/
    e2e/
    fixtures/
  logs/
```

---

---

## 0) Project setup & conventions
**Goal:** Standardize repo, env, configs, and naming; avoid tech debt later.
- [X] Create repo + virtual environment + `.env.example` + `pyproject.toml` (or `requirements.txt`).
  - Initialize Git repository with proper `.gitignore`.
  - Setup Python virtual environment with recommended version.
  - Create `.env.example` with placeholders for environment variables.
  - Define dependencies in `pyproject.toml` or `requirements.txt`.
- [ ] Define config files: `crawl_allowlist.yaml`, `retrieval.yaml`, `graph.yaml` (limits, thresholds).
  - Determine config schema for each file.
  - Include comments/examples for user guidance.
  - Validate YAML syntax and loadability.
- [ ] Decide doc identifiers: `doc_id = stable hash(url + canonical title)`, `chunk_id = int`.
  - Specify hashing algorithm (e.g., SHA256).
  - Document identifier generation logic clearly.
  - Ensure uniqueness and collision resistance.
- [ ] Add logging policy (structured logs), and run IDs (UUIDv4) for traceability.
  - Choose logging format (e.g., JSON).
  - Implement run ID generation at start of each run.
  - Integrate logging calls throughout initial codebase.
- DoD: Repo initializes cleanly; configs exist; logging path works.  
- Artifacts: baseline README, configs, empty folders as above.

---

## 1) Core graph scaffold & state boundaries
**Goal:** Compile a minimal LangGraph with strict **global** vs **local** state and partial updates.
- [ ] Specify **Global State** fields: `run_id, query, subtopics[], sections[], retrieval_stats, memory_refs, now`.
  - Define data types and default values.
  - Document intended usage of each field.
  - Ensure immutability or controlled mutation rules.
- [ ] Specify **Local Section State**: `title, draft_chunks[], evidence[], confidence, critic_notes[], needs_human, status`.
  - Clarify ownership boundaries per section.
  - Define lifecycle and update triggers.
  - Prepare schema validations.
- [ ] Wire minimal nodes: `topic_planner → for_each_subtopic(subgraph stub) → aggregate → final_output`.
  - Create stub implementations with NOP behavior.
  - Define input/output contracts.
  - Setup orchestration flow with minimal data passing.
- [ ] Enable checkpointer (in-memory for now).
  - Implement checkpoint save/load API.
  - Test checkpointing on partial graph runs.
  - Ensure checkpoint restores correct state slices.
- DoD: Graph compiles; NOP-nodes run; partial updates modify only their ownership.  
- Artifacts: State contracts (doc), compiled graph skeleton.

---

## 2) Data ring: web intake & normalization (no RAG yet)
**Goal:** Fetch a small, high-quality corpus and normalize it deterministically.
- [ ] Fill `crawl_allowlist.yaml` (3–5 domains/paths). Respect robots.txt.
  - Research target domains for quality and relevance.
  - Specify exact paths or URL patterns.
  - Implement robots.txt compliance checks.
- [ ] Implement fetcher: save **raw HTML** + `fetch_log.jsonl` (url, status, ttfb, size).
  - Use robust HTTP client with retry/backoff.
  - Log timing metrics precisely.
  - Store raw HTML in structured folders.
- [ ] HTML→text normalization: remove boilerplate; preserve H1–H3; keep lists as sentences.
  - Identify boilerplate patterns to strip (ads, navbars).
  - Parse HTML headings and convert to markdown or plain text.
  - Flatten lists while preserving semantic meaning.
- [ ] Store normalized docs under `data/corpus/` with metadata (`url, title, published_at, updated_at, language, source_type`).
  - Extract metadata reliably from HTML or HTTP headers.
  - Use consistent file naming scheme (e.g., doc_id).
  - Validate metadata completeness.
- [ ] **Dedup**: canonical URL and normalized-text hash; skip duplicates.
  - Implement canonicalization rules for URLs.
  - Compute and store text hashes.
  - Check for duplicates before saving.
- DoD: ≥20 documents normalized, metadata present; determinism verified on repeat run.  
- Artifacts: `data/raw_html/`, `data/corpus/`, logs.

---

## 3) Chunking & local retrieval (BM25/TF-IDF)
**Goal:** Prepare chunk-level units and a simple local search as the baseline retriever.
- [ ] Chunk size ≈ 400–800 tokens with 10–15% overlap; cut on semantic boundaries (headings/paragraphs).
  - Tokenize text consistently (specify tokenizer).
  - Detect semantic boundaries heuristically.
  - Calculate overlap and adjust chunk boundaries.
- [ ] Emit JSONL entries `{doc_id, chunk_id, url, title, section, text, language, source_type, published_at, tokens}`.
  - Assign unique chunk IDs per document.
  - Include all required metadata fields.
  - Validate JSONL formatting and encoding.
- [ ] Build **RetrievalAdapter** interface (index/search/stats) and implement **LocalBM25Adapter**.
  - Define interface methods and expected behaviors.
  - Integrate existing BM25 library or implement from scratch.
  - Provide indexing and query APIs.
- [ ] Add payload-like filters: date cutoff, domain allowlist, language, source_type.
  - Design filter syntax and application logic.
  - Test filter combinations for correctness.
  - Expose filters via retrieval config.
- DoD: Given a sample query, top-k returns sensible chunks from ≥2 distinct domains.  
- Artifacts: `data/chunks/*.jsonl`, `data/indexes/*`, retrieval config.

---

## 4) Subgraph per subtopic (map-style execution)
**Goal:** Run each subtopic in isolation with its **local state** slice; enable parallelism later.
- [ ] `topic_planner` yields 3–7 subtopics with rationale/keywords per subtopic (stored in state).
  - Implement topic segmentation logic.
  - Generate rationale text for each subtopic.
  - Store subtopics in global state.
- [ ] Implement a dispatcher that maps `subtopics[]` to subgraph invocations.
  - Design dispatch API with input/output contracts.
  - Support parallel or sequential invocation modes.
  - Handle error propagation per subgraph.
- [ ] Subgraph skeleton stages: `retrieve_candidates_local → rank_and_compose → draft_section(stream)`.
  - Implement stub nodes for each stage.
  - Define data flow and intermediate state.
  - Prepare streaming draft output mechanism.
- [ ] Add anti-loop limits and step counters in local state.
  - Track number of iterations per section.
  - Enforce maximum allowed steps.
  - Log warnings on limit breaches.
- DoD: Each subtopic produces at least a streaming draft of 2–3 paragraphs.  
- Artifacts: Subgraph compiled and callable; dispatcher logic.

---

## 5) Draft generation with streaming & citations
**Goal:** Surface partial results quickly and enforce citation scaffolding.
- [ ] Stream paragraphs into `sections[i].draft_chunks[]` as they are produced.
  - Implement incremental output buffering.
  - Update local state atomically.
  - Provide hooks for UI or logs to consume streams.
- [ ] Enforce citation markers like `[doc_id#chunk_id]` inline in the draft.
  - Define citation syntax and placement rules.
  - Validate citations against available chunks.
  - Prevent citation duplication or omission.
- [ ] Enforce diversity: not more than N adjacent chunks from the same doc.
  - Track document provenance per chunk.
  - Implement checks during draft composition.
  - Adjust chunk selection to maintain diversity.
- DoD: UI/logs show streamed paragraphs with proper citation markers.  
- Artifacts: Streaming hooks/logs, citation policy doc.

---

## 6) Local fact-checking & confidence model
**Goal:** Verify key claims against **independent** chunks and compute a numeric confidence.
- [ ] Extract “checkable claims” from a draft (simple heuristic + prompts).
  - Define heuristics for claim detection (e.g., sentences with facts).
  - Optionally use prompt-based extraction via LLM.
  - Store extracted claims in local state.
- [ ] Re-query local index per claim; require ≥2 distinct domains for “strong” facts.
  - Formulate queries from claims.
  - Retrieve supporting chunks with domain metadata.
  - Count distinct domains and assess support strength.
- [ ] Compute `confidence = f(coverage, domain_diversity, recency)` with weights in config.
  - Define mathematical formula or model.
  - Normalize input factors.
  - Allow configurable weights per factor.
- [ ] Annotate `critic_notes[]` for gaps/contradictions; tag risky paragraphs.
  - Detect missing evidence or conflicting info.
  - Add structured notes to local state.
  - Flag paragraphs needing revision or human review.
- DoD: Confidence is reproducible on the same corpus; claims without support are flagged.  
- Artifacts: Confidence spec, test fixtures with expected flags.

---

## 7) Critic gate, replanning, and anti-loops
**Goal:** Route by confidence and avoid infinite cycles.
- [ ] Thresholds: `pass ≥ 0.75`, `ask_human 0.4–0.75`, `revise < 0.4` (configurable).
  - Define threshold constants in config files.
  - Implement gating logic based on confidence scores.
  - Log routing decisions for audit.
- [ ] `revise`: expand/alter retrieval (synonyms, k↑, relax filters); regenerate only problematic paragraphs.
  - Identify paragraphs with low confidence.
  - Adjust retrieval parameters dynamically.
  - Trigger selective re-drafting of affected text.
- [ ] Hard limits: max 2 revisions per section; global step ceiling per run.
  - Track revision counts per section.
  - Enforce global step counters.
  - Fail gracefully if limits exceeded.
- DoD: On low-evidence topics the system revises up to limit and exits gracefully.  
- Artifacts: Routing policy, revision strategies list.

---

## 8) Human-in-the-loop (HiTL)
**Goal:** Minimal yet effective human review when the model is unsure.
- [ ] Present: disputed claims, pro/contra sources, 1–2 rewrite options.
  - Design data format for review payload.
  - Include evidence summaries and claim context.
  - Provide alternative draft snippets for comparison.
- [ ] Actions: approve/rewrite/omit; update `needs_human`, `confidence`, and draft.
  - Define action handlers and state updates.
  - Persist reviewer inputs atomically.
  - Reflect changes in downstream processing.
- [ ] Persist reviewer decisions for audit.
  - Store decisions in durable storage (e.g., JSON or DB).
  - Link decisions to run IDs and sections.
  - Enable retrieval for later analysis.
- DoD: “Gray zone” routes through HiTL; state reflects decisions correctly.  
- Artifacts: Review UX sketch (even CLI), decision log format.

---

## 9) Persistent memory & checkpointing
**Goal:** Reuse prior knowledge and be resilient to failures.
- [ ] Implement persistent memory store (`data/memory/`): `good_sources`, `bad_domains`, `review_patterns`, historic confidence.
  - Design schema for memory artifacts.
  - Provide read/write APIs.
  - Ensure concurrency safety.
- [ ] Read priors at retrieval time (e.g., boost good domains).
  - Integrate memory lookups into retrieval filters or scoring.
  - Test impact on retrieval results.
  - Allow dynamic updates during runs.
- [ ] Enable LangGraph checkpointer; verify idempotency of nodes.
  - Hook checkpointing into graph execution lifecycle.
  - Test restart scenarios.
  - Detect and avoid duplicated side effects.
- DoD: Restart mid-run resumes from last checkpoint; new runs use priors.  
- Artifacts: Memory files/DB, checkpoint snapshots, resume test logs.

---

## 10) Telemetry, metrics, and golden tests
**Goal:** Observe quality and prevent regressions.
- [ ] Metrics: domain diversity, recall@k on hand-labeled golden queries, fact-coverage, contradiction-rate, avg-confidence, human-calls rate.
  - Define metric calculation formulas.
  - Instrument code to collect relevant data points.
  - Aggregate metrics per run and per section.
- [ ] Structured logs with `run_id`/`section` tags; simple dashboards (even CSV).
  - Standardize log formats for metrics.
  - Implement export to CSV or JSON.
  - Provide scripts or templates for dashboards.
- [ ] Golden set: 5–10 curated queries + expected evidence.
  - Select representative queries.
  - Annotate expected outputs and evidence chunks.
  - Automate test runs and result comparisons.
- DoD: Metrics computed per run; golden tests pass consistently.  
- Artifacts: `tests/e2e/` cases, metrics CSVs, evaluation script spec.

---

## 11) Final assembly & report
**Goal:** Ship a coherent research report with citations and limitations.
- [ ] `aggregate_sections → polish_and_citations → final_output`.
  - Implement section aggregation logic.
  - Apply formatting and citation insertion rules.
  - Generate final output document(s).
- [ ] Produce artifacts: `report.md` (executive summary + sections + citations), `sources.csv`, `limitations.md`.
  - Export report with consistent styles.
  - Compile source metadata into CSV.
  - Summarize known limitations transparently.
- [ ] Verify style and structure (consistent headings, citation format).
  - Validate markdown syntax.
  - Check citation numbering and cross-references.
  - Enforce style guide rules.
- DoD: Deterministic report generated from the same corpus and query.  
- Artifacts: Final docs under `outputs/` (create folder).

---

## 12) E2E & fault-tolerance tests
**Goal:** Ensure stability across failures and restarts.
- [ ] Inject failures (network/parsing) and verify resume via checkpoints.
  - Simulate network timeouts and errors.
  - Corrupt or truncate intermediate data.
  - Confirm checkpoint recovery correctness.
- [ ] Verify idempotency (re-running nodes doesn’t duplicate data).
  - Re-run graph nodes multiple times.
  - Check for duplicated chunks or artifacts.
  - Assert state consistency.
- [ ] Load test small batches (3–5 parallel subtopics).
  - Run parallel executions.
  - Monitor resource usage and latency.
  - Detect race conditions or deadlocks.
- DoD: E2E passes with simulated faults; no data corruption/duplication.  
- Artifacts: Fault-injection scenarios, logs.

---

## 13) Prepare for Qdrant RAG (later)
**Goal:** Make the swap-in seamless.
- [ ] Finalize **RetrievalAdapter** contract (`index/search/stats`) and config parity.
  - Document interface methods and parameters.
  - Ensure config options map to Qdrant equivalents.
  - Define error handling and fallback behaviors.
- [ ] Document mapping from local chunk schema → Qdrant payload (domain, date, lang, source_type, section).
  - Specify field names and types.
  - Describe any transformations or normalizations.
  - Provide examples for payload construction.
- [ ] Migration checklist: create collection, upsert chunks, switch search to Qdrant, keep filters identical.
  - Outline step-by-step migration procedure.
  - Validate data integrity post-migration.
  - Test search equivalence before full switch.
- DoD: Dry-run spec complete; only adapter code would change at RAG time.  
- Artifacts: Adapter spec doc, mapping table, migration checklist.

---

## 14) Extensions backlog (optional)
- Hybrid retrieval (dense + sparse) when on Qdrant.  
- Date fading/recency priors.  
- Per-paragraph evidence list in final report.  
- UI panel for HiTL with side-by-side evidence.  
- Multi-language support and translation consistency checks.  

---

## Quick daily/weekly execution hint (no dates)
- **Ring #1 (Data):** crawl → parse → chunk → index (small corpus first).  
- **Ring #2 (Reasoning):** retrieve → compose → draft (stream) → fact-check → critic → HiTL → finalize.  
Ship Ring #2 on a micro-corpus, then scale Ring #1.

---

## Checklists (copy into issues)
**Idempotency**: inputs hashed, dedup safe, re-run doesn’t duplicate chunks.  
**Anti-loops**: per-section revisions ≤ 2, global step cap works.  
**Diversity**: at least 2 domains per section context.  
**Confidence**: reproducible; supported facts ≥ 2 distinct domains.  
**HiTL**: decisions persisted; confidence updated accordingly.  
**Resume**: restart mid-run resumes at correct node without side effects.
