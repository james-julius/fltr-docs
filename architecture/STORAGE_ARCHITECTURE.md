# FLTR Storage Architecture

This document describes how FLTR stores and retrieves document data across its storage systems.

## Overview

FLTR uses three primary storage systems:

| System | Purpose | Location |
|--------|---------|----------|
| **Milvus** | Vector embeddings + chunk metadata | Milvus Lite (dev) / Zilliz Cloud (prod) |
| **PostgreSQL** | Dataset/document metadata, user data | Local (dev) / Neon (prod) |
| **Cloudflare R2** | Raw files, extracted images, checkpoints | R2 buckets |

---

## Milvus Vector Store

### Collection Structure

FLTR uses a **single shared collection** called `fltr_documents` for all datasets. Documents are partitioned logically using the `dataset_id` field.

**Schema** (defined in `fastapi/database/vector_store.py`):

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         fltr_documents Collection                            │
├─────────────────┬──────────────────┬────────────────────────────────────────┤
│ Field           │ Type             │ Description                            │
├─────────────────┼──────────────────┼────────────────────────────────────────┤
│ id              │ INT64 (PK, auto) │ Primary key                            │
│ vector          │ FLOAT_VECTOR     │ 1536-dim embedding (text-embedding-3)  │
│ text            │ VARCHAR(65535)   │ Chunk content (currently with context) │
│ dataset_id      │ VARCHAR(64)      │ Dataset UUID for filtering              │
│ document_id     │ VARCHAR(64)      │ Document UUID                          │
│ filename         │ VARCHAR(512)     │ Original filename                       │
│ chunk_index     │ INT64            │ Chunk position in document             │
│ document_type   │ VARCHAR(128)     │ File type (pdf, docx, etc.)            │
│ chunk_type      │ VARCHAR(16)      │ "text", "image", or "ocr"              │
│ page            │ INT64            │ Page number (for PDFs)                 │
│ image_index     │ INT64            │ Image position on page                 │
│ ocr_confidence   │ FLOAT            │ OCR quality score (0-1)                │
│ image_r2_key    │ VARCHAR(512)     │ R2 key for image retrieval             │
│ image_size      │ VARCHAR(32)      │ Image dimensions (e.g., "1161x171")    │
│ keywords        │ VARCHAR(2000)    │ Comma-separated keywords (BM25 index)  │
│ chunk_id        │ VARCHAR(64)      │ Unique chunk identifier                 │
│ parent_chunk_id │ VARCHAR(64)      │ Parent chunk ID (for child chunks)     │
│ is_parent       │ BOOL             │ Whether this is a parent chunk         │
└─────────────────┴──────────────────┴────────────────────────────────────────┘
```

### Indexes

| Field | Index Type | Purpose |
|-------|------------|---------|
| `vector` | FLAT | Vector similarity search (COSINE metric) |
| `keywords` | INVERTED | BM25 keyword search (hybrid retrieval) |

### Chunk Types

FLTR stores three types of chunks:

1. **`text`** - Regular text content from documents
2. **`image`** - Vision model descriptions of images
3. **`ocr`** - OCR-extracted text from images

---

## Multimodal Document Handling

### Text-Based Embedding Strategy

FLTR uses a **text-based multimodal approach** rather than direct multimodal embeddings:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Multimodal Processing Flow                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Image from PDF                                                            │
│        │                                                                    │
│        ▼                                                                    │
│   ┌─────────────────────────────────────────┐                               │
│   │  1. Extract image from document         │                               │
│   │  2. Upload to R2 storage                │                               │
│   └─────────────────────────────────────────┘                               │
│        │                                                                    │
│        ├────────────────┬───────────────────┐                               │
│        ▼                ▼                   ▼                               │
│   ┌──────────┐    ┌──────────┐       ┌──────────────┐                       │
│   │   OCR    │    │  Vision  │       │ Store in R2  │                       │
│   │ (Tesseract)   │  Model   │       │ for retrieval│                       │
│   └──────────┘    └──────────┘       └──────────────┘                       │
│        │                │                                                   │
│        ▼                ▼                                                   │
│   "Text from     "A recipe showing                                          │
│    image..."      braised meat with                                         │
│                   vegetables..."                                            │
│        │                │                                                   │
│        └────────┬───────┘                                                   │
│                 ▼                                                           │
│   ┌─────────────────────────────────────────┐                               │
│   │  Combine into text representation       │                               │
│   │  "OCR: ... | Vision: ..."               │                               │
│   └─────────────────────────────────────────┘                               │
│                 │                                                           │
│                 ▼                                                           │
│   ┌─────────────────────────────────────────┐                               │
│   │  text-embedding-3-small (OpenAI)        │                               │
│   │  → 1536-dimension vector                │                               │
│   └─────────────────────────────────────────┘                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why Text-Based?

1. **Single embedding model**: All chunks (text, image, OCR) use the same `text-embedding-3-small` model
2. **Unified search**: Vector similarity works the same way for all content types
3. **Cost effective**: No need for expensive multimodal embedding models
4. **Rich descriptions**: Vision models provide semantic understanding beyond raw pixels

### Image Chunk Metadata

For image chunks (`chunk_type = "image"` or `"ocr"`):

| Field | Example | Description |
|-------|---------|-------------|
| `image_r2_key` | `datasets/abc123/doc456/images/page_5_img_2.png` | R2 storage path |
| `ocr_confidence` | `0.85` | Tesseract OCR quality score |
| `page` | `5` | Source page number |
| `image_index` | `2` | Image position on page |
| `image_size` | `"800x600"` | Original dimensions |

### Image Storage in R2

Images are stored in R2 with the following key structure:

```
{dataset_id}/{document_id}/images/page_{N}_img_{M}.png
```

When a search result includes an image chunk, the application can:
1. Read `image_r2_key` from chunk metadata
2. Generate presigned URL or fetch from R2
3. Display alongside the text description

---

## Contextual Retrieval

### Current Implementation

FLTR implements Anthropic's "contextual retrieval" technique by prepending document context to each chunk before embedding:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Current text field content:                                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│ Document: Marcella Hazan's Cookbook                                         │
│ Filename: chapter5.pdf                                                      │
│ Document Type: pdf                                                          │
│ Summary: Classic Italian recipes from the renowned chef...                  │
│ Section: Braised Meats                                                      │
│                                                                             │
│ Cut the veal shanks into 2-inch pieces. Season with salt and pepper...      │
└─────────────────────────────────────────────────────────────────────────────┘
          ↑                                          ↑
       Context prefix                         Actual chunk content
       (for embedding quality)                (what users want to see)
```

### Problem

The `text` field contains both context and content, which:
- Makes search results display metadata instead of actual content
- Wastes storage by duplicating metadata
- Confuses the purpose of the `text` field

### Planned Improvement

Separate storage:
- `text`: Actual chunk content only (for display)
- `context`: Metadata prefix (for reference)
- Embedding: Generated from `context + text` (for retrieval quality)

---

## Parent-Child Chunking

For complex documents, FLTR supports hierarchical chunking:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Parent Chunk                                   │
│                         (larger context window)                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ is_parent: true                                                       │  │
│  │ chunk_id: "parent_abc123"                                             │  │
│  │ Full section text with complete context...                            │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                              │                                              │
│          ┌───────────────────┼───────────────────┐                          │
│          ▼                   ▼                   ▼                          │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐                    │
│  │ Child Chunk │     │ Child Chunk │     │ Child Chunk │                    │
│  │ (for search)│     │ (for search)│     │ (for search)│                    │
│  ├─────────────┤     ├─────────────┤     ├─────────────┤                    │
│  │ is_parent:  │     │ is_parent:  │     │ is_parent:  │                    │
│  │   false     │     │   false     │     │   false     │                    │
│  │ parent_id:  │     │ parent_id:  │     │ parent_id:  │                    │
│  │   parent_abc│     │   parent_abc│     │   parent_abc│                    │
│  └─────────────┘     └─────────────┘     └─────────────┘                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Retrieval flow**:
1. Search matches small, specific child chunks
2. Retrieve parent chunk for full context
3. Pass parent content to LLM for better answers

---

## PostgreSQL Schema

### Key Tables

**datasets**
- `id` (UUID): Primary key
- `name`: Dataset display name
- `description`: Dataset description
- `user_id`: Owner reference
- `created_at`, `updated_at`

**documents**
- `id` (UUID): Primary key
- `dataset_id`: Foreign key to datasets
- `filename`: Original file name
- `object_key`: R2 storage key
- `status`: Processing status (pending, processing, completed, failed)
- `chunk_count`: Number of chunks created
- `created_at`, `updated_at`

**document_assets**
- `id` (UUID): Primary key
- `document_id`: Foreign key to documents
- `asset_type`: "image", "table", etc.
- `r2_key`: R2 storage path
- `metadata`: JSON with dimensions, page number, etc.

---

## R2 Storage Structure

```
fltr-datasets-{env}/
├── {dataset_id}/
│   ├── {document_id}/
│   │   ├── original.pdf           # Original uploaded file
│   │   ├── content.md             # Extracted markdown
│   │   ├── images/
│   │   │   ├── page_1_img_0.png
│   │   │   ├── page_1_img_1.png
│   │   │   └── ...
│   │   └── processing/
│   │       └── checkpoint.json    # Processing state
│   └── ...
└── ...
```

---

## Search Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Search Query                                   │
│                          "osso buco recipe"                                 │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Embed Query                                       │
│                    text-embedding-3-small                                   │
│                      → 1536-dim vector                                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
┌─────────────────────────────┐     ┌─────────────────────────────┐
│      Vector Search          │     │      BM25 Keyword Search    │
│  COSINE similarity on       │     │  INVERTED index on          │
│  `vector` field              │     │  `keywords` field            │
└─────────────────────────────┘     └─────────────────────────────┘
                    │                               │
                    └───────────────┬───────────────┘
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Reciprocal Rank Fusion                              │
│                   Combine and re-rank results                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Return Top-K Chunks                                │
│           with text, metadata, and similarity scores                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## File References

| Component | File |
|-----------|------|
| Milvus schema | `fastapi/database/vector_store.py` |
| Chunking strategies | `modal/services/chunking/strategies/*.py` |
| Contextual augmentation | `modal/services/text_processing/text_analyzer.py` |
| Multimodal processing | `modal/services/chunking/strategies/multimodal_strategy.py` |
| Image extraction | `modal/services/document_processing/image_extractor.py` |
| Embedding generation | `modal/services/embedding/embedding_service.py` |
| PostgreSQL models | `fastapi/models/` |
| R2 operations | `fastapi/services/r2_service.py` |
