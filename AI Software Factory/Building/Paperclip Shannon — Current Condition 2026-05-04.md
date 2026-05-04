---
title: "Paperclip Shannon — Current Condition 2026-05-04"
type: state-snapshot
status: current
tags: [factory-building, paperclip, state, condition, snapshot]
date: 2026-05-04
---

# Paperclip Shannon — Current Condition (2026-05-04)

> **Purpose.** Single-page snapshot of the `paperclip-shannon` fork as it stands today. Read this before starting a new session so you don't re-derive what's already been built.

---

## Repository Facts

| Item | Value |
|---|---|
| Path on VPS | `/home/paperclip/paperclip-shannon` |
| Branch | `main` |
| Head commit | `214171a4` — fix: make NewProjectDialog body scrollable (2026-04-30) |
| Fork base | Paperclip V1 (April 2026 vintage) |
| Owner | `paperclip` user on VPS (84.247.150.72) |
| Dev server | `http://84.247.150.72:3100` — run as `pnpm dev --bind lan` |

---

## Modification Steps Status

All 10 Shannon modification steps are **complete**.

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

See [[Paperclip Shannon — Modification Plan]] for full details per step.

---

## Pipeline Stages

Issues move through these stages automatically via `NEXT_COMMAND:` markers:

```
spec → design → implementation → sandboxed → tested → shipped
```

- Stages stored in `issues.pipeline_stage` (nullable — null = unmanaged issue)
- Stage labels defined in `packages/shared/src/constants.ts` → `PIPELINE_STAGE_LABELS`
- `tested → shipped` requires human approval via `request_confirmation` checkpoint

---

## Skills Inventory (30 total)

### Standalone skills (`skills/`) — 22 skills

**Foundation pipeline (Phase 1–3):**

| Skill | Purpose |
|---|---|
| `foundation--discover` | Project discovery, Supabase setup, `.env.local` write |
| `foundation--plan` | Architecture planning, ADR generation |
| `foundation--shape-spec` | Feature spec from user intent |
| `foundation--inject-standards` | Inject coding standards into project context |
| `foundation--status` | Read current pipeline/project status |
| `foundation--validate` | Validate foundation: Supabase running, no placeholder envs |

**Architecture (Phase 4):**

| Skill | Purpose |
|---|---|
| `architecture--new-feature` | Schema design, migrations, API contracts; runs `supabase db reset` |
| `architecture--review` | Architecture review against non-negotiables |

**Design (Phase 4):**

| Skill | Purpose |
|---|---|
| `design--import` | Import design tokens |
| `design--system` | Design system audit |

**Data fetching:**

| Skill | Purpose |
|---|---|
| `data-fetching--review` | TanStack Query / Server Action audit |

**Implementation (Phase 5):**

| Skill | Purpose |
|---|---|
| `implementation--new-feature` | Feature build following approved spec |
| `implementation--review` | Code review against non-negotiables |

**Sandbox lifecycle:**

| Skill | Purpose |
|---|---|
| `sandbox--up` | Start dev server + Supabase; install Playwright if needed; save PID to `.sandbox/dev.pid` |
| `sandbox--down` | Kill dev server via saved PID; fallback to port-based kill |
| `sandbox--test` | Run `pnpm test:run` + Playwright screenshots; agent reads PNGs via vision |
| `sandbox--status` | Read-only health check for Supabase stack + dev server |

**Worker dispatch:**

| Skill | Purpose |
|---|---|
| `worker--dispatch` | Python `asyncio.gather()` dispatch; workers use `WORKER_API_KEY` + `claude-sonnet-4-6`; cost reported to Paperclip |

**Meta/tooling:**

| Skill | Purpose |
|---|---|
| `paperclip` | Paperclip platform reference |
| `paperclip-create-agent` | Create a new Paperclip agent |
| `paperclip-create-plugin` | Create a new Paperclip plugin |
| `para-memory-files` | Para memory file management |

### Agent skills (`.agents/skills/`) — 8 skills

| Skill | Purpose |
|---|---|
| `company-creator` | Bootstrap a Shannon factory company (opinionated 4-agent template) |
| `create-agent-adapter` | Create a Paperclip agent adapter |
| `deal-with-security-advisory` | Handle security advisory workflow |
| `doc-maintenance` | Documentation maintenance |
| `pr-report` | PR status report |
| `prcheckloop` | PR check loop |
| `release` | Release workflow |
| `release-changelog` | Changelog generation for release |

---

## Auth Model

| Role | Auth | Why |
|---|---|---|
| CEO, PM, UI/UX Lead, SWE Lead | Subscription (`claude login`) | Orchestrators — judgment + coordination only |
| Code Workers, Design Workers, Test Workers | `WORKER_API_KEY` + `claude-sonnet-4-6` | Raw `messages.create()` API calls |

`WORKER_API_KEY` is intentionally distinct from `ANTHROPIC_API_KEY` — Claude Code does not pick it up as orchestrator auth. The key can come from a completely different Anthropic account.

**Setup:**
1. `su - paperclip && claude login` (Max subscription)
2. Set `WORKER_API_KEY=sk-ant-...` on SWE Lead and UI/UX Lead via Paperclip UI → Configuration → Environment Variables

---

## Shannon Company Template

The canonical 4-agent company structure:

```
CEO → PM → UI/UX Lead → SWE Lead
```

Template file: `.agents/skills/company-creator/references/shannon-company.md`

Placeholders: `{PROJECT_NAME}`, `{project-slug}`, `{project-path}`

First generated company: `companies/shannon-factory-2/`

---

## Database Schema Additions (Shannon-specific)

| Addition | Location | What |
|---|---|---|
| `issues.pipeline_stage` | `packages/db/src/schema/issues.ts` | Nullable text column tracking pipeline position |
| Migration `0071_tired_hellion.sql` | `packages/db/src/migrations/` | `ALTER TABLE "issues" ADD COLUMN "pipeline_stage" text` |
| `PIPELINE_STAGES` constant | `packages/shared/src/constants.ts` | `["spec","design","implementation","sandboxed","tested","shipped"]` |
| `NEXT_COMMAND:` parsing | `server/src/routes/issues.ts` | `parseNextCommand()` — auto-advances stage + reassigns agent on comment |

---

## Recent Fix History

| Date | Commit | What |
|---|---|---|
| 2026-04-30 | `214171a4` | NewProjectDialog body scrollable (flex + max-h-[90vh]) |
| 2026-04-30 | `9d423414` | worker--dispatch: Anthropic SDK + WORKER_API_KEY + Sonnet |
| 2026-04-30 | `0ec9e693` | worker--dispatch: revert to `claude -p` (intermediate) |
| 2026-04-30 | `a76e9609` | Company delete: 17 missing FK tables added |
| 2026-04-30 | `c2944415` | Sandbox skills + canonical shannon-company template |
| 2026-04-29 | `f21f9ca4` | Supabase CLI automation in foundation--discover |
| 2026-04-29 | `d8bce61d` | Fix: migration 0071 stripped to pipeline_stage only |
| 2026-04-29 | `fe0185ae` | Fix: hidden DialogTitle for NewIssueDialog accessibility |

---

## Open Items (Remaining Work)

### Must do before first live production run

1. **End-to-end pipeline test** — No live test of the full `spec → shipped` pipeline has been run yet. Create a test company, create a real issue, walk it through all stages manually.

2. **ngrok URL rotation** — `WEBHOOK_URL` in `.env` must be updated manually after every VPS restart. No script auto-reads the new ngrok URL. (See [[Main Index]] Gap #5)

3. **Telegram bridge as systemd service** — Currently requires manual `tmux bridge` start after every reboot. No auto-restart. (See [[Main Index]] Gap #3)

4. **Paperclip patches not automated** — `analyst` and `marketing` `AGENT_ROLES` enum patches must be re-applied manually after every `paperclipai` reinstall. (See [[Main Index]] Gap #4)

### Lower priority

5. **`webapp-gunawan/.claude/agents/` still missing** — Quinn's bootstrap `cp` will fail on Gunawan stack. Five-minute fix. (See [[Main Index]] Gap #1)

6. **No `cwd` pinned for non-SWE agents** — CEO, PM, UI/UX Lead, and 3 of 7 Santoso agents have no working directory set. Non-determinism risk.

7. **Santoso `status=error` in Paperclip** — The Santoso Protocol company shows error status. Needs log investigation before any Santoso-related automation is trusted.

---

## What Works Right Now

- Paperclip dev server starts and runs on port 3100
- Company creation via `company-creator` skill produces valid Shannon company packages
- `pipeline_stage` field exists on issues and advances via `NEXT_COMMAND:`
- Worker dispatch skill dispatches parallel API calls with cost reporting
- Sandbox skills (up/down/test/status) manage dev server lifecycle
- Supabase CLI automation sets up local DB in Docker automatically
- Company delete cleans up all 17 associated tables cleanly
- Auth split: orchestrators on subscription, workers on separate API key

---

## Related

- [[Paperclip Shannon — Modification Plan]] — step-by-step implementation log
- [[Paperclip as White-Label Base — Strategic Assessment]] — why Paperclip, fork maintenance risk
- [[Orchestrator-Worker Architecture]] — the pipeline this implements
- [[Paperclip Agent Auth — Subscription vs API]] — auth setup guide
- [[VPS-Shannon/paperclip-dev-setup]] — how to run the server
- [[VPS-Shannon/shannon-company-setup]] — how to use the Shannon company
