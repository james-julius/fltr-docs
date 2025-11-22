# Feedback System Documentation

## Overview

A comprehensive feedback system has been implemented across the Next.js application, allowing users to submit feature requests, bug reports, improvements, and general feedback.

## Components

### 1. Database Schema

**Table: `feedback`**

Located in: `nextjs/src/database/schema.ts`

Fields:
- `id` (text, primary key) - Unique identifier
- `userId` (text, nullable) - User ID if authenticated
- `type` (text) - Type of feedback: `feature_request`, `bug_report`, `improvement`, or `general`
- `title` (text) - Brief summary
- `description` (text) - Detailed description
- `email` (text, nullable) - Optional email for anonymous users
- `url` (text, nullable) - Page where feedback was submitted
- `userAgent` (text, nullable) - Browser/device information
- `status` (text) - Status: `new`, `in_progress`, `resolved`, or `closed` (default: `new`)
- `priority` (text) - Priority level: `low`, `medium`, `high`, or `critical` (default: `medium`)
- `metadata` (text, nullable) - JSON string for additional context
- `createdAt` (timestamp) - Creation timestamp
- `updatedAt` (timestamp) - Last update timestamp

### 2. API Endpoint

**POST `/api/feedback`**

Located in: `nextjs/src/app/api/feedback/route.ts`

Request Body:
```json
{
  "type": "feature_request | bug_report | improvement | general",
  "title": "Brief summary",
  "description": "Detailed description",
  "email": "optional@email.com",
  "url": "https://example.com/page"
}
```

Response:
```json
{
  "success": true,
  "message": "Thank you for your feedback!",
  "id": "feedback_id"
}
```

Features:
- Supports both authenticated and anonymous submissions
- Automatically captures user ID from session if available
- Records the page URL and user agent
- Returns appropriate error messages for validation failures

### 3. UI Component

**FeedbackButton Component**

Located in: `nextjs/src/components/layout/feedback-button.tsx`

Features:
- Fixed position button in bottom-right corner
- Modal dialog with comprehensive form
- Four feedback types with icons and descriptions:
  - 🔆 Feature Request - Suggest a new feature
  - 🐛 Bug Report - Report a bug or issue
  - ✨ Improvement - Suggest an improvement
  - 💬 General Feedback - General comments
- Form fields:
  - Type selector with visual icons
  - Title (max 200 characters)
  - Description (max 2000 characters with counter)
  - Optional email field
- Form validation
- Loading states
- Toast notifications for success/error
- Automatic URL capture

### 4. Integration

The feedback button is globally available across all pages via the root layout:

Located in: `nextjs/src/app/layout.tsx`

## Database Migration

A migration file has been generated at:
```
nextjs/migrations/0003_boring_jocasta.sql
```

To apply the migration:
```bash
cd nextjs
pnpm run db:migrate
```

Or with explicit DATABASE_URL:
```bash
DATABASE_URL="postgresql://admin:password@127.0.0.1:5432/fltr_auth" pnpm run db:migrate
```

## Usage

### For Users

1. Click the "Feedback" button in the bottom-right corner of any page
2. Select the type of feedback
3. Fill in the title and description
4. Optionally provide an email for follow-up
5. Submit the feedback

### For Developers

#### Viewing Feedback

Query the database directly:
```sql
SELECT * FROM feedback ORDER BY created_at DESC;
```

Or create an admin dashboard to view and manage feedback.

#### Filtering by Type
```sql
SELECT * FROM feedback WHERE type = 'bug_report' ORDER BY created_at DESC;
```

#### Filtering by Status
```sql
SELECT * FROM feedback WHERE status = 'new' ORDER BY priority DESC;
```

#### Updating Feedback Status
```sql
UPDATE feedback
SET status = 'in_progress', updated_at = NOW()
WHERE id = 'feedback_id';
```

## Future Enhancements

Potential improvements to consider:

1. **Admin Dashboard**
   - Create a dedicated page to view and manage all feedback
   - Filter by type, status, priority
   - Search functionality
   - Bulk actions

2. **Email Notifications**
   - Notify admins of new feedback
   - Send updates to users when their feedback status changes

3. **Attachments**
   - Allow users to upload screenshots for bug reports
   - Store attachments in cloud storage

4. **Voting System**
   - Let users upvote existing feedback
   - Prioritize based on community interest

5. **Internal Notes**
   - Add admin-only notes to feedback items
   - Track discussion and decisions

6. **Analytics**
   - Dashboard showing feedback trends
   - Most common issues
   - User satisfaction metrics

## Security Considerations

- Anonymous feedback is allowed but can be disabled if spam becomes an issue
- Email validation on the frontend
- Input sanitization on the backend
- Rate limiting can be added to prevent abuse
- User agent tracking helps identify potential abuse patterns

## Dependencies

- `nanoid` - For generating unique feedback IDs
- `better-auth` - For session management
- `drizzle-orm` - Database ORM
- `sonner` - Toast notifications
- `lucide-react` - Icons
- `shadcn/ui` - UI components (Dialog, Select, Input, Label, Textarea, Button)
