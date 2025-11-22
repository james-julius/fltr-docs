# FLTR Document Extraction Pipeline - Quick Reference

## The Problem in One Sentence
**FLTR extracts and stores images, tables, and OCR text from PDFs, but the chat UI only displays text—images and rich content are never shown to users.**

---

## Where Things Happen

### 1. Document Processing (Backend)
```
User uploads PDF
  ↓
Modal Function (modal_app.py @ process_document)
  ├─ Download from R2
  ├─ Parse with PyMuPDF4LLM (modal/services/multimodal_processor.py:41-244)
  │  └─ Extracts: markdown + 20-50 images per document
  ├─ Process images with Vision API (vision_service.py:64-150)
  │  ├─ Classify: chart/table/diagram/photo
  │  ├─ Extract OCR text (RapidOCR)
  │  └─ Generate descriptions (GPT-4V/Claude/Gemini)
  ├─ Create chunks (document_processor.py:168-243)
  │  └─ Text chunks + image metadata
  └─ Store in Milvus + R2 + PostgreSQL
     ├─ Vectors: text-embedding-3-small
     ├─ Images: Cloudflare R2
     └─ Assets: DocumentAssets table
```

### 2. Database Storage

**PostgreSQL Tables:**
- `documents`: file_name, status, chunk_count, asset_count
- `document_assets`: asset_type, object_key (R2), extraction_metadata (JSON)

**Milvus Collection (`fltr_documents`):**
```
Fields: text, filename, chunk_index, dataset_id, document_id,
        chunk_type, page, image_index, ocr_confidence, image_r2_key, image_size
Index: embedding (1536 dims)
```

**R2 Storage:**
```
datasets/{dataset_id}/documents/{document_id}/
  ├─ file.pdf
  └─ images/
     ├─ image_0_hash.png
     ├─ image_1_hash.png
     └─ ...
```

### 3. Search Flow (What User Sees)

```
User types question in chat
  ↓
NextJS searchThisDataset tool (route.ts:53-133)
  ↓
FastAPI /api/v1/mcp/query/{dataset_id}/search (mcp.py:60-201)
  ├─ Generate query embedding
  ├─ Milvus vector search (filtering by dataset_id)
  └─ Build response:
     {
       "contexts": [
         {
           "content": "text chunk...",
           "relevance_score": 0.92,
           "metadata": {
             "image_url": "https://r2.../image.png",  // ⭐ Generated but unused
             "image_r2_key": "datasets/...",
             "chunk_type": "text|ocr",
             "ocr_confidence": 0.95,
             "page": 3,
             ...
           }
         }
       ]
     }
  ↓
AI Model (GPT-4o) receives search results
  └─ Generates markdown response (text only)
  ↓
React MarkdownDisplay component
  └─ Renders markdown (no image support!)
  ↓
User sees: Text only ❌
```

---

## Key Code Locations

### Extraction Pipeline
| Step | File | Lines | What |
|------|------|-------|------|
| Parse | `modal/services/multimodal_processor.py` | 41-244 | PyMuPDF4LLM + Docling |
| Vision | `modal/services/vision_service.py` | 64-150 | Image classification & descriptions |
| Chunk | `modal/services/document_processor.py` | 168-243 | Create vector chunks |
| Store | `modal/modal_app.py` | 750-850 | Store in Milvus + create DocumentAssets |

### API & Search
| Function | File | Lines | Purpose |
|----------|------|-------|---------|
| GET assets | `fastapi/routers/dataset.py` | 339-395 | List extracted images |
| GET search | `fastapi/routers/mcp.py` | 60-201 | Vector search (returns image URLs) |
| embed search | `fastapi/services/embedding_service.py` | 149-213 | Milvus vector query |

### Chat UI
| Component | File | Lines | Issue |
|-----------|------|-------|-------|
| Chat page | `nextjs/.../[slug]/chat/page.tsx` | 138-186 | Sets up chat interface |
| Chat interface | `nextjs/.../chat/chat-interface.tsx` | 48-250 | Calls search tool + renders response |
| Markdown | `nextjs/.../markdown-display.tsx` | 12-110 | **No image handler!** |
| Tools display | `nextjs/.../tool-invocation-display.tsx` | 21-46 | Shows JSON, not images |

---

## What's Extracted (Real Examples)

### Image Asset in DocumentAssets Table
```json
{
  "id": "uuid",
  "document_id": "uuid",
  "asset_type": "extracted_image",
  "object_key": "datasets/ds1/documents/doc1/images/img_0.png",
  "file_size": 45230,
  "content_type": "image/png",
  "page_number": 3,
  "image_index": 0,
  "extraction_metadata": {
    "ocr_text": "Revenue Report 2024\nQ1: $50M\nQ2: $52M",
    "ocr_confidence": 0.92,
    "description": "A bar chart showing quarterly revenue trends",
    "image_type": "chart_bar",
    "vision_confidence": 0.87,
    "vision_provider": "gpt4v",
    "structured_data": null,
    "format": "pymupdf4llm"
  }
}
```

### Milvus Chunk with Image Reference
```json
{
  "id": 12345,
  "text": "Revenue Report 2024...",
  "chunk_type": "text",
  "filename": "annual_report.pdf",
  "document_id": "uuid",
  "dataset_id": "uuid",
  "page": 3,
  "image_index": 0,
  "image_r2_key": "datasets/ds1/documents/doc1/images/img_0.png",
  "image_size": "1024x768",
  "ocr_confidence": 0.92,
  "embedding": [0.001, 0.002, ...]  // 1536 dims
}
```

### Search Result Returned to Chat
```json
{
  "content": "Revenue Report 2024\nQ1: $50M\nQ2: $52M...",
  "relevance_score": 0.92,
  "metadata": {
    "filename": "annual_report.pdf",
    "chunk_type": "text",
    "page": 3,
    "image_index": 0,
    "ocr_confidence": 0.92,
    "image_url": "https://d.../image.png?signature=...",  // ⭐ NEVER USED
    "image_size": "1024x768"
  }
}
```

---

## The Gap: What's Missing

### 1. No Image in Markdown
- ❌ Search tool doesn't embed `![alt](url)` in markdown
- ❌ AI model only sees URLs in metadata, not in content
- ❌ MarkdownDisplay has no `img` renderer

### 2. No Asset Gallery
- ✅ API exists: `GET /datasets/{id}/documents/{doc_id}/assets`
- ❌ Never called from chat UI
- ❌ No component to display images

### 3. Tool Results Are JSON
- ✅ ToolInvocationDisplay shows tool outputs
- ❌ Only renders as `<pre>JSON`
- ❌ No special handling for image URLs

---

## Numbers

**In a typical 100-page PDF:**
- Text chunks: 200-500
- Extracted images: 20-50
- Asset records created: 20-50
- Images never shown: **20-50** ❌

**Storage breakdown:**
- Milvus vectors: ~1KB per chunk × 300 = ~300KB
- R2 images: ~50-100KB each × 30 = ~1.5-3MB per document
- DocumentAssets records: ~1KB each × 30 = ~30KB

---

## How to Verify

### Check if images exist in database:
```sql
SELECT COUNT(*) FROM document_assets WHERE asset_type = 'extracted_image';
```

### Check Milvus chunks with images:
```python
# Via FastAPI
GET /api/v1/datasets/{id}/query?query=chart&limit=10
# Look for chunks with chunk_type="ocr" and image_r2_key set
```

### Check R2 storage:
```bash
# List images for a document
aws s3 ls s3://bucket/datasets/{dataset_id}/documents/{document_id}/images/
```

---

## Quick Wins to Surface Content

1. **Add image handler to MarkdownDisplay** (5 lines)
   - Render `<img>` tags from markdown
   - Show images below text in chat

2. **Include images in search response** (10 lines)
   - Format as markdown in chat API
   - Pass to AI model

3. **Create image gallery component** (50 lines)
   - Show extracted images with metadata
   - Display in collapsible section

4. **Enhance tool display** (20 lines)
   - Detect images in metadata
   - Render with `<img>` instead of JSON

---

## Files Changed After First Processing

After a PDF is uploaded and processed by Modal:

**PostgreSQL:**
- `documents`: 1 new row (status: ready, chunk_count: 250, asset_count: 30)
- `document_assets`: 30 new rows (extracted_image type)

**Milvus (fltr_documents collection):**
- 250 new vectors (200 text chunks, 50 image chunks)

**R2 (Cloudflare):**
- 1 original PDF: `datasets/uuid/documents/uuid/file.pdf` (5MB)
- 30 images: `datasets/uuid/documents/uuid/images/img_*.png` (2-5MB total)

**Total new data:** ~7-10MB per document
