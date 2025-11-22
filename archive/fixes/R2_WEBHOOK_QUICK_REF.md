# R2 Webhook Configuration - Quick Reference

## Your Questions & Answers

### Q1: Is the webhook triggered on ALL file uploads to R2?
**Answer: YES** - Every PutObject event triggers

```
User document upload     → PutObject event → Queue
Extracted image upload   → PutObject event → Queue
Checkpoint upload        → PutObject event → Queue  ← FILTERED OUT
```

Filtering happens **downstream** in worker/API, not at R2 level.

---

### Q2: Can we filter events at the R2 level?
**Answer: YES** - This is the BEST place to filter

**R2 supports:**
- Prefix filtering: `documents/` (only that prefix)
- Suffix filtering: exclude `*.json.gz` (checkpoints)
- Event type: only PutObject (already done)

**How to configure:**
1. Cloudflare Dashboard
2. R2 → fltr-datasets → Settings
3. Event Notifications → Add Notification
4. Set Suffix: `*.json.gz` to exclude checkpoints

**Result:** Checkpoints never even reach the queue!

---

### Q3: Should we use different buckets?
**Answer: NO** - Single bucket is better

| Aspect | Single Bucket | Multiple Buckets |
|--------|---------------|------------------|
| Cost | Lower | Potentially higher |
| Complexity | Simple | More to manage |
| Filtering | Excellent | Complete isolation |
| **Recommendation** | ✅ YES | Only if needed |

**Use single bucket with R2-level suffix filtering.**

---

### Q4: Should we have separate webhook endpoints?
**Answer: NO** - Current architecture is optimal

```
Current flow:
┌─────────┬──────────────┬──────────┐
│ R2      │ CF Queue     │ Worker   │ → Process
│ (all)   │ (filtered)   │ (filter) │
└─────────┴──────────────┴──────────┘

Better: Add checkpoint filter to existing endpoints
(you already have 3 filter layers!)
```

---

## Current State

### What's Working ✅
- R2 → Queue → Worker → FastAPI flow
- Metadata-based filtering (`x-fltr-skip-processing` flag)
- Path-based filtering (`/images/`, `/tables/`, `/charts/`)
- Multiple defense layers (belt-and-suspenders)

### What's Missing ❌
- `/checkpoints/` pattern not filtered in worker
- `/checkpoints/` pattern not filtered in FastAPI
- No R2-level suffix filter configured
- Checkpoint metadata might not set skip flag

---

## The Fix (5 minutes)

### 1. Cloudflare Worker
**File:** `cloudflare-workers/dataset-upload-notification-worker/src/index.ts`

Add after line 85:
```typescript
// Skip checkpoints
if (notification.object.key.includes('/checkpoints/')) {
    console.log('⏭️  Skipping checkpoint file:', notification.object.key);
    message.ack();
    continue;
}
```

### 2. FastAPI Webhook
**File:** `fastapi/routers/webhooks.py`

Change line 53 from:
```python
if "/images/" in object_key or "/tables/" in object_key or "/charts/" in object_key:
```

To:
```python
if ("/images/" in object_key or "/tables/" in object_key or 
    "/charts/" in object_key or "/checkpoints/" in object_key):
```

### 3. Checkpoint Manager
**File:** `modal/services/checkpoint_manager.py`

Verify metadata (lines 83-88) includes:
```python
Metadata={
    "x-fltr-skip-processing": "true",  # ← CRITICAL
    "x-fltr-checkpoint": "true",
    "x-fltr-stage": stage,
}
```

---

## Verify the Fix

```bash
# 1. Deploy worker
cd cloudflare-workers/dataset-upload-notification-worker
npm run deploy

# 2. Test: Upload a checkpoint
aws s3 cp test.json.gz \
  s3://fltr-datasets/test-id/doc-id/checkpoints/test.json.gz \
  --endpoint-url https://ACCOUNT.r2.cloudflarestorage.com

# 3. Check: No message should appear in queue
# Cloudflare Dashboard > Queues > fltr-document-processing
# (should be empty for checkpoint upload)

# 4. Test: Upload a regular document
aws s3 cp test.pdf \
  s3://fltr-datasets/test-id/test.pdf \
  --endpoint-url https://ACCOUNT.r2.cloudflarestorage.com

# 5. Verify: Should appear in queue
# (should see new message in queue)
```

---

## File Locations

```
Your FLTR project
├── cloudflare-workers/
│   └── dataset-upload-notification-worker/
│       ├── src/index.ts              ← UPDATE (add /checkpoints/ filter)
│       ├── wrangler.toml             ← Review
│       └── README.md                 ← Update docs
├── fastapi/
│   ├── routers/webhooks.py           ← UPDATE (add /checkpoints/ filter)
│   └── services/queue_consumer.py    ← Already filters downstream
└── modal/
    └── services/
        ├── checkpoint_manager.py     ← VERIFY metadata
        └── storage.py                ← Sets metadata flags
```

---

## Understanding the Path Structure

```
R2 Bucket: fltr-datasets

✅ PROCESS THESE:
  {dataset_id}/document.pdf              ← User uploads
  {dataset_id}/image.png                 ← User uploads
  
⏭️ SKIP THESE:
  {dataset_id}/{doc_id}/images/page1.png ← Extracted assets
  {dataset_id}/{doc_id}/tables/table1.json ← Extracted assets
  {dataset_id}/{doc_id}/charts/chart1.json ← Extracted assets
  {dataset_id}/{doc_id}/checkpoints/parsed.json.gz ← INTERNAL
  {dataset_id}/{doc_id}/checkpoints/chunks.json.gz ← INTERNAL
  {dataset_id}/{doc_id}/checkpoints/embeddings.json.gz ← INTERNAL
```

---

## Optional: R2 Event Notification Config

For maximum efficiency (prevents unnecessary queue messages):

**Cloudflare Dashboard:**
1. R2 > fltr-datasets > Settings
2. Event Notifications > Add Notification
3. Configuration:
   - Event type: Object Create ✅
   - Prefix: (leave empty)
   - **Suffix: *.json.gz** ← Blocks checkpoint uploads!
   - Target: Queue (fltr-document-processing) ✅

**Result:** Checkpoints never trigger queue events in the first place!

---

## Summary Table

| Layer | Current | Missing | Fix |
|-------|---------|---------|-----|
| R2 Event Notification | PutObject filter ✅ | Suffix filter ❌ | Add `*.json.gz` |
| Cloudflare Worker | `/images/`, `/tables/`, `/charts/` ✅ | `/checkpoints/` ❌ | Add pattern |
| FastAPI Webhook | `/images/`, `/tables/`, `/charts/` ✅ | `/checkpoints/` ❌ | Add pattern |
| Metadata Flag | Set in storage.py ✅ | Not in checkpoint_mgr ⚠️ | Verify both |

---

## Risk Assessment

**Risk Level: LOW**
- Changes are additive (only add filters, don't remove)
- Filters are explicit and well-tested pattern
- Multiple layers ensure safety
- Easy to verify and rollback

**Confidence: HIGH**
- Problem clearly identified
- Solution follows existing patterns
- Matches architecture principles
- No breaking changes

---

## Key Insight

Your R2 webhook setup is **sophisticated but incomplete**. You have:
- ✅ Metadata-based filtering (good!)
- ✅ Path-based filtering (defensive!)
- ✅ Multiple layers (robust!)
- ❌ But missing `/checkpoints/` everywhere

**The fix is simple:** Add `/checkpoints/` to your existing filters.

**Why it matters:** Prevents wasted queue messages, Modal invocations, and confusion in logs.

