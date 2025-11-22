# Stripe Webhook Credits Fix - 2025-01-20

## Problem
Stripe webhooks were being received in production but credits were not being granted. The error was:

```
[Stripe Webhook] ❌ Error in onSubscriptionComplete: TypeError: fetch failed
Error: connect ECONNREFUSED 127.0.0.1:8000
```

## Root Cause
The Next.js application in production was trying to call `http://localhost:8000` (FastAPI) to grant credits, but FastAPI isn't running on localhost in production. The webhook code was using `process.env.FASTAPI_URL` which was set to `http://localhost:8000` by default.

## Solution
Consolidated all FastAPI URL references to use `NEXT_PUBLIC_API_URL` instead of `FASTAPI_URL` for consistency.

### Files Changed

1. **nextjs/src/lib/auth.ts** (4 occurrences)
   - Changed `process.env.FASTAPI_URL` → `process.env.NEXT_PUBLIC_API_URL`

2. **nextjs/src/lib/payments/webhook-helpers.ts** (1 occurrence)
   - Changed `process.env.FASTAPI_URL` → `process.env.NEXT_PUBLIC_API_URL`

3. **nextjs/src/app/api/proxy/[...path]/route.ts** (1 occurrence)
   - Changed `process.env.FASTAPI_URL` → `process.env.NEXT_PUBLIC_API_URL`

4. **nextjs/.env.example**
   - Removed `FASTAPI_URL=http://localhost:8000`
   - Updated `NEXT_PUBLIC_API_URL` with better documentation

## Deployment Steps

### 1. Update Vercel Environment Variable

**Option A: Using Vercel CLI**
```bash
cd nextjs
vercel env add NEXT_PUBLIC_API_URL production
# When prompted, enter: https://api.tryfltr.com
```

**Option B: Using Vercel Dashboard**
1. Go to Vercel dashboard → Your project → Settings → Environment Variables
2. Add/update `NEXT_PUBLIC_API_URL` = `https://api.tryfltr.com` (Production)
3. Remove old `FASTAPI_URL` if it exists

### 2. Redeploy

```bash
# Push changes to git
git add .
git commit -m "fix: use NEXT_PUBLIC_API_URL for webhook FastAPI calls"
git push

# Or redeploy manually
vercel --prod
```

### 3. Verify

After deployment:
1. Test subscription upgrade flow in production
2. Check Vercel logs for webhook processing
3. Verify credits are granted correctly

Look for these success indicators in logs:
```
[Stripe Webhook] Received event: checkout.session.completed
[Stripe Webhook] Granting X credits to user...
[Stripe Webhook] ✅ Successfully granted X credits to user
```

## Testing in Production

1. Go to https://www.tryfltr.com/billing
2. Upgrade/change subscription plan
3. Complete checkout with test card: `4242 4242 4242 4242`
4. Check Vercel logs to verify:
   - Webhook received
   - FastAPI call successful (status 200)
   - Credits granted

## Additional Notes

- `NEXT_PUBLIC_API_URL` is used for both client-side and server-side API calls
- The variable must be set at build time for Next.js to include it
- For local development, use `http://localhost:8000`
- For production, use `https://api.tryfltr.com`
