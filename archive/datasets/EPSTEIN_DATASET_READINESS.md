# Epstein Dataset Processing Readiness Assessment

**Date:** 2025-11-13
**Dataset:** Epstein Court Documents
**Total Files:** 1,525 PDFs
**Total Size:** 22 GB
**Status:** ⚠️ **REQUIRES CONFIGURATION CHANGES**

---

## Executive Summary

Your FLTR system **requires configuration changes** before processing this dataset. The main issue is **memory constraints for large files** (1.3GB+). With adjustments, the system can handle this workload.

### Quick Status

| Component | Status | Action Required |
|-----------|--------|-----------------|
| **Upload Infrastructure** | ✅ Ready | None |
| **Modal Memory** | ⚠️ **INSUFFICIENT** | **Increase to 8GB** |
| **Modal Timeout** | ⚠️ At Risk | **Increase to 2-3 hours** |
| **Processing Capacity** | ✅ Ready | None (100 containers) |
| **Storage (R2)** | ✅ Ready | None |
| **Database** | ⚠️ Monitor | Watch connection pool |
| **Vector Store (Milvus)** | ✅ Ready | None |
| **Cost Budget** | ⚠️ High | Review estimates below |

---

## Critical Issues Found

### 🔴 CRITICAL: Memory Insufficient for Large Files

**Problem:**
- Current Modal memory: **4GB per container** ([modal_app.py:436](modal/modal_app.py#L436))
- Largest file: **1.3 GB** (22-1426_Documents.pdf)
- Multiple files: **400-500 MB** (BOP records, DOJ report)
- Many files: **100-200 MB**

**Why This Fails:**
When processing a 1.3GB PDF:
1. Load PDF into memory: **~1.3 GB**
2. Parse with PyMuPDF: **~2-3 GB** (document structure, text extraction)
3. Image extraction: **+500 MB - 1 GB** (depends on image count)
4. Vision processing buffers: **+200-500 MB**
5. **Total peak memory: 4-6 GB**

The 4GB limit will cause **Out of Memory (OOM) crashes** on large files.

**Solution Required:**
```python
# In modal_app.py line 436, change:
memory=4096,  # OLD: 4GB
# TO:
memory=8192,  # NEW: 8GB (sufficient for 1.5GB+ PDFs)
```

**Cost Impact:** ~$0.02/hour increase per container = **$30-50 extra** for full dataset

---

### 🟡 MEDIUM: Timeout May Be Insufficient

**Problem:**
- Current timeout: **1 hour (3600s)** ([modal_app.py:437](modal/modal_app.py#L437))
- Large file processing estimate:
  - 1.3GB PDF with images: **45-90 minutes**
  - 400-500MB PDFs: **20-45 minutes**

**Why This May Fail:**
Large PDFs with many images can exceed 1 hour:
- Download: 30-120s (1.3GB from R2)
- Parse: 10-30 min (PyMuPDF + image extraction)
- Image processing: 5-20 min (vision API calls)
- Chunking: 2-5 min
- Embeddings: 3-8 min (large docs = many chunks)
- Vector store: 2-5 min

**Solution:**
```python
# In modal_app.py line 437, change:
timeout=3600,  # OLD: 1 hour
# TO:
timeout=7200,  # NEW: 2 hours (safe buffer)
```

**Cost Impact:** No additional cost (only matters if timeout is hit)

---

## Dataset Analysis

### File Size Distribution

| Size Range | Count | Examples |
|------------|-------|----------|
| **> 1 GB** | 1 | 22-1426 (1.3GB) |
| **400-600 MB** | 4 | BOP Parts 1-4, DOJ Report |
| **200-400 MB** | 3 | 20-3061, Epstein Part 20, 24 |
| **100-200 MB** | 8 | TECS Records, Trial Day 14, etc. |
| **50-100 MB** | ~30 | Case compilations |
| **10-50 MB** | ~100 | Standard court docs |
| **< 10 MB** | ~1,379 | Individual filings |

### Processing Time Estimates

**Assumptions:**
- 100 concurrent containers
- Small files (< 10MB): 2-5 min each
- Medium files (10-50MB): 5-15 min each
- Large files (50-100MB): 15-30 min each
- Very large files (100MB+): 30-90 min each
- Giant files (1GB+): 60-120 min each

**Timeline:**

| Phase | Duration | Notes |
|-------|----------|-------|
| **Upload** | 4-8 hours | 22GB @ 1-2MB/s presigned upload |
| **Processing** | 12-18 hours | 100 parallel containers |
| **Total** | **16-26 hours** | Upload + processing overlap |

**Breakdown:**
- 1,379 small files: 2-5 min each = **2-7 hours** (batches of 100)
- 100 medium files: 5-15 min each = **5-15 hours** (batches of 100)
- 30 large files: 15-30 min each = **5-9 hours** (batches of 30)
- 16 very large files: 30-90 min each = **8-24 hours** (batches of 16)
- 1 giant file: 60-120 min = **1-2 hours**

With 100 containers running in parallel, the **longest files determine total time**.

---

## Cost Estimates

### One-Time Processing Costs

| Component | Calculation | Cost |
|-----------|-------------|------|
| **Modal Compute** | 1,525 docs × 10 min avg × $0.15/hr | **$380-500** |
| **OpenAI Embeddings** | ~76,250 chunks × $0.00002/chunk | **$1.50** |
| **OpenAI Vision** | ~15,000 images × $0.0015/image | **$22.50** |
| **R2 Class A Ops** | 1,525 uploads × $0.0036/1000 | **$0.01** |
| **R2 Class B Ops** | ~10,000 reads × $0.0005/1000 | **$0.01** |
| **TOTAL ONE-TIME** | | **$404-524** |

### Monthly Recurring Costs

| Component | Calculation | Cost |
|-----------|-------------|------|
| **R2 Storage** | 22 GB × $0.015/GB | **$0.33** |
| **Milvus Vectors** | ~76,250 vectors × 1536 dims | **$0** (shared collection) |
| **PostgreSQL** | 1,525 rows + metadata | **< $1** |
| **TOTAL RECURRING** | | **~$1.33/month** |

**Note:** Modal compute dominates cost due to large file sizes and vision processing.

---

## Required Changes

### 1. Increase Modal Memory (CRITICAL)

**File:** [modal/modal_app.py:436](modal/modal_app.py#L436)

```python
@app.function(
    image=image,
    cpu=8.0,
    memory=8192,  # CHANGE: 4096 -> 8192 (8GB for large PDFs)
    timeout=7200,  # CHANGE: 3600 -> 7200 (2 hours for large files)
    volumes={"/cache": volume},
    # ... rest of config
)
async def process_document_modal(...):
```

### 2. Update Vision Worker Memory (RECOMMENDED)

**File:** [modal/modal_app.py:189](modal/modal_app.py#L189)

```python
@app.function(
    image=image,
    cpu=2.0,
    memory=4096,  # CHANGE: 2048 -> 4096 (for large images from 1GB PDFs)
    timeout=900,   # CHANGE: 600 -> 900 (15 min for large batches)
    # ... rest of config
)
async def process_image_batch_worker(...):
```

### 3. Deploy Updated Configuration

```bash
cd modal
modal deploy modal_app.py
```

This will update the serverless functions with new memory/timeout settings.

---

## Processing Strategy

### Recommended Approach: Phased Processing

**Phase 1: Test Small Batch (1-2 hours)**
- Upload 10 small files (< 10MB)
- Verify processing completes successfully
- Check costs and timing

**Phase 2: Test Large Files (2-4 hours)**
- Upload 3 large files:
  - 1 × 100MB file
  - 1 × 400MB file
  - 1 × 1.3GB file (22-1426)
- Monitor memory usage in Modal dashboard
- Verify no OOM errors

**Phase 3: Full Upload (18-26 hours)**
- Upload all 1,525 files via presigned URLs
- Processing starts automatically via R2 webhooks
- Monitor progress in Modal dashboard

### Upload Strategy

The system uses **presigned URLs** which bypass API bottlenecks:

```bash
# You can upload multiple files in parallel
# The FastAPI bulk upload endpoint provides presigned URLs for direct R2 upload

# Example: Upload 100 files at once
curl -X POST "https://your-api.com/api/v1/datasets/{dataset_id}/documents/bulk-upload" \
  -H "Content-Type: application/json" \
  -d '{
    "files": [
      {"filename": "doc1.pdf", "size": 1234567},
      {"filename": "doc2.pdf", "size": 2345678},
      ...
    ]
  }'

# Returns presigned URLs for direct upload to R2
# Then upload files directly to R2 using those URLs
```

---

## Monitoring Checklist

### During Processing

Monitor these metrics:

**Modal Dashboard:**
- [ ] Container memory usage (should stay < 80%)
- [ ] Function duration (should stay < timeout)
- [ ] Error rate (should be < 1%)
- [ ] Queue depth (monitor if > 100 pending)

**Database Queries:**
```sql
-- Check processing status
SELECT status, COUNT(*)
FROM documents
WHERE dataset_id = 'your-dataset-id'
GROUP BY status;

-- Find failed documents
SELECT id, filename, error
FROM documents
WHERE dataset_id = 'your-dataset-id'
  AND status = 'failed';

-- Check largest documents
SELECT filename, chunk_count, asset_count
FROM documents
WHERE dataset_id = 'your-dataset-id'
ORDER BY chunk_count DESC
LIMIT 20;
```

**R2 Storage:**
```bash
# Check upload progress
modal run -m modal.storage list_objects --prefix "your-dataset-id/"

# Or use AWS CLI
aws s3 ls s3://your-bucket/your-dataset-id/ --recursive --human-readable --summarize
```

---

## Failure Recovery

### If Large Files Fail

**Symptoms:**
- Status: `failed`
- Error: "Out of Memory" or "Timeout exceeded"

**Recovery:**
1. Increase memory to 12GB or 16GB for retry
2. Re-trigger processing:
   ```bash
   # Via API
   curl -X POST "https://your-api.com/api/v1/documents/{doc-id}/reprocess"

   # Or directly via Modal
   modal run modal_app.py::process_document_modal \
     --dataset-id "xxx" \
     --object-key "xxx/filename.pdf" \
     --task-id "xxx"
   ```

### Checkpoint Resumption

The system has **automatic checkpoint resumption**:
- If a document fails partway through (e.g., after parsing but before embedding)
- Re-processing will resume from the last checkpoint
- Saves time and API costs

**Checkpoints saved:**
- After parse (parsed document JSON)
- After chunk (chunks JSON)
- After embed (embeddings array)

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| **OOM on large files** | 🔴 HIGH | 🔴 CRITICAL | Increase memory to 8GB |
| **Timeout on 1.3GB file** | 🟡 MEDIUM | 🟡 HIGH | Increase timeout to 2 hours |
| **High API costs** | 🟢 LOW | 🟡 MEDIUM | Costs estimated, acceptable |
| **Upload time** | 🟢 LOW | 🟢 LOW | Presigned URLs handle well |
| **Database locks** | 🟢 LOW | 🟢 LOW | Connection pool sufficient |

---

## Final Checklist

Before processing the full dataset:

- [ ] **Increase Modal memory to 8GB** (modal_app.py:436)
- [ ] **Increase Modal timeout to 2 hours** (modal_app.py:437)
- [ ] **Deploy updated Modal config** (`modal deploy`)
- [ ] **Test with 10 small files** (Phase 1)
- [ ] **Test with 3 large files including 1.3GB** (Phase 2)
- [ ] **Set up monitoring dashboard** (Modal + database queries)
- [ ] **Allocate $500 budget** for processing costs
- [ ] **Schedule 24-hour processing window** (16-26 hours estimated)

---

## Recommendation

**Status: CONDITIONAL GO**

Your system **can process this dataset**, but **requires configuration changes first**:

1. **Critical (Must Do):**
   - Increase memory to 8GB
   - Increase timeout to 2 hours
   - Deploy changes

2. **Strongly Recommended:**
   - Test with large files first (Phase 2)
   - Set up monitoring

3. **Timeline:**
   - Config changes: 15 minutes
   - Testing: 2-4 hours
   - Full processing: 18-26 hours
   - **Total: 1-2 days**

4. **Budget:**
   - One-time: **$404-524**
   - Monthly: **$1.33/month**

**Do NOT proceed** without increasing memory - large files will fail with OOM errors.

---

## Questions?

If you encounter issues:

1. Check Modal logs: `modal app logs fltr`
2. Check database status: SQL queries above
3. Check failed documents: Look for error messages
4. Recovery: Use checkpoint resumption to retry

**Next Steps:**
1. Make the config changes above
2. Deploy: `cd modal && modal deploy modal_app.py`
3. Test Phase 1: 10 small files
4. Test Phase 2: 3 large files (including 1.3GB)
5. If tests pass → Full upload

Good luck! 🚀
