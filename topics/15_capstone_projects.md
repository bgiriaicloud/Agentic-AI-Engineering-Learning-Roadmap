# Stage 15: Capstone Projects — Specifications & Architecture Blueprints

This module contains the detailed specifications, architecture diagrams, and component maps for your capstone projects across all levels.

---

## 🟢 LEVEL 1: Beginner Capstone — Structured Information Extractor

An API that takes unstructured profiles (resumes, invoices, support tickets) and returns a validated JSON object conforming to a target Pydantic schema.

### 🏗️ Directory Structure
```
structured_extractor/
├── app/
│   ├── __init__.py
│   ├── main.py          # FastAPI application
│   ├── schemas.py       # Pydantic schemas
│   └── extractor.py     # Google GenAI model interaction
├── tests/
│   └── test_extractor.py
├── .env
├── requirements.txt
└── README.md
```

### ⚙️ Component Map
```
[ Client Text Payload ] ──> [ FastAPI POST /extract ] ──> [ Pydantic Validation ]
                                                                 │
                                                                 ▼
[ Validated JSON Response ] <── [ Extracted Schema ] <─── [ Google GenAI Model ]
```

### 🚀 Key Implementation Steps:
1.  **Define Target Schemas:** Write Pydantic models with detailed descriptions for each field to guide the model.
2.  **Enforce Output Format:** Call the Gemini API configuration setting the response MIME type to `application/json` and passing your Pydantic class to the `response_schema` parameter.
3.  **Validate Results:** Wrap the model call in a try-except block to handle validation errors and return structured validation feedback if fields are missing or incorrectly formatted.

---

## 🟡 LEVEL 2: Intermediate Capstone — Advanced Document RAG Chatbot

A Q&A system that processes uploaded PDF manuals, indexes their content using hybrid search, and generates grounded answers with citation references.

### 🏗️ Directory Structure
```
document_rag_chatbot/
├── app/
│   ├── main.py          # FastAPI server
│   ├── ingest.py        # PDF extraction & semantic chunking
│   ├── vector_store.py  # PGVector database connections
│   ├── search.py        # Hybrid dense-sparse retrieval
│   └── generator.py     # Prompt assembly & streaming generation
├── data/
│   └── manuals/         # Uploaded documents directory
├── requirements.txt
└── README.md
```

### ⚙️ Component Map
```
[ PDF Document ] ──> [ Semantic Chunking ] ──> [ Embedding Model ] ──> [ PGVector Store ]
                                                                             ▲
                                                                             │ (Hybrid query)
[ Grounded Answer ] <── [ LLM Generator ] <── [ Cohere Rerank ] <────────────┘
```

### 🚀 Key Implementation Steps:
1.  **Extract & Clean:** Use a parsing library (like PyPDF or LlamaIndex Reader) to clean the raw text.
2.  **Semantic Chunking:** Calculate embedding similarities between sentences to split text where context changes, preserving context compared to character-based chunking.
3.  **Hybrid Search:** Configure PGVector to query dense embeddings, and run a parallel BM25 search on a text search index. Combine results using Reciprocal Rank Fusion (RRF).
4.  **Rerank Chunks:** Pass the top 20 retrieved chunks to a Cohere Rerank model (`rerank-english-v3.0`) to select the top 5 most relevant contexts.
5.  **Assemble Grounded Prompts:** Instruct the LLM to write responses using only the provided context, and include citation markers (e.g., `[Manual_v2.pdf, Page 12]`) in the output.

---

## 🔴 LEVEL 3: Advanced Capstone — Multi-Agent Enterprise Assistant

A team of collaborative agents (Researcher, SQL Analyst, Writer, and Supervisor) that coordinates research, analyzes databases, and writes marketing drafts, complete with human approval gates.

### 🏗️ Directory Structure
```
multi_agent_assistant/
├── app/
│   ├── agents/
│   │   ├── supervisor.py     # LangGraph orchestration state machine
│   │   ├── researcher.py     # Web search agent
│   │   ├── sql_analyst.py    # Database analyst agent
│   │   └── writer.py         # Report editor agent
│   ├── tools/
│   │   ├── search_tool.py
│   │   └── db_tool.py
│   ├── main.py               # FastAPI server
│   └── state.py              # Shared graph schema
├── frontend/                 # React UI displaying agent chat trajectories
├── requirements.txt
└── README.md
```

### ⚙️ Component Map
```
                          ┌──────────────┐
                          │  Supervisor  │ <─── [ Human approval gate ]
                          └──────┬───────┘
                                 │
            ┌────────────────────┼────────────────────┐
            ▼                    ▼                    ▼
     [ Researcher ]       [ SQL Analyst ]          [ Writer ]
     (Web Search Tool)    (Sales DB Access)       (Drafting Tools)
```

### 🚀 Key Implementation Steps:
1.  **Define Shared Graph State:** Design a central state schema that stores conversation history, worker updates, and active tasks.
2.  **Build Specialized Worker Nodes:** Write nodes containing specific system instructions and tools.
3.  **Build Supervisor Orchestrator:** Program a supervisor node that analyzes inputs, selects the next worker, and routes execution dynamically.
4.  **Implement Human Approval Gates:** Configure the supervisor node to pause execution before running write actions or returning final reports, saving the state to a database and resuming only after receiving human feedback.

---

## 🏆 LEVEL 4: Expert Capstone — Production-Grade Multi-Agent Platform

An enterprise platform deploying multiple agents via the Model Context Protocol (MCP), managed by a LangGraph supervisor, hosted on Google Kubernetes Engine (GKE), and monitored using LangSmith.

### 🏗️ Directory Structure
```
enterprise_platform/
├── infra/
│   ├── terraform/       # Terraform configs for GCP GKE and Cloud SQL
│   └── kubernetes/      # K8s Deployment and Service YAMLs
├── mcp_servers/
│   ├── database_server/ # MCP server exposing DB access
│   └── search_server/   # MCP server exposing search APIs
├── agent_orchestrator/
│   ├── main.py          # LangGraph execution engine
│   ├── gateway.py       # LiteLLM proxy and semantic cache
│   └── tracing.py       # LangSmith instrumentation configs
├── security/
│   └── guardrails.co    # NeMo Guardrails configuration files
└── README.md
```

### ⚙️ Component Map
```
                  [ Global Load Balancer ]
                             │
                             ▼
                    [ API Gateway Proxy ]
                             │
                             ▼
                  [ LiteLLM Cache Gateway ]
                             │
                             ▼
                [ LangGraph Orchestrator ] <──> [ NeMo Guardrails ]
                             │
            ┌────────────────┴────────────────┐
            ▼                                 ▼
      [ MCP Client ]                     [ MCP Client ]
            │                                 │
    (JSON-RPC / stdio)                (JSON-RPC / HTTP)
            ▼                                 ▼
   [ Database Server ]                 [ Search Server ]
```

### 🚀 Key Implementation Steps:
1.  **Provision Infrastructure:** Write Terraform files to launch a private GKE cluster with NVIDIA L4 GPU node pools, setting up Cloud SQL databases and VPC Private Service Connect.
2.  **Deploy Model Serving:** Deploy a vLLM server on GKE to serve an open-weight model (e.g., Llama 3 70B). Set up a LiteLLM gateway proxy in front of it to handle prompt caching and routing.
3.  **Deploy MCP Servers:** Write and deploy database and search MCP servers in separate containers, registering them with the orchestrator client.
4.  **Configure Security Middleware:** Implement input and output scanning using NeMo Guardrails and Llama Guard, and write log data to an audit database.
5.  **Monitor with LangSmith:** Wire up LangSmith tracing to monitor agent loops, tool response times, token counts, and execution costs.
