# Scripts for R2 Image Links & Document Production Fixes

This document outlines the scripts available for reviewing R2 image links and fixing production issues for documents.

## Available Scripts

### 1. `migrate_image_r2_keys.py` - Backfill R2 Image URLs

**Purpose**: Migrates older documents that are missing image R2 keys.

**What it does**:
- Finds documents with OCR chunks missing `image_r2_key`
- **Verifies PDF exists in R2 before processing** (safely skips if missing)
- Checks if images already exist in R2 (optimization - skips re-extraction)
- Downloads original PDFs from R2 only if images need to be uploaded
- Extracts images from PDFs
- Uploads images to R2 at the correct path: `{dataset_id}/{document_id}/images/{page}_{index}.{ext}`
- Updates Milvus chunks with `image_r2_key` values

**Usage**:
```bash
# From fastapi directory:
cd fastapi

# Dry run (preview what would be done)
python scripts/migrate_image_r2_keys.py --dry-run

# Migrate specific document
python scripts/migrate_image_r2_keys.py --document-id <doc_id>

# Migrate with limit
python scripts/migrate_image_r2_keys.py --limit 10
```

**Location**: `fastapi/scripts/migrate_image_r2_keys.py`

---

### 2. `diagnose_missing_object_keys.py` - Comprehensive Document Health Check

**Purpose**: Comprehensive diagnosis of documents with missing R2 objects, orphaned chunks, or missing image links.

**What it does**:
- Finds documents with failed status or NoSuchKey errors
- Checks if `object_key` exists in R2 (tries URL-encoded variations)
- Checks Milvus for chunks and image_r2_key status
- Determines document state:
  - **null_pointer**: No R2 file, no chunks → Delete document
  - **orphaned_chunks**: No R2 file, has chunks → Delete chunks or mark deleted
  - **missing_links**: Has R2 file, has chunks, missing image_r2_key → Run migration
  - **not_processed**: Has R2 file, no chunks → Re-process document
  - **healthy**: Has R2 file, has chunks, all links present → No action needed
- Provides state-based fix recommendations
- Can automatically fix object_key when a match is found
- Can automatically mark null pointer documents as deleted

**Usage**:
```bash
# From fastapi directory:
cd fastapi

# Check failed documents (comprehensive diagnosis)
python scripts/diagnose_missing_object_keys.py --status failed

# Check with error filter
python scripts/diagnose_missing_object_keys.py --status failed --error-filter "NoSuchKey"

# Auto-fix object_key when matches found
python scripts/diagnose_missing_object_keys.py --status failed --fix

# Mark null pointer documents as deleted (safe - only if no R2, no chunks)
python scripts/diagnose_missing_object_keys.py --status failed --mark-deleted

# Check all problematic documents (failed + stuck)
python scripts/diagnose_missing_object_keys.py --status all --limit 50
```

**Options**:
- `--status`: Filter by status (`failed`, `stuck`, `all`) - default: `failed`
- `--limit`: Limit number of documents to check
- `--fix`: Automatically fix `object_key` when a match is found
- `--error-filter`: Filter by error message (e.g., `NoSuchKey`)
- `--mark-deleted`: Automatically mark null pointer documents (no R2, no chunks) as deleted

**Output**:
- Shows document state for each document
- Groups documents by state for easy action planning
- Provides specific fix recommendations based on state

**Location**: `fastapi/scripts/diagnose_missing_object_keys.py`

---

### 3. `check_document.py` - Check Document Status

**Purpose**: Quick check of a specific document's status and details.

**What it does**:
- Shows document details (status, file name, chunk count, etc.)
- Displays processing metadata
- Checks if document is stuck
- Provides recommendations based on status

**Usage**:
```bash
# From fastapi directory:
cd fastapi
python scripts/check_document.py <document_id>
```

**Example**:
```bash
python scripts/check_document.py ba2fb51a-f94f-4e92-b566-1ba51aa6a9c0
```

**Location**: `fastapi/scripts/check_document.py`

---

### 4. `debug_production_document.py` - Debug Production Documents via API

**Purpose**: Debug production documents through the API (no direct database access needed).

**What it does**:
- Fetches document details from production API
- Analyzes processing metadata
- Provides recommendations based on document status
- Works with production environment

**Usage**:
```bash
# From fastapi directory:
cd fastapi
python scripts/debug_production_document.py <document_id> --dataset-slug <slug>
# Will prompt for API key
```

**Example**:
```bash
python scripts/debug_production_document.py 7e026e87-0e85-4a50-b579-3fa9d563e663 --dataset-slug "faa-a%26p-general-handbook-2e8e6af0"
```

**Options**:
- `--dataset-slug`: Dataset slug (if known)
- `--dataset-id`: Dataset UUID (if known)
- `--api-url`: API base URL (default: `https://api.tryfltr.com`)

**Location**: `fastapi/scripts/debug_production_document.py`

---

### 5. `trigger_stuck_documents.py` - Retrigger Stuck Documents

**Purpose**: Manually trigger processing for documents stuck in "uploaded" status.

**What it does**:
- Finds documents stuck in "uploaded" status
- Triggers Modal processing for each document
- Reports success/failure counts

**Usage**:
```bash
# Dry run (show what would be processed)
python scripts/trigger_stuck_documents.py --dry-run

# Trigger processing
python scripts/trigger_stuck_documents.py --limit 10
```

**Options**:
- `--limit`: Limit number of documents to process
- `--dry-run`: Show what would be processed without actually doing it

**Location**: `fastapi/scripts/trigger_stuck_documents.py`

---

## Recommended Workflow

**Note**: All commands assume you're in the `fastapi` directory:
```bash
cd fastapi
```

### Step 1: Comprehensive Health Check

Run the enhanced diagnostic script to get a complete picture of document states:

```bash
# Check all problematic documents (failed + stuck)
python scripts/diagnose_missing_object_keys.py --status all --limit 50
```

This will show:
- Documents grouped by state (null_pointer, orphaned_chunks, missing_links, not_processed, healthy)
- Specific fix recommendations for each state
- Summary statistics

### Step 2: Handle Documents by State

Based on the diagnosis, take appropriate actions:

#### **Null Pointers** (no R2, no chunks)
```bash
# Mark as deleted (safe - no data loss)
python scripts/diagnose_missing_object_keys.py --status failed --mark-deleted
```

#### **Orphaned Chunks** (no R2, has chunks)
- **Option 1**: Delete chunks manually (if file is truly gone)
- **Option 2**: Mark document as deleted (preserves chunks for potential recovery)
- **Option 3**: If file exists elsewhere, fix object_key first:
```bash
python scripts/diagnose_missing_object_keys.py --status failed --fix
```

#### **Missing Image Links** (has R2, has chunks, missing image_r2_key)
```bash
# Dry run first
python scripts/migrate_image_r2_keys.py --dry-run --document-id <doc_id>

# Then migrate
python scripts/migrate_image_r2_keys.py --document-id <doc_id>
```

#### **Not Processed** (has R2, no chunks)
```bash
# Retrigger processing
python scripts/trigger_stuck_documents.py --document-id <doc_id>
```

### Step 3: Fix Object Key Mismatches

If documents have wrong object_key values (but file exists with different key):

```bash
# Check what would be fixed
python scripts/diagnose_missing_object_keys.py --status failed --error-filter "NoSuchKey"

# Auto-fix when matches found
python scripts/diagnose_missing_object_keys.py --status failed --fix
```

### Step 4: Batch Migrate Image Links

For documents that need image links backfilled:

```bash
# Dry run first
python scripts/migrate_image_r2_keys.py --dry-run --limit 10

# Then run for real (will skip images that already exist)
python scripts/migrate_image_r2_keys.py --limit 10
```

### Step 5: Verify Results

Re-run the diagnostic to verify fixes:

```bash
python scripts/diagnose_missing_object_keys.py --status failed --limit 20
```

---

## Common Issues & Solutions

### Issue: Images Not Loading in Documents

**Symptoms**: OCR chunks exist but images don't display in the UI.

**Solution**:
```bash
# Backfill image R2 keys for affected documents
python scripts/migrate_image_r2_keys.py --document-id <doc_id>
```

### Issue: Document Status "Failed" with NoSuchKey Error

**Symptoms**: Document shows as failed with error about missing object in R2.

**Solution**:
```bash
# Diagnose and fix missing object keys
python scripts/diagnose_missing_object_keys.py --status failed --error-filter "NoSuchKey" --fix
```

### Issue: Document Stuck in "Uploaded" Status

**Symptoms**: Document uploaded but never started processing.

**Solution**:
```bash
# Retrigger processing
python scripts/trigger_stuck_documents.py --limit 10
```

### Issue: Need to Debug Production Document

**Symptoms**: Document has issues in production, need to investigate without DB access.

**Solution**:
```bash
# Debug via API
python scripts/debug_production_document.py <document_id> --dataset-slug <slug>
```

---

## Script Locations

All scripts are located in: `fastapi/scripts/`

**Note**: All scripts should be run from the `fastapi` directory:
```bash
cd fastapi
python scripts/<script_name>.py [options]
```

- `migrate_image_r2_keys.py`
- `diagnose_missing_object_keys.py`
- `check_document.py`
- `debug_production_document.py`
- `trigger_stuck_documents.py`

---

## Important Notes

1. **Always use `--dry-run` first** when available to preview changes before applying them
2. **Backup considerations**: Some scripts modify the database - ensure you have backups or are working on a test environment first
3. **Production access**: `debug_production_document.py` requires API key authentication
4. **R2 credentials**: Scripts that interact with R2 require proper R2 credentials in environment variables
5. **Database connection**: Most scripts connect directly to the database - ensure you're connected to the correct environment

---

## Case Study: Missing R2 Objects Diagnostic

### Example Output: All Documents Missing from R2

**Date**: 2025-01-XX
**Dataset ID**: `3deed0c5-3a5a-43ec-ac54-9ac44044fd3b`
**Status**: All 20 documents failed with NoSuchKey errors

**Command Run**:
```bash
python scripts/diagnose_missing_object_keys.py --status failed --limit 20
```

**Diagnostic Results**:
```
📋 Found 20 documents with status 'failed'
✅ Objects exist in R2: 0
❌ Objects not found: 20
💡 Documents with suggested fixes: 0
```

**Key Findings**:
- All documents belong to the same dataset (`3deed0c5-3a5a-43ec-ac54-9ac44044fd3b`)
- All have `failed` status with NoSuchKey errors
- Script tried both original and URL-encoded key variants
- No matching objects found in R2 with the dataset prefix
- Script searched for objects with prefix `3deed0c5-3a5a-43ec-ac54-9ac44044fd3b/` but found none

**Affected Documents** (Complete List):
1. `2020.11 DOJ Office of Professional Responsibility Report_part06of06.pdf` (ID: `dc105c70-cbba-42f0-a799-8a41617558a3`)
2. `20-3061_Documents_part04of05.pdf` (ID: `11c464d0-baf1-413d-bbad-fb0d766de557`)
3. `20-3061_Documents_part01of05.pdf` (ID: `2bde2761-06a8-497d-999b-0809309f0734`)
4. `22-1426_Documents_part08of19.pdf` (ID: `cfcc3153-36e0-4f5a-9d1a-0eb8b81d1100`)
5. `2020.11 DOJ Office of Professional Responsibility Report_part01of06.pdf` (ID: `d0ac31fd-1634-42ce-ac70-a4fc079ebcb1`)
6. `20-3061_Documents_part02of05.pdf` (ID: `ba150bdd-5230-4308-87b6-1f05660bc623`)
7. `22-1426_Documents_part17of19.pdf` (ID: `a864a347-647b-4595-bc22-6f281c281601`)
8. `20-3061_Documents_part03of05.pdf` (ID: `bfacb59f-7370-484c-97b6-6866397a15f8`)
9. `2023.06 OIG Memorandum 23-085_part01of02.pdf` (ID: `c54b4d34-26b8-4c5a-ac0a-6eedc3d5652e`)
10. `204_part02of03.pdf` (ID: `e9f5d705-62f1-4228-9ab5-4fefa7ab42d1`)
11. `2020.11 DOJ Office of Professional Responsibility Report_part02of06.pdf` (ID: `7e763fcc-ba56-4134-8a85-5159a56e919f`)
12. `204-03_part02of05.pdf` (ID: `26ef6836-cce3-483a-9bcd-8a0abd991ce7`)
13. `2023.06 OIG Memorandum 23-085_part02of02.pdf` (ID: `e084653c-91fa-425a-899d-cd0f469c1ad5`)
14. `2020.11 DOJ Office of Professional Responsibility Report_part03of06.pdf` (ID: `46352cab-e990-4966-ac33-a1dc9c4cedcc`)
15. `204_part01of03.pdf` (ID: `d21ec282-7b64-40f9-be2e-ed4a7e44ffc1`)
16. `20-3061_Documents_part05of05.pdf` (ID: `eb849080-4bce-4acf-95e6-46dc33b7704b`)
17. `204_part03of03.pdf` (ID: `2f910d46-e7c3-4546-927c-7353a9bdfcdb`)
18. `2020.11 DOJ Office of Professional Responsibility Report_part04of06.pdf` (ID: `6f302e2e-770d-40cf-9bc4-561e3c1ead0c`)
19. `21-58_Documents Pt 1 of 4_part02of03.pdf` (ID: `b402661f-0614-41b8-aac9-a0b4199aa06f`)
20. `21-58_Documents Pt 2 of 4_part02of02.pdf` (ID: `b8c45886-7b7d-47d3-8deb-4c691a593880`)

**Error Pattern**:
All documents show the same error:
```
An error occurred (NoSuchKey) when calling the GetObject operation: The specified key does not exist.
```

**Key Variants Tried**:
- Original key: `3deed0c5-3a5a-43ec-ac54-9ac44044fd3b/{filename}`
- URL-encoded key: `3deed0c5-3a5a-43ec-ac54-9ac44044fd3b/{url_encoded_filename}` (for files with spaces)

**Possible Causes**:
1. **Files never uploaded**: Upload process failed before completion
2. **Files deleted**: Objects were removed from R2 after upload
3. **Wrong bucket**: Files uploaded to different R2 bucket
4. **Dataset migration**: Files moved to different dataset location
5. **Upload URL expired**: Presigned URLs expired before upload completed

**Recommended Actions**:
1. **Check R2 bucket directly**: List all objects with dataset prefix to see if files exist elsewhere
   ```bash
   # Use R2 dashboard or AWS CLI to list objects
   aws s3 ls s3://bucket-name/3deed0c5-3a5a-43ec-ac54-9ac44044fd3b/ --endpoint-url=https://<account-id>.r2.cloudflarestorage.com
   ```

2. **Check upload logs**: Review logs around the time these documents were created to see if uploads completed

3. **Check document creation timestamps**: Verify if documents were created but uploads never completed

4. **Re-upload if needed**: If files are truly missing, documents may need to be re-uploaded

5. **Mark as deleted**: If files are confirmed missing and cannot be recovered, update document status accordingly

**Next Steps**:
```bash
# 1. Check if any objects exist for this dataset
# (Manual R2 inspection or AWS CLI)

# 2. Review document creation/upload timeline
python scripts/check_document.py <document_id>

# 3. If files are found elsewhere, update object_key
# (Manual database update or script modification)

# 4. If files are truly missing, consider:
#    - Re-uploading from source
#    - Marking documents as deleted
#    - Updating status to reflect actual state
```

---

### Analysis: What We Can Learn From This Run

#### 1. **Systematic Failure Pattern**

**Observation**: All 20 failed documents belong to the same dataset (`3deed0c5-3a5a-43ec-ac54-9ac44044fd3b`)

**Implications**:
- This is **not** a random failure - it's dataset-specific
- Likely caused by a bulk upload operation that failed
- Could indicate a problem with how this particular dataset was processed
- Suggests a single point of failure rather than individual document issues

**Action Items**:
- Check if other documents in this dataset succeeded
- Review upload logs for this specific dataset
- Investigate if there was a bulk upload operation that failed mid-process

#### 2. **Multi-Part PDF Pattern**

**Observation**: All documents are multi-part PDFs (e.g., `part01of06`, `part02of05`, `part03of03`)

**Implications**:
- These are likely split PDFs from a larger document
- Multi-part uploads may have different failure modes than single-file uploads
- Could indicate an issue with:
  - Sequential upload handling
  - Upload URL generation for multi-part files
  - Presigned URL expiration during multi-part uploads

**Action Items**:
- Review how multi-part PDFs are handled in the upload flow
- Check if presigned URLs expire before all parts are uploaded
- Verify if there's a race condition in multi-part upload processing

#### 3. **Complete Dataset Failure**

**Observation**: Script searched for objects with prefix `3deed0c5-3a5a-43ec-ac54-9ac44044fd3b/` and found **none**

**Implications**:
- **No files exist in R2 for this entire dataset prefix**
- This suggests:
  - Files were never uploaded (most likely)
  - Files were uploaded to a different location/bucket
  - Files were deleted after upload but before processing
  - Dataset was created but upload step never completed

**Action Items**:
- Check if dataset has any successful documents
- Verify dataset creation timestamp vs document creation timestamps
- Check if there's a different R2 bucket or path where files might exist
- Review if there was a dataset migration or reorganization

#### 4. **URL Encoding Attempts**

**Observation**: Script tried both original and URL-encoded variants for files with spaces

**Implications**:
- Script correctly handles URL encoding edge cases
- The issue is **not** a URL encoding problem
- Files are genuinely missing, not just misnamed

**Action Items**:
- No need to investigate URL encoding issues for these documents
- Focus on why files were never uploaded in the first place

#### 5. **Error Consistency**

**Observation**: All documents show identical error: `NoSuchKey` when calling `GetObject`

**Implications**:
- Consistent error pattern suggests a systematic issue
- Not caused by individual file corruption or upload failures
- Likely a batch operation that failed entirely

**Action Items**:
- Check for batch upload operations around the time these documents were created
- Review if there's a limit on concurrent uploads that was exceeded
- Investigate if there was a service outage during upload window

#### 6. **Document Type Pattern**

**Observation**: All documents are DOJ/OIG reports (Department of Justice / Office of Inspector General)

**Implications**:
- These may have been uploaded as part of a specific dataset/collection
- Could indicate a bulk import from an external source
- May have specific file characteristics (size, format) that caused issues

**Action Items**:
- Check if there's a pattern in file sizes or formats
- Review if these documents were part of a bulk import operation
- Verify if there are other DOJ/OIG documents that succeeded

---

### Key Insights & Recommendations

#### Immediate Actions:
1. **Check dataset completeness**: Query database to see if ANY documents in this dataset succeeded
   ```sql
   SELECT status, COUNT(*)
   FROM documents
   WHERE dataset_id = '3deed0c5-3a5a-43ec-ac54-9ac44044fd3b'
   GROUP BY status;
   ```

2. **Review upload timeline**: Check when these documents were created vs when uploads should have occurred
   ```bash
   python scripts/check_document.py dc105c70-cbba-42f0-a799-8a41617558a3
   ```

3. **Check R2 bucket directly**: Use R2 dashboard or AWS CLI to verify if ANY objects exist for this dataset
   ```bash
   aws s3 ls s3://bucket-name/3deed0c5-3a5a-43ec-ac54-9ac44044fd3b/ \
     --endpoint-url=https://<account-id>.r2.cloudflarestorage.com \
     --recursive
   ```

#### Long-term Improvements:
1. **Add upload verification**: After upload completes, verify file exists in R2 before marking as "uploaded"
2. **Better error handling**: Track upload failures separately from processing failures
3. **Bulk upload monitoring**: Add logging/metrics for bulk upload operations
4. **Presigned URL validation**: Check URL expiration before attempting uploads
5. **Multi-part upload handling**: Review and improve handling of multi-part PDF uploads

#### Prevention Strategies:
1. **Upload confirmation**: Require R2 webhook confirmation before marking document as "uploaded"
2. **Retry logic**: Implement automatic retry for failed uploads
3. **Health checks**: Periodic verification that "uploaded" documents actually exist in R2
4. **Alerting**: Set up alerts when multiple documents from same dataset fail

---

### Validation: Does This Validate Our Fix Plan?

#### ❌ **The `--fix` Flag Will NOT Work for These Documents**

**Why**: The diagnostic script found **zero matching objects** in R2. The `--fix` flag only works when:
- The script finds an object in R2 that matches the document (with a different key)
- It can automatically update the `object_key` in the database

**What the results show**:
```
✅ Objects exist in R2: 0
❌ Objects not found: 20
💡 Documents with suggested fixes: 0
```

**Conclusion**: The `--fix` approach is **not applicable** here because there are no objects to match against.

#### ✅ **What This Tells Us About Our Fix Strategy**

**1. The Diagnostic Script Works Correctly**
- ✅ Successfully checked R2 for objects
- ✅ Tried URL-encoded variants (handled edge cases)
- ✅ Searched dataset prefix (comprehensive search)
- ✅ Correctly identified that no objects exist

**2. We Need a Different Fix Strategy**

Since `--fix` won't work, we have these options:

**Option A: Re-upload Documents** (if source files available)
```bash
# This would require:
# 1. Having access to original source files
# 2. Re-creating upload URLs
# 3. Re-uploading files to R2
# 4. Updating document status
```

**Option B: Mark Documents as Deleted** (if files are truly gone)
```sql
-- Update document status to reflect reality
UPDATE documents
SET status = 'deleted',
    error = 'File not found in R2 - marked as deleted'
WHERE id IN (
    'dc105c70-cbba-42f0-a799-8a41617558a3',
    '11c464d0-baf1-413d-bbad-fb0d766de557',
    -- ... other IDs
);
```

**Option C: Investigate Further** (before deciding)
```bash
# 1. Check if ANY documents in this dataset succeeded
python scripts/check_document.py <any_doc_id_from_dataset>

# 2. Check R2 bucket directly for ANY objects
aws s3 ls s3://bucket-name/3deed0c5-3a5a-43ec-ac54-9ac44044fd3b/ --recursive

# 3. Review document creation timestamps
# (Check if documents were created but uploads never happened)
```

#### 📊 **Fix Strategy Decision Matrix**

| Scenario | Fix Approach | Validation Status |
|----------|-------------|-------------------|
| **Objects exist with different key** | Use `--fix` flag | ✅ Would work |
| **Objects exist but wrong dataset prefix** | Manual `object_key` update | ⚠️ Requires investigation |
| **No objects exist (this case)** | Re-upload OR mark deleted | ❌ `--fix` won't work |
| **Objects deleted after upload** | Re-upload from source | ❌ `--fix` won't work |
| **Wrong bucket** | Update bucket config + re-upload | ❌ `--fix` won't work |

#### ✅ **Validated Learnings**

1. **The diagnostic script is working as designed**
   - It correctly identifies when objects don't exist
   - It properly tries multiple key variants
   - It provides accurate reporting

2. **The `--fix` flag has limitations**
   - Only works when objects exist but keys don't match
   - Cannot fix cases where objects are completely missing
   - Requires manual intervention for missing objects

3. **We need additional fix strategies**
   - Script should be enhanced to handle "no objects found" case
   - Could add option to mark documents as deleted
   - Could add option to generate re-upload URLs

#### 🎯 **What We Learned: Comprehensive Fix Strategy Needed**

The diagnostic results reveal we need a **multi-step validation and fix process**:

**Current State Analysis**:
- ✅ Database has document records
- ❌ R2 has no objects (files never uploaded or deleted)
- ❓ Milvus chunk status unknown (need to check)

**What We Should Check**:
1. **Does document exist in R2?** → Checked ✅ (Result: No)
2. **Does document have chunks in Milvus?** → Need to check
3. **If no R2 file AND no chunks** → Document is a null pointer, should be deleted
4. **If R2 file exists but no chunks** → Re-process document
5. **If R2 file exists AND chunks exist but no image links** → Run migration script

#### 📋 **Validated Fix Plan Decision Tree**

```
For each document with issues:

1. Check R2 for document file
   ├─ ✅ EXISTS → Go to step 2
   └─ ❌ NOT FOUND → Go to step 3

2. Check Milvus for chunks
   ├─ ✅ HAS CHUNKS → Check image_r2_key
   │   ├─ ✅ HAS IMAGE LINKS → Document is healthy ✅
   │   └─ ❌ MISSING IMAGE LINKS → Run migrate_image_r2_keys.py
   └─ ❌ NO CHUNKS → Re-process document (trigger Modal processing)

3. Document file not in R2
   ├─ Check Milvus for chunks
   │   ├─ ✅ HAS CHUNKS → Orphaned chunks (file deleted but chunks remain)
   │   │   └─ Action: Delete chunks OR mark document as deleted
   │   └─ ❌ NO CHUNKS → Null pointer (document never processed)
   │       └─ Action: Delete document record (cleanup)
   └─ Check if source file available
       ├─ ✅ AVAILABLE → Re-upload and process
       └─ ❌ NOT AVAILABLE → Mark as deleted
```

#### ✅ **Plan Validation: What the Results Tell Us**

**For These 20 Documents**:
1. ✅ **Diagnostic script correctly identified the problem** - No R2 objects found
2. ⚠️ **We need to check Milvus** - Do these documents have chunks?
3. ⚠️ **We need to check source availability** - Can we re-upload?
4. ✅ **Our fix plan needs enhancement** - Current `--fix` only handles key mismatches

**What's Missing from Current Plan**:
- ❌ No check for Milvus chunks when R2 file is missing
- ❌ No automatic deletion of null pointer documents
- ❌ No re-processing trigger for documents with files but no chunks
- ❌ No distinction between "never uploaded" vs "deleted after upload"

#### 🔧 **Enhanced Fix Plan (Validated by Results)**

**Step 1: Comprehensive Document Health Check**
```python
# For each document:
1. Check R2: Does object_key exist?
2. Check Milvus: Does document have chunks?
3. Check chunks: Do chunks have image_r2_key?
4. Determine state and apply appropriate fix
```

**Step 2: Apply Fix Based on State**

| R2 File | Milvus Chunks | Image Links | State | Fix Action |
|---------|---------------|-------------|-------|------------|
| ✅ | ✅ | ✅ | Healthy | None needed |
| ✅ | ✅ | ❌ | Missing links | Run `migrate_image_r2_keys.py` |
| ✅ | ❌ | N/A | Not processed | Re-process document |
| ❌ | ✅ | N/A | Orphaned chunks | Delete chunks OR mark deleted |
| ❌ | ❌ | N/A | Null pointer | Delete document record |

**Step 3: Automated Actions**
- **Null pointers** (no R2, no chunks) → Auto-delete document record
- **Orphaned chunks** (no R2, has chunks) → Warn + offer to delete chunks
- **Missing links** (has R2, has chunks, no links) → Run migration script
- **Not processed** (has R2, no chunks) → Trigger re-processing

#### 🔧 **Recommended Next Steps**

**Immediate**:
1. **Verify dataset status**: Check if ANY documents in this dataset succeeded
   ```sql
   SELECT status, COUNT(*)
   FROM documents
   WHERE dataset_id = '3deed0c5-3a5a-43ec-ac54-9ac44044fd3b'
   GROUP BY status;
   ```

2. **Check R2 directly**: Verify if objects exist anywhere in R2 for this dataset
   ```bash
   # Check entire bucket for this dataset ID
   aws s3 ls s3://bucket-name/ --recursive | grep 3deed0c5-3a5a-43ec-ac54-9ac44044fd3b
   ```

3. **Review one document in detail**: Understand the upload timeline
   ```bash
   python scripts/check_document.py dc105c70-cbba-42f0-a799-8a41617558a3
   ```

**If files are truly missing**:
- **Option 1**: Mark documents as deleted (if source unavailable)
- **Option 2**: Re-upload from source (if source available)
- **Option 3**: Investigate why bulk upload failed and prevent recurrence

**Script Enhancement Opportunity**:
Consider adding to `diagnose_missing_object_keys.py`:
- `--mark-deleted`: Mark documents as deleted when no objects found
- `--generate-reupload-urls`: Generate new upload URLs for missing files
- `--check-other-buckets`: Check alternative R2 buckets/locations

---

## Related Documentation

- `OCR_IMAGE_RETRIEVAL_IMPLEMENTATION.md` - Details on image extraction and storage
- `R2_WEBHOOK_START_HERE.md` - R2 webhook handling
- `DOCUMENT_PIPELINE_INVESTIGATION_INDEX.md` - Document processing pipeline details
- `FIXES_2025-11-12.md` - Recent fixes for image processing issues

