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

## Step 12 — Roundtable Debate Flow (MAD Asimetris)

**Spec:** [[Roundtable Discussion Architecture]]

**Goal:** Insert SWE Lead validation gate between PM brief and execution, with shared Discussion Board context for all four agents.

### Sub-tasks

- [ ] **12.1 — Discussion Board storage.** Decide and implement:
  - [ ] Option (a): new column `issues.discussion_board` (JSONB) — append-only tagged sections
  - [ ] Option (b): structured comment-thread aggregation — Paperclip parses tagged comments at read time
  - **Recommendation:** Option (b) — leverages existing comment infrastructure, no migration risk

- [ ] **12.2 — Signal parser.** Add helper sibling to `parseNextCommand()` in `server/src/routes/issues.ts`:
  - [ ] `parseDebateSignal(comment)` returns `{ tag, version?, payload }`
  - [ ] Handles: `[TASK]`, `[PM BRIEF v<n>]`, `[CONFIRMED]`, `[CONCERNS]`, `[DECISION]`, `[UI/UX SPEC]`, `[LGTM]`, `[NEEDS_DESIGN: yes/no]`

- [ ] **12.3 — Routing logic.** New service `server/src/services/debate-router.ts`:
  - [ ] Determines next assignee based on most recent signal on issue
  - [ ] Tracks PM revision count (in-memory derive from comment scan)
  - [ ] Triggers escalation to CEO at revision count = 2 with persistent `[CONCERNS]`
  - [ ] Triggers human `request_confirmation` when CEO emits decision-stuck signal
  - [ ] Emits `NEXT_COMMAND: stage=design` (or `implementation`) on `[LGTM]`

- [ ] **12.4 — `≥2 concerns` server-side validation.** Reject SWE Lead comments that emit `[CONFIRMED]` without ≥2 enumerated lines following. Returns 400 with clear error so the agent retries with valid output.

- [ ] **12.5 — Agent skill prompt updates.** In `.agents/skills/company-creator/references/shannon-company.md`:
  - [ ] CEO instructions: emit `[TASK]` + `[NEEDS_DESIGN: yes/no]` flag
  - [ ] PM instructions: emit `[PM BRIEF v<n>]` with required sections
  - [ ] SWE Lead instructions: read full Discussion Board before output; emit `[CONFIRMED]` or `[CONCERNS]` with ≥2 enumerated items; emit `[LGTM]` only as final signal
  - [ ] UI/UX instructions: triggered only when `[NEEDS_DESIGN: yes]`; emits `[UI/UX SPEC]`

- [ ] **12.6 — UI surfacing.** Issue detail view in Paperclip dashboard shows:
  - [ ] Current debate state (waiting on whom)
  - [ ] PM revision count
  - [ ] Visual diff of [PM BRIEF v1] → [PM BRIEF v2]
  - **Optional for v1** — text-only board is acceptable initially

- [ ] **12.7 — Hard upper bound.** Add max-round-trips counter (default 5). At 5, force-escalate to human checkpoint regardless of signals.

- [ ] **12.8 — End-to-end test.** Create one test issue, drive through:
  - [ ] CEO `[TASK]` → PM `[PM BRIEF v1]` → SWE `[CONCERNS]` → PM `[PM BRIEF v2]` → SWE `[CONFIRMED]` → `[LGTM]`
  - [ ] Verify pipeline_stage advances to `design` or `implementation`
  - [ ] Verify deadlock path: force 2 SWE concerns → CEO `[DECISION]` → resume

### Acceptance criteria

- [ ] All four agents produce correctly tagged output
- [ ] Server enforces signal grammar (rejects malformed)
- [ ] Discussion Board reconstructible from comment thread alone (no separate state)
- [ ] CEO escalation path works in both directions: agent-resolved and human-resolved
- [ ] UI/UX skipped when `[NEEDS_DESIGN: no]`

### Risks

- Signal grammar drift — agents emitting almost-correct tags. Mitigation: 12.4 server-side validation forces correctness.
- Performance: full comment scan per routing decision. Mitigation: cache last-known signal per issue in service layer; rebuild only on new comment.

---

## Implementation Order

```
Step 11 → Step 12
billing    debate
```

**Why this order:**
1. Step 11 reduces operating cost immediately — every day of delay = burnt API credits.
2. Step 12 is a larger, riskier change. Doing it on top of stabilized billing means cost overrun during debate-flow rollout is bounded.
3. Steps are independent — Step 11 does not depend on Step 12 or vice versa, but sequencing reduces blast radius.

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
