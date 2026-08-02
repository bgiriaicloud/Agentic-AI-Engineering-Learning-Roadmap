# Stage 13: AI FinOps — Study Guide & Notebook

This module covers AI FinOps: managing, tracking, and optimizing costs in production AI systems.

---

## 📅 Study Checklist
- [ ] Implement exact and semantic caching layers using Redis and GPTCache.
- [ ] Configure dynamic model routing based on query complexity to reduce token costs.
- [ ] Differentiate between Spot and On-Demand GPU hosting costs.
- [ ] Set up scale-to-zero configurations for serverless container workloads.
- [ ] Optimize vector database configurations using scalar or binary quantization.
- [ ] Use provider Batch APIs to run offline data processing tasks at a discount.

---

## 💾 Semantic Caching with Redis

In production, users frequently ask identical or semantically similar questions (e.g., *"How do I reset my password?"* and *"How can I change my password?"*).
Using standard caching (matching exact string keys) fails on these variations. **Semantic Caching** embeds the incoming query and performs a vector search in a cache database first:

```
User Query ──> [ Embedding Model ] ──> Vector Embedding
                                             │
                                             ▼
                                  [ Semantic Cache DB ]
                                  Does similarity score exceed threshold? (e.g., >= 0.96)
                                  /                     \
                             Yes /                       \ No
                                ▼                         ▼
                     [ Return Cached Output ]      [ Query Target LLM ]
                                                          │
                                                          ▼
                                                  [ Update Cache DB ]
```

### Benefits of Semantic Caching:
*   **Zero LLM cost:** Returning cached responses avoids paying for LLM tokens.
*   **Reduced Latency:** Cache lookups take milliseconds compared to seconds for model generation.

---

## 🔀 Dynamic Model Routing

Not every query requires a large, expensive model (e.g., Claude 3.5 Sonnet or GPT-4o). Simple tasks can be handled by smaller, cheaper models (e.g., Llama 3 8B or Gemini 2.5 Flash).

```python
# Conceptual example of a model router
def route_query_by_complexity(query: str) -> str:
    # 1. Simple heuristic: classify query length, keywords, or structure
    # Alternatively, use a small classification model to analyze query intent
    words = query.split()
    
    if len(words) < 5 and any(kw in query.lower() for kw in ["hello", "thanks", "help", "hi"]):
        # Very simple routing
        return "gemini-2.5-flash"
    
    if "evaluate code" in query.lower() or "optimize SQL" in query.lower():
        # High complexity routing
        return "claude-3-5-sonnet"
        
    return "meta-llama-3-8b"
```

---

## 🗜️ Optimizing Vector Database Costs

Vector databases consume significant memory (RAM) to keep vector indexes (like HNSW) loaded for fast queries. To reduce costs:
1.  **Scalar Quantization (SQ):** Converts 32-bit floating-point numbers (FP32) in embeddings to 8-bit integers (INT8). This reduces memory consumption by 75% with a minimal drop in search accuracy.
2.  **Binary Quantization (BQ):** Compresses vectors to binary values (1 or 0 depending on if the float value is positive or negative). This reduces memory consumption by up to 95%, though it requires a hybrid search approach to maintain accuracy.

---

## ❓ Common Interview Q&As

#### Q1: How does a semantic cache invalidation strategy work?
**Answer:** Invalidation strategies for semantic caches are more complex than standard caches:
1.  **Time-To-Live (TTL):** Set expiration dates on cache keys to automatically clear old entries.
2.  **Explicit Invalidation:** When database values change, clear all keys whose vector embeddings are close to the updated data (using vector similarity lookups to identify relevant cache entries).

#### Q2: What are the primary trade-offs between On-Demand GPU hosting and GPU Spot Instances?
**Answer:**
- **On-Demand Instances:** Guarantee availability. Best for real-time APIs where system outages are unacceptable.
- **Spot Instances:** Offer discounts of up to 60–90% compared to on-demand rates. However, the cloud provider can reclaim the hardware at any time with minimal warning. Spot instances are best for offline batch processing or training workflows that support checkpoints and can resume automatically when new nodes are provisioned.
