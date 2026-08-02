# Stage 1: Foundations — Study Guide & Notebook

This module covers the core software engineering and mathematical prerequisites required for Generative AI and Agentic AI.

---

## 📅 Study Checklist
- [ ] Understand Python concurrency: synchronous execution vs threads vs `asyncio` event loops.
- [ ] Write asynchronous code using `async/await`, manage asynchronous tasks, and gather concurrent execution results.
- [ ] Implement robust Git workflows: feature branching, merging, conflict resolution, and rebasing.
- [ ] Write simple Bash scripts, manage file permissions, and monitor system resources on remote CLI servers.
- [ ] Interface with REST APIs, parse headers, validate payloads using Pydantic, and handle rate-limiting.
- [ ] Solve linear algebra problems involving vectors, matrices, dot products, and matrix multiplication.
- [ ] Understand probability fundamentals: Bayes' theorem, standard distributions, mean, variance, and simple linear regression.

---

## 🐍 Python Async Concurrency Deep-Dive

In GenAI applications, operations are frequently IO-bound (e.g., waiting for LLM API responses, database queries, vector retrieval). Using synchronous code blocks your application. Python's `asyncio` allows co-operative multitasking on a single thread.

### Event Loop, Tasks, and Futures
- **Event Loop:** The engine that runs asynchronous tasks, handles network IO events, and switches between tasks when they are waiting.
- **Coroutines:** Functions defined with `async def`. They yield execution back to the loop using the `await` keyword.
- **Tasks:** Wrappers for coroutines that schedule them on the event loop to run concurrently.

### Practical Code Example: Fetching LLM APIs Concurrently
```python
import asyncio
import time
from typing import List, Dict

# Mock asynchronous API caller representing an LLM inference request
async def call_llm_api(prompt: str, request_id: int) -> Dict[str, str]:
    print(f"[Request {request_id}] Sending prompt: '{prompt}'...")
    # Simulate network latency (yielding control to the event loop)
    await asyncio.sleep(2.0)
    print(f"[Request {request_id}] Response received!")
    return {"request_id": str(request_id), "response": f"Response for prompt '{prompt}'"}

async def main():
    prompts = [
        "What is the capital of France?",
        "Explain quantum computing in one sentence.",
        "How do transformers work?",
        "Write a quicksort in Python."
    ]
    
    start_time = time.time()
    
    # Schedule all LLM calls concurrently
    tasks = [call_llm_api(prompt, idx) for idx, prompt in enumerate(prompts)]
    
    # Wait for all tasks to complete and gather results
    results: List[Dict[str, str]] = await asyncio.gather(*tasks)
    
    end_time = time.time()
    print("\nAll tasks completed!")
    print(f"Total execution time: {end_time - start_time:.2f} seconds (Expected: ~2.0s, not 8.0s)")
    print(results)

if __name__ == "__main__":
    asyncio.run(main())
```

---

## ⚙️ Git & GitHub Production Workflows

In a collaborative enterprise environment, raw commits to `main` are restricted. The **GitHub Flow** or **Git Flow** branching models are enforced.

### Core Commands Cheatsheet
| Operation | Command | Explanation |
| :--- | :--- | :--- |
| Create & Switch Branch | `git checkout -b feature/topic-name` | Creates a new branch and checks it out. |
| Stage Changes | `git add .` | Stages modified files for commit. |
| Commit | `git commit -m "feat: add LLM call"` | Commits with semantic commit messages. |
| Rebase with Main | `git pull --rebase origin main` | Pulls main changes and replays your work on top. |
| Push Branch | `git push -u origin feature/topic-name` | Pushes local branch to remote repository. |

---

## 🐧 Linux & CLI Administration

AI model hosting and servers operate on Linux distributions (Ubuntu, Debian, RHEL).

### Critical Shell commands for AI Engineers:
1.  **System Monitoring:**
    *   `top` or `htop`: View active CPU, memory usage, and running processes.
    *   `nvidia-smi`: Monitor GPU memory usage, temperature, and CUDA cores (Crucial for deep learning!).
2.  **Text Search & Manipulation:**
    *   `grep -ri "api_key" .`: Search recursively for the string "api_key" in files.
    *   `tail -f /var/log/syslog`: Tail logs in real-time.
3.  **Process Management:**
    *   `kill -9 <PID>`: Force termination of a hung python process.

---

## 🔢 Mathematics for AI Foundations

### 1. Linear Algebra
Neural networks represent inputs as vectors (points in space). Embeddings are vectors with hundreds of dimensions.
*   **Dot Product:** Measures the alignment of two vectors. 
    $$\mathbf{a} \cdot \mathbf{b} = \sum_{i=1}^{n} a_i b_i$$
    If the dot product is high, the vectors point in similar directions (semantic similarity).
*   **Matrix Multiplication:** The fundamental operation of feedforward layers. If input vector $X$ of size $1 \times N$ is multiplied by weight matrix $W$ of size $N \times M$, the resulting vector has size $1 \times M$, transforming the representation.

### 2. Probability & Statistics
*   **Bayes' Theorem:** Calculates conditional probability, forming the basis for statistical pattern prediction.
    $$P(A|B) = \frac{P(B|A)P(A)}{P(B)}$$
*   **Mean and Variance:** Used in normalization operations (Batch Normalization, Layer Normalization) to stabilize neural network weight training.

---

## ❓ Common Interview Q&As

#### Q1: What is the difference between concurrency and parallelism, and how does Python's Global Interpreter Lock (GIL) affect them?
**Answer:** Concurrency is about *dealing* with lots of things at once (structuring your program to execute parts out-of-order, like IO waits). Parallelism is about *doing* lots of things at once (running tasks on multiple CPU cores simultaneously). 
Python has a Global Interpreter Lock (GIL) that prevents multiple native threads from executing Python bytecodes at once. Therefore, multi-threading in Python does not achieve true parallelism for CPU-bound tasks. However, for IO-bound tasks (like LLM API calls), `asyncio` or threading work extremely well because the thread releases the GIL while waiting for network responses. For true CPU-bound parallelism (e.g., custom data parsing, tokenization), Python's `multiprocessing` library must be used.

#### Q2: What is the difference between a GET and a POST HTTP request, and which status codes indicate success vs failure?
**Answer:**
- **GET:** Requests data from a specified resource without changing system state. Parameters are sent in the URL query string.
- **POST:** Submits data to be processed to a specified resource, often resulting in a change in state or side-effects. Data is enclosed in the request body (typically JSON).
- **Status Codes:**
  - `2xx` (e.g., `200 OK`, `201 Created`): Success.
  - `3xx` (e.g., `301 Moved`): Redirection.
  - `4xx` (e.g., `400 Bad Request`, `401 Unauthorized`, `429 Too Many Requests`): Client-side errors.
  - `5xx` (e.g., `500 Internal Server Error`, `503 Service Unavailable`): Server-side errors.
