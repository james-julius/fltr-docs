# Schema Verification Report
**Date**: 2025-11-05
**Status**: ⚠️ **CRITICAL MISMATCHES FOUND**

## Executive Summary

The local database schema has **3 critical mismatches** between what exists in the database vs. what's defined in code:

| Issue | Database (Reality) | SQLModel | Drizzle | Status |
|-------|-------------------|----------|---------|--------|
| `credit_transactions.user_id` FK | → `user_credits.user_id` | ✓ Matches | ✗ → `users.id` | **MISMATCH** |
| `api_key_credits.user_id` FK | → `user_credits.user_id` | ✓ Matches | ✗ → `users.id` | **MISMATCH** |
| `teams.owner_id` FK | → `users.id` | ✗ → `user_credits.user_id` | ✓ Matches | **MISMATCH** |

## What This Means

The database was created by **FastAPI's SQLModel** (via `init_auth_db()`), but **Drizzle schema definitions don't match**. This creates two problems:

1. **Drizzle is supposed to be source of truth** but it's out of sync
2. **If we generate new migrations**, Drizzle will try to "fix" FKs that are actually correct

## Detailed Comparison

### 1. users Table ✅
**Status**: Perfect match across all three

**Database**:
```sql
Table "users"
- id (text, PK)
- name (text, NOT NULL)
- email (text, NOT NULL, UNIQUE)
- email_verified (boolean, NOT NULL)
- image (text)
- avatar (text)
- avatar_url (text)
- created_at (timestamp, NOT NULL)
- updated_at (timestamp, NOT NULL)
- stripe_customer_id (text)
```

**Drizzle** ([schema.ts:3-20](nextjs/src/database/schema.ts:3-20)): ✅ Matches
**SQLModel**: N/A (Better Auth owns this table)

---

### 2. user_credits Table ⚠️
**Status**: Schema matches, but column types differ slightly

**Database**:
```sql
Table "user_credits"
- user_id (character varying, PK, FK → users.id CASCADE) ✅
- credits (integer, NOT NULL, default 0)
- total_credits_purchased (integer, NOT NULL, default 0)
- team_id (character varying, FK → teams.id)
- created_at (timestamp, NOT NULL)
- updated_at (timestamp, NOT NULL)
```

**Drizzle** ([schema.ts:96-109](nextjs/src/database/schema.ts:96-109)): ✅ Matches structure
**SQLModel** ([credits.py:49-62](fastapi/models/credits.py:49-62)): ✅ Matches structure

**Minor Issue**:
- Database uses `character varying` (from SQLModel)
- Drizzle defines `text`
- **Impact**: None (functionally equivalent in PostgreSQL)

---

### 3. teams Table ❌
**Status**: CRITICAL FK MISMATCH

**Database**:
```sql
Table "teams"
- id (text, PK)
- name (text, NOT NULL)
- credits (integer, NOT NULL, default 0)
- owner_id (text, FK → users.id CASCADE) ← ACTUAL FK
- created_at (timestamp, NOT NULL)
- updated_at (timestamp, NOT NULL)
```

**Drizzle** ([schema.ts:82-93](nextjs/src/database/schema.ts:82-93)):
```typescript
ownerId: text("owner_id").references(() => users.id, { onDelete: "cascade" })
```
✅ **Correct**: FK to `users.id`

**SQLModel** ([credits.py:72-73](fastapi/models/credits.py:72-73)):
```python
owner_id: Optional[str] = Field(
    default=None, foreign_key="user_credits.user_id", alias="ownerId")
```
❌ **WRONG**: FK to `user_credits.user_id`

**Root Cause**: SQLModel definition is incorrect, but database has correct FK (created by earlier migration)

**Fix Needed**: Update SQLModel to match Drizzle

---

### 4. credit_transactions Table ❌
**Status**: CRITICAL FK MISMATCH

**Database**:
```sql
Table "credit_transactions"
- id (character varying, PK)
- user_id (character varying, FK → user_credits.user_id, indexed) ← ACTUAL FK
- team_id (character varying, FK → teams.id)
- amount (integer, NOT NULL)
- type (character varying, NOT NULL, indexed)
- operation (character varying)
- resource_id (character varying)
- api_key_id (character varying)
- transaction_metadata (text)
- created_at (timestamp, NOT NULL)
```

**Drizzle** ([schema.ts:111-126](nextjs/src/database/schema.ts:111-126)):
```typescript
userId: text("user_id")
    .notNull()
    .references(() => users.id, { onDelete: "cascade" })
```
❌ **WRONG**: FK to `users.id`

**SQLModel** ([credits.py:85](fastapi/models/credits.py:85)):
```python
user_id: str = Field(foreign_key="user_credits.user_id", index=True, alias="userId")
```
✅ **Correct**: FK to `user_credits.user_id`

**Root Cause**: Drizzle schema is incorrect

**Why the database FK is correct**:
- Credit transactions should reference the credit record, not the user directly
- This ensures referential integrity (can't delete user_credits while transactions exist)
- Follows normalized database design

**Fix Needed**: Update Drizzle to match SQLModel

---

### 5. api_key_credits Table ❌
**Status**: CRITICAL FK MISMATCH

**Database**:
```sql
Table "api_key_credits"
- api_key (character varying, PK)
- user_id (character varying, FK → user_credits.user_id) ← ACTUAL FK
- team_id (character varying, FK → teams.id)
- credits (integer, NOT NULL, default 0)
- daily_limit (integer)
- monthly_limit (integer)
- created_at (timestamp, NOT NULL)
- updated_at (timestamp, NOT NULL)
```

**Drizzle** ([schema.ts:128-143](nextjs/src/database/schema.ts:128-143)):
```typescript
userId: text("user_id")
    .notNull()
    .references(() => users.id, { onDelete: "cascade" })
```
❌ **WRONG**: FK to `users.id`

**SQLModel** ([credits.py:105](fastapi/models/credits.py:105)):
```python
user_id: str = Field(foreign_key="user_credits.user_id", alias="userId")
```
✅ **Correct**: FK to `user_credits.user_id`

**Root Cause**: Same as credit_transactions - Drizzle schema is incorrect

**Fix Needed**: Update Drizzle to match SQLModel

---

## Foreign Key Architecture Analysis

### Current Database Reality (CORRECT)

```
users (Better Auth)
  ↓ (1:1)
user_credits ← credit_transactions.user_id
             ← api_key_credits.user_id

teams.owner_id → users.id
```

**This makes sense because**:
1. `credit_transactions` records operations on credit accounts → should FK to `user_credits`
2. `api_key_credits` are credit pools → should FK to `user_credits`
3. `teams.owner_id` references the team owner → should FK to `users` (person, not credits)

### Drizzle's Incorrect View

```
users (Better Auth)
  ↓ (1:1)
user_credits

credit_transactions.user_id → users.id  ❌ WRONG
api_key_credits.user_id → users.id      ❌ WRONG
teams.owner_id → users.id               ✅ CORRECT
```

### SQLModel's View (Mostly Correct)

```
users (Better Auth)
  ↓ (1:1)
user_credits ← credit_transactions.user_id  ✅ CORRECT
             ← api_key_credits.user_id      ✅ CORRECT

teams.owner_id → user_credits.user_id       ❌ WRONG
```

---

## Required Fixes

### Fix 1: Update Drizzle Schema (credit_transactions)

**File**: [nextjs/src/database/schema.ts:113-115](nextjs/src/database/schema.ts:113-115)

**Change from**:
```typescript
userId: text("user_id")
    .notNull()
    .references(() => users.id, { onDelete: "cascade" }),
```

**Change to**:
```typescript
userId: text("user_id")
    .notNull()
    .references(() => userCredits.userId, { onDelete: "cascade" }),
```

### Fix 2: Update Drizzle Schema (api_key_credits)

**File**: [nextjs/src/database/schema.ts:130-132](nextjs/src/database/schema.ts:130-132)

**Change from**:
```typescript
userId: text("user_id")
    .notNull()
    .references(() => users.id, { onDelete: "cascade" }),
```

**Change to**:
```typescript
userId: text("user_id")
    .notNull()
    .references(() => userCredits.userId, { onDelete: "cascade" }),
```

### Fix 3: Update SQLModel (teams)

**File**: [fastapi/models/credits.py:72-73](fastapi/models/credits.py:72-73)

**Change from**:
```python
owner_id: Optional[str] = Field(
    default=None, foreign_key="user_credits.user_id", alias="ownerId")
```

**Change to**:
```python
owner_id: Optional[str] = Field(
    default=None, foreign_key="users.id", alias="ownerId")
```

---

## Action Plan

1. **Apply Drizzle fixes** (credit_transactions and api_key_credits)
2. **Apply SQLModel fix** (teams)
3. **Verify no schema drift**:
   ```bash
   cd nextjs
   pnpm run db:generate
   ```
   Should output: "No schema changes, nothing to migrate 😴"

4. **Test locally**:
   - Start FastAPI and verify no errors
   - Test credit operations
   - Test team operations

5. **Commit changes**:
   ```bash
   git add nextjs/src/database/schema.ts fastapi/models/credits.py
   git commit -m "Fix: Align Drizzle and SQLModel FK constraints"
   ```

6. **Deploy to production** (no migration needed - database is already correct!)

---

## Why This Happened

1. **FastAPI created tables first** via `init_auth_db()` using SQLModel
2. **SQLModel had mostly correct FK constraints** (except teams.owner_id)
3. **Drizzle schema was written later** and didn't match the existing database
4. **Database matches SQLModel** because SQLModel created it
5. **Drizzle thinks it's source of truth** but it's actually out of sync

## Risk Assessment

**Current Risk**: 🟡 Medium
- Database is functionally correct
- Both services query correctly
- But schema drift will cause issues if we generate new migrations

**After Fixes**: 🟢 Low
- All three sources of truth will match
- Future migrations will be accurate
- No risk of accidental schema changes

---

## Summary

✅ **Database structure is correct** (created by SQLModel)
⚠️ **Drizzle schema has 2 wrong FK constraints**
⚠️ **SQLModel has 1 wrong FK constraint**
🔧 **3 simple fixes needed before production deployment**

Once fixed, all three sources of truth will align perfectly.
