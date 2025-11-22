# FLTR MCP Server Setup Guide

Official Model Context Protocol (MCP) server for FLTR datasets marketplace.

## Overview

The FLTR MCP server provides AI clients (like Claude Desktop) with direct access to query datasets using the official MCP protocol. It implements:

- ✅ **Official MCP Protocol** (JSON-RPC 2.0)
- ✅ **stdio Transport** (primary) - for AI clients
- ✅ **HTTP SSE Transport** (optional) - for web integrations
- ✅ **Vector Search Integration** - semantic search via Milvus
- ✅ **Credit System** - usage tracking (planned)
- ✅ **Four MCP Tools** - search, list, batch query, get info

## Architecture

```
MCP Client (Claude Desktop, etc.)
    ↓ stdio/JSON-RPC
MCP Server (mcp_server.py)
    ↓ Python SDK
Database Services (SQLModel, Milvus)
    ↓
FLTR Datasets
```

## Installation

### Prerequisites

1. **Python 3.10+** installed
2. **PostgreSQL** database running
3. **Milvus** vector database running
4. **FLTR FastAPI** backend configured

### Install MCP SDK

```bash
cd /Users/jamesjulius/Coding/FLTR/fastapi
pip install mcp>=1.0.0
```

Dependencies are automatically installed:
- `mcp` - Official MCP SDK
- `anyio` - Async I/O support
- `httpx-sse` - Server-Sent Events
- `pydantic` - Data validation

## Configuration

### For Claude Desktop

1. **Find your Claude Desktop config:**
   - macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
   - Windows: `%APPDATA%\Claude\claude_desktop_config.json`
   - Linux: `~/.config/Claude/claude_desktop_config.json`

2. **Add FLTR MCP server:**

```json
{
  "mcpServers": {
    "fltr-datasets": {
      "command": "python",
      "args": [
        "/absolute/path/to/FLTR/fastapi/mcp_server.py",
        "--transport",
        "stdio"
      ],
      "env": {
        "DATABASE_URL": "postgresql://admin:password@localhost:5432/fltr_auth",
        "OPENAI_API_KEY": "your-openai-api-key-here"
      }
    }
  }
}
```

3. **Update paths:**
   - Replace `/absolute/path/to/FLTR/` with your actual FLTR directory
   - Update `DATABASE_URL` with your PostgreSQL connection string
   - Set `OPENAI_API_KEY` for embedding generation

4. **Restart Claude Desktop**

## Usage

### Available Tools

Once configured, Claude Desktop will automatically discover these tools:

#### 1. `search_dataset`
Search within a specific dataset using natural language.

**Parameters:**
- `dataset_id` (string, required) - UUID of dataset
- `query` (string, required) - Search query
- `limit` (integer, optional) - Results to return (1-20, default: 5)

**Example:**
```
Search dataset abc-123 for "What are the key risk factors?" with limit 10
```

#### 2. `list_datasets`
Discover available datasets in FLTR marketplace.

**Parameters:**
- `category` (string, optional) - Filter by category (Finance, Legal, Healthcare, Technology, Research)

**Example:**
```
List all Finance datasets
```

#### 3. `get_dataset_info`
Get detailed information about a specific dataset.

**Parameters:**
- `dataset_id` (string, required) - UUID of dataset

**Example:**
```
Get info for dataset abc-123
```

#### 4. `batch_search_datasets`
Search multiple datasets simultaneously.

**Parameters:**
- `queries` (array, required) - Array of queries with dataset_id, query, and limit

**Example:**
```
Batch search datasets:
- Dataset abc-123 for "revenue growth"
- Dataset def-456 for "market share"
```

## Testing

### Test from Command Line

```bash
# Test stdio transport (how Claude Desktop connects)
cd /Users/jamesjulius/Coding/FLTR/fastapi
python mcp_server.py --transport stdio

# Test HTTP SSE transport (for web integrations)
python mcp_server.py --transport sse --host 0.0.0.0 --port 8001
```

### Test with MCP Inspector

Use the official MCP Inspector tool:

```bash
npx @modelcontextprotocol/inspector python mcp_server.py
```

This opens a web UI to test tool calls interactively.

### Test Tool Calls

Example JSON-RPC request (stdio):

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "search_dataset",
    "arguments": {
      "dataset_id": "abc-123-def-456",
      "query": "What are the key risk factors?",
      "limit": 5
    }
  }
}
```

## Comparison: MCP Server vs REST API

### MCP Server (`mcp_server.py`)
- **Protocol:** JSON-RPC 2.0 over stdio
- **Clients:** AI tools (Claude Desktop, Cursor, etc.)
- **Transport:** stdio (primary), HTTP SSE (optional)
- **Auth:** Not required (runs locally)
- **Use Case:** Direct AI assistant integration

### REST API (`routers/mcp.py`)
- **Protocol:** HTTP/REST with JSON
- **Clients:** Web browsers, mobile apps
- **Transport:** HTTP only
- **Auth:** OAuth 2.1 with scopes
- **Use Case:** Web applications, API integrations

**Both interfaces share:**
- Same database (PostgreSQL + Milvus)
- Same services (EmbeddingService, vector search)
- Same dataset access

## Authentication & Credits

### Current Status

**MCP Server (stdio):**
- ⚠️ No authentication required
- ⚠️ No credit deduction (runs locally)
- ✅ Direct database access

**Rationale:** stdio transport runs locally as a subprocess of the AI client. Authentication happens at the OS/filesystem level (only the user who installed it can run it).

### Future: HTTP SSE with OAuth

For production HTTP SSE deployments, add OAuth token verification:

```python
# In mcp_server.py, add:
from middleware.oauth_scopes import verify_token

@app.call_tool()
async def call_tool(name: str, arguments: dict, context: dict) -> list[TextContent]:
    # Extract and verify OAuth token from context
    token = context.get("oauth_token")
    user_id = await verify_token(token, required_scope="mcp:query")

    # Deduct credits
    from middleware.credit_gate import deduct_credits
    await deduct_credits(user_id, amount=1, operation="MCP_QUERY")

    # ... rest of tool logic
```

## Troubleshooting

### "Database connection failed"
- Check `DATABASE_URL` in config
- Ensure PostgreSQL is running
- Test connection: `psql $DATABASE_URL`

### "Milvus connection error"
- Start Milvus: `docker-compose up -d milvus`
- Check Milvus health: `curl http://localhost:9091/healthz`

### "No embedding API key"
- Set `OPENAI_API_KEY` in environment
- Or use local embeddings (update `EmbeddingService`)

### "Tool not found in Claude"
- Restart Claude Desktop completely
- Check config file syntax (valid JSON)
- Check absolute paths in config
- View logs: `~/Library/Logs/Claude/`

### "Python not found"
- Use full path to Python: `/usr/bin/python3`
- Or use virtual environment: `/path/to/venv/bin/python`

## Development

### Project Structure

```
fastapi/
├── mcp_server.py              # Official MCP server (NEW)
├── routers/
│   ├── mcp.py                 # REST API endpoints (existing)
│   └── mcp_auth_metadata.py   # OAuth discovery (existing)
├── services/
│   └── embedding_service.py   # Shared embedding service
├── database/
│   ├── sql_store.py          # PostgreSQL
│   └── vector_store.py       # Milvus
└── mcp_server_config.json    # Example config
```

### Adding New Tools

1. **Define tool in `list_tools()`:**

```python
Tool(
    name="my_new_tool",
    description="What this tool does",
    inputSchema={
        "type": "object",
        "properties": {
            "param1": {"type": "string", "description": "..."}
        },
        "required": ["param1"]
    }
)
```

2. **Implement handler in `call_tool()`:**

```python
elif name == "my_new_tool":
    return await handle_my_new_tool(arguments)
```

3. **Add handler function:**

```python
async def handle_my_new_tool(args: dict) -> list[TextContent]:
    # Your logic here
    return [TextContent(type="text", text="Result")]
```

## Next Steps

- [ ] Add OAuth authentication for HTTP SSE transport
- [ ] Implement credit deduction for tool calls
- [ ] Add rate limiting for production deployments
- [ ] Create `.mcpb` package for one-click install
- [ ] Publish to MCP directory
- [ ] Add telemetry and usage analytics

## Resources

- **MCP Specification:** https://modelcontextprotocol.io
- **MCP Python SDK:** https://github.com/modelcontextprotocol/python-sdk
- **FLTR Documentation:** See `MCP_DEPLOYMENT_PLAN.md`
- **FLTR OAuth Guide:** See `fastapi/MCP_OAUTH_AUTHORIZATION.md`

## Support

For issues with:
- **MCP Server:** Check this guide and server logs
- **REST API:** See `nextjs/MCP_INTEGRATION.md`
- **Dataset Upload:** See main FLTR documentation
- **Claude Desktop:** Check Claude's MCP documentation

---

**Status:** ✅ Official MCP Protocol Implementation Complete

Built with the official MCP Python SDK for guaranteed spec compliance.
