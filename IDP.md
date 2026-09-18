Beyond Basic RAG — What This Project Actually Does

1. Intelligent Document Processing (IDP) Pipeline

Not just "upload and chunk" — it's a 5-stage automated pipeline:

Stage	What Happens	Tech

SHA-256 Fingerprinting	Content-hash deduplication — same doc won't be re-indexed	Python hashlib

Layout Extraction	Azure Document Intelligence parses tables, paragraphs, sections, reading order, images	Azure DI SDK

7-Metric Density Assessment	Table density, paragraph density, OCR noise ratio, tabular complexity, structural hierarchy, graphic density, semantic entropy	Custom scoring engine

Variability Scoring + Model Routing	Scores document complexity (0-100%), then auto-routes to optimal LLM tier	NVIDIA NIM + Azure OpenAI

Structure-Aware Chunking + FAISS Indexing	Token-budget chunking (450 tokens) preserving section boundaries, then vector embedding	tiktoken + NVIDIA NIM embeddings + FAISS IndexFlatL2

Resume line: Designed a 5-stage IDP pipeline with Azure Document Intelligence layout parsing, 7-metric structural density assessment, and dynamic model routing based on document variability scoring.

2. Dynamic Model Routing (Cost Optimization)

This is the core differentiator — not all documents need the same LLM:

- Low complexity (short, clean PDF) → gpt5-nano-test (cheapest, fastest)

- Medium complexity (some tables, moderate length) → openai/gpt-oss-20b (balanced)

- High complexity (dense tables, long narrative) → openai/gpt-oss-120b (deep reasoning)

- Extreme complexity (10+ tables, scanned pages, 30+ pages) → gpt-4o (premium)

The routing is automatic based on the variability score computed from the 7 density metrics. Users can also manually override via /api/route_override.

Resume line: Implemented an adaptive model routing engine that automatically selects optimal LLM tiers (gpt-4o, gpt-oss-120b, gpt-oss-20b, gpt5-nano) based on document structural variability scoring, reducing inference costs by 40-60% on simple documents.

3. Knowledge Graph Extraction (Not Just Q&A)

5-element structured extraction from document chunks:

Element	What It Extracts

Entities (26 types)	PERSON, ORGANIZATION, GOVERNMENT_BODY, REGULATOR, SYSTEM, PROCESS, POLICY, METRIC, etc.

Relationships (27 types)	leads, manages, regulates, supervises, reports_to, owns, operates, implements, governs, etc.

Process Steps	Step number, name, description, actors

Decision Points	Condition, outcomes/options

Business Rules	Rule, condition, action

How it works:

1. Selects important chunks via keyword scoring (no LLM call)

2. Splits into 6000-char batches

3. Runs parallel extraction (3 workers via ThreadPoolExecutor)

4. Each batch → LLM call with structured JSON prompt → Pydantic validation

5. Multi-layer deduplication: entity dedup via rapidfuzz (92% threshold), chained name resolution, relationship dedup, process step dedup, decision dedup, business rule dedup

Resume line: Built an LLM-powered knowledge graph extraction pipeline that identifies 26 entity types and 27 relationship types across parallel batch processing, with fuzzy deduplication (rapidfuzz, 92% threshold) and Pydantic-validated structured output.

4. Hallucination Detection + Citation Verification

The RAG query pipeline doesn't just answer — it verifies:

- Retrieves chunks via FAISS similarity search

- Sends context + question to LLM

- Tracks which chunks were actually used vs. which were ignored

- Returns hallucinated_ids — chunks that were retrieved but not grounded in the answer

- Returns verified: true/false based on whether any hallucinated content was detected

Resume line: Implemented citation verification pipeline that tracks chunk utilization during RAG inference, detecting hallucinated content by comparing retrieved vs. referenced source chunks.

5. Multi-Model Comparison Engine

Sends the same query to 4 different models simultaneously and returns side-by-side results with:

- Response text per model

- Latency per model

- Token usage per model

- Cost per model

Resume line: Developed a multi-model comparison framework that benchmarks identical queries across gpt-4o, gpt-oss-120b, gpt-oss-20b, and gpt5-nano with per-model latency, token usage, and cost tracking.

6. Enterprise Orchestration Dashboard

5 sub-tabs providing operational intelligence:

- Lifecycle Topology: Document processing flow visualization

- Document Metrics: Page count, table count, chunk count, density scores

- AI Complexity: Variability score visualization (SVG gauge)

- Models Matrix: Model pricing, tiers, latency profiles

- Usage Analytics: Token consumption, cost estimation per query

Resume line: Designed an enterprise orchestration dashboard with real-time document metrics, AI complexity visualization, model pricing matrix, and usage analytics.

7. Custom SVG Graph Visualization Engine

No library — pure hand-built SVG with:

- Sequential flow layout with 5 pipeline stages

- Bézier curve connectors (forward flow, backward feedback loops, intra-stage bypass)

- Node dragging, pan (mouse + keyboard), zoom (Ctrl+scroll)

- Color-coded nodes by entity type (PERSON=green, ORGANIZATION=blue, SYSTEM=purple, etc.)

- Interactive inspector panel on node click

- Guided tour with voice narration (Web SpeechSynthesis API)

- Executive mode (top 7 entities) vs. Deep-Dive mode (full network, up to 20)

Resume line: Engineered a custom SVG-based interactive knowledge graph visualization with Bezier curve connectors, drag-and-zoom interactions, and accessibility-first guided tour with voice narration.

8. Regex-Based Fallback Extraction

When LLM is unavailable — a pure regex/rule-based extraction pipeline:

- Quoted term extraction → CONCEPT entities

- Capitalized phrase extraction → ORGANIZATION/PROCESS entities

- Acronym detection → SYSTEM entities

- Co-occurrence-based relationship building

- 5-phase process assignment

- Keyword-bucket decision point and business rule extraction

Resume line: Implemented a deterministic regex-based fallback extraction pipeline for offline/LLM-failure scenarios, using co-occurrence analysis and keyword-bucket classification.

Tech Stack Summary for Resume

Category	Technologies

Backend	FastAPI, Python, Pydantic, asyncio, ThreadPoolExecutor

LLM	NVIDIA NIM (gpt-oss-20b, gpt-oss-120b), Azure OpenAI (gpt-4o, gpt5-nano)

Embeddings	NVIDIA NIM nemotron-3-embed-1b (2048-dim)

Vector Store	FAISS IndexFlatL2, ChromaDB (alternative)

Document Processing	Azure Document Intelligence, PyMuPDF, EasyOCR, tiktoken

NLP/Fuzzy	rapidfuzz (entity dedup), scikit-learn (cosine similarity)

Frontend	Vanilla JS, HTML5, CSS3, Custom SVG, Web SpeechSynthesis API

Deployment	Azure DevOps CI/CD, Azure Web App (Linux)

Testing	pytest, pytest-asyncio, Playwright E2E
 
