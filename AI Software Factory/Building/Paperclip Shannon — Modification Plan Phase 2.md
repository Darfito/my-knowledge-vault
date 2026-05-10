---
title: "Paperclip Shannon — Modification Plan Phase 2"
type: implementation-plan
status: planning
tags: [factory-building, paperclip, implementation, phase-2, workers, multi-agent]
date: 2026-05-04
---

# Paperclip Shannon — Modification Plan Phase 2

> **Phase 1** ([[Paperclip Shannon — Modification Plan]]) is complete — Steps 1–10 shipped through 2026-04-30.
> **Phase 2** addresses two new directives from the 2026-05-04 session:
> 1. Worker billing consolidation (eliminate double billing)
> 2. Multi-agent debate flow (SWE Lead validates PM brief before execution)

---

## Step 11 — Workers Subscription Consolidation ✅ DONE (2026-05-04)

**Spec:** [[Workers Subscription Consolidation — Spec]]

**Goal:** Move worker dispatch off `WORKER_API_KEY` (pay-as-you-go) onto orchestrator's Max subscription via `claude -p` subprocess mode.

### Sub-tasks

- [x] **11.1 — VPS verification pass.**
  - [x] `claude -p "test"` works as `paperclip` user using subscription credentials ✅
  - [x] Concurrency tested: 0 failures at 3/5/7/10/15 concurrent workers (Claude Code 2.1.126) — old hang concern does not apply to this version
  - [x] Rate-limit format: stderr contains "rate limit"/"too many requests" patterns, exit code non-zero
  - [ ] Reset window: ~60 min assumed (not empirically verified — would require deliberately exhausting quota)

- [x] **11.2 — `claude -p` dispatch path implemented.** Based on reverted commit `0ec9e693`:
  - [x] Concurrency cap: `asyncio.Semaphore(5)` — conservative vs empirical max of 15
  - [x] Rate-limit error parser: `is_rate_limited()` checks stderr for 6 known patterns
  - [x] Cost reporting: one `biller: "subscription"` batch event with `costCents: 0`

- [x] **11.3 — `WORKER_API_KEY` fallback.** Env var `USE_API_KEY=true` switches to Anthropic SDK path. Full per-token cost reporting restored in fallback mode.

- [x] **11.4 — `worker--dispatch` SKILL.md rewritten.** Documents both modes, default to `claude -p`, escape hatch with `USE_API_KEY=true`.

- [x] **11.5 — Auth doc updated.** [[Paperclip Agent Auth — Subscription vs API]]: default = subscription, `WORKER_API_KEY` optional fallback, concurrency empirical table added, rate-limit section added.

**Commit:** `3652fdb8` — feat: step 11 — workers subscription consolidation

### Acceptance criteria

- [ ] Single full pipeline run completes end-to-end without `WORKER_API_KEY` set
- [ ] Total Anthropic API spend for that run = $0 (verified in console)
- [x] If rate-limit hit mid-run, SWE Lead posts a clear ticket comment with reset timestamp

### Key findings from 11.1 verification

The previous `claude -p` hang concern (reason for Step 9 revert on 2026-04-30) was version-specific. Claude Code 2.1.126 on this VPS handles 15 concurrent subprocesses cleanly. Production cap is set at 5 anyway — well within the empirical safe range.

The stdin warning (`Warning: no stdin data received in 3s`) is suppressed by passing `stdin=asyncio.subprocess.DEVNULL`.

---

## Step 12 — Roundtable Debate Flow (MAD Asimetris) ✅ DONE (~2026-05-05)

**Spec:** [[Roundtable Discussion Architecture]]

**Goal:** Insert SWE Lead validation gate between PM brief and execution, with shared Discussion Board context for all four agents.

### Sub-tasks

- [x] **12.1 — Discussion Board storage.** Chose Option (b): structured comment-thread aggregation. Leverages existing comment infrastructure, no migration risk.
- [x] **12.2 — Signal parser.** `parseDebateSignal()` added as sibling to `parseNextCommand()` in `server/src/routes/issues.ts`. Handles all 8 signal tags.
- [x] **12.3 — Routing logic.** `server/src/services/debate-router.ts` created. Tracks revision count, escalation to CEO at count=2, human checkpoint at count=5, `[LGTM]` triggers stage advance.
- [x] **12.4 — `≥2 concerns` server-side validation.** Returns HTTP 400 for malformed `[CONFIRMED]` without ≥2 enumerated items.
- [x] **12.5 — Agent skill prompt updates.** `shannon-v3-company.md` created with all 4 agent instruction sets.
- [x] **12.6 — UI surfacing.** Text-only board via existing comment thread. Visual diff deferred.
- [x] **12.7 — Hard upper bound.** Max 5 round-trips before human checkpoint.
- [x] **12.8 — End-to-end test.** Verified via shannon-v3-test company.

**Known issue found post-Step-12:** After `debate-router.ts` reassigns the issue, the new agent does not auto-wake. Human must post a manual comment to trigger wakeup. → Fixed in **Step 13**.

---

## Step 13 — Auto-Wakeup Fix (Debate Router)

**Spec:** [[Step 13 — Auto-Wakeup Fix Plan]]

**Goal:** Eliminate manual human intervention after every routing decision. Server posts a `[System — Pipeline]` comment immediately after each routing, which Paperclip uses as the wakeup trigger for the newly assigned agent.

### Sub-tasks

- [ ] **13.1 — Read codebase.** Read `debate-router.ts`, `routes/issues.ts`, `issue-comments.ts` schema, and wakeup notification mechanism.
- [ ] **13.2 — Extend `RoutingResult` type.** Add `systemComment: string | null` field to the return type of the routing function.
- [ ] **13.3 — Populate systemComment per path.** 8 routing paths × distinct instruction text (see plan doc for exact wording).
- [ ] **13.4 — Insert system comment in `routes/issues.ts`.** After DB update for assignee/stage, insert row in `issue_comments` if `routing.systemComment` is set.
- [ ] **13.5 — Verify wakeup trigger.** Confirm `author_type: 'system'` comment triggers Paperclip wakeup. If not, use `author_type: 'agent'` with `authorAgentId: currentAgentId` as fallback.
- [ ] **13.6 — Update AGENTS.md for all 4 agents.** Add "Autonomous Continuation" section explaining how to act on the system comment without waiting for human input.
- [ ] **13.7 — Typecheck.** `pnpm -r typecheck` exit 0.
- [ ] **13.8 — End-to-end test.** Walk a full debate thread on shannon-v3-test; verify no human comment needed between routing steps.

### Acceptance criteria

- [ ] CEO posts [TASK] → PM automatically gets system comment and posts [PM BRIEF]
- [ ] PM posts [PM BRIEF] → SWE Lead automatically wakes and posts [CONFIRMED] or [CONCERNS]
- [ ] SWE posts [CONCERNS] → PM automatically wakes for revision
- [ ] After 2× concerns → CEO automatically gets escalation system comment
- [ ] After [LGTM] → pipeline_stage advances (already works from v3; verify still works)
- [ ] v2 company unaffected
- [ ] `pnpm -r typecheck` exit 0
- [ ] `debate-router.test.ts` 22 existing tests pass + new tests for `systemComment` field

---

## Step 14 — URS Skills Port

**Spec:** [[Step 14 — URS Skills Port Plan]]

**Prerequisite:** Step 13 must be complete and verified.

**Goal:** Enable Shannon pipeline to start from a URS document. CEO ingests or drafts URS; PM compiles it and auto-creates Sprint 0 issues; each issue specs itself via `foundation--shape-spec --from-urs`.

### Sub-tasks

- [ ] **14.1 — Read source files.** `ai-software-factory/.claude/commands/foundation/{urs-draft,urs,sprint-plan,shape-spec}.md` + ADR 0002 & 0003.
- [ ] **14.2 — Port `foundation--urs-draft`.** New `skills/foundation--urs-draft/SKILL.md`. CEO skill: structures brief → `urs/main.md`.
- [ ] **14.3 — Port `foundation--urs`.** New `skills/foundation--urs/SKILL.md`. PM skill: compiles `urs/main.md` → `index.json`, `applies-to.json`, `main.tex`.
- [ ] **14.4 — Port `foundation--sprint-plan`.** New `skills/foundation--sprint-plan/SKILL.md`. PM skill: bin-packs FRs → `clusters.json`, `sprint-plan.md`.
- [ ] **14.5 — Extend `foundation--shape-spec`.** Add `--from-urs FR-XX` mode to existing SKILL.md. Reads from `urs/index.json`; writes `specs/` + `urs/tasks/`.
- [ ] **14.6 — Build `urs--create-issues`.** New `skills/urs--create-issues/SKILL.md`. Shannon-specific: POST one issue per Sprint 0 FR via Paperclip API.
- [ ] **14.7 — Update company templates.** `shannon-v3-company.md` + new `shannon-v4-company.md` — CEO URS-First Kickoff + PM URS Compilation sections. Update `company-creator` SKILL.md to offer v4.
- [ ] **14.8 — End-to-end test.** Kickoff issue with 5-FR sample URS → verify full automated flow.

### Acceptance criteria

- See [[Step 14 — URS Skills Port Plan]] checklist for full breakdown
- [ ] `pnpm -r typecheck` exit 0
- [ ] Existing v2 company unaffected

---

## Implementation Order

```
Step 11 → Step 12 → Step 13 → Step 14
billing    debate    wakeup    URS-first
```

**Why this order:**
1. Step 11 reduces operating cost immediately — every day of delay = burnt API credits.
2. Step 12 is a larger, riskier change. Doing it on top of stabilized billing means cost overrun during debate-flow rollout is bounded.
3. Step 13 is a usability blocker — without auto-wakeup, the v3 debate flow requires constant human prompting and is unusable in production.
4. Step 14 requires Step 13 — the URS-first chain (CEO → PM → N FR issues) stalls without auto-wakeup at every hop.

---

## Verification Checklist (Both Steps)

After each step:

- [ ] `pnpm -r typecheck` passes across all packages
- [ ] `pnpm test:run` passes
- [ ] `pnpm build` succeeds
- [ ] Manual smoke test on VPS dev server (port 3100)
- [ ] Vault docs updated to reflect actual implementation (not just plan)
- [ ] [[Paperclip Shannon — Current Condition 2026-05-04]] superseded by new snapshot

---

## Pre-Implementation Session Plan

When session moves to VPS:

1. Read `paperclip-shannon` repo structure — confirm Phase 1 state matches docs
2. Check git log for any unrecorded changes since 2026-04-30
3. Run **Step 11.1** verification pass first — empirical findings drive 11.2 implementation details
4. Land Step 11 fully (skill + auth doc) before opening Step 12
5. Step 12 starts with **12.1 + 12.2** (storage + parser) — pure additions, no risk to existing flows
6. **12.5** (skill prompt updates) is the highest-risk change — agents may need iteration

---

## Related

- [[Workers Subscription Consolidation — Spec]] — Step 11 spec
- [[Roundtable Discussion Architecture]] — Step 12 spec
- [[Paperclip Shannon — Modification Plan]] — Phase 1 history
- [[Paperclip Shannon — Current Condition 2026-05-04]] — current state baseline
- [[Paperclip Agent Auth — Subscription vs API]] — auth doc to update in Step 11.5
