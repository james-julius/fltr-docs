# Production Bulk Upload Guide (api.tryfltr.com)

**Server:** https://api.tryfltr.com
**Environment:** DigitalOcean + Modal Production
**Status:** ✅ Ready to use

---

## Quick Start for Production

### Step 1: Get Your Production API Key

You have two options:

#### Option A: Generate on Production Server (SSH)

```bash
# SSH into your DigitalOcean server
ssh user@your-server-ip

# Navigate to FastAPI directory
cd /app  # or wherever FastAPI is deployed

# Generate API key
python -c "from auth.api_key import generate_api_key; print(generate_api_key())"

# Copy the output key (starts with fltr_)
```

#### Option B: Generate Locally and Add to Production

```bash
# Generate locally
cd fastapi
python generate_api_key.py

# Output: fltr_1a2b3c4d5e6f7g8h9i0j...

# Then add to production via DigitalOcean:
# 1. Go to DigitalOcean App Platform
# 2. Your App → Settings → Environment Variables
# 3. Add/Edit: ADMIN_API_KEY=fltr_YOUR_KEY_HERE
# 4. Click Save (will trigger redeploy)
```

### Step 2: Configure Upload Script for Production

```bash
# Set production API URL and key
export FLTR_API_URL="https://api.tryfltr.com"
export FLTR_API_KEY="fltr_YOUR_PRODUCTION_KEY_HERE"

# Test connection
curl -H "X-API-Key: $FLTR_API_KEY" \
  "$FLTR_API_URL/api/v1/datasets" \
  | jq .
```

### Step 3: Upload to Production

```bash
# Same commands as local, just with production env vars set

# Option 1: Upload split files (RECOMMENDED)
python scripts/split_large_epstein_pdfs.py

python scripts/bulk_upload.py \
  --dataset-id "YOUR_DATASET_ID" \
  --source-dir "datasets/Epstein Files overview/SPLIT_PDFs" \
  --api-url "https://api.tryfltr.com" \
  --api-key "$FLTR_API_KEY"

# Option 2: Upload all files
python scripts/bulk_upload.py \
  --dataset-id "YOUR_DATASET_ID" \
  --source-dir "datasets/Epstein Files overview/RECONSTRUCTED_PDFs" \
  --api-url "https://api.tryfltr.com" \
  --api-key "$FLTR_API_KEY"
```

---

## Production vs Local Differences

| Aspect | Local (localhost:8000) | Production (api.tryfltr.com) |
|--------|------------------------|------------------------------|
| **API URL** | http://localhost:8000 | https://api.tryfltr.com |
| **API Key** | Generate in fastapi/ | Generate via SSH or DigitalOcean |
| **Processing** | Local Modal dev | Production Modal |
| **Storage** | Local R2/Dev bucket | Production R2 bucket |
| **Costs** | Modal dev credits | **Real $ charges** ⚠️ |
| **Database** | Local PostgreSQL | Production PostgreSQL |
| **Milvus** | Local/Dev cluster | Production Zilliz cluster |

---

## Cost Considerations (IMPORTANT)

### Production Costs for 1,525 Files

```
Modal Processing: $290-350
OpenAI Embeddings: $1.65
OpenAI Vision: $24.75
R2 Storage: $0.33/month
Milvus (Zilliz): Existing cost
-------------------
TOTAL: ~$316-376 one-time
```

**These are REAL charges to your accounts!**

Make sure you:
- [ ] Have budget approved
- [ ] Verify Modal production environment is configured
- [ ] Check current credit balance
- [ ] Have OpenAI API credits available
- [ ] Understand monthly recurring costs ($1-2/month storage)

---

## Production Setup Checklist

### 1. Verify Production Environment

```bash
# Check production API is responding
curl https://api.tryfltr.com/api/v1/health
# Should return: {"status": "ok"} or auth required message

# Check Modal production is deployed
modal app list | grep fltr
# Should show: fltr (deployed)

# Check Modal webhook
echo $MODAL_WEBHOOK_URL
# Should be: https://fltr--fltr-document-processing-fastapi-app.modal.run/process
```

### 2. Generate/Configure API Key

Choose one method:

**Method 1: Via DigitalOcean Dashboard**
1. Go to DigitalOcean App Platform
2. Select your app
3. Settings → Environment Variables
4. Add: `ADMIN_API_KEY=fltr_YOUR_KEY`
5. Save (triggers redeploy ~3-5 min)

**Method 2: Via SSH**
```bash
# SSH to server
ssh your-user@your-server

# Generate key
python -c "from auth.api_key import generate_api_key; print(generate_api_key())"

# Add to environment (method depends on your setup)
# For docker: Update docker-compose env
# For systemd: Update .env file
# Then restart FastAPI
```

### 3. Create Production Dataset

```bash
# Via API
curl -X POST "https://api.tryfltr.com/api/v1/datasets" \
  -H "X-API-Key: $FLTR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Epstein Court Documents - Production",
    "description": "1,525 PDFs from court cases"
  }' | jq .

# Save the returned "id" field
export DATASET_ID="the-uuid-from-response"
```

Or create via web UI at https://tryfltr.com

---

## Upload Commands (Production)

### Setup Environment

Create a `.env.production` file:

```bash
# .env.production
FLTR_API_URL=https://api.tryfltr.com
FLTR_API_KEY=fltr_YOUR_PRODUCTION_KEY_HERE
FLTR_DATASET_ID=your-dataset-uuid-here
```

Then source it:

```bash
source .env.production
```

### Upload Commands

**Dry Run First (Recommended):**

```bash
python scripts/bulk_upload.py \
  --dataset-id "$FLTR_DATASET_ID" \
  --source-dir "datasets/Epstein Files overview/RECONSTRUCTED_PDFs" \
  --api-url "$FLTR_API_URL" \
  --api-key "$FLTR_API_KEY" \
  --dry-run
```

**Actual Upload (Split Files - Recommended):**

```bash
# 1. Split large files first
python scripts/split_large_epstein_pdfs.py

# 2. Upload splits
python scripts/bulk_upload.py \
  --dataset-id "$FLTR_DATASET_ID" \
  --source-dir "datasets/Epstein Files overview/SPLIT_PDFs" \
  --api-url "$FLTR_API_URL" \
  --api-key "$FLTR_API_KEY" \
  --batch-size 100 \
  --concurrency 50
```

**Upload Without Splitting (Not Recommended):**

```bash
python scripts/bulk_upload.py \
  --dataset-id "$FLTR_DATASET_ID" \
  --source-dir "datasets/Epstein Files overview/RECONSTRUCTED_PDFs" \
  --api-url "$FLTR_API_URL" \
  --api-key "$FLTR_API_KEY" \
  --batch-size 50 \
  --concurrency 25
```

---

## Monitoring Production Upload

### 1. Watch Upload Progress

```bash
# Local terminal (where you ran the script)
tail -f bulk_upload_*.log
```

### 2. Check Production Database

```bash
# SSH to production
ssh your-user@your-server

# Or connect directly if you have credentials
psql "$PRODUCTION_DATABASE_URL" -c "
  SELECT status, COUNT(*)
  FROM documents
  WHERE dataset_id = '$FLTR_DATASET_ID'
  GROUP BY status;
"
```

### 3. Monitor Modal Processing

```bash
# View live logs
modal app logs fltr

# Or visit Modal dashboard
# https://modal.com/apps
```

### 4. Check Costs

**Modal Dashboard:**
- https://modal.com → Usage
- Filter by date
- See compute costs

**OpenAI Dashboard:**
- https://platform.openai.com/usage
- API Usage tab

**R2 Storage:**
- Cloudflare dashboard → R2
- Check storage size

---

## Expected Timeline (Production)

```
Upload Phase: 2-4 hours
├─ 1,525 files (or 1,650 split)
├─ 22 GB total
├─ Direct to R2 via presigned URLs
└─ Parallel upload (50 concurrent)

Processing Phase: 4-6 hours
├─ Modal auto-scales to 100 containers
├─ Parallel document processing
├─ OpenAI embeddings generation
└─ Milvus vector storage

Total: 6-10 hours (upload + processing overlap)
```

---

## Troubleshooting Production

### "Authentication failed"

Check API key:
```bash
# Test the key
curl -H "X-API-Key: $FLTR_API_KEY" \
  https://api.tryfltr.com/api/v1/datasets

# If 401/403: key is invalid or not configured
```

**Fix:**
1. Regenerate key
2. Add to DigitalOcean environment
3. Wait for redeploy
4. Test again

### "Insufficient credits"

Check credit balance:
```bash
curl -H "X-API-Key: $FLTR_API_KEY" \
  https://api.tryfltr.com/api/v1/credits/balance | jq .
```

**Cost:** 1,525 files × 10 credits = **15,250 credits needed**

If insufficient:
- Add credits via billing system
- Or adjust credit requirements in code

### "Connection timeout"

Your production server might have firewall rules. Check:
```bash
# Test connectivity
curl -v https://api.tryfltr.com/api/v1/health

# If timeout: check DigitalOcean firewall rules
```

### "Modal webhook failed"

Check Modal deployment:
```bash
# List deployed apps
modal app list

# Should show:
# fltr (deployed)

# If not deployed:
cd modal
modal deploy modal_app.py
```

Then update `MODAL_WEBHOOK_URL` in DigitalOcean environment.

### High upload failure rate

Reduce concurrency:
```bash
python scripts/bulk_upload.py \
  --concurrency 25 \  # Reduced from 50
  --batch-size 50 \   # Reduced from 100
  ...
```

---

## Safety Checks Before Production Upload

### Pre-Flight Checklist

- [ ] **API key generated and tested** ✓
- [ ] **Production environment verified** (Modal deployed, DB accessible)
- [ ] **Dataset created** (have dataset UUID)
- [ ] **Credits available** (need 15,250 for 1,525 files)
- [ ] **Budget approved** (~$316-376 one-time cost)
- [ ] **Dry run completed successfully**
- [ ] **Monitoring set up** (can access logs)
- [ ] **Backup plan** (what if upload fails)
- [ ] **Off-hours upload** (lower server load)
- [ ] **Team notified** (others aware of large processing job)

### Dry Run Test

```bash
# Test with 10 small files first
python scripts/bulk_upload.py \
  --dataset-id "$FLTR_DATASET_ID" \
  --source-dir "datasets/Epstein Files overview/RECONSTRUCTED_PDFs" \
  --api-url "$FLTR_API_URL" \
  --api-key "$FLTR_API_KEY" \
  --batch-size 10 \
  --dry-run

# Should output:
# Found 1525 documents
# [DRY RUN] No files will be uploaded
```

---

## Emergency: Stop Upload

If you need to stop the upload mid-way:

**Stop upload script:**
```bash
# Press Ctrl+C in terminal
# Upload script will stop gracefully
```

**Uploaded files will still process:**
- Files already in R2 will trigger webhooks
- Modal will process them
- Can't stop Modal processing once started
- But: only charged for what gets processed

**To prevent more processing:**
- Disable Modal webhook in DigitalOcean env
- Or pause Modal deployment

---

## Cost Tracking

### Track as Upload Progresses

```bash
# Check Modal costs (every hour)
# Visit: https://modal.com/usage

# Check document processing status
psql "$PRODUCTION_DATABASE_URL" -c "
  SELECT
    status,
    COUNT(*) as count,
    ROUND(AVG(chunk_count)) as avg_chunks
  FROM documents
  WHERE dataset_id = '$FLTR_DATASET_ID'
  GROUP BY status;
"
```

### Expected Costs Breakdown

```
After 25% complete (~380 files):
- Modal: ~$75
- OpenAI: ~$6
- Total: ~$81

After 50% complete (~760 files):
- Modal: ~$150
- OpenAI: ~$13
- Total: ~$163

After 100% complete (1,525 files):
- Modal: ~$316
- OpenAI: ~$26
- Total: ~$342
```

---

## Verification After Upload

### 1. Check All Files Processed

```bash
psql "$PRODUCTION_DATABASE_URL" -c "
  SELECT status, COUNT(*)
  FROM documents
  WHERE dataset_id = '$FLTR_DATASET_ID'
  GROUP BY status;
"

# Expected:
# status | count
# -------+-------
# ready  | 1525  (or 1650 if split)
# failed | 0-5   (< 1%)
```

### 2. Test Search

```bash
curl -X POST "https://api.tryfltr.com/api/v1/datasets/$FLTR_DATASET_ID/search" \
  -H "X-API-Key: $FLTR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "Epstein",
    "limit": 10
  }' | jq .
```

### 3. Check Storage

```bash
# Check R2 storage size
# Cloudflare dashboard → R2 → fltr-datasets bucket
# Should show: ~22 GB + extracted images
```

### 4. Review Costs

```bash
# Modal: https://modal.com/usage
# OpenAI: https://platform.openai.com/usage
# R2: Cloudflare dashboard
```

---

## Quick Reference Card

### Environment Setup
```bash
export FLTR_API_URL="https://api.tryfltr.com"
export FLTR_API_KEY="fltr_YOUR_KEY"
export FLTR_DATASET_ID="your-dataset-id"
```

### Upload Command
```bash
python scripts/bulk_upload.py \
  --dataset-id "$FLTR_DATASET_ID" \
  --source-dir "datasets/Epstein Files overview/SPLIT_PDFs" \
  --api-url "$FLTR_API_URL" \
  --api-key "$FLTR_API_KEY"
```

### Monitor
```bash
# Logs
tail -f bulk_upload_*.log

# Database
psql "$DB" -c "SELECT status, COUNT(*) FROM documents WHERE dataset_id='$ID' GROUP BY status;"

# Modal
modal app logs fltr
```

### Costs
```bash
# Expected: $316-376 one-time
# Recurring: $1-2/month
```

---

## Support

If issues arise:

1. **Check logs first:**
   - Upload: `bulk_upload_*.log`
   - Modal: `modal app logs fltr`
   - FastAPI: DigitalOcean app logs

2. **Common fixes:**
   - Auth errors: Regenerate API key
   - Timeout: Reduce concurrency
   - Modal errors: Redeploy Modal app
   - Rate limits: Add delays between batches

3. **Rollback plan:**
   - Delete dataset via API
   - Credits are refunded for failed uploads
   - R2 storage auto-cleaned after dataset deletion

---

**Ready to upload to production!** 🚀

Remember:
- ✅ Use `--dry-run` first
- ✅ Monitor costs
- ✅ Off-hours upload recommended
- ✅ Have rollback plan ready
