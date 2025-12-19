# n8n Integration Testing Plan

This document outlines the comprehensive testing strategy for FLTR's n8n integration before enabling paid user access.

## Test Environment Setup

### Prerequisites
```bash
# 1. Local n8n instance (recommended for testing)
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n

# Access at http://localhost:5678

# 2. Test API key from production
# Get from: https://app.fltr.com/settings/api-keys
export FLTR_API_KEY="fltr_sk_test_..."
export FLTR_API_URL="https://api.fltr.com/v1"

# 3. Test dataset with documents
# Create at: https://app.fltr.com/datasets/new
export TEST_DATASET_ID="ds_..."
```

---

## Test Categories

### 1. API Endpoint Testing

#### 1.1 Query Endpoint (`/mcp/query`)
**Test Case 1.1.1: Basic Query**
```bash
# Expected: 200 OK with results
curl -X GET "${FLTR_API_URL}/mcp/query?datasetId=${TEST_DATASET_ID}&query=test&limit=5" \
  -H "Authorization: Bearer ${FLTR_API_KEY}" \
  -H "Content-Type: application/json"

# Verify:
# ✅ Response status: 200
# ✅ Response contains: dataset_id, dataset_name, contexts array
# ✅ Each context has: content, relevance_score, metadata
# ✅ Response time < 2 seconds
```

**Test Case 1.1.2: Query with Reranking**
```bash
# Expected: 200 OK with reranked results
curl -X GET "${FLTR_API_URL}/mcp/query?datasetId=${TEST_DATASET_ID}&query=test&limit=5&enable_reranking=true&rerank_top_k=15" \
  -H "Authorization: Bearer ${FLTR_API_KEY}"

# Verify:
# ✅ reranking_used: true in response
# ✅ Results have rerank_score instead of distance
# ✅ Results are better quality than without reranking
```

**Test Case 1.1.3: Search Mode Variations**
```bash
# Test vector search
curl -X GET "${FLTR_API_URL}/mcp/query?datasetId=${TEST_DATASET_ID}&query=test&search_mode=vector" \
  -H "Authorization: Bearer ${FLTR_API_KEY}"

# Test keyword search
curl -X GET "${FLTR_API_URL}/mcp/query?datasetId=${TEST_DATASET_ID}&query=test&search_mode=keyword" \
  -H "Authorization: Bearer ${FLTR_API_KEY}"

# Test hybrid search
curl -X GET "${FLTR_API_URL}/mcp/query?datasetId=${TEST_DATASET_ID}&query=test&search_mode=hybrid" \
  -H "Authorization: Bearer ${FLTR_API_KEY}"

# Verify:
# ✅ All three modes return results
# ✅ Hybrid mode charges 2 credits
# ✅ Vector/keyword modes charge 1 credit
```

**Test Case 1.1.4: Invalid Parameters**
```bash
# Test invalid limit (too high)
curl -X GET "${FLTR_API_URL}/mcp/query?datasetId=${TEST_DATASET_ID}&query=test&limit=100" \
  -H "Authorization: Bearer ${FLTR_API_KEY}"
# Expected: 422 validation error

# Test invalid search_mode
curl -X GET "${FLTR_API_URL}/mcp/query?datasetId=${TEST_DATASET_ID}&query=test&search_mode=invalid" \
  -H "Authorization: Bearer ${FLTR_API_KEY}"
# Expected: Defaults to hybrid, returns 200

# Test missing query parameter
curl -X GET "${FLTR_API_URL}/mcp/query?datasetId=${TEST_DATASET_ID}&limit=5" \
  -H "Authorization: Bearer ${FLTR_API_KEY}"
# Expected: 422 validation error
```

#### 1.2 Batch Query Endpoint (`/mcp/batch-query`)

**Test Case 1.2.1: Single Batch Query**
```bash
curl -X POST "${FLTR_API_URL}/mcp/batch-query" \
  -H "Authorization: Bearer ${FLTR_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "queries": [
      {
        "dataset_id": "'"${TEST_DATASET_ID}"'",
        "query": "test query 1",
        "limit": 3
      }
    ]
  }'

# Verify:
# ✅ Status: 200
# ✅ batch_results array with 1 result
# ✅ successful: 1, failed: 0
# ✅ Credits charged: 1
```

**Test Case 1.2.2: Multiple Batch Queries**
```bash
curl -X POST "${FLTR_API_URL}/mcp/batch-query" \
  -H "Authorization: Bearer ${FLTR_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "queries": [
      {"dataset_id": "'"${TEST_DATASET_ID}"'", "query": "query 1", "limit": 3},
      {"dataset_id": "'"${TEST_DATASET_ID}"'", "query": "query 2", "limit": 3},
      {"dataset_id": "'"${TEST_DATASET_ID}"'", "query": "query 3", "limit": 3}
    ]
  }'

# Verify:
# ✅ Status: 200
# ✅ batch_results array with 3 results
# ✅ successful: 3, failed: 0
# ✅ Credits charged: 3 (1 per query)
```

**Test Case 1.2.3: Batch with Invalid Dataset**
```bash
curl -X POST "${FLTR_API_URL}/mcp/batch-query" \
  -H "Authorization: Bearer ${FLTR_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "queries": [
      {"dataset_id": "00000000-0000-0000-0000-000000000000", "query": "test", "limit": 3}
    ]
  }'

# Verify:
# ✅ Status: 200 (doesn't fail entire batch)
# ✅ batch_results[0] has error field
# ✅ error: "Dataset not found" or similar
# ✅ No credits charged for failed query
```

#### 1.3 List Datasets Endpoint (`/mcp/datasets`)

**Test Case 1.3.1: Authenticated List**
```bash
curl -X GET "${FLTR_API_URL}/mcp/datasets" \
  -H "Authorization: Bearer ${FLTR_API_KEY}"

# Verify:
# ✅ Status: 200
# ✅ datasets array present
# ✅ Includes public datasets + owned private datasets
# ✅ Each dataset has: id, name, slug, description, category, document_count
```

**Test Case 1.3.2: Unauthenticated List**
```bash
curl -X GET "${FLTR_API_URL}/mcp/datasets"

# Verify:
# ✅ Status: 200
# ✅ datasets array present
# ✅ Only includes public datasets
```

**Test Case 1.3.3: Filter by Category**
```bash
curl -X GET "${FLTR_API_URL}/mcp/datasets?category=technology" \
  -H "Authorization: Bearer ${FLTR_API_KEY}"

# Verify:
# ✅ Status: 200
# ✅ All returned datasets have category: "technology"
```

#### 1.4 Manifest Endpoint (`/mcp/manifest`)

**Test Case 1.4.1: Get Dataset Manifest**
```bash
curl -X GET "${FLTR_API_URL}/mcp/manifest/${TEST_DATASET_ID}" \
  -H "Authorization: Bearer ${FLTR_API_KEY}"

# Verify:
# ✅ Status: 200
# ✅ mcpVersion: "1.0"
# ✅ resources array with search endpoint
# ✅ metadata with dataset info
```

---

### 2. Authentication Testing

#### 2.1 API Key Authentication

**Test Case 2.1.1: Valid API Key**
```bash
curl -X GET "${FLTR_API_URL}/mcp/datasets" \
  -H "Authorization: Bearer ${FLTR_API_KEY}"

# Verify:
# ✅ Status: 200
# ✅ Request succeeds
```

**Test Case 2.1.2: Invalid API Key**
```bash
curl -X GET "${FLTR_API_URL}/mcp/datasets" \
  -H "Authorization: Bearer fltr_sk_invalid_key"

# Verify:
# ✅ Status: 401
# ✅ Error: "Invalid API key" or similar
```

**Test Case 2.1.3: Missing API Key**
```bash
curl -X GET "${FLTR_API_URL}/mcp/datasets"

# Verify:
# ✅ Status: 200 (public datasets only)
# ✅ Limited to public datasets
```

**Test Case 2.1.4: Malformed Authorization Header**
```bash
curl -X GET "${FLTR_API_URL}/mcp/datasets" \
  -H "Authorization: InvalidFormat"

# Verify:
# ✅ Status: 401
# ✅ Error about malformed header
```

#### 2.2 OAuth/Session Authentication

**Test Case 2.2.1: Valid Session Token**
```bash
# Get session token from browser after login
export SESSION_TOKEN="..."

curl -X GET "${FLTR_API_URL}/mcp/datasets" \
  -H "Authorization: Bearer ${SESSION_TOKEN}"

# Verify:
# ✅ Status: 200
# ✅ Access to user's private datasets
```

#### 2.3 Dataset Access Control

**Test Case 2.3.1: Access Own Private Dataset**
```bash
curl -X GET "${FLTR_API_URL}/mcp/query?datasetId=${PRIVATE_DATASET_ID}&query=test" \
  -H "Authorization: Bearer ${FLTR_API_KEY}"

# Verify:
# ✅ Status: 200
# ✅ Query succeeds
```

**Test Case 2.3.2: Access Others' Private Dataset**
```bash
# Use different user's API key
curl -X GET "${FLTR_API_URL}/mcp/query?datasetId=${PRIVATE_DATASET_ID}&query=test" \
  -H "Authorization: Bearer ${OTHER_USER_API_KEY}"

# Verify:
# ✅ Status: 403
# ✅ Error: "Access denied" or "Dataset not found"
```

**Test Case 2.3.3: Access Public Dataset (Unauthenticated)**
```bash
curl -X GET "${FLTR_API_URL}/mcp/query?datasetId=${PUBLIC_DATASET_ID}&query=test&limit=5"

# Verify:
# ✅ Status: 200 (if anonymous access enabled)
# OR
# ✅ Status: 401 (if auth required)
```

---

### 3. Rate Limiting Testing

#### 3.1 Anonymous Rate Limits (50/hour)

**Test Case 3.1.1: Anonymous Within Limit**
```bash
# Send 49 requests (under limit)
for i in {1..49}; do
  curl -X GET "${FLTR_API_URL}/mcp/datasets"
  sleep 0.1
done

# Verify:
# ✅ All requests return 200
# ✅ X-RateLimit-Remaining decreases
```

**Test Case 3.1.2: Anonymous Exceeds Limit**
```bash
# Send 51 requests (over limit)
for i in {1..51}; do
  echo "Request $i:"
  curl -i -X GET "${FLTR_API_URL}/mcp/datasets" 2>&1 | grep -E "HTTP/|X-RateLimit"
  sleep 0.1
done

# Verify:
# ✅ First 50 requests: 200 OK
# ✅ Request 51: 429 Too Many Requests
# ✅ Response headers:
#    - X-RateLimit-Limit: 50
#    - X-RateLimit-Remaining: 0
#    - X-RateLimit-Reset: <timestamp>
#    - Retry-After: <seconds>
```

#### 3.2 API Key Rate Limits (1000/hour)

**Test Case 3.2.1: API Key Within Limit**
```bash
# Send 100 requests with API key
for i in {1..100}; do
  curl -X GET "${FLTR_API_URL}/mcp/datasets" \
    -H "Authorization: Bearer ${FLTR_API_KEY}"
  sleep 0.1
done

# Verify:
# ✅ All requests return 200
# ✅ X-RateLimit-Limit: 1000
```

**Test Case 3.2.2: API Key Exceeds Limit**
```bash
# Send 1001 requests (would need script)
# This is time-consuming, so test with smaller endpoint limit instead
```

#### 3.3 OAuth Rate Limits (15000/hour)

**Test Case 3.3.1: Session Token High Volume**
```bash
# Send 200 requests with session token
for i in {1..200}; do
  curl -X GET "${FLTR_API_URL}/mcp/datasets" \
    -H "Authorization: Bearer ${SESSION_TOKEN}"
  sleep 0.1
done

# Verify:
# ✅ All requests return 200
# ✅ X-RateLimit-Limit: 15000
```

#### 3.4 Endpoint-Specific Rate Limits

**Test Case 3.4.1: Batch Query Limit (100/hour)**
```bash
# Send 101 batch queries
for i in {1..101}; do
  echo "Batch query $i:"
  curl -i -X POST "${FLTR_API_URL}/mcp/batch-query" \
    -H "Authorization: Bearer ${FLTR_API_KEY}" \
    -H "Content-Type: application/json" \
    -d '{"queries":[{"dataset_id":"'"${TEST_DATASET_ID}"'","query":"test","limit":3}]}' \
    2>&1 | grep -E "HTTP/|X-RateLimit"
  sleep 1
done

# Verify:
# ✅ First 100: 200 OK
# ✅ Request 101: 429 Too Many Requests
```

#### 3.5 Rate Limit Headers

**Test Case 3.5.1: Verify Response Headers**
```bash
curl -i -X GET "${FLTR_API_URL}/mcp/datasets" \
  -H "Authorization: Bearer ${FLTR_API_KEY}"

# Verify headers present:
# ✅ X-RateLimit-Limit: 1000
# ✅ X-RateLimit-Remaining: 999 (or less)
# ✅ X-RateLimit-Reset: <unix timestamp>
```

---

### 4. Credit/Billing Testing

#### 4.1 Credit Deduction

**Test Case 4.1.1: Query Deducts 1 Credit (Vector/Keyword)**
```bash
# Check user credits before (API keys draw from user balance, not separate pools)
CREDITS_BEFORE=$(curl -X GET "${FLTR_API_URL}/credits" \
  -H "Authorization: Bearer ${FLTR_API_KEY}" | jq '.balance')

# Run query
curl -X GET "${FLTR_API_URL}/mcp/query?datasetId=${TEST_DATASET_ID}&query=test&search_mode=vector" \
  -H "Authorization: Bearer ${FLTR_API_KEY}"

# Check user credits after
CREDITS_AFTER=$(curl -X GET "${FLTR_API_URL}/credits" \
  -H "Authorization: Bearer ${FLTR_API_KEY}" | jq '.balance')

# Verify:
# ✅ CREDITS_AFTER = CREDITS_BEFORE - 1 (deducted from user balance)
```

**Test Case 4.1.2: Hybrid Query Deducts 2 Credits**
```bash
# Check credits before
CREDITS_BEFORE=$(curl -X GET "${FLTR_API_URL}/credits" \
  -H "Authorization: Bearer ${FLTR_API_KEY}" | jq '.balance')

# Run hybrid query
curl -X GET "${FLTR_API_URL}/mcp/query?datasetId=${TEST_DATASET_ID}&query=test&search_mode=hybrid" \
  -H "Authorization: Bearer ${FLTR_API_KEY}"

# Check credits after
CREDITS_AFTER=$(curl -X GET "${FLTR_API_URL}/credits" \
  -H "Authorization: Bearer ${FLTR_API_KEY}" | jq '.balance')

# Verify:
# ✅ CREDITS_AFTER = CREDITS_BEFORE - 2
```

**Test Case 4.1.3: Batch Query Deducts N Credits**
```bash
# Check credits before
CREDITS_BEFORE=$(curl -X GET "${FLTR_API_URL}/credits" \
  -H "Authorization: Bearer ${FLTR_API_KEY}" | jq '.balance')

# Run batch with 5 queries
curl -X POST "${FLTR_API_URL}/mcp/batch-query" \
  -H "Authorization: Bearer ${FLTR_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "queries": [
      {"dataset_id":"'"${TEST_DATASET_ID}"'","query":"q1","limit":3},
      {"dataset_id":"'"${TEST_DATASET_ID}"'","query":"q2","limit":3},
      {"dataset_id":"'"${TEST_DATASET_ID}"'","query":"q3","limit":3},
      {"dataset_id":"'"${TEST_DATASET_ID}"'","query":"q4","limit":3},
      {"dataset_id":"'"${TEST_DATASET_ID}"'","query":"q5","limit":3}
    ]
  }'

# Check credits after
CREDITS_AFTER=$(curl -X GET "${FLTR_API_URL}/credits" \
  -H "Authorization: Bearer ${FLTR_API_KEY}" | jq '.balance')

# Verify:
# ✅ CREDITS_AFTER = CREDITS_BEFORE - 5
```

#### 4.2 Insufficient Credits

**Test Case 4.2.1: Query with 0 Credits**
```bash
# Drain credits to 0 (admin can set balance)
# OR use test API key with 0 balance

curl -X GET "${FLTR_API_URL}/mcp/query?datasetId=${TEST_DATASET_ID}&query=test" \
  -H "Authorization: Bearer ${ZERO_CREDIT_API_KEY}"

# Verify:
# ✅ Status: 402 Payment Required
# ✅ Error: "Insufficient credits"
```

#### 4.3 Credit Refunds (System Errors)

**Test Case 4.3.1: Refund on 500 Error**
```bash
# Trigger a 500 error (e.g., by using invalid dataset configuration)
# This is harder to test - would need to manually create error condition

# Verify:
# ✅ Credits are refunded when system error occurs
# ✅ Credit transaction has refund record
```

#### 4.4 Admin Credit Bypass

**Test Case 4.4.1: Admin User Bypasses Credits**
```bash
# Use admin API key
curl -X GET "${FLTR_API_URL}/mcp/query?datasetId=${TEST_DATASET_ID}&query=test" \
  -H "Authorization: Bearer ${ADMIN_API_KEY}"

# Verify:
# ✅ Status: 200
# ✅ No credits deducted from admin account
```

---

### 5. n8n Workflow Testing

#### 5.1 Basic Workflow

**Test Case 5.1.1: HTTP Request Node Query**

**n8n Workflow**:
1. Create new workflow
2. Add "Manual Trigger" node
3. Add "HTTP Request" node:
   - Method: GET
   - URL: `https://api.fltr.com/v1/mcp/query?datasetId={{$json.dataset_id}}&query={{$json.query}}&limit=10`
     - limit: 5
   - Authentication: Generic Credential Type → Header Auth
     - Name: `Authorization`
     - Value: `Bearer YOUR_API_KEY`
4. Execute workflow

**Test Data**:
```json
{
  "dataset_id": "ds_...",
  "query": "test search"
}
```

**Verify**:
- ✅ Workflow executes successfully
- ✅ HTTP Request returns 200
- ✅ Output has contexts array
- ✅ Each context has content, relevance_score, metadata

#### 5.2 Email to Search Workflow

**Test Case 5.2.1: Email Trigger → FLTR Query → Reply**

**n8n Workflow**:
1. Email Trigger (IMAP) - Watch inbox
2. HTTP Request - FLTR Query:
   ```
   URL: https://api.fltr.com/v1/mcp/query?datasetId=DATASET_ID&query={{$json.subject}}+{{$json.text}}&limit=5
   ```
3. Code Node - Format results
4. Send Email - Reply with results

**Test**:
- Send test email to monitored inbox
- Wait for workflow to trigger

**Verify**:
- ✅ Workflow triggers on new email
- ✅ FLTR query executes with email content
- ✅ Reply email sent with search results
- ✅ Credits deducted correctly

#### 5.3 Slack Bot Workflow

**Test Case 5.3.1: Slack Question → FLTR → Reply**

**n8n Workflow**:
1. Slack Trigger - New message in channel
2. IF Node - Check if message contains "?"
3. HTTP Request - FLTR Query
4. Slack - Post reply

**Test**:
- Post question in monitored Slack channel

**Verify**:
- ✅ Workflow triggers on question
- ✅ FLTR returns relevant answer
- ✅ Bot replies in thread

#### 5.4 Batch Processing Workflow

**Test Case 5.4.1: Multiple Queries in Loop**

**n8n Workflow**:
1. Manual Trigger with array of queries
2. Split in Batches (size: 1)
3. HTTP Request - FLTR Query
4. Aggregate results

**Test Data**:
```json
{
  "queries": [
    "What is RAG?",
    "How does vector search work?",
    "What is semantic search?"
  ]
}
```

**Verify**:
- ✅ Loop processes all queries
- ✅ Each query gets results
- ✅ Credits charged: 3 (1 per query)

#### 5.5 Error Handling Workflow

**Test Case 5.5.1: Rate Limit Error → Wait → Retry**

**n8n Workflow**:
1. HTTP Request - FLTR Query
2. Error Trigger - On 429 status
3. Wait Node - `{{$json.headers['Retry-After']}}` seconds
4. HTTP Request - Retry query

**Test**:
- Trigger rate limit by sending many requests
- Verify retry logic works

**Verify**:
- ✅ Error caught on 429
- ✅ Wait node delays correctly
- ✅ Retry succeeds after waiting

#### 5.6 Webhook Integration

**Test Case 5.6.1: External Service → Webhook → FLTR**

**n8n Workflow**:
1. Webhook Trigger (POST)
2. HTTP Request - FLTR Query with webhook data
3. Webhook Response - Return results

**Test**:
```bash
curl -X POST https://your-n8n.com/webhook/fltr-search \
  -H "Content-Type: application/json" \
  -d '{"query": "test search"}'
```

**Verify**:
- ✅ Webhook receives request
- ✅ FLTR query executes
- ✅ Response returned to caller

---

### 6. Error Handling Testing

#### 6.1 HTTP Error Codes

**Test Case 6.1.1: 400 Bad Request**
```bash
curl -X GET "${FLTR_API_URL}/mcp/query?datasetId=${TEST_DATASET_ID}&limit=999" \
  -H "Authorization: Bearer ${FLTR_API_KEY}"

# Verify:
# ✅ Status: 422 (validation error)
# ✅ Error message describes issue
```

**Test Case 6.1.2: 401 Unauthorized**
```bash
curl -X GET "${FLTR_API_URL}/mcp/query?datasetId=${TEST_DATASET_ID}&query=test" \
  -H "Authorization: Bearer invalid_key"

# Verify:
# ✅ Status: 401
# ✅ Error: "Invalid API key"
```

**Test Case 6.1.3: 404 Not Found**
```bash
curl -X GET "${FLTR_API_URL}/mcp/query?datasetId=00000000-0000-0000-0000-000000000000&query=test" \
  -H "Authorization: Bearer ${FLTR_API_KEY}"

# Verify:
# ✅ Status: 404
# ✅ Error: "Dataset not found"
```

**Test Case 6.1.4: 429 Rate Limit**
```bash
# Hit rate limit (covered in section 3)

# Verify:
# ✅ Status: 429
# ✅ Retry-After header present
# ✅ X-RateLimit-Reset header present
```

**Test Case 6.1.5: 500 Server Error**
```bash
# Hard to trigger intentionally
# Monitor production logs for any 500s

# Verify:
# ✅ Credits automatically refunded
# ✅ Error logged for debugging
```

---

### 7. Performance Testing

#### 7.1 Response Time

**Test Case 7.1.1: Query Response Time**
```bash
# Measure response time
time curl -X GET "${FLTR_API_URL}/mcp/query?datasetId=${TEST_DATASET_ID}&query=test&limit=5" \
  -H "Authorization: Bearer ${FLTR_API_KEY}"

# Verify:
# ✅ Response time < 2 seconds (without reranking)
# ✅ Response time < 4 seconds (with reranking)
```

**Test Case 7.1.2: Batch Query Performance**
```bash
# Measure batch query time
time curl -X POST "${FLTR_API_URL}/mcp/batch-query" \
  -H "Authorization: Bearer ${FLTR_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "queries": [
      {"dataset_id":"'"${TEST_DATASET_ID}"'","query":"q1","limit":3},
      {"dataset_id":"'"${TEST_DATASET_ID}"'","query":"q2","limit":3},
      {"dataset_id":"'"${TEST_DATASET_ID}"'","query":"q3","limit":3}
    ]
  }'

# Verify:
# ✅ Batch of 3 queries < 5 seconds total
```

#### 7.2 Concurrent Requests

**Test Case 7.2.1: Parallel Queries**
```bash
# Send 10 parallel requests
for i in {1..10}; do
  curl -X GET "${FLTR_API_URL}/mcp/query?datasetId=${TEST_DATASET_ID}&query=test$i" \
    -H "Authorization: Bearer ${FLTR_API_KEY}" &
done
wait

# Verify:
# ✅ All requests complete successfully
# ✅ No timeouts or connection errors
```

---

### 8. Integration Testing Checklist

Before going live, verify:

#### 8.1 API Functionality
- [ ] All endpoints return correct status codes
- [ ] Query results are relevant and accurate
- [ ] Reranking improves result quality
- [ ] Search modes (vector/keyword/hybrid) work correctly
- [ ] Batch queries process all requests
- [ ] Error messages are helpful and accurate

#### 8.2 Authentication
- [ ] API keys authenticate correctly
- [ ] Invalid API keys are rejected
- [ ] Session tokens work for OAuth users
- [ ] Private datasets enforce access control
- [ ] Public datasets are accessible without auth

#### 8.3 Rate Limiting
- [ ] Anonymous limit: 50 req/hour enforced
- [ ] API key limit: 1000 req/hour enforced
- [ ] OAuth limit: 15000 req/hour enforced
- [ ] Rate limit headers are accurate
- [ ] 429 errors include Retry-After
- [ ] Endpoint-specific limits work

#### 8.4 Billing
- [ ] Credits deducted before query execution
- [ ] Hybrid queries charge 2 credits
- [ ] Vector/keyword queries charge 1 credit
- [ ] Batch queries charge N credits
- [ ] Insufficient credits block requests (402)
- [ ] System errors refund credits
- [ ] Admin users bypass credit checks

#### 8.5 n8n Workflows
- [ ] HTTP Request node works with FLTR API
- [ ] API key authentication in n8n works
- [ ] Query results parse correctly in n8n
- [ ] Code nodes can process FLTR responses
- [ ] Error handling works in workflows
- [ ] Webhooks integrate successfully

#### 8.6 Documentation
- [ ] All examples in docs work correctly
- [ ] Error codes are documented
- [ ] Rate limit handling is documented
- [ ] Credit costs are clear
- [ ] Authentication methods are explained

---

## Test Execution Order

### Phase 1: Smoke Tests (30 minutes)
1. Basic query with API key
2. List datasets
3. Check rate limit headers
4. Verify credit deduction

### Phase 2: Comprehensive API Testing (2 hours)
1. All endpoint variations
2. Parameter validation
3. Error handling
4. Authentication scenarios

### Phase 3: Rate Limiting (1 hour)
1. Anonymous limits
2. API key limits
3. Endpoint-specific limits
4. Header validation

### Phase 4: Billing Testing (1 hour)
1. Credit deduction accuracy
2. Different search modes
3. Batch queries
4. Insufficient credits
5. Refund scenarios

### Phase 5: n8n Integration (2 hours)
1. Basic workflow
2. Email automation
3. Slack bot
4. Batch processing
5. Error handling

### Phase 6: Performance & Load (1 hour)
1. Response time benchmarks
2. Concurrent requests
3. Large result sets

---

## Test Results Template

```markdown
# Test Execution Report
Date: YYYY-MM-DD
Tester: [Name]
Environment: [Production/Staging]

## Summary
- Total Tests: X
- Passed: Y
- Failed: Z
- Blocked: N

## Critical Issues
- [ ] None found
- [ ] Issue 1: [Description]
- [ ] Issue 2: [Description]

## Test Results by Category

### API Endpoints
- Query Endpoint: ✅ PASS / ❌ FAIL
- Batch Query: ✅ PASS / ❌ FAIL
- List Datasets: ✅ PASS / ❌ FAIL
- Manifest: ✅ PASS / ❌ FAIL

### Authentication
- API Key Auth: ✅ PASS / ❌ FAIL
- OAuth Auth: ✅ PASS / ❌ FAIL
- Access Control: ✅ PASS / ❌ FAIL

### Rate Limiting
- Anonymous Limits: ✅ PASS / ❌ FAIL
- API Key Limits: ✅ PASS / ❌ FAIL
- OAuth Limits: ✅ PASS / ❌ FAIL

### Billing
- Credit Deduction: ✅ PASS / ❌ FAIL
- Insufficient Credits: ✅ PASS / ❌ FAIL
- Refunds: ✅ PASS / ❌ FAIL

### n8n Integration
- Basic Workflow: ✅ PASS / ❌ FAIL
- Email Automation: ✅ PASS / ❌ FAIL
- Error Handling: ✅ PASS / ❌ FAIL

## Recommendation
✅ APPROVED FOR PRODUCTION
❌ NOT READY - Critical issues found
⚠️ CONDITIONAL - Minor issues to address

## Notes
[Additional observations, concerns, or recommendations]
```

---

## Quick Start Testing Script

Save this as `test-n8n-integration.sh`:

```bash
#!/bin/bash

# Configuration
export FLTR_API_URL="https://api.fltr.com/v1"
export FLTR_API_KEY="fltr_sk_..."
export TEST_DATASET_ID="ds_..."

echo "🧪 FLTR n8n Integration Test Suite"
echo "===================================="

# Test 1: Basic Query
echo -e "\n1️⃣  Testing basic query..."
response=$(curl -s -w "%{http_code}" -o /tmp/response.json \
  -X GET "${FLTR_API_URL}/mcp/query?datasetId=${TEST_DATASET_ID}&query=test&limit=5" \
  -H "Authorization: Bearer ${FLTR_API_KEY}")

if [ "$response" = "200" ]; then
  echo "✅ Basic query: PASS"
else
  echo "❌ Basic query: FAIL (HTTP $response)"
fi

# Test 2: List Datasets
echo -e "\n2️⃣  Testing list datasets..."
response=$(curl -s -w "%{http_code}" -o /tmp/response.json \
  -X GET "${FLTR_API_URL}/mcp/datasets" \
  -H "Authorization: Bearer ${FLTR_API_KEY}")

if [ "$response" = "200" ]; then
  echo "✅ List datasets: PASS"
else
  echo "❌ List datasets: FAIL (HTTP $response)"
fi

# Test 3: Invalid API Key
echo -e "\n3️⃣  Testing invalid API key..."
response=$(curl -s -w "%{http_code}" -o /dev/null \
  -X GET "${FLTR_API_URL}/mcp/datasets" \
  -H "Authorization: Bearer invalid_key")

if [ "$response" = "401" ]; then
  echo "✅ Invalid API key rejected: PASS"
else
  echo "❌ Invalid API key: FAIL (Expected 401, got $response)"
fi

# Test 4: Rate Limit Headers
echo -e "\n4️⃣  Testing rate limit headers..."
headers=$(curl -s -I \
  -X GET "${FLTR_API_URL}/mcp/datasets" \
  -H "Authorization: Bearer ${FLTR_API_KEY}" | grep -i "X-RateLimit")

if [ -n "$headers" ]; then
  echo "✅ Rate limit headers present: PASS"
  echo "$headers"
else
  echo "❌ Rate limit headers missing: FAIL"
fi

# Test 5: Batch Query
echo -e "\n5️⃣  Testing batch query..."
response=$(curl -s -w "%{http_code}" -o /tmp/response.json \
  -X POST "${FLTR_API_URL}/mcp/batch-query" \
  -H "Authorization: Bearer ${FLTR_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"queries":[{"dataset_id":"'"${TEST_DATASET_ID}"'","query":"test","limit":3}]}')

if [ "$response" = "200" ]; then
  echo "✅ Batch query: PASS"
else
  echo "❌ Batch query: FAIL (HTTP $response)"
fi

echo -e "\n===================================="
echo "Test suite complete!"
echo "Review results above ☝️"
```

Make executable and run:
```bash
chmod +x test-n8n-integration.sh
./test-n8n-integration.sh
```

---

## Conclusion

This comprehensive test plan covers:
- ✅ All API endpoints
- ✅ Authentication scenarios
- ✅ Rate limiting enforcement
- ✅ Credit/billing accuracy
- ✅ n8n workflow integration
- ✅ Error handling
- ✅ Performance benchmarks

**Estimated total testing time**: ~8 hours for full suite

**Quick validation**: 30 minutes using the smoke test script

After passing all tests, you're ready to enable paid user access to n8n workflows! 🚀
