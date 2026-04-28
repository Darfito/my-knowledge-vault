---
title: "Personas & Skills Matrix"
type: reference
tags: [factory-building, personas, skills, paperclip]
source: ai-software-factory-conversation.md
date: 2026-04-25
---

# Personas & Skills Matrix

MVP roster: **CEO, PM, UI/UX, SWE Lead, plus Code / Design / Test Workers.**

Personas are classified by execution type — see [[Orchestrator-Worker Architecture]] for the full rationale. Full Agents and Light Agents are managed by Paperclip. Workers are API calls dispatched by their parent Agent's skills; Paperclip has no visibility into them.

No dedicated Reviewer persona — the SWE Lead + Worker output review covers it. A separate Reviewer adds overhead without clear value at MVP scale. (`/implementation:review` is a skill the SWE Lead runs on collected Worker output.)

---

## Skills per Persona

| Persona | Type | Model | Global Skills | Project Skills | External Skills |
|---|---|---|---|---|---|
| CEO | Full Agent | Sonnet / Opus | — | — | `docx`, `pptx` (strategy docs/briefings) |
| PM | Full Agent | Sonnet | `foundation:inject-standards` | `foundation:discover`, `foundation:plan`, `foundation:status`, `foundation:shape-spec`, `foundation:validate` | `docx` (specs), `xlsx` (backlog exports) |
| UI/UX | Light Agent | Sonnet | `foundation:inject-standards` | `design:import`, `design:system` | `pdf-reading` (Figma exports), `docx`/`pdf` (design docs) |
| SWE Lead | Full Agent | Sonnet / Opus | `architecture:review`, `implementation:review`, `data-fetching:review`, `foundation:inject-standards`, `sandbox:up/down/test/status`, `worker:dispatch` | `architecture:new-feature`, `implementation:new-feature` | Code-review / git skills from skills.sh |
| **Code Worker** | **Worker** | **Haiku** | — | Writes one component/module per call | — |
| **Design Worker** | **Worker** | **Haiku** | — | Generates CSS/Tailwind/tokens from design spec | — |
| **Test Worker** | **Worker** | **Haiku** | — | Writes unit/integration tests for a specific module | — |

---

## Skill Assignment Mechanism

Paperclip's **skills manager** handles per-persona assignment for Agents. Each agent loads only the skills assigned to its role — prevents context bloat and keeps each agent focused on its lane.

Workers do not use the skills manager. They receive a single system prompt and task message directly from the dispatching Agent's `worker:dispatch` skill.

---

## Deferred Personas

**DevOps** — handles `deployment:release → deployment:k8s-config` chain. Deferred to **Phase 3+** because production deployments stay out of Paperclip's scope until then.

**QA Agent** — in the Orchestrator-Worker model, QA test generation is handled by Test Workers dispatched by the SWE Lead. A dedicated QA Agent (for exploratory/regression testing strategy) remains on the backlog.

---

## Related
- [[Lifecycle - Project vs Feature]] — which persona executes which step
- [[Pipeline Model & Ticket Strategy]] — how tickets are routed between personas
- [[Skill Naming Convention]] — how skill names are structured to avoid collisions
