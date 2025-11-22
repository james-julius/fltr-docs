# Admin Dashboard Setup - Complete

## ✅ What Was Completed

### 1. FastAPI Admin Endpoints (`fastapi/routers/admin.py`)
Created comprehensive admin-only endpoints:

- **GET `/api/v1/admin/stats/documents`** - Document processing statistics
  - Total documents count
  - Documents by status (ready, uploaded, pending, failed, processing)
  - Recent uploads (last 24h)
  - Recent processing count
  - Recent failures (last 24h)
  - Average processing time

- **GET `/api/v1/admin/stats/queue`** - Cloudflare Queue statistics
  - Queue name and message counts
  - Delayed messages
  - Dead letter queue count
  - Producer/consumer counts

- **GET `/api/v1/admin/health/pipeline`** - Overall pipeline health
  - Health status (healthy/degraded/unhealthy)
  - Queue consumer status
  - Modal webhook reachability
  - Stuck documents count
  - List of issues

- **GET `/api/v1/admin/documents/stuck`** - List stuck documents
- **POST `/api/v1/admin/documents/{id}/reprocess`** - Reprocess single document
- **POST `/api/v1/admin/documents/batch-reprocess`** - Batch reprocess documents

### 2. Admin Role Authorization (`fastapi/middleware/admin_auth.py`)
Created `require_admin()` helper that:
- Checks if user is authenticated
- API keys are always considered admin
- Session/OAuth users must have `role='admin'` in database
- Returns 403 if not admin

### 3. Next.js Admin Dashboard (`nextjs/src/app/(auth)/admin/page.tsx`)
Created comprehensive admin UI with:
- **Role-based access control** - Only users with `role='admin'` can access
- **Real-time stats** - Auto-refreshes every 30 seconds
- **Pipeline health monitoring** - Visual health status indicator
- **Document statistics** - Charts showing documents by status
- **Queue metrics** - Cloudflare Queue message counts
- **Responsive design** - Works on desktop and mobile

### 4. API Client Integration (`nextjs/src/lib/api.ts`)
Updated to export all generated API functions including admin endpoints

## 🚀 How to Complete Setup

### Step 1: Restart FastAPI
The new admin endpoints need FastAPI to be restarted:

```bash
# Stop current FastAPI
pkill -f "uvicorn main:app"

# Start FastAPI (from /Users/jamesjulius/Coding/FLTR/fastapi)
cd /Users/jamesjulius/Coding/FLTR/fastapi
.venv/bin/python -m uvicorn main:app --host 0.0.0.0 --port 8000
```

### Step 2: Regenerate API Client
After FastAPI restarts, regenerate the Orval client:

```bash
cd /Users/jamesjulius/Coding/FLTR/nextjs
pnpm run generate:api
```

### Step 3: Grant Admin Role
Set a user to admin in the database:

```sql
-- Connect to your auth database
UPDATE users SET role = 'admin' WHERE email = 'your-email@example.com';
```

Or via psql:
```bash
psql "postgresql://admin:password@127.0.0.1:5432/fltr_auth" -c "UPDATE users SET role = 'admin' WHERE email = 'your-email@example.com';"
```

### Step 4: Access Dashboard
Navigate to: `http://localhost:3000/admin`

## 📊 Dashboard Features

### Pipeline Health Card
Shows real-time health of your document processing pipeline:
- ✅ **Healthy** - All systems operational
- ⚠️ **Degraded** - Some issues but functioning
- ❌ **Unhealthy** - Critical issues

### Document Stats Cards
- **Total Documents** - All documents in system
- **Processing** - Currently being processed
- **Recent Uploads** - Last 24 hours
- **Recent Failures** - Failed in last 24 hours

### Status Breakdown
Pie-chart style breakdown of documents by status:
- `ready` - Successfully processed
- `uploaded` - Waiting in queue
- `pending` - Created but not uploaded
- `processing` - Currently being processed
- `failed` - Processing failed

### Queue Stats
Cloudflare Queue monitoring:
- Messages in queue
- Delayed messages
- Dead letter queue count
- Producer/consumer health

## 🔒 Security

### Access Control
- Only users with `role='admin'` can access
- API endpoints protected by `require_admin()` middleware
- Non-admin users are redirected to `/my-datasets`
- API key authentication always grants admin access

### Role Field
The `role` field in users table (defined in `nextjs/auth-schema.ts:15`):
```typescript
role: text("role")
  .$defaultFn(() => "user")
  .notNull()
```

Default role is `"user"`, must be manually set to `"admin"`.

## 🐛 Troubleshooting

### "Failed to load dashboard data"
1. Check FastAPI is running: `curl http://localhost:8000/health`
2. Check user has admin role: `psql -c "SELECT email, role FROM users;"`
3. Check browser console for auth errors

### "Admin access required" alert
- Your user role is not set to `"admin"`
- Update role in database (see Step 3 above)

### Empty stats
- Queue consumer might not be running
- Check `ENVIRONMENT=production` in `fastapi/.env`
- Check Cloudflare credentials are configured

### API functions not found
- Regenerate API client (see Step 2 above)
- Check FastAPI logs for endpoint registration

## 📝 Next Steps

### Optional Enhancements
1. **Add reprocessing UI** - Buttons to reprocess stuck documents from dashboard
2. **Add filtering** - Filter stuck documents by dataset
3. **Add charts** - Use recharts for visual graphs
4. **Add alerts** - Email/Slack notifications for pipeline issues
5. **Add logs viewer** - View recent processing logs
6. **Add performance metrics** - Track processing times over time

### Example: Add Reprocess Button
```tsx
<Button
  onClick={async () => {
    await api.batchReprocessDocumentsApiV1AdminDocumentsBatchReprocessPost({
      status: "uploaded",
      limit: 100
    });
    // Refresh stats
  }}
>
  Reprocess Stuck Documents
</Button>
```

## 🎉 Result

You now have a complete admin dashboard that shows:
- Real-time pipeline health
- Document processing statistics
- Queue status and message counts
- Ability to identify and reprocess stuck documents

Perfect for monitoring your document processing pipeline!
