# Local Development Setup

## Overview

For local development, document processing runs via Modal. There are two approaches:

1. **Modal Dev Mode** (Recommended): Run Modal webhook locally, processing happens in Modal cloud
2. **Full Modal Deploy**: Deploy to Modal cloud, use production setup locally

---

## ⚠️ Critical: Database Accessibility

**IMPORTANT:** Modal runs in the cloud and needs network access to your databases.

### ✅ What Works
- Cloud-hosted PostgreSQL (Supabase, Neon, Railway, DigitalOcean, etc.)
- Cloud-hosted Milvus (Zilliz Cloud, self-hosted on cloud VM)
- Cloudflare R2 / AWS S3 (always cloud-based)
- Databases exposed via Cloudflare Tunnel or ngrok

### ❌ What Does NOT Work
- `localhost:5432` PostgreSQL ❌
- `localhost:19530` Milvus ❌
- Any service bound to `127.0.0.1` ❌
- Private networks without tunnels ❌

### Why?
Modal processes documents in cloud containers that cannot reach `localhost`. The architecture uses a **shared database pattern**:

```
FastAPI (localhost) ──→ PostgreSQL ←── Modal (cloud)
                          ↑
                    Both services need access!
```

Modal doesn't callback to FastAPI - instead, both services read/write the same database directly.

### Choose Your Database Approach

**See "Database Options" section below** before proceeding. You must set up accessible databases first.

---

## Database Options

Choose one of these approaches before setting up Modal:

### Option 1: Cloud-Hosted Database (Recommended ⭐)

**Best for:** Everyone, especially if new to Modal or want simplest setup

**Pros:**
- ✅ No tunnel setup needed
- ✅ Works out of the box
- ✅ Production-like environment
- ✅ Free tiers available

**Cons:**
- ⚠️ Requires internet connection
- ⚠️ Data stored in cloud (use encryption)

**Recommended Services:**

**A. Supabase (Easiest)**
```bash
# 1. Sign up at https://supabase.com (free tier: 500MB)
# 2. Create new project
# 3. Copy connection string from Settings → Database
# 4. Use in .env:
DATABASE_URL=postgresql://postgres:[password]@db.[project-ref].supabase.co:5432/postgres
AUTH_DATABASE_URL=postgresql://postgres:[password]@db.[project-ref].supabase.co:5432/postgres
```

**B. Neon (Serverless Postgres)**
```bash
# 1. Sign up at https://neon.tech (free tier: 0.5GB)
# 2. Create database
# 3. Copy connection string
# 4. Use in .env (replace user/pass/host):
DATABASE_URL=postgresql://user:pass@ep-xxx-xxx.us-east-2.aws.neon.tech/fltr?sslmode=require
```

**C. Railway**
```bash
# 1. Sign up at https://railway.app
# 2. Create PostgreSQL service
# 3. Copy DATABASE_URL from variables
```

For Milvus, use **Zilliz Cloud** (https://zilliz.com) - free tier available.

---

### Option 2: Cloudflare Tunnel (For Local PostgreSQL)

**Best for:** Developers who want fully local databases

**Pros:**
- ✅ Keep data local
- ✅ No cloud database costs
- ✅ Full control

**Cons:**
- ⚠️ More complex setup
- ⚠️ Must keep tunnel running
- ⚠️ Tunnel URL changes on restart

**Setup Guide:**

See dedicated "Cloudflare Tunnel Setup" section below for complete instructions.

---

### Option 3: ngrok (Alternative Tunnel)

**Best for:** Quick temporary setup or testing

**Pros:**
- ✅ Easiest tunnel setup
- ✅ Good for testing

**Cons:**
- ⚠️ URL changes on each restart (free tier)
- ⚠️ Limited hours/month (free tier)

**Quick Setup:**
```bash
# 1. Install ngrok
brew install ngrok  # macOS
# Or download from https://ngrok.com

# 2. Expose PostgreSQL
ngrok tcp 5432

# 3. Copy the forwarding URL (e.g., tcp://0.tcp.ngrok.io:12345)
# 4. Use in .env:
DATABASE_URL=postgresql://user:pass@0.tcp.ngrok.io:12345/fltr
```

**Note:** Free tier URLs change on each restart - you'll need to update Modal secrets frequently.

---

## Prerequisites

**After choosing your database approach:**

```bash
# Install Modal CLI
pip install modal

# Authenticate with Modal (one-time setup)
modal token new

# This will open a browser to authenticate with your Modal account
# Free tier: 30 credits/month (enough for ~100-200 document processing runs)
```

---

## Approach 1: Modal Dev Mode (Recommended)

Run the Modal webhook **locally** while processing happens in Modal cloud. This is fastest for development.

### Step 1: Start Modal in Dev Mode

```bash
cd modal

# This starts a local webhook server that forwards to Modal cloud
modal serve modal_app.py
```

**Output:**
```
✓ Initialized. View run at https://modal.com/...
✓ Created objects.
├── 🔨 Created mount /Users/.../modal
└── 🔨 Created process_document_task.
✓ App deployed! 🎉

View Deployment: https://modal.com/apps/fltr/main

Webhook URLs:
  └─ POST => https://your-username--fltr-webhook-fastapi-app-dev.modal.run/process
```

**Copy the webhook URL** from the output.

### Step 2: Configure FastAPI

In `fastapi/.env`:
```bash
# Use the dev webhook URL from modal serve output
MODAL_WEBHOOK_URL=https://your-username--fltr-webhook-fastapi-app-dev.modal.run/process
USE_MODAL_FOR_PROCESSING=true
MODAL_ENABLED=true

# Other required vars
DATABASE_URL=postgresql://...
AUTH_DATABASE_URL=postgresql://...
MILVUS_URI=...
OPENAI_API_KEY=sk-...
R2_ACCOUNT_ID=...
R2_ACCESS_KEY_ID=...
R2_SECRET_ACCESS_KEY=...
R2_BUCKET_NAME=fltr-datasets
```

### Step 3: Start FastAPI

```bash
cd fastapi
./start-local-dev.sh
```

### Step 4: Test Document Upload

```bash
# Upload a document
curl -X POST http://localhost:8000/api/v1/webhooks/r2-upload \
  -H "Content-Type: application/json" \
  -d '{
    "object": {"key": "datasets/YOUR-DATASET-ID/test.pdf"},
    "bucket": "fltr-datasets",
    "eventType": "object.create",
    "eventTime": "2024-01-01T00:00:00Z",
    "datasetId": "YOUR-DATASET-ID"
  }'
```

### How It Works

```
User Upload
    ↓
FastAPI (localhost:8000) ──┐
    │                       │
    │ Trigger via HTTPS     │
    ↓                       │
Modal Webhook (local)       │ Both access
    │                       │ shared database
    │ Process in cloud      │
    ↓                       ↓
PostgreSQL Database ←───────┘
Milvus Vector DB
```

**Key points:**
- FastAPI triggers Modal via HTTPS (always works)
- Modal updates database directly (needs network access!)
- FastAPI polls database for status updates
- No callback from Modal to FastAPI

**Benefits:**
- ✅ Fast iteration (no deploy needed)
- ✅ Local debugging of webhook logic
- ✅ Processing happens in Modal cloud (no local dependencies)
- ✅ Hot reload on code changes

**Limitations:**
- ⚠️ Modal webhook must stay running
- ⚠️ Uses Modal credits (but free tier is generous)

---

## Approach 2: Full Modal Deploy

Deploy Modal to cloud, use production-like setup locally.

### Step 1: Deploy Modal

```bash
cd modal
modal deploy modal_app.py
```

**Output:**
```
✓ App deployed! 🎉

View Deployment: https://modal.com/apps/fltr/main

Webhook URLs:
  └─ POST => https://your-username--fltr-webhook-fastapi-app.modal.run/process
```

**Copy the webhook URL** (without `-dev`).

### Step 2: Configure FastAPI

Same as Approach 1, but use the **deployed** webhook URL:

```bash
MODAL_WEBHOOK_URL=https://your-username--fltr-webhook-fastapi-app.modal.run/process
```

### Step 3: Start FastAPI

```bash
cd fastapi
./start-local-dev.sh
```

### How It Works

```
User Upload
    ↓
FastAPI (localhost:8000) ──┐
    │                       │
    │ Trigger via HTTPS     │
    ↓                       │
Modal Cloud (webhook)       │ Both access
    │                       │ shared database
    │ Process in cloud      │
    ↓                       ↓
PostgreSQL Database ←───────┘
Milvus Vector DB
```

**Same as dev mode, but webhook also runs in cloud**

**Benefits:**
- ✅ Production-like setup
- ✅ No need to keep `modal serve` running
- ✅ Stable webhook URL

**Limitations:**
- ⚠️ Requires redeploy on Modal code changes
- ⚠️ Uses Modal credits

---

## Cloudflare Tunnel Setup (Optional)

**Use this if you chose Option 2 in Database Options** - exposing local PostgreSQL to Modal.

### Prerequisites

```bash
# macOS
brew install cloudflare/cloudflare/cloudflared

# Linux
wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb

# Windows
# Download from https://github.com/cloudflare/cloudflared/releases
```

### Step 1: Authenticate

```bash
cloudflared tunnel login
```

This opens a browser to authenticate with Cloudflare. Choose your account/domain.

### Step 2: Create Tunnel

```bash
# Create a named tunnel for PostgreSQL
cloudflared tunnel create fltr-postgres

# Output will show your tunnel ID and credentials file location
# Tunnel credentials written to: ~/.cloudflared/<tunnel-id>.json
```

### Step 3: Configure Tunnel

Create `~/.cloudflared/config.yml`:

```yaml
tunnel: fltr-postgres
credentials-file: /Users/YOUR_USER/.cloudflared/<tunnel-id>.json

ingress:
  # PostgreSQL (main database)
  - hostname: fltr-postgres.YOUR_DOMAIN.com
    service: tcp://localhost:5432

  # Milvus (if also local)
  - hostname: fltr-milvus.YOUR_DOMAIN.com
    service: tcp://localhost:19530

  # Catch-all rule (required)
  - service: http_status:404
```

### Step 4: Create DNS Records

```bash
# Add DNS records for your tunnel hostnames
cloudflared tunnel route dns fltr-postgres fltr-postgres.YOUR_DOMAIN.com
cloudflared tunnel route dns fltr-postgres fltr-milvus.YOUR_DOMAIN.com
```

### Step 5: Run Tunnel

```bash
# Run in foreground (for testing)
cloudflared tunnel run fltr-postgres

# Or run as background service
cloudflared service install
sudo systemctl start cloudflared
```

### Step 6: Update Modal Secrets

Now update your `.env` and Modal secrets to use the tunnel URLs:

```bash
# In fastapi/.env and modal/.env
DATABASE_URL=postgresql://user:pass@fltr-postgres.YOUR_DOMAIN.com:5432/fltr
AUTH_DATABASE_URL=postgresql://user:pass@fltr-postgres.YOUR_DOMAIN.com:5432/fltr_auth
MILVUS_URI=https://fltr-milvus.YOUR_DOMAIN.com:19530
```

### Step 7: Test Connection from Modal

```bash
# Start Modal dev mode
cd modal
modal serve modal_app.py

# In another terminal, get container ID
modal container list

# Shell into running container
modal shell <container-id>

# Inside container, test connection
psql $DATABASE_URL
# Should connect successfully!
```

### Troubleshooting Tunnel

**Tunnel disconnects:**
```bash
# Check tunnel status
cloudflared tunnel info fltr-postgres

# View tunnel logs
journalctl -u cloudflared -f  # Linux
tail -f /var/log/cloudflared.log  # macOS
```

**Connection refused:**
- Verify PostgreSQL is listening on localhost:5432
- Check firewall rules
- Verify credentials file path is correct

**DNS not resolving:**
- Wait 5-10 minutes for DNS propagation
- Use `nslookup fltr-postgres.YOUR_DOMAIN.com`
- Check Cloudflare dashboard for DNS records

---

## Debugging with Modal Tools

Modal provides powerful debugging tools for development.

### Hot Reload with `modal serve`

**Already covered above** - `modal serve` watches for file changes and automatically redeploys:

```bash
cd modal
modal serve modal_app.py

# Edit any file in modal/ directory
# Changes deploy automatically - no restart needed!
```

### Debug Shells - Access Running Containers

Get a shell inside a running Modal container:

```bash
# 1. List running containers
modal container list

# Output shows container IDs:
# Container ID: ta-01JK486KEYDM90QCAV2YBMCEFB
# App Name: fltr

# 2. Shell into container
modal shell ta-01JK486KEYDM90QCAV2YBMCEFB

# 3. Inside container, you can:
root ~ → ls -la                    # Explore filesystem
root ~ → ps aux                    # See running processes
root ~ → psql $DATABASE_URL        # Test database connection
root ~ → curl $MILVUS_URI          # Test Milvus connection
root ~ → cat /proc/meminfo         # Check memory usage
root ~ → env                       # View environment variables
```

**Use cases:**
- Verify database connection from Modal's perspective
- Check if secrets are set correctly
- Inspect mounted files
- Debug networking issues

### Execute Commands in Container

Run a single command without entering shell:

```bash
# Check database connection
modal container exec <container-id> psql $DATABASE_URL -c "SELECT version();"

# View environment
modal container exec <container-id> env | grep DATABASE

# Check file contents
modal container exec <container-id> ls /root

# Test network connectivity
modal container exec <container-id> curl -v https://api.openai.com
```

### Interactive Debugging with Breakpoints

Add breakpoints directly in your Modal code:

```python
# In modal/modal_app.py or any service file
@app.function(...)
def process_document_modal(dataset_id, object_key, task_id):
    # Add a breakpoint
    breakpoint()  # Execution pauses here

    # Or use modal.interact() for more control
    modal.interact()

    # Continue with processing
    file_bytes = download_from_r2(object_key)
    ...
```

**Run with interactive mode:**
```bash
modal run -i modal_app.py
```

When execution hits the breakpoint, you get a Python debugger:
```python
(Pdb) print(dataset_id)
'ae53b835-cddd-4914-a157-a132f5939940'
(Pdb) print(object_key)
'datasets/ae53b835/test.pdf'
(Pdb) next  # Step to next line
(Pdb) continue  # Resume execution
```

### IPython REPL in Modal

For more advanced debugging:

```python
@app.function()
def process_document_modal(dataset_id, object_key, task_id):
    # Process up to a certain point
    file_bytes = download_from_r2(object_key)

    # Drop into IPython REPL
    modal.interact()
    import IPython
    IPython.embed()

    # Now you can interactively explore variables:
    # >>> len(file_bytes)
    # >>> type(file_bytes)
    # >>> file_bytes[:100]
```

### Live Container Profiling

When a container seems stuck, use the Modal dashboard to see what it's doing:

1. Go to https://modal.com/apps/fltr/main
2. Click on a running function
3. Navigate to "Containers" tab
4. Click "Live Profiling" on a stuck container
5. See real-time code execution

**Shows:**
- Which function is running
- Stack trace of current execution
- Time spent in each function
- CPU/memory usage

### View Real-Time Logs

```bash
# Follow logs in real-time
modal app logs fltr::fltr --follow

# Filter for specific keywords
modal app logs fltr::fltr --follow | grep "ERROR"
modal app logs fltr::fltr --follow | grep "database"

# View recent logs
modal app logs fltr::fltr

# View logs for specific call
modal app logs fltr::fltr --call-id <call-id>
```

---

## Monitoring Modal Execution

### View Logs

```bash
# Real-time logs
modal app logs fltr::fltr --follow

# Recent logs
modal app logs fltr::fltr
```

### View Dashboard

Visit: https://modal.com/apps/fltr/main

Shows:
- Function calls
- Execution time
- Memory usage
- Errors
- Cost

### Debug Failed Runs

```bash
# List recent calls
modal app list-calls fltr::fltr

# Get details for a specific call
modal app logs fltr::fltr --call-id <call-id>
```

---

## Troubleshooting

### "Modal webhook not accessible"

**Problem:** FastAPI can't reach Modal webhook

**Fix:**
```bash
# Test Modal webhook directly
curl -X POST https://your-webhook-url/process \
  -H "Content-Type: application/json" \
  -d '{"dataset_id": "test", "object_key": "test.pdf", "task_id": "123"}'

# Should return: {"status": "queued", "call_id": "..."}
```

If this fails:
1. Check `modal serve` is still running (Approach 1)
2. Verify webhook URL is correct in `.env`
3. Check Modal dashboard for errors

### "ModuleNotFoundError" in Modal

**Problem:** Modal can't find your services

**Fix:** Make sure you're running from the `modal/` directory:
```bash
cd modal  # Important!
modal serve modal_app.py
```

### Document stuck in "uploaded" status

**Problem:** Modal processing not triggering

**Debug:**
```bash
# Check FastAPI logs
cd fastapi
tail -f logs/app.log | grep "Modal"

# Should see:
# 🚀 [Routing] Using Modal for document processing
# ✅ Queued Modal task: call_id

# Check Modal logs
modal app logs fltr::fltr --follow
```

**Common causes:**
1. `USE_MODAL_FOR_PROCESSING=false` in `.env`
2. Wrong webhook URL
3. Modal credits exhausted
4. Environment variables not set in Modal
5. **Database connectivity issues** (see below)

### "Can't connect to database" from Modal

**Problem:** Modal container can't reach your database

**Symptoms:**
- Documents stuck in "uploaded"
- Modal logs show connection errors
- Database timeout errors

**Debug using Modal shell:**
```bash
# Get container ID
modal container list

# Shell into container
modal shell <container-id>

# Test database connection
root ~ → psql $DATABASE_URL
# If this fails, Modal can't reach your database!

# Test with curl (if using cloud database)
root ~ → curl -v telnet://<db-host>:5432
```

**Fixes:**

**If using localhost database** ❌
- Switch to cloud database (Supabase, Neon, Railway)
- OR set up Cloudflare Tunnel (see section above)
- OR use ngrok to expose PostgreSQL

**If using cloud database** ✅
- Verify DATABASE_URL is correct in Modal secrets
- Check firewall rules allow Modal IPs
- Verify SSL settings (add `?sslmode=require` if needed)
- Test connection from Modal container (see above)

**If using Cloudflare Tunnel:**
- Check tunnel is running: `cloudflared tunnel info fltr-postgres`
- Verify DNS is resolving: `nslookup fltr-postgres.YOUR_DOMAIN.com`
- Test from Modal container: `psql $DATABASE_URL`

### "Can't connect to Milvus" from Modal

**Problem:** Similar to database - Modal can't reach Milvus

**Debug:**
```bash
# In Modal container
modal shell <container-id>

# Test Milvus connection
root ~ → curl -v $MILVUS_URI
root ~ → python -c "from pymilvus import MilvusClient; client = MilvusClient(uri='$MILVUS_URI'); print(client.list_collections())"
```

**Fixes:**
- Use Zilliz Cloud (recommended for simplicity)
- OR expose local Milvus via Cloudflare Tunnel
- OR deploy Milvus to cloud VM

### Environment Variables in Modal

Modal needs environment variables too! Set them in `modal_app.py`:

```python
# Already configured in modal_app.py:
secrets=[
    modal.Secret.from_dict({
        "DATABASE_URL": os.getenv("DATABASE_URL"),
        "OPENAI_API_KEY": os.getenv("OPENAI_API_KEY"),
        # etc...
    })
]
```

**To add new env var:**
1. Add to your local `.env` in `modal/` directory
2. Add to the secrets list in `modal_app.py`
3. Redeploy: `modal deploy modal_app.py`

---

## Quick Start (TLDR)

**First time:**
```bash
# 1. Setup Modal
pip install modal
modal token new

# 2. Start Modal dev server
cd modal
modal serve modal_app.py
# Copy webhook URL from output

# 3. Configure FastAPI
cd ../fastapi
echo "MODAL_WEBHOOK_URL=<your-webhook-url>" >> .env
echo "USE_MODAL_FOR_PROCESSING=true" >> .env

# 4. Start FastAPI
./start-local-dev.sh
```

**Daily development:**
```bash
# Terminal 1: Modal
cd modal
modal serve modal_app.py

# Terminal 2: FastAPI
cd fastapi
./start-local-dev.sh
```

---

## Cost Considerations

**Modal Free Tier:**
- 30 credits/month
- ~$0.10-0.30 per document (depending on size)
- ~100-300 documents/month free

**For heavier usage:**
- Upgrade to Modal paid plan ($20/month + usage)
- Or implement local Celery processing (requires adding back document_processor.py)

**Check usage:**
```bash
modal app stats fltr::fltr
```

Visit: https://modal.com/settings/billing

---

## Comparison: Modal Dev vs Deploy

| Feature | `modal serve` (Dev) | `modal deploy` (Prod) |
|---------|---------------------|----------------------|
| Webhook runs | Locally | Modal Cloud |
| Processing runs | Modal Cloud | Modal Cloud |
| Hot reload | ✅ Yes | ❌ No (need redeploy) |
| Stable URL | ❌ Changes | ✅ Stable |
| Best for | Active development | Testing/Production |

**Recommendation:** Use `modal serve` during active development, `modal deploy` when testing full flow.

---

## Need Help?

- **Modal docs**: https://modal.com/docs
- **Modal Discord**: https://discord.gg/modal
- **Check Modal status**: https://status.modal.com

## Next Steps

1. ✅ Set up Modal (`modal token new`)
2. ✅ Start dev mode (`modal serve modal_app.py`)
3. ✅ Configure FastAPI with webhook URL
4. ✅ Test document upload
5. ✅ Monitor in Modal dashboard
