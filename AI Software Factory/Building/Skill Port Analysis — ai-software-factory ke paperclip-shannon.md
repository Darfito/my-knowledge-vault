---
title: "Skill Port Analysis — ai-software-factory → paperclip-shannon"
type: analysis
status: complete
tags: [factory-building, skills, paperclip, porting, quality]
date: 2026-04-27
---

# Skill Port Analysis — ai-software-factory → paperclip-shannon

> **Source:** `D:\Side-Mission\project-2\refrensi\ai-software-factory\.claude\commands\`
> **Target:** `D:\Side-Mission\project-2\refrensi\labs\paperclip-shannon\skills\`

All 13 skills ported on 2026-04-27. `paperclip-shannon/skills/` now contains the full factory skill set alongside the existing Paperclip-native skills.

---

## Why Skills Matter Even Though Paperclip Is Already Agentic

Paperclip makes agents capable of acting autonomously — wakeups, issue assignment, pipeline advancement, NEXT_COMMAND parsing. That is already in place.

What skills add is **how to act consistently**. These are two different problems:

> **Being agentic** = the agent can take action on its own.
> **Having skills** = the agent follows a specific, repeatable procedure when it acts.

Without skills, an agent relies entirely on Claude's general knowledge. The output may be correct, but it is non-deterministic — the same task run twice may produce two different structures, miss a required field, or ignore a project-specific convention.

With skills, the procedure is explicit and encoded. The agent is not improvising from general knowledge — it is following a defined sequence of steps that enforce the project's specific standards.

### The surgical protocol analogy

A surgeon is already capable of operating independently (agentic). Surgical protocols do not make the surgeon capable — they ensure that capability is exercised in the right order, without skipping steps, even under pressure or in unfamiliar situations.

Skills are the surgical protocols of the factory pipeline.

### Concrete example: PM writing a spec

| | Without `foundation--shape-spec` | With `foundation--shape-spec` |
|---|---|---|
| **Format** | Whatever Claude decides looks good | Follows `_template.md` every time |
| **Multi-tenancy** | May or may not mention tenant isolation | Always adds "Tenant A cannot access Tenant B's data" when `multiTenant: true` |
| **Data shape** | May describe it in prose | Always includes typed fields with index and RLS notes |
| **Output location** | Not saved, or saved to a random path | Always saved to `.claude/docs/specs/{feature-name}.md` |
| **Pipeline state** | Board has no visibility | `project-state.md` updated with timestamp |

### Why this matters downstream

The pipeline is sequential. If the PM spec is inconsistent, the UI/UX design spec is built on shaky ground. If the design spec has arbitrary tokens, the workers produce inconsistent CSS. If the workers produce inconsistent code, the SWE Lead's review checklist fails to catch it because there is no baseline to compare against.

Skills solve this by making each stage's output **predictable input** for the next stage. The pipeline becomes a chain of defined contracts, not a chain of best-effort attempts.

### The general knowledge ceiling

Claude's general knowledge of Next.js, Supabase, and RLS is good — but it is generic. It does not know:

- That **this project** uses `SECURITY INVOKER` on all RPCs
- That **this project's** actions must return `ActionResult<T>` and never throw
- That **this project** has a `riskZones` config that determines how strict a review must be
- That **this project** already established a `src/features/{domain}/` structure in the first feature

A skill reads `project-config.json` and the project's own standards docs. It applies **project-specific** knowledge, not generic Claude knowledge. That is the gap that cannot be closed by writing better agent instructions alone.

---

## Conversion Map

| Source command | Target skill | Agent | Status |
|---|---|---|---|
| `foundation/inject-standards.md` | `foundation--inject-standards` | All agents | ✅ done |
| `foundation/discover.md` | `foundation--discover` | PM | ✅ done |
| `foundation/plan.md` | `foundation--plan` | PM | ✅ done |
| `foundation/shape-spec.md` | `foundation--shape-spec` | PM | ✅ done |
| `foundation/validate.md` | `foundation--validate` | PM | ✅ done |
| `foundation/status.md` | `foundation--status` | PM | ✅ done |
| `design-os/import.md` | `design--import` | UI/UX Lead | ✅ done |
| `design-os/system.md` | `design--system` | UI/UX Lead | ✅ done |
| `architecture-os/new-feature.md` | `architecture--new-feature` | SWE Lead | ✅ done |
| `architecture-os/review.md` | `architecture--review` | SWE Lead | ✅ done |
| `implementation-os/new-feature.md` | `implementation--new-feature` | SWE Lead | ✅ done |
| `implementation-os/review.md` | `implementation--review` | SWE Lead | ✅ done |
| `data-fetching-os/review.md` | `data-fetching--review` | SWE Lead | ✅ done |

**Not ported (Phase 3+ or one-time setup):**
- `foundation/init.md` — project setup, run manually once per project
- `deployment-os/k8s-config.md`, `deployment-os/release.md` — Phase 3+
- `qa-os/new-tests.md`, `qa-os/fix.md` — handled by Test Workers

---

## Conversion approach

Each slash command was adapted to the Paperclip runtime with these changes:

1. Added SKILL.md frontmatter (`name`, `description`)
2. Replaced "Ask the user: ..." → "Read from issue context / existing files in `cwd`; post a clarifying comment if info is missing"
3. Replaced "Run `/other:command` next" → "Post a comment suggesting the next skill to run"
4. Removed `COMMAND_COMPLETE` blocks (not used in Paperclip)
5. Replaced "Tell the user / confirm before writing" → "Write directly; post a summary comment on the issue"

The technical content — checklists, SQL templates, file structure patterns, zone rules — was preserved verbatim.

---

## What Does NOT Get Ported to paperclip-shannon

The content under `.claude/docs/` in `ai-software-factory` is **not ported** — it is project-specific standards that travel with the project template, not with the Paperclip runtime:

| Directory | Contents | Lives in |
|---|---|---|
| `.claude/docs/architecture-os/` | Schema conventions, RPC standards, API contracts | Project directory (agent `cwd`) |
| `.claude/docs/design-os/` | Design system tokens, screen specs | Project directory |
| `.claude/docs/foundation/` | Product mission, tech standards, auth model | Project directory |
| `.claude/docs/implementation-os/` | Implementation standards | Project directory |
| `.claude/docs/qa-os/` | Testing strategy, checklists | Project directory |
| `.claude/project-config.json` | Multi-tenant flag, auth model, regulated status | Project directory |
| `.claude/docs/standards-index.yml` | Index used by inject-standards | Project directory |

Skills know **how to read** these files. The content depends on the specific project being built.

---

## What improves with the 13 skills

### Quality gates become structural

| Gate | Enforcing skill | Guarantee |
|---|---|---|
| Every migration has RLS | `architecture--new-feature` | RLS in the same migration, never as a separate step |
| Every action has an auth check | `implementation--new-feature` | `requireAuth()` is always the first call |
| Input is always validated | `implementation--new-feature` | `safeParse()` before any database call |
| No raw throws from actions | `implementation--review` | Return `ActionResult<T>` — never throw |
| No `any` types | `implementation--review` | Type safety checked per file |
| Consistent design tokens | `design--system` | Tokens sourced from the real design system |

### Pipeline tracking via `project-state.md`

Every skill updates `project-state.md` with a stage timestamp. The board operator can read one file to see the status of every feature. No feature gets lost in the pipeline without a trace.

### Overall improvement

| Dimension                 | Without skills                           | With skills                                                           |
| ------------------------- | ---------------------------------------- | --------------------------------------------------------------------- |
| **Output consistency**    | Depends on Claude's general knowledge    | Deterministic, follows project standards                              |
| **Quality gates**         | Model must remember the standards        | Baked into every step of every skill                                  |
| **Traceability**          | None                                     | `project-state.md` updated automatically                              |
| **Worker prompt quality** | SWE Lead dispatches based on assumptions | SWE Lead dispatches based on spec + API contract + official migration |

---

## Related
- [[Personas & Skills Matrix]] — full skills list per persona
- [[Paperclip Shannon — Modification Plan]] — modification steps
- [[Orchestrator-Worker Architecture]] — the architecture being implemented
