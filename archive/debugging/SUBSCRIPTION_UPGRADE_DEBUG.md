# Subscription Upgrade Debugging Guide

## Issue
Users are seeing: "This subscription cannot be updated because the subscription update feature in the portal configuration is disabled" when trying to upgrade subscriptions in production.

## Root Cause Analysis

The error suggests that Better Auth is redirecting to the Stripe billing portal instead of creating a checkout session, even when `subscriptionId` is passed.

## Current Implementation

1. **Client-side (`SubscriptionButton`, `PaymentModal`)**:
   - Passes `subscriptionId` to `authClient.subscription.upgrade()` when `activeSub.stripeSubscriptionId` exists
   - This should force Better Auth to create a checkout session instead of redirecting to billing portal

2. **Server-side (`getCheckoutSessionParams` hook)**:
   - Receives `subscription` parameter when `subscriptionId` is provided
   - Adds metadata to track subscription changes

## Debugging Steps

### 1. Check Browser Console Logs

When a user clicks upgrade, check the browser console for:
- `[SubscriptionButton] Passing subscriptionId to force checkout session: <id>`
- `[SubscriptionButton] Redirecting to Stripe Checkout: <url>`
- OR `[SubscriptionButton] ERROR: Better Auth returned billing portal URL instead of checkout!`

### 2. Verify `activeSub` Data

Check that `activeSub.stripeSubscriptionId` is present:
```javascript
console.log("Active subscription:", activeSub)
console.log("Stripe subscription ID:", activeSub?.stripeSubscriptionId)
```

### 3. Check Network Requests

In browser DevTools → Network tab:
- Look for request to `/api/auth/subscription/upgrade`
- Check the request payload - does it include `subscriptionId`?
- Check the response - what URL is returned?

### 4. Check Server Logs

Look for logs from `getCheckoutSessionParams`:
- `[Checkout] getCheckoutSessionParams called:`
- `hasExistingSubscription: true/false`
- `existingSubscriptionId: <id>`

## Possible Issues

### Issue 1: `activeSub` Not Loaded Correctly
**Symptom**: `activeSub` is `null` or `undefined` when upgrade is clicked
**Solution**: Ensure subscription is fetched before rendering upgrade buttons

### Issue 2: `stripeSubscriptionId` Missing
**Symptom**: `activeSub` exists but `activeSub.stripeSubscriptionId` is missing
**Solution**: Check that Better Auth is returning `stripeSubscriptionId` in subscription list

### Issue 3: Better Auth Server-Side Redirect
**Symptom**: `subscriptionId` is passed but Better Auth still redirects to portal
**Solution**: This might be a Better Auth bug or version issue. Check Better Auth version and consider:
- Updating to latest version
- Filing a bug report with Better Auth
- Using a workaround (see below)

## Workaround: Direct Stripe Checkout Creation

If Better Auth continues to redirect to billing portal, we can create checkout sessions directly:

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

1. **Add production logging** to capture:
   - Whether `subscriptionId` is being passed
   - What URL Better Auth returns
   - Whether it's checkout or billing portal

2. **Check Better Auth version** and update if needed

3. **Verify Stripe configuration** - ensure billing portal subscription updates are disabled (as intended)

4. **Test with different scenarios**:
   - User with active subscription upgrading to different plan
   - User with active subscription upgrading to same plan (should be prevented)
   - User without subscription (should work normally)

## Expected Behavior

When `subscriptionId` is passed to `authClient.subscription.upgrade()`:
- Better Auth should create a Stripe Checkout Session
- The returned URL should be `https://checkout.stripe.com/...`
- User should complete checkout and return to success URL
- Webhook should cancel old subscription and grant credits for new one

## Current Status

- ✅ Client-side code passes `subscriptionId` correctly
- ✅ Server-side hook handles subscription changes
- ✅ Webhook cancels old subscriptions
- ⚠️ Need to verify Better Auth is respecting `subscriptionId` parameter

