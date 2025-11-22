# Migrating Page Numbers for Existing Documents

## Problem
Existing documents processed before the page number extraction fix have all chunks with `page: 0`, preventing proper image matching and migration.

## Solutions

### Option 1: Full Reprocessing (Recommended for Most Cases)

**Pros:**
- Ensures all data is up-to-date with latest processing logic
- Fixes page numbers automatically
- Updates all metadata, embeddings, and image references

**Cons:**
- More expensive (re-runs OCR, vision models, embeddings)
- Takes longer
- May change chunk boundaries/text

**How to:**

#### Single Document
```bash
# Via API
curl -X POST "https://your-api.com/api/v1/admin/documents/{document_id}/reprocess" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

#### Batch Reprocessing
```bash
# Via API
curl -X POST "https://your-api.com/api/v1/admin/documents/batch-reprocess" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "dataset_id": "your-dataset-id",
    "status": "ready",
    "limit": 100
  }'
```

#### Via Python Script
```python
from fastapi.routers.admin import reprocess_document
# Use admin endpoint to trigger reprocessing
```

### Option 2: Migration Script Update (For Documents with Existing Images)

Update the migration script to:
1. Extract page numbers from existing R2 image filenames
2. Update Milvus chunks with correct page numbers
3. Re-extract images if needed

**Pros:**
- Faster than full reprocessing
- Preserves existing embeddings
- Only updates page numbers

**Cons:**
- Only works if images already exist in R2
- May not work if images were stored with old naming format

**How to:**

```bash
# Run migration script with --fix-page-numbers flag
python fastapi/scripts/restore_migrate_image_r2_keys.py \
  --document-id YOUR_DOCUMENT_ID \
  --fix-page-numbers
```

### Option 3: Hybrid Approach

1. First, try migration script to fix page numbers from existing images
2. For documents without images or with incorrect image paths, use reprocessing

## Implementation Plan

### Step 1: Identify Affected Documents

```sql
-- Find documents with chunks that have page = 0
SELECT DISTINCT d.id, d.file_name, d.dataset_id, COUNT(*) as chunk_count
FROM documents d
JOIN (
  SELECT DISTINCT document_id
  FROM milvus_chunks
  WHERE page = 0 AND (chunk_type = 'image' OR chunk_type = 'ocr' OR ocr_confidence > 0)
) AS affected ON d.id = affected.document_id
WHERE d.status = 'ready'
GROUP BY d.id, d.file_name, d.dataset_id;
```

### Step 2: Check if Images Exist in R2

For each document, check if images exist:
- Old format: `{dataset_id}/{document_id}/images/{index}_{filename}`
- New format: `{dataset_id}/{document_id}/images/{page}_{index}.{ext}`

### Step 3: Choose Migration Strategy

- **Images exist in old format**: Use migration script to extract page numbers from filenames
- **Images exist in new format**: Use migration script to update chunks
- **No images**: Use reprocessing

## Migration Script Enhancement

The migration script should be enhanced to:

1. **Extract page numbers from R2 image filenames**
   - Parse `{index}_{filename}` format to extract page from filename
   - Example: `0_document.pdf-5-full.png` → page 5

2. **Update chunks with correct page numbers**
   - Query chunks with `page = 0`
   - Match chunks to images by `image_index`
   - Update chunk `page` field in Milvus

3. **Re-extract images if needed**
   - If images don't exist, extract from PDF
   - Use correct page numbers from PDF extraction

## Example Workflow

```python
# 1. Find affected documents
affected_docs = find_documents_with_page_zero()

# 2. For each document
for doc in affected_docs:
    # Check if images exist
    images_exist = check_images_in_r2(doc)

    if images_exist:
        # Try migration
        migrate_page_numbers_from_images(doc)
    else:
        # Reprocess
        reprocess_document(doc.id)
```

## Testing

Before running on production:

1. Test on a single document
2. Verify page numbers are updated correctly
3. Verify images can be retrieved
4. Verify migration script works with both old and new formats

## Rollback Plan

If migration causes issues:
1. Documents can be reprocessed to restore correct state
2. Chunks can be deleted and recreated
3. Images in R2 are preserved (not deleted)

