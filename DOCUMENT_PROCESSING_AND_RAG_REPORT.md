# FLTR Document Processing & RAG Architecture Report

This document provides a comprehensive overview of the document processing pipeline and RAG (Retrieval Augmented Generation) techniques used in the FLTR application.

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Document Processing Pipeline](#document-processing-pipeline)
3. [Chunking Strategies](#chunking-strategies)
4. [Multimodal Processing](#multimodal-processing)
5. [RAG Query Techniques](#rag-query-techniques)
6. [Vector Storage & Retrieval](#vector-storage--retrieval)
7. [Configuration Reference](#configuration-reference)
8. [Architecture Diagrams](#architecture-diagrams)

---

## Executive Summary

FLTR implements a sophisticated document processing and retrieval system with the following key capabilities:

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **PDF Parsing** | PyMuPDF4LLM + Docling | Extract text & images with semantic structure |
| **Image Processing** | RapidOCR + Vision Models | Text extraction & visual understanding |
| **Chunking** | Semantic + Parent-Child | Structure-aware document splitting |
| **Embeddings** | OpenAI text-embedding-3-small | 1536-dim vectors for semantic search |
| **Vector Store** | Milvus/Zilliz | Fast semantic + keyword search |
| **Reranking** | Cohere rerank-english-v3.0 | Neural relevance scoring |
| **Image Storage** | Cloudflare R2 | Presigned URL access |

---

## Document Processing Pipeline

### Pipeline Stages

The document processing pipeline consists of **6 sequential stages** with checkpoint resumption capability:

```
1. DOWNLOAD  → Fetch file from R2 storage
2. PARSE     → Extract text and images (PyMuPDF4LLM/Docling)
3. CHUNK     → Split into semantic chunks
4. EMBED     → Generate vector embeddings
5. STORE     → Save to Milvus + metadata to PostgreSQL
6. DB_UPDATE → Create DocumentAsset records, mark ready
```

**Key Files:**
- `modal/services/pipeline_orchestrator.py` - Main orchestration with checkpoints
- `modal/services/document_processor.py` - High-level parsing & chunking API
- `modal/services/parsing/document_parsers.py` - Format-specific parsers

### PDF Parsing

**Primary Parser: PyMuPDF4LLM**
- Preserves semantic document structure (headers, sections)
- Extracts images with page tracking
- Generates clean markdown output
- DPI: 120 (optimized for cost vs quality)

**Fallback Parser: Docling**
- Enhanced image extraction at 2.0x scale
- Used when PyMuPDF4LLM fails
- Generates picture-level images

**PDF Validation & Repair:**
- Validates PDF headers (checks first 1024 bytes per PDF spec)
- Repairs offset headers by removing leading garbage bytes
- Fixes MediaBox issues for proper page dimensions
- Uses PyPDF2 for structural validation

### Checkpoint System

Processing can resume from any failed stage:

```
Checkpoints saved to R2:
├── {dataset_id}/{document_id}/parsed.json    - Parsed document
├── {dataset_id}/{document_id}/chunks.json    - Chunk data
└── {dataset_id}/{document_id}/embeddings.json - Vector embeddings
```

---

## Chunking Strategies

FLTR uses a **Strategy Pattern** with 4 chunking approaches:

### 1. Parent-Child Strategy (Default)

Creates hierarchical chunks for optimal retrieval:

```
Document → Semantic Sections → Parent Chunks (3000 chars)
                                     ↓
                               Child Chunks (1000 chars)

Search Flow: Query matches Children → Return Parents to LLM
```

**Configuration:**
```python
parent_chunk_size = 3000    # Context unit (returned to LLM)
child_chunk_size = 1000     # Search unit (matched against query)
child_overlap = 200         # 20% overlap between children
use_semantic = True         # Structure-aware splitting
```

**Benefits:**
- Children provide precise semantic matching
- Parents provide full context for LLM consumption
- Reduces context fragmentation

### 2. Semantic Strategy

Structure-aware splitting that respects document hierarchy:

- Detects markdown headers (#, ##, ###)
- Keeps content between headers together
- Breaks at paragraph boundaries
- Tracks section context (parent_header, level)

**Best for:** Technical documentation, manuals, structured reports

### 3. Multimodal Strategy

Combines text and images intelligently:

1. Extract text chunks using semantic chunking
2. Replace image references with `[IMAGE]` placeholders
3. Create separate **image chunks** with:
   - Vision model description
   - OCR extracted text
   - Image classification
   - Surrounding context
4. Link images to text chunks by position

**Image Chunk Content:**
```
Image Description: <vision model output>
Text from Image: <OCR extracted text>
Context: <surrounding text 500 chars before/after>
```

### 4. Simple Strategy (Fallback)

Fixed-size chunking without structure awareness:
- Basic sliding window approach
- Used when other strategies fail

### Strategy Selection

```python
Priority:
1. Parent-Child (if no images) ← Most common
2. Multimodal (if images present)
3. Semantic (if enabled)
4. Simple (fallback)
```

---

## Multimodal Processing

### Image Extraction

**PyMuPDF4LLM** extracts images during markdown generation:
- DPI: 120 (reduced from 150 for cost optimization)
- Saves to temp directory, later moved to R2
- Preserves page structure and semantic context

**R2 Key Structure:**
```
{dataset_id}/{document_id}/images/{page}_{index}.{ext}
Example: abc123/doc456/images/0_0.png
```

### Vision Model Integration

**Supported Providers:**

| Provider | Model | Use Case |
|----------|-------|----------|
| **Gemini** | gemini-2.0-flash | Primary (fast, cost-effective) |
| **GPT-4V** | gpt-4o | Fallback (reliable) |
| **Claude** | claude-3-5-sonnet | Alternative |

**Vision Configuration:**
```python
VisionConfig:
├── provider: VisionProvider.GEMINI_VISION
├── fallback_provider: VisionProvider.GPT4_VISION
├── generate_descriptions: True
├── classify_image_type: True
├── extract_structured_data: False  # Disabled for cost
├── max_tokens: 300
├── temperature: 0.1  # Low for consistency
├── ocr_first: True   # Try OCR before vision
└── ocr_confidence_threshold: 0.7
```

**Image Type Classification:**
```python
ImageType: chart_bar, chart_line, chart_pie, table,
           diagram, photo, screenshot, illustration, other
```

### OCR Processing

**Engine:** RapidOCR (ONNX Runtime)
- Fast local OCR processing
- Returns: (extracted_text, confidence_score)
- Used for text-heavy images

**Cost Optimization (OCR-First):**
```
Run OCR and Vision in PARALLEL (1.5-2x speedup)
    ↓
If OCR confidence ≥ 0.7:
    Skip expensive vision processing
    Saves 40-60% on vision API costs
```

### Image Filtering

Removes low-value images before upload:

```python
# ALWAYS KEEP (high-value types):
chart_bar, chart_line, chart_pie, table, diagram, flowchart

# FILTER OUT:
├── Cover images (page ≤ 1, index = 0)
├── Low confidence (OCR < 0.3 AND Vision < 0.5)
├── Decorative types with low confidence
└── Tiny images (<5KB or <100x100px)
```

### Parallel Image Processing

Images are processed in parallel using Modal workers:
- **Batch size:** 15 images per worker
- **Concurrency:** Up to 100 containers
- **Fallback:** Sequential processing on worker failure

---

## RAG Query Techniques

### Search Flow

```
User Query
    ↓
[Query Parser] - Classify type, extract filters
    ↓
[Query Expansion]
├── Multi-Query (3 LLM-generated variations)
├── HyDE (hypothetical answer embedding)
└── Image Query Enhancement (deterministic)
    ↓
[Embedding] - text-embedding-3-small (1536-dim)
    ↓
[Search Strategy]
├── Vector Search (semantic similarity)
├── Keyword Search (BM25)
└── Hybrid Search (RRF merge)
    ↓
[Reranking] - Cohere neural reranker
    ↓
[Image Filtering] - Threshold-based
    ↓
Response with presigned image URLs
```

### Search Strategies

#### Vector Search
- Pure semantic similarity using cosine distance
- Fast, good for conceptual queries
- Limitations: May miss exact keyword matches

#### Keyword Search (BM25)
- Text-based matching on keywords field
- 97 stop words removed
- Best for exact term matching

#### Hybrid Search (RRF)
Combines vector + keyword using Reciprocal Rank Fusion:

```python
RRF_K = 60  # Standard from literature

score(doc) = Σ 1/(k + rank_i + 1)
```

**Benefits:**
- Balances semantic and exact matching
- Better recall through diverse sources
- Parameter-free (no tuning needed)

### Query Enhancement

#### Multi-Query Expansion
- LLM generates 3 diverse query variations
- Different phrasings, definitions, perspectives
- Model: gpt-4o-mini, Temperature: 0.7

#### HyDE (Hypothetical Document Embeddings)
- Generate ideal answer, then embed it
- Better for complex/analytical queries
- Triggered by: "explain", "describe", query length > 3 words

#### Image Query Enhancement
Deterministic enhancement for visual queries:
```python
"Key herbs" → adds "basil illustration"
"Pasta tools" → adds "pasta roller illustration"
```

### Reranking

**Model:** Cohere rerank-english-v3.0

**Two-Stage Process:**
```
Stage 1: Fetch more candidates
├── limit=5 → fetch 15 results (3x multiplier)
└── Max fetch: 100 results

Stage 2: Neural reranking
├── Pass query + docs to Cohere API
├── Get relevance scores (0.0-1.0)
└── Return top-k by rerank_score
```

**Image Filtering Post-Rerank:**
- Text chunks: Keep all passing reranking
- Image chunks: Only keep if rerank_score ≥ 0.7 OR distance ≤ 0.3
- Prevents decorative images from dominating results

---

## Vector Storage & Retrieval

### Milvus Collection Schema

**Collection:** `fltr_documents` (shared across all datasets)

| Field | Type | Purpose |
|-------|------|---------|
| `id` | INT64 | Auto-incremented primary key |
| `vector` | FLOAT_VECTOR(1536) | OpenAI embedding |
| `text` | VARCHAR(65535) | Chunk content |
| `dataset_id` | VARCHAR(64) | Dataset filter |
| `document_id` | VARCHAR(64) | Source document |
| `filename` | VARCHAR(512) | Original file |
| `chunk_type` | VARCHAR(16) | text/image/ocr |
| `chunk_index` | INT64 | Sequence number |
| `page` | INT64 | Page number |
| `image_index` | INT64 | Image position |
| `image_r2_key` | VARCHAR(512) | R2 storage path |
| `ocr_confidence` | FLOAT | OCR quality (0-1) |
| `keywords` | VARCHAR(2000) | BM25 indexed |
| `parent_id` | INT64 (nullable) | Parent chunk reference |

**Indexes:**
- Vector: FLAT index, COSINE metric
- Keywords: INVERTED index for BM25

### Parent-Child Retrieval

```
Search on child chunks (dense, specific)
    ↓
Match query to child embeddings
    ↓
Fetch parent chunks via parent_id
    ↓
Return larger parent context to LLM
```

---

## Configuration Reference

### Chunking Configuration

```python
# modal/services/chunking/chunking_config.py

parent_chunk_size = 3000
parent_overlap = 200
child_chunk_size = 1000
child_overlap = 200
semantic_buffer_size = 1
semantic_breakpoint_percentile = 95
max_keywords = 15
min_keyword_length = 3
max_summary_length = 200
```

### Search Configuration

```python
# fastapi/config/search_constants.py

# RRF
RRF_K = 60

# Reranking
RERANK_FETCH_MULTIPLIER = 3
RERANK_MAX_FETCH = 100
RERANK_MODEL = "rerank-english-v3.0"

# Embeddings
EMBEDDING_MODEL = "text-embedding-3-small"
EMBEDDING_DIMENSION = 1536

# Query Processing
QUERY_PARSER_MODEL = "gpt-4o-mini"
MULTI_QUERY_MODEL = "gpt-4o-mini"
HYDE_MODEL = "gpt-4o-mini"

# Image Thresholds
MIN_OCR_CONFIDENCE = 0.3
MIN_VISION_CONFIDENCE_DECORATIVE = 0.5
MIN_VISION_CONFIDENCE_NO_OCR = 0.6
COVER_IMAGE_MAX_PAGE = 1
```

### Vision Configuration

```python
# Default vision config

provider = VisionProvider.GEMINI_VISION
fallback_provider = VisionProvider.GPT4_VISION
generate_descriptions = True
classify_image_type = True
extract_structured_data = False
max_tokens = 300
temperature = 0.1
min_confidence = 0.5
ocr_first = True
ocr_confidence_threshold = 0.7
```

---

## Architecture Diagrams

### Document Processing Flow

```
User Upload
    ↓
modal_app.py (webhook)
    ↓
pipeline_orchestrator.run_pipeline()
    ├─ _download_stage()
    │   └─ R2.download_document()
    │
    ├─ _parse_stage()
    │   └─ DocumentParserFactory.parse_document()
    │       └─ PDFParser.parse()
    │           ├─ validate_pdf()
    │           └─ MultimodalProcessor.process_pdf_with_context()
    │               ├─ pymupdf4llm.to_markdown()
    │               ├─ [Parallel: OCR + Vision per image]
    │               └─ upload_image_to_r2()
    │
    ├─ _chunk_stage()
    │   └─ ChunkingStrategyFactory.chunk_document()
    │       ├─ ParentChildChunkingStrategy
    │       ├─ MultimodalChunkingStrategy
    │       └─ SemanticChunkingStrategy
    │
    ├─ _embed_stage()
    │   └─ OpenAI text-embedding-3-small
    │
    ├─ _store_stage()
    │   └─ store_in_milvus()
    │
    └─ _db_update_stage()
        ├─ create_document_assets()
        └─ mark_document_ready()
```

### RAG Query Flow

```
User Query
    ↓
/mcp/query/{dataset_id}/search
    ↓
┌─────────────────────────────────────┐
│ Query Enhancement                    │
├─────────────────────────────────────┤
│ ├─ Query Parser (classify, filter)  │
│ ├─ Multi-Query Expansion (3 vars)   │
│ ├─ HyDE (hypothetical answer)       │
│ └─ Image Query Enhancement          │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ Embedding                           │
├─────────────────────────────────────┤
│ text-embedding-3-small → 1536-dim   │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ Search (Hybrid by default)          │
├─────────────────────────────────────┤
│ Vector Search ──┐                   │
│                 ├─→ RRF Merge       │
│ Keyword Search ─┘                   │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ Reranking (Cohere)                  │
├─────────────────────────────────────┤
│ ├─ Fetch 3x candidates              │
│ ├─ Neural relevance scoring         │
│ └─ Image filtering (threshold 0.7)  │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ Response Formatting                 │
├─────────────────────────────────────┤
│ ├─ Generate presigned image URLs    │
│ ├─ Format as MCP response           │
│ └─ Include metadata                 │
└─────────────────────────────────────┘
    ↓
Response: {
  contexts: [...],
  total_results: N,
  reranking_used: true
}
```

### Image Processing Pipeline

```
PDF Document
    ↓
PyMuPDF4LLM Extract
    ↓
┌─────────────────────────┐
│ For each image:         │
├─────────────────────────┤
│                         │
│   ┌─────────┐ ┌───────┐ │
│   │   OCR   │ │Vision │ │ ← Parallel
│   │RapidOCR │ │Gemini │ │
│   └────┬────┘ └───┬───┘ │
│        │          │     │
│        └────┬─────┘     │
│             ↓           │
│     Filter Decision     │
│    (cover/decorative)   │
│             ↓           │
│      Upload to R2       │
│             ↓           │
│    Create Image Chunk   │
│   (description + OCR)   │
└─────────────────────────┘
    ↓
Embed with text-embedding-3-small
    ↓
Store in Milvus (chunk_type: "image")
```

---

## Key Files Reference

| Category | File | Purpose |
|----------|------|---------|
| **Orchestration** | `modal/services/pipeline_orchestrator.py` | Main pipeline with checkpoints |
| **Parsing** | `modal/services/parsing/document_parsers.py` | PDF/document extraction |
| **Multimodal** | `modal/services/multimodal_processor.py` | Image extraction & vision |
| **Vision** | `modal/services/vision_service.py` | Vision model integration |
| **OCR** | `modal/services/ocr_service.py` | RapidOCR text extraction |
| **Chunking** | `modal/services/chunking/strategies/` | All chunking strategies |
| **Search** | `fastapi/services/embedding_service.py` | Core search pipeline |
| **Hybrid** | `fastapi/services/hybrid_search_service.py` | RRF merging |
| **Strategies** | `fastapi/services/search_strategies/` | Search strategy implementations |
| **Query** | `fastapi/services/query/` | Multi-query, HyDE, parsing |
| **Vector DB** | `fastapi/database/vector_store.py` | Milvus schema & operations |
| **Constants** | `fastapi/config/search_constants.py` | All configuration constants |
| **MCP API** | `fastapi/routers/mcp.py` | Query endpoint orchestration |

---

## Performance Considerations

### Processing Optimizations

1. **Parallel Image Processing**: 15 images per Modal worker, up to 100 containers
2. **OCR-First Strategy**: Saves 40-60% on vision API costs
3. **Checkpoint Resume**: Restart from any failed stage
4. **DPI Reduction**: 120 DPI (from 150) reduces image size

### Search Optimizations

1. **Hybrid Search**: Better recall than vector-only
2. **Reranking**: Fetch 3x candidates, return best k
3. **Parent-Child**: Search on children, return parent context
4. **Image Filtering**: Strict thresholds prevent noise

### Cost Optimizations

1. **OCR before Vision**: Skip vision if OCR confidence ≥ 0.7
2. **Gemini primary**: Cheaper than GPT-4V with good quality
3. **Rerank multiplier**: 3x fetch balances quality vs API calls
4. **Image filtering**: Remove decorative images before upload

---

*Generated: December 2024*
*Version: 1.0*
