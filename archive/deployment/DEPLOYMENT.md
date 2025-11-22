# FLTR Deployment Guide

## Database Migrations on Vercel

### Overview
The credit system requires database schema changes. Here's how to handle migrations for Vercel deployment.

### Migration Files Location
- Location: `/nextjs/migrations/`
- Files:
  - `0000_nifty_bruce_banner.sql` - Initial Better Auth schema
  - `0001_abandoned_human_fly.sql` - Credit system schema

### Running Migrations

#### Option 1: Automatic on Deploy (Recommended)
Add a build command to apply migrations before building:

```json
// nextjs/package.json
{
  "scripts": {
    "build": "npm run db:push && next build",
    "vercel-build": "npm run db:push && next build"
  }
}
```

#### Option 2: Manual via Vercel CLI
```bash
# Install Vercel CLI
npm i -g vercel

# Link to your project
vercel link

# Run migration command
vercel env pull .env.local
npm run db:push
```

#### Option 3: Direct SQL Execution
Connect to your production database and run the SQL manually:

```sql
-- From 0001_abandoned_human_fly.sql

-- Create credit tables
CREATE TABLE IF NOT EXISTS "api_key_credits" (
    "api_key" text PRIMARY KEY NOT NULL,
    "user_id" text NOT NULL,
    "team_id" text,
    "credits" integer DEFAULT 0 NOT NULL,
    "daily_limit" integer,
    "monthly_limit" integer,
    "created_at" timestamp NOT NULL,
    "updated_at" timestamp NOT NULL
);

CREATE TABLE IF NOT EXISTS "credit_transactions" (
    "id" text PRIMARY KEY NOT NULL,
    "user_id" text NOT NULL,
    "team_id" text,
    "amount" integer NOT NULL,
    "type" text NOT NULL,
    "operation" text,
    "resource_id" text,
    "api_key_id" text,
    "metadata" text,
    "created_at" timestamp NOT NULL
);

CREATE TABLE IF NOT EXISTS "teams" (
    "id" text PRIMARY KEY NOT NULL,
    "name" text NOT NULL,
    "credits" integer DEFAULT 0 NOT NULL,
    "owner_id" text,
    "created_at" timestamp NOT NULL,
    "updated_at" timestamp NOT NULL
);

-- Add credit columns to users table
ALTER TABLE "users" ADD COLUMN IF NOT EXISTS "credits" integer DEFAULT 0 NOT NULL;
ALTER TABLE "users" ADD COLUMN IF NOT EXISTS "total_credits_purchased" integer DEFAULT 0 NOT NULL;
ALTER TABLE "users" ADD COLUMN IF NOT EXISTS "team_id" text;

-- Add foreign key constraints
ALTER TABLE "api_key_credits" ADD CONSTRAINT "api_key_credits_user_id_users_id_fk"
    FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;

ALTER TABLE "api_key_credits" ADD CONSTRAINT "api_key_credits_team_id_teams_id_fk"
    FOREIGN KEY ("team_id") REFERENCES "public"."teams"("id") ON DELETE cascade ON UPDATE no action;

ALTER TABLE "credit_transactions" ADD CONSTRAINT "credit_transactions_user_id_users_id_fk"
    FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;

ALTER TABLE "credit_transactions" ADD CONSTRAINT "credit_transactions_team_id_teams_id_fk"
    FOREIGN KEY ("team_id") REFERENCES "public"."teams"("id") ON DELETE set null ON UPDATE no action;

ALTER TABLE "teams" ADD CONSTRAINT "teams_owner_id_users_id_fk"
    FOREIGN KEY ("owner_id") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;

ALTER TABLE "users" ADD CONSTRAINT "users_team_id_teams_id_fk"
    FOREIGN KEY ("team_id") REFERENCES "public"."teams"("id") ON DELETE no action ON UPDATE no action;
```

### Environment Variables Required

#### Next.js (Vercel)
```bash
# Database
DATABASE_URL=postgresql://...  # Vercel Postgres URL
AUTH_DATABASE_URL=postgresql://...  # Same as DATABASE_URL

# FastAPI Backend URL
NEXT_PUBLIC_API_URL=https://your-fastapi-domain.com

# Stripe
STRIPE_SECRET_KEY=sk_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Better Auth
BETTER_AUTH_SECRET=...
BETTER_AUTH_URL=https://your-app.vercel.app
```

#### FastAPI (Separate Deployment)
```bash
# Database
DATABASE_URL=postgresql://...  # Main database
AUTH_DATABASE_URL=postgresql://...  # Better Auth database (same as DATABASE_URL)

# CORS
CORS_ORIGINS=["https://your-app.vercel.app"]

# OpenAI
OPENAI_API_KEY=sk-...

# Milvus
USE_MILVUS_LITE=false
MILVUS_URI=https://...zillizcloud.com
MILVUS_TOKEN=...

# Cloudflare
R2_ACCESS_KEY_ID=...
R2_SECRET_ACCESS_KEY=...
R2_ACCOUNT_ID=...
R2_BUCKET_NAME=...

# Modal
MODAL_ENABLED=true
MODAL_WEBHOOK_URL=https://...modal.run/process
```

### Verifying Migration Success

After deployment, verify the migration was successful:

1. Check Vercel deployment logs for migration output
2. Connect to database and verify tables exist:
```sql
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public'
  AND table_name IN ('credit_transactions', 'teams', 'api_key_credits');

SELECT column_name
FROM information_schema.columns
WHERE table_name = 'users'
  AND column_name IN ('credits', 'total_credits_purchased', 'team_id');
```

3. Test the credit balance endpoint:
```bash
curl https://your-api.com/api/v1/credits/balance \
  -H "Authorization: Bearer YOUR_SESSION_TOKEN"
```

### Initial Credit Allocation

After migration, grant initial credits to existing users:

```bash
# Run the seed script
npm run db:seed:credits
```

Or manually via SQL:
```sql
UPDATE users
SET credits = 100, total_credits_purchased = 100
WHERE credits = 0;
```

### Troubleshooting

#### Migration fails with "relation already exists"
- Migrations are partially applied
- Run only the missing ALTER TABLE statements manually

#### "column does not exist" errors
- Migration didn't run
- Check deployment logs
- Run `npm run db:push` manually

#### Credit balance shows 0 or null
- Migration successful but no credits allocated
- Run seed script or manual UPDATE query

### Rollback Plan

If you need to rollback the credit system:

```sql
-- Remove credit columns from users
ALTER TABLE "users" DROP COLUMN IF EXISTS "credits";
ALTER TABLE "users" DROP COLUMN IF EXISTS "total_credits_purchased";
ALTER TABLE "users" DROP COLUMN IF EXISTS "team_id";

-- Drop credit tables
DROP TABLE IF EXISTS "api_key_credits" CASCADE;
DROP TABLE IF EXISTS "credit_transactions" CASCADE;
DROP TABLE IF EXISTS "teams" CASCADE;
```

## FastAPI Deployment

### Recommended Platforms
1. **Railway** - Easiest, built-in PostgreSQL
2. **Render** - Good free tier
3. **Fly.io** - Global edge deployment
4. **Google Cloud Run** - Serverless

### Deployment Checklist
- [ ] Set all environment variables
- [ ] Configure CORS_ORIGINS to include Vercel domain
- [ ] Update AUTH_DATABASE_URL to production Postgres
- [ ] Set up Milvus Cloud (Zilliz) connection
- [ ] Configure Modal webhook URL
- [ ] Test credit endpoints with session auth

### Health Checks
```bash
# Basic health
curl https://your-api.com/health

# Test authenticated endpoint
curl https://your-api.com/api/v1/credits/balance \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json"
```
