# Document Processing Cost Analysis

**Generated**: October 31, 2024  
**Based on**: Real production logs from Modal + OpenAI

---

## 🎯 Cost Per Document (Production Data)

### Actual Performance Logs

| File Size | Pages | Chunks | Processing Time | Modal Cost | OpenAI Cost | **Total Cost** |
|-----------|-------|--------|-----------------|------------|-------------|----------------|
| 225 KB | 10 | 60 | 35.21s | $0.0103 | $0.0009 | **$0.0112** |
| 1.5 MB | 50 | 127 | 61.85s | $0.0181 | $0.0019 | **$0.0200** |

### Extrapolated Estimates

| File Size | Est. Pages | Est. Chunks | Est. Time | Modal Cost | OpenAI Cost | **Total Cost** |
|-----------|------------|-------------|-----------|------------|-------------|----------------|
| 500 KB | 20 | 120 | 45s | $0.0131 | $0.0018 | **$0.0149** |
| 2 MB | 80 | 200 | 80s | $0.0234 | $0.0030 | **$0.0264** |
| 5 MB | 200 | 500 | 200s | $0.0584 | $0.0075 | **$0.0659** |
| 10 MB | 400 | 1,000 | 400s | $0.1168 | $0.0150 | **$0.1318** |
| 25 MB | 1,000 | 2,500 | 1,000s | $0.2920 | $0.0375 | **$0.3295** |
| 50 MB | 2,000 | 5,000 | 2,000s | $0.5840 | $0.0750 | **$0.6590** |
| 100 MB | 4,000 | 10,000 | 4,000s | $1.1680 | $0.1500 | **$1.3180** |

---

## 💰 Cost Components Breakdown

### Modal Compute Costs

**Configuration:**
- CPU: 8.0 cores @ $0.000033/core-second
- Memory: 8 GB @ $0.0000035/GiB-second
- **Combined rate**: $0.000292/second

**Formula:**
```
Modal Cost = Processing Time (seconds) × $0.000292
```

**Performance Profile (from logs):**
- Download: ~1-3% of time (R2 → Modal)
- **Parse: ~86-89% of time** (Docling PDF parsing - BOTTLENECK)
- Chunk: <1% of time (simple text splitting)
- Embed: ~4-7% of time (OpenAI API calls)
- Store: ~2-4% of time (Milvus insertion)
- DB Update: ~1-2% of time (PostgreSQL)

### OpenAI Embedding Costs

**Model:** `text-embedding-3-small`
- **Price**: $0.020 per 1M tokens
- **Chunk size**: 1,000 characters ≈ 750 tokens
- **Cost per chunk**: ~$0.000015

**Formula:**
```
OpenAI Cost = Number of Chunks × $0.000015
```

**Chunking ratio observed:**
- 10 pages → 60 chunks (6 chunks/page)
- 50 pages → 127 chunks (2.5 chunks/page)
- Average: ~2-6 chunks per page depending on content density

---

## 📊 Real-World Usage Scenarios

### Scenario 1: Academic Research Papers (Most Common)
- **Typical size**: 500 KB - 2 MB
- **Pages**: 10-50
- **Cost per document**: $0.015 - $0.026
- **1,000 papers/month**: $15 - $26

### Scenario 2: Legal Documents
- **Typical size**: 1-5 MB
- **Pages**: 20-100
- **Cost per document**: $0.020 - $0.066
- **1,000 docs/month**: $20 - $66

### Scenario 3: Healthcare Records
- **Typical size**: 2-10 MB
- **Pages**: 50-200
- **Cost per document**: $0.026 - $0.132
- **1,000 records/month**: $26 - $132

### Scenario 4: Books/Large Documents
- **Typical size**: 10-50 MB
- **Pages**: 200-1,000
- **Cost per document**: $0.132 - $0.659
- **100 books/month**: $13.20 - $65.90

### Scenario 5: Extreme Cases (Scanned Documents)
- **Typical size**: 50-100 MB
- **Pages**: 1,000-2,000
- **Cost per document**: $0.659 - $1.318
- **100 docs/month**: $65.90 - $131.80

---

## 🔧 Cost Optimization Opportunities

### Current Bottleneck: PDF Parsing (86-89% of time)

**Why it's slow:**
- Docling uses RapidOCR for text extraction
- CPU-only processing (no GPU acceleration currently)
- MediaBox repair adds overhead for malformed PDFs

**Optimization Options:**

| Option | Time Savings | Cost Impact | Implementation |
|--------|--------------|-------------|----------------|
| **GPU Acceleration** | 50-70% faster | +30% cost | Add `gpu="T4"` to Modal function |
| **Reduce CPU cores** | 20% slower | -50% cost | Use 4 cores instead of 8 |
| **Skip MediaBox repair** | 5-10% faster | -5% cost | Only repair on parsing failure |
| **Pre-process PDFs** | 40-60% faster | Variable | Validate/normalize before upload |
| **Caching** | 100% for duplicates | Storage cost | Cache parsed documents |

### Recommended Configuration for Cost/Speed Balance

**Option A: Current (Balanced)**
```python
cpu=8.0, memory=8192, gpu=None
```
- **Pro**: Good balance, no GPU costs
- **Con**: Slower for complex PDFs
- **Cost**: $0.000292/second

**Option B: Speed-Optimized**
```python
cpu=4.0, memory=8192, gpu="T4"
```
- **Pro**: 50-70% faster parsing
- **Con**: Higher per-second cost
- **Cost**: ~$0.00040/second (but processes faster)

**Option C: Cost-Optimized**
```python
cpu=4.0, memory=4096, gpu=None
```
- **Pro**: 50% cheaper per second
- **Con**: 20-30% slower
- **Cost**: $0.000146/second

---

## 📈 Monthly Cost Projections by Tier

### Free Tier (100 documents/month)
- **Average doc size**: 1 MB (typical research paper)
- **Cost per doc**: $0.020
- **Monthly total**: **$2.00**
- **With 50% margin**: **$4.00 buffer needed**

### Paid Tier (500 documents/month)
- **Average doc size**: 2 MB (mixed content)
- **Cost per doc**: $0.026
- **Monthly total**: **$13.00**
- **With 50% margin**: **$26.00 buffer needed**

### Pro Tier (1,500 documents/month)
- **Average doc size**: 3 MB (larger documents)
- **Cost per doc**: $0.035
- **Monthly total**: **$52.50**
- **With 50% margin**: **$105.00 buffer needed**

---

## 🎛️ Credit System Recommendations

Based on these costs, here are recommended credit values for document processing:

### Document Upload Credits (by file size)

| File Size Range | Credits | Actual Cost | Markup |
|-----------------|---------|-------------|--------|
| < 500 KB | 1 credit | $0.015 | 1 credit = $0.015 |
| 500 KB - 2 MB | 2 credits | $0.015-$0.026 | Covers range |
| 2 MB - 5 MB | 4 credits | $0.026-$0.066 | Covers range |
| 5 MB - 10 MB | 8 credits | $0.066-$0.132 | Covers range |
| 10 MB - 25 MB | 15 credits | $0.132-$0.329 | Covers range |
| 25 MB - 50 MB | 30 credits | $0.329-$0.659 | Covers range |
| 50 MB - 100 MB | 60 credits | $0.659-$1.318 | Covers range |

**Alternatively: Dynamic Pricing**
```python
# More accurate but complex
credits = max(1, ceil(file_size_mb * 0.6))
```

### Credit Value Model
```
1 credit = $0.015 average processing cost
```

This allows:
- 100 credits = ~66 documents (avg 1.5 MB each)
- 500 credits = ~333 documents
- 1,500 credits = ~1,000 documents

---

## 🚨 Edge Cases & Considerations

### Timeout Risk
- **Current timeout**: 3,600s (1 hour)
- **Risk threshold**: 100+ MB files could timeout
- **Mitigation**: Set max file size limit at 50 MB

### Cold Start Impact
- **Current**: `min_containers=1` keeps one warm
- **Cold start time**: ~10-15 seconds
- **Cost**: ~$0.30/day = $9/month
- **Alternative**: Set to 0 for -$9/month but +10s latency on first doc

### Concurrent Processing
- **Max containers**: 10
- **Peak concurrent cost**: 10 × $0.000292/sec = $0.00292/sec
- **If all 10 running for 1 minute**: $0.175
- **Risk**: Upload spikes could be expensive

### Failed Documents
- **Refund policy needed**: System errors should refund credits
- **User errors** (unsupported format): No refund
- **Partial processing**: Proportional refund

---

## 📋 Summary & Recommendations

### Key Metrics
- **Typical document (1-2 MB)**: $0.020 - $0.026
- **Average across all sizes**: ~$0.030
- **85-90%** of cost is Modal compute time
- **10-15%** of cost is OpenAI embeddings

### Pricing Strategy
1. **Simple fixed rate**: 2 credits per document (any size < 5MB)
2. **Dynamic pricing**: 1 credit per MB (more fair, more complex)
3. **Tiered pricing**: Use the table above (recommended)

### Infrastructure Optimizations
1. ✅ **Keep current config** (8 CPU, no GPU) for now
2. 🔍 **Monitor**: Track average processing times monthly
3. ⚡ **Consider GPU** if users complain about speed (50MB+ files)
4. 💰 **Consider CPU reduction** if cost becomes primary concern
5. 📊 **Implement file size limits**: 50 MB max to avoid timeouts

### Credit System Integration
- Charge credits **before** processing (reserve)
- Refund on system failure
- No refund on user error (invalid file type)
- Show estimated credits in UI before upload
- Track actual costs vs. credits charged for margin analysis

---

**Next Steps:**
1. Implement dynamic credit calculation based on file size
2. Add cost tracking to database (actual Modal + OpenAI costs per document)
3. Monitor margin: credits charged vs. actual costs
4. Adjust credit pricing quarterly based on actual usage patterns








