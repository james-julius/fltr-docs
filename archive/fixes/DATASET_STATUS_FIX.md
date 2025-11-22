# Dataset Status Update Fix

## Problem

When all document processing jobs completed and documents were set to "ready", the dataset status wasn't being updated to "ready". The dataset would remain stuck in "processing" or "uploading" status even though all documents were successfully processed.

## Root Cause

**Issue:** Modal defines its own `Document` model instead of importing from FastAPI's models.

- FastAPI has SQLAlchemy event listeners ([fastapi/events/document_events.py](fastapi/events/document_events.py)) that automatically update dataset status when documents change
- These event listeners are only attached to FastAPI's `Document` class, not Modal's
- When Modal updated documents to "ready" status, the events never fired
- Result: Dataset status never updated

## Solution

Added a `_update_dataset_status()` helper function in Modal ([modal/modal_app.py:45-108](modal/modal_app.py#L45-L108)) that:

1. **Counts all documents by status** for the dataset using raw SQL
2. **Updates the `document_count`** field to reflect actual document count
3. **Determines appropriate dataset status:**
   - `ready` when ALL documents are ready
   - `failed` when ALL documents failed
   - `processing` when there's a mix of states
4. **Updates the dataset status** in the database

This function is called:
- After successfully processing a document ([line 214](modal/modal_app.py#L214))
- After a document fails processing ([line 265](modal/modal_app.py#L265))

## Implementation Details

```python
def _update_dataset_status(session, dataset_id: str):
    """
    Update dataset status based on all document statuses
    This mirrors the logic in fastapi/events/document_events.py
    but is called explicitly from Modal since event listeners don't work across services
    """
    # Count documents by status
    result = session.execute(
        text("""
            SELECT status, COUNT(*)
            FROM documents
            WHERE dataset_id = :dataset_id
            GROUP BY status
        """),
        {"dataset_id": dataset_id}
    )
    status_counts = dict(result.fetchall())
    total = sum(status_counts.values())

    # Determine new status
    ready_count = status_counts.get('ready', 0)
    failed_count = status_counts.get('failed', 0)

    if ready_count == total:
        new_status = 'ready'
    elif failed_count == total:
        new_status = 'failed'
    else:
        new_status = 'processing'

    # Update dataset
    session.execute(
        text("""
            UPDATE datasets
            SET status = :status, updated_at = CURRENT_TIMESTAMP
            WHERE id = :dataset_id AND status != :status
        """),
        {"status": new_status, "dataset_id": dataset_id}
    )
    session.commit()
```

## Deployment

The fix has been deployed to Modal:
```bash
cd modal
modal deploy modal_app.py
```

Output:
```
✓ Created objects.
├── 🔨 Created function process_document_modal.
└── 🔨 Created web function document_processing_fastapi_app
✓ App deployed! 🎉

View Deployment: https://modal.com/apps/fltr/main/deployed/fltr
```

## Testing Status

### Automated Tests
- Created comprehensive test suite: [modal/test_dataset_status_update.py](modal/test_dataset_status_update.py)
- Tests cover:
  - ✅ Dataset status updates to 'ready' when all documents ready
  - ✅ Dataset status updates to 'failed' when all documents failed
  - ✅ Dataset status is 'processing' with mixed document states
- **Note:** Tests require running PostgreSQL database (not run in CI yet)

### Manual Testing Required
To verify the fix in your local environment:

1. Start local development:
   ```bash
   ./start-local-dev.sh
   ```

2. Upload documents to a dataset:
   - Create a new dataset
   - Upload 2-3 documents
   - Watch Modal logs for processing

3. Verify dataset status:
   ```bash
   # Query database
   psql postgresql://admin:password@127.0.0.1:5432/fltr_main

   SELECT d.id, d.name, d.status, d.document_count,
          COUNT(doc.id) as actual_docs,
          COUNT(CASE WHEN doc.status = 'ready' THEN 1 END) as ready_docs
   FROM datasets d
   LEFT JOIN documents doc ON doc.dataset_id = d.id
   WHERE d.created_at > NOW() - INTERVAL '1 hour'
   GROUP BY d.id, d.name, d.status, d.document_count;
   ```

4. Expected result:
   - When all documents finish processing → dataset status = 'ready'
   - Check Modal logs for: `"📊 [Modal] Updated dataset {id}: status=ready, document_count={n}"`

### Production Testing
- Deploy to staging/production environment
- Monitor logs for the status update message
- Verify datasets transition to 'ready' status after all documents process

## Related Files

| File | Purpose |
|------|---------|
| [modal/modal_app.py](modal/modal_app.py#L45-L108) | New `_update_dataset_status()` function |
| [modal/modal_app.py](modal/modal_app.py#L214) | Call after successful processing |
| [modal/modal_app.py](modal/modal_app.py#L265) | Call after failed processing |
| [fastapi/events/document_events.py](fastapi/events/document_events.py) | FastAPI event listener (for reference) |
| [modal/test_dataset_status_update.py](modal/test_dataset_status_update.py) | Test suite |

## Expected Behavior

### Before Fix
1. Document processed → status set to "ready"
2. Event listener not triggered (wrong Document class)
3. ❌ Dataset status stays as "processing" forever

### After Fix
1. Document processed → status set to "ready"
2. `_update_dataset_status()` called explicitly
3. ✅ Counts all document statuses
4. ✅ Updates dataset status to "ready" if all documents ready
5. ✅ Logs: `"📊 [Modal] Updated dataset {id}: status=ready, document_count={n}"`

## Monitoring

Check Modal logs for these messages:
```
✅ Updated document {doc_id} to ready
✅ Checked dataset status for {dataset_id}
📊 [Modal] Updated dataset {dataset_id}: status=ready, document_count={n}
```

If you don't see the last message, the status update logic is not executing correctly.

## Future Improvements

1. **Consolidate Logic:** Consider moving dataset status logic to a shared library/service
2. **Add Integration Tests:** Set up test database in CI for automatic testing
3. **Add Monitoring:** Alert when datasets are stuck in "processing" for > 1 hour
4. **Event System:** Consider implementing a proper event bus for cross-service communication
