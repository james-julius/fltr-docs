# Modal Quick Start - 5 Minute Setup

Get Modal running in 5 commands.

## 1. Install & Authenticate (1 min)

```bash
pip install modal
modal token new
```

## 2. Create Secrets (2 min)

```bash
modal secret create fltr-r2-credentials \
  R2_ACCOUNT_ID="" \
  R2_SECRET_ACCESS_KEY="" \
  R2_BUCKET_NAME=""

modal secret create fltr-openai-api-key OPENAI_API_KEY=""

modal secret create fltr-postgres-url POSTGRES_URL=""

modal secret create fltr-milvus-credentials \
  MILVUS_URI=""
  MILVUS_TOKEN=""
```

## 3. Deploy (1 min)

```bash
cd fastapi
modal deploy modal_app.py
```

Copy the webhook URL from output.

## 4. Configure FastAPI (30 seconds)

Add to `.env`:
```bash
MODAL_ENABLED=true
MODAL_WEBHOOK_URL=https://yourapp--webhook-process-document.modal.run
MODAL_ROLLOUT_PERCENTAGE=10  # Start with 10%
```

## 5. Restart & Test (30 seconds)

```bash
# Restart FastAPI
uvicorn main:app --reload

# Upload a document and watch logs
```

## Verify It Works

1. **Check Modal Dashboard**: [modal.com/apps](https://modal.com/apps)
2. **View Logs**: `modal app logs fltr-document-processing`
3. **Check Database**: Document should be marked "ready"

## Gradual Rollout

```bash
# Week 1: 10%
MODAL_ROLLOUT_PERCENTAGE=10

# Week 2: 25%
MODAL_ROLLOUT_PERCENTAGE=25

# Week 3: 50%
MODAL_ROLLOUT_PERCENTAGE=50

# Week 4: 100%
USE_MODAL_FOR_PROCESSING=true
```

## Rollback

```bash
# Immediate rollback to Celery
USE_MODAL_FOR_PROCESSING=false
MODAL_ROLLOUT_PERCENTAGE=0
```

Restart FastAPI - done!

## Common Commands

```bash
# View logs
modal app logs fltr-document-processing --follow

# List secrets
modal secret list

# Redeploy after code changes
modal deploy modal_app.py

# View usage/costs
# Go to: https://modal.com/usage
```

## Cost Estimate

**Current (Celery)**: $50/month fixed

**Modal (Estimated)**:
- 3000 docs/month × $0.006/doc = **$18/month**
- **Save $32/month (64%)**

## Need Help?

See full guide: [MODAL_DEPLOYMENT.md](../MODAL_DEPLOYMENT.md)
