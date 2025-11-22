# Migration Options - Page Number Fix

## Current Situation

The migration script is running but not updating chunks because:
1. ✅ It finds documents with `page=0` chunks
2. ✅ It finds images in R2
3. ❌ It can't extract page numbers from R2 image filenames (old format: `0_tmplg9eqtl2.pdf-0-full.png`)
4. ❌ Without page numbers, it can't match chunks to images

## Option 1: Deploy Fix and Re-run (Recommended)

**What:** Deploy the updated `migrate_image_r2_keys.py` with improved filename parsing, then re-run migration.

**Pros:**
- Clean solution
- Handles both old and new image formats
- Fixes the root cause

**Cons:**
- Requires deployment
- Need to wait for deployment

**Steps:**
1. Deploy updated `fastapi/scripts/migrate_image_r2_keys.py`
2. Re-run migration script
3. Should work correctly

**Time:** ~5 minutes to deploy + ~11 minutes to migrate = **~16 minutes total**

---

## Option 2: Extract Page Numbers Directly from PDFs

**What:** Modify migration script to extract page numbers from PDFs directly instead of relying on R2 filenames.

**How it works:**
- When extracting images from PDF, we get `(page, index)` pairs
- Use these to update chunks directly
- More reliable since PDF is source of truth

**Pros:**
- Works immediately (no deployment needed)
- More reliable (PDF is source of truth)
- Doesn't depend on R2 filename format

**Cons:**
- Requires downloading PDFs (slower, more expensive)
- More complex logic
- Still need to deploy eventually for new documents

**Implementation:**
```python
# When extracting images from PDF:
for img in extracted_images:
    page_0_indexed = img['page'] - 1  # Convert 1-indexed to 0-indexed
    # Update chunks with this page number
    # Match by image_index
```

**Time:** ~30-60 minutes (needs to download PDFs)

---

## Option 3: Hybrid Approach

**What:**
1. Try to extract page numbers from R2 filenames (current approach)
2. If that fails, extract from PDF and update chunks
3. Then proceed with image migration

**Pros:**
- Fast for documents with parseable filenames
- Falls back to PDF extraction for problematic ones
- Best of both worlds

**Cons:**
- More complex code
- Still need deployment

**Time:** ~15-20 minutes (mix of fast and slow)

---

## Option 4: Manual Fix for Critical Documents

**What:** Only migrate documents that are actively being used, let others wait.

**Pros:**
- Minimal effort
- Fixes what matters most
- Can do full migration later

**Cons:**
- Doesn't fix everything
- Still need full migration eventually

**Time:** ~5-10 minutes for critical docs

---

## Recommendation

**Go with Option 1 (Deploy Fix and Re-run)** because:
1. ✅ Cleanest solution
2. ✅ Fixes the root cause
3. ✅ Fastest overall (no PDF downloads needed)
4. ✅ Works for all documents uniformly

The fix I made handles the old format correctly:
- `0_tmplg9eqtl2.pdf-0-full.png` → extracts `page=0, index=0`
- `5_document.pdf-2-full.png` → extracts `page=2, index=5`

Once deployed, the migration should work smoothly.

---

## Quick Deploy Checklist

Files that need deployment:
- ✅ `fastapi/scripts/migrate_image_r2_keys.py` (improved parsing)
- ✅ `fastapi/scripts/batch_migrate_page_numbers.py` (batch script)

After deployment, run:
```bash
docker exec api-container python3 scripts/batch_migrate_page_numbers.py --batch-size 10
```

---

## If Option 1 Doesn't Work

If after deployment the migration still has issues, we can:
1. Add Option 2 as fallback (extract from PDF)
2. Debug the specific documents that fail
3. Use Option 3 (hybrid approach)

But Option 1 should work - the fix handles the filename format correctly.

