# Document Processing Cleanup Plan

## Executive Summary

**Goal**: Remove duplicate document processing code from FastAPI now that Modal handles all document processing.

**Status**: Document processing has been fully migrated to Modal. The FastAPI repo contains duplicate Celery-based document processing code that is no longer needed.

**Scope**: Remove ONLY document processing code. Keep Celery infrastructure for other background jobs.

**Impact**:
- Remove ~660 lines of duplicate document processing code
- Keep Celery/Redis for other background tasks
- Simplify document processing flow (Modal only, no fallback)
- Reduce maintenance burden for document processing

**Risk**: Low - Modal has been tested and is working in production. Fallback to Celery is no longer needed for documents.

---

## Current State

### Duplicate Code (TO REMOVE)

1. **fastapi/services/document_processor.py** (433 lines)
   - Complete document processing pipeline
   - Download, parse, chunk, embed, store
   - Duplicates: modal/services/document_processor.py

2. **fastapi/tasks.py** (227 lines)
   - Celery task definitions for document processing
   - ProcessDocumentTask class
   - Wraps DocumentProcessor

### Code That Uses Celery (TO UPDATE)

1. **fastapi/services/dataset_service.py**
   - Lines 289-313: Modal/Celery fallback logic in retry_document()
   - Keep Modal, remove Celery fallback

2. **fastapi/services/queue_consumer.py**
   - Lines 233-279: Modal/Celery fallback logic
   - Keep Modal, remove Celery fallback

3. **fastapi/routers/webhooks.py**
   - Line 71: USE_MODAL_FOR_PROCESSING check
   - Simplify to always use Modal

4. **fastapi/services/jobs_service.py**
   - May have Celery references (need to check)

### Celery Infrastructure (TO KEEP)

**IMPORTANT**: Keep Celery for other background jobs!

**Keep these files**:
1. **fastapi/celery_app.py** - Celery configuration for other tasks
2. **fastapi/services/jobs_service.py** - Task status checking (may be used by other jobs)

**Keep all Celery dependencies** in requirements.txt:
- celery, redis, flower, billiard, kombu, vine, amqp
- All Celery-related packages

**Keep Celery configuration**:
- REDIS_URL, CELERY_* config
- Celery can still be used for non-document background tasks

### Configuration (TO UPDATE)

1. **fastapi/config.py**
   - Line 97: `USE_MODAL_FOR_PROCESSING: bool = False`
   - Change default to True (Modal is proven for documents)
   - Keep all Celery config for other tasks

2. **fastapi/.env.example**
   - Update documentation to clarify: Modal for documents, Celery for other jobs

---

## Removal Plan

### Phase 1: Update Code to Remove Fallback

#### Step 1.1: Update dataset_service.py

**File**: fastapi/services/dataset_service.py
**Lines**: 285-313 (retry_document method)

**Change**:
```python
# OLD (with fallback):
use_modal = settings.USE_MODAL_FOR_PROCESSING and modal_client.is_available()

if use_modal:
    # Modal code
    ...
    except Exception as modal_error:
        use_modal = False

if not use_modal:
    # Celery fallback
    from tasks import process_document
    task = process_document.delay(...)

# NEW (Modal only):
from services.modal_client import modal_client

print(f"🚀 [Retry] Using Modal for document {document_id}")
result = await modal_client.process_document(
    dataset_id=str(dataset_id),
    object_key=document.object_key,
    task_id=str(document.id)
)
print(f"✅ Retry queued to Modal: {result.get('call_id')}")
```

#### Step 1.2: Update queue_consumer.py

**File**: fastapi/services/queue_consumer.py
**Lines**: 233-279

**Change**:
```python
# OLD (with fallback):
use_modal = settings.USE_MODAL_FOR_PROCESSING and modal_client.is_available()

if use_modal:
    # Modal code
    ...

# Fallback to Celery

# NEW (Modal only):
from services.modal_client import modal_client

print(f"🚀 [Routing] Using Modal for {object_key}")
result = await modal_client.process_document(
    dataset_id=dataset_id,
    object_key=object_key,
    task_id=str(document.id)
)
print(f"✅ Queued Modal task: {result.get('call_id')}")
```

#### Step 1.3: Update webhooks.py

**File**: fastapi/routers/webhooks.py
**Line**: 71

**Change**:
```python
# OLD:
use_modal = settings.USE_MODAL_FOR_PROCESSING and modal_client.is_available()

# NEW (always Modal):
# Modal processing only
```

### Phase 2: Remove Duplicate Files

#### Step 2.1: Remove Document Processing Files

```bash
# Remove document processing tasks (ONLY if it contains ONLY document processing)
rm fastapi/tasks.py

# Remove duplicate document processor
rm fastapi/services/document_processor.py
```

**IMPORTANT - Verify before deleting**:
- Check if tasks.py has any non-document-processing tasks (other Celery tasks)
- If tasks.py has other tasks, KEEP IT and only remove the ProcessDocumentTask class
- Check if document_processor.py has any unique logic not in Modal version

#### Step 2.2: Keep Celery Infrastructure

**DO NOT DELETE**:
- `fastapi/celery_app.py` - Keep for other background jobs
- `fastapi/services/jobs_service.py` - Keep for task status checking
- Any other Celery task files (non-document processing)

### Phase 3: Update Configuration

#### Step 3.1: Update config.py

**File**: fastapi/config.py

**Change default**:
```python
# OLD:
USE_MODAL_FOR_PROCESSING: bool = False

# NEW (Modal is proven for documents):
USE_MODAL_FOR_PROCESSING: bool = True
```

**Keep all other Celery config**:
```python
REDIS_URL: str = "redis://localhost:6379/0"  # Keep for other Celery tasks
# Keep any other Celery-specific config
```

#### Step 3.2: Update .env.example

**File**: fastapi/.env.example

**Update documentation**:
```bash
# Document Processing (Modal only)
USE_MODAL_FOR_PROCESSING=true  # Use Modal for documents (recommended)
MODAL_WEBHOOK_URL=https://your-fastapi-instance.com/webhooks/modal/completion

# Background Jobs (Celery)
REDIS_URL=redis://localhost:6379/0  # For other background tasks
```

### Phase 4: Dependencies

#### Step 4.1: Keep All Dependencies

**File**: fastapi/requirements.txt

**NO CHANGES NEEDED** - Keep all Celery dependencies for other background jobs:
```
celery==5.4.0         # Keep - for other background tasks
redis==5.2.1          # Keep - for Celery and potentially caching
flower==2.0.1         # Keep - for monitoring Celery tasks
billiard==4.2.2       # Keep - Celery dependency
kombu==5.5.4          # Keep - Celery dependency
vine==5.1.0           # Keep - Celery dependency
amqp==5.3.1           # Keep - Celery dependency
# ... all other Celery-related packages
```

**Rationale**: Celery infrastructure remains for other background jobs (emails, cleanup tasks, etc.)

### Phase 5: Update Documentation

#### Step 5.1: Update DEPLOYMENT.md

**File**: fastapi/DEPLOYMENT.md

**Update to clarify**:
- Modal handles all document processing (no fallback)
- Celery/Redis still used for other background tasks
- USE_MODAL_FOR_PROCESSING=true is recommended default
- Celery workers may still be needed for non-document tasks

#### Step 5.2: Update README (if exists)

Remove Celery setup instructions, worker commands, etc.

#### Step 5.3: Create MIGRATION_FROM_CELERY.md (optional)

Document the migration for historical reference.

---

## Risk Assessment

### Low Risk ✅

1. **Modal is production-tested**: Already running in production successfully
2. **No data loss**: Document processing state stored in database, not Celery
3. **Reversible**: Can restore from git if needed
4. **No downtime**: Changes don't affect running services

### Medium Risk ⚠️

1. **In-flight Celery tasks**: If any Celery workers are still running
   - **Mitigation**: Stop Celery workers before deploying
   - **Check**: Verify no pending Celery jobs in Redis

2. **Hidden Celery usage**: Code paths we didn't identify
   - **Mitigation**: Grep entire codebase for Celery imports
   - **Check**: Run full test suite

### How to Mitigate

#### Before Removal:

```bash
# 1. Check for any pending Celery jobs
redis-cli -h your-redis-host
> KEYS celery*
> QUIT

# 2. Stop all Celery workers
pkill -f celery

# 3. Grep for all Celery usage
cd fastapi
grep -r "from tasks import" .
grep -r "from celery" .
grep -r "\.delay(" .
grep -r "celery_app" .

# 4. Check if Redis is used elsewhere
grep -r "redis" . | grep -v celery | grep -v requirements
```

#### After Removal:

```bash
# 1. Run full test suite
pytest

# 2. Check imports
python -c "from main import app"  # Should not error

# 3. Test document processing
# Upload a test document and verify it processes
```

---

## Verification Checklist

### Before Starting

- [ ] Confirm Modal is working in production
- [ ] Verify no pending Celery jobs (check Redis)
- [ ] Stop all Celery workers
- [ ] Create backup branch: `git checkout -b backup/before-celery-removal`

### During Removal

- [ ] Update dataset_service.py (remove fallback)
- [ ] Update queue_consumer.py (remove fallback)
- [ ] Update webhooks.py (remove fallback)
- [ ] Update jobs_service.py (if needed)
- [ ] Remove celery_app.py
- [ ] Remove tasks.py
- [ ] Remove services/document_processor.py (check for unique logic first)
- [ ] Update config.py (remove Celery config)
- [ ] Update .env.example (remove Celery vars)
- [ ] Update requirements.txt (remove Celery deps)
- [ ] Update DEPLOYMENT.md

### After Removal

- [ ] Run: `pip install -r requirements.txt` (verify no errors)
- [ ] Run: `python -c "from main import app"` (verify no import errors)
- [ ] Run: `pytest` (verify all tests pass)
- [ ] Test document upload and processing
- [ ] Check logs for any Celery-related errors
- [ ] Deploy to staging environment
- [ ] Monitor for 24 hours
- [ ] Deploy to production

---

## Why Keep Celery Infrastructure?

**Celery is still useful for**:
- Future background tasks (emails, reports, cleanup jobs)
- Scheduled tasks (cron-like jobs)
- Non-document processing workloads
- Tasks that don't need Modal's specialized environment

**What we're removing**:
- Only the duplicate document processing code
- Only the fallback logic that switches from Modal to Celery for documents

**Result**:
- Modal handles ALL document processing (proven, reliable)
- Celery available for other background jobs
- Simpler document processing flow
- Less duplicate code

---

## Timeline

**Estimated time**: 2-4 hours

1. **Code updates** (1 hour):
   - Update 3-4 files to remove fallback logic
   - Update config files

2. **File removal** (15 minutes):
   - Delete 3 files
   - Update requirements.txt

3. **Testing** (1-2 hours):
   - Run tests
   - Manual testing
   - Deploy to staging

4. **Documentation** (30 minutes):
   - Update DEPLOYMENT.md
   - Update README

5. **Deployment** (varies):
   - Deploy to production
   - Monitor

---

## Next Steps

1. **Review this plan** with team
2. **Create backup branch**: `git checkout -b backup/before-celery-removal`
3. **Start with Phase 1**: Update code to remove fallback logic
4. **Test thoroughly** before removing files
5. **Deploy to staging** first
6. **Monitor for 24 hours** before production deployment

---

## Questions to Answer

1. **Does tasks.py contain ONLY document processing?**
   - Action: Read tasks.py fully before deleting
   - If it has other tasks: Keep file, remove only ProcessDocumentTask
   - If it's ONLY documents: Delete entire file

2. **Are there any other Celery tasks planned?** (future background jobs)
   - This is why we're keeping Celery infrastructure
   - Answer: Yes, may be useful for future tasks

3. **Should we remove document processing fallback?**
   - Decision: YES - Modal is proven, fallback adds complexity

4. **When should we do this?** (timing)
   - Recommendation: During low-traffic period
   - Or: After next major release

---

## Files Summary

### Files to DELETE:
1. `fastapi/tasks.py` (227 lines) - ONLY if it contains ONLY document processing tasks
2. `fastapi/services/document_processor.py` (433 lines)

**Total**: ~660 lines removed

### Files to KEEP (Celery Infrastructure):
1. `fastapi/celery_app.py` - For other background jobs
2. `fastapi/services/jobs_service.py` - Task status checking
3. All Celery dependencies in requirements.txt

### Files to UPDATE:
1. `fastapi/services/dataset_service.py` (remove Celery fallback for documents)
2. `fastapi/services/queue_consumer.py` (remove Celery fallback for documents)
3. `fastapi/routers/webhooks.py` (simplify to Modal only for documents)
4. `fastapi/config.py` (change USE_MODAL_FOR_PROCESSING default to True)
5. `fastapi/.env.example` (update documentation)
6. `fastapi/DEPLOYMENT.md` (clarify Modal for docs, Celery for other tasks)

### Files to CHECK:
1. `fastapi/services/jobs_service.py` (may have Celery references)
2. Any worker start scripts
3. Any Dockerfiles that start Celery workers

---

## Rollback Plan

If something goes wrong:

```bash
# 1. Revert all changes
git reset --hard backup/before-celery-removal

# 2. Reinstall dependencies
pip install -r requirements.txt

# 3. Restart services
# (deploy previous version)

# 4. Investigate what went wrong
# (check logs, errors)
```

**Backup strategy**: Keep backup branch for at least 30 days after removal.

---

## Success Criteria

✅ **Successful removal means**:

1. Document processing works (via Modal only)
2. No Celery imports in codebase
3. No Celery dependencies in requirements.txt
4. All tests pass
5. Production deployment successful
6. No errors in logs for 24 hours
7. Reduced Docker image size
8. Simplified deployment process

---

## Contact

If you have questions or concerns about this removal:

- Review this document
- Check Modal deployment status
- Test thoroughly in staging
- Monitor production closely after deployment

**Remember**: This is a targeted cleanup. We're removing duplicate document processing code while keeping Celery for other background jobs. Modal is proven for documents, and Celery remains available for other tasks.
