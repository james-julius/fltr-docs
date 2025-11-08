# 🔄 Cloudflare Queue Setup - HTTP Pull Mode

## Overview

The FastAPI backend uses **HTTP Pull Mode** to consume messages from Cloudflare Queues. This allows the FastAPI service to poll messages directly from the queue API.

## Architecture

```
R2 Upload Event
    ↓
dataset-upload-notification-worker (Worker)
    ↓
Cloudflare Queue (fltr-document-processing)
    ↓
FastAPI HTTP Pull Consumer (queue_consumer.py)
    ↓
Celery Task (process_document)
```

## Setup Steps

### 1. Enable HTTP Pull Mode on Queue

You need to enable HTTP pull mode for your Cloudflare Queue. **This is required** - without it, you'll get a 405 error when trying to pull messages.

#### Option A: Via Wrangler CLI (Recommended)

```bash
# Add HTTP pull consumer to your queue
wrangler queues consumer http add fltr-document-processing \
  --batch-size 10 \
  --max-retries 3 \
  --message-retries 3 \
  --visibility-timeout 1800

# Verify it's enabled
wrangler queues list
```

#### Option B: Via Cloudflare Dashboard

1. Go to [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. Navigate to **Workers & Pages** > **Queues**
3. Select your queue: `fltr-document-processing`
4. Go to **Settings** tab
5. Under **HTTP Pull**, click **Enable**
6. Configure:
   - **Batch Size**: 10 messages
   - **Visibility Timeout**: 1800 seconds (30 minutes)
   - **Max Retries**: 3

### 2. Get Queue Credentials

You need three values for your `.env` file:

```bash
# Get your account ID
wrangler whoami

# Get queue ID (from wrangler queues list or dashboard)
wrangler queues list
# Look for the queue ID (32-character alphanumeric string)

# Create an API token with Queue edit permissions
# Go to: https://dash.cloudflare.com/profile/api-tokens
# Create token with:
# - Permissions: Account > Queue > Edit
# - Account Resources: Include > Your Account
```

### 3. Configure FastAPI Environment

Add to your `.env` file:

```env
# Cloudflare Queue Configuration
CF_ACCOUNT_ID=your-account-id-here
CF_QUEUE_ID=your-32-char-queue-id-here
CF_API_TOKEN=your-api-token-here
```

The queue consumer will start automatically when FastAPI starts. If credentials are missing, it will log a warning and continue without queue polling.

### 4. Deploy FastAPI

The queue consumer will automatically start when FastAPI starts:

```bash
# Local development
uvicorn main:app --reload

# Production (via your deployment method)
# Queue consumer runs as background task
```

## How It Works

### Consumer Behavior

The FastAPI `queue_consumer.py` service:

1. **Polls every 5 seconds** when queue is empty
2. **Pulls batches of 10 messages** when available
3. **Acknowledges messages** after successful processing
4. **Retries failed messages** by not acknowledging them
5. **Visibility timeout** prevents duplicate processing

### Message Flow

```python
# 1. Pull messages from queue
messages = await consumer.pull_messages(batch_size=10)

# 2. Process each message
for msg in messages:
    event_body = msg.get("body", {})
    object_key = event_body.get("object", {}).get("key", "")

    # 3. Queue Celery task
    task = process_document.delay(
        dataset_id=dataset_id,
        object_key=object_key,
    )

    # 4. Acknowledge message (removes from queue)
    await consumer.ack_message(lease_id)
```

### Error Handling

- **HTTP 405 Error**: Queue doesn't have HTTP pull enabled → Enable it
- **HTTP 401 Error**: Invalid API token → Check CF_API_TOKEN
- **HTTP 404 Error**: Invalid queue ID → Check CF_QUEUE_ID
- **No messages**: Either queue is empty (normal) or credentials are wrong

## Monitoring

### Logs

Check FastAPI logs for consumer status:

```bash
# Successful startup
🚀 Queue consumer started, polling for messages...

# Receiving messages
📥 Received 3 messages from CF Queue
📄 Processing event: object-create for dataset-123/file.pdf
✅ Acknowledged message lease-id-here

# Errors
❌ CF Queue error: [...]
⚠️  Queue consumer disabled (missing CF credentials)
```

### Cloudflare Dashboard

Monitor queue health:
1. Go to Queue in Cloudflare Dashboard
2. Check **Messages** tab for:
   - Total messages
   - Messages in flight
   - Messages delivered

## Troubleshooting

### 405 Method Not Allowed

**Problem**: HTTP pull mode not enabled on queue

**Solution**:
```bash
wrangler queues consumer http add fltr-document-processing
```

### Consumer Not Starting

**Problem**: Missing credentials

**Solution**: Check all three env vars are set:
```bash
echo $CF_ACCOUNT_ID
echo $CF_QUEUE_ID
echo $CF_API_TOKEN
```

### No Messages Being Processed

**Problem**: Queue is in push mode OR worker consumer is active

**Solution**:
- Make sure HTTP pull is enabled (see above)
- Disable the `queue-consumer-worker` if it's deployed:
  ```bash
  # Option 1: Delete the worker
  cd cloudflare-workers/queue-consumer-worker
  wrangler delete

  # Option 2: Remove queue consumer binding from wrangler.toml
  # Remove [[queues.consumers]] section
  ```

### Messages Getting Stuck

**Problem**: Visibility timeout too short or task taking too long

**Solution**: Increase visibility timeout:
```bash
wrangler queues consumer http update fltr-document-processing \
  --visibility-timeout 3600  # 1 hour
```

## Alternative: Worker Push Mode

If you prefer to use a Cloudflare Worker consumer instead:

1. Don't set CF queue credentials in FastAPI (consumer will be disabled)
2. Deploy the `queue-consumer-worker`
3. Worker will push to FastAPI webhook: `/api/v1/webhooks/r2-upload`

See `cloudflare-workers/queue-consumer-worker/` for details.

## Performance

### Pull Mode Advantages
- ✅ Centralized processing logic in FastAPI
- ✅ Easy to debug and monitor
- ✅ Direct access to database and Celery
- ✅ No additional Worker deployment

### Pull Mode Considerations
- ⚠️ Requires FastAPI to be running
- ⚠️ Polling introduces slight delay (5 seconds when idle)
- ⚠️ Uses API calls (quota limits apply)

## API Reference

### Pull Messages Endpoint

```
POST https://api.cloudflare.com/client/v4/accounts/{account_id}/queues/{queue_id}/messages/pull

Headers:
  Authorization: Bearer {api_token}
  Content-Type: application/json

Body:
{
  "batch_size": 10,
  "visibility_timeout_secs": 1800
}
```

### Acknowledge Messages Endpoint

```
POST https://api.cloudflare.com/client/v4/accounts/{account_id}/queues/{queue_id}/messages/ack

Headers:
  Authorization: Bearer {api_token}
  Content-Type: application/json

Body:
{
  "acks": ["lease-id-1", "lease-id-2"]
}
```

## Resources

- [Cloudflare Queues Docs](https://developers.cloudflare.com/queues/)
- [HTTP Pull Consumer](https://developers.cloudflare.com/queues/configuration/pull-consumers/)
- [Queue API Reference](https://developers.cloudflare.com/api/operations/queues-pull-messages)

