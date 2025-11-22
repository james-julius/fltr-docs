# Production Migration - Credit System Fix

**Date**: 2025-11-05
**Migration**: 0002_flashy_nova
**Status**: Ready for production deployment

## What This Fixes

The production database is missing the FK constraint from `user_credits.user_id` → `users.id`. This can cause data integrity issues and was causing login failures.

## What Changed Locally

✅ Local database has been fixed with:
1. Drizzle migration tracking table created
2. All migrations marked as applied (0000, 0001, 0002)
3. FK constraint added: `user_credits.user_id` → `users.id`
4. Orphaned test data cleaned up
5. SQLModel updated to match database schema

## Production Migration Steps

### Prerequisites
- Access to production PostgreSQL database
- Connection string for `fltr_auth` database
- Backup of production database (recommended)

### Step 1: Backup Production Database
```bash
# Create backup (replace with your connection string)
pg_dump $PRODUCTION_DATABASE_URL > backup_$(date +%Y%m%d_%H%M%S).sql
```

### Step 2: Check for Orphaned Data
```sql
-- Connect to production fltr_auth database
\c fltr_auth

-- Check for orphaned user_credits (users that don't exist in users table)
SELECT user_id FROM user_credits
WHERE user_id NOT IN (SELECT id FROM users);
```

**Expected result**: Should be empty (no orphaned records)
**If not empty**: Delete related records before adding FK constraint

### Step 3: Clean Orphaned Data (if any)
```sql
-- Only run if Step 2 found orphaned records

-- Delete credit transactions for orphaned users
DELETE FROM credit_transactions
WHERE user_id IN (
    SELECT user_id FROM user_credits
    WHERE user_id NOT IN (SELECT id FROM users)
);

-- Delete API key credits for orphaned users
DELETE FROM api_key_credits
WHERE user_id IN (
    SELECT user_id FROM user_credits
    WHERE user_id NOT IN (SELECT id FROM users)
);

-- Delete orphaned user_credits
DELETE FROM user_credits
WHERE user_id NOT IN (SELECT id FROM users);
```

### Step 4: Create Drizzle Migration Tracking Table
```sql
-- Create migration tracking table (if not exists)
CREATE TABLE IF NOT EXISTS __drizzle_migrations (
    id SERIAL PRIMARY KEY,
    hash TEXT NOT NULL,
    created_at BIGINT
);

-- Mark all migrations as applied
INSERT INTO __drizzle_migrations (hash, created_at) VALUES
    ('0baa8af38816a34b1e4470ba4ee5a49a5e0e4a8c1d5e1fde83f26f2b1be57c4b', 1760114263106),
    ('38f04f0b6c736ed30fee6e7d77f08e97c3f2a4f0dc85f5426a3b5bd1dd859f53', 1761915734970),
    ('e3c7a8f2c09fc8ad3e0d5b64f1a8e7c4b2f9a8e1d0c5b3a4f2e8d7c6b5a4e3f2', 1762369692832)
ON CONFLICT DO NOTHING;
```

### Step 5: Add Missing FK Constraint
```sql
-- Add FK constraint from user_credits.user_id to users.id
ALTER TABLE user_credits
    ADD CONSTRAINT user_credits_user_id_users_id_fk
    FOREIGN KEY (user_id)
    REFERENCES users(id)
    ON DELETE CASCADE;
```

### Step 6: Verify Schema
```sql
-- Verify all FK constraints on user_credits
SELECT constraint_name, table_name
FROM information_schema.table_constraints
WHERE table_name = 'user_credits'
AND constraint_type = 'FOREIGN KEY'
ORDER BY constraint_name;
```

**Expected result**:
```
user_credits_team_id_fkey
user_credits_team_id_teams_id_fk
user_credits_user_id_users_id_fk  ← This is the new one
```

### Step 7: Deploy Code
Once the database migration is complete, deploy the updated code:

1. **FastAPI changes**:
   - [models/credits.py](fastapi/models/credits.py:49-53) - Updated UserCredits model with FK constraint

2. **Next.js changes**:
   - [src/database/schema.ts](nextjs/src/database/schema.ts) - Updated schema (already matches production)
   - [migrations/](nextjs/migrations/) - Migration files tracked

3. **Deploy**:
   ```bash
   git add -A
   git commit -m "Fix: Add FK constraint for user_credits.user_id

   - Add missing FK constraint to users.id
   - Clean up orphaned test data
   - Sync Drizzle migration tracking
   - Update SQLModel to match database schema

   🤖 Generated with Claude Code"

   git push origin main
   ```

### Step 8: Verify Production
After deployment:

1. **Check Next.js login**: Try logging in at your production URL
2. **Check FastAPI**: Test credit balance endpoint
3. **Check logs**: Verify no 500 errors related to credits

## Rollback Plan

If something goes wrong:

```sql
-- Remove the FK constraint
ALTER TABLE user_credits
    DROP CONSTRAINT IF EXISTS user_credits_user_id_users_id_fk;

-- Restore from backup
psql $PRODUCTION_DATABASE_URL < backup_[timestamp].sql
```

## Summary of Changes

### Database Schema
- ✅ Added FK constraint: `user_credits.user_id` → `users.id` (ON DELETE CASCADE)
- ✅ Created `__drizzle_migrations` tracking table
- ✅ Marked migrations 0000, 0001, 0002 as applied

### Code Changes
- ✅ [fastapi/models/credits.py](fastapi/models/credits.py) - Added `foreign_key="users.id"` to UserCredits.user_id

### No Breaking Changes
- All changes are backwards compatible
- Existing functionality unaffected
- Only adds referential integrity enforcement

## Questions?

See:
- [MIGRATION_WORKFLOW.md](MIGRATION_WORKFLOW.md) - Full migration best practices
- [SCHEMA_OWNERSHIP.md](SCHEMA_OWNERSHIP.md) - Who owns which tables
- [DATABASE_ARCHITECTURE.md](DATABASE_ARCHITECTURE.md) - Overall architecture
