---
title: "Paperclip Shannon — Modification Plan"
type: implementation-plan
status: complete
tags: [factory-building, paperclip, implementation, pipeline, workers]
date: 2026-04-25
---

# Paperclip Shannon — Modification Plan

> **Codebase:** `D:\Side-Mission\project-2\refrensi\labs\paperclip-shannon`
> **Based on:** Paperclip V1 (already forked — `paperclip-shannon` has pre-existing diffs from upstream)
> **Goal:** Adapt Paperclip to support the AI Software Factory's Orchestrator-Worker pipeline

---

## Baseline Findings (Read Before Implementing)

From reading the source code, these things already work without any changes:

| Feature                                                     | Status                    | Location                                 |
| ----------------------------------------------------------- | ------------------------- | ---------------------------------------- |
| Ticket `assigneeAgentId` reassignment                       | ✅ already works           | `issues.ts` schema + `updateIssueSchema` |
| `AGENT_ROLES` has `ceo`, `pm`, `designer`, `engineer`, `qa` | ✅ already correct         | `packages/shared/src/constants.ts`       |
| Worker cost reporting endpoint                              | ✅ already works           | `POST /companies/:companyId/cost-events` |
| `approvals.type` is free-text (not enum)                    | ✅ no schema change needed | `packages/db/src/schema/approvals.ts`    |

The only **missing** piece is the `pipeline_stage` field on issues and the `NEXT_COMMAND:` pipeline trigger.

---

## Step 1 — Add `pipeline_stage` to Issues ✅ DONE (2026-04-25)

**What:** Add a nullable `pipeline_stage` text column to the `issues` table. Not all issues are factory pipeline issues — null means "unmanaged issue, regular workflow."

**Status:** Typecheck passes clean across all 20 packages. Migration generated: `packages/db/src/migrations/0071_tired_hellion.sql` — key line: `ALTER TABLE "issues" ADD COLUMN "pipeline_stage" text;`

**Files changed:**

| File | What changed |
|---|---|
| `packages/db/src/schema/issues.ts` | Added `pipelineStage: text("pipeline_stage")` column |
| `packages/shared/src/constants.ts` | Added `PIPELINE_STAGES`, `PipelineStage`, `PIPELINE_STAGE_LABELS` |
| `packages/shared/src/index.ts` | Exported the three new constants/types |
| `packages/shared/src/validators/issue.ts` | Added `pipelineStage` to `createIssueSchema` + `updateIssueSchema` |
| `packages/db/src/migrations/0071_tired_hellion.sql` | Migration: `ALTER TABLE "issues" ADD COLUMN "pipeline_stage" text` |

---

## Step 2 — `NEXT_COMMAND:` Pipeline Trigger ✅ DONE (2026-04-25)

**What:** When an agent posts a comment containing a `NEXT_COMMAND:` marker, Paperclip automatically advances `pipeline_stage` and reassigns the issue to the next agent.

**Status:** Implemented. `parseNextCommand()` helper added to routes, wired after `svc.addComment()`. Activity log entry `issue.pipeline_advanced` emitted on every successful advance.

**Files changed:**

| File | What changed |
|---|---|
| `server/src/services/issues.ts` | Added `pipelineStage` to `issueListSelect` |
| `server/src/routes/issues.ts` | Added `parseNextCommand()` helper + wired into comment handler |

---

## Step 3 — Shannon Company Bootstrap Skill ✅ DONE (2026-04-27)

**What:** Overhaul `.agents/skills/company-creator/` to always create Shannon-style factory companies. Opinionated: no generic mode, fixed 4-agent structure with worker dispatch.

**Files changed:**

| File                                                           | What changed                                                                   |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `.agents/skills/company-creator/SKILL.md`                      | Rewritten as Shannon-specific; always creates CEO → PM → UI/UX Lead → SWE Lead |
| `.agents/skills/company-creator/references/example-company.md` | Replaced with Shannon factory example (Product Advisor)                        |
| `.agents/skills/company-creator/references/shannon-company.md` | New; canonical Shannon package template                                        |

---

## Step 4 — Worker Cost Reporting ✅ DONE (2026-04-27)

**What:** Created `worker--dispatch` skill with the exact POST payload for reporting worker token costs to Paperclip. Workers are single `messages.create()` calls dispatched in parallel by the SWE Lead.

**Key decisions:**
- Workers use `claude-haiku-4-5-20251001` (default; can be overridden to Sonnet for judgment-required tasks)
- `biller: "worker"` + `billingType: "worker_dispatch"` distinguishes worker costs from direct agent costs
- Cost reported per worker call (not aggregated) for per-task breakdown in the board dashboard
- Cost reporting failures are non-fatal (logged as warnings, execution continues)
- Dispatch script uses `asyncio.gather()` — all workers fire in parallel, results collected together

**Files changed:**

| File | What changed |
|---|---|
| `skills/worker--dispatch/SKILL.md` | New; full dispatch skill with Python script, worker prompt templates, and cost reporting |

---

## Step 5 — Review Checkpoint ✅ DONE (2026-04-27)

**What:** Pipeline checkpoint before advancing from `sandboxed` to `tested`. Uses Paperclip's native `request_confirmation` interaction — no server code changes needed.

**Key decisions:**
- Replaced the free-text CHECKPOINT comment approach with `request_confirmation` interaction
- Board sees a structured approval card (Accept / Reject with reason) in the issue thread
- `continuationPolicy: "wake_assignee"` wakes SWE Lead after board accepts
- On acceptance: SWE Lead posts `NEXT_COMMAND: stage=tested` to advance pipeline
- On rejection: SWE Lead reads rejection reason, fixes issues, re-runs sandbox, creates new checkpoint
- Idempotency key: `confirmation:{issueId}:checkpoint:sandboxed`

**Files changed:**

| File | What changed |
|---|---|
| `.agents/skills/company-creator/references/shannon-company.md` | Updated SWE Lead instructions to use `request_confirmation` instead of free-text comment |

---

## Step 6 — Supabase CLI Automation ✅ DONE (2026-04-30)

**What:** Agents now set up and start the local Supabase stack automatically via Docker. No cloud account needed, no tier limits.

**Key decisions:**
- Local Docker mode only (not cloud) — `supabase start` spins up full Supabase stack in Docker
- `foundation--discover` Step 6 installs Supabase CLI, runs `supabase init` + `supabase start`, parses real credentials from `supabase status`, writes `.env.local` with no placeholders
- `architecture--new-feature` Step 6 runs `npx supabase db reset` after writing migrations to keep local DB in sync
- `foundation--validate` now checks `supabase/config.toml` exists, `.env.local` has no placeholders, and local stack is running
- One-time VPS prerequisite: `sudo usermod -aG docker paperclip` (already done)

**Files changed:**

| File | What changed |
|---|---|
| `skills/foundation--discover/SKILL.md` | Added Step 6: Docker check, Supabase CLI install, init, start, credential write |
| `skills/architecture--new-feature/SKILL.md` | Added `npx supabase db reset` after migration write |
| `skills/foundation--validate/SKILL.md` | Added Supabase config, env, and stack running checks |

---

## Step 7 — Sandbox Skills with Playwright ✅ DONE (2026-04-30)

**What:** 4 new sandbox skills created. SWE Lead can start/stop/test/check the dev server and visually inspect the running app via headless Playwright screenshots.

**Key decisions:**
- `sandbox--up`: installs Playwright if needed (`npx playwright install --with-deps chromium`), ensures Supabase is running, starts `pnpm dev` in background, saves PID to `.sandbox/dev.pid`
- `sandbox--test`: runs `pnpm test:run` + Playwright screenshots of discovered routes → agent reads PNGs with vision capability
- `sandbox--down`: kills dev server via saved PID, falls back to port-based kill
- `sandbox--status`: read-only health check for both Supabase and dev server
- Playwright runs headless on the VPS (no display needed) — install once with `sudo npx playwright install --with-deps chromium`
- One active project at a time — no port collision handling needed on this VPS

**Files changed:**

| File | What changed |
|---|---|
| `skills/sandbox--up/SKILL.md` | New |
| `skills/sandbox--down/SKILL.md` | New |
| `skills/sandbox--test/SKILL.md` | New |
| `skills/sandbox--status/SKILL.md` | New |

---

## Step 8 — Company Package Fixes ✅ DONE (2026-04-30)

**What:** Fixed two bugs that were breaking company creation and a gap in PM instructions.

**Bug 1 — `shannon-company.md` missing:** `company-creator/SKILL.md` referenced `references/shannon-company.md` but the file was never created. Created it as a canonical template with `{PROJECT_NAME}`, `{project-slug}`, `{project-path}` placeholders.

**Bug 2 — Sandbox skills missing from SWE Lead:** `example-company.md` listed `sandbox--up/down/test/status` in SWE Lead skills but those skills didn't exist. Fixed by Step 7.

**Gap — PM foundation chain not triggered:** PM had `foundation--discover` and `foundation--plan` in its skill list but no instruction to run them. Fixed: PM now checks if `.claude/docs/foundation/product-mission.md` exists before spec work — runs full foundation chain on first use, skips on repeat.

**Files changed:**

| File | What changed |
|---|---|
| `.agents/skills/company-creator/references/shannon-company.md` | New — canonical template with placeholders |
| `companies/shannon-factory-2/` | New — first generated company package from template |

---

## Step 9 — Worker Dispatch: Subscription Mode ✅ DONE (2026-04-30)

**What:** Workers now use `claude -p` (Claude Code print mode) instead of `anthropic.messages.create()`. No API key required — uses the subscription credentials already on the server.

**Key decision — `claude -p` vs API key:**

| | `claude -p` (current) | API key (opt-in) |
|---|---|---|
| Cost | Subscription (free) | ~$0.001/worker (Haiku) |
| Speed | ~3-8s/worker | ~0.5-1s/worker |
| Model | Sonnet | Haiku |
| Parallel safety | Limited | Safe at 10+ concurrent |
| Cost visibility | None | Full dashboard reporting |

Decision: use `claude -p` for testing. Switch to API key when shipping to customers (one-file change, documented inline in the skill).

**Files changed:**

| File | What changed |
|---|---|
| `skills/worker--dispatch/SKILL.md` | Replaced Anthropic SDK script with `claude -p` subprocess script; API key mode documented as opt-in |

---

## Step 10 — Company Delete Bug Fix ✅ DONE (2026-04-30)

**What:** `DELETE /api/companies/:id` returned 500 due to 17 tables with `companyId` FKs missing from the delete transaction. Also fixed delete order (`cost_events` must be deleted before `heartbeat_runs`).

**Missing tables added:** `issueThreadInteractions`, `issueAttachments`, `issueLabels`, `issueRelations`, `issueWorkProducts`, `issueExecutionDecisions`, `issueInboxArchives`, `agentConfigRevisions`, `budgetIncidents`, `budgetPolicies`, `inboxDismissals`, `feedbackVotes`, `workspaceOperations`, `workspaceRuntimeServices`, `executionWorkspaces`, `environments`, `environmentLeases`.

**Files changed:**

| File | What changed |
|---|---|
| `server/src/services/companies.ts` | Added 17 missing tables to delete transaction; moved `costEvents` before `heartbeatRuns` |

---

## Implementation Order

```
Step 1 → Step 2 → Step 3 → Step 4 → Step 5 → Step 6 → Step 7 → Step 8 → Step 9 → Step 10
schema   wiring   bootstrap  skill    checkpoint  supabase  sandbox  pkg-fix  workers  delete-fix
```

All steps done. paperclip-shannon is fully adapted for the AI Software Factory Orchestrator-Worker pipeline.

---

## After Each Step — Verification Checklist

- [ ] `pnpm -r typecheck` passes
- [ ] `pnpm test:run` passes
- [ ] `pnpm build` succeeds
- [ ] Manual smoke test: create issue, post `NEXT_COMMAND:` comment, verify stage advances

---

## Related
- [[Orchestrator-Worker Architecture]] — the architecture this implements
- [[Paperclip Compatibility — Orchestrator-Worker]] — confirms what does/doesn't need changing
- [[Pipeline Model & Ticket Strategy]] — the pipeline stages and handoff logic
- [[Personas & Skills Matrix]] — the agent roster being bootstrapped in Step 3
