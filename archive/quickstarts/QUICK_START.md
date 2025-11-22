# FLTR Quick Start Guide

## 🚀 Start Developing in 5 Minutes

## ⚠️ IMPORTANT: Database Setup First!

**Modal runs in the cloud and needs network-accessible databases.**

### ❌ Will NOT Work:
- `localhost:5432` PostgreSQL
- `localhost:19530` Milvus
- Any service on 127.0.0.1

### ✅ What Works:
- Cloud databases (Supabase, Neon, Railway) ← **Recommended!**
- Cloudflare Tunnel or ngrok for local databases
- See [LOCAL_DEVELOPMENT.md](LOCAL_DEVELOPMENT.md) for setup options

---

### Prerequisites

**Choose ONE of these before proceeding:**

**Option A: Cloud Databases (Easiest ⭐)**
- PostgreSQL: Supabase/Neon/Railway (all have free tiers)
- Milvus: Zilliz Cloud (free tier available)
- No tunnel needed!

**Option B: Local Databases with Tunnel**
- Set up Cloudflare Tunnel (see LOCAL_DEVELOPMENT.md)
- More complex but keeps data local

**Also need:**
- Modal account (free tier is fine)
- OpenAI API key
- R2/S3 storage configured

---

### 1. Setup Modal (One-Time)

```bash
pip install modal
modal token new
# Opens browser to authenticate with Modal
```

### 2. Start Modal Dev Server

```bash
cd modal
modal serve modal_app.py
```

**Copy the webhook URL** from the output (looks like `https://your-username--fltr-webhook-fastapi-app-dev.modal.run/process`)

### 3. Configure Environment

Create `fastapi/.env`:
```bash
# Modal Configuration
MODAL_WEBHOOK_URL=<webhook-url-from-step-2>
USE_MODAL_FOR_PROCESSING=true
MODAL_ENABLED=true

# Databases (MUST be accessible from cloud!)
# Option A: Cloud databases (recommended)
DATABASE_URL=postgresql://user:pass@db.xxx.supabase.co:5432/postgres
AUTH_DATABASE_URL=postgresql://user:pass@db.xxx.supabase.co:5432/postgres

# Option B: Local with tunnel
# DATABASE_URL=postgresql://user:pass@fltr-postgres.yourdomain.com:5432/fltr
# AUTH_DATABASE_URL=postgresql://user:pass@fltr-postgres.yourdomain.com:5432/fltr_auth

# Vector Database (MUST be accessible from cloud!)
# Option A: Zilliz Cloud (recommended)
MILVUS_URI=https://in03-xxx.api.gcp-us-west1.zillizcloud.com
MILVUS_TOKEN=your-zilliz-token

# Option B: Local with tunnel
# MILVUS_URI=https://fltr-milvus.yourdomain.com:19530

# AI
OPENAI_API_KEY=sk-...

# Storage (R2/Cloudflare)
R2_ACCOUNT_ID=your_account_id
R2_ACCESS_KEY_ID=your_access_key
R2_SECRET_ACCESS_KEY=your_secret_key
R2_BUCKET_NAME=fltr-datasets

# Optional: Cloudflare Queue (for production)
CF_QUEUE_ID=
CF_API_TOKEN=

# Auth
BETTER_AUTH_SECRET=your-secret-key-here
BETTER_AUTH_URL=http://localhost:3000
```

### 4. Start Backend (FastAPI)

```bash
cd fastapi
./start-local-dev.sh
# API available at http://localhost:8000
```

### 5. Start Frontend (Next.js)

```bash
cd nextjs
pnpm install
pnpm dev
# App available at http://localhost:3000
```

---

## ✅ Verify It's Working

### Test Document Processing

```bash
# 1. Create a dataset via the UI at http://localhost:3000

# 2. Upload a document (PDF, TXT, etc.)

# 3. Watch the logs:
# - FastAPI logs: Should see "🚀 [Routing] Using Modal"
# - Modal logs: Run `modal app logs fltr::fltr --follow`

# 4. Check document status in UI - should go:
#    uploaded → processing → ready
```

### Check Services

```bash
# FastAPI health
curl http://localhost:8000/health

# Modal webhook
curl -X POST <your-modal-webhook-url>/process \
  -H "Content-Type: application/json" \
  -d '{"dataset_id":"test","object_key":"test.pdf","task_id":"123"}'
# Should return: {"status": "queued", "call_id": "..."}

# Milvus
# Should show collections if any exist
curl http://localhost:9091/api/v1/collections

# PostgreSQL
psql -U user -d fltr -c "SELECT version();"
```

---

## 📁 Project Structure

```
FLTR/
├── fastapi/          # FastAPI backend (API, webhooks, auth)
├── nextjs/           # Next.js frontend (UI, auth UI)
├── modal/            # Modal app (document processing)
├── cloudflare-workers/  # Queue consumers (production only)
└── docs/             # Documentation

Key files:
├── LOCAL_DEVELOPMENT.md     # Detailed local setup
├── DEPLOYMENT.md            # Production deployment
├── DATABASE_ARCHITECTURE.md # Database design
└── QUICK_START.md          # This file
```

---

## 📖 Next Steps

- **Local Development**: See [LOCAL_DEVELOPMENT.md](LOCAL_DEVELOPMENT.md) for detailed guide
- **Architecture**: See [DATABASE_ARCHITECTURE.md](DATABASE_ARCHITECTURE.md)
- **Deployment**: See [fastapi/DEPLOYMENT.md](fastapi/DEPLOYMENT.md)
- **API Docs**: Visit http://localhost:8000/docs when FastAPI is running

---

## 🐛 Common Issues

### "Modal webhook not accessible"
```bash
# Check Modal is still running
modal app logs fltr::fltr

# Restart Modal dev server
cd modal && modal serve modal_app.py
```

### "Database connection failed" / "Document stuck in 'uploaded'"

**Most common issue!** Modal can't reach your database.

**Symptoms:**
- Documents stuck in "uploaded" status
- Modal logs show connection timeout/errors
- Processing never starts

**Quick Fix:**
```bash
# Test database from Modal's perspective
modal container list  # Get container ID
modal shell <container-id>

# Inside container:
root ~ → psql $DATABASE_URL
# If this fails, Modal can't reach your database!
```

**Solutions:**

1. **Using localhost?** ❌ Switch to:
   - Supabase (easiest - free tier)
   - OR set up Cloudflare Tunnel (see LOCAL_DEVELOPMENT.md)

2. **Using cloud database?** ✅ Verify:
   - DATABASE_URL is correct
   - Firewall allows Modal access
   - SSL settings (`?sslmode=require` if needed)

3. **Using tunnel?** Check:
   - Tunnel is running: `cloudflared tunnel info fltr-postgres`
   - DNS resolves: `nslookup fltr-postgres.yourdomain.com`

**See [LOCAL_DEVELOPMENT.md](LOCAL_DEVELOPMENT.md) for detailed database setup.**

### "Milvus connection failed"
Same issue as database - Modal can't reach Milvus.

**Solutions:**
- Use Zilliz Cloud (recommended - free tier)
- OR expose local Milvus via Cloudflare Tunnel
- OR deploy Milvus to cloud VM

---

## 💡 Development Tips

### Hot Reload
- **FastAPI**: Auto-reloads on file changes
- **Next.js**: Auto-reloads on file changes
- **Modal**: Use `modal serve` for hot reload (dev mode)

### Running Tests
```bash
# Backend tests
cd fastapi && pytest

# Frontend tests
cd nextjs && pnpm test
```

### Database Migrations
```bash
# Generate migration after schema changes
cd nextjs && pnpm db:generate

# Apply migrations
pnpm db:migrate

# Push schema directly (dev only)
pnpm db:push
```

### Modal Development
```bash
# Dev mode (hot reload)
modal serve modal_app.py

# Deploy to cloud
modal deploy modal_app.py

# View logs
modal app logs fltr::fltr --follow

# Check usage/costs
modal app stats fltr::fltr
```

---

## 🎯 Recommended Workflow

**Daily development:**

```bash
# Terminal 1: Modal
cd modal && modal serve modal_app.py

# Terminal 2: FastAPI
cd fastapi && ./start-local-dev.sh

# Terminal 3: Next.js
cd nextjs && pnpm dev

# Terminal 4: Watch logs
modal app logs fltr::fltr --follow
```

**Before committing:**
```bash
# Run tests
cd fastapi && pytest
cd nextjs && pnpm test

# Check types
cd nextjs && pnpm check-types

# Format code
cd fastapi && black .
cd nextjs && pnpm format
```

---

## 📚 Documentation Map

- `LOCAL_DEVELOPMENT.md` - Complete local setup guide (you are here)
- `fastapi/DEPLOYMENT.md` - Production deployment guide
- `DATABASE_ARCHITECTURE.md` - Database design and architecture
- `DATABASES.md` - Quick database reference
- `MODAL_DEPLOYMENT_STRATEGY.md` - Modal deployment options
- `CELERY_REMOVAL_PLAN.md` - Document processing cleanup history

---

## 💬 Need Help?

1. Check the detailed guides above
2. Search GitHub issues
3. Check Modal dashboard for errors: https://modal.com/apps
4. Review logs: `modal app logs fltr::fltr`

Happy coding! 🎉
