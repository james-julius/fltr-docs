# Dual ORM Architecture: Drizzle + SQLModel

## Why Two ORMs?

FLTR uses **two different ORMs** working together harmoniously:
- **Drizzle ORM** (Next.js / TypeScript) - Schema definition and migrations
- **SQLModel** (FastAPI / Python) - Runtime database operations

This is intentional and follows best practices for polyglot microservices.

## The Architecture

```
┌─────────────────────────────────────────────────┐
│              PostgreSQL Databases                │
│  ┌─────────────────┐    ┌──────────────────┐   │
│  │   fltr_auth     │    │      fltr        │   │
│  │  (auth+credits) │    │   (datasets)     │   │
│  └─────────────────┘    └──────────────────┘   │
└─────────────────────────────────────────────────┘
           ▲                        ▲
           │                        │
    ┌──────┴────────┐      ┌────────┴────────┐
    │  Drizzle ORM  │      │   SQLModel      │
    │   (Schema)    │      │   (Runtime)     │
    └───────────────┘      └─────────────────┘
           ▲                        ▲
           │                        │
    ┌──────┴────────┐      ┌────────┴────────┐
    │   Next.js     │      │    FastAPI      │
    │  (TypeScript) │      │    (Python)     │
    │               │      │                 │
    │  - Frontend   │      │  - API          │
    │  - Auth (UI)  │      │  - MCP Server   │
    │  - Payments   │      │  - Embeddings   │
    └───────────────┘      │  - Credits      │
                           └─────────────────┘
```

## Division of Responsibilities

### Drizzle ORM (Next.js)
**Purpose**: Schema definition and migrations for `fltr_auth` database

**Responsibilities**:
- ✅ Define database schema in TypeScript
- ✅ Generate SQL migrations when schema changes
- ✅ Track migration history
- ✅ Type-safe database queries (Next.js only)
- ✅ Run migrations on deploy

**Files**:
- [nextjs/src/database/schema.ts](nextjs/src/database/schema.ts) - TypeScript schema definitions
- [nextjs/migrations/](nextjs/migrations/) - Generated SQL migration files
- [nextjs/migrations/meta/](nextjs/migrations/meta/) - Migration metadata and journal

**When to use**:
```typescript
// Defining new tables/columns
export const newTable = pgTable("new_table", {
  id: text("id").primaryKey(),
  name: text("name").notNull()
})
```

**Workflow**:
1. Update [schema.ts](nextjs/src/database/schema.ts)
2. Run `pnpm run db:generate` (creates migration SQL)
3. Review generated migration
4. Commit schema + migration together
5. Deploy (migrations run automatically)

### SQLModel (FastAPI)
**Purpose**: Runtime database operations for BOTH databases

**Responsibilities**:
- ✅ Query data at runtime
- ✅ Business logic operations
- ✅ Credit transactions
- ✅ Dataset/document operations
- ✅ Table creation on startup (legacy)
- ✅ Pydantic validation

**Files**:
- [fastapi/models/credits.py](fastapi/models/credits.py) - Credit system models
- [fastapi/models/dataset.py](fastapi/models/dataset.py) - Dataset models
- [fastapi/database/sql_store.py](fastapi/database/sql_store.py) - Database init

**When to use**:
```python
# Runtime queries and business logic
user_credits = session.query(UserCredits).filter_by(user_id=user_id).first()
user_credits.credits -= cost
session.commit()
```

**Workflow**:
1. Update model in [models/credits.py](fastapi/models/credits.py)
2. Mirror the Drizzle schema structure
3. Use in API endpoints/services
4. Let Drizzle handle migrations

## How They Work Together

### Single Source of Truth
**Drizzle defines the schema**, SQLModel mirrors it:

**Drizzle (Schema Definition)**:
```typescript
// nextjs/src/database/schema.ts
export const userCredits = pgTable("user_credits", {
    userId: text("user_id")
        .primaryKey()
        .references(() => users.id, { onDelete: "cascade" }),
    credits: integer("credits").default(0).notNull(),
    totalCreditsPurchased: integer("total_credits_purchased").default(0).notNull(),
    // ...
})
```

**SQLModel (Runtime Mirror)**:
```python
# fastapi/models/credits.py
class UserCredits(SQLModel, table=True):
    __tablename__ = "user_credits"

    user_id: str = Field(
        primary_key=True,
        foreign_key="users.id",
        alias="userId"
    )
    credits: int = Field(default=0)
    total_credits_purchased: int = Field(
        default=0,
        alias="totalCreditsPurchased"
    )
    # ...
```

Both define the **same table**, just in their native language!

### Schema Change Workflow

When schema needs to change:

1. **Update Drizzle schema** (source of truth)
   ```bash
   # Edit nextjs/src/database/schema.ts
   vim nextjs/src/database/schema.ts
   ```

2. **Generate migration**
   ```bash
   cd nextjs
   pnpm run db:generate
   ```

3. **Update SQLModel to match**
   ```bash
   # Edit fastapi/models/credits.py
   vim fastapi/models/credits.py
   ```

4. **Test locally**
   ```bash
   psql $DATABASE_URL < migrations/0003_new_migration.sql
   ```

5. **Commit both together**
   ```bash
   git add nextjs/src/database/schema.ts
   git add nextjs/migrations/0003_*
   git add fastapi/models/credits.py
   git commit -m "Add new_field to user_credits"
   ```

6. **Deploy** - Vercel runs migrations automatically

## Benefits of This Approach

### 1. Best Tool for Each Job
- **Drizzle**: Type-safe migrations, TypeScript-first, excellent DX for Next.js
- **SQLModel**: Pydantic integration, perfect for FastAPI, Python-first

### 2. Separation of Concerns
- **Next.js** owns the schema definition (`fltr_auth` tables)
- **FastAPI** owns the business logic (credit operations, embeddings)
- Clear boundaries prevent conflicts

### 3. Type Safety Everywhere
- **TypeScript**: Drizzle generates types from schema
- **Python**: SQLModel uses Pydantic for validation
- Both catch errors at compile time

### 4. Independent Scaling
- Next.js can query `fltr_auth` for auth/user data
- FastAPI can query both `fltr` (datasets) and `fltr_auth` (credits)
- Each service uses its optimal stack

### 5. Migration Safety
- Drizzle tracks all migrations in `__drizzle_migrations` table
- SQL migrations are explicit and reviewable
- Rollback strategy built-in

## Common Patterns

### Pattern 1: Adding a New Column
```typescript
// 1. Drizzle - Add column to schema
export const userCredits = pgTable("user_credits", {
    // ... existing columns ...
    lastUsedAt: timestamp("last_used_at"), // NEW
})
```

```bash
# 2. Generate migration
pnpm run db:generate
```

```python
# 3. SQLModel - Mirror the change
class UserCredits(SQLModel, table=True):
    # ... existing fields ...
    last_used_at: Optional[datetime] = Field(default=None, alias="lastUsedAt")
```

### Pattern 2: Querying Data (FastAPI)
```python
from models.credits import UserCredits
from database.sql_store import get_auth_session

def get_user_credits(user_id: str):
    session = next(get_auth_session())  # Auth database
    return session.query(UserCredits).filter_by(user_id=user_id).first()
```

### Pattern 3: Querying Data (Next.js)
```typescript
import { db } from '@/database/db'
import { userCredits } from '@/database/schema'
import { eq } from 'drizzle-orm'

async function getUserCredits(userId: string) {
  return await db.select()
    .from(userCredits)
    .where(eq(userCredits.userId, userId))
}
```

## Migration Tracking

### Drizzle Migrations Table
```sql
CREATE TABLE __drizzle_migrations (
    id SERIAL PRIMARY KEY,
    hash TEXT NOT NULL,
    created_at BIGINT
);
```

This table tracks which migrations have been applied:
- `id`: Sequential migration number
- `hash`: Unique hash of migration file
- `created_at`: Unix timestamp

### Why Not SQLAlchemy Migrations (Alembic)?
- Alembic requires Python-first workflow
- Drizzle is more TypeScript-native
- We want Next.js (frontend) to own the auth database schema
- FastAPI just "reads" the schema at runtime

## Troubleshooting

### Schema Drift Detected
```bash
# Check if schemas match
cd nextjs
pnpm run db:generate
```

If output says "No schema changes" → ✅ In sync
If it generates new migration → ⚠️ Drift detected

**Fix**: Apply the migration or update schema to match database

### SQLModel vs Drizzle Mismatch
**Symptom**: Errors like "column doesn't exist" or FK constraint violations

**Fix**:
1. Drizzle is source of truth - check [schema.ts](nextjs/src/database/schema.ts)
2. Compare with SQLModel in [credits.py](fastapi/models/credits.py)
3. Update SQLModel to match Drizzle
4. Regenerate migrations if needed

### Missing Migration Tracking
**Symptom**: `ERROR: relation "__drizzle_migrations" does not exist`

**Fix**: See [PRODUCTION_MIGRATION.md](PRODUCTION_MIGRATION.md) Step 4

## Related Documentation

- [SCHEMA_OWNERSHIP.md](SCHEMA_OWNERSHIP.md) - Who owns which tables
- [MIGRATION_WORKFLOW.md](MIGRATION_WORKFLOW.md) - Detailed migration process
- [PRODUCTION_MIGRATION.md](PRODUCTION_MIGRATION.md) - Current production migration steps
- [DATABASE_ARCHITECTURE.md](DATABASE_ARCHITECTURE.md) - Overall architecture

## Summary

**Drizzle + SQLModel = Perfect Together**

- 🎯 **Drizzle** = Schema & Migrations (TypeScript-first)
- ⚡ **SQLModel** = Runtime Operations (Python-first)
- 🔄 **Both** work with the same PostgreSQL databases
- ✅ **Result** = Type-safe, scalable, maintainable architecture

This is a **best practice** for polyglot microservices!
