# 🚀 START HERE - FLTR Quick Test

Your dataset marketplace is ready! Here's how to see it in action in 5 minutes.

## Step 1: Seed Test Data (2 min)

```bash
cd /Users/jamesjulius/Coding/FLTR/fastapi
chmod +x seed_datasets.sh
./seed_datasets.sh
```

This creates 12 beautiful test datasets with:
- Real-looking data (SEC Filings, Court Cases, Research Papers, etc.)
- Mix of free and paid datasets
- Various categories and statuses
- Realistic stats (downloads, views, ratings)

## Step 2: Start FastAPI (30 sec)

```bash
cd /Users/jamesjulius/Coding/FLTR/fastapi
python main.py
```

Should see:
```
🚀 Starting FLTR API in development mode
✅ SQL database initialized
✅ Milvus client connected
```

> **Note**: Document processing uses Modal.com (serverless). For testing with actual file uploads, see `QUICKSTART.md` for Modal setup.

## Step 3: Start Next.js (30 sec)

In a new terminal:

```bash
cd /Users/jamesjulius/Coding/FLTR/nextjs
pnpm dev
```

## Step 4: Set OpenAI API Key (30 sec)

Create `nextjs/.env.local`:
```bash
OPENAI_API_KEY=sk-your-api-key-here
NEXT_PUBLIC_API_URL=http://localhost:8000
```

## Step 5: Test It! (2 min)

Open your browser and explore:

### 📦 Browse Datasets
**http://localhost:3000/datasets**

You should see:
- 12 beautiful dataset cards
- Search bar and category filter
- Mix of featured/verified badges
- Free and paid pricing
- Real stats (views, downloads, ratings)

Try:
- ✅ **Search**: Type "court" to find legal datasets
- ✅ **Filter**: Select "Finance" from dropdown
- ✅ **Click**: Click any card to see detail page
- ✅ **Scroll**: See all 12 datasets

### 🔐 Test Auth Integration
**http://localhost:3000/profile**

1. If not logged in, you'll be redirected to sign in
2. Create an account or sign in
3. See your user data loaded from FastAPI!

### 🤖 Try AI Chat with MCP
**http://localhost:3000/chat**

The AI can now query your datasets! Try:
- "What datasets do you have?"
- "Show me finance datasets"
- "Search SEC filings for Tesla"
- "Compare court cases and SEC filings about insider trading"

You'll see:
- Blue cards showing tool calls
- Real-time dataset searches
- AI answers with proper citations
- Multiple datasets queried simultaneously

See `MCP_QUICK_TEST.md` for full testing guide!

### 📱 Try Other Pages
- `/` - Landing page
- `/filters` - Filter datasets
- `/chat` - AI chat **← NOW WITH MCP!** 🎉
- `/blog` - Blog
- `/contact` - Contact form
- `/settings` - Account settings (requires login)
- `/billing` - Subscriptions (requires login)

## 🎨 What You'll See

### Dataset Cards Show:
- **Title & Description**: Clear, readable info
- **Category Badge**: Finance, Legal, Healthcare, etc.
- **Status Badge**: Ready, Processing, or Uploading
- **Stats Row**: 👁️ Views | ⬇️ Downloads | ⭐ Rating
- **Document Count**: How many docs in the dataset
- **Pricing**: "Free" badge or "$29/month"
- **Featured Badge**: Stars on featured datasets
- **Verified Badge**: Checkmark on verified ones

### Filters Work:
- **Search**: Real-time client-side search
- **Category**: Server-side filtering via API
- **Stats**: Shows count of filtered results

## 🐛 Troubleshooting

### No Datasets Showing?

1. **Check FastAPI is running**:
   ```bash
   curl http://localhost:8000/api/v1/datasets
   ```
   Should return JSON array

2. **Check browser console** (F12):
   Look for network errors or failed requests

3. **Verify seeding worked**:
   ```bash
   psql -U admin -h 127.0.0.1 -d fltr -c "SELECT COUNT(*) FROM dataset;"
   ```
   Should show 12

### API Errors?

Check that:
- PostgreSQL is running
- Database `fltr` exists
- Both apps use same `DATABASE_URL`
- FastAPI running on port 8000
- No CORS errors in console

## 📊 The Seeded Datasets

You'll see these 12 datasets:

### Free Datasets (7)
1. **SEC Financial Filings** - 152k docs (Finance) ⭐
2. **Medical Research Papers** - 75k docs (Healthcare) ⭐
3. **Academic CS Papers** - 198k docs (Research) ⭐
4. **Supreme Court Opinions** - 8k docs (Legal) ⭐
5. **FDA Drug Approvals** - 45k docs (Healthcare)
6. **Crypto Whitepapers** - 12k docs (Finance)
7. **Environmental Reports** - 28k docs (Research) 🔄

### Paid Datasets (5)
1. **Federal Court Cases** - 487k docs @ $29/mo (Legal)
2. **Patent Documents** - 312k docs @ $49/mo (Tech)
3. **Earnings Transcripts** - 34k docs @ $79/mo (Finance)
4. **ML Datasets Collection** - 567k docs @ $99/mo (Tech) ⭐
5. **Building Codes** - 5k docs (Research) 🔄

⭐ = Featured | 🔄 = Processing/Uploading

## 🎯 Next Steps

Once you see it working:

1. ✅ **Test different filters** - Try all categories
2. ✅ **Create an account** - Test auth flow
3. ✅ **Visit /profile** - See FastAPI integration
4. 🔨 **Implement detail page** - Click a dataset
5. 🔨 **Add create form** - Let users create datasets
6. 🔨 **Upload documents** - Add files to datasets

## 📚 Documentation

- `QUICKSTART.md` - Full setup guide
- `MCP_QUICK_TEST.md` - **Test AI chat with MCP** 🔥
- `nextjs/MCP_INTEGRATION.md` - Complete MCP guide
- `fastapi/SEEDING.md` - Seeding details
- `READY_TO_USE.md` - What's working now
- `BETTER_AUTH_INTEGRATION.md` - Auth guide
- `IMPLEMENTATION_SUMMARY.md` - Everything built

## 🎉 Success Checklist

You're successful when you can:

- ✅ See 12 datasets at `/datasets`
- ✅ Search and filter work
- ✅ Cards show all info correctly
- ✅ Can click cards (even if detail page not done)
- ✅ Create account and log in
- ✅ See profile data from FastAPI
- ✅ **NEW**: AI chat queries datasets via MCP
- ✅ **NEW**: Tool calls appear in chat
- ✅ **NEW**: AI provides cited answers from datasets

**That's it! You're fully set up with a working MCP-powered AI marketplace!** 🚀

---

_Need help? Check the docs above or inspect the code - everything is well-commented!_

