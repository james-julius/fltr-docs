# FLTR Database Schema Design

## Overview
FLTR is a context-as-a-service platform for AI applications. The database schema needs to support:
- Dataset management and curation
- User authentication and profiles
- Monetization (subscriptions, usage tracking)
- Vector storage (Milvus) for semantic search
- MCP endpoint generation and management

## Architecture

### SQL Database (PostgreSQL/SQLite)
Handles relational data, user management, metadata, and transactions.

### Vector Database (Milvus)
Handles embeddings and semantic search for dataset content.

---

## Core Entities

### 1. Users
Manages user accounts, authentication, and profiles.

```sql
users
├── id (uuid, primary key)
├── email (string, unique, indexed)
├── username (string, unique, indexed)
├── full_name (string, nullable)
├── avatar_url (string, nullable)
├── role (enum: curator, developer, admin)
├── is_verified (boolean)
├── stripe_customer_id (string, nullable, indexed)
├── created_at (timestamp)
└── updated_at (timestamp)
```

**Relationships:**
- One-to-many with Datasets (as owner)
- One-to-many with Subscriptions
- One-to-many with ApiKeys

---

### 2. Datasets (Filters)
Core entity representing curated knowledge bases.

```sql
datasets
├── id (uuid, primary key)
├── owner_id (uuid, foreign key -> users.id)
├── name (string, indexed)
├── slug (string, unique, indexed)
├── description (text)
├── category (enum: medical, legal, travel, finance, tech, other)
├── tags (array/jsonb)
├── visibility (enum: public, private, premium)
├── status (enum: uploading, processing, ready, failed)
├── collection_name (string, indexed) -- Milvus collection reference
├── document_count (integer)
├── embedding_dimension (integer)
├── embedding_model (string)
├── pricing_model (enum: free, one_time, subscription)
├── price_monthly (decimal, nullable)
├── price_annual (decimal, nullable)
├── rating_average (decimal)
├── download_count (integer)
├── view_count (integer)
├── featured (boolean)
├── verified (boolean)
├── metadata (jsonb) -- Custom fields, sources, etc.
├── created_at (timestamp)
└── updated_at (timestamp)
```

**Relationships:**
- Many-to-one with Users (owner)
- One-to-many with DatasetVersions
- Many-to-many with Tags
- One-to-many with Reviews
- One-to-many with Subscriptions

**Milvus Collection Schema:**
Each dataset gets its own Milvus collection with:
- `vector` (embedding)
- `text` (original text chunk)
- `metadata` (source, page, etc.)
- `dataset_id` (reference back to SQL)

---

### 3. Dataset Versions
Track dataset updates without breaking integrations.

```sql
dataset_versions
├── id (uuid, primary key)
├── dataset_id (uuid, foreign key -> datasets.id)
├── version (string) -- e.g., "1.0.0", "1.1.0"
├── changelog (text)
├── collection_name (string) -- Milvus collection for this version
├── is_active (boolean)
├── document_count (integer)
├── created_at (timestamp)
└── created_by (uuid, foreign key -> users.id)
```

**Relationships:**
- Many-to-one with Datasets

---

### 4. Dataset Sources
Track where data comes from for each dataset.

```sql
dataset_sources
├── id (uuid, primary key)
├── dataset_id (uuid, foreign key -> datasets.id)
├── source_type (enum: pdf, csv, url, api, manual)
├── source_url (string, nullable)
├── source_name (string)
├── file_size (bigint, nullable)
├── file_path (string, nullable)
├── last_synced_at (timestamp, nullable)
├── sync_frequency (enum: manual, daily, weekly, monthly, nullable)
├── is_active (boolean)
├── metadata (jsonb)
├── created_at (timestamp)
└── updated_at (timestamp)
```

**Relationships:**
- Many-to-one with Datasets

---

### 5. Tags / Categories
Organize and discover datasets.

```sql
tags
├── id (uuid, primary key)
├── name (string, unique, indexed)
├── slug (string, unique, indexed)
├── description (text, nullable)
├── icon (string, nullable)
└── usage_count (integer)

dataset_tags (junction table)
├── dataset_id (uuid, foreign key -> datasets.id)
├── tag_id (uuid, foreign key -> tags.id)
└── PRIMARY KEY (dataset_id, tag_id)
```

---

### 6. Subscriptions
Track who can access premium datasets.

```sql
subscriptions
├── id (uuid, primary key)
├── user_id (uuid, foreign key -> users.id)
├── dataset_id (uuid, foreign key -> datasets.id)
├── plan (enum: monthly, annual)
├── status (enum: active, cancelled, expired, failed)
├── stripe_subscription_id (string, nullable, indexed)
├── current_period_start (timestamp)
├── current_period_end (timestamp)
├── cancel_at_period_end (boolean)
├── created_at (timestamp)
└── updated_at (timestamp)
```

**Relationships:**
- Many-to-one with Users
- Many-to-one with Datasets

---

### 7. API Keys
MCP endpoint authentication.

```sql
api_keys
├── id (uuid, primary key)
├── user_id (uuid, foreign key -> users.id)
├── name (string)
├── key_hash (string, indexed) -- Hashed API key
├── prefix (string) -- First few chars for display (e.g., "fltr_12345...")
├── permissions (jsonb) -- Scopes: read, write, admin
├── rate_limit_per_minute (integer)
├── last_used_at (timestamp, nullable)
├── expires_at (timestamp, nullable)
├── is_active (boolean)
├── created_at (timestamp)
└── revoked_at (timestamp, nullable)
```

**Relationships:**
- Many-to-one with Users
- One-to-many with ApiUsage

---

### 8. API Usage / Analytics
Track API calls for billing and analytics.

```sql
api_usage
├── id (uuid, primary key)
├── api_key_id (uuid, foreign key -> api_keys.id)
├── dataset_id (uuid, foreign key -> datasets.id, nullable)
├── endpoint (string)
├── method (string)
├── status_code (integer)
├── response_time_ms (integer)
├── tokens_used (integer, nullable) -- For embedding calls
├── timestamp (timestamp, indexed)
└── metadata (jsonb)
```

**Relationships:**
- Many-to-one with ApiKeys
- Many-to-one with Datasets (nullable)

**Note:** Consider time-series database or partitioning for high volume.

---

### 9. Reviews / Ratings
Community feedback on datasets.

```sql
reviews
├── id (uuid, primary key)
├── dataset_id (uuid, foreign key -> datasets.id)
├── user_id (uuid, foreign key -> users.id)
├── rating (integer) -- 1-5 stars
├── title (string)
├── comment (text)
├── helpful_count (integer)
├── verified_purchase (boolean)
├── created_at (timestamp)
└── updated_at (timestamp)
```

**Relationships:**
- Many-to-one with Datasets
- Many-to-one with Users

---

### 10. Curator Earnings
Track revenue sharing for data providers.

```sql
curator_earnings
├── id (uuid, primary key)
├── curator_id (uuid, foreign key -> users.id)
├── dataset_id (uuid, foreign key -> datasets.id)
├── subscription_id (uuid, foreign key -> subscriptions.id, nullable)
├── amount_cents (bigint)
├── currency (string)
├── revenue_share_percentage (decimal) -- 85%, 90%, etc.
├── payout_status (enum: pending, processing, paid, failed)
├── stripe_transfer_id (string, nullable)
├── period_start (timestamp)
├── period_end (timestamp)
├── created_at (timestamp)
└── paid_at (timestamp, nullable)
```

**Relationships:**
- Many-to-one with Users (curator)
- Many-to-one with Datasets
- Many-to-one with Subscriptions (nullable)

---

## Indexes & Performance

### Critical Indexes
```sql
-- Users
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_stripe_customer ON users(stripe_customer_id);

-- Datasets
CREATE INDEX idx_datasets_owner ON datasets(owner_id);
CREATE INDEX idx_datasets_slug ON datasets(slug);
CREATE INDEX idx_datasets_category ON datasets(category);
CREATE INDEX idx_datasets_visibility_status ON datasets(visibility, status);
CREATE INDEX idx_datasets_featured ON datasets(featured) WHERE featured = true;

-- API Keys
CREATE INDEX idx_api_keys_key_hash ON api_keys(key_hash);
CREATE INDEX idx_api_keys_user ON api_keys(user_id);

-- API Usage (consider partitioning by timestamp)
CREATE INDEX idx_api_usage_timestamp ON api_usage(timestamp);
CREATE INDEX idx_api_usage_api_key ON api_usage(api_key_id);
CREATE INDEX idx_api_usage_dataset ON api_usage(dataset_id);

-- Subscriptions
CREATE INDEX idx_subscriptions_user ON subscriptions(user_id);
CREATE INDEX idx_subscriptions_dataset ON subscriptions(dataset_id);
CREATE INDEX idx_subscriptions_status ON subscriptions(status);
```

---

## Migration Strategy

### Phase 1: MVP (Current)
- Users (basic)
- Datasets (basic)
- API Keys

### Phase 2: Marketplace
- Tags/Categories
- Reviews
- Subscriptions
- Dataset Sources

### Phase 3: Monetization
- Curator Earnings
- API Usage tracking
- Dataset Versions

### Phase 4: Scale
- Time-series analytics
- Caching layer (Redis)
- Read replicas

---

## Technology Stack

**SQL Database:**
- Development: SQLite (simple, file-based)
- Production: PostgreSQL (scalable, JSONB support)

**Vector Database:**
- Development: Milvus Lite (embedded, no server needed)
- Production: Zilliz Cloud or self-hosted Milvus

**ORM:**
- SQLModel (type-safe, Pydantic integration)

**Migrations:**
- Alembic (SQLModel compatible)

---

## Security Considerations

1. **API Keys:** Store hashed, never plaintext
2. **User Data:** Encrypt PII at rest
3. **Rate Limiting:** Per API key, per user
4. **Access Control:** Row-level security for private datasets
5. **Audit Logs:** Track all data access for compliance

---

## Next Steps

1. Implement MVP models (Users, Datasets, API Keys)
2. Set up Alembic migrations
3. Create FastAPI CRUD operations
4. Build MCP endpoint generator
5. Integrate with Next.js frontend
