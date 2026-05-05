---
title: "Paperclip Agents vs Workers — Shared Subscription Model"
type: reference
status: current
tags: [paperclip, auth, subscription, agents, workers, rate-limits, shannon]
date: 2026-05-05
---

# Paperclip Agents vs Workers — Shared Subscription Model

> **Why this doc exists.** Step 11 (2026-05-04) moved workers from `WORKER_API_KEY` onto the same Claude Max subscription the orchestrator agents already use. After the change, both flavors of Claude calls authenticate the same way — but they are still two structurally different process flavors. This page captures the practical implications: how they overlap, where they differ, and what the shared rate-limit pool means for capacity planning.

For the broader auth setup (where to log in, how to set fallback keys), see [[Paperclip Agent Auth — Subscription vs API]].

---

## TL;DR

Same Anthropic account, same `~/.claude/` token, same rolling rate window — but:

- **Agents** are **long-lived Claude Code sessions** per role (CEO, PM, SWE Lead, UI/UX Lead). One process per agent, woken on issue activity.
- **Workers** are **one-shot `claude -p` subprocesses** dispatched in parallel batches of ≤5 by the SWE Lead via the `worker--dispatch` skill.

Both auth-resolve to the same subscription token, both report `costCents: 0` / `biller: "subscription"`, and both consume from the same rate-limit bucket. The `WORKER_API_KEY` fallback exists specifically to re-route only the worker traffic onto a separate API key when subscription pressure is too high.

---

## Process model — side by side

| | Agent (orchestrator) | Worker |
|---|---|---|
| Process type | Long-lived Claude Code session | One-shot `claude -p "<prompt>"` subprocess |
| Spawned by | Paperclip's heartbeat loop (`claude_local` adapter) | SWE Lead invoking the `worker--dispatch` skill |
| Lifetime | Stays warm between issue interactions; resumed on `wakeup` | Spawn → run prompt → exit; no memory across calls |
| Tool access | Full Claude Code toolset (Read/Edit/Write/Bash/MCP/etc.) | Whatever the prompt + cwd allow inside `claude -p` |
| State carrier | `heartbeat_runs` row + agent runtime state in DB | None — caller (SWE Lead) assembles full context per call |
| Concurrency | One process per agent (4 agents typical for Shannon) | `asyncio.Semaphore(5)` enforced by `worker--dispatch` |
| Cost reporting | Per-run heartbeat events | One batch event per dispatch, `biller: "subscription"`, `costCents: 0` |

---

## Auth — what's actually shared

Both invocations resolve to the same source:

```
agent process / worker subprocess
        │
        └─ inherits CLAUDE_CONFIG_DIR / ~/.claude/
                                    │
                                    └─ subscription token from `claude login`
                                              │
                                              └─ Anthropic Max plan account
                                                      │
                                                      └─ shared rate-limit window (~60 min rolling)
```

Concrete consequences of "same subscription":

1. **One rate-limit pool.** Every Claude call across all 4 agents AND all worker subprocesses draws from the same Max-plan rolling window. SWE Lead dispatching 5 workers while CEO/PM/UI-UX are also active all counts.
2. **Zero per-token billing.** Both paths produce `costCents: 0`. Pre-Step-11 workers used `WORKER_API_KEY` and produced real `cost_events` rows with Sonnet token pricing — that line item is gone for default operation.
3. **One account-level failure mode.** If the subscription token expires, gets revoked, or rate-limits, both agents and workers fail at the same time. Pre-Step-11, API-key workers were independent — agents could keep running while only workers got rate-limited, or vice versa.

---

## Capacity envelope

| Scenario | Concurrent Claude calls |
|---|---|
| Idle | 0 |
| One agent thinking (e.g. PM drafting brief) | 1 |
| All 4 agents simultaneously woken | 4 |
| SWE Lead dispatching workers at the cap | 5 |
| Worst case: 4 agents active + SWE Lead dispatching workers | 4 + 5 = 9 |

Step 11.1 verification (see [[Development Log/2026-05-04]]) ran 15 concurrent `claude -p` subprocesses with zero failures on Claude Code 2.1.126. The 9-call worst case has comfortable headroom locally. The bottleneck is Anthropic-side rate-limiting (subscription bucket), not the local process model.

---

## When the difference matters

- **Worker bursts.** When SWE Lead dispatches workers, you can suddenly add 5 calls on top of an active agent's usage. Watch for stderr lines containing "rate limit" / "too many requests" — `is_rate_limited()` in `worker--dispatch` parses 6 such patterns and exits non-zero with a clear message.
- **Long-running agent + parallel workers.** A single agent processing a long task while workers run in parallel does not conflict locally (separate processes), but they DO share the rate window. If the bucket drains faster than expected, the cause is usually agent + worker overlap.
- **Fallback escape.** If you hit the limit mid-pipeline and need to keep moving, set `USE_API_KEY=true` + `WORKER_API_KEY=<sk-ant-...>` on the SWE Lead. That re-routes ONLY the workers off subscription onto a separate API key (which can be from a different Anthropic account). Restores per-token billing for workers and frees subscription budget for the four agents to keep going. Unset (or set `USE_API_KEY=false`) to revert.

---

## Diagnostics

Spot-check the auth path in use:

```bash
# Confirm the OS user that runs paperclip is logged in
su - paperclip
claude -p "say hello in one word" 2>&1 | head -3

# Confirm no API key shadows the subscription
env | grep -i ANTHROPIC_API_KEY     # should be empty for the agent processes
```

When SWE Lead dispatches workers, check the activity log entries for `cost_events`:

- `biller: "subscription"`, `costCents: 0` → subscription path active (default after Step 11)
- Per-token entries with non-zero `costCents` → fallback API-key path active (`USE_API_KEY=true`)

---

## Related

- [[Paperclip Agent Auth — Subscription vs API]] — setup steps and fallback configuration
- [[Development Log/2026-05-04]] — Step 11 implementation notes and concurrency verification
- [[Paperclip Shannon — Modification Plan Phase 2]] — Step 11 sub-tasks and acceptance criteria
- [[Orchestrator-Worker Architecture]] — the higher-level architecture this auth model serves
