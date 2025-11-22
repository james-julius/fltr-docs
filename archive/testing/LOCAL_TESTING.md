# Local Testing Guide

This guide explains how to test the FLTR document processing flow locally with Modal.

## Quick Start

```bash
./start-local-dev.sh
```

The script automatically:
- ✅ Uses your named tunnel if `.env.tunnel` exists (recommended)
- ✅ Falls back to quick tunnel if no `.env.tunnel` found

This sets up:
- ✅ Local R2 storage (files stay on your machine)
- ✅ Cloudflare Tunnel (exposes local storage to Modal)
- ✅ Modal in dev mode (hot reload for code changes)
- ✅ FastAPI configured for Modal processing
- ✅ Full production-like document processing

## Architecture

### Processing Flow
```
Upload → Local R2 Proxy → Tunnel → FastAPI → Modal Cloud → Milvus
```

**What happens:**
1. Files upload to your local storage (`.wrangler-state/`)
2. Cloudflare Tunnel exposes local storage publicly
3. Modal downloads files from tunnel URL
4. Modal processes with Docling (PDF parsing, chunking, embeddings)
5. Results stored in Milvus vector database
6. Document status updated in PostgreSQL

**Why this works:**
- Files never leave your machine (stored locally)
- Modal runs in cloud but accesses your local files via tunnel
- Test production code path without uploading to cloud storage

## Services Started

1. **R2 Upload Proxy** (localhost:8787) - Local file storage
2. **Cloudflare Tunnel** - Exposes local R2 to internet
3. **Modal** (dev mode) - Serverless document processing
4. **FastAPI** (localhost:8000) - API server

## Prerequisites

Install these tools before running the script:

```bash
# Redis (for sessions/caching)
brew install redis
redis-server

# Wrangler (for local R2 proxy)
npm install -g wrangler

# Cloudflared (for tunnel)
brew install cloudflare/cloudflare/cloudflared

# Modal (for document processing)
pip install modal
modal token new
```

### Optional: Named Tunnel Setup

For a permanent tunnel URL that never changes, see [TUNNEL_SETUP.md](TUNNEL_SETUP.md).

Quick tunnels work fine but generate random URLs each run. Named tunnels give you a stable URL like `https://local-webhooks.tryfltr.com`.

## After Starting Services

1. **Start Next.js** (in another terminal):
   ```bash
   cd nextjs && npm run dev
   ```

2. **Open the app**:
   http://localhost:3000/my-datasets/create

3. **Upload files** and watch the logs!

## Monitoring Logs

The script automatically tails Modal logs, but you can monitor all logs:

```bash
tail -f logs/modal.log       # Document processing (with emoji indicators!)
tail -f logs/fastapi.log     # API requests
tail -f logs/r2-proxy.log    # File uploads
tail -f logs/tunnel.log      # Tunnel status
```

### Modal Processing Indicators

Watch for these emojis in `logs/modal.log`:
- 🚀 Starting processing
- 📥 Downloading from R2
- 📄 Parsing document with Docling
- ✂️  Chunking into segments
- 🧮 Generating embeddings (OpenAI)
- 💾 Storing in Milvus
- ✅ Success / ❌ Error

## Troubleshooting

### "Failed to get tunnel URL"
- Check `logs/tunnel.log` for errors
- Ensure port 8787 is not in use
- Try restarting the script

### "Modal webhook URL not found"
- Check `logs/modal.log` for errors
- Ensure you ran `modal token new`
- Verify Modal secrets are configured

### "NoSuchKey" errors in Modal
- This was the original issue - now fixed!
- Modal now fetches from your local R2 via tunnel
- Files are stored locally but accessible to Modal cloud

## File Storage Location

All uploaded files are stored locally:
```
cloudflare-workers/r2-upload-proxy/.wrangler-state/
```

Even though Modal processes files in the cloud, the actual file storage remains on your machine.

## Stopping Services

Press `Ctrl+C` in the terminal running `start-local-dev.sh`

All services (R2 proxy, tunnel, Modal, FastAPI) will be stopped automatically.

## Why This Setup?

This gives you the best of both worlds:

✅ **Local file storage** - No cloud storage costs, full control
✅ **Production processing** - Real Docling, actual Modal code path
✅ **Fast iteration** - Modal hot-reloads on code changes
✅ **Privacy** - Files never uploaded to cloud storage
✅ **Debugging** - See exact errors Modal would encounter in production
