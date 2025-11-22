# Upload Decision Guide: Local vs Production

**Quick Answer:** Use **Production** if you want the data accessible in production. Use **Local** only for testing.

---

## Option 1: Upload to Production (api.tryfltr.com) ✅ RECOMMENDED

### When to Use
- ✅ You want the data accessible in your production app
- ✅ You're ready to pay real costs ($316-376)
- ✅ This is the final dataset, not a test

### Pros
- Data immediately available in production
- No need to migrate later
- Real performance metrics
- Production-grade processing

### Cons
- **Real $ costs** (~$316-376)
- Need production API key
- Affects production credits
- Can't easily undo

### Setup Time
- **5-10 minutes** (API key + config)

### Command
```bash
export FLTR_API_URL="https://api.tryfltr.com"
export FLTR_API_KEY="fltr_YOUR_PRODUCTION_KEY"

python scripts/bulk_upload.py \
  --dataset-id "$DATASET_ID" \
  --source-dir "datasets/Epstein Files overview/SPLIT_PDFs" \
  --api-url "$FLTR_API_URL" \
  --api-key "$FLTR_API_KEY"
```

---

## Option 2: Upload to Local (localhost:8000)

### When to Use
- ✅ Testing the upload process
- ✅ Validating data quality
- ✅ Experimenting with settings
- ✅ Development work

### Pros
- Free (Modal dev environment)
- Easy to delete and retry
- No production impact
- Fast iteration

### Cons
- Data only in local database
- Need to re-upload to production later
- Dev Modal environment (slower?)
- Not accessible to users

### Setup Time
- **2 minutes** (generate local API key)

### Command
```bash
./scripts/setup_bulk_upload.sh

python scripts/bulk_upload.py \
  --dataset-id "$DATASET_ID" \
  --source-dir "datasets/Epstein Files overview/SPLIT_PDFs"
```

---

## Side-by-Side Comparison

| Aspect | Production | Local |
|--------|------------|-------|
| **URL** | api.tryfltr.com | localhost:8000 |
| **Cost** | **$316-376 real $** | Free (dev credits) |
| **Data Access** | Production app | Only local |
| **Setup** | 5-10 min | 2 min |
| **Speed** | Production Modal | Dev Modal |
| **Risk** | Real costs, real data | No risk |
| **Undo** | Hard (costs already spent) | Easy (delete dataset) |
| **Best For** | Final upload | Testing |

---

## Recommended Workflow

### 1. **Test Locally First** (Optional but Smart)

```bash
# Test with 10 files locally
./scripts/setup_bulk_upload.sh

python scripts/bulk_upload.py \
  --dataset-id "test-local-dataset-id" \
  --source-dir "datasets/Epstein Files overview/RECONSTRUCTED_PDFs" \
  --batch-size 10

# Verify:
# - Upload works
# - Processing completes
# - Search works
# - Cost reasonable
```

### 2. **Upload to Production** (When Ready)

```bash
# Set production credentials
export FLTR_API_URL="https://api.tryfltr.com"
export FLTR_API_KEY="fltr_YOUR_PRODUCTION_KEY"
export FLTR_DATASET_ID="production-dataset-id"

# Upload all files
python scripts/bulk_upload.py \
  --dataset-id "$FLTR_DATASET_ID" \
  --source-dir "datasets/Epstein Files overview/SPLIT_PDFs" \
  --api-url "$FLTR_API_URL" \
  --api-key "$FLTR_API_KEY"
```

---

## Quick Decision Tree

```
Do you need this data in production RIGHT NOW?
├─ YES → Upload to Production
│         See: PRODUCTION_BULK_UPLOAD.md
│
└─ NO → Are you testing?
    ├─ YES → Upload to Local
    │         See: BULK_UPLOAD_SETUP.md
    │
    └─ NO → Upload to Production anyway
              (you'll need it eventually)
```

---

## For Your Use Case (Epstein Dataset)

**You probably want: Production ✅**

Why:
- This is your real dataset (1,525 court documents)
- You want it searchable in production
- You're ready for the cost (~$316-376)
- No reason to upload twice (local then prod)

**Steps:**
1. Read [PRODUCTION_BULK_UPLOAD.md](PRODUCTION_BULK_UPLOAD.md)
2. Generate production API key
3. Split large PDFs
4. Upload to production
5. Monitor processing

---

## Cost Comparison

### Upload to Local
```
Modal: Free (dev credits)
OpenAI: Free (or test key)
Total: $0
```

### Upload to Production
```
Modal: $290-350
OpenAI: $26.40
R2: $0.33/month
Total: $316-376 one-time
```

**BUT:** If you test local then upload to production, you pay **modal costs twice** (once for local test, once for production). Better to just upload to production once.

---

## FAQ

### "Should I test locally first?"

**Only if:**
- You're unsure if the data is good
- You want to verify search quality
- You're testing code changes
- Budget is tight and you want to estimate exact cost

**Skip local test if:**
- You're confident in the data
- You trust the pipeline
- You want data in production ASAP
- You understand the costs

### "Can I upload local data to production later?"

**No easy migration.** You would need to:
1. Export vectors from local Milvus
2. Import to production Milvus
3. Copy R2 storage
4. Migrate database rows

It's easier to just upload to production directly.

### "What if something goes wrong in production?"

You can:
- Delete the dataset (refunds unused credits)
- Retry failed documents individually
- Re-upload specific files

But costs already spent on successfully processed documents can't be refunded.

---

## My Recommendation

**Upload directly to production** using [PRODUCTION_BULK_UPLOAD.md](PRODUCTION_BULK_UPLOAD.md):

1. Split PDFs (15 min)
2. Generate production API key (5 min)
3. Create dataset in production UI (2 min)
4. Run upload with `--dry-run` first (1 min)
5. Run actual upload (3-4 hours)
6. Monitor processing (4-6 hours)

**Total: 7-10 hours start to finish**

---

## Next Steps

Choose your path:

### → Production Upload
Read: **[PRODUCTION_BULK_UPLOAD.md](PRODUCTION_BULK_UPLOAD.md)**

### → Local Upload (Testing)
Read: **[BULK_UPLOAD_SETUP.md](BULK_UPLOAD_SETUP.md)**

### → Compare Strategies
Read: **[EPSTEIN_DATASET_WITH_SPLITTING.md](EPSTEIN_DATASET_WITH_SPLITTING.md)**
