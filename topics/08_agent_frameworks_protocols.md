# Stage 8: Agent Frameworks & Protocols — Study Guide & Notebook

This module covers the open protocols, developer kits, and security frameworks used to build interoperable, production-ready AI agents.

---

## 📅 Study Checklist
- [ ] Understand the architecture of the Model Context Protocol (MCP).
- [ ] Build a custom MCP server in Python to expose databases and APIs as tools.
- [ ] Set up secure gRPC/JSON-RPC communication channels between agents.
- [ ] Use Google ADK (AGY SDK) to declare and deploy an agent.
- [ ] Implement secure sandboxing for tool execution (using Docker or WASM).
- [ ] Configure least-privilege IAM scopes for agent API access keys.

---

## 🔌 Model Context Protocol (MCP)

Developed by Anthropic, the **Model Context Protocol (MCP)** is an open standard that decouples AI models (clients) from their data sources and tools (servers). Instead of building custom integrations for every tool, developers write standard MCP servers that any compliant AI client can query.

```
┌─────────────────────────────────┐
│           AI Client             │ (e.g., Cursor, Claude Desktop, custom agent)
└───────────────┬─────────────────┘
                │
         MCP Protocol (JSON-RPC)
                │
                ▼
┌─────────────────────────────────┐
│          MCP Server             │ (Exposes Prompts, Resources, and Tools)
└──────┬────────┬────────┬────────┘
       │        │        │
       ▼        ▼        ▼
    [files]  [db SQL]  [APIs]
```

### Core Concepts of MCP:
1.  **Resources:** Read-only data sources exposed to the client (e.g., local files, database schemas, API readouts).
2.  **Tools:** Executable functions that the client can request the server to run (e.g., write to database, execute test command).
3.  **Prompts:** Pre-defined prompt templates that the client can load and execute.

### Practical Python MCP Server Code Example
Below is an example of an MCP server using the official Python MCP SDK:

```python
import mcp.server.fastapi
from mcp.types import Tool, Resource
from pydantic import BaseModel

# Initialize the server
server = mcp.server.fastapi.Server(name="db_query_server")

# Define tools
@server.tool(
    name="query_sales_db",
    description="Run queries on the database to extract sales data."
)
def query_sales_db(query_str: str) -> str:
    # Safe implementation: mock query results
    # In production, run actual database queries with sanitization
    return f"Results for query: '{query_str}' - Total Sales: $45,000"

# Expose server resources
@server.resource("schema://sales_table")
def get_schema() -> str:
    return "TABLE sales (id INT, date DATE, item VARCHAR, price DECIMAL)"

if __name__ == "__main__":
    # Start the server using standard standard I/O (stdio) or HTTP transport
    # server.start()
    print("MCP Server configured.")
```

---

## 🔒 Tool Security & Sandboxing

Allowing an autonomous agent to execute shell commands, run Python scripts, or edit databases can pose significant security risks. Enterprise platforms implement a **Zero-Trust Tool Execution** architecture:

```
[ Agent Gateway ] ── (1) Requests Code Execution ──> [ Sandbox Controller ]
                                                             │
                                                     (2) Spins up ephemeral
                                                             │
                                                             ▼
                                                   [ gVisor / WASM Sandbox ]
                                                   - Read-only File System
                                                   - No Network Access
                                                   - CPU/RAM limits
```

### Security Measures:
1.  **Isolation (gVisor/WASM):** Run code execution tools inside sandboxed container runtimes (like Google's **gVisor**) or WebAssembly (**WASM**) blocks that restrict access to the host kernel.
2.  **Network Restrictions:** Disable outbound network access in the sandbox to prevent agents from exfiltrating data.
3.  **Timeouts and Resource Limits:** Set strict CPU and RAM limits to protect against infinite loops or resource exhaustion.

---

## ❓ Common Interview Q&As

#### Q1: How does the Model Context Protocol (MCP) differ from traditional function calling?
**Answer:** Traditional function calling requires the developer to manually define schemas, write routing code, and manage the execution context inside their application. 
MCP standardizes this process. The AI client uses a standard protocol to negotiate capabilities with the server automatically. This allows you to write an MCP tool server once and register it across multiple AI clients (e.g., IDEs, Slack bots, and orchestrators) without modifying client code.

#### Q2: What security patterns protect against an agent executing malicious SQL commands via tool calling?
**Answer:**
1.  **Parametrization:** Never allow agents to execute raw SQL input strings. Force the tool interface to accept structured parameters and run queries using parameterized statements (e.g., ORM models).
2.  **Least Privilege:** Configure database users used by the agent to have read-only access (no `DROP TABLE`, `DELETE`, or `UPDATE` permissions) unless explicitly authorized.
3.  **Human Gateways:** Force high-risk queries or write actions to pause and request human authorization.
