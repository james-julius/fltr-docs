# Production Database Nuke & Rebuild

**Date**: 2025-11-05
**Status**: ✅ Ready to execute
**Risk**: 🟢 LOW (No production users)

## ✅ Local Status

- ✅ Both local databases nuked and rebuilt
- ✅ Schema perfectly aligned (Drizzle + SQLModel + Database)
- ✅ All FK constraints correct
- ✅ No schema drift detected

## 🎯 Production Plan

Since **no production users exist**, we'll nuke both production databases and let deployment rebuild them automatically.

---

## Step 1: Connect to Production Databases

You'll need connection strings for:
- Production `fltr` database
- Production `fltr_auth` database

Get these from your hosting provider (Neon, Supabase, Railway, etc.)

---

## Step 2: Nuke Production `fltr` Database

Connect via TablePlus, psql, or your database client:

```sql
-- Connect to production fltr database
-- Run this SQL:

DROP SCHEMA public CASCADE;
CREATE SCHEMA public;
GRANT ALL ON SCHEMA public TO postgres;
GRANT ALL ON SCHEMA public TO public;

-- Verify empty
SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'public';
-- Should return: 0
```

---

## Step 3: Nuke Production `fltr_auth` Database

Connect via TablePlus, psql, or your database client:

```sql
-- Connect to production fltr_auth database
-- Run this SQL:

DROP SCHEMA public CASCADE;
CREATE SCHEMA public;
GRANT ALL ON SCHEMA public TO postgres;
GRANT ALL ON SCHEMA public TO public;

-- Verify empty
SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'public';
-- Should return: 0
```

---

## Step 4: Commit and Deploy Code

Now that databases are empty, deploy the fixed code:

```bash
# Make sure all changes are committed
git status

# If needed, commit the schema fixes
git add fastapi/models/credits.py
git commit -m "fix: Align FK constraints across Drizzle and SQLModel

- Update SQLModel to match Drizzle schema
- All credit-related FKs now point to users.id
- Separate user_credits table for credit management
- Clean separation between fltr and fltr_auth databases

🤖 Generated with Claude Code"

# Push to trigger deployment
git push origin main
```

---

## Step 5: Automatic Rebuild on Deploy

### What Happens Automatically

**Vercel (Next.js):**
1. Runs `pnpm run build`
2. Which runs `pnpm run db:migrate`
3. Drizzle migrations create all `fltr_auth` tables:
   - users
   - sessions
   - accounts
   - verifications
   - subscriptions
   - teams
   - **user_credits** ← Separate table!
   - credit_transactions
   - api_key_credits

**FastAPI (On first startup):**
1. Runs `init_db()` → creates `fltr` tables:
   - datasets
   - documents
   - processing_jobs

2. Runs `init_auth_db()` → creates credit tables in `fltr_auth`:
   - Already exist from Drizzle ✅
   - SQLModel just verifies they exist

---

## Step 6: Verify Production Deployment

### Check Vercel Deployment Logs

Look for:
```
🔄 Running database migrations...
✅ Database migrations completed successfully
```

### Check FastAPI Logs

Look for:
```
🚀 Starting FLTR API in production mode
✅ SQL database initialized
✅ Auth database initialized
```

### Manual Verification (Optional)

Connect to production databases and verify tables:

**Production `fltr`:**
```sql
SELECT tablename FROM pg_tables WHERE schemaname = 'public' ORDER BY tablename;
-- Should show:
-- datasets
-- documents
-- processing_jobs
```

**Production `fltr_auth`:**
```sql
SELECT tablename FROM pg_tables WHERE schemaname = 'public' ORDER BY tablename;
-- Should show:
-- accounts
-- api_key_credits
-- credit_transactions
-- sessions
-- subscriptions
-- teams
-- user_credits
-- users
-- verifications
```

### Verify FK Constraints

```sql
-- Connect to fltr_auth
SELECT
    tc.table_name,
    kcu.column_name,
    ccu.table_name AS foreign_table,
    ccu.column_name AS foreign_column
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu
  ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage ccu
  ON ccu.constraint_name = tc.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
  AND tc.table_name IN ('user_credits', 'credit_transactions', 'api_key_credits')
ORDER BY tc.table_name;

-- Should show:
-- api_key_credits.user_id → users.id ✅
-- credit_transactions.user_id → users.id ✅
-- user_credits.user_id → users.id ✅
```

---

## Step 7: Test the Application

### Test Authentication (Next.js)
1. Go to your production URL
2. Try to sign up with a new account
3. Verify:
   - User created in `fltr_auth.users`
   - User credits auto-created in `fltr_auth.user_credits`

### Test Credit System (FastAPI)
```bash
# Get your production FastAPI URL
FASTAPI_URL="https://your-fastapi-url.com"

# Test credit balance endpoint (requires auth token)
curl -H "Authorization: Bearer YOUR_TOKEN" \
  "$FASTAPI_URL/api/v1/credits/balance"

# Should return:
# {"credits": 100, "total_credits_purchased": 0, "team_id": null, "team_credits": null}
```

### Test Dataset Creation
```bash
# Create a dataset (should consume credits)
curl -X POST "$FASTAPI_URL/api/v1/datasets" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Test Dataset", "description": "Testing"}'

# Verify dataset created in fltr.datasets
# Verify credit transaction in fltr_auth.credit_transactions
```

---

## Rollback Plan

If something goes wrong, you can rollback to the previous state:

### Option 1: Revert Code Deploy
```bash
# Rollback to previous commit
git revert HEAD
git push origin main
```

### Option 2: Restore Databases from Backup

If you took backups before Step 2/3:
```bash
# Restore fltr database
psql $PROD_FLTR_URL < backup_fltr_$(date +%Y%m%d).sql

# Restore fltr_auth database
psql $PROD_FLTR_AUTH_URL < backup_fltr_auth_$(date +%Y%m%d).sql
```

**Note**: Since databases were empty, there's nothing to restore. Rollback is just redeploying previous code.

---

## Expected Timeline

| Step | Duration | Risk |
|------|----------|------|
| 1. Connect to databases | 2 min | None |
| 2. Nuke fltr | 30 sec | None (empty) |
| 3. Nuke fltr_auth | 30 sec | None (empty) |
| 4. Commit & deploy | 2 min | Low |
| 5. Automatic rebuild | 3-5 min | Low |
| 6. Verify deployment | 3 min | None |
| 7. Test application | 5 min | None |
| **Total** | **~15 min** | **🟢 LOW** |

---

## Pre-Flight Checklist

- [ ] Confirmed no production users exist
- [ ] Have access to production database consoles
- [ ] Have connection strings for both databases
- [ ] Local databases tested and working
- [ ] All code changes committed
- [ ] Deployment pipeline is healthy
- [ ] Can monitor deployment logs

## Post-Flight Checklist

- [ ] Both databases emptied successfully
- [ ] Code deployed without errors
- [ ] Vercel migration logs show success
- [ ] FastAPI startup logs show success
- [ ] All tables created in correct databases
- [ ] FK constraints verified
- [ ] Can sign up new user
- [ ] User credits auto-created
- [ ] Can create dataset
- [ ] Credit transaction recorded

---

## Success Criteria

✅ **fltr database** has only: datasets, documents, processing_jobs
✅ **fltr_auth database** has all auth + credit tables
✅ **user_credits** is separate table (not columns in users)
✅ All FK constraints point to correct tables
✅ New user signup works
✅ Credits auto-created on signup
✅ Dataset creation works and consumes credits
✅ No errors in logs

---

## Contact

If anything goes wrong, check:
- [EXPECTED_VS_ACTUAL_SCHEMAS.md](EXPECTED_VS_ACTUAL_SCHEMAS.md) - What should exist
- [DUAL_ORM_ARCHITECTURE.md](DUAL_ORM_ARCHITECTURE.md) - How Drizzle + SQLModel work together
- [SCHEMA_OWNERSHIP.md](SCHEMA_OWNERSHIP.md) - Who owns which tables
- [MIGRATION_WORKFLOW.md](MIGRATION_WORKFLOW.md) - Full migration best practices

---

🚀 **Ready to execute!**
