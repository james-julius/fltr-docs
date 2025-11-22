# Milvus Lite Connection Issue

## Problem
PyMilvus versions 2.3.0+ have a breaking change where `MilvusClient(uri="file_path")` no longer works for Milvus Lite. The error is:

```
ConnectionConfigException: Illegal uri: [/path/to/file.db], expected form 'http[s]://[user:password@]example.com:12345'
```

## Root Cause
The pymilvus library changed how it parses URIs, and now only accepts HTTP/HTTPS URLs, not file paths for Milvus Lite.

## Temporary Solution
We've temporarily disabled Milvus initialization in `main.py` to allow the credit system to function. This doesn't affect credit operations but will break vector search functionality.

## Permanent Solutions

### Option 1: Use milvus-lite package (Recommended)
Install the dedicated Milvus Lite package:
```bash
pip install milvus-lite
```

Then update `database/vector_store.py`:
```python
if settings.USE_MILVUS_LITE:
    from milvus import default_server
    default_server.start()
    return MilvusClient(uri="http://localhost:19530")
```

### Option 2: Pin to older pymilvus version
```bash
pip install 'pymilvus<2.3.0'
```

### Option 3: Switch to Zilliz Cloud
For production, use Zilliz Cloud instead:
```bash
USE_MILVUS_LITE=false
MILVUS_URI=https://your-instance.zillizcloud.com
MILVUS_TOKEN=your_token_here
```

## Status
- ❌ Milvus Lite currently not working
- ✅ Credit system fully functional
- ✅ All other API endpoints working
- ⚠️  Vector search endpoints will fail

## To Re-enable Milvus
1. Uncomment lines 51-53 in `fastapi/main.py`
2. Implement one of the permanent solutions above
3. Restart the FastAPI server
