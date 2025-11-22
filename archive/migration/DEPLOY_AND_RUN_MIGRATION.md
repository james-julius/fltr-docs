# Deploy and Run Page Number Migration

## Step 1: Deploy Changes

Deploy the updated code to your server. The changes include:
- `modal/services/multimodal_processor.py` - Page number extraction fix
- `modal/modal_app.py` - Worker updates for page numbers
- `fastapi/scripts/migrate_image_r2_keys.py` - Enhanced migration script
- `fastapi/scripts/batch_migrate_page_numbers.py` - Batch migration script

## Step 2: Check How Many Documents Are Affected

First, let's see how many documents need migration:

```bash
# SSH into your server
ssh your-server

# Navigate to your FastAPI directory
cd /path/to/fastapi

# Activate your virtual environment (if using one)
source venv/bin/activate  # or your venv path

# Run a quick check
python3 -c "
from database.vector_store import get_milvus_client

milvus_client = get_milvus_client()
chunks = milvus_client.query(
    collection_name='fltr_documents',
    filter='page == 0 && (chunk_type == \"ocr\" || chunk_type == \"image\" || ocr_confidence > 0.0)',
    output_fields=['document_id']
)

affected_docs = len(set(c['document_id'] for c in chunks))
print(f'Affected documents: {affected_docs}')
print(f'Total affected chunks: {len(chunks)}')
print(f'Estimated migration time: {affected_docs * 0.5 / 60:.1f} minutes')
"
```

## Step 3: Test on a Single Document (Recommended)

Test the migration on one document first to make sure everything works:

```bash
# Find a document ID to test with (or use a known one)
# Then run dry-run first
python3 scripts/batch_migrate_page_numbers.py \
  --document-id YOUR_DOCUMENT_ID \
  --dry-run

# If dry-run looks good, run for real on that one document
python3 scripts/batch_migrate_page_numbers.py \
  --document-id YOUR_DOCUMENT_ID \
  --batch-size 1
```

## Step 4: Run Dry-Run on All Documents

Before migrating everything, do a dry-run to see what would happen:

```bash
# Dry-run on first 10 documents (to test)
python3 scripts/batch_migrate_page_numbers.py \
  --limit 10 \
  --batch-size 5 \
  --dry-run

# If that looks good, dry-run on all documents
python3 scripts/batch_migrate_page_numbers.py \
  --batch-size 10 \
  --dry-run
```

## Step 5: Run Actual Migration

Once you're confident, run the actual migration:

```bash
# Option 1: Start with a small batch (recommended)
python3 scripts/batch_migrate_page_numbers.py \
  --limit 50 \
  --batch-size 10

# Option 2: Run on all documents
python3 scripts/batch_migrate_page_numbers.py \
  --batch-size 10

# Option 3: Process more in parallel (faster but more load)
python3 scripts/batch_migrate_page_numbers.py \
  --batch-size 20
```

## Step 6: Monitor Progress

The script will show progress as it runs. You can also check progress manually:

```bash
# Check how many documents still have page=0
python3 -c "
from database.vector_store import get_milvus_client

milvus_client = get_milvus_client()
chunks = milvus_client.query(
    collection_name='fltr_documents',
    filter='page == 0 && (chunk_type == \"ocr\" || chunk_type == \"image\" || ocr_confidence > 0.0)',
    output_fields=['document_id']
)

remaining = len(set(c['document_id'] for c in chunks))
print(f'Documents still needing migration: {remaining}')
"
```

## Alternative: Run in Background with Logging

If you want to run it in the background and save output:

```bash
# Run in background and save logs
nohup python3 scripts/batch_migrate_page_numbers.py \
  --batch-size 10 \
  > migration_log_$(date +%Y%m%d_%H%M%S).log 2>&1 &

# Check the process
ps aux | grep batch_migrate

# View the log
tail -f migration_log_*.log

# Or check progress
tail -100 migration_log_*.log
```

## Using Docker (If Applicable)

If you're running in Docker:

```bash
# Find your API container
docker ps | grep api

# Run migration script in container
docker exec -it YOUR_API_CONTAINER_NAME python3 scripts/batch_migrate_page_numbers.py \
  --batch-size 10 \
  --dry-run

# Then run for real
docker exec -it YOUR_API_CONTAINER_NAME python3 scripts/batch_migrate_page_numbers.py \
  --batch-size 10
```

## Troubleshooting

### If migration fails partway through:

The script is designed to handle errors gracefully. Failed documents will be logged. You can:

1. **Check the error log** - The script shows errors at the end
2. **Re-run** - It's safe to re-run, it will skip already-migrated documents
3. **Run on specific failed documents** - Use `--document-id` for specific ones

### If you need to stop the migration:

```bash
# Find the process
ps aux | grep batch_migrate

# Kill it (safely - it will finish current batch)
kill -SIGTERM <PID>

# Or if needed
kill -9 <PID>
```

### Check if a specific document was migrated:

```bash
python3 -c "
from database.vector_store import get_milvus_client

milvus_client = get_milvus_client()
doc_id = 'YOUR_DOCUMENT_ID'

# Check if document has chunks with page > 0
chunks = milvus_client.query(
    collection_name='fltr_documents',
    filter=f'document_id == \"{doc_id}\" && page > 0',
    output_fields=['chunk_id', 'page', 'image_index']
)

print(f'Found {len(chunks)} chunks with page > 0')
for chunk in chunks[:5]:
    print(f'  Chunk {chunk[\"chunk_id\"]}: page={chunk[\"page\"]}, index={chunk[\"image_index\"]}')
"
```

## Recommended Execution Plan

1. **Check scope** - See how many documents are affected
2. **Test on 1 document** - Verify it works
3. **Dry-run on 10 documents** - Check for issues
4. **Run on 50 documents** - Small batch to verify
5. **Run on all documents** - Full migration

## Expected Output

The script will show:
- Progress: `X/Y documents (Z%)`
- Success/failure counts
- Images extracted/uploaded
- Chunks updated
- Time estimates
- Error details (if any)

Example output:
```
📦 Batch 1/10 (10 documents)
   ✅ doc-123: 15 chunks updated
   ✅ doc-456: 8 chunks updated
   ...

   📊 Progress: 10/100 (10.0%)
   ✅ Successful: 10
   ❌ Failed: 0
   ⏱️  Rate: 2.5 docs/sec, ETA: 36.0 minutes
```

## Safety Notes

- ✅ The script is idempotent - safe to run multiple times
- ✅ It handles errors gracefully - one failure won't stop the batch
- ✅ Dry-run mode lets you preview changes
- ✅ Batch processing prevents overwhelming the system
- ⚠️  Monitor the first batch to ensure it's working correctly
- ⚠️  Check logs if you see many failures

