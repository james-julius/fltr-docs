# FLTR MCP Deployment & Traction Plan

## Overview
This document outlines the strategy for deploying Model Context Protocol (MCP) integration for FLTR, enabling users to query datasets directly from Claude Desktop and other MCP-compatible clients.

---

## MCP Ecosystem Landscape

### What is MCP?
Model Context Protocol (MCP) is an open standard created by Anthropic that enables AI applications to securely connect to external data sources and tools. It provides a standardized way for LLMs to access context beyond their training data.

### Current Ecosystem Size
- **6,490+ MCP servers** cataloged and growing (PulseMCP directory)
- **Official Anthropic servers**: Filesystem, Git, GitHub, Postgres, Slack, Google Drive, Puppeteer, Fetch, Sequential Thinking, Memory
- **Popular community servers**: AWS S3, Airtable, Gmail, Notion, Linear, Salesforce
- **Enterprise integrations**: 500+ business apps via platforms like Composio

### Distribution Methods (2025)

#### Traditional Installation
```json
{
  "mcpServers": {
    "server-name": {
      "command": "python",
      "args": ["-m", "package_name"],
      "env": {
        "API_KEY": "user-key"
      }
    }
  }
}
```
Users manually edit `claude_desktop_config.json`

#### Claude Desktop Extensions (.mcpb) - NEW!
- Bundle entire MCP server with dependencies
- One-click installation (double-click file)
- Automatic configuration
- **This is our primary target**

### Why FLTR + MCP is Powerful

**For Users:**
- Query datasets directly in Claude conversations
- No context switching between web UI and chat
- Natural language data analysis
- Chain queries across multiple datasets
- Build custom AI agents with access to curated data

**For Dataset Creators:**
- New distribution channel (6,490+ existing MCP users)
- Lower friction for data access
- Enables AI-native data products
- Monetize through credit-based usage

**For FLTR Platform:**
- Differentiation from traditional data marketplaces
- AI-first positioning
- Increased dataset visibility and usage
- Network effects from MCP ecosystem

### Real-World MCP Examples

Understanding successful MCP servers helps us design FLTR's integration:

#### Official Anthropic Servers

**Postgres MCP**
- **What it does**: Connect Claude to PostgreSQL databases
- **Tools**: `query`, `list_tables`, `describe_table`, `create_table`
- **Lesson for FLTR**: Clear tool names, schema inspection is valuable

**GitHub MCP**
- **What it does**: Search code, create issues, manage PRs
- **Tools**: `search_code`, `create_issue`, `get_file_contents`
- **Lesson for FLTR**: Natural language search is key

**Filesystem MCP**
- **What it does**: Secure local file operations
- **Tools**: `read_file`, `write_file`, `list_directory`, `search_files`
- **Lesson for FLTR**: Need both browsing AND querying tools

#### Community Success Stories

**AWS S3 MCP** (Community)
- Enables Claude to read/write to S3 buckets
- Popular for data analysis workflows
- **Lesson**: Integration with data storage is highly valued

**Notion MCP** (Community)
- Query Notion databases, create pages
- Used for personal knowledge management
- **Lesson**: Users want to work with their own data

**Airtable MCP** (Community)
- Read/write access to Airtable bases
- Schema inspection included
- **Lesson**: Structured data access is in demand

### How FLTR Fits In

FLTR would be the **first curated data marketplace** in the MCP ecosystem:
- Not just personal data (Notion, Airtable)
- Not just raw storage (S3, Drive)
- **Curated, purchasable datasets accessible via natural language**

This is a unique positioning that doesn't currently exist in the ecosystem.

---

## Integration Approach Comparison

We have three main integration options. Here's a detailed comparison:

### Option 1: Official Python Package in MCP Directory

**How it works:**
```bash
# Installation
pip install fltr-mcp
# or with uvx (no install needed)
uvx fltr-mcp --dataset-id abc123 --api-key xyz
```

**Configuration:**
```json
{
  "fltr": {
    "command": "python",
    "args": ["-m", "fltr_mcp"],
    "env": {
      "FLTR_API_KEY": "user-api-key-here"
    }
  }
}
```

**Pros:**
- ✅ Professional, discoverable in official directories
- ✅ Single install, works for all FLTR datasets
- ✅ Easy to update (pip upgrade)
- ✅ Familiar pattern for developers
- ✅ Can support both global and per-dataset modes

**Cons:**
- ❌ Requires manual config editing
- ❌ Setup friction for non-technical users
- ❌ Users need Python installed

**Best for:** Developers, technical users, long-term adoption

---

### Option 2: .mcpb Extension Files (One-Click Install)

**How it works:**
```
User clicks "Connect to Claude" → Downloads dataset-name.mcpb → Double-click → Done
```

**Two variants:**

#### A. Global FLTR MCP (fltr.mcpb)
- Access all datasets user owns
- Single authentication
- Dataset discovery tools included

#### B. Per-Dataset MCP (sec-filings-2024.mcpb)
- Optimized for specific dataset
- Pre-configured dataset ID
- Lighter weight, focused tools

**Pros:**
- ✅ Zero configuration required
- ✅ Non-technical users can install
- ✅ Perfect for dataset marketplace ("Download → Use in Claude")
- ✅ Can bundle Python runtime if needed
- ✅ Automatic updates possible

**Cons:**
- ❌ Newer format, less documentation
- ❌ Need to build .mcpb generation pipeline
- ❌ Separate file per dataset (for per-dataset approach)

**Best for:** Mass market, ease of adoption, dataset marketplace integration

---

### Option 3: Copy-Paste Config Generator

**How it works:**
Dataset page shows generated config, user copies and pastes into their Claude config file.

```json
{
  "fltr-sec-filings": {
    "command": "uvx",
    "args": ["fltr-mcp", "--dataset-id", "abc123", "--api-key", "xyz789"]
  }
}
```

**Pros:**
- ✅ No backend infrastructure needed
- ✅ Works immediately if user has uvx
- ✅ Easy to implement
- ✅ Good for developer users

**Cons:**
- ❌ Still requires manual editing
- ❌ API key visible in config (security concern)
- ❌ Less user-friendly than .mcpb
- ❌ Hard to update or revoke access

**Best for:** Developer early adopters, MVP testing

---

### Recommended Strategy: Hybrid Approach

**Phase 1:** Start with Option 1 (Python package) + Option 3 (copy-paste configs)
- Fast to build
- Validates the concept
- Gets us into MCP directories quickly

**Phase 2:** Add Option 2 (.mcpb files) once validated
- Research .mcpb format
- Build generation pipeline
- Maximize user adoption

---

## Credit Tracking Implementation Patterns

A critical question: How do we track credit usage when users query via MCP?

### Pattern 1: API Key Based Metering (Recommended)

**Implementation:**
```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP(name="FLTR Dataset")

@mcp.tool()
async def query_dataset(query: str, dataset_id: str) -> dict:
    """Query a FLTR dataset with natural language"""
    # Get API key from environment
    api_key = os.getenv("FLTR_API_KEY")

    # Call FLTR API
    response = await fltr_api.query(
        dataset_id=dataset_id,
        query=query,
        api_key=api_key  # ← Backend tracks usage by this key
    )

    # Return results with usage info
    return {
        "data": response.data,
        "credits_used": response.credits_used,
        "credits_remaining": response.credits_remaining,
        "query_cost": response.query_cost
    }
```

**Backend Flow:**
1. MCP server sends API key with every request
2. FLTR API validates key
3. Deducts credits from user's account
4. Returns usage info in response
5. User sees credit balance in Claude

**Pros:**
- ✅ Simple implementation
- ✅ Works with existing API infrastructure
- ✅ Easy to track and bill
- ✅ Users control their keys
- ✅ Can generate dataset-specific keys

**Cons:**
- ❌ API key visible in config file
- ❌ User must manage keys manually

---

### Pattern 2: OAuth Flow (More Complex)

**Implementation:**
```python
@mcp.tool()
async def query_dataset(ctx: Context, query: str, dataset_id: str) -> dict:
    # Get OAuth token from context (stored after initial auth)
    token = await ctx.get_oauth_token()

    # Use token for API calls
    response = await fltr_api.query(
        dataset_id=dataset_id,
        query=query,
        bearer_token=token
    )

    return response
```

**Flow:**
1. First use triggers OAuth flow in browser
2. User authenticates with FLTR
3. Token stored securely by MCP client
4. All subsequent requests use token
5. Backend tracks usage by user account

**Pros:**
- ✅ More secure (no API keys in config)
- ✅ Better UX (authenticate once)
- ✅ Can request specific scopes
- ✅ Easy to revoke access

**Cons:**
- ❌ Complex to implement
- ❌ Requires OAuth infrastructure
- ❌ MCP OAuth support still maturing
- ❌ Harder to debug

---

### Pattern 3: Hybrid - Dataset-Specific Keys

**Implementation:**
Generate a unique, scoped API key per dataset connection:

```python
# When user clicks "Connect to Claude"
dataset_key = fltr.generate_scoped_key(
    user_id=user.id,
    dataset_id=dataset.id,
    scopes=["query", "schema"],
    expires_in="90d"
)

# Key only works for this specific dataset
# Easy to revoke if needed
```

**Pros:**
- ✅ Granular access control
- ✅ Easy to revoke per-dataset
- ✅ Better security than global keys
- ✅ Can track usage per dataset easily

**Cons:**
- ❌ User needs multiple keys for multiple datasets
- ❌ Key management complexity

---

### Recommended: Start with Pattern 1, Migrate to Pattern 3

**Phase 1:** API key-based (simple, fast to ship)
**Phase 2:** Add dataset-specific keys (better security)
**Phase 3:** Consider OAuth for enterprise users

---

## Two-Tier MCP Strategy

### 1. Global FLTR MCP (Platform Level)
**Package:** `fltr.mcpb`

**Tools:**
- `search_datasets(query)` - Find datasets across FLTR marketplace
- `query_dataset(dataset_id, query)` - Query any dataset user has access to
- `list_my_datasets()` - Show user's purchased/accessible datasets
- `get_dataset_schema(dataset_id)` - Inspect dataset structure
- `browse_categories()` - Explore dataset marketplace
- `get_credit_balance()` - Check remaining credits

**Use Case:**
> "Claude, find me financial datasets about tech companies, then query the most relevant one for Apple's revenue trends"

**Installation:**
- One-click `.mcpb` file download
- Works across all user's datasets
- Single API key authentication

### 2. Per-Dataset MCP (Dataset Level)
**Package:** `{dataset-name}.mcpb`

**Tools:**
- `query(query)` - Query this specific dataset
- `get_schema()` - View dataset columns/structure
- `get_stats()` - Dataset stats, size, and credit cost
- `sample_data(limit)` - Preview sample rows
- `filter_data(filters)` - Apply structured filters

**Use Case:**
> "Claude, analyze this SEC filings dataset for trends in tech sector governance changes over the past 3 years"

**Installation:**
- "Connect to Claude" button on each dataset page
- Downloads dataset-specific `.mcpb` file
- Pre-configured with dataset ID
- Zero configuration needed

---

## Phase 1: Foundation (Week 1-2)

### Build Core Infrastructure

**1.1 Create Python MCP Server Package**
- Build `fltr-mcp` using FastMCP framework
- Support dataset ID and API key via args/env vars
- Implement core tools:
  - `query_dataset(dataset_id, query, api_key)`
  - `get_schema(dataset_id, api_key)`
  - `filter_data(dataset_id, filters, api_key)`
  - `get_credit_balance(api_key)`
- Return credit usage in responses
- Add error handling and rate limiting
- Include credit cost estimates before queries

**1.2 Distribution Setup**
- Publish to PyPI as `fltr-mcp`
- Create GitHub repository with examples
- Support both installation methods:
  - `pip install fltr-mcp`
  - `uvx fltr-mcp` (no install needed)

**1.3 Testing & Validation**
- Test with 3-5 pilot datasets
- Validate credit tracking works correctly
- Test in Claude Desktop with real queries
- Verify error messages are clear and helpful

---

## Phase 2: User Experience (Week 2-3)

### Make Integration Effortless

**2.1 Auto-Generated Configs**
- Add "Connect to Claude" button on dataset pages
- Generate ready-to-paste config JSON for manual setup
- Generate downloadable `.mcpb` file for one-click install
- Include user's API key automatically (with privacy warnings)
- Add copy-to-clipboard functionality

**2.2 Documentation**
- Quick-start guide (< 5 min to first query)
- Video walkthrough (2-3 min)
- Troubleshooting FAQ
- Example prompts users can try per dataset type
- Installation instructions for both methods

**2.3 Developer Experience**
- Clear error messages with credit balance
- Tool descriptions optimized for LLM understanding
- Sample use cases per dataset type
- Credit cost transparency in tool descriptions

---

## Phase 3: Distribution & Discovery (Week 3-4)

### Get FLTR Into the Ecosystem

**3.1 Official MCP Directory Submission**
- Submit to [modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers)
- Submit to [PulseMCP directory](https://www.pulsemcp.com/servers)
- Submit to [MCP Server Finder](https://www.mcpserverfinder.com/)
- Get listed on Claude Directory

**3.2 Build .mcpb Extension Support**
- Research .mcpb format requirements
- Generate per-dataset .mcpb files dynamically
- Create global FLTR .mcpb file
- One-click installation experience
- Host downloads on FLTR domain

**3.3 SEO & Discoverability**
- Create landing page: "FLTR MCP Integration"
- Blog post: "Query [Your Niche] Data in Claude"
- Schema markup for search engines
- Tag datasets with "MCP-compatible" badge
- Add MCP section to homepage

---

## Phase 4: Launch & Awareness (Week 4-5)

### Drive Initial Adoption

**4.1 Soft Launch**
- Email existing users about MCP support
- Offer bonus credits for first MCP connection
- Collect feedback from early adopters
- Create feedback loop for improvements

**4.2 Content Marketing**
- Technical blog post on implementation
- Twitter/X thread with live demo
- Post to relevant communities:
  - r/ClaudeAI
  - r/LangChain
  - r/LLMs
  - r/MachineLearning
- Share on Hacker News (Show HN)
- LinkedIn post targeting data professionals

**4.3 Community Engagement**
- Join MCP Discord/Slack communities
- Answer questions about data access via MCP
- Share example use cases
- Contribute to MCP discussions
- Offer to help others integrate

---

## Phase 5: Growth & Iteration (Week 5-8)

### Build Momentum

**5.1 Partner with Dataset Creators**
- Reach out to top dataset publishers
- Highlight MCP as distribution channel
- Feature "MCP-ready" datasets prominently
- Create co-marketing opportunities
- Revenue sharing incentives

**5.2 Advanced Features**
- Streaming support for large results
- Caching for common queries
- Multi-dataset queries in single prompt
- Usage analytics dashboard
- Query history and favorites

**5.3 Case Studies & Social Proof**
- Document 3-5 compelling use cases
- Get testimonials from power users
- Create demo videos showing real workflows
- Measure and publish usage metrics
- Feature user success stories

---

## Phase 6: Scale & Optimize (Ongoing)

### Maximize Impact

**6.1 Analytics & Metrics**
Track:
- MCP installations (global vs per-dataset)
- Queries per dataset via MCP
- Credit usage through MCP vs web
- User retention rates
- Query complexity and patterns

Monitor:
- Error rates and types
- Popular datasets via MCP
- Time to first query
- Support tickets related to MCP

**6.2 Community Building**
- Create templates for common query patterns
- Build example agents that use FLTR datasets
- Host office hours for integration help
- Feature community-built integrations
- Reward power users

**6.3 Enterprise Features** (Long-term)
- Team API key management
- Usage quotas and billing tiers
- Private dataset MCP servers
- Custom MCP server deployments
- SSO integration
- Audit logs

---

## Key Success Metrics

### Week 4 Targets:
- ✅ 50+ MCP server installations
- ✅ Listed in 3+ MCP directories
- ✅ 10+ active users making queries
- ✅ Documentation published

### Week 8 Targets:
- ✅ 200+ installations
- ✅ 100+ datasets connected via MCP
- ✅ Featured use case on social media
- ✅ 5+ organic mentions/shares
- ✅ 1000+ queries executed

### Month 3 Targets:
- ✅ Top 20 MCP server by usage
- ✅ 1000+ installations
- ✅ Partnership with 3+ major dataset creators
- ✅ Clear ROI on credit usage vs growth
- ✅ 10,000+ queries executed

---

## Quick Wins (Priority Order)

1. **This Week**: Build basic Python MCP server (`fltr-mcp`)
2. **Next Week**: Add "Connect to Claude" button to dataset pages
3. **Week 3**: Submit to MCP directories + generate `.mcpb` files
4. **Week 4**: Launch announcement across all channels

---

## Resources Needed

### Development
- **Week 1-2**: 20-30 hours (Python MCP server + API integration)
- **Week 3-4**: 15-20 hours (.mcpb generation + UI integration)
- **Ongoing**: 5 hours/week (maintenance + features)

### Design
- **Week 2**: 5 hours (UI for config generation + docs)
- **Week 3**: 3 hours (Landing page + badges)

### Marketing
- **Week 4**: 10 hours (content creation, social posts)
- **Ongoing**: 3 hours/week (community engagement)

### Support
- **Ongoing**: 5 hours/week (community support, Q&A)

---

## Technical Architecture

### MCP Server Structure
```
fltr-mcp/
├── src/
│   ├── fltr_mcp/
│   │   ├── __init__.py
│   │   ├── server.py          # Main FastMCP server
│   │   ├── tools.py           # MCP tool implementations
│   │   ├── client.py          # FLTR API client
│   │   └── auth.py            # API key handling
├── tests/
├── examples/
├── README.md
├── pyproject.toml
└── LICENSE
```

### Example Implementation

Here's what the core MCP server code would look like:

```python
# src/fltr_mcp/server.py
from mcp.server.fastmcp import FastMCP
from typing import Optional
import os
from .client import FLTRClient

# Initialize MCP server
mcp = FastMCP(name="FLTR")

# Initialize FLTR API client
def get_client():
    api_key = os.getenv("FLTR_API_KEY")
    if not api_key:
        raise ValueError("FLTR_API_KEY environment variable not set")
    return FLTRClient(api_key=api_key)

@mcp.tool()
async def search_datasets(query: str, category: Optional[str] = None) -> dict:
    """
    Search for datasets across the FLTR marketplace

    Args:
        query: Natural language search query
        category: Optional category filter (e.g., "finance", "healthcare")

    Returns:
        List of matching datasets with descriptions and pricing
    """
    client = get_client()
    results = await client.search_datasets(query=query, category=category)

    return {
        "datasets": results,
        "count": len(results),
        "credits_remaining": client.get_credits()
    }

@mcp.tool()
async def query_dataset(dataset_id: str, query: str) -> dict:
    """
    Query a specific dataset using natural language

    Args:
        dataset_id: The ID of the dataset to query
        query: Natural language query (e.g., "Show me all records from 2024")

    Returns:
        Query results with credit usage information
    """
    client = get_client()

    # Estimate cost before querying
    estimate = await client.estimate_query_cost(dataset_id, query)

    # Execute query
    result = await client.query(dataset_id=dataset_id, query=query)

    return {
        "data": result.data,
        "row_count": len(result.data),
        "credits_used": result.credits_used,
        "credits_remaining": result.credits_remaining,
        "query_cost_estimate": estimate.cost
    }

@mcp.tool()
async def get_dataset_schema(dataset_id: str) -> dict:
    """
    Get the schema/structure of a dataset

    Args:
        dataset_id: The ID of the dataset

    Returns:
        Dataset schema including column names, types, and descriptions
    """
    client = get_client()
    schema = await client.get_schema(dataset_id)

    return {
        "dataset_id": dataset_id,
        "name": schema.name,
        "columns": schema.columns,
        "row_count": schema.row_count,
        "size_mb": schema.size_mb,
        "credits_per_query": schema.credits_per_query
    }

@mcp.tool()
async def list_my_datasets() -> dict:
    """
    List all datasets the user has access to

    Returns:
        List of accessible datasets with usage stats
    """
    client = get_client()
    datasets = await client.list_my_datasets()

    return {
        "datasets": datasets,
        "count": len(datasets),
        "total_credits_remaining": client.get_credits()
    }

@mcp.tool()
async def get_credit_balance() -> dict:
    """
    Check remaining credit balance

    Returns:
        Current credit balance and usage stats
    """
    client = get_client()
    balance = await client.get_balance()

    return {
        "credits_remaining": balance.remaining,
        "credits_used_today": balance.used_today,
        "credits_total": balance.total,
        "subscription_tier": balance.tier
    }

# Run the server
if __name__ == "__main__":
    mcp.run()
```

This implementation provides:
- ✅ Natural language dataset search
- ✅ Schema inspection before querying
- ✅ Cost estimates before execution
- ✅ Credit balance tracking
- ✅ Clear error messages
- ✅ Type hints for better IDE support

### Authentication Flow
1. User provides API key via environment variable or CLI arg
2. MCP server validates key with FLTR API
3. Each query includes key for credit tracking
4. Response includes remaining credit balance
5. Clear errors if credits exhausted

### Credit Tracking
- Every MCP query deducts credits on backend
- Response headers include:
  - `X-Credits-Used`: Credits consumed by this query
  - `X-Credits-Remaining`: User's remaining balance
- Tools show credit cost estimates in descriptions

---

## Risk Mitigation

### Technical Risks
- **Risk**: MCP spec changes
  - **Mitigation**: Stay updated with MCP community, test against beta releases

- **Risk**: Rate limiting/abuse
  - **Mitigation**: Implement server-side rate limits, credit checks before queries

### Business Risks
- **Risk**: Low adoption
  - **Mitigation**: Start with pilot users, iterate on feedback quickly

- **Risk**: Credit usage cannibalization
  - **Mitigation**: Track web vs MCP usage, optimize pricing if needed

### Competitive Risks
- **Risk**: Other data platforms add MCP
  - **Mitigation**: Be first to market, build best UX, integrate deeply

---

## Next Steps

1. **Immediate**: Review and approve this plan
2. **Week 1 Day 1**: Set up Python project structure
3. **Week 1 Day 2-3**: Implement core MCP tools
4. **Week 1 Day 4-5**: Test with real datasets
5. **Week 2 Day 1-2**: Build config generation on dataset pages
6. **Week 2 Day 3-5**: Create documentation and examples
7. **Week 3 Day 1-2**: Research and implement `.mcpb` format
8. **Week 3 Day 3-5**: Submit to directories and launch

---

## Questions to Resolve

1. **Pricing**: Same credit costs for MCP vs web queries?
2. **API Keys**: Generate separate MCP-specific keys or reuse existing?
3. **Rate Limits**: Different limits for MCP vs web?
4. **Analytics**: Track MCP usage separately in dashboard?
5. **Support**: Dedicated support channel for MCP users?

---

## References

- [MCP Official Specification](https://modelcontextprotocol.io/)
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [FastMCP Documentation](https://github.com/jlowin/fastmcp)
- [Claude Desktop Extensions (.mcpb)](https://www.anthropic.com/engineering/desktop-extensions)
- [MCP Server Directory](https://github.com/modelcontextprotocol/servers)

---

## Summary

FLTR's MCP integration positions us as the **first curated data marketplace in the MCP ecosystem**. By combining:
- ✅ One-click `.mcpb` installation (both global and per-dataset)
- ✅ Python-based open source implementation
- ✅ Credit-based metering with transparent usage tracking
- ✅ Natural language querying of premium datasets

We can tap into the growing ecosystem of 6,490+ MCP users and establish FLTR as the go-to platform for AI-accessible data.

**Next Action**: Begin Phase 1 (Python MCP server development)

---

**Last Updated**: 2025-11-05 (Enhanced with ecosystem context and implementation patterns)
**Status**: Planning Phase - Ready for Development
**Owner**: [Your Name]
