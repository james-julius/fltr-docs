# Reranking Implementation Guide

## Overview

Reranking significantly improves search relevance by reordering initial semantic search results using a specialized relevance model. This implementation uses **Cohere Rerank v3.5**, which provides state-of-the-art relevance scoring.

## Setup

### 1. Get Cohere API Key

Sign up at [cohere.com](https://cohere.com) and get your API key from the dashboard.

**Pricing**: ~$1 per 1000 searches (very affordable for the quality improvement)

### 2. Add to Environment Variables

```bash
# In fastapi/.env
COHERE_API_KEY=your_cohere_api_key_here
COHERE_RERANK_MODEL=rerank-v3.5  # Or rerank-multilingual-v3.0 for multilingual
```

### 3. Restart FastAPI Server

```bash
cd fastapi
.venv/bin/python -m uvicorn main:app --reload
```

## How It Works

### Architecture

```
User Query
    ↓
Generate Embedding (OpenAI)
    ↓
Fetch Initial Results (Milvus) - e.g., 15 results
    ↓
Rerank Results (Cohere) - Select top 5
    ↓
Return to User
```

### Default Behavior

- **Reranking is ENABLED by default** (`enable_reranking=true`)
- **Initial fetch**: `limit * 3` (e.g., if `limit=5`, fetches 15 results)
- **Final results**: Returns top `limit` after reranking

### Example

```python
# Request: limit=5, enable_reranking=true
# Step 1: Fetch 15 results from Milvus (5 * 3)
# Step 2: Rerank all 15 using Cohere
# Step 3: Return top 5 reranked results
```

## API Usage

### 1. Search Endpoint (Direct)

```bash
# With reranking (default)
curl -X POST "http://localhost:8000/api/v1/datasets/{dataset_id}/embeddings/search" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "machine learning techniques",
    "limit": 5,
    "enable_reranking": true
  }'
```

**Response includes**:
```json
{
  "results": [
    {
      "id": "...",
      "text": "...",
      "distance": 0.234,
      "rerank_score": 0.987,  // ← Cohere relevance score
      "meta_data": {...}
    }
  ],
  "reranking_used": true,
  "initial_results_count": 15  // Fetched 15, returned 5
}
```

### 2. MCP Endpoint (Chat Interface)

```bash
# Chat queries use reranking by default
curl "http://localhost:8000/api/v1/mcp/query/{dataset_id}/search?query=your+question&limit=5&enable_reranking=true"
```

**Query parameters**:
- `enable_reranking` (bool, default: `true`) - Enable/disable reranking
- `rerank_top_k` (int, optional) - Number of initial results to fetch (default: `limit * 3`)

## Configuration Options

### Per-Request Control

```python
# Disable reranking for a specific request
{
  "query": "...",
  "limit": 5,
  "enable_reranking": false  # Use only semantic search
}
```

### Custom Initial Fetch Size

```python
# Fetch 50 results, rerank to top 10
{
  "query": "...",
  "limit": 10,
  "enable_reranking": true,
  "rerank_top_k": 50  # Fetch more for better reranking pool
}
```

### Global Settings (config.py)

```python
# Change reranking model
COHERE_RERANK_MODEL = "rerank-multilingual-v3.0"  # For non-English
```

## Performance Impact

### Latency

- **Semantic search only**: ~200-300ms
- **With reranking**: ~400-600ms (+200-300ms)

### Quality Improvement

Based on typical RAG benchmarks:
- **10-30% improvement** in top-5 precision
- **Especially effective** for:
  - Nuanced queries (not just keyword matching)
  - Technical/domain-specific content
  - Queries with multiple interpretations

### Cost

- **OpenAI Embedding**: ~$0.00002 per query
- **Cohere Rerank**: ~$0.001 per query (50x more but still cheap)
- **Total per 1000 queries**: ~$1.02

## Monitoring

### Check if Reranking is Working

Look for logs:
```
[RERANK] Fetching 15 results for reranking (final: 5)
[RERANK] Reranking 15 documents with Cohere rerank-v3.5
[RERANK] Successfully reranked results. Top score: 0.987
[RERANK] Reranked 15 → 5 results
🎯 [MCP] Reranking applied: 15 → 5
```

### Check Response Metadata

```json
{
  "reranking_used": true,  // ← Confirms reranking was applied
  "initial_results_count": 15,
  "results": [
    {
      "rerank_score": 0.987  // ← Present only if reranked
    }
  ]
}
```

## Troubleshooting

### Reranking Not Applied

**Check 1**: API key set?
```bash
cd fastapi
grep COHERE_API_KEY .env
```

**Check 2**: Look for warning logs
```
[RERANK] Warning: COHERE_API_KEY not set, skipping reranking
```

**Check 3**: Feature disabled in request?
```python
# Make sure enable_reranking is true
{"enable_reranking": true}
```

### Fallback Behavior

If reranking fails (API error, timeout, etc.), the system **automatically falls back** to original semantic search results:

```
[RERANK] Warning: Reranking failed, using original results: ConnectionError
```

## Advanced Usage

### A/B Testing

Compare with and without reranking:

```python
# Query A: With reranking
response_a = search(query, enable_reranking=True)

# Query B: Without reranking (baseline)
response_b = search(query, enable_reranking=False)

# Compare relevance scores and user feedback
```

### Optimal Parameters

For most use cases:
- **limit**: 5-10 (more than 10 rarely needed)
- **rerank_top_k**: `limit * 3` to `limit * 5`
  - Too low: Misses potentially relevant results
  - Too high: Wastes API costs with no benefit

Example configurations:

| Use Case | limit | rerank_top_k | Rationale |
|----------|-------|--------------|-----------|
| Chat (conversational) | 5 | 15 (3x) | Fast, focused |
| Research | 10 | 50 (5x) | Comprehensive |
| Quick lookup | 3 | 10 (3x) | Minimal latency |

## Alternatives (Not Implemented)

If you want to avoid external API dependency:

### Option 1: Self-Hosted BGE Reranker

```python
# Would require:
# - Modal/GPU instance for inference
# - BAAI/bge-reranker-v2-m3 model
# Pros: No API costs, full control
# Cons: Infrastructure complexity, slower
```

### Option 2: Milvus Built-in Reranking

```python
# PyMilvus supports reranking but less accurate
milvus.search(..., rerank=RRFRanker())
# Pros: No extra API calls
# Cons: Lower quality than Cohere
```

## Metrics to Track

1. **Reranking usage rate**: % of queries with reranking enabled
2. **Score distribution**: Average rerank_score values
3. **Latency impact**: P50/P99 with vs without reranking
4. **User satisfaction**: Click-through rate on top results
5. **API costs**: Cohere usage per day/month

## Next Steps

To further improve RAG quality:

1. ✅ **Reranking** (implemented)
2. 🔄 **Hybrid search** (BM25 + semantic)
3. 🔄 **Query expansion** (generate variations)
4. 🔄 **Result caching** (Redis for common queries)
5. 🔄 **Relevance threshold** (filter low-quality results)

---

## Quick Reference

**Enable reranking**: Already on by default! ✅

**Disable reranking**: Set `enable_reranking=false` in request

**Check status**: Look for `reranking_used: true` in response

**Cost**: ~$1 per 1000 searches

**Improvement**: 10-30% better relevance
