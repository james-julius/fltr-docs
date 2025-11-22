# FLTR MCP Launch Checklist (Two-Sided Marketplace)

**Supply Goal:** 20-50 datasets by Week 1, 200 by Month 6
**Demand Goal:** 100 installations in Week 1, 2,000 by Month 6
**Revenue Goal:** $100 GMV Week 1, $25,000 by Month 6

---

## 📦 Week 1: Seed Supply + Package & Publish (Days 1-7)

### Day 1-2: Seed Datasets + Create PyPI Package

#### Creator Acquisition (Supply Side)

- [ ] **Upload Initial Datasets (FLTR Team)**
  - [ ] SEC Filings (10-K, 10-Q, 8-K) - Set to $0.10/query
  - [ ] Legal Case Law (Federal & State) - Set to $0.15/query
  - [ ] Market Research Reports - Set to $0.20/query
  - [ ] Academic Papers (arXiv) - Set to FREE
  - [ ] Research Citations (PubMed) - Set to FREE

- [ ] **Create Sample Queries for Each Dataset**
  - [ ] SEC: "Find NVIDIA's AI revenue mentions"
  - [ ] Legal: "Search California employment law cases"
  - [ ] Research: "Query papers about transformer models"

- [ ] **Set Up Creator Infrastructure**
  - [ ] Basic creator onboarding flow (web UI)
  - [ ] Creator dashboard (show revenue, queries)
  - [ ] Stripe Connect integration (for payouts)

- [ ] **Identify Early Creator Partners**
  - [ ] Research 10 Kaggle top contributors (10k+ downloads)
  - [ ] Find 5 API providers on RapidAPI
  - [ ] Reach out to 3 research institutions
  - [ ] Draft outreach email (80% revenue share offer)

#### Consumer Packaging (Demand Side)

- [ ] Create PyPI Package Structure

- [ ] Create package structure:
  ```
  fltr-mcp/
  ├── pyproject.toml
  ├── setup.py
  ├── README.md
  ├── LICENSE
  ├── fltr_mcp/
  │   ├── __init__.py
  │   ├── server.py      (copy from fastapi/mcp_server.py)
  │   ├── cli.py         (CLI wrapper)
  │   └── config.py      (configuration)
  └── tests/
      └── test_server.py
  ```

- [ ] Write `pyproject.toml`:
  ```toml
  [project]
  name = "fltr-mcp"
  version = "1.0.0"
  description = "Query curated datasets from Claude Desktop via MCP"
  authors = [{name = "FLTR", email = "support@fltr.com"}]
  readme = "README.md"
  requires-python = ">=3.10"
  dependencies = [
      "mcp>=1.0.0",
      "sqlmodel>=0.0.14",
      "psycopg2-binary>=2.9.9",
      "pymilvus>=2.4.8",
      "openai>=1.10.0",
      "python-dotenv>=1.0.0"
  ]

  [project.scripts]
  fltr-mcp = "fltr_mcp.cli:main"
  ```

- [ ] Write comprehensive README.md with:
  - Hero image/GIF
  - Quick start (< 2 minutes to first query)
  - Examples (financial analysis, research, etc.)
  - Configuration options
  - Available datasets
  - Troubleshooting

- [ ] Add CLI wrapper (`fltr_mcp/cli.py`):
  ```python
  import argparse
  import asyncio
  from fltr_mcp.server import main

  def cli():
      parser = argparse.ArgumentParser(description="FLTR MCP Server")
      parser.add_argument("--api-key", help="Your FLTR API key")
      parser.add_argument("--transport", default="stdio", choices=["stdio", "sse"])
      args = parser.parse_args()

      asyncio.run(main())

  if __name__ == "__main__":
      cli()
  ```

### Day 3: Creator Outreach + Build Package

#### Creator Outreach (Supply Side)

- [ ] **Email Early Creator Partners**
  - [ ] Send personalized emails to 10 Kaggle contributors
  - [ ] Contact 5 API providers with partnership offer
  - [ ] Reach out to 3 research institutions
  - [ ] Highlight: 80% revenue share + featured placement

**Email Template:**
```
Subject: Monetize Your [Dataset Name] via AI Assistants (MCP)

Hi [Name],

Saw your [dataset] on [platform] - impressive quality!

We're launching FLTR, where data creators earn passive income
when AI assistants query their datasets.

Perfect for launch:
- 80% revenue share (vs. standard 70%)
- We handle hosting, embeddings, MCP distribution
- Featured on homepage
- Co-marketing (blog/case study)

5-min call to discuss?

[Your name]
```

- [ ] **Create Creator Landing Page**
  - [ ] "Monetize Your Dataset" page
  - [ ] Value props: passive income, no infrastructure
  - [ ] Calculator: "Earn $X/mo from Y queries"
  - [ ] CTA: "Apply as Early Creator"

#### Consumer Packaging (Demand Side)

- [ ] **Test & Build Package**
  - [ ] Test local installation:
    ```bash
    pip install -e .
    fltr-mcp --help
    ```

  - [ ] Build package:
    ```bash
    python -m build
    # Creates dist/fltr_mcp-1.0.0.tar.gz and .whl
    ```

  - [ ] Test installation from wheel:
    ```bash
    pip install dist/fltr_mcp-1.0.0-py3-none-any.whl
    fltr-mcp --version
    ```

### Day 4: Publish + Creator Follow-up

#### Consumer Launch (Demand Side)

- [ ] **Publish to PyPI**
  - [ ] Create PyPI account (if needed)
  - [ ] Generate API token
  - [ ] Upload to Test PyPI first:
    ```bash
    twine upload --repository testpypi dist/*
    pip install --index-url https://test.pypi.org/simple/ fltr-mcp
    ```

  - [ ] Upload to production PyPI:
    ```bash
    twine upload dist/*
    ```

  - [ ] Verify installation:
    ```bash
    pip install fltr-mcp
    fltr-mcp --help
    ```

#### Creator Follow-up (Supply Side)

- [ ] **Follow Up with Creator Partners**
  - [ ] Schedule calls with interested creators
  - [ ] Walk through onboarding process
  - [ ] Help upload first dataset (if needed)
  - [ ] Set up Stripe Connect for payouts

- [ ] **Verify Creator Infrastructure**
  - [ ] Test creator dashboard (revenue display)
  - [ ] Test dataset upload flow
  - [ ] Verify pricing configuration
  - [ ] Test sample queries on each dataset

### Day 5: Distribution + Creator Marketing

- [ ] **MCP GitHub Directory**
  - Fork: https://github.com/modelcontextprotocol/servers
  - Add to `servers.json`:
    ```json
    {
      "name": "fltr-datasets",
      "description": "Query 100+ curated datasets via natural language",
      "url": "https://github.com/fltr/fltr-mcp",
      "install": "pip install fltr-mcp",
      "categories": ["data", "research", "finance", "legal"],
      "author": "FLTR",
      "stars": 0
    }
    ```
  - Submit PR

- [ ] **PulseMCP Directory**
  - Visit: https://pulsemcp.com
  - Click "Submit Server"
  - Fill in details:
    - Name: "FLTR Datasets"
    - Description: "The first curated dataset marketplace accessible via MCP. Query financial data, legal documents, market research, and more from Claude."
    - Install command: `pip install fltr-mcp`
    - GitHub: https://github.com/fltr/fltr-mcp
    - Categories: Data, Research, Finance
  - Submit

- [ ] **Smithery.ai** (MCP directory)
  - Visit: https://smithery.ai
  - Submit server listing

#### Creator Marketing (Supply Side)

- [ ] **Create Creator-Focused Landing Page**
  - [ ] "Monetize Your Dataset" page live
  - [ ] Revenue calculator widget
  - [ ] Creator testimonials (from early partners)
  - [ ] CTA: "Apply as Creator"

- [ ] **Announce Creator Program**
  - [ ] Post on r/datasets - "Monetize your datasets via AI"
  - [ ] Post on Indie Hackers - "New passive income opportunity"
  - [ ] Share in data science communities (Kaggle forums, etc.)

### Day 6-7: Create Launch Content (Two-Sided)

- [ ] **Demo Video** (3 minutes - two-sided)
  - **Consumer Side (0:00-1:30):**
    - What is FLTR MCP?
    - Installation: pip install + config
    - Demo: Query SEC filings from Claude
    - Show multi-dataset analysis
  - **Creator Side (1:30-2:30):**
    - Show creator dashboard
    - Display revenue from queries
    - Highlight: "Earn 70% per query"
  - **Call to Action (2:30-3:00):**
    - Consumers: fltr.com/install
    - Creators: fltr.com/creators

- [ ] **Launch Blog Post** (2,000 words - two-sided)
  ```markdown
  # Introducing FLTR MCP: The AI-Native Data Marketplace

  ## What is FLTR MCP?
  - Two-sided marketplace
  - Creators monetize, consumers query
  - All via Claude Desktop

  ## For Creators: Monetize Your Data
  - Upload dataset, earn 70% per query
  - No infrastructure costs
  - AI-native distribution

  ## For Consumers: Query via Natural Language
  - 100+ curated datasets
  - No downloads, just queries
  - SEC filings, legal docs, research, and more

  ## Getting Started
  - Creators: Upload your first dataset
  - Consumers: 2-minute installation

  ## Example Use Cases
  - Financial Analysis (consumer)
  - Creator earnings story (creator)
  - Legal Research (consumer)

  ## What's Next
  - Open creator platform in Month 5
  - 200 datasets by Month 6
  - $25k GMV goal
  ```

- [ ] **GitHub README**
  - Add hero GIF (Claude querying dataset)
  - Quick start section
  - 3-5 code examples
  - Link to full docs

- [ ] **Social Media Assets**
  - Twitter/X thread (6-8 tweets)
  - LinkedIn post
  - Reddit post template
  - Screenshots for sharing

---

## 🚀 Week 2: Launch & Promotion (Days 8-14)

### Day 8: Soft Launch

- [ ] Email existing FLTR users:
  ```
  Subject: Query FLTR Datasets Directly in Claude Desktop

  We just launched MCP support - now you can search your datasets
  without leaving Claude. Takes 2 minutes to set up.

  [Watch 2-min demo] [Install now]
  ```

- [ ] Tweet announcement thread
- [ ] Post in FLTR Discord/Slack

### Day 9: Reddit Launch

- [ ] Post to r/ClaudeAI:
  ```
  Title: [Tool] Query curated datasets from Claude Desktop via MCP

  Hey r/ClaudeAI! I built an MCP server that lets you query 100+
  curated datasets directly in Claude Desktop.

  Use cases: SEC filings analysis, legal research, market data

  [Demo video] [GitHub repo] [5-minute setup guide]
  ```

- [ ] Post to r/LocalLLaMA
- [ ] Post to r/ArtificialIntelligence
- [ ] Post to r/datascience

### Day 10: Hacker News

- [ ] Post Show HN (Saturday 8am PT is best):
  ```
  Title: Show HN: Query curated datasets from Claude Desktop

  Link: https://github.com/fltr/fltr-mcp

  Text:
  I built an MCP server that brings 100+ curated datasets into Claude.

  Examples: Search SEC filings, case law, market research - all via
  natural language in Claude Desktop.

  Technical: Uses Model Context Protocol (stdio), vector search (Milvus),
  semantic embeddings (OpenAI). Open source.

  Would love feedback on the approach!
  ```

### Day 11-12: Community Engagement

- [ ] MCP Discord:
  - #announcements channel
  - #show-and-tell channel
  - Answer questions in #help

- [ ] AI Tool communities:
  - Cursor Discord
  - Continue.dev Discord
  - Windsurf community

- [ ] Developer communities:
  - Dev.to article
  - Hashnode blog
  - Medium cross-post

### Day 13-14: Respond & Iterate

- [ ] Monitor GitHub issues
- [ ] Respond to Reddit comments
- [ ] Update docs based on questions
- [ ] Track installation metrics
- [ ] Gather testimonials

---

## 📊 Success Metrics (Week 1 Goals)

- [ ] **100+ PyPI downloads**
- [ ] **50+ GitHub stars**
- [ ] **Listed in 2+ MCP directories**
- [ ] **10+ user testimonials**
- [ ] **5+ datasets queried via MCP**
- [ ] **1+ partnership conversation started**

---

## 🎯 Month 2-3: Growth Phase

### Content Marketing

- [ ] Weekly blog post:
  - Week 1: "5 Ways Financial Analysts Use FLTR + Claude"
  - Week 2: "Building a Research Agent with MCP"
  - Week 3: "The Complete Guide to MCP Datasets"
  - Week 4: "Case Study: How [User] Analyzes 100 Companies/Day"

- [ ] Video tutorials:
  - Financial analysis walkthrough
  - Research workflow demo
  - Building custom agents

### Partnerships

- [ ] Reach out to Anthropic Developer Relations
- [ ] Contact Cursor team about featured integration
- [ ] Partner with 3 dataset creators for co-marketing

### Product Improvements

- [ ] Add .mcpb one-click install
- [ ] Create dataset starter packs
- [ ] Build example agents (financial, legal, research)

---

## 💰 Monetization Checkpoints

### Week 2
- [ ] Track credit usage from MCP queries
- [ ] Calculate revenue per MCP user
- [ ] Survey users: "What would you pay for unlimited MCP?"

### Month 2
- [ ] Launch "MCP Pro" tier ($19/mo unlimited queries)
- [ ] Add team/enterprise pricing
- [ ] Implement affiliate program (10% for referrals)

### Month 3
- [ ] Revenue target: $500 MRR from MCP users
- [ ] 20+ datasets with active MCP usage
- [ ] 5+ enterprise pilot customers

---

## 🎬 Quick Wins (Do These First)

1. **Today:**
   - [ ] Create GitHub repo `fltr-mcp`
   - [ ] Copy `mcp_server.py` to new package structure
   - [ ] Write basic README

2. **Tomorrow:**
   - [ ] Add `pyproject.toml` and build config
   - [ ] Test local installation
   - [ ] Record 2-minute demo video

3. **Day 3:**
   - [ ] Publish to PyPI
   - [ ] Tweet announcement
   - [ ] Email existing users

4. **Weekend:**
   - [ ] Submit to MCP directories
   - [ ] Post to Reddit/HN
   - [ ] Monitor feedback

---

## 📞 Support Resources

### Documentation
- Installation guide: https://docs.fltr.com/mcp
- API reference: https://docs.fltr.com/api
- Examples: https://github.com/fltr/fltr-mcp/examples

### Community
- Discord: https://discord.gg/fltr
- GitHub Issues: Report bugs
- Email: support@fltr.com

### Monitoring
- PyPI stats: https://pypistats.org/packages/fltr-mcp
- GitHub insights: Watch stars, forks, issues
- Usage dashboard: Track MCP queries in Datadog

---

## ✅ Definition of Done

**Week 1 Complete:**
- ✅ Package published to PyPI
- ✅ Listed in 2+ MCP directories
- ✅ 100+ installations
- ✅ Demo video published
- ✅ Launch blog post live

**Month 1 Complete:**
- ✅ 500+ installations
- ✅ 5+ use case examples published
- ✅ Featured in MCP newsletter
- ✅ 1+ partnership secured

**Month 3 Complete:**
- ✅ 1,000+ installations
- ✅ $500+ MRR from MCP usage
- ✅ Top 20 MCP server by installs
- ✅ Case study with notable customer

---

**Start Date:** [Today's date]

**Owner:** [Team member]

**Status:** Ready to launch 🚀
