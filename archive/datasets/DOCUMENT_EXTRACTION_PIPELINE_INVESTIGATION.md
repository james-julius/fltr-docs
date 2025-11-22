# FLTR Document Processing & Extraction Pipeline - Investigation Report

## Executive Summary

The FLTR codebase has a sophisticated multi-modal document processing pipeline that extracts and stores images, tables, OCR text, and structured data, but **the rich content (images and tables) is NOT being displayed in the chat interface**. The infrastructure exists to retrieve and serve this content, but it's not being utilized in the UI.

---

## 1. DOCUMENT EXTRACTION PIPELINE

### 1.1 What's Being Extracted

#### Documents are processed through multiple stages:

1. **PDF Parsing & Validation** (`modal/services/pdf_validator.py`)
   - Validates PDF structure
   - Repairs MediaBox issues
   - Handles corrupted PDFs

2. **Multimodal Processing** (`modal/services/multimodal_processor.py` - Lines 41-244)
   - **PyMuPDF4LLM** (primary): Extracts markdown with semantic structure
   - **Docling** (fallback): Enhanced image extraction
   - Both support full image extraction with context preservation

3. **Image Processing** (`modal/services/vision_service.py` - Lines 79-150)
   - Classification: Charts (bar/line/pie), tables, diagrams, photos, screenshots, illustrations
   - OCR extraction: Uses RapidOCR for text-heavy images
   - Vision descriptions: Via GPT-4V, Claude Vision, or Gemini Vision
   - Structured data extraction: For charts and tables (optional)

4. **Storage & Chunking** (`modal/services/document_processor.py` - Lines 168-243)
   - Creates multimodal chunks with image metadata
   - Simple text chunking for non-PDF documents
   - Tracks chunk types: "text" or "image"

---

## 2. CONTENT TYPES BEING EXTRACTED & STORED

### 2.1 Text Content
- Document markdown (via PyMuPDF4LLM or Docling)
- Paragraph-level chunks (1000 chars, 200 char overlap)
- Filename and document type metadata

### 2.2 Images
**Extracted at three levels:**

#### A. Document Assets Table (`fastapi/models/document_asset.py` - Lines 21-51)
```
DocumentAsset:
  - asset_type: "extracted_image", "table", "chart", "diagram", "thumbnail"
  - object_key: R2 storage path
  - content_type: MIME type (e.g., "image/png")
  - page_number: Source page
  - image_index: Index within document
  - extraction_metadata: JSON with OCR, vision data, etc.
  - r2_url: Presigned download URL (generated on request)
```

#### B. Vector Chunks with Image Metadata (`modal/modal_app.py` - Lines 807-838)
Images stored with metadata in Milvus:
- `chunk_type`: "ocr" or "text"
- `image_r2_key`: R2 object location
- `page`: Page number
- `image_index`: Position in document
- `ocr_confidence`: OCR accuracy score
- `image_size`: Dimensions

#### C. Image Data in Memory During Processing (`modal/services/multimodal_processor.py` - Lines 208-244)
```python
image_data = [
  {
    "index": image_index,
    "filename": filename,
    "format": "png/jpg",
    "r2_key": "datasets/{dataset_id}/documents/{document_id}/images/{hash}",
    "size": bytes,
    "ocr_text": extracted_text,
    "ocr_confidence": confidence,
    "description": vision_model_description,
    "image_type": classification,
    "vision_confidence": confidence,
    "vision_provider": "gpt4v|claude|gemini",
    "structured_data": {...}, # For charts/tables
    "surrounding_context": text_near_image,
    "format": "pymupdf4llm|docling"
  }
]
```

### 2.3 Tables
- Extracted as **markdown tables** in document content
- Also stored as **structured data** in image assets (if `extract_structured_data=True`)
- Currently stored in vector chunks but **NOT separately indexed**

### 2.4 Charts & Diagrams
- Extracted as **images** with type classification
- Vision models generate semantic descriptions
- Optional structured data extraction (currently disabled)

### 2.5 OCR Data
- Text extracted from image-heavy PDFs
- Confidence scores stored with chunks
- Fast fallback option (RapidOCR before vision models)

---

## 3. DATABASE SCHEMA & STORAGE

### 3.1 SQL Database (`fastapi/models/document.py`, `fastapi/models/document_asset.py`)

#### Documents Table
```sql
documents:
  - id: UUID (primary key)
  - dataset_id: UUID (foreign key)
  - file_name: string
  - object_key: string (R2 path)
  - status: "pending|uploaded|processing|ready|failed"
  - chunk_count: int (# vectors in Milvus)
  - asset_count: int (# extracted images/tables)
  - processing_metadata: JSON (stage tracking, retries, costs)
  - created_at, updated_at: timestamps
```

#### DocumentAssets Table
```sql
document_assets:
  - id: UUID
  - document_id: UUID (foreign key)
  - asset_type: "extracted_image|table|chart|diagram|thumbnail"
  - object_key: string (R2 path)
  - file_size: int (bytes)
  - content_type: MIME type
  - page_number: int
  - image_index: int
  - extraction_metadata: JSON
    {
      "ocr_text": "...",
      "ocr_confidence": 0.95,
      "description": "A bar chart showing...",
      "image_type": "chart_bar",
      "vision_confidence": 0.87,
      "vision_provider": "gpt4v",
      "structured_data": {...},
      "surrounding_context": "...",
      "format": "pymupdf4llm"
    }
  - created_at, updated_at: timestamps
```

### 3.2 Vector Store (Milvus)

**Collection**: `fltr_documents`
**Fields stored for each chunk:**
```
- text: string (chunk text)
- filename: string
- chunk_index: int
- document_type: string
- dataset_id: UUID
- document_id: UUID
- chunk_type: "text|ocr|image" ⭐
- page: int (for OCR chunks)
- image_index: int
- ocr_confidence: float
- image_r2_key: string ⭐
- image_size: string
- [embedding]: vector (1536 dims, text-embedding-3-small)
```

**Key constraint**: Milvus is a **vector database**, not optimized for storing large binary assets. Images are stored in **R2 (Cloudflare)**, Milvus just tracks references.

### 3.3 Object Storage (R2 / Cloudflare)

**Location structure:**
```
datasets/
  {dataset_id}/
    documents/
      {document_id}/
        file.pdf (original)
        images/
          image_0_hash.png
          image_1_hash.png
          ...
    chunks/ (optional, for serialized chunks)
```

---

## 4. CHAT INTERFACE - WHAT'S DISPLAYED

### 4.1 API Flow

**User Query** → **Chat API** → **Search Tool** → **FastAPI Backend** → **Milvus Search** → **Response**

#### NextJS Chat Page (`nextjs/src/app/(public)/datasets/[slug]/chat/page.tsx` - Lines 138-186)
- Uses `ChatInterface` component
- Passes API endpoint: `/api/chat/dataset/{datasetId}`
- Shows suggested queries (category-specific)

#### Chat Interface Component (`nextjs/src/components/chat/chat-interface.tsx` - Lines 48-250)
- Displays markdown content via `MarkdownDisplay`
- Shows tool invocations via `ToolInvocationDisplay`
- Currently displays:
  - User messages (text)
  - AI responses (markdown)
  - Tool results (JSON in `<pre>` blocks)

#### Tool Invocation Display (`nextjs/src/components/chat/tool-invocation-display.tsx` - Lines 21-46)
```tsx
// Shows tool name, inputs, and OUTPUTS
// Outputs rendered as <pre> JSON (no formatting)
<ToolOutput output={<pre>{JSON.stringify(result, null, 2)}</pre>} />
```

#### Markdown Display Component (`nextjs/src/components/chat/markdown-display.tsx` - Lines 12-110)
- Renders markdown with `react-markdown` + `remark-gfm`
- Supports tables (lines 74-103)
- **Does NOT support embedded images**
- Code renderer for inline/block (lines 54-73)
- No component for rendering `<img>` tags from markdown

### 4.2 Search API Response

**Endpoint**: `/api/chat/dataset/{datasetId}` (NextJS) → `/api/v1/mcp/query/{dataset_id}/search` (FastAPI)

**File**: `nextjs/src/app/api/chat/dataset/[datasetId]/route.ts` - Lines 53-133

**Tool Output** (lines 113-122):
```typescript
results: data.contexts.map((ctx: any) => ({
    content: ctx.content,        // Text chunk only!
    relevance: ctx.relevance_score,
    metadata: ctx.metadata       // Includes image_url, but not used
}))
```

**FastAPI Response** (`fastapi/routers/mcp.py` - Lines 124-195):
```python
# Returns contexts with FULL metadata including:
metadata = {
    "filename": "...",
    "chunk_type": "text|ocr",
    "page": page_number,
    "image_index": image_index,
    "ocr_confidence": confidence,
    "image_url": presigned_url,  # ⭐ Generated on-the-fly
    "image_size": "1024x768"
}
```

**KEY FINDING**: The search results include `image_url` in metadata (line 169), but:
1. ✅ Image URL is **generated** (presigned)
2. ❌ Image URL is **never displayed** in the chat UI

---

## 5. WHAT'S AVAILABLE BUT NOT SHOWN

### 5.1 Extracted Images
**Status**: ✅ Extracted → ✅ Stored → ✅ Retrievable → ❌ Not Displayed

- **Extracted by**: Vision models (GPT-4V, Claude, Gemini)
- **Stored in**: DocumentAssets table + R2
- **Tracked in vectors**: `chunk_type="ocr"`, `image_r2_key` field
- **Generated metadata**: descriptions, classifications, OCR text
- **Available API**: `GET /api/v1/datasets/{dataset_id}/documents/{document_id}/assets` (lines 339-395)

### 5.2 Image Descriptions & Context
**Status**: ✅ Generated → ✅ Stored → ❌ Not Displayed

**Stored in**: `extraction_metadata` JSON field
```json
{
  "description": "A bar chart showing quarterly revenue from 2021-2024",
  "image_type": "chart_bar",
  "vision_confidence": 0.92,
  "ocr_text": "Q1 2021: $50M\nQ2 2021: $52M...",
  "structured_data": {...}
}
```

### 5.3 Tables
**Status**: ✅ Markdown format in chunks → ✅ Can display in markdown → ⚠️ Rendering works, but no special handling

- Extracted as markdown tables by PyMuPDF4LLM
- Stored in vector chunks as text
- `MarkdownDisplay` component CAN render tables (lines 74-103)
- **However**: No special formatting or emphasis

### 5.4 Charts & Structured Data
**Status**: ✅ Extracted → ✅ Stored → ❌ Not Displayed

- Vision models can extract structured data from charts
- Currently disabled (`extract_structured_data=False`)
- Would include: axis values, data points, labels
- Could be visualized as interactive charts

### 5.5 Page Numbers & References
**Status**: ✅ Stored → ⚠️ Not useful without display

- Each chunk includes `page` field for scanned/OCR content
- Useful for "see page X" citations
- Not displayed in search results

---

## 6. KEY FINDINGS - WHAT'S MISSING

### Critical Gap: Image Display in Chat

**Problem**: Search results include images but chat UI doesn't display them.

**Current flow**:
```
Search Tool (FastAPI)
  ↓ [Returns: {content, relevance_score, metadata{image_url}}]
  ↓
Chat API (NextJS)
  ↓ [Passes to: format for AI model]
  ↓
AI Model (GPT-4o)
  ↓ [Receives: text + metadata with image_url]
  ↓ [Responds with: markdown text ONLY]
  ↓
Chat Interface (React)
  ↓ [Renders: markdown (no images)]
  ↓
User sees: Text only
```

### Missing Components

1. **Image URL in Markdown**
   - ❌ Search API doesn't include images in markdown format
   - ❌ AI model receives image URLs but doesn't embed them in response
   - ❌ Chat UI can't render embedded images (no `img` handler in MarkdownDisplay)

2. **Visual Asset Routing**
   - ✅ Asset API exists: `GET /datasets/{id}/documents/{doc_id}/assets`
   - ✅ Returns DocumentAssetResponse with presigned R2 URLs
   - ❌ Never called from chat interface
   - ❌ No UI component to display returned assets

3. **Image-aware Tool Results**
   - ❌ ToolInvocationDisplay renders outputs as `<pre>JSON` only
   - ❌ No special handling for image URLs in metadata
   - ❌ No gallery/carousel component for multiple images

4. **Table Visualization**
   - ✅ Markdown tables render (via `remark-gfm`)
   - ⚠️ Basic styling only (no color, sorting, etc.)
   - ❌ No export/download option
   - ❌ No distinction from regular text tables

---

## 7. DETAILED FILE LOCATIONS & CODE REFERENCES

### Document Processing (Modal)
| Component | File | Key Lines |
|-----------|------|-----------|
| Multimodal Processing | `modal/services/multimodal_processor.py` | 41-244 |
| PDF Parsing | `modal/services/document_processor.py` | 16-166 |
| Vision Service | `modal/services/vision_service.py` | 64-150 |
| Chunking | `modal/services/document_processor.py` | 168-243 |
| Asset Creation | `modal/modal_app.py` | 807-838 |

### Database Models & APIs
| Component | File | Key Lines |
|-----------|------|-----------|
| Document Model | `fastapi/models/document.py` | 22-56 |
| DocumentAsset Model | `fastapi/models/document_asset.py` | 21-52 |
| Asset Repository | `fastapi/repositories/document_asset_repository.py` | 12-103 |
| Asset API Endpoints | `fastapi/routers/dataset.py` | 337-450 |
| Search API | `fastapi/routers/mcp.py` | 60-201 |
| Embedding Service | `fastapi/services/embedding_service.py` | 23-97, 149-211 |

### Chat UI
| Component | File | Key Lines |
|-----------|------|-----------|
| Chat Page | `nextjs/src/app/(public)/datasets/[slug]/chat/page.tsx` | 138-186 |
| Chat Interface | `nextjs/src/components/chat/chat-interface.tsx` | 48-250 |
| Markdown Display | `nextjs/src/components/chat/markdown-display.tsx` | 12-110 |
| Tool Display | `nextjs/src/components/chat/tool-invocation-display.tsx` | 21-46 |
| Search Tool | `nextjs/src/app/api/chat/dataset/[datasetId]/route.ts` | 53-133 |

---

## 8. DATA FLOW DIAGRAM

```
PDF Upload
    ↓
[Modal] parse_document()
    ↓ (extracts text + images)
[Modal] chunk_document()
    ↓
[Milvus] Store chunks
    ├─ text chunks
    └─ image metadata (chunk_type="ocr", image_r2_key)
    ↓
[PostgreSQL] Document + DocumentAssets tables
    ├─ document: chunk_count, asset_count
    └─ document_assets: type, r2_key, extraction_metadata
    ↓
User Query in Chat
    ↓
[FastAPI] /api/v1/mcp/query/{dataset_id}/search
    ↓
[Milvus] Vector search
    ↓ (returns text + metadata)
[FastAPI] Build response
    ├─ contexts[] with content
    └─ metadata { image_url, ocr_text, ... }
    ↓
[NextJS] /api/chat/dataset/{datasetId}
    ↓ (search tool invocation)
    ├─ Passes to AI model
    ├─ AI generates markdown response
    └─ Missing: image embedding!
    ↓
[React] ChatInterface
    ├─ Renders markdown
    └─ User sees: text only ❌
```

---

## 9. SUMMARY TABLE

| Content Type | Extracted | Stored | Retrieved | Displayed |
|---|---|---|---|---|
| **Text** | ✅ | ✅ (Milvus) | ✅ | ✅ |
| **Images** | ✅ (vision API) | ✅ (R2 + DocumentAssets) | ✅ (URLs in metadata) | ❌ |
| **OCR Text** | ✅ (RapidOCR) | ✅ (Milvus metadata) | ✅ | ⚠️ (as text only) |
| **Tables** | ✅ (markdown) | ✅ (Milvus) | ✅ | ✅ (markdown) |
| **Charts** | ✅ (images + vision) | ✅ (R2 + assets) | ✅ (URLs) | ❌ |
| **Descriptions** | ✅ (vision API) | ✅ (extraction_metadata JSON) | ✅ | ❌ |
| **Structured Data** | ⚠️ (disabled) | ✅ (if enabled) | ✅ | ❌ |
| **Page References** | ✅ | ✅ (Milvus) | ✅ | ❌ |

---

## 10. RECOMMENDATIONS

To surface extracted rich content in chat:

1. **Embed images in markdown**
   - Format: `![alt](image_url)` in search results
   - Pass through to AI model
   - Add image support to MarkdownDisplay component

2. **Display image gallery**
   - Extract images from search results
   - Show in collapsible section below text
   - Link to full asset detail

3. **Enhance tool result visualization**
   - Detect images in metadata
   - Render with `<img>` tags instead of JSON
   - Show metadata (confidence, type, OCR text)

4. **Improve table presentation**
   - Apply better styling
   - Add sortable/filterable features
   - Show source document link

5. **Leverage structured data**
   - Enable `extract_structured_data=True` for charts
   - Generate interactive visualizations
   - Show as recharts/plotly instead of image

