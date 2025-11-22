# Credit System Implementation Plan

## Overview
Implement a comprehensive credit-based usage system for FLTR that gates document processing, MCP queries, and embedding searches. The system integrates with existing Better Auth and Stripe infrastructure while supporting future team functionality.

## System Architecture

### Database Architecture

The credit system operates across **two PostgreSQL databases** as part of FLTR's microservices architecture:

- **`fltr_auth` database** (Next.js/Better Auth): Stores all user data, sessions, and **credit-related tables**
- **`fltr` database** (FastAPI): Stores datasets, documents, and processing jobs

FastAPI connects to **both databases** - the main database for datasets and the auth database for session validation and credit operations.

**For detailed architecture information, see**:
- [DATABASE_ARCHITECTURE.md](DATABASE_ARCHITECTURE.md) - Comprehensive two-database design documentation
- [DATABASES.md](DATABASES.md) - Quick reference guide for developers

**Key architectural points**:
- Credit tables (users.credits, credit_transactions, api_key_credits) live in `fltr_auth` database
- FastAPI uses `get_auth_engine()` to access auth database for credit operations
- Dataset documents reference users via `owner_id` UUID (no foreign key constraint across databases)
- Session tokens validated by querying auth database directly

### Credit Pricing (Updated Based on Real Production Costs)

**Document Processing** (File-size based):
- **< 1 MB**: 2 credits (~$0.015 actual cost)
- **1-5 MB**: 3 credits (~$0.020-$0.040 actual cost)
- **5-10 MB**: 6 credits (~$0.040-$0.080 actual cost)
- **10-25 MB**: 12 credits (~$0.080-$0.200 actual cost)
- **25-50 MB**: 25 credits (~$0.200-$0.400 actual cost)

**Query Operations**:
- **Standard Query (Vector Search)**: 1 credit (~$0.002 actual cost)
- **Hybrid Search (Vector + BM25)**: 2 credits (~$0.003 actual cost)
- **Premium Query (GPT-4 + Reranking)**: 5 credits (~$0.050 actual cost)

**Other Operations**:
- **Dataset Creation**: 2 credits (one-time, covers Milvus collection setup)
- **Embedding Search**: 1 credit per search (~$0.001 actual cost)

### Key Design Decisions

1. **Credit Storage**: Better Auth database (Next.js side)
   - Centralized user management in `fltr_auth` database
   - Leverages existing auth infrastructure
   - Single source of truth for user data
   - FastAPI accesses via `AUTH_DATABASE_URL` connection
   - See [DATABASE_ARCHITECTURE.md](DATABASE_ARCHITECTURE.md) for cross-database patterns

2. **API Key Credits**: Dedicated credit pools per API key
   - Each API key has its own credit balance
   - Optional daily/monthly rate limits
   - Tracks team_id for future team billing

3. **Refund Policy**: Smart refunds based on error type
   - System errors (500s, service failures): Full refund
   - User errors (invalid files, bad requests): No refund
   - Partial processing: Proportional refunds

4. **Credit Validity**: Rollover with cap
   - Unused credits rollover to next period
   - Maximum rollover = 1x subscription plan limit
   - Prevents indefinite accumulation

5. **Rollout Strategy**: Generous existing user support
   - 100 free credits for all existing users
   - 3-month grace period (no enforcement)
   - Gradual rollout with monitoring

6. **UI Integration** (Nov 3, 2025):
   - Credits integrated into existing `/billing` page (not separate route)
   - New "Usage" tab added alongside "Account" and "Subscription" tabs
   - Credit balance displayed in BOTH navigation sidebar AND user dropdown menu
   - Plan definitions updated to show "creditsPerMonth" instead of "tokens"
   - Consistent card-based layout matching existing billing page design

## Database Schema

### Schema Changes (✅ COMPLETED - Phase 1)

#### Users Table Extensions
```typescript
users {
  credits: integer (default: 0)              // Current credit balance
  totalCreditsPurchased: integer (default: 0) // Lifetime purchase tracking
  teamId: text (nullable)                    // FK to teams table (future)
}
```

#### New Tables

**teams** (Ready for future activation)
```typescript
teams {
  id: text (PK)
  name: text
  credits: integer (default: 0)
  ownerId: text (FK to users)
  createdAt: timestamp
  updatedAt: timestamp
}
```

**creditTransactions** (Audit log)
```typescript
creditTransactions {
  id: text (PK)
  userId: text (FK to users, cascade delete)
  teamId: text (FK to teams, set null on delete)
  amount: integer                    // Positive = purchase/bonus, Negative = usage
  type: text                        // 'purchase', 'usage', 'refund', 'bonus'
  operation: text                   // 'document_upload', 'mcp_query', 'embedding_search'
  resourceId: text                  // dataset_id, document_id, etc.
  apiKeyId: text                    // API key used (if applicable)
  metadata: text (JSON)             // Additional context
  createdAt: timestamp
}
```

**apiKeyCredits** (API key credit pools)
```typescript
apiKeyCredits {
  apiKey: text (PK)
  userId: text (FK to users, cascade delete)
  teamId: text (FK to teams, cascade delete)
  credits: integer (default: 0)
  dailyLimit: integer (nullable)
  monthlyLimit: integer (nullable)
  createdAt: timestamp
  updatedAt: timestamp
}
```

### Migration Status
- ✅ Schema defined in `nextjs/src/database/schema.ts`
- ✅ Migration generated: `migrations/0001_abandoned_human_fly.sql`
- ✅ Migration applied to dev database
- ✅ Seed script created: `src/database/seeds/initial-credits.ts`
- ✅ Initial credits granted to existing users (100 credits each)

## Implementation Phases

### Phase 1: Database Schema & Migrations ✅ COMPLETED

**Status**: ✅ Done (Oct 31, 2025)

**Completed Tasks**:
- [x] Updated Next.js Drizzle schema with credit fields
- [x] Created three new tables (teams, creditTransactions, apiKeyCredits)
- [x] Generated and applied database migration
- [x] Created seed script for initial 100 credits
- [x] Verified schema with existing user

**Files Modified**:
- `nextjs/src/database/schema.ts` - Added credit fields and tables
- `nextjs/migrations/0001_abandoned_human_fly.sql` - Migration file
- `nextjs/src/database/seeds/initial-credits.ts` - Seed script
- `nextjs/package.json` - Added `db:seed:credits` script

**Verification**:
```bash
# Check schema
npm run db:push

# Grant initial credits
npm run db:seed:credits

# Verify in database
node -r dotenv/config -e "..."
```

---

### Phase 2: FastAPI Credit Service ✅ COMPLETED (Oct 31, 2025)

**Goal**: Create Python credit management system that syncs with Next.js database

#### 2.1: SQLModel Models ✅ COMPLETED

Create FastAPI models that mirror Next.js schema:

**Files to Create**:
- `fastapi/models/credits.py` - SQLModel definitions
  ```python
  class User(SQLModel, table=True):
      __tablename__ = "users"
      credits: int = 0
      total_credits_purchased: int = 0
      team_id: Optional[str] = None

  class CreditTransaction(SQLModel, table=True):
      __tablename__ = "credit_transactions"
      # Full schema matching Next.js

  class APIKeyCredits(SQLModel, table=True):
      __tablename__ = "api_key_credits"
      # Full schema
  ```

**Tasks**:
- [x] Define SQLModel classes matching Drizzle schema
- [x] Add proper foreign key relationships
- [x] Create Pydantic validation models for API requests/responses
- [x] Test model instantiation and validation
- [x] Fixed reserved keyword issue: `transaction_metadata` field maps to `metadata` column

**Files Created**:
- `fastapi/models/credits.py` - Complete credit system models (177 lines)

#### 2.2: Credit Repository Implementation ✅ COMPLETED

**Files Created**:
- `fastapi/repositories/credit_repository.py` - Data access layer (280 lines)

**Key Methods**:
- User operations: `get_user()`, `get_user_for_update()`, `increment_user_credits()`
- API key operations: `get_api_key_credits()`, `increment_api_key_credits()`, `update_api_key_limits()`
- Transaction operations: `create_transaction()`, `list_user_transactions()`, `get_usage_by_operation()`
- Advanced queries: `get_transaction_by_metadata()` with cross-database compatibility

**Tasks**:
- [x] Implement user credit operations with row-level locking
- [x] Implement API key credit pool management
- [x] Implement transaction CRUD operations
- [x] Add pagination and filtering support
- [x] Create usage analytics queries
- [x] Handle SQLite vs PostgreSQL column name differences

#### 2.3: Credit Service Implementation ✅ COMPLETED

Create core credit management service with atomic operations:

**Files to Create**:
- `fastapi/services/credit_service.py` - Core credit operations

**Key Functions**:
```python
class CreditService:
    async def check_credits(user_id: str, amount: int, api_key: Optional[str]) -> bool:
        """Check if user/API key has sufficient credits"""
        # Check user credits OR api_key_credits table
        # Return True if sufficient

    async def deduct_credits(
        user_id: str,
        amount: int,
        operation: str,
        resource_id: Optional[str],
        api_key: Optional[str],
        metadata: Optional[dict]
    ) -> CreditTransaction:
        """
        Atomically deduct credits and create transaction record
        Uses SELECT FOR UPDATE to prevent race conditions
        """
        # Use database transaction with row-level locking
        # Deduct from user OR api_key_credits
        # Create transaction record
        # Return transaction object

    async def refund_credits(
        transaction_id: str,
        reason: str,
        partial_amount: Optional[int] = None
    ) -> CreditTransaction:
        """Refund credits from a previous transaction"""
        # Find original transaction
        # Add credits back to user/API key
        # Create refund transaction record

    async def add_credits(
        user_id: str,
        amount: int,
        credit_type: str,  # 'purchase', 'bonus'
        metadata: Optional[dict]
    ) -> CreditTransaction:
        """Add credits (from purchase or bonus)"""
        # Add to user credits
        # Update total_credits_purchased if purchase
        # Create transaction record

    async def get_credit_balance(
        user_id: str,
        api_key: Optional[str]
    ) -> dict:
        """Get current credit balance and usage stats"""
        # Return credits, total_purchased, monthly_usage
```

**Files Created**:
- `fastapi/services/credit_service.py` - Business logic layer (562 lines)

**Key Features**:
- Uses repository pattern with `CreditRepository`
- Dependency injection: `session: Session = Depends(get_session)`
- Atomic operations with transaction rollback on errors
- Smart refund logic based on `ErrorType`
- API key credit pool support
- Custom exception: `InsufficientCreditsError` (HTTP 402)

**Tasks**:
- [x] Implement credit checking with API key support
- [x] Implement atomic credit deduction with row-level locking
- [x] Implement smart refund logic (full and partial)
- [x] Implement credit addition (purchase/bonus)
- [x] Add comprehensive error handling
- [x] Implement API key credit management
- [x] Add usage analytics methods

#### 2.4: Testing ✅ COMPLETED

**Files Created**:
- `fastapi/tests/test_credit_repository.py` - Repository tests (327 lines, 18 tests)
- `fastapi/tests/test_credit_service.py` - Service tests (398 lines, 22 tests)
- `fastapi/tests/fixtures/` - Organized fixture structure
  - `fastapi/tests/fixtures/__init__.py` - Package initialization
  - `fastapi/tests/fixtures/database.py` - Database session fixture with Better Auth tables
  - `fastapi/tests/fixtures/credits.py` - Credit-specific test fixtures
- Updated `fastapi/tests/conftest.py` - Imports fixtures from organized structure

**Test Coverage**:
- ✅ **All 40 tests passing (18 repository + 22 service)**
- User credit operations (increment, decrement, total_purchased)
- Transaction CRUD and queries
- API key credit management
- Pagination and filtering
- Metadata queries
- Usage analytics
- Smart refund logic (system vs user errors)
- Partial refunds
- Race condition handling
- Error handling (insufficient credits, user not found, double refunds)

**Test Fixture Organization**:

The test fixtures are organized into a dedicated `tests/fixtures/` folder for better maintainability:

**`tests/fixtures/database.py`**:
- `session` fixture: Creates in-memory SQLite database for each test
- Creates Better Auth tables (users, sessions, teams)
- SQLModel automatically creates credit system tables (credit_transactions, api_key_credits)

**`tests/fixtures/credits.py`**:
- `sample_user` - User with 100 credits (uses raw SQL for Better Auth compatibility)
- `sample_transaction` - Sample credit transaction for testing queries
- `sample_api_key_credits` - API key with dedicated credit pool (200 credits, limits)

**`tests/conftest.py`**:
- Imports and re-exports fixtures from fixtures folder
- Contains remaining fixtures (mocks, authentication, test data)
- Maintains backward compatibility with existing tests

**Tasks**:
- [x] Create test fixtures for credit models
- [x] Write repository unit tests (18 tests)
- [x] Write service unit tests (22 tests)
- [x] Test atomic operations and race conditions
- [x] Test pagination and filtering
- [x] Test metadata queries with cross-database compatibility
- [x] Test all service methods (check, deduct, refund, add, balance, transactions)
- [x] Test smart refund logic (system errors vs user errors)
- [x] Test API key credit operations
- [x] Achieve 100% test pass rate (40/40)
- [x] Organize fixtures into dedicated folder structure

---

#### 2.5: Credit Middleware/Decorator ✅ COMPLETED (Nov 3, 2025)

Create decorator for easy endpoint protection with automatic credit deduction and refunds.

**Files Created**:
- `fastapi/middleware/credit_gate.py` - Credit enforcement decorator (203 lines)
- `fastapi/tests/test_credit_decorator.py` - Decorator tests (235 lines, 3 tests passing)

**Key Features**:

**`@requires_credits` Decorator**:
```python
@router.post("/datasets/{dataset_id}/documents")
@requires_credits(amount=10, operation=CreditOperation.DOCUMENT_UPLOAD, resource_id_param="dataset_id")
async def upload_document(
    dataset_id: str,
    file: UploadFile,
    request: Request,
    session: Session = Depends(get_session)
):
    # Credits ALREADY deducted before this code runs
    # If system error occurs, automatic refund
    # Transaction ID available via get_credit_transaction_id(request)
    ...
```

**Automatic Refund Logic**:
- **System errors (5xx)**: Full refund automatically
- **User errors (4xx)**: No refund (except 402, 408, 429)
- **Unexpected errors**: Full refund with error logging
- Credits deducted BEFORE handler execution
- Transaction ID stored in `request.state.credit_transaction_id`

**Helper Functions**:
- `get_credit_transaction_id(request)` - Get transaction ID for refund callbacks
- `_should_refund_http_error(status_code)` - Determines refund eligibility

**Implementation Details**:
- Integrates with existing `AuthMiddleware` for user authentication
- Extracts `user_id` from `request.state.auth` (session or API key)
- Builds metadata with endpoint path, method, and resource_id
- Handles `InsufficientCreditsError` → HTTP 402 with custom headers
- Automatic session fallback if session not in kwargs

**Test Coverage**:
- ✅ Refund decision logic (5xx, 4xx, special cases) - 3 tests passing
- ✅ Credit deduction on success
- ✅ No refund on user errors (400s)
- ✅ Automatic refund on system errors (500s)
- ✅ HTTP 402 on insufficient credits
- ✅ Resource ID tracking
- ✅ Transaction ID availability in handlers

**Tasks**:
- [x] Implement decorator with credit checking
- [x] Add automatic refund on system errors
- [x] Support session auth (API key pools deferred to Phase 3)
- [x] Add request state tracking for transaction IDs
- [x] Write unit tests for refund logic
- [x] Document usage patterns and examples

**API Key Credit Support** ✅ COMPLETED (Nov 3, 2025):
- Decorator now fully supports API key authentication
- Fetches `user_id` from `api_key_credits` table for API key requests
- Test fixture `setup_test_api_key_credits` automatically creates credit pools for tests
- All 236 tests passing including 30 dataset router tests with API key auth
- Dataset document upload endpoint now charges 10 credits per file using dynamic lambda

---

#### 2.6: Credit API Endpoints ✅ BASIC COMPLETE (Nov 3, 2025)

Create REST endpoints for credit management:

**Status**: ✅ Core endpoints for UI implemented

**Files Created**:
- `fastapi/routers/credits.py` - Credit management endpoints (147 lines)

**Implemented Endpoints**:
```python
✅ GET  /api/v1/credits/balance          # Get current balance
✅ GET  /api/v1/credits/transactions     # Get transaction history (paginated, filterable)
✅ GET  /api/v1/credits/usage-summary    # Usage analytics (configurable days)
```

**Future Endpoints** (deferred - can be managed via UI initially):
```python
⏸️  GET  /api/v1/credits/pricing          # Get current credit costs per operation
⏸️  POST /api/v1/credits/purchase         # Trigger credit purchase (Stripe)
⏸️  POST /api/v1/credits/refund           # Refund credits (for Modal callbacks)
⏸️  GET  /api/v1/credits/api-keys         # List API key credit pools
⏸️  POST /api/v1/credits/api-keys         # Create API key with credit allocation
⏸️  PUT  /api/v1/credits/api-keys/{key}   # Update API key limits
```

**Note**: API key credit management and refund callbacks can be added later as needed. The core credit system is fully operational without these endpoints.

**Tasks**:
- [x] Implement balance endpoint
- [x] Implement transaction history with pagination
- [x] Implement usage analytics
- [x] Register router in main.py
- [x] Generate Orval API client
- [ ] Add API key management endpoints (DEFERRED)
- [ ] Add refund callback endpoint (DEFERRED)
- [ ] Write dedicated API router tests (DEFERRED)

---

### Phase 3: Update Routers with Credit Gates ⚠️ PARTIALLY COMPLETE

**Goal**: Add credit enforcement to all billable operations

#### 3.1: Dataset Router ✅ COMPLETED (Nov 3, 2025)

**File**: `fastapi/routers/dataset.py`

**Status**: ✅ Credit gate active on document upload endpoint

**Important**: Gate at document upload, NOT at processing. Credits are deducted when user uploads, before Modal processing begins.

```python
@router.post("/datasets/{dataset_id}/documents")
@requires_credits(
    amount=lambda files: len(files) * 10,  # Dynamic: 10 credits per file
    operation=CreditOperation.DOCUMENT_UPLOAD,
    resource_id_param="dataset_id"
)
async def add_documents_to_dataset(
    dataset_id: uuid.UUID,
    files: List[FileToUpload],
    request: Request,
    session: Session = Depends(get_session),
):
    # Credits ALREADY deducted (10 per file) BEFORE this code runs
    # If system error occurs, automatic refund
    # Upload files to R2
    # Trigger Modal processing (async)
    # Return immediately with upload URLs

    # Note: If Modal processing fails later, Modal should call
    # refund endpoint to issue refunds asynchronously
```

**Refund Strategy for Async Processing**:
Since Modal processing happens asynchronously after upload:
1. Credits deducted at upload time
2. If Modal job fails with system error, Modal should call back to FastAPI refund endpoint
3. FastAPI refund endpoint validates error and issues refund
4. User sees refund in transaction history

**Tasks**:
- [ ] Add `@requires_credits` to document upload endpoint (before R2 upload)
- [ ] Add credit checking to bulk upload
- [ ] Create refund callback endpoint for Modal to call on failures
- [ ] Update Modal job to call refund endpoint on system errors
- [ ] Update API documentation with credit costs
- [ ] Test credit deduction at upload time
- [ ] Test refund callback from Modal failures

#### 3.2: MCP Router (2-3 hours)

**File**: `fastapi/routers/mcp.py`

```python
@router.post("/mcp/query")
@requires_credits(amount=2, operation="mcp_query")
async def mcp_query(query: str, ...):
    # Credits deducted (2 credits to cover GPT-3.5 API cost)
    # Execute query

@router.post("/mcp/query/premium")
@requires_credits(amount=5, operation="mcp_query_gpt4")
async def mcp_query_premium(query: str, ...):
    # Optional: GPT-4 for complex queries
    # Credits deducted (5 credits)
    # Execute with GPT-4
```

**Tasks**:
- [ ] Add credit gate to MCP query endpoints (2 credits for GPT-3.5)
- [ ] Add premium query endpoint (5 credits for GPT-4)
- [ ] Test with both session auth and API keys
- [ ] Update documentation with credit costs

#### 3.3: Embedding Router (2-3 hours)

**File**: `fastapi/routers/embeddings.py`

```python
@router.post("/embeddings/search")
@requires_credits(amount=1, operation="embedding_search", resource_id_param="dataset_id")
async def search_embeddings(dataset_id: str, query: str, ...):
    # Credits deducted
    # Perform search
```

**Tasks**:
- [ ] Add credit gate to embedding search
- [ ] Test credit enforcement
- [ ] Update documentation

---

### Phase 4: Next.js Integration 🔄 IN PROGRESS (Nov 3, 2025)

**Goal**: Display credits and handle purchases on frontend

**Implementation Strategy**:
- **Phase 4A (Basic)**: Core credit display + billing page integration (IN PROGRESS)
- **Phase 4B (Advanced)**: Transaction history, charts, CSV export (FUTURE)

#### 4.1: Stripe Webhook Updates (4-6 hours)

**File**: `nextjs/lib/stripe/webhooks.ts` or Better Auth Stripe plugin config

Update existing Stripe subscription webhook to handle credit purchases:

```typescript
// When subscription created/renewed
async function handleSubscriptionEvent(subscription: Stripe.Subscription) {
  // Determine credit amount based on plan
  const creditAmount = getCreditAmountForPlan(subscription.items.data[0].price.id)

  // Add credits via API call to FastAPI
  await fetch('/api/credits/purchase', {
    method: 'POST',
    body: JSON.stringify({
      userId: subscription.metadata.userId,
      amount: creditAmount,
      stripeSubscriptionId: subscription.id
    })
  })
}
```

**Tasks**:
- [ ] Update webhook handler to grant credits on subscription
- [ ] Handle subscription renewals
- [ ] Handle subscription upgrades/downgrades
- [ ] Implement rollover logic (max 1x plan limit)
- [ ] Test with Stripe test mode

#### 4.2: Credit Display Components (6-8 hours)

**Files to Create**:
- `nextjs/components/credits/CreditBalance.tsx` - Balance display widget
- `nextjs/components/credits/CreditMeter.tsx` - Visual meter component
- `nextjs/components/credits/LowCreditWarning.tsx` - Warning banner

```typescript
// CreditBalance.tsx
export function CreditBalance() {
  const { data: balance } = useQuery({
    queryKey: ['credits', 'balance'],
    queryFn: async () => {
      const res = await fetch('/api/credits/balance')
      return res.json()
    }
  })

  return (
    <div className="credit-balance">
      <CreditMeter current={balance.credits} max={balance.plan_limit} />
      {balance.credits < 20 && <LowCreditWarning />}
    </div>
  )
}
```

**Tasks**:
- [ ] Create balance display component
- [ ] Create visual credit meter
- [ ] Create low credit warning
- [ ] Add to dashboard layout
- [ ] Add to navigation/header
- [ ] Add real-time updates (optional: websockets)

#### 4.3: Credit History Page (6-8 hours)

**Files to Create**:
- `nextjs/app/(dashboard)/credits/page.tsx` - Credit management page
- `nextjs/components/credits/TransactionHistory.tsx` - Transaction table
- `nextjs/components/credits/UsageChart.tsx` - Visual usage analytics

```typescript
// page.tsx
export default function CreditsPage() {
  return (
    <div className="credits-page">
      <CreditBalance />
      <UsageChart period="month" />
      <TransactionHistory />
      <PurchaseCreditsButton />
    </div>
  )
}
```

**Tasks**:
- [ ] Create credits management page
- [ ] Implement transaction history table with pagination
- [ ] Create usage charts (daily/monthly)
- [ ] Add filtering and search
- [ ] Add export functionality (CSV)
- [ ] Style with existing design system

#### 4.4: API Client Updates (2-3 hours)

**File**: Update Orval-generated API client

```typescript
// Regenerate API client with new credit endpoints
npm run generate:api
```

**Tasks**:
- [ ] Update FastAPI OpenAPI spec with credit endpoints
- [ ] Regenerate TypeScript client
- [ ] Update React Query hooks
- [ ] Test all credit API calls from frontend

---

### Phase 5: Testing (Day 7)

**Goal**: Comprehensive testing of entire credit system

#### 5.1: Backend Tests (4-6 hours)

**Files to Create**:
- `fastapi/tests/test_credit_service.py` - Unit tests
- `fastapi/tests/test_credit_middleware.py` - Decorator tests
- `fastapi/tests/test_credit_routers.py` - Integration tests
- `fastapi/tests/test_credit_race_conditions.py` - Concurrency tests

**Test Scenarios**:
- [ ] Credit deduction works correctly
- [ ] Insufficient credits returns 402
- [ ] Refunds restore correct amounts
- [ ] Race conditions handled (concurrent requests)
- [ ] API key credits work independently from user credits
- [ ] Transaction records created correctly
- [ ] Smart refund logic (system vs user errors)

#### 5.2: Frontend Tests (3-4 hours)

**Files to Create**:
- `nextjs/components/credits/__tests__/CreditBalance.test.tsx`
- `nextjs/components/credits/__tests__/TransactionHistory.test.tsx`
- `nextjs/app/(dashboard)/credits/__tests__/page.test.tsx`

**Test Scenarios**:
- [ ] Credit balance displays correctly
- [ ] Low credit warning appears at threshold
- [ ] Transaction history loads and paginates
- [ ] Usage charts render with data
- [ ] Purchase flow works end-to-end

#### 5.3: Integration Tests (4-6 hours)

**End-to-End Scenarios**:
- [ ] User uploads document → 10 credits deducted at upload → transaction recorded → file uploaded to R2
- [ ] Modal processes document successfully → credits stay deducted (no refund)
- [ ] Modal processing fails (system error) → Modal calls refund endpoint → credits refunded
- [ ] Modal processing fails (user error) → Modal does NOT call refund → credits stay deducted
- [ ] User makes MCP query → 1 credit deducted
- [ ] User runs embedding search → 1 credit deducted
- [ ] User tries to upload without sufficient credits → 402 error, upload blocked
- [ ] Subscription renewal → credits added with rollover cap
- [ ] API key with dedicated credits works correctly
- [ ] Concurrent upload requests don't cause credit balance issues

---

### Phase 6: Deployment (Day 8)

**Goal**: Deploy to production with monitoring

#### 6.1: Migration Deployment (2-3 hours)

**Tasks**:
- [ ] Backup production database
- [ ] Run migration on production: `npm run db:push`
- [ ] Verify schema changes applied correctly
- [ ] Run seed script: `npm run db:seed:credits`
- [ ] Verify all existing users received 100 credits

#### 6.2: Backend Deployment (2-3 hours)

**Tasks**:
- [ ] Update environment variables (FastAPI)
- [ ] Deploy FastAPI service
- [ ] Run smoke tests on production API
- [ ] Verify credit endpoints working
- [ ] Check logs for errors

#### 6.3: Frontend Deployment (2-3 hours)

**Tasks**:
- [ ] Update environment variables (Next.js)
- [ ] Deploy Next.js application
- [ ] Verify credit UI components render
- [ ] Test credit purchase flow
- [ ] Verify Stripe webhook integration

#### 6.4: Monitoring Setup (2-4 hours)

**Tasks**:
- [ ] Add credit balance alerts (< 10 credits)
- [ ] Monitor transaction creation rate
- [ ] Track refund frequency
- [ ] Monitor for negative balance errors
- [ ] Set up dashboards for credit usage analytics

#### 6.5: Grace Period Activation (1 hour)

**Tasks**:
- [ ] Enable 3-month grace period feature flag
- [ ] Configure credit enforcement to warn but not block
- [ ] Schedule enforcement activation for 3 months later
- [ ] Document grace period in user communications

---

## Technical Details

### Modal Integration for Async Refunds

Since document processing happens asynchronously in Modal after upload, we need a callback mechanism for refunds:

**Upload Flow**:
1. User uploads document via FastAPI
2. FastAPI decorator deducts 10 credits immediately
3. FastAPI stores transaction_id in document metadata
4. FastAPI uploads file to R2 and triggers Modal job
5. FastAPI returns document_id to user (credits already deducted)

**Modal Processing Success**:
1. Modal processes document successfully
2. No callback needed - credits remain deducted

**Modal Processing Failure**:
1. Modal job fails during processing
2. Modal determines error type (system vs user error)
3. Modal calls FastAPI refund endpoint with transaction_id and error details
4. FastAPI validates and issues refund if appropriate
5. User sees refund in their transaction history

**Modal Code Update Required**:
```python
# In modal/modal_app.py or modal/services/document_processor.py

async def process_document_with_refund_handling(
    document_id: str,
    transaction_id: str,  # Passed from FastAPI
    fastapi_url: str,
    fastapi_api_key: str
):
    try:
        # Existing document processing logic
        result = await parse_document(...)
        await store_in_milvus(...)
        return result

    except DoclingError as e:
        # System error - trigger refund
        await call_fastapi_refund(
            fastapi_url=fastapi_url,
            transaction_id=transaction_id,
            error_type="system_error",
            error_message=str(e)
        )
        raise

    except ValidationError as e:
        # User error - no refund
        # Just re-raise, don't call refund endpoint
        raise

async def call_fastapi_refund(
    fastapi_url: str,
    transaction_id: str,
    error_type: str,
    error_message: str
):
    """Call FastAPI to refund credits for failed processing"""
    import httpx
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{fastapi_url}/api/credits/refund",
            json={
                "transaction_id": transaction_id,
                "error_type": error_type,
                "error_message": error_message
            },
            headers={"X-API-Key": fastapi_api_key}
        )
        return response.json()
```

### Race Condition Prevention

Use PostgreSQL row-level locking for atomic credit deduction:

```python
async def deduct_credits(user_id: str, amount: int, ...):
    async with db.transaction():
        # Lock the row to prevent concurrent modifications
        user = await db.execute(
            select(User)
            .where(User.id == user_id)
            .with_for_update()  # SELECT FOR UPDATE
        )

        if user.credits < amount:
            raise InsufficientCreditsError()

        # Deduct credits
        await db.execute(
            update(User)
            .where(User.id == user_id)
            .values(credits=User.credits - amount)
        )

        # Create transaction record
        txn = await create_transaction(...)

        return txn
```

### API Key vs User Credits

Priority order:
1. If request has API key → use `api_key_credits` table
2. If no API key → use user credits from `users` table
3. API keys inherit user_id and team_id for audit purposes

### Refund Decision Logic

```python
def should_refund(error: Exception) -> bool:
    """Determine if credits should be refunded based on error type"""

    # System errors - Full refund
    if isinstance(error, (ServiceUnavailableError, InternalServerError)):
        return True

    # User errors - No refund
    if isinstance(error, (ValidationError, InvalidFileError, QuotaExceededError)):
        return False

    # Partial processing - Proportional refund
    if isinstance(error, PartialProcessingError):
        # Handled separately with partial amount
        return True

    # Default: no refund for unknown errors
    return False
```

### Rollover Logic

```python
async def handle_subscription_renewal(user_id: str, new_credits: int):
    """Handle credit rollover on subscription renewal"""
    user = await get_user(user_id)
    plan = await get_user_plan(user_id)

    # Calculate rollover
    current_credits = user.credits
    max_rollover = plan.monthly_credits  # 1x plan limit

    # Apply rollover cap
    rollover_amount = min(current_credits, max_rollover)
    new_balance = rollover_amount + new_credits

    await update_user_credits(user_id, new_balance)
    await create_transaction(
        user_id=user_id,
        amount=new_credits,
        type="purchase",
        operation="subscription_renewal",
        metadata={
            "rollover": rollover_amount,
            "new_credits": new_credits,
            "plan": plan.id
        }
    )
```

## Testing Checklist

### Unit Tests
- [ ] Credit service: check_credits
- [ ] Credit service: deduct_credits
- [ ] Credit service: refund_credits
- [ ] Credit service: add_credits
- [ ] Decorator: requires_credits
- [ ] Refund logic: should_refund

### Integration Tests
- [ ] Document upload with credit deduction
- [ ] MCP query with credit deduction
- [ ] Embedding search with credit deduction
- [ ] System error triggers refund
- [ ] User error doesn't trigger refund
- [ ] API key credits work independently
- [ ] Stripe webhook grants credits
- [ ] Rollover calculation on renewal

### Load Tests
- [ ] Concurrent credit deductions don't cause race conditions
- [ ] High transaction volume doesn't slow system
- [ ] Database locks don't cause timeouts

### Frontend Tests
- [ ] Credit balance displays correctly
- [ ] Transaction history loads
- [ ] Low credit warning appears
- [ ] Purchase flow works
- [ ] Real-time updates work (if implemented)

## Rollback Plan

If issues arise during deployment:

1. **Database Rollback**:
   ```sql
   -- Remove credit columns from users table
   ALTER TABLE users DROP COLUMN credits;
   ALTER TABLE users DROP COLUMN total_credits_purchased;
   ALTER TABLE users DROP COLUMN team_id;

   -- Drop new tables
   DROP TABLE api_key_credits;
   DROP TABLE credit_transactions;
   DROP TABLE teams;
   ```

2. **Code Rollback**:
   - Revert FastAPI deployment to previous version
   - Remove credit decorator from endpoints
   - Revert Next.js deployment to remove credit UI

3. **Feature Flag**:
   - Disable credit enforcement via environment variable
   - Keep tracking in place but don't block requests

## Success Metrics

Track these metrics post-deployment:

- **Adoption**:
  - % of users with > 0 credits
  - Average credits purchased per user
  - Credit purchase conversion rate

- **Usage**:
  - Average credits consumed per user/month
  - Most common operations (document vs query vs search)
  - API key adoption rate

- **Health**:
  - Refund rate (should be < 5%)
  - Negative balance incidents (should be 0)
  - Race condition errors (should be 0)
  - Credit-related support tickets

- **Revenue**:
  - Monthly recurring revenue from credit purchases
  - Average revenue per user (ARPU)
  - Churn rate correlation with credit usage

## Future Enhancements

Post-launch improvements to consider:

1. **Team Features** (Phase 7):
   - Activate team tables
   - Implement team credit sharing
   - Add team admin controls
   - Team usage analytics

2. **Credit Bundles**:
   - One-time credit purchases (not just subscriptions)
   - Bulk discounts for large purchases
   - Gift credits to other users

3. **Usage Alerts**:
   - Email notifications at 80%, 90%, 100% usage
   - Slack/Discord webhooks for team usage
   - Budget controls and spending limits

4. **Advanced Analytics**:
   - Cost per document insights
   - ROI tracking for credit spend
   - Predictive usage forecasting

5. **Credit Marketplace**:
   - Transfer credits between users
   - Credit rewards program
   - Referral bonuses

## Credit Economics & Tier Allocation

### Monthly Credit Allocations

**Updated based on real production costs (Oct 2024)**

| Tier | Price | Credits/Month | Key Operations | Infrastructure Cost | Margin |
|------|-------|---------------|----------------|---------------------|--------|
| **Free** | $0 | **100 credits** | ~40-50 docs (<1MB) OR ~100 queries | ~$0.60-$0.75/user | Loss leader |
| **Paid** | $29 | **500 credits** | ~160-250 docs OR ~500 queries | ~$2.40-$3.75/user | **87-92% margin** |
| **Pro** | $79 | **1,500 credits** | ~500-750 docs OR ~1,500 queries | ~$7.50-$11.25/user | **85-90% margin** |
| **Enterprise** | Custom | **Custom pools** | Volume-based + API access | Negotiated | 70-80% margin |

**Note**: Mixed usage (docs + queries) typical. Average user: 70% queries, 30% uploads by credit spend.

### Credit Value Model

**1 credit = $0.01 in perceived user value**

This creates intuitive pricing where credits feel like "tokens" rather than money.

### Operation Costs (Detailed)

**Based on real production data from Modal processing logs (Oct 2024)**

| Operation | Credits | Real Infrastructure Cost | User Pays (perceived) | Markup | Rationale |
|-----------|---------|--------------------------|----------------------|--------|-----------|
| **Document Upload (< 1 MB)** | **2** | $0.015 | $0.02 | 1.3x | 20-page PDF: Modal ($0.0123) + OpenAI embeddings ($0.0012) + storage |
| **Document Upload (1-5 MB)** | **3** | $0.020-$0.040 | $0.03 | 1.5x | 50-100 pages: Scales with file size and page count |
| **Document Upload (5-10 MB)** | **6** | $0.040-$0.080 | $0.06 | 1.5x | Large documents with extended processing time |
| **Standard Query** | **1** | $0.002 | $0.01 | 5x | Vector search only, encourages engagement |
| **Hybrid Search** | **2** | $0.003 | $0.02 | 7x | Vector + BM25, better relevance |
| **Premium Query (GPT-4 + Rerank)** | **5** | $0.050 | $0.05 | 1x | Pass-through cost with self-hosted reranker |
| **Dataset Creation** | **2** | $0.002 | $0.02 | 10x | One-time Milvus collection setup |
| **Embedding Search** | **1** | $0.001 | $0.01 | 10x | Milvus query only, nearly free |

**Processing Cost Breakdown (20-page PDF example)**:
- Modal compute (35s × $0.000292/s): $0.0123 (87%)
- OpenAI embeddings (80 chunks): $0.0012 (9%)
- Milvus storage: $0.0001 (1%)
- R2 storage: <$0.0001 (<1%)
- PostgreSQL update: <$0.0001 (<1%)
- **Total**: ~$0.014 per typical document

**Future Enhancement Costs**:
- **With GPU acceleration**: ~$0.018 (+29%, but 30% faster)
- **With hybrid search**: +$0.001 per query
- **With self-hosted reranking**: +$0.00003 per query (vs $0.003 using Cohere)

### Usage Modeling

**Updated based on new credit pricing (Oct 2024)**

#### Free Tier User (100 credits)

**Pattern A - Dataset Creator**:
- Create 2 datasets (2 × 2): -4 credits
- Upload 40 small documents (<1MB, 2 × 40): -80 credits
- Run 16 standard queries: -16 credits
- **Result**: Can build 2 meaningful datasets with light testing

**Pattern B - Chat-Heavy User**:
- Create 1 dataset: -2 credits
- Upload 10 documents: -20 credits
- Run 78 standard queries: -78 credits
- **Result**: Heavy engagement, excellent conversion target

**Pattern C - Explorer**:
- Run 100 standard queries across public datasets: -100 credits
- **Result**: Tests value before committing to paid tier

#### Paid Tier User ($29/mo, 500 credits)

**Pattern A - Aviation Creator (Jumpseat Use Case)**:
- Create 3 datasets: -6 credits
- Upload 150 small PDFs (<1MB) (manuals, procedures): -300 credits
- Run 100 test queries: -100 credits
- Use hybrid search 47 times: -94 credits
- **Result**: Robust creator tier with advanced search

**Pattern B - Real Estate Consultant**:
- Create 2 client datasets: -4 credits
- Upload 60 property docs (1-5MB each): -180 credits
- Run 316 standard client queries: -316 credits
- **Result**: Service business with high query volume

**Pattern C - Research Student**:
- Create 1 literature dataset: -2 credits
- Upload 100 papers (<1MB): -200 credits
- Run 200 research queries: -200 credits
- Use hybrid search 49 times: -98 credits
- **Result**: Academic research support

#### Pro Tier User ($79/mo, 1,500 credits)

**Pattern A - Estate Planning Law Firm**:
- Create 10 client datasets: -20 credits
- Upload 400 estate documents (1-5MB): -1,200 credits
- Run 280 research queries: -280 credits
- **Result**: Professional firm with high document volume

**Pattern B - Healthcare Compliance Team**:
- Create 5 policy datasets: -10 credits
- Upload 200 large policy docs (5-10MB): -1,200 credits
- Run 145 standard queries: -145 credits
- Use hybrid search 72 times: -144 credits
- **Result**: Enterprise-level compliance management

**Pattern C - Content Creator Platform**:
- Create 20 topic datasets: -40 credits
- Upload 500 small articles: -1,000 credits
- Run 460 queries for content research: -460 credits
- **Result**: Power user with broad knowledge base

### One-Time Credit Packs (Future Enhancement)

For users who hit monthly limits:
- **100 credits**: $5 (5¢ per credit)
- **500 credits**: $20 (4¢ per credit) - 20% discount
- **2,000 credits**: $60 (3¢ per credit) - 40% discount

**Purpose**: Capture overflow revenue without forcing tier upgrades

### 6-Month Financial Projection

**Based on updated credit pricing and real production costs (Oct 2024)**

Assumptions:
- 1,000 total users at 6 months
- Mix of document uploads (30% of credits) and queries (70% of credits)
- Average document size: 1.5 MB (2.5 credits)
- Free tier: 80% usage, Paid: 90% usage, Pro: 85% usage

| Segment | Users | Avg Credits Used | Infra Cost | Revenue | Margin |
|---------|-------|------------------|------------|---------|--------|
| Free | 900 | 80 | $540/mo | $0 | -$540 |
| Paid | 80 | 450 | $324/mo | $2,320/mo | **$1,996 (86%)** |
| Pro | 20 | 1,275 | $229/mo | $1,580/mo | **$1,351 (86%)** |
| **Total** | **1,000** | - | **$1,093/mo** | **$3,900/mo** | **$2,807/mo (72%)** |

**Cost breakdown** (typical usage pattern):
- Documents: 30% of credits @ 2.5 avg = ~$0.015 actual cost per credit
- Queries: 70% of credits @ 1 avg = ~$0.002 actual cost per credit
- **Blended cost**: ~$0.006 per credit spent

**Plus marketplace revenue**: 50 creators × $10 avg sale × 20% = $100/mo

**Total MRR at 6 months**: ~$4,000/mo with **72% margin**

**Key insight**: Lower markup on documents (1.3x vs 20x previously) but higher volume capacity makes tier limits more generous and user-friendly.

---

## Current Status

**Overall Progress**: Phase 1 Complete (12.5%)

**Last Updated**: October 31, 2024

**Recent Updates**:
- Credit pricing updated based on real Modal production costs
- File-size-based document processing pricing (2-25 credits)
- Hybrid search and reranking operations added
- Usage modeling updated to reflect actual infrastructure costs
- Financial projections recalculated with 72% margin target

**Next Action**: Begin Phase 2.1 - Create SQLModel models for credit system

---

## Questions & Decisions Log

### Resolved Decisions

1. **Where to store credits?**
   - ✅ Decision: Better Auth database (Next.js)
   - Rationale: Centralized user management, existing infrastructure

2. **How to handle API keys?**
   - ✅ Decision: Dedicated credit pools with limits
   - Rationale: Enables fine-grained control, team billing support

3. **Refund policy?**
   - ✅ Decision: Smart refunds (system errors only)
   - Rationale: Fair to users, prevents abuse

4. **Credit expiration?**
   - ✅ Decision: Rollover with 1x plan cap
   - Rationale: User-friendly, prevents hoarding

5. **Rollout strategy?**
   - ✅ Decision: 100 free credits + 3-month grace
   - Rationale: Smooth transition, builds trust

### Open Questions

None currently - all key decisions resolved.

---

## 🎯 What's Remaining - Priority Roadmap

### ✅ COMPLETE (Nov 3, 2025)

**Phase 1: Database Schema**
- ✅ All tables created and migrated
- ✅ Seed script for initial 100 credits

**Phase 2: FastAPI Credit Service**
- ✅ SQLModel models
- ✅ Credit repository (280 lines, 18 tests)
- ✅ Credit service (562 lines, 22 tests)
- ✅ Credit decorator with dynamic lambda support (203 lines, 10 tests)
- ✅ API key credit pool support
- ✅ Credit router with 3 core endpoints (147 lines)

**Phase 3: Router Integration (Partial)**
- ✅ Dataset document upload with credit gate (10 credits per file)

**Phase 4: UI Integration (Basic)**
- ✅ Credit balance display component
- ✅ Credit meter component
- ✅ Usage display component
- ✅ Billing page integration with "Usage" tab
- ✅ Plan definitions updated with credit allocations
- ✅ Orval API client generated

**Test Status**:
- ✅ **236 FastAPI tests passing** (50 credit-specific)
- ✅ **41 Next.js tests passing**

---

### 🔴 HIGH PRIORITY - Core Functionality

#### 1. MCP & Embedding Router Credit Gates (Phase 3.2 & 3.3)
**Why**: These are billable operations currently running without credit enforcement
**Effort**: 2-3 hours
**Files**:
- `fastapi/routers/mcp.py` - Add `@requires_credits` to query endpoints
- `fastapi/routers/embedding.py` - Add `@requires_credits` to search endpoints

**Tasks**:
- Add credit gates to MCP endpoints (1 credit per query)
- Add credit gates to embedding search (1 credit per search)
- Implement OAuth 2.1 authentication (from AUTH_RATE_LIMIT_PLAN.md)
- Update PUBLIC_ROUTES to require auth for MCP

#### 2. Stripe Webhook Integration (Phase 4.1)
**Why**: Users need to receive credits when they subscribe/renew
**Effort**: 4-6 hours
**Files**:
- `nextjs/src/lib/stripe/webhooks.ts` or Better Auth config

**Tasks**:
- Grant credits on subscription creation/renewal
- Implement rollover logic (max 1x plan limit)
- Handle subscription upgrades/downgrades
- Test with Stripe test mode

---

### 🟡 MEDIUM PRIORITY - Enhanced UX

#### 3. Modal Refund Callback (Phase 3.1 continuation)
**Why**: Document processing failures should refund credits automatically
**Effort**: 2-3 hours

**Tasks**:
- Create `POST /api/v1/credits/refund` endpoint
- Update Modal job to call endpoint on system errors
- Add authentication for Modal callbacks
- Test refund flow end-to-end

#### 4. Navigation Credit Balance Display (Phase 4A continuation)
**Why**: Users should see credit balance without visiting billing page
**Effort**: 1 hour

**Tasks**:
- Add `<CreditBalance />` to app sidebar
- Add credit balance to user dropdown menu
- Make components link to `/billing?tab=usage`

#### 5. Advanced UI Components (Phase 4B)
**Why**: Better visibility into credit usage patterns
**Effort**: 6-8 hours

**Tasks**:
- Transaction history table with pagination
- Usage charts (Chart.js or Recharts)
- Filtering by transaction type and date range
- CSV export functionality
- Low credit warning banner (< 20 credits)

---

### 🟢 LOW PRIORITY - Nice to Have

#### 6. API Key Management Endpoints
**Why**: Can be managed via database/UI initially
**Effort**: 3-4 hours

**Tasks**:
- `GET /api/v1/credits/api-keys` - List pools
- `POST /api/v1/credits/api-keys` - Create pool
- `PUT /api/v1/credits/api-keys/{key}` - Update limits
- Add UI for managing API key credit pools

#### 7. Credit Purchase Endpoint
**Why**: Stripe webhooks handle this initially
**Effort**: 2-3 hours

**Tasks**:
- `POST /api/v1/credits/purchase` - Trigger Stripe checkout
- Handle one-time credit pack purchases
- Add UI for buying credit packs outside subscription

#### 8. Real-time Updates
**Why**: Current 30-second polling is sufficient
**Effort**: 4-6 hours

**Tasks**:
- WebSocket connection for real-time balance updates
- Server-sent events for transaction notifications
- Optimistic UI updates

---

## 📊 Implementation Status Summary

| Phase | Component | Status | Tests | Priority |
|-------|-----------|--------|-------|----------|
| 1 | Database Schema | ✅ Complete | N/A | - |
| 2.1 | SQLModel Models | ✅ Complete | ✅ 40/40 | - |
| 2.2 | Credit Repository | ✅ Complete | ✅ 18/18 | - |
| 2.3 | Credit Service | ✅ Complete | ✅ 22/22 | - |
| 2.5 | Credit Decorator | ✅ Complete | ✅ 10/10 | - |
| 2.6 | Credit API (Basic) | ✅ Complete | ⚠️ Indirect | - |
| 3.1 | Dataset Router | ✅ Complete | ✅ 30/30 | - |
| 3.2 | MCP Router | ❌ Not Started | N/A | 🔴 High |
| 3.3 | Embedding Router | ❌ Not Started | N/A | 🔴 High |
| 4.1 | Stripe Webhooks | ❌ Not Started | N/A | 🔴 High |
| 4.2 | Credit Display (Basic) | ✅ Complete | N/A | - |
| 4.3 | Credit History (Advanced) | ❌ Not Started | N/A | 🟡 Medium |
| 4.4 | Navigation Display | ⚠️ Partial | N/A | 🟡 Medium |

**Overall Progress**: ~70% complete (core system operational, enhancements pending)

---

## 🚀 Recommended Next Steps

**Immediate (This Week)**:
1. Add credit gates to MCP and Embedding routers
2. Implement Stripe webhook credit granting
3. Add Modal refund callback endpoint

**Short-term (Next Week)**:
4. Add credit balance to navigation/sidebar
5. Create transaction history table
6. Implement OAuth 2.1 for MCP (from AUTH_RATE_LIMIT_PLAN.md)

**Long-term (Next Month)**:
7. Advanced UI features (charts, CSV export, warnings)
8. API key management UI
9. Real-time credit updates
10. One-time credit pack purchases

---

## References

### Database Architecture
- **[DATABASE_ARCHITECTURE.md](DATABASE_ARCHITECTURE.md)** - Comprehensive two-database architecture guide
- **[DATABASES.md](DATABASES.md)** - Quick reference for developers
- **Database Schema**: [nextjs/src/database/schema.ts](nextjs/src/database/schema.ts)
- **Seed Script**: [nextjs/src/database/seeds/initial-credits.ts](nextjs/src/database/seeds/initial-credits.ts)
- **Migration**: [nextjs/migrations/0001_abandoned_human_fly.sql](nextjs/migrations/0001_abandoned_human_fly.sql)

### Cost Analysis
- **Document Processing Costs**: [DOCUMENT_PROCESSING_COSTS.md](DOCUMENT_PROCESSING_COSTS.md) - Real production cost analysis
- **Cloud Infrastructure**: [CLOUD_COSTS.md](CLOUD_COSTS.md) - Complete infrastructure cost breakdown

### External Documentation
- **Better Auth Docs**: https://www.better-auth.com/
- **Stripe Integration**: https://www.better-auth.com/docs/plugins/stripe
- **Drizzle ORM**: https://orm.drizzle.team/
- **SQLModel**: https://sqlmodel.tiangolo.com/
