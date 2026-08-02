# Stage 5: RAG — Retrieval Augmented Generation — Study Guide & Notebook

This module covers Retrieval Augmented Generation (RAG), the primary architectural pattern for grounding LLMs in external enterprise datasets.

---

## 📅 Study Checklist
- [ ] Implement token-based, character-based, and semantic chunking strategies.
- [ ] Understand vector indexing mechanisms (HNSW, IVF) and distance metrics (cosine, dot product).
- [ ] Build a hybrid search pipeline combining BM25 keyword matching and dense vector search.
- [ ] Set up a Cross-Encoder reranker to improve the relevance of retrieved chunks.
- [ ] Explain the components of Graph RAG and Agentic RAG patterns.
- [ ] Evaluate a RAG pipeline using the Ragas framework (faithfulness, answer relevance).

---

## 🏗️ The RAG Pipeline

A production-grade RAG pipeline consists of two separate phases: **Ingestion** and **Retrieval & Generation**.

```
[ INGESTION PHASE ]
Raw Documents ──> Data Cleaning ──> Chunking ──> Embeddings ──> Vector Index

[ RETRIEVAL & GENERATION PHASE ]
User Query ──> Dense Embedding ┐
               Keyword Search  ├─> Hybrid Search ──> Reranking ──> Grounded Prompt ──> LLM
```

---

## 🗂️ Chunking Strategies

A raw document (e.g., a 100-page PDF manual) cannot be passed to an embedding model all at once. It must be broken down into chunks:

1.  **Character/Token Chunking:** Splits text into fixed counts of characters or tokens, using a set overlap size (e.g., 500 characters with 50 characters overlap). If not carefully managed, this can split sentences mid-thought, losing context.
2.  **Semantic Chunking:** Analyzes text flow, computes embedding distances between adjacent sentences, and splits chunks only when a significant semantic change occurs.
3.  **Parent-Child Retriever (Hierarchical):** Stores small chunks (child chunks) in the vector database for high-precision retrieval, but links them to larger parent chunks. When a child chunk matches a query, the larger parent chunk is retrieved and passed to the LLM, preserving the surrounding context.

---

## 🔍 Hybrid Search & Reciprocal Rank Fusion (RRF)

Dense vector search is effective for conceptual queries but can fail on exact keyword matches, SKU numbers, or specific terms. Production systems use **Hybrid Search**:
1.  **Dense Retrieval:** Generates vector embeddings and performs nearest-neighbor vector search (semantic similarity).
2.  **Sparse Retrieval:** Performs keyword matching using algorithms like **BM25**.
3.  **Reciprocal Rank Fusion (RRF):** Combines the rank scores from both search runs into a single, merged ranking.
    $$\text{RRF Score}(d) = \sum_{m \in M} \frac{1}{k + r_m(d)}$$
    Where $r_m(d)$ is the rank of document $d$ in retrieval run $m$, and $k$ is a constant (usually $\approx 60$).

---

## 🎯 Reranking with Cross-Encoders

Standard vector search uses **Bi-Encoders** (where query and documents are embedded separately, and compared via cosine similarity). This is fast but has lower accuracy.
**Rerankers** use **Cross-Encoders**, passing both the query and the retrieved document together through a network to compute a direct relevance score.
*   **Workflow:** Retrieve the top 50 matches using fast vector search, then pass them through a Cross-Encoder to select the top 5 most relevant chunks to send to the LLM.

---

## 📊 RAG Evaluation with Ragas

The **Ragas** framework evaluates RAG pipelines using four core metrics:

```
                  ┌──────────────────────┐
                  │      User Query      │
                  └──────────┬───────────┘
                             │
                      Answer Relevance
                             │
                             ▼
┌────────────────┐     ┌───────────┐     ┌────────────────┐
│   Contexts     │<────┼── Faith   │<────┤   Generated    │
│  (Retrieved)   │     │  fulness  │     │    Response    │
└───────┬────────┘     └───────────┘     └────────────────┘
        │
Context Recall / Precision
        │
        ▼
┌────────────────┐
│  Ground Truth  │
└────────────────┘
```

*   **Context Recall:** Measures if all relevant information needed to answer the query was successfully retrieved.
*   **Context Precision:** Measures if the retrieved chunks were highly relevant to the query, minimizing noise.
*   **Faithfulness:** Measures if the generated response is strictly grounded in the retrieved context (detecting hallucinations).
*   **Answer Relevance:** Measures if the generated response directly addresses the user's query.

---

## ❓ Common Interview Q&As

#### Q1: What is the "Lost in the Middle" phenomenon, and how do you design around it?
**Answer:** Research shows that LLMs are best at processing information located at the very beginning or end of their input context. Information placed in the middle of long prompts is often missed. To mitigate this:
1.  Avoid sending too many retrieved chunks to the model.
2.  Use a Cross-Encoder reranker to limit retrieved chunks to the top 3–5 items.
3.  Implement context compression to extract and pass only the most relevant sentences.

#### Q2: What is the difference between HNSW and IVF vector indexes?
**Answer:** Both are Approximate Nearest Neighbor (ANN) search algorithms used to speed up vector lookups:
- **Hierarchical Navigable Small World (HNSW):** Builds a multi-layer graph index. Top layers have long-range links for fast routing, while bottom layers contain highly dense local clusters. It is fast and accurate, but requires significant memory (RAM).
- **Inverted File Index (IVF):** Clusters vector space into regions using k-means. During queries, it calculates distance only to the closest cluster centroids, searching only within those clusters. It has a smaller memory footprint than HNSW, but lookup speeds are slightly slower and search recall can be lower.
