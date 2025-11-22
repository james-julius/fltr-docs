# Thread Management Testing Checklist

## Setup & Prerequisites

1. ✅ Database migration generated: `migrations/0006_amused_tigra.sql`
2. ✅ Run migration: `cd nextjs && pnpm db:migrate`
3. ✅ Verify tables exist: `threads` and `messages` in `fltr_auth` database

## Core Functionality Tests

### 1. Thread Creation
- [ ] **Dataset Chat**: Start a new conversation in a dataset chat page
  - [ ] Thread is created automatically on first message
  - [ ] Thread title is auto-generated from first message
  - [ ] Thread appears in sidebar immediately
  - [ ] Thread is scoped to that dataset

- [ ] **General Chat**: Start a new conversation in general chat
  - [ ] Thread is created automatically on first message
  - [ ] Thread title is auto-generated from first message
  - [ ] Thread appears in sidebar immediately
  - [ ] Thread is not scoped to any dataset

### 2. Message Persistence
- [ ] Send a message in a thread
  - [ ] User message is saved to database
  - [ ] Assistant response is saved after streaming completes
  - [ ] Refresh page - messages persist
  - [ ] Switch threads and come back - messages persist

### 3. Thread Loading
- [ ] Load existing thread
  - [ ] Messages load correctly
  - [ ] Message order is correct (chronological)
  - [ ] Message content displays correctly (text, tool calls, etc.)

### 4. Thread Sidebar
- [ ] **List Threads**
  - [ ] Dataset chat shows only threads for that dataset
  - [ ] General chat shows only general threads
  - [ ] Threads are sorted by `updatedAt` (most recent first)
  - [ ] Last message preview shows correctly
  - [ ] Empty state shows when no threads exist

- [ ] **Search/Filter**
  - [ ] Search filters threads by title
  - [ ] Search works with partial matches
  - [ ] Clear search shows all threads again

- [ ] **Thread Selection**
  - [ ] Click thread loads it in chat interface
  - [ ] Active thread is highlighted
  - [ ] Switching threads preserves state

### 5. Thread Actions

- [ ] **Rename Thread**
  - [ ] Right-click → Rename opens dialog
  - [ ] Enter new title and save
  - [ ] Title updates in sidebar immediately
  - [ ] Title persists after refresh

- [ ] **Archive Thread**
  - [ ] Right-click → Archive
  - [ ] Thread disappears from main list
  - [ ] Thread appears in archived filter (if implemented)
  - [ ] Unarchive restores thread

- [ ] **Delete Thread**
  - [ ] Right-click → Delete
  - [ ] Confirmation dialog appears
  - [ ] Thread is deleted from database
  - [ ] All messages are cascade deleted
  - [ ] Thread disappears from sidebar

- [ ] **Share Thread**
  - [ ] Right-click → Share
  - [ ] Share token is generated
  - [ ] Share URL is displayed in dialog
  - [ ] Copy URL works
  - [ ] Shared thread is accessible via `/threads/shared/[token]`
  - [ ] Shared thread is read-only (no new messages)

### 6. New Thread Button
- [ ] Click "New Thread" button
  - [ ] Creates new empty thread
  - [ ] Switches to new thread
  - [ ] Can start typing immediately

## Edge Cases & Error Handling

### 7. Error Scenarios
- [ ] **Network Errors**
  - [ ] Thread creation fails → Error message shown, chat still works
  - [ ] Message save fails → Error logged, chat continues
  - [ ] Thread load fails → Error message shown

- [ ] **Empty States**
  - [ ] No threads → Empty state message
  - [ ] Thread with no messages → Empty state in chat
  - [ ] Empty message content → Handled gracefully

- [ ] **Concurrent Operations**
  - [ ] Create thread while another is loading
  - [ ] Delete thread while viewing it
  - [ ] Rename thread while it's active

- [ ] **Invalid Data**
  - [ ] Invalid thread ID → 404 error
  - [ ] Invalid share token → 404 error
  - [ ] Missing required fields → 400 error

### 8. Data Integrity
- [ ] **Cascade Deletes**
  - [ ] Delete thread → All messages deleted
  - [ ] Delete user → All threads deleted (if implemented)

- [ ] **Foreign Keys**
  - [ ] Thread references valid user
  - [ ] Message references valid thread
  - [ ] Dataset ID references valid dataset (no FK constraint, but should validate)

### 9. Performance
- [ ] **Large Thread Lists**
  - [ ] 50+ threads load efficiently
  - [ ] Pagination works (if implemented)
  - [ ] Search is responsive

- [ ] **Long Conversations**
  - [ ] Thread with 100+ messages loads correctly
  - [ ] Scrolling works smoothly
  - [ ] Message rendering is performant

## UI/UX Tests

### 10. Visual States
- [ ] **Loading States**
  - [ ] Thread list shows loading spinner
  - [ ] Thread messages show loading state
  - [ ] Creating thread shows feedback

- [ ] **Active States**
  - [ ] Active thread is visually distinct
  - [ ] Hover states work on thread items
  - [ ] Context menu appears on right-click

- [ ] **Responsive Design**
  - [ ] Sidebar works on mobile (if applicable)
  - [ ] Thread list is scrollable
  - [ ] Dialogs are properly sized

## Integration Tests

### 11. Chat API Integration
- [ ] **Dataset Chat API**
  - [ ] Thread ID is passed in request body
  - [ ] Messages are saved correctly
  - [ ] Tool calls are preserved

- [ ] **General Chat API**
  - [ ] Thread ID is passed in request body
  - [ ] Messages are saved correctly
  - [ ] Tool calls are preserved

### 12. Authentication
- [ ] **Protected Routes**
  - [ ] Unauthenticated users cannot access threads
  - [ ] Users can only see their own threads
  - [ ] Shared threads are publicly accessible

## Database Verification

### 13. Schema Validation
```sql
-- Verify tables exist
SELECT * FROM information_schema.tables
WHERE table_schema = 'public'
AND table_name IN ('threads', 'messages');

-- Verify columns
\d threads
\d messages

-- Verify indexes
\d+ threads
\d+ messages

-- Test cascade delete
INSERT INTO threads (id, user_id, title, type, created_at, updated_at)
VALUES ('test-thread', 'test-user', 'Test', 'general', NOW(), NOW());

INSERT INTO messages (id, thread_id, role, content, created_at)
VALUES ('test-msg', 'test-thread', 'user', '[]'::jsonb, NOW());

DELETE FROM threads WHERE id = 'test-thread';
-- Verify messages are also deleted
```

## Known Issues & Limitations

1. **No Real-time Updates**: Thread sidebar doesn't update automatically when another tab creates a thread
2. **No Pagination**: Thread list shows all threads (limited to 50 in API)
3. **No Unarchive**: Archive functionality exists but unarchive UI may not be implemented
4. **Share Token Security**: Share tokens are long but not expiring (consider adding expiration)

## Migration Checklist

- [ ] Run migration in development: `pnpm db:migrate`
- [ ] Verify migration success
- [ ] Test all functionality in development
- [ ] Deploy to staging
- [ ] Run migration in staging
- [ ] Test in staging
- [ ] Deploy to production
- [ ] Run migration in production
- [ ] Monitor for errors

## Rollback Plan

If issues are found:

1. **Database Rollback**:
   ```sql
   DROP TABLE IF EXISTS messages CASCADE;
   DROP TABLE IF EXISTS threads CASCADE;
   ```

2. **Code Rollback**: Revert to previous commit

3. **Data Backup**: Ensure backups exist before migration

