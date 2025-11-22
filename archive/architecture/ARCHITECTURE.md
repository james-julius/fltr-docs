# FLTR Architecture Documentation

Complete system architecture for the FLTR AI-ready dataset platform.

**Last Updated**: October 31, 2024
**Version**: 2.0 (Modal Migration)

## System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        FLTR Platform                             │
│                   Context as a Service                           │
└─────────────────────────────────────────────────────────────────┘

┌──────────────┐      ┌──────────────┐      ┌──────────────────┐
│   Next.js    │◄────►│   FastAPI    │◄────►│  Cloudflare R2   │
│   Frontend   │      │   Backend    │      │  Object Storage  │
└──────────────┘      └──────────────┘      └──────────────────┘
       │                     │                        │
       │                     │                        │
       │                     │                        ▼
       │                     │              ┌──────────────────┐
       │                     │              │  Modal.com       │
       │                     │              │  Doc Processing  │
       │                     │              └──────────────────┘
       │                     │                        │
       │                     ▼                        ▼
       │              ┌──────────────┐      ┌──────────────────┐
       │              │ PostgreSQL   │      │    Milvus        │
       │              │  Metadata    │      │  Vector Store    │
       │              └──────────────┘      └──────────────────┘
       │                     │
       └────────────────────►│
                             ▼
                      ┌──────────────┐
                      │    OpenAI    │
                      │  Embeddings  │
                      └──────────────┘
```

## Architecture Highlights

### Key Design Decisions

1. **Modal for Document Processing**: Heavy processing (PDF parsing, embedding generation) runs on Modal's serverless GPU infrastructure
2. **Shared Milvus Collection**: Single `fltr_documents` collection with `dataset_id` filtering (not per-dataset collections)
3. **Direct R2 Uploads**: Presigned URLs allow client-side uploads, no API throughput bottleneck
4. **SQLAlchemy Event System**: Automatic dataset status updates triggered by document changes
5. **Optimized API Server**: DigitalOcean droplet downsized from $50→$24/mo after Modal migration

## Document Processing Pipeline

### 1. Upload Flow

```
Client → FastAPI (presigned URL) → R2 → Modal Webhook → Modal Processing
```

**Detailed Steps:**

1. User creates dataset (POST `/api/v1/datasets`)
2. Frontend receives `dataset` + `upload_urls[]` with presigned R2 URLs
3. Client uploads files **directly to R2** using PUT (no API throughput bottleneck)
4. R2 triggers webhook to FastAPI `/api/v1/webhooks/r2-upload`
5. FastAPI creates Document record with `status: "processing"`
6. FastAPI calls Modal function via HTTP: `process_document_modal.remote()`
7. Modal downloads from R2, processes, stores embeddings, updates database

**File Structure in R2:**
```
{datasetId}/
  ├── document1.pdf
  ├── document2.json
  └── subfolder/
      └── nested.txt
```

### 2. Processing Flow (Modal)

```
R2 → FastAPI Webhook → Modal Function → [Download, Parse, Chunk, Embed, Store]
```

**Modal Processing Steps:**

```python
@app.function(
    cpu=8.0,
    memory=8192,
    min_containers=1,  # Keep warm to reduce cold starts
    timeout=600,
    image=image
)
def process_document_modal(dataset_id: str, document_id: str, object_key: str, filename: str):
    # 1. Download from R2 (CLOUDFLARE_R2_* env vars)
    file_bytes = download_from_r2(object_key)

    # 2. Validate & Repair PDF (if needed)
    if filename.endswith('.pdf'):
        file_bytes = validate_and_repair_pdf(file_bytes)

    # 3. Parse document using Docling
    # - PDF: Advanced layout analysis, table extraction
    # - JSON/CSV/TXT: Direct parsing
    document = parse_document(file_bytes, filename)

    # 4. Chunk text (1000 chars, 200 overlap)
    chunks = chunk_document(document, chunk_size=1000, overlap=200)

    # 5. Generate embeddings (OpenAI text-embedding-3-small)
    embeddings = generate_embeddings(chunks)  # Batch API

    # 6. Store in Milvus (shared collection)
    store_in_milvus(
        collection_name="fltr_documents",  # Single shared collection
        chunks=chunks,
        embeddings=embeddings,
        metadata={
            "dataset_id": dataset_id,
            "document_id": document_id,
            "filename": filename
        }
    )

    # 7. Flush Milvus to make data immediately searchable
    milvus_client.flush("fltr_documents")

    # 8. Update database (triggers SQLAlchemy events)
    update_document_status(document_id, status="ready", chunk_count=len(chunks))
```

**Processing Time:**
- Small files (< 1MB): 10-30 seconds
- Medium files (1-10MB): 30-90 seconds
- Large files (> 10MB): 90-300 seconds

**Performance Breakdown** (typical PDF):
- Download: 5-10% of time
- Parse (Docling): 40-50% of time
- Chunk: 5-10% of time
- Embed (OpenAI): 20-30% of time
- Store (Milvus): 10-15% of time
- DB Update: < 5% of time

### 3. Status Management (SQLAlchemy Events)

Dataset status is **automatically updated** via SQLAlchemy event listeners:

```python
@event.listens_for(Document, 'after_insert')
@event.listens_for(Document, 'after_update')
@event.listens_for(Document, 'after_delete')
def update_dataset_status_on_document_change(mapper, connection, target):
    # Query document statuses for this dataset
    status_counts = count_documents_by_status(target.dataset_id, connection)

    # Update dataset.document_count
    update_document_count(target.dataset_id, connection)

    # Determine dataset status
    if status_counts['failed'] > 0 and status_counts['ready'] == 0:
        new_status = 'failed'
    elif status_counts['processing'] > 0 or status_counts['pending'] > 0:
        new_status = 'processing'
    else:
        new_status = 'ready'

    # Update dataset status
    update_dataset_status(target.dataset_id, new_status, connection)
```

**No manual status updates needed** - the system automatically reflects document state.

## Technology Stack

### Frontend
- **Next.js 14** - React framework with App Router
- **TypeScript** - Type safety
- **Shadcn UI** - Component library
- **TanStack Query** - Data fetching (via Orval hooks)
- **Orval** - API client generation from OpenAPI
- **Better Auth** - Authentication

### Backend (FastAPI)
- **FastAPI** - Python async web framework
- **SQLModel** - ORM (PostgreSQL)
- **Pydantic** - Validation
- **SQLAlchemy Events** - Automatic status updates
- **PostgreSQL** - Relational metadata
- **Redis** - Session storage

### Document Processing (Modal)
- **Modal.com** - Serverless compute platform
- **Docling** - Advanced PDF parsing (layout, tables, OCR)
- **PyPDF2** - PDF validation and repair
- **OpenAI API** - Embeddings (text-embedding-3-small)
- **CPU**: 8 cores, 8GB RAM (optimized for Docling)
- **min_containers=1** - Keeps one container warm to reduce cold starts

### Storage & Databases
- **Cloudflare R2** - Object storage (S3-compatible, no egress fees)
- **PostgreSQL** - Relational metadata (DigitalOcean or Supabase)
- **Milvus Cloud** - Vector database (shared `fltr_documents` collection)
- **Redis** - Session management

### Infrastructure
- **DigitalOcean** - FastAPI app server ($24/mo, 2 vCPU, 4GB RAM)
- **Modal.com** - Document processing (pay-per-second)
- **Vercel** - Next.js frontend hosting
- **Cloudflare** - R2 storage, Workers (optional)

## API Endpoints

### Datasets
- `POST /api/v1/datasets` - Create dataset + get upload URLs
- `GET /api/v1/datasets` - List datasets
- `GET /api/v1/datasets/{id}` - Get dataset
- `PATCH /api/v1/datasets/{id}` - Update dataset
- `PATCH /api/v1/datasets/{id}/settings` - Update dataset settings (visibility, etc.)
- `DELETE /api/v1/datasets/{id}` - Delete dataset + vectors
- `POST /api/v1/datasets/{id}/publish` - Publish dataset

### Documents
- `GET /api/v1/datasets/{id}/documents` - List documents in dataset
- `POST /api/v1/datasets/{id}/documents` - Add documents to existing dataset
- `DELETE /api/v1/documents/{id}` - Delete document + vectors
- `POST /api/v1/documents/{id}/retry` - Retry failed document processing

### Storage
- `POST /api/v1/storage/upload-url` - Get presigned upload URL (POST)
- `POST /api/v1/storage/upload-url-put` - Get presigned upload URL (PUT)
- `POST /api/v1/storage/bulk-upload-urls` - Get multiple upload URLs
- `GET /api/v1/storage/files/{dataset_id}` - List files in dataset
- `DELETE /api/v1/storage/files/{dataset_id}/{file_name}` - Delete file from R2

### MCP (Model Context Protocol)
- `GET /api/v1/mcp/datasets` - List datasets available for chat
- `GET /api/v1/mcp/query/{dataset_id}/search` - Semantic search in dataset
- `POST /api/v1/mcp/batch-query` - Batch search across multiple datasets
- `GET /api/v1/mcp/{dataset_id}/info` - Get dataset metadata

### Embeddings
- `POST /api/v1/embeddings/search` - Semantic search (legacy, use MCP instead)

### Webhooks (Internal)
- `POST /api/v1/webhooks/r2-upload` - R2 upload notification (triggers Modal)

### Health
- `GET /health` - Service health check

## Data Models

### Dataset
```python
class Dataset(SQLModel, table=True):
    id: UUID
    owner_id: UUID
    name: str
    slug: str  # URL-friendly identifier
    description: str
    category: str  # finance, legal, healthcare, tech, research
    visibility: str  # public, private
    status: str  # uploading, processing, ready, failed (auto-updated)
    collection_name: str  # Always "fltr_documents" (shared collection)
    document_count: int  # Auto-updated via events
    embedding_dimension: int  # 1536 for text-embedding-3-small
    embedding_model: str  # "text-embedding-3-small"
    pricing_model: str  # free, paid
    view_count: int
    created_at: datetime
    updated_at: datetime
```

### Document
```python
class Document(SQLModel, table=True):
    id: UUID
    dataset_id: UUID
    filename: str
    object_key: str  # R2 path: {dataset_id}/{filename}
    file_size: int
    mime_type: str
    status: str  # pending, processing, ready, failed
    chunk_count: int
    error: str | None
    created_at: datetime
    updated_at: datetime
```

### Milvus Schema (Shared Collection)
```python
collection_name = "fltr_documents"

schema = {
    "id": "VARCHAR(36)",  # Document chunk ID
    "text": "VARCHAR(65535)",  # Chunk text
    "embedding": "FLOAT_VECTOR(1536)",  # OpenAI embedding
    "dataset_id": "VARCHAR(36)",  # For filtering by dataset
    "document_id": "VARCHAR(36)",
    "filename": "VARCHAR(255)",
    "document_type": "VARCHAR(50)",  # pdf, json, csv, txt
    "chunk_index": "INT64",
    "metadata": "JSON"  # Additional metadata
}
```

**Key Point**: All datasets share the `fltr_documents` collection. Queries filter by `dataset_id` metadata field.

## Configuration

### Environment Variables

**FastAPI (.env):**
```bash
# Database
DATABASE_URL=postgresql://user:pass@host:5432/fltr

# Milvus
MILVUS_HOST=xxx.aws-us-west-2.vectordb.zillizcloud.com
MILVUS_PORT=19530
MILVUS_TOKEN=xxx
MILVUS_DB_NAME=default

# Cloudflare R2
R2_ACCOUNT_ID=xxx
R2_ACCESS_KEY_ID=xxx
R2_SECRET_ACCESS_KEY=xxx
R2_BUCKET_NAME=fltr-datasets
R2_ENDPOINT_URL=https://xxx.r2.cloudflarestorage.com

# OpenAI
OPENAI_API_KEY=sk-xxx

# Modal (for triggering functions)
MODAL_TOKEN_ID=xxx
MODAL_TOKEN_SECRET=xxx

# Redis (Sessions)
REDIS_URL=redis://localhost:6379/0

# Better Auth
BETTER_AUTH_SECRET=xxx
BETTER_AUTH_URL=http://localhost:3000
```

**Modal (modal/secrets):**
```bash
# Created via: modal secret create fltr-secrets

# Database
DATABASE_URL=postgresql://...

# Milvus
MILVUS_HOST=xxx
MILVUS_PORT=19530
MILVUS_TOKEN=xxx

# Cloudflare R2
CLOUDFLARE_R2_ACCESS_KEY_ID=xxx
CLOUDFLARE_R2_SECRET_ACCESS_KEY=xxx
CLOUDFLARE_R2_BUCKET_NAME=fltr-datasets
CLOUDFLARE_R2_ENDPOINT_URL=https://xxx.r2.cloudflarestorage.com
CLOUDFLARE_R2_ACCOUNT_ID=xxx

# OpenAI
OPENAI_API_KEY=sk-xxx
```

**Next.js (.env.local):**
```bash
NEXT_PUBLIC_API_URL=http://localhost:8000
BETTER_AUTH_SECRET=xxx
BETTER_AUTH_URL=http://localhost:3000
DATABASE_URL=postgresql://...
```

## Deployment

### Local Development

```bash
# 1. Start PostgreSQL (Docker or local)
docker run -d \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=fltr \
  postgres:15

# 2. Start Redis
docker run -d -p 6379:6379 redis:7

# 3. Start FastAPI
cd fastapi
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn main:app --reload --port 8000

# 4. Start Next.js
cd nextjs
pnpm install
pnpm dev  # Port 3000

# 5. Deploy Modal (one-time setup)
cd modal
modal deploy modal_app.py
```

### Production Deployment

**1. FastAPI (DigitalOcean Droplet)**
```bash
# Recommended: $24/mo Basic Regular (2 vCPU, 4GB RAM)

# Setup
apt update && apt upgrade -y
apt install python3.10 python3-pip python3-venv nginx -y

# Deploy
cd /opt/fltr/fastapi
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn main:app --host 0.0.0.0 --port 8000

# Use systemd for auto-restart
systemctl enable fltr-api
```

**2. Modal Deployment**
```bash
cd modal
modal deploy modal_app.py

# Modal handles:
# - Auto-scaling
# - Cold start management
# - Secrets management
# - Logging
```

**3. Next.js (Vercel)**
```bash
cd nextjs
vercel deploy --prod

# Configure environment variables in Vercel dashboard
```

**4. Database (Managed PostgreSQL)**
- DigitalOcean Managed Database ($15-40/mo)
- OR Supabase Pro ($25/mo)
- OR Railway ($20-40/mo)

**5. Milvus (Zilliz Cloud)**
- 0.5 CU for starter ($45/mo)
- Scale as needed

## Monitoring & Observability

### Modal Monitoring
- **Dashboard**: https://modal.com/apps
- **Logs**: `modal logs process_document_modal`
- **Metrics**: Function runtime, success rate, cold starts
- **Cost**: Per-second billing visible in dashboard

### FastAPI Monitoring
```bash
# Health check
curl http://localhost:8000/health

# Database status
psql -c "SELECT status, COUNT(*) FROM datasets GROUP BY status;"

# Failed documents
psql -c "SELECT * FROM documents WHERE status='failed';"
```

### Milvus Monitoring
```python
from pymilvus import connections, utility

connections.connect(...)
stats = utility.get_collection_stats("fltr_documents")
print(f"Total entities: {stats['row_count']}")
```

### Performance Metrics

**Target Latencies:**
- Upload URL generation: < 500ms
- Document upload to R2: < 5s (depends on size)
- Processing start: < 10s (if warm container)
- Small file processing: < 30s
- Large file processing: < 5 minutes

**Monitoring Tools:**
- **Modal Dashboard**: Function performance, costs
- **PostgreSQL**: Query performance, connection counts
- **Milvus Dashboard**: Query latency, vector counts
- **Vercel Analytics**: Frontend performance

## Security

### Authentication
- **Better Auth** - Modern auth for Next.js
- **Session-based** - HTTP-only cookies
- **API Key Support** - For programmatic access (X-API-Key header)
- **JWT Tokens** - Bearer token auth (future)

### Storage Security
- **Presigned URLs** - Time-limited (5 minutes), scoped to specific object
- **R2 Private Bucket** - No public access
- **Dataset Isolation** - Files organized by `{datasetId}/` prefix
- **Deletion** - Cascading deletes (dataset → documents → vectors → files)

### Network Security
- **CORS** - Restricted origins (Next.js frontend only)
- **HTTPS** - TLS for all API calls
- **Secret Management** - Modal secrets, environment variables

### Data Privacy
- **User Isolation** - `owner_id` filtering on all queries
- **Public/Private Datasets** - Visibility controls
- **MCP Endpoints** - Public datasets accessible without auth

## Performance

### Throughput
- **Upload**: Direct R2 uploads (no API bottleneck)
- **Processing**: Modal auto-scales based on load
- **Queries**: Milvus supports 10K+ QPS

### Latency Optimization
1. **Modal `min_containers=1`**: Reduces cold starts
2. **Milvus Flush**: Makes vectors immediately searchable
3. **Batch Embeddings**: 100 chunks per OpenAI API call
4. **Index Optimization**: HNSW index for fast similarity search

### Cost Optimization
1. **Shared Milvus Collection**: Avoids per-dataset collection limits
2. **Modal Pay-Per-Use**: No idle costs
3. **R2 No Egress Fees**: 60% cheaper than S3
4. **Optimized Server**: $24/mo vs $50/mo (52% savings)

### Scalability
- **Horizontal**: Modal auto-scales functions
- **Vertical**: Increase Modal CPU/memory if needed
- **Database**: PostgreSQL read replicas
- **Milvus**: Increase CU or self-host for large scale

## Error Handling

### Modal Retry Strategy
- **Automatic Retries**: Modal handles transient failures
- **Manual Retry**: `POST /api/v1/documents/{id}/retry`
- **Timeout**: 600 seconds (10 minutes)
- **Logging**: All errors logged to Modal dashboard

### Document Status Flow
```
pending → processing → ready
                    → failed → (manual retry) → processing
```

### Common Errors

**1. PDF Parsing Failures**
- **Cause**: Invalid PDF, missing MediaBox
- **Solution**: Automatic PDF repair with PyPDF2
- **Fallback**: Manual upload in different format

**2. Embedding API Errors**
- **Cause**: Rate limits, network issues
- **Solution**: Modal retries with exponential backoff
- **Monitoring**: Check Modal logs

**3. Milvus Connection Errors**
- **Cause**: Network timeout, collection not found
- **Solution**: Connection retry, collection auto-creation
- **Fix**: Verify MILVUS_TOKEN and MILVUS_HOST

## Migration History

### V2.0 - Modal Migration (October 2024)

**Changed:**
- ❌ Removed: Celery workers, Redis queue, Flower dashboard
- ❌ Removed: Cloudflare Queue consumer
- ✅ Added: Modal.com for document processing
- ✅ Added: SQLAlchemy event-based status updates
- ✅ Changed: Per-dataset collections → Shared `fltr_documents` collection
- ✅ Optimized: DigitalOcean server from $50/mo → $24/mo

**Benefits:**
- **Simpler Architecture**: No Celery/Redis/Queue management
- **Better Performance**: GPU-accelerated PDF parsing
- **Lower Costs**: Pay-per-second vs fixed infrastructure
- **Auto-Scaling**: Modal handles load automatically
- **Easier Maintenance**: Fewer moving parts

**Migration Guide**: See `MODAL_MIGRATION.md`

## Future Enhancements

### Short Term (Q4 2024)
- [x] Modal integration (completed)
- [x] Shared Milvus collection (completed)
- [x] SQLAlchemy event system (completed)
- [ ] Dataset analytics dashboard
- [ ] User quotas and rate limiting
- [ ] Cost tracking per dataset/user

### Medium Term (Q1 2025)
- [ ] Semantic chunking (vs fixed-size)
- [ ] Multi-modal embeddings (text + images)
- [ ] Real-time progress updates (WebSockets)
- [ ] Dataset versioning
- [ ] Advanced PDF features (tables, forms)
- [ ] Batch processing optimizations

### Long Term (Q2+ 2025)
- [ ] Self-hosted embedding models (cost reduction)
- [ ] Custom fine-tuned models per dataset
- [ ] Dataset marketplace
- [ ] API usage analytics
- [ ] Enterprise features (SSO, audit logs)

## Testing

### FastAPI Tests
```bash
cd fastapi
pytest tests/ -v

# Specific test suites
pytest tests/test_dataset_router.py -v
pytest tests/test_mcp_router.py -v
pytest tests/test_document_processor.py -v
```

### Modal Tests
```bash
cd modal
pytest tests/ -v

# Run in Modal environment
modal run tests/test_document_processor.py
```

### Next.js Tests
```bash
cd nextjs
pnpm test

# Watch mode
pnpm test -- --watch
```

### E2E Test Flow
1. Create dataset via API → returns `upload_urls`
2. Upload file to presigned URL (PUT)
3. Wait for webhook → Modal processing
4. Poll document status until `ready`
5. Verify `dataset.status = 'ready'`
6. Query vectors via MCP endpoint
7. Verify search results returned

## Resources

### Documentation
- **FastAPI Docs**: http://localhost:8000/docs (Swagger UI)
- **Modal Dashboard**: https://modal.com/apps
- **Milvus Docs**: https://milvus.io/docs
- **Cloudflare R2 Docs**: https://developers.cloudflare.com/r2/

### Monitoring
- **Modal Logs**: `modal logs process_document_modal --follow`
- **FastAPI Logs**: Check systemd journal or docker logs
- **Database**: Use psql or pgAdmin
- **Milvus**: Zilliz Cloud dashboard

### Support
For issues:
1. **Check Modal logs**: Function failures, timeouts
2. **Check database**: `SELECT * FROM documents WHERE status='failed'`
3. **Check API health**: `curl http://localhost:8000/health`
4. **Review PRODUCTION_FIX_STEPS.md**: Known issues and fixes

---

**Version**: 2.0
**Last Updated**: October 31, 2024
**Next Review**: After reaching 1,000+ documents/day or major feature releases
