---
title: "Reference — dispatch skill"
type: reference
tags: [factory-building, workers, reference, parallel-execution, claude-code]
source: https://github.com/bassimeledath/dispatch
date: 2026-04-25
---

# Reference — `dispatch` skill

> Source: https://github.com/bassimeledath/dispatch
> Install: `npx skills add bassimeledath/dispatch -g`

A Claude Code skill that fans out tasks to background workers running in fresh context windows. Used here as a **design reference** for the factory's `worker:dispatch` skill — specifically for its fan-out mechanics and parallel coordination pattern.

---

## What `dispatch` Does (Relevant Parts)

### 1. Fan-out + parallel execution
User describes work → dispatcher creates a checklist → N background workers pick up tasks simultaneously → results collected as workers finish. The main session stays lean and responsive.

```
/dispatch we launch Thursday, need pre-launch sweep:
1) security audit (JWT, sessions, Stripe) — use opus, worktree
2) performance testing (load tests, N+1 queries, DB indexes) — sonnet, worktree
3) frontend audit (WCAG accessibility, bundle size, error boundaries)
```
→ Three workers dispatch immediately in parallel. Main session unblocked.

### 2. Per-task model routing
Tasks can specify which model to use: `use opus to review`, `use gemini to refactor`. If unspecified, uses configured default. This maps directly to the factory's pattern of using Haiku for workers and Sonnet/Opus for agents.

### 3. Non-blocking collect
Workers report back as they finish. The orchestrator does not block waiting for the slowest worker — it receives results incrementally.

---

## What to Borrow for `worker:dispatch`

| Pattern from `dispatch` | How to apply in `worker:dispatch` |
|---|---|
| Checklist-based task decomposition | SWE Lead breaks work into a task list before dispatching |
| Parallel async fan-out | `asyncio.gather()` — all workers fire at once, results collected together |
| Per-task model specification | Default to Haiku; allow Sonnet for tasks flagged as "judgment-required" |
| Incremental result collection | Collect as workers finish, pass full batch to SWE Lead for review |

---

## What NOT to Borrow

| `dispatch` feature | Why skip it |
|---|---|
| Full Claude Code session per worker | Too expensive — defeats token-efficiency goal. Our workers are single `messages.create()` calls. |
| Bi-directional Q&A (worker asks user questions) | Workers must not ask questions. Ambiguity should be resolved by the SWE Lead Agent *before* dispatch. |
| IPC / inter-process communication | Not needed — workers are in-process async calls, not separate processes. |
| Worktree isolation per worker | Workers return text output only; they do not write files directly. |

---

## Key Design Constraint (Factory-Specific)

`dispatch` workers are full sessions — expensive but capable. Factory workers are single API calls — cheap and focused. The trade-off:

- `dispatch` solves **context overflow** (main session gets too full)
- Factory workers solve **token cost** (execution tasks should not run as full agents)

These are compatible goals — a factory session could use `dispatch`-style fan-out *while* keeping each worker as a lightweight Haiku call. The fan-out mechanic is reusable; the worker implementation is not.

---

## Related
- [[Orchestrator-Worker Architecture]] — the factory pattern this reference informs
- [[Paperclip Compatibility — Orchestrator-Worker]] — how workers fit into the Paperclip control plane
- [[Personas & Skills Matrix]] — `worker:dispatch` is assigned to SWE Lead
