# GraphRAG Per Dataset Implementation Plan

## Overview

Add GraphRAG (Graph-based Retrieval Augmented Generation) capabilities per dataset with support for incremental file uploads. The design integrates with the existing RAG infrastructure while adding knowledge graph capabilities for enhanced multi-hop reasoning and context retrieval.

---

## Architecture

### Current Flow
```
Document Upload → R2 → Webhook → Modal Pipeline → Milvus Storage
                                     ↓
                 (Download → Parse → Chunk → Embed → Store → DB_Update)
```

### Proposed GraphRAG Flow
```
Document Upload → R2 → Webhook → Modal Pipeline → [Milvus + Neo4j]
                                     ↓
     (Download → Parse → Chunk → EXTRACT_GRAPH → Embed → Store → DB_Update)
                                     ↓
                         [Entity Resolution + Graph Update]
```

---

## Decision: Neo4j vs PostgreSQL

### Recommendation: **Neo4j AuraDB**

| Factor | PostgreSQL + NetworkX | Neo4j AuraDB |
|--------|----------------------|--------------|
| **Graph Traversal** | O(n) JOINs per hop | O(1) index-free adjacency |
| **Query Language** | Complex recursive SQL | Native Cypher |
| **Scale** | <50K entities OK | 100K+ entities performant |
| **Infrastructure** | Already have | New service (~$65/mo) |
| **Graph Algorithms** | NetworkX (load to memory) | GDS library (native) |
| **Visualization** | Build custom | Neo4j Browser/Bloom |

### Why Neo4j for GraphRAG:
1. **Multi-hop queries are core to GraphRAG** - Neo4j excels at "find all entities within 2 hops"
2. **Incremental updates** - Native graph mutations without full reload
3. **Cypher simplicity** - `MATCH (a)-[:RELATED*1..3]->(b)` vs recursive CTEs
4. **Future-proof** - Community detection, PageRank, path finding built-in

### Managed Options:
- **Neo4j AuraDB Free**: 200K nodes, 400K relationships (good for dev)
- **Neo4j AuraDB Pro**: ~$65/mo, auto-scaling, backups

---

## 1. Graph Storage (Neo4j)

### Node Types

```cypher
// Entity Node
(:Entity {
  id: UUID,
  dataset_id: UUID,
  name: String,
  canonical_name: String,
  entity_type: String,  // PERSON, ORGANIZATION, CONCEPT, etc.
  description: String,
  embedding: [Float],   // 1536-dim for similarity
  mention_count: Int,
  created_at: DateTime,
  updated_at: DateTime
})

// Document Source Node (for provenance tracking)
(:DocumentSource {
  id: UUID,
  document_id: UUID,
  dataset_id: UUID,
  chunk_id: String,
  text_span: String
})
```

### Relationship Types

```cypher
// Entity-to-Entity relationships
(:Entity)-[:WORKS_FOR {weight: Float, evidence_count: Int}]->(:Entity)
(:Entity)-[:LOCATED_IN]->(:Entity)
(:Entity)-[:RELATED_TO]->(:Entity)
(:Entity)-[:PART_OF]->(:Entity)
(:Entity)-[:CREATED_BY]->(:Entity)
(:Entity)-[:REFERENCES]->(:Entity)
(:Entity)-[:DEPENDS_ON]->(:Entity)
(:Entity)-[:CONTRADICTS]->(:Entity)
(:Entity)-[:SUPPORTS]->(:Entity)

// Provenance tracking
(:Entity)-[:MENTIONED_IN]->(:DocumentSource)
(:Entity)-[:EXTRACTED_FROM {confidence: Float}]->(:DocumentSource)
```

### Indexes

```cypher
CREATE INDEX entity_dataset FOR (e:Entity) ON (e.dataset_id);
CREATE INDEX entity_name FOR (e:Entity) ON (e.canonical_name);
CREATE INDEX entity_type FOR (e:Entity) ON (e.entity_type);
CREATE VECTOR INDEX entity_embedding FOR (e:Entity) ON (e.embedding)
  OPTIONS {indexConfig: {`vector.dimensions`: 1536, `vector.similarity_function`: 'cosine'}};
```

---

## 2. PostgreSQL Metadata Tables

Keep lightweight metadata in PostgreSQL for fast lookups:

**`graph_metadata`** (per-dataset config)
```sql
CREATE TABLE graph_metadata (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id UUID UNIQUE NOT NULL REFERENCES datasets(id) ON DELETE CASCADE,

    -- Stats (synced from Neo4j periodically)
    entity_count INTEGER DEFAULT 0,
    relationship_count INTEGER DEFAULT 0,

    -- Configuration
    entity_types JSONB DEFAULT '[]',
    relationship_types JSONB DEFAULT '[]',
    extraction_config JSONB DEFAULT '{}',

    -- Neo4j connection info (if per-dataset isolation needed)
    neo4j_database VARCHAR(128) DEFAULT 'neo4j',

    -- Cache versioning
    graph_version INTEGER DEFAULT 0,
    last_sync_at TIMESTAMP,

    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

**Dataset Model Extension**:
```python
# Add to /fastapi/models/dataset.py
enable_graph_rag: bool = Field(default=False)
graph_config: Optional[Dict[str, Any]] = Field(default=None, sa_column=Column(JSON))
```

---

## 3. Entity & Relationship Extraction

### New Pipeline Stage: EXTRACT_GRAPH

Add after CHUNK stage in Modal pipeline.

**GraphExtractionService** (`/modal/services/graph_extraction_service.py`)

```python
class GraphExtractionService:
    """Extract entities and relationships from document chunks using LLM."""

    async def extract_from_chunks(
        self,
        chunks: List[DocumentChunk],
        dataset_id: str,
        document_id: str,
        config: GraphExtractionConfig
    ) -> GraphExtractionResult:
        """
        Process chunks in batches, extract entities/relationships.
        Uses GPT-4o-mini with structured JSON output.
        """

    async def _extract_batch(
        self,
        chunk_batch: List[DocumentChunk],
        context: str
    ) -> BatchExtractionResult:
        """
        LLM call to extract from batch of 5 chunks.
        Returns entities with: name, type, description, aliases
        Returns relationships with: source, target, type, description
        """
```

**Extraction Prompt Template**:
```
Extract entities and relationships from the following text chunks.

Entity Types: PERSON, ORGANIZATION, LOCATION, DATE, EVENT, PRODUCT,
              CONCEPT, DOCUMENT, TECHNOLOGY, LAW, REGULATION, METRIC, PROCESS

Relationship Types: WORKS_FOR, LOCATED_IN, RELATED_TO, PART_OF,
                   CREATED_BY, OCCURRED_AT, REFERENCES, DEPENDS_ON,
                   CONTRADICTS, SUPPORTS, PRECEDED_BY, FOLLOWED_BY

Return JSON:
{
  "entities": [
    {"name": "...", "type": "...", "description": "...", "aliases": [...]}
  ],
  "relationships": [
    {"source": "...", "target": "...", "type": "...", "description": "..."}
  ]
}

Text chunks:
{chunks}
```

### Entity Resolution

**EntityResolutionService** (`/modal/services/entity_resolution_service.py`)

```python
class EntityResolutionService:
    """Resolve and deduplicate entities across documents."""

    async def resolve_entities(
        self,
        new_entities: List[ExtractedEntity],
        dataset_id: str,
        similarity_threshold: float = 0.85
    ) -> List[ResolvedEntity]:
        """
        1. Generate embeddings for new entity names + descriptions
        2. Query Neo4j for similar existing entities (vector similarity)
        3. Merge if similarity > threshold
        4. Return resolved entities (new or merged)
        """

    async def _find_similar_entities(
        self,
        entity: ExtractedEntity,
        dataset_id: str,
        top_k: int = 5
    ) -> List[SimilarEntity]:
        """Vector similarity search in Neo4j."""
```

---

## 4. Incremental Graph Updates

**GraphUpdateService** (`/modal/services/graph_update_service.py`)

```python
class GraphUpdateService:
    """Manage incremental graph updates as documents are added/removed."""

    async def add_document_to_graph(
        self,
        dataset_id: str,
        document_id: str,
        extraction_result: GraphExtractionResult
    ) -> GraphUpdateResult:
        """
        1. Resolve entities against existing graph
        2. Create new entities, merge duplicates
        3. Create relationships with evidence tracking
        4. Update mention counts
        5. Increment graph_version in PostgreSQL
        """

    async def remove_document_from_graph(
        self,
        dataset_id: str,
        document_id: str
    ) -> GraphUpdateResult:
        """
        1. Find all DocumentSource nodes for this document
        2. For each linked Entity:
           - Decrement mention_count
           - If mention_count == 0: delete entity
        3. Remove relationships with only this document as evidence
        4. Delete DocumentSource nodes
        5. Increment graph_version
        """
```

### Cypher for Document Removal:
```cypher
// Find entities to potentially remove
MATCH (e:Entity)-[:MENTIONED_IN]->(ds:DocumentSource {document_id: $doc_id})
WITH e, count(ds) as doc_mentions

// Get total mentions for each entity
MATCH (e)-[:MENTIONED_IN]->(all_ds:DocumentSource)
WITH e, doc_mentions, count(all_ds) as total_mentions

// Delete entities that only appeared in this document
WHERE total_mentions = doc_mentions
DETACH DELETE e

// For remaining entities, just remove the document source link
MATCH (e:Entity)-[r:MENTIONED_IN]->(ds:DocumentSource {document_id: $doc_id})
DELETE r, ds

// Update mention counts
MATCH (e:Entity)-[:MENTIONED_IN]->(ds:DocumentSource)
WHERE e.dataset_id = $dataset_id
WITH e, count(ds) as new_count
SET e.mention_count = new_count
```

---

## 5. Graph-Enhanced Retrieval

### Neo4jGraphService (`/fastapi/services/neo4j_graph_service.py`)

```python
class Neo4jGraphService:
    """Graph operations using Neo4j."""

    def __init__(self):
        self.driver = neo4j.GraphDatabase.driver(
            settings.NEO4J_URI,
            auth=(settings.NEO4J_USER, settings.NEO4J_PASSWORD)
        )

    async def find_entities_by_name(
        self,
        dataset_id: str,
        entity_names: List[str],
        fuzzy: bool = True
    ) -> List[GraphEntity]:
        """Find entities matching query terms."""

    async def get_subgraph(
        self,
        dataset_id: str,
        entity_ids: List[str],
        max_hops: int = 2,
        limit: int = 50
    ) -> SubgraphResult:
        """
        Extract subgraph around seed entities.
        Returns nodes and edges for context expansion.
        """

    async def find_paths(
        self,
        dataset_id: str,
        source_entity: str,
        target_entity: str,
        max_hops: int = 3
    ) -> List[GraphPath]:
        """Find shortest paths between entities."""
```

### Subgraph Extraction Cypher:
```cypher
// Get 2-hop neighborhood around seed entities
MATCH (seed:Entity)
WHERE seed.id IN $entity_ids AND seed.dataset_id = $dataset_id

CALL {
  WITH seed
  MATCH path = (seed)-[*1..2]-(neighbor:Entity)
  WHERE neighbor.dataset_id = $dataset_id
  RETURN path
  LIMIT 50
}

WITH collect(path) as paths
UNWIND paths as p
WITH nodes(p) as nodes, relationships(p) as rels
UNWIND nodes as n
UNWIND rels as r
RETURN collect(distinct n) as entities, collect(distinct r) as relationships
```

### GraphSearchStrategy (`/fastapi/services/search_strategies/graph_strategy.py`)

```python
class GraphSearchStrategy:
    """Combine vector search with graph context expansion."""

    async def search(
        self,
        dataset_id: str,
        query: str,
        top_k: int = 10,
        use_graph_expansion: bool = True,
        max_hops: int = 2
    ) -> List[EnrichedSearchResult]:
        """
        1. Vector search in Milvus → top chunks
        2. Extract entities from query (lightweight NER)
        3. Find matching entities in Neo4j graph
        4. Get subgraph context (2-hop neighborhood)
        5. Enrich results with graph context
        6. Re-rank based on entity overlap
        """
```

### Result Enrichment:
```python
@dataclass
class GraphContext:
    entities: List[GraphEntity]           # Entities found in chunk
    related_entities: List[GraphEntity]   # Neighbors from graph
    relationships: List[GraphRelationship]
    entity_descriptions: Dict[str, str]

@dataclass
class EnrichedSearchResult:
    chunk: SearchResult
    graph_context: Optional[GraphContext]
    entity_overlap_score: float  # How many query entities appear
```

---

## 6. API Endpoints

**New Router**: `/fastapi/routers/graph.py`

```python
@router.get("/datasets/{dataset_id}/graph/stats")
async def get_graph_stats(dataset_id: UUID) -> GraphStats:
    """Get entity/relationship counts and types."""

@router.get("/datasets/{dataset_id}/graph/entities")
async def list_entities(
    dataset_id: UUID,
    entity_type: Optional[str] = None,
    search: Optional[str] = None,
    limit: int = 50,
    offset: int = 0
) -> PaginatedEntities:
    """List entities with optional filtering."""

@router.get("/datasets/{dataset_id}/graph/entities/{entity_id}")
async def get_entity(dataset_id: UUID, entity_id: UUID) -> EntityDetail:
    """Get entity with all relationships."""

@router.get("/datasets/{dataset_id}/graph/subgraph")
async def get_subgraph(
    dataset_id: UUID,
    entity_ids: List[UUID],
    max_hops: int = 2
) -> SubgraphResponse:
    """Extract subgraph for visualization (nodes + edges JSON)."""

@router.post("/datasets/{dataset_id}/graph/search")
async def graph_search(
    dataset_id: UUID,
    request: GraphSearchRequest
) -> GraphSearchResponse:
    """Graph-enhanced search with context expansion."""
```

---

## 7. Files to Create/Modify

### New Files

| File | Purpose |
|------|---------|
| `/fastapi/models/graph.py` | SQLModel for GraphMetadata, Pydantic models for API |
| `/fastapi/services/neo4j_graph_service.py` | Neo4j connection and graph operations |
| `/fastapi/services/search_strategies/graph_strategy.py` | Graph-enhanced search |
| `/fastapi/routers/graph.py` | Graph API endpoints |
| `/fastapi/config/neo4j.py` | Neo4j connection settings |
| `/modal/services/graph_extraction_service.py` | LLM entity/relationship extraction |
| `/modal/services/entity_resolution_service.py` | Entity deduplication |
| `/modal/services/graph_update_service.py` | Incremental graph updates |
| `/fastapi/alembic/versions/xxxx_add_graph_metadata.py` | Migration for graph_metadata table |

### Modified Files

| File | Changes |
|------|---------|
| `/fastapi/models/dataset.py` | Add `enable_graph_rag`, `graph_config` |
| `/modal/constants.py` | Add `EXTRACT_GRAPH` stage |
| `/modal/services/pipeline_orchestrator.py` | Add graph extraction stage |
| `/fastapi/services/rag/base_rag_backend.py` | Add `search_with_graph` method |
| `/fastapi/services/rag/milvus_backend.py` | Implement graph-enhanced search |
| `/fastapi/models/embedding.py` | Add `use_graph_expansion` parameter |
| `/fastapi/pyproject.toml` | Add `neo4j` dependency |
| `/modal/pyproject.toml` | Add `neo4j` dependency |

---

## 8. Implementation Phases

### Phase 1: Infrastructure Setup
- Set up Neo4j AuraDB instance
- Add neo4j Python driver to dependencies
- Create `/fastapi/config/neo4j.py` with connection settings
- Create Alembic migration for `graph_metadata` table
- Add `enable_graph_rag` field to Dataset model

### Phase 2: Extraction Pipeline
- Implement `GraphExtractionService` (LLM extraction)
- Implement `EntityResolutionService` (embedding-based dedup)
- Add `EXTRACT_GRAPH` stage to Modal pipeline orchestrator
- Integrate with checkpoint system for resumability
- Test with sample documents

### Phase 3: Graph Management
- Implement `GraphUpdateService` for incremental updates
- Handle document deletion (cascade entity removal)
- Build `Neo4jGraphService` for queries
- Add subgraph extraction

### Phase 4: Search Integration
- Implement `GraphSearchStrategy`
- Integrate with existing Milvus search flow
- Add graph context to search results
- Update `EmbeddingSearchQuery` model

### Phase 5: API & Polish
- Create `/fastapi/routers/graph.py` endpoints
- Add visualization endpoint (Cytoscape.js compatible JSON)
- Testing and documentation
- Performance optimization

---

## 9. Cost Considerations

### Neo4j AuraDB
- **Free tier**: 200K nodes, 400K relationships (dev/small datasets)
- **Pro tier**: ~$65/month, auto-scaling

### LLM Extraction (GPT-4o-mini)
- ~$0.15/1M input tokens
- Average document: ~100K tokens processed
- **Cost per document**: ~$0.015
- **1,000 documents**: ~$15

### Optimization Strategies
- Batch processing (5 chunks per LLM call)
- Only extract for `enable_graph_rag=True` datasets
- Cache extraction results for re-processing
- Use GPT-4o-mini (sufficient quality, lower cost)

---

## 10. Configuration

### Environment Variables
```bash
# Neo4j
NEO4J_URI=neo4j+s://xxxxx.databases.neo4j.io
NEO4J_USER=neo4j
NEO4J_PASSWORD=xxxxx
NEO4J_DATABASE=neo4j

# Graph extraction
GRAPH_EXTRACTION_MODEL=gpt-4o-mini
GRAPH_EXTRACTION_BATCH_SIZE=5
ENTITY_RESOLUTION_THRESHOLD=0.85
```

### Per-Dataset Config (`graph_config` JSON)
```json
{
  "entity_types": ["PERSON", "ORGANIZATION", "CONCEPT"],
  "relationship_types": ["WORKS_FOR", "RELATED_TO"],
  "extraction_model": "gpt-4o-mini",
  "max_entities_per_chunk": 10,
  "enable_entity_resolution": true,
  "resolution_threshold": 0.85
}
```
