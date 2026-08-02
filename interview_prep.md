# Enterprise Interview Preparation Guide 2026 — AI & Agentic AI Roles

This guide provides targeted preparation material, deep-dive system design scenarios, coding questions, and architectural mock interviews for the four core profiles in modern AI engineering:
1.  **AI Engineer** (Focus: Implementation, coding, fine-tuning, RAG pipelines).
2.  **GenAI Architect** (Focus: Systems design, scalability, LLMOps, cost-performance tradeoffs).
3.  **Agentic AI Architect** (Focus: Multi-agent coordination, stateful graphs, protocols, tool security).
4.  **Forward Deployed Engineer** (Focus: Enterprise integration, customer-facing delivery, legacy connections).

---

## 🎯 Role 1: AI Engineer (Implementation & Coding)

AI Engineers are responsible for writing production code, connecting models to APIs, fine-tuning open-weights models, building RAG indexing pipelines, and writing evaluations.

### ❓ Top 10 Interview Questions & Answers

#### Q1: Write an asynchronous python function that takes a query, calls an embedding model, retrieves the top 3 matching chunks from a PGVector database, and formats a grounded prompt for the LLM.
**Answer:**
```python
import asyncio
from typing import List, Dict
import psycopg2
from google import genai
from google.genai import types

# Shared client
client = genai.Client()

async def get_embedding(text: str) -> List[float]:
    # Mocking embedding generation asynchronously
    response = client.models.embed_content(
        model="text-embedding-004",
        contents=text
    )
    return response.embeddings[0].values

async def query_vector_db(query_vector: List[float], limit: int = 3) -> List[Dict[str, str]]:
    # In production, use async pg drivers like asyncpg
    # Placeholder showing SQL query for PGVector matching
    conn = psycopg2.connect("dbname=ai_db user=admin password=secret host=localhost")
    cur = conn.cursor()
    # <=> represents cosine distance operator in PGVector
    cur.execute(
        "SELECT content, file_name FROM document_chunks ORDER BY embedding <=> %s::vector LIMIT %s;",
        (query_vector, limit)
    )
    rows = cur.fetchall()
    cur.close()
    conn.close()
    return [{"content": r[0], "source": r[1]} for r in rows]

async def generate_grounded_answer(user_query: str) -> str:
    # 1. Embed query
    query_vector = await get_embedding(user_query)
    
    # 2. Retrieve chunks
    chunks = await query_vector_db(query_vector, limit=3)
    
    # 3. Format context
    context = "\n\n".join([f"Source: {c['source']}\nContent: {c['content']}" for c in chunks])
    
    prompt = f"""
    You are an assistant. Answer the query based strictly on the provided context below.
    If you do not know the answer from the context, state that you cannot find the answer.
    
    Context:
    {context}
    
    Query: {user_query}
    Answer:
    """
    
    # 4. Generate response
    response = client.models.generate_content(
        model="gemini-2.5-flash",
        contents=prompt
    )
    return response.text
```

#### Q2: How do you address the vanishing/exploding gradient problem in deep neural networks?
**Answer:**
*   **Activation Functions:** Swap saturating activations (sigmoid/tanh) for non-saturating ones like **ReLU** or **GeLU**, which do not compress inputs into a narrow range.
*   **Normalization:** Add Layer Normalization (applied in Transformers) or Batch Normalization to scale inputs across layers.
*   **Residual Connections:** Add identity skip connections ($x + f(x)$) to allow gradients to flow directly back through layers during backpropagation.
*   **Gradient Clipping:** Clip gradients to a maximum threshold value to prevent Exploding Gradients during training updates.

#### Q3: Explain the difference between BPE and WordPiece tokenizers.
**Answer:**
*   **Byte-Pair Encoding (BPE):** Starts with individual characters and iteratively merges the most *frequent* adjacent token pairs in the corpus.
*   **WordPiece:** Similar to BPE, but instead of merging by frequency, it selects merges that maximize the *likelihood* of the training data according to a unigram language model (maximizing the probability of token $A$ and $B$ being merged relative to their individual occurrences).

#### Q4: Why is Cosine Similarity generally preferred over Euclidean Distance for measuring document similarity?
**Answer:** Cosine similarity measures the angle between two vectors, ignoring their magnitudes. In document search, a long document and a short document might cover the same topics (similar vector directions), but the longer document will have a higher word count, resulting in a much larger vector magnitude. Euclidean distance would rank them as dissimilar because of this length difference, whereas cosine similarity registers them as highly similar.

#### Q5: Walk through the hyperparameter choices you would configure for a QLoRA fine-tuning script.
**Answer:**
*   **Rank ($r$):** The dimension of the low-rank updates (typically 8 or 16; higher ranks capture more complexity but consume more VRAM).
*   **Alpha ($\alpha$):** A scaling factor for the adapter weights (usually set to $2 \times r$).
*   **Target Modules:** Specifying which weights to adapt (e.g., `q_proj`, `v_proj`, `k_proj`, `o_proj`, `gate_proj`).
*   **Quantization:** Load the base model in 4-bit NormalFloat (NF4) with Double Quantization enabled.
*   **LoRA Dropout:** Set between 0.05 to 0.1 to prevent overfitting.

#### Q6: How do you evaluate a text embedding model for a specific enterprise domain?
**Answer:** Create a domain-specific evaluation dataset containing pairs of search queries and ground-truth documents. Use evaluation metrics like:
*   **Recall@K:** Does the correct document appear in the top $K$ search results?
*   **MRR (Mean Reciprocal Rank):** Evaluates how close the most relevant document is to the top search result.
*   **NDCG (Normalized Discounted Cumulative Gain):** Evaluates retrieval quality by factoring in the relevance order of search results.

#### Q7: What is the purpose of the scalar $\sqrt{d_k}$ in the scaled dot-product attention calculation?
**Answer:** For large embedding dimensions ($d_k$), the dot products grow large in magnitude. This pushes the subsequent `softmax` function into regions with extremely small gradients (the vanishing gradient problem). Dividing the dot product by $\sqrt{d_k}$ scales the values down, maintaining numerical stability during training.

#### Q8: How would you debug an LLM pipeline that is failing validation checks in Pydantic?
**Answer:**
1.  Lower the model's `temperature` to 0.0 to ensure deterministic formatting.
2.  Refine field descriptions in your Pydantic schema to clearly explain constraints to the model.
3.  Inject examples (few-shot prompting) of correctly formatted JSON schemas directly into the prompt.
4.  Implement a retry loop that passes the validation error message back to the LLM, prompting it to correct its output format.

#### Q9: What is the KV Cache, and how does it optimize inference latency?
**Answer:** During autoregressive generation, the model calculates Key and Value tensors for all previous tokens in the prompt at every generation step. Recomputing these tensors for every new token is computationally wasteful. The **KV Cache** stores these calculated tensors in GPU memory, allowing the model to compute Q, K, and V values only for the newly generated token at each step, reducing computation time.

#### Q10: How do you identify if a fine-tuned model has overfit on its training dataset?
**Answer:** Track loss metrics during training. If the training loss continues to fall while the validation loss begins to rise or plateau, the model is overfitting. Validate model checkpoint performance against a separate benchmark dataset to ensure it generalizes well to unseen data.

---

## 🏛️ Role 2: GenAI Architect (Systems Design & LLMOps)

GenAI Architects focus on system scalability, prompt management, model serving configurations, hybrid search architectures, caching strategies, cost tracking, and security perimeters.

### 🏛️ Mock System Design Scenario

#### The Challenge: Design a real-time, enterprise-grade Q&A system for a global company with 50,000 employees. The system must query 1,000,000 internal documents, maintain sub-second response times, ensure data privacy, and keep API token costs to a minimum.

#### The Architectural Solution:
```
                                     [ Global HTTPS Load Balancer ]
                                                   │
                                                   ▼
                                        [ API Gateway (LiteLLM) ]
                                        - Rate limiting
                                        - Dynamic cost tracking
                                                   │
                            ┌──────────────────────┴──────────────────────┐
                            ▼                                             ▼
                 [ Semantic Cache (Redis) ]                    [ Security Guardrails ]
                 - Cosine Similarity check                     - Llama Guard input scan
                            │                                             │
                            ▼ (Cache Miss)                                ▼
                 [ Query Router Service ]                      [ Hybrid Search (Qdrant) ]
                 - Route simple queries to Llama 3 8B          - BM25 text match + Vector
                 - Route complex queries to Claude 3.5         - Reciprocal Rank Fusion
                            │                                             │
                            ▼                                             ▼
                 [ vLLM Serving (GKE) ] ◄── [ Rerank (Cohere) ] ◄─────────┘
                 - BF16 precision
                 - PagedAttention Enabled
```

*   **Ingestion Pipeline:** Parse documents using serverless workers, apply semantic chunking, generate embeddings, and store them in a Qdrant cluster on GKE using HNSW indexing.
*   **Retrieval Pipeline:** Run a hybrid search query (combining Qdrant vector search and BM25 text match) and pass the results to a Cohere Reranker to filter context down to the top 5 chunks.
*   **Serving Layer:** Deploy vLLM on a GKE cluster with NVIDIA L4 GPUs to serve open-weight models, using LiteLLM in front as an API Gateway proxy.
*   **Cost & Latency Optimization:** Implement semantic caching using Redis to resolve common questions without calling the LLM. Implement dynamic query routing to send simple questions to cheaper models.
*   **Security:** Intercept inputs and outputs using Llama Guard, redact PII, and write all transactions to an immutable logging database for compliance audits.

### ❓ Top 10 Architect Questions & Answers

#### Q1: What metrics determine the performance of an inference serving architecture?
**Answer:**
*   **Time to First Token (TTFT):** The latency from sending the request to receiving the first token (affects perceived user responsiveness).
*   **Inter-Token Latency (ITL):** The average time to generate subsequent tokens (determines reading flow smoothness).
*   **Throughput:** Total tokens generated per second across all concurrent active sessions.
*   **GPU VRAM Efficiency:** Minimizing memory fragmentation to maximize the number of concurrent request batches.

#### Q2: How do you design a high-availability vector search database architecture for 10M embeddings with sub-second retrieval?
**Answer:**
*   **Indexing:** Use HNSW indexing for fast graph-based lookups.
*   **Quantization:** Apply scalar quantization (INT8) to reduce the memory footprint of vectors, keeping the index loaded in RAM.
*   **Horizontal Scaling:** Set up database replica nodes (read replicas) to handle search traffic, and partition the index across multiple write nodes (sharding) by document category or user workspace.
*   **Caching:** Place a caching layer in front of the vector database to store common queries.

#### Q3: Contrast vLLM, Hugging Face TGI, and Triton Inference Server.
**Answer:**
*   **vLLM:** Optimized for high throughput and memory management using PagedAttention. Best for serving standard open-weight transformer models.
*   **TGI (Text Generation Inference):** Developed by Hugging Face. Feature-rich, supporting speculative decoding and watermarking. Optimized for production LLM deployments.
*   **Triton Inference Server:** Multi-framework engine developed by NVIDIA. Can serve non-transformer models (e.g., CNNs, traditional ML) alongside LLMs. Best for mixed model architecture workloads.

#### Q4: How do you handle document access control (ACLs) in a production RAG pipeline?
**Answer:** Do not rely on the LLM to enforce data permissions. The document metadata must store access control lists (user groups, workspace IDs, and permission levels). When a user runs a query, pass their authentication token to the API gateway, decode their permission groups, and apply these groups as metadata filter rules in the vector database query. The database will only return documents the user is authorized to read.

#### Q5: What is Speculative Decoding, and what are its architectural requirements?
**Answer:** Speculative decoding is an optimization technique used to speed up generation times:
*   A small, cheap **draft model** (e.g., Llama 8B) generates a sequence of candidate tokens quickly.
*   A larger, expensive **target model** (e.g., Llama 70B) evaluates the draft sequence in a single forward pass, accepting or rejecting tokens based on its probability distribution.
*   **Architectural Requirement:** The draft and target models must share the exact same tokenizer vocabulary, and both must fit within active GPU memory.

#### Q6: How do you implement semantic drift detection on production user queries?
**Answer:** Use an observability worker to capture and embed incoming user queries. Store these embeddings in an analysis database, and run periodic checks calculating the cosine distance or Population Stability Index (PSI) between the production query embeddings and your validation benchmark set. If the similarity score falls below a set threshold, it indicates that user queries have drifted, triggering validation tests on your models and prompts.

#### Q7: Describe how you would manage prompt versions across multiple development environments.
**Answer:** Never store prompt templates inside application code. Use a centralized **Prompt Registry** (like LangSmith or Langfuse). Applications pull prompts dynamically using a unique ID and version tag (e.g., `prompt_customer_reply:production`). Prompts are updated and tested in staging environments, and promoted to production tags independently of application code deployments.

#### Q8: How do you mitigate rate-limiting errors when using commercial LLM APIs at scale?
**Answer:**
1.  Deploy a model gateway proxy (like LiteLLM) to load-balance requests across multiple API keys, accounts, and cloud providers.
2.  Implement an asynchronous queue (e.g., Celery/Redis) to buffer batch requests.
3.  Configure client-side retries with exponential backoff and randomized jitter to prevent API storms.

#### Q9: What are the latency and security trade-offs of using private VPC endpoints vs public APIs for model hosting?
**Answer:**
*   **Public APIs:** Easy to set up, but traffic travels over the public internet, posing security compliance risks. Data usage policies must be carefully reviewed.
*   **Private VPC Endpoints (e.g., Vertex AI PSC, AWS PrivateLink):** Traffic remains isolated within your private network, satisfying SOC 2 and HIPAA security requirements. However, network setup is more complex, and network data transfer costs can be higher.

#### Q10: How do you design an LLMOps pipeline that automatically validates new prompts in CI/CD?
**Answer:**
1.  A developer submits a pull request with an updated prompt template.
2.  The GitHub Actions runner triggers a test pipeline.
3.  The pipeline runs the prompt template against a golden test dataset.
4.  An LLM-as-a-judge model evaluates the output responses for completeness, accuracy, and safety constraints.
5.  If evaluation scores drop below staging thresholds, the build is blocked.

---

## 🤖 Role 3: Agentic AI Architect (Autonomous Systems)

Agentic AI Architects focus on state-graph orchestrators, persistent execution environments, Model Context Protocol (MCP) ecosystems, multi-agent communication, planning algorithms, and tool security.

### ❓ Top 10 Agentic Questions & Answers

#### Q1: How do you design a state-graph agent in LangGraph that supports loops and self-reflection?
**Answer:**
```python
from typing import TypedDict, List
from langgraph.graph import StateGraph, END

# 1. Define the shared state schema
class AgentState(TypedDict):
    query: str
    draft: str
    critique: str
    iterations: int

# 2. Define node functions
def draft_node(state: AgentState) -> dict:
    # LLM writes draft based on critique if available
    prompt = f"Draft report for: {state['query']}. Previous critique: {state.get('critique', 'None')}"
    # Call LLM...
    return {"draft": "This is a draft report output...", "iterations": state.get("iterations", 0) + 1}

def critique_node(state: AgentState) -> dict:
    # LLM evaluates draft and suggests improvements
    return {"critique": "The draft needs more statistics and concrete metrics."}

# 3. Define routing logic
def should_continue(state: AgentState) -> str:
    if state["iterations"] >= 3 or "looks good" in state.get("critique", "").lower():
        return "end"
    return "draft"

# 4. Build and compile the graph
workflow = StateGraph(AgentState)
workflow.add_node("drafter", draft_node)
workflow.add_node("critiquer", critique_node)

workflow.set_entry_point("drafter")
workflow.add_edge("drafter", "critiquer")
# Conditional routing loop
workflow.add_conditional_edges(
    "critiquer",
    should_continue,
    {
        "draft": "drafter",
        "end": END
    }
)
app = workflow.compile()
```

#### Q2: What is the Model Context Protocol (MCP) client-server architecture?
**Answer:** MCP separates AI clients (e.g., IDEs or agent frameworks) from tool and data integrations. MCP servers expose available tools, resources, and prompt templates using standard JSON-RPC protocol messages. Compliant clients can register and call these tools dynamically without writing custom API wrapper integrations for each tool.

#### Q3: How do you prevent an autonomous agent from entering infinite execution loops when calling tools?
**Answer:**
*   Set a maximum iteration limit on the agent loop (e.g., limit execution to 10 loops).
*   Implement loop detection algorithms that track tool calls. If the agent calls the same tool with the same arguments multiple times, break the loop and alert the user.
*   Refine tool descriptions and prompt guidelines to help the model identify when a task cannot be completed.

#### Q4: How do you design a long-running agent platform that can resume execution after server failures?
**Answer:**
*   Use a task queue (like Celery or BullMQ) to manage agent executions as asynchronous tasks.
*   Configure the database to save a snapshot (checkpoint) of the agent's state graph after every node execution.
*   If a worker container crashes, load the last saved checkpoint from the database and resume execution from that node.

#### Q5: Describe the difference between episodic and semantic memory in AI agents.
**Answer:**
*   **Episodic Memory:** Chronological logs of past conversations and completed actions, indexed in relational databases. Helps the agent maintain context across a conversation.
*   **Semantic Memory:** General concepts, rules, user preferences, and lessons learned. Embeddings are stored in vector databases, allowing the agent to retrieve relevant facts as needed.

#### Q6: How do you secure database write tools in an autonomous agent platform?
**Answer:**
1.  Configure the database credentials used by the agent to have least-privilege permissions (restricting access to specific tables and schemas).
2.  Avoid running raw SQL query input strings. Use parameterized queries or structured API routes.
3.  Inject human-in-the-loop approval gates for high-risk write actions.

#### Q7: Contrast sequential and supervisor multi-agent system design patterns.
**Answer:**
*   **Sequential Pattern:** Agents are arranged in a linear pipeline. The output of one agent passes directly to the next. Best for structured, predictable workflows.
*   **Supervisor Pattern:** A coordinator agent manages the state, delegates tasks to workers dynamically, reads worker responses, and decides on the next assignments. Best for complex, non-deterministic tasks.

#### Q8: What is the role of self-reflection in modern agentic reasoning?
**Answer:** Self-reflection allows the model to evaluate and correct its own outputs. When an agent runs a tool and encounters an error (e.g., a python compiler error or a failed API call), the reflection step passes the error logs back to the model, prompting it to analyze what went wrong, adjust its arguments, and retry the execution.

#### Q9: How do you evaluate the reliability and success rate of a multi-agent system?
**Answer:** Run automated simulation sweeps against a golden test dataset. Set up evaluator agents to score the production agent's outputs for completeness, accuracy, execution cost, and safety constraint compliance, compiling metrics to track performance regressions before deployments.

#### Q10: How do you sandboxed execution code generated by AI agents?
**Answer:** Use isolated container runtimes (like Google's **gVisor**) or WebAssembly (**WASM**) sandboxes to execute code tools. Configure the sandbox to have read-only file system access, CPU/RAM limits, and disable outbound network access to prevent malicious code execution or data exfiltration.

---

## 🚀 Role 4: Forward Deployed Engineer (Enterprise & Delivery)

Forward Deployed Engineers (FDEs) work directly with customers to scope proof-of-concepts, integrate AI systems with legacy databases, deploy models inside private cloud environments, and optimize performance.

### ❓ Top 10 FDE Questions & Answers

#### Q1: A customer requests a RAG system but has strict data privacy rules that forbid sending data to external APIs. How do you design and deploy a solution?
**Answer:**
*   **Deployment:** Deploy open-weight models (e.g., Llama 3 8B or 70B) inside the customer's private cloud network (VPC) using serving engines like vLLM on a Kubernetes cluster (GKE/EKS).
*   **Database:** Run an instance of an open-source vector database (like Qdrant or PGVector) within the same private network.
*   **Embeddings:** Run local embedding models (like Hugging Face BGE models) on the cluster, keeping all data isolation within the customer's VPC.

#### Q2: How do you convince an enterprise customer's security team that an AI agent will not leak sensitive customer data?
**Answer:**
1.  Implement input and output filters to identify and redact PII.
2.  Deploy the agent within a private network partition (VPC) with no public internet access.
3.  Configure database tool credentials to have least-privilege read-only scopes.
4.  Integrate human-in-the-loop gates to require approvals for sensitive database updates.
5.  Maintain a complete audit log of all tool calls and agent thoughts for compliance reviews.

#### Q3: A customer's internal database contains unstructured data spread across legacy systems (SQL Server, PDFs, Sharepoint). How do you design an ingestion pipeline?
**Answer:**
*   Deploy serverless ingestion workers to extract text from Sharepoint and PDF files (using OCR where necessary).
*   Use structured data connectors to extract records from SQL Server database tables.
*   Clean the raw text, apply chunking strategies, generate embeddings, and store them in a centralized vector database with metadata tags indicating the source system for permission filtering.

#### Q4: How do you scope a 4-week proof-of-concept (POC) project for an enterprise customer?
**Answer:**
*   **Week 1 (Align & Access):** Define success metrics, identify target data sources, and secure network access credentials.
*   **Week 2 (Data Ingestion & Basic Pipeline):** Build the document parser, set up the vector database, and implement a basic RAG search pipeline.
*   **Week 3 (Refining & User Interface):** Implement rerankers, refine prompt structures, and build a simple UI dashboard.
*   **Week 4 (Evaluation & Hand-off):** Run automated evaluation tests, review performance with stakeholders, and document the handover process.

#### Q5: A customer reports that the deployed RAG chatbot is occasionally returning incorrect answers. How do you debug this?
**Answer:**
1.  **Analyze Tracing Logs:** Review the trace logs (using LangSmith or Phoenix) to isolate where the failure occurred.
2.  **Verify Retrieval Quality:** Check if the correct context documents were successfully retrieved. If not, adjust chunking sizes or index configurations.
3.  **Evaluate Generation Grounding:** If the correct context was retrieved but the model generated an incorrect answer, update prompt guidelines to instruct the model to stick strictly to the context, or adjust model parameters.

#### Q6: How do you handle database connector authentications securely in customer VPCs?
**Answer:** Avoid hardcoding credentials. Use the customer cloud provider's Secrets Manager service to store connection strings, and configure the application containers to retrieve secrets dynamically at runtime using IAM Workload Identity configurations.

#### Q7: Contrast dense vector search and sparse keyword search.
**Answer:**
- **Dense Vector Search:** Converts text into high-dimensional embeddings, capturing conceptual meaning and synonym matches.
- **Sparse Keyword Search (BM25):** Tracks word frequencies, matching exact terms, SKU numbers, and product codes. Production systems combine both using hybrid search.

#### Q8: A customer has a high volume of time-sensitive queries. How do you optimize latency?
**Answer:**
1.  Implement semantic caching to return cached responses for common questions.
2.  Set up model serving with continuous batching and PagedAttention (using vLLM) to handle concurrent requests.
3.  Use speculative decoding or route simple queries to smaller, faster models.

#### Q9: How do you manage stakeholder expectations when an AI model behaves non-deterministically?
**Answer:** Educate stakeholders that generative models operate on probability distributions. Build automated testing pipelines to run batch evaluations, tracking error rates as statistical ranges rather than expecting absolute answers, and implement guardrails to catch out-of-bounds responses.

#### Q10: What steps do you take when a customer's legacy database schema changes, breaking your agent's query tools?
**Answer:**
*   Expose databases through structured API routes or views rather than allowing the agent to query raw tables directly.
*   Write tool schemas that explicitly define the expected fields and types.
*   If a schema change occurs, update the API route or tool description to align with the new schema without breaking the core agent engine.

---

## 📈 System Metrics & Evaluation Reference

| Metric | Target Goal | Optimization Strategy |
| :--- | :--- | :--- |
| **Time to First Token (TTFT)** | $< 800\text{ ms}$ | Prompt caching, Speculative decoding, Model gateways. |
| **Inter-Token Latency (ITL)** | $< 30\text{ ms/token}$ | Continuous batching, GPU scale allocations, smaller models. |
| **Context Recall** | $> 90\%$ | Semantic chunking, Hybrid search, Cross-Encoder reranking. |
| **Faithfulness (No Hallucinations)** | $> 98\%$ | Output guardrails, strict prompt grounding rules. |
| **Semantic Cache Hit Rate** | $30\% - 50\%$ | Dynamic vector similarity indexing (similarity threshold $\ge 0.96$). |

---
*Created By Biswanath Giri*
