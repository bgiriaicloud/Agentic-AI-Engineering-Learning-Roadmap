# Stage 4: LLM Application Development — Study Guide & Notebook

This module covers orchestrating language models within software applications, including session memory management, streaming, and stateful flow engines like LangGraph.

---

## 📅 Study Checklist
- [ ] Connect and query models using Google GenAI, OpenAI, and Anthropic SDKs.
- [ ] Implement token-by-token response streaming using asynchronous API endpoints.
- [ ] Design and implement memory management strategies to handle context drift.
- [ ] Build a stateful application flow containing cyclical loops using LangGraph.
- [ ] Integrate external tools dynamically by passing schema definitions to LLMs.

---

## 🌊 Streaming and Server-Sent Events (SSE)

Large language models can take several seconds to generate complete responses. To improve the user experience, applications stream tokens in real-time. This is achieved using **Server-Sent Events (SSE)**, which push data from the server to the client over a persistent HTTP connection.

Here is a practical FastAPI endpoint implementation demonstrating how to stream an LLM response:

```python
import asyncio
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from google import genai

app = FastAPI()

async def get_llm_stream(prompt: str):
    # Initialize the Google GenAI client
    client = genai.Client()
    
    # Call the streaming model endpoint
    response = client.models.generate_content_stream(
        model='gemini-2.5-flash',
        contents=prompt
    )
    
    for chunk in response:
        # yield tokens as they arrive
        yield f"data: {chunk.text}\n\n"
        await asyncio.sleep(0.01) # Yield execution back to event loop

@app.get("/stream")
def stream_prompt(prompt: str = "Write a 200-word essay about trees."):
    return StreamingResponse(get_llm_stream(prompt), media_type="text/event-stream")

# To run: uvicorn filename:app --reload
```

---

## 🧠 LLM Memory Management Patterns

Because LLMs are stateless, every API request must include the entire conversation history. As conversations grow, this history can exceed the model's context window limit. Common memory patterns include:

1.  **Buffer Memory:** Passes the raw history of the last $N$ messages. Easy to implement, but consumes tokens quickly.
2.  **Window Memory:** Keeps only the last $K$ conversational turns, discarding older messages. Keeps token usage predictable, but older context is lost.
3.  **Summary Memory:** Uses an LLM to generate a summary of older parts of the conversation. When older messages are discarded, their summary is injected as context, preserving long-term themes.

---

## 🕸️ Stateful Graphs: Introduction to LangGraph

Simple LLM applications use linear chains. However, complex applications require loops, conditional routing, and persistent state. **LangGraph** models these applications as stateful graphs:

*   **State:** A central, shared data structure (often a Python dict or Pydantic model) that is updated as execution passes through nodes.
*   **Nodes:** Python functions that take the current state, perform operations (e.g., call an LLM or execute a tool), and return updates to the state.
*   **Edges:** Define the path between nodes. **Conditional Edges** use routing functions to decide which node to visit next.

```
       ┌──────────────┐
       │  Start Node  │
       └──────┬───────┘
              │
              ▼
       ┌──────────────┐
  ┌───>│  Agent Node  │
  │    └──────┬───────┘
  │           │
  │    [Conditional Edge]
  │       Is Tool Called?
  │       /            \
 Yes     /              \ No
  │     ▼                ▼
┌─┴──────────┐     ┌────────────┐
│ Tool Node  │     │  End Node  │
└────────────┘     └────────────┘
```

---

## ❓ Common Interview Q&As

#### Q1: What is LangChain Expression Language (LCEL) and what benefits does it offer?
**Answer:** LCEL is a declarative language designed to easily chain LangChain components together. It uses the pipe operator (`|`) to pipe outputs from one component to the inputs of the next. Under the hood, LCEL automatically supports:
- Asynchronous execution (`ainvoke`, `astream`).
- Parallel step execution (`RunnableMap` / `RunnableParallel`).
- Built-in logging, tracing, and prompt fallbacks.

#### Q2: How do you implement conversational state persistence across multiple HTTP requests?
**Answer:** Because HTTP is stateless, the server must persist the conversation history in an external database (e.g., Redis, PostgreSQL, or MongoDB) keyed by a unique `session_id`. When a request arrives, the server retrieves the historical messages, appends the new user message, calls the LLM, saves the updated history, and returns the response. For complex agent systems, LangGraph uses **Checkpointers** to save the entire graph state at each step, enabling session resuming and execution rollbacks.
