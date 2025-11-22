# FLTR Cloud Infrastructure Costs

## 📊 Complete Cost Breakdown & Estimates

This document provides a comprehensive breakdown of cloud infrastructure costs for the FLTR platform across all services and usage tiers.

**Last Updated**: October 31, 2024

---

## 🎯 Executive Summary

### Cost Per User Tier (Monthly)

| Usage Tier | Documents/Month | Total Cost | Cost Per Document |
|------------|-----------------|------------|-------------------|
| **Free Tier** | 0-100 | $0-15 | $0.15 |
| **Starter** | 1,000 | $35-50 | $0.035-0.050 |
| **Growth** | 10,000 | $230-280 | $0.023-0.028 |
| **Scale** | 100,000 | $2,100-2,500 | $0.021-0.025 |
| **Enterprise** | 1M+ | $20,000-25,000 | $0.020-0.025 |

### Cost Components Overview

| Service | Purpose | Pricing Model | Est. % of Total |
|---------|---------|---------------|-----------------|
| Modal.com | Document Processing | Per-second compute | 35-45% |
| OpenAI API | Embeddings & Chat | Per token | 25-35% |
| DigitalOcean | FastAPI App Server | Fixed monthly | 15-20% |
| Milvus Cloud | Vector Storage | Per GB + queries | 8-12% |
| Cloudflare R2 | File Storage | Per GB + ops | 3-5% |
| PostgreSQL | Metadata DB | Per instance | 3-5% |
| Vercel | Frontend Hosting | Per request | 2-4% |
| Cloudflare Workers | Proxy & Tasks | Per request | <1% |

---

## 💻 Modal.com - Document Processing

### Configuration

**GPU Processing Function:**
- GPU: NVIDIA T4
- CPU: 4 cores
- Memory: 16GB
- Scales to zero when idle

**Webhook Endpoint:**
- CPU only (lightweight)
- Scales to zero by default (can keep warm for +$190/month)

### Pricing

**GPU Processing:**
- T4 GPU: **$0.70/hour** ($0.000195/second)
- 4 CPU cores: **$0.32/hour** ($0.000089/second)
- 16GB memory: **$0.19/hour** ($0.000053/second)
- **Total: $1.21/hour** or **$0.000337/second**

**Processing Time:**
- Average document: **6 seconds** with GPU (vs 60 seconds CPU-only)
- Cost per document: **$0.002**

**Cold Start Costs:**
- Cold start time: 20-40 seconds
- Occurs after 10 minutes of inactivity
- Cold start cost: ~$0.01 per cold start
- Amortized over batch processing

### Monthly Cost Estimates

| Documents/Month | Processing Time | Modal Cost | Notes |
|-----------------|-----------------|------------|-------|
| 100 | 10 minutes | $0.20 | Mostly cold starts |
| 1,000 | 1.7 hours | $2.00 | Some cold starts |
| 10,000 | 17 hours | $20 | Minimal cold start impact |
| 100,000 | 167 hours | $200 | Continuous processing |
| 1,000,000 | 1,670 hours | $2,000 | High volume discount possible |

### Optimization Options

**Keep Webhook Warm:**
- Cost: +$190/month ($6.24/day)
- Benefit: Instant response, no cold starts
- Recommended: Only for >1,000 docs/day

**Volume Discounts:**
- Modal offers enterprise pricing at scale
- Contact Modal sales for >$5,000/month usage

---

## 🤖 OpenAI API - Embeddings & Chat

### Configuration

**Embeddings:**
- Model: `text-embedding-3-small`
- Dimension: 1536
- Use case: Document chunks for vector search

**Chat Completions:**
- Model: `gpt-4-turbo` or `gpt-3.5-turbo`
- Use case: Query answering, document synthesis

### Pricing

**Embeddings (text-embedding-3-small):**
- **$0.02 per 1M tokens**
- Average chunk: ~200 tokens
- Average document: 5,000 tokens → 25 chunks
- Cost per document: **$0.00025** (embeddings only)

**Chat Completions:**

| Model | Input | Output | Typical Query Cost |
|-------|-------|--------|-------------------|
| GPT-3.5-turbo | $0.50/1M | $1.50/1M | $0.002-0.005 |
| GPT-4-turbo | $10/1M | $30/1M | $0.05-0.15 |
| GPT-4o | $5/1M | $15/1M | $0.025-0.075 |

### Monthly Cost Estimates

**Embeddings Only** (indexing documents):

| Documents/Month | Total Tokens | Embedding Cost |
|-----------------|--------------|----------------|
| 100 | 500K | $0.01 |
| 1,000 | 5M | $0.10 |
| 10,000 | 50M | $1.00 |
| 100,000 | 500M | $10 |
| 1,000,000 | 5B | $100 |

**Query Costs** (with GPT-3.5-turbo):

| Queries/Month | Avg Context | Query Cost |
|---------------|-------------|------------|
| 100 | 2K tokens | $0.30 |
| 1,000 | 2K tokens | $3.00 |
| 10,000 | 2K tokens | $30 |
| 100,000 | 2K tokens | $300 |

**Total OpenAI Estimate** = Embeddings + Queries

- Light usage (100 docs, 500 queries/mo): **$2**
- Medium usage (1K docs, 5K queries/mo): **$15**
- Heavy usage (10K docs, 50K queries/mo): **$150**
- Enterprise (100K docs, 500K queries/mo): **$1,500**

### Optimization Options

**Use Smaller Models:**
- GPT-3.5-turbo: 90% cheaper than GPT-4
- GPT-4o-mini: 60% cheaper than GPT-4o

**Cache Responses:**
- OpenAI caching: 50% discount on cached prompts
- Implement application-level caching

**Batch Processing:**
- Use OpenAI Batch API: 50% discount
- Good for non-real-time embedding generation

---

## 🗄️ Milvus Cloud - Vector Database

### Configuration

- Vector dimension: 1536 (OpenAI embeddings)
- Index type: HNSW or IVF_FLAT
- Storage: Cloud-managed Milvus (Zilliz Cloud)

### Pricing (Zilliz Cloud)

**Compute Units (CU):**
- 1 CU = 1 vCPU + 4GB RAM
- Starting: 0.5 CU minimum
- Cost: **$0.12/hour per CU** (~$90/month per CU)

**Storage:**
- Vector storage: **$0.10/GB/month**
- Hot storage for fast retrieval

**Free Tier:**
- 1 CU for 7 days
- 5GB storage
- Good for development

### Monthly Cost Estimates

| Documents | Chunks | Storage | CU Needed | Monthly Cost |
|-----------|--------|---------|-----------|--------------|
| 1,000 | 25K | 0.5GB | 0.5 CU | $45 + $0.05 |
| 10,000 | 250K | 5GB | 0.5 CU | $45 + $0.50 |
| 100,000 | 2.5M | 50GB | 1 CU | $90 + $5 |
| 1,000,000 | 25M | 500GB | 2 CU | $180 + $50 |

**Note**: Self-hosted Milvus (on your own servers) eliminates these costs but adds infrastructure complexity.

### Optimization Options

**Self-Host Milvus:**
- On AWS/GCP/DigitalOcean VPS
- Cost: $40-100/month for VPS
- Saves money at scale but requires maintenance

**Use Lighter Embeddings:**
- `text-embedding-3-small` (1536 dim)
- Custom smaller models (768 or 384 dim)
- Reduces storage by 50-75%

**Implement Caching:**
- Cache common queries
- Reduces CU usage

---

## 📦 Cloudflare R2 - File Storage

### Configuration

- Object storage for uploaded documents
- S3-compatible API
- No egress fees

### Pricing

**Storage:**
- **$0.015/GB/month** (first 10GB free)

**Class A Operations** (writes):
- **$4.50 per million** requests
- Upload, create, list operations

**Class B Operations** (reads):
- **$0.36 per million** requests
- Download, get operations

### Monthly Cost Estimates

| Documents | Avg Size | Total Storage | Monthly Cost |
|-----------|----------|---------------|--------------|
| 100 | 1MB | 100MB | Free |
| 1,000 | 1MB | 1GB | Free |
| 10,000 | 1MB | 10GB | Free |
| 100,000 | 1MB | 100GB | $1.35 |
| 1,000,000 | 1MB | 1TB | $15 |

**Operations Cost:**
- 100K uploads: $0.45
- 1M reads: $0.36
- Generally negligible

### Why R2 vs S3?

| Feature | R2 | S3 |
|---------|----|----|
| Storage | $0.015/GB | $0.023/GB |
| Egress | **Free** | $0.09/GB |
| Operations | Cheaper | More expensive |
| **Savings** | **~60% cheaper** | Baseline |

---

## 🗃️ PostgreSQL - Metadata Database

### Configuration

- Stores user data, dataset metadata, document status
- Not vector storage (that's Milvus)
- Managed PostgreSQL recommended

### Pricing Options

**Supabase (Managed Postgres):**
- Free tier: Up to 500MB, 2 CPU
- Pro: $25/month (8GB DB, better CPU)
- Team: $599/month (larger)

**Railway.app:**
- $5/month base + usage
- ~$20-40/month typical

**AWS RDS:**
- t3.micro: $15/month
- t3.small: $30/month
- t3.medium: $60/month

**DigitalOcean Managed DB:**
- Basic: $15/month (1GB RAM)
- Standard: $60/month (4GB RAM)

### Monthly Cost Estimates

| Usage Level | Recommended | Cost |
|-------------|-------------|------|
| Development | Supabase Free | $0 |
| Starter | Supabase Pro | $25 |
| Growth | Railway/DO Basic | $30-40 |
| Scale | AWS RDS t3.medium | $60 |
| Enterprise | AWS RDS m5.large | $200+ |

### Optimization Options

**Use Connection Pooling:**
- PgBouncer or Supabase built-in
- Reduces connection overhead

**Optimize Queries:**
- Add indexes for common queries
- Use EXPLAIN ANALYZE for slow queries

**Archive Old Data:**
- Move old documents to cold storage
- Keep active data in hot database

---

## 💻 DigitalOcean - FastAPI Application Server

### Current Status: Recently Optimized ✅

**Date Optimized:** October 31, 2024

### Background

After refactoring document processing to Modal.com (October 30, 2024), the FastAPI application server's resource requirements dropped dramatically. The server now only handles:
- API requests (lightweight)
- Database queries (PostgreSQL)
- Session management
- Redis caching

**Heavy processing (document parsing, embedding generation) now happens on Modal.**

### Pre-Refactor Configuration (Deprecated)

**Previous Setup:**
- Size: $50/month droplet
- CPU: 4 vCPUs (Premium Intel or Regular)
- RAM: 8GB
- Storage: 160GB SSD
- **Problem:** Massive over-provisioning for current workload

**Observed Usage (Post-Refactor, after 11AM):**
- CPU: 5-10% baseline, occasional spikes to 40-50%
- Memory: ~20% utilization (~1.6GB out of 8GB)
- Load Average: 0.5-1.0 baseline, 2-3 max
- **Conclusion:** 80-90% of resources sitting idle

### Recommended Configuration

**Selected:** Basic Regular - $24/month
- **2 vCPUs** (Regular, not Premium Intel)
- **4 GB RAM** (critical advantage over Premium Intel's 2GB)
- **60 GB SSD**
- **4 TB bandwidth**

**Cost Savings:**
- **$26/month saved** ($312/year)
- **52% reduction** in server costs
- Still provides comfortable headroom for growth

### Why Regular Over Premium Intel?

At the same $24/month price point, two options were available:

| Feature | Premium Intel | Regular | Winner |
|---------|---------------|---------|--------|
| CPU Type | Premium (faster) | Standard | - |
| vCPUs | 2 | 2 | Tie |
| RAM | **2 GB** | **4 GB** | ✅ Regular |
| Storage | 60 GB | 60 GB | Tie |
| Bandwidth | 3 TB | 4 TB | Regular |
| Best For | CPU-intensive | Memory-intensive | - |

**Decision Rationale:**
1. **Current workload is I/O-bound, not CPU-bound** (database queries, network requests)
2. **Memory is the bottleneck** for this stack:
   - PostgreSQL benefits heavily from RAM for query caching
   - Redis requires in-memory storage
   - FastAPI + Uvicorn workers need memory per connection
   - OS overhead: ~500MB
3. **2GB would be dangerously close to limit:**
   ```
   OS:         ~500 MB
   PostgreSQL: ~500 MB
   Redis:      ~200 MB
   FastAPI:    ~300-500 MB
   Total:      1.5-1.7 GB (85%+ utilization!)
   ```
4. **4GB provides healthy headroom:**
   ```
   Services:   ~1.7 GB used
   Available:  2.3 GB buffer (comfortable)
   ```
5. **Premium Intel's faster CPU doesn't help** when CPU usage is already at 5-10%

### Monthly Cost Estimates

| Configuration | Monthly Cost | Annual Cost | Use Case |
|---------------|--------------|-------------|----------|
| ~~$50 droplet (Previous)~~ | ~~$50~~ | ~~$600~~ | Over-provisioned |
| **$24 droplet (Current)** | **$24** | **$288** | **Recommended ✅** |
| $12 droplet (Aggressive) | $12 | $144 | Risky for production |

**Savings vs Previous:** $312/year (52% reduction)

### Optimization Timeline

**Phase 1 (Completed):** Move document processing to Modal
- Date: October 30, 2024
- Impact: Freed up 80-90% of server resources
- Next: Monitor usage for 1 week

**Phase 2 (Recommended):** Downgrade to $24/month
- Target: November 2024
- Risk: Low (comfortable headroom)
- Monitoring: Watch memory usage closely for first week

**Phase 3 (Optional):** Consider $12/month if underutilized
- Target: Q1 2025
- Requirement: Sustained <50% memory usage
- Risk: Medium (less buffer for traffic spikes)

### Monitoring Recommendations

**Watch these metrics:**
1. **Memory usage:** Should stay below 60% consistently
2. **CPU usage:** Current 5-10% is healthy
3. **Response times:** Should remain stable after downgrade
4. **Database connection pool:** May need tuning with less RAM

**Set alerts for:**
- Memory usage >75%
- CPU sustained >70%
- Response time >500ms
- Database connection errors

### Alternative Hosting Options

If DigitalOcean becomes limiting, consider:

| Provider | Configuration | Cost | Pros | Cons |
|----------|--------------|------|------|------|
| Railway.app | 4GB RAM | $20-30/mo | Simple, auto-scaling | Variable pricing |
| Render.com | Starter | $25/mo | Easy setup | Less control |
| AWS Lightsail | 2GB RAM | $12/mo | Cheap, AWS ecosystem | Less powerful |
| Fly.io | 1GB shared CPU | $15-25/mo | Edge deployment | More complex |

**Recommendation:** Stick with DigitalOcean $24/month for now. It's a known quantity with good performance.

### Cost Impact on Total Infrastructure

**Before Refactor:**
- DigitalOcean: $50/month
- Modal: $0 (not using)
- **Total Compute:** $50/month

**After Refactor:**
- DigitalOcean: $24/month (52% reduction)
- Modal: ~$2-20/month (scales with usage)
- **Total Compute:** $26-44/month for light-medium usage

**Net Effect:** Similar or lower costs, but now scales properly with actual processing workload instead of requiring fixed capacity.

---

## 🌐 Vercel - Frontend Hosting

### Configuration

- Next.js application
- Serverless functions for API routes
- CDN for static assets

### Pricing

**Hobby (Free):**
- 100GB bandwidth/month
- Unlimited websites
- Serverless functions: 100 GB-hours

**Pro ($20/month):**
- 1TB bandwidth
- 1000 GB-hours functions
- Analytics included

**Enterprise (Custom):**
- Unlimited bandwidth
- Custom SLA
- Dedicated support

### Monthly Cost Estimates

| Users | Page Views | Recommended | Cost |
|-------|------------|-------------|------|
| <100 | <10K | Hobby | Free |
| 100-1,000 | 10K-100K | Pro | $20 |
| 1,000-10K | 100K-1M | Pro | $20 |
| 10K+ | 1M+ | Enterprise | $500+ |

**Vercel is typically cheap** - frontend hosting is inexpensive compared to backend processing.

### Optimization Options

**Use Static Generation:**
- Pre-render pages at build time
- Reduces serverless function calls

**Optimize Images:**
- Next.js Image optimization is built-in
- Reduces bandwidth

**Consider Alternatives:**
- Netlify (similar pricing)
- Self-host on VPS ($5-20/month)

---

## ☁️ Cloudflare Workers - Proxy & Webhooks

### Configuration

- R2 upload proxy
- Dataset upload notification worker
- Queue consumer worker

### Pricing

**Free Tier:**
- 100,000 requests/day
- 10ms CPU time per request
- More than enough for most use cases

**Paid Plans:**
- $5/month base
- $0.50 per million requests after 10M
- $0.02 per million GB-seconds

### Monthly Cost Estimates

For typical usage: **$0-5/month**

Workers are extremely cheap - unlikely to exceed free tier unless doing millions of requests.

---

## 💰 Total Cost Scenarios

### Scenario 1: Solo Developer / Prototype

**Usage:**
- 50 documents/month
- 200 queries/month
- 2-3 users

**Monthly Costs:**
- Modal: $0.10
- OpenAI: $1-2
- Milvus: $0 (free tier or self-hosted)
- R2: $0 (free tier)
- PostgreSQL: $0 (Supabase free)
- Vercel: $0 (Hobby)
- Workers: $0 (free tier)

**Total: ~$1-5/month**

---

### Scenario 2: Early Startup

**Usage:**
- 1,000 documents/month
- 5,000 queries/month
- 50 active users

**Monthly Costs:**
- Modal: $2
- OpenAI: $15-20 (embeddings + queries)
- DigitalOcean: $24 (FastAPI server)
- Milvus: $45 (Zilliz 0.5 CU)
- R2: $0 (free tier)
- PostgreSQL: $25 (Supabase Pro)
- Vercel: $20 (Pro)
- Workers: $0

**Total: ~$130-135/month**

**Revenue Needed:** ~25-30 paying users at $5/month

---

### Scenario 3: Growing Business

**Usage:**
- 10,000 documents/month
- 50,000 queries/month
- 500 active users

**Monthly Costs:**
- Modal: $20
- OpenAI: $150-180
- DigitalOcean: $24 (FastAPI server)
- Milvus: $90 (1 CU + storage)
- R2: $0.50
- PostgreSQL: $40 (Railway/DO)
- Vercel: $20
- Workers: $5

**Total: ~$350-380/month**

**Revenue Needed:** ~70-75 paying users at $5/month

**Margin:** Could support freemium model with 10% conversion

---

### Scenario 4: Established SaaS

**Usage:**
- 100,000 documents/month
- 500,000 queries/month
- 5,000 active users

**Monthly Costs:**
- Modal: $200
- OpenAI: $1,500-1,800
- DigitalOcean: $50-100 (scaled up or multiple servers)
- Milvus: $180 (2 CU + storage)
- R2: $5
- PostgreSQL: $100 (larger instance)
- Vercel: $20-50
- Workers: $5

**Total: ~$2,060-2,460/month**

**Revenue Needed:** ~400-500 paying users at $5/month

**Margin:** Healthy margins with proper pricing

---

### Scenario 5: Enterprise Scale

**Usage:**
- 1,000,000 documents/month
- 5,000,000 queries/month
- 50,000 active users

**Monthly Costs:**
- Modal: $2,000-2,500 (volume discounts)
- OpenAI: $15,000-18,000
- Milvus: $500-800 (self-hosted or enterprise)
- R2: $50
- PostgreSQL: $300-500
- Vercel: $500+
- Workers: $50

**Total: ~$18,000-22,000/month**

**Revenue Needed:** $50,000/month (2.5x multiple)

**At this scale:** Negotiate enterprise contracts, consider self-hosting more infrastructure

---

## 📈 Cost Scaling & Unit Economics

### Cost Per Document Processed

| Volume | Processing | Embedding | Storage (1yr) | Total |
|--------|-----------|-----------|---------------|-------|
| 1 doc | $0.002 | $0.0003 | $0.0002 | **$0.0025** |
| 100 docs | $0.20 | $0.03 | $0.02 | **$0.25** |
| 1K docs | $2 | $0.10 | $0.18 | **$2.28** |
| 10K docs | $20 | $1 | $1.80 | **$22.80** |
| 100K docs | $200 | $10 | $18 | **$228** |

### Cost Per Query

| Model | Input | Output | Total | Notes |
|-------|-------|--------|-------|-------|
| GPT-3.5 | $0.001 | $0.002 | **$0.003** | Fast, cheap |
| GPT-4o | $0.01 | $0.03 | **$0.04** | Better quality |
| GPT-4 | $0.02 | $0.06 | **$0.08** | Best quality, expensive |

**Recommendation:** Start with GPT-3.5, upgrade to GPT-4o for premium users

---

## 💡 Cost Optimization Strategies

### 1. Implement Tiered Pricing

**Free Tier:**
- 10 documents/month
- 50 queries/month
- Cost to serve: ~$0.50/month
- Purpose: Acquisition, trial

**Starter ($10/month):**
- 100 documents/month
- 500 queries/month
- Cost to serve: ~$3/month
- Margin: 70%

**Pro ($50/month):**
- 1,000 documents/month
- 5,000 queries/month
- Cost to serve: ~$15/month
- Margin: 70%

**Enterprise (Custom):**
- Unlimited
- Volume discounts
- Custom contracts

### 2. Batch Processing

**Benefits:**
- OpenAI Batch API: 50% discount
- Better Modal container utilization
- Reduced cold starts

**Implementation:**
- Queue non-urgent documents
- Process in batches every hour
- Use for bulk uploads

### 3. Caching Strategy

**Query Caching:**
- Cache common queries
- Redis/Upstash ($10-30/month)
- Saves 30-50% on OpenAI costs

**Embedding Caching:**
- Store embeddings in vector DB (already doing this)
- Don't re-embed same content

### 4. Right-Size Infrastructure

**Current Setup:**
- T4 GPU: $1.21/hour (only when processing)
- Scales to zero: No idle costs
- Good: Already optimized!

**Database:**
- Start small, upgrade as needed
- Don't over-provision early

### 5. Monitor and Alert

**Set up cost alerts:**
- Modal dashboard
- OpenAI usage dashboard
- Cloudflare analytics
- Set budget alerts at 80% threshold

### 6. Consider Self-Hosting at Scale

**When to self-host (>$5K/month spend):**

**Pros:**
- 50-70% cost reduction
- More control
- Predictable costs

**Cons:**
- DevOps overhead
- Maintenance burden
- Less flexibility

**What to self-host:**
- ✅ Milvus (easy, big savings)
- ✅ PostgreSQL (if large)
- ⚠️ Modal replacement (complex, use Lambda/Fargate)
- ❌ OpenAI (no alternative with same quality)

---

## 🎯 Pricing Strategy Recommendations

### Recommended User Pricing

Based on cost analysis and 60-70% target margin:

| Plan | Price | Documents | Queries | Cost | Margin |
|------|-------|-----------|---------|------|--------|
| Free | $0 | 10 | 50 | $0.50 | Loss leader |
| Starter | $15/mo | 100 | 500 | $3 | 80% |
| Pro | $49/mo | 500 | 2,500 | $12 | 75% |
| Business | $149/mo | 2,000 | 10,000 | $40 | 73% |
| Enterprise | Custom | Unlimited | Unlimited | Variable | 65-70% |

### Additional Revenue Streams

**Usage Overage:**
- $0.10 per additional document
- $0.01 per additional query
- Covers costs + margin

**API Access:**
- $99/month for API access
- Higher rate limits
- Programmatic access

**Premium Models:**
- Charge extra for GPT-4
- +$20/month for GPT-4o access
- Covers higher API costs

---

## 📊 Cost Monitoring & Alerts

### Tools to Use

**Modal Dashboard:**
- Track compute usage
- Set budget alerts
- Monitor function calls

**OpenAI Dashboard:**
- Daily token usage
- Cost breakdown by model
- Set hard limits

**Cloudflare Analytics:**
- R2 storage growth
- Worker invocations
- Bandwidth usage

**Custom Monitoring:**
```python
# Track costs per user/dataset
from datetime import datetime

def log_operation_cost(
    user_id: str,
    operation: str,
    cost: float
):
    """Log operation costs for monitoring"""
    # Store in database
    # Aggregate in analytics
    # Alert if exceeding thresholds
    pass
```

### Alert Thresholds

**Set alerts at:**
- 80% of monthly budget
- Unusual spike in usage (2x daily average)
- Per-user anomalies (>10x their normal usage)
- Failed operations (costing money with no value)

---

## 🚀 Cost Roadmap

### Phase 1: MVP (Months 1-3)
- Use all free tiers
- Optimize for development speed
- Expected: $10-50/month
- Target: 10-50 beta users

### Phase 2: Launch (Months 3-6)
- Upgrade to paid tiers as needed
- Implement basic monitoring
- Expected: $100-300/month
- Target: 50-200 paying users
- Revenue: $500-2,000/month

### Phase 3: Growth (Months 6-12)
- Optimize costs per unit
- Implement caching
- Expected: $500-1,500/month
- Target: 200-1,000 paying users
- Revenue: $5,000-20,000/month

### Phase 4: Scale (Year 2+)
- Consider self-hosting components
- Negotiate enterprise contracts
- Volume discounts
- Expected: $5,000-20,000/month
- Target: 1,000-10,000 users
- Revenue: $50,000-200,000/month

---

## 🎓 Key Takeaways

### ✅ What's Good About Current Setup

1. **Modal T4 GPU:** Faster AND cheaper per document
2. **Scales to Zero:** No fixed costs, pay only for usage
3. **R2 Storage:** 60% cheaper than S3
4. **Free Tiers:** Can start for <$10/month
5. **Right-Sized:** Not over-provisioned

### ⚠️ Watch Out For

1. **OpenAI Costs:** Will be 30-40% of total spend
2. **GPT-4 Pricing:** 20x more expensive than GPT-3.5
3. **Query Volume:** Can explode with active users
4. **Milvus Cloud:** Consider self-hosting at scale

### 💡 Quick Wins

1. **Implement query caching** → Save 30-50% on OpenAI
2. **Use GPT-3.5 by default** → Save 90% vs GPT-4
3. **Batch processing** → Save 50% on embeddings
4. **Monitor per-user costs** → Identify abuse early
5. **Set hard limits** → Prevent bill shock

---

## 📞 When to Negotiate Enterprise Contracts

**Modal.com:**
- Contact: >$5,000/month spend
- Ask for: Volume discounts, reserved capacity

**OpenAI:**
- Contact: >$10,000/month spend
- Ask for: Volume discounts, dedicated capacity, rate limits

**Milvus/Zilliz:**
- Contact: >2 CU needed or >$500/month
- Ask for: Annual discount, custom SLA

---

## 📝 Cost Tracking Template

### Monthly Cost Tracking Sheet

```
Date: ___________
Total Documents Processed: ___________
Total Queries Handled: ___________
Active Users: ___________

Breakdown:
- Modal.com:          $______
- OpenAI API:         $______
- Milvus Cloud:       $______
- Cloudflare R2:      $______
- PostgreSQL:         $______
- Vercel:             $______
- Cloudflare Workers: $______
- Other:              $______

Total Infrastructure: $______
Revenue:              $______
Gross Margin:         ______%

Cost Per Document:    $______
Cost Per Query:       $______
Cost Per User:        $______
```

---

## 🔮 Future Considerations

### As You Scale, Consider

1. **Custom Embedding Models:**
   - Fine-tune smaller models
   - 75% cost reduction
   - Requires ML expertise

2. **Self-Hosted LLMs:**
   - Llama 3, Mistral, etc.
   - Fixed cost instead of per-token
   - Good for >100K queries/month

3. **Multi-Cloud Strategy:**
   - Split workloads across providers
   - Optimize for each service's strengths
   - Negotiate better rates

4. **Serverless GPU Alternatives:**
   - RunPod, Together AI, Replicate
   - Compare pricing with Modal
   - May be cheaper at scale

---

**Document Version:** 1.1
**Last Updated:** October 31, 2024
**Next Review:** When monthly costs exceed $1,000 or significant architecture changes occur.

**Recent Changes (v1.1):**
- Added DigitalOcean FastAPI server optimization section
- Documented 52% cost reduction on application server ($50 → $24/month)
- Analysis of Regular vs Premium Intel configurations
- Post-Modal refactor resource utilization data

For questions or to discuss cost optimization strategies, see your infrastructure team.

