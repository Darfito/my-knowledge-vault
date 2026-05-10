---
title: "Paperclip Shannon — Modification Plan Phase 2"
type: implementation-plan
status: complete
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

## Step 13 — Auto-Wakeup Fix (Debate Router) ✅ DONE (2026-05-10)

**Spec:** [[Step 13 — Auto-Wakeup Fix Plan]]

**Goal:** Eliminate manual human intervention after every routing decision. Server posts a `[System — Pipeline]` comment immediately after each routing, which Paperclip uses as the wakeup trigger for the newly assigned agent.

### Sub-tasks

- [x] **13.1 — Read codebase.** Read `debate-router.ts`, `routes/issues.ts`, `issue-comments.ts` schema, and wakeup notification mechanism.
- [x] **13.2 — Extend `RoutingResult` type.** Add `systemComment?: string` field to `DebateRouterDecision` type.
- [x] **13.3 — Populate systemComment per path.** 8 routing paths × distinct instruction text. LGTM has no systemComment (SWE stays on issue).
- [x] **13.4 — Insert system comment in `routes/issues.ts`.** After `currentIssue = advanced`, calls `svc.addComment(issueId, systemComment, {})` with empty actor (both authorAgentId and authorUserId null = system comment).
- [x] **13.5 — Fix DECISION exclusion from round-trip cap.** DECISION removed from ROUND_TRIP_TAGS — it's a CEO override, not a PM↔SWE debate round-trip. Prevents cap from firing before DECISION routes to PM.
- [x] **13.6 — Update AGENTS.md for all 4 agents in v3 template.** Added "Autonomous Continuation" section to CEO, PM, UI/UX Lead, SWE Lead.
- [x] **13.7 — Typecheck.** `pnpm -r typecheck` exit 0.
- [x] **13.8 — Unit tests.** 31/31 tests pass (22 existing routing tests + 9 new systemComment tests).

### Acceptance criteria

- [x] `pnpm -r typecheck` exit 0
- [x] `debate-router.test.ts` 31 tests pass (22 original + 9 systemComment)
- Remaining e2e tests pending real company test

---

## Step 14 — URS Skills Port ✅ DONE (2026-05-10)

**Spec:** [[Step 14 — URS Skills Port Plan]]

**Prerequisite:** Step 13 — complete ✅

**Goal:** Enable Shannon pipeline to start from a URS document. CEO ingests or drafts URS; PM compiles it and auto-creates Sprint 0 issues; each issue specs itself via `foundation--shape-spec --from-urs`.

### Sub-tasks

- [x] **14.1 — Source files.** ai-software-factory directory not present on VPS — skills built from Step 14 plan spec directly.
- [x] **14.2 — `foundation--urs-draft`.** Created `skills/foundation--urs-draft/SKILL.md`. CEO skill: auto-detects pre-structured URS vs prose brief; writes `urs/main.md`; posts [TASK] for PM.
- [x] **14.3 — `foundation--urs`.** Created `skills/foundation--urs/SKILL.md`. PM skill: compiles → `urs/index.json`, `applies-to.json`, `main.tex`; updates `project-state.md`.
- [x] **14.4 — `foundation--sprint-plan`.** Created `skills/foundation--sprint-plan/SKILL.md`. PM skill: complexity formula (tables×2 + applies_to + risk_zone + deps); Sprint 0 seeds auth FR first; bin-packs remaining FRs.
- [x] **14.5 — Extend `foundation--shape-spec`.** Added `--from-urs FR-XX` mode at top of existing SKILL.md. Reads from `urs/index.json`; writes `specs/{slug}.md` + `urs/tasks/FR-XX.json`. Existing non-URS flow unchanged.
- [x] **14.6 — `urs--create-issues`.** Created `skills/urs--create-issues/SKILL.md`. Shannon-specific: uses PAPERCLIP_* env vars; POST one issue per Sprint 0 FR; sets pipelineStage=spec + assigneeAgentId=PM; marks kickoff issue shipped.
- [x] **14.7 — Company templates.** Created `shannon-v4-company.md` (new, not modifying v3 — preserves fallback). Updated `company-creator` SKILL.md to offer v2/v3/v4 with when-to-use table.
- [ ] **14.8 — End-to-end test.** Kickoff issue with 5-FR sample URS → verify full automated flow. (Pending real company test.)

### Acceptance criteria

- [x] Skills created / extended without breaking existing non-URS flow
- [x] v3 company template unchanged (v4 is a separate file)
- [ ] End-to-end test with real 5-FR URS kickoff (pending)

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
