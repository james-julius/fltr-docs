# Modal Retry Fix

## Problem
When clicking the retry button for failed documents, the system was falling back to Celery instead of using Modal, resulting in errors:

```
Error: Failed to parse document: Docling is required for .pdf files.
Please install: pip install docling
```

## Root Cause
The `retry_document_processing` method in `DatasetService` was:
1. A **sync** function trying to call `asyncio.run()`
2. In FastAPI's async context, you can't use `asyncio.run()` (event loop already running)
3. This caused an exception, triggering fallback to Celery
4. Celery workers don't have Docling installed (only Modal does)

## Solution
Made the retry flow fully async:

### 1. Service Layer (`services/dataset_service.py`)
```python
# Before:
def retry_document_processing(self, ...):
    result = asyncio.run(modal_client.process_document(...))  # ❌ Fails

# After:
async def retry_document_processing(self, ...):
    result = await modal_client.process_document(...)  # ✅ Works
```

### 2. Router Layer (`routers/dataset.py`)
```python
# Before:
async def retry_document_processing(...):
    return service.retry_document_processing(...)  # ❌ Missing await

# After:
async def retry_document_processing(...):
    return await service.retry_document_processing(...)  # ✅ Proper await
```

## Files Changed
- ✅ `fastapi/services/dataset_service.py` - Made `retry_document_processing` async
- ✅ `fastapi/routers/dataset.py` - Added `await` for async call

## Testing
After deploying, retry button should:
1. ✅ Log: `🚀 [Retry] Using Modal for document {id}`
2. ✅ Queue to Modal successfully
3. ✅ Process with Docling (no missing dependency errors)

If Modal fails for any reason:
1. ⚠️ Log: `❌ [Modal] Failed to retry: {error}`
2. 🔄 Log: `🔄 [Fallback] Switching to Celery`
3. ✅ Falls back gracefully

## Related Issues Fixed
This was part of the larger Modal integration work:
- ✅ Queue consumer routes to Modal ([queue_consumer.py](services/queue_consumer.py))
- ✅ Retry button routes to Modal (this fix)
- ✅ Webhook routes to Modal ([webhooks.py](routers/webhooks.py))

All processing paths now properly use Modal when `USE_MODAL_FOR_PROCESSING=true` is set.
