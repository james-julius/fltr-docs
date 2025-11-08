# Modal Migration Plan: Document Processing

## Overview

This document outlines the strategy for migrating heavy document processing tasks from Celery workers on DigitalOcean to Modal.com serverless functions, while keeping Celery for lightweight background jobs.

## Current Problem

- 3x Celery workers maxing out $50/month DigitalOcean CPU droplet
- Document processing (PDF parsing with Docling) is CPU/memory intensive
- Fixed cost regardless of usage
- Limited scalability

## Goals

1. **Reduce infrastructure costs** - Pay only for actual compute time
2. **Improve scalability** - Auto-scale based on demand
3. **Maintain reliability** - Keep existing Celery for lightweight tasks
4. **Enable GPU workloads** - Future-proof for ML/AI features

## Architecture: Hybrid Approach

### Celery (DigitalOcean) - Lightweight Background Jobs
- Email notifications
- Webhook deliveries
- Database cleanup/maintenance
- Analytics updates
- Job status updates
- Quick API integrations

### Modal (Serverless) - Heavy Compute Tasks
- **Document processing** (PDF parsing with Docling) ⭐ Primary migration target
- Embedding generation (with optional GPU)
- Large file transformations
- Future ML inference tasks
- Batch processing operations

## Migration Phases

### Phase 1: Setup & Configuration (Week 1)

#### 1.1 Modal Account Setup
- [ ] Create Modal account at modal.com
- [ ] Install Modal CLI: `pip install modal`
- [ ] Authenticate: `modal token new`
- [ ] Create Modal secrets:
  ```bash
  modal secret create fltr-r2-credentials \
    R2_ACCOUNT_ID=xxx \
    R2_ACCESS_KEY_ID=xxx \
    R2_SECRET_ACCESS_KEY=xxx \
    R2_BUCKET_NAME=xxx

  modal secret create fltr-openai-api-key \
    OPENAI_API_KEY=xxx

  modal secret create fltr-postgres-url \
    POSTGRES_URL=postgresql://user:pass@host:5432/db

  modal secret create fltr-milvus-url \
    MILVUS_URI=http://your-milvus:19530
  ```

#### 1.2 Add Modal to Dependencies
```bash
# Add to requirements.txt
modal>=0.63.0
```

#### 1.3 Create Modal App Structure
- [ ] Create `modal_app.py` with document processing function
- [ ] Test basic Modal function deployment
- [ ] Verify secrets are accessible

### Phase 2: Refactor Document Processor (Week 1-2)

#### 2.1 Make DocumentProcessor Modal-Compatible

**Current Issues:**
- DocumentProcessor imports FastAPI dependencies
- Relies on local file paths
- Tightly coupled to Celery task context

**Required Changes:**

```python
# services/document_processor.py

class DocumentProcessor:
    """Make this class environment-agnostic"""

    def __init__(self, dataset_id: str, object_key: str, session=None):
        self.dataset_id = dataset_id
        self.object_key = object_key
        self.session = session  # Accept session from outside

        # Make R2 client initialization lazy
        self._r2_client = None

    @property
    def r2_client(self):
        """Lazy initialization works in both environments"""
        if self._r2_client is None:
            from config import settings  # Import only when needed
            self._r2_client = boto3.client(...)
        return self._r2_client
```

#### 2.2 Extract Core Processing Functions

Create `services/document_processing_core.py`:
```python
"""
Pure functions for document processing
No FastAPI, Celery, or framework dependencies
Can run in Modal or Celery
"""

async def download_from_r2(object_key: str, r2_config: dict) -> bytes:
    """Framework-agnostic R2 download"""
    pass

async def parse_document_with_validation(
    file_bytes: bytes,
    filename: str
) -> Dict[str, Any]:
    """PDF validation + Docling parsing"""
    pass

async def chunk_document(document: Dict, dataset_id: str) -> List[Dict]:
    """Pure chunking logic"""
    pass

async def generate_embeddings_batch(texts: List[str], api_key: str) -> List[List[float]]:
    """Pure embedding generation"""
    pass
```

#### 2.3 Copy Supporting Files to Modal

Create `modal_support/` directory:
```
fastapi/
├── modal_app.py
├── modal_support/
│   ├── __init__.py
│   ├── pdf_validator.py       # Copy from services/
│   ├── processing_core.py     # New extracted functions
│   └── models.py              # Minimal SQLModel definitions
```

### Phase 3: Modal Integration (Week 2)

#### 3.1 Update Modal Function

Refactor `modal_app.py` to use extracted core functions:

```python
@app.function(
    image=image,
    cpu=2.0,
    memory=8192,
    timeout=3600,
    volumes={"/cache": volume},
    secrets=[...],
)
async def process_document_modal(
    dataset_id: str,
    object_key: str,
    task_id: str  # For tracking
) -> Dict[str, Any]:
    """Use extracted core functions"""
    from modal_support.processing_core import (
        download_from_r2,
        parse_document_with_validation,
        chunk_document,
        generate_embeddings_batch,
    )

    # All pure functions, no framework coupling
    file_bytes = await download_from_r2(object_key, r2_config)
    document = await parse_document_with_validation(file_bytes, filename)
    chunks = await chunk_document(document, dataset_id)
    embeddings = await generate_embeddings_batch(texts, api_key)

    # Store and update database
    await store_results(...)

    return {"status": "completed", "chunks": len(chunks)}
```

#### 3.2 Deploy Modal App

```bash
# Deploy to Modal
modal deploy modal_app.py

# This creates a webhook URL like:
# https://yourapp--webhook-process-document.modal.run
```

### Phase 4: Integration with Existing System (Week 2-3)

#### 4.1 Create Modal Client Wrapper

```python
# services/modal_client.py

import httpx
from typing import Optional

class ModalClient:
    """Client for triggering Modal functions"""

    def __init__(self):
        self.webhook_url = os.getenv("MODAL_WEBHOOK_URL")
        self.enabled = os.getenv("MODAL_ENABLED", "false").lower() == "true"

    async def process_document(
        self,
        dataset_id: str,
        object_key: str,
        task_id: str
    ) -> dict:
        """Trigger Modal document processing"""
        if not self.enabled:
            raise ValueError("Modal is not enabled")

        async with httpx.AsyncClient(timeout=10.0) as client:
            response = await client.post(
                self.webhook_url,
                json={
                    "dataset_id": dataset_id,
                    "object_key": object_key,
                    "task_id": task_id,
                }
            )
            response.raise_for_status()
            return response.json()
```

#### 4.2 Update Webhook Handler

Modify the R2 upload webhook to trigger Modal:

```python
# routers/webhooks.py

@router.post("/r2-upload-complete")
async def handle_r2_upload_complete(request: Request):
    """Handle R2 upload completion - trigger Modal or Celery"""

    data = await request.json()
    object_key = data.get("object_key")
    dataset_id = extract_dataset_id(object_key)

    # Create processing job record
    task_id = str(uuid.uuid4())
    job = ProcessingJob(
        id=uuid.uuid4(),
        task_id=task_id,
        dataset_id=dataset_id,
        status="pending",
        processor="modal",  # Track which system processed it
    )
    session.add(job)
    session.commit()

    # Use environment variable to toggle between Modal and Celery
    use_modal = os.getenv("USE_MODAL_FOR_PROCESSING", "false").lower() == "true"

    if use_modal:
        # Trigger Modal
        try:
            modal_client = ModalClient()
            result = await modal_client.process_document(
                dataset_id=dataset_id,
                object_key=object_key,
                task_id=task_id,
            )
            return {"status": "queued_modal", "call_id": result.get("call_id")}
        except Exception as e:
            logger.error(f"Modal trigger failed: {e}, falling back to Celery")
            # Fallback to Celery if Modal fails
            use_modal = False

    if not use_modal:
        # Use existing Celery
        from tasks import process_document
        task = process_document.delay(str(dataset_id), object_key)
        return {"status": "queued_celery", "task_id": task.id}
```

#### 4.3 Add Feature Flag Support

```python
# config.py

class Settings(BaseSettings):
    # ... existing settings ...

    # Modal configuration
    MODAL_ENABLED: bool = False
    MODAL_WEBHOOK_URL: Optional[str] = None
    USE_MODAL_FOR_PROCESSING: bool = False
```

```bash
# .env
MODAL_ENABLED=true
MODAL_WEBHOOK_URL=https://yourapp--webhook-process-document.modal.run
USE_MODAL_FOR_PROCESSING=true
```

### Phase 5: Testing & Validation (Week 3)

#### 5.1 Test in Development

- [ ] Process sample PDFs through Modal
- [ ] Verify PDF validation works
- [ ] Check database updates are correct
- [ ] Confirm Milvus vectors are stored
- [ ] Test error handling and retries

#### 5.2 Add Monitoring

```python
# Add to modal_app.py

from modal import CloudBucketMount

# Store logs in cloud bucket for analysis
logs_mount = CloudBucketMount(
    "s3://your-logs-bucket",
    secret=modal.Secret.from_name("aws-credentials"),
)

@app.function(
    cloud_bucket_mounts={"/logs": logs_mount}
)
async def process_document_modal(...):
    # Log to file for persistence
    with open(f"/logs/{task_id}.log", "w") as f:
        f.write(f"Started: {datetime.now()}\n")
        # ... processing ...
        f.write(f"Completed: {datetime.now()}\n")
```

#### 5.3 Create Comparison Dashboard

Track metrics for Celery vs Modal:
- Processing time
- Success rate
- Cost per document
- Error types

### Phase 6: Gradual Rollout (Week 4)

#### 6.1 Canary Deployment (10%)

```python
# Randomly send 10% of jobs to Modal
import random

if random.random() < 0.1:  # 10% chance
    use_modal = True
else:
    use_modal = False
```

#### 6.2 Monitor & Adjust

Watch for:
- Cold start times (first run slower)
- Error rates
- Cost tracking
- User complaints

#### 6.3 Increase Rollout

- Week 4: 10% Modal
- Week 5: 25% Modal
- Week 6: 50% Modal
- Week 7: 100% Modal

### Phase 7: Cleanup (Week 5+)

#### 7.1 Scale Down Celery Workers

Once 100% on Modal:
- Reduce DigitalOcean droplet to smaller size
- Keep 1 lightweight Celery worker for non-processing tasks
- Remove document processing from Celery tasks

#### 7.2 Remove Dead Code

- Archive old Celery document processing task
- Clean up unused imports
- Update documentation

## Cost Analysis

### Current Setup (DigitalOcean)
- **Fixed Cost**: $50/month
- **Capacity**: 3 workers, ~100 docs/day max
- **Cost per doc**: $0.017 (assuming 3000 docs/month)

### Projected Modal Costs

**Assumptions:**
- 2 vCPU, 8GB RAM
- Average processing time: 30 seconds per document
- 3000 documents/month

**Calculation:**
- CPU cost: $0.0001/CPU-second × 2 CPUs × 30 seconds = $0.006
- Memory: Included with CPU
- **Cost per doc**: ~$0.006
- **Monthly cost (3000 docs)**: ~$18/month

**Savings**: $32/month (64% reduction)

**Scalability**: Can handle bursts up to 1000 concurrent documents without infrastructure changes

## Rollback Plan

If issues arise:

1. **Immediate Rollback**: Set `USE_MODAL_FOR_PROCESSING=false` in environment
2. **Celery workers still running**: No downtime
3. **Database compatible**: Both systems use same schema
4. **Keep Modal deployment**: Can switch back easily

## Success Metrics

- [ ] 50% cost reduction
- [ ] < 5% error rate increase
- [ ] Average processing time < 2 minutes
- [ ] No customer complaints
- [ ] Can handle 10x traffic spikes

## Files to Create/Modify

### New Files
- [x] `modal_app.py` - Modal application
- [ ] `modal_support/processing_core.py` - Pure processing functions
- [ ] `modal_support/pdf_validator.py` - Copy of validator
- [ ] `services/modal_client.py` - Modal API client
- [ ] `tests/test_modal_integration.py` - Modal tests

### Modified Files
- [ ] `routers/webhooks.py` - Add Modal trigger
- [ ] `services/document_processor.py` - Make environment-agnostic
- [ ] `config.py` - Add Modal settings
- [ ] `requirements.txt` - Add Modal SDK
- [ ] `.env.example` - Document Modal variables

## Timeline

| Week | Tasks | Deliverable |
|------|-------|-------------|
| 1 | Setup Modal, refactor processor | Modal account + basic function |
| 2 | Integration, webhook updates | Feature flag ready |
| 3 | Testing, monitoring setup | Tested in dev environment |
| 4 | Canary rollout (10% → 50%) | Production validation |
| 5 | Full rollout, Celery scale-down | 100% on Modal |

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Modal cold starts slow | High | Pre-warm containers, cache models |
| Modal downtime | High | Keep Celery fallback, monitoring alerts |
| Cost overruns | Medium | Set budget alerts, monitor per-doc costs |
| Database connection limits | Medium | Use connection pooling, manage sessions |
| PDF validation breaks | High | Extensive testing, gradual rollout |

## Next Steps

1. ✅ Create this migration plan
2. ⏳ Set up Modal account and secrets
3. ⏳ Fix test failures (docling import issues)
4. ⏳ Refactor DocumentProcessor for environment-agnosticism
5. ⏳ Deploy basic Modal function
6. ⏳ Test with sample documents

---

**Document Version**: 1.0
**Last Updated**: 2025-01-XX
**Owner**: Engineering Team
**Review Date**: After Phase 3 completion
