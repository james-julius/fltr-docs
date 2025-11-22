# Document Extraction Pipeline Investigation - File Index

This investigation examines the FLTR document processing pipeline from extraction through display in the chat interface.

## Key Documents

### 1. **DOCUMENT_PIPELINE_FINDINGS_SUMMARY.txt** (START HERE)
- **Length**: 12 KB
- **Best For**: Quick overview and executive summary
- **Contains**:
  - Key finding (images extracted but not displayed)
  - Complete pipeline overview
  - Database schemas
  - What's extracted vs. what's shown
  - Code locations with line numbers
  - Critical gaps
  - Recommendations
  - Verification queries

### 2. **DOCUMENT_EXTRACTION_PIPELINE_INVESTIGATION.md** (DETAILED ANALYSIS)
- **Length**: 15 KB
- **Best For**: Deep dive understanding and implementation details
- **Contains**:
  - Detailed extraction process (4 stages)
  - Content types with examples
  - Database schema explanations
  - API flow diagrams
  - What's available but not shown (with status indicators)
  - Data flow diagrams
  - Summary comparison tables
  - Recommendations for each content type

### 3. **DOCUMENT_EXTRACTION_QUICK_REFERENCE.md** (CHEAT SHEET)
- **Length**: 7.5 KB
- **Best For**: Quick lookup while developing
- **Contains**:
  - Problem statement (one sentence)
  - Where things happen (visual flow)
  - Database storage locations
  - Search flow diagram
  - Key code locations (compact table)
  - Real JSON examples of extracted data
  - What's missing (gaps)
  - Numbers (impact)
  - How to verify
  - Quick wins

---

## Investigation Scope

### Questions Answered

1. **What document extraction is currently happening?**
   - PDFs parsed with PyMuPDF4LLM
   - 20-50 images extracted per document
   - Vision models process images (classification, OCR, descriptions)
   - Tables extracted as markdown
   - Structured data extraction available (disabled)

2. **What types of content are extracted and stored?**
   - Text: Full markdown with semantic structure
   - Images: Binary files in R2 + metadata in DB
   - OCR: Extracted from scanned images
   - Tables: Markdown format
   - Descriptions: Vision model summaries
   - Classifications: Chart/table/diagram/photo types

3. **How are documents stored in database?**
   - PostgreSQL: `documents` (metadata) + `document_assets` (extracted content)
   - Milvus: Vector chunks with image references
   - R2: Binary image files
   - See Database Schema sections in referenced documents

4. **How does chat display document content?**
   - Receives markdown from AI model
   - Renders via react-markdown with table support
   - **Missing**: No image support in MarkdownDisplay component

5. **What rich content is available but not shown?**
   - Images: ✅ extracted, stored, retrievable → ❌ not displayed
   - Image descriptions: ✅ generated, stored → ❌ not shown
   - OCR text: ✅ extracted, stored → ⚠️ only as text
   - Charts: ✅ extracted, stored → ❌ not displayed
   - Page references: ✅ stored → ❌ not shown

---

## Key Findings by File

### DOCUMENT_PIPELINE_FINDINGS_SUMMARY.txt

**Critical Gap Identified:**
```
Infrastructure: 95% complete (extraction, storage, APIs all working)
Chat UI: 20% complete (displays text only)
Result: 30-50% of content never shown to users
```

**Missing Components:**
1. No image handler in MarkdownDisplay
2. No image embedding in search response
3. No asset gallery component
4. Tool results shown as JSON only

**Code Locations (line numbers included):**
- `modal/services/multimodal_processor.py:41-244` → Image extraction
- `fastapi/routers/dataset.py:337-450` → Asset endpoints
- `fastapi/routers/mcp.py:60-201` → Search with image URLs
- `nextjs/.../markdown-display.tsx:12-110` → **No img handler**

### DOCUMENT_EXTRACTION_PIPELINE_INVESTIGATION.md

**Detailed Extraction Process:**
1. PDF parsing (PyMuPDF4LLM primary, Docling fallback)
2. Image extraction (20-50 per document)
3. Vision model processing (classification, OCR, descriptions)
4. Chunking with metadata
5. Storage in Milvus + PostgreSQL + R2

**DocumentAsset Table Structure:**
```sql
asset_type: "extracted_image|table|chart|diagram|thumbnail"
extraction_metadata: JSON {
  "ocr_text": "...",
  "ocr_confidence": 0.92,
  "description": "A bar chart showing...",
  "image_type": "chart_bar",
  "vision_confidence": 0.87,
  "vision_provider": "gpt4v",
  "structured_data": {...},
  "surrounding_context": "..."
}
```

**Data Flow Showing Gap:**
```
Search Results (includes image_url)
    ↓
Chat API (extracts only content, not image)
    ↓
AI Model (generates markdown text only)
    ↓
Chat UI (renders markdown without image support)
    ↓
User sees: Text only ❌
```

### DOCUMENT_EXTRACTION_QUICK_REFERENCE.md

**Real Data Examples:**
- DocumentAsset JSON
- Milvus chunk with image reference
- Search result returned to chat

**Numbers at a Glance:**
- 100-page PDF: 20-50 extracted images, 0 shown
- 7-10MB total data per document
- 1 search API call per query, 0 image display calls

**Quick Verification Commands:**
```sql
SELECT COUNT(*) FROM document_assets 
WHERE asset_type = 'extracted_image';
```

---

## How to Use This Investigation

### If you're implementing image display:
1. Start with **DOCUMENT_PIPELINE_FINDINGS_SUMMARY.txt** → understand the gap
2. Reference **DOCUMENT_EXTRACTION_QUICK_REFERENCE.md** → see exact code locations
3. Check **DOCUMENT_EXTRACTION_PIPELINE_INVESTIGATION.md** → understand data structures

### If you're debugging storage issues:
1. Use **DOCUMENT_EXTRACTION_QUICK_REFERENCE.md** → verification commands
2. Check **DOCUMENT_EXTRACTION_PIPELINE_INVESTIGATION.md** → database schema
3. Cross-reference **DOCUMENT_PIPELINE_FINDINGS_SUMMARY.txt** → storage locations

### If you're optimizing the pipeline:
1. See **DOCUMENT_EXTRACTION_PIPELINE_INVESTIGATION.md** → extraction stages
2. Check **DOCUMENT_EXTRACTION_QUICK_REFERENCE.md** → numbers and storage breakdown
3. Reference **DOCUMENT_PIPELINE_FINDINGS_SUMMARY.txt** → recommendations

---

## Code Location Quick Reference

| Component | File | Lines | Purpose |
|-----------|------|-------|---------|
| **Extraction** | `modal/services/multimodal_processor.py` | 41-244 | PyMuPDF4LLM + vision |
| **Vision Models** | `modal/services/vision_service.py` | 64-150 | Image classification |
| **Chunking** | `modal/services/document_processor.py` | 168-243 | Create chunks |
| **Asset Storage** | `modal/modal_app.py` | 807-838 | Create DocumentAssets |
| **DocumentAsset Model** | `fastapi/models/document_asset.py` | 21-52 | Schema definition |
| **Asset API** | `fastapi/routers/dataset.py` | 337-450 | GET/DELETE endpoints |
| **Search API** | `fastapi/routers/mcp.py` | 60-201 | Query endpoint |
| **Embedding Search** | `fastapi/services/embedding_service.py` | 149-213 | Milvus search |
| **Chat Page** | `nextjs/.../[slug]/chat/page.tsx` | 138-186 | UI entry point |
| **Chat Interface** | `nextjs/.../chat-interface.tsx` | 48-250 | Main component |
| **Markdown Render** | `nextjs/.../markdown-display.tsx` | 12-110 | **NO IMG HANDLER** |
| **Tool Display** | `nextjs/.../tool-invocation-display.tsx` | 21-46 | JSON only |

---

## Investigation Metadata

- **Investigation Date**: November 13, 2025
- **Codebase**: FLTR (Modal + FastAPI + NextJS)
- **Scope**: Document processing pipeline end-to-end
- **Status**: Complete
- **Confidence**: High (verified with code inspection)

---

## Recommendations Summary

| Priority | Action | Impact | Effort |
|----------|--------|--------|--------|
| 1 | Add img renderer to MarkdownDisplay | Show images in chat | 5 lines |
| 2 | Embed images in search response | Make images available to AI | 10 lines |
| 3 | Create image gallery component | Display extracted images | 50 lines |
| 4 | Enhance tool visualization | Show images instead of JSON | 20 lines |
| 5 | Improve table presentation | Better styling + features | 50+ lines |

---

## Related Documentation

- `DOCUMENT_ASSET_IMPLEMENTATION.md` - Earlier documentation on asset handling
- `DOCUMENT_PROCESSING_COSTS.md` - Cost analysis of extraction pipeline

