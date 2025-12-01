# Fix Image Loading Failures in Dataset Detail Page

## Problem

Images were failing to load on the dataset detail page (`/datasets/[slug]/documents/[documentId]`) due to:

1. **Presigned URL Expiration**: Presigned URLs expire after 1 hour (`R2_PRESIGNED_URL_EXPIRY = 3600`), causing images to fail if the page is open longer
2. **Presigned URL Generation Failures**: If URL generation fails, `r2_url` is set to `None`, causing images to fail
3. **Not Using R2 Image Proxy Worker**: An R2 image proxy worker exists and is used in chat, but the document detail page wasn't using it

## Solution

Updated FastAPI to use the R2 image proxy worker URL instead of presigned URLs. The worker:
- **Doesn't expire** (no 1-hour limit)
- **More reliable** (handles access control and errors better)
- **Zero egress costs** (already achieved)
- **Already deployed** and working in chat

## Changes Made

### 1. Added Configuration

**File**: `fastapi/config/__init__.py`
- Added `R2_IMAGE_WORKER_URL: str = ""` configuration option

### 2. Created Helper Function

**File**: `fastapi/routers/dataset.py`
- Added `generate_image_url()` function that:
  - Converts `object_key` to worker URL format
  - Extracts filename from object_key: `{dataset_id}/{document_id}/images/{filename}`
  - Returns worker URL if configured, `None` otherwise (for fallback)

### 3. Updated Asset Endpoints

**File**: `fastapi/routers/dataset.py`

**`list_document_assets` endpoint**:
- Now tries worker URL first
- Falls back to presigned URL if worker not configured
- Better error handling and logging

**`get_document_asset` endpoint**:
- Same pattern: worker URL first, presigned URL fallback
- Updated docstring to reflect new behavior

### 4. Updated Model Documentation

**File**: `fastapi/models/document_asset.py`
- Updated `r2_url` comment to reflect it can be worker URL or presigned URL

## Configuration Required

Set the `R2_IMAGE_WORKER_URL` environment variable in FastAPI:

```bash
# Production
R2_IMAGE_WORKER_URL=https://r2-image-proxy.your-subdomain.workers.dev

# Development (if running worker locally)
R2_IMAGE_WORKER_URL=http://localhost:8787
```

If not configured, the system falls back to presigned URLs (existing behavior).

## URL Format

**Object Key Format**: `{dataset_id}/{document_id}/images/{page}_{index}.{ext}`

**Worker URL Format**: `{worker_url}/{dataset_id}/{document_id}/images/{filename}`

Example:
- Object Key: `abc123/doc456/images/0_1.png`
- Worker URL: `https://worker.example.com/abc123/doc456/images/0_1.png`

## Benefits

- ✅ **No expiration**: Images work indefinitely
- ✅ **More reliable**: Worker handles errors better than presigned URLs
- ✅ **Consistent**: Same approach as chat markdown display
- ✅ **Backward compatible**: Falls back to presigned URLs if worker not configured

## Testing

1. Set `R2_IMAGE_WORKER_URL` in FastAPI environment
2. Restart FastAPI server
3. Navigate to dataset detail page
4. Verify images load correctly
5. Verify images still load after 1+ hour (no expiration)

## Files Modified

1. `fastapi/config/__init__.py` - Added `R2_IMAGE_WORKER_URL` config
2. `fastapi/routers/dataset.py` - Added helper function and updated endpoints
3. `fastapi/models/document_asset.py` - Updated documentation

