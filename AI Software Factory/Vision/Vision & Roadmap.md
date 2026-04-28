---
title: "Vision & Roadmap"
type: planning
tags: [factory-vision, roadmap, strategy]
date: 2026-04-22
---

# AI Software Factory — Vision & Roadmap

> This folder captures **what we want the factory to become** — plans, roadmap, and strategic goals. For how we're currently building it, see [[AI Software Factory/Building/]].

---

## What the Factory Is

An autonomous pipeline that takes a product idea from human intent → spec → design → implementation → tested artifact, orchestrated by Paperclip with minimal human intervention.

The team follows an **Orchestrator-Worker model** — see [[Orchestrator-Worker Architecture]]:
- **Full Agents** (CEO, PM, SWE Lead): reasoning-heavy roles that orchestrate and review
- **Light Agent** (UI/UX): judgment-required but minimal context
- **Workers** (Code, Design, Test): single-call executors dispatched by agents for all repetitive "dirty work"

This distinction is driven by token efficiency: workers use Haiku with no history or tools, cutting execution-task costs by ~60–80% vs. running all roles as full agents.

---

## Phase Roadmap

### Phase 1 — MVP (In Progress)
- Per-project and per-feature lifecycle working end-to-end
- Single-ticket pipeline model (Option 2) in Paperclip
- File-based `.env.sandbox` secret injection
- Skill naming convention + collision-check script (M1.8)
- `sandbox:up/test/down/status` skills operational
- `pnpm test` convention enforced

### Phase 2
- Supabase-backed encrypted secret store (pgsodium)
- `secrets:set` skill for agents
- Admin dashboard for human-managed production secrets
- Structured back-edge support (QA/Engineer can bounce ticket back one stage)
- Risk Zones (Principle 1 from Blueprint Analysis) — per-path governance in `.claude/project-config.json`
- VCM (Verification Coverage Metric) as a measurable, tracked over time

### Phase 3+
- DevOps persona — `deployment:release → deployment:k8s-config`
- GMM + AAI tracking (`AAI ≤ VCM × GMM` invariant from Blueprint Analysis)
- Headless execution layer (job-queue + status protocol — prerequisite for PLP integration)
- Provenance signing / SBOM / structured audit log

---

## Strategic Targets

- **PLP Software Development Module** — wrap the working factory as a Software Development Module on NexusPLM. Target markets: Kalbe, Pharma, MedDev, Fintech. See `product-vision-plp-software-module.pdf`.
- **Santoso Protocol as empirical proof** — 7-agent team coordinated by Paperclip, <90 min, <$60, real product shipped. This validates the Blueprint's risk-proportional governance thesis.

---

## Open Strategic Questions

- When does the factory become a **callable service** (headless) vs an interactive Claude Code ritual?
- What is the exact integration surface between `NEXT_COMMAND:` markers and Paperclip's assignment triggers?
- At what scale does Option 2 (single ticket) break down and require moving to Option 3 (epic + sub-tickets)?

---

## Related
- [[Main Index]] — Section 5 (Gaps & Opportunities) is the canonical implementation gap list
- [[AI Software Factory/Building/]] — current implementation decisions
- [[Projects/]] — outputs produced by the factory
