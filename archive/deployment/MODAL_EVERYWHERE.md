# Using Modal for All Document Processing

## Current State

FLTR has **two processing modes**:
- **Celery** (local, requires Redis): `USE_MODAL_FOR_PROCESSING=false`
- **Modal** (serverless): `USE_MODAL_FOR_PROCESSING=true`

## Recommendation: Use Modal Everywhere ✅

Simplify your stack by using Modal for both local development and production.

### Why Modal Everywhere?

**Pros:**
- ✅ **Identical code paths** - Dev exactly matches production
- ✅ **No local infrastructure** - No Redis, no Celery workers
- ✅ **Modal free tier** - 30 CPU hours/month free
- ✅ **Auto-scaling** - Modal handles scaling automatically
- ✅ **Faster development** - No local worker processes to manage
- ✅ **Simpler onboarding** - New devs: `modal token set` and done

**Cons:**
- ⚠️ Requires internet connection for processing
- ⚠️ ~500ms cold start (mitigated by warm containers)

### Setup for Local Development

1. **Install Modal CLI:**
   ```bash
   pip install modal
   ```

2. **Authenticate:**
   ```bash
   modal token set
   ```

3. **Deploy your Modal function:**
   ```bash
   cd modal
   modal deploy modal_app.py
   ```

4. **Update your `.env`:**
   ```bash
   # In fastapi/.env
   USE_MODAL_FOR_PROCESSING=true
   MODAL_WEBHOOK_URL=https://yourapp--fltr-process-document.modal.run
   ```

5. **Remove Celery/Redis (optional cleanup):**
   ```bash
   # No longer needed:
   # - docker-compose up redis
   # - celery -A celery_app worker
   ```

That's it! Start FastAPI and uploads will process through Modal.

### Local Development Workflow

```bash
# Terminal 1: Start FastAPI
cd fastapi
python main.py

# Terminal 2: Hot reload Modal function (optional)
cd modal
modal deploy modal_app.py --watch

# Upload a document - it processes through Modal!
```

### Keeping Celery (If You Must)

If you want to keep Celery for offline development:

1. Keep `USE_MODAL_FOR_PROCESSING=false` in `.env`
2. Run Redis: `docker-compose up redis`
3. Run Celery: `celery -A celery_app worker --loglevel=info`

The code will automatically fallback to Celery if Modal is unavailable.

### Production Setup

Production already uses Modal. Just ensure:

```bash
# In production env
USE_MODAL_FOR_PROCESSING=true
MODAL_WEBHOOK_URL=https://yourapp--fltr-process-document.modal.run
```

## Migration Path

### Step 1: Test Modal Locally

```bash
cd modal
modal deploy modal_app.py

# Update fastapi/.env
USE_MODAL_FOR_PROCESSING=true
MODAL_WEBHOOK_URL=<from modal deploy output>

# Test a document upload
```

### Step 2: Remove Celery Dependencies (Optional)

Once confident Modal works:

```bash
# Stop Redis
docker-compose down redis

# Remove from requirements.txt (optional - can keep for backward compatibility)
# - celery
# - redis

# Remove docker-compose.yaml redis service (optional)
```

### Step 3: Update Documentation

Update your team's setup docs to remove Redis/Celery steps.

## Cost Comparison

### Modal Free Tier:
- 30 CPU hours/month
- ~3,600 documents/month (assuming 30s per document)
- Perfect for development and small teams

### Beyond Free Tier:
- Pay-as-you-go: ~$0.000008/CPU second
- Example: 10,000 documents/month × 30s = ~$24/month

### Celery Alternative:
- Redis hosting: $10-50/month
- Worker VM: $5-20/month
- Total: $15-70/month + maintenance overhead

**Modal wins on both cost and simplicity for most use cases.**

## Troubleshooting

### "Modal is not available"

**Symptom**: Uploads fail with "Modal webhook URL not configured"

**Solution**:
```bash
# 1. Check Modal deployment
modal app list

# 2. Get webhook URL
modal app logs fltr --function process_document

# 3. Update .env
MODAL_WEBHOOK_URL=<your-webhook-url>
```

### "Cold start delays"

**Symptom**: First upload after inactivity takes longer

**Solution**: Modal keeps containers warm. For heavy usage:
```python
# In modal_app.py
@app.function(
    min_containers=1,  # Keep 1 container always warm
    ...
)
```

### "Rate limits"

**Symptom**: Many concurrent uploads slow down

**Solution**: Modal auto-scales. Increase concurrency:
```python
# In modal_app.py
@app.function(
    max_containers=10,  # Allow up to 10 concurrent containers
    ...
)
```

## Decision Matrix

| Use Case | Recommendation |
|----------|---------------|
| **Solo developer** | Modal everywhere ✅ |
| **Small team (<5)** | Modal everywhere ✅ |
| **Enterprise (>100 docs/min)** | Modal with warm containers ✅ |
| **Offline development required** | Keep Celery fallback ⚠️ |
| **Very high volume (>1M docs/month)** | Evaluate Celery vs Modal cost ⚠️ |

## Summary

**For 90% of use cases: Use Modal everywhere.**

It's simpler, cheaper, and eliminates infrastructure complexity. The code already supports both modes, so you can always fallback to Celery if needed.

**Next Steps:**
1. Test Modal locally (10 minutes)
2. If it works, update `.env` permanently
3. Remove Celery from local setup
4. Update team documentation

---

**Questions?** See [modal/README.md](modal/README.md) for Modal-specific documentation.
