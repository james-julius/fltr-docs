# Document Processing Pipeline

Complete implementation of Cloudflare Queue → FastAPI + Celery integration for asynchronous document processing.

## Architecture

```
R2 Upload → CF Queue → FastAPI Consumer → Celery Worker → Milvus + Postgres
```

### Flow

1. **Client uploads file** to R2 via presigned URL (`{datasetId}/{filename}`)
2. **R2 triggers** Cloudflare Queue event (object-create)
3. **FastAPI consumer** polls CF Queue, extracts dataset info
4. **Celery task queued** with dataset_id and object_key
5. **Celery worker** downloads, parses, chunks, embeds, stores

## Components

### 1. Celery App ([celery_app.py](celery_app.py:1))
- Broker: Redis
- Backend: Redis
- Task tracking, retries, time limits
- 3 workers with 4 concurrent tasks each

### 2. Queue Consumer ([services/queue_consumer.py](services/queue_consumer.py:1))
- Polls Cloudflare Queue every 5s when idle
- Batch size: 10 messages
- Visibility timeout: 30 minutes
- Auto-started in FastAPI lifespan

### 3. Processing Task ([tasks.py](tasks.py:1))
- Async Celery task with 3 retries
- Exponential backoff: 60s, 120s, 240s
- Updates dataset and job status
- Handles errors gracefully

### 4. Document Processor ([services/document_processor.py](services/document_processor.py:1))
- Downloads from R2
- Parses: JSON, TXT, CSV (PDF placeholder)
- Chunks with overlap (1000 chars, 200 overlap)
- Embeds via OpenAI text-embedding-3-small
- Stores in Milvus

### 5. Database Models ([models/processing.py](models/processing.py:1))
- `ProcessingJob`: Tracks Celery tasks
- Fields: task_id, status, error, retries
- Auto-created via SQLModel

### 6. Monitoring Endpoints ([routers/jobs.py](routers/jobs.py:1))
- `GET /api/v1/jobs/{task_id}` - Task status
- `GET /api/v1/jobs/dataset/{dataset_id}/status` - Dataset progress
- `GET /api/v1/jobs/` - List all jobs

## Setup

### Local Development (Docker Compose)

```bash
# Copy environment file
cp .env.example .env

# Update .env with your credentials:
# - Cloudflare R2 (R2_ACCOUNT_ID, R2_ACCESS_KEY_ID, R2_SECRET_ACCESS_KEY)
# - Cloudflare Queue (CF_ACCOUNT_ID, CF_QUEUE_NAME, CF_API_TOKEN)
# - OpenAI (OPENAI_API_KEY)

# Start all services
docker-compose up

# Services:
# - FastAPI: http://localhost:8000
# - Flower (Celery monitor): http://localhost:5555
# - PostgreSQL: localhost:5432
# - Redis: localhost:6379
# - Milvus: localhost:19530
```

### Manual Setup

```bash
# Install dependencies
pip install -r requirements.txt

# Start Redis
redis-server

# Start Celery worker
celery -A celery_app worker --loglevel=info --concurrency=4

# Start Flower (optional)
celery -A celery_app flower --port=5555

# Start FastAPI
uvicorn main:app --reload
```

## Cloudflare Setup

### 1. Create R2 Bucket
```bash
# Via dashboard: https://dash.cloudflare.com > R2
# Bucket name: fltr-datasets
# Enable event notifications
```

### 2. Create Queue
```bash
# Via dashboard: https://dash.cloudflare.com > Queues
# Queue name: fltr-document-processing
# Type: Standard
```

### 3. Connect R2 to Queue
```bash
# R2 Bucket Settings > Event Notifications
# Add notification:
#   - Event: object-create
#   - Target: Queue (fltr-document-processing)
#   - Prefix: (leave empty to catch all uploads)
```

### 4. Get API Token
```bash
# https://dash.cloudflare.com > Profile > API Tokens
# Create Token:
#   - Template: Edit Cloudflare Workers
#   - Permissions:
#     - Account > Workers Scripts > Edit
#     - Account > Queue > Read
#   - Account Resources: Include your account
```

## Usage

### Upload Flow

```python
# 1. Client creates dataset
response = await api.POST("/api/v1/datasets", {
    "body": {
        "name": "My Dataset",
        "files": [{"file_name": "doc.pdf"}]
    }
})
dataset_id = response.data["dataset"]["id"]
upload_url = response.data["upload_urls"][0]["upload_url"]

# 2. Client uploads to R2
await fetch(upload_url, {
    "method": "PUT",
    "body": file,
    "headers": {"Content-Type": "application/pdf"}
})

# 3. R2 triggers queue (automatic)
# 4. FastAPI consumer processes (automatic)
# 5. Celery worker processes (automatic)

# 6. Check status
status = await api.GET(f"/api/v1/jobs/dataset/{dataset_id}/status")
# {
#   "dataset": {"status": "processing", "document_count": 1},
#   "jobs": [{"status": "processing", "task_id": "..."}]
# }
```

### Manual Task Trigger (Development)

```python
from tasks import process_document

# Queue task manually
task = process_document.delay(
    dataset_id="550e8400-e29b-41d4-a716-446655440000",
    object_key="550e8400-e29b-41d4-a716-446655440000/document.pdf"
)

print(f"Task ID: {task.id}")
```

## Monitoring

### Flower Dashboard
- URL: http://localhost:5555
- Real-time task tracking
- Worker stats
- Retry history

### API Endpoints
```bash
# Get task status
curl http://localhost:8000/api/v1/jobs/{task_id}

# Get dataset progress
curl http://localhost:8000/api/v1/jobs/dataset/{dataset_id}/status

# List all jobs
curl http://localhost:8000/api/v1/jobs/?status=processing
```

### Database Queries
```sql
-- Check processing jobs
SELECT * FROM processing_jobs WHERE status = 'processing';

-- Check failed jobs
SELECT * FROM processing_jobs WHERE status = 'failed';

-- Dataset status
SELECT id, name, status, document_count
FROM datasets
WHERE status IN ('uploading', 'processing');
```

## Error Handling

### Retries
- Automatic: 3 retries with exponential backoff
- Manual: Re-queue failed tasks via Flower

### Failed Tasks
```python
# Check failed tasks
curl http://localhost:8000/api/v1/jobs/?status=failed

# Retry manually
from celery_app import app
task = app.AsyncResult(task_id)
task.retry()
```

### Queue Visibility
- Messages become visible again after 30 minutes if not ACKed
- Failed messages stay in queue for retry
- Dead letter queue (optional): Configure in Cloudflare

## Performance Tuning

### Celery Workers
```bash
# Scale workers
docker-compose up --scale celery_worker=5

# Adjust concurrency (tasks per worker)
celery -A celery_app worker --concurrency=8
```

### Batch Processing
- Embedding batch size: 100 chunks
- Queue pull batch: 10 messages
- Chunking: 1000 chars, 200 overlap

### Memory Management
- Worker restart after 50 tasks (prevents leaks)
- Task time limits: 50min soft, 1hr hard

## Production Considerations

### Security
- Use Redis AUTH in production
- Encrypt R2 buckets
- Rotate API tokens regularly
- Use secrets management (Vault, AWS Secrets Manager)

### Scalability
- Use Redis Cluster for HA
- Scale Celery workers horizontally
- Monitor queue depth
- Set up auto-scaling

### Monitoring
- Integrate with Prometheus
- Set up alerts (Sentry, PagerDuty)
- Log aggregation (CloudWatch, Datadog)

### Backups
- Redis: AOF + RDB persistence
- PostgreSQL: Daily backups
- R2: Versioning enabled

## Troubleshooting

### Queue Consumer Not Starting
```bash
# Check logs
docker-compose logs api | grep "Queue consumer"

# Verify CF credentials
curl -H "Authorization: Bearer $CF_API_TOKEN" \
  https://api.cloudflare.com/client/v4/accounts/$CF_ACCOUNT_ID/queues
```

### Tasks Not Processing
```bash
# Check Celery workers
celery -A celery_app inspect active

# Check Redis connection
redis-cli ping

# View worker logs
docker-compose logs celery_worker
```

### R2 Events Not Triggering
```bash
# Verify event notification
# Dashboard > R2 > Bucket > Event Notifications

# Test upload
aws s3 cp test.txt s3://fltr-datasets/test-dataset/test.txt \
  --endpoint-url https://$R2_ACCOUNT_ID.r2.cloudflarestorage.com
```

## Testing

### Unit Tests
```bash
pytest tests/test_document_processor.py
```

### Integration Test
```bash
# 1. Upload file
# 2. Wait 30 seconds
# 3. Check job status
curl http://localhost:8000/api/v1/jobs/dataset/{dataset_id}/status
```

## Future Enhancements

- [ ] Docling integration for PDF parsing
- [ ] Semantic chunking (RecursiveCharacterTextSplitter)
- [ ] Support for more file types (DOCX, PPTX, HTML)
- [ ] Progress tracking (% complete)
- [ ] WebSocket notifications
- [ ] Batch embedding optimization
- [ ] Multi-modal embeddings (text + images)
- [ ] Cost tracking per dataset

## Dependencies

```
celery==5.4.0
redis==5.2.1
flower==2.0.1
boto3==1.40.50
openai==1.10.0
pymilvus==2.3.5
```

## Resources

- [Celery Docs](https://docs.celeryq.dev/)
- [Cloudflare Queue Docs](https://developers.cloudflare.com/queues/)
- [Flower Docs](https://flower.readthedocs.io/)
- [Milvus Docs](https://milvus.io/docs)
