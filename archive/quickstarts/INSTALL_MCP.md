# 📦 Install MCP - Final Steps

Your MCP integration is code-complete! Just need to install one dependency and set your API key.

## Step 1: Install AI SDK (30 seconds)

```bash
cd /Users/jamesjulius/Coding/FLTR/nextjs
pnpm add ai
```

This installs the Vercel AI SDK which powers:
- Tool calling
- Streaming responses
- React hooks for chat

## Step 2: Set OpenAI API Key (1 minute)

Create or update `nextjs/.env.local`:

```bash
# OpenAI API Key (required for MCP tools)
OPENAI_API_KEY=sk-proj-your-api-key-here

# API URL (already set probably)
NEXT_PUBLIC_API_URL=http://localhost:8000
```

Get your API key from: https://platform.openai.com/api-keys

## Step 3: Start Everything (1 minute)

```bash
# Terminal 1 - FastAPI (if not already running)
cd /Users/jamesjulius/Coding/FLTR/fastapi
python main.py

# Terminal 2 - Next.js
cd /Users/jamesjulius/Coding/FLTR/nextjs
pnpm dev
```

## Step 4: Test It! (2 minutes)

Visit: **http://localhost:3000/chat**

Try these prompts:

### 1. Basic Discovery
```
What datasets do you have?
```

**Expected**: Lists all 12 datasets

### 2. Category Filter
```
Show me finance datasets
```

**Expected**: Shows SEC Filings, Earnings Calls, Crypto

### 3. Search
```
Search SEC filings for Tesla
```

**Expected**:
- Blue tool call card appears
- Shows "Calling listDatasets..."
- Shows "Calling searchDataset..."
- AI responds with relevant information

## What You Should See

### Tool Call Visualization
```
┌──────────────────────────────────┐
│ 🗄️ Used listDatasets            │
│ 🔍 Finance                       │
│ Found 3 datasets                 │
└──────────────────────────────────┘

┌──────────────────────────────────┐
│ 🗄️ Used searchDataset            │
│ 🔍 Tesla                         │
│ 📚 SEC Financial Filings         │
│ Found 5 results                  │
└──────────────────────────────────┘

AI: Based on SEC Financial Filings, I found...
```

## Troubleshooting

### "Cannot find module 'ai/react'"

**Solution**: Run `pnpm add ai` in the nextjs directory

### "OpenAI API Error: 401 Unauthorized"

**Solution**: Check your `.env.local` has correct `OPENAI_API_KEY`

### "Failed to search dataset"

**Solution**:
1. Check FastAPI is running: `curl http://localhost:8000/api/v1/mcp/datasets`
2. Verify datasets exist: `cd fastapi && ./seed_datasets.sh`

### AI responds but no tool calls

**Solution**:
1. Check OpenAI API key is set
2. Try more explicit prompts: "Use searchDataset to find..."
3. Check browser console for errors

## Files Created

✅ All code is ready!

- `nextjs/src/lib/mcp/dataset-tools.ts` - 4 MCP tools
- `nextjs/src/app/api/chat/route.ts` - Updated with tools
- `nextjs/src/app/(public)/chat/page.tsx` - Enhanced UI
- `MCP_INTEGRATION.md` - Full documentation
- `MCP_QUICK_TEST.md` - Test guide
- `MCP_COMPLETE.md` - Complete overview

## What's Next

After testing works:

1. ✅ Try different prompts (see `MCP_QUICK_TEST.md`)
2. ✅ Test error handling
3. ✅ Try complex multi-step queries
4. 🔨 Customize system prompt
5. 🔨 Add more tools
6. 🔨 Enhance UI with collapsible results

## Success Checklist

- [ ] Installed `pnpm add ai`
- [ ] Set `OPENAI_API_KEY` in `.env.local`
- [ ] FastAPI running on port 8000
- [ ] Next.js running on port 3000
- [ ] Can ask "What datasets do you have?"
- [ ] Blue tool call cards appear
- [ ] AI provides dataset-aware answers
- [ ] Can search specific datasets

## Need Help?

1. **Check docs**: `MCP_INTEGRATION.md` has full details
2. **Test guide**: `MCP_QUICK_TEST.md` has example prompts
3. **Browser console**: F12 to see errors
4. **FastAPI logs**: Check terminal for API errors

---

## 🎉 You're Almost There!

Just install the AI SDK and set your API key, then you'll have:

- ✅ Full MCP integration
- ✅ Real-time dataset search
- ✅ AI-powered semantic queries
- ✅ Multi-dataset comparison
- ✅ Beautiful tool visualization

**Two commands away from Context as a Service!** 🚀

```bash
pnpm add ai
# Add OPENAI_API_KEY to .env.local
# Visit http://localhost:3000/chat
```

