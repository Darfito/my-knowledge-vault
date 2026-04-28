---
title: "Orchestrator-Worker Architecture"
type: architecture-decision
status: decided
tags: [factory-building, architecture, agents, workers, token-efficiency]
date: 2026-04-25
---

# Orchestrator-Worker Architecture

## Motivation

Running every persona as a full agent is expensive. Full agents carry heavy context (CLAUDE.md, system prompt, conversation history, tool definitions) on every API call. For "dirty work" — writing component code, generating CSS, producing test files — that reasoning overhead produces no benefit. The factory separates **thinking roles** (Orchestrators / Agents) from **executing roles** (Workers) to minimize token spend without sacrificing output quality.

---

## Core Distinction

| Dimension | Orchestrator (Agent) | Worker |
|---|---|---|
| API call type | Multi-turn, looping | Single `messages.create()` |
| Tool use | Yes | No |
| Reasoning / thinking | Yes | No |
| Conversation history | Full | None |
| Recommended model | Sonnet 4.6 / Opus 4.7 | Haiku 4.5 |
| Context size | Full (CLAUDE.md + spec + history) | Minimal (task prompt only) |
| Token cost | High | Low |
| Example task | "What should be built?" | "Write this component" |

---

## Persona Classification

### Full Agents (Orchestrators)
These roles require judgment, multi-step reasoning, and cross-context decisions. They cannot be reduced to workers.

| Persona | Why Full Agent |
|---|---|
| **CEO** | Parses PDF/PRD, decomposes into work plan, makes strategic trade-offs |
| **PM** | Writes spec from ambiguous input, validates alignment with user intent |
| **SWE Lead** | Reviews all worker output, corrects errors, decides architecture |

### Light Agent
This role requires some judgment but can run with minimal context and a cheaper model.

| Persona | Why Light Agent | Constraints |
|---|---|---|
| **UI/UX** | Must make creative design decisions — layout, hierarchy, component naming | Uses Sonnet; loads spec + design system only (no full CLAUDE.md); outputs a design spec, not final code |

> **When UI/UX can become a Worker:** If a fully codified design system already exists before the pipeline runs, UI/UX can be reduced to a Worker that applies existing rules mechanically. Without a pre-existing design system, worker-only output will be inconsistent.

### Workers
These roles execute a single, well-scoped task. They receive explicit instructions from an Orchestrator and return output. No reasoning loop required.

| Worker | Triggered by | Task |
|---|---|---|
| **Code Worker** | SWE Lead Agent | Write one component / module per call |
| **Design Worker** | UI/UX Agent | Generate CSS / Tailwind / tokens from design spec |
| **Test Worker** | SWE Lead Agent | Write unit or integration tests for a specific module |

Workers are **not** Paperclip agents. They are API calls made inside a skill owned by the Orchestrator that dispatched them. Paperclip has no visibility into Worker execution.

---

## Pipeline Overview

```
PDF / PRD (input)
      │
      ▼
  CEO Agent  ──────────────────────────────────────────────┐
  (full)                                                    │
  Parses document, produces structured work plan           │
      │                                                     │
      ▼                                                     │
  PM Agent                                                  │
  (full)                                                    │
  Shapes spec, resolves ambiguities                         │
      │                                                     │
      ▼                                                     │
  UI/UX Agent                                              │
  (light)                                                   │
  Produces design spec / component inventory               │
    │         │                                            │
    ▼         ▼                                            │
 Design    Design     ← Workers (parallel, Haiku)          │
 Worker1   Worker2                                          │
    │         │                                            │
    └────┬────┘                                            │
         ▼                                                  │
  SWE Lead Agent                                           │
  (full)                                                    │
  Reviews design worker output, dispatches code work       │
    │         │         │                                  │
    ▼         ▼         ▼                                  │
 Code      Code      Test      ← Workers (parallel, Haiku) │
 Worker1   Worker2   Worker                                 │
    │         │         │                                  │
    └────┬────┘─────────┘                                  │
         ▼                                                  │
  SWE Lead Agent                                           │
  Reviews all output, corrects errors                      │
         │                                                  │
         ▼                                                  │
  ┌──────────────────┐                                     │
  │  HUMAN CHECKPOINT │ ◄───────────────────────────────────┘
  │  Review & Approve │   (CEO loops back if rejected)
  └──────────────────┘
         │
         ▼
      Shipped
```

---

## Human-in-the-Loop Checkpoints

Checkpoints are **designed pause points**, not interrupts. The pipeline pauses and surfaces a summary for human review before proceeding.

| Checkpoint | Triggered after | Human action |
|---|---|---|
| **Spec approval** | PM Agent finishes spec | Approve / redirect spec |
| **Design approval** | UI/UX Agent + Design Workers finish | Approve / adjust design direction |
| **Code review** | SWE Lead reviews all worker output | Approve / request corrections |

If the human rejects at any checkpoint, the relevant Agent re-runs with the human's feedback injected as a correction message. Workers are re-dispatched with the corrected instructions.

---

## Worker Implementation Pattern

A Worker is a single async function call. Example (Python / Anthropic SDK):

```python
import anthropic

client = anthropic.Anthropic()

async def dispatch_worker(task: str) -> str:
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=4096,
        system="You are a precise code executor. Write exactly what is specified. Output code only, no explanation.",
        messages=[{"role": "user", "content": task}]
    )
    return response.content[0].text
```

Multiple workers can be dispatched in parallel using `asyncio.gather()` or equivalent, then their results are collected by the Orchestrator Agent for review.

---

## Token Efficiency Rationale

| Scenario | Old (all agents) | New (orchestrator-worker) |
|---|---|---|
| Write 10 components | 10 × full agent cost | 1 SWE agent turn + 10 × worker cost |
| Write 20 tests | 20 × full agent cost | 1 SWE agent turn + 20 × worker cost |
| Estimated savings | baseline | ~60–80% reduction on execution tasks |

The key insight: Agents are expensive because they carry heavy context on every turn. Workers receive only what they need for one task — a tight system prompt and the task description. No history, no tools, no CLAUDE.md.

---

## Relationship to Paperclip

Paperclip orchestrates **Agents only**. The Agent → Worker dispatch happens inside a skill, invisible to Paperclip. This means:

- Ticket lifecycle (spec → design → implementation → tested → shipped) still maps to the Agent pipeline
- Worker execution is an implementation detail of a skill, not a pipeline stage
- Paperclip's token budget calculations should account for Agent turns only; Worker costs are separate

---

## Related
- [[Personas & Skills Matrix]] — updated to include Agent/Worker type column
- [[Pipeline Model & Ticket Strategy]] — ticket stages map to Agent transitions, not Worker calls
- [[Lifecycle - Project vs Feature]] — which Agent executes which lifecycle stage
