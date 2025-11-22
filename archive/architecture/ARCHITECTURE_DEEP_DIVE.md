# FLTR RAG Architecture - Deep Dive Analysis
## For Planning 8 New RAG Features

**Analysis Date:** November 9, 2025  
**Codebase:** /Users/jamesjulius/Coding/FLTR  
**Scope:** Comprehensive architecture analysis for 8 new feature implementations

---

## EXECUTIVE SUMMARY

### Current Architecture Status
- **Maturity:** Production-ready (~70% of enterprise RAG capability)
- **Infrastructure:** Modern serverless + cloud-native
- **Tech Stack:** FastAPI + Modal + Milvus + PostgreSQL + R2
- **Key Strength:** Multimodal document processing (PDFs + Vision Models)
- **Critical Gaps:** No hierarchical awareness, basic chunking, no versioning

### Planning Recommendations
1. **Use shared Milvus collection** - Already optimized for multi-dataset queries
2. **Leverage existing Modal pipeline** - Don't rebuild; extend it
3. **Extend Document model** - Add metadata fields for new features
4. **Respect credit system** - New features will need credit gates
5. **Pattern:** Service → Repository → Router flow is established

---

## ARCHITECTURE DIAGRAM

```
┌─────────────────────────────────────────────────────────────────┐
│                     CLIENT LAYER                                 │
│  (Next.js Frontend / MCP Clients / API Consumers)                │
└──────────────────────┬──────────────────────────────────────────┘
                       │ HTTP/REST/MCP
                       ↓
┌─────────────────────────────────────────────────────────────────┐
│                   FASTAPI LAYER (Main Service)                  │
│                      [:8000]                                     │
├─────────────────────────────────────────────────────────────────┤
│  Routes:                           Services:                     │
│  • /datasets          →            DatasetService               │
│  • /datasets/{id}/embeddings →     EmbeddingService             │
│  • /storage           →            R2StorageService             │
│  • /auth              →            AuthService (Better Auth)     │
│  • /credits           →            CreditService                │
│  • /mcp               →            MCPService                    │
│  • /jobs              →            JobsService                   │
│                                                                   │
│  Middleware:                       Data Layer:                   │
│  • AuthMiddleware     →            Repositories:                │
│  • CreditGate         →            - DatasetRepository          │
│  • CORS               →            - DocumentRepository         │
│                                    - CreditRepository           │
└────────────┬─────────────────────────────────────────────────────┘
             │
    ┌────────┼────────┐
    ↓                 ↓
┌──────────────┐  ┌──────────────────────┐
│  PostgreSQL  │  │  Modal Serverless    │
│   [:5432]    │  │  Functions           │
├──────────────┤  ├──────────────────────┤
│ Tables:      │  │ process_document_    │
│ - datasets   │  │   modal()            │
│ - documents  │  │                      │
│ - credits    │  │ Services (mounted):  │
│ - auth       │  │ • pdf_validator      │
│              │  │ • storage            │
│              │  │ • document_processor │
│              │  │ • embeddings         │
│              │  │ • vector_store       │
│              │  │ • multimodal_proc.   │
│              │  │ • vision_service     │
│              │  │ • ocr_service        │
└──────────────┘  └──────────────────────┘
    │                     │
    │              ┌──────┴──────┐
    │              ↓             ↓
    │         ┌─────────┐   ┌──────────┐
    │         │ OpenAI  │   │ Vision   │
    │         │Embeddings   │(GPT4V,  │
    │         │API      │   │Claude,  │
    │         │         │   │Gemini)  │
    │         └─────────┘   └──────────┘
    │
    ↓
  ┌─────────────────────┐
  │ Milvus/Zilliz      │
  │ Vector Database    │
  │    [:19530]        │
  ├─────────────────────┤
  │ Collection:        │
  │ fltr_documents     │
  │ (14 fields)        │
  │ (Shared across     │
  │  all datasets)     │
  └─────────────────────┘

  ┌──────────────────────┐
  │ Cloudflare R2        │
  │ Object Storage       │
  ├──────────────────────┤
  │ Paths:               │
  │ {dataset_id}/        │
  │   {filename}         │
  │   images/            │
  │     {page}_{idx}.png │
  └──────────────────────┘
```

---

## 1. DOCUMENT PROCESSING PIPELINE

### 1.1 Flow: Upload → Processing → Embedding → Storage

```
User Upload (Client)
      ↓
Create Dataset (FastAPI)
      ├─ Generate presigned R2 URLs
      └─ Return upload URLs
      ↓
Client Uploads File to R2 (Direct upload - bypasses FastAPI)
      ↓
Cloudflare Notification (Webhook or Queue)
      ├─ POST to FastAPI /webhooks/upload-complete
      ├─ Creates Document record (status: "uploaded")
      ├─ Charges credits (document_upload operation)
      └─ Triggers Modal processing
      ↓
Modal Process Document
      ├─ Download from R2
      ├─ Validate PDF
      ├─ Parse Document (PyMuPDF4LLM + Docling)
      ├─ Extract Images & Run Vision Models
      ├─ Chunk Document
      ├─ Generate Embeddings (OpenAI API)
      ├─ Store in Milvus
      ├─ Update Document.status → "ready"
      ├─ Update Dataset.document_count
      └─ Update Dataset.status (when all docs ready)
      ↓
Search/Retrieval (User)
      ├─ Query embedding generation
      ├─ Milvus semantic search (filtered by dataset_id)
      ├─ Return chunks + metadata + presigned image URLs
      └─ Charge search credits
```

### 1.2 Key Entry Points

**FastAPI Routers:**
- `POST /api/v1/datasets` → Create dataset + generate upload URLs
- `POST /api/v1/datasets/{id}/upload-complete` → Webhook for document uploads
- `POST /api/v1/datasets/{id}/embeddings/search` → Semantic search

**Modal Functions:**
- `process_document_modal(dataset_id, object_key, task_id)` → Main pipeline

**Data Flow:**
```python
# FastAPI → Modal (HTTP POST to webhook)
{
    "dataset_id": str,
    "object_key": str,
    "task_id": str  # Document ID
}

# Modal → FastAPI (Status updates)
{
    "status": "completed" | "failed",
    "chunks": int,
    "error": str (if failed)
}
```

### 1.3 Async/Sync Patterns

**FastAPI (Sync → Async):**
```python
# Service layer (sync)
class DatasetService:
    def create_dataset(self, ...):  # Sync
        ...

# Routers (async)
@router.post("")
async def create_dataset(...):  # Async
    await service.create_dataset(...)

# Credit gate (async decorator)
@requires_credits(amount=N, operation=Op, resource_id_param="id")
async def endpoint(...):
    ...
```

**Modal (Pure Async):**
```python
@app.function()
async def process_document_modal(...):
    # All operations are async
    file_bytes = await download_from_r2(...)
    document = await parse_document(...)
    chunks = await chunk_document(...)
    embeddings = await generate_embeddings(...)
    await store_in_milvus(...)
```

### 1.4 Error Handling

**Document Level:**
```
Document Status: pending → uploaded → processing → ready | failed

Error Handling:
1. PDF validation fails → Document.status = "failed", error message
2. Parsing fails → Document.status = "failed", error message
3. Modal timeout → Document.status = "failed", credit refund triggered
4. Milvus insert fails → Retry with exponential backoff
```

**Dataset Level:**
```
Dataset Status Logic (in modal_app.py::_update_dataset_status):
- All ready → dataset.status = "ready"
- All failed → dataset.status = "failed"
- Some ready/processing → dataset.status = "processing"
- None ready → dataset.status = "uploading"
```

**Credit Refunds:**
- Triggered via Modal error handler
- Calls FastAPI `/credits/refund` endpoint
- Transaction tracked with `operation: "document_upload"`

---

## 2. DATABASE SCHEMA

### 2.1 Current Tables

**PostgreSQL Schema:**

```sql
-- Main dataset table
CREATE TABLE datasets (
    id UUID PRIMARY KEY,
    owner_id UUID NOT NULL,
    name VARCHAR NOT NULL,
    slug VARCHAR UNIQUE NOT NULL,
    description TEXT,
    category VARCHAR,
    visibility VARCHAR DEFAULT 'public',
    status VARCHAR DEFAULT 'uploading',        -- uploading|processing|ready|failed
    collection_name VARCHAR,                   -- Always "fltr_documents"
    document_count INT DEFAULT 0,
    embedding_dimension INT DEFAULT 1536,
    embedding_model VARCHAR DEFAULT 'text-embedding-3-small',
    pricing_model VARCHAR DEFAULT 'free',
    price_monthly FLOAT,
    price_annual FLOAT,
    rating_average FLOAT DEFAULT 0.0,
    download_count INT DEFAULT 0,
    view_count INT DEFAULT 0,
    featured BOOLEAN DEFAULT FALSE,
    verified BOOLEAN DEFAULT FALSE,
    system_prompt TEXT,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    INDEX idx_owner_id (owner_id),
    INDEX idx_status (status),
    UNIQUE KEY unique_slug (slug)
);

-- Document/file tracking
CREATE TABLE documents (
    id UUID PRIMARY KEY,
    dataset_id UUID NOT NULL,
    file_name VARCHAR NOT NULL,
    object_key VARCHAR UNIQUE NOT NULL,        -- R2 key
    content_type VARCHAR,
    file_size INT,
    status VARCHAR DEFAULT 'pending',          -- pending|uploaded|processing|ready|failed
    chunk_count INT DEFAULT 0,
    error TEXT,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    FOREIGN KEY (dataset_id) REFERENCES datasets(id),
    INDEX idx_dataset_id (dataset_id),
    INDEX idx_status (status),
    INDEX idx_object_key (object_key)
);

-- Credit transactions
CREATE TABLE credit_transactions (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    resource_id UUID,                          -- document_id or dataset_id
    operation VARCHAR NOT NULL,                -- document_upload|embedding_search|...
    type VARCHAR NOT NULL,                     -- usage|refund|purchase
    amount INT NOT NULL,
    balance_before INT,
    balance_after INT,
    metadata JSON,
    created_at TIMESTAMP,
    INDEX idx_user_id (user_id),
    INDEX idx_resource_id (resource_id),
    INDEX idx_operation (operation)
);

-- Auth (Better Auth integration)
CREATE TABLE user_accounts (
    id VARCHAR PRIMARY KEY,
    name VARCHAR,
    email VARCHAR UNIQUE,
    credits INT DEFAULT 100,
    created_at TIMESTAMP
);
```

**Milvus Schema (Vector Database):**

```
Collection: fltr_documents (Shared across all datasets)

Fields:
├─ id                   (INT64, auto-increment, primary key)
├─ vector               (FLOAT_VECTOR, dim=1536)
├─ text                 (VARCHAR, max 65535)
├─ dataset_id           (VARCHAR, 64)         ← Used for filtering
├─ document_id          (VARCHAR, 64)
├─ filename             (VARCHAR, 512)
├─ chunk_index          (INT64)
├─ document_type        (VARCHAR, 128)
├─ chunk_type           (VARCHAR, 16)         ← "text" or "ocr"
├─ page                 (INT64)
├─ image_index          (INT64)
├─ ocr_confidence       (FLOAT)
├─ image_r2_key        (VARCHAR, 512)
└─ image_size           (VARCHAR, 32)

Index:
├─ FLAT index on vector field
├─ No partition strategy (all datasets in one collection)
└─ Filtering at query time by dataset_id
```

### 2.2 Relationships & Indexes

**Key Relationships:**
```
datasets (1) ←→ (N) documents
           ↓
        Milvus fltr_documents (vectors filtered by dataset_id)
```

**Index Strategy:**
```
PostgreSQL Indexes:
- datasets.slug (UNIQUE) - Fast dataset lookup by slug
- datasets.status - Filtering by upload status
- documents.dataset_id - List documents for dataset
- documents.status - Status filtering
- documents.object_key - By R2 key

Milvus Indexes:
- vector field - FLAT index (sufficient for <1M vectors)
- Filter expressions at query time (no secondary indexes)
```

### 2.3 For 8 New RAG Features

**Schema Changes Required:**

**For Hierarchical Chunking:**
```sql
ALTER TABLE documents ADD COLUMN (
    hierarchy_json JSON,                       -- {H1: [...], H2: [...]}
    semantic_boundaries BOOLEAN
);

-- In Milvus:
Add to fltr_documents:
- parent_chunk_id (INT64) - For hierarchical retrieval
- hierarchy_level (INT64) - 0=root, 1=section, 2=subsection
```

**For Change Detection/Versioning:**
```sql
ALTER TABLE documents ADD COLUMN (
    version INT DEFAULT 1,
    hash_content VARCHAR(64),                  -- SHA-256 for change detection
    previous_version_id UUID                   -- Link to previous version
);
```

**For Custom Metadata:**
```sql
ALTER TABLE documents ADD COLUMN (
    custom_metadata JSON                       -- User-defined fields
);

-- In Milvus:
Extend fltr_documents with:
- metadata_keys (VARCHAR) - Searchable metadata
```

**For Usage Analytics:**
```sql
CREATE TABLE document_access_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    dataset_id UUID NOT NULL,
    document_id UUID NOT NULL,
    user_id UUID,
    access_type VARCHAR,                       -- search|download|view
    timestamp TIMESTAMP,
    INDEX idx_dataset_id (dataset_id),
    INDEX idx_user_id (user_id)
);
```

---

## 3. VECTOR STORE INTEGRATION

### 3.1 Milvus Architecture

**Current Setup:**
```python
# Single shared collection for all datasets
collection_name = "fltr_documents"

# Filtering strategy (not partitions)
filter_expr = f'dataset_id == "{dataset_id}"'
```

**Connection Management:**
```python
# Development: Milvus Lite (file-based)
if settings.USE_MILVUS_LITE:
    client = MilvusClient(uri="data/milvus_lite.db")

# Production: Zilliz Cloud
else:
    client = MilvusClient(
        uri=settings.MILVUS_URI,
        token=settings.MILVUS_TOKEN
    )
```

### 3.2 Insertion Pattern

**Modal → Milvus:**
```python
# Location: /modal/services/vector_store.py::store_in_milvus()

data = []
for chunk, embedding in zip(chunks, embeddings):
    data.append({
        "vector": embedding,
        "text": chunk["text"],
        "dataset_id": dataset_id,
        "document_id": document_id,
        "filename": chunk["metadata"]["filename"],
        "chunk_index": chunk["metadata"]["chunk_index"],
        "document_type": chunk["metadata"]["document_type"],
        # ... 6 more OCR fields
    })

result = client.insert(collection_name="fltr_documents", data=data)
# Auto-flush (pymilvus 2.x+) - no manual flush needed
```

**Important:** Each document upload creates N vectors (one per chunk), all with same `document_id`.

### 3.3 Query Pattern

**FastAPI → Milvus:**
```python
# Location: /fastapi/services/embedding_service.py::_search_milvus()

# 1. Generate query embedding
query_vector = openai_client.embeddings.create(
    model="text-embedding-3-small",
    input=[query.query]
).data[0].embedding

# 2. Build filter expression
filter_expr = f'dataset_id == "{dataset_id}"'
if query.filter_expr:
    filter_expr = f'({filter_expr}) && ({query.filter_expr})'

# 3. Search
results = milvus.search(
    collection_name="fltr_documents",
    data=[query_vector],
    limit=query.limit,  # Default 10, max 20
    filter=filter_expr,
    output_fields=[
        "text", "filename", "chunk_index", "document_type",
        "chunk_type", "page", "image_index", "ocr_confidence",
        "image_r2_key", "image_size"
    ]
)

# 4. Format results with presigned image URLs
search_results = []
for hits in results:
    for hit in hits:
        entity = hit["entity"]
        search_results.append({
            "id": hit["id"],
            "text": entity["text"],
            "distance": hit["distance"],
            "meta_data": {
                "image_url": generate_presigned_url(entity["image_r2_key"])
                # ... + all metadata fields
            }
        })
```

### 3.4 Optimization Notes

**Current Limitations:**
- ❌ No partitions (single collection for all datasets)
- ❌ FLAT index only (linear search, not HNSW)
- ❌ No reranking capability
- ❌ No hybrid search (semantic + keyword)

**For New Features:**
- Keep FLAT index (works well < 5M vectors)
- Filter by `dataset_id` at query time (sufficient for isolation)
- Add secondary index for `chunk_type` if filtering by OCR vs text
- Extend with custom metadata fields as JSON

---

## 4. API ENDPOINTS & ROUTES

### 4.1 Current Endpoints

**Dataset Management:**
```
POST   /api/v1/datasets                    Create dataset + get upload URLs
GET    /api/v1/datasets                    List datasets (with filters)
GET    /api/v1/datasets/{dataset_id}       Get dataset by ID
GET    /api/v1/datasets/slug/{slug}        Get dataset by slug
PATCH  /api/v1/datasets/{dataset_id}       Update dataset
DELETE /api/v1/datasets/{dataset_id}       Delete dataset
GET    /api/v1/datasets/categories         List distinct categories
```

**Document Search & Embeddings:**
```
POST   /api/v1/datasets/{dataset_id}/embeddings/search    Search embeddings
GET    /api/v1/datasets/{dataset_id}/embeddings/stats     Get embedding stats
```

**Storage & Uploads:**
```
POST   /api/v1/storage/presigned-url       Generate presigned upload URL
POST   /api/v1/storage/download-url        Generate presigned download URL
POST   /api/v1/webhooks/upload-complete    Document upload webhook
```

**Authentication:**
```
POST   /api/v1/auth/login                  User login
POST   /api/v1/auth/logout                 User logout
POST   /api/v1/auth/oauth-callback         OAuth callback
GET    /api/v1/auth/me                     Get current user
```

**Credits:**
```
GET    /api/v1/credits/balance             Get user credit balance
POST   /api/v1/credits/refund              Refund credits (system error)
GET    /api/v1/credits/history             Get transaction history
```

**MCP Endpoint:**
```
GET    /mcp/download/{dataset_slug}        Download dataset (MCP compatible)
POST   /api/v1/mcp/query                   Search in dataset (MCP compatible)
```

### 4.2 Request/Response Examples

**Create Dataset:**
```json
// POST /api/v1/datasets
{
    "owner_id": "uuid",
    "name": "Medical Records",
    "slug": "medical-records-123abc",
    "description": "Patient intake forms",
    "category": "medical",
    "visibility": "private",
    "files": [
        {"file_name": "form1.pdf", "content_type": "application/pdf"},
        {"file_name": "form2.pdf", "content_type": "application/pdf"}
    ]
}

// Response 201
{
    "dataset": {
        "id": "uuid",
        "name": "Medical Records",
        "status": "uploading",
        "document_count": 0,
        ...
    },
    "upload_urls": [
        {
            "object_key": "uuid/form1.pdf",
            "upload_url": "https://r2.../upload",
            "fields": {...},
            "expires_in": 3600
        },
        ...
    ]
}
```

**Search Embeddings:**
```json
// POST /api/v1/datasets/{dataset_id}/embeddings/search
{
    "query": "patient age and medical history",
    "limit": 10,
    "filter_expr": 'chunk_type == "text"'  // Optional Milvus filter
}

// Response 200
{
    "query": "patient age and medical history",
    "dataset_id": "uuid",
    "results": [
        {
            "id": 12345,
            "text": "Patient age: 45 years old...",
            "distance": 0.85,
            "meta_data": {
                "filename": "form1.pdf",
                "chunk_index": 3,
                "chunk_type": "text",
                "page": 1,
                "image_url": "https://r2.../presigned-url?expires=...",
                "image_size": "1920x1080"
            }
        },
        ...
    ]
}
```

### 4.3 Authentication & Authorization

**Current System:**
- Better Auth (session-based)
- JWT tokens (API key auth)
- Credit gate middleware

```python
# @requires_credits decorator
@requires_credits(
    amount=1,
    operation=CreditOperation.EMBEDDING_SEARCH,
    resource_id_param="dataset_id"
)
async def endpoint(...):
    # Credits deducted before endpoint runs
    # Auto-refunded on HTTPException
    ...
```

### 4.4 For New Features

**Pattern to Follow:**

```python
# 1. Add to router
@router.post("/feature-endpoint")
@requires_credits(amount=N, operation=CreditOperation.FEATURE_NAME)
async def feature_endpoint(..., service: NewFeatureService = Depends()):
    return service.feature_implementation()

# 2. Create service
class NewFeatureService:
    def __init__(self, session: Session = Depends(get_session)):
        self.session = session
        self.repo = FeatureRepository(session)
    
    def feature_implementation(self):
        # Business logic here

# 3. Create repository (if needed)
class FeatureRepository:
    def __init__(self, session: Session):
        self.session = session
    
    def query_feature_data(self):
        # Data access
```

---

## 5. CONFIGURATION & ENVIRONMENT

### 5.1 Settings System

**Location:** `/fastapi/config.py`

**Key Sections:**
```python
class Settings(BaseSettings):
    # Environment
    ENVIRONMENT: str = "development"  # dev | staging | prod
    
    # API
    API_HOST: str = "0.0.0.0"
    API_PORT: int = 8000
    API_PREFIX: str = "/api/v1"
    
    # CORS
    CORS_ORIGINS: List[str] = [...]  # Next.js, frontend URLs
    
    # Milvus
    USE_MILVUS_LITE: bool = True      # dev: lite, prod: cloud
    MILVUS_LITE_PATH: str = "data/milvus_lite.db"
    MILVUS_URI: str = ""               # prod: https://xxx.zillizcloud.com
    MILVUS_TOKEN: str = ""             # prod: token
    
    # Vector Config
    EMBEDDING_MODEL: str = "text-embedding-3-small"
    EMBEDDING_DIMENSION: int = 1536
    VECTOR_METRIC_TYPE: str = "COSINE"
    
    # Database
    DATABASE_URL: str = "postgresql://..."
    AUTH_DATABASE_URL: str = ""        # Better Auth DB
    
    # Credits
    INITIAL_USER_CREDITS: int = 100    # 0 for prod
    
    # OpenAI
    OPENAI_API_KEY: str = ""
    
    # R2 Storage
    R2_ACCOUNT_ID: str = ""
    R2_ACCESS_KEY_ID: str = ""
    R2_SECRET_ACCESS_KEY: str = ""
    R2_BUCKET_NAME: str = "fltr-datasets"
    R2_PRESIGNED_URL_EXPIRY: int = 3600
    
    # Modal
    USE_MODAL_FOR_PROCESSING: bool = True
    MODAL_WEBHOOK_URL: str = ""        # From modal deploy
    
    # Auth
    JWT_SECRET_KEY: str = "change-me"
    JWT_ALGORITHM: str = "HS256"
```

### 5.2 Environment Files

**Development (.env):**
```bash
ENVIRONMENT=development
DATABASE_URL=postgresql://admin:password@localhost:5432/fltr
USE_MILVUS_LITE=true
MILVUS_LITE_PATH=data/milvus_lite.db
OPENAI_API_KEY=sk-...
R2_ACCOUNT_ID=...
R2_ACCESS_KEY_ID=...
R2_SECRET_ACCESS_KEY=...
MODAL_WEBHOOK_URL=https://username--fltr-document-processing-fastapi-app.modal.run/process
USE_MODAL_FOR_PROCESSING=true
INITIAL_USER_CREDITS=100
```

**Production (.env.prod):**
```bash
ENVIRONMENT=production
DATABASE_URL=postgresql://user:pass@prod-host:5432/fltr_prod
USE_MILVUS_LITE=false
MILVUS_URI=https://xxx.zillizcloud.com
MILVUS_TOKEN=...
OPENAI_API_KEY=sk-...
R2_ACCOUNT_ID=...
R2_ACCESS_KEY_ID=...
R2_SECRET_ACCESS_KEY=...
MODAL_WEBHOOK_URL=https://prod-webhook-url
USE_MODAL_FOR_PROCESSING=true
INITIAL_USER_CREDITS=0
```

### 5.3 Feature Flags (For New Features)

**Add to config.py:**
```python
class Settings(BaseSettings):
    # Feature flags
    ENABLE_HIERARCHICAL_CHUNKING: bool = True
    ENABLE_CHANGE_DETECTION: bool = True
    ENABLE_KNOWLEDGE_GRAPH: bool = False
    ENABLE_RERANKING: bool = False
    ENABLE_HYBRID_SEARCH: bool = False
    ENABLE_CUSTOM_METADATA: bool = True
    
    # Feature costs (credits)
    CREDIT_COST_DOCUMENT_UPLOAD: int = 1
    CREDIT_COST_EMBEDDING_SEARCH: int = 1
    CREDIT_COST_ADVANCED_SEARCH: int = 2
    CREDIT_COST_RERANKING: int = 1
```

---

## 6. TESTING INFRASTRUCTURE

### 6.1 Current Test Setup

**Location:** `/fastapi/tests/`

**Test Types:**
```
tests/
├─ conftest.py                        # Global fixtures
├─ fixtures/
│  ├─ database.py                     # DB fixtures
│  └─ ...
├─ test_auth_router.py
├─ test_dataset_router.py
├─ test_credit_service.py
├─ test_embedding_router.py
├─ test_config.py
└─ agents/                            # MCP agent tests
   └─ test_mcp_agent_scenarios.py
```

**Conftest Fixtures:**
```python
# Location: /fastapi/tests/conftest.py

@pytest.fixture
def test_db():
    """Test database session"""
    # Create temp DB
    # Setup schema
    yield session
    # Cleanup

@pytest.fixture
def test_client():
    """FastAPI test client"""
    return TestClient(app)

@pytest.fixture
def auth_headers():
    """Authentication headers"""
    return {"Authorization": "Bearer token"}

@pytest.fixture
def mock_milvus():
    """Mock Milvus client"""
    # Returns mock with search/insert/etc
```

### 6.2 Test Pattern

```python
# Example test
def test_search_embeddings(test_client, test_db, mock_milvus):
    # 1. Setup
    dataset = create_test_dataset(test_db)
    insert_test_vectors(mock_milvus, dataset.id)
    
    # 2. Execute
    response = test_client.post(
        f"/api/v1/datasets/{dataset.id}/embeddings/search",
        json={"query": "test", "limit": 10},
        headers={"Authorization": "Bearer token"}
    )
    
    # 3. Assert
    assert response.status_code == 200
    assert len(response.json()["results"]) == 10
```

### 6.3 pytest.ini Configuration

```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = -v --tb=short
```

### 6.4 For New Features

**Test Strategy:**
```
For each new feature:
1. Unit tests (service logic)
2. Integration tests (service + repo + DB)
3. Endpoint tests (router + service)
4. Modal tests (if processing involved)

Naming pattern: test_<feature>_<scenario>.py

Examples:
- test_hierarchical_chunking_basic.py
- test_change_detection_versioning.py
- test_knowledge_graph_extraction.py
```

---

## 7. DATA FLOW DETAILS

### 7.1 Document Upload Flow (Detailed)

```
Step 1: Client requests upload URLs
└─ POST /api/v1/datasets
   ├─ Service: DatasetService.create_dataset()
   ├─ Repo: DatasetRepository.create()
   └─ Generate presigned R2 URLs via R2StorageService
      └─ boto3 s3.generate_presigned_post()

Step 2: Client uploads file directly to R2
└─ PUT/POST to presigned URL (bypasses FastAPI)
   └─ File stored at: {dataset_id}/{filename}

Step 3: R2 triggers webhook
└─ POST /api/v1/webhooks/upload-complete
   ├─ Body: {"dataset_id": str, "object_key": str, "size": int}
   ├─ Create Document record (status: "uploaded")
   ├─ Charge DOCUMENT_UPLOAD credits
   │  └─ CreditService.charge_operation()
   │     └─ Create credit_transaction row
   └─ Trigger Modal processing
      └─ POST to MODAL_WEBHOOK_URL
         ├─ Payload: {"dataset_id": str, "object_key": str, "task_id": str}
         └─ Modal.spawn(process_document_modal, args)

Step 4: Modal processes document
└─ async def process_document_modal(dataset_id, object_key, task_id)
   ├─ Download file from R2
   ├─ Validate PDF (if .pdf)
   ├─ Parse: PyMuPDF4LLM (semantic) or Docling (fallback)
   ├─ Extract images → Vision models (GPT-4V, Claude, Gemini)
   ├─ Upload extracted images back to R2
   ├─ Chunk document (1000 chars, 200 overlap)
   ├─ Generate embeddings (OpenAI, batch of 100)
   ├─ Insert vectors to Milvus (with metadata)
   ├─ Update Document: status="ready", chunk_count=N
   ├─ Call _update_dataset_status(dataset_id)
   │  └─ Count ready/failed/processing docs
   │     └─ Update Dataset.status accordingly
   └─ On error: set status="failed", error msg, trigger credit refund

Step 5: Credit refund (if error)
└─ Modal calls POST /api/v1/credits/refund
   ├─ Find original DOCUMENT_UPLOAD transaction
   ├─ Create refund transaction
   └─ Update user.credits
```

### 7.2 Search Flow (Detailed)

```
Step 1: User initiates search
└─ POST /api/v1/datasets/{dataset_id}/embeddings/search
   ├─ Body: {"query": str, "limit": int, "filter_expr": str (optional)}
   ├─ Auth check
   ├─ Credit check (requires 1 credit)
   │  └─ Deduct credit immediately
   └─ Route to EmbeddingService.search_embeddings()

Step 2: Service processes search
└─ EmbeddingService.search_embeddings(dataset_id, query)
   ├─ Verify dataset exists and status="ready"
   ├─ Generate query embedding
   │  └─ OpenAI client.embeddings.create()
   │     └─ Returns vector[1536]
   └─ Call _search_milvus(dataset_id, query_vector, query)

Step 3: Milvus search
└─ milvus.search()
   ├─ Filter: dataset_id == "{dataset_id}"
   ├─ If query.filter_expr: AND with additional filters
   │  └─ Example: 'chunk_type == "ocr" && page > 10'
   ├─ Similarity search on vector field
   ├─ Return top-k results
   │  └─ Each result: {id, vector, distance, entity{...}}
   └─ Output fields: text, filename, chunk_index, etc.

Step 4: Format results
└─ For each result entity:
   ├─ Get text from Milvus
   ├─ Get metadata (filename, chunk_index, etc.)
   ├─ If image_r2_key present:
   │  └─ Generate presigned download URL
   │     └─ boto3 s3.generate_presigned_url()
   └─ Return EmbeddingSearchResult

Step 5: Return to client
└─ Response 200
   {
       "query": str,
       "dataset_id": str,
       "results": [
           {
               "id": int,
               "text": str,
               "distance": float,
               "meta_data": {
                   "filename": str,
                   "image_url": str (presigned, expires 1hr),
                   ...
               }
           },
           ...
       ]
   }
```

---

## 8. KEY INTEGRATION POINTS FOR NEW FEATURES

### 8.1 Where to Integrate

```
Feature Type → Integration Point → How to Extend

Document Metadata
├─ Add fields to Document model (/fastapi/models/document.py)
├─ Add fields to documents table (migration)
├─ Pass metadata through Modal pipeline
└─ Index in Milvus (new fields)

Search Enhancements
├─ Extend EmbeddingService._search_milvus()
├─ Add filter expressions
├─ Rerank results in EmbeddingService
└─ Return additional metadata in EmbeddingSearchResult

Processing Pipeline
├─ Extend Modal services in /modal/services/
├─ Modify chunk_document() or parse_document()
├─ Store results in Milvus (new fields)
└─ Update in PostgreSQL (new Document fields)

API Endpoints
├─ Add new router in /fastapi/routers/
├─ Create service + repository
├─ Use @requires_credits decorator
└─ Follow existing pattern (Router → Service → Repo → DB)

Database
├─ Add columns to datasets or documents
├─ Add new tables if needed
├─ Create Alembic migration
└─ Update SQLModel definitions
```

### 8.2 Existing Extension Points

**Modal Services (Add New Services):**
```python
# /modal/services/new_feature.py
async def feature_function(params):
    # Called from process_document_modal()
    ...

# In modal_app.py, call it:
result = await new_feature_function(...)
```

**Milvus Fields (Add Without Migration):**
```python
# In vector_store.py::create_collection()
fields.append(
    FieldSchema(
        name="new_field",
        dtype=DataType.VARCHAR,
        max_length=512
    )
)

# Then insert:
data.append({
    "vector": embedding,
    "new_field": value,
    ...
})
```

**Credit Operations (Add New Operation Types):**
```python
# In models/credits.py
class CreditOperation(str, Enum):
    DOCUMENT_UPLOAD = "document_upload"
    EMBEDDING_SEARCH = "embedding_search"
    NEW_FEATURE = "new_feature"  # Add here
    
# Then use:
@requires_credits(
    amount=N,
    operation=CreditOperation.NEW_FEATURE,
    resource_id_param="dataset_id"
)
```

---

## 9. POTENTIAL CONFLICTS & CHALLENGES

### 9.1 Architecture Constraints

| Constraint | Impact | Workaround |
|-----------|--------|-----------|
| Single Milvus collection | All datasets mixed, filter at query time | Sufficient for <5M vectors; can partition later |
| Shared dataset DB | No per-dataset isolation at DB level | Enforce isolation in application logic |
| Modal deployment | 30s webhook timeout, ~10s cold start | Keep Modal functions stateless; use fast I/O |
| FastAPI sync services | Can't do long-running async ops | Use Modal for heavy lifting; service is orchestrator |
| PostgreSQL + Milvus sync | Two sources of truth | Modal updates both; use transaction IDs for audit trail |
| Chunking strategy | Fixed size, ignores document structure | Add hierarchy fields; don't break existing chunks |

### 9.2 Common Integration Issues

**Problem: Modal timeout**
- Symptom: Document processing takes >1hr
- Cause: Large documents or slow Vision API
- Solution: Chunk in Modal before processing; use batch operations

**Problem: Milvus filter explosion**
- Symptom: Search queries slow with complex filters
- Cause: Too many nested AND/OR conditions
- Solution: Pre-filter in FastAPI before calling Milvus

**Problem: R2 presigned URL expiry**
- Symptom: Image URLs expire during user session
- Cause: 1-hour default expiry too short
- Solution: Regenerate URLs on search; cache images in CDN

**Problem: Embedding dimension mismatch**
- Symptom: "Dimension mismatch" error in Milvus
- Cause: Different embedding models have different dims
- Solution: Per-dataset embedding_dimension field (exists); enforce consistency

**Problem: Document re-processing**
- Symptom: Duplicate chunks in Milvus
- Cause: Same file uploaded twice
- Solution: Check `document.object_key` uniqueness before processing

### 9.3 Feature Implementation Risks

**Risk: Breaking change to chunking**
- If you change chunk size, old chunks incompatible with new
- Mitigation: Add version field to chunks; support multiple versions in search

**Risk: Milvus schema rigidity**
- Can't add non-default fields to existing collections
- Mitigation: Create new collection with version suffix; migrate on read

**Risk: Credit system abuse**
- User can spam searches to exhaust credits
- Mitigation: Rate limiting on credit gate; per-user per-minute limits

**Risk: Modal cold starts**
- First document processing request takes ~30s
- Mitigation: Keep 1 container warm (costs ~$0.30/day); use min_containers=1

**Risk: Image extraction failures**
- Vision API might fail or return low-confidence results
- Mitigation: Use fallback providers; score confidence; allow manual review

---

## 10. RECOMMENDED IMPLEMENTATION APPROACH FOR 8 FEATURES

### Phase 1: Low-Hanging Fruit (2-3 features)
1. **Enhanced Metadata** - Add custom fields to Document
2. **Simple Reranking** - Post-process Milvus results with cross-encoder
3. **Usage Analytics** - New access_logs table, no chunking changes

### Phase 2: Core Infrastructure (2-3 features)
4. **Hierarchical Chunking** - Extend chunk with hierarchy_level, parent_chunk_id
5. **Change Detection** - Hash content, version field, diff algorithm
6. **Hybrid Search** - Add keyword search alongside vector search

### Phase 3: Advanced (2-3 features)
7. **Knowledge Graph** - Extract entities during parsing, new entities table
8. **Semantic Caching** - Cache embeddings for identical queries

### Implementation Pattern (For Each Feature)

```python
# 1. Add to config.py
ENABLE_FEATURE_NAME: bool = True
CREDIT_COST_FEATURE: int = 1

# 2. Add to models
class FeatureData(SQLModel, table=True):
    __tablename__ = "feature_data"
    # Fields

# 3. Add to repository
class FeatureRepository:
    def query_feature(self):
        pass

# 4. Add to service
class FeatureService:
    def __init__(self, session: Session = Depends(get_session)):
        self.repo = FeatureRepository(session)
    
    def feature_logic(self):
        pass

# 5. Add to router
@router.post("/feature")
@requires_credits(amount=1, operation=CreditOperation.FEATURE)
async def feature_endpoint(..., service: FeatureService = Depends()):
    return service.feature_logic()

# 6. Add tests
def test_feature():
    pass
```

---

## SUMMARY TABLE: Integration Checklist for Each Feature

| Area | Artifact | Changes Needed |
|------|----------|-----------------|
| **Configuration** | config.py | Add ENABLE_FEATURE flag, credit cost |
| **Database** | Document/Dataset model | Add fields, create migration |
| **Vector Store** | Milvus schema | Add fields to collection |
| **Modal** | document_processor.py | Extract feature data |
| **FastAPI** | Service + Repository | Query/update feature data |
| **API** | Router | Add endpoints with credit gate |
| **Testing** | tests/ | Unit + integration + endpoint tests |

---

**Generated:** November 9, 2025  
**For:** Planning 8 new RAG features  
**Status:** Ready for implementation
