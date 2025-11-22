# Thread Management Tests

## Test Coverage Summary

Comprehensive test suite has been created for thread management functionality:

### ✅ Tests Created

1. **`src/__tests__/lib/thread-title.test.ts`** (16 tests) ✅ PASSING
   - Tests for `generateThreadTitle` utility function
   - Edge cases: empty strings, long messages, unicode, special characters
   - Sentence boundary detection
   - Word boundary truncation
   - Hard truncation fallback

2. **`src/__tests__/api/threads-route.test.ts`** (API route tests)
   - GET /api/threads - List threads with filtering
   - POST /api/threads - Create threads (general & dataset)
   - Title generation from first message
   - Validation errors
   - Authentication checks

3. **`src/__tests__/api/threads-id-route.test.ts`** (Thread CRUD tests)
   - GET /api/threads/[threadId] - Fetch single thread
   - PATCH /api/threads/[threadId] - Update thread (title, archived)
   - DELETE /api/threads/[threadId] - Delete thread
   - Authorization checks

4. **`src/__tests__/api/threads-messages-route.test.ts`** (Message tests)
   - GET /api/threads/[threadId]/messages - List messages
   - POST /api/threads/[threadId]/messages - Create messages
   - User and assistant message creation
   - Content validation
   - Thread update timestamp

5. **`src/__tests__/api/threads-share-route.test.ts`** (Share functionality tests)
   - POST /api/threads/[threadId]/share - Generate share token
   - GET /api/threads/[threadId]/share - Get existing share token
   - Share URL generation
   - Token reuse

6. **`src/__tests__/api/threads-shared-route.test.ts`** (Public shared thread tests)
   - GET /api/threads/shared/[token] - Public access
   - No authentication required
   - Message loading
   - Dataset thread support

7. **`src/__tests__/hooks/use-threads.test.ts`** (Hook API tests)
   - Hook export verification
   - URL construction
   - Request format validation
   - Error handling

## Test Status

- ✅ **thread-title.test.ts**: All 16 tests passing
- ⚠️ **API route tests**: Need schema mock fixes (mocking strategy needs adjustment)
- ⚠️ **Hook tests**: Need environment setup for client-side hooks

## Running Tests

```bash
# Run all thread-related tests
cd nextjs
pnpm test thread

# Run specific test file
pnpm test thread-title

# Run with coverage
pnpm test:coverage
```

## Test Categories

### Unit Tests
- Thread title generation utility
- URL construction
- Request/response format validation

### Integration Tests (API Routes)
- Thread CRUD operations
- Message persistence
- Share token generation
- Authentication & authorization
- Error handling

### Edge Cases Covered
- Empty inputs
- Long messages
- Unicode characters
- Missing authentication
- Invalid thread IDs
- Database errors
- Network failures

## Next Steps

1. **Fix API Route Test Mocks**: Adjust schema mocking to work with Drizzle ORM imports
2. **Add E2E Tests**: Full integration tests with real database
3. **Component Tests**: Test React components with React Testing Library
4. **Performance Tests**: Test with large thread lists and long conversations

## Test Files Created

```
nextjs/src/__tests__/
├── lib/
│   └── thread-title.test.ts ✅
├── api/
│   ├── threads-route.test.ts
│   ├── threads-id-route.test.ts
│   ├── threads-messages-route.test.ts
│   ├── threads-share-route.test.ts
│   └── threads-shared-route.test.ts
└── hooks/
    └── use-threads.test.ts
```

## Coverage Goals

- ✅ Utility functions: 100%
- 🎯 API routes: 80%+ (needs mock fixes)
- 🎯 Hooks: 70%+ (needs React context setup)
- 🎯 Components: 60%+ (future work)

