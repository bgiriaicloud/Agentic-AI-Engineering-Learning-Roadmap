# Stage 11: Production AI / LLMOps — Study Guide & Notebook

This module covers LLMOps: deploying, optimizing, tracing, and monitoring large language models at scale.

---

## 📅 Study Checklist
- [ ] Configure and run a model server using vLLM, optimizing parameters for concurrent request limits.
- [ ] Implement speculative decoding to reduce inference latency.
- [ ] Set up LiteLLM as a proxy gateway to handle routing, prompt caching, and cost tracking.
- [ ] Build a prompt registry that version-controls templates independently from application code.
- [ ] Integrate LangSmith or Phoenix tracing into an agentic application.
- [ ] Implement semantic drift monitoring on production user prompts.

---

## ⚡ Inference Optimization Mechanics

Serving LLMs in production requires maximizing request throughput while minimizing latency (measured in time to first token - TTFT). Key optimization techniques include:

1.  **Continuous Batching:** Traditional batching waits for a set pool of requests before running inference. Continuous batching groups incoming requests dynamically at the token level, starting new prompts immediately as older generations finish.
2.  **PagedAttention:** Developed by the vLLM team, PagedAttention resolves VRAM fragmentation from storing Key-Value (KV) cache tensors. It partitions the KV cache into non-contiguous physical blocks, similar to virtual memory paging in operating systems, reducing memory waste by up to 96%.
3.  **Speculative Decoding:** Accelerates inference by using a small, cheap draft model to generate candidate tokens, which are verified by a larger target model in a single forward pass, reducing generation times for common text structures.

---

## 🎛️ Enterprise API Gateway Architecture (LiteLLM)

In production, applications should not interface directly with individual provider APIs. Instead, route requests through a unified model gateway layer (like **LiteLLM**):

```
┌─────────────────┐     ┌─────────────────────┐     ┌────────────────────────┐
│  Client App     │ ──> │    LiteLLM Proxy    │ ──> │ Active LLM API (vLLM)  │
└─────────────────┘     └──────────┬──────────┘     └────────────────────────┘
                                   │ (Fallback route)
                                   ▼
                        ┌─────────────────────┐
                        │ Backup Provider API │
                        └─────────────────────┘
```

*   **Unified Interface:** Provides a standard OpenAI-compatible API format regardless of the underlying model backend.
*   **Load Balancing & Routing:** Distributes requests across multiple API keys, deployments, and regions to prevent rate limiting.
*   **Failover & Fallbacks:** Automatically routes requests to backup providers if the primary server fails or encounters errors.
*   **Cost Tracking:** Tracks token usage and calculates costs per request, department, or user ID.

---

## 🔬 Observability: Tracing and Logging

Debugging agentic workflows requires tracking more than standard request/response logs. Developers must trace the entire execution path (trajectories):

*   **Trace Spans:** Capture individual steps within a workflow (e.g., Prompt formatted $\rightarrow$ LLM query $\rightarrow$ Parser function $\rightarrow$ Vector Database retrieval).
*   **Inputs & Outputs:** Log the raw parameters and outputs for every tool call and agent thought.
*   **Metrics Dashboard:** Monitor latency, token count, cost, error rates, and user feedback ratings in real-time.

```python
# Instrumenting an application with LangSmith tracing
import os
from google import genai

# Set environment variables to enable automatic tracing
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your-langsmith-api-key"
os.environ["LANGCHAIN_PROJECT"] = "production-agent"

# Standard client calls are now automatically traced and logged to your project
client = genai.Client()
response = client.models.generate_content(
    model='gemini-2.5-flash',
    contents="Translate 'Hello, world' to Spanish."
)
```

---

## ❓ Common Interview Q&As

#### Q1: What is semantic drift in production LLM applications, and how do you monitor for it?
**Answer:** Semantic drift occurs when the distribution of production user queries shifts significantly from the inputs the model was evaluated on. 
To monitor for drift:
1.  Generate and store vector embeddings for incoming user queries.
2.  Save these embeddings in an observability database.
3.  Periodically calculate the distance (using metrics like cosine distance or population stability index) between production query embeddings and validation set embeddings. A significant shift in distance indicates a change in user intent or query styles.

#### Q2: How does vLLM achieve higher request throughput than traditional Hugging Face Transformers serving?
**Answer:** vLLM achieves higher throughput primarily through **PagedAttention** and **Continuous Batching**. Traditional serving allocates static, contiguous memory blocks for the KV cache of each request, leading to VRAM fragmentation and limiting the number of concurrent requests. PagedAttention dynamic allocation allows vLLM to utilize up to 96% of available VRAM, doubling or tripling the number of concurrent requests the server can process.
