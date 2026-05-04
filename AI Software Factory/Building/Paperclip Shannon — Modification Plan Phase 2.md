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

## Step 11 — Workers Subscription Consolidation

**Spec:** [[Workers Subscription Consolidation — Spec]]

**Goal:** Move worker dispatch off `WORKER_API_KEY` (pay-as-you-go) onto orchestrator's Max subscription via `claude -p` subprocess mode.

### Sub-tasks

- [ ] **11.1 — VPS verification pass.** Before code changes, verify on the VPS:
  - [ ] `claude -p "test"` runs as `paperclip` user using subscription credentials
  - [ ] Default model on Max plan via `claude -p` (Sonnet 4.6 expected)
  - [ ] Concurrent `claude -p` hang threshold (start at 5, reduce until stable)
  - [ ] Rate-limit error format on stderr/exit code
  - [ ] Reset window duration on Max $200 plan (current Anthropic policy)

- [ ] **11.2 — Restore `claude -p` dispatch path.** Pull the reverted implementation from commit `0ec9e693` as reference. Re-apply with adjustments:
  - [ ] Concurrency cap: hard-coded to 3 (below empirical threshold)
  - [ ] Rate-limit error parser: detects subscription-exhausted errors and surfaces them as ticket comments instead of silent failure
  - [ ] Cost reporting: emit one `biller: "subscription"` event per dispatch batch instead of per-token (placeholder ledger only)

- [ ] **11.3 — Add `WORKER_API_KEY` fallback flag.** Keep current Anthropic SDK path behind `--use-api-key` flag in dispatch invocation. SWE Lead uses fallback only when explicitly needed (e.g., parallel batch >5).

- [ ] **11.4 — Update `worker--dispatch` SKILL.md.** Document both modes, default to `claude -p`, document the fallback escape hatch.

- [ ] **11.5 — Update auth doc.** Refresh [[Paperclip Agent Auth — Subscription vs API]]:
  - [ ] Default worker mode = `claude -p`
  - [ ] `WORKER_API_KEY` now optional (fallback only)
  - [ ] Add rate-limit handling section

### Acceptance criteria

- [ ] Single full pipeline run completes end-to-end without `WORKER_API_KEY` set
- [ ] Total Anthropic API spend for that run = $0 (verified in console)
- [ ] If rate-limit hit mid-run, SWE Lead posts a clear ticket comment with reset timestamp

### Risks

- Subscription rate-limit hit on a long pipeline run forces operator intervention. Mitigation: 11.2 surface clearly; operator can flip fallback flag temporarily.
- `claude -p` output format changes between Claude Code versions. Mitigation: pin Claude Code version on VPS, document in setup doc.

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
