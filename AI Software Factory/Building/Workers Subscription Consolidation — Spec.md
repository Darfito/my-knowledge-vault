---
title: "Workers Subscription Consolidation — Spec"
type: implementation-spec
status: proposed
tags: [paperclip, workers, subscription, billing, cost-optimization]
date: 2026-05-04
---

# Workers Subscription Consolidation — Spec

> **Goal.** Eliminate double billing in Paperclip Shannon by moving worker dispatch from `WORKER_API_KEY` (pay-as-you-go) onto the orchestrator's Max subscription. Targets a flat $200/month total cost ceiling instead of subscription + variable API spend.

---

## Problem Statement

Current Shannon auth split (see [[Paperclip Agent Auth — Subscription vs API]]):

| Layer | Auth | Billing |
|---|---|---|
| Orchestrators (CEO, PM, UI/UX, SWE Lead) | Max subscription via `claude login` | Flat $200/mo |
| Workers (Code, Design, Test) | `WORKER_API_KEY` + Anthropic SDK + Sonnet 4.6 | Pay-as-you-go per token |

This means **two parallel billing streams**. The orchestrator subscription has a flat cap with time-window resets (~5h rolling window per Anthropic Max plan), while the worker API key just keeps charging on every call — no reset, no ceiling beyond the spending limit set in the Anthropic console.

For a factory that dispatches dozens of workers per ticket, the API line item dominates total cost. The user's stated preference: **"better $200 flat than $200 + unbounded API spend."**

---

## Why The Current Setup Exists

From [[Paperclip Shannon — Modification Plan]] Step 9 (2026-04-30): workers were intentionally moved to `WORKER_API_KEY` + Sonnet because of two real constraints:

1. **Parallelism.** Direct `messages.create()` calls scale to 10+ concurrent workers cleanly. The previous `claude -p` subprocess mode (commit `0ec9e693`, reverted same day) had concurrency limits — too many parallel `claude -p` processes hang.
2. **Cost reporting.** Anthropic SDK calls return token usage which the dispatch skill posts to `/api/cost-events` with `biller: "worker"` for per-task breakdown in the board dashboard. `claude -p` output does not surface token counts the same way.

These are not arguments against consolidation — they are constraints the consolidation must respect.

---

## Three Migration Options

### Option A — `claude -p` subprocess mode (revisited)

Workers invoked via `claude -p "<task>"` subprocess. Output captured from stdout. Auth piggybacks on the orchestrator's `claude login` session.

| Aspect | Detail |
|---|---|
| Billing | Inside Max subscription — no separate API charge |
| Reset behavior | Subject to Max rate-limit window (~5h rolling) |
| Parallelism | Capped at ~3–5 concurrent processes (per existing troubleshooting note) |
| Cost reporting to Paperclip | Lost — `claude -p` does not expose token counts in stdout by default |
| Implementation effort | Medium — already existed at commit `0ec9e693`; can be re-derived |

### Option B — `Agent` tool subagents from orchestrator session

SWE Lead spawns workers as Claude Code subagents via the `Agent` tool instead of subprocesses. Same Claude Code session, same subscription billing.

| Aspect | Detail |
|---|---|
| Billing | Inside Max subscription |
| Reset behavior | Same Max rate-limit window |
| Parallelism | Bounded by Claude Code's internal subagent concurrency (unverified — needs test on VPS) |
| Cost reporting to Paperclip | No native hook — would need a wrapper that emits cost events |
| Implementation effort | High — `worker--dispatch` skill must be rewritten from Python `asyncio.gather()` to `Agent` tool dispatch, and SWE Lead must run inside a Claude Code session with Agent tool access |

### Option C — Hybrid (selective)

Cheap, high-volume worker calls (formatting, single-file CSS, boilerplate tests) → `claude -p` (subscription). Heavy reasoning workers that benefit from Sonnet → keep on `WORKER_API_KEY`.

| Aspect | Detail |
|---|---|
| Billing | Mixed — most volume on subscription, residual API spend for selective tasks |
| Implementation effort | Medium — dispatch skill needs a routing layer that picks mode per task |

---

## Recommended Path

**Option A first**, with the parallelism cap accepted as a known constraint, because:

1. It directly cancels the API line item — primary user goal.
2. The reverted `claude -p` implementation is recoverable from git history (commit `0ec9e693`).
3. Cost reporting loss is acceptable in exchange for predictable billing — Paperclip dashboard becomes "subscription is flat, no per-task cost to track."
4. If the parallelism cap proves painful in practice, **then** layer Option C on top: keep `claude -p` as default, fall back to `WORKER_API_KEY` only when SWE Lead explicitly requests >5 parallel workers.

Option B is deferred — too much rewrite risk, and the `Agent` tool concurrency limit on Paperclip's hosted Claude Code is unverified.

---

## Key Design Decisions

| Decision | Choice | Reasoning |
|---|---|---|
| Default worker mode | `claude -p` subprocess | Subscription billing, primary user requirement |
| Concurrency cap | 3 parallel workers per dispatch | Below the empirical hang threshold |
| Model | Whatever `claude -p` defaults to (Sonnet 4.6 on Max) | Matches current Sonnet quality |
| Cost reporting | Skipped for `claude -p` workers; emit a single "subscription-billed" event per dispatch batch | Avoid fake per-token estimates |
| `WORKER_API_KEY` | Kept as fallback path, gated behind explicit flag | Escape hatch if subscription rate-limit is hit mid-pipeline |
| Rate-limit hit behavior | SWE Lead pauses, posts pipeline comment "rate-limited until {timestamp}", waits for window reset | No silent failure |

---

## Open Questions

1. **Does `claude -p` on the VPS use the `paperclip` user's `claude login` credentials?** Must verify before implementation. If subprocess inherits the wrong shell environment, auth will fail silently.
2. **What is the actual rate-limit window on Max $200 plan in 2026-05?** Anthropic may have changed it. Verify on the VPS by deliberately exhausting the limit once and timing the reset.
3. **How does SWE Lead detect a rate-limit error from `claude -p`?** Need a stderr/exit-code parser in the dispatch skill.
4. **Should we keep `claude-sonnet-4-6` model pinning or accept the default?** `claude -p --model claude-sonnet-4-6` may or may not be supported on the subscription tier — verify.

---

## Out of Scope

- Migrating CEO/PM/UI/UX Lead auth — already on subscription, no change.
- Changing pipeline stages or `NEXT_COMMAND:` semantics.
- Cost reporting redesign — tracked separately if Option A proves stable.

---

## Related

- [[Paperclip Agent Auth — Subscription vs API]] — current auth model
- [[Orchestrator-Worker Architecture]] — the architecture this preserves
- [[Paperclip Shannon — Modification Plan]] — Step 9 history of `claude -p` revert
- [[Roundtable Discussion Architecture]] — Topic 2 spec, independent of this consolidation
