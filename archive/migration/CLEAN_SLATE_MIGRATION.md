# Clean Slate Migration - Nuke & Rebuild All Databases

**Date**: 2025-11-05
**Status**: No production users - safe to nuke everything

## Why This Approach

Since there are **no production users yet**, we can:
1. Drop all existing tables
2. Run migrations from scratch
3. Ensure perfect alignment between code and database
4. Start with a clean, correct schema

## Migration Steps

### Step 1: Nuke Local Databases

#### Local `fltr` Database
```bash
psql postgresql://admin:password@127.0.0.1:5432/fltr << 'EOF'
-- Drop all tables in fltr database
DROP SCHEMA public CASCADE;
CREATE SCHEMA public;
GRANT ALL ON SCHEMA public TO admin;
GRANT ALL ON SCHEMA public TO public;

-- Verify empty
\dt
EOF
```

#### Local `fltr_auth` Database
```bash
psql postgresql://admin:password@127.0.0.1:5432/fltr_auth << 'EOF'
-- Drop all tables in fltr_auth database
DROP SCHEMA public CASCADE;
CREATE SCHEMA public;
GRANT ALL ON SCHEMA public TO admin;
GRANT ALL ON SCHEMA public TO public;

-- Verify empty
\dt
EOF
```

### Step 2: Rebuild Local Databases

#### Rebuild Local `fltr_auth` (Drizzle migrations)
```bash
cd /Users/jamesjulius/Coding/FLTR/nextjs

# Run all Drizzle migrations
pnpm run db:migrate

# Verify tables created
psql postgresql://admin:password@127.0.0.1:5432/fltr_auth -c "\dt"
```

**Expected tables**:
- users
- sessions
- accounts
- verifications
- subscriptions
- teams
- user_credits
- credit_transactions
- api_key_credits
- __drizzle_migrations

#### Rebuild Local `fltr` (FastAPI SQLModel)
```bash
cd /Users/jamesjulius/Coding/FLTR/fastapi

# Start server (will run init_db() and init_auth_db())
python -m uvicorn main:app --port 8000

# In another terminal, verify tables
psql postgresql://admin:password@127.0.0.1:5432/fltr -c "\dt"
```

**Expected tables in fltr**:
- datasets
- documents
- processing_jobs

**Expected tables in fltr_auth** (additional from SQLModel):
- None! Drizzle already created everything

### Step 3: Nuke Production Databases

⚠️ **DANGER ZONE** - Only proceed if you're 100% sure no production data exists

#### Production `fltr` Database
```sql
-- Connect to PRODUCTION fltr database
-- Run in your database client (TablePlus, etc.)

DROP SCHEMA public CASCADE;
CREATE SCHEMA public;
GRANT ALL ON SCHEMA public TO postgres;
GRANT ALL ON SCHEMA public TO public;
```

#### Production `fltr_auth` Database
```sql
-- Connect to PRODUCTION fltr_auth database
-- Run in your database client (TablePlus, etc.)

DROP SCHEMA public CASCADE;
CREATE SCHEMA public;
GRANT ALL ON SCHEMA public TO postgres;
GRANT ALL ON SCHEMA public TO public;
```

### Step 4: Rebuild Production Databases

#### Option A: Run Migrations Locally, Push Schema

```bash
# 1. Get production connection strings
export PROD_FLTR_AUTH_URL="postgresql://user:pass@host:5432/fltr_auth"
export PROD_FLTR_URL="postgresql://user:pass@host:5432/fltr"

# 2. Run Drizzle migrations on production fltr_auth
cd /Users/jamesjulius/Coding/FLTR/nextjs
DATABASE_URL=$PROD_FLTR_AUTH_URL pnpm run db:migrate

# 3. Run FastAPI to create fltr tables
cd /Users/jamesjulius/Coding/FLTR/fastapi
# Update .env temporarily to point to production
# Start server once to run init_db()
DATABASE_URL=$PROD_FLTR_URL AUTH_DATABASE_URL=$PROD_FLTR_AUTH_URL python -m uvicorn main:app
# Stop after initialization completes
```

#### Option B: Deploy and Let Vercel Run Migrations

```bash
# 1. Commit all changes
git add -A
git commit -m "feat: Clean database schema with proper FK constraints

- Separate user_credits table for credit management
- Proper FK constraints (user_credits.user_id -> users.id)
- Clean separation between fltr and fltr_auth databases
- Drizzle migrations for fltr_auth
- SQLModel initialization for fltr

🤖 Generated with Claude Code"

# 2. Push to trigger deployment
git push origin main

# Vercel will:
# - Run `pnpm run build` which includes `pnpm run db:migrate`
# - Drizzle will create all fltr_auth tables
# - When Next.js starts, tables are ready

# FastAPI will:
# - Run init_db() on startup
# - Create datasets, documents, processing_jobs in fltr database
# - Run init_auth_db() on startup (creates credit tables if missing, but Drizzle already did this)
```

### Step 5: Verify Everything

#### Verify Local
```bash
# Check fltr database
psql postgresql://admin:password@127.0.0.1:5432/fltr -c "
SELECT schemaname, tablename
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY tablename;"

# Should show:
# - datasets
# - documents
# - processing_jobs

# Check fltr_auth database
psql postgresql://admin:password@127.0.0.1:5432/fltr_auth -c "
SELECT schemaname, tablename
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY tablename;"

# Should show:
# - __drizzle_migrations
# - accounts
# - api_key_credits
# - credit_transactions
# - sessions
# - subscriptions
# - teams
# - user_credits
# - users
# - verifications
```

#### Verify Production
```sql
-- Check production fltr database
SELECT schemaname, tablename
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY tablename;

-- Should match local (datasets, documents, processing_jobs)

-- Check production fltr_auth database
SELECT schemaname, tablename
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY tablename;

-- Should match local (all auth + credit tables)
```

#### Verify FK Constraints
```bash
# Check fltr_auth FKs
psql $DATABASE_URL -c "
SELECT
    tc.table_name,
    kcu.column_name,
    ccu.table_name AS foreign_table_name,
    ccu.column_name AS foreign_column_name,
    tc.constraint_name
FROM information_schema.table_constraints AS tc
JOIN information_schema.key_column_usage AS kcu
  ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage AS ccu
  ON ccu.constraint_name = tc.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
  AND tc.table_schema = 'public'
ORDER BY tc.table_name, kcu.column_name;"
```

**Expected FK constraints**:
- `user_credits.user_id` → `users.id` (CASCADE)
- `user_credits.team_id` → `teams.id`
- `credit_transactions.user_id` → `users.id` (CASCADE)
- `credit_transactions.team_id` → `teams.id` (SET NULL)
- `api_key_credits.user_id` → `users.id` (CASCADE)
- `api_key_credits.team_id` → `teams.id` (CASCADE)
- `teams.owner_id` → `users.id` (CASCADE)
- `sessions.user_id` → `users.id` (CASCADE)
- `accounts.user_id` → `users.id` (CASCADE)

### Step 6: Test Everything

```bash
# 1. Start local servers
cd /Users/jamesjulius/Coding/FLTR/fastapi
python -m uvicorn main:app --port 8000 &

cd /Users/jamesjulius/Coding/FLTR/nextjs
pnpm dev &

# 2. Test login
# - Open http://localhost:3000
# - Try to sign up
# - Verify user created in fltr_auth.users
# - Verify user_credits created automatically

# 3. Test credit system
# - Make API call that consumes credits
# - Verify credit_transaction created
# - Verify user_credits.credits decremented

# 4. Test FastAPI
curl http://localhost:8000/health
curl http://localhost:8000/api/v1/credits/balance
```

## What Gets Fixed

✅ **No duplicate users tables** - users only in fltr_auth
✅ **Separate user_credits table** - credits managed properly
✅ **Correct FK constraints** - all FKs point to right tables
✅ **Clean migration history** - Drizzle tracks all migrations
✅ **Both ORMs in sync** - Drizzle and SQLModel match perfectly

## Rollback Plan

If something goes wrong, you can restore from the backup you took before Step 3:

```bash
# Restore local
psql postgresql://admin:password@127.0.0.1:5432/fltr_auth < backup_local_fltr_auth.sql
psql postgresql://admin:password@127.0.0.1:5432/fltr < backup_local_fltr.sql

# Restore production
psql $PROD_FLTR_AUTH_URL < backup_prod_fltr_auth.sql
psql $PROD_FLTR_URL < backup_prod_fltr.sql
```

## Pre-Migration Checklist

- [ ] Verified no production users exist
- [ ] Backed up all 4 databases
- [ ] Committed all code changes
- [ ] Tested locally first
- [ ] Production environment variables ready
- [ ] Monitoring/alerting ready to catch issues

## Post-Migration Checklist

- [ ] All tables created in correct databases
- [ ] FK constraints verified
- [ ] Drizzle migration tracking table exists
- [ ] Can sign up new user successfully
- [ ] User credits auto-created on signup
- [ ] FastAPI credit endpoints work
- [ ] Next.js login works
- [ ] No errors in logs

---

**Let's do this! 🚀**
