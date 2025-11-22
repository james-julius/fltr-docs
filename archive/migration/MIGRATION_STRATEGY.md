# Page Number Migration Strategy - Performance & Prioritization

## Time Estimates

### Migration Script (Fast Path)
- **Per document:** ~5-30 seconds
  - List images in R2: ~1-2 seconds
  - Extract page numbers: <1 second
  - Update chunks: ~2-5 seconds
  - Image migration: ~10-20 seconds (if needed)
- **100 documents:** ~8-50 minutes
- **1000 documents:** ~1.5-8 hours

### Full Reprocessing (Slow Path)
- **Per document:** ~30 seconds - 5 minutes
  - Depends on document size, number of images, vision processing
- **100 documents:** ~50 minutes - 8 hours
- **1000 documents:** ~8-83 hours (1-3 days)

## Optimization Strategies

### 1. Prioritize Active Documents

Only migrate documents that are actually being used:

```python
# Find documents with recent queries/usage
# Migrate these first, others can wait
```

### 2. Batch Processing with Limits

Process in smaller batches to avoid overwhelming the system:

```bash
# Process 10 at a time
for i in {1..10}; do
  python fastapi/scripts/migrate_image_r2_keys.py --limit 10
  sleep 60  # Wait between batches
done
```

### 3. Parallel Processing

Run multiple migration processes in parallel (if safe):

```bash
# Split documents into batches and process in parallel
# Be careful not to overload Milvus/R2
```

### 4. Incremental Migration

Only migrate when documents are accessed:

```python
# Add lazy migration: when a document is queried,
# check if it needs migration and fix on-the-fly
```

### 5. Skip Documents Without Images

If a document has no images in R2 and no image chunks, skip it:

```python
# Check if document actually needs migration
# Skip if no image-related chunks exist
```

## Recommended Approach

### Phase 1: Quick Win (Low Effort, High Value)
1. **Identify documents that actually need fixing**
   - Only documents with image/OCR chunks
   - Skip pure text documents

2. **Check if images exist in R2**
   - If yes → Use fast migration script
   - If no → Can defer to Phase 2

3. **Process in batches of 50-100**
   - Monitor progress
   - Stop if issues arise

### Phase 2: Lazy Migration (On-Demand)
1. **Add migration check to document retrieval**
   - When document is accessed, check if page=0
   - If yes, trigger migration automatically
   - User doesn't wait, migration happens in background

2. **Gradually fixes documents as they're used**
   - No upfront cost
   - Only fixes what's actually needed

### Phase 3: Background Job (Optional)
1. **Set up scheduled job**
   - Process N documents per day
   - Low priority, doesn't impact users
   - Eventually all documents get fixed

## Implementation: Lazy Migration

Add to document retrieval endpoint:

```python
async def get_document_chunks(document_id: str):
    # Check if document needs migration
    needs_migration = check_if_needs_page_migration(document_id)

    if needs_migration:
        # Trigger async migration (don't wait)
        asyncio.create_task(migrate_document_async(document_id))

    # Return chunks immediately (may have page=0 temporarily)
    return get_chunks(document_id)
```

## Monitoring Progress

Track migration progress:

```python
# Add migration status tracking
migration_status = {
    "total_documents": 1000,
    "migrated": 150,
    "failed": 5,
    "pending": 845,
    "estimated_time_remaining": "6 hours"
}
```

## Cost Considerations

### Migration Script
- **R2 API calls:** ~$0.0004 per 1000 requests
- **Milvus queries:** Minimal cost
- **Total:** Very low cost

### Full Reprocessing
- **Modal compute:** ~$0.10-1.00 per document (depends on size)
- **Vision API:** ~$0.01-0.10 per image
- **OCR:** ~$0.001 per image
- **Total:** Can be expensive for large documents

## Recommendation

**Start with lazy migration:**
1. Add migration check to document access
2. Fix documents as they're used
3. Monitor which documents are actually accessed
4. Only proactively migrate frequently-used documents

**Benefits:**
- No upfront time/cost
- Fixes what matters most
- Can still do batch migration later if needed
- Users don't notice any delay

## Quick Script: Find Documents That Need Migration

```python
# Find documents with page=0 chunks
from database.vector_store import get_milvus_client

milvus_client = get_milvus_client()

# Count affected chunks
filter_expr = 'page == 0 && (chunk_type == "ocr" || chunk_type == "image" || ocr_confidence > 0.0)'
chunks = milvus_client.query(
    collection_name="fltr_documents",
    filter=filter_expr,
    output_fields=["document_id"]
)

# Count unique documents
affected_docs = set(c['document_id'] for c in chunks)
print(f"Found {len(affected_docs)} documents with page=0 chunks")
print(f"Total affected chunks: {len(chunks)}")

# Estimate time
migration_time = len(affected_docs) * 0.5  # 30 seconds per doc average
print(f"Estimated migration time: {migration_time/60:.1f} minutes ({migration_time/3600:.1f} hours)")
```

