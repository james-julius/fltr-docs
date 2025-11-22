# 🚀 MCP Quick Test Guide

Test your AI chat with real dataset access in 5 minutes!

## Setup (2 minutes)

### 1. Install Missing Dependencies
```bash
cd /Users/jamesjulius/Coding/FLTR/nextjs
pnpm install ai
```

### 2. Set OpenAI API Key
Create/update `nextjs/.env.local`:
```bash
OPENAI_API_KEY=sk-your-api-key-here
NEXT_PUBLIC_API_URL=http://localhost:8000
```

### 3. Start Servers
```bash
# Terminal 1 - FastAPI
cd /Users/jamesjulius/Coding/FLTR/fastapi
python main.py

# Terminal 2 - Next.js
cd /Users/jamesjulius/Coding/FLTR/nextjs
pnpm dev
```

## Test Prompts (3 minutes)

Visit: **http://localhost:3000/chat**

### Test 1: Dataset Discovery
```
What datasets do you have?
```

**Expected**:
- AI calls `listDatasets` tool
- Shows blue card: "Calling listDatasets..."
- Lists all 12 datasets with descriptions

### Test 2: Category Filter
```
Show me finance datasets
```

**Expected**:
- AI calls `listDatasets` with `category: "Finance"`
- Shows: SEC Filings, Earnings Calls, Crypto Whitepapers

### Test 3: Dataset Info
```
Tell me about the Supreme Court Opinions dataset
```

**Expected**:
- AI calls `listDatasets` to find it
- AI calls `getDatasetInfo` with the UUID
- Shows document count, description, etc.

### Test 4: Search Dataset
```
Search SEC filings for Tesla
```

**Expected**:
- AI calls `listDatasets` to find SEC dataset
- AI calls `searchDataset` with query "Tesla"
- Shows blue card: "Calling searchDataset..."
- Shows: "Found X results"
- AI responds with relevant excerpts

### Test 5: Multi-Dataset Query
```
Compare what court cases and SEC filings say about insider trading
```

**Expected**:
- AI calls `listDatasets` to find both datasets
- AI calls `batchSearch` with both queries
- Shows results from both datasets
- AI synthesizes the information

### Test 6: Complex Workflow
```
Find me all healthcare datasets, then search the medical research one for cancer treatments
```

**Expected**:
- AI calls `listDatasets` with category "Healthcare"
- AI calls `searchDataset` on Medical Research Papers
- Shows multiple tool calls
- Provides comprehensive answer

## What You'll See

### Tool Call Card (Blue)
```
┌────────────────────────────────────┐
│ 🗄️ Used searchDataset              │
│ 🔍 Tesla                           │
│ 📚 SEC Financial Filings           │
│ Found 5 results                    │
└────────────────────────────────────┘
```

### AI Response
```
Based on SEC Financial Filings, I found information about Tesla:

1. [Relevant excerpt from filing 1...]
2. [Relevant excerpt from filing 2...]
...

Would you like more details on any of these?
```

## Troubleshooting

### "No datasets found" Error

**Problem**: Database is empty

**Fix**:
```bash
cd fastapi
./seed_datasets.sh
```

### AI Doesn't Use Tools

**Problem**: Missing OpenAI API key or wrong configuration

**Fix**:
1. Check `.env.local` has `OPENAI_API_KEY`
2. Restart Next.js server
3. Try more explicit prompts: "Use the searchDataset tool to find..."

### "Module not found 'pymilvus'" Error

**Problem**: FastAPI missing dependency

**Fix**:
```bash
cd fastapi
pip install pymilvus
```

### Tool Call Shows Error

**Problem**: FastAPI not running or dataset doesn't exist

**Fix**:
1. Check FastAPI is running: `curl http://localhost:8000/api/v1/mcp/datasets`
2. Verify datasets exist: `./seed_datasets.sh`
3. Check browser console for error details

## Success Checklist

- ✅ AI responds to questions
- ✅ Blue tool call cards appear
- ✅ Tool names and queries visible
- ✅ Results show "Found X results"
- ✅ AI provides context-aware answers
- ✅ Multiple tool calls work in sequence

## Advanced Tests

Once basic tests work, try:

### Chained Queries
```
1. What datasets have legal information?
2. Search the federal court cases for antitrust
3. Now search supreme court for the same thing
4. Compare the results
```

### Category Exploration
```
Give me an overview of all technology datasets,
then search the patents one for artificial intelligence inventions
```

### Batch Operations
```
Search both medical research and FDA approvals
for information about COVID-19 vaccines
```

## Expected Performance

- **Simple query** (list datasets): 2-3 seconds
- **Single search**: 3-5 seconds
- **Batch search**: 5-10 seconds
- **Complex multi-step**: 10-15 seconds

## What's Happening Behind the Scenes

```
User: "Search SEC filings for Tesla"
  ↓
AI thinks: "I need to find SEC dataset, then search it"
  ↓
Tool call 1: listDatasets({ category: "Finance" })
  → Returns: [SEC Filings dataset with UUID]
  ↓
Tool call 2: searchDataset({
  datasetId: "abc-123",
  query: "Tesla"
})
  → FastAPI generates embeddings
  → Milvus vector search
  → Returns relevant documents
  ↓
AI: "Based on SEC Financial Filings, I found..."
```

## Next Steps After Testing

If everything works:

1. ✅ Try different categories
2. ✅ Test error handling (search non-existent dataset)
3. ✅ Try complex multi-turn conversations
4. 🔨 Customize tool descriptions
5. 🔨 Add more tools (save results, export, etc.)
6. 🔨 Improve UI for tool results

## Need Help?

Check:
- `nextjs/MCP_INTEGRATION.md` - Full integration docs
- Browser console (F12) - Error messages
- FastAPI logs - Server-side errors
- Network tab - API call details

## Congratulations! 🎉

You now have an AI that can:
- Intelligently discover datasets
- Search using semantic understanding
- Query multiple sources simultaneously
- Provide context-aware, cited answers

**This is the core of FLTR - Context as a Service for AI!** 🚀

