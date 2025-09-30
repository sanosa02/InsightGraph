# InsightGraph: High-Level Project Overview

## Project Summary
InsightGraph is a **self-correcting research assistant** built on top of LangGraph.  
It is designed to take a user’s query, automatically gather relevant information from the web, process and verify the findings, and generate structured research reports.  
The system is built to be **modular, resilient, and extendable**, with the ability to integrate RAG (Retrieval-Augmented Generation) and vector storage (e.g., Qdrant) at later stages.

---

## Core Goals
- Automate research workflow: from crawling → parsing → chunking → retrieval → drafting → fact-checking → reporting.
- Demonstrate advanced LangGraph concepts:
  - **Global vs Local State** management.
  - **Sub-graphs** for per-topic workflows.
  - **Streaming outputs** with partial state updates.
  - **Critic/self-correction** loops.
  - **Human-in-the-loop** for edge cases.
  - **Checkpointing** for fault tolerance.
  - **Persistent memory** for cross-run learning.
- Architected for **seamless upgrade to RAG** with Qdrant.

---

## System Architecture (High-Level)

1. **Data Ring (Web Intake)**
   - Crawl allowed websites, save raw HTML.
   - Normalize content: extract clean text, headings, metadata.
   - Chunk text into manageable pieces (400–800 tokens with overlap).
   - Store normalized documents and chunks.

2. **Reasoning Ring (Graph Workflow)**
   - **Topic Planner**: expand user query into subtopics.
   - **Subgraph per subtopic**: retrieve relevant chunks, draft sections, fact-check, apply critic, ask human if needed.
   - **Aggregation**: compile all sections into a coherent report.
   - **Polish & Citations**: enforce style, add references, export report.

3. **Resilience & Learning**
   - Checkpoint system state after every node for fault tolerance.
   - Persistent memory: record good/bad sources, domain reputations, review patterns.
   - Metrics & telemetry: coverage, contradictions, confidence scores, human interventions.

---

## Key Features
- **End-to-end automation** of research reports.
- **Streamed draft generation** with inline citations.
- **Fact-checking module** requiring multiple independent sources per claim.
- **Critic node** with configurable thresholds and revision loops.
- **Human-in-the-loop** decision points for uncertain outputs.
- **Configurable filters** (domains, dates, languages, source types).

---

## Future Extensions
- Plug-in **Qdrant RAG adapter** for vector search (dense + sparse hybrid retrieval).
- Multi-language support and translation consistency checks.
- UI for human review with evidence side-by-side.
- Advanced analytics: domain diversity, recency fading, contradiction heatmaps.

---

## Project Value
- Serves as a **pet project** to learn advanced LangGraph patterns.
- Demonstrates real-world workflow automation with modular design.
- Provides a sandbox to experiment with **retrieval, reasoning, and resilience** strategies.
