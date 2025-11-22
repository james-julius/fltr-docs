# Checkout Redirect Fix - 2025-01-20

## Problem
After completing Stripe checkout, users are being redirected to the homepage `/` instead of `/billing/usage`.

## Root Cause
The subscription upgrade code was passing **relative URLs** (`/billing/usage`) to Better Auth's `subscription.upgrade()` method, but Stripe requires **absolute URLs** (`https://www.tryfltr.com/billing/usage`).

When Better Auth receives relative URLs, it may not properly convert them to absolute URLs before passing them to Stripe, causing Stripe to use its default redirect behavior (homepage).

## How It Works

Better Auth's `subscription.upgrade()` method expects absolute URLs for `successUrl` and `cancelUrl` parameters. These URLs are passed directly to Stripe when creating a checkout session.

**Before (Broken):**
```typescript
const upgradeParams = {
    plan: plan.name,
    successUrl: "/billing/usage",  // ❌ Relative URL
    cancelUrl: "/billing"           // ❌ Relative URL
}
```

**After (Fixed):**
```typescript
const baseUrl = window.location.origin  // e.g., "https://www.tryfltr.com"
const upgradeParams = {
    plan: plan.name,
    successUrl: `${baseUrl}/billing/usage`,  // ✅ Absolute URL
    cancelUrl: `${baseUrl}/billing`          // ✅ Absolute URL
}
```

## Solution

The fix has been applied to both upgrade locations:

1. ✅ **[nextjs/src/app/(auth)/billing/subscription-button.tsx](nextjs/src/app/(auth)/billing/subscription-button.tsx)** - Main subscription upgrade button
2. ✅ **[nextjs/src/components/payment-modal.tsx](nextjs/src/components/payment-modal.tsx)** - Payment modal component

### Deploy the Fix

Simply deploy the code changes:

```bash
git add .
git commit -m "fix: use absolute URLs for Stripe checkout redirects"
git push
```

No environment variable changes needed - the fix uses `window.location.origin` to dynamically construct absolute URLs based on the current domain.

## Verification

After redeployment, test the checkout flow:

1. Go to https://www.tryfltr.com/billing
2. Click upgrade on any plan
3. Complete checkout with test card: `4242 4242 4242 4242`
4. **Expected**: Redirect to `https://www.tryfltr.com/billing/usage`
5. **Before fix**: Redirects to `https://www.tryfltr.com/`

## Important Notes

- Uses `window.location.origin` to dynamically get the current domain
- Works in all environments (local, staging, production) without config changes
- Stripe requires absolute URLs for checkout session redirects
- The fix ensures users are redirected to the correct page after checkout completion

## Code Changes

### subscription-button.tsx (Lines 64-75)
```typescript
// Before
successUrl: "/billing/usage",
cancelUrl: "/billing"

// After
const baseUrl = window.location.origin
successUrl: `${baseUrl}/billing/usage`,
cancelUrl: `${baseUrl}/billing`
```

### payment-modal.tsx (Lines 121-132)
```typescript
// Same fix applied
const baseUrl = window.location.origin
successUrl: `${baseUrl}/billing/usage`,
cancelUrl: `${baseUrl}/billing`
```
