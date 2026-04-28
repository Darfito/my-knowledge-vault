---
title: "Agent Capabilities vs Skills vs Instructions"
type: reference
tags: [paperclip, agents, capabilities, skills, configuration]
date: 2026-04-28
---

# Agent Capabilities vs Skills vs Instructions

Three things define what a Paperclip agent can do. They serve different purposes and live in different places.

---

## 1. AGENTS.md — The Core Instructions

The `AGENTS.md` file is the agent's **main system prompt**. Paperclip injects it on every heartbeat. It defines:

- The agent's role and identity
- What work it accepts and from whom
- What it produces and in what format
- Who it hands off to and how
- The execution contract (how it behaves autonomously)

This is the most important file. An agent with no skills and no capabilities description will still behave correctly as long as its `AGENTS.md` is well-written.

For the Shannon factory agents, these live at:
```
/home/paperclip/shannon-factory/agents/
├── ceo/AGENTS.md
├── pm/AGENTS.md
├── uiux-lead/AGENTS.md
└── swe-lead/AGENTS.md
```

---

## 2. Skills — Modular Reference Docs

Skills are **additional markdown documents** injected into the agent's context alongside `AGENTS.md`. They teach the agent how to use a specific tool, protocol, or pattern.

An agent declares which skills it uses in the `skills:` list in its `AGENTS.md` frontmatter:

```yaml
---
name: SWE Lead
skills:
  - paperclip
  - worker-dispatch
---
```

### Skill types

| Type | Description |
|---|---|
| **Platform-native** | Built into Paperclip. Available to any agent without being in the package. |
| **Package skill** | Shipped inside the company package. Lives at `skills/<name>/SKILL.md`. |
| **External (referenced)** | Sourced from a GitHub URL. Listed in the skill's `metadata.sources`. |

### The `paperclip` skill (platform-native)

Every Shannon agent references the `paperclip` skill. It is **not** in the package — Paperclip provides it automatically. It teaches agents how to:
- Post comments on issues
- Update issue fields (stage, assignee)
- Create approvals and `request_confirmation` checkpoints
- Interact with the Paperclip board API

This is why the import showed warnings like "skill paperclip not in package" — that is expected and correct behavior.

### The `worker-dispatch` skill (package skill)

Used by SWE Lead and UI/UX Lead. Teaches them how to dispatch parallel Haiku worker API calls via `asyncio.gather()`, collect results, and report per-worker token cost back to Paperclip. Lives at:
```
/home/paperclip/shannon-factory/skills/worker-dispatch/SKILL.md
```

---

## 3. Capabilities Field — Display Label Only

The **Capabilities** field in the agent's Configuration tab (UI path: `/<company>/agents/<slug>/configuration`) is a **free-text description** — like a one-line résumé.

Example: `"Node.js, PostgreSQL, API design"`

It is injected into the system prompt context so other agents and humans can see what this agent does at a glance (org chart, approval cards, dashboard). **It has no functional effect** — leaving it empty does not change how the agent behaves.

Fill it in if you want clean org chart tooltips. Skip it if you don't care about the display layer.

---

## Summary

| Source | Lives in | Controls | Required? |
|---|---|---|---|
| `AGENTS.md` | Company package | Actual behavior, role, pipeline position | Yes |
| Skills | Package or platform | Extra reference docs injected into context | Recommended |
| Capabilities field | Paperclip UI / DB | Display label in org chart and approvals | No |

---

## Shannon Factory Agent Stack

| Agent | AGENTS.md | Skills used |
|---|---|---|
| CEO | `agents/ceo/AGENTS.md` | `paperclip` (platform-native) |
| PM | `agents/pm/AGENTS.md` | `paperclip` (platform-native) |
| UI/UX Lead | `agents/uiux-lead/AGENTS.md` | `paperclip`, `worker-dispatch` |
| SWE Lead | `agents/swe-lead/AGENTS.md` | `paperclip`, `worker-dispatch` |
