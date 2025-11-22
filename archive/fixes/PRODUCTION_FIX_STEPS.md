# Production Fix: MCP Query 500 Error

## Problem

The MCP query endpoint is returning 500 Internal Server Error because:
1. The `document_count` field on datasets wasn't being automatically updated
2. Datasets with processed documents have `document_count = 0` in the database
3. The MCP endpoint checks `document_count > 0` before allowing queries

## Quick Diagnosis

Check if this is the issue by running this SQL query in production:

```sql
-- Check datasets with mismatched document counts
SELECT
    d.id,
    d.name,
    d.document_count as stored_count,
    (SELECT COUNT(*) FROM documents WHERE dataset_id = d.id) as actual_count
FROM datasets d
WHERE d.document_count != (SELECT COUNT(*) FROM documents WHERE dataset_id = d.id);
```

If you see rows with `stored_count = 0` but `actual_count > 0`, that's the problem.

## Solution

### Option 1: Run Migration Script (Recommended)

```bash
# SSH into production server
cd /path/to/fastapi

# Run the migration to fix all datasets
python migrations/fix_document_counts.py
```

This will:
- Check all datasets
- Update `document_count` to match actual document count
- Show what was fixed

### Option 2: Quick SQL Fix

If you can't run the Python script, run this SQL directly:

```sql
-- Update document_count for all datasets
UPDATE datasets d
SET document_count = (
    SELECT COUNT(*)
    FROM documents
    WHERE dataset_id = d.id
),
updated_at = CURRENT_TIMESTAMP;
```

### Option 3: Trigger Event Listener (Slowest)

Just touch any document in the affected dataset:

```sql
-- Find a document in the broken dataset
SELECT id FROM documents
WHERE dataset_id = '7cf258fb-02c8-46e3-ad64-7b0983659640'
LIMIT 1;

-- Update it to trigger the event listener
UPDATE documents
SET updated_at = CURRENT_TIMESTAMP
WHERE id = '<document_id_from_above>';
```

The event listener will automatically fix the dataset's `document_count`.

## What Was Fixed in Code

### 1. Event Listener Now Updates `document_count`
**File:** `fastapi/events/document_events.py`
```python
# Now automatically updates document_count on every document change
connection.execute(
    text("UPDATE datasets SET document_count = :count ..."),
    {"count": total, "dataset_id": str(dataset_id)}
)
```

### 2. Fixed Batch Query to Use Shared Collection
**File:** `fastapi/routers/mcp.py`
- Changed from `dataset.collection_name` to `"fltr_documents"`
- Added proper filtering by `dataset_id`
- Fixed metadata structure

### 3. Added Better Error Logging
Now logs:
- Query being searched
- Dataset document_count
- Number of results found
- Full traceback on errors

## Verify the Fix

After running the migration, test the query again:

```bash
curl "https://your-api.com/api/v1/mcp/query/7cf258fb-02c8-46e3-ad64-7b0983659640/search?query=dataset%20overview&limit=5"
```

You should see:
- Logs showing the search being performed
- Results returned (not 500 error)

## Check Logs

Look for these log messages:
```
🔍 [MCP] Searching dataset {id} with query: {query}...
📊 [MCP] Dataset document_count: {count}
✅ [MCP] Found {n} results
```

If you see:
```
❌ [MCP] Query failed: {error}
❌ [MCP] Traceback: {traceback}
```

That gives you the detailed error to debug further.

## Prevent Future Issues

The event listener is now running, so `document_count` will stay in sync automatically. No manual updates needed going forward!

## Rollback Plan

If something goes wrong:

1. **Revert the code changes** (deploy previous version)
2. **Document counts remain correct** (the migration is safe and idempotent)
3. **Re-run migration** if needed: `python migrations/fix_document_counts.py`









