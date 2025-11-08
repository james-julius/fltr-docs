# Modal Deployment Guide

Complete step-by-step guide to deploy document processing to Modal.com

## Prerequisites

- [ ] Modal account created at [modal.com](https://modal.com)
- [ ] Python environment with `modal` package installed
- [ ] Access to production credentials (R2, OpenAI, Postgres, Milvus)

## Step 1: Install Modal SDK

```bash
cd fastapi
pip install modal>=0.63.0
```

## Step 2: Authenticate with Modal

```bash
# This will open a browser window for authentication
modal token new
```

You should see:
```
✓ Initialized token.
```

## Step 3: Create Modal Secrets

Modal secrets store your credentials securely. Create these via CLI or Dashboard.

### Option A: Via CLI (Recommended)

```bash
# R2 Credentials
modal secret create fltr-r2-credentials \
  R2_ACCOUNT_ID=your_account_id \
  R2_ACCESS_KEY_ID=your_access_key \
  R2_SECRET_ACCESS_KEY=your_secret_key \
  R2_BUCKET_NAME=fltr-datasets

# OpenAI API Key
modal secret create fltr-openai-api-key \
  OPENAI_API_KEY=sk-your-key-here

# PostgreSQL Connection
modal secret create fltr-postgres-url \
  POSTGRES_URL=postgresql://user:pass@host:5432/database

# Milvus/Zilliz Connection
modal secret create fltr-milvus-credentials \
  MILVUS_URI=https://your-instance.zillizcloud.com \
  MILVUS_TOKEN=your_token_here
```

### Option B: Via Dashboard

1. Go to [modal.com/secrets](https://modal.com/secrets)
2. Click "Create Secret"
3. Add each group of credentials as shown above

## Step 4: Test Modal App Locally

```bash
# Run the webhook locally to test
modal serve modal_app.py
```

This will:
- Build the container image (first time takes ~2-3 minutes)
- Start a local dev server
- Give you a temporary webhook URL

Test with:
```bash
curl -X POST https://your-dev-url.modal.run \
  -H "Content-Type: application/json" \
  -d '{
    "dataset_id": "test-uuid",
    "object_key": "test/file.pdf"
  }'
```

## Step 5: Deploy to Production

```bash
# Deploy the Modal app
modal deploy modal_app.py
```

You should see output like:
```
✓ Created objects.
├── 🔨 Created mount /Users/.../fastapi/services
├── 🔨 Created fltr-document-processing/process_document_modal
└── 🔨 Created web function fltr-document-processing/webhook_process_document => https://yourapp--webhook-process-document.modal.run
✓ App deployed! 🎉

View Deployment: https://modal.com/apps/yourapp
```

**Important:** Copy the webhook URL from the output!

## Step 6: Configure FastAPI to Use Modal

Update your `.env` file:

```bash
# Enable Modal
MODAL_ENABLED=true
MODAL_WEBHOOK_URL=https://yourapp--webhook-process-document.modal.run

# Start with 0% rollout for testing
MODAL_ROLLOUT_PERCENTAGE=0
USE_MODAL_FOR_PROCESSING=false
```

Restart your FastAPI server:
```bash
uvicorn main:app --reload
```

## Step 7: Test End-to-End

### Test 1: Direct Modal Trigger

```bash
# Set 100% Modal mode temporarily
export USE_MODAL_FOR_PROCESSING=true

# Upload a file via your FastAPI app
# Check logs to confirm it routes to Modal
```

### Test 2: Check Modal Dashboard

1. Go to [modal.com/apps](https://modal.com/apps)
2. Click on "fltr-document-processing"
3. View the "Runs" tab
4. You should see your processing job

### Test 3: Verify Results

Check that:
- [ ] Document status updated to "ready" in Postgres
- [ ] Vectors stored in Milvus
- [ ] No errors in Modal logs

## Step 8: Gradual Rollout

Once testing passes, gradually increase traffic:

### Week 1: 10% Rollout
```bash
# .env
MODAL_ROLLOUT_PERCENTAGE=10
USE_MODAL_FOR_PROCESSING=false  # Keep false for gradual rollout
```

Monitor for 2-3 days:
- Check error rates (Modal vs Celery)
- Compare processing times
- Watch costs in Modal dashboard

### Week 2: 25% Rollout
```bash
MODAL_ROLLOUT_PERCENTAGE=25
```

### Week 3: 50% Rollout
```bash
MODAL_ROLLOUT_PERCENTAGE=50
```

### Week 4: 100% Modal
```bash
USE_MODAL_FOR_PROCESSING=true
MODAL_ROLLOUT_PERCENTAGE=100  # Or keep this, flag above overrides
```

## Step 9: Monitor & Optimize

### View Logs
```bash
# Stream logs from Modal
modal app logs fltr-document-processing
```

### Check Costs
Go to [modal.com/usage](https://modal.com/usage) to see:
- CPU seconds used
- Cost per document
- Total monthly spend

### Performance Optimization

If processing is slow:

1. **Increase CPU**: Edit `modal_app.py`
   ```python
   cpu=4.0,  # Up from 2.0
   ```

2. **Increase Memory**:
   ```python
   memory=16384,  # Up from 8192 (16GB)
   ```

3. **Enable Model Caching**: Already configured via volume mount

4. **Batch Processing**: Process multiple documents per call (advanced)

## Step 10: Scale Down Celery (Optional)

Once 100% on Modal with no issues:

1. **Reduce DigitalOcean Droplet Size**
   - Keep 1 lightweight worker for non-processing tasks
   - Scale down from $50/month to $12-18/month

2. **Update Celery Config**
   - Keep Celery for: emails, webhooks, cleanup tasks
   - Remove document processing from Celery entirely

## Troubleshooting

### Issue: "Secret not found"
**Solution:** Make sure you created the secret exactly as named:
```bash
modal secret list  # View all secrets
```

### Issue: Container build fails
**Solution:** Check that `services/processing_core.py` and `services/pdf_validator.py` exist:
```bash
ls -la services/
```

### Issue: Webhook returns 500 error
**Solution:** Check Modal logs:
```bash
modal app logs fltr-document-processing --follow
```

### Issue: High cold start times
**Solution:**
1. Modal caches containers for ~15 minutes of inactivity
2. Enable "keep warm" for production (paid feature)
3. Pre-warm containers with dummy calls

### Issue: Out of memory
**Solution:** Increase memory allocation:
```python
memory=16384,  # 16GB
```

## Rollback Plan

If issues arise:

### Immediate Rollback
```bash
# .env
USE_MODAL_FOR_PROCESSING=false
MODAL_ROLLOUT_PERCENTAGE=0
```

Restart FastAPI - all traffic goes back to Celery immediately.

### Disable Modal Entirely
```bash
MODAL_ENABLED=false
```

## Cost Monitoring

Set up budget alerts:

1. Go to [modal.com/settings/billing](https://modal.com/settings/billing)
2. Set monthly budget limit (e.g., $50)
3. Add alert at 80% of budget

Expected costs for 3000 docs/month:
- **CPU time**: ~$15-20/month (2 vCPU × 30s avg)
- **Storage**: ~$1/month (model cache)
- **Total**: ~$18/month (64% savings vs current $50)

## Support

- **Modal Docs**: [modal.com/docs](https://modal.com/docs)
- **Modal Discord**: Join for support
- **Modal Status**: [status.modal.com](https://status.modal.com)

## Next Steps After Deployment

- [ ] Set up monitoring alerts
- [ ] Configure cost budgets
- [ ] Document Modal URLs in team wiki
- [ ] Train team on Modal dashboard
- [ ] Plan Celery scale-down

---

**Deployment Checklist:**
- [ ] Modal account created
- [ ] Secrets configured
- [ ] Local testing passed
- [ ] Production deployment successful
- [ ] Webhook URL added to .env
- [ ] End-to-end test passed
- [ ] Gradual rollout started
- [ ] Monitoring set up
- [ ] Team notified
