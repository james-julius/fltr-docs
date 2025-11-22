# OpenRouter vs Vercel AI Gateway: Comparison for FLTR

**Date**: January 2025
**Context**: Choosing between OpenRouter and Vercel AI Gateway for chat inference in FLTR

---

## Executive Summary

Both platforms provide unified APIs to access multiple AI models. This document compares them specifically for FLTR's use case: a Next.js app on Vercel with a credit-based billing system, requiring multi-model support, cost tracking, and high reliability.

**Quick Recommendation**: **Vercel AI Gateway** for better cost efficiency, platform integration, and native AI SDK support.

---

## Feature Comparison

| Feature | OpenRouter | Vercel AI Gateway | Winner |
|---------|-----------|-------------------|--------|
| **Model Selection** | 579+ models | ~100+ models | OpenRouter |
| **Cost Markup** | 5-15% markup | 0% markup (including BYOK) | Vercel |
| **AI SDK Integration** | Compatible | Native integration | Vercel |
| **Platform Integration** | Third-party | Native Vercel integration | Vercel |
| **Spend Monitoring** | Basic dashboard | Built-in observability | Vercel |
| **Reliability** | High (established) | High (Vercel infrastructure) | Tie |
| **Fallback Support** | Manual | Automatic retries | Vercel |
| **Embeddings Support** | Yes | Yes | Tie |
| **BYOK Support** | Limited | Full support, 0% markup | Vercel |
| **Setup Complexity** | Medium | Low (if on Vercel) | Vercel |

---

## OpenRouter

### Pros ✅

1. **Massive Model Selection**
   - 579+ models available
   - Includes niche/experimental models
   - Better for users who need specific models

2. **Established Provider**
   - Been around longer
   - Proven reliability
   - Large user base

3. **Model Discovery**
   - Excellent model browsing interface
   - Model rankings and comparisons
   - Community ratings

4. **Flexible Pricing**
   - Pay-as-you-go
   - No minimum commitments
   - Good for experimentation

5. **Already Implemented**
   - Migration work already done
   - Working in production
   - Team familiar with it

### Cons ❌

1. **Cost Markup**
   - 5-15% markup on tokens
   - Reduces margins on credit system
   - BYOK still has markup

2. **Third-Party Dependency**
   - Separate platform to manage
   - Additional API key to secure
   - Less integrated with Vercel platform

3. **Limited Observability**
   - Basic dashboard
   - Less integration with Vercel logs/monitoring
   - Harder to correlate with app performance

4. **Manual Fallback Configuration**
   - Need to implement retry logic yourself
   - More code to maintain

5. **AI SDK Compatibility**
   - Works but not native
   - Requires custom configuration
   - May miss SDK optimizations

---

## Vercel AI Gateway

### Pros ✅

1. **Zero Cost Markup**
   - 0% markup on tokens
   - Including Bring Your Own Key (BYOK)
   - Better margins for credit system
   - **Cost Impact**: Save 5-15% on all inference costs

2. **Native AI SDK Integration**
   - Built for Vercel AI SDK 5
   - Already using `ai` package v5.0.68
   - Seamless integration
   - Automatic optimizations

3. **Platform Integration**
   - Native Vercel integration
   - Unified monitoring and logs
   - Better observability
   - Correlate with deployment metrics

4. **Built-in Reliability**
   - Automatic retries
   - Built-in fallback chains
   - Load balancing
   - Less code to maintain

5. **Spend Monitoring**
   - Built-in spend tracking
   - Better integration with credit system
   - Real-time usage metrics
   - Alerts and budgets

6. **Simplified Setup**
   - Single API key (Vercel)
   - No additional accounts
   - Unified billing

7. **Embeddings Support**
   - Native embeddings support
   - Could potentially replace OpenAI embeddings
   - Unified API for all AI operations

### Cons ❌

1. **Smaller Model Selection**
   - ~100+ models vs 579+
   - May not have niche models
   - Newer models may take time to add

2. **Vercel Lock-in**
   - Requires Vercel platform
   - Less flexibility if migrating away
   - Platform-specific solution

3. **Newer Service**
   - Less established than OpenRouter
   - May have growing pains
   - Fewer community resources

4. **Migration Required**
   - Need to migrate from OpenRouter
   - Testing required
   - Potential downtime during transition

---

## Cost Analysis

### Scenario: 1M tokens/month

**OpenRouter** (with 10% markup):
- Base cost: $100
- OpenRouter markup: $10 (10%)
- **Total: $110**

**Vercel AI Gateway** (0% markup):
- Base cost: $100
- Vercel markup: $0
- **Total: $100**

**Savings: $10/month (9%)**

### At Scale: 10M tokens/month

**OpenRouter**: $1,100/month
**Vercel AI Gateway**: $1,000/month
**Annual Savings: $1,200**

### Impact on Credit System

With FLTR's credit system charging users:
- **OpenRouter**: Lower margins due to markup
- **Vercel AI Gateway**: Higher margins, more competitive pricing

---

## Technical Considerations

### Migration Effort

**From OpenRouter to Vercel AI Gateway**:
- **Low effort** (1-2 hours)
- Change base URL: `https://openrouter.ai/api/v1` → `https://ai-gateway.vercel.sh/v1`
- Change API key: `OPENROUTER_API_KEY` → `AI_GATEWAY_API_KEY`
- Update model IDs (may need minor adjustments)
- Test with existing models

**Code Changes Required**:
```typescript
// Current (OpenRouter)
const openRouterProvider = createOpenAI({
    baseURL: "https://openrouter.ai/api/v1",
    apiKey: process.env.OPENROUTER_API_KEY,
})

// New (Vercel AI Gateway)
const gatewayProvider = createOpenAI({
    baseURL: "https://ai-gateway.vercel.sh/v1",
    apiKey: process.env.AI_GATEWAY_API_KEY,
})
```

### Model Availability

**Models Available on Both**:
- GPT-4o, GPT-4 Turbo, GPT-3.5 Turbo
- Claude 3.5 Sonnet, Claude 3 Haiku, Claude 3 Opus
- Gemini 1.5 Pro, Gemini 1.5 Flash
- Llama 3.1 70B, Llama 3.1 8B
- Mistral Large, Mixtral 8x7B

**OpenRouter Only** (examples):
- Some experimental models
- Niche providers
- Community models

**Vercel AI Gateway Only**:
- Some Vercel-optimized models
- Better integration with Vercel ecosystem

---

## Use Case Analysis

### For FLTR Specifically

**Current State**:
- ✅ Already on Vercel platform
- ✅ Using Vercel AI SDK 5
- ✅ Credit-based billing system
- ✅ Need cost efficiency
- ✅ Need reliability
- ✅ Need multi-model support

**Vercel AI Gateway Advantages**:
1. **Cost Efficiency**: 0% markup improves credit system margins
2. **Platform Integration**: Better observability and monitoring
3. **Native SDK**: Seamless AI SDK integration
4. **Reliability**: Built-in retries and fallbacks
5. **Simplified**: One less third-party service

**OpenRouter Advantages**:
1. **Model Selection**: More models available
2. **Already Working**: No migration needed
3. **Proven**: Established provider

---

## Recommendation

### Choose Vercel AI Gateway If:
- ✅ Cost efficiency is important (0% markup)
- ✅ You want better platform integration
- ✅ You're already on Vercel (which you are)
- ✅ You use Vercel AI SDK (which you do)
- ✅ You need better observability
- ✅ Standard models are sufficient

### Choose OpenRouter If:
- ✅ You need specific niche models
- ✅ You want maximum model selection
- ✅ Migration effort is a concern
- ✅ You prefer third-party independence

---

## Migration Plan (If Choosing Vercel AI Gateway)

### Phase 1: Setup (30 minutes)
1. Get Vercel AI Gateway API key from Vercel dashboard
2. Add `AI_GATEWAY_API_KEY` to environment variables
3. Update `turbo.json` to include new env var

### Phase 2: Code Changes (1 hour)
1. Update `instrumented-openrouter.ts` → `instrumented-gateway.ts`
2. Change base URL to `https://ai-gateway.vercel.sh/v1`
3. Update model IDs if needed
4. Update imports in chat routes

### Phase 3: Testing (1 hour)
1. Test with all model tiers (budget/standard/premium)
2. Verify streaming works
3. Verify tool calling works
4. Verify PostHog tracking works
5. Test fallback behavior

### Phase 4: Deployment (30 minutes)
1. Deploy to staging
2. Monitor for 24 hours
3. Deploy to production
4. Keep OpenRouter as backup for 1 week

**Total Time**: ~3 hours
**Risk**: Low (can rollback easily)

---

## Conclusion

For FLTR's specific use case, **Vercel AI Gateway is the better choice** because:

1. **Cost**: 0% markup saves 5-15% on all inference costs
2. **Integration**: Native Vercel platform integration
3. **SDK**: Built for Vercel AI SDK 5 (already using)
4. **Reliability**: Built-in retries and fallbacks
5. **Observability**: Better monitoring and spend tracking

The only significant advantage OpenRouter has is model selection (579+ vs ~100+), but for FLTR's use case, the standard models available on Vercel AI Gateway should be sufficient.

**Recommendation**: Migrate to Vercel AI Gateway for better cost efficiency and platform integration.

---

## References

- [Vercel AI Gateway Docs](https://vercel.com/docs/ai-gateway)
- [OpenRouter Models](https://openrouter.ai/models)
- [Vercel AI SDK](https://sdk.vercel.ai/docs)

---

**Last Updated**: January 2025
**Next Review**: After 3 months of usage (if migrated)


