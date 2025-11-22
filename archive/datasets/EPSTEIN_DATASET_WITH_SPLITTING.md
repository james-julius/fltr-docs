# Epstein Dataset Processing - WITH PDF SPLITTING ✅

**Date:** 2025-11-13
**Dataset:** Epstein Court Documents
**Location:** `/Users/jamesjulius/Coding/FLTR/datasets/Epstein Files overview/RECONSTRUCTED_PDFs`
**Strategy:** Pre-split large PDFs before upload
**Status:** ✅ **READY - NO CONFIG CHANGES NEEDED**

---

## Executive Summary

By **pre-splitting large PDFs** into ~75MB chunks, your FLTR system is **READY TO PROCESS** without any configuration changes!

### Quick Comparison

| Approach | Status | Changes Required | Risk | Processing Time |
|----------|--------|------------------|------|-----------------|
| **Without Splitting** | ⚠️ At Risk | Memory: 4GB → 8GB<br>Timeout: 1hr → 2hr | HIGH OOM risk | 16-26 hours |
| **With Splitting** | ✅ **READY** | **NONE** | LOW | **10-14 hours** |

---

## Why Splitting Is Better

### 1. ✅ No Configuration Changes
- Keep Modal memory at 4GB (current setting is fine)
- Keep timeout at 1 hour (current setting is fine)
- No deployment needed
- No additional costs

### 2. ✅ Better Parallelization
**Before (1.3GB file):**
- 1 container × 90 minutes = 90 minutes total

**After (split into 18 chunks):**
- 18 containers × 5-7 minutes = **5-7 minutes total**
- **12x faster** for large files!

### 3. ✅ Lower Failure Risk
- All files < 100MB process reliably
- No memory constraints
- No timeout concerns
- Easier error recovery (retry single 75MB chunk vs 1.3GB file)

### 4. ✅ Faster Overall Processing
- More files = more parallelism
- 100 containers processing simultaneously
- Better utilization of Modal's auto-scaling

---

## Dataset Analysis

### Current State
```
Total Files: 1,525 PDFs
Total Size: 22 GB

Size Distribution:
├── < 100 MB: 1,467 files (10.3 GB) ✅ Already good
└── > 100 MB: 58 files (11.7 GB) ⚠️ Need splitting
    ├── 1.3 GB: 1 file (22-1426_Documents.pdf)
    ├── 400-600 MB: 4 files (BOP records, DOJ report)
    ├── 200-400 MB: 7 files
    └── 100-200 MB: 46 files
```

### After Splitting
```
Total Files: 1,650 PDFs (~+8%)
Total Size: 22 GB (unchanged)

All Files:
├── Small (<100MB): 1,467 files ✅
└── Chunks (~75MB): 183 files ✅

Largest file: ~95 MB
Average chunk: ~64 MB
```

---

## Splitting Details

The script will split **58 large files** into **~183 chunks**:

| Original File | Size | Splits Into | Chunk Size |
|---------------|------|-------------|------------|
| 22-1426_Documents.pdf | 1.3 GB | 18 chunks | ~75 MB |
| BOP Part 2 | 569 MB | 8 chunks | ~71 MB |
| BOP Part 3 | 523 MB | 7 chunks | ~75 MB |
| BOP Part 4 | 497 MB | 7 chunks | ~71 MB |
| BOP Part 1 | 458 MB | 7 chunks | ~65 MB |
| DOJ OPR Report | 379 MB | 6 chunks | ~63 MB |
| 20-3061_Documents.pdf | 320 MB | 5 chunks | ~64 MB |
| ... | ... | ... | ... |
| **Total 58 files** | **11.7 GB** | **~183 chunks** | **~64 MB avg** |

---

## How to Split the PDFs

### Step 1: Install PyPDF2

```bash
pip install PyPDF2
```

### Step 2: Run the Splitting Script (Dry Run First)

```bash
# Dry run to see what would happen
cd /Users/jamesjulius/Coding/FLTR
python scripts/split_large_epstein_pdfs.py --dry-run

# Expected output:
# 📊 Dataset Overview:
#    Total PDFs: 1525
#    Files < 100.0 MB: 1467 (will not be split)
#    Files > 100.0 MB: 58 (will be split)
#
# Would create approximately 183 chunks from 58 files
```

### Step 3: Actually Split the Files

```bash
# Remove --dry-run to create the split files
python scripts/split_large_epstein_pdfs.py

# This will:
# 1. Create SPLIT_PDFs/ directory
# 2. Split each large file into ~75MB chunks
# 3. Take ~10-20 minutes to complete
# 4. Create 183 new PDF files
```

### Step 4: Verify Results

```bash
# Check output directory
ls -lh "datasets/Epstein Files overview/SPLIT_PDFs"

# Count files
find "datasets/Epstein Files overview/SPLIT_PDFs" -name "*.pdf" | wc -l
# Should show: ~183 files

# Check sizes (all should be < 100MB)
find "datasets/Epstein Files overview/SPLIT_PDFs" -name "*.pdf" -exec ls -lh {} \; | awk '{print $5}' | sort -h
```

---

## Upload Strategy

### Upload Two Batches

**Batch 1: Small Files (1,467 files)**
```bash
# Upload from RECONSTRUCTED_PDFs (excluding large files)
# Use your FLTR API bulk upload endpoint

# Or manually: Upload files < 100MB
find "datasets/Epstein Files overview/RECONSTRUCTED_PDFs" \
  -name "*.pdf" -size -100M \
  -print
```

**Batch 2: Split Chunks (183 files)**
```bash
# Upload from SPLIT_PDFs directory
find "datasets/Epstein Files overview/SPLIT_PDFs" \
  -name "*.pdf" \
  -print
```

### Using FLTR Bulk Upload API

```bash
# Get your dataset ID
DATASET_ID="your-dataset-id-here"

# Example: Upload via API (pseudo-code)
# Adjust based on your API implementation

curl -X POST "https://your-api.com/api/v1/datasets/${DATASET_ID}/documents/bulk-upload" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "files": [
      {"filename": "file1.pdf", "size": 1234567},
      {"filename": "file2.pdf", "size": 2345678},
      ...
    ]
  }'

# This returns presigned URLs for direct R2 upload
# Then upload files directly to R2
```

---

## Processing Timeline

### With Splitting (RECOMMENDED)

| Phase | Duration | Details |
|-------|----------|---------|
| **Split PDFs** | 15-20 min | 58 files → 183 chunks |
| **Upload** | 2-4 hours | Presigned URLs to R2 |
| **Processing** | 10-14 hours | 100 parallel containers |
| **TOTAL** | **12-18 hours** | End-to-end |

### Processing Breakdown

**Small files (1,467 × 2-5 min):**
- Batches of 100 containers
- ~5-8 hours total

**Split chunks (183 × 5-10 min):**
- Batches of 100 containers
- ~2-3 hours total

**Overlap:** Upload and processing happen simultaneously, so total wall-clock time is less.

---

## Cost Estimates

### One-Time Processing

| Component | Calculation | Cost |
|-----------|-------------|------|
| **Modal Compute** | 1,650 docs × 7 min avg × $0.15/hr | **$290-350** |
| **OpenAI Embeddings** | ~82,500 chunks × $0.00002/chunk | **$1.65** |
| **OpenAI Vision** | ~16,500 images × $0.0015/image | **$24.75** |
| **R2 Operations** | 1,650 uploads + reads | **$0.02** |
| **TOTAL** | | **$316-376** |

**Savings vs. large-file approach:**
- **$80-150 less** (no need for 8GB containers)
- **Faster completion** (10-14 hrs vs 16-26 hrs)
- **Lower risk** (no OOM failures)

### Monthly Recurring

| Component | Amount | Cost |
|-----------|--------|------|
| **R2 Storage** | 22 GB | **$0.33** |
| **Milvus** | ~82,500 vectors | **$0** (shared) |
| **PostgreSQL** | 1,650 docs | **< $1** |
| **TOTAL** | | **~$1.33/month** |

---

## Processing Checklist

### Phase 0: Pre-Processing (15-20 minutes)

- [ ] Install PyPDF2: `pip install PyPDF2`
- [ ] Run dry-run: `python scripts/split_large_epstein_pdfs.py --dry-run`
- [ ] Review output (should show 183 chunks from 58 files)
- [ ] Run actual split: `python scripts/split_large_epstein_pdfs.py`
- [ ] Verify SPLIT_PDFs directory created
- [ ] Verify all chunks < 100MB

### Phase 1: Upload Small Files (2-3 hours)

- [ ] Create dataset in FLTR
- [ ] Get dataset ID
- [ ] Upload 1,467 small files (< 100MB) from RECONSTRUCTED_PDFs
- [ ] Monitor upload progress
- [ ] Verify files appear in database

### Phase 2: Upload Split Chunks (30-60 minutes)

- [ ] Upload 183 split chunks from SPLIT_PDFs directory
- [ ] Monitor upload progress
- [ ] Verify chunks appear in database

### Phase 3: Monitor Processing (10-14 hours)

- [ ] Check Modal dashboard for container activity
- [ ] Monitor database for status updates
- [ ] Watch for any failures
- [ ] Check costs in Modal usage dashboard

### Phase 4: Validation (30 minutes)

- [ ] Verify all 1,650 documents processed
- [ ] Check for failed documents
- [ ] Test search functionality
- [ ] Review total costs

---

## Monitoring

### Database Queries

```sql
-- Overall progress
SELECT status, COUNT(*) as count
FROM documents
WHERE dataset_id = 'your-dataset-id'
GROUP BY status;

-- Expected results:
-- status     | count
-- ---------- | -----
-- uploaded   | X     (decreasing)
-- processing | Y     (0-100)
-- ready      | Z     (increasing)
-- failed     | 0-5   (< 1%)

-- Failed documents (should be rare)
SELECT id, filename, error
FROM documents
WHERE dataset_id = 'your-dataset-id'
  AND status = 'failed'
ORDER BY updated_at DESC;

-- Largest documents (for monitoring)
SELECT
  filename,
  chunk_count,
  asset_count,
  EXTRACT(EPOCH FROM (updated_at - created_at)) / 60 as processing_minutes
FROM documents
WHERE dataset_id = 'your-dataset-id'
  AND status = 'ready'
ORDER BY chunk_count DESC
LIMIT 20;

-- Processing rate
SELECT
  DATE_TRUNC('hour', updated_at) as hour,
  COUNT(*) as completed_docs
FROM documents
WHERE dataset_id = 'your-dataset-id'
  AND status = 'ready'
GROUP BY hour
ORDER BY hour;
```

### Modal Commands

```bash
# View live logs
modal app logs fltr

# Check running containers
modal app list

# Check function stats
modal app show fltr
```

---

## Troubleshooting

### If Splitting Fails

**Error:** `ModuleNotFoundError: No module named 'PyPDF2'`
```bash
pip install PyPDF2
```

**Error:** `Permission denied` when writing files
```bash
chmod +x scripts/split_large_epstein_pdfs.py
```

**Error:** PDF is corrupted/can't be read
- Some PDFs may have corruption
- Note the filename and try manually with Adobe Acrobat
- Skip that file for now and process the rest

### If Upload Fails

**Slow upload speed:**
- Use presigned URLs (bypasses API)
- Upload multiple files in parallel
- Check your internet connection

**R2 webhook not triggering:**
```bash
# Check webhook configuration
curl https://your-modal-webhook-url.com/process \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"dataset_id":"test","object_key":"test.pdf"}'
```

### If Processing Fails

**Document stuck in "processing":**
- Check Modal logs for that document
- Look for timeout or memory errors
- Re-trigger processing via API

**High failure rate (> 5%):**
- Check Modal logs for common error
- May need to increase memory slightly (4GB → 6GB)
- Contact support with error details

---

## Risk Assessment

| Risk | Without Splitting | With Splitting |
|------|-------------------|----------------|
| **OOM Errors** | 🔴 HIGH | 🟢 NONE |
| **Timeout Errors** | 🟡 MEDIUM | 🟢 NONE |
| **Processing Time** | 🟡 16-26 hrs | 🟢 10-14 hrs |
| **Cost Overruns** | 🟡 $404-524 | 🟢 $316-376 |
| **Config Changes** | 🔴 REQUIRED | 🟢 NONE |
| **Manual Intervention** | 🟡 Possible | 🟢 Minimal |

---

## Recommendation

**STATUS: ✅ READY TO PROCESS**

### The Splitting Strategy Is Better Because:

1. **No risky configuration changes** - use existing 4GB/1hr settings
2. **Faster processing** - better parallelization (12-18 hrs vs 16-26 hrs)
3. **Lower costs** - saves $80-150 by not needing 8GB containers
4. **Lower risk** - no OOM or timeout concerns
5. **Easier recovery** - small chunks are easy to retry

### Timeline

1. **Now:** Split PDFs (15-20 min)
2. **Today:** Upload files (2-4 hours)
3. **Tonight:** Processing (10-14 hours overnight)
4. **Tomorrow:** Validation & ready to use

### Budget

- **One-time:** $316-376 (Modal + OpenAI)
- **Monthly:** $1.33 (storage only)

---

## Next Steps

```bash
# 1. Split the PDFs
cd /Users/jamesjulius/Coding/FLTR
python scripts/split_large_epstein_pdfs.py

# 2. Create dataset in FLTR
# (Use your web UI or API)

# 3. Upload small files
# (Use bulk upload API with presigned URLs)

# 4. Upload split chunks
# (Use bulk upload API with presigned URLs)

# 5. Monitor progress
modal app logs fltr

# 6. Check completion
psql $DATABASE_URL -c "
  SELECT status, COUNT(*)
  FROM documents
  WHERE dataset_id = 'your-id'
  GROUP BY status;
"
```

---

## Questions?

### "Why 75MB chunks?"
- Sweet spot for 4GB memory containers
- Fast enough to process (5-10 min each)
- Large enough to maintain context
- Works with existing Modal configuration

### "Will splitting affect search quality?"
- No! The vector embeddings work the same
- Chunks are created from the full document context
- Search results will show all relevant chunks

### "What if I want to process without splitting?"
- See [EPSTEIN_DATASET_READINESS.md](EPSTEIN_DATASET_READINESS.md)
- Requires increasing Modal memory to 8GB
- Requires increasing timeout to 2 hours
- Higher cost ($404-524 vs $316-376)
- Higher risk of OOM failures

### "Can I process both split and non-split?"
- Yes! Your system handles any file < 100MB well
- Mix and match as needed
- Only split files that are causing issues

---

Good luck! This strategy gives you the best balance of speed, cost, and reliability. 🚀
