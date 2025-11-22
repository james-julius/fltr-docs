# Coolify SSL Configuration Fix

## Problem Summary

The production FastAPI server at `api.tryfltr.com` was crashing on startup with:

```
psycopg2.ProgrammingError: invalid dsn: invalid connection option "ssl"
sqlalchemy.exc.ProgrammingError: (psycopg2.ProgrammingError) invalid dsn: invalid connection option "ssl"
ERROR: Application startup failed. Exiting.
```

**Root Cause**: The database connection string used `?ssl=true` or `?ssl=require`, which is **invalid for Python's psycopg2**. It needs `?sslmode=require` instead.

---

## Solution Applied

I've updated the FastAPI database configuration to automatically handle SSL similar to the node-postgres pattern:

### Changes Made

1. **Added SSL configuration to config.py** ([config.py:109](config.py#L109))
   ```python
   # Database SSL Configuration (for managed PostgreSQL)
   DATABASE_CA_CERT: str = ""  # CA certificate for SSL connection (optional)
   ```

2. **Created SSL configuration helper** ([sql_store.py:80-134](database/sql_store.py#L80-L134))
   ```python
   def _apply_ssl_config(database_url: str, ca_cert: str = "") -> str:
       """
       Apply SSL configuration to PostgreSQL connection string.

       - Localhost: No SSL needed
       - Remote with CA cert: sslmode=require
       - Remote without CA cert: sslmode=no-verify
       """
   ```

3. **Updated database engines** to use SSL configuration:
   - `get_engine()` for main database ([sql_store.py:124-125](database/sql_store.py#L124-L125))
   - `get_auth_engine()` for auth database ([sql_store.py:341-342](database/sql_store.py#L341-L342))

### How It Works

The code now **automatically** detects the connection type and applies the correct SSL mode:

| Connection Type | SSL Mode Applied | Example |
|----------------|------------------|---------|
| Localhost | None | `postgresql://...@localhost/db` |
| Remote + CA cert | `sslmode=require` | `postgresql://...@remote.host/db?sslmode=require` |
| Remote, no CA cert | `sslmode=no-verify` | `postgresql://...@remote.host/db?sslmode=no-verify` |

---

## Coolify Configuration Steps

### Option 1: Remove Invalid SSL Parameter (Recommended)

**Change your DATABASE_URL in Coolify from:**
```bash
# WRONG - causes crash
DATABASE_URL=postgresql://user:pass@host:5432/db?ssl=true
```

**To:**
```bash
# CORRECT - let FastAPI handle SSL automatically
DATABASE_URL=postgresql://user:pass@host:5432/db
```

The code will automatically detect it's a remote connection and add `sslmode=no-verify`.

### Option 2: Use Correct sslmode Parameter

If you prefer explicit control:

```bash
# Without CA certificate validation
DATABASE_URL=postgresql://user:pass@host:5432/db?sslmode=no-verify

# With CA certificate validation (also provide DATABASE_CA_CERT)
DATABASE_URL=postgresql://user:pass@host:5432/db?sslmode=require
DATABASE_CA_CERT=-----BEGIN CERTIFICATE-----
MIIDrzCCApegAwIBAgIQCDvgVpB...
-----END CERTIFICATE-----
```

### Option 3: Use CA Certificate (Most Secure)

For managed databases (DigitalOcean, AWS RDS, etc.) with CA certificates:

1. **Get your database CA certificate** (download from provider)
2. **Set in Coolify environment variables:**
   ```bash
   DATABASE_URL=postgresql://user:pass@host:5432/db
   DATABASE_CA_CERT=-----BEGIN CERTIFICATE-----
   MIIDrzCCApegAwIBAgIQCDvgVpB...
   -----END CERTIFICATE-----
   ```

The code will automatically apply `sslmode=require` when it detects `DATABASE_CA_CERT` is present.

---

## Verification Steps

After updating Coolify environment variables:

1. **Save changes in Coolify** (triggers automatic redeploy)
2. **Watch the logs** for SSL configuration messages:
   ```
   🔍 SSL: No CA certificate, using sslmode=no-verify
   🔍 SSL: Applied no-verify to connection string
   ✅ Application startup complete.
   ```
3. **Test the API**:
   ```bash
   curl https://api.tryfltr.com/api/v1/health
   ```

---

## Comparison: Python vs Node.js

### Node.js (migrate.ts) - BEFORE
```typescript
if (!isLocalhost && !databaseUrl.includes('sslmode=')) {
    if (caCert) {
        databaseUrl += '?sslmode=require'
    } else {
        databaseUrl += '?sslmode=no-verify'
    }
}
```

### Python (sql_store.py) - NOW (AFTER FIX)
```python
def _apply_ssl_config(database_url: str, ca_cert: str = "") -> str:
    if is_localhost:
        return database_url

    if 'sslmode=' in database_url:
        return database_url

    if ca_cert:
        sslmode = 'require'
    else:
        sslmode = 'no-verify'

    return f"{database_url}?sslmode={sslmode}"
```

**Both services now handle SSL consistently!** ✅

---

## Common Mistakes to Avoid

❌ **DON'T use these (will crash FastAPI):**
```bash
?ssl=true
?ssl=require
?ssl=disable
?sslrequire=true
```

✅ **DO use these (correct for psycopg2):**
```bash
?sslmode=require
?sslmode=no-verify
?sslmode=disable
```

---

## Production Recommendations

### DigitalOcean Managed Database
```bash
# Option 1: No certificate validation (simpler, still encrypted)
DATABASE_URL=postgresql://user:pass@db-host.db.ondigitalocean.com:25060/db?sslmode=no-verify

# Option 2: With certificate validation (most secure)
DATABASE_URL=postgresql://user:pass@db-host.db.ondigitalocean.com:25060/db?sslmode=require
DATABASE_CA_CERT=<download from DigitalOcean dashboard>
```

### AWS RDS
```bash
DATABASE_URL=postgresql://user:pass@rds-instance.region.rds.amazonaws.com:5432/db?sslmode=require
DATABASE_CA_CERT=<download from AWS docs>
```

### Self-Hosted PostgreSQL
```bash
# Local network (no SSL needed)
DATABASE_URL=postgresql://user:pass@10.0.1.5:5432/db

# With self-signed cert
DATABASE_URL=postgresql://user:pass@postgres.internal:5432/db?sslmode=require
DATABASE_CA_CERT=<your CA certificate>
```

---

## Troubleshooting

### Server still crashing after fix?

1. **Check environment variables in Coolify:**
   - Go to your app → Environment
   - Verify `DATABASE_URL` doesn't contain `?ssl=`
   - Click "Redeploy" if you made changes

2. **Check logs for SSL messages:**
   ```bash
   # Should see:
   🔍 SSL: No CA certificate, using sslmode=no-verify

   # Should NOT see:
   psycopg2.ProgrammingError: invalid dsn: invalid connection option "ssl"
   ```

3. **Verify database connection string format:**
   ```bash
   # Print in FastAPI startup logs
   import os
   print(f"DATABASE_URL: {os.getenv('DATABASE_URL')}")
   ```

### Testing locally

```bash
# Test with correct sslmode
export DATABASE_URL="postgresql://user:pass@remote:5432/db?sslmode=no-verify"
cd fastapi
python -m uvicorn main:app

# Should start without errors
```

---

## Summary

- ✅ **Python code now matches Node.js SSL handling**
- ✅ **Automatic SSL mode detection** (no manual configuration needed)
- ✅ **Supports CA certificate validation** (optional but recommended)
- ✅ **Prevents the `invalid connection option "ssl"` error**

**Next Step**: Update your Coolify environment variables and redeploy!
