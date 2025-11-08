# End-to-End Local Development Setup

Complete guide for testing the full dataset upload → processing flow locally.

## 🏗️ Architecture Overview

```
Next.js (localhost:3000)
    ↓ (1. Request presigned URL)
FastAPI (localhost:8000)
    ↓ (2. Generate presigned URL for fltr-datasets-dev)
Browser
    ↓ (3. Upload file directly to R2)
R2 Bucket (fltr-datasets-dev) [REAL CLOUDFLARE]
    ↓ (4. Trigger notification event)
Notification Worker (localhost:8788) [LOCAL with REMOTE R2]
    ↓ (5. Publish to queue)
Queue (fltr-document-processing) [SIMULATED LOCALLY]
    ↓ (6. Consume messages)
Queue Consumer Worker (localhost:8787) [LOCAL]
    ↓ (7. POST webhook to FastAPI)
FastAPI (localhost:8000)
    ↓ (8. Process document)
Done! ✅
```

## 🚀 Quick Start

### Step 1: Start FastAPI (Terminal 1)

```bash
cd fastapi

# Set environment to development (auto-uses fltr-datasets-dev bucket)
export ENVIRONMENT=development

# Start FastAPI
uvicorn main:app --reload --port 8000
```

**Verify**: Visit http://localhost:8000/health - should return healthy status

### Step 2: Start Queue Consumer Worker (Terminal 2)

```bash
cd cloudflare-workers/queue-consumer-worker

# Run with local development settings
npx wrangler dev --port 8787
```

**What this does:**
- ✅ Worker code runs locally
- ✅ Queue binding is simulated locally
- ✅ Forwards messages to http://localhost:8000

### Step 3: Start Notification Worker (Terminal 3)

```bash
cd cloudflare-workers/dataset-upload-notification-worker

# Run with remote R2 bucket
npx wrangler dev --remote --port 8788
```

**What this does:**
- ✅ Worker code runs locally
- ✅ R2 binding connects to REAL `fltr-datasets-dev` bucket
- ✅ Publishes to local simulated queue
- ✅ Detects uploads to dev bucket

**Important**: Using `--remote` connects to the real dev R2 bucket and listens for actual upload events!

### Step 4: Start Next.js (Terminal 4)

```bash
cd nextjs

# Ensure .env has correct API URL
# NEXT_PUBLIC_API_URL=http://localhost:8000  # Direct
# OR
# NEXT_PUBLIC_API_URL=/api/proxy            # Via proxy

npm run dev
```

**Verify**: Visit http://localhost:3000/my-datasets/create

## 🧪 Testing the Full Flow

### Test Upload:

1. **Open Next.js**: http://localhost:3000/my-datasets/create
2. **Create a dataset**:
   - Name: "Test Dataset"
   - Description: "Testing local flow"
   - Category: Documentation
3. **Upload a file**: Any supported file (PDF, JSON, CSV, etc.)
4. **Watch the logs**:

**Terminal 1 (FastAPI):**
```
INFO: Using R2 bucket: fltr-datasets-dev (environment: development)
INFO: Generated presigned URL for: test-dataset/sample.pdf
INFO: Received webhook from queue consumer
```

**Terminal 2 (Queue Consumer):**
```
Processing queue message: { bucket: "fltr-datasets-dev", object: "..." }
Forwarding to FastAPI: http://localhost:8000/api/v1/webhooks/...
```

**Terminal 3 (Notification Worker):**
```
R2 upload detected: fltr-datasets-dev/dataset-xxx/sample.pdf
Published to queue: fltr-document-processing
```

**Terminal 4 (Next.js):**
```
Upload complete: 100%
Dataset created successfully!
```

## 📁 File Organization

Files uploaded during local development go to:
- **Bucket**: `fltr-datasets-dev` (real Cloudflare R2)
- **Path**: `dataset-{uuid}/{filename}`
- **Cleanup**: Manually delete test files from Cloudflare dashboard

## 🔧 Configuration Details

### FastAPI Auto-Bucket Switching

When `ENVIRONMENT=development`, FastAPI automatically uses `fltr-datasets-dev`:

```python
# config.py
@property
def r2_bucket_name_for_env(self) -> str:
    if self.ENVIRONMENT == "development":
        return f"{self.R2_BUCKET_NAME}-dev"  # fltr-datasets-dev
    return self.R2_BUCKET_NAME                # fltr-datasets
```

### Worker Remote Binding

The notification worker uses `--remote` flag to connect to real R2:

```bash
npx wrangler dev --remote
```

This uses the `preview_bucket_name` from wrangler.toml:
```toml
[[r2_buckets]]
binding = "fltr_datasets"
bucket_name = "fltr-datasets"           # Production
preview_bucket_name = "fltr-datasets-dev"  # Development (used with --remote)
```

### Queue Simulation

The queue is simulated locally - both workers connect to the same local queue instance when running `wrangler dev` simultaneously.

## 🐛 Troubleshooting

### Upload succeeds but no notification

**Issue**: Notification worker not detecting uploads

**Solutions**:
1. Ensure notification worker is running with `--remote` flag
2. Check that `preview_bucket_name = "fltr-datasets-dev"` is in wrangler.toml
3. Verify you're logged in: `npx wrangler login`
4. Check bucket name matches in FastAPI and wrangler.toml

### Queue messages not being consumed

**Issue**: Consumer worker not receiving messages

**Solutions**:
1. Both workers must be running simultaneously
2. Queue names must match in both wrangler.toml files
3. Check consumer logs: `tail -f consumer.log`

### FastAPI not receiving webhooks

**Issue**: Consumer can't reach FastAPI

**Solutions**:
1. Verify FastAPI is running on http://localhost:8000
2. Check `.dev.vars` has correct FASTAPI_URL
3. Ensure API key matches between worker and FastAPI

### "Bucket not found" error

**Issue**: Dev bucket doesn't exist

**Solutions**:
1. Create bucket manually: `npx wrangler r2 bucket create fltr-datasets-dev`
2. Or deploy notification worker once: `npx wrangler deploy` (creates preview bucket)
3. Verify with: `npx wrangler r2 bucket list`

## 📊 Monitoring

### View R2 Files

```bash
# List files in dev bucket
npx wrangler r2 object list fltr-datasets-dev

# Download a file
npx wrangler r2 object get fltr-datasets-dev/dataset-xxx/file.pdf --file=./downloaded.pdf

# Delete test files
npx wrangler r2 object delete fltr-datasets-dev/dataset-xxx/file.pdf
```

### View Queue Messages

Queue is simulated locally - check worker logs to see messages.

### Check FastAPI Logs

All webhook calls are logged in Terminal 1 where FastAPI is running.

## 🎯 Best Practices

1. **Use dev bucket only**: Never point local dev to production bucket
2. **Clean up test files**: Regularly delete test uploads from dev bucket
3. **Test incrementally**: Test each component separately before full e2e
4. **Use meaningful test data**: Name datasets/files clearly for easy identification
5. **Monitor costs**: Even dev bucket operations have costs (though minimal)

## 🚢 Moving to Production

When ready to deploy:

1. **FastAPI**: Set `ENVIRONMENT=production` in Coolify
2. **Workers**: Deploy without `--remote` flag:
   ```bash
   npx wrangler deploy
   ```
3. **Next.js**: Update `NEXT_PUBLIC_API_URL` to production API

Production will automatically use:
- R2: `fltr-datasets` (production bucket)
- Queue: Real Cloudflare Queue (not simulated)
- All workers deployed to Cloudflare edge
