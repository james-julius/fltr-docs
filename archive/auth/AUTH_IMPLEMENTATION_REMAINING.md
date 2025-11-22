# Authentication, Rate Limiting & MCP OAuth - Remaining Implementation
**FLTR - Complete Security & Credit System**

**Status:** Implementation Roadmap
**Created:** November 5, 2025
**Last Updated:** November 5, 2025

---

## Executive Summary

### What We Have ✅
- ✅ **Better Auth Integration** - Session validation working
- ✅ **API Key Authentication** - Working alongside sessions
- ✅ **Credit System** - Full implementation with deduction/refunds
- ✅ **MCP Endpoints Protected** - Auth required for queries
- ✅ **Redis Configured** - Ready for rate limiting

### What We Need ❌
- ❌ **OAuth Discovery Endpoints** - For AI tools (Claude Desktop, VS Code)
- ❌ **Rate Limiting Middleware** - Prevent abuse/DDoS
- ❌ **MCP-Compliant 401 Responses** - Include discovery metadata
- ❌ **Tests** - OAuth flow and rate limiting tests
- ❌ **Documentation** - User guide for authentication

### Estimated Effort
- **OAuth Discovery:** 4-6 hours
- **Rate Limiting:** 6-8 hours
- **Testing:** 4-6 hours
- **Documentation:** 2-3 hours
- **Total:** ~2-3 days

---

## Current State Analysis

### Authentication Flow (Working)

```
┌─────────────────────────────────────────────────────────────┐
│                    CURRENT STATE                             │
├─────────────────┬─────────────────────┬────────────────────┤
│   Web Browser   │   API Clients       │   MCP Tools        │
│   (Next.js)     │   (Developers)      │   (Manual Config)  │
└────────┬────────┴──────────┬──────────┴─────────┬──────────┘
         │                   │                     │
         │ Session Token     │ API Key             │ API Key
         │ (Better Auth)     │ (X-API-Key)         │ (X-API-Key)
         │                   │                     │
         └───────────────────┼─────────────────────┘
                             │
                             │ ⚠️ NO RATE LIMITING
                             │
                    ┌────────▼────────┐
                    │  Auth Middleware│
                    │  ✅ Working     │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Credit Gate    │
                    │  ✅ Working     │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
         ┌────▼────┐    ┌───▼────┐    ┌───▼────┐
         │ Dataset │    │  MCP   │    │Embedding│
         │ Router  │    │ Router │    │ Router  │
         └─────────┘    └────────┘    └─────────┘
```

### What's Missing

```
┌─────────────────────────────────────────────────────────────┐
│                    TARGET STATE                              │
├─────────────────┬─────────────────────┬────────────────────┤
│   Web Browser   │   API Clients       │   AI Tools         │
│   (Next.js)     │   (Developers)      │   (Claude, VS Code)│
└────────┬────────┴──────────┬──────────┴─────────┬──────────┘
         │                   │                     │
         │ Session Token     │ API Key             │ OAuth 2.1
         │ (Better Auth)     │ (X-API-Key)         │ (Auto-discover)
         │                   │                     │
         └───────────────────┼─────────────────────┘
                             │
                             │ 🆕 DISCOVER OAUTH
                             │    /.well-known/
                             │
                    ┌────────▼────────┐
                    │  Rate Limiter   │  🆕 NEW
                    │  (Redis-backed) │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Auth Middleware│
                    │  ✅ Working     │
                    │  + OAuth header │  🆕 ENHANCE
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Credit Gate    │
                    │  ✅ Working     │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
         ┌────▼────┐    ┌───▼────┐    ┌───▼────┐
         │ Dataset │    │  MCP   │    │Embedding│
         │ Router  │    │ Router │    │ Router  │
         └─────────┘    └────────┘    └─────────┘
```

---

## Implementation Tasks

### TASK 1: OAuth Discovery Endpoints (Priority: HIGH)
**Goal:** Enable AI tools to auto-discover OAuth server
**Estimated Time:** 4-6 hours
**Files to Create:** 1 new file
**Files to Modify:** 2 existing files

#### 1.1 Create MCP Auth Metadata Router

**New File:** `fastapi/routers/mcp_auth_metadata.py`

```python
"""
MCP OAuth Discovery Endpoints

Provides metadata for MCP clients to discover authorization server
and required scopes. Compliant with RFC 9728 (Protected Resource Metadata).
"""

from fastapi import APIRouter
from config import settings

router = APIRouter(tags=["mcp-auth"])


@router.get("/.well-known/oauth-protected-resource")
async def protected_resource_metadata():
    """
    Protected Resource Metadata for MCP Server

    Per RFC 9728, this endpoint tells MCP clients:
    1. What resource this is (your MCP server)
    2. Where to find the authorization server (Better Auth)
    3. What scopes are supported

    Called by: Claude Desktop, VS Code MCP extensions
    """
    return {
        "resource": settings.API_BASE_URL,
        "authorization_servers": [
            settings.AUTH_SERVER_URL
        ],
        "scopes_supported": [
            "mcp:query",      # Query MCP endpoints
            "mcp:tools",      # Access all MCP tools
            "mcp:create",     # Create MCP endpoints
        ],
        "bearer_methods_supported": ["header"],
        "resource_documentation": f"{settings.API_BASE_URL}/docs",
    }


@router.get("/.well-known/oauth-authorization-server")
async def authorization_server_metadata():
    """
    OAuth Authorization Server Metadata

    Points to Better Auth's OAuth endpoints.
    Better Auth already provides OAuth 2.1 - we just surface the metadata.
    """
    return {
        "issuer": settings.AUTH_SERVER_URL,
        "authorization_endpoint": f"{settings.AUTH_SERVER_URL}/authorize",
        "token_endpoint": f"{settings.AUTH_SERVER_URL}/token",
        "response_types_supported": ["code"],
        "grant_types_supported": ["authorization_code", "refresh_token"],
        "scopes_supported": [
            "openid",
            "profile",
            "email",
            "mcp:query",
            "mcp:tools",
            "mcp:create"
        ],
        "code_challenge_methods_supported": ["S256"],  # PKCE required
    }
```

**Why this matters:**
- When Claude Desktop connects to your MCP server, it sends a request
- Your server returns 401 with a `WWW-Authenticate` header pointing to `/.well-known/oauth-protected-resource`
- Claude fetches that metadata, discovers Better Auth endpoints
- User sees OAuth consent screen in browser
- Token automatically used for all future requests

---

#### 1.2 Update Config with OAuth URLs

**File:** `fastapi/config.py`

**Add these settings:**

```python
class Settings(BaseSettings):
    # ... existing settings ...

    # OAuth Configuration for MCP
    API_BASE_URL: str = "https://api.fltr.com"  # Your FastAPI URL
    AUTH_SERVER_URL: str = "https://fltr.com/api/auth"  # Your Next.js Better Auth URL

    # Or load from environment:
    # API_BASE_URL: str = Field(..., env="API_BASE_URL")
    # AUTH_SERVER_URL: str = Field(..., env="AUTH_SERVER_URL")
```

**Environment Variables to Add:**

```bash
# .env
API_BASE_URL=http://localhost:8000  # Development
AUTH_SERVER_URL=http://localhost:3000/api/auth  # Development

# Production:
# API_BASE_URL=https://api.fltr.com
# AUTH_SERVER_URL=https://fltr.com/api/auth
```

---

#### 1.3 Register Router in Main App

**File:** `fastapi/main.py`

**Add these lines:**

```python
# Import the new router
from routers import mcp_auth_metadata

# Register it (order doesn't matter, but put near other routers)
app.include_router(mcp_auth_metadata.router)
```

**Note:** The router uses `/.well-known/` paths without the `/api/v1` prefix, so DON'T add `prefix="/api/v1"` when including it.

---

#### 1.4 Enhance 401 Responses with Discovery Metadata

**File:** `fastapi/middleware/auth.py`

**Current 401 response (line 126-144):**

```python
if not auth_info:
    headers = {"WWW-Authenticate": "Bearer, ApiKey"}
    # ... CORS headers ...
    return JSONResponse(
        status_code=status.HTTP_401_UNAUTHORIZED,
        content={"detail": "Authentication required..."},
        headers=headers
    )
```

**Enhanced version with MCP discovery:**

```python
if not auth_info:
    # Build WWW-Authenticate header with OAuth discovery
    from config import settings

    www_authenticate = (
        f'Bearer realm="mcp", '
        f'resource_metadata="{settings.API_BASE_URL}/.well-known/oauth-protected-resource"'
    )

    headers = {"WWW-Authenticate": www_authenticate}

    # ... CORS headers (keep existing code) ...

    return JSONResponse(
        status_code=status.HTTP_401_UNAUTHORIZED,
        content={
            "detail": "Authentication required. Provide either X-API-Key or Authorization Bearer token.",
            "hint": "Use X-API-Key header for API keys, or Authorization: Bearer <token> for session auth",
            "oauth_discovery": f"{settings.API_BASE_URL}/.well-known/oauth-protected-resource"
        },
        headers=headers
    )
```

**Why this matters:**
- MCP clients parse the `WWW-Authenticate` header
- They automatically fetch the `resource_metadata` URL
- This triggers the OAuth discovery flow without manual configuration

---

#### 1.5 Testing OAuth Discovery

**Manual Testing:**

```bash
# Test 1: Check metadata endpoint
curl http://localhost:8000/.well-known/oauth-protected-resource | jq

# Expected output:
# {
#   "resource": "http://localhost:8000",
#   "authorization_servers": ["http://localhost:3000/api/auth"],
#   "scopes_supported": ["mcp:query", "mcp:tools", "mcp:create"],
#   ...
# }

# Test 2: Check 401 includes discovery
curl -i http://localhost:8000/api/v1/mcp/query/some-id/endpoint

# Expected headers:
# HTTP/1.1 401 Unauthorized
# WWW-Authenticate: Bearer realm="mcp", resource_metadata="..."
```

**Deliverables:**
- [ ] Create `routers/mcp_auth_metadata.py`
- [ ] Add `API_BASE_URL` and `AUTH_SERVER_URL` to `config.py`
- [ ] Update `.env` and `.env.example` with new variables
- [ ] Register router in `main.py`
- [ ] Enhance 401 responses in `middleware/auth.py`
- [ ] Test both metadata endpoints return valid JSON
- [ ] Test 401 responses include `WWW-Authenticate` header with discovery URL

---

### TASK 2: Rate Limiting Middleware (Priority: CRITICAL)
**Goal:** Prevent API abuse with Redis-backed rate limiting
**Estimated Time:** 6-8 hours
**Files to Create:** 1 new file
**Files to Modify:** 2 existing files

#### 2.1 Why Rate Limiting is Critical

**Current Risk:** Your API has NO rate limiting, meaning:
- ❌ A single user can make unlimited requests
- ❌ Runaway scripts can drain credits instantly
- ❌ DDoS attacks can overwhelm the server
- ❌ OpenAI API costs can explode from excessive embedding calls
- ❌ Milvus can be overloaded with search queries

**With Rate Limiting:**
- ✅ Anonymous users: 50 requests/hour (discovery only)
- ✅ API key users: 1,000 requests/hour
- ✅ Session users: 5,000 requests/hour
- ✅ Endpoint-specific limits (batch queries more restrictive)
- ✅ Graceful 429 responses with retry information

---

#### 2.2 Create Rate Limiting Middleware

**New File:** `fastapi/middleware/rate_limit.py`

```python
"""
Rate Limiting Middleware for FLTR API

Uses Redis with sliding window algorithm for distributed rate limiting.
Tiered limits based on authentication method.
"""

from typing import Optional
from fastapi import Request, HTTPException
from fastapi.responses import JSONResponse
from datetime import datetime, timedelta
import redis.asyncio as redis
from config import settings
from middleware.auth import get_current_auth


# Redis connection pool
_redis_client: Optional[redis.Redis] = None


async def get_redis() -> redis.Redis:
    """Get or create Redis connection"""
    global _redis_client
    if _redis_client is None:
        _redis_client = redis.from_url(
            settings.REDIS_URL,
            encoding="utf-8",
            decode_responses=True
        )
    return _redis_client


class RateLimitTier:
    """Rate limit configurations per tier"""

    # Unauthenticated (IP-based)
    ANONYMOUS = {
        "requests": 50,
        "window": 3600,  # 1 hour
    }

    # API Key authenticated
    API_KEY = {
        "requests": 1000,
        "window": 3600,
    }

    # Session/OAuth authenticated
    SESSION = {
        "requests": 5000,
        "window": 3600,
    }

    # Per-endpoint overrides (more restrictive)
    ENDPOINTS = {
        "/api/v1/mcp/batch-query": {
            "requests": 100,
            "window": 3600,
        },
        "/api/v1/embeddings/batch": {
            "requests": 200,
            "window": 3600,
        },
    }


async def check_rate_limit(
    request: Request,
    limit: int,
    window: int,
) -> dict:
    """
    Check if request is within rate limit using sliding window

    Returns:
        dict with rate limit info for headers

    Raises:
        HTTPException: If rate limit exceeded
    """
    try:
        redis_client = await get_redis()
    except Exception as e:
        # If Redis is down, log warning and allow request
        print(f"WARNING: Redis unavailable for rate limiting: {e}")
        return {
            "limit": limit,
            "remaining": limit,
            "reset": int((datetime.utcnow() + timedelta(seconds=window)).timestamp())
        }

    now = datetime.utcnow()
    window_start = now - timedelta(seconds=window)

    # Create unique key for this user/IP
    identifier = get_rate_limit_identifier(request)
    rate_limit_key = f"rate_limit:{request.url.path}:{identifier}"

    # Remove old entries outside window
    await redis_client.zremrangebyscore(
        rate_limit_key,
        0,
        window_start.timestamp()
    )

    # Count requests in current window
    current_count = await redis_client.zcard(rate_limit_key)

    if current_count >= limit:
        # Rate limit exceeded
        oldest = await redis_client.zrange(rate_limit_key, 0, 0, withscores=True)
        if oldest:
            retry_after = int(window - (now.timestamp() - oldest[0][1]))
        else:
            retry_after = window

        raise HTTPException(
            status_code=429,
            detail=f"Rate limit exceeded. Try again in {retry_after} seconds.",
            headers={
                "X-RateLimit-Limit": str(limit),
                "X-RateLimit-Remaining": "0",
                "X-RateLimit-Reset": str(int(oldest[0][1] + window if oldest else now.timestamp() + window)),
                "Retry-After": str(retry_after)
            }
        )

    # Add current request to window
    await redis_client.zadd(
        rate_limit_key,
        {str(now.timestamp()): now.timestamp()}
    )

    # Set expiry on key (cleanup)
    await redis_client.expire(rate_limit_key, window)

    # Return headers info
    return {
        "limit": limit,
        "remaining": limit - current_count - 1,
        "reset": int(now.timestamp() + window)
    }


def get_rate_limit_identifier(request: Request) -> str:
    """
    Get unique identifier for rate limiting

    Priority:
    1. User ID (if session auth)
    2. API key hash (if API key auth)
    3. IP address (if unauthenticated)
    """
    auth = get_current_auth(request)

    if auth:
        if auth['type'] == 'session':
            return f"user:{auth['user']['id']}"
        elif auth['type'] == 'api_key':
            # Hash API key for privacy
            import hashlib
            hashed = hashlib.sha256(auth['api_key'].encode()).hexdigest()[:16]
            return f"apikey:{hashed}"

    # Fallback to IP address
    forwarded = request.headers.get("X-Forwarded-For")
    if forwarded:
        return f"ip:{forwarded.split(',')[0].strip()}"

    client_host = request.client.host if request.client else "unknown"
    return f"ip:{client_host}"


async def rate_limit_middleware(request: Request, call_next):
    """
    Middleware to enforce rate limits before processing request
    """
    # Skip rate limiting for health checks and docs
    if request.url.path in ['/', '/health', '/docs', '/openapi.json']:
        return await call_next(request)

    # Determine rate limit tier
    auth = get_current_auth(request)

    if auth:
        if auth['type'] == 'session':
            tier = RateLimitTier.SESSION
        else:  # api_key
            tier = RateLimitTier.API_KEY
    else:
        tier = RateLimitTier.ANONYMOUS

    # Check for endpoint-specific limits
    endpoint_limit = None
    for pattern, limit_config in RateLimitTier.ENDPOINTS.items():
        if request.url.path.startswith(pattern):
            endpoint_limit = limit_config
            break

    # Use endpoint limit if specified, otherwise use tier limit
    if endpoint_limit:
        limit = endpoint_limit["requests"]
        window = endpoint_limit["window"]
    else:
        limit = tier["requests"]
        window = tier["window"]

    # Check rate limit
    try:
        rate_info = await check_rate_limit(request, limit, window)
    except HTTPException as e:
        # Rate limit exceeded - return 429
        return JSONResponse(
            status_code=e.status_code,
            content={"detail": e.detail},
            headers=e.headers
        )

    # Process request
    response = await call_next(request)

    # Add rate limit headers to successful response
    response.headers["X-RateLimit-Limit"] = str(rate_info["limit"])
    response.headers["X-RateLimit-Remaining"] = str(rate_info["remaining"])
    response.headers["X-RateLimit-Reset"] = str(rate_info["reset"])

    return response
```

---

#### 2.3 Register Rate Limiting in Main App

**File:** `fastapi/main.py`

**Add rate limiting middleware BEFORE auth middleware (order matters!):**

```python
from middleware.rate_limit import rate_limit_middleware
from middleware.auth import AuthMiddleware

# ... existing middleware ...

# Add rate limiting FIRST (runs before auth)
@app.middleware("http")
async def add_rate_limiting(request: Request, call_next):
    return await rate_limit_middleware(request, call_next)

# Then auth middleware (existing)
app.add_middleware(AuthMiddleware)
```

**Why order matters:**
- Rate limiting should run FIRST to block abusive requests early
- Auth middleware runs SECOND to validate credentials
- Credit gate runs THIRD (via decorator on specific endpoints)

---

#### 2.4 Environment Variables

**File:** `.env`

```bash
# Redis Configuration
REDIS_URL=redis://localhost:6379/0

# Production:
# REDIS_URL=redis://your-redis-host:6379/0
```

**Note:** Redis is already configured in `config.py:94`, so you just need to ensure the environment variable is set.

---

#### 2.5 Testing Rate Limiting

**Manual Testing:**

```bash
# Test 1: Anonymous user hits limit
for i in {1..51}; do
  echo "Request $i"
  curl -i http://localhost:8000/api/v1/mcp/datasets
done

# First 50 should succeed with X-RateLimit-* headers
# 51st should return 429 with Retry-After header

# Test 2: API key user has higher limit
for i in {1..10}; do
  curl -H "X-API-Key: your-key" \
       http://localhost:8000/api/v1/mcp/datasets
done

# Should see X-RateLimit-Limit: 1000 (not 50)

# Test 3: Check rate limit headers
curl -i -H "X-API-Key: your-key" \
     http://localhost:8000/api/v1/mcp/datasets

# Expected headers:
# X-RateLimit-Limit: 1000
# X-RateLimit-Remaining: 999
# X-RateLimit-Reset: 1730899200
```

**Deliverables:**
- [ ] Create `middleware/rate_limit.py`
- [ ] Register rate limiting in `main.py` (BEFORE auth middleware)
- [ ] Ensure `REDIS_URL` is in `.env`
- [ ] Start Redis: `redis-server` (or use Docker)
- [ ] Test anonymous rate limits (50/hour)
- [ ] Test API key rate limits (1000/hour)
- [ ] Test session rate limits (5000/hour)
- [ ] Verify 429 responses include `Retry-After` header
- [ ] Verify all responses include `X-RateLimit-*` headers

---

### TASK 3: Testing (Priority: MEDIUM)
**Goal:** Comprehensive tests for OAuth discovery and rate limiting
**Estimated Time:** 4-6 hours
**Files to Create:** 2 new test files

#### 3.1 OAuth Discovery Tests

**New File:** `fastapi/tests/test_oauth_discovery.py`

```python
"""
Tests for MCP OAuth Discovery Endpoints
"""

import pytest
from fastapi.testclient import TestClient


class TestOAuthDiscovery:
    """Test OAuth metadata discovery endpoints"""

    def test_protected_resource_metadata(self, client: TestClient):
        """MCP clients can discover authorization server"""
        response = client.get("/.well-known/oauth-protected-resource")
        assert response.status_code == 200

        data = response.json()
        assert "resource" in data
        assert "authorization_servers" in data
        assert "scopes_supported" in data
        assert "mcp:query" in data["scopes_supported"]
        assert "mcp:tools" in data["scopes_supported"]

    def test_authorization_server_metadata(self, client: TestClient):
        """OAuth server metadata is accessible"""
        response = client.get("/.well-known/oauth-authorization-server")
        assert response.status_code == 200

        data = response.json()
        assert "authorization_endpoint" in data
        assert "token_endpoint" in data
        assert "response_types_supported" in data
        assert "code" in data["response_types_supported"]

    def test_401_includes_discovery_metadata(self, client: TestClient):
        """Unauthenticated MCP request returns 401 with discovery info"""
        response = client.get("/api/v1/mcp/query/some-dataset-id/endpoint")
        assert response.status_code == 401
        assert "WWW-Authenticate" in response.headers

        www_auth = response.headers["WWW-Authenticate"]
        assert "Bearer" in www_auth
        assert "resource_metadata" in www_auth
        assert ".well-known/oauth-protected-resource" in www_auth


class TestMCPWithAuth:
    """Test MCP endpoints work with different auth methods"""

    def test_mcp_datasets_is_public(self, client: TestClient):
        """Discovery endpoint remains public"""
        response = client.get("/api/v1/mcp/datasets")
        assert response.status_code == 200

    def test_mcp_manifest_is_public(self, client: TestClient, sample_dataset):
        """Manifest endpoint is public"""
        response = client.get(f"/api/v1/mcp/manifest/{sample_dataset.id}")
        assert response.status_code in [200, 404]  # 404 if no manifest yet
        assert response.status_code != 401

    def test_mcp_query_requires_auth(self, client: TestClient):
        """Query endpoint requires authentication"""
        response = client.get("/api/v1/mcp/query/some-id/endpoint?query=test")
        assert response.status_code == 401

    def test_mcp_query_with_api_key(self, client: TestClient, auth_headers, sample_dataset):
        """API key auth works for MCP queries"""
        response = client.get(
            f"/api/v1/mcp/query/{sample_dataset.id}/test-endpoint?query=test",
            headers=auth_headers
        )
        # Should not return 401 (may return 404 if endpoint doesn't exist, or 402 if no credits)
        assert response.status_code != 401
```

---

#### 3.2 Rate Limiting Tests

**New File:** `fastapi/tests/test_rate_limit.py`

```python
"""
Tests for Rate Limiting Middleware
"""

import pytest
from fastapi.testclient import TestClient
import time


class TestRateLimitTiers:
    """Test different rate limit tiers"""

    def test_anonymous_user_limited(self, client: TestClient):
        """Anonymous users have strict rate limits"""
        # Make requests until we hit limit
        responses = []
        for i in range(52):  # Limit is 50/hour
            response = client.get("/api/v1/mcp/datasets")
            responses.append(response)

        # First 50 should succeed
        assert all(r.status_code == 200 for r in responses[:50])

        # 51st should be rate limited
        assert responses[50].status_code == 429
        assert "Retry-After" in responses[50].headers

    def test_api_key_higher_limit(self, client: TestClient, auth_headers):
        """API key users have moderate limits"""
        response = client.get(
            "/api/v1/mcp/datasets",
            headers=auth_headers
        )
        assert response.status_code == 200
        assert "X-RateLimit-Limit" in response.headers
        assert int(response.headers["X-RateLimit-Limit"]) == 1000

    def test_session_highest_limit(self, client: TestClient, session_headers):
        """Session users have generous limits"""
        response = client.get(
            "/api/v1/mcp/datasets",
            headers=session_headers
        )
        assert response.status_code == 200
        assert int(response.headers["X-RateLimit-Limit"]) == 5000


class TestRateLimitHeaders:
    """Test rate limit headers are included"""

    def test_rate_limit_headers_present(self, client: TestClient, auth_headers):
        """Rate limit headers are always included"""
        response = client.get(
            "/api/v1/mcp/datasets",
            headers=auth_headers
        )
        assert "X-RateLimit-Limit" in response.headers
        assert "X-RateLimit-Remaining" in response.headers
        assert "X-RateLimit-Reset" in response.headers

    def test_rate_limit_decrements(self, client: TestClient, auth_headers):
        """Rate limit remaining decrements with each request"""
        response1 = client.get("/api/v1/mcp/datasets", headers=auth_headers)
        remaining1 = int(response1.headers["X-RateLimit-Remaining"])

        response2 = client.get("/api/v1/mcp/datasets", headers=auth_headers)
        remaining2 = int(response2.headers["X-RateLimit-Remaining"])

        assert remaining2 == remaining1 - 1


class TestEndpointSpecificLimits:
    """Test endpoint-specific rate limits"""

    def test_batch_query_stricter_limit(self, client: TestClient, auth_headers):
        """Batch endpoints have stricter limits"""
        # Note: This endpoint might not exist yet, adjust as needed
        response = client.post(
            "/api/v1/mcp/batch-query",
            headers=auth_headers,
            json={"queries": []}
        )
        # Batch query limit is 100, not 1000
        if "X-RateLimit-Limit" in response.headers:
            assert int(response.headers["X-RateLimit-Limit"]) == 100


class TestRateLimitRecovery:
    """Test rate limit recovery and errors"""

    def test_429_response_format(self, client: TestClient):
        """429 responses include helpful information"""
        # Hit rate limit
        for i in range(51):
            response = client.get("/api/v1/mcp/datasets")

        assert response.status_code == 429
        assert "Retry-After" in response.headers
        assert "X-RateLimit-Reset" in response.headers

        data = response.json()
        assert "detail" in data
        assert "seconds" in data["detail"].lower()
```

---

**Deliverables:**
- [ ] Create `tests/test_oauth_discovery.py`
- [ ] Create `tests/test_rate_limit.py`
- [ ] Ensure Redis is running for rate limit tests
- [ ] Run tests: `pytest tests/test_oauth_discovery.py -v`
- [ ] Run tests: `pytest tests/test_rate_limit.py -v`
- [ ] All tests pass

---

### TASK 4: Documentation (Priority: LOW)
**Goal:** User-facing documentation for authentication
**Estimated Time:** 2-3 hours
**Files to Create:** 1 new doc
**Files to Modify:** 1 existing file

#### 4.1 Authentication User Guide

**New File:** `docs/AUTHENTICATION.md`

```markdown
# FLTR Authentication Guide

FLTR supports three authentication methods for different use cases.

## Authentication Methods

### 1. Session Authentication (Web App)
For users accessing FLTR through the web interface.

**How it works:**
- Login via Better Auth (Google, GitHub, email/password)
- Session tokens automatically included in API requests
- Best for interactive use in browser

**Usage:**
No manual configuration needed - handled by Next.js frontend.

---

### 2. API Keys (Developers)
For programmatic access and integrations.

**Getting your API key:**
1. Login to FLTR dashboard
2. Navigate to Settings → API Keys
3. Click "Generate New API Key"
4. Copy the key (only shown once)

**Usage:**
```bash
curl -H "X-API-Key: your_api_key_here" \
     https://api.fltr.com/api/v1/datasets
```

**Rate limits:**
- 1,000 requests per hour
- Shared across all API keys for your account

---

### 3. OAuth 2.1 (AI Tools)
For AI tools and MCP clients (Claude Desktop, VS Code, Cursor).

**How it works:**
1. AI tool connects to FLTR MCP server
2. Tool automatically discovers OAuth endpoints
3. Browser opens for one-time authorization
4. You consent to scope access
5. Tool receives access token
6. All future requests authenticated automatically

**Supported scopes:**
- `mcp:query` - Query MCP endpoints for semantic search
- `mcp:tools` - Access all MCP tools and resources
- `mcp:create` - Create MCP endpoint configurations

**Rate limits:**
- 5,000 requests per hour
- Same as web session limits

**Setup in Claude Desktop:**

```json
{
  "mcpServers": {
    "fltr": {
      "url": "https://api.fltr.com",
      "transport": "http",
      "authentication": "oauth"
    }
  }
}
```

Claude will automatically handle OAuth flow on first connection.

---

## Rate Limiting

All API requests are rate-limited based on authentication method:

| Auth Method | Limit | Window | Use Case |
|-------------|-------|--------|----------|
| Unauthenticated | 50 req/hour | 1 hour | Discovery only |
| API Key | 1,000 req/hour | 1 hour | Standard tier |
| Session/OAuth | 5,000 req/hour | 1 hour | Premium tier |

### Rate Limit Headers

All responses include rate limit information:

```
X-RateLimit-Limit: 1000         # Total requests allowed
X-RateLimit-Remaining: 950      # Requests remaining
X-RateLimit-Reset: 1730899200   # Unix timestamp when limit resets
```

### 429 Too Many Requests

When you exceed rate limits:

```json
{
  "detail": "Rate limit exceeded. Try again in 3542 seconds."
}
```

**Response headers:**
```
Retry-After: 3542              # Seconds until next request allowed
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1730899200
```

**Best practices:**
- Check `X-RateLimit-Remaining` before making burst requests
- Implement exponential backoff when receiving 429
- Cache results when possible to reduce API calls

---

## Credits vs Rate Limits

**Rate limits** and **credits** are separate:

- **Rate limits** prevent abuse (applies to ALL requests)
- **Credits** are consumed for expensive operations (embeddings, queries)

You can be:
- ✅ Within rate limit, but out of credits → 402 Payment Required
- ✅ Within credit balance, but hit rate limit → 429 Too Many Requests
- ❌ Out of both → 429 first, then 402 if you have credits

---

## Troubleshooting

### "Authentication required"

**Error:**
```json
{
  "detail": "Authentication required. Provide either X-API-Key or Authorization Bearer token."
}
```

**Solution:**
- For API calls: Add `X-API-Key` header
- For web app: Check you're logged in
- For MCP tools: Complete OAuth flow

---

### "Rate limit exceeded"

**Error:**
```json
{
  "detail": "Rate limit exceeded. Try again in 3542 seconds."
}
```

**Solution:**
- Wait for the time specified in `Retry-After` header
- Reduce request frequency
- Upgrade to session/OAuth auth for higher limits
- Implement request throttling in your client

---

### "Insufficient credits"

**Error:**
```json
{
  "detail": "Insufficient credits. Required: 10, Available: 5"
}
```

**Solution:**
- Purchase more credits in dashboard
- This is separate from rate limiting

---

## Security Best Practices

### API Keys
- ❌ Don't commit API keys to version control
- ❌ Don't share API keys publicly
- ✅ Use environment variables: `export FLTR_API_KEY=...`
- ✅ Rotate keys regularly
- ✅ Revoke compromised keys immediately

### OAuth Tokens
- ✅ Tokens expire automatically (handled by MCP client)
- ✅ Refresh tokens are used transparently
- ✅ Revoke access anytime in dashboard

---

## Support

Questions? Contact support@fltr.com or open an issue on GitHub.
```

---

#### 4.2 Update Main README

**File:** `README.md`

**Add authentication section:**

```markdown
## Authentication

FLTR supports three authentication methods:

### Web App (Session Auth)
Login via Better Auth with Google, GitHub, or email/password. Session tokens are automatically managed.

### API Keys (Developers)
Generate API keys from your dashboard for programmatic access.

```bash
curl -H "X-API-Key: your_key" https://api.fltr.com/api/v1/datasets
```

### OAuth 2.1 (AI Tools)
MCP clients (Claude Desktop, VS Code) auto-discover OAuth endpoints and handle authentication transparently.

**Rate Limits:**
- Anonymous: 50 requests/hour
- API Keys: 1,000 requests/hour
- Sessions/OAuth: 5,000 requests/hour

See [docs/AUTHENTICATION.md](docs/AUTHENTICATION.md) for detailed guide.
```

---

**Deliverables:**
- [ ] Create `docs/AUTHENTICATION.md`
- [ ] Update `README.md` with authentication section
- [ ] Add examples for all three auth methods
- [ ] Document rate limits and troubleshooting
- [ ] Add security best practices

---

## Implementation Checklist

### Phase 1: OAuth Discovery (4-6 hours)
- [ ] Create `routers/mcp_auth_metadata.py`
- [ ] Add `API_BASE_URL` and `AUTH_SERVER_URL` to config
- [ ] Update `.env` with new variables
- [ ] Register router in `main.py`
- [ ] Enhance 401 responses with `WWW-Authenticate` header
- [ ] Test metadata endpoints
- [ ] Test 401 includes discovery URL

### Phase 2: Rate Limiting (6-8 hours)
- [ ] Create `middleware/rate_limit.py`
- [ ] Register middleware in `main.py` (BEFORE auth)
- [ ] Ensure Redis is running
- [ ] Test anonymous limits (50/hour)
- [ ] Test API key limits (1000/hour)
- [ ] Test session limits (5000/hour)
- [ ] Test 429 responses
- [ ] Test rate limit headers on all responses

### Phase 3: Testing (4-6 hours)
- [ ] Create `tests/test_oauth_discovery.py`
- [ ] Create `tests/test_rate_limit.py`
- [ ] All OAuth tests pass
- [ ] All rate limit tests pass
- [ ] Add tests to CI/CD pipeline

### Phase 4: Documentation (2-3 hours)
- [ ] Create `docs/AUTHENTICATION.md`
- [ ] Update `README.md`
- [ ] Add troubleshooting guide
- [ ] Document all three auth methods
- [ ] Add rate limit documentation

---

## Testing Strategy

### Manual Testing Workflow

```bash
# 1. Start services
redis-server &
cd fastapi && uvicorn main:app --reload

# 2. Test OAuth discovery
curl http://localhost:8000/.well-known/oauth-protected-resource | jq
curl http://localhost:8000/.well-known/oauth-authorization-server | jq

# 3. Test rate limiting (anonymous)
for i in {1..51}; do
  echo "Request $i:"
  curl -i http://localhost:8000/api/v1/mcp/datasets | grep -E "HTTP|X-RateLimit"
done

# 4. Test rate limiting (API key)
curl -H "X-API-Key: $FLTR_API_KEY" \
     http://localhost:8000/api/v1/mcp/datasets | jq

# 5. Test 401 with discovery
curl -i http://localhost:8000/api/v1/mcp/query/test-id/endpoint | \
  grep -E "WWW-Authenticate|oauth"
```

### Automated Testing

```bash
# Run all tests
pytest tests/ -v

# Run specific test suites
pytest tests/test_oauth_discovery.py -v
pytest tests/test_rate_limit.py -v

# Run with coverage
pytest tests/ --cov=middleware --cov=routers --cov-report=html
```

---

## Deployment Checklist

### Environment Variables

```bash
# .env (development)
API_BASE_URL=http://localhost:8000
AUTH_SERVER_URL=http://localhost:3000/api/auth
REDIS_URL=redis://localhost:6379/0

# .env.production
API_BASE_URL=https://api.fltr.com
AUTH_SERVER_URL=https://fltr.com/api/auth
REDIS_URL=redis://your-redis-instance:6379/0
```

### Infrastructure

- [ ] Redis instance deployed and accessible
- [ ] Better Auth configured with OAuth scopes
- [ ] Environment variables set in production
- [ ] Health checks updated to include Redis

### Monitoring

- [ ] Add rate limit metrics to dashboard
- [ ] Alert on high 429 rate (may indicate limits too strict)
- [ ] Alert on Redis connection failures
- [ ] Monitor OAuth flow success rate

---

## Success Criteria

### Phase 1: OAuth Discovery
- [x] Metadata endpoints return valid JSON
- [x] MCP clients can discover authorization server
- [x] 401 responses include discovery metadata
- [x] No breaking changes to existing auth

### Phase 2: Rate Limiting
- [x] Rate limits enforced per tier
- [x] Redis connection stable
- [x] All responses include rate limit headers
- [x] 429 errors return helpful retry information
- [x] Graceful degradation if Redis unavailable

### Phase 3: Testing
- [x] All tests pass
- [x] Coverage > 80% for new code
- [x] Integration tests cover full OAuth flow
- [x] Rate limit tests cover all tiers

### Phase 4: Documentation
- [x] Authentication guide complete
- [x] README updated
- [x] Troubleshooting guide included
- [x] Security best practices documented

---

## Future Enhancements (Out of Scope)

Not included in this implementation, but worth considering later:

1. **Dynamic Rate Limits**
   - Adjust limits based on user's credit balance
   - Premium tier users get higher limits

2. **Cost-Based Rate Limiting**
   - Heavy endpoints count more toward limit
   - Batch queries = 5 requests, simple queries = 1 request

3. **OAuth Scope Enforcement**
   - Fine-grained permissions per scope
   - Scope-based access control in endpoints

4. **Rate Limit Dashboard**
   - Real-time visualization of rate limit usage
   - Alerts when approaching limits

5. **IP Allowlisting**
   - Bypass rate limits for trusted IPs
   - For internal services and monitoring

---

## Conclusion

This implementation adds critical security and compliance features:

**Security:**
- ✅ Rate limiting prevents abuse and DDoS
- ✅ OAuth discovery enables secure MCP integration
- ✅ Credits and rate limits work together

**Developer Experience:**
- ✅ AI tools auto-discover OAuth endpoints
- ✅ Clear error messages with retry information
- ✅ Rate limit headers in every response

**Business:**
- ✅ MCP-compliant (industry standard)
- ✅ Prevents cost explosion from abuse
- ✅ Supports all major AI tools (Claude, VS Code, etc.)

**Estimated Total Effort:** 16-23 hours (~2-3 days)

---

**Document Status:** ✅ Ready for Implementation
**Last Updated:** November 5, 2025
**Next Step:** Begin with Task 2 (Rate Limiting) - highest priority for security
