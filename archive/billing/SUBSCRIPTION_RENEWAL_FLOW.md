# Subscription Renewal Flow Documentation

## Overview

This document explains how subscription renewals work in the FLTR system, from Stripe webhooks to credit allocation.

## The Flow

```
Stripe → Better Auth → Next.js Webhook Handler → FastAPI → Credit Service → Database
```

## Detailed Step-by-Step

### 1. Stripe Sends Webhook Event

When a subscription renews monthly, Stripe sends an `invoice.paid` webhook event to:
```
https://www.tryfltr.com/api/auth/stripe/webhook
```

This event includes:
- Invoice details
- Subscription ID
- Customer ID
- Billing reason: `subscription_cycle` (indicates renewal)

### 2. Better Auth Receives Webhook

Better Auth's Stripe plugin handles the webhook at `/api/auth/stripe/webhook` and:
- Verifies the webhook signature using `STRIPE_WEBHOOK_SECRET`
- Parses the event
- Calls the `onEvent` handler defined in [nextjs/src/lib/auth.ts](nextjs/src/lib/auth.ts)

### 3. Next.js Webhook Handler Processes Event

Location: [nextjs/src/lib/auth.ts:335-446](nextjs/src/lib/auth.ts#L335-L446)

**Step 3a: Event Type Check**
```typescript
if (event.type === "invoice.paid") {
    // Process renewal
}
```

**Step 3b: Verify It's a Renewal**
```typescript
if (!isSubscriptionRenewal(invoice)) {
    // Skip if not a renewal (e.g., first payment)
    return
}
```

The `isSubscriptionRenewal()` function checks:
- `invoice.billing_reason === "subscription_cycle"` (monthly renewal)
- Not the initial invoice creation

**Step 3c: Get User and Plan Information**
```typescript
// Extract subscription ID from invoice
const subscriptionId = invoice.subscription

// Retrieve full subscription details from Stripe
const subscription = await stripeClient.subscriptions.retrieve(subscriptionId)

// Get price ID to determine plan
const priceId = subscription.items.data[0]?.price.id

// Find user by subscription ID
const userId = await getUserIdFromStripeSubscriptionId(subscriptionId)

// Get plan configuration
const plan = getPlanFromPriceId(priceId)
const creditsToGrant = plan.limits.creditsPerMonth
```

**Step 3d: Call FastAPI to Grant Credits**
```typescript
const fastApiUrl = process.env.NEXT_PUBLIC_API_URL || "http://localhost:8000"
const response = await fetch(
    `${fastApiUrl}/api/v1/credits/subscription-renewal`,
    {
        method: "POST",
        headers: {
            "Content-Type": "application/json",
            "X-Webhook-Secret": process.env.INTERNAL_WEBHOOK_SECRET || ""
        },
        body: JSON.stringify({
            user_id: userId,
            new_credits: creditsToGrant,
            plan_limit: planLimit,
            meta_data: {
                subscription_id: subscriptionId,
                plan_name: plan.name,
                price_id: priceId,
                invoice_id: invoice.id,
                event_type: "subscription_renewal"
            }
        })
    }
)
```

### 4. FastAPI Credits Endpoint Receives Request

Location: [fastapi/routers/credits.py:484](fastapi/routers/credits.py#L484)

**Step 4a: Validate Request**
```python
@router.post("/subscription-renewal", response_model=CreditTransactionResponse)
async def handle_subscription_renewal(
    request_data: SubscriptionRenewalRequest,
    session: Session = Depends(get_auth_session),
    x_webhook_secret: Optional[str] = Header(None, alias="X-Webhook-Secret"),
):
    # Verify internal webhook secret
    verify_internal_webhook_secret(x_webhook_secret)
```

**Step 4b: Process Renewal**
```python
transaction = credit_service.handle_subscription_renewal(
    user_id=user_id,
    new_credits=new_credits,
    plan_limit=plan_limit,
    metadata={
        "stripe_event_id": stripe_event_id,
        "stripe_event_type": event_type,
        **meta_data
    }
)
```

### 5. Credit Service Handles Renewal Logic

Location: `fastapi/services/credit_service.py`

The `handle_subscription_renewal()` method:

1. **Gets Current Credits**
   ```python
   current_balance = self.get_balance(user_id)
   ```

2. **Calculates Rollover**
   - If user has unused credits from previous month
   - Rolls over up to plan limit (e.g., if limit is 100, max rollover is 100)
   - Example: User has 30 credits left, gets 100 new credits + 30 rollover = 130 total

3. **Applies Plan Limit Cap**
   - Ensures total credits don't exceed 2x plan limit
   - Example: Basic plan (100 credits/month) can have max 200 credits total

4. **Creates Credit Transaction**
   ```python
   transaction = CreditTransaction(
       user_id=user_id,
       amount=final_credits_granted,
       type="subscription_renewal",
       operation="credit",
       transaction_metadata=metadata
   )
   ```

5. **Updates User Balance**
   - Adds new credits to user's account
   - Records transaction in database

### 6. Response Flows Back

```
FastAPI → Next.js → Logs Success
```

The Next.js webhook handler logs:
```
[Stripe Webhook] Successfully granted {creditsToGrant} credits
```

## Event Types Handled

### 1. `checkout.session.completed`
- Triggers when user completes initial subscription checkout
- Grants first month's credits
- Creates subscription record in database

### 2. `invoice.paid` (Renewal)
- Triggers every billing cycle (monthly)
- Grants new credits with rollover logic
- Only processes if `billing_reason === "subscription_cycle"`

### 3. `customer.subscription.updated`
- Triggers when user changes plans (upgrade/downgrade)
- Handled by Better Auth automatically
- May trigger additional credit allocation

## Key Functions

### In Next.js ([nextjs/src/lib/auth.ts](nextjs/src/lib/auth.ts))

- **`isSubscriptionRenewal(invoice)`** - Checks if invoice is a renewal
  - Returns `true` if `billing_reason === "subscription_cycle"`

- **`getUserIdFromStripeSubscriptionId(subscriptionId)`** - Finds user by subscription
  - Queries database for user with matching Stripe subscription ID

- **`getPlanFromPriceId(priceId)`** - Gets plan configuration
  - Maps Stripe price ID to plan details (Basic, Pro, Premium)

### In FastAPI ([fastapi/services/credit_service.py](fastapi/services/credit_service.py))

- **`handle_subscription_renewal()`** - Core renewal logic
  - Calculates rollover
  - Applies limits
  - Creates transaction
  - Updates balance

## Environment Variables Required

### Next.js
```bash
STRIPE_WEBHOOK_SECRET=whsec_...        # Stripe webhook signing secret
NEXT_PUBLIC_API_URL=https://api.tryfltr.com  # FastAPI URL
INTERNAL_WEBHOOK_SECRET=your-secret    # Shared secret with FastAPI
```

### FastAPI
```bash
INTERNAL_WEBHOOK_SECRET=your-secret    # Must match Next.js
DATABASE_URL=postgresql://...          # Database connection
```

## Security

1. **Stripe Webhook Verification**
   - Better Auth verifies webhook signature
   - Ensures event came from Stripe

2. **Internal Webhook Secret**
   - Next.js → FastAPI calls require `X-Webhook-Secret` header
   - Prevents unauthorized credit allocation

3. **Database Transactions**
   - All credit operations are atomic
   - Ensures consistency even if webhook is retried

## Error Handling

### Stripe Retries
- If webhook handler fails, Stripe retries automatically
- Idempotency handled by transaction IDs
- Prevents double-credit allocation

### Logging
All steps log extensively:
```
[Stripe Webhook] Received event: invoice.paid
[Stripe Webhook] Granting 100 credits to user abc123
[Stripe Webhook] Successfully granted 100 credits
```

## Testing

### Test Monthly Renewal Locally
```bash
# Terminal 1: Start Stripe webhook listener
stripe listen --forward-to localhost:3000/api/auth/stripe/webhook

# Terminal 2: Trigger renewal event
stripe trigger invoice.paid
```

### Production Testing
- Use Stripe test mode
- Complete actual checkout with test card
- Wait for renewal or trigger manually in Stripe dashboard

## Credit Rollover Example

**Scenario:** User has Basic plan (100 credits/month)

**Month 1:**
- Start: 0 credits
- Granted: 100 credits
- Used: 70 credits
- End: 30 credits remaining

**Month 2 (Renewal):**
- Previous balance: 30 credits
- New credits: 100 credits
- Rollover: 30 credits (under limit)
- Total: 130 credits ✅

**Month 3 (Another Renewal):**
- Previous balance: 150 credits (if unused)
- New credits: 100 credits
- Rollover: 100 credits (capped at plan limit)
- Total: 200 credits (capped at 2x plan limit) ✅

## Troubleshooting

### Credits Not Granted on Renewal

1. **Check Webhook Logs** (Vercel/Next.js logs)
   ```
   [Stripe Webhook] Received event: invoice.paid
   ```

2. **Check if Renewal was Detected**
   ```
   [Stripe Webhook] Skipping invoice - billing_reason is ...
   ```

3. **Check FastAPI Call**
   ```
   [Stripe Webhook] Calling FastAPI to grant credits
   [Stripe Webhook] Successfully granted ... credits
   ```

4. **Check FastAPI Logs**
   ```
   💰 [Credits] Processing subscription renewal for user ...
   💰 [Credits] ✅ Successfully granted ... credits
   ```

### Common Issues

**Issue:** `ECONNREFUSED` error
- **Cause:** `NEXT_PUBLIC_API_URL` not set or incorrect
- **Fix:** Set to production FastAPI URL

**Issue:** `401 Unauthorized` from FastAPI
- **Cause:** `INTERNAL_WEBHOOK_SECRET` mismatch
- **Fix:** Ensure both Next.js and FastAPI have same secret

**Issue:** Credits granted on first payment but not renewal
- **Cause:** `isSubscriptionRenewal()` check failing
- **Fix:** Check invoice `billing_reason` in logs

## Related Files

- [nextjs/src/lib/auth.ts](nextjs/src/lib/auth.ts) - Webhook handlers
- [nextjs/src/lib/payments/webhook-helpers.ts](nextjs/src/lib/payments/webhook-helpers.ts) - Helper functions
- [nextjs/src/lib/payments/plans.ts](nextjs/src/lib/payments/plans.ts) - Plan configurations
- [fastapi/routers/credits.py](fastapi/routers/credits.py) - Credits endpoint
- [fastapi/services/credit_service.py](fastapi/services/credit_service.py) - Credit business logic
