1. 7-Metric Density Assessment

This happens in server.py:328-369 after Azure Document Intelligence extracts the layout. The raw counts come from core/azure_di.py:

Metric	How It's Calculated

Table Density	table_count / page_count → "X tables/page (Y total)"

Paragraph Density	paragraph_count / page_count → "X paras/page (Y total)"

OCR Noise Ratio	(ocr_pages / page_count) * 100 → "X% scanned (Y/Z pages)"

Tabular Complexity	total_cells = sum(row_count * column_count for each table)

Structural Hierarchy	section_count from Azure DI sections

Graphic Density	image_count from page.get_images() per page

Semantic Entropy	Formula: min(96, max(20, round((avg_tables * 15) + (avg_paras * 0.8) + (ocr_ratio * 0.3))))

The raw data sources (core/azure_di.py:36-123):

- PyMuPDF (fitz.open) for local fallback metrics

- Azure Document Intelligence prebuilt-layout model for structured extraction

- page.find_tables().tables → table count

- page.get_text("blocks") → paragraph count

- page.get_text("text", strip=True) < 50 chars → marks page as OCR-needed

- page.get_images() → image count

Resume line: Computed 7 structural density metrics from Azure Document Intelligence layout analysis — table density, paragraph density, OCR noise ratio, tabular complexity (cell-level), structural hierarchy, graphic density, and semantic entropy index.

2. Variability Score Generation

This happens in pipelines/ingest.py:803-853. It's a two-layer system:

Layer 1: LLM-Based Scoring (primary)

pipelines/ingest.py:analyze_document_variability()

1. Takes 4 inputs: paragraph_count, section_count, azure_page_count, azure_table_count

2. Builds a prompt (build_variability_prompt) with these metrics + scoring guidance (ranges: 20-40 simple, 40-60 moderate, 60-80 complex, 80-100 extreme)

3. Sends to NVIDIA NIM (openai/gpt-oss-20b) with json_mode=True, temperature=0

4. Parses JSON response: {variability_score, variability_reason, recommended_model, model_reason}

5. If LLM fails → falls through to Layer 2

Layer 2: Deterministic Fallback (server.py:440-450)

raw_calc = (avg_tables * 14) + (avg_paras * 0.7) + (ocr_ratio * 0.35)

variability_score = int(min(98, max(12, round(raw_calc))))

Then hardcoded routing rules:

- table_count > 10 or ocr_pages > 4 or page_count > 30 → gpt-4o

- table_count > 4 or paragraph_count > 40 → gpt-oss-120b

- page_count > 3 or table_count > 1 → gpt-oss-20b

- else → gpt5-nano-test

The routing decision then maps score → model:

Score Range	Model

20-40	gpt5-nano-test

40-60	openai/gpt-oss-20b

60-80	openai/gpt-oss-120b

80-100	gpt-4o

Tech: NVIDIA NIM API (OpenAI SDK compatible) → /v1/chat/completions with response_format: {type: "json_object"}

Resume line: Built a two-layer variability scoring engine — LLM-based analysis via NVIDIA NIM with structured JSON output, with deterministic formula fallback (tables*14 + paras*0.7 + ocr*0.35) for offline resilience, routing documents to optimal LLM tiers.

3. Chunking Logic

This is in core/chunker.py. It's structure-aware, token-budget chunking:

Step 1: Page Extraction (core/extractor.py)

- PyMuPDF opens PDF

- Per page: page.find_tables() → table chunks (markdown format)

- Per page: page.get_text("text", sort=True) → text chunks

- Pages with <50 chars → flagged needs_ocr=True

- Tables are atomic (never split)

Step 2: Token-Based Chunking (core/chunker.py:66-188)

max_tokens = 450  (under embedding model's 512 limit)

overlap_tokens = 50  (for context continuity)

Algorithm:

1. Split text on double newlines → paragraphs

2. Heading merging: If a paragraph is a short single line with no trailing punctuation → attach it to the next paragraph (headings stay with their content)

3. For each paragraph:

- If para_tokens > 450 → flush current chunk, then split oversized paragraph

- If current_tokens + para_tokens > 450 → emit current chunk, build overlap from last 50 tokens of current chunk, start new chunk

- Else → append to current chunk

4. Oversized paragraph splitting (_split_oversized):

- First try splitting by newlines (line-by-line)

- Then try splitting by sentences (.  delimiter)

- Last resort: return as-is

Step 3: Document-Level Chunking (chunk_document)

- Iterates all pages

- Tables → atomic chunks (never split, chunk_type="table")

- Text → passed through chunk_text()

- OCR text → same as text but tagged chunk_type="ocr"

Resume line: Implemented structure-aware chunking with 450-token budget, 50-token overlap, heading-attachment heuristics, and atomic table preservation, using tiktoken cl100k_base encoding.

4. Knowledge Graph Structured Extraction

This is the most complex part. Two parallel systems exist:

System A: LLM-Based Extraction (pipelines/graph.py)

Pipeline (16 steps):

1. Get all chunks from FAISS: vectorstore.get_all_by_doc(doc_id)

2. Select important chunks (_select_important_chunks):

- Keyword scoring: 27 keywords like "process", "workflow", "system", "organization", "role", etc.

- Each keyword hit = +10 points

- Sentence count (max 5) + length bonus (1 if >500 chars)

- Top 70 chunks selected, restored to document order

3. Split into batches (_split_chunks_into_batches):

- Max 6000 chars per batch

- Large chunks split mid-text

4. Parallel extraction (3 workers via ThreadPoolExecutor):

- Each batch → one LLM call

- Prompt from prompts/graph_extract.txt with __GRAPH_CONTEXT__ placeholder

- json_mode=True, temperature=0.0, max_tokens=7000

5. Parse response: Clean markdown fences → isolate JSON → GraphExtraction.model_validate(data) (Pydantic)

6. Merge successful batches (failed batches skipped, no retry)

7. Entity deduplication (deduplicate_entities):

- Group by type (PERSON, ORGANIZATION, etc.)

- Within each type: rapidfuzz.fuzz.token_sort_ratio with threshold 92%

- Longer name wins as canonical

- Chained name resolution (A→B→C collapses to A→C)

8. Apply canonical names to all relationships, process step actors

9. Relationship dedup: Unique (source, target, relation) tuples

10. Process step dedup: Unique (step_number, name, description) tuples

11. Decision point dedup: Unique (name, description) tuples

12. Business rule dedup: Unique (name, condition, action) tuples

The 5 Extraction Elements

What the LLM extracts per batch (from prompts/graph_extract.txt):

{

  "entities": [{"name": "", "type": "ENTITY_TYPE", "description": ""}],

  "relationships": [{"source": "", "target": "", "relation": "", "description": ""}],

  "process_steps": [{"step": 1, "name": "", "description": "", "actors": [""]}],

  "decision_points": [{"condition": "", "description": "", "outcomes": [""]}],

  "business_rules": [{"rule": "", "description": "", "condition": "", "action": ""}]

}

Limits: 100 entities, 200 relationships, 50 steps, 30 decisions, 30 rules per batch.

Pydantic Models (pipelines/graph.py:261-358):

- Entity(name, type, description)

- Relationship(source, target, relation, description)

- ProcessStep(step_number, name, description, actors)

- DecisionPoint(name, description, options)

- BusinessRule(name, description, condition, action)

System B: Regex-Based Fallback (server.py:1144-1249)

When LLM is unavailable:

- Quoted terms: "Term" or 'Term' → CONCEPT entities

- Capitalized phrases: Word Word Word (2-4 words, title case) → ORGANIZATION/PROCESS

- Acronyms: NATO, FIFA → SYSTEM entities

- Known first names: lookup table of 100+ names → PERSON entities

- Blacklist filtering: 80+ stopwords to exclude

- Currency/table notation filtering: 170+ codes to exclude

- Relationships via co-occurrence in same sentence

Tech stack: Pydantic (validation), rapidfuzz (fuzzy dedup), ThreadPoolExecutor (parallel), NVIDIA NIM (LLM), tiktoken (token counting)

Resume line: Engineered a dual-mode knowledge graph extraction pipeline — LLM-based parallel extraction (3 workers, 6000-char batches) with Pydantic-validated structured output across 26 entity types and 27 relationship types, plus multi-layer deduplication (rapidfuzz 92% threshold, chained name resolution), with deterministic regex fallback for offline scenarios.
 
