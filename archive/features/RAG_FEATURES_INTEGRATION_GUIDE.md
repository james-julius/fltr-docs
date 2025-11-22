# 8 New RAG Features - Integration Planning Guide

**Date:** November 9, 2025  
**Purpose:** High-level planning document for 8 new RAG feature implementations  
**Reference:** See ARCHITECTURE_DEEP_DIVE.md for detailed architecture

---

## QUICK FACTS ABOUT FLTR ARCHITECTURE

### What You Have (70% Enterprise RAG)
- ✅ Multi-format document parsing (PDF, DOCX, PPTX, etc.)
- ✅ Multimodal processing (Vision models + OCR)
- ✅ Serverless scalability (Modal)
- ✅ Vector search (Milvus)
- ✅ Credit-based usage tracking
- ✅ R2 object storage with presigned URLs
- ✅ PostgreSQL for metadata
- ✅ API authentication (Better Auth)

### What You're Missing (30% to Enterprise)
- ❌ Hierarchical document awareness
- ❌ Document versioning/change detection
- ❌ Semantic caching
- ❌ Advanced reranking
- ❌ Hybrid search (BM25 + vector)
- ❌ Knowledge graph extraction
- ❌ Custom metadata/tagging
- ❌ Usage analytics/audit logging

---

## PHASED APPROACH: 3 PHASES, 8 FEATURES

### PHASE 1: QUICK WINS (Weeks 1-2)

#### Feature 1: Custom Metadata & Tagging
**Effort:** 2 days  
**Dependencies:** None  

What you need:
- Add `custom_metadata: JSON` field to Document model
- Add `/api/v1/datasets/{id}/documents/{id}/metadata` PATCH endpoint
- Store in PostgreSQL documents table
- Pass through Modal → Milvus search results

```sql
ALTER TABLE documents ADD COLUMN custom_metadata JSON;
```

Integration:
- DocumentRepository.update_metadata()
- DocumentService.update_document_metadata()
- New router endpoint with @requires_credits

---

#### Feature 2: Usage Analytics
**Effort:** 3 days  
**Dependencies:** None

What you need:
- New table: `document_access_logs`
- Track: search, download, view operations
- Route: `/api/v1/analytics/dataset-stats`

```sql
CREATE TABLE document_access_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    dataset_id UUID,
    document_id UUID,
    user_id UUID,
    access_type VARCHAR(20),  -- search|download|view
    query_text TEXT,
    timestamp TIMESTAMP,
    INDEX idx_dataset_id (dataset_id),
    INDEX idx_user_id (user_id),
    INDEX idx_timestamp (timestamp)
);
```

Integration:
- Add logging to embedding search endpoint
- Service: AnalyticsService
- No changes to core pipeline

---

#### Feature 3: Simple Reranking
**Effort:** 2 days  
**Dependencies:** Feature 1

What you need:
- Post-process Milvus results with cross-encoder
- Use open-source model (BAAI/bge-reranker-base)
- Add to EmbeddingService._search_milvus()

Integration:
- Import: `from sentence_transformers import CrossEncoder`
- Call after Milvus search:
```python
# Rerank results
reranker = CrossEncoder('BAAI/bge-reranker-base')
texts = [r.text for r in search_results]
scores = reranker.predict([(query.query, t) for t in texts])
# Re-sort by scores
```
- Optional query param: `?use_reranking=true`
- Charge 1 extra credit if enabled

---

### PHASE 2: CORE INFRASTRUCTURE (Weeks 3-4)

#### Feature 4: Hierarchical Chunking
**Effort:** 5 days  
**Dependencies:** None, but affects Modal pipeline

What you need:
- Extend Document model with hierarchy metadata
- Modify Modal's `chunk_document()` function
- Add hierarchy awareness to Milvus schema

```sql
ALTER TABLE documents ADD COLUMN (
    hierarchy_json JSON,
    semantic_boundaries BOOLEAN
);
```

Milvus additions:
```python
# In vector_store.py::create_collection()
fields.extend([
    FieldSchema(name="parent_chunk_id", dtype=DataType.INT64, default_value=0),
    FieldSchema(name="hierarchy_level", dtype=DataType.INT64, default_value=0),
])
```

Integration:
- New logic in `chunk_document()`:
  - Parse markdown headers (H1, H2, H3)
  - Track section relationships
  - Store parent-child links
- Milvus: Insert hierarchy_level + parent_chunk_id
- Search: Optional `include_related_chunks=true` to return siblings

Challenge: Backwards compatibility - don't re-chunk existing documents

---

#### Feature 5: Document Versioning & Change Detection
**Effort:** 4 days  
**Dependencies:** Feature 4

What you need:
- Track document versions in PostgreSQL
- Hash content for change detection
- Store previous version references

```sql
ALTER TABLE documents ADD COLUMN (
    version INT DEFAULT 1,
    hash_content VARCHAR(64),        -- SHA-256
    previous_version_id UUID,
    change_summary TEXT
);
```

Integration:
- Modal: Calculate hash before processing
```python
import hashlib
hash_content = hashlib.sha256(file_bytes).hexdigest()
```
- Check if hash exists → skip reprocessing
- If new hash → create new version record
- DocumentRepository.create_new_version()
- API: GET /api/v1/datasets/{id}/documents/{id}/versions

---

#### Feature 6: Hybrid Search (Keyword + Vector)
**Effort:** 6 days  
**Dependencies:** Feature 4

What you need:
- Add BM25 keyword search alongside vector search
- Combine scores: 0.7 * vector_score + 0.3 * keyword_score

Integration:
- Use PostgreSQL full-text search OR
- Use Elasticsearch sidecar (optional, complex)
- Simplest: Store text index in PostgreSQL

```sql
ALTER TABLE documents ADD FULLTEXT INDEX ft_text (text);
```

EmbeddingService extension:
```python
def _hybrid_search(self, dataset_id, query_text, limit):
    # Vector search
    vector_results = self._search_milvus(...)
    
    # Keyword search
    keyword_results = self._search_postgres_fulltext(...)
    
    # Merge scores
    combined = merge_results(vector_results, keyword_results, 
                            weights=(0.7, 0.3))
    return combined[:limit]
```

API: POST /api/v1/datasets/{id}/embeddings/hybrid-search
Charge: 2 credits (more expensive than vector-only)

---

### PHASE 3: ADVANCED (Weeks 5-6)

#### Feature 7: Knowledge Graph Extraction
**Effort:** 7 days  
**Dependencies:** Feature 4

What you need:
- Extract entities during document processing
- Build relationship graph
- New tables: entities, relationships

```sql
CREATE TABLE entities (
    id UUID PRIMARY KEY,
    dataset_id UUID,
    document_id UUID,
    entity_text VARCHAR,
    entity_type VARCHAR,      -- PERSON|ORGANIZATION|LOCATION|...
    confidence FLOAT,
    created_at TIMESTAMP,
    INDEX idx_dataset_id (dataset_id)
);

CREATE TABLE entity_relationships (
    id UUID PRIMARY KEY,
    entity_1_id UUID,
    entity_2_id UUID,
    relationship_type VARCHAR,  -- RELATED_TO|MENTIONED_IN|...
    weight FLOAT,
    created_at TIMESTAMP,
    FOREIGN KEY (entity_1_id) REFERENCES entities(id),
    FOREIGN KEY (entity_2_id) REFERENCES entities(id)
);
```

Integration:
- Modal: Use spaCy or transformer-based NER
```python
# In document_processor.py
import spacy
nlp = spacy.load("en_core_web_sm")
doc = nlp(parsed_document["content"])
entities = [(ent.text, ent.label_) for ent in doc.ents]
```
- Store in PostgreSQL
- New service: KnowledgeGraphService
- API: GET /api/v1/datasets/{id}/knowledge-graph

Challenge: Scaling NER for large documents - batch processing

---

#### Feature 8: Semantic Caching
**Effort:** 5 days  
**Dependencies:** Features 1-3 complete

What you need:
- Cache query embeddings + results
- Detect similar queries using distance < 0.01
- Simple Redis-based caching

Integration:
- Add Redis config to config.py:
```python
REDIS_URL: str = "redis://localhost:6379/1"
SEMANTIC_CACHE_TTL: int = 3600  # 1 hour
```

- EmbeddingService modification:
```python
async def search_embeddings(self, dataset_id, query):
    # Check cache
    cached = await redis_client.get(f"query:{query_vector}:{dataset_id}")
    if cached:
        return cached
    
    # Execute search
    results = self._search_milvus(...)
    
    # Cache results
    await redis_client.setex(
        f"query:{query_vector}:{dataset_id}",
        self.cache_ttl,
        json.dumps(results)
    )
    return results
```

Charge: No extra credits (cache hit = free)

---

## IMPLEMENTATION PRIORITIES

### Must Do First (Required for other features)
1. Feature 1: Custom Metadata
2. Feature 4: Hierarchical Chunking

### Should Do Early (Foundation for advanced features)
3. Feature 2: Usage Analytics
4. Feature 5: Document Versioning

### Nice to Have (Optional enhancements)
5. Feature 3: Simple Reranking
6. Feature 6: Hybrid Search
7. Feature 7: Knowledge Graph
8. Feature 8: Semantic Caching

---

## DEVELOPMENT WORKFLOW

### For Each Feature:

```
1. CREATE BRANCH
   git checkout -b feature/hierarchical-chunking

2. DATABASE CHANGES (if needed)
   a) Create Alembic migration
   b) Test migration locally
   c) Commit migration

3. MODEL CHANGES (SQLModel)
   a) Add fields to /fastapi/models/
   b) Update repositories
   c) Commit models

4. SERVICE LOGIC (Business layer)
   a) Create service class
   b) Add to /fastapi/services/
   c) Create repository if needed
   d) Commit service

5. API ENDPOINTS (Routers)
   a) Create new router or extend existing
   b) Add @requires_credits decorator
   c) Commit endpoints

6. MODAL CHANGES (if processing affected)
   a) Extend /modal/services/
   b) Modify process_document_modal()
   c) Test locally with modal run
   d) Deploy: modal deploy modal_app.py
   e) Commit Modal changes

7. TESTS
   a) Unit tests (service logic)
   b) Integration tests (service + DB)
   c) Endpoint tests (router)
   d) Modal tests (if applicable)
   e) Commit tests

8. DOCUMENTATION
   a) Update ARCHITECTURE_DEEP_DIVE.md
   b) Add to feature flag docs
   c) Create CHANGELOG entry

9. PULL REQUEST
   git push origin feature/hierarchical-chunking
   Create PR, get review, merge
```

---

## TESTING STRATEGY

### Unit Tests
```python
# tests/test_hierarchical_chunking_service.py
def test_chunk_extraction_hierarchy():
    # Test chunk parsing preserves hierarchy
    
def test_parent_child_relationships():
    # Test parent_chunk_id links are correct
```

### Integration Tests
```python
# tests/test_hierarchical_chunking_integration.py
def test_hierarchy_data_persists():
    # Create document → Parse → Check DB → Verify hierarchy
    
def test_hierarchy_in_milvus():
    # Verify hierarchy fields in Milvus collection
```

### Endpoint Tests
```python
# tests/test_hierarchical_chunking_router.py
def test_search_with_hierarchy():
    # POST /embeddings/search with hierarchy filter
```

---

## RISK MITIGATION

### Backwards Compatibility
- Don't delete existing columns
- Don't remove Milvus fields
- Use feature flags to enable/disable features
- Test migrations with production data backup

### Data Integrity
- Transaction IDs for audit trail
- Idempotent operations (can retry safely)
- Soft deletes (never hard-delete documents)
- Regular backups before major changes

### Performance
- Add indexes on new searchable fields
- Test with realistic dataset sizes (100k+ documents)
- Monitor Modal cold starts
- Profile Milvus query latency

---

## COST IMPACT

### Storage
- Custom metadata: ~100 bytes per document → ~100MB per 1M docs
- Versioning: ~50 bytes per version → minimal
- Analytics logs: ~200 bytes per query → ~20GB per 1M queries
- Knowledge graph: ~500 bytes per entity → variable

### Computation
- Reranking: +0.5s per search (use reranker cache)
- Hybrid search: +100ms per search
- NER extraction: +2s per document
- Semantic caching: Saves computation, reduces cost

### Credits
```
Current:
- Document upload: 1 credit
- Embedding search: 1 credit

After features:
- Advanced search (reranking): 2 credits
- Hybrid search: 2 credits
- Knowledge graph query: 3 credits
- Standard search: 1 credit (unchanged)
```

---

## SUCCESS METRICS

Track these during implementation:

1. **Feature Adoption**
   - % of searches using new features
   - Time-to-value (days to first user)

2. **Performance**
   - Search latency (target: <500ms)
   - Modal processing time (target: <2min for 100-page PDF)
   - Cache hit rate (target: >50%)

3. **Quality**
   - Test coverage (target: >80%)
   - Error rate (target: <0.1%)
   - User satisfaction (feedback loop)

4. **Cost**
   - Cost per search (trending down with caching)
   - Storage growth rate
   - API call reduction (due to caching)

---

## REFERENCE

**Full Architecture Document:** `/Users/jamesjulius/Coding/FLTR/ARCHITECTURE_DEEP_DIVE.md`

**Key Files to Know:**
- `/fastapi/config.py` - Feature flags, settings
- `/fastapi/models/` - SQLModel definitions
- `/fastapi/services/` - Business logic
- `/fastapi/repositories/` - Data access
- `/fastapi/routers/` - API endpoints
- `/modal/services/` - Processing pipeline
- `/fastapi/database/vector_store.py` - Milvus integration

**Commands:**
```bash
# Run tests
pytest /fastapi/tests

# Run local API
cd /fastapi && uvicorn main:app --reload

# Deploy Modal changes
cd /modal && modal deploy modal_app.py

# Create DB migration
cd /fastapi && alembic revision --autogenerate -m "Add feature X"
```

---

**Ready to start implementation!**
