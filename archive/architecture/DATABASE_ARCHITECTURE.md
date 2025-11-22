# FLTR Database Architecture

## Overview

FLTR uses a **two-database architecture** to separate concerns between authentication/user management and dataset/document services. This design follows microservices best practices while maintaining operational simplicity for early-stage development.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    FLTR Platform                        │
└─────────────────────────────────────────────────────────┘
                              │
                              ▼
         ┌────────────────────┴────────────────────┐
         │                                          │
         ▼                                          ▼
┌─────────────────┐                        ┌─────────────────┐
│    Next.js      │                        │    FastAPI      │
│  Auth Service   │                        │ Dataset Service │
│                 │                        │                 │
│  Port: 3000     │                        │  Port: 8000     │
└─────────────────┘                        └─────────────────┘
         │                                          │
         │ DATABASE_URL                             │ DATABASE_URL
         ▼                                          ▼
┌─────────────────┐                        ┌─────────────────┐
│  fltr_auth DB   │◄───────────────────────│   fltr DB       │
│                 │   AUTH_DATABASE_URL    │                 │
│  [Auth Tables]  │   (cross-database      │  [Dataset       │
│  • users        │    connection)         │   Tables]       │
│  • sessions     │                        │  • datasets     │
│  • accounts     │                        │  • documents    │
│  • subs         │                        │  • proc_jobs    │
│                 │                        │                 │
│  [Credit Tables]│                        │  owner_id: UUID │
│  • user_credits │                        │  (references    │
│  • credit_trans │                        │   users.id)     │
│  • api_key_cred │                        │                 │
│  • teams        │                        │                 │
└─────────────────┘                        └─────────────────┘
```

## Database Ownership

### Database 1: `fltr_auth` (Authentication & Credits)

**Managed By**: Next.js service
**ORM**: Drizzle ORM
**Purpose**: User identity, authentication, subscriptions, and credit management

#### Tables:

##### Core Authentication Tables (Better Auth)
- **users**: User accounts and profiles
  - Primary key: `id` (text/UUID)
  - Fields: name, email, image, stripe_customer_id, credits, team_id
  - Indexed: email (unique), team_id

- **sessions**: Active user sessions
  - Primary key: `id` (text/UUID)
  - Fields: token (unique), user_id, expires_at, ip_address, user_agent
  - Foreign keys: user_id → users.id (cascade delete)
  - Indexed: token (unique), user_id

- **accounts**: OAuth provider accounts
  - Primary key: `id` (text/UUID)
  - Fields: provider_id, user_id, access_token, refresh_token, etc.
  - Foreign keys: user_id → users.id (cascade delete)

- **verifications**: Email verification tokens
  - Primary key: `id` (text/UUID)
  - Fields: identifier, value, expires_at

- **subscriptions**: Stripe subscription records
  - Primary key: `id` (text/UUID)
  - Fields: stripe_subscription_id, status, period_start, period_end, seats

##### Credit System Tables
- **user_credits**: Credit balances per user
  - NOTE: This is represented by the `credits` field in the `users` table
  - Not a separate table

- **credit_transactions**: Audit log for all credit operations
  - Primary key: `id` (text/UUID)
  - Fields: user_id, amount, type, operation, resource_id, metadata
  - Types: 'purchase', 'usage', 'refund', 'bonus'
  - Operations: 'document_upload', 'mcp_query', 'embedding_search'
  - Foreign keys: user_id → users.id (cascade delete)
  - Indexed: user_id, type, created_at

- **api_key_credits**: Per-API-key credit pools
  - Primary key: `api_key` (text)
  - Fields: user_id, team_id, credits, daily_limit, monthly_limit
  - Foreign keys:
    - user_id → users.id (cascade delete)
    - team_id → teams.id (cascade delete)

- **teams**: Team/organization records (future functionality)
  - Primary key: `id` (text/UUID)
  - Fields: name, credits, owner_id
  - Foreign keys: owner_id → users.id (cascade delete)

**Schema Location**: [nextjs/src/database/schema.ts](nextjs/src/database/schema.ts)
**Migrations**: [nextjs/migrations/](nextjs/migrations/)

---

### Database 2: `fltr` (Datasets & Documents)

**Managed By**: FastAPI service
**ORM**: SQLModel (SQLAlchemy + Pydantic)
**Purpose**: Dataset management, document storage, and processing

#### Tables:

- **datasets**: Dataset metadata and configuration
  - Primary key: `id` (UUID)
  - Fields:
    - owner_id (UUID) - References users.id in fltr_auth (NO FK CONSTRAINT)
    - name, slug, description, category
    - visibility, status, collection_name (Milvus)
    - pricing_model, price_monthly, price_annual
    - document_count, rating_average, download_count
  - Indexed: name, slug (unique), category, status, owner_id, collection_name

- **documents**: Uploaded files and processing status
  - Primary key: `id` (UUID)
  - Fields: dataset_id, filename, object_key (R2), file_size, mime_type
  - Fields: status, chunk_count, error
  - Foreign keys: dataset_id → datasets.id (cascade delete)
  - Indexed: dataset_id, status

- **processing_jobs**: Background job tracking
  - Primary key: `id` (UUID)
  - Fields: dataset_id, task_id, status, error, retries
  - Foreign keys: dataset_id → datasets.id (cascade delete)
  - Indexed: dataset_id, status

**Schema Location**: [fastapi/models/](fastapi/models/)
**Migrations**: [fastapi/migrations/](fastapi/migrations/)

---

## Design Rationale

### Why Two Databases?

1. **Service Boundaries**: Clear separation between auth (Next.js) and datasets (FastAPI)
2. **Independent Scaling**: Each service can scale its database independently
3. **Technology Freedom**: Next.js uses Drizzle, FastAPI uses SQLModel/SQLAlchemy
4. **Single Source of Truth**: User data lives in one place (fltr_auth)
5. **Operational Simplicity**: Both services share auth database, avoiding data sync complexity

### Why Not Full Microservices Isolation?

**Full isolation would require**:
- FastAPI to call Next.js API for every session validation
- Complex event-driven user sync (message queues)
- Eventual consistency handling
- Significantly more operational complexity

**Trade-off**: FastAPI has a **read-only dependency** on fltr_auth for session validation and a **read-write dependency** for credit management. This is acceptable for an early-stage product and provides:
- Fast session validation (direct DB query)
- Atomic credit operations (no distributed transactions)
- Simpler architecture (no message queues yet)

---

## Cross-Database Patterns

### Pattern 1: Session Validation (FastAPI → Auth DB)

**Use Case**: Validate user session tokens in FastAPI endpoints

**Flow**:
1. Client sends request with `Authorization: Bearer <token>` header
2. FastAPI middleware extracts token
3. Middleware queries `fltr_auth` database:
   ```sql
   SELECT u.id, u.name, u.email
   FROM sessions s
   JOIN users u ON s.user_id = u.id
   WHERE s.token = ? AND s.expires_at > NOW()
   ```
4. If valid, attach user info to request state
5. Route handler accesses via `get_current_auth(request)`

**Code Location**: [fastapi/middleware/auth.py](fastapi/middleware/auth.py)

**Connection Management**:
```python
# fastapi/database/sql_store.py
def get_auth_engine():
    """Returns engine connected to AUTH_DATABASE_URL"""
    auth_db_url = os.getenv('AUTH_DATABASE_URL') or settings.DATABASE_URL
    return create_engine(auth_db_url, pool_size=10, max_overflow=20)
```

**Key Points**:
- FastAPI caches the auth database connection (connection pooling)
- Session tokens are random strings, not JWTs (no encryption/decryption)
- Better Auth manages token generation and expiration

---

### Pattern 2: Credit Management (FastAPI → Auth DB)

**Use Case**: Check and deduct credits for operations

**Flow**:
1. FastAPI endpoint receives request (already authenticated)
2. Credit service checks balance in `fltr_auth.users.credits`
3. If sufficient, deduct credits atomically with `SELECT FOR UPDATE`
4. Log transaction in `fltr_auth.credit_transactions`
5. Proceed with operation (e.g., document processing)
6. On failure, refund credits

**Code Location**:
- [fastapi/services/credit_service.py](fastapi/services/credit_service.py)
- [fastapi/repositories/credit_repository.py](fastapi/repositories/credit_repository.py)

**Example**:
```python
# Check balance
balance = credit_service.get_balance(user_id)
if balance.credits < required_credits:
    raise InsufficientCreditsError()

# Deduct credits (atomic)
transaction = credit_service.deduct_credits(
    user_id=user_id,
    amount=10,
    operation=CreditOperation.DOCUMENT_UPLOAD,
    resource_id=dataset_id
)

# Process document...

# On error, refund
if processing_failed:
    credit_service.refund_credits(
        transaction_id=transaction.id,
        error_type=ErrorType.PROCESSING_ERROR
    )
```

**Key Points**:
- All credit operations use `SELECT FOR UPDATE` for atomicity
- Transactions are logged for audit trail
- Automatic refunds on system errors

---

### Pattern 3: Dataset Ownership (FastAPI References Auth DB)

**Use Case**: Store dataset ownership without duplicating user data

**Implementation**:
```python
# fastapi/models/dataset.py
class Dataset(SQLModel, table=True):
    id: UUID
    owner_id: UUID  # References users.id in fltr_auth (NO FK!)
    name: str
    # ... other fields
```

**Key Points**:
- `owner_id` is a simple UUID field, **not a foreign key**
- No database-level referential integrity
- Application-level validation ensures owner_id exists
- Dataset queries join with auth DB when user info needed:
  ```python
  # Get dataset with owner info
  dataset = db.query(Dataset).filter_by(id=dataset_id).one()
  owner = auth_db.query(User).filter_by(id=dataset.owner_id).one()
  ```

**Why No Foreign Key?**:
- Foreign keys can't span databases in PostgreSQL
- Application-level validation is sufficient
- Allows potential future migration to separate DB servers

---

### Pattern 4: Subscription Webhooks (Next.js → FastAPI)

**Use Case**: Grant credits after subscription purchase

**Flow**:
1. Stripe webhook hits Next.js endpoint
2. Next.js updates `fltr_auth.subscriptions`
3. Next.js calls FastAPI endpoint to grant credits:
   ```typescript
   await fetch(`${FASTAPI_URL}/api/v1/credits/add`, {
     method: 'POST',
     body: JSON.stringify({
       user_id: userId,
       amount: creditsToGrant,
       type: 'subscription_renewal'
     })
   })
   ```
4. FastAPI updates `fltr_auth.users.credits`
5. FastAPI logs transaction in `fltr_auth.credit_transactions`

**Code Location**:
- [nextjs/src/lib/auth.ts](nextjs/src/lib/auth.ts) (Better Auth plugin)
- [fastapi/routers/credits.py](fastapi/routers/credits.py)

---

## Environment Configuration

### Next.js Service

```bash
# .env
DATABASE_URL=postgresql://user:password@host:5432/fltr_auth
FASTAPI_URL=http://localhost:8000

# OAuth providers (Better Auth)
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GITHUB_CLIENT_ID=...
GITHUB_CLIENT_SECRET=...

# Stripe
STRIPE_SECRET_KEY=...
STRIPE_WEBHOOK_SECRET=...
```

### FastAPI Service

```bash
# .env
# Main database for datasets
DATABASE_URL=postgresql://user:password@host:5432/fltr

# Auth database for session validation + credits
AUTH_DATABASE_URL=postgresql://user:password@host:5432/fltr_auth

# Fallback: If AUTH_DATABASE_URL not set, uses DATABASE_URL
```

**Important**:
- `DATABASE_URL` in FastAPI points to `fltr` database
- `AUTH_DATABASE_URL` in FastAPI points to `fltr_auth` database
- If `AUTH_DATABASE_URL` is missing, FastAPI falls back to `DATABASE_URL`

---

## Database Initialization

### Next.js (fltr_auth)

```bash
cd nextjs

# Generate migrations
pnpm drizzle-kit generate

# Apply migrations
pnpm drizzle-kit migrate

# Or use the helper script
pnpm db:migrate
```

**Code**: Drizzle handles migrations automatically on startup

---

### FastAPI (both databases)

```bash
cd fastapi

# Initialize main database (fltr)
# Creates: datasets, documents, processing_jobs
python -c "from database.sql_store import init_db; init_db()"

# Initialize auth database (fltr_auth) - credit tables only
# Creates: user_credits, teams, credit_transactions, api_key_credits
# Better Auth tables already exist from Next.js migrations
python -c "from database.sql_store import init_auth_db; init_auth_db()"
```

**Code Location**: [fastapi/database/sql_store.py](fastapi/database/sql_store.py)

**Important**:
- `init_db()` creates tables in `DATABASE_URL` (fltr database)
- `init_auth_db()` creates tables in `AUTH_DATABASE_URL` (fltr_auth database)
- These run automatically on FastAPI startup
- Better Auth tables (users, sessions, accounts) are managed by Next.js migrations

---

## Data Flow Examples

### Example 1: Create Dataset

```
1. User authenticates via Next.js (Better Auth)
   → Session created in fltr_auth.sessions

2. Client requests dataset creation
   POST /api/v1/datasets
   Authorization: Bearer <session-token>

3. FastAPI middleware validates token
   → Query fltr_auth.sessions + users
   → Extract user.id

4. FastAPI creates dataset
   → INSERT INTO fltr.datasets (owner_id=user.id, ...)

5. Return dataset to client
```

**Databases Touched**:
- fltr_auth (read: session validation)
- fltr (write: dataset creation)

---

### Example 2: Upload & Process Document

```
1. Client uploads document
   POST /api/v1/datasets/{id}/documents
   Authorization: Bearer <token>

2. FastAPI validates session
   → Query fltr_auth.sessions + users

3. FastAPI checks credits
   → Query fltr_auth.users.credits
   → If insufficient, return 402 Payment Required

4. FastAPI deducts credits (atomic)
   → UPDATE fltr_auth.users SET credits = credits - 10
   → INSERT INTO fltr_auth.credit_transactions

5. FastAPI uploads to R2
   → Store file in Cloudflare R2

6. FastAPI creates document record
   → INSERT INTO fltr.documents

7. FastAPI queues processing job
   → INSERT INTO fltr.processing_jobs
   → Send to Cloudflare Queue

8. Background worker processes document
   → Chunk text, generate embeddings, store in Milvus
   → UPDATE fltr.documents (status='ready')
   → UPDATE fltr.datasets (document_count += 1)

9. On error, refund credits
   → UPDATE fltr_auth.users SET credits = credits + 10
   → INSERT INTO fltr_auth.credit_transactions (type='refund')
```

**Databases Touched**:
- fltr_auth (read: session, credits; write: credit deduction/refund)
- fltr (write: document, processing_job)

---

### Example 3: Subscribe & Grant Credits

```
1. User purchases subscription in Next.js
   → Stripe Checkout session

2. Stripe webhook fires
   → POST /api/webhooks/stripe

3. Next.js webhook handler
   → UPDATE fltr_auth.subscriptions
   → Calculate credits to grant (e.g., 1000 credits)

4. Next.js calls FastAPI
   → POST http://localhost:8000/api/v1/credits/add
   → Body: { user_id, amount: 1000, type: 'purchase' }

5. FastAPI updates credits
   → UPDATE fltr_auth.users SET credits = credits + 1000
   → INSERT INTO fltr_auth.credit_transactions

6. User can now upload documents
```

**Databases Touched**:
- fltr_auth (write: subscriptions, users.credits, credit_transactions)

---

## Migration Strategy

### Adding New Tables

**Auth-related tables** (users, credits, subscriptions):
1. Add to Next.js Drizzle schema: `nextjs/src/database/schema.ts`
2. Generate migration: `pnpm drizzle-kit generate`
3. Apply migration: `pnpm drizzle-kit migrate`
4. If FastAPI needs access, add SQLModel definition in `fastapi/models/`

**Dataset-related tables** (datasets, documents):
1. Add SQLModel class in `fastapi/models/`
2. Add to `init_db()` in `fastapi/database/sql_store.py`
3. Restart FastAPI (tables created automatically)
4. For production, create SQL migration in `fastapi/migrations/`

### Modifying Existing Tables

**Auth tables**:
1. Modify Drizzle schema in Next.js
2. Generate and apply Drizzle migration
3. Update corresponding SQLModel in FastAPI (if exists)

**Dataset tables**:
1. Modify SQLModel in FastAPI
2. Create Alembic migration (for production)
3. Apply migration manually

---

## Testing Strategy

### Test Mode (SQLite)

Both services support SQLite for testing:

```bash
# Use in-memory SQLite
DATABASE_URL=sqlite:///:memory:
AUTH_DATABASE_URL=sqlite:///:memory:
```

**Middleware Detection**:
[fastapi/middleware/credit_gate.py](fastapi/middleware/credit_gate.py) detects SQLite and uses the injected test session instead of creating a new auth session.

```python
# Smart session detection
if "sqlite" in str(db.bind.url):
    # Test mode: Use injected session with all tables
    session = db
else:
    # Production: Create fresh auth session
    session = Session(get_auth_engine())
```

### Integration Tests

**FastAPI Tests**:
- [fastapi/tests/](fastapi/tests/)
- Use SQLite with all tables in single database
- Mock Cloudflare services (R2, Queue)
- Test credit operations end-to-end

**Next.js Tests**:
- Unit tests for auth flows
- Mock Stripe webhooks
- Test subscription → credit flow

---

## Monitoring & Observability

### Key Metrics

**Database Connections**:
- FastAPI maintains two connection pools (main + auth)
- Monitor connection pool exhaustion
- Default: 10 connections, 20 max overflow

**Cross-Database Queries**:
- Every authenticated FastAPI request = 1 auth DB query
- Credit operations = 2-3 auth DB queries (check, deduct, log)
- Monitor auth DB query latency

**Credit Operations**:
- Track credit transaction volume
- Monitor refund rate (should be low)
- Alert on unusual credit patterns

### Health Checks

```bash
# FastAPI health endpoint
GET /health

# Response:
{
  "status": "healthy",
  "services": {
    "database": "connected",      # Main DB (fltr)
    "milvus": "connected",         # Vector DB
    "auth_database": "connected"   # Auth DB (fltr_auth)
  }
}
```

---

## Troubleshooting

### Common Issues

#### 1. "Table 'users' not found in fltr database"

**Cause**: Code is querying the wrong database

**Solution**:
- User queries should use `get_auth_engine()`, not `get_engine()`
- Check that credit operations use `CreditRepository` (uses auth engine)

#### 2. "Foreign key constraint violation on owner_id"

**Cause**: Attempting to create FK to users table across databases

**Solution**:
- `datasets.owner_id` should be a plain UUID field, not a foreign key
- Validate owner_id exists in application code, not database

#### 3. "Session validation fails"

**Cause**: FastAPI can't connect to auth database

**Solution**:
- Verify `AUTH_DATABASE_URL` is set correctly
- Check auth database is running and accessible
- Verify network connectivity between services

#### 4. "Credit tables not found"

**Cause**: `init_auth_db()` not run

**Solution**:
```bash
# FastAPI should run this on startup automatically
# If not, run manually:
python -c "from database.sql_store import init_auth_db; init_auth_db()"
```

#### 5. "Tests failing with 'wrong database' errors"

**Cause**: Test suite using wrong session

**Solution**:
- Ensure test fixtures inject correct database session
- Use SQLite in-memory for tests (single database)
- Check middleware detection logic for SQLite

---

## Future Considerations

### When to Migrate to Full Service Isolation

Consider full microservices isolation when:
- Multiple teams own different services
- Services need independent deployment cycles
- Scaling requirements differ significantly
- Regulatory requirements mandate data isolation

### Migration Path to Isolated Services

1. **Add User Sync Events**:
   - Next.js publishes user create/update/delete events
   - FastAPI consumes events and maintains user cache in fltr database

2. **Implement Session Service**:
   - Centralized session validation service
   - Both Next.js and FastAPI call session service
   - Removes auth DB dependency from FastAPI

3. **Add Credit Service**:
   - Standalone credit management service
   - Owns credit data and exposes API
   - Both Next.js and FastAPI call credit service

4. **Gradual Cutover**:
   - Phase 1: Dual writes (current + new pattern)
   - Phase 2: Read from new pattern, write to both
   - Phase 3: Fully cut over to new pattern
   - Phase 4: Remove old cross-database connections

---

## Summary

**Current Architecture**: Shared Auth Database Pattern
- ✅ Single source of truth for users
- ✅ Simple session validation
- ✅ Atomic credit operations
- ✅ Appropriate for early-stage product
- ⚠️ Cross-database dependency (acceptable trade-off)

**Key Principles**:
1. Auth database (`fltr_auth`) owns user identity and credits
2. Main database (`fltr`) owns datasets and documents
3. FastAPI connects to both databases as needed
4. No user data duplication
5. Application-level referential integrity for owner_id
6. Clear service boundaries with pragmatic coupling

**For More Information**:
- [DATABASES.md](DATABASES.md) - Quick reference guide
- [CREDIT_SYSTEM_IMPLEMENTATION.md](CREDIT_SYSTEM_IMPLEMENTATION.md) - Credit system details
- [fastapi/database/sql_store.py](fastapi/database/sql_store.py) - Database connection code
- [nextjs/src/database/schema.ts](nextjs/src/database/schema.ts) - Auth database schema
