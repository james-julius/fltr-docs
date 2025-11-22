# Vercel Logs Analysis for Subscription Upgrade Issue

## What to Check in Vercel Logs

When reviewing Vercel logs for the subscription upgrade issue, look for these key indicators:

### 1. Check if `getCheckoutSessionParams` is Being Called

**Search for**: `[Checkout] getCheckoutSessionParams called`

**What it means**:
- ✅ **If you see this log**: Better Auth is attempting to create a checkout session. The issue might be in the checkout session creation itself.
- ❌ **If you DON'T see this log**: Better Auth is redirecting to billing portal BEFORE calling `getCheckoutSessionParams`. This means Better Auth is checking for existing subscriptions server-side and redirecting without creating a checkout session.

### 2. Check for Subscription Change Detection

**Search for**: `[Checkout] Subscription change - existing subscription`

**What it means**:
- If you see this log, it means `getCheckoutSessionParams` received the `subscription` parameter, which indicates `subscriptionId` was passed correctly from the client.

### 3. Check Client-Side Logs (Browser Console)

**Search for in browser console**:
- `[SubscriptionButton] Passing subscriptionId to force checkout session: <id>`
- `[SubscriptionButton] Redirecting to Stripe Checkout: <url>`
- `[SubscriptionButton] ERROR: Better Auth returned billing portal URL instead of checkout!`

### 4. Check Network Requests

In browser DevTools → Network tab, look for:
- Request to `/api/auth/subscription/upgrade`
- Check the **request payload** - does it include `subscriptionId`?
- Check the **response** - what URL is returned? Is it `checkout.stripe.com` or `billing.stripe.com`?

## Most Likely Issue

Based on the error message "This subscription cannot be updated because the subscription update feature in the portal configuration is disabled", Better Auth is redirecting to the Stripe billing portal instead of creating a checkout session.

**This suggests**:
1. Better Auth is detecting an existing subscription server-side
2. Better Auth is redirecting to billing portal BEFORE checking if `subscriptionId` was passed
3. OR Better Auth is not respecting the `subscriptionId` parameter

## Potential Solutions

### Solution 1: Verify `subscriptionId` is Being Passed

Check the network request payload in browser DevTools. The request to `/api/auth/subscription/upgrade` should include:
```json
{
  "plan": "pro",
  "successUrl": "/billing",
  "cancelUrl": "/billing",
  "subscriptionId": "sub_xxxxx"  // ← This must be present
}
```

### Solution 2: Check Better Auth Version

Current version: `@better-auth/stripe: ^1.2.12`

Check if there's a newer version that fixes this issue:
```bash
npm outdated @better-auth/stripe
```

### Solution 3: Add Server-Side Logging

Add logging to see what Better Auth is receiving. However, since Better Auth handles this internally, we might need to check the actual Better Auth source code or file a bug report.

### Solution 4: Workaround - Direct Stripe Checkout

If Better Auth continues to redirect to billing portal, we can create checkout sessions directly via Stripe API, bypassing Better Auth's upgrade method entirely.

## Next Steps

1. **Check Vercel Function Logs** for:
   - `[Checkout] getCheckoutSessionParams called` - if missing, Better Auth is redirecting before checkout creation
   - Any errors related to subscription upgrades

2. **Check Browser Console** when user clicks upgrade:
   - Is `subscriptionId` being logged?
   - What URL is returned?

3. **Check Network Tab**:
   - Does the request include `subscriptionId`?
   - What's the response?

4. **If `getCheckoutSessionParams` is NOT being called**:
   - This confirms Better Auth is redirecting to portal before checkout creation
   - We may need to use a workaround or update Better Auth

5. **If `getCheckoutSessionParams` IS being called but still redirects**:
   - The issue is in checkout session creation
   - Check if there's an error in the checkout session params

## Expected Log Flow (Success Case)

1. Client: `[SubscriptionButton] Passing subscriptionId to force checkout session: sub_xxx`
2. Server: `[Checkout] getCheckoutSessionParams called: { hasExistingSubscription: true, ... }`
3. Server: `[Checkout] Subscription change - existing subscription: sub_xxx`
4. Client: `[SubscriptionButton] Redirecting to Stripe Checkout: https://checkout.stripe.com/...`

## Actual Log Flow (Failure Case)

1. Client: `[SubscriptionButton] Passing subscriptionId to force checkout session: sub_xxx`
2. ❌ **Missing**: `[Checkout] getCheckoutSessionParams called` (Better Auth redirected before calling hook)
3. Client: Error or redirect to `billing.stripe.com` with error message

