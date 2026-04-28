---
title: "Pipeline Model & Ticket Strategy"
type: architecture-decision
status: decided
tags: [factory-building, pipeline, tickets, paperclip, workflow]
source: ai-software-factory-conversation.md
date: 2026-04-22
---

# Pipeline Model & Ticket Strategy

## Core Decision: Pipeline, not Fan-out

**Fan-out**: CEO assigns to PM, UI/UX, Engineer, QA all at once — they work in parallel.  
**Pipeline**: CEO → PM → UI/UX → Engineer → QA — one handoff at a time, each stage sees the prior stage's output.

**Pipeline wins for software development.** UI/UX needs PM's spec before designing. Engineer needs design before implementing. QA needs implementation before testing. Fan-out produces disconnected work and wasted effort.

---

## Three Options Evaluated

### Option 1 — Dependency Tickets
CEO creates root ticket for PM. PM finishes → creates child ticket for UI/UX (blocked-by the PM ticket). UI/UX finishes → creates child ticket for Engineer. And so on.

- ✓ Maps cleanly to Paperclip's ticket model. Full traceability. One owner per ticket.
- ✗ Each persona must know what ticket to create next and who to assign it to.

### Option 2 — Single Ticket, Sequential Stages ← **CHOSEN**
One ticket per feature. Ticket has a `stage` field: `spec → design → implementation → sandboxed → tested → shipped`. Assignee changes as stage advances.

- ✓ One feature = one ticket = one conversation thread
- ✓ Maps to the ai-software-factory pipeline 1:1
- ✓ Parallel-assignment problem solved structurally: only one person assigned at a time
- ✗ Paperclip reassignment support needs verification

### Option 3 — Hybrid (Epic + Sub-tickets)
One parent epic (CEO creates). PM creates sub-tickets per stage, routed sequentially. Parent stays open until all sub-tickets close.

- ✓ Executive visibility (CEO sees one thing) + granular traceability (each stage has its own ticket)
- ✗ More machinery. Easier to get out of sync.

---

## Why Option 2 Wins for MVP

1. Matches the factory pipeline 1:1 — the `stage` field maps directly to `spec → architecture → implementation → sandboxed → tested → reviewed → shipped`.
2. One feature = one ticket = one conversation thread. When debugging a feature three weeks later, everything is in one place.
3. Parallel-assignment problem is solved structurally — one person assigned at a time because only one person is responsible at the current stage.

**Upgrade path**: Move to Option 3 when executive-level dashboards showing only epics are needed.

---

## Enforcement Mechanism

Each persona's skills include an explicit **"what's next" step**. For example:
- `foundation:shape-spec` ends by saying "hand off to UI/UX: move ticket to `design` stage, assign to UI/UX agent."
- `design:import` ends with "hand off to Engineer."

The skill content itself encodes the pipeline — agents don't figure it out. This mirrors the `NEXT_COMMAND:` markers already in ai-software-factory commands; Paperclip just needs to respect those markers as assignment triggers.

---

## Back-edge Support (Deferred)

When QA finds a bug → ticket bounces back to Engineer (not PM). Change stage back to `implementation`, reassign.

**Not MVP.** Deferred backlog item: *"Back-edge support: Engineer/QA/UI-UX can bounce a ticket back one stage with a reason comment."*

---

## Related
- [[Lifecycle - Project vs Feature]] — the lifecycle the pipeline enforces
- [[Personas & Skills Matrix]] — who is assigned at each stage
