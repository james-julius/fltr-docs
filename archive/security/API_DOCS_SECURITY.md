# API Documentation Security

## Overview

The FLTR API uses environment-based configuration to control API documentation exposure:

- **Development**: Interactive docs available at `/docs`, `/redoc`, and `/openapi.json`
- **Production**: All interactive docs disabled for security

## How It Works

### 1. FastAPI Configuration

The app conditionally disables docs in production ([main.py:76-86](fastapi/main.py#L76-L86)):

```python
app = FastAPI(
    title="FLTR API",
    description="Context as a Service - AI-ready dataset platform via MCP",
    version="0.1.0",
    lifespan=lifespan,
    # Security: Disable interactive docs in production
    docs_url="/docs" if settings.ENVIRONMENT != "production" else None,
    redoc_url="/redoc" if settings.ENVIRONMENT != "production" else None,
    openapi_url="/openapi.json" if settings.ENVIRONMENT != "production" else None,
)
```

### 2. Environment Variable

Set the environment in your `.env` file or deployment:

```bash
# Development (default)
ENVIRONMENT=development

# Production
ENVIRONMENT=production
```

### 3. Production Behavior

When `ENVIRONMENT=production`:

- ✅ `/docs` → 404 Not Found
- ✅ `/redoc` → 404 Not Found
- ✅ `/openapi.json` → 404 Not Found
- ✅ API endpoints still work normally
- ✅ Authentication still required for protected routes

## Accessing OpenAPI Spec for Build Process

Even though `/openapi.json` is disabled in production, you can still generate the OpenAPI specification for build processes.

### Method 1: Local File (Recommended for MVP)

The OpenAPI spec is generated as a local file and committed to the repository:

```bash
# Generate OpenAPI spec file
cd fastapi
python scripts/generate_openapi_spec.py --output openapi.json

# Commit the file
git add fastapi/openapi.json
git commit -m "chore: update OpenAPI spec"

# Next.js build will automatically use this file
cd ../nextjs
pnpm generate:api
```

The Next.js generation script (`scripts/generate-api-safe.mjs`) and Orval config (`orval.config.ts`) automatically use the local file at `fastapi/openapi.json`.

**Benefits:**
- ✅ No authentication needed
- ✅ No server access required during builds
- ✅ Works in CI/CD without API availability
- ✅ Faster builds (no network requests)

**When to regenerate:**
- After adding/modifying FastAPI endpoints
- After changing request/response models
- Before committing API changes

### Method 2: Build Script (For FastAPI Build Process)

Use the provided Python generation script:

```bash
cd fastapi
python scripts/generate_openapi_spec.py --output openapi.json
```

This generates the full OpenAPI spec without starting the server.

### Method 3: Programmatic Access

```python
from main import app

# Get OpenAPI schema (works even when docs_url=None)
openapi_schema = app.openapi()

# Use for code generation, documentation, etc.
import json
with open("openapi.json", "w") as f:
    json.dump(openapi_schema, f, indent=2)
```

### Method 3: Development Server

Run the server in development mode:

```bash
ENVIRONMENT=development uvicorn main:app --reload
curl http://localhost:8000/openapi.json > openapi.json
```

## Integration with Build Process

### Next.js Example

Add to your `package.json`:

```json
{
  "scripts": {
    "generate:api": "cd ../fastapi && python scripts/generate_openapi_spec.py --output ../nextjs/openapi.json",
    "build": "npm run generate:api && next build"
  }
}
```

### CI/CD Pipeline Example

```yaml
# .github/workflows/deploy.yml
- name: Generate OpenAPI spec
  run: |
    cd fastapi
    python scripts/generate_openapi_spec.py --output openapi.json

- name: Generate TypeScript client
  run: |
    npx openapi-typescript fastapi/openapi.json -o nextjs/types/api.ts
```

## Security Benefits

### Before (Exposed Docs)
- ❌ API structure visible to attackers
- ❌ Endpoint discovery made easy
- ❌ Parameter validation rules exposed
- ❌ Authentication requirements visible

### After (Environment-Based)
- ✅ Production API structure hidden
- ✅ No endpoint enumeration
- ✅ Development experience unchanged
- ✅ Build process still supported

## Testing

### Verify Development Mode

```bash
ENVIRONMENT=development uvicorn main:app
curl http://localhost:8000/docs  # Should return HTML
curl http://localhost:8000/openapi.json  # Should return JSON
```

### Verify Production Mode

```bash
ENVIRONMENT=production uvicorn main:app
curl http://localhost:8000/docs  # Should return 404
curl http://localhost:8000/openapi.json  # Should return 404
```

### Verify API Still Works

```bash
ENVIRONMENT=production uvicorn main:app
curl http://localhost:8000/health  # Should return {"status": "healthy"}
```

## Frequently Asked Questions

### Q: Can I still use OpenAPI tooling?

**A:** Yes! Use the generation script to create `openapi.json`, then use any OpenAPI tooling:
- `openapi-generator` for client libraries
- `openapi-typescript` for TypeScript types
- `swagger-ui` for custom documentation
- `redoc` for static documentation

### Q: What if I need docs in production for debugging?

**A:** Options:
1. Temporarily set `ENVIRONMENT=development` (not recommended)
2. Create a separate staging environment with docs enabled
3. Use logs and monitoring instead
4. Access the app programmatically: `app.openapi()`

### Q: Does this affect API functionality?

**A:** No. Only the interactive documentation endpoints are affected. All API routes work exactly the same.

### Q: What about staging/testing environments?

**A:** Set `ENVIRONMENT=development` or `ENVIRONMENT=staging` to enable docs in non-production environments.

## Related Security Measures

This is part of a comprehensive security approach:

1. ✅ **Authentication Middleware** - All sensitive endpoints require auth
2. ✅ **Dataset Ownership Verification** - Cross-user access prevention
3. ✅ **Webhook Signature Validation** - Prevent webhook fraud
4. ✅ **Audit Logging** - Complete compliance trail
5. ✅ **API Docs Security** - Production docs disabled ← **You are here**

See [SECURITY_FIXES_IMPLEMENTED.md](SECURITY_FIXES_IMPLEMENTED.md) for full security details.

---

**Implementation Date**: 2025-11-17
**Status**: Production Ready
**Breaking Changes**: None (backward compatible)
