# FLTR Quick Start Guide

Get FLTR up and running locally in 10 minutes.

**Last Updated**: October 31, 2024 (V2.0 - Modal Architecture)

## Prerequisites

- Python 3.10+
- Node.js 18+
- PostgreSQL (local or Docker)
- Cloudflare account (for R2 storage)
- Modal account (for document processing)

## Local Development Setup

### 1. Clone and Install

```bash
cd /path/to/FLTR

# Backend
cd fastapi
python -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
pip install -r requirements.txt

# Frontend
cd ../nextjs
pnpm install
```

### 2. Setup PostgreSQL

**Option A: Docker (Recommended)**
```bash
docker run -d \
  --name fltr-postgres \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=fltr \
  postgres:15

# Create database (if using existing Postgres)
psql -U postgres -c "CREATE DATABASE fltr;"
```

**Option B: Local PostgreSQL**
```bash
# macOS with Homebrew
brew install postgresql@15
brew services start postgresql@15
createdb fltr

# Ubuntu/Debian
sudo apt install postgresql-15
sudo systemctl start postgresql
sudo -u postgres createdb fltr
```

### 3. Setup Redis (for sessions)

```bash
# Docker (Recommended)
docker run -d --name fltr-redis -p 6379:6379 redis:7

# OR via Homebrew (macOS)
brew install redis
brew services start redis

# OR via apt (Ubuntu)
sudo apt install redis-server
sudo systemctl start redis
```

### 4. Configure Environment

#### FastAPI Backend

Create `fastapi/.env`:
```bash
# Database
DATABASE_URL=postgresql://postgres:password@localhost:5432/fltr

# Milvus (Zilliz Cloud - sign up at zilliz.com)
MILVUS_HOST=your-instance.aws-us-west-2.vectordb.zillizcloud.com
MILVUS_PORT=19530
MILVUS_TOKEN=your_token_here
MILVUS_DB_NAME=default

# Cloudflare R2 (sign up at cloudflare.com)
R2_ACCOUNT_ID=your_account_id
R2_ACCESS_KEY_ID=your_access_key
R2_SECRET_ACCESS_KEY=your_secret_key
R2_BUCKET_NAME=fltr-datasets
R2_ENDPOINT_URL=https://your_account_id.r2.cloudflarestorage.com

# OpenAI (get key at platform.openai.com)
OPENAI_API_KEY=sk-your-key-here

# Modal (for webhook callbacks)
MODAL_WEBHOOK_URL=https://your-app--webhook-process-document.modal.run

# Redis
REDIS_URL=redis://localhost:6379/0

# Better Auth
BETTER_AUTH_SECRET=your-secret-here-generate-with-openssl-rand-base64-32
BETTER_AUTH_URL=http://localhost:3000
```

#### Next.js Frontend

Create `nextjs/.env.local`:
```bash
# API
NEXT_PUBLIC_API_URL=http://localhost:8000

# Better Auth
BETTER_AUTH_SECRET=same-as-fastapi-secret
BETTER_AUTH_URL=http://localhost:3000
DATABASE_URL=postgresql://postgres:password@localhost:5432/fltr

# OpenAI (for chat)
OPENAI_API_KEY=sk-your-key-here
```

### 5. Setup Modal (Document Processing)

Modal handles all heavy document processing (PDF parsing, embedding generation).

```bash
# Install Modal CLI
pip install modal

# Authenticate (opens browser)
modal token new

# Create Modal secrets
cd modal

# Create a combined secret with all credentials
modal secret create fltr-secrets \
  DATABASE_URL=postgresql://... \
  MILVUS_HOST=your-instance.vectordb.zillizcloud.com \
  MILVUS_PORT=19530 \
  MILVUS_TOKEN=your_token \
  CLOUDFLARE_R2_ACCESS_KEY_ID=your_key \
  CLOUDFLARE_R2_SECRET_ACCESS_KEY=your_secret \
  CLOUDFLARE_R2_BUCKET_NAME=fltr-datasets \
  CLOUDFLARE_R2_ENDPOINT_URL=https://....r2.cloudflarestorage.com \
  CLOUDFLARE_R2_ACCOUNT_ID=your_account \
  OPENAI_API_KEY=sk-your-key

# Deploy Modal app
modal deploy modal_app.py

# You'll get a webhook URL like:
# https://your-app--webhook-process-document.modal.run
#
# Copy this URL to fastapi/.env as MODAL_WEBHOOK_URL
```

### 6. Initialize Database

```bash
cd fastapi

# Run migrations (if any)
# alembic upgrade head

# Seed test data (optional)
chmod +x seed_datasets.sh
./seed_datasets.sh
```

### 7. Start Services

**Terminal 1: FastAPI**
```bash
cd fastapi
source .venv/bin/activate
uvicorn main:app --reload --port 8000
```

**Terminal 2: Next.js**
```bash
cd nextjs
pnpm dev
```

### 8. Verify Setup

Open your browser:

- **Frontend**: http://localhost:3000
- **API Docs**: http://localhost:8000/docs
- **Health Check**: http://localhost:8000/health

## Test Document Upload

1. Go to http://localhost:3000
2. Create an account
3. Create a new dataset
4. Upload a document (PDF, TXT, JSON, CSV)
5. Watch processing happen on Modal

**Check Modal logs:**
```bash
modal logs process_document_modal --follow
```

## Architecture Overview

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│  Next.js    │─────▶│  FastAPI    │─────▶│ Cloudflare  │
│  Frontend   │      │  Backend    │      │     R2      │
└─────────────┘      └─────────────┘      └─────────────┘
                            │                      │
                            │                      │
                            ▼                      ▼
                     ┌─────────────┐      ┌─────────────┐
                     │ PostgreSQL  │      │  Modal.com  │
                     │  Metadata   │      │  Processing │
                     └─────────────┘      └─────────────┘
                                                  │
                                                  ▼
                                          ┌─────────────┐
                                          │   Milvus    │
                                          │   Vectors   │
                                          └─────────────┘
```

**Key Points:**
- **No Celery workers** - Modal handles all processing
- **Direct R2 uploads** - Client uploads files via presigned URLs
- **Automatic scaling** - Modal scales based on load
- **Pay-per-second** - Only pay for actual processing time

## Common Issues

### "Connection refused" errors

**PostgreSQL not running:**
```bash
# Check status
docker ps | grep postgres
# Or for local install
pg_isready

# Restart if needed
docker start fltr-postgres
```

**Redis not running:**
```bash
# Check status
docker ps | grep redis
# Or
redis-cli ping  # Should return "PONG"

# Restart if needed
docker start fltr-redis
```

### "Milvus connection failed"

- Verify `MILVUS_HOST` and `MILVUS_TOKEN` in `.env`
- Check Zilliz Cloud dashboard for instance status
- Ensure firewall allows connections

### "R2 access denied"

- Verify R2 credentials in `.env`
- Check R2 bucket exists and has correct permissions
- Ensure `R2_ENDPOINT_URL` is correct

### Modal deployment fails

```bash
# Check authentication
modal token list

# Re-authenticate if needed
modal token new

# Verify secrets exist
modal secret list

# Check Modal logs
modal logs process_document_modal
```

## Development Workflow

### Running Tests

**Backend:**
```bash
cd fastapi
pytest tests/ -v
```

**Frontend:**
```bash
cd nextjs
pnpm test
```

### Database Migrations

```bash
cd fastapi

# Create migration
alembic revision -m "description"

# Apply migrations
alembic upgrade head

# Rollback
alembic downgrade -1
```

### Regenerate API Client

When you update FastAPI endpoints:

```bash
cd nextjs
pnpm run generate:api
```

This uses Orval to generate TypeScript types and React Query hooks from OpenAPI.

## Production Deployment

See `ARCHITECTURE.md` for detailed production deployment guide.

**Quick overview:**

1. **FastAPI** → DigitalOcean droplet ($24/mo recommended)
2. **Next.js** → Vercel (automatic with git push)
3. **Modal** → Already deployed (`modal deploy`)
4. **PostgreSQL** → Managed database (DigitalOcean, Supabase, Railway)
5. **Milvus** → Zilliz Cloud (managed service)
6. **R2** → Cloudflare (already set up)

## Next Steps

Once everything is running:

1. ✅ **Read** `START_HERE.md` for quick testing guide
2. ✅ **Review** `ARCHITECTURE.md` for system architecture
3. ✅ **Test** MCP chat integration (`MCP_QUICK_TEST.md`)
4. ✅ **Check** `CLOUD_COSTS.md` for cost optimization

## Resources

- **API Documentation**: http://localhost:8000/docs
- **Modal Dashboard**: https://modal.com/apps
- **Milvus Docs**: https://milvus.io/docs
- **Cloudflare R2 Docs**: https://developers.cloudflare.com/r2/

---

**Need Help?**

1. Check `ARCHITECTURE.md` for system design
2. Review `docs/archive/` for migration guides
3. Check Modal logs: `modal logs process_document_modal --follow`
4. Verify environment variables in `.env` files
5. Check service status: PostgreSQL, Redis, Modal

**Version**: 2.0 (Modal Architecture)
**Last Updated**: October 31, 2024
