# FLTR Feature Parity Roadmap: Reaching Ragie-Level Capabilities

**Document Version:** 1.0
**Last Updated:** 2025-11-09
**Status:** Planning Phase

---

## Executive Summary

This document outlines a comprehensive roadmap to bring FLTR to feature parity with Ragie's advanced RAG capabilities while maintaining our competitive advantages (open-source, cost-optimized, MCP-native architecture).

**Current State:**
- ✅ Production-ready document processing (PDF, DOCX, etc.)
- ✅ Excellent PDF validation/repair
- ✅ Cost-optimized serverless architecture (Modal + R2)
- ✅ MCP (Model Context Protocol) native
- ⚠️ Basic fixed-size chunking
- ⚠️ Single embedding provider (OpenAI)
- ❌ No multimodal support
- ❌ No data connectors
- ❌ No GraphRAG

**Target State (6-9 months):**
- ✅ Advanced semantic chunking with context preservation
- ✅ Multimodal RAG (text, images, tables, audio, video)
- ✅ GraphRAG for relationship-aware retrieval
- ✅ 50+ data connectors via LlamaHub
- ✅ Hybrid search (vector + keyword + graph)
- ✅ Enterprise features (SSO, audit logs, analytics)

---

## Table of Contents

1. [Gap Analysis: FLTR vs Ragie](#gap-analysis-fltr-vs-ragie)
2. [Phase 1: Foundation (Months 1-2)](#phase-1-foundation-months-1-2)
3. [Phase 2: Advanced Chunking & Multimodal (Months 3-4)](#phase-2-advanced-chunking--multimodal-months-3-4)
4. [Phase 3: GraphRAG & Knowledge Graphs (Months 5-6)](#phase-3-graphrag--knowledge-graphs-months-5-6)
5. [Phase 4: Data Connectors & Integrations (Months 7-8)](#phase-4-data-connectors--integrations-months-7-8)
6. [Phase 5: Enterprise & Polish (Month 9)](#phase-5-enterprise--polish-month-9)
7. [Technical Specifications](#technical-specifications)
8. [LlamaHub Integration Strategy](#llamahub-integration-strategy)
9. [GraphRAG Architecture](#graphrag-architecture)
10. [Cost Analysis](#cost-analysis)
11. [Success Metrics](#success-metrics)

---

## Gap Analysis: FLTR vs Ragie

### Legend
- ✅ Feature Complete
- ⚠️ Partial Implementation
- ❌ Not Implemented
- 🚀 Competitive Advantage

| Category | Feature | FLTR Status | Ragie Status | Priority |
|----------|---------|-------------|--------------|----------|
| **Parsing** | PDF, DOCX, PPTX, XLSX | ✅ Docling | ✅ | - |
| | PDF Repair/Validation | ✅ Excellent | ✅ | - |
| | HTML, Markdown, TXT | ✅ | ✅ | - |
| | JSON, CSV structured data | ⚠️ CSV only | ✅ | Medium |
| | Audio transcription | ❌ | ✅ | High |
| | Video processing | ❌ | ✅ | High |
| | Web scraping | ❌ | ✅ | High |
| **Chunking** | Fixed-size chunking | ✅ | ✅ | - |
| | Semantic/Sentence-aware | ❌ | ✅ | **CRITICAL** |
| | Token-based limits | ❌ | ✅ | High |
| | Table-aware chunking | ❌ | ✅ | High |
| | Code-aware chunking | ❌ | ✅ | Medium |
| | Markdown structure preservation | ❌ | ✅ | High |
| | Sliding window with overlap | ✅ | ✅ | - |
| | Parent-child relationships | ❌ | ✅ | Medium |
| **Embeddings** | OpenAI embeddings | ✅ | ✅ | - |
| | Multiple embedding providers | ❌ | ✅ | High |
| | Local/self-hosted models | ❌ | ✅ | Medium |
| | Multimodal embeddings (CLIP) | ❌ | ✅ | High |
| | Fine-tuned embeddings | ❌ | ⚠️ | Low |
| **Indexing** | Vector index (Milvus) | ✅ | ✅ | - |
| | Keyword index (BM25) | ❌ | ✅ | High |
| | Summary index | ❌ | ✅ | Medium |
| | Graph index (relationships) | ❌ | ⚠️ | **CRITICAL** |
| | Metadata filtering | ✅ | ✅ | - |
| **Retrieval** | Semantic vector search | ✅ | ✅ | - |
| | Hybrid search (vector+keyword) | ❌ | ✅ | High |
| | Two-pass retrieval + reranking | ❌ | ✅ | High |
| | GraphRAG traversal | ❌ | ⚠️ | **CRITICAL** |
| | Contextual compression | ❌ | ✅ | Medium |
| **Multimodal** | Image extraction | ⚠️ OCR ready | ✅ | High |
| | Image description/captioning | ❌ | ✅ | High |
| | Table extraction/structure | ⚠️ Docling | ✅ | High |
| | Diagram understanding | ❌ | ✅ | Medium |
| | Audio transcription (Whisper) | ❌ | ✅ | High |
| | Video frame analysis | ❌ | ✅ | Medium |
| **Data Sources** | Direct file upload | ✅ | ✅ | - |
| | Notion connector | ❌ | ✅ | High |
| | Google Drive connector | ❌ | ✅ | High |
| | Slack connector | ❌ | ✅ | Medium |
| | Database connectors | ❌ | ✅ | Medium |
| | Web scraping/crawling | ❌ | ✅ | High |
| | GitHub connector | ❌ | ✅ | Medium |
| | Confluence connector | ❌ | ✅ | Medium |
| | OneDrive connector | ❌ | ✅ | Medium |
| **Features** | Document versioning | ❌ | ✅ | Low |
| | Real-time progress updates | ❌ | ✅ | Medium |
| | Analytics dashboard | ❌ | ✅ | Medium |
| | Usage tracking | ✅ Credits | ✅ | - |
| | SSO/SAML | ❌ | ✅ | Low |
| | Audit logs | ❌ | ✅ | Low |
| | Webhook notifications | ⚠️ R2 only | ✅ | Medium |
| | Batch operations | ⚠️ Partial | ✅ | Medium |
| **Architecture** | Open-source/self-hosted | 🚀 Yes | ❌ No | - |
| | MCP native | 🚀 Yes | ❌ No | - |
| | Cost-optimized (Modal+R2) | 🚀 Yes | ⚠️ Unknown | - |
| | Serverless scaling | 🚀 Yes | ✅ | - |

---

## Phase 1: Foundation (Months 1-2)

**Goal:** Build the infrastructure for advanced chunking, hybrid search, and multimodal support.

### 1.1 Chunking Engine Refactor

**Current State:**
```python
# modal/services/document_processor.py
chunk_size = 1000  # Hardcoded
chunk_overlap = 200  # Hardcoded
```

**Target Architecture:**
```python
# New: modal/services/chunking/
├── base.py              # ChunkerInterface, ChunkingStrategy enum
├── semantic.py          # Sentence/paragraph-aware chunking
├── token_based.py       # Token-aware chunking (tiktoken)
├── table_aware.py       # Keep tables intact
├── code_aware.py        # Preserve code blocks
├── markdown_aware.py    # Respect markdown structure
└── factory.py           # ChunkerFactory.create(strategy, config)
```

**Implementation Tasks:**
- [ ] Design `ChunkerInterface` with extensibility in mind
- [ ] Implement `SemanticChunker` using NLTK/spaCy for sentence boundaries
- [ ] Implement `TokenBasedChunker` using tiktoken (OpenAI tokenizer)
- [ ] Implement `TableAwareChunker` to detect and preserve tables from Docling
- [ ] Implement `MarkdownAwareChunker` using markdown-it-py parser
- [ ] Create `ChunkerFactory` for strategy selection
- [ ] Add chunking strategy to Dataset model (per-dataset configuration)
- [ ] Update Modal `chunk_document()` to use new system
- [ ] Add comprehensive tests for each chunker
- [ ] Benchmark performance (latency, quality metrics)

**Key Libraries:**
- `tiktoken` - Token counting for OpenAI models
- `nltk` or `spaCy` - Sentence tokenization
- `markdown-it-py` - Markdown parsing
- `langchain-text-splitters` (optional) - Pre-built chunkers

**Configuration Schema:**
```python
class ChunkingConfig(BaseModel):
    strategy: ChunkingStrategy  # semantic, token, table_aware, code_aware, markdown
    max_chunk_size: int = 1000  # In tokens (not characters)
    chunk_overlap: int = 200     # In tokens
    min_chunk_size: int = 100
    respect_document_boundaries: bool = True
    preserve_tables: bool = True
    preserve_code_blocks: bool = True
    sentence_splitter: str = "nltk"  # nltk, spacy, none
```

**Deliverables:**
- Pluggable chunking system with 5+ strategies
- Per-dataset chunking configuration
- Migration path for existing datasets (re-chunk)
- Performance benchmarks (quality + speed)

**Estimated Time:** 3 weeks

---

### 1.2 Hybrid Search Infrastructure

**Goal:** Add keyword search (BM25) alongside vector search for better recall.

**New Components:**
```python
# fastapi/services/search/
├── base.py              # SearchInterface
├── vector_search.py     # Current Milvus search (refactored)
├── keyword_search.py    # BM25 using Elasticsearch or Milvus Scalar Index
├── hybrid_search.py     # Combine vector + keyword with RRF (Reciprocal Rank Fusion)
└── reranker.py          # Cross-encoder reranking (optional)
```

**Implementation Options:**

**Option A: Elasticsearch** (Better keyword search, more infrastructure)
- Add Elasticsearch 8.x to stack
- Index documents alongside Milvus
- Use ES BM25 for keyword search
- Combine results with Reciprocal Rank Fusion (RRF)
- **Cost:** ~$20/mo for managed ES (Elastic Cloud)

**Option B: Milvus Scalar Index** (Simpler, same infrastructure)
- Use Milvus's built-in scalar indexing (INVERTED index)
- Store keywords in separate Milvus field
- Run keyword + vector search in parallel
- Combine with RRF
- **Cost:** $0 additional (uses existing Milvus)

**Recommendation: Option B (Milvus Scalar Index)** - Simpler, no new infrastructure

**Implementation Tasks:**
- [ ] Add `keywords` field to Milvus schema (VARCHAR, INVERTED index)
- [ ] Extract keywords during chunking (TF-IDF or KeyBERT)
- [ ] Implement `KeywordSearchService` using Milvus scalar search
- [ ] Implement `HybridSearchService` with RRF algorithm
- [ ] Add `search_mode` parameter to MCP endpoints (vector, keyword, hybrid)
- [ ] Add reranker (optional): Cross-encoder with `sentence-transformers`
- [ ] Update credit system (hybrid search = 1.5x credits?)
- [ ] Benchmarks: hybrid vs vector-only on BEIR dataset

**RRF Algorithm:**
```python
def reciprocal_rank_fusion(vector_results, keyword_results, k=60):
    """Combine rankings using RRF"""
    scores = defaultdict(float)
    for rank, result in enumerate(vector_results):
        scores[result.id] += 1 / (k + rank + 1)
    for rank, result in enumerate(keyword_results):
        scores[result.id] += 1 / (k + rank + 1)
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)
```

**Deliverables:**
- BM25 keyword search working in Milvus
- Hybrid search with RRF
- Optional cross-encoder reranking
- Benchmarks showing improvement over vector-only

**Estimated Time:** 2 weeks

---

### 1.3 Multiple Embedding Providers

**Goal:** Support multiple embedding models with fallback and per-dataset configuration.

**New Architecture:**
```python
# modal/services/embeddings/
├── base.py              # EmbeddingProvider interface
├── openai.py            # OpenAIEmbeddingProvider
├── voyage.py            # VoyageAI (better quality)
├── cohere.py            # Cohere Embed v3 (multilingual)
├── local.py             # Local models via sentence-transformers
├── multimodal.py        # CLIP for image+text embeddings
└── factory.py           # EmbeddingProviderFactory
```

**Supported Providers:**
1. **OpenAI** (current) - `text-embedding-3-small`, `text-embedding-3-large`
2. **Voyage AI** - `voyage-large-2`, `voyage-code-2` (better for code)
3. **Cohere** - `embed-multilingual-v3.0` (105 languages)
4. **Local** - `sentence-transformers/all-MiniLM-L6-v2` (free, fast, 384 dims)
5. **Multimodal** - `openai/clip-vit-large-patch14` (text + images)

**Implementation Tasks:**
- [ ] Design `EmbeddingProvider` interface with `embed()` method
- [ ] Implement providers for OpenAI, Voyage, Cohere, local
- [ ] Add `embedding_provider` and `embedding_model` to Dataset model
- [ ] Update Modal to select provider per dataset
- [ ] Add dimension validation (ensure Milvus collection matches)
- [ ] Implement fallback chain (primary → secondary → local)
- [ ] Add embedding model benchmarks (quality, speed, cost)
- [ ] Update documentation with provider comparison

**Configuration:**
```python
class EmbeddingConfig(BaseModel):
    provider: str  # openai, voyage, cohere, local, clip
    model: str     # e.g., "text-embedding-3-small"
    dimension: int  # 384, 1536, 1024, etc.
    fallback_provider: Optional[str] = "local"
    batch_size: int = 100
```

**Deliverables:**
- 5+ embedding providers supported
- Per-dataset provider selection
- Automatic fallback on API failures
- Cost/quality comparison table

**Estimated Time:** 2 weeks

---

## Phase 2: Advanced Chunking & Multimodal (Months 3-4)

**Goal:** Implement semantic chunking and full multimodal support (images, tables, audio).

### 2.1 Semantic Chunking with LangChain

**Using LangChain's battle-tested text splitters:**

```bash
pip install langchain-text-splitters tiktoken
```

**Implementation:**
```python
from langchain_text_splitters import (
    RecursiveCharacterTextSplitter,
    MarkdownHeaderTextSplitter,
    TokenTextSplitter,
    SemanticChunker
)

# Semantic chunking with sentence boundaries
semantic_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    length_function=len,
    separators=["\n\n", "\n", ". ", " ", ""],  # Respect boundaries
    is_separator_regex=False,
)

# Token-based chunking (OpenAI token limits)
token_splitter = TokenTextSplitter(
    chunk_size=512,  # Tokens, not characters
    chunk_overlap=50,
    encoding_name="cl100k_base",  # GPT-3.5/4 tokenizer
)

# Markdown-aware chunking (keeps headers with content)
markdown_splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=[
        ("#", "Header 1"),
        ("##", "Header 2"),
        ("###", "Header 3"),
    ]
)
```

**Implementation Tasks:**
- [ ] Integrate LangChain text splitters into `ChunkerFactory`
- [ ] Implement `SemanticChunker` using `RecursiveCharacterTextSplitter`
- [ ] Implement `TokenBasedChunker` using `TokenTextSplitter`
- [ ] Implement `MarkdownChunker` using `MarkdownHeaderTextSplitter`
- [ ] Add metadata preservation (headers, section context)
- [ ] Test on diverse documents (legal, code, markdown, etc.)
- [ ] Benchmark quality improvements vs fixed-size chunking

**Deliverables:**
- Production-ready semantic chunking
- Token-aware chunking (no more token limit errors)
- Markdown structure preservation
- Quality metrics showing improvement

**Estimated Time:** 2 weeks

---

### 2.2 Table Extraction & Structured Data

**Goal:** Extract tables from documents and store as structured data + embeddings.

**Approach:**
1. Use Docling's table extraction (already available!)
2. Store tables as both:
   - **Structured JSON** in metadata
   - **Markdown representation** in text chunk
   - **Separate vector embedding** for table description

**Implementation:**
```python
# modal/services/table_extractor.py
from docling.datamodel.document import TableItem

async def extract_tables(docling_result) -> List[Dict]:
    """Extract tables from Docling result"""
    tables = []
    for element in docling_result.document.iterate_items():
        if isinstance(element, TableItem):
            tables.append({
                "markdown": element.export_to_markdown(),
                "data": element.export_to_dataframe().to_dict(),
                "caption": element.caption,
                "page": element.page_number,
            })
    return tables
```

**New Milvus Schema:**
```python
# Add fields to fltr_documents collection
fields = [
    # ... existing fields ...
    "chunk_type": VARCHAR(50),  # "text", "table", "image", "code"
    "table_data": JSON,         # Structured table data (optional)
    "parent_chunk_id": VARCHAR(36),  # For hierarchical retrieval
]
```

**Implementation Tasks:**
- [ ] Extract tables from Docling results
- [ ] Store tables as separate chunks with `chunk_type="table"`
- [ ] Keep tables intact (don't split across chunks)
- [ ] Generate table descriptions using LLM (GPT-4-mini)
- [ ] Embed both table markdown + description
- [ ] Add table-specific retrieval mode
- [ ] UI: Render tables in search results

**Deliverables:**
- Tables extracted and stored separately
- Tables rendered properly in UI
- Table-aware search and retrieval

**Estimated Time:** 2 weeks

---

### 2.3 Image Processing & OCR

**Goal:** Extract images, run OCR, generate descriptions, embed visually.

**Architecture:**
```python
# modal/services/multimodal/
├── image_extractor.py   # Extract images from PDFs
├── ocr_service.py       # RapidOCR + Tesseract fallback
├── image_captioner.py   # GPT-4-Vision or CLIP
└── multimodal_chunker.py
```

**Implementation:**
```python
# Use RapidOCR (already in dependencies!)
from rapidocr_onnxruntime import RapidOCR

async def extract_text_from_image(image_bytes: bytes) -> str:
    """OCR with RapidOCR"""
    ocr = RapidOCR()
    result, elapse = ocr(image_bytes)
    return " ".join([line[1] for line in result])

# Generate image descriptions
async def describe_image(image_bytes: bytes) -> str:
    """Generate image description using GPT-4-Vision"""
    import openai
    client = openai.OpenAI()

    response = client.chat.completions.create(
        model="gpt-4-vision-preview",
        messages=[{
            "role": "user",
            "content": [
                {"type": "text", "text": "Describe this image in detail for RAG retrieval."},
                {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{base64_image}"}}
            ]
        }],
        max_tokens=300
    )
    return response.choices[0].message.content
```

**Implementation Tasks:**
- [ ] Extract images from PDFs using `pymupdf` or Docling
- [ ] Run OCR on images using RapidOCR
- [ ] Generate image descriptions using GPT-4-Vision
- [ ] Store images in R2 with references in Milvus
- [ ] Embed image descriptions alongside text
- [ ] Add `chunk_type="image"` with image URL in metadata
- [ ] UI: Display images in search results

**Deliverables:**
- Images extracted from PDFs
- OCR text + AI-generated descriptions
- Images searchable via descriptions
- Images displayed in UI

**Estimated Time:** 2 weeks

---

### 2.4 Audio & Video Processing

**Goal:** Transcribe audio/video, extract keyframes, index temporal data.

**Services:**
```python
# modal/services/multimodal/
├── audio_transcriber.py  # Whisper API or local Whisper
├── video_processor.py    # Extract keyframes + audio
└── temporal_chunker.py   # Time-aware chunking
```

**Implementation:**
```python
# Transcribe audio with Whisper
import openai

async def transcribe_audio(audio_bytes: bytes) -> Dict:
    """Transcribe using OpenAI Whisper API"""
    client = openai.OpenAI()

    transcription = client.audio.transcriptions.create(
        model="whisper-1",
        file=audio_bytes,
        response_format="verbose_json",  # Includes timestamps
    )

    return {
        "text": transcription.text,
        "segments": transcription.segments,  # Time-stamped segments
        "language": transcription.language,
    }

# Extract video keyframes
import cv2
async def extract_keyframes(video_path: str, interval_seconds: int = 10) -> List[bytes]:
    """Extract keyframes every N seconds"""
    video = cv2.VideoCapture(video_path)
    fps = video.get(cv2.CAP_PROP_FPS)
    frames = []

    frame_interval = int(fps * interval_seconds)
    frame_count = 0

    while True:
        ret, frame = video.read()
        if not ret:
            break
        if frame_count % frame_interval == 0:
            _, buffer = cv2.imencode('.jpg', frame)
            frames.append(buffer.tobytes())
        frame_count += 1

    return frames
```

**Temporal Chunking:**
```python
# Chunk transcripts by time segments
def chunk_transcript_by_time(segments: List[Dict], max_duration: int = 60):
    """Group transcript segments into ~60 second chunks"""
    chunks = []
    current_chunk = []
    current_duration = 0

    for segment in segments:
        duration = segment["end"] - segment["start"]
        if current_duration + duration > max_duration and current_chunk:
            chunks.append(current_chunk)
            current_chunk = [segment]
            current_duration = duration
        else:
            current_chunk.append(segment)
            current_duration += duration

    if current_chunk:
        chunks.append(current_chunk)

    return chunks
```

**Implementation Tasks:**
- [ ] Add audio file support (MP3, WAV, M4A)
- [ ] Add video file support (MP4, MOV, AVI)
- [ ] Transcribe audio using Whisper API
- [ ] Extract keyframes from video (every 10 seconds)
- [ ] Chunk transcripts by time segments
- [ ] Describe keyframes using GPT-4-Vision
- [ ] Store with temporal metadata (timestamps)
- [ ] UI: Video player with transcript sync

**Deliverables:**
- Audio transcription with timestamps
- Video keyframe extraction + descriptions
- Temporal search (find by time range)
- Video player with synced transcript

**Estimated Time:** 3 weeks

---

## Phase 3: GraphRAG & Knowledge Graphs (Months 5-6)

**Goal:** Implement GraphRAG for relationship-aware retrieval and multi-hop reasoning.

### 3.1 GraphRAG Overview

**What is GraphRAG?**
GraphRAG combines traditional RAG with knowledge graph traversal to find related information through entity relationships, not just semantic similarity.

**Example:**
- **Traditional RAG:** "Who is the CEO of OpenAI?" → Returns chunks mentioning "OpenAI CEO"
- **GraphRAG:** Traverses: `OpenAI` --[CEO]--> `Sam Altman` --[Co-founded]--> `Y Combinator` --[Invested in]--> `Reddit`

**Benefits:**
- Multi-hop reasoning (find information 2-3 steps away)
- Relationship-aware retrieval (find connected entities)
- Better for complex queries requiring context
- Improved citation/provenance tracking

**Architecture:**

```
Document → Entity Extraction → Knowledge Graph → Graph Index → Graph Traversal → Context
                   ↓                    ↓
              (GPT-4-mini)         (Neo4j or NetworkX)
```

---

### 3.2 Knowledge Graph Construction

**Option A: Neo4j** (Dedicated graph database - recommended)
- Native graph database
- Cypher query language
- Excellent visualization
- Managed cloud: $65/mo for Aura DB

**Option B: NetworkX + PostgreSQL** (Cheaper, more work)
- Store graph as adjacency list in PostgreSQL
- Use NetworkX for traversal
- No additional cost
- Less performant for large graphs

**Recommendation: Start with NetworkX + PostgreSQL, migrate to Neo4j when scale demands it**

**Implementation:**

```python
# fastapi/services/knowledge_graph/
├── entity_extractor.py   # Extract entities using LLM
├── relationship_extractor.py  # Extract relationships
├── graph_builder.py      # Build knowledge graph
├── graph_index.py        # Index graph in Neo4j/NetworkX
└── graph_traversal.py    # Query and traverse graph
```

**Entity Extraction:**
```python
import openai

async def extract_entities(text: str) -> List[Dict]:
    """Extract named entities using GPT-4-mini"""
    client = openai.OpenAI()

    response = client.chat.completions.create(
        model="gpt-4-mini",
        messages=[{
            "role": "system",
            "content": """Extract named entities from the text. Return as JSON:
            [
                {"entity": "OpenAI", "type": "Organization"},
                {"entity": "Sam Altman", "type": "Person"},
                ...
            ]"""
        }, {
            "role": "user",
            "content": text
        }],
        response_format={"type": "json_object"}
    )

    return json.loads(response.choices[0].message.content)

async def extract_relationships(text: str, entities: List[Dict]) -> List[Dict]:
    """Extract relationships between entities"""
    response = client.chat.completions.create(
        model="gpt-4-mini",
        messages=[{
            "role": "system",
            "content": f"""Extract relationships between these entities: {entities}
            Return as JSON:
            [
                {{"source": "OpenAI", "relation": "CEO", "target": "Sam Altman"}},
                ...
            ]"""
        }, {
            "role": "user",
            "content": text
        }],
        response_format={"type": "json_object"}
    )

    return json.loads(response.choices[0].message.content)
```

**Graph Storage (PostgreSQL schema):**
```sql
-- Nodes (entities)
CREATE TABLE graph_nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id UUID REFERENCES datasets(id) ON DELETE CASCADE,
    entity_name VARCHAR(255) NOT NULL,
    entity_type VARCHAR(100),  -- Person, Organization, Location, etc.
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(dataset_id, entity_name)
);

-- Edges (relationships)
CREATE TABLE graph_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id UUID REFERENCES datasets(id) ON DELETE CASCADE,
    source_id UUID REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_id UUID REFERENCES graph_nodes(id) ON DELETE CASCADE,
    relationship VARCHAR(100) NOT NULL,  -- CEO, Founded, Invested, etc.
    weight FLOAT DEFAULT 1.0,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Document-Entity links (provenance)
CREATE TABLE document_entities (
    document_id UUID REFERENCES documents(id) ON DELETE CASCADE,
    node_id UUID REFERENCES graph_nodes(id) ON DELETE CASCADE,
    chunk_index INT,
    PRIMARY KEY (document_id, node_id)
);

CREATE INDEX idx_graph_nodes_dataset ON graph_nodes(dataset_id);
CREATE INDEX idx_graph_edges_dataset ON graph_edges(dataset_id);
CREATE INDEX idx_graph_edges_source ON graph_edges(source_id);
CREATE INDEX idx_graph_edges_target ON graph_edges(target_id);
```

**Implementation Tasks:**
- [ ] Design graph schema (nodes, edges, provenance)
- [ ] Implement entity extraction using GPT-4-mini
- [ ] Implement relationship extraction
- [ ] Build knowledge graph during document processing
- [ ] Store graph in PostgreSQL (or Neo4j)
- [ ] Add graph visualization endpoint (D3.js or vis.js)
- [ ] Implement graph traversal algorithms (BFS, DFS, shortest path)
- [ ] Add graph statistics to dataset (num_nodes, num_edges, etc.)

**Deliverables:**
- Knowledge graph built from documents
- Entity and relationship extraction working
- Graph stored in database
- Basic visualization in UI

**Estimated Time:** 3 weeks

---

### 3.3 GraphRAG Retrieval

**Implementation:**
```python
# fastapi/services/search/graph_search.py

class GraphRAGService:
    async def search(
        self,
        query: str,
        dataset_id: str,
        max_hops: int = 2,
        limit: int = 10
    ) -> List[Dict]:
        """
        GraphRAG retrieval with multi-hop traversal

        1. Extract query entities
        2. Find matching graph nodes
        3. Traverse relationships (up to max_hops)
        4. Retrieve chunks from connected nodes
        5. Rank by relevance + graph distance
        """

        # Step 1: Extract entities from query
        query_entities = await self.extract_entities(query)

        # Step 2: Find matching nodes in graph
        matching_nodes = await self.find_nodes(dataset_id, query_entities)

        # Step 3: Traverse graph to find related entities
        related_nodes = await self.traverse_graph(
            matching_nodes,
            max_hops=max_hops
        )

        # Step 4: Get document chunks for all nodes
        chunks = await self.get_chunks_for_nodes(related_nodes)

        # Step 5: Rank by relevance + graph distance
        ranked_chunks = self.rank_by_relevance_and_distance(
            query, chunks, matching_nodes
        )

        return ranked_chunks[:limit]

    async def traverse_graph(
        self,
        start_nodes: List[UUID],
        max_hops: int = 2
    ) -> List[Dict]:
        """BFS traversal up to max_hops away"""
        visited = set()
        queue = [(node, 0) for node in start_nodes]  # (node_id, distance)
        results = []

        while queue:
            node_id, distance = queue.pop(0)

            if node_id in visited or distance > max_hops:
                continue

            visited.add(node_id)
            results.append({"node_id": node_id, "distance": distance})

            # Get neighbors
            neighbors = await self.get_neighbors(node_id)
            for neighbor in neighbors:
                queue.append((neighbor["id"], distance + 1))

        return results
```

**Ranking Strategy:**
```python
def rank_by_relevance_and_distance(
    query: str,
    chunks: List[Dict],
    start_nodes: List[UUID]
) -> List[Dict]:
    """
    Score = (0.6 * semantic_similarity) + (0.4 * graph_proximity)

    - semantic_similarity: Cosine similarity from vector search
    - graph_proximity: 1 / (1 + graph_distance)
    """
    scored_chunks = []

    for chunk in chunks:
        # Get vector similarity
        semantic_score = chunk["relevance_score"]  # From Milvus

        # Get graph distance to query entities
        graph_distance = chunk["graph_distance"]  # From traversal
        graph_score = 1.0 / (1.0 + graph_distance)

        # Combined score
        final_score = 0.6 * semantic_score + 0.4 * graph_score

        chunk["final_score"] = final_score
        scored_chunks.append(chunk)

    return sorted(scored_chunks, key=lambda x: x["final_score"], reverse=True)
```

**Implementation Tasks:**
- [ ] Implement `GraphRAGService` with traversal
- [ ] Add entity extraction to queries
- [ ] Implement multi-hop graph traversal (BFS)
- [ ] Combine vector similarity with graph proximity
- [ ] Add new MCP endpoint: `/mcp/graph-query/{dataset_id}`
- [ ] Add graph explainability (show traversal path)
- [ ] Benchmark: GraphRAG vs traditional RAG on complex queries
- [ ] UI: Visualize retrieval path in graph

**Deliverables:**
- Working GraphRAG retrieval
- Multi-hop reasoning (2-3 hops)
- Better results on relationship queries
- Graph-based explainability

**Estimated Time:** 3 weeks

---

### 3.4 Community Detection & Summaries (Microsoft GraphRAG)

**Advanced:** Implement Microsoft's GraphRAG community detection for better global understanding.

**Microsoft GraphRAG Approach:**
1. Build knowledge graph from documents
2. Detect communities using Leiden algorithm
3. Generate summaries for each community
4. Use community summaries for high-level queries

**Benefits:**
- Better answers to global questions ("What are the main themes?")
- Hierarchical understanding (community → sub-community)
- Improved efficiency (search summaries first, then details)

**Implementation:**
```python
# pip install python-louvain networkx

import networkx as nx
import community as community_louvain

async def detect_communities(graph: nx.Graph) -> Dict[int, List[str]]:
    """Detect communities using Louvain/Leiden algorithm"""
    partition = community_louvain.best_partition(graph)

    # Group nodes by community
    communities = {}
    for node, comm_id in partition.items():
        if comm_id not in communities:
            communities[comm_id] = []
        communities[comm_id].append(node)

    return communities

async def summarize_community(community_nodes: List[str], dataset_id: str) -> str:
    """Generate summary for a community using LLM"""
    # Get all chunks related to community entities
    chunks = await get_chunks_for_entities(community_nodes, dataset_id)
    combined_text = " ".join([c["text"] for c in chunks[:50]])  # Limit context

    # Generate summary
    response = client.chat.completions.create(
        model="gpt-4-mini",
        messages=[{
            "role": "system",
            "content": "Summarize the main themes and relationships in this community of entities."
        }, {
            "role": "user",
            "content": combined_text
        }],
        max_tokens=500
    )

    return response.choices[0].message.content
```

**Schema Extension:**
```sql
CREATE TABLE graph_communities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id UUID REFERENCES datasets(id) ON DELETE CASCADE,
    community_id INT NOT NULL,
    summary TEXT,
    node_ids UUID[] NOT NULL,  -- Array of node UUIDs
    level INT DEFAULT 0,  -- For hierarchical communities
    created_at TIMESTAMP DEFAULT NOW()
);
```

**Implementation Tasks:**
- [ ] Integrate NetworkX community detection (Louvain)
- [ ] Generate community summaries using GPT-4-mini
- [ ] Store communities in database
- [ ] Add hierarchical community detection (multi-level)
- [ ] Use community summaries for global queries
- [ ] UI: Show community structure and summaries

**Deliverables:**
- Community detection working
- LLM-generated community summaries
- Hierarchical understanding of document corpus
- Better global query answering

**Estimated Time:** 2 weeks

---

## Phase 4: Data Connectors & Integrations (Months 7-8)

**Goal:** Add 50+ data connectors using LlamaHub (LlamaIndex data loaders).

### 4.1 LlamaHub Integration Strategy

**What is LlamaHub?**
LlamaHub is LlamaIndex's collection of 100+ data loaders for various sources (Notion, Google Drive, databases, etc.). They handle authentication, pagination, and data normalization.

**Repository:** https://github.com/run-llama/llama_hub

**Architecture:**
```python
# modal/services/connectors/
├── base.py              # ConnectorInterface
├── llamahub_adapter.py  # Adapter for LlamaHub loaders
├── notion.py            # Notion connector (via LlamaHub)
├── google_drive.py      # Google Drive connector
├── slack.py             # Slack connector
├── github.py            # GitHub connector
└── factory.py           # ConnectorFactory
```

**Installation:**
```bash
pip install llama-index
pip install llama-hub  # Or install individual loaders
```

---

### 4.2 Priority Connectors (Top 10)

**Implementation Order (by user demand):**

1. **Notion** - Most requested
2. **Google Drive** - Enterprise essential
3. **Slack** - Team knowledge
4. **GitHub** - Code repositories
5. **Confluence** - Enterprise wikis
6. **OneDrive** - Microsoft enterprise
7. **Web Scraper** - Public websites
8. **PostgreSQL** - Database connector
9. **Airtable** - No-code databases
10. **Figma** - Design documents

---

### 4.3 Notion Connector Implementation

**Example Implementation:**
```python
# modal/services/connectors/notion.py
from llama_hub.notion import NotionPageReader

class NotionConnector:
    def __init__(self, integration_token: str):
        self.reader = NotionPageReader(integration_token=integration_token)

    async def sync(self, dataset_id: str, config: Dict) -> Dict:
        """
        Sync Notion workspace to dataset

        Args:
            dataset_id: Target dataset
            config: {
                "page_ids": ["page-id-1", "page-id-2"],  # Optional: specific pages
                "database_ids": ["db-id-1"],  # Optional: databases
                "sync_all": False  # Sync entire workspace
            }
        """
        # Load documents from Notion
        if config.get("sync_all"):
            documents = self.reader.load_data()
        elif config.get("page_ids"):
            documents = self.reader.load_data(page_ids=config["page_ids"])
        elif config.get("database_ids"):
            documents = self.reader.load_data(database_id=config["database_ids"][0])

        # Process each document
        results = []
        for doc in documents:
            # Upload to R2
            object_key = f"{dataset_id}/notion/{doc.id_}.md"
            await upload_to_r2(object_key, doc.text.encode())

            # Create document record
            document_id = await create_document(
                dataset_id=dataset_id,
                filename=doc.metadata.get("title", "Untitled"),
                object_key=object_key,
                source="notion",
                source_id=doc.id_,
                metadata=doc.metadata
            )

            # Trigger Modal processing
            await trigger_modal_processing(dataset_id, object_key, document_id)

            results.append({"document_id": document_id, "status": "queued"})

        return {
            "synced": len(results),
            "documents": results
        }
```

**API Endpoint:**
```python
# fastapi/routers/connectors.py

@router.post("/api/v1/datasets/{dataset_id}/connectors/notion/sync")
async def sync_notion(
    dataset_id: UUID,
    config: NotionSyncConfig,
    current_user: User = Depends(get_current_user)
):
    """Sync Notion workspace to dataset"""

    # Get integration token from user credentials
    creds = await get_connector_credentials(current_user.id, "notion")

    connector = NotionConnector(integration_token=creds.access_token)
    result = await connector.sync(dataset_id, config.dict())

    # Deduct credits (1 credit per document synced)
    await deduct_credits(
        user_id=current_user.id,
        amount=result["synced"],
        operation="connector_sync",
        metadata={"connector": "notion", "dataset_id": str(dataset_id)}
    )

    return result
```

**OAuth Flow:**
```python
# Store connector credentials per user
CREATE TABLE connector_credentials (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    connector_name VARCHAR(50) NOT NULL,  -- notion, google_drive, slack, etc.
    access_token TEXT NOT NULL,
    refresh_token TEXT,
    expires_at TIMESTAMP,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(user_id, connector_name)
);

# OAuth endpoints
@router.get("/api/v1/connectors/notion/auth")
async def notion_oauth_start():
    """Redirect to Notion OAuth"""
    return RedirectResponse(f"https://api.notion.com/v1/oauth/authorize?client_id={NOTION_CLIENT_ID}&redirect_uri={REDIRECT_URI}")

@router.get("/api/v1/connectors/notion/callback")
async def notion_oauth_callback(code: str, current_user: User = Depends(get_current_user)):
    """Handle Notion OAuth callback"""
    # Exchange code for token
    token = await exchange_notion_code(code)

    # Store credentials
    await store_connector_credentials(
        user_id=current_user.id,
        connector_name="notion",
        access_token=token["access_token"]
    )

    return {"status": "connected"}
```

**Implementation Tasks:**
- [ ] Install llama-index and llama-hub
- [ ] Design `ConnectorInterface` and adapter pattern
- [ ] Implement Notion connector with OAuth
- [ ] Add connector credentials storage
- [ ] Add sync endpoints for Notion
- [ ] Add periodic sync (cron jobs)
- [ ] UI: Connector configuration page
- [ ] UI: OAuth flow
- [ ] Test with real Notion workspace

**Deliverables:**
- Notion connector working end-to-end
- OAuth authentication flow
- Periodic sync (daily/hourly)
- UI for connector management

**Estimated Time:** 2 weeks

---

### 4.4 Additional Connectors (Batch Implementation)

**Using LlamaHub Loaders:**

```python
# Install specific loaders as needed
pip install llama-hub-google-drive
pip install llama-hub-slack
pip install llama-hub-github

# Implement similar to Notion
from llama_hub.google_drive import GoogleDriveReader
from llama_hub.slack import SlackReader
from llama_hub.github import GithubRepositoryReader
```

**Connector Matrix:**

| Connector | LlamaHub Loader | OAuth Required | Priority | Estimated Time |
|-----------|-----------------|----------------|----------|----------------|
| Notion | `NotionPageReader` | Yes | High | 2 weeks |
| Google Drive | `GoogleDriveReader` | Yes | High | 2 weeks |
| Slack | `SlackReader` | Yes | High | 1 week |
| GitHub | `GithubRepositoryReader` | Yes | Medium | 1 week |
| Confluence | `ConfluenceReader` | Yes | Medium | 1 week |
| OneDrive | `OneDriveReader` | Yes | Medium | 2 weeks |
| Web Scraper | `BeautifulSoupWebReader` | No | High | 1 week |
| PostgreSQL | `DatabaseReader` | No | Medium | 1 week |
| Airtable | `AirtableReader` | Yes | Low | 1 week |
| Figma | `FigmaReader` | Yes | Low | 1 week |

**Batch Implementation Strategy:**
1. Implement 3 connectors at once (Notion, Google Drive, Slack)
2. Build reusable OAuth infrastructure
3. Create connector testing framework
4. Document connector API for community contributions

**Implementation Tasks:**
- [ ] Build generic OAuth handler (reusable across connectors)
- [ ] Create connector testing framework
- [ ] Implement priority connectors (Notion, Drive, Slack)
- [ ] Add connector status dashboard
- [ ] Add sync logs and error handling
- [ ] Document connector development guide
- [ ] Open-source connector SDK for community

**Deliverables:**
- 10+ connectors working
- Reusable OAuth infrastructure
- Connector management UI
- Community contribution guide

**Estimated Time:** 6 weeks (parallel with other Phase 4 tasks)

---

### 4.5 Web Scraper & Crawler

**Goal:** Scrape and index public websites for RAG.

**Implementation:**
```bash
pip install playwright beautifulsoup4 html2text
playwright install chromium
```

```python
# modal/services/connectors/web_scraper.py
from playwright.async_api import async_playwright
from bs4 import BeautifulSoup
import html2text

class WebScraper:
    async def scrape_url(self, url: str) -> Dict:
        """Scrape a single URL"""
        async with async_playwright() as p:
            browser = await p.chromium.launch()
            page = await browser.new_page()

            await page.goto(url, wait_until="networkidle")
            html = await page.content()

            await browser.close()

        # Parse HTML
        soup = BeautifulSoup(html, "html.parser")

        # Remove scripts, styles
        for script in soup(["script", "style"]):
            script.decompose()

        # Convert to markdown
        h = html2text.HTML2Text()
        h.ignore_links = False
        markdown = h.handle(str(soup))

        return {
            "url": url,
            "title": soup.title.string if soup.title else url,
            "content": markdown,
            "metadata": {
                "scraped_at": datetime.utcnow().isoformat(),
                "content_length": len(markdown)
            }
        }

    async def crawl_website(
        self,
        start_url: str,
        max_pages: int = 100,
        same_domain_only: bool = True
    ) -> List[Dict]:
        """Crawl entire website"""
        from urllib.parse import urljoin, urlparse

        visited = set()
        to_visit = [start_url]
        results = []

        domain = urlparse(start_url).netloc

        while to_visit and len(results) < max_pages:
            url = to_visit.pop(0)

            if url in visited:
                continue

            # Check domain restriction
            if same_domain_only and urlparse(url).netloc != domain:
                continue

            try:
                # Scrape page
                page_data = await self.scrape_url(url)
                results.append(page_data)
                visited.add(url)

                # Extract links
                soup = BeautifulSoup(page_data["raw_html"], "html.parser")
                for link in soup.find_all("a", href=True):
                    absolute_url = urljoin(url, link["href"])
                    if absolute_url not in visited:
                        to_visit.append(absolute_url)

            except Exception as e:
                print(f"Failed to scrape {url}: {e}")

        return results
```

**API Endpoint:**
```python
@router.post("/api/v1/datasets/{dataset_id}/connectors/web/scrape")
async def scrape_website(
    dataset_id: UUID,
    config: WebScraperConfig,
    current_user: User = Depends(get_current_user)
):
    """
    Scrape website and add to dataset

    config:
        url: str  # Starting URL
        max_pages: int = 100
        same_domain_only: bool = True
        recursive: bool = True  # Crawl entire site vs single page
    """
    scraper = WebScraper()

    if config.recursive:
        pages = await scraper.crawl_website(
            config.url,
            max_pages=config.max_pages,
            same_domain_only=config.same_domain_only
        )
    else:
        pages = [await scraper.scrape_url(config.url)]

    # Process each page
    for page in pages:
        object_key = f"{dataset_id}/web/{urlparse(page['url']).path}.md"
        await upload_to_r2(object_key, page["content"].encode())

        document_id = await create_document(
            dataset_id=dataset_id,
            filename=page["title"],
            object_key=object_key,
            source="web",
            source_url=page["url"]
        )

        await trigger_modal_processing(dataset_id, object_key, document_id)

    return {"scraped_pages": len(pages)}
```

**Implementation Tasks:**
- [ ] Implement single-page scraper with Playwright
- [ ] Implement recursive crawler
- [ ] Add robots.txt compliance
- [ ] Add rate limiting (respect target server)
- [ ] Add sitemap.xml parsing
- [ ] Handle JavaScript-heavy sites
- [ ] Add scheduled re-scraping (keep fresh)
- [ ] UI: Web scraper configuration

**Deliverables:**
- Single-page and recursive web scraping
- Robots.txt compliance
- Scheduled re-scraping
- Website monitoring (detect changes)

**Estimated Time:** 2 weeks

---

## Phase 5: Enterprise & Polish (Month 9)

**Goal:** Add enterprise features, analytics, and polish the user experience.

### 5.1 Real-time Progress Updates (WebSockets)

**Current:** No visibility into processing progress
**Target:** Real-time updates via WebSockets

**Implementation:**
```python
# fastapi/websockets/processing.py
from fastapi import WebSocket

active_connections: Dict[str, List[WebSocket]] = {}

@app.websocket("/ws/datasets/{dataset_id}/processing")
async def processing_websocket(websocket: WebSocket, dataset_id: str):
    await websocket.accept()

    # Add to active connections
    if dataset_id not in active_connections:
        active_connections[dataset_id] = []
    active_connections[dataset_id].append(websocket)

    try:
        while True:
            await websocket.receive_text()  # Keep alive
    except:
        active_connections[dataset_id].remove(websocket)

# Send progress updates from Modal
async def broadcast_progress(dataset_id: str, progress: Dict):
    """Send progress to all connected clients"""
    if dataset_id in active_connections:
        for ws in active_connections[dataset_id]:
            await ws.send_json(progress)
```

**Modal Integration:**
```python
# In modal_app.py, send progress updates
async def process_document_modal(dataset_id: str, object_key: str, task_id: str):
    # ... existing code ...

    # After each step, send update
    await send_progress_update(dataset_id, task_id, {
        "step": "parsing",
        "progress": 0.2,
        "message": "Parsing document..."
    })

    # ... continue processing ...
```

**UI Component (React):**
```typescript
// Use WebSocket to show real-time progress
const { data, isConnected } = useWebSocket(`/ws/datasets/${datasetId}/processing`)

return (
  <Progress value={data?.progress * 100} />
  <p>{data?.message}</p>
)
```

**Implementation Tasks:**
- [ ] Add WebSocket endpoint for processing updates
- [ ] Send progress from Modal at each step
- [ ] Update Next.js UI to use WebSocket
- [ ] Add progress bars to dataset page
- [ ] Show document-level progress
- [ ] Add error notifications via WebSocket

**Deliverables:**
- Real-time progress updates
- Progress bars in UI
- Better user experience

**Estimated Time:** 1 week

---

### 5.2 Analytics Dashboard

**Metrics to Track:**
- Document processing stats (success rate, avg time)
- Search query analytics (popular queries, avg latency)
- Credit usage over time
- Dataset growth (documents, chunks, vectors)
- Error rates and types
- User activity (MAU, DAU)

**Database Schema:**
```sql
CREATE TABLE analytics_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    event_type VARCHAR(50) NOT NULL,  -- upload, search, error, etc.
    dataset_id UUID,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_analytics_events_user ON analytics_events(user_id);
CREATE INDEX idx_analytics_events_type ON analytics_events(event_type);
CREATE INDEX idx_analytics_events_created ON analytics_events(created_at);
```

**API Endpoints:**
```python
@router.get("/api/v1/analytics/overview")
async def get_analytics_overview(current_user: User = Depends(get_current_user)):
    """Get analytics overview for user"""
    return {
        "total_documents": await count_documents(current_user.id),
        "total_searches": await count_searches(current_user.id),
        "credits_used_this_month": await get_monthly_credit_usage(current_user.id),
        "avg_search_latency": await get_avg_search_latency(current_user.id),
        "most_searched_datasets": await get_popular_datasets(current_user.id, limit=5)
    }
```

**UI:**
- Dashboard with charts (recharts or Chart.js)
- Graphs: Credit usage over time, search volume, document processing rate
- Tables: Recent queries, error logs, popular datasets

**Implementation Tasks:**
- [ ] Design analytics schema
- [ ] Track events throughout codebase
- [ ] Build analytics API endpoints
- [ ] Create analytics dashboard UI
- [ ] Add charts (credit usage, search volume, etc.)
- [ ] Add export to CSV/JSON

**Deliverables:**
- Analytics dashboard in UI
- Credit usage tracking
- Search analytics
- Dataset growth metrics

**Estimated Time:** 2 weeks

---

### 5.3 Enterprise Features

**SSO/SAML Authentication:**
```bash
pip install python-saml3
```

```python
# Support enterprise SSO
@router.post("/api/v1/auth/saml/login")
async def saml_login(saml_response: str):
    """Handle SAML assertion"""
    # Parse SAML response
    # Create/login user
    # Return session
    pass
```

**Audit Logs:**
```sql
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    action VARCHAR(100) NOT NULL,  -- create_dataset, delete_document, etc.
    resource_type VARCHAR(50),
    resource_id UUID,
    ip_address INET,
    user_agent TEXT,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_audit_logs_user ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_created ON audit_logs(created_at);
```

**Role-Based Access Control (RBAC):**
```python
class Role(str, Enum):
    ADMIN = "admin"
    EDITOR = "editor"
    VIEWER = "viewer"

class Permission(str, Enum):
    READ_DATASET = "dataset:read"
    WRITE_DATASET = "dataset:write"
    DELETE_DATASET = "dataset:delete"
    MANAGE_USERS = "user:manage"

# Check permissions
@router.delete("/api/v1/datasets/{dataset_id}")
async def delete_dataset(
    dataset_id: UUID,
    current_user: User = Depends(get_current_user)
):
    if not has_permission(current_user, Permission.DELETE_DATASET):
        raise HTTPException(403, "Insufficient permissions")
    # ... delete dataset ...
```

**Implementation Tasks:**
- [ ] Add SAML/SSO authentication
- [ ] Implement audit logging
- [ ] Add RBAC with roles and permissions
- [ ] Add organization/team management
- [ ] Add usage quotas per organization
- [ ] Add billing integration (Stripe)

**Deliverables:**
- SSO authentication working
- Audit logs for compliance
- RBAC for team collaboration
- Organization management

**Estimated Time:** 3 weeks

---

### 5.4 Performance Optimizations

**Database:**
- [ ] Add database connection pooling (PgBouncer)
- [ ] Add query caching (Redis)
- [ ] Optimize slow queries (EXPLAIN ANALYZE)
- [ ] Add database indexes where missing

**Modal:**
- [ ] Optimize chunking performance (parallelization)
- [ ] Cache embedding model downloads
- [ ] Reduce cold starts (increase min_containers?)
- [ ] Batch processing for multiple documents

**Milvus:**
- [ ] Optimize vector index (switch from FLAT to IVF_FLAT or HNSW)
- [ ] Add result caching for common queries
- [ ] Implement query result pagination

**Frontend:**
- [ ] Add server-side rendering (Next.js already supports)
- [ ] Optimize bundle size (code splitting)
- [ ] Add loading states and skeletons
- [ ] Implement infinite scroll for large result sets

**Estimated Time:** 2 weeks

---

## Technical Specifications

### Chunking Strategies Comparison

| Strategy | Use Case | Pros | Cons | Implementation |
|----------|----------|------|------|----------------|
| **Fixed-size** | Simple text | Fast, predictable | Breaks context | Current |
| **Semantic** | General documents | Preserves meaning | Slower | LangChain RecursiveCharacterTextSplitter |
| **Token-based** | LLM context | Respects token limits | Requires tokenizer | LangChain TokenTextSplitter + tiktoken |
| **Sentence-aware** | Short-form content | Natural boundaries | May exceed limits | NLTK/spaCy |
| **Markdown-aware** | Documentation | Preserves structure | Markdown-specific | LangChain MarkdownHeaderTextSplitter |
| **Table-aware** | Data tables | Keeps tables intact | Complex parsing | Custom (Docling tables) |
| **Code-aware** | Source code | Preserves syntax | Language-specific | LangChain LanguageRecursiveTextSplitter |

---

### Embedding Models Comparison

| Provider | Model | Dimension | Cost (per 1M tokens) | Quality | Speed | Use Case |
|----------|-------|-----------|----------------------|---------|-------|----------|
| OpenAI | text-embedding-3-small | 1536 | $0.02 | ★★★★☆ | Fast | General |
| OpenAI | text-embedding-3-large | 3072 | $0.13 | ★★★★★ | Medium | High quality |
| Voyage AI | voyage-large-2 | 1536 | $0.10 | ★★★★★ | Medium | General (better) |
| Voyage AI | voyage-code-2 | 1536 | $0.10 | ★★★★★ | Medium | Code |
| Cohere | embed-multilingual-v3.0 | 1024 | $0.10 | ★★★★☆ | Fast | Multilingual |
| Local | all-MiniLM-L6-v2 | 384 | Free | ★★★☆☆ | Very fast | Budget |
| OpenAI | clip-vit-large-patch14 | 768 | N/A | ★★★★☆ | Medium | Multimodal |

---

### GraphRAG vs Traditional RAG

| Feature | Traditional RAG | GraphRAG |
|---------|----------------|----------|
| **Query Type** | "What is X?" | "How is X related to Y?" |
| **Retrieval Method** | Vector similarity only | Vector + Graph traversal |
| **Context Depth** | Single hop (direct match) | Multi-hop (2-3 degrees away) |
| **Relationship Awareness** | None | Explicit relationships |
| **Best For** | Factual lookup | Complex reasoning, exploration |
| **Complexity** | Simple | High |
| **Cost** | Low (embeddings only) | Higher (embeddings + LLM for entities) |
| **Accuracy** | Good for direct queries | Better for relationship queries |

**When to Use GraphRAG:**
- Complex multi-hop reasoning ("Who influenced X who worked at Y?")
- Relationship-heavy domains (social networks, knowledge bases)
- Citation/provenance tracking
- Exploratory queries ("Tell me about X and related topics")

**When to Use Traditional RAG:**
- Simple factual queries ("What is X?")
- Cost-sensitive applications
- Low-latency requirements
- Domains with weak relationships

---

## LlamaHub Integration Strategy

### What is LlamaHub?

LlamaHub (now part of LlamaIndex) is a collection of 100+ data loaders for various sources. Each loader handles:
- Authentication
- API pagination
- Rate limiting
- Data normalization
- Incremental sync

**GitHub:** https://github.com/run-llama/llama_hub

### Available Loaders (Top 50)

| Category | Loaders | Examples |
|----------|---------|----------|
| **Productivity** | Notion, Google Drive, Dropbox, OneDrive, Box | 5 |
| **Communication** | Slack, Discord, Telegram, WhatsApp, Gmail | 5 |
| **Project Management** | Jira, Asana, Trello, Linear, Monday | 5 |
| **Documentation** | Confluence, GitBook, ReadMe, Docusaurus | 4 |
| **Code** | GitHub, GitLab, Bitbucket | 3 |
| **Databases** | PostgreSQL, MongoDB, MySQL, Snowflake, BigQuery | 5 |
| **CRM** | Salesforce, HubSpot, Pipedrive | 3 |
| **Design** | Figma, Sketch | 2 |
| **Web** | BeautifulSoup, Playwright, Selenium | 3 |
| **File Formats** | PDF, DOCX, CSV, JSON, XML, Markdown | 6 |
| **Cloud Storage** | S3, GCS, Azure Blob | 3 |
| **Social Media** | Twitter, Reddit, YouTube | 3 |
| **Others** | Airtable, Zendesk, Intercom, Wikipedia | 4+ |

### Integration Architecture

```python
# modal/services/connectors/llamahub_adapter.py

class LlamaHubAdapter:
    """Generic adapter for LlamaHub loaders"""

    def __init__(self, loader_name: str, **loader_kwargs):
        """
        Initialize loader by name

        Args:
            loader_name: e.g., "NotionPageReader", "GoogleDriveReader"
            loader_kwargs: Loader-specific config
        """
        # Dynamically import loader
        module = __import__(f"llama_hub.{loader_name.lower()}", fromlist=[loader_name])
        loader_class = getattr(module, loader_name)
        self.loader = loader_class(**loader_kwargs)

    async def load_documents(self) -> List[Dict]:
        """Load documents using LlamaHub loader"""
        # Call loader (sync)
        documents = self.loader.load_data()

        # Convert to FLTR format
        return [self._convert_document(doc) for doc in documents]

    def _convert_document(self, llamaindex_doc) -> Dict:
        """Convert LlamaIndex Document to FLTR format"""
        return {
            "content": llamaindex_doc.text,
            "metadata": llamaindex_doc.metadata,
            "source_id": llamaindex_doc.id_,
        }

# Usage
adapter = LlamaHubAdapter("NotionPageReader", integration_token="...")
documents = await adapter.load_documents()
```

### Connector Configuration Schema

```python
# fastapi/models/connector.py

class ConnectorConfig(SQLModel, table=True):
    id: UUID = Field(primary_key=True)
    user_id: UUID = Field(foreign_key="users.id")
    dataset_id: UUID = Field(foreign_key="datasets.id")
    connector_type: str  # notion, google_drive, slack, etc.
    config: Dict  # Connector-specific config
    sync_frequency: str  # manual, hourly, daily, weekly
    last_synced: Optional[datetime]
    status: str  # active, paused, error
    error_message: Optional[str]
    created_at: datetime
    updated_at: datetime
```

### Incremental Sync Strategy

**Problem:** Don't want to re-process unchanged documents

**Solution:** Track sync checkpoints
```python
class SyncCheckpoint(SQLModel, table=True):
    id: UUID = Field(primary_key=True)
    connector_config_id: UUID = Field(foreign_key="connector_config.id")
    source_id: str  # External document ID
    last_modified: datetime  # From source system
    document_id: UUID  # Our document ID
    hash: str  # Content hash (for change detection)
```

**Sync Algorithm:**
```python
async def incremental_sync(connector_config: ConnectorConfig):
    # Load new/modified documents from source
    adapter = LlamaHubAdapter(connector_config.connector_type, **connector_config.config)
    source_documents = await adapter.load_documents()

    # Get existing checkpoints
    checkpoints = await get_checkpoints(connector_config.id)
    checkpoint_map = {cp.source_id: cp for cp in checkpoints}

    for source_doc in source_documents:
        checkpoint = checkpoint_map.get(source_doc["source_id"])

        # Calculate hash
        content_hash = hashlib.sha256(source_doc["content"].encode()).hexdigest()

        # Check if changed
        if checkpoint and checkpoint.hash == content_hash:
            print(f"Skipping unchanged document: {source_doc['source_id']}")
            continue

        # Upload and process
        if checkpoint:
            # Update existing document
            await update_document(checkpoint.document_id, source_doc)
        else:
            # Create new document
            await create_and_process_document(connector_config.dataset_id, source_doc)

        # Update checkpoint
        await upsert_checkpoint(
            connector_config.id,
            source_doc["source_id"],
            content_hash,
            source_doc["metadata"].get("last_modified")
        )
```

---

## GraphRAG Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        Document Upload                       │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│                      Document Processing                     │
│  1. Parse (Docling)                                         │
│  2. Chunk (Semantic)                                        │
│  3. Extract Entities (GPT-4-mini)   ◄───────────────────┐  │
│  4. Extract Relationships (GPT-4-mini)                   │  │
│  5. Embed Text (OpenAI)                                  │  │
└─┬──────────────────────────┬─────────────────────────────┼──┘
  │                          │                             │
  │                          │                             │
  ▼                          ▼                             ▼
┌──────────────┐   ┌──────────────────┐   ┌────────────────────┐
│ Milvus       │   │ PostgreSQL       │   │ Knowledge Graph    │
│ (Vectors)    │   │ (Metadata)       │   │ (Neo4j/NetworkX)   │
└──────────────┘   └──────────────────┘   └────────────────────┘
       │                    │                      │
       │                    │                      │
       └────────────────────┴──────────────────────┘
                            │
                            ▼
               ┌────────────────────────┐
               │   Query Processing     │
               │  1. Extract entities   │
               │  2. Vector search      │
               │  3. Graph traversal    │
               │  4. Rank + merge       │
               └────────────────────────┘
                            │
                            ▼
                     ┌─────────────┐
                     │   Results   │
                     └─────────────┘
```

### Entity Extraction Prompt

```python
ENTITY_EXTRACTION_PROMPT = """
Extract named entities from the following text. Return ONLY valid JSON.

Entity Types:
- Person: Individual people
- Organization: Companies, institutions, teams
- Location: Cities, countries, addresses
- Date: Dates and time periods
- Product: Products, services, tools
- Technology: Programming languages, frameworks, platforms
- Concept: Abstract ideas, methodologies

Text:
{text}

Return format:
[
  {"entity": "OpenAI", "type": "Organization"},
  {"entity": "Sam Altman", "type": "Person"},
  {"entity": "GPT-4", "type": "Product"}
]
"""

RELATIONSHIP_EXTRACTION_PROMPT = """
Extract relationships between these entities: {entities}

Relationship Types:
- CEO/Founder/Employee (Person → Organization)
- Founded/Created (Person → Organization/Product)
- Located_In (Organization → Location)
- Part_Of (Organization → Organization)
- Uses (Organization → Technology)
- Related_To (any → any)

Text:
{text}

Return format:
[
  {"source": "Sam Altman", "relation": "CEO", "target": "OpenAI"},
  {"source": "OpenAI", "relation": "Created", "target": "GPT-4"}
]
"""
```

### Graph Schema

**Node Types:**
- Person
- Organization
- Location
- Date
- Product
- Technology
- Concept

**Edge Types:**
- CEO, Founder, Employee
- Founded, Created
- Located_In
- Part_Of, Subsidiary_Of
- Uses, Implements
- Related_To (generic)

**Node Properties:**
- `name`: Entity name
- `type`: Entity type
- `description`: LLM-generated description
- `dataset_id`: Source dataset
- `mentioned_in`: Array of document IDs

**Edge Properties:**
- `relation`: Relationship type
- `weight`: Confidence score (0-1)
- `source_document`: Document ID where found
- `source_chunk`: Chunk index

### Graph Algorithms

**1. Breadth-First Search (BFS):**
```python
def bfs_traverse(start_node, max_depth=2):
    """Find all nodes within N hops"""
    visited = set()
    queue = [(start_node, 0)]
    results = []

    while queue:
        node, depth = queue.pop(0)
        if node in visited or depth > max_depth:
            continue

        visited.add(node)
        results.append((node, depth))

        # Get neighbors
        for neighbor in get_neighbors(node):
            queue.append((neighbor, depth + 1))

    return results
```

**2. Shortest Path:**
```python
def find_shortest_path(start_node, end_node, max_hops=3):
    """Find shortest path between two entities"""
    import networkx as nx

    try:
        path = nx.shortest_path(graph, start_node, end_node)
        if len(path) - 1 <= max_hops:
            return path
    except nx.NetworkXNoPath:
        return None
```

**3. Community Detection (Louvain):**
```python
import community as community_louvain

def detect_communities(graph):
    """Detect communities using Louvain algorithm"""
    partition = community_louvain.best_partition(graph)

    # Group by community
    communities = {}
    for node, comm_id in partition.items():
        if comm_id not in communities:
            communities[comm_id] = []
        communities[comm_id].append(node)

    return communities
```

**4. PageRank (Entity Importance):**
```python
import networkx as nx

def calculate_entity_importance(graph):
    """Calculate PageRank scores for entities"""
    pagerank = nx.pagerank(graph, alpha=0.85)
    return sorted(pagerank.items(), key=lambda x: x[1], reverse=True)
```

### GraphRAG Query Flow

**Example Query:** "How is Sam Altman related to Y Combinator?"

```python
# Step 1: Extract entities from query
query_entities = ["Sam Altman", "Y Combinator"]

# Step 2: Find nodes in graph
sam_node = find_node(name="Sam Altman")
yc_node = find_node(name="Y Combinator")

# Step 3: Find path
path = find_shortest_path(sam_node, yc_node, max_hops=3)
# Result: [Sam Altman] --[Co-founded]--> [OpenAI] --[Funded by]--> [Y Combinator]

# Step 4: Get documents for each node in path
documents = []
for node in path:
    docs = get_documents_mentioning(node)
    documents.extend(docs)

# Step 5: Rank documents
ranked_docs = rank_by_relevance_and_path(query, documents, path)

# Step 6: Return context
return {
    "path": path,
    "relationships": [edge for edge in path],
    "documents": ranked_docs[:5]
}
```

---

## Cost Analysis

### Current Monthly Costs (Est.)

| Service | Usage | Cost |
|---------|-------|------|
| DigitalOcean Droplet (FastAPI) | 1 server | $24/mo |
| Vercel (Next.js hosting) | Free tier | $0 |
| Modal (serverless compute) | ~1000 docs/mo | ~$50/mo |
| Cloudflare R2 (storage) | 100 GB storage | ~$1.50/mo |
| Milvus Cloud (Zilliz) | Shared collection | ~$20/mo |
| PostgreSQL (Supabase) | Free tier | $0 |
| OpenAI API (embeddings) | ~1M tokens/mo | ~$20/mo |
| **Total** | | **~$115/mo** |

### Projected Costs After Implementation

| Service | New Usage | Cost | Change |
|---------|-----------|------|--------|
| DigitalOcean | Same | $24/mo | - |
| Vercel | Same | $0 | - |
| Modal | +50% (more processing) | ~$75/mo | +$25 |
| R2 | +2x (multimodal) | ~$3/mo | +$1.50 |
| Milvus Cloud | +3x vectors | ~$60/mo | +$40 |
| PostgreSQL | Upgrade (graph data) | ~$25/mo | +$25 |
| OpenAI API | +5x (GPT-4-mini, Whisper) | ~$100/mo | +$80 |
| **Neo4j Aura** (optional) | Graph database | ~$65/mo | +$65 |
| Elasticsearch (optional) | BM25 search | ~$20/mo | +$20 |
| **Total (without Neo4j/ES)** | | **~$287/mo** | +$172 |
| **Total (with Neo4j)** | | **~$352/mo** | +$237 |
| **Total (with Neo4j + ES)** | | **~$372/mo** | +$257 |

**Recommendation:** Start without Neo4j and Elasticsearch, use PostgreSQL + NetworkX for graphs and Milvus scalar index for keyword search. Add Neo4j and ES when scale demands it.

**Cost per User (at scale):**
- At 100 users: ~$3.70/user/mo
- At 1000 users: ~$0.37/user/mo
- At 10,000 users: ~$0.04/user/mo

---

## Success Metrics

### Phase 1 (Foundation)

- [ ] **Chunking Quality:**
  - Semantic chunking reduces broken contexts by >50%
  - Token-based chunking eliminates token limit errors
  - User satisfaction score >4/5

- [ ] **Hybrid Search:**
  - Hybrid search improves recall by >20% on BEIR benchmark
  - Query latency stays <200ms
  - User-reported relevance improves by >30%

- [ ] **Multiple Embeddings:**
  - 3+ embedding providers integrated
  - Fallback reduces downtime to <1%
  - Cost per query reduces by >20% (using local models)

### Phase 2 (Multimodal)

- [ ] **Semantic Chunking:**
  - 90% of users report better chunk quality
  - Markdown structure preserved in 95% of docs
  - Code blocks remain intact in 100% of cases

- [ ] **Tables:**
  - Tables extracted from 95% of PDFs
  - Table search accuracy >80%
  - Users can find table data via natural language queries

- [ ] **Images:**
  - OCR accuracy >90% on clear images
  - Image descriptions generated for >95% of images
  - Users can search images by description

- [ ] **Audio/Video:**
  - Transcription accuracy >90% (Whisper)
  - Keyframes extracted every 10 seconds
  - Temporal search works (<5s accuracy)

### Phase 3 (GraphRAG)

- [ ] **Knowledge Graph:**
  - Entity extraction accuracy >85%
  - Relationship extraction accuracy >75%
  - Graph builds in <5 minutes for 1000-doc dataset

- [ ] **GraphRAG Retrieval:**
  - Multi-hop queries answered correctly >70% of time
  - GraphRAG improves complex query performance by >40%
  - Graph traversal adds <100ms latency

- [ ] **Communities:**
  - Community detection finds meaningful clusters
  - Community summaries rated >4/5 by users
  - Global queries answered using communities

### Phase 4 (Connectors)

- [ ] **LlamaHub Integration:**
  - 10+ connectors working by Month 8
  - OAuth success rate >95%
  - Incremental sync reduces re-processing by >80%

- [ ] **Notion:**
  - Syncs 1000-page workspace in <10 minutes
  - Incremental sync works in <1 minute
  - Users rate integration >4/5

- [ ] **Web Scraper:**
  - Crawls 100-page site in <5 minutes
  - Respects robots.txt 100% of time
  - Change detection accuracy >90%

### Phase 5 (Enterprise)

- [ ] **Real-time Updates:**
  - Progress updates delivered with <1s latency
  - 100% of processing steps report progress
  - WebSocket uptime >99.9%

- [ ] **Analytics:**
  - Dashboard loads in <2s
  - All key metrics displayed
  - Users find analytics useful (>4/5 rating)

- [ ] **Enterprise Features:**
  - SSO login success rate >99%
  - Audit logs capture 100% of actions
  - RBAC prevents unauthorized access 100% of time

---

## Timeline Summary

| Phase | Duration | Key Deliverables | Est. Cost Increase |
|-------|----------|------------------|-------------------|
| **Phase 1: Foundation** | Months 1-2 | Semantic chunking, hybrid search, multiple embeddings | +$50/mo |
| **Phase 2: Multimodal** | Months 3-4 | Image OCR, tables, audio transcription | +$70/mo |
| **Phase 3: GraphRAG** | Months 5-6 | Knowledge graphs, GraphRAG retrieval, communities | +$65/mo (with Neo4j) |
| **Phase 4: Connectors** | Months 7-8 | Notion, Drive, Slack, web scraper, 10+ connectors | +$20/mo |
| **Phase 5: Enterprise** | Month 9 | Real-time updates, analytics, SSO, RBAC | +$0 |
| **Total** | 9 months | Full Ragie parity + GraphRAG | ~$257/mo total |

---

## Risk Mitigation

### Technical Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| LLM API downtime | Medium | High | Multi-provider fallback, local models |
| GraphRAG quality issues | Medium | Medium | Extensive testing, human-in-the-loop validation |
| Performance degradation | Low | High | Load testing, caching, database optimization |
| Cost overruns | Medium | Medium | Usage limits, cost monitoring, alerts |

### Business Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| Ragie releases major update | Low | Medium | Focus on open-source advantage, MCP integration |
| Low user adoption | Medium | High | Beta program, user feedback loop, marketing |
| Competitor pricing | Medium | Medium | Emphasize cost transparency, self-hosting option |

---

## Next Steps

1. **Week 1-2:** Review and approve roadmap
2. **Week 3:** Set up development environment for Phase 1
3. **Week 4:** Begin Phase 1 implementation (chunking refactor)
4. **Month 2-9:** Execute phases sequentially
5. **Ongoing:** User feedback, iteration, optimization

---

## Appendix: Additional Resources

### Documentation
- **LlamaIndex:** https://docs.llamaindex.ai/
- **LangChain Text Splitters:** https://python.langchain.com/docs/modules/data_connection/document_transformers/
- **Docling:** https://github.com/DS4SD/docling
- **Neo4j:** https://neo4j.com/docs/
- **Microsoft GraphRAG:** https://github.com/microsoft/graphrag

### Papers
- "Graph Retrieval-Augmented Generation" (Microsoft, 2024)
- "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks" (Meta, 2020)
- "BEIR: A Heterogeneous Benchmark for Zero-shot Evaluation of Information Retrieval Models" (2021)

### Community
- LlamaIndex Discord: https://discord.gg/llamaindex
- Modal Slack: https://modal.com/slack
- Milvus Slack: https://milvus.io/slack

---

**End of Document**
