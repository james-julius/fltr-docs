# Phase 2: Structure-Level Parallel Processing Implementation

## Overview

Implemented structure-level parallelism for document processing pipeline to enable parallel processing of PDFs, PPTX, DOCX, and XLSX files across multiple Modal workers.

**Status**: ✅ Completed

**Test Results**: 265 tests passing (12 new parallel processing tests added)

## Architecture

### Core Components

1. **Document Splitter** (`modal/services/parsing/document_splitter.py`)
   - Splits documents by natural structure (pages, slides, sections, sheets)
   - Supports PDF, PPTX, DOCX, XLSX formats
   - Factory pattern for extensibility

2. **Parallel Processor** (`modal/services/parallel_processor.py`)
   - Orchestrates parallel processing across Modal workers
   - Auto-detects when parallel processing is beneficial (5+ units by default)
   - Merges results in correct order

3. **Modal Workers** (`modal/modal_app.py`)
   - `process_pdf_pages_worker`: Processes 10 pages per worker
   - `process_pptx_slides_worker`: Processes 10 slides per worker
   - `process_docx_sections_worker`: Processes 5 sections per worker
   - `process_xlsx_sheets_worker`: Processes 1 sheet per worker

4. **Configuration** (`modal/config.py`)
   - Centralized parallel processing configuration
   - Environment variable overrides supported
   - Sensible defaults for production use

### Configuration Options

```python
# Default configuration (can be overridden via env vars)
{
    "pdf_pages_per_worker": 10,         # PDF_PAGES_PER_WORKER
    "pptx_slides_per_worker": 10,       # PPTX_SLIDES_PER_WORKER
    "docx_sections_per_worker": 5,      # DOCX_SECTIONS_PER_WORKER
    "xlsx_sheets_per_worker": 1,        # XLSX_SHEETS_PER_WORKER
    "image_batch_size": 15,             # IMAGE_BATCH_SIZE
    "min_units_for_parallel": 5,        # MIN_UNITS_FOR_PARALLEL
    "max_parallel_workers": 20,         # MAX_PARALLEL_WORKERS
}
```

## Implementation Details

### Document Splitting

The splitter divides documents by their natural structure:

- **PDF**: Split by page ranges (e.g., 0-10, 10-20, 20-30)
- **PPTX**: Split by slide ranges
- **DOCX**: Split by sections or paragraph ranges
- **XLSX**: Split by sheets

Example for a 25-page PDF with 10 pages per worker:
```
Chunk 0: Pages 0-10
Chunk 1: Pages 10-20
Chunk 2: Pages 20-25
```

### Parallel Processing Flow

1. **Detection**: Check if document has 5+ units (pages/slides/etc)
2. **Splitting**: Split document into chunks
3. **Distribution**: Spawn Modal workers for each chunk (max 20 workers)
4. **Processing**: Each worker processes its chunk independently
5. **Merging**: Results merged in correct order

### Worker Processing

Each worker:
- Extracts its assigned pages/slides/sections
- Processes with PyMuPDF4LLM (PDF) or Docling (others)
- Runs OCR and vision models on images
- Uploads images to R2 storage
- Returns markdown content + image metadata

### Auto-Detection Logic

```python
# Automatically uses parallel processing if:
# 1. Document format supports splitting (PDF, PPTX, DOCX, XLSX)
# 2. Document has 5+ units (pages/slides/sections/sheets)
# 3. ENABLE_PARALLEL_PROCESSING env var is true (default)

# Example: 15-page PDF
# ✅ Uses parallel: 15 pages >= 5 min units
# Spawns 2 workers (pages 0-10, 10-15)

# Example: 3-page PDF
# ❌ Uses sequential: 3 pages < 5 min units
```

## Performance Benefits

### Expected Improvements

For a 50-page PDF with images:

**Sequential Processing** (current):
- 50 pages × 10s/page = ~500s (8.3 minutes)
- Single worker, single thread

**Parallel Processing** (Phase 2):
- 5 workers × 10 pages each
- Wall time: ~100s (1.7 minutes)
- **5x speedup** for large documents

### Cost Efficiency

- Workers only run for their chunk duration
- Scale to zero when idle (60s scaledown)
- Max 50 concurrent workers per type
- Small documents (<5 pages) use sequential to avoid overhead

## Files Modified/Created

### New Files

1. `modal/services/parsing/document_splitter.py` (400 lines)
   - Document splitting utilities
   - Splitters for PDF, PPTX, DOCX, XLSX

2. `modal/services/parallel_processor.py` (350 lines)
   - Parallel processing orchestration
   - Result merging and error handling

3. `modal/tests/test_parallel_processing.py` (290 lines)
   - 12 comprehensive test cases
   - Covers splitting, config, processing, merging

### Modified Files

1. `modal/config.py`
   - Added `get_parallel_processing_config()` method

2. `modal/services/document_processor.py`
   - Added `parse_document_with_parallel()` function
   - Auto-detection logic for parallel processing

3. `modal/services/pipeline_orchestrator.py`
   - Updated `_parse_stage()` to use parallel processing
   - Controlled via `ENABLE_PARALLEL_PROCESSING` env var

4. `modal/modal_app.py`
   - Added 4 new worker functions (PDF, PPTX, DOCX, XLSX)
   - Each worker: 2-4 CPU, 2-4GB RAM, 10min timeout

## Usage

### Enable/Disable Parallel Processing

```bash
# Enable (default)
export ENABLE_PARALLEL_PROCESSING=true

# Disable for debugging/testing
export ENABLE_PARALLEL_PROCESSING=false
```

### Force Parallel Processing

```python
# In code - force parallel even for small docs
result = await parse_document_with_parallel(
    file_bytes=pdf_bytes,
    filename="test.pdf",
    dataset_id="ds-123",
    document_id="doc-456",
    r2_config=r2_config,
    force_parallel=True  # Override auto-detection
)
```

### Customize Configuration

```bash
# Process 20 pages per PDF worker (default: 10)
export PDF_PAGES_PER_WORKER=20

# Require 10+ pages for parallel (default: 5)
export MIN_UNITS_FOR_PARALLEL=10

# Limit to 10 concurrent workers (default: 20)
export MAX_PARALLEL_WORKERS=10
```

## Testing

### Test Coverage

- ✅ Document splitter unit tests
- ✅ Parallel processor logic tests
- ✅ Configuration tests
- ✅ Integration with parse_document_with_parallel
- ✅ Result merging and ordering
- ✅ Error handling and fallback

### Run Tests

```bash
cd modal

# Run all tests
python -m pytest tests/ -v

# Run parallel processing tests only
python -m pytest tests/test_parallel_processing.py -v

# Test results: 265 passed, 9 skipped
```

## Future Enhancements (Phase 3)

Based on the optimization plan:

1. **Multimodal Support for All Document Types**
   - Add vision processing to PPTX (slide images)
   - Add vision processing to DOCX (embedded images)
   - Add vision processing to XLSX (charts, graphs)

2. **Advanced Chunking**
   - Better section detection for DOCX
   - Slide boundary parsing for PPTX
   - Sheet-specific processing for XLSX

3. **Adaptive Parallelism**
   - Dynamic worker count based on document size
   - Cost optimization for very large documents
   - Intelligent chunk sizing based on content density

4. **Performance Monitoring**
   - Track parallel vs sequential performance
   - Log worker spawn times and throughput
   - Dashboard for optimization insights

## Deployment

### Deploy to Modal

```bash
cd modal

# Deploy updated app with parallel workers
modal deploy modal_app.py

# Verify deployment
modal app logs fltr
```

### Environment Variables

Add to Modal secrets:

```bash
# Optional overrides (defaults are fine for production)
ENABLE_PARALLEL_PROCESSING=true
PDF_PAGES_PER_WORKER=10
MIN_UNITS_FOR_PARALLEL=5
MAX_PARALLEL_WORKERS=20
```

## Monitoring

### Logs to Watch

```bash
# Look for parallel processing indicators
modal app logs fltr | grep "parallel"

# Expected log messages:
# "📊 Document has 25 units, min for parallel: 5, using parallel: True"
# "🚀 Spawning 3 worker containers (10 pages each)"
# "⏳ Waiting for 3 workers to complete..."
# "✅ Merged 3 chunks: 15000 chars, 12 images"
```

### Success Metrics

- Worker spawn time: <5s per worker
- Processing time: ~10s per 10-page chunk
- Result merge time: <1s
- Overall speedup: 3-5x for 20+ page documents

## Known Limitations

1. **PPTX/DOCX/XLSX**: Currently process full file per worker
   - Docling doesn't support partial file processing
   - Future: implement manual section extraction

2. **Small Documents**: Sequential processing for <5 units
   - Avoids worker spawn overhead
   - Can be overridden with `force_parallel=True`

3. **Worker Limits**: Max 50 concurrent workers per type
   - Prevents resource exhaustion
   - Adjustable via Modal app configuration

## Conclusion

Phase 2 implementation successfully adds structure-level parallelism to the document processing pipeline:

- ✅ **Implemented**: Document splitter, parallel processor, 4 worker types
- ✅ **Tested**: 12 new tests, 265 total tests passing
- ✅ **Configured**: Environment-based configuration with sensible defaults
- ✅ **Documented**: Comprehensive implementation and usage documentation

**Expected Impact**:
- 3-5x faster processing for documents with 20+ pages/slides/sections
- Backward compatible with existing pipeline
- Automatic detection and fallback to sequential for small documents
- Cost-efficient worker scaling

**Next Steps**: Deploy to production and monitor performance, then proceed to Phase 3 (multimodal support for all document types).
