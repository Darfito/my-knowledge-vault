---
title: "Paperclip Compatibility — Orchestrator-Worker"
type: architecture-decision
status: decided
tags: [factory-building, paperclip, workers, compatibility, token-efficiency]
date: 2026-04-25
---

# Paperclip Compatibility — Orchestrator-Worker Architecture

> **Short answer:** Paperclip requires no breaking changes for the Orchestrator-Worker model. Workers are invisible to Paperclip — they run inside Agent skills as direct API calls. Two small additive items are worth planning.

---

## What Does NOT Need to Change in Paperclip

| Paperclip Component | Why No Change Needed |
|---|---|
| `agents` table & org tree | Workers are not agents. They have no entry in the org chart, no heartbeat, no budget row. |
| `issues` / task lifecycle | Workers do not create or close issues. Only Agents transition ticket stages. |
| Heartbeat system | Worker dispatch happens *inside* a single heartbeat execution — it is not a separate heartbeat. |
| `process` / `http` adapters | SWE Lead Agent still runs via the process adapter exactly as before. |
| Auth & API keys | Workers use the same Anthropic API key as the dispatching Agent's environment — no new key model needed. |
| Approval mechanism | Existing board-level approval gates are unaffected. |

Paperclip's view of the world does not change: it sees CEO → PM → UI/UX → SWE Lead as agents with heartbeats and tickets. The fact that SWE Lead internally dispatches workers is an implementation detail of a skill.

---

## Item 1 — Worker Cost Reporting (Additive, No Schema Change)

**The gap:** Workers make Anthropic API calls directly (not through Paperclip's adapter layer), so their token costs are not automatically tracked by Paperclip's budget system.

**The fix:** The `worker:dispatch` skill must read `response.usage` from each worker's API response and report those tokens to Paperclip's existing cost event API after every worker batch completes.

```
Worker API call finishes
  → read response.usage.input_tokens + output_tokens
  → POST /api/cost-events  { agent_id, task_id, input_tokens, output_tokens, model }
```

This is **new logic in the skill**, not a change to Paperclip's schema or API. Paperclip's cost event ingestion endpoint already exists (V1 scope).

**Priority:** Must-have before production use. Without it, SWE Lead's budget tracking will under-report actual token spend.

---

## Item 2 — Human Checkpoint Approval Type (Phase 2, Optional for MVP)

**The gap:** Paperclip V1's approval mechanism covers board approvals for hires and CEO strategy proposals. The factory's human checkpoint (human reviews worker output before pipeline continues) is not a named approval type.

**MVP workaround:** Use issue comments + manual board action. The SWE Lead posts a comment: `CHECKPOINT: Ready for human review. Approve to continue.` The board operator reads it and manually advances the ticket stage.

**Phase 2 addition:** Add a new approval type `review_checkpoint` to Paperclip's approval model. SWE Lead requests approval programmatically; Paperclip blocks pipeline progression until the board approves or redirects.

This is a clean additive schema change — one new enum value in the approvals model, one new UI state in the board.

---

## Item 3 — `worker:dispatch` Skill (New Skill, Not a Paperclip Change)

The `worker:dispatch` skill is the only new artifact required for the Orchestrator-Worker model to work. It lives in the SWE Lead's skill directory and is responsible for:

1. Receiving a task list from the SWE Lead Agent
2. Dispatching N workers in parallel (async API calls to Anthropic, model: `claude-haiku-4-5-20251001`)
3. Collecting all results
4. Reporting token costs to Paperclip (see Item 1)
5. Returning collected output to the SWE Lead for review

This is a skill file — it has no dependency on Paperclip internals and does not require any Paperclip code change.

---

## Summary

```
Change required in Paperclip codebase?    NO
Change required in Paperclip schema?      NO (Phase 2: one approval enum value)

New work required:
  ✅ worker:dispatch skill        — new skill, not a Paperclip change
  ✅ Cost reporting in skill      — POST to existing /api/cost-events endpoint
  💡 review_checkpoint approval  — Phase 2 addition, MVP uses comments instead
```

---

## Related
- [[Orchestrator-Worker Architecture]] — full architecture and pipeline diagram
- [[Pipeline Model & Ticket Strategy]] — ticket stages remain Agent-driven
- [[Personas & Skills Matrix]] — worker:dispatch is assigned to SWE Lead
- [[Paperclip Integration Unknowns]] — other open Paperclip questions (reassignment, NEXT_COMMAND, etc.)
