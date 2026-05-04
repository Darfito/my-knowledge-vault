---
title: "Paperclip as White-Label Base — Strategic Assessment"
type: architecture-decision
status: decided
tags: [factory-building, paperclip, strategy, white-label, architecture]
date: 2026-04-29
---

# Paperclip as White-Label Base — Strategic Assessment

> **Verdict: Yes — correct strategic direction.** The risk is not strategic but operational (fork maintenance).

---

## Context

This document captures the validation of using Paperclip (forked as `paperclip-shannon`) as the white-label base for the AI Software Factory product. Evaluated against vault context and external research as of 2026-04-29.

---

## Why Paperclip is the Right Base

| Reason | Detail |
|---|---|
| **Right primitives** | Org chart, skills, tickets, approval gates, budgets — exactly what a software factory needs |
| **MIT licensed** | Freely forkable, white-labelable, no licensing risk |
| **Multi-tenancy native** | One deployment, multiple client companies with full data isolation = B2B model |
| **Company config export/import** | Shannon templates can be shipped as a one-click package (Clipmart feature incoming) |
| **Framework agnostic** | Workers can be any language/model — not locked to Python like CrewAI/AutoGen |
| **UI included** | Dashboard, board, audit logs — not built from scratch |

---

## What Shannon Adds on Top of Vanilla Paperclip

This is the core value-add — not selling "Paperclip", but the pipeline and template on top of it.

| Shannon Addition | What It Does |
|---|---|
| **Orchestrator-Worker pipeline** | Haiku workers for execution (10× cheaper), Sonnet orchestrators for judgment. Upstream Paperclip has no concept of this. |
| **Pipeline stages** | `spec → design → implementation → sandboxed → tested` with `NEXT_COMMAND:` auto-advancement. Upstream is a general task board with no software delivery lifecycle. |
| **Shannon company template** | Opinionated 4-agent structure (CEO → PM → UI/UX Lead → SWE Lead) pre-wired with skills. Client gets a working factory, not a blank org chart. |
| **Structured human checkpoint** | `request_confirmation` gate at `sandboxed → tested`. Upstream approval system was designed for hiring, not code review gates. |

---

## Comparison with Alternatives

| Platform | Their Bet | Why Paperclip Wins for This Use Case |
|---|---|---|
| **LangGraph** | Graph-based workflow for developers; great for branching/conditional logic | No UI, no multi-tenancy, no company metaphor, Python-centric internals |
| **CrewAI / AutoGen** | Python agent coordination | No dashboard, no budgets, no client-facing product layer |
| **Raw Claude Code / API** | Bare API, you build everything | No structure, no cost tracking, no board for non-technical operators |
| **Custom-built** | Full control | 6–12 months to reach Paperclip feature parity on board UI alone |

Paperclip is the only open-source platform that frames agents as a *company structure* rather than a *workflow graph.* That framing is what makes it naturally white-labelable for a software factory product.

---

## Current State of the Fork

All 10 modification steps are complete in `paperclip-shannon` (as of 2026-04-25 to 2026-04-30):

| Step | What | Status |
|---|---|---|
| 1 | `pipeline_stage` column + migration | ✅ Done |
| 2 | `NEXT_COMMAND:` pipeline trigger wired into comment handler | ✅ Done |
| 3 | Shannon company bootstrap skill (opinionated 4-agent structure) | ✅ Done |
| 4 | Worker cost reporting to `/api/cost-events` | ✅ Done |
| 5 | Review checkpoint using native `request_confirmation` | ✅ Done |
| 6 | Supabase CLI automation in foundation pipeline | ✅ Done |
| 7 | Sandbox skills with Playwright (up/down/test/status) | ✅ Done |
| 8 | Company package fixes (template + PM foundation chain) | ✅ Done |
| 9 | Workers: Anthropic SDK + `WORKER_API_KEY` + `claude-sonnet-4-6` | ✅ Done |
| 10 | Company delete: 17 missing FK tables + correct delete order | ✅ Done |

All changes are **additive and isolated** — no breaking changes to Paperclip's schema or API surface.

For full current state including skills inventory and open items, see [[Paperclip Shannon — Current Condition 2026-05-04]].

---

## The Real Risk: Fork Maintenance

Paperclip launched March 2026 and hit 38k GitHub stars rapidly. A fast-moving upstream creates fork drift risk.

**Why this matters:**
- Upstream breaking changes are likely as the project matures
- `paperclip-shannon` will diverge; merging upstream gets painful over time
- Open questions (skills manager, sandbox ports, milestone map) may be resolved upstream in conflicting ways

**Mitigation:**
- Keep all Shannon changes additive and isolated (already done correctly)
- Watch upstream `schema/` and `routes/` specifically — those are where the fork diffs live
- Plan a monthly upstream sync cadence before the delta becomes unmanageable

---

## Bottom Line

Using Paperclip as the white-label base is the right bet **because** the product is not "Paperclip" — it is the software delivery lifecycle, Shannon templates, and orchestrator-worker cost model that sit on top of it. Paperclip provides the company UI and multi-tenancy layer for free.

The risk is operational, not strategic. Active fork maintenance is the only ongoing cost.

---

## Related
- [[Paperclip Shannon — Modification Plan]] — all 5 implementation steps
- [[Paperclip Compatibility — Orchestrator-Worker]] — confirms no breaking changes needed
- [[Orchestrator-Worker Architecture]] — the pipeline architecture this base supports
- [[Paperclip Integration Unknowns]] — open questions to resolve
