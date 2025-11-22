# Vercel Build Fix - API Client Generation

## Problem Summary

Your Vercel builds are failing because the Next.js build tries to generate the TypeScript API client from FastAPI's `/openapi.json` endpoint, but FastAPI isn't running during the build:

```
⚠️  ResolverError: Error downloading https://fltr-api.geniedi.com/openapi.json: HTTP ERROR 503
Error: One or more project failed, see above for details
```

## ✅ Solution Implemented

I've implemented a **safe API generation system** that fixes this issue:

### Changes Made:

1. **Created safe generation script** ([nextjs/scripts/generate-api-safe.mjs](nextjs/scripts/generate-api-safe.mjs))
   - Checks if FastAPI is available before generating
   - Falls back to existing files if API is down
   - Won't fail builds in CI

2. **Updated `.gitignore`** ([nextjs/.gitignore](nextjs/.gitignore))
   - Generated API client files are now **committed to git**
   - Vercel builds can use these files instead of regenerating

3. **Updated build script** ([nextjs/package.json](nextjs/package.json))
   - Changed from: `pnpm run generate:api && next build`
   - Changed to: `node scripts/generate-api-safe.mjs && next build`

### Documentation Created:

- [nextjs/API_CLIENT_GENERATION.md](nextjs/API_CLIENT_GENERATION.md) - Full documentation

## 🚀 What You Need to Do

### Step 1: Commit the Generated API Client Files

The generated files already exist locally but aren't in git. You need to commit them:

```bash
cd nextjs

# Check what files will be added
git status src/lib/api/generated/

# Add all generated files
git add src/lib/api/generated/

# Commit them
git commit -m "fix: commit generated API client for Vercel builds"
```

### Step 2: Push to Trigger Vercel Build

```bash
git push origin main
```

### Step 3: Verify Vercel Build

Go to Vercel dashboard and watch the build logs. You should see:

```
=== Safe API Client Generation ===
✓ Generated API client files already exist
Checking API availability at https://fltr-api.geniedi.com/openapi.json...
✗ API unavailable: [connection error]
✓ Using existing generated files (API unavailable)
This is normal during CI builds where FastAPI is not running.
```

Then the Next.js build should proceed successfully! ✅

## 🔧 Going Forward

### When FastAPI Schema Changes

After updating FastAPI endpoints or models:

```bash
cd nextjs

# Make sure FastAPI is running locally
cd ../fastapi && python main.py &

# Regenerate API client
pnpm generate:api

# Commit the changes
git add src/lib/api/generated/
git commit -m "chore: regenerate API client after FastAPI changes"
git push
```

### Available Scripts

```bash
# Normal generation (requires FastAPI running)
pnpm generate:api

# Safe generation (won't fail if API is down)
pnpm generate:api:safe

# Auto-regenerate on FastAPI changes
pnpm api:watch
```

## 📊 How It Works

### Before (Broken):
```
Vercel Build
    ↓
Run: pnpm generate:api
    ↓
Fetch: https://fltr-api.geniedi.com/openapi.json
    ↓
❌ 503 Service Unavailable
    ↓
❌ Build Fails
```

### After (Fixed):
```
Vercel Build
    ↓
Run: node scripts/generate-api-safe.mjs
    ↓
Check: Is FastAPI available?
    ↓
No → Check: Do generated files exist in git?
    ↓
Yes → ✅ Use existing files
    ↓
✅ Continue Next.js build
```

## 🔍 Verification Checklist

Before pushing, verify:

- [ ] Generated files exist locally:
  ```bash
  ls -la nextjs/src/lib/api/generated/
  ```

- [ ] Files are NOT in `.gitignore`:
  ```bash
  grep "generated" nextjs/.gitignore
  # Should see commented out line: # /src/lib/api/generated/**/*
  ```

- [ ] Build script updated:
  ```bash
  grep "build.*generate" nextjs/package.json
  # Should see: "build": "node scripts/generate-api-safe.mjs && next build"
  ```

- [ ] Script exists and is executable:
  ```bash
  ls -la nextjs/scripts/generate-api-safe.mjs
  ```

- [ ] Test locally:
  ```bash
  cd nextjs
  pnpm build
  # Should succeed whether FastAPI is running or not
  ```

## 🆘 If Build Still Fails

1. **Check Vercel logs** for the exact error
2. **Verify files are committed**:
   ```bash
   git ls-files nextjs/src/lib/api/generated/
   # Should show multiple files
   ```
3. **Check file permissions**:
   ```bash
   ls -la nextjs/scripts/generate-api-safe.mjs
   # Should show -rwxr-xr-x (executable)
   ```
4. **Test build locally**:
   ```bash
   cd nextjs
   rm -rf .next
   pnpm build
   ```

## 📝 Why This Approach?

We chose to **commit generated files** because:

- ✅ **Simple**: No extra infrastructure needed
- ✅ **Reliable**: Builds don't depend on external services
- ✅ **Works offline**: Developers can build without FastAPI
- ✅ **Git-tracked**: API changes are visible in diffs
- ✅ **Fast**: No generation time in CI

Alternative approaches (for later):
- Upload OpenAPI spec to CDN
- Publish API client as npm package
- Generate in FastAPI CI and artifact

For now, committing files is the most pragmatic solution.

## 🎯 Next Steps

1. ✅ Commit generated files (see Step 1 above)
2. ✅ Push to trigger Vercel build
3. ✅ Verify build succeeds
4. ✅ Update your workflow to regenerate after FastAPI changes

---

**Questions?** See [nextjs/API_CLIENT_GENERATION.md](nextjs/API_CLIENT_GENERATION.md) for full documentation.






