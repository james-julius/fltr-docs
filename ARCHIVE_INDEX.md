# Documentation Archive Index

This directory contains historical documentation, migration guides, and archived implementation notes. Most of this documentation represents past decisions, completed work, or deprecated features.

## 📁 Archive Organization

### `/archive/assessments/`
Code audits, readiness assessments, and analysis reports:
- Code quality assessments
- Dataset readiness evaluations
- Large-scale ingestion assessments
- Documentation audits

### `/archive/architecture/`
Architecture decisions and system design documents:
- Architecture deep dives and analysis
- Dual ORM architecture notes
- Schema verification and ownership
- Observer pattern implementations

### `/archive/auth/`
Authentication implementation notes:
- Auth implementation remaining work
- Rate limiting plans
- Better Auth billing portal redirects

### `/archive/billing/`
Payment and subscription system documentation:
- Billing system status and implementation
- Credit system implementation
- Stripe billing portal configuration
- Subscription management and renewal flows
- Payment flow test coverage

### `/archive/costs/`
Cost analysis and infrastructure decisions:
- Cloud cost analysis
- OpenRouter vs Vercel AI Gateway comparisons

### `/archive/datasets/`
Dataset processing and upload guides:
- Bulk upload setup and production guides
- Document processing pipeline investigations
- Document extraction and asset implementation
- Epstein dataset processing notes
- Upload decision guides

### `/archive/debugging/`
Debugging notes and issue investigations:
- Debugging Modal issues
- Subscription upgrade debugging

### `/archive/deployment/`
Deployment strategies and production notes:
- Celery removal plans (not executed - Celery kept)
- Modal deployment strategies
- Modal migration notes
- Production migration guides
- Production nuke and rebuild procedures

### `/archive/features/`
Feature implementation guides:
- Content filtering
- Feedback system
- OCR image retrieval
- RAG features integration
- Reranking guides
- General implementation summaries

### `/archive/fixes/`
Bug fixes and issue resolutions:
- Checkout redirect fixes
- Coolify SSL fixes
- Dataset status fixes
- Document upload URL encoding fixes
- Modal retry fixes
- Net usage fixes
- R2 webhook fixes and analysis
- Security fixes
- Vercel build fixes
- Webhook fixes (multiple)
- Production fix steps

### `/archive/issues/`
Specific issue tracking:
- Milvus issues
- Page number migration guides
- Bug reports

### `/archive/mcp/`
MCP server setup and distribution:
- MCP business model decisions
- MCP deployment plans
- MCP distribution strategies
- MCP implementation summaries
- MCP launch checklists
- MCP marketplace models
- MCP positioning and readiness
- MCP test verification

### `/archive/migration/`
Migration guides and strategies:
- Clean slate migration
- Deploy and run migration
- Migration options and strategies
- Migration workflows
- Modal migration
- Page number migration
- Production migration

### `/archive/quickstarts/`
Quick start and setup guides (may be outdated):
- Installation guides
- MCP quick tests
- Quick start guides
- Start here guides
- Ready to use guides

### `/archive/reference/`
Reference documentation:
- Database architecture
- Database schema documentation

### `/archive/roadmap/`
Product roadmaps and planning:
- Launch requirements
- Ragie parity roadmap
- User-first roadmap
- Scalability assessments

### `/archive/security/`
Security implementations and audits:
- API documentation security
- Security plans and test results

### `/archive/setup/`
Setup and configuration guides:
- Admin dashboard setup
- ConvertKit setup
- Local development setup
- Tunnel setup

### `/archive/testing/`
Test coverage and strategies:
- Credit system test reports
- Payment flow test coverage
- Test coverage gap analysis
- Thread management testing

## 🔍 Finding Documentation

### For Current Information
- See the main [README.md](../README.md) in the project root
- Check service-specific READMEs:
  - [FastAPI README](../fastapi/README.md)
  - [NextJS README](../nextjs/README.md)
  - [Modal README](../modal/README.md)

### For Historical Context
Browse the relevant archive category above, or search by keyword:

```bash
# Search all archived docs
cd docs/archive
grep -r "keyword" .

# Find files by name pattern
find docs/archive -name "*PATTERN*"
```

## ⚠️ Important Notes

1. **Archived documentation may be outdated** - Always verify information against current code
2. **Some strategies were not implemented** - e.g., CELERY_REMOVAL_PLAN.md was not executed
3. **Migration docs are historical** - Useful for understanding past decisions
4. **Security docs should be reviewed** - May contain outdated security information

## 📝 Documentation Standards

For current documentation practices, see:
- [CONTRIBUTING.md](../CONTRIBUTING.md) - Contribution guidelines
- [TODO_ISSUES.md](../TODO_ISSUES.md) - Current action items

## 🗑️ Deprecated

The `/deprecated/` directory contains documentation for features that have been fully removed or replaced. This is separate from `/archive/` which contains historical but potentially still-relevant documentation.
