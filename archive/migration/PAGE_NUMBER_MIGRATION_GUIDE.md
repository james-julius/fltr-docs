# Page Number Migration Guide

## Overview

This guide explains how to migrate existing documents that have `page: 0` due to the bug where page numbers weren't extracted from PyMuPDF4LLM image filenames.

## Solutions

### Option 1: Migration Script (Recommended for Documents with Existing Images)

The enhanced migration script can now:
1. Detect chunks with `page: 0`
2. List existing images in R2
3. Extract page numbers from image filenames
4. Update chunks with correct page numbers
5. Continue with normal image migration

**Usage:**
```bash
# Single document
python fastapi/scripts/migrate_image_r2_keys.py --document-id YOUR_DOCUMENT_ID

# Dry run first
python fastapi/scripts/migrate_image_r2_keys.py --document-id YOUR_DOCUMENT_ID --dry-run

# Multiple documents (with limit)
python fastapi/scripts/migrate_image_r2_keys.py --limit 10
```

**What it does:**
1. Queries chunks with `page >= 0` (includes page=0)
2. Lists all images in R2 for the document
3. Extracts page numbers from image filenames (supports both old and new formats)
4. Updates chunks in Milvus with correct page numbers
5. Proceeds with normal image migration

**Supported image formats:**
- New format: `{page}_{index}.{ext}` (e.g., `5_0.png`)
- Old format: `{index}_{filename}` (e.g., `0_document.pdf-5-full.png`)
  - Extracts page number from filename using PyMuPDF4LLM pattern

### Option 2: Full Reprocessing (For Documents Without Images)

If images don't exist in R2, use full reprocessing:

**Via API:**
```bash
# Single document
curl -X POST "https://your-api.com/api/v1/admin/documents/{document_id}/reprocess" \
  -H "Authorization: Bearer YOUR_TOKEN"

# Batch reprocess
curl -X POST "https://your-api.com/api/v1/admin/documents/batch-reprocess" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "dataset_id": "your-dataset-id",
    "status": "ready",
    "limit": 100
  }'
```

**Pros:**
- Ensures all data is up-to-date
- Fixes page numbers automatically
- Updates all metadata

**Cons:**
- More expensive (re-runs OCR, vision, embeddings)
- Takes longer

## Migration Workflow

### Step 1: Identify Affected Documents

```python
# Query to find documents with page=0 chunks
from database.vector_store import get_milvus_client
from database.sql_store import get_session
from models.document import Document
from sqlmodel import select

milvus_client = get_milvus_client()
session = get_session()

# Find chunks with page=0
filter_expr = 'page == 0 && (chunk_type == "ocr" || chunk_type == "image" || ocr_confidence > 0.0)'
chunks = milvus_client.query(
    collection_name="fltr_documents",
    filter=filter_expr,
    output_fields=["document_id"]
)

# Get unique document IDs
affected_doc_ids = set(c['document_id'] for c in chunks)
print(f"Found {len(affected_doc_ids)} affected documents")
```

### Step 2: Check Image Availability

For each document, check if images exist in R2:
- If images exist → Use migration script
- If no images → Use reprocessing

### Step 3: Run Migration

```bash
# Test on one document first
python fastapi/scripts/migrate_image_r2_keys.py \
  --document-id YOUR_DOCUMENT_ID \
  --dry-run

# If dry run looks good, run for real
python fastapi/scripts/migrate_image_r2_keys.py \
  --document-id YOUR_DOCUMENT_ID
```

### Step 4: Verify Results

```python
# Check that page numbers were updated
filter_expr = f'document_id == "{document_id}" && page > 0'
chunks = milvus_client.query(
    collection_name="fltr_documents",
    filter=filter_expr,
    output_fields=["chunk_id", "page", "image_index"]
)
print(f"Found {len(chunks)} chunks with page > 0")
```

## Troubleshooting

### Issue: Migration script finds no images

**Solution:** Images may not exist in R2. Use reprocessing instead.

### Issue: Page numbers still 0 after migration

**Possible causes:**
1. Images don't exist in R2
2. Image filenames don't contain page numbers
3. Image format is not recognized

**Solution:** Use full reprocessing to extract images fresh from PDF.

### Issue: Migration script fails to update chunks

**Check:**
1. Milvus connection is working
2. Chunks have `chunk_id` field
3. Chunks have vectors (required for upsert)

**Solution:** Check Milvus logs and ensure chunks have all required fields.

## Testing

Before running on production:

1. **Test on single document:**
   ```bash
   python fastapi/scripts/migrate_image_r2_keys.py \
     --document-id TEST_DOCUMENT_ID \
     --dry-run
   ```

2. **Verify page numbers:**
   - Check that chunks are updated with correct page numbers
   - Verify images can be retrieved using page numbers

3. **Test image retrieval:**
   - Verify that image URLs work correctly
   - Check that page numbers match actual PDF pages

## Rollback

If migration causes issues:

1. **Reprocess document:**
   ```bash
   curl -X POST "https://your-api.com/api/v1/admin/documents/{document_id}/reprocess" \
     -H "Authorization: Bearer YOUR_TOKEN"
   ```

2. **Delete and recreate chunks:**
   - Delete chunks from Milvus
   - Reprocess document to recreate chunks

3. **Images in R2 are preserved:**
   - Migration doesn't delete images
   - Can always reprocess to fix issues

## Performance Considerations

- **Migration script:** Fast, only updates metadata
- **Reprocessing:** Slower, re-runs full pipeline
- **Batch operations:** Use limits to avoid overwhelming system

## Next Steps

1. Test migration script on a few documents
2. Verify page numbers are correct
3. Run batch migration for affected documents
4. Monitor for any issues
5. Update any documents that still have issues

