# Migration Workflow & Best Practices

## Pre-Migration Checklist

Before making ANY schema change:

1. ✅ Check `SCHEMA_OWNERSHIP.md` - which service owns this table?
2. ✅ Check both codebases - does FastAPI also use this table?
3. ✅ Review existing migrations - what's the current state?
4. ✅ Check production database - what's actually deployed?

## Next.js (Drizzle) Migrations

### Step 1: Update Schema
```bash
cd nextjs
# Edit src/database/schema.ts
```

### Step 2: Generate Migration
```bash
pnpm run db:generate
# Creates migrations/XXXX_name.sql
```

### Step 3: Review Generated SQL
```bash
cat migrations/XXXX_name.sql

# Check for:
# ❌ Dropping columns with data
# ❌ Adding NOT NULL without defaults
# ❌ Breaking foreign keys
# ✅ Using IF EXISTS for safety
# ✅ Data migration paths
```

### Step 4: Test Locally First
```bash
# Apply to local database
pnpm run db:migrate

# Or manually:
psql postgresql://admin:password@127.0.0.1:5432/fltr_auth < migrations/XXXX_name.sql
```

### Step 5: Verify Tables
```sql
\dt                    -- List all tables
\d users               -- Describe users table
\d user_credits        -- Describe user_credits table
SELECT COUNT(*) FROM user_credits;  -- Check data migrated
```

### Step 6: Update FastAPI Models (If Shared Table)
```bash
cd ../fastapi
# Edit models/credits.py if table is shared
```

### Step 7: Commit
```bash
git add src/database/schema.ts migrations/
git commit -m "feat: add user_credits table"
git push
```

### Step 8: Deploy
- Vercel automatically runs migrations during build
- Check build logs for migration success
- Verify production database after deploy

---

## FastAPI (SQLModel) Schema Changes

### For Auth Database Tables (user_credits, credit_transactions)

**Option 1: Generate Drizzle Migration (Recommended)**

Since Next.js owns the migration system for fltr_auth:

```bash
cd nextjs

# 1. Update Next.js schema
# Edit src/database/schema.ts

# 2. Generate migration
pnpm run db:generate

# 3. Update FastAPI model to match
cd ../fastapi
# Edit models/credits.py
```

**Option 2: Manual Migration (For Complex Changes)**

```bash
# 1. Write SQL migration
cd fastapi/migrations
nano 002_add_credit_expiry.sql

# 2. Apply manually
psql $AUTH_DATABASE_URL < migrations/002_add_credit_expiry.sql

# 3. Update FastAPI model
# Edit models/credits.py

# 4. Add to Next.js schema
cd ../../nextjs
# Edit src/database/schema.ts
# Run: pnpm run db:generate
```

### For Main Database Tables (datasets, documents)

```bash
cd fastapi

# 1. Update SQLModel
# Edit models/dataset.py

# 2. For production, write SQL migration
cd migrations
nano 003_add_dataset_tags.sql

# 3. Apply to production
psql $DATABASE_URL < migrations/003_add_dataset_tags.sql
```

---

## Common Scenarios

### Adding a Column

**To user_credits (credit system):**
```bash
cd nextjs
# Edit src/database/schema.ts - add column to userCredits
pnpm run db:generate
pnpm run db:migrate  # Test locally

cd ../fastapi
# Edit models/credits.py - add field to UserCredits class
```

**To datasets (FastAPI only):**
```bash
cd fastapi
# Edit models/dataset.py - add field
# SQLModel auto-creates on next startup (development)
# For production: Write SQL migration
```

### Removing a Column

```bash
# 1. Deploy code that stops using the column
# 2. Wait 24-48 hours (rollback safety)
# 3. Then drop the column via migration
```

### Renaming a Column

```bash
# Use a 3-step migration:
# 1. Add new column
# 2. Copy data from old to new
# 3. Drop old column (after deploy + wait period)
```

### Adding a Foreign Key

```bash
# Ensure referential integrity BEFORE adding FK:
DELETE FROM credit_transactions
WHERE user_id NOT IN (SELECT id FROM users);

ALTER TABLE credit_transactions
ADD CONSTRAINT credit_transactions_user_id_fkey
FOREIGN KEY (user_id) REFERENCES users(id);
```

---

## Safety Rules

### ❌ Never Do This

1. **Drop columns immediately**
   - Old code may still reference them
   - Deploy code changes first, wait, then drop

2. **Add NOT NULL without default**
   ```sql
   -- ❌ BAD
   ALTER TABLE users ADD COLUMN status text NOT NULL;

   -- ✅ GOOD
   ALTER TABLE users ADD COLUMN status text DEFAULT 'active' NOT NULL;
   ```

3. **Run migrations manually in production without testing locally**

4. **Mix credit system changes across both databases**
   - Keep credits in fltr_auth only

5. **Skip migration files and edit schema directly**

### ✅ Always Do This

1. **Test migrations locally first**
   ```bash
   # On local fltr_auth database
   psql postgresql://admin:password@127.0.0.1:5432/fltr_auth < migrations/XXXX.sql
   ```

2. **Add IF EXISTS / IF NOT EXISTS**
   ```sql
   CREATE TABLE IF NOT EXISTS user_credits (...);
   ALTER TABLE users DROP COLUMN IF EXISTS credits;
   ```

3. **Use DEFAULT values**
   ```sql
   ALTER TABLE users ADD COLUMN status text DEFAULT 'active' NOT NULL;
   ```

4. **Check foreign keys**
   ```sql
   -- Verify data integrity before adding FK
   SELECT COUNT(*) FROM credit_transactions ct
   LEFT JOIN users u ON ct.user_id = u.id
   WHERE u.id IS NULL;
   ```

5. **Commit migrations with code changes**
   - Migration + code change = one commit
   - Makes rollbacks easier

---

## Rollback Strategy

### If Migration Fails During Deploy

**Option 1: Fix Forward**
```bash
# Create a new migration that fixes the issue
cd nextjs
# Edit schema to fix
pnpm run db:generate
git add migrations/
git commit -m "fix: correct migration issue"
git push
```

**Option 2: Rollback (Use Carefully)**
```bash
# 1. Revert the commit
git revert HEAD

# 2. Manually undo migration in production
psql $DATABASE_URL

# Run the inverse operations:
ALTER TABLE users ADD COLUMN credits integer;
DROP TABLE user_credits;
```

### If Data is Corrupted

```bash
# Restore from backup
# Your database provider should have point-in-time recovery
# Neon, Supabase, Vercel Postgres all have this feature
```

---

## Migration Testing Checklist

Before deploying ANY migration:

- [ ] Tested on local database
- [ ] Verified data integrity
- [ ] Checked foreign key constraints
- [ ] Updated FastAPI models (if shared table)
- [ ] Code still works with old schema (deploy code first if needed)
- [ ] Rollback plan documented
- [ ] Production backup confirmed available

---

## Monitoring Migrations

### Check Migration Status

**Next.js (Drizzle):**
```sql
-- Check what migrations have run
SELECT * FROM drizzle.__drizzle_migrations;
```

**Production:**
```bash
# Check Vercel build logs
# Look for:
# 🔄 Running database migrations...
# ✅ Database migrations completed successfully
```

### Verify Schema Matches Code

```bash
# Generate a fresh migration from current schema
cd nextjs
pnpm run db:generate

# If it generates nothing, schema matches database ✅
# If it generates changes, database is out of sync ❌
```

---

## Emergency Procedures

### Production is Down Due to Migration

1. **Check Vercel logs** - find the error
2. **Quick fix options:**
   - Revert deploy to previous version
   - Or: Push fix-forward migration
3. **Never manually ALTER production while app is running**
   - Can cause lock contention
   - Use maintenance mode if available

### Schema Drift Detected

If local and production schemas don't match:

```bash
# 1. Export production schema
pg_dump $PROD_DATABASE_URL --schema-only > prod_schema.sql

# 2. Export local schema
pg_dump postgresql://localhost/fltr_auth --schema-only > local_schema.sql

# 3. Diff them
diff prod_schema.sql local_schema.sql

# 4. Generate migration to bring local to match production
# Then deploy
```

---

## Summary

**Golden Rules:**
1. **Next.js owns fltr_auth migrations** (via Drizzle)
2. **FastAPI owns fltr migrations** (via manual SQL)
3. **Always test locally first**
4. **Never skip the migration file**
5. **Keep schema and code in sync**

**When in Doubt:**
- Check `SCHEMA_OWNERSHIP.md`
- Test locally
- Ask before dropping columns
- Document breaking changes
