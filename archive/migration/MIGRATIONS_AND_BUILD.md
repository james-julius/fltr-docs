# Database Migrations & Build Process

## Overview

FLTR has separate build commands for **local development** and **production deployment** to handle database migrations appropriately.

## Build Commands

### Local Development: `pnpm build`

```bash
pnpm build
```

**What it does:**
1. ✅ Generates/checks API client (FastAPI → TypeScript)
2. ✅ Builds Next.js application
3. ⊘ **Does NOT run migrations** (no DATABASE_URL required)

**Use cases:**
- Local builds for testing
- Pre-push hooks
- CI builds that don't need migrations
- When you don't have DATABASE_URL set locally

### Production: `pnpm build:prod`

```bash
pnpm build:prod
```

**What it does:**
1. ✅ Runs database migrations (`pnpm db:migrate`)
2. ✅ Generates/checks API client
3. ✅ Builds Next.js application

**Use cases:**
- Vercel deployments
- Production builds
- Staging environments
- Anytime migrations need to run

## Database Migrations

### Manual Migration (Local Development)

When you change the database schema:

```bash
# 1. Update schema
vim src/database/schema.ts

# 2. Generate migration
pnpm db:generate

# 3. Apply migration (requires DATABASE_URL)
pnpm db:migrate
```

**Environment variable required:**
```bash
DATABASE_URL=postgresql://user:password@localhost:5432/fltr_auth
```

### Automatic Migration (Production)

Vercel automatically runs migrations via:

```json
// vercel.json
{
  "buildCommand": "pnpm build:prod"
}
```

**Environment variable set in Vercel:**
- `DATABASE_URL` - Points to production PostgreSQL database

## Pre-Push Hook Behavior

Your `.githooks/pre-push` hook runs `pnpm build` (NOT `pnpm build:prod`):

```bash
#!/usr/bin/env bash
echo "[nextjs pre-push] Running build (API client generation only, no DB migrations)..."
pnpm build
```

**Why no migrations in pre-push?**
- ✅ **Faster pushes** - No database connection required
- ✅ **Works offline** - Can push without DATABASE_URL
- ✅ **Safer** - Won't accidentally modify production database
- ✅ **Cleaner** - Migrations run once in production, not on every push

## Vercel Deployment Flow

```
git push
    ↓
Vercel receives push
    ↓
pnpm install
    ↓
pnpm build:prod
    ↓
┌─────────────────────────────┐
│ 1. pnpm db:migrate          │
│    ✅ Migrations run         │
└─────────────────────────────┘
    ↓
┌─────────────────────────────┐
│ 2. API client generation    │
│    ✅ Uses committed files   │
└─────────────────────────────┘
    ↓
┌─────────────────────────────┐
│ 3. next build               │
│    ✅ Production build       │
└─────────────────────────────┘
    ↓
✅ Deploy
```

## Common Workflows

### Workflow 1: Schema Change

```bash
# 1. Modify schema
vim src/database/schema.ts

# 2. Generate migration file
pnpm db:generate

# 3. Apply migration locally (test it)
pnpm db:migrate

# 4. Commit everything
git add src/database/schema.ts migrations/
git commit -m "feat: add new user field"

# 5. Push (pre-push hook runs pnpm build)
git push

# 6. Vercel deploys (runs pnpm build:prod with migrations)
```

### Workflow 2: Code-Only Change

```bash
# 1. Modify code
vim src/app/page.tsx

# 2. Commit
git commit -am "fix: update landing page"

# 3. Push (pre-push hook runs pnpm build - no migrations needed)
git push

# 4. Vercel deploys (runs pnpm build:prod - migrations already applied)
```

### Workflow 3: FastAPI Schema Change

```bash
# 1. Modify FastAPI endpoints
vim ../fastapi/routers/datasets.py

# 2. Regenerate API client
pnpm generate:api

# 3. Pre-push hook automatically regenerates + commits
git push

# 4. Vercel uses committed API client
```

## Environment Variables

### Local Development

```bash
# Optional - only needed if running migrations locally
DATABASE_URL=postgresql://user:password@localhost:5432/fltr_auth
```

### Vercel Production

Required environment variables in Vercel dashboard:

```bash
# Database
DATABASE_URL=postgresql://...           # Production PostgreSQL

# API
NEXT_PUBLIC_API_URL=https://fltr-api.geniedi.com

# Auth (Better Auth)
BETTER_AUTH_SECRET=...
BETTER_AUTH_URL=https://your-domain.com

# OAuth providers
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GITHUB_CLIENT_ID=...
GITHUB_CLIENT_SECRET=...

# Stripe
STRIPE_SECRET_KEY=...
STRIPE_WEBHOOK_SECRET=...
```

## Troubleshooting

### "DATABASE_URL is not set" during local build

**Symptom**: `pnpm build` fails with DATABASE_URL error

**Solution**: Use `pnpm build` (not `pnpm build:prod`) for local development

### "DATABASE_URL is not set" during git push

**Symptom**: Pre-push hook fails

**Cause**: Hook is running `pnpm build:prod` instead of `pnpm build`

**Solution**: Check `.githooks/pre-push` runs `pnpm build`

### Migrations not running in production

**Symptom**: Production database schema is outdated

**Cause**: Vercel `buildCommand` not using `pnpm build:prod`

**Solution**: Check `vercel.json` has `"buildCommand": "pnpm build:prod"`

### Migration fails in production

**Symptom**: Vercel build fails during migration step

**Possible causes:**
1. `DATABASE_URL` not set in Vercel environment
2. Database not accessible from Vercel
3. Migration syntax error
4. Breaking schema change

**Debug steps:**
1. Check Vercel logs for specific error
2. Verify `DATABASE_URL` environment variable
3. Test migration locally first: `pnpm db:migrate`
4. Check database connection from Vercel

## Scripts Reference

| Script | Command | Migrations | API Client | Next.js Build | Use Case |
|--------|---------|------------|------------|---------------|----------|
| `dev` | `next dev --turbopack` | ❌ | ❌ | ❌ | Local dev server |
| `build` | `generate-api-safe.mjs && next build` | ❌ | ✅ | ✅ | Local builds, pre-push |
| `build:prod` | `db:migrate && generate-api-safe.mjs && next build` | ✅ | ✅ | ✅ | Production, Vercel |
| `db:migrate` | `tsx src/database/migrate.ts` | ✅ | ❌ | ❌ | Run migrations manually |
| `db:generate` | `drizzle-kit generate` | 🔧 | ❌ | ❌ | Generate migration files |
| `db:push` | `drizzle-kit push` | 🚀 | ❌ | ❌ | Push schema without migrations |
| `generate:api` | `orval --config orval.config.ts` | ❌ | ✅ | ❌ | Regenerate API client |

## File Locations

- **Migration runner**: [src/database/migrate.ts](src/database/migrate.ts)
- **Schema definition**: [src/database/schema.ts](src/database/schema.ts)
- **Migration files**: [migrations/](migrations/)
- **Vercel config**: [vercel.json](vercel.json)
- **Pre-push hook**: [.githooks/pre-push](.githooks/pre-push)
- **API generation**: [scripts/generate-api-safe.mjs](scripts/generate-api-safe.mjs)

## Best Practices

1. ✅ **Always test migrations locally** before pushing
2. ✅ **Commit migration files** with schema changes
3. ✅ **Use `pnpm build`** for local testing
4. ✅ **Let Vercel run migrations** automatically
5. ❌ **Don't run migrations in pre-push hooks**
6. ❌ **Don't skip migration testing**
7. ❌ **Don't modify migrations after they're deployed**

## Related Documentation

- [DATABASE_ARCHITECTURE.md](DATABASE_ARCHITECTURE.md) - Two-database architecture
- [DATABASES.md](DATABASES.md) - Quick reference for developers
- [API_CLIENT_GENERATION.md](nextjs/API_CLIENT_GENERATION.md) - API client setup
- [VERCEL_BUILD_FIX.md](VERCEL_BUILD_FIX.md) - Original build fix documentation
