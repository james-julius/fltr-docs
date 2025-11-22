# Debugging Modal Guide

Complete guide for debugging Modal document processing issues in FLTR.

---

## Quick Debugging Commands

```bash
# View real-time logs
modal app logs fltr::fltr --follow

# List running containers
modal container list

# Shell into container
modal shell <container-id>

# Check function history
modal app list-calls fltr::fltr

# Check usage/costs
modal app stats fltr::fltr
```

---

## Common Issues & Solutions

### 1. Documents Stuck in "Uploaded" Status

**Symptom:** Documents never progress beyond "uploaded" status

**Causes & Fixes:**

#### A. Database Connectivity (Most Common!)

Modal can't reach your database.

**Test:**
```bash
# Get container ID
modal container list

# Shell into container
modal shell <container-id>

# Test database connection
root ~ → psql $DATABASE_URL
```

**If connection fails:**
- ❌ Using `localhost:5432`? Switch to cloud database or tunnel
- ✅ Using cloud database? Verify URL, firewall, SSL settings
- 🔧 Using tunnel? Check tunnel is running and DNS resolves

**See:** [LOCAL_DEVELOPMENT.md](LOCAL_DEVELOPMENT.md#database-options) for setup

#### B. Modal Credits Exhausted

**Test:**
```bash
modal app stats fltr::fltr
# Check remaining credits
```

**Fix:** Upgrade plan at https://modal.com/settings/billing

#### C. Environment Variables Missing

**Test:**
```bash
modal container list
modal shell <container-id>

# Check environment
root ~ → env | grep DATABASE
root ~ → env | grep OPENAI
root ~ → env | grep MILVUS
```

**Fix:** Update Modal secrets in `modal/modal_app.py`:
```python
secrets=[
    modal.Secret.from_dict({
        "DATABASE_URL": os.getenv("DATABASE_URL"),
        "OPENAI_API_KEY": os.getenv("OPENAI_API_KEY"),
        # etc...
    })
]
```

#### D. Modal Webhook Not Triggered

**Test:**
```bash
# Check FastAPI logs
cd fastapi
tail -f logs/app.log | grep "Modal"

# Should see:
# 🚀 [Routing] Using Modal for document processing
# ✅ Queued Modal task: call_id
```

**Fix:**
- Verify `USE_MODAL_FOR_PROCESSING=true` in `.env`
- Check `MODAL_WEBHOOK_URL` is correct
- Test webhook: `curl -X POST $MODAL_WEBHOOK_URL -d '{"dataset_id":"test","object_key":"test.pdf","task_id":"123"}'`

---

### 2. Modal Container Errors

**Symptom:** Modal logs show exceptions or errors

#### A. Module Import Errors

```
ModuleNotFoundError: No module named 'services'
```

**Fix:**
- Run from correct directory: `cd modal && modal serve modal_app.py`
- Check mounts in `modal_app.py` include services directory
- Verify `__init__.py` exists in services folder

#### B. Database Connection Timeout

```
psycopg2.OperationalError: could not connect to server: Connection timed out
```

**Fix:**
- Database is not accessible from Modal (see section 1A above)
- Add `?connect_timeout=10` to DATABASE_URL
- Check database firewall rules

#### C. Milvus Connection Failed

```
MilvusException: <MilvusException: (code=1, message=Fail connecting to server...)>
```

**Fix:**
- Use Zilliz Cloud (https://zilliz.com) - recommended
- OR expose local Milvus via Cloudflare Tunnel
- Verify MILVUS_URI and MILVUS_TOKEN in Modal secrets

#### D. OpenAI API Errors

```
openai.error.AuthenticationError: Incorrect API key provided
```

**Fix:**
- Verify OPENAI_API_KEY is set in Modal secrets
- Check API key is valid: `curl https://api.openai.com/v1/models -H "Authorization: Bearer $OPENAI_API_KEY"`
- Check OpenAI account has credits

---

### 3. Slow Processing

**Symptom:** Documents take very long to process

**Debug:**

1. **Check Modal dashboard** - https://modal.com/apps/fltr/main
   - View execution time breakdown
   - Check memory usage
   - View function call history

2. **Use live profiling:**
   - Click on running container
   - Navigate to "Containers" tab
   - Click "Live Profiling"
   - See what code is running in real-time

3. **Add timing logs:**
```python
import time

@app.function()
def process_document_modal(dataset_id, object_key, task_id):
    start = time.time()

    file_bytes = download_from_r2(object_key)
    print(f"⏱️ Download: {time.time() - start:.2f}s")

    doc = parse_document(file_bytes, object_key)
    print(f"⏱️ Parse: {time.time() - start:.2f}s")

    chunks = chunk_document(doc)
    print(f"⏱️ Chunk: {time.time() - start:.2f}s")

    embeddings = generate_embeddings(chunks)
    print(f"⏱️ Embed: {time.time() - start:.2f}s")
```

**Common bottlenecks:**
- Large PDFs (100+ pages) → Takes longer to parse
- Many chunks (1000+) → Embedding generation is slow
- Docling parsing complex PDFs → Can take 30-60s per document

---

### 4. Modal Webhook 503 Errors

**Symptom:** FastAPI gets 503 when calling Modal webhook

**Causes:**

#### A. Cold Start
First request after idle can take 30-60s to spawn container.

**Fix:** Already configured with `min_containers=1` for webhook

#### B. Modal Service Down
Check https://status.modal.com

#### C. Wrong URL
**Test:**
```bash
curl -X POST $MODAL_WEBHOOK_URL/process \
  -H "Content-Type: application/json" \
  -d '{"dataset_id":"test","object_key":"test.pdf","task_id":"123"}'

# Should return: {"status": "queued", "call_id": "..."}
```

---

## Debugging Tools

### 1. Hot Reload with `modal serve`

**For active development:**
```bash
cd modal
modal serve modal_app.py

# Edit any file - changes deploy automatically!
```

**Benefits:**
- Instant feedback on code changes
- No need to redeploy
- Perfect for iterative debugging

### 2. Debug Shells

**Access running Modal containers:**
```bash
# List containers
modal container list

# Shell into specific container
modal shell <container-id>

# Inside container, you can:
root ~ → ls -la              # Explore filesystem
root ~ → ps aux              # See processes
root ~ → psql $DATABASE_URL  # Test database
root ~ → curl $MILVUS_URI    # Test Milvus
root ~ → env                 # View environment
root ~ → cat /proc/meminfo   # Check memory
```

**Execute single command:**
```bash
# Without entering shell
modal container exec <container-id> psql $DATABASE_URL -c "SELECT version();"
modal container exec <container-id> env | grep DATABASE
modal container exec <container-id> ls /root
```

### 3. Interactive Debugging with Breakpoints

**Add to Modal code:**
```python
@app.function()
def process_document_modal(dataset_id, object_key, task_id):
    # Add breakpoint
    breakpoint()  # Pauses execution

    # Continue processing
    file_bytes = download_from_r2(object_key)
    ...
```

**Run with interactive mode:**
```bash
modal run -i modal_app.py
```

**Debugger commands:**
```python
(Pdb) print(dataset_id)          # Print variable
(Pdb) print(object_key)
(Pdb) next                       # Next line
(Pdb) step                       # Step into function
(Pdb) continue                   # Continue execution
(Pdb) list                       # Show code
(Pdb) help                       # Show commands
```

### 4. IPython REPL

**For advanced exploration:**
```python
@app.function()
def process_document_modal(dataset_id, object_key, task_id):
    file_bytes = download_from_r2(object_key)

    # Drop into IPython
    modal.interact()
    import IPython
    IPython.embed()

    # Interactive exploration:
    # >>> len(file_bytes)
    # >>> file_bytes[:100]
    # >>> import json
    # >>> data = json.loads(file_bytes)
```

### 5. View Logs

**Real-time logs:**
```bash
# Follow all logs
modal app logs fltr::fltr --follow

# Filter for errors
modal app logs fltr::fltr --follow | grep ERROR
modal app logs fltr::fltr --follow | grep Exception

# Filter for specific document
modal app logs fltr::fltr --follow | grep <dataset-id>
```

**Historical logs:**
```bash
# Recent logs
modal app logs fltr::fltr

# Specific call
modal app logs fltr::fltr --call-id <call-id>

# List recent calls
modal app list-calls fltr::fltr
```

### 6. Live Container Profiling

**When container seems stuck:**

1. Go to https://modal.com/apps/fltr/main
2. Click on your function
3. Navigate to "Containers" tab
4. Click "Live Profiling" on stuck container

**Shows:**
- Current function executing
- Stack trace
- Time in each function
- CPU/memory usage

---

## Testing Modal Locally

### Test Document Processing End-to-End

```bash
# 1. Start Modal in dev mode
cd modal
modal serve modal_app.py

# 2. In another terminal, upload a test document
cd fastapi
curl -X POST http://localhost:8000/api/v1/webhooks/r2-upload \
  -H "Content-Type: application/json" \
  -d '{
    "object": {"key": "datasets/<dataset-id>/test.pdf"},
    "bucket": "fltr-datasets",
    "eventType": "object.create",
    "eventTime": "2024-01-01T00:00:00Z",
    "datasetId": "<dataset-id>"
  }'

# 3. Watch Modal logs
modal app logs fltr::fltr --follow
```

### Test Database Connection from Modal

```bash
# Get running container
modal container list

# Shell in
modal shell <container-id>

# Test connection
root ~ → psql $DATABASE_URL -c "SELECT version();"
root ~ → psql $DATABASE_URL -c "SELECT id, status FROM documents LIMIT 5;"
```

### Test Milvus Connection from Modal

```bash
# In Modal container
modal shell <container-id>

# Test with Python
root ~ → python << EOF
from pymilvus import MilvusClient
import os

client = MilvusClient(
    uri=os.getenv('MILVUS_URI'),
    token=os.getenv('MILVUS_TOKEN')
)
print("Collections:", client.list_collections())
EOF
```

### Test R2 Access from Modal

```bash
# In Modal container
modal shell <container-id>

# Test with boto3
root ~ → python << EOF
import boto3
import os

client = boto3.client(
    's3',
    endpoint_url=f"https://{os.getenv('R2_ACCOUNT_ID')}.r2.cloudflarestorage.com",
    aws_access_key_id=os.getenv('R2_ACCESS_KEY_ID'),
    aws_secret_access_key=os.getenv('R2_SECRET_ACCESS_KEY'),
    region_name='auto'
)
print("Buckets:", [b['Name'] for b in client.list_buckets()['Buckets']])
EOF
```

---

## Performance Optimization

### Check Resource Usage

```bash
# View app stats
modal app stats fltr::fltr

# View specific function stats
modal app function-stats fltr::fltr::process_document_modal
```

### Monitor Costs

```bash
# Check credit usage
modal app stats fltr::fltr

# View billing dashboard
# https://modal.com/settings/billing
```

### Optimize for Cost

**Tips:**
- Use smaller container sizes if possible
- Batch embedding generation (already doing 100 per batch)
- Consider caching parsed documents
- Monitor for stuck containers

---

## Environment Variables Checklist

**Required in Modal secrets (set in modal_app.py):**

- [ ] DATABASE_URL (must be accessible from cloud!)
- [ ] AUTH_DATABASE_URL (must be accessible from cloud!)
- [ ] MILVUS_URI (must be accessible from cloud!)
- [ ] MILVUS_TOKEN (if using Zilliz Cloud)
- [ ] OPENAI_API_KEY
- [ ] R2_ACCOUNT_ID
- [ ] R2_ACCESS_KEY_ID
- [ ] R2_SECRET_ACCESS_KEY
- [ ] R2_BUCKET_NAME

**Common mistakes:**
- Using localhost URLs ❌
- Missing SSL parameters for cloud databases
- Typos in environment variable names
- Using old/revoked API keys

---

## Getting Help

**If stuck:**

1. **Check Modal logs:**
   ```bash
   modal app logs fltr::fltr --follow
   ```

2. **Test from Modal's perspective:**
   ```bash
   modal shell <container-id>
   # Test each service connection
   ```

3. **Check Modal status:**
   - https://status.modal.com

4. **Modal documentation:**
   - https://modal.com/docs
   - https://modal.com/docs/guide/debugging

5. **Modal Discord:**
   - https://discord.gg/modal

6. **FLTR documentation:**
   - [LOCAL_DEVELOPMENT.md](LOCAL_DEVELOPMENT.md)
   - [QUICK_START.md](QUICK_START.md)

---

## Debug Checklist

When a document won't process, check:

- [ ] Modal is running (`modal serve` or deployed)
- [ ] Database is accessible from cloud (test with `modal shell`)
- [ ] Milvus is accessible from cloud
- [ ] R2 credentials are correct
- [ ] OpenAI API key is valid and has credits
- [ ] Modal credits available
- [ ] Environment variables set in Modal secrets
- [ ] `USE_MODAL_FOR_PROCESSING=true` in FastAPI .env
- [ ] MODAL_WEBHOOK_URL is correct
- [ ] Document exists in R2
- [ ] Document format is supported (.pdf, .txt, .docx, etc.)

**Still stuck?** Shell into Modal container and test each connection manually.
