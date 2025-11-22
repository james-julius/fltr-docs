# Expected vs Actual Database Schemas

## 🎯 What We Want (Target State)

### `fltr` Database (Dataset/Document Management)
**Purpose**: Stores all dataset and document processing data

**Tables**:
```
datasets
├─ id (PK)
├─ user_id (FK → fltr_auth.users.id)
├─ name
├─ description
├─ created_at
└─ updated_at

documents
├─ id (PK)
├─ dataset_id (FK → datasets.id)
├─ filename
├─ content
├─ status
├─ created_at
└─ updated_at

processing_jobs
├─ id (PK)
├─ document_id (FK → documents.id)
├─ status
├─ created_at
└─ updated_at
```

**NO users table here!** ❌

---

### `fltr_auth` Database (Auth & Credits)
**Purpose**: Authentication, user management, and credit system

**Tables**:
```
users (Better Auth - NO credit columns)
├─ id (PK)
├─ name
├─ email (UNIQUE)
├─ email_verified
├─ image
├─ avatar
├─ avatar_url
├─ created_at
├─ updated_at
└─ stripe_customer_id

sessions (Better Auth)
├─ id (PK)
├─ user_id (FK → users.id CASCADE)
├─ token (UNIQUE)
├─ expires_at
├─ created_at
├─ updated_at
├─ ip_address
└─ user_agent

accounts (Better Auth)
├─ id (PK)
├─ user_id (FK → users.id CASCADE)
├─ account_id
├─ provider_id
├─ access_token
├─ refresh_token
├─ ... (OAuth fields)
├─ created_at
└─ updated_at

verifications (Better Auth)
├─ id (PK)
├─ identifier
├─ value
├─ expires_at
├─ created_at
└─ updated_at

subscriptions (Stripe)
├─ id (PK)
├─ plan
├─ reference_id
├─ stripe_customer_id
├─ stripe_subscription_id
├─ status
├─ period_start
├─ period_end
├─ cancel_at_period_end
├─ seats
├─ trial_start
└─ trial_end

teams (Credit System)
├─ id (PK)
├─ name
├─ credits
├─ owner_id (FK → users.id CASCADE) ✅
├─ created_at
└─ updated_at

user_credits (Credit System - SEPARATE TABLE)
├─ user_id (PK, FK → users.id CASCADE) ✅
├─ credits (default 0)
├─ total_credits_purchased (default 0)
├─ team_id (FK → teams.id)
├─ created_at
└─ updated_at

credit_transactions (Credit System)
├─ id (PK)
├─ user_id (FK → users.id CASCADE) ✅
├─ team_id (FK → teams.id SET NULL)
├─ amount
├─ type (purchase/usage/refund/bonus)
├─ operation (document_upload/mcp_query/etc)
├─ resource_id
├─ api_key_id
├─ transaction_metadata (JSON)
└─ created_at

api_key_credits (Credit System)
├─ api_key (PK)
├─ user_id (FK → users.id CASCADE) ✅
├─ team_id (FK → teams.id CASCADE)
├─ credits
├─ daily_limit
├─ monthly_limit
├─ created_at
└─ updated_at

__drizzle_migrations (Drizzle Tracking)
├─ id (SERIAL PK)
├─ hash
└─ created_at
```

---

## 📊 Current State

### Local `fltr` Database
**Status**: ✅ **EMPTY (Just Nuked)**
- Currently has no tables (we just dropped the schema)
- Ready to be rebuilt with correct tables

### Local `fltr_auth` Database
**Status**: ✅ **CORRECT (Already has proper structure)**
- Has all tables with correct structure
- Has `user_credits` as separate table
- Has `users` WITHOUT credit columns
- FKs are correct

### Production `fltr` Database
**Status**: ❌ **HAS WRONG users TABLE**

**Current tables**:
```
users (SHOULD NOT EXIST HERE)
├─ id
├─ email
├─ name
├─ email_verified
├─ image
├─ stripe_customer_id
├─ credits ❌ (should be in user_credits)
├─ total_credits_purchased ❌ (should be in user_credits)
├─ team_id ❌ (should be in user_credits)
├─ created_at
└─ updated_at
```

**What it SHOULD have**:
- datasets table only
- documents table only
- processing_jobs table only

### Production `fltr_auth` Database
**Status**: ❌ **MISSING user_credits TABLE**

**Current users table** (WRONG):
```
users
├─ id
├─ name
├─ email
├─ email_verified
├─ image
├─ avatar
├─ avatar_url
├─ created_at
├─ updated_at
├─ stripe_customer_id
├─ team_id ❌ (should be in user_credits)
├─ total_credits_purchased ❌ (should be in user_credits)
└─ credits ❌ (should be in user_credits)
```

**Missing**: `user_credits` table entirely

---

## 🔄 Foreign Key Relationships (Target)

### Auth Database FKs
```
users (root table)
  ↓
├─ sessions.user_id → users.id (CASCADE)
├─ accounts.user_id → users.id (CASCADE)
├─ teams.owner_id → users.id (CASCADE)
├─ user_credits.user_id → users.id (CASCADE)
├─ credit_transactions.user_id → users.id (CASCADE)
└─ api_key_credits.user_id → users.id (CASCADE)

teams
  ↓
├─ user_credits.team_id → teams.id
├─ credit_transactions.team_id → teams.id (SET NULL)
└─ api_key_credits.team_id → teams.id (CASCADE)
```

**Key Point**: All credit-related FKs reference `users.id` directly, NOT `user_credits.user_id`

### Main Database FKs (fltr)
```
fltr_auth.users (in other database)
  ↓
datasets.user_id → fltr_auth.users.id (cross-database FK)
  ↓
documents.dataset_id → datasets.id (CASCADE)
  ↓
processing_jobs.document_id → documents.id (CASCADE)
```

---

## 📋 Migration Actions Needed

### Local Databases

#### ✅ `fltr` - Just rebuild
```bash
# Start FastAPI (will run init_db())
cd fastapi && python -m uvicorn main:app --port 8000

# Creates: datasets, documents, processing_jobs
```

#### ✅ `fltr_auth` - Already correct, do nothing
```bash
# Verify only:
psql $DATABASE_URL -c "\dt"
```

### Production Databases

#### ❌ `fltr` - Nuke and rebuild
```sql
-- Drop entire schema
DROP SCHEMA public CASCADE;
CREATE SCHEMA public;

-- Then let FastAPI create correct tables
```

#### ❌ `fltr_auth` - Run migration
```sql
-- Create user_credits table
CREATE TABLE user_credits (
    user_id TEXT PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    credits INTEGER NOT NULL DEFAULT 0,
    total_credits_purchased INTEGER NOT NULL DEFAULT 0,
    team_id TEXT REFERENCES teams(id),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Migrate existing data
INSERT INTO user_credits (user_id, credits, total_credits_purchased, team_id, created_at, updated_at)
SELECT id, credits, total_credits_purchased, team_id, created_at, updated_at
FROM users;

-- Drop old columns
ALTER TABLE users DROP COLUMN credits;
ALTER TABLE users DROP COLUMN total_credits_purchased;
ALTER TABLE users DROP COLUMN team_id;

-- Create migration tracking
CREATE TABLE __drizzle_migrations (
    id SERIAL PRIMARY KEY,
    hash TEXT NOT NULL,
    created_at BIGINT
);

INSERT INTO __drizzle_migrations (hash, created_at) VALUES
    ('0baa8af38816a34b1e4470ba4ee5a49a5e0e4a8c1d5e1fde83f26f2b1be57c4b', 1760114263106),
    ('38f04f0b6c736ed30fee6e7d77f08e97c3f2a4f0dc85f5426a3b5bd1dd859f53', 1761915734970),
    ('e3c7a8f2c09fc8ad3e0d5b64f1a8e7c4b2f9a8e1d0c5b3a4f2e8d7c6b5a4e3f2', 1762369692832);
```

---

## ✅ Verification Queries

### After Migration - Run These

#### Check `fltr` has correct tables
```sql
SELECT tablename FROM pg_tables
WHERE schemaname = 'public'
ORDER BY tablename;

-- Should show:
-- datasets
-- documents
-- processing_jobs
```

#### Check `fltr_auth` has correct tables
```sql
SELECT tablename FROM pg_tables
WHERE schemaname = 'public'
ORDER BY tablename;

-- Should show:
-- __drizzle_migrations
-- accounts
-- api_key_credits
-- credit_transactions
-- sessions
-- subscriptions
-- teams
-- user_credits ← MUST EXIST
-- users
-- verifications
```

#### Check `users` table structure
```sql
\d users

-- Should NOT have:
-- ❌ credits
-- ❌ total_credits_purchased
-- ❌ team_id
```

#### Check `user_credits` table exists
```sql
\d user_credits

-- Should show:
-- ✅ user_id (PK, FK to users.id)
-- ✅ credits
-- ✅ total_credits_purchased
-- ✅ team_id
-- ✅ created_at
-- ✅ updated_at
```

#### Check all FKs are correct
```sql
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
  AND tc.table_name IN ('user_credits', 'credit_transactions', 'api_key_credits', 'teams')
ORDER BY tc.table_name;

-- Should show:
-- user_credits.user_id → users.id ✅
-- user_credits.team_id → teams.id ✅
-- credit_transactions.user_id → users.id ✅
-- credit_transactions.team_id → teams.id ✅
-- api_key_credits.user_id → users.id ✅
-- api_key_credits.team_id → teams.id ✅
-- teams.owner_id → users.id ✅
```

---

## 🎯 Summary

### Database Separation Logic

**fltr database = Data/Processing**
- Datasets
- Documents
- Processing jobs
- Vector embeddings (Milvus)

**fltr_auth database = Users/Auth/Money**
- User accounts
- Authentication (Better Auth)
- Credit balances (`user_credits` table)
- Credit transactions
- Teams
- Subscriptions

### Key Architectural Decision

**Credits are in a separate `user_credits` table because:**
1. FastAPI owns 90% of credit operations
2. Better separation of concerns
3. Easier to scale credit system independently
4. Cleaner audit trail with `credit_transactions`

### Schema Ownership

- **Drizzle (Next.js)**: Defines schema for `fltr_auth`
- **SQLModel (FastAPI)**: Mirrors Drizzle, adds `fltr` tables
- **Drizzle is source of truth** for schema definitions
- **Both must stay in sync** for FK constraints to work

---

Ready to proceed with the clean slate migration!
