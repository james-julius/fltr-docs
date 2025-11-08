# Cloudflare Worker Authentication Update

## ✅ What Changed

Updated the `dataset-upload-notification-worker` to use the same API key authentication as the FastAPI backend.

## 🔐 Security Improvement

**Before**: Webhooks were not authenticated (anyone could call them)
**After**: Webhooks require `X-API-Key` header with valid API key

## 📝 Changes Made

### 1. Worker Code (`dataset-upload-notification-worker`)

#### `src/index.ts`
- Added `FLTR_API_KEY: string` to `Env` interface
- Pass API key to `FltrApi` constructor

#### `src/server.ts`
- Added `apiKey` parameter to constructor
- Include `X-API-Key` header in webhook requests

#### `test/index.spec.ts`
- Added `FLTR_API_KEY` to mock environment
- Pass API key to `FltrApi` constructor in tests
- Verify `X-API-Key` header in expected fetch calls
- ✅ All 8 tests passing

#### `wrangler.toml`
- Added comment about setting `FLTR_API_KEY` as a secret

#### `README.md`
- Updated setup instructions to use `FLTR_API_KEY`
- Replaced `MICROSERVICE_TOKEN` references with `FLTR_API_KEY`
- Added instructions to generate keys with `generate_api_key.py`

### 2. FastAPI Middleware (`fastapi/middleware/auth.py`)

**Removed** storage and webhooks from public routes:
- Storage endpoints now require authentication
- Webhook endpoints now require authentication

This prevents unauthorized access while still allowing authenticated external services (like the worker) to call them.

## 🚀 Deployment Steps

### 1. Generate API Key

```bash
cd fastapi
python generate_api_key.py
```

Example output:
```
🔑 Generated 1 API key(s):
============================================================
1. fltr_vB34uczRu_c8_b6n77uYTmskPMwUWwAEGLPf00iOUsY
============================================================
```

### 2. Add to FastAPI (Production)

```env
# In your production .env or environment variables
API_KEYS=fltr_vB34uczRu_c8_b6n77uYTmskPMwUWwAEGLPf00iOUsY,<other-keys>
```

### 3. Set Worker Secret

```bash
cd dataset-upload-notification-worker

# Set the API key as a secret
wrangler secret put FLTR_API_KEY
# When prompted, paste: fltr_vB34uczRu_c8_b6n77uYTmskPMwUWwAEGLPf00iOUsY
```

### 4. Deploy Worker

```bash
npm run deploy
```

### 5. Verify

```bash
# Check secret is set
wrangler secret list

# Should show:
# FLTR_API_KEY

# Test upload (will trigger webhook)
# Upload a file to R2 and check worker logs
wrangler tail
```

## 🧪 Testing

All worker tests pass with authentication:

```bash
cd dataset-upload-notification-worker
npm test
```

Output:
```
✓ test/index.spec.ts (8 tests) 64ms
Test Files  1 passed (1)
     Tests  8 passed (8)
```

## 📊 Authentication Flow

```
R2 Upload → CF Worker → Cloudflare Queue
                  ↓
           (Queue failure)
                  ↓
         FastAPI Webhook
         (checks X-API-Key)
                  ↓
         ✅ Authorized → Process
         ❌ No key → 401 Error
```

## 🔑 Key Management

### Generating Keys

```bash
cd fastapi
python generate_api_key.py         # Generate 1 key
python generate_api_key.py --count 3  # Generate 3 keys
```

### Using Multiple Keys

You can have different keys for different purposes:

```env
# Production .env
API_KEYS=fltr_worker_key_xxxx,fltr_testing_key_yyyy,fltr_partner_key_zzzz
```

- **Worker key**: Used by Cloudflare worker
- **Testing key**: Used for development/testing
- **Partner key**: For external integrations

### Revoking Keys

Remove from `API_KEYS` environment variable and restart FastAPI:

```env
# Remove compromised key
API_KEYS=fltr_worker_key_xxxx,fltr_testing_key_yyyy
# (removed fltr_partner_key_zzzz)
```

## 🛡️ Security Benefits

1. **Webhook Protection**: Prevents unauthorized webhook calls
2. **Storage Protection**: Upload URL generation requires authentication
3. **Audit Trail**: Can track which key is used for what
4. **Easy Rotation**: Generate new keys and update worker secret
5. **Multiple Keys**: Different keys for different services

## 📚 Related Documentation

- `/fastapi/AUTH_SETUP.md` - Complete auth setup guide
- `/fastapi/SIMPLE_AUTH_GUIDE.md` - Auth usage guide
- `/dataset-upload-notification-worker/README.md` - Worker setup

## 🎯 Summary

✅ Worker now authenticates webhook calls with API key
✅ Storage and webhook endpoints are protected
✅ All tests passing (worker + FastAPI)
✅ Simple key management with `generate_api_key.py`
✅ Production ready!

**Next step**: Deploy worker with API key secret set.

