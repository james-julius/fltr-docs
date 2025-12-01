# Document Processing Pipeline Optimization Plan

## Overview

Optimize document processing pipeline to achieve **5-10x speedup** for large documents across all supported formats through structure-level parallelism and multimodal enhancements.

**Supported Document Types:**
- **PDF** (PyMuPDF4LLM + Multimodal processing)
- **DOCX, PPTX, XLSX** (Docling parser)
- **HTML, MD, TXT** (Docling parser)

---

## Current Architecture

### Processing Flow (All Document Types)

```
Document Upload → R2 → Webhook → Modal Pipeline
                                     ↓
                    (Download → Parse → Chunk → Embed → Store → DB_Update)
                                     ↓
                         [Type-Specific Parser]
                                     ↓
                    PDF: PyMuPDF4LLM + Images (sequential)
                    DOCX/PPTX/XLSX: Docling (sequential, no images)
```

### Current Bottlenecks

#### 1. PDF Processing (PyMuPDF4LLM)
- **Single-threaded**: Entire document processed sequentially
- **Image batch size too small**: 5 images per worker
- **All images loaded before batching**: Memory inefficient
- **Sequential OCR + Vision**: Each image processed one at a time
- **No page-level parallelism**: Can't process pages concurrently

#### 2. DOCX/PPTX/XLSX Processing (Docling)
- **Single-threaded**: Entire document processed sequentially
- **No multimodal support**: Images in DOCX/PPTX ignored (returns `"images": []`)
- **No structural parallelism**: Can't process slides/sections concurrently
- **No image extraction**: PPTX slides with charts/diagrams lost

#### 3. Universal Issues
- No caching of parsed content for re-processing
- No early filtering of low-value content
- No parallel structure-level processing

---

## Optimization Strategy: Structure-Level Parallelism

### Core Principle

Instead of processing entire document sequentially, **split by natural structure and process in parallel**:

| Document Type | Natural Structure | Worker Split |
|---------------|------------------|--------------|
| **PDF** | Pages | 10 pages per worker |
| **PPTX** | Slides | 10 slides per worker |
| **DOCX** | Sections/page breaks | 5 sections per worker |
| **XLSX** | Sheets | 1 sheet per worker |
| **HTML** | Major sections/headers | Configurable |

---

## Phase 1: Quick Wins (1-2 days)

### 1.1 Increase Image Batch Size (PDF)

**Current**: 5 images per worker
**Proposed**: 15 images per worker

```python
# In modal/services/multimodal_processor.py
BATCH_SIZE = 15  # Up from 5
```

**Expected Speedup**: 1.5-2x (fewer workers, less overhead)

---

### 1.2 Parallelize OCR + Vision Within Worker (PDF)

**Current**: Sequential per image
**Proposed**: Concurrent OCR + Vision

```python
async def process_single_image(image_bytes: bytes) -> Dict:
    """Process a single image with parallel OCR + Vision."""
    # Run OCR and Vision in parallel
    ocr_task = asyncio.create_task(ocr_image(image_bytes))
    vision_task = asyncio.create_task(vision_service.analyze_image(image_bytes))

    ocr_result, vision_result = await asyncio.gather(ocr_task, vision_task)

    # Upload to R2 (after both complete)
    r2_key = await upload_to_r2(image_bytes)

    return {
        "ocr": ocr_result,
        "vision": vision_result,
        "r2_key": r2_key
    }
```

**Expected Speedup**: 1.5-2x per worker

---

### 1.3 Smarter Content Filtering (All Types)

Filter low-value content early to avoid processing.

```python
def should_extract_image(img_obj, page_num, total_pages):
    """Quick pre-filter before extraction."""
    # Skip cover page decorative images
    if page_num <= 1 and not looks_like_chart_or_table(img_obj):
        return False

    # Skip tiny images (<100x100)
    if img_obj.width < 100 or img_obj.height < 100:
        return False

    # Skip decorative images (too small file size)
    if len(img_obj.image_bytes) < 5000:  # < 5KB
        return False

    return True
```

**Expected Speedup**: 1.2-1.5x (fewer images to process)

---

### Phase 1 Summary

**Combined Expected Speedup**: 2-3x
**Effort**: 1-2 days
**Risk**: Low (incremental changes to existing code)

---

## Phase 2: Structure-Level Parallelism (3-5 days) ⭐ **Highest Impact**

### 2.1 PDF: Page-Level Parallelism

Split PDF into page ranges and process in parallel.

```python
async def process_pdf_parallel_pages(
    file_bytes: bytes,
    filename: str,
    max_pages_per_worker: int = 10
) -> Dict[str, Any]:
    """
    Split PDF into page ranges and process in parallel.

    Flow:
    1. Get PDF page count
    2. Split into ranges (e.g., 0-9, 10-19, 20-29, ...)
    3. Spawn worker for each range
    4. Each worker:
       - Extracts pages (PyMuPDF)
       - Runs PyMuPDF4LLM on just those pages
       - Processes images from those pages
    5. Merge results (markdown + images)
    """
    import fitz  # PyMuPDF

    # Get page count
    doc = fitz.open(stream=file_bytes, filetype="pdf")
    total_pages = doc.page_count
    doc.close()

    # Calculate page ranges
    page_ranges = []
    for start in range(0, total_pages, max_pages_per_worker):
        end = min(start + max_pages_per_worker, total_pages)
        page_ranges.append((start, end))

    # Spawn workers in parallel
    worker_calls = []
    for start_page, end_page in page_ranges:
        call = process_pdf_page_range_worker.spawn(
            file_bytes=file_bytes,
            start_page=start_page,
            end_page=end_page,
            filename=filename,
            dataset_id=dataset_id,
            document_id=document_id
        )
        worker_calls.append(call)

    # Wait for all page range workers
    page_results = await asyncio.gather(*[c.get.aio() for c in worker_calls])

    # Merge markdown and images
    merged_markdown = "\n\n".join([r["markdown"] for r in page_results])
    merged_images = []
    for r in page_results:
        merged_images.extend(r["images"])

    return {
        "content": merged_markdown,
        "images": merged_images
    }
```

**Expected Speedup**: 5-10x for large PDFs (100+ pages)

---

### 2.2 PPTX: Slide-Level Parallelism

Split PowerPoint into slide ranges and process in parallel.

```python
async def process_pptx_parallel_slides(
    file_bytes: bytes,
    filename: str,
    max_slides_per_worker: int = 10
) -> Dict[str, Any]:
    """
    Split PPTX into slide ranges and process in parallel.

    Flow:
    1. Extract slide count from PPTX
    2. Split into ranges (e.g., 0-9, 10-19, ...)
    3. Spawn worker for each range
    4. Each worker:
       - Extracts slides
       - Runs Docling on slide subset
       - Extracts images from slides
    5. Merge results
    """
    from pptx import Presentation
    import io

    # Get slide count
    prs = Presentation(io.BytesIO(file_bytes))
    total_slides = len(prs.slides)

    # Calculate slide ranges
    slide_ranges = []
    for start in range(0, total_slides, max_slides_per_worker):
        end = min(start + max_slides_per_worker, total_slides)
        slide_ranges.append((start, end))

    # Spawn workers in parallel
    worker_calls = []
    for start_slide, end_slide in slide_ranges:
        call = process_pptx_slide_range_worker.spawn(
            file_bytes=file_bytes,
            start_slide=start_slide,
            end_slide=end_slide,
            filename=filename,
            dataset_id=dataset_id,
            document_id=document_id
        )
        worker_calls.append(call)

    # Wait and merge
    slide_results = await asyncio.gather(*[c.get.aio() for c in worker_calls])

    merged_markdown = "\n\n---\n\n".join([r["markdown"] for r in slide_results])
    merged_images = []
    for r in slide_results:
        merged_images.extend(r["images"])

    return {
        "content": merged_markdown,
        "images": merged_images
    }
```

**Expected Speedup**: 5-10x for large PPTX (50+ slides)

---

### 2.3 DOCX: Section-Level Parallelism

Split Word documents by sections/page breaks.

```python
async def process_docx_parallel_sections(
    file_bytes: bytes,
    filename: str,
    max_pages_per_worker: int = 5  # Approximate pages
) -> Dict[str, Any]:
    """
    Split DOCX into sections and process in parallel.

    Uses heuristic based on page breaks or section headers.
    """
    from docx import Document
    import io

    # Parse document structure
    doc = Document(io.BytesIO(file_bytes))

    # Split by section breaks or heading hierarchy
    sections = split_docx_by_sections(doc)

    # Spawn workers
    worker_calls = []
    for section_idx, section_content in enumerate(sections):
        call = process_docx_section_worker.spawn(
            section_bytes=section_content,
            section_idx=section_idx,
            filename=filename,
            dataset_id=dataset_id,
            document_id=document_id
        )
        worker_calls.append(call)

    # Wait and merge
    section_results = await asyncio.gather(*[c.get.aio() for c in worker_calls])

    merged_markdown = "\n\n".join([r["markdown"] for r in section_results])
    merged_images = []
    for r in section_results:
        merged_images.extend(r["images"])

    return {
        "content": merged_markdown,
        "images": merged_images
    }
```

**Expected Speedup**: 3-5x for large DOCX (20+ pages)

---

### 2.4 XLSX: Sheet-Level Parallelism

Process each Excel sheet in parallel.

```python
async def process_xlsx_parallel_sheets(
    file_bytes: bytes,
    filename: str
) -> Dict[str, Any]:
    """Process each Excel sheet in parallel."""
    import openpyxl
    import io

    # Load workbook
    wb = openpyxl.load_workbook(io.BytesIO(file_bytes), read_only=True)
    sheet_names = wb.sheetnames

    # Spawn workers for each sheet
    worker_calls = []
    for sheet_name in sheet_names:
        call = process_xlsx_sheet_worker.spawn(
            file_bytes=file_bytes,
            sheet_name=sheet_name,
            filename=filename,
            dataset_id=dataset_id,
            document_id=document_id
        )
        worker_calls.append(call)

    # Wait and merge
    sheet_results = await asyncio.gather(*[c.get.aio() for c in worker_calls])

    # Merge with sheet separators
    merged_markdown = "\n\n# Sheet Separator\n\n".join([r["markdown"] for r in sheet_results])

    return {
        "content": merged_markdown,
        "images": []  # XLSX rarely has extractable images
    }
```

**Expected Speedup**: 3-5x for multi-sheet workbooks

---

### Phase 2 Summary

**Combined Expected Speedup**: 5-10x for large documents
**Effort**: 3-5 days
**Risk**: Medium (requires new worker functions, merge logic)

---

## Phase 3: Multimodal Support for All Types (2-3 days)

### 3.1 Add Image Extraction to DOCX/PPTX

Extend `MultimodalProcessor` to support non-PDF documents.

```python
class MultimodalProcessor:
    """Process images from any document type."""

    async def extract_images_from_docx(
        self,
        file_bytes: bytes,
        filename: str
    ) -> List[Dict]:
        """Extract embedded images from DOCX."""
        from docx import Document
        import io

        doc = Document(io.BytesIO(file_bytes))
        images = []

        for rel in doc.part.rels.values():
            if "image" in rel.target_ref:
                image_data = rel.target_part.blob
                images.append({
                    "data": image_data,
                    "format": rel.target_ref.split('.')[-1]
                })

        return images

    async def extract_images_from_pptx(
        self,
        file_bytes: bytes,
        filename: str
    ) -> List[Dict]:
        """Extract images from PPTX slides."""
        from pptx import Presentation
        import io

        prs = Presentation(io.BytesIO(file_bytes))
        images = []

        for slide_idx, slide in enumerate(prs.slides):
            for shape in slide.shapes:
                if hasattr(shape, "image"):
                    image_data = shape.image.blob
                    images.append({
                        "data": image_data,
                        "format": shape.image.ext,
                        "slide": slide_idx
                    })

        return images
```

**Expected Benefit**: Better extraction quality for PPTX presentations with charts/diagrams

---

### 3.2 Vision Analysis for All Image Types

Apply vision models to extracted images from any document type.

```python
async def process_document_with_multimodal(
    file_bytes: bytes,
    filename: str,
    document_type: str,
    dataset_id: str,
    document_id: str,
    r2_config: Dict[str, str]
) -> Dict[str, Any]:
    """Unified multimodal processing for all document types."""

    # Extract images based on type
    if document_type == "pdf":
        images = await extract_images_from_pdf(file_bytes)
    elif document_type == "docx":
        images = await extract_images_from_docx(file_bytes)
    elif document_type == "pptx":
        images = await extract_images_from_pptx(file_bytes)
    else:
        images = []

    # Process images with vision models (same pipeline for all)
    if images:
        processed_images = await process_images_parallel(
            images=images,
            context=markdown_context,
            dataset_id=dataset_id,
            document_id=document_id,
            r2_config=r2_config
        )
    else:
        processed_images = []

    return {
        "content": markdown,
        "images": processed_images
    }
```

---

### Phase 3 Summary

**Expected Benefit**: Multimodal support for DOCX/PPTX (currently missing)
**Effort**: 2-3 days
**Risk**: Low (extends existing multimodal pipeline)

---

## Implementation Plan Summary

### Phase 1: Quick Wins (1-2 days)
- ✅ Increase image batch size (15 per worker)
- ✅ Parallelize OCR + Vision within worker
- ✅ Smarter content filtering

**Expected**: 2-3x speedup

### Phase 2: Structure-Level Parallelism (3-5 days) ⭐
- ✅ PDF: Page-level parallelism (10 pages/worker)
- ✅ PPTX: Slide-level parallelism (10 slides/worker)
- ✅ DOCX: Section-level parallelism
- ✅ XLSX: Sheet-level parallelism

**Expected**: 5-10x speedup for large documents

### Phase 3: Multimodal for All Types (2-3 days)
- ✅ Extract images from DOCX/PPTX
- ✅ Apply vision models to all image types
- ✅ Cache parsed content for re-processing

**Expected**: Better quality + faster re-processing

---

## File Changes Required

### New Files

| File | Purpose |
|------|---------|
| `/modal/services/parsing/parallel_pdf_processor.py` | PDF page-range parallelism |
| `/modal/services/parsing/parallel_pptx_processor.py` | PPTX slide-range parallelism |
| `/modal/services/parsing/parallel_docx_processor.py` | DOCX section-range parallelism |
| `/modal/services/parsing/parallel_xlsx_processor.py` | XLSX sheet parallelism |
| `/modal/workers/document_range_worker.py` | Generic worker for document ranges |
| `/modal/services/parsing/document_splitter.py` | Split documents by structure |

### Modified Files

| File | Changes |
|------|---------|
| `/modal/services/multimodal_processor.py` | Increase BATCH_SIZE, parallelize OCR+Vision, add DOCX/PPTX image extraction |
| `/modal/services/parsing/document_parsers.py` | Add parallel processing paths for each parser |
| `/modal/modal_app.py` | Add new worker functions for document ranges |
| `/modal/services/pipeline_orchestrator.py` | Route to parallel processors for large documents |
| `/modal/constants.py` | Add configuration for parallel processing thresholds |

---

## Configuration

Add to environment/config:

```python
# Document processing optimization settings
ENABLE_STRUCTURE_LEVEL_PARALLELISM = True

# Per-document-type thresholds for parallel processing
PDF_PARALLEL_PAGE_THRESHOLD = 20  # Parallelize if >20 pages
PPTX_PARALLEL_SLIDE_THRESHOLD = 15  # Parallelize if >15 slides
DOCX_PARALLEL_SECTION_THRESHOLD = 10  # Parallelize if >10 sections
XLSX_PARALLEL_SHEET_THRESHOLD = 3  # Parallelize if >3 sheets

# Worker configuration
MAX_PAGES_PER_WORKER = 10  # PDF
MAX_SLIDES_PER_WORKER = 10  # PPTX
MAX_SECTIONS_PER_WORKER = 5  # DOCX

# Image processing
IMAGE_BATCH_SIZE = 15  # Up from 5
ENABLE_OCR_VISION_PARALLEL = True
MIN_IMAGE_SIZE_BYTES = 5000

# Caching
ENABLE_PARSE_CACHING = True
```

---

## Success Criteria

- ✅ Parse time for 100-page PDF: **< 30 seconds** (currently ~4 minutes)
- ✅ Parse time for 50-slide PPTX: **< 20 seconds** (currently ~2 minutes)
- ✅ Parse time for large DOCX: **< 15 seconds** (currently ~1 minute)
- ✅ No regression in extraction accuracy
- ✅ No cost increase > 20%
- ✅ All existing tests pass
- ✅ Multimodal support for PPTX/DOCX images

---

## Monitoring & Metrics

Track these metrics per document type:

```python
# Before optimization (current)
parse_time_per_pdf_page = 2.5s
parse_time_per_pptx_slide = 2.0s
parse_time_per_docx_page = 1.5s

# After Phase 1 (quick wins)
parse_time_per_pdf_page = 1.0s
parse_time_per_pptx_slide = 0.8s
parse_time_per_docx_page = 0.6s

# After Phase 2 (structure parallelism)
parse_time_per_pdf_page = 0.25s  # 10 workers in parallel
parse_time_per_pptx_slide = 0.2s  # 10 workers in parallel
parse_time_per_docx_page = 0.3s  # 5 workers in parallel
```

---

## Testing Strategy

1. **Unit tests** for document splitting logic (all types)
2. **Integration tests** with sample documents:
   - PDF: 10, 50, 100 pages
   - PPTX: 10, 30, 50 slides
   - DOCX: 5, 15, 30 pages
   - XLSX: 2, 5, 10 sheets
3. **Benchmark suite** to measure speedup per document type
4. **Comparison tests**: Old vs new pipeline side-by-side

```python
async def test_parse_speed_all_types():
    docs = [
        ("small.pdf", 10, "pdf"),
        ("large.pdf", 100, "pdf"),
        ("presentation.pptx", 30, "pptx"),
        ("report.docx", 20, "docx"),
        ("spreadsheet.xlsx", 5, "xlsx"),
    ]

    for doc, expected_units, doc_type in docs:
        start = time.time()
        result = await parse_document_optimized(doc)
        elapsed = time.time() - start

        assert len(result["content"]) > 0
        print(f"{doc} ({doc_type}): {elapsed:.2f}s ({elapsed/expected_units:.2f}s per unit)")
```

---

## Risk Mitigation

### Risk 1: Structure splitting breaks content flow
**Mitigation**: Add overlap between ranges, test with various document types

### Risk 2: Too many workers = rate limiting
**Mitigation**: Limit concurrent workers with semaphore, add retry logic

### Risk 3: Merge logic creates duplicate/missing content
**Mitigation**: Extensive integration tests, validate merged output

### Risk 4: DOCX/PPTX image extraction quality issues
**Mitigation**: Compare with manual extraction, add quality checks

---

## Cost Considerations

**Modal Costs**:
- Worker spawn overhead: ~$0.001 per worker
- Compute: ~$0.0001 per CPU-second

**Example** (100-page PDF with 50 images):
- **Current**: 1 main + 10 image workers = ~$0.011 + 250s compute
- **Phase 1**: 1 main + 4 image workers = ~$0.005 + 100s compute
- **Phase 2**: 10 page workers + 4 image workers = ~$0.014 + 25s compute

**Net Result**: Phase 2 costs slightly more in workers but **10x faster** = better UX

---

## Next Steps

1. Review plan with team
2. Prioritize phases based on document type usage (PDF vs DOCX vs PPTX)
3. Start with Phase 1 (quick wins) for immediate gains
4. Implement Phase 2 for most-used document type first (likely PDF)
5. Extend to other document types incrementally
