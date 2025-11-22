# FLTR Databases - Quick Reference

> **TL;DR**: FLTR uses two PostgreSQL databases - `fltr_auth` for users/auth/credits (managed by Next.js), and `fltr` for datasets/documents (managed by FastAPI). FastAPI connects to both.

## Which Database Owns What?

| Table | Database | Service | Purpose |
|-------|----------|---------|---------|
| **users** | `fltr_auth` | Next.js | User accounts and profiles |
| **sessions** | `fltr_auth` | Next.js | Active sessions (Better Auth) |
| **accounts** | `fltr_auth` | Next.js | OAuth provider accounts |
| **verifications** | `fltr_auth` | Next.js | Email verification tokens |
| **subscriptions** | `fltr_auth` | Next.js | Stripe subscriptions |
| **credit_transactions** | `fltr_auth` | Next.js + FastAPI | Credit audit log |
| **api_key_credits** | `fltr_auth` | Next.js + FastAPI | Per-API-key credit pools |
| **teams** | `fltr_auth` | Next.js + FastAPI | Team/organization records |
| **datasets** | `fltr` | FastAPI | Dataset metadata |
| **documents** | `fltr` | FastAPI | Uploaded documents |
| **processing_jobs** | `fltr` | FastAPI | Background job tracking |

## Environment Variables

### Next.js (`nextjs/.env`)

```bash
# Auth database (includes users, sessions, credits)
DATABASE_URL=postgresql://user:password@localhost:5432/fltr_auth

# FastAPI endpoint for credit operations
FASTAPI_URL=http://localhost:8000

# Better Auth (OAuth providers)
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GITHUB_CLIENT_ID=...
GITHUB_CLIENT_SECRET=...
TWITTER_CLIENT_ID=...
TWITTER_CLIENT_SECRET=...

# Stripe
STRIPE_SECRET_KEY=...
STRIPE_WEBHOOK_SECRET=...
```

### FastAPI (`fastapi/.env`)

```bash
# Main database (datasets, documents, processing jobs)
DATABASE_URL=postgresql://user:password@localhost:5432/fltr

# Auth database (users, sessions, credits)
AUTH_DATABASE_URL=postgresql://user:password@localhost:5432/fltr_auth

# If AUTH_DATABASE_URL is not set, falls back to DATABASE_URL
```

**Important**: `DATABASE_URL` means different things in each service!
- **Next.js**: `DATABASE_URL` → `fltr_auth` database
- **FastAPI**: `DATABASE_URL` → `fltr` database

## Connection Patterns

### When to Use Which Connection?

#### In FastAPI:

```python
from database.sql_store import get_engine, get_auth_engine

# Use get_engine() for:
# - Dataset queries (datasets, documents, processing_jobs)
engine = get_engine()
with Session(engine) as session:
    dataset = session.query(Dataset).filter_by(id=dataset_id).first()

# Use get_auth_engine() for:
# - Session validation (users, sessions)
# - Credit operations (users.credits, credit_transactions)
# - User info lookups
auth_engine = get_auth_engine()
with Session(auth_engine) as session:
    user = session.query(User).filter_by(id=user_id).first()
```

#### In Next.js:

```typescript
import { db } from './database/connection'

// db always connects to fltr_auth
// All tables (users, sessions, subscriptions, credits) are in this database
const user = await db.query.users.findFirst({
  where: eq(users.id, userId)
})
```

## Common Queries

### Get User Info (FastAPI)

```python
from database.sql_store import get_auth_engine
from sqlalchemy.orm import Session

auth_engine = get_auth_engine()
with Session(auth_engine) as session:
    result = session.execute(
        text("SELECT id, name, email, credits FROM users WHERE id = :user_id"),
        {"user_id": user_id}
    )
    user = result.first()
```

### Validate Session Token (FastAPI)

```python
from database.sql_store import get_auth_engine
from sqlalchemy.orm import Session
from sqlalchemy import text
from datetime import datetime

auth_engine = get_auth_engine()
with Session(auth_engine) as session:
    result = session.execute(
        text("""
            SELECT u.id, u.name, u.email
            FROM sessions s
            JOIN users u ON s.user_id = u.id
            WHERE s.token = :token
            AND s.expires_at > :now
        """),
        {"token": token, "now": datetime.utcnow()}
    )
    if result:
        return result.first()
```

### Get Dataset with Owner (FastAPI)

```python
from database.sql_store import get_engine, get_auth_engine
from sqlalchemy.orm import Session

# Get dataset from main DB
engine = get_engine()
with Session(engine) as session:
    dataset = session.query(Dataset).filter_by(id=dataset_id).first()

# Get owner from auth DB
auth_engine = get_auth_engine()
with Session(auth_engine) as session:
    owner = session.execute(
        text("SELECT id, name, email FROM users WHERE id = :user_id"),
        {"user_id": dataset.owner_id}
    ).first()
```

### Check Credit Balance (FastAPI)

```python
from services.credit_service import CreditService
from database.sql_store import get_auth_engine
from sqlalchemy.orm import Session

auth_engine = get_auth_engine()
with Session(auth_engine) as session:
    credit_service = CreditService(session)
    balance = credit_service.get_balance(user_id)
    print(f"User has {balance.credits} credits")
```

### Deduct Credits (FastAPI)

```python
from services.credit_service import CreditService
from models.credits import CreditOperation

# Service handles session creation internally
credit_service = CreditService()
transaction = credit_service.deduct_credits(
    user_id=user_id,
    amount=10,
    operation=CreditOperation.DOCUMENT_UPLOAD,
    resource_id=dataset_id
)
```

## Database Initialization

### First Time Setup

```bash
# 1. Create both databases
createdb fltr_auth
createdb fltr

# 2. Initialize Next.js auth database
cd nextjs
pnpm install
pnpm db:migrate    # Applies Drizzle migrations

# 3. Initialize FastAPI databases
cd fastapi
pip install -r requirements.txt
# Tables are created automatically on first startup
python main.py

# Or initialize manually:
python -c "from database.sql_store import init_db, init_auth_db; init_db(); init_auth_db()"
```

### Reset Databases (Development)

```bash
# WARNING: This deletes all data!

# Reset auth database
dropdb fltr_auth && createdb fltr_auth
cd nextjs && pnpm db:migrate

# Reset main database
dropdb fltr && createdb fltr
cd fastapi && python -c "from database.sql_store import init_db; init_db()"

# Reset auth tables in auth database (credit system)
cd fastapi && python -c "from database.sql_store import init_auth_db; init_auth_db()"
```

## Troubleshooting

### "Table 'users' not found"

**Symptom**: `sqlalchemy.exc.ProgrammingError: relation "users" does not exist`

**Cause**: Querying the wrong database

**Solution**:
```python
# ❌ Wrong - queries fltr database
engine = get_engine()
session.query(User).first()  # Error: users not in this DB!

# ✅ Correct - queries fltr_auth database
auth_engine = get_auth_engine()
session.query(User).first()  # Success!
```

### "AUTH_DATABASE_URL not set"

**Symptom**: FastAPI can't validate sessions

**Solution**:
```bash
# Add to fastapi/.env
AUTH_DATABASE_URL=postgresql://user:password@localhost:5432/fltr_auth

# Or, if both DBs are on same server, it will fallback to DATABASE_URL
```

### "Foreign key constraint error on owner_id"

**Symptom**: Can't create dataset with owner_id

**Cause**: Trying to add FK constraint across databases (not supported)

**Solution**:
- `datasets.owner_id` should be a plain UUID field (no FK constraint)
- Validate user exists in application code:
  ```python
  # Check user exists before creating dataset
  auth_engine = get_auth_engine()
  with Session(auth_engine) as session:
      user_exists = session.query(User).filter_by(id=owner_id).first()
      if not user_exists:
          raise ValueError("User not found")
  ```

### "Credit tables not created"

**Symptom**: `relation "credit_transactions" does not exist`

**Solution**:
```bash
# Run auth database initialization
cd fastapi
python -c "from database.sql_store import init_auth_db; init_auth_db()"

# Or restart FastAPI (runs automatically)
python main.py
```

### "Tests failing - wrong database"

**Symptom**: Tests can't find tables

**Solution**: Tests should use SQLite in-memory with all tables in single database

```python
# conftest.py
@pytest.fixture
def db():
    engine = create_engine("sqlite:///:memory:")
    # Create ALL tables in single SQLite database
    SQLModel.metadata.create_all(engine)
    with Session(engine) as session:
        yield session
```

## API Endpoints

### Health Check

```bash
curl http://localhost:8000/health
```

Response:
```json
{
  "status": "healthy",
  "services": {
    "database": "connected",       # Main DB (fltr)
    "milvus": "connected",          # Vector DB
    "auth_database": "connected"    # Auth DB (fltr_auth)
  }
}
```

### Credit Balance

```bash
curl -H "Authorization: Bearer <token>" \
  http://localhost:8000/api/v1/credits/balance
```

Response:
```json
{
  "credits": 97,
  "total_credits_purchased": 100,
  "created_at": "2025-01-01T00:00:00Z"
}
```

### Credit History

```bash
curl -H "Authorization: Bearer <token>" \
  "http://localhost:8000/api/v1/credits/transactions?limit=10"
```

## Testing

### Unit Tests (FastAPI)

```bash
cd fastapi
pytest tests/
```

Tests use SQLite in-memory database with all tables.

### Integration Tests

```bash
# Start all services
docker-compose up -d postgres milvus redis

# Run Next.js
cd nextjs && pnpm dev

# Run FastAPI
cd fastapi && python main.py

# Test flow
curl -X POST http://localhost:3000/api/auth/login  # Get session token
curl -H "Authorization: Bearer <token>" \
  http://localhost:8000/api/v1/datasets  # Use in FastAPI
```

## Schema Files

| Service | Schema Location | Migration Location |
|---------|----------------|-------------------|
| **Next.js** | [nextjs/src/database/schema.ts](nextjs/src/database/schema.ts) | [nextjs/migrations/](nextjs/migrations/) |
| **FastAPI** | [fastapi/models/](fastapi/models/) | [fastapi/migrations/](fastapi/migrations/) |

### Adding New Tables

**Auth-related** (users, credits, etc.):
1. Edit `nextjs/src/database/schema.ts`
2. Run `pnpm drizzle-kit generate`
3. Run `pnpm db:migrate`
4. If FastAPI needs access, add SQLModel in `fastapi/models/`

**Dataset-related** (datasets, documents):
1. Add SQLModel in `fastapi/models/`
2. Update `init_db()` in `fastapi/database/sql_store.py`
3. Restart FastAPI (auto-creates tables)

## Quick Commands

```bash
# View tables in fltr_auth
psql fltr_auth -c "\dt"

# View tables in fltr
psql fltr -c "\dt"

# Count users
psql fltr_auth -c "SELECT COUNT(*) FROM users"

# Count datasets
psql fltr -c "SELECT COUNT(*) FROM datasets"

# Check credit transactions
psql fltr_auth -c "SELECT user_id, amount, type, operation FROM credit_transactions ORDER BY created_at DESC LIMIT 10"

# Check dataset ownership
psql fltr -c "SELECT id, name, owner_id FROM datasets LIMIT 10"
```

## Architecture Diagram

```
┌──────────────┐         ┌──────────────┐
│   Next.js    │         │   FastAPI    │
│   :3000      │         │   :8000      │
└──────┬───────┘         └───────┬──────┘
       │                         │
       │ DATABASE_URL            │ DATABASE_URL
       ▼                         ▼
┌─────────────────┐      ┌─────────────────┐
│   fltr_auth     │◄─────│   fltr          │
│   (Auth DB)     │ AUTH │   (Main DB)     │
│                 │  DB  │                 │
│ • users         │ URL  │ • datasets      │
│ • sessions      │      │ • documents     │
│ • credits       │      │ • proc_jobs     │
└─────────────────┘      └─────────────────┘
```

## References

- [DATABASE_ARCHITECTURE.md](DATABASE_ARCHITECTURE.md) - Comprehensive architecture guide
- [CREDIT_SYSTEM_IMPLEMENTATION.md](CREDIT_SYSTEM_IMPLEMENTATION.md) - Credit system details
- [fastapi/database/sql_store.py](fastapi/database/sql_store.py) - Connection management
- [nextjs/src/database/schema.ts](nextjs/src/database/schema.ts) - Auth database schema

---

**Need Help?** Check [DATABASE_ARCHITECTURE.md](DATABASE_ARCHITECTURE.md) for detailed explanations of design decisions and patterns.
