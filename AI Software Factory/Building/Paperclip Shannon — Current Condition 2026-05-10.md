---
title: "Paperclip Shannon — Current Condition 2026-05-10"
type: state-snapshot
status: current
tags: [factory-building, paperclip, state, condition, snapshot]
date: 2026-05-10
supersedes: "Paperclip Shannon — Current Condition 2026-05-04"
---

# Paperclip Shannon — Current Condition (2026-05-10)

> **Purpose.** Single-page snapshot of the `paperclip-shannon` fork as it stands today. Read this before starting a new session so you don't re-derive what's already been built.

---

## Repository Facts

| Item | Value |
|---|---|
| Path on VPS | `/home/paperclip/paperclip-shannon` |
| Branch | `master` |
| Head commit | Post-Step-12 (exact hash TBD — verify with `git log --oneline -1` on VPS) |
| Fork base | Paperclip V1 (April 2026 vintage) |
| Owner | `paperclip` user on VPS (84.247.150.72) |
| Dev server | `http://84.247.150.72:3100` — run as `pnpm dev --bind lan` |

---

## Modification Steps Status

| Step | What | Status | Date |
|---|---|---|---|
| 1 | `pipeline_stage` column + `PIPELINE_STAGES` constant | ✅ Done | 2026-04-25 |
| 2 | `NEXT_COMMAND:` pipeline trigger in comment handler | ✅ Done | 2026-04-25 |
| 3 | Shannon company bootstrap skill (opinionated 4-agent) | ✅ Done | 2026-04-27 |
| 4 | Worker cost reporting to `/api/cost-events` | ✅ Done | 2026-04-27 |
| 5 | Review checkpoint via `request_confirmation` | ✅ Done | 2026-04-27 |
| 6 | Supabase CLI automation in foundation pipeline | ✅ Done | 2026-04-29 |
| 7 | Sandbox skills with Playwright (up/down/test/status) | ✅ Done | 2026-04-30 |
| 8 | Company package fixes (template + PM foundation chain) | ✅ Done | 2026-04-30 |
| 9 | Workers: Anthropic SDK + `WORKER_API_KEY` + Sonnet 4.6 | ✅ Done | 2026-04-30 |
| 10 | Company delete bug: 17 missing FK tables added | ✅ Done | 2026-04-30 |
| 11 | Workers subscription consolidation: `claude -p` default | ✅ Done | 2026-05-04 |
| 12 | Roundtable debate flow (MAD Asimetris) + `debate-router.ts` | ✅ Done | ~2026-05-05 |
| 13 | Auto-wakeup fix: system comment as wakeup trigger | 📋 Planned | — |
| 14 | URS-first skills port (5 skills + shape-spec extension) | 📋 Planned | — |

See [[Step 13 — Auto-Wakeup Fix Plan]] and [[Step 14 — URS Skills Port Plan]] for implementation details.

---

## Pipeline Stages

Issues move through these stages:

```
spec → design → implementation → sandboxed → tested → shipped
```

**In v2 companies:** Advance via `NEXT_COMMAND:` markers posted by agents.

**In v3/v4 companies (spec stage only):** Advance via `[LGTM]` signal from SWE Lead, routed by `debate-router.ts`. All other stages still use `NEXT_COMMAND:`.

- `tested → shipped` requires human approval via `request_confirmation` checkpoint
- Stages stored in `issues.pipeline_stage` (nullable — null = unmanaged issue)
- Stage labels defined in `packages/shared/src/constants.ts` → `PIPELINE_STAGE_LABELS`

---

## Company Templates

Three variants exist (bootstrapped via `company-creator` skill):

| Template | File | Spec stage | Auto-wakeup | URS lane |
|---|---|---|---|---|
| v2 | `shannon-company.md` | Linear `NEXT_COMMAND:` | ❌ Manual | ❌ |
| v3 | `shannon-v3-company.md` | Roundtable debate (`debate-router.ts`) | ❌ Manual | ❌ |
| v4 | `shannon-v4-company.md` | Roundtable debate (same as v3) | ✅ System comment | ✅ URS-first |

See [[Shannon Company Templates — v2 vs v3]] and [[Shannon Company Templates — v4]] for details.

---

## Skills Inventory (30 existing + 5 planned)

### Existing Standalone Skills (`skills/`) — 22 skills

**Foundation pipeline:**

| Skill | Purpose |
|---|---|
| `foundation--discover` | Project discovery, Supabase setup, `.env.local` write |
| `foundation--plan` | Architecture planning, ADR generation |
| `foundation--shape-spec` | Feature spec from user intent (v4: adds `--from-urs` mode) |
| `foundation--inject-standards` | Inject coding standards into project context |
| `foundation--status` | Read current pipeline/project status |
| `foundation--validate` | Validate foundation: Supabase running, no placeholder envs |

**Architecture:**

| Skill | Purpose |
|---|---|
| `architecture--new-feature` | Schema design, migrations, API contracts; runs `supabase db reset` |
| `architecture--review` | Architecture review against non-negotiables |

**Design:**

| Skill | Purpose |
|---|---|
| `design--import` | Import design tokens |
| `design--system` | Design system audit |

**Data fetching:**

| Skill | Purpose |
|---|---|
| `data-fetching--review` | TanStack Query / Server Action audit |

**Implementation:**

| Skill | Purpose |
|---|---|
| `implementation--new-feature` | Feature build following approved spec |
| `implementation--review` | Code review against non-negotiables |

**Sandbox lifecycle:**

| Skill | Purpose |
|---|---|
| `sandbox--up` | Start dev server + Supabase; install Playwright if needed |
| `sandbox--down` | Kill dev server via saved PID |
| `sandbox--test` | Run `pnpm test:run` + Playwright screenshots |
| `sandbox--status` | Read-only health check |

**Worker dispatch:**

| Skill | Purpose |
|---|---|
| `worker--dispatch` | `asyncio.gather()` dispatch; workers use `claude -p` subscription; cost reported to Paperclip |

**Meta/tooling:**

| Skill | Purpose |
|---|---|
| `paperclip` | Paperclip platform reference |
| `paperclip-create-agent` | Create a new Paperclip agent |
| `paperclip-create-plugin` | Create a new Paperclip plugin |
| `para-memory-files` | Para memory file management |

### Planned Skills for Step 14 (v4 only)

| Skill | Agent | Purpose |
|---|---|---|
| `foundation--urs-draft` | CEO | Structure a freeform brief into a proper `urs/main.md` |
| `foundation--urs` | PM | Compile `urs/main.md` → `index.json`, `applies-to.json`, `main.tex` |
| `foundation--sprint-plan` | PM | Bin-pack FRs into sprints using complexity formula |
| `urs--create-issues` | PM | POST one Paperclip issue per Sprint 0 FR via API |
| `foundation--shape-spec` (extend) | PM | Add `--from-urs FR-XX` mode; existing flow unchanged |

### Agent Skills (`.agents/skills/`) — 8 skills (unchanged)

| Skill | Purpose |
|---|---|
| `company-creator` | Bootstrap a Shannon factory company (v2, v3, or v4) |
| `create-agent-adapter` | Create a Paperclip agent adapter |
| `deal-with-security-advisory` | Handle security advisory workflow |
| `doc-maintenance` | Documentation maintenance |
| `pr-report` | PR status report |
| `prcheckloop` | PR check loop |
| `release` | Release workflow |
| `release-changelog` | Changelog generation for release |

---

## Auth Model

| Role | Auth | Mode |
|---|---|---|
| CEO, PM, UI/UX Lead, SWE Lead | Subscription (`claude login`) | Claude Code agent session |
| Workers (default) | Subscription (`claude login`) | `claude -p` subprocess — no extra billing |
| Workers (fallback) | `WORKER_API_KEY` + SDK | Set `USE_API_KEY=true` on SWE Lead — opt-in only |

---

## Database Schema Additions (Shannon-specific)

| Addition | Location | What |
|---|---|---|
| `issues.pipeline_stage` | `packages/db/src/schema/issues.ts` | Nullable text column tracking pipeline position |
| Migration `0071_tired_hellion.sql` | `packages/db/src/migrations/` | `ALTER TABLE "issues" ADD COLUMN "pipeline_stage" text` |
| `PIPELINE_STAGES` constant | `packages/shared/src/constants.ts` | `["spec","design","implementation","sandboxed","tested","shipped"]` |
| `NEXT_COMMAND:` parsing | `server/src/routes/issues.ts` | `parseNextCommand()` — auto-advances stage + reassigns agent on comment |
| `debate-router.ts` | `server/src/services/debate-router.ts` | Pure routing logic for roundtable debate signals (Step 12) |
| Signal validation gate | `server/src/routes/issues.ts` | Rejects `[CONFIRMED]` without ≥2 items; enforces signal grammar |

---

## Open Items (Blocking Production)

### Must do before first live production run

1. **Step 13 — Auto-wakeup fix.** Agents don't wake up after `debate-router.ts` reassigns them. Human has to post a manual comment to trigger the wakeup. Fix: server posts a system comment immediately after each routing decision. See [[Step 13 — Auto-Wakeup Fix Plan]].

2. **Step 14 — URS skills port.** URS-first kickoff lane not yet ported from `ai-software-factory`. See [[Step 14 — URS Skills Port Plan]].

3. **End-to-end pipeline test.** No live test of the full `spec → shipped` pipeline with v3/v4 debate flow has been completed.

4. **ngrok URL rotation.** `WEBHOOK_URL` in `.env` must be updated manually after every VPS restart.

5. **Telegram bridge as systemd service.** Currently requires manual `tmux bridge` start after every reboot.

6. **Paperclip patches not automated.** `analyst` and `marketing` `AGENT_ROLES` enum patches must be re-applied manually after every `paperclipai` reinstall.

### Lower priority

7. **`webapp-gunawan/.claude/agents/` still missing.** Quinn's bootstrap `cp` will fail on Gunawan stack.

8. **No `cwd` pinned for non-SWE agents.** CEO, PM, UI/UX Lead have no working directory set. Non-determinism risk.

9. **Santoso `status=error` in Paperclip.** Needs log investigation before any Santoso automation is trusted.

---

## Related

- [[Paperclip Shannon — Modification Plan]] — Phase 1 history (Steps 1–10)
- [[Paperclip Shannon — Modification Plan Phase 2]] — Phase 2 history (Steps 11–14)
- [[Step 13 — Auto-Wakeup Fix Plan]] — current active work
- [[Step 14 — URS Skills Port Plan]] — next up after Step 13
- [[Shannon Company Templates — v2 vs v3]] — company template comparison
- [[Shannon Company Templates — v4]] — v4 design
- [[VPS-Shannon/paperclip-dev-setup]] — how to run the server
- [[VPS-Shannon/shannon-company-setup]] — how to use the Shannon company
