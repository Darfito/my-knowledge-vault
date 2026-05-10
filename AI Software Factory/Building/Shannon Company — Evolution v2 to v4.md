---
title: "Shannon Company — Evolution from v2 to v4"
type: history
status: current
tags: [paperclip, shannon, history, v2, v3, v4, evolution]
date: 2026-05-10
---

# Shannon Company — Evolution from v2 to v4

> **Purpose.** Explains *why* each company version was built — what wasn't working, what design decisions were made, and what changed at each step. Not just a feature list, but the context behind every jump.

---

## Timeline

```
2026-04-27  Shannon Company v2 born (Step 3)
2026-05-05  Shannon Company v3 born (Step 12)
2026-05-10  Shannon Company v4 planned (Steps 13–14)
```

---

## v2 — Sequential Pipeline (2026-04-27)

### Why it was created

Before v2, Shannon had no standardized company bootstrap. Every setup was done manually — create agents one by one, fill in instructions, assign skills. No consistency between projects.

Step 3 introduced the `company-creator` skill and the first template (`shannon-company.md`). This is v2.

### Architecture

```
CEO → PM → UI/UX Lead → SWE Lead → Workers
```

Each handoff used a `NEXT_COMMAND:` marker written by the agent at the end of its comment. The server's `parseNextCommand()` reads it and automatically updates `pipeline_stage` and changes the assignee.

### What worked well

- Simple and deterministic — no ambiguity about who speaks next
- `NEXT_COMMAND:` is easy to debug: if an agent forgets to write it, the pipeline stalls but the problem is obvious
- No complex new server logic — everything lives in agent instructions

### Problems that emerged

**Sycophancy problem.** PM could write a technically infeasible brief, and SWE Lead would execute it without pushback. No formal mechanism for SWE Lead to say "this can't be done because X." The only escape was `request_confirmation` to a human, which interrupts the pipeline.

**No spec validation.** Ambiguous briefs and immature specs went straight to implementation. Bugs and rework were expensive.

---

## v3 — Roundtable Debate / MAD Asimetris (2026-05-05)

### Why it was created

After v2 ran on several test issues, the same failure pattern kept appearing: PM writes a brief → SWE Lead executes immediately → issues discovered in the implementation stage, too late. A validation gate was needed.

Step 12 implemented **MAD Asimetris** (Multi-Agent Debate, asymmetric) — SWE Lead becomes a mandatory validator before spec can advance.

### Architecture

```
CEO [TASK] + [NEEDS_DESIGN: yes/no]
  ↓
PM [PM BRIEF v1]
  ↓
SWE Lead: [CONFIRMED] (≥2 verification questions) OR [CONCERNS] (≥2 enumerated)
  ↓ if [CONCERNS]
PM [PM BRIEF v2]  (max 2 revisions)
  ↓ if still [CONCERNS] after 2 revisions
CEO [DECISION: ...]
  ↓ if [CONFIRMED]
UI/UX Lead [UI/UX SPEC]  (only if NEEDS_DESIGN: yes)
  ↓
SWE Lead [LGTM] → pipeline advance automatic
```

Routing is handled by `debate-router.ts` on the server — not the agents deciding who goes next, but the server reading signal tags and updating `assignee_agent_id`.

### Key design decisions in v3

**"Discussion Board" = the existing comment thread.** Storing discussion state in a JSONB column (`issues.discussion_board`) was considered, but the decision was to use the existing comment thread. Server scans comments to derive state. No migration risk, no new columns.

**SWE Lead reports to CEO, not UI/UX.** In v2, SWE Lead was under UI/UX in the reporting line. In v3, SWE Lead needs to escalate directly to CEO for deadlock resolution. So `reportsTo` was changed to `ceo`.

**`≥2 items` enforced at server level.** Agent instructions alone aren't enough — LLMs can forget or shortcut. `[CONFIRMED]` or `[CONCERNS]` without ≥2 enumerated items is rejected HTTP 400.

**UI/UX is no longer mandatory.** In v2, UI/UX was always invoked. In v3, CEO flags `[NEEDS_DESIGN: yes/no]` upfront. If `no`, UI/UX is skipped — faster pipeline for backend/data tasks.

### Problems discovered after v3 shipped

**Auto-wakeup doesn't work.** After `debate-router.ts` changes `assignee_agent_id` in the database, the newly assigned agent doesn't automatically wake up. Paperclip only fires a wakeup event when a **new comment** arrives on an issue — not when the assignee changes. So every routing step required a human to post a manual comment. Not scalable.

This drove Step 13.

---

## v4 — Auto-Wakeup + URS-First (2026-05-10, Planned)

### Why it was created

Two gaps identified after v3 was verified:

1. **Auto-wakeup gap.** Every debate step required human intervention. You can't call a pipeline autonomous if each agent needs to be manually "poked" to start.

2. **Kickoff is still freeform.** The pipeline always started from an ad-hoc task description written by a human. No standard way to begin from a formal requirements document (URS). Every project required dedicated PM time to "translate" requirements into a format Shannon could execute.

Both gaps are resolved in v4 without changing the core v3 debate flow.

### What changes in v4

**Step 13 — Server posts a system comment as the wakeup trigger.**

Every time `debate-router.ts` completes a routing decision, the server immediately inserts a new row in `issue_comments` with the body `[System — Pipeline] <tailored instruction for the next agent>`. This new comment fires the Paperclip wakeup event. The agent wakes up, reads the system comment, knows what to do, and starts working immediately.

Minimal change: just adds a `systemComment` field to `RoutingResult` type + one insert statement. Routing logic is unchanged.

**Step 14 — URS-first ingestion lane.**

Five new skills are added to PM (and one to CEO):

```
CEO: foundation--urs-draft   → draft a URS from raw brief
PM:  foundation--urs         → compile urs/main.md → index.json
PM:  foundation--sprint-plan → bin-pack FRs into sprints (complexity formula from ADR 0003)
PM:  urs--create-issues      → POST one Paperclip issue per Sprint 0 FR via API
PM:  foundation--shape-spec  → extended with --from-urs mode (reads spec from index.json)
```

New flow:

```
Human creates 1 kickoff issue (paste URS or brief)
→ CEO structures/copies to urs/main.md
→ PM compiles + sprint plan + creates N FR issues
→ Each FR issue auto-wakes PM → PM runs shape-spec --from-urs
→ Normal v3 debate per FR
```

From 1 kickoff issue, the pipeline generates N Sprint 0 issues automatically and immediately begins spec debate — no human intervention after the initial kickoff.

### Key design decisions in v4

**New company (v4), not a hot-update to v3.**

Agent instructions are baked in at company creation time. An existing v3 company won't know about the URS-first flow or Autonomous Continuation. A new template `shannon-v4-company.md` is created — v3 remains usable for projects that don't need the URS lane.

**v4 is a superset of v3, not a replacement.**

Everything in v3 (debate router, signal tags, validation gate, deadlock escalation) is unchanged. v4 only *adds* — no breaking changes. Projects already running on v3 can stay on v3.

**`urs--create-issues` is Shannon-specific — no `ai-software-factory` equivalent.**

In the factory, commands write output to the local filesystem. In Shannon, output must become Paperclip issues via API. This skill is written from scratch, not ported.

---

## Summary Comparison

| | v2 | v3 | v4 |
|---|---|---|---|
| Step introduced | Step 3 | Step 12 | Steps 13–14 |
| Date | 2026-04-27 | ~2026-05-05 | Planned 2026-05-10 |
| Handoff mechanism | `NEXT_COMMAND:` | Signal tags + `debate-router.ts` | Same as v3 |
| SWE validation gate | ❌ | ✅ | ✅ |
| Auto-wakeup | ❌ | ❌ | ✅ (system comment) |
| URS-first lane | ❌ | ❌ | ✅ (5 new skills) |
| Human touches per spec stage | 1 per handoff | 1 per routing step (bug) | 0 (fully autonomous spec stage) |
| Server complexity | Low | Medium | Medium + system comment insert |
| PM skill count | 5 | 5 | 8 |

---

## Lessons from Each Version

**v2 → v3:** Agents that execute specs without validation are a liability. Adding one validation gate has a real cost (more comments, more round-trips) but the benefit is larger: specs that are more mature before entering implementation.

**v3 → v4:** Autonomous doesn't just mean "agents execute" — it means "no human in the loop." Auto-wakeup is a prerequisite for truthfully calling this pipeline autonomous. URS-first is a natural extension: if the pipeline can self-execute, more structured input (URS) produces better, more traceable output.

---

## Related

- [[Shannon Company Templates — v2 vs v3]] — feature comparison table
- [[Shannon Company Templates — v4]] — v4 detailed design
- [[Step 13 — Auto-Wakeup Fix Plan]] — auto-wakeup implementation
- [[Step 14 — URS Skills Port Plan]] — URS skills implementation
- [[Roundtable Discussion Architecture]] — MAD Asimetris spec implemented in v3/v4
- [[Paperclip Shannon — Modification Plan Phase 2]] — Phase 2 implementation order
