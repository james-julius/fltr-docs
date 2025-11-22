# Why "Subscription Update Feature Disabled" Error Occurs

## The Misleading Error

The error message **"This subscription cannot be updated because the subscription update feature in the portal configuration is disabled"** is misleading because:

1. **Better Auth is redirecting to billing portal** instead of creating a checkout session
2. **We've intentionally disabled subscription updates in the billing portal** (we use Checkout Sessions instead)
3. **Stripe rejects the redirect** with this error message

## Root Cause

Better Auth has **internal logic** that decides between checkout and billing portal **BEFORE** calling `getCheckoutSessionParams`. Here's what happens:

### Better Auth's Internal Flow (v1.2.12 / v1.3.27)

1. **Client calls** `authClient.subscription.upgrade({ plan, subscriptionId })`
2. **Better Auth server-side checks**:
   - Does user have an existing subscription?
   - Does the plan name match the existing subscription?
   - If YES → Try to redirect to billing portal (for "updates")
   - If NO → Create checkout session (for "new subscriptions" or "plan changes")

3. **Problem**: Even when `subscriptionId` is passed, Better Auth might:
   - Check the database for existing subscriptions
   - Find a matching plan name (case-sensitive or case-insensitive)
   - Decide to use billing portal instead of checkout
   - Redirect to billing portal → Stripe rejects it → Error

### Why This Happens

Better Auth's internal logic seems to be:
- **If plan matches existing subscription** → Use billing portal (for "updating" the same plan)
- **If plan differs** → Use checkout session (for "changing" plans)

But we want **ALL** subscription changes to use checkout sessions, not the billing portal.

## Evidence

From production logs, we see:
- `[SubscriptionButton] Passing subscriptionId to force checkout session: sub_xxx`
- But then: `POST /api/auth/subscription/upgrade 400 (Bad Request)`
- Error: "subscription update feature disabled"

This suggests Better Auth is:
1. Receiving the `subscriptionId` parameter
2. Checking the database for existing subscriptions
3. Finding a matching plan (possibly case-insensitive)
4. Redirecting to billing portal BEFORE calling `getCheckoutSessionParams`
5. Stripe rejects the redirect → Error

## Current Protection Layers

We've added multiple layers of protection:

### 1. Client-Side (subscription-button.tsx, payment-modal.tsx)
- ✅ Checks if plan matches before calling API
- ✅ Only passes `subscriptionId` when switching to different plan
- ✅ Disables button when on current plan

### 2. Server-Side (getCheckoutSessionParams)
- ✅ Validates plan name match (case-insensitive)
- ✅ Throws error if same plan detected
- ⚠️ **BUT**: This only runs if Better Auth decides to create a checkout session

### 3. The Gap

**If Better Auth redirects to billing portal BEFORE calling `getCheckoutSessionParams`, our server-side validation never runs!**

## Solution Options

### Option 1: Check if `getCheckoutSessionParams` is Being Called

**Check Vercel logs** when the error occurs:
- If you see `[Checkout] getCheckoutSessionParams called` → Our validation is running, but Better Auth still redirects
- If you DON'T see it → Better Auth is redirecting before checkout creation

### Option 2: Update Better Auth Version

Current: `@better-auth/stripe: ^1.2.12` (lock file shows `1.3.27` installed)

Check if newer versions fix this:
```bash
npm outdated @better-auth/stripe
```

### Option 3: Force Checkout Session Creation

We might need to ensure Better Auth always creates checkout sessions. Check Better Auth documentation for:
- A flag to force checkout sessions
- A way to disable billing portal redirects
- A hook that runs before the redirect decision

### Option 4: Direct Stripe Checkout (Workaround)

If Better Auth continues to redirect, we can bypass it entirely:

```typescript
// Create checkout session directly via Stripe API
const checkoutSession = await stripe.checkout.sessions.create({
    customer: activeSub.stripeCustomerId,
    mode: "subscription",
    line_items: [{ price: plan.priceId, quantity: 1 }],
    success_url: "/billing",
    cancel_url: "/billing",
    metadata: {
        userId: user.id,
        existingSubscriptionId: activeSub.stripeSubscriptionId,
        changeType: "subscription_change"
    }
})
```

## Next Steps

1. **Check Vercel logs** for `[Checkout] getCheckoutSessionParams called` when error occurs
2. **If NOT called**: Better Auth is redirecting before checkout creation → Need to investigate Better Auth's internal logic
3. **If CALLED**: Our validation runs but Better Auth still redirects → Might be a Better Auth bug
4. **Check Better Auth version** and update if newer version fixes this
5. **Consider filing a bug report** with Better Auth if this is unexpected behavior

## Expected Behavior

According to Better Auth documentation, when `subscriptionId` is passed:
- Better Auth should create a checkout session
- The checkout session should update the existing subscription
- Billing portal should NOT be used

But the actual behavior seems different, suggesting either:
- A bug in Better Auth
- Undocumented behavior
- A configuration issue

## Key Insight

**The error message is misleading** - it's not about Stripe's billing portal configuration. It's about Better Auth's decision to redirect to the billing portal instead of creating a checkout session, even when `subscriptionId` is provided.

