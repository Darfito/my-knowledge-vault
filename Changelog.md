---
title: "Changelog — Document Registry"
type: navigation
tags: [index, changelog, navigation]
date: 2026-04-27
---

# Changelog — Document Registry

All vault documents sorted by **last updated date, newest first**. Update this file whenever you create or significantly revise a document.

> **How to use:** Scan from the top to catch what's new. Each entry has a one-line summary of what the document covers so you can judge relevance without opening it.

---

## 2026-04-28

| Document | What it covers |
|---|---|
| [[AI Software Factory/Using/Agent Capabilities vs Skills vs Instructions]] | New — explains difference between AGENTS.md, skills, and the capabilities display field; includes Shannon agent stack table |

---

## 2026-04-27

| Document | What it covers |
|---|---|
| [[AI Software Factory/Building/Paperclip Shannon — Modification Plan]] | All 5 steps ✅ complete — pipeline_stage, NEXT_COMMAND, company-creator overhaul, worker--dispatch skill, request_confirmation checkpoint |
| [[AI Software Factory/Building/Skill Port Analysis — ai-software-factory ke paperclip-shannon]] | ✅ complete — all 13 skills ported; includes "agentic ≠ procedural" reasoning |
| [[Development Log/2026-04-27]] | Session log — Steps 3–5 completed, 13 skills ported, vault translated to English |
| [[VPS-Shannon/shannon-company-setup]] | New — complete setup guide: what the Shannon company is, import steps, access grant, day-to-day pipeline usage |
| [[VPS-Shannon/paperclip-dev-setup]] | Expanded "No company access" troubleshooting — added `/instance/settings/access` recipe for granting UI-created accounts access without the invite flow |
| [[VPS-Shannon/README]] | Translated to English |
| [[VPS-Shannon/cheatsheet]] | Translated to English |
| [[VPS-Shannon/claude-credentials-setup]] | Translated to English |
| [[VPS-Shannon/paperclip-dev-setup]] | Translated to English |
| [[VPS-Shannon/santoso-protocol-project]] | Translated to English |
| [[VPS-Shannon/k3s-kubernetes]] | Translated to English |
| [[VPS-Shannon/services-deployment]] | Translated to English |

---

## 2026-04-25

| Document | What it covers |
|---|---|
| [[AI Software Factory/Building/Paperclip Shannon — Modification Plan]] | Step-by-step plan; Steps 1 & 2 ✅ done — pipeline_stage column + NEXT_COMMAND: trigger implemented and typechecked |
| [[AI Software Factory/Building/Reference — dispatch skill]] | Design reference for `worker:dispatch` — what to borrow (fan-out, parallel collect) and what not to (full sessions, IPC) |
| [[AI Software Factory/Building/Paperclip Compatibility — Orchestrator-Worker]] | What in Paperclip needs to change (almost nothing) for the worker model; 2 additive items |
| [[AI Software Factory/Building/Orchestrator-Worker Architecture]] | Full architecture: agent vs worker classification, pipeline diagram, human checkpoints, token savings |
| [[AI Software Factory/Building/Personas & Skills Matrix]] | Updated with Type + Model columns; Code/Design/Test Workers added; QA moved to backlog |
| [[AI Software Factory/Vision/Vision & Roadmap]] | Updated "What the Factory Is" to reflect Orchestrator-Worker model |

---

## 2026-04-22

| Document | What it covers |
|---|---|
| [[AI Software Factory/Vision/Product Feedback]] | Token visibility feedback; session-end token reporting fix |
| [[AI Software Factory/Building/Pipeline Model & Ticket Strategy]] | Why single-ticket sequential stage model (Option 2) was chosen over fan-out |
| [[AI Software Factory/Building/Lifecycle - Project vs Feature]] | Two-tier cycle: per-project (once) vs per-feature (repeated) command chains |
| [[AI Software Factory/Building/Secret Injection Strategy]] | MVP file-based `.env.sandbox`; Phase 2 pgsodium encrypted store design |
| [[AI Software Factory/Building/Skill Naming Convention]] | Double-dash namespace convention; collision-check script requirement (M1.8) |
| [[AI Software Factory/Building/Test Command Convention]] | How `sandbox:test` resolves which test command to run from `package.json` |
| [[Open Questions/Paperclip Integration Unknowns]] | Blocking questions: ticket reassignment, NEXT_COMMAND triggers, skills manager, sandbox ports, milestone map |
| [[Open Questions/Factory Design Questions]] | Broader design questions: headless layer, mobile protocol, webapp/mobile divergence, end-to-end happy path |
| [[Action Items/MVP Sprint]] | Prioritized, session-sized tasks for the MVP sprint |
| [[Action Items/Phase 2 Backlog]] | Intentionally deferred items: secrets, back-edge support, Blueprint governance metrics |
| [[Development Log/2026-04-22]] | Session log for 2026-04-22 |

---

## 2026-04-18

| Document | What it covers |
|---|---|
| [[Development Log/2026-04-18]] | Session log for 2026-04-18 — initial vault setup |

---

## How to Update This File

When you (or Claude) create or significantly update a document:
1. Add a row to the correct date block (newest first)
2. If the date block doesn't exist yet, add a new `## YYYY-MM-DD` heading
3. Keep the one-liner tight — what does the doc decide or describe?

You do not need to list every minor edit. Only log:
- New documents
- Decisions that changed (e.g., architecture shift, new constraint discovered)
- Status changes (e.g., a question got resolved, an action item was completed)
