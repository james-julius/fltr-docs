# Bulk Upload Setup Guide

**Status:** ⚠️ **REQUIRES API KEY CONFIGURATION**

Your system is almost ready! You just need to generate an API key to use the bulk upload script.

---

## Quick Setup (5 minutes)

### Step 1: Generate API Key

```bash
cd fastapi
python generate_api_key.py
```

**Output:**
```
🔑 Generated 1 API key(s):
============================================================
1. fltr_1a2b3c4d5e6f7g8h9i0j...
============================================================

📝 To use these keys:
1. Add to your .env file:
   ADMIN_API_KEY=fltr_1a2b3c4d5e6f7g8h9i0j...
```

### Step 2: Add Key to .env File

```bash
# Add this line to fastapi/.env
echo "ADMIN_API_KEY=fltr_YOUR_KEY_HERE" >> fastapi/.env
```

### Step 3: Restart FastAPI Server

```bash
# If running via start-local-dev.sh
# Press Ctrl+C and restart:
./start-local-dev.sh

# Or if running manually:
pkill -f "uvicorn main:app"
cd fastapi && uvicorn main:app --reload
```

### Step 4: Install Upload Script Dependencies

```bash
pip install httpx tqdm
```

### Step 5: Test the Setup

```bash
# Export the API key
export FLTR_API_KEY="fltr_YOUR_KEY_HERE"

# Test with dry-run
python scripts/bulk_upload.py \
  --dataset-id "test-123" \
  --source-dir "datasets/Epstein Files overview/RECONSTRUCTED_PDFs" \
  --dry-run
```

**Expected Output:**
```
📊 Dataset Overview:
   Total PDFs: 1525
   Found 1525 documents

[DRY RUN] No files will be uploaded
```

---

## Configuration Checklist

- [x] FastAPI server running ✅
- [x] Database configured ✅
- [x] R2 storage configured ✅
- [x] Modal webhook configured ✅
- [ ] **API key generated** ⚠️ **YOU NEED THIS**
- [ ] **API key in .env** ⚠️ **YOU NEED THIS**
- [ ] **Server restarted** ⚠️ **AFTER ADDING KEY**
- [ ] httpx installed
- [ ] tqdm installed

---

## Step-by-Step Upload Process

### Before You Start

1. **Split large PDFs** (if using split strategy):
   ```bash
   pip install PyPDF2
   python scripts/split_large_epstein_pdfs.py
   ```

2. **Create a dataset** (if you don't have one):
   ```bash
   # Via UI: http://localhost:3000/datasets/new
   # Or via API:
   curl -X POST "http://localhost:8000/api/v1/datasets" \
     -H "X-API-Key: $FLTR_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{
       "name": "Epstein Court Documents",
       "description": "1,525 PDFs from court cases"
     }'

   # Save the returned dataset ID!
   ```

3. **Note your dataset ID** - you'll need it for upload

---

## Upload Commands

### Option 1: Upload Small Files Only

```bash
# Set your API key
export FLTR_API_KEY="fltr_YOUR_KEY_HERE"

# Set your dataset ID
DATASET_ID="your-dataset-uuid-here"

# Upload files < 100MB
python scripts/bulk_upload.py \
  --dataset-id "$DATASET_ID" \
  --source-dir "datasets/Epstein Files overview/RECONSTRUCTED_PDFs" \
  --batch-size 100 \
  --concurrency 50
```

### Option 2: Upload Split Chunks (Recommended)

```bash
# First, split the large files
python scripts/split_large_epstein_pdfs.py

# Then upload split chunks
python scripts/bulk_upload.py \
  --dataset-id "$DATASET_ID" \
  --source-dir "datasets/Epstein Files overview/SPLIT_PDFs" \
  --batch-size 100 \
  --concurrency 50
```

### Option 3: Upload Everything (Mixed Sizes)

```bash
# Upload both small files and splits
# Upload small files first
python scripts/bulk_upload.py \
  --dataset-id "$DATASET_ID" \
  --source-dir "datasets/Epstein Files overview/RECONSTRUCTED_PDFs" \
  --batch-size 100 \
  --concurrency 50

# Then upload splits
python scripts/bulk_upload.py \
  --dataset-id "$DATASET_ID" \
  --source-dir "datasets/Epstein Files overview/SPLIT_PDFs" \
  --batch-size 100 \
  --concurrency 50
```

---

## Upload Script Options

```bash
python scripts/bulk_upload.py \
  --dataset-id <UUID>               # Required: Your dataset ID
  --source-dir <PATH>               # Required: Directory with PDFs
  --api-url <URL>                   # Optional: API URL (default: http://localhost:8000)
  --api-key <KEY>                   # Optional: Or use $FLTR_API_KEY env var
  --batch-size <N>                  # Optional: Files per batch (default: 100)
  --concurrency <N>                 # Optional: Parallel uploads (default: 50)
  --dry-run                         # Optional: Test without uploading
  --verbose                         # Optional: Debug logging
```

### Environment Variables

```bash
export FLTR_API_URL="http://localhost:8000"   # Or your production URL
export FLTR_API_KEY="fltr_YOUR_KEY_HERE"      # Your API key
```

---

## What Happens During Upload

### 1. File Discovery
- Script scans source directory for PDFs
- Calculates total size and file count
- Shows estimated credit cost (10 credits per file)

### 2. Batch Processing
- Requests presigned URLs from FastAPI (100 files at a time)
- Uploads files directly to R2 in parallel (50 concurrent)
- Skips duplicates automatically

### 3. Automatic Processing
- R2 triggers webhook to Modal after each upload
- Modal processes documents automatically
- Vector embeddings stored in Milvus
- Status updates in PostgreSQL

### 4. Progress Tracking
```
Uploading: 100%|████████████████| 1525/1525 [00:45<00:00, 33.89files/s]
```

### 5. Completion Summary
```
================================================================================
Upload Complete!
================================================================================
Total Files: 1525
Uploaded: 1523
Failed: 2
Skipped: 0
Total Size: 22.00 GB
```

---

## Monitoring Upload Progress

### Check Upload Status

```bash
# Watch log file
tail -f bulk_upload_YYYYMMDD_HHMMSS.log
```

### Check Processing Status (Database)

```bash
psql $DATABASE_URL -c "
  SELECT status, COUNT(*)
  FROM documents
  WHERE dataset_id = 'your-dataset-id'
  GROUP BY status;
"
```

**Expected Output:**
```
  status    | count
------------+-------
 uploaded   |   100  (queued for processing)
 processing |    50  (currently processing)
 ready      |  1375  (completed)
 failed     |     0  (should be 0 or low)
```

### Monitor Modal Processing

```bash
# Watch Modal logs
modal app logs fltr

# Check Modal dashboard
# https://modal.com/apps
```

---

## Troubleshooting

### "ERROR: API key not provided"

```bash
# Make sure you exported the key
export FLTR_API_KEY="fltr_YOUR_KEY_HERE"

# Or pass it directly
python scripts/bulk_upload.py \
  --api-key "fltr_YOUR_KEY_HERE" \
  ...
```

### "Authentication required"

Your API key is not in the .env file:

```bash
# Add it to fastapi/.env
echo "ADMIN_API_KEY=fltr_YOUR_KEY_HERE" >> fastapi/.env

# Restart server
pkill -f uvicorn
cd fastapi && uvicorn main:app --reload
```

### "403 Forbidden" or "401 Unauthorized"

Check that:
1. API key is correct (no typos)
2. Server was restarted after adding key
3. Key starts with `fltr_`

### "Connection refused"

FastAPI server not running:

```bash
# Start it
cd fastapi && uvicorn main:app --reload

# Or use the dev script
./start-local-dev.sh
```

### "Module not found: httpx"

Install dependencies:

```bash
pip install httpx tqdm
```

### Slow Upload Speed

- Reduce concurrency: `--concurrency 25`
- Reduce batch size: `--batch-size 50`
- Check your internet upload speed

### High Failure Rate

Check logs for specific errors:
```bash
grep "Failed to upload" bulk_upload_*.log
```

Common causes:
- Network timeout (increase timeout in script)
- R2 rate limiting (reduce concurrency)
- Corrupted PDF files

---

## Cost Calculation

### Credits

```
Files: 1,525 (or 1,650 with splits)
Cost per file: 10 credits
Total: 15,250-16,500 credits
```

### Modal Compute

```
Processing time: 4-6 hours
Modal cost: ~$0.15/hour per container
100 containers × 6 hours × $0.15 = $90
```

### OpenAI

```
Embeddings: ~82,500 chunks × $0.00002 = $1.65
Vision: ~16,500 images × $0.0015 = $24.75
```

### **Total: ~$116 + credits**

---

## Quick Reference

### Generate API Key
```bash
cd fastapi && python generate_api_key.py
```

### Add to .env
```bash
echo "ADMIN_API_KEY=fltr_YOUR_KEY" >> fastapi/.env
```

### Upload (Small Files)
```bash
export FLTR_API_KEY="fltr_YOUR_KEY"
python scripts/bulk_upload.py \
  --dataset-id "your-uuid" \
  --source-dir "datasets/Epstein Files overview/RECONSTRUCTED_PDFs"
```

### Upload (Split Files)
```bash
python scripts/split_large_epstein_pdfs.py
python scripts/bulk_upload.py \
  --dataset-id "your-uuid" \
  --source-dir "datasets/Epstein Files overview/SPLIT_PDFs"
```

### Monitor
```bash
# Logs
tail -f bulk_upload_*.log

# Database
psql $DATABASE_URL -c "SELECT status, COUNT(*) FROM documents WHERE dataset_id='your-id' GROUP BY status;"

# Modal
modal app logs fltr
```

---

## Next Steps

1. ✅ Generate API key
2. ✅ Add to .env
3. ✅ Restart server
4. ✅ Install dependencies
5. ✅ (Optional) Split large PDFs
6. ✅ Create dataset
7. ✅ Run upload
8. ✅ Monitor progress
9. ✅ Verify completion

**Ready to go!** 🚀

---

## Need Help?

Check the logs:
- Upload logs: `bulk_upload_*.log`
- FastAPI logs: Console where uvicorn is running
- Modal logs: `modal app logs fltr`

Common issues:
- [BULK_UPLOAD_SETUP.md](BULK_UPLOAD_SETUP.md) - This file
- [EPSTEIN_DATASET_WITH_SPLITTING.md](EPSTEIN_DATASET_WITH_SPLITTING.md) - Full guide
- [scripts/README_SPLITTING.md](scripts/README_SPLITTING.md) - Split strategy
