---
title: "Single-Agent vs Multi-Agent — Patterns & Tradeoffs"
type: knowledge-reference
status: active
tags: [factory-building, architecture, agents, multi-agent, langgraph, agentic-ai]
source: "Agentic AI: Single vs Multi-Agent Systems — Ida Silfverskiöld, Data Science Collective (Oct 28 2025)"
date: 2026-04-30
---

# Single-Agent vs Multi-Agent — Patterns & Tradeoffs

## Core Framing

Agentic AI = **programming with natural language**. Instead of rigid explicit code, you instruct LLMs to route data and perform actions through plain language.

LLMs are best understood as a **communication layer** sitting on top of structured systems and existing data sources — not as standalone reasoners. Without clean data, they hallucinate. With clean data, they adapt to messy real-world input and ambiguous instructions well.

> "If we don't have access to clean, structured data, we start making things up. Same with LLMs."

Key principle: **if you can build part of a workflow programmatically, do it.** LLMs are not better than a calculator at math — give the LLM access to a calculator instead.

---

## Single-Agent Workflow

**Definition:** One LLM + one system prompt + multiple tools. The agent decides which tools to use and when.

```
User Message
     │
     ▼
  LLM Agent
  (one system prompt)
  ┌──────────────────────────────┐
  │  tool_A  tool_B  tool_C ...  │
  └──────────────────────────────┘
     │
     ▼
  Final Output
```

### Strengths
- Fast to build and run
- Good for Q&A and human-in-the-loop use cases
- Sufficient when a human can follow up to probe deeper

### Weaknesses
- **Control problem**: no matter how detailed the system prompt, the model may not follow requests or use all provided tools
- Too many tools → agent picks only some, or picks wrong ones
- Cannot reliably self-orchestrate complex multi-step research
- Results are surface-level; agent "makes shortcuts" when tasks grow complex

### When to use
- Simple Q&A or lookup tasks
- Human-in-the-loop workflows (human follows up with more specific questions)
- Rapid prototyping to validate an idea

---

## Multi-Agent Workflow

**Definition:** Multiple LLMs, each with a specific scope, limited tools, and clear instructions. Requires careful architectural planning before building.

Two main patterns:

### 1. Hierarchical (Recommended for precision tasks)
A lead/orchestrator agent delegates tasks to sub-agents or teams. Each agent/team has one responsibility.

```
User Message
     │
     ▼
  Lead Agent (Orchestrator)
  Receives task, plans, delegates
     │            │
     ▼            ▼
 Research       Editing
 Team Lead      Team Lead
  │    │         │    │
 A1   A2        B1   B2
(workers/sub-agents with limited tools each)
     │
     ▼
  Final Summary (advanced LLM)
```

- Each sub-agent: limited scope, not too many tools
- Lower-level models (e.g. Gemini Flash) for execution agents
- Advanced model (e.g. GPT-4/5) for final summarization
- **Shared memory space** ("research pad") lets one team write findings that another team reads

### 2. Collaborative (Good for exploration, hard to control for precision)
Agents work in parallel without strict hierarchy. Breaks functions into modules but is harder to control for precise output. Good for prototyping/playing around, not ideal for production workflows that require consistent accuracy.

---

## Quality Comparison

The article demonstrated a **tech news research** use case:

| Dimension | Single Agent | Multi-Agent (Hierarchical) |
|---|---|---|
| Speed | Fast (seconds) | Slow (several minutes) |
| Depth of research | Surface-level, 3–4 categories | Deep per-keyword + per-topic research |
| Tool coverage | May skip tools | Each agent forced to use its assigned tools |
| Output quality | OK for quick brief | Investor-grade weekly brief with "Why it matters" analysis |
| Control | Low | High — each team's behavior is explicitly designed |

---

## Architecture Design Rules

1. **Plan the process before building** — agentic workflows are just another form of system design. Define steps, structure logic, decide how each part behaves.
2. **Limited scope per agent** — each agent should have few, specific tools. Too many tools = agent takes shortcuts.
3. **Use programmatic logic where possible** — don't force LLMs to do structured data processing. Reserve them for interpretation, judgment, and summarization.
4. **Memory management matters** — don't dump all messages into shared state and give every agent access. Separate state between teams (use scratchpads per team).
5. **Use weaker models for workers, stronger for summarizers** — Gemini Flash for execution, GPT-4/5 or equivalent for final output synthesis.
6. **Error handling and guardrails** — add guardrails so agents always use their assigned tools; parse user queries into structured format before passing to agents.
7. **Clean data is non-negotiable** — the better the data source, the less error handling you need, and the faster/cheaper the system runs.

---

## LangGraph Notes (Framework)

- Graph-based framework built on LangChain
- More technical and complex than CrewAI/AutoGen
- Preferred by many experienced developers
- Routes (edges) can be set up statically (simple) or dynamically (complex multi-agent)
- LangSmith Studio = visual debugger / runner for LangGraph workflows
- Author preference: no framework at all for production, but patterns learned from LangGraph are worth absorbing

---

## Relationship to Orchestrator-Worker Architecture

This article's "hierarchical multi-agent" pattern maps directly to the factory's **Orchestrator-Worker** model:

| Article concept | Factory equivalent |
|---|---|
| Lead/orchestrator agent | CEO / PM / SWE Lead Agent |
| Sub-agents with limited tools | Workers (Haiku, no tools) |
| Shared "research pad" | State / scratchpad passed between agents |
| Editing team summarizer | SWE Lead review pass |
| Collaborative = harder to control | Reason we chose hierarchical (Orchestrator-Worker) |

Key alignment: **collaborative agents are flexible but imprecise; hierarchical agents are controlled and production-ready** — which is why the factory went with the orchestrator-worker model.

---

## Related
- [[Orchestrator-Worker Architecture]] — how we implement the hierarchical pattern in the factory
- [[Personas & Skills Matrix]] — role definitions that map to orchestrator vs. worker
- [[Pipeline Model & Ticket Strategy]] — pipeline stages that orchestrators manage
