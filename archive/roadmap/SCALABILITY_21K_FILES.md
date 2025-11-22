# Processing 21,000 Files - Critical Bottlenecks & Fixes

## Executive Summary

Your system will **definitely break** at 21,000 files due to:
1. ⚠️ **Modal: Only 10 concurrent processors** → 210+ hours to complete
2. ⚠️ **Queue: Pulling 10 messages at a time** → 3 hours just to queue
3. ⚠️ **Database: No connection pool** → Exhausted at ~5 concurrent ops
4. ⚠️ **Credits: Per-document transactions with locks** → Lock contention nightmare
5. ⚠️ **OpenAI: No rate limiting** → 429 errors at ~50 concurrent docs

**Bottom line:** Without fixes, 21k files will take **9+ days** and crash multiple times.
**With fixes:** Can complete in **~14 hours** reliably.

---

## CRITICAL - Fix These First (30 minutes total)

### 1. Modal Concurrency Limit ⚠️ BREAKS IMMEDIATELY

**Current:** 10 concurrent document processors
**Result:** 21,000 files ÷ 10 = 2,100 serial batches × 6 min = **210 hours (9 days)**

**File:** [modal/modal_app.py:441](modal/modal_app.py#L441)
```python
# CURRENT
max_containers=10,  # ← CRITICAL BOTTLENECK

# FIX
max_containers=100,  # 10x throughput, completes in ~21 hours
```

**Why 100?**
- Modal supports up to 1000 containers
- Each container uses 8GB RAM (~$0.50/hour)
- 100 containers = $50/hour during processing
- Cost for 21k files: ~$1,050 (21 hours × $50)

---

### 2. Queue Consumer Batch Size ⚠️ BREAKS AT UPLOAD

**Current:** Pulling 10 messages every 5 seconds = 120 messages/min
**Result:** 21,000 ÷ 120 = **175 minutes (3 hours) just to queue files!**

**File:** [fastapi/services/queue_consumer.py:151](fastapi/services/queue_consumer.py#L151)
```python
# CURRENT
messages = await consumer.pull_messages(batch_size=10)

# FIX
messages = await consumer.pull_messages(batch_size=100)  # Cloudflare max
```

**Additional:** Make processing parallel:
```python
# CURRENT
for message in messages:
    await process_message(message)  # Serial!

# FIX
await asyncio.gather(*[
    process_message(msg) for msg in messages
], return_exceptions=True)  # Parallel!
```

---

### 3. Database Connection Pool ⚠️ BREAKS AT 5-10 CONCURRENT

**Current:** Default PostgreSQL pool = 5 connections
**Result:** Connection exhausted errors at ~10 concurrent document completions

**File:** [fastapi/database/sql_store.py:116-119](fastapi/database/sql_store.py#L116-L119)
```python
# CURRENT
_engine = create_engine(
    settings.DATABASE_URL,
    echo=False,
    # No pool configuration!
)

# FIX
_engine = create_engine(
    settings.DATABASE_URL,
    echo=False,
    pool_size=50,              # Base pool
    max_overflow=100,          # Burst to 150 total
    pool_pre_ping=True,        # Verify connections
    pool_recycle=3600,         # Recycle after 1 hour
    connect_args={
        "connect_timeout": 10,
        "keepalives": 1,
        "keepalives_idle": 30,
        "keepalives_interval": 10,
        "keepalives_count": 5,
    }
)
```

---

### 4. Credit System - Batch Deduction ⚠️ LOCK CONTENTION

**Current:** 21,000 separate `SELECT FOR UPDATE` transactions
**Result:** Lock queue buildup, timeouts, failed transactions

**Problem Location:** [fastapi/services/credit_service.py:155](fastapi/services/credit_service.py#L155)
```python
user_credits = self.repo.get_user_credits_for_update(user_id)  # Row lock!
```

**Fix:** Batch deduct credits upfront, refund failures later

**New Endpoint:** `POST /api/v1/credits/deduct-batch`
```python
def deduct_batch_credits(
    user_id: str,
    document_count: int,
    cost_per_document: int = 10
):
    """Deduct credits for entire batch upfront"""
    total_cost = document_count * cost_per_document

    # Single transaction for entire batch
    with session.begin():
        user_credits = repo.get_user_credits_for_update(user_id)
        if user_credits.balance < total_cost:
            raise InsufficientCreditsError()

        # Create single transaction for batch
        transaction = CreditTransaction(
            user_id=user_id,
            amount=-total_cost,
            type="batch_deduction",
            description=f"Batch processing {document_count} documents",
            metadata={"document_count": document_count}
        )
        session.add(transaction)
        user_credits.balance -= total_cost
        session.commit()

    return transaction.id

def refund_failed_documents(
    transaction_id: str,
    failed_count: int,
    cost_per_document: int = 10
):
    """Refund credits for failed documents"""
    refund_amount = failed_count * cost_per_document
    # Simple credit increase, no locks needed
```

**Usage in Upload Flow:**
```python
# At dataset upload time
transaction_id = credit_service.deduct_batch_credits(
    user_id=user.id,
    document_count=len(files),
    cost_per_document=10
)

# Store transaction_id in dataset metadata
dataset.credit_transaction_id = transaction_id

# Later, when documents fail
if document.status == "failed":
    failed_count += 1

if failed_count > 0:
    credit_service.refund_failed_documents(
        transaction_id=dataset.credit_transaction_id,
        failed_count=failed_count
    )
```

---

## HIGH PRIORITY - Fix Next (1-2 hours total)

### 5. OpenAI Rate Limiting ⚠️ BREAKS AT ~50 CONCURRENT

**Current:** No rate limiting, no retry logic
**Result:** 429 errors from OpenAI at scale

**File:** [modal/services/embeddings.py](modal/services/embeddings.py)
```python
# ADD rate limiter
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type
)
import openai

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, min=4, max=60),
    retry=retry_if_exception_type(openai.RateLimitError)
)
def generate_embeddings_with_retry(texts: list[str]) -> list[list[float]]:
    """Generate embeddings with exponential backoff retry"""
    # ... existing code ...
```

**Install:** `pip install tenacity`

---

### 6. Dataset Status Update Contention

**Current:** Updates dataset status after EVERY document completion
**Result:** Table lock contention with 21k updates

**File:** [modal/modal_app.py:49-112](modal/modal_app.py#L49-L112)
```python
# CURRENT
def _update_dataset_status(session, dataset_id: str):
    # Called 21,000 times!
    session.execute(text("UPDATE datasets SET status = ..."))

# FIX - Add debouncing
import time
_last_status_update = {}

def _update_dataset_status_debounced(session, dataset_id: str):
    """Update dataset status, but not more than once per 10 seconds"""
    now = time.time()
    last_update = _last_status_update.get(dataset_id, 0)

    if now - last_update < 10:  # Skip if updated in last 10 seconds
        return

    _last_status_update[dataset_id] = now
    _update_dataset_status(session, dataset_id)
```

---

### 7. Checkpoint Size Limits

**Current:** No size limit on checkpoints
**Result:** Out of memory errors on documents with >500 images

**File:** [modal/services/checkpoint_manager.py:70-93](modal/services/checkpoint_manager.py#L70-L93)
```python
async def save_checkpoint(self, ...):
    json_data = json.dumps(data, default=str)
    compressed_data = gzip.compress(json_data.encode('utf-8'))

    # ADD size check
    checkpoint_size_mb = len(compressed_data) / (1024 * 1024)
    if checkpoint_size_mb > 100:  # Skip if >100MB
        print(f"⚠️  Checkpoint too large ({checkpoint_size_mb:.1f}MB), skipping")
        return None

    # ... rest of code ...
```

---

## MEDIUM PRIORITY - Optimize Later

### 8. Reduce Modal Memory

**Current:** 8GB per container
**Fix:** Reduce to 4GB for most documents

```python
# Use adaptive memory based on file size
memory = 4096 if file_size < 50_000_000 else 8192  # 50MB threshold
```

### 9. Visibility Timeout

**Current:** 30 minutes
**Fix:** Increase to 2 hours to match Modal timeout

[fastapi/services/queue_consumer.py:47](fastapi/services/queue_consumer.py#L47)
```python
visibility_timeout: int = 7200  # 2 hours
```

### 10. Image Worker Limit

**Current:** 50 image processing workers
**Fix:** Increase to 200 for large batches

[modal/modal_app.py:197](modal/modal_app.py#L197)
```python
max_containers=200,  # Up from 50
```

---

## EXPECTED PERFORMANCE

| Metric | Before Fixes | After Critical Fixes | After All Fixes |
|--------|-------------|---------------------|-----------------|
| **Queue Time** | 3 hours | 15 minutes | 10 minutes |
| **Processing Time** | 210 hours | 21 hours | 14 hours |
| **Concurrent Docs** | 10 | 100 | 100 |
| **Failures** | High (DB, API limits) | Low | Very Low |
| **Total Time** | **9 days** | **21 hours** | **14 hours** |
| **Cost (Modal)** | ~$1,050 | ~$1,050 | ~$700 |

---

## TESTING PLAN

### 1. Smoke Test (100 files)
```bash
# Upload 100 files
# Monitor:
- Queue consumer logs
- Modal container count (should see 100 active)
- Database connections (check pg_stat_activity)
- OpenAI rate limit errors
```

### 2. Load Test (1,000 files)
```bash
# Upload 1,000 files
# Monitor:
- Credit deduction (should be 1 transaction, not 1000)
- Database lock waits (check pg_locks)
- Modal memory usage
- Checkpoint sizes
```

### 3. Full Test (21,000 files)
```bash
# Upload 21,000 files
# Monitor:
- Total completion time (target: <24 hours)
- Failure rate (target: <1%)
- Cost tracking
- Error patterns
```

---

## MONITORING QUERIES

### Database Connections
```sql
SELECT count(*), state
FROM pg_stat_activity
WHERE datname = 'fltr_auth'
GROUP BY state;
```

### Lock Contention
```sql
SELECT * FROM pg_locks
WHERE NOT granted
ORDER BY relation;
```

### Processing Progress
```sql
SELECT
    d.id as dataset_id,
    d.status as dataset_status,
    COUNT(*) as total_docs,
    SUM(CASE WHEN doc.status = 'ready' THEN 1 ELSE 0 END) as completed,
    SUM(CASE WHEN doc.status = 'processing' THEN 1 ELSE 0 END) as processing,
    SUM(CASE WHEN doc.status = 'failed' THEN 1 ELSE 0 END) as failed
FROM datasets d
JOIN documents doc ON doc.dataset_id = d.id
GROUP BY d.id, d.status;
```

---

## FILES TO MODIFY

**Critical (30 min):**
1. `modal/modal_app.py` - Line 441 (max_containers)
2. `fastapi/services/queue_consumer.py` - Line 151 (batch_size) and parallel processing
3. `fastapi/database/sql_store.py` - Line 116 (connection pool)
4. `fastapi/services/credit_service.py` - New batch deduction methods

**High Priority (1-2 hours):**
5. `modal/services/embeddings.py` - Add retry logic
6. `modal/modal_app.py` - Line 49 (debounce status updates)
7. `modal/services/checkpoint_manager.py` - Line 70 (size limits)

**Medium Priority:**
8. Memory optimization
9. Timeout adjustments
10. Worker limits
