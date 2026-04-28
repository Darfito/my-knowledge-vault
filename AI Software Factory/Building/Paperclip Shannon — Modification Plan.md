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

## Implementation Order (complete)

```
Step 1 → Step 2 → Step 3 → Step 4 → Step 5
schema   wiring   bootstrap  skill    checkpoint
```

All steps done. paperclip-shannon is now fully adapted for the AI Software Factory Orchestrator-Worker pipeline.

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
