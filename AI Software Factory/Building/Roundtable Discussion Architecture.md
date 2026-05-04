---
title: "Roundtable Discussion Architecture"
type: architecture-spec
status: proposed
tags: [paperclip, multi-agent, debate, mad-asimetris, roundtable, factory-building]
date: 2026-05-04
---

# Roundtable Discussion Architecture

> **Goal.** Replace Paperclip Shannon's strictly sequential `CEO → PM → UI/UX → SWE Lead` handoff with a structured debate pattern in which SWE Lead validates PM's interpretation of a CEO task **before** execution begins. Eliminates the failure mode where PM over-promises or misreads technical constraints and SWE Lead silently inherits a bad brief.

---

## Pattern Classification

This is **not** pure Roundtable / Society of Minds (free brainstorming until consensus). It is **MAD Asimetris** (Multi-Agent Debate, asymmetric) with shared context:

| Pure Roundtable | This Architecture |
|---|---|
| No hierarchy | Hierarchy: CEO is Judge, PM/SWE/UI-UX are Debaters |
| Free order | Structured order: CEO → PM → SWE → optional UI/UX |
| Unbounded discussion | Hard cap: 2 PM revisions before escalation |
| Consensus by exhaustion | Consensus by SWE `[CONFIRMED]` signal |

Reference research: see _Multi-Agent Communication Patterns_ session notes — MAD Asimetris (Liang 2023, [arxiv 2305.19118](https://arxiv.org/abs/2305.19118)).

---

## Roles

| Role | Function | Authority |
|---|---|---|
| Human | Origin of all tasks. Resolves deadlocks CEO cannot. | Final |
| CEO | Interprets human input, frames task, breaks deadlocks. | Highest among agents |
| PM Lead | Decomposes task into stories, scope, dependencies, estimates. | Drafts and revises |
| SWE Lead | Reviews PM brief from technical perspective. Confirms or challenges. Dispatches workers post-confirm. | Validation gate |
| UI/UX Lead | Joins only when design scope present. Outputs design spec. | Optional contributor |

---

## Shared Context: Discussion Board

All agents read every prior agent's output. Paperclip Shannon maintains a cumulative document — the **Discussion Board** — and injects it into each agent's context.

```
Discussion Board (per ticket)
├── [TASK] from CEO
├── [PM BRIEF v1]
├── [CONCERNS] from SWE Lead
├── [PM BRIEF v2]
├── [CONFIRMED] from SWE Lead
├── [UI/UX SPEC]   ← optional
└── [LGTM] from SWE Lead
```

Storage: append-only, scoped to one issue. Lives in `issues.discussion_board` (new column) or as a thread of structured comments parsed by tag.

---

## Flow

```
HUMAN INPUT
    │
    ▼
┌─────────┐
│   CEO   │   Frames task → emits [TASK]
└────┬────┘
     ▼
┌─────────┐
│PM LEAD  │   Reads [TASK] → emits [PM BRIEF v1]
└────┬────┘
     ▼
┌──────────┐
│SWE LEAD  │   Reads [TASK] + [PM BRIEF]
└────┬─────┘   MUST output ≥2 questions/concerns
     │         Then either:
     │
     ├── [CONFIRMED] ────────────────────────────────────┐
     │                                                   │
     └── [CONCERNS]                                       │
              │                                           │
              ▼                                           │
         PM Lead emits [PM BRIEF v2]                     │
         SWE Lead reviews again                           │
              │                                           │
              ├── [CONFIRMED] ────────────────────────────┤
              │                                           │
              └── still [CONCERNS] after v2 → ESCALATE   │
                       │                                  │
                       ▼                                  │
                 ┌─────────┐                              │
                 │   CEO   │  Reads full board            │
                 └────┬────┘  Emits [DECISION]            │
                      │                                   │
                      ├── CEO resolves → [CONFIRMED] ─────┤
                      │                                   │
                      └── CEO needs human input           │
                               │                          │
                               ▼                          │
                          HUMAN CHECKPOINT                │
                          Human responds → CEO re-frames  │
                          → loop back to PM               │
                                                          │
                                          ┌───────────────┘
                                          ▼
                                  Design scope present?
                                   │              │
                                  [Yes]         [No]
                                   │              │
                                   ▼              │
                              ┌──────────┐        │
                              │  UI/UX   │        │
                              └────┬─────┘        │
                                   ▼              ▼
                              ┌─────────────────────────┐
                              │ SWE LEAD — [LGTM]       │
                              │ Worker dispatch starts  │
                              └─────────────────────────┘
```

---

## Output Format (Signals)

Paperclip Shannon parses these tags from agent comments to drive routing. Format must be exact — tag at start of a line.

### CEO

```
[TASK: <one-line summary>]
<full brief: goal, constraints, acceptance criteria>
```

### PM Lead

```
[PM BRIEF v<n>]
Stories:
  - ...
Scope: in/out
Dependencies: ...
Estimate: ...
Open questions for SWE: ...
```

### SWE Lead

```
[CONFIRMED]
or
[CONCERNS]

Required (≥2):
1. <question or technical concern>
2. <question or technical concern>
```

The `≥2 concerns` rule is a **sycophancy guard**. Without it, SWE Lead drifts toward rubber-stamping PM briefs. If SWE Lead genuinely has no concerns, it must still surface 2 verification questions ("did you account for X? did you verify Y?") before issuing `[CONFIRMED]`.

### CEO (escalation)

```
[DECISION: <resolution>]
<reasoning, addresses both PM brief and SWE concerns>
```

### UI/UX Lead

```
[UI/UX SPEC]
<design spec output>
```

### SWE Lead (final)

```
[LGTM]
<one-line dispatch note>
```

---

## Termination Conditions

| Condition | Action |
|---|---|
| SWE emits `[CONFIRMED]` | Advance to UI/UX (if design scope) or directly to LGTM |
| 2 PM revisions exhausted, SWE still `[CONCERNS]` | Escalate to CEO |
| CEO emits `[DECISION]` | Treated as override — pipeline advances regardless of SWE concerns |
| CEO unable to decide | Trigger human checkpoint via `request_confirmation` |
| Human responds | CEO re-frames `[TASK]`, loop restarts from PM |

Hard upper bound: max 5 total round-trips per issue before forced escalation. Prevents runaway discussion cost.

---

## Failure Modes & Mitigations

| Failure mode | Source | Mitigation |
|---|---|---|
| **Sycophancy** — SWE rubber-stamps every brief | All MAD-style patterns | `≥2 concerns` rule on every SWE turn, even when confirming |
| **Deadlock** — PM and SWE never converge | Asymmetric debate | Hard cap 2 revisions → CEO escalation |
| **Scope creep** — debate wanders from CEO's task | Roundtable patterns | Each agent must reference `[TASK]` tag in its output |
| **Echo chamber** — agents agree on a wrong premise | Shared context exposure | CEO has authority to override with `[DECISION]` even when PM/SWE agree |
| **Token bloat** — Discussion Board grows unboundedly | Shared context | Board stores tagged sections only; full agent output preserved per round but board never replayed beyond current issue |

---

## Integration with Paperclip Shannon Pipeline

This architecture **replaces the spec stage** of the existing pipeline (`spec → design → implementation → sandboxed → tested → shipped`). The spec stage becomes:

```
spec stage (NEW):
  CEO frames → PM brief → SWE confirm/debate → [LGTM]
  → emit NEXT_COMMAND: stage=design (or stage=implementation if no UI/UX)
```

Existing pipeline mechanisms preserved:
- `pipeline_stage` advances via `NEXT_COMMAND:` after `[LGTM]`
- `request_confirmation` interaction used for human checkpoint on CEO escalation
- Worker dispatch unchanged (orthogonal to this spec)

---

## Implementation Surface in Paperclip Shannon

| Component | Change |
|---|---|
| `issues.discussion_board` (new column or comment-thread aggregation) | Stores cumulative tagged outputs |
| Signal parser (new helper, pairs with existing `parseNextCommand()`) | Detects `[CONFIRMED]`, `[CONCERNS]`, `[DECISION]`, `[LGTM]` etc. |
| Revision counter | Tracks PM brief version per issue |
| Routing logic | Picks next assignee based on most recent signal |
| UI/UX trigger | Heuristic on CEO/PM brief: presence of "UI", "design", "screen", "page" → activate UI/UX round |
| Agent skills | Each agent's skill prompt updated to require tagged output format |
| `company-creator` template | Updated to bootstrap agents with debate-aware instructions |

---

## Out of Scope

- Worker dispatch changes — covered separately in [[Workers Subscription Consolidation — Spec]]
- Pipeline stages after spec (design/implementation/sandboxed/etc.) — unchanged
- Multi-issue parallelism — one Discussion Board per issue, no cross-issue debate

---

## Open Questions

1. **Where does the Discussion Board live physically?** New table column, structured comment thread, or sidecar file in `companies/<slug>/`? Must verify what fits Paperclip's existing data model.
2. **How does Paperclip detect "design scope present"?** Heuristic on text, an explicit flag in CEO brief, or always run UI/UX? Recommendation: explicit `[NEEDS_DESIGN: yes/no]` line in CEO `[TASK]` output.
3. **Does the `≥2 concerns` rule survive system-prompt drift?** Need to test that SWE Lead reliably emits concerns even on trivial tasks. Fallback: server-side validation rejects SWE comments that are `[CONFIRMED]` without ≥2 enumerated lines.
4. **Should `[DECISION]` from CEO trigger an audit log entry?** Useful for retro analysis of where pipelines stalled.

---

## Related

- [[Orchestrator-Worker Architecture]] — agent vs worker split this preserves
- [[Pipeline Model & Ticket Strategy]] — `NEXT_COMMAND:` mechanism this layers on
- [[Personas & Skills Matrix]] — agent roles being modified
- [[Workers Subscription Consolidation — Spec]] — orthogonal Topic 1 spec
- [[Single-Agent vs Multi-Agent — Patterns & Tradeoffs]] — pattern background
