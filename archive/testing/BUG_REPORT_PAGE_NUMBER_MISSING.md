# Bug Report: Page Numbers Missing from Chunk Metadata

## Problem
All chunks from PDF documents processed with PyMuPDF4LLM have `page: 0` instead of their actual page numbers. This prevents the migration script from finding matching images in the PDF.

## Root Cause
When PyMuPDF4LLM extracts images from PDFs, it generates filenames like:
- `filename-0-full.png` (page 0, full page image)
- `filename-1-full.png` (page 1, full page image)
- `filename-2-0.png` (page 2, image index 0)

The page number is embedded in the filename, but **it is never extracted and stored** in:
1. Image data returned from `_process_single_image()`
2. Chunk metadata created in `create_multimodal_chunks()`

## Affected Code Locations

### 1. `modal/services/multimodal_processor.py`

**Line 156-165**: When preparing images for batch processing, only `image_index` is passed, not page number:
```python
image_batch_inputs.append({
    "image_bytes": image_bytes,
    "image_index": idx,  # ❌ Only sequential index, not page number
    "image_filename": image_path.name,
    "image_format": image_path.suffix[1:].upper()
})
```

**Line 228-236**: Same issue in sequential processing fallback:
```python
for idx, image_path in enumerate(sorted(image_files)):
    image_result = await self._process_single_image(
        image_path=image_path,
        image_index=idx,  # ❌ Only sequential index
        ...
    )
```

**Line 318-419**: `_process_single_image()` receives `image_index` but never extracts page number from filename:
```python
async def _process_single_image(
    self,
    image_path: Path,
    image_index: int,  # ❌ No page number parameter
    ...
) -> Optional[Dict[str, Any]]:
    image_filename = image_path.name
    # ❌ Page number never extracted from filename

    return {
        "index": image_index,  # ❌ No "page" field
        "filename": image_filename,
        ...
    }
```

**Line 611-643**: `create_multimodal_chunks()` creates chunks but page number is missing:
```python
chunks.append({
    "text": chunk_text,
    "chunk_id": chunk_id,
    "chunk_type": "image",
    "metadata": {
        "image_index": img_data["index"],  # ❌ No "page" field
        ...
    }
})
```

### 2. `modal/services/vector_store.py`

**Line 71**: Defaults to 0 if page is missing:
```python
"page": chunk["metadata"].get("page", 0),  # ❌ Defaults to 0 if missing
```

## Impact
- Migration script cannot match chunks to images (all chunks have `page: 0`)
- Image retrieval fails because page numbers are incorrect
- OCR chunks cannot be properly associated with their source pages

## Solution
1. Extract page number from PyMuPDF4LLM filenames (format: `filename-{page}-{type}.{ext}`)
2. Store page number in image data returned from `_process_single_image()`
3. Include page number in chunk metadata when creating multimodal chunks
4. Update R2 key generation to use page number instead of sequential index

## PyMuPDF4LLM Filename Format
PyMuPDF4LLM generates filenames in the format:
- `{basename}-{page}-{type}.{ext}`
- Examples:
  - `document.pdf-0-full.png` → page 0, full page
  - `document.pdf-1-full.png` → page 1, full page
  - `document.pdf-2-0.png` → page 2, image index 0
  - `document.pdf-2-1.png` → page 2, image index 1

The page number is the second component (0-indexed).

