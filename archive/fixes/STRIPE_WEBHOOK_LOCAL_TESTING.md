# Stripe Webhook Local Testing Guide

This guide helps you test Stripe subscription webhooks locally for faster iteration.

## Prerequisites

1. **Install Stripe CLI**:
   ```bash
   # macOS
   brew install stripe/stripe-cli/stripe

   # Or download from https://stripe.com/docs/stripe-cli
   ```

2. **Login to Stripe CLI**:
   ```bash
   stripe login
   ```

3. **Get your test webhook secret** (you'll get this when forwarding):
   ```bash
   stripe listen --forward-to localhost:3000/api/auth/webhook
   ```
   This will output a webhook secret like `whsec_...` - save this!

## Setup

### 1. Environment Variables

Add to your `.env.local` (or `.env`):

```bash
# Stripe Test Mode
NEXT_PUBLIC_STRIPE_TEST_MODE=true

# Stripe Test Keys (get from Stripe Dashboard > Developers > API keys)
STRIPE_SECRET_KEY_TEST=sk_test_...
STRIPE_WEBHOOK_SECRET_TEST=whsec_...  # From `stripe listen` command above

# FastAPI URL (for credit granting)
FASTAPI_URL=http://localhost:8000
```

### 2. Start Your Servers

**Terminal 1 - Next.js:**
```bash
cd nextjs
pnpm dev
```

**Terminal 2 - FastAPI:**
```bash
cd fastapi
# Start your FastAPI server (however you normally do it)
```

**Terminal 3 - Stripe CLI (Webhook Forwarding):**
```bash
stripe listen --forward-to localhost:3000/api/auth/webhook
```

This will:
- Forward webhook events from Stripe to your local Next.js server
- Output a webhook secret (starts with `whsec_`)
- Show all webhook events in real-time

### 3. Test Subscription Flow

1. **Start the webhook listener** (Terminal 3 above)
2. **Copy the webhook secret** from the output
3. **Update your `.env.local`** with `STRIPE_WEBHOOK_SECRET_TEST=whsec_...`
4. **Restart Next.js** to pick up the new webhook secret
5. **Go to billing page** and create a test subscription
6. **Use Stripe test card**: `4242 4242 4242 4242` (any future date, any CVC)
7. **Watch the logs** in all three terminals

## Testing Specific Events

### Test Checkout Completion

```bash
# Trigger a checkout.session.completed event
stripe trigger checkout.session.completed
```

### Test Subscription Renewal

```bash
# Trigger an invoice.paid event (for renewals)
stripe trigger invoice.paid
```

### Test with Real Checkout Session

1. Create a checkout session in your app
2. Complete it with test card `4242 4242 4242 4242`
3. The webhook will fire automatically

## Debugging

### Check Webhook Events

```bash
# List recent events
stripe events list

# Get details of a specific event
stripe events retrieve evt_...

# Resend a webhook event
stripe events resend evt_...
```

### View Logs

**Next.js logs** will show:
- `[Stripe Webhook] Received event: checkout.session.completed`
- `[Stripe Webhook] Processing checkout.session.completed:`
- `[Stripe Webhook] Calling FastAPI to grant credits:`
- `[Stripe Webhook] ✅ Successfully granted X credits`

**FastAPI logs** will show:
- Credit transaction creation
- Any errors in credit granting

### Common Issues

1. **Webhook secret mismatch**: Make sure `STRIPE_WEBHOOK_SECRET_TEST` matches the output from `stripe listen`
2. **Port conflicts**: Make sure port 3000 is available for Next.js
3. **FastAPI not running**: Credits won't be granted if FastAPI is down
4. **Wrong environment**: Make sure you're using test mode (`NEXT_PUBLIC_STRIPE_TEST_MODE=true`)

## Quick Test Script

See `scripts/test-stripe-webhook.sh` for an automated test script.





