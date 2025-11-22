# Document Upload URL Encoding Fix

## Problem

Duplicate document entries were being created when uploading files with special characters (spaces, #, ?, etc.) in their filenames. This occurred because:

1. **Initial document creation**: Documents are created with unencoded `object_key` (e.g., `uuid/my file.pdf`)
2. **R2 storage**: When files are uploaded via the Cloudflare Worker proxy, R2 may store them with URL-encoded keys (e.g., `uuid/my%20file.pdf`)
3. **Webhook/Queue notification**: R2 sends notifications with the encoded object key
4. **Lookup failure**: The webhook handler was using exact string matching, so it couldn't find the existing document and created a duplicate

## Solution

### Changes Made

1. **Webhook Handler** (`fastapi/routers/webhooks.py`):
   - Changed from direct query `session.query(Document).filter_by(object_key=object_key)` to using `DocumentRepository.get_by_object_key()` which handles encoding variants
   - Added filename decoding when extracting from object_key: `unquote(object_key.split("/")[-1])`

2. **Queue Consumer** (`fastapi/services/queue_consumer.py`):
   - Same changes: use repository method and decode filename

3. **Repository Method** (`fastapi/repositories/document_repository.py`):
   - Already had encoding variant handling, but now it's being used consistently
   - Checks multiple candidate keys:
     - Original key (as-is)
     - URL-decoded version (`unquote`)
     - URL-encoded version (`quote`)

## Document Upload Lifecycle

```
1. User uploads file "my file.pdf"
   ↓
2. Frontend calls POST /api/v1/datasets/{id}/documents
   ↓
3. Backend creates Document record:
   - object_key: "uuid/my file.pdf" (unencoded)
   - status: "uploading"
   ↓
4. Client uploads to R2 via proxy URL
   - PUT to: proxy.com/uuid/my file.pdf
   ↓
5. Cloudflare Worker stores in R2
   - R2 may store as: "uuid/my%20file.pdf" (encoded)
   ↓
6. R2 sends webhook/queue notification
   - object.key: "uuid/my%20file.pdf" (encoded)
   ↓
7. Webhook handler:
   - Uses get_by_object_key() which checks:
     * "uuid/my%20file.pdf" (original encoded)
     * "uuid/my file.pdf" (decoded) ✓ MATCHES!
   - Updates existing document (no duplicate created)
   - Decodes filename: "my file.pdf"
   ↓
8. Document status updated: "uploading" → "uploaded" → "processing"
```

## Testing

Added comprehensive tests:

1. **Repository Tests** (`test_repositories.py`):
   - `test_get_by_object_key_url_encoded`: Find document with encoded key when stored unencoded
   - `test_get_by_object_key_url_decoded`: Find document with decoded key when stored encoded
   - `test_get_by_object_key_special_characters`: Test various special characters (spaces, #, ?, [, ], %)

2. **Webhook Tests** (`test_webhooks_router.py`):
   - `test_r2_upload_webhook_url_encoded_object_key`: Verify webhook updates existing document instead of creating duplicate
   - `test_r2_upload_webhook_special_characters_encoding`: Test various special characters
   - `test_r2_upload_webhook_filename_decoding`: Verify filename is correctly decoded

## Key Files Modified

- `fastapi/routers/webhooks.py`: Webhook handler now uses repository method
- `fastapi/services/queue_consumer.py`: Queue consumer now uses repository method
- `fastapi/repositories/document_repository.py`: Already had encoding handling (no changes needed)
- `fastapi/tests/test_repositories.py`: Added URL encoding tests
- `fastapi/tests/test_webhooks_router.py`: Added webhook URL encoding tests

## Verification

All tests pass:
- ✅ `test_get_by_object_key_url_encoded`
- ✅ `test_get_by_object_key_special_characters`
- ✅ `test_r2_upload_webhook_url_encoded_object_key`

## Impact

- **Prevents duplicate documents** when filenames contain special characters
- **Maintains backward compatibility** with existing documents
- **Handles edge cases** where documents might be stored with encoded keys
- **No breaking changes** - existing functionality continues to work

