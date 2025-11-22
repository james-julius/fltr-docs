# FLTR MCP Business Model Decision

**Critical Question:** Are we a RAG-as-a-Service accessible via MCP, or a Dataset Marketplace with local MCP servers?

---

## 🔀 Two Models Compared

### Model A: RAG-as-a-Service via MCP (Current Implementation)

**How it works:**
```
User's Claude Desktop → MCP (stdio) → FLTR API → FLTR's Milvus/DB → Results
```

**User Experience:**
1. Install FLTR MCP server: `pip install fltr-mcp`
2. Configure with API key
3. Query datasets via Claude (data stays on FLTR servers)
4. Pay per query (credit system)

**Characteristics:**
- ✅ Data hosted by FLTR
- ✅ Users pay per query (credit model)
- ✅ No data downloads
- ✅ Central updates (new datasets available instantly)
- ✅ Security (data doesn't leave FLTR)
- ❌ Requires internet connection
- ❌ Higher latency (API calls)
- ❌ Lock-in to FLTR service

**Revenue Model:**
- Credit-based: $0.01 per query
- Subscription: $19/mo unlimited queries
- Enterprise: Custom pricing

**Examples:**
- Perplexity API (RAG as a service)
- OpenAI Assistants API (hosted vector stores)
- Pinecone (hosted vector DB)

---

### Model B: Dataset Marketplace + Local MCP (Download & Run)

**How it works:**
```
User's Claude Desktop → Local MCP Server → Local SQLite/Milvus → Results
```

**User Experience:**
1. Purchase dataset on FLTR marketplace
2. Download dataset bundle (includes MCP server)
3. Run locally: `fltr-mcp --dataset sec-filings-2024.db`
4. Query via Claude (all local, no internet needed)
5. One-time payment per dataset

**Characteristics:**
- ✅ Data runs locally (fast, private)
- ✅ No ongoing costs (one-time purchase)
- ✅ Works offline
- ✅ User owns the data
- ❌ Downloads can be large (GBs)
- ❌ Updates require new downloads
- ❌ Setup complexity (local vector DB)
- ❌ Version fragmentation

**Revenue Model:**
- Dataset sales: $49-$499 per dataset
- Bundles: $999 for 10 datasets
- No recurring revenue

**Examples:**
- Kaggle (download datasets)
- AWS Data Exchange (download data)
- Hugging Face Datasets (download + local processing)

---

## 🎯 Recommended: Hybrid Model

**Best of both worlds:**

### Tier 1: Free/Freemium - Browse & Sample
- Free MCP server with 100 free queries/month
- Access to sample datasets
- Limited to top 3 results per query
- Goal: User acquisition, viral growth

### Tier 2: Cloud RAG (Model A)
- $19/mo for unlimited queries
- $0.01/query pay-as-you-go
- All datasets accessible via API
- No downloads required
- **Target:** Casual users, researchers, analysts

### Tier 3: Dataset Purchase + Local MCP (Model B)
- Buy individual datasets: $49-$499
- Includes local MCP server
- Unlimited queries (runs locally)
- Download dataset + embeddings bundle
- **Target:** Power users, enterprises, offline use

### Tier 4: Enterprise (Hybrid)
- Custom deployment (cloud or on-prem)
- White-label MCP server
- Direct database access
- SLA guarantees
- **Target:** Large organizations

---

## 📊 Strategic Comparison

| Factor | Model A (RAG SaaS) | Model B (Marketplace) | Hybrid |
|--------|-------------------|----------------------|---------|
| **User acquisition** | ⭐⭐⭐⭐ (Easy install) | ⭐⭐ (Complex setup) | ⭐⭐⭐⭐⭐ (Multiple entry points) |
| **Recurring revenue** | ⭐⭐⭐⭐⭐ (Subscription) | ⭐ (One-time only) | ⭐⭐⭐⭐⭐ (Both models) |
| **Viral potential** | ⭐⭐⭐⭐ (Easy to share) | ⭐⭐ (Harder to demo) | ⭐⭐⭐⭐⭐ (Free tier viral) |
| **Margins** | ⭐⭐⭐ (Infrastructure costs) | ⭐⭐⭐⭐⭐ (Sell once, no hosting) | ⭐⭐⭐⭐ (Blended) |
| **Lock-in** | ⭐⭐⭐⭐⭐ (Strong) | ⭐ (Weak) | ⭐⭐⭐⭐ (Flexible) |
| **Privacy** | ⭐⭐ (Data on our servers) | ⭐⭐⭐⭐⭐ (Local) | ⭐⭐⭐⭐ (User choice) |
| **Speed** | ⭐⭐⭐ (API latency) | ⭐⭐⭐⭐⭐ (Local) | ⭐⭐⭐⭐ (Both options) |
| **Complexity** | ⭐⭐⭐⭐ (Simple for users) | ⭐⭐ (Complex setup) | ⭐⭐⭐ (More SKUs to manage) |

---

## 💡 Current Implementation = Model A

**Our existing MCP server is RAG-as-a-Service:**

```python
# Connects to FLTR's hosted infrastructure
DATABASE_URL = "postgresql://admin:password@localhost:5432/fltr_auth"
milvus = get_milvus_client()  # FLTR's Milvus instance
```

**This means:**
- Users install MCP server locally (thin client)
- All queries go to FLTR API
- Data stays on FLTR servers
- Credit-based pricing

---

## 🚀 Distribution Strategy Alignment

### For Model A (Current) - RAG SaaS

**Distribution:**
1. ✅ PyPI package: `pip install fltr-mcp`
2. ✅ Users configure with API key
3. ✅ Free tier: 100 queries/month
4. ✅ Upgrade to paid plans for more

**Monetization:**
```
Free → $0 (100 queries/month)
Starter → $19/mo (unlimited basic datasets)
Pro → $49/mo (all datasets, priority)
Enterprise → Custom (SLA, white-label)
```

**Market Position:**
- "Perplexity for Datasets"
- "RAG-as-a-Service for Curated Data"
- Compete with: OpenAI Assistants, Pinecone

**Distribution Strategy:**
- Focus on ease of use (2-minute setup)
- Free tier for viral growth
- Community edition for developers
- Enterprise edition for companies

---

### For Model B - Dataset Marketplace

**Distribution:**
1. ❌ Not current implementation
2. Would require:
   - Dataset export functionality
   - Local vector DB (SQLite + DuckDB)
   - Standalone MCP server per dataset
   - Download/license management

**Monetization:**
```
Individual Dataset → $49-$499 (one-time)
Bundle (10 datasets) → $999 (one-time)
Updates → $99/year per dataset
```

**Market Position:**
- "Kaggle meets MCP"
- "Downloadable datasets with AI interface"
- Compete with: Kaggle, AWS Data Exchange

**Distribution Strategy:**
- Focus on dataset quality
- One-time payment (higher price)
- Update subscriptions for new data
- Enterprise licensing

---

### For Hybrid Model (Recommended)

**Distribution:**
1. ✅ Start with Model A (current)
2. ✅ Add free tier (100 queries/month)
3. Later: Add downloadable datasets (Model B)

**Phased Rollout:**

**Phase 1 (Now):** Cloud RAG Only
- Launch MCP server as RAG SaaS
- Free tier: 100 queries/month
- Paid tier: $19/mo unlimited
- Focus on user acquisition

**Phase 2 (Month 3):** Add Dataset Downloads
- Popular datasets available for download
- Includes local MCP server
- Price: $99-$499 per dataset
- Target: Enterprise, offline use

**Phase 3 (Month 6):** Full Hybrid
- Users choose: Cloud or Local
- Bundle pricing: Cloud subscription + dataset downloads
- White-label for enterprise

---

## 🎯 RECOMMENDATION

### Start with Model A (Current Implementation) ✅

**Why:**
1. **Lower friction** - Users just install and query
2. **Viral growth** - Free tier drives adoption
3. **Recurring revenue** - Subscription model
4. **Fast iteration** - Update datasets instantly
5. **Already built** - Current implementation

**Adjust Distribution Strategy:**

Instead of positioning as "dataset marketplace," position as:

> **"RAG-as-a-Service for Curated Datasets"**
>
> Query 100+ curated datasets directly from Claude Desktop.
> No downloads, no setup, just natural language queries.

**Pricing:**
```
Free:     100 queries/month  (viral growth)
Starter:  $19/mo unlimited   (power users)
Pro:      $49/mo + priority  (professionals)
Enterprise: Custom           (teams)
```

**Key Metrics:**
- User signups (target: 1,000 in Month 1)
- Query volume (target: 10,000 queries/month)
- Conversion rate (free → paid: 5%)
- Monthly recurring revenue

---

## 📝 Updated Distribution Strategy

### Week 1: Launch Cloud RAG MCP

**Messaging:**
- ❌ "Download datasets and query locally"
- ✅ "Query curated datasets from Claude - no downloads needed"

**Value Props:**
1. **2-minute setup** - `pip install fltr-mcp`, add API key, done
2. **100 free queries** - Try before you buy
3. **100+ datasets** - Finance, legal, research, more
4. **Always up-to-date** - New datasets appear automatically
5. **Pay as you grow** - Free tier → $19/mo → Enterprise

**Target Users:**
- Financial analysts querying SEC filings
- Researchers searching academic papers
- Legal professionals searching case law
- Data scientists finding training data

**Go-to-Market:**
1. Free tier drives signups (100 queries/month)
2. Power users convert to $19/mo (unlimited basic)
3. Professionals upgrade to $49/mo (all datasets)
4. Enterprises get custom deployments

---

## 🔄 Migration Path to Hybrid (Future)

### If We Later Add Model B (Downloads):

**Month 3:** Survey users
- "Would you pay $99 to download SEC filings dataset and run locally?"
- "How important is offline access?"
- "Do you have privacy/compliance requirements?"

**Month 6:** Launch downloadable datasets
- Top 10 most-queried datasets available for download
- Price: $99-$499 per dataset
- Includes local MCP server
- Lifetime updates: +$49/year

**Positioning:**
- Cloud: "Easy, always updated, pay-as-you-go"
- Local: "Fast, private, one-time payment"
- Hybrid: "Best of both - subscribe + download favorites"

---

## ✅ Decision: Start with Model A

**Current implementation is perfect for:**
- Rapid user acquisition (free tier)
- Recurring revenue (subscriptions)
- Fast iteration (cloud-hosted)
- Viral growth (easy sharing)

**Adjust these docs:**
- [x] MCP_DISTRIBUTION_STRATEGY.md → Emphasize cloud RAG model
- [x] MCP_LAUNCH_CHECKLIST.md → Add free tier onboarding
- [x] MCP_SERVER_SETUP.md → Clarify it's a hosted service

**Next Steps:**
1. Add free tier (100 queries/month)
2. Update pricing page with MCP tiers
3. Adjust marketing to "RAG SaaS" positioning
4. Track conversion metrics (free → paid)

---

**Approved:** [Decision maker]

**Date:** [Today]

**Status:** Model A (Cloud RAG) - Ready to launch 🚀
