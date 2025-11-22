# FLTR User-First Roadmap: Find Users, Then Build

**Document Version:** 2.0
**Last Updated:** 2025-11-09
**Status:** ACTIVE - User Discovery Phase

---

## Executive Summary

**OLD APPROACH (WRONG):**
- ❌ Spend 9 months building features
- ❌ Assume we know what users want
- ❌ Copy Ragie's feature list
- ❌ Build 50+ connectors nobody asked for

**NEW APPROACH (RIGHT):**
- ✅ Find 10 paying users in Month 1
- ✅ Build ONLY what they ask for
- ✅ Validate every feature before coding
- ✅ Iterate based on real feedback

---

## Current Reality Check

**What we have:**
- ✅ Working RAG system (upload, chunk, embed, search)
- ✅ MCP integration (unique advantage)
- ✅ Cost-optimized architecture
- ✅ Open-source

**What we DON'T have:**
- ❌ Any paying users
- ❌ User feedback
- ❌ Validated pain points
- ❌ Product-market fit

**The Problem:**
We're planning 9 months of development based on ZERO user input. This is backwards.

---

## Phase 0: User Discovery (Weeks 1-4)

**GOAL: Get 10 people to pay you $50 TODAY**

If people won't pay you today for a promise, they won't pay you in 9 months after you build everything.

### Week 1: Distribution Blitz

**Goal: Talk to 50 potential users**

**Where to find them:**

1. **Online Communities** (2 hours/day)
   - HackerNews "Who's Hiring" thread → Find companies using RAG
   - Reddit r/MachineLearning → Post "Building open-source RAG, what connector would you pay for?"
   - Reddit r/LangChain → "MCP-native RAG system - feedback wanted"
   - Twitter #BuildInPublic → Share your MCP advantage
   - LinkedIn → DM 20 AI engineers at startups

2. **Direct Outreach** (2 hours/day)
   - Email 30 companies using Ragie (check their customers page)
   - Subject: "Open-source alternative to Ragie with MCP support"
   - Message 20 people who starred similar GitHub repos

3. **Create Landing Pages** (1 day)
   ```
   fltr.dev/notion     → "Search your Notion workspace"
   fltr.dev/gdrive     → "Search Google Drive"
   fltr.dev/slack      → "Search Slack history"
   fltr.dev/enterprise → "Self-hosted RAG for enterprises"
   ```
   - Each page has "Request Early Access" button
   - Track which pages get most interest

**Deliverables:**
- [ ] 50 conversations with potential users
- [ ] 100 email addresses collected
- [ ] Data on which connectors get most interest
- [ ] List of common pain points

**Success Metric:** 20+ people express interest and provide contact info

---

### Week 2: Customer Development Interviews

**Goal: Understand the real problem**

**Interview Script:**

Call/Zoom with 20 people who expressed interest. Ask:

1. **Discovery:**
   - "What's your biggest challenge with finding information at work?"
   - "Which apps contain your most important information?"
   - "How do you currently search across multiple sources?"

2. **Validation:**
   - "Have you tried [Ragie/competitors]? Why did/didn't you use it?"
   - "What would you pay $50/month for?"
   - "If I built this tomorrow, would you use it this week?"

3. **Pricing:**
   - "How much do you currently spend on [similar tools]?"
   - "Would you pay $49/month? $99/month? $299/month?"
   - "Would your company pay for this?"

4. **Connectors:**
   - "Which 3 tools would you connect first?"
   - "Which connector would make you sign up TODAY?"

**Take Notes On:**
- ✅ Exact words they use to describe their problem
- ✅ Which tools they mention most
- ✅ Price sensitivity
- ✅ Urgency level (nice-to-have vs. must-have)
- ✅ Who the decision maker is (them vs. their boss)

**Deliverables:**
- [ ] 20 detailed interview notes
- [ ] Top 3 most-requested connectors (WITH DATA)
- [ ] Price point validation
- [ ] 5+ quotes for landing page

**Success Metric:** 5+ people say "I would pay for this today"

---

### Week 3: Pre-sell and Validate

**Goal: Get 10 people to pay $50 upfront**

**Create a "Founder's Edition" Offer:**

```
FLTR Founder's Edition - $50 one-time

What you get:
- Early access when we launch (4-6 weeks)
- [ONE CONNECTOR] - the one YOU vote for
- Lifetime 50% discount
- Direct Slack channel with founders
- Vote on roadmap priorities

Limited to first 50 users.
```

**How to sell it:**

1. **Email everyone from Week 1:**
   ```
   Subject: I talked to 20 people like you. Here's what I'm building.

   Hi [Name],

   Last week you said you'd pay for [THEIR EXACT WORDS].

   I talked to 20 people. The #1 most requested connector is [X].

   I'm building it. Launch in 4-6 weeks.

   50 spots available at $50 (normally $49/month).
   You get lifetime 50% off.

   Pay now: [Stripe checkout link]

   If I don't ship by [DATE], full refund.

   - James
   ```

2. **Post on all communities:**
   - "Building open-source RAG. Pre-selling 50 spots. Here's what people asked for..."

3. **Set up Stripe checkout:**
   ```python
   # Simple Stripe checkout
   stripe.checkout.Session.create(
       payment_method_types=['card'],
       line_items=[{
           'price_data': {
               'currency': 'usd',
               'product_data': {'name': 'FLTR Founder\'s Edition'},
               'unit_amount': 5000,  # $50
           },
           'quantity': 1,
       }],
       mode='payment',
       success_url='https://fltr.dev/success',
   )
   ```

**Deliverables:**
- [ ] Stripe checkout page live
- [ ] 100 emails sent
- [ ] 10 posts across communities
- [ ] Landing page with social proof

**Success Metric:** 10 people pay $50 = $500 revenue = VALIDATION

---

### Week 4: Analyze and Decide

**Goal: Use real data to build roadmap**

**Questions to Answer:**

1. **Did anyone pay?**
   - YES → You have validation. Build what they asked for.
   - NO → Your idea is wrong. Pivot or shut down.

2. **Which connector won?**
   - Count votes from paying customers ONLY
   - Build ONLY that connector first

3. **What's the common problem?**
   - Extract themes from interviews
   - Focus on the most painful problem

4. **What's the price point?**
   - Did people hesitate at $50?
   - Would they pay $99? $299?

**Create New Roadmap Based on Data:**

Example (if Notion wins):
```
Month 1: Build Notion connector
Month 2: Get 10 more users, improve based on feedback
Month 3: Build 2nd most-requested connector
Month 4: Scale
```

**Deliverables:**
- [ ] Data-driven roadmap (1 page)
- [ ] Clear next steps
- [ ] Email update to all 100 contacts

**Success Metric:** Clear plan based on REAL user demand, not guesses

---

## Phase 1: Build for Your 10 Users (Month 2)

**GOAL: Make your 10 paying users successful**

**DON'T:**
- ❌ Build features they didn't ask for
- ❌ Add complexity
- ❌ Try to scale yet

**DO:**
- ✅ Talk to them weekly
- ✅ Fix their bugs immediately
- ✅ Build ONLY what they need
- ✅ Make them love you

### Week 5-6: Build MVP of Top-Requested Connector

**Example: Notion Connector** (if that won)

**Minimum Viable Connector:**
```python
# What you NEED:
- [ ] OAuth with Notion
- [ ] Sync all pages
- [ ] Manual sync button
- [ ] Basic error handling

# What you DON'T need yet:
- ❌ Incremental sync
- ❌ Scheduled sync
- ❌ Selective page sync
- ❌ Database sync
- ❌ Real-time updates
```

**Ship fast, iterate faster:**
- Week 5: Build core functionality
- Week 6: Test with 3 users, fix bugs

**Deliverables:**
- [ ] Working connector deployed
- [ ] 3 users successfully syncing
- [ ] Bug list from real usage

---

### Week 7-8: Onboard All 10 Users & Get Feedback

**Goal: 8/10 users actively using the product**

**Onboarding Process:**
1. Personal Zoom call with each user
2. Watch them set it up
3. See where they struggle
4. Fix issues immediately

**Weekly Check-ins:**
- "What did you search for this week?"
- "Did you find what you needed?"
- "What's frustrating about it?"
- "What would make you use it more?"

**Measure:**
- [ ] How many searches per user per week?
- [ ] What are they searching for?
- [ ] What queries fail?
- [ ] What connector do they ask for next?

**Deliverables:**
- [ ] 8/10 users active
- [ ] List of top 10 feature requests (with vote counts)
- [ ] Clear understanding of next priority

**Success Metric:** 8/10 users search at least once per day

---

## Phase 2: Get to 50 Users (Month 3-4)

**GOAL: Grow to 50 paying users**

Only do this if Phase 1 worked (8/10 users are active).

### What to Build

**Build ONLY:**
1. The #2 most-requested connector (from your 10 users)
2. The top 3 feature requests (from your 10 users)

**DON'T build:**
- GraphRAG (unless users specifically asked for it)
- 50 connectors (unless users need them)
- Enterprise features (unless enterprises are your users)

### How to Grow

**Leverage your 10 happy users:**
1. Ask for testimonials
2. Ask for referrals ("Who else has this problem?")
3. Ask to write case studies
4. Ask to tweet about it

**Double down on what's working:**
- Which channel brought your first 10 users?
- Do more of that

**Pricing:**
- Raise prices to $99/month for new users
- Your 10 founders keep their 50% discount
- If people won't pay $99, your product isn't valuable enough yet

**Deliverables:**
- [ ] 50 paying users
- [ ] $5,000/month revenue
- [ ] Strong retention (8/10 users still active from Month 1)

---

## Decision Tree: What to Build Next

After Month 2, use this to decide:

```
                     ┌─ Are 8/10 users active?
                     │
                 YES │                    NO
                     │                     │
                     ▼                     ▼
          Build connector #2      Fix the core product
                     │                     │
                     │                     ▼
                     │           Why aren't they using it?
                     │                     │
                     ▼                     ▼
          Get to 50 users          Fix that problem first
                     │
                     ▼
    ┌─ Do 40+ users want feature X?
    │
YES │                               NO
    │                                │
    ▼                                ▼
Build it                    Don't build it
```

---

## Anti-Roadmap: Things NOT to Build

**DO NOT build these unless 10+ paying users explicitly request them:**

- ❌ GraphRAG
- ❌ Multimodal RAG
- ❌ 50+ connectors
- ❌ Enterprise SSO
- ❌ Audit logs
- ❌ Analytics dashboard
- ❌ Custom embedding providers
- ❌ Hybrid search (unless users complain about search quality)
- ❌ Advanced chunking (unless users complain about chunk quality)

**Why?**
Because you're GUESSING these are needed. Guessing kills startups.

---

## Success Metrics

### Month 1 (User Discovery)
- [ ] 50 conversations with potential users
- [ ] 10 people paid $50 upfront
- [ ] Clear understanding of top pain point
- [ ] Top 3 connectors ranked by demand

### Month 2 (Build for 10)
- [ ] 8/10 users actively using product
- [ ] 5+ searches per user per week
- [ ] <5 critical bugs
- [ ] Top 5 feature requests documented

### Month 3-4 (Scale to 50)
- [ ] 50 paying users
- [ ] $5,000/month revenue
- [ ] 80% retention from Month 1
- [ ] <2% churn rate

### Month 5+ (Product-Market Fit)
- [ ] $10,000/month revenue
- [ ] Users asking "When can I pay you more?"
- [ ] Unsolicited testimonials
- [ ] Users referring other users

---

## The Real Roadmap

```
Month 1: Find 10 paying users
         ↓
Month 2: Make them successful
         ↓
Month 3: Get to 50 users
         ↓
Month 4: Improve retention
         ↓
Month 5: Build what users ask for
         ↓
Month 6: Scale
```

Everything else is a distraction.

---

## Next Actions (This Week)

**Monday:**
- [ ] Create 3 landing pages (Notion, Google Drive, Slack)
- [ ] Set up Stripe checkout for $50 pre-sale
- [ ] Post on HN, Reddit, Twitter

**Tuesday-Thursday:**
- [ ] Email 50 potential users
- [ ] Post in 10 communities
- [ ] DM 30 people on LinkedIn/Twitter

**Friday:**
- [ ] Count: How many people paid?
- [ ] Count: How many clicked which landing page?
- [ ] Decide: Which connector to build first?

**Next Week:**
- [ ] 10 customer development calls
- [ ] Refine messaging based on feedback
- [ ] Start building top-requested connector

---

## The Hard Truth

**If you can't get 10 people to pay $50 in Month 1:**
- Your idea is wrong
- Your target market is wrong
- Your messaging is wrong
- You need to pivot

**Don't spend 9 months building something nobody wants.**

Build the minimum, get paying users, iterate based on REAL feedback.

That's how you build a successful product.

---

**END OF DOCUMENT**

Next step: Execute Week 1 of Phase 0.
