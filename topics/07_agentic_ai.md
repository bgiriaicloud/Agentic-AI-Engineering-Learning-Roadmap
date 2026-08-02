# Stage 7: Agentic AI — Study Guide & Notebook

This module covers Agentic AI, moving from simple LLM apps to autonomous agent loops that plan, reflect, leverage memory, and invoke tools.

---

## 📅 Study Checklist
- [ ] Diagram the core ReAct agent execution loop.
- [ ] Understand the differences between episodic, semantic, and working memory.
- [ ] Build an autonomous planner agent using task decomposition.
- [ ] Differentiate between deterministic workflow agents and autonomous loops.
- [ ] Design and write a multi-agent system with distinct agent roles.
- [ ] Implement a human-in-the-loop approval gate for high-risk actions.

---

## 🔁 The ReAct (Reasoning & Acting) Paradigm

The **ReAct** loop is the fundamental execution loop for autonomous agents. It forces the agent to alternate between reasoning steps ("Thoughts") and action steps ("Actions").

```
             ┌──────────────────────┐
             │      User Input      │
             └──────────┬───────────┘
                        │
                        ▼
             ┌──────────────────────┐
             │       THOUGHT        │ <───┐
             │  (Agent plan steps)  │     │
             └──────────┬───────────┘     │
                        │                 │
                        ▼                 │
             ┌──────────────────────┐     │
             │        ACTION        │     │
             │   (Tool call JSON)   │     │
             └──────────┬───────────┘     │
                        │                 │
                        ▼                 │
             ┌──────────────────────┐     │
             │     OBSERVATION      │ ────┘
             │    (Tool output)     │
             └──────────────────────┘
```

1.  **Thought:** The agent reasons about its current state and plans its next action (e.g., *"To find the customer's order status, I first need to look up their email address."*).
2.  **Action:** The agent decides to call an external tool with specific parameters (e.g., calling tool `lookup_user(email="biswanath@example.com")`).
3.  **Observation:** The application executes the tool locally, captures the output, and appends it to the agent's context window (e.g., *"Tool output: Order ID 9944 found. Status: Shipped."*).
4.  **Loop:** The agent reads this new context, reasons again, and either calls another tool or generates its final response.

---

## 🧠 Memory Systems for Agents

An agent needs memory to execute multi-turn, stateful tasks:

| Memory Class | Duration | Implementation | Purpose |
| :--- | :--- | :--- | :--- |
| **Working Memory** | Ephemeral (Current Run). | Shared state dictionary, context window messages. | Tracking intermediate plans and raw tool output lists during the current execution. |
| **Episodic Memory** | Persistent (Session). | Relational Database (SQL/NoSQL) containing chat histories. | Remembering past interactions and dialogue sequences from previous turns. |
| **Semantic Memory** | Persistent (Cross-Session). | Vector Database containing embedding indexing. | Storing high-level user preferences, facts, and lessons learned. |

---

## 🏢 Workflows vs Autonomous Agents

*   **Workflows (Chains/DAGs):** Linear or branched execution paths where the sequence of steps is pre-programmed and deterministic. The LLM is used only for processing information within each step.
*   **Autonomous Agents:** The LLM manages the execution loop, dynamically deciding which tools to call and what steps to execute next.

```
Deterministic Workflow:
[User Input] ──> [LLM Step 1] ──> [Tool Execution] ──> [LLM Step 2] ──> [Response]

Autonomous Agent:
[User Input] ──> [LLM Planner] ──> (Loop: Thought -> Action -> Observation) ──> [Response]
```

---

## 👥 Multi-Agent Architectures

For complex problems, a single agent with too many tools can become overloaded and lose focus. Multi-agent systems partition tasks among specialized agents:
1.  **Collaboration:** Specialized agents communicate and share state updates directly (e.g., a coder agent passes code to a test runner agent).
2.  **Orchestration (Supervisor-Worker):** A coordinator agent distributes work to sub-agents, reads their outputs, and decides when the final goal has been met.

---

## ❓ Common Interview Q&As

#### Q1: What is the risk of "Agent Drift", and how do you protect against it?
**Answer:** Agent drift occurs when an autonomous agent loses focus on its primary goal over a long execution loop. As the context window fills with intermediate tool execution logs, the agent can become distracted, execute irrelevant tools, or enter infinite loops. 
To protect against drift:
1.  Enforce strict maximum iteration limits (e.g., limit execution to 10 loops).
2.  Compress intermediate tool outputs, keeping only summaries in the context window.
3.  Use structural guardrails like state-graph transitions (e.g., LangGraph) to force the agent to progress through defined phases.

#### Q2: How do you implement a Human-in-the-loop (HIL) mechanism for sensitive tool actions?
**Answer:**
1.  **Interception:** When the agent requests a sensitive tool call (e.g., `send_wire_transfer`), pause execution.
2.  **State Saving:** Serialize and save the current state of the agent to a database.
3.  **Notification:** Send a notification to the user (e.g., via Slack, Webhook, or email) containing the proposed arguments.
4.  **Resume:** When the human approves or rejects the action, resume the agent execution loop with the updated state and the user's feedback.
