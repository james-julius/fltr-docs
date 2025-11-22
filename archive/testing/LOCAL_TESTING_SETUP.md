# Local Testing Setup Guide

This guide explains how to set up local test data (R2 files and Milvus vectors) for full end-to-end testing without requiring production cloud resources.

## Overview

The local testing setup allows you to:
- Test document search functionality locally
- Work with real data without hitting production APIs
- Debug issues with actual datasets
- Develop features offline

## Architecture

- **Local R2 Storage**: Files stored in `cloudflare-workers/r2-upload-proxy/.wrangler-state/`
- **Local Milvus-Lite**: Vectors stored in `fastapi/data/milvus_lite.db`
- **Production Sources**: Data synced from production R2 and Milvus (Zilliz Cloud)

## Prerequisites

### 1. Environment Variables

You need production credentials to sync data:

**Production R2** (for syncing files):
```bash
R2_ACCOUNT_ID=your_account_id
R2_ACCESS_KEY_ID=your_access_key
R2_SECRET_ACCESS_KEY=your_secret_key
R2_BUCKET_NAME=fltr-datasets  # or your production bucket name
```

**Production Milvus** (for syncing vectors):
```bash
MILVUS_URI=https://xxx.zillizcloud.com
MILVUS_TOKEN=your_token
# OR
MILVUS_USER=your_user
MILVUS_PASSWORD=your_password
```

**Local Configuration** (already set):
```bash
USE_MILVUS_LITE=True
MILVUS_LITE_PATH=data/milvus_lite.db
R2_UPLOAD_PROXY_URL=http://localhost:8787  # Optional, defaults to this
```

### 2. Services Running

Before syncing, ensure these services are running:

1. **Local R2 Proxy**:
   ```bash
   cd cloudflare-workers/r2-upload-proxy
   npx wrangler dev --port 8787 --persist-to=./.wrangler-state
   ```

2. **FastAPI** (optional, but needed for database access):
   ```bash
   cd fastapi
   python -m uvicorn main:app --reload --port 8000
   ```

### 3. Python Dependencies

Install required Python packages:
```bash
cd fastapi
pip install boto3 pymilvus
```

**Optional** (for progress bars):
```bash
pip install tqdm
# Or install all processing dependencies:
pip install "fltr-api[processing]"
```

Note: Scripts will work without `tqdm`, but won't show progress bars.

## Step-by-Step Sync Instructions

### Option 1: Interactive Sync (Recommended)

Run the combined sync script without arguments to get an interactive dataset selector:

```bash
./scripts/sync-local-test-data.sh
```

This will:
1. Show you a list of available datasets
2. Let you choose which dataset to sync
3. Sync R2 data (documents and images)
4. Sync Milvus vectors

### Option 2: Sync Specific Dataset

If you know the dataset ID:

```bash
./scripts/sync-local-test-data.sh <dataset_id>
```

Example:
```bash
./scripts/sync-local-test-data.sh 123e4567-e89b-12d3-a456-426614174000
```

### Option 3: Sync Multiple Datasets

```bash
./scripts/sync-local-test-data.sh <dataset_id_1> <dataset_id_2> <dataset_id_3>
```

### Option 4: Individual Scripts

You can also run the scripts individually:

**Sync R2 data only**:
```bash
cd fastapi
python scripts/sync_r2_data.py [--dataset-id DATASET_ID] [--dry-run]
```

**Sync Milvus vectors only**:
```bash
cd fastapi
python scripts/sync_milvus_vectors.py [--dataset-id DATASET_ID] [--dry-run]
```

## Dry Run Mode

Preview what would be synced without actually syncing:

```bash
./scripts/sync-local-test-data.sh --dry-run
# Or individually:
python fastapi/scripts/sync_r2_data.py --dry-run
python fastapi/scripts/sync_milvus_vectors.py --dry-run
```

## Verifying Sync Worked

### 1. Check R2 Data

Local R2 files are stored in:
```
cloudflare-workers/r2-upload-proxy/.wrangler-state/
```

You can verify files exist:
```bash
ls -lh cloudflare-workers/r2-upload-proxy/.wrangler-state/
```

### 2. Check Milvus Vectors

Local Milvus-Lite database:
```
fastapi/data/milvus_lite.db
```

Verify collection exists and has data:
```bash
cd fastapi
python -c "
from database.vector_store import get_milvus_client
client = get_milvus_client()
print('Collections:', client.list_collections())
if client.has_collection('fltr_documents'):
    result = client.query('fltr_documents', 'dataset_id == \"YOUR_DATASET_ID\"', limit=1)
    print(f'Vectors found: {len(result)}')
"
```

### 3. Test Search

Start your local development environment and test search:

```bash
# Start all services
./start-local-dev.sh

# In another terminal, start Next.js
cd nextjs
npm run dev
```

Then test search functionality in the UI at `http://localhost:3000`

## Troubleshooting

### Issue: "Local R2 proxy not running"

**Solution**: Start the R2 proxy:
```bash
cd cloudflare-workers/r2-upload-proxy
npx wrangler dev --port 8787 --persist-to=./.wrangler-state
```

### Issue: "Production R2 credentials not configured"

**Solution**: Set environment variables:
```bash
export R2_ACCOUNT_ID=your_account_id
export R2_ACCESS_KEY_ID=your_access_key
export R2_SECRET_ACCESS_KEY=your_secret_key
export R2_BUCKET_NAME=fltr-datasets
```

### Issue: "Production Milvus URI not configured"

**Solution**: Set Milvus credentials:
```bash
export MILVUS_URI=https://xxx.zillizcloud.com
export MILVUS_TOKEN=your_token
```

### Issue: "No vectors found for dataset"

**Possible causes**:
- Dataset hasn't been processed in production yet
- Dataset ID is incorrect
- Vectors were deleted

**Solution**: Check production Milvus directly or process the dataset first.

### Issue: Images not loading in UI

**Solution**: Update Next.js environment variable:
```bash
# In nextjs/.env.local
NEXT_PUBLIC_R2_IMAGE_WORKER_URL=http://localhost:8787
```

Then restart Next.js dev server.

### Issue: "Collection does not exist locally"

**Solution**: The collection should be created automatically. If not:
```bash
cd fastapi
python -c "from database.vector_store import ensure_default_collection; ensure_default_collection()"
```

### Issue: Sync is slow

**Note**: Large datasets can take time to sync. The scripts show progress bars. For very large datasets, consider syncing a subset first.

## Adding More Test Data

To add more datasets:

1. Run the sync script again with new dataset IDs
2. The scripts support incremental updates - existing data won't be duplicated
3. You can sync multiple datasets in one command

## Data Storage Locations

- **R2 Files**: `cloudflare-workers/r2-upload-proxy/.wrangler-state/`
- **Milvus Vectors**: `fastapi/data/milvus_lite.db`
- **Database Records**: PostgreSQL (local or remote, depending on `DATABASE_URL`)

## Notes

- **File Size**: Large datasets may take significant time to sync. Progress bars show status.
- **Incremental Updates**: Scripts can be re-run safely - they won't duplicate existing data.
- **Image URLs**: Frontend may need `NEXT_PUBLIC_R2_IMAGE_WORKER_URL` pointing to local proxy.
- **Database Records**: Dataset/document records in PostgreSQL are separate from R2/Milvus data. You may need to ensure database records exist for full testing.
- **Milvus Schema**: Local Milvus-Lite collection should match production schema (keywords field, etc.). The `ensure_default_collection()` function handles this.

## Next Steps

After syncing data:

1. Verify data is accessible
2. Test search functionality
3. Test document viewing
4. Test image display
5. Develop and debug features locally

For questions or issues, check the script logs or review the source code in `fastapi/scripts/`.

