# R2 Webhook Configuration Analysis

## Overview
Your system uses Cloudflare R2 with event notifications that trigger document processing. Checkpoint files are **currently being filtered out** at multiple levels to prevent them from being re-processed as documents.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        R2 BUCKET (fltr-datasets)                │
│                                                                   │
│  Stores: User documents + Extracted assets + Checkpoints        │
│                                                                   │
│  Paths:                                                           │
│  • {dataset_id}/{filename}                ← User documents      │
│  • {dataset_id}/{doc_id}/images/*.png     ← Extracted images   │
│  • {dataset_id}/{doc_id}/tables/*.json    ← Extracted tables   │
│  • {dataset_id}/{doc_id}/checkpoints/*.gz ← Checkpoints        │
└──────────┬──────────────────────────────────────────────────────┘
           │ All PutObject events
           ▼
┌────────────────────────────────────────────────────────────┐
│  R2 Event Notifications Configuration                      │
│  (Cloudflare Dashboard > R2 > fltr-datasets > Settings)    │
│                                                             │
│  ✅ Event Type: "Object Create" (PutObject)               │
│  ✅ Target: Cloudflare Queue (fltr-document-processing)   │
│  ❓ Prefix Filter: (empty - receives ALL uploads)          │
│  ❓ Suffix Filter: (not configured)                        │
└──────────┬────────────────────────────────────────────────┘
           │ Sends R2ObjectCreateNotification event
           ▼
┌────────────────────────────────────────────────────────────┐
│  Cloudflare Worker: dataset-upload-notification-worker    │
│  (cloudflare-workers/dataset-upload-notification-worker)  │
│                                                             │
│  Processes ALL R2 PutObject events and filters:            │
│  1. Non-PutObject actions (DeleteObject, etc.)            │
│  2. Metadata flag: x-fltr-skip-processing = 'true'        │
│  3. Path patterns: /images/, /tables/, /charts/            │
│  4. Checkpoint files: /checkpoints/                        │
└──────────┬────────────────────────────────────────────────┘
           │ Queue to Cloudflare Queue OR fallback webhook
           ▼
┌────────────────────────────────────────────────────────────┐
│  Document Processing Pipeline                              │
│  • Modal: process_document_modal()                         │
│  • FastAPI: /webhooks/r2-upload endpoint                  │
│  • Queue Consumer: Processes messages                      │
└────────────────────────────────────────────────────────────┘
```

---

## Current Filtering Mechanisms

### 1. **Cloudflare Worker Level** (PRIMARY)
**File:** `/Users/jamesjulius/Coding/FLTR/cloudflare-workers/dataset-upload-notification-worker/src/index.ts`

**Filtering order:**
1. **Non-PutObject actions** (lines 57-61)
   - Only processes `action === 'PutObject'`
   - Skips DeleteObject, etc.

2. **Metadata flag** (lines 74-80) [PREFERRED - NEW]
   ```typescript
   const skipProcessing = notification.object.customMetadata?.['x-fltr-skip-processing'];
   if (skipProcessing === 'true') {
     message.ack();
     continue;
   }
   ```

3. **Path-based filtering** (lines 83-90) [FALLBACK - BACKWARD COMPATIBLE]
   ```typescript
   if (notification.object.key.includes('/images/') ||
       notification.object.key.includes('/tables/') ||
       notification.object.key.includes('/charts/')) {
     message.ack();
     continue;
   }
   ```

**Missing:** Checkpoint filtering at this level
- Checkpoint path: `{dataset_id}/{document_id}/checkpoints/{stage}.json.gz`
- Should add: `notification.object.key.includes('/checkpoints/')`

### 2. **FastAPI Webhook Level** (FALLBACK)
**File:** `/Users/jamesjulius/Coding/FLTR/fastapi/routers/webhooks.py`

**Lines 50-55:** Filters extracted assets
```python
if "/images/" in object_key or "/tables/" in object_key or "/charts/" in object_key:
    print(f"⏭️  Skipping asset file (not a document): {object_key}")
    return {"status": "skipped", "reason": "asset_file"}
```

**Missing:** Checkpoint filtering at this level too
- Should add: `"/checkpoints/" in object_key`

### 3. **Queue Consumer Level** (ADDITIONAL LAYER)
**File:** `/Users/jamesjulius/Coding/FLTR/fastapi/services/queue_consumer.py`

Receives messages from Cloudflare Queue and processes them.

---

## How Checkpoints Are Uploaded

**File:** `/Users/jamesjulius/Coding/FLTR/modal/services/checkpoint_manager.py`

```python
def _get_checkpoint_key(self, dataset_id: str, document_id: str, stage: str) -> str:
    """Generate R2 key for a checkpoint"""
    return f"{dataset_id}/{document_id}/checkpoints/{stage}.json.gz"

async def save_checkpoint(self, dataset_id, document_id, stage, data):
    # Upload to R2
    self.s3.put_object(
        Bucket=self.bucket_name,
        Key=checkpoint_key,
        Body=compressed_data,
        Metadata={
            "x-fltr-checkpoint": "true",      # ← CUSTOM METADATA
            "x-fltr-stage": stage,
            ...
        }
    )
```

**Key observation:** Checkpoints already set custom metadata `x-fltr-checkpoint: true` and `x-fltr-skip-processing` flag (via `storage.py` line 94), but...

---

## The Problem

**Why checkpoints might trigger processing:**

1. **R2 Event Notifications Configuration is Missing/Incomplete**
   - The README (line 73-76) shows how to configure via Cloudflare Dashboard
   - But there's NO PREFIX filter configured
   - **Current state:** ALL uploads (including checkpoints) trigger the queue

2. **Worker filters out checkpoints partially**
   - Metadata flag is checked, but checkpoints might not set it
   - Path-based filter doesn't include `/checkpoints/` pattern
   - Checkpoint manager SHOULD set `x-fltr-skip-processing: true`

3. **Fallback webhook also lacks checkpoint filtering**
   - Lines 51-55 in `fastapi/routers/webhooks.py` skip `/images/`, `/tables/`, `/charts/`
   - Missing `/checkpoints/` check

---

## Answers to Your Questions

### Q1: Is the webhook triggered on ALL file uploads to R2?

**Yes, currently.** The R2 event notification is configured WITHOUT a prefix filter, so every PutObject event (including checkpoints) is sent to the Cloudflare Queue.

**Evidence:**
- `/cloudflare-workers/dataset-upload-notification-worker/README.md` (line 75): "Prefix: (leave empty for all uploads)"
- The worker receives `R2ObjectCreateNotification` for ALL uploads
- Filtering happens DOWNSTREAM in the worker/API

### Q2: Can we filter events at the R2 level (e.g., by prefix/path)?

**Yes, absolutely.** This is the BEST place to filter.

**R2 Event Notification supports:**
- ✅ **Prefix filtering** - e.g., only send events for keys starting with `{dataset_id}/` (excluding extracted assets)
- ✅ **Suffix filtering** - e.g., exclude `.json.gz` files (checkpoints)
- ✅ **Event type filtering** - e.g., only PutObject events (already done)

**Configuration location:**
- Cloudflare Dashboard > R2 > fltr-datasets > Settings > Event Notifications
- Or via API: `PUT /accounts/{account_id}/r2/buckets/{bucket_name}/notification-rules`

### Q3: Should we use different buckets for user documents vs checkpoints?

**Not necessary, but could be an option.**

**Pros of separate buckets:**
- Complete isolation - different webhook rules
- Easier to manage costs/quotas
- Cleaner separation of concerns

**Cons of separate buckets:**
- More infrastructure to manage
- Higher costs (might have minimum bucket fees)
- More complex code

**Better solution:** Use prefix filtering (cheaper, simpler)

### Q4: Should we have separate webhook endpoints?

**No, better to filter at R2 level.**

**Current architecture is already optimal:**
1. R2 → Cloudflare Queue (via worker) - PRIMARY path
2. Worker → Webhook fallback - if queue fails
3. FastAPI webhook - catches anything that slips through

**But:** Should add checkpoint pattern to filters everywhere for belt-and-suspenders approach.

---

## Recommendations

### IMMEDIATE FIXES (Do these)

**1. Add checkpoint filtering to Cloudflare Worker**
File: `cloudflare-workers/dataset-upload-notification-worker/src/index.ts`

Add after line 85:
```typescript
// Skip checkpoints
if (notification.object.key.includes('/checkpoints/')) {
    console.log('⏭️  Skipping checkpoint file:', notification.object.key);
    message.ack();
    continue;
}
```

**2. Add checkpoint filtering to FastAPI webhook**
File: `fastapi/routers/webhooks.py`

Line 53, change to:
```python
if ("/images/" in object_key or "/tables/" in object_key or 
    "/charts/" in object_key or "/checkpoints/" in object_key):
    print(f"⏭️  Skipping asset/checkpoint file: {object_key}")
    return {"status": "skipped", "reason": "asset_file"}
```

**3. Ensure checkpoints set skip-processing metadata**
File: `modal/services/checkpoint_manager.py`

Line 83-88, verify/ensure:
```python
Metadata={
    "x-fltr-skip-processing": "true",  # ← ADD THIS
    "x-fltr-checkpoint": "true",
    "x-fltr-stage": stage,
    ...
}
```

### OPTIONAL IMPROVEMENTS (For efficiency)

**4. Configure R2 Event Notification with prefix filter**

In Cloudflare Dashboard:
- Go to R2 > fltr-datasets > Settings > Event Notifications
- Add Notification Rule:
  - Event type: Object Create
  - Prefix: `[dataset_id_prefix]/` (or leave empty for now)
  - Suffix: Do NOT include `*.json.gz` 
  - Target: Cloudflare Queue
  - ⚠️ NOTE: Prefix filtering by dataset ID is tricky (dynamic). Better to filter by suffix.

**Alternative suffix-based approach:**
```
- Event: Object Create
- Suffix: Exclude "*.json.gz" (for checkpoints)
- Target: Queue
```

### LONG-TERM ARCHITECTURE

Consider documenting the path structure in a spec:

**R2 Bucket Structure:**
```
fltr-datasets/
├── {dataset_id}/
│   ├── document.pdf              ← PROCESS (user document)
│   ├── image.png                 ← PROCESS (user document)
│   └── {document_id}/
│       ├── images/               ← SKIP (extracted assets)
│       │   ├── page1.png
│       │   └── page2.png
│       ├── tables/               ← SKIP (extracted assets)
│       │   └── table1.json
│       ├── charts/               ← SKIP (extracted assets)
│       │   └── chart1.json
│       └── checkpoints/          ← SKIP (internal processing)
│           ├── parsed.json.gz
│           ├── chunks.json.gz
│           └── embeddings.json.gz
```

---

## Testing

After applying fixes:

```bash
# 1. Deploy Cloudflare Worker
cd cloudflare-workers/dataset-upload-notification-worker
npm run deploy

# 2. Run FastAPI tests
cd fastapi
python -m pytest tests/test_webhooks_router.py -v

# 3. Manual test: Upload checkpoint to R2
aws s3 cp test_checkpoint.json.gz \
  s3://fltr-datasets/test-dataset/doc-123/checkpoints/ \
  --endpoint-url https://{account}.r2.cloudflarestorage.com

# 4. Verify: Should NOT appear in processing queue
# Check CloudFlare Dashboard > Queues > fltr-document-processing
# Message log should be empty for checkpoint upload
```

---

## Files Affected

### Configuration
- `/Users/jamesjulius/Coding/FLTR/cloudflare-workers/dataset-upload-notification-worker/wrangler.toml`
- `/Users/jamesjulius/Coding/FLTR/fastapi/config.py`

### Code
- `/Users/jamesjulius/Coding/FLTR/cloudflare-workers/dataset-upload-notification-worker/src/index.ts` ← UPDATE
- `/Users/jamesjulius/Coding/FLTR/fastapi/routers/webhooks.py` ← UPDATE
- `/Users/jamesjulius/Coding/FLTR/modal/services/checkpoint_manager.py` ← VERIFY
- `/Users/jamesjulius/Coding/FLTR/modal/services/storage.py` ← VERIFY

### Documentation
- `/Users/jamesjulius/Coding/FLTR/cloudflare-workers/dataset-upload-notification-worker/README.md` ← UPDATE DOCS

---

## Summary

Your R2 webhook setup is **reasonable but incomplete**:

✅ **What's working:**
- Primary path: R2 → Queue → Worker → FastAPI works
- Metadata-based filtering approach is sophisticated
- Multiple layers of defense (worker + webhook)
- Fallback mechanisms in place

❌ **What's missing:**
- Checkpoints not explicitly filtered in worker (path pattern missing)
- Checkpoints not explicitly filtered in FastAPI webhook
- No R2-level prefix/suffix filtering configured
- Missing `/checkpoints/` pattern in several places

🎯 **Best practice solution:**
1. Add `/checkpoints/` filtering everywhere (belt-and-suspenders)
2. Configure R2 event notifications with suffix filter `*.json.gz`
3. Ensure checkpoint uploads set `x-fltr-skip-processing: true` metadata

This is a **low-risk, high-confidence fix** that consolidates your existing approach.
