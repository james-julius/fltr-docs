# FLTR Launch Requirements: Real User Needs

**Document Version:** 1.0
**Last Updated:** 2025-11-09
**Status:** ACTIVE - Building for Real Users

---

## Current Situation

**Real Users:**
- ✅ **1 paying client** (or committed client)
- ✅ **1 co-founder** dogfooding in existing business

**Validated Requirements:**
- ✅ **OCR** - They need to extract text from images/scans
- ✅ **Reranking** - They need better search results

This is REAL demand, not speculation.

---

## Critical Path to Launch

**Goal: Make these 2 users successful ASAP**

### Priority 1: OCR (Week 1-2)

**User Need:**
"I need to search text in scanned documents/images"

**Minimum Implementation:**

```python
# modal/services/ocr/
├── ocr_service.py       # Main OCR service
├── image_extractor.py   # Extract images from PDFs
└── text_processor.py    # Clean and process OCR output
```

**What to Build:**

1. **Image Extraction from PDFs** (2 days)
   ```python
   # Use existing RapidOCR (already in dependencies!)
   from rapidocr_onnxruntime import RapidOCR

   async def extract_and_ocr_images(pdf_path: str) -> List[Dict]:
       """Extract images from PDF and run OCR"""
       ocr = RapidOCR()

       # Extract images using pymupdf
       import fitz  # PyMuPDF
       doc = fitz.open(pdf_path)

       results = []
       for page_num in range(len(doc)):
           page = doc[page_num]
           images = page.get_images()

           for img_idx, img in enumerate(images):
               xref = img[0]
               base_image = doc.extract_image(xref)
               image_bytes = base_image["image"]

               # Run OCR
               ocr_result, elapsed = ocr(image_bytes)
               if ocr_result:
                   text = " ".join([line[1] for line in ocr_result])
                   results.append({
                       "page": page_num + 1,
                       "image_index": img_idx,
                       "text": text,
                       "confidence": ocr_result[0][2] if ocr_result else 0
                   })

       return results
   ```

2. **Integration with Document Processing** (1 day)
   ```python
   # In modal/services/document_processor.py

   async def process_document_with_ocr(object_key: str, document_id: str):
       # ... existing PDF processing ...

       # Add OCR step
       if has_images(pdf_path):
           ocr_results = await extract_and_ocr_images(pdf_path)

           # Append OCR text to document text
           for ocr_result in ocr_results:
               text_chunks.append({
                   "text": ocr_result["text"],
                   "metadata": {
                       "source": "ocr",
                       "page": ocr_result["page"],
                       "confidence": ocr_result["confidence"]
                   }
               })
   ```

3. **Testing** (1 day)
   - Test with client's actual scanned documents
   - Verify OCR quality
   - Check processing time

**Success Criteria:**
- [ ] Can extract text from scanned PDFs
- [ ] OCR text appears in search results
- [ ] Client/co-founder can find information in scanned docs
- [ ] <30 seconds processing time for typical document

**What NOT to Build:**
- ❌ GPT-4 Vision descriptions (add later if needed)
- ❌ Multiple OCR engines
- ❌ OCR language selection
- ❌ Image enhancement/preprocessing
- ❌ Handwriting recognition

---

### Priority 2: Reranking (Week 2-3)

**User Need:**
"Search results aren't relevant enough. Top results miss what I'm looking for."

**Minimum Implementation:**

```python
# fastapi/services/search/
├── reranker.py          # Reranking service
└── search_service.py    # Update with reranking
```

**What to Build:**

1. **Cross-Encoder Reranker** (2 days)
   ```python
   # fastapi/services/search/reranker.py
   from sentence_transformers import CrossEncoder

   class RerankerService:
       def __init__(self):
           # Use fast, lightweight reranker
           self.model = CrossEncoder('cross-encoder/ms-marco-MiniLM-L-6-v2')

       async def rerank(
           self,
           query: str,
           results: List[Dict],
           top_k: int = 10
       ) -> List[Dict]:
           """
           Rerank search results using cross-encoder

           1. Get top 50 results from vector search
           2. Rerank using cross-encoder
           3. Return top K
           """
           # Get scores for each result
           pairs = [[query, result["text"]] for result in results]
           scores = self.model.predict(pairs)

           # Add scores and sort
           for i, result in enumerate(results):
               result["rerank_score"] = float(scores[i])

           # Sort by rerank score
           reranked = sorted(results, key=lambda x: x["rerank_score"], reverse=True)

           return reranked[:top_k]
   ```

2. **Integration with Search** (1 day)
   ```python
   # Update MCP query endpoint

   @app.post("/mcp/query/{dataset_id}")
   async def query_dataset(
       dataset_id: str,
       query: str,
       rerank: bool = True,  # Enable by default
       limit: int = 10
   ):
       # Step 1: Vector search (get more results)
       initial_limit = 50 if rerank else limit
       vector_results = await vector_search(query, initial_limit)

       # Step 2: Rerank if enabled
       if rerank and len(vector_results) > 0:
           reranker = RerankerService()
           final_results = await reranker.rerank(
               query,
               vector_results,
               top_k=limit
           )
       else:
           final_results = vector_results[:limit]

       return final_results
   ```

3. **Performance Testing** (1 day)
   - Measure latency increase
   - Test with real queries from client/co-founder
   - Compare relevance before/after

**Success Criteria:**
- [ ] Search feels more accurate
- [ ] Client/co-founder report better results
- [ ] <500ms added latency
- [ ] Can toggle reranking on/off per query

**What NOT to Build:**
- ❌ Multiple reranking models
- ❌ Fine-tuned rerankers
- ❌ GPU acceleration (start with CPU)
- ❌ Reranking analytics/metrics
- ❌ A/B testing infrastructure

---

## Implementation Timeline

### Week 1: OCR
**Monday-Tuesday:**
- [ ] Implement image extraction from PDFs
- [ ] Integrate RapidOCR

**Wednesday-Thursday:**
- [ ] Add OCR step to document processing pipeline
- [ ] Test with client's documents

**Friday:**
- [ ] Deploy to production
- [ ] Have client test with real documents
- [ ] Fix any critical bugs

### Week 2: Reranking Setup
**Monday-Tuesday:**
- [ ] Implement cross-encoder reranking
- [ ] Add reranking service

**Wednesday-Thursday:**
- [ ] Integration with search endpoints
- [ ] Test with real queries

**Friday:**
- [ ] Deploy to production
- [ ] Get feedback from client/co-founder
- [ ] Measure improvement

### Week 3: Polish & Feedback
**Monday-Friday:**
- [ ] Fix bugs based on real usage
- [ ] Optimize performance
- [ ] Daily check-ins with users
- [ ] Document what works/doesn't work

---

## Technical Specifications

### OCR Implementation

**Library:** RapidOCR (already in dependencies)
- Fast (ONNX runtime)
- Good accuracy
- Supports multiple languages
- No API costs

**Image Extraction:** PyMuPDF (fitz)
- Extract embedded images from PDFs
- Handles most PDF formats
- Fast processing

**Storage:**
```python
# Store OCR text with metadata
chunk = {
    "text": ocr_text,
    "metadata": {
        "chunk_type": "ocr",
        "page": page_number,
        "confidence": confidence_score,
        "source": "rapidocr"
    }
}
```

**Processing Flow:**
```
PDF → Extract Pages → Find Images → OCR Each Image →
→ Combine with Document Text → Chunk → Embed → Store
```

---

### Reranking Implementation

**Model:** `cross-encoder/ms-marco-MiniLM-L-6-v2`
- Fast (6-layer model)
- Good quality
- Works on CPU
- Free (runs locally)

**Two-Stage Retrieval:**
```
Query → Vector Search (top 50) → Rerank (top 10) → Return
```

**Why Two-Stage?**
- Vector search is fast but less precise
- Reranker is slow but very precise
- Two-stage = fast + accurate

**Latency:**
- Vector search: ~50ms
- Reranking 50 results: ~300ms
- **Total: ~350ms** (acceptable)

---

## User Validation Checklist

### For Client
**Before deploying:**
- [ ] Test OCR on their actual scanned documents
- [ ] Verify search quality meets their needs
- [ ] Confirm processing time is acceptable

**After deploying:**
- [ ] Daily check-in for first week
- [ ] Ask: "Did you find what you needed today?"
- [ ] Track: What documents are they uploading?
- [ ] Track: What are they searching for?

### For Co-Founder
**Before deploying:**
- [ ] Dogfood in their business
- [ ] Test with their actual use case
- [ ] Get honest feedback

**After deploying:**
- [ ] Weekly check-in
- [ ] Ask: "What's frustrating?"
- [ ] Ask: "What would make this 10x better?"
- [ ] Track: Daily usage patterns

---

## Success Metrics

### Week 1 (OCR)
- [ ] Client successfully processes 10+ scanned documents
- [ ] OCR text is searchable
- [ ] 0 critical bugs
- [ ] Client gives positive feedback

### Week 2 (Reranking)
- [ ] Search results feel more relevant (qualitative)
- [ ] Client finds what they need in top 3 results
- [ ] <500ms query latency
- [ ] Both users give positive feedback

### Week 3 (Polish)
- [ ] Client using product daily
- [ ] Co-founder using product in their business
- [ ] <3 bugs reported
- [ ] Both users say "this is useful"

---

## What Comes Next?

**After Week 3, ask your users:**

1. **"What's the next most important thing?"**
   - Build that next
   - Don't guess

2. **"Would you pay more for X?"**
   - Validates if feature is valuable
   - Prioritizes by willingness to pay

3. **"Who else has this problem?"**
   - Get referrals
   - Validate if problem is common

**Then update this document with the NEXT real requirement.**

---

## Anti-Requirements

**DO NOT build unless client/co-founder specifically asks:**

- ❌ GPT-4 Vision for image descriptions
- ❌ Multiple OCR engines
- ❌ Hybrid search (vector + keyword)
- ❌ Advanced chunking strategies
- ❌ GraphRAG
- ❌ Data connectors
- ❌ Multiple embedding providers
- ❌ Analytics dashboard
- ❌ Audit logs
- ❌ SSO

**Why?**
You have 2 users. Build ONLY what they need.

---

## Cost Analysis

**Current:**
- Modal: ~$50/mo
- R2: ~$2/mo
- Milvus: ~$20/mo
- OpenAI embeddings: ~$20/mo
- **Total: ~$92/mo**

**After OCR + Reranking:**
- RapidOCR: $0 (runs in Modal)
- Reranker: $0 (runs in FastAPI)
- Slightly more Modal compute: +$10/mo
- **Total: ~$102/mo**

**Cost per user:** $51/month (for 2 users)

As you scale to 10 users: $10/month/user
As you scale to 100 users: $1/month/user

---

## Deployment Checklist

### Before OCR Launch
- [ ] Test on 5+ scanned documents
- [ ] Verify OCR text quality
- [ ] Check processing time
- [ ] Get client approval

### Before Reranking Launch
- [ ] Test with 20+ real queries
- [ ] Measure latency
- [ ] Compare results quality
- [ ] Get user approval

### Post-Launch (both)
- [ ] Monitor error logs
- [ ] Track processing times
- [ ] Daily user check-ins
- [ ] Fix bugs within 24 hours

---

## Next Steps (Start Monday)

**This Week:**
1. **Monday:** Start implementing image extraction + OCR
2. **Tuesday:** Finish OCR integration
3. **Wednesday:** Test with client's documents
4. **Thursday:** Deploy and get feedback
5. **Friday:** Fix any issues, start reranking

**Goal:** Have OCR working by end of week.

---

**REMEMBER:** You have 2 real users. Make them successful FIRST. Everything else is a distraction.

---

**END OF DOCUMENT**

Next: Implement OCR this week.
