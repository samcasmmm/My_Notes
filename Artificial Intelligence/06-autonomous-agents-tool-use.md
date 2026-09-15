[🏠 Back to AI Index](./README.md) • [⬅️ Prev: Modern RAG](./05-modern-rag-vector-databases.md) • [Next: Fine-Tuning & LLMOps ➡️](./07-llmops-fine-tuning-evals.md)

<div align="center">
  <h1>06. Autonomous AI Agents & Tool Calling</h1>
  <p><b>The ReAct Framework, Function Calling, Model Context Protocol (MCP), Memory, & Multi-Agent Systems</b></p>
</div>

---

## 📑 Module Index
- [1. What Makes an LLM an "Agent"?](#1-what-makes-an-llm-an-agent)
  - [The 4 Pillars: Planning, Memory, Tools, & Action](#the-4-pillars-of-an-agent)
- [2. The ReAct Pattern (Reasoning + Acting)](#2-the-react-pattern-reasoning--acting)
  - [The Thought ➔ Action ➔ Observation Execution Loop](#react-execution-loop)
- [3. Function Calling & Tool Use Internals](#3-function-calling--tool-use-internals)
  - [JSON Schema Specifications & Execution Lifecycles](#tool-calling-lifecycle)
- [4. Agent Memory Systems](#4-agent-memory-systems)
  - [Short-Term vs Working Memory vs Long-Term Semantic Memory](#agent-memory-architectures)
- [5. Model Context Protocol (MCP)](#5-model-context-protocol-mcp)
  - [Standardizing Tools, Prompts & Resources Across LLMs](#mcp-standard)
- [6. Multi-Agent Systems & State Graphs](#6-multi-agent-systems--state-graphs)
  - [Supervisor-Worker & Multi-Agent Collaboration (LangGraph / CrewAI)](#multi-agent-orchestration)
  - [Human-in-the-Loop (HITL) & Safety Guardrails](#human-in-the-loop)

---

## 1. What Makes an LLM an "Agent"?

A standard LLM is purely reactive (Input text $\rightarrow$ Output text). An **Autonomous Agent** uses the LLM as a **reasoning engine** to interact dynamically with external environments:

```
                          ┌────────────────────────┐
                          │   PLANNING / REASONING │
                          │  (LLM Thought Engine)  │
                          └───────────┬────────────┘
                                      │
        ┌─────────────────────────────┼─────────────────────────────┐
        ▼                             ▼                             ▼
┌──────────────┐              ┌──────────────┐              ┌──────────────┐
│    MEMORY    │              │  TOOL USE /  │              │    ACTION    │
│ (Short/Long) │              │  APIs / CODE │              │ (Environment)│
└──────────────┘              └──────────────┘              └──────────────┘
```

---

## 2. The ReAct Pattern (Reasoning + Acting)

Introduced by Yao et al., **ReAct** solves the hallucination problem by interleaving reasoning traces with real-world tool execution:

```
User: "Check the price of AAPL stock and convert $1000 worth to EUR."

Thought 1: I need to find the current stock price of Apple (AAPL).
Action 1:  get_stock_price(ticker="AAPL")
Observation 1: {"ticker": "AAPL", "price_usd": 225.50}

Thought 2: $1000 / 225.50 = 4.434 shares. Now I need the USD to EUR exchange rate.
Action 2:  get_exchange_rate(from="USD", to="EUR")
Observation 2: {"rate": 0.92}

Thought 3: 1000 USD * 0.92 = 920 EUR. I have all facts needed to answer.
Final Answer: "AAPL is trading at $225.50. $1,000 equals €920.00, which buys approximately 4.43 shares."
```

---

## 3. Function Calling & Tool Use Internals

When using Function Calling, the model does **not** execute the function directly. Instead:
1. You provide the model with a list of available tools defined as **JSON Schemas**.
2. If needed, the LLM outputs a structured JSON object containing tool name and arguments.
3. Your client code executes the local function and passes the result back as a `tool` role message.

```
[ User Prompt + Tool JSON Schemas ] ──► [ LLM ] ──► [ tool_calls JSON Payload ]
                                                              │
[ LLM Final Answer ] ◄── [ LLM ] ◄── [ Tool Result JSON ] ◄── [ Backend Executes Code ]
```

```python
# Example OpenAI / Anthropic Tool Schema Definition
tools = [
    {
        "type": "function",
        "function": {
            "name": "fetch_user_order",
            "description": "Fetches order status and tracking info by Order ID.",
            "parameters": {
                "type": "object",
                "properties": {
                    "order_id": {
                        "type": "string",
                        "description": "Unique order identifier, e.g. ORD-98124"
                    }
                },
                "required": ["order_id"]
            }
        }
    }
]
```

---

## 4. Agent Memory Systems

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ 1. SHORT-TERM MEMORY (In-Context Window)                                               │
│    • Current conversation history and recent tool observations.                        │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 2. WORKING / SCRATCHPAD MEMORY                                                         │
│    • Active plan checklist, current sub-goal state, temporary variables.               │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 3. LONG-TERM SEMANTIC MEMORY                                                           │
│    • Vector Database indexing past user preferences, episodic logs, and learned facts. │
│    • Retrieved via similarity search on relevant user queries.                         │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Model Context Protocol (MCP)

Created to eliminate the chaos of writing custom API integration glue for every different AI agent and IDE.

```
┌──────────────┐         Standardized JSON-RPC         ┌───────────────────────────┐
│   MCP Host   │ ◄───────────────────────────────────► │        MCP Server         │
│ (Claude, IDE,│         Transport (stdio / SSE)       │ (PostgreSQL, GitHub,      │
│  Custom App) │                                       │  Filesystem, Browser APIs)│
└──────────────┘                                       └───────────────────────────┘
```

- **Resources:** Static/dynamic data streams (files, database tables).
- **Tools:** Executable functions callable by the model.
- **Prompts:** Reusable pre-crafted prompt templates.

---

## 6. Multi-Agent Systems & State Graphs

Complex enterprise workflows exceed the capacity of a single monolithic prompt. **Multi-Agent Systems** decompose tasks across specialized collaborating agents.

### Supervisor-Worker Architecture (LangGraph)

```
                              [ SUPERVISOR AGENT ]
                              (Orchestrator & Router)
                                   │        │
                     ┌─────────────┘        └─────────────┐
                     ▼                                    ▼
             [ RESEARCH AGENT ]                   [ CODER AGENT ]
             (Searches web / docs)                (Writes & tests code)
                     │                                    │
                     └────────────► [ REVIEWER AGENT ] ◄──┘
                                   (Verifies output)
```

- **State Graphs:** Agents maintain a typed shared state dictionary. Execution moves across directed graph nodes and cycles conditionally until verification criteria are satisfied.

---

### Human-in-the-Loop (HITL)

> [!IMPORTANT]
> Any action with side-effects (e.g., executing SQL `DROP TABLE`, transferring money, deleting files, sending public emails) MUST require explicit user authorization before execution via confirmation interrupts.

---

[➡️ Continue to Module 07: Fine-Tuning, Alignment & LLMOps](./07-llmops-fine-tuning-evals.md)
