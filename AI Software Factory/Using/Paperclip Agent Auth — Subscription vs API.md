---
title: "Paperclip Agent Auth — Subscription vs API"
type: reference
tags: [paperclip, auth, api, subscription, workers, rate-limits]
date: 2026-05-04
---

# Paperclip Agent Auth — Subscription vs API

Paperclip supports two authentication modes for running agents. The right choice depends on what the agent needs to do.

---

## Two Auth Modes

### 1. Claude Code CLI (Subscription)

When the adapter is `claude_local` and **no `ANTHROPIC_API_KEY` is set**, Claude Code CLI authenticates via `claude login` — your Claude Max/Pro subscription. This means:

- No per-token billing
- Subject to Max plan rolling rate-limit window (~60 min reset)
- Requires `claude login` on the server running Paperclip
- Works for both orchestrator agents **and** `claude -p` worker subprocesses

### 2. Direct API (`ANTHROPIC_API_KEY`)

When `ANTHROPIC_API_KEY` is set in the agent's environment, Claude Code uses that key instead of subscription credentials. The adapter warning tells you which mode is active:

> `ANTHROPIC_API_KEY is set. Claude will use API-key auth instead of subscription credentials.`

Direct API is required only for `WORKER_API_KEY` fallback mode — not for normal operation.

---

## Shannon Factory Auth Model

All agents run on subscription. Workers default to `claude -p` subprocess mode — same subscription, no additional billing.

| Role | Auth | Mode |
|---|---|---|
| CEO | Subscription (`claude login`) | Claude Code agent session |
| PM | Subscription (`claude login`) | Claude Code agent session |
| UI/UX Lead | Subscription (`claude login`) | Claude Code agent session |
| SWE Lead | Subscription (`claude login`) | Claude Code agent session |
| **Workers (default)** | **Subscription (`claude login`)** | **`claude -p` subprocess** |
| Workers (fallback) | `WORKER_API_KEY` + Anthropic SDK | Direct `messages.create()` — opt-in only |

Workers are NOT separate Claude Code sessions — they are `claude -p` subprocess calls dispatched in parallel by the `worker--dispatch` skill, using the same subscription credentials as the orchestrators.

---

## Setup

### Step 1 — Log in Claude Code on the VPS

```bash
su - paperclip
claude login
# Follow the browser auth flow with your Max account
```

Credentials stored at `~/.claude/` for the `paperclip` user. All orchestrators and `claude -p` workers inherit this.

### Step 2 — Remove `ANTHROPIC_API_KEY` from all agents

In each agent's Configuration tab → Environment variables → delete `ANTHROPIC_API_KEY` if set.

Or confirm it's absent from `~/paperclip-shannon/.env`.

### Step 3 — (Optional) Add `WORKER_API_KEY` for fallback

Only needed if you hit the subscription rate limit mid-pipeline or want >5 workers in parallel beyond the cap.

In Paperclip UI → SWE Lead → Configuration → Environment variables:
- `WORKER_API_KEY` = `sk-ant-...` (a separate Anthropic API key — can be from a different account)
- `USE_API_KEY` = `true` (tells the dispatch script to use API key mode)

CEO, PM, UI/UX Lead do not need these.

### Step 4 — Restart the dev server

```bash
pnpm dev:stop && pnpm dev --bind lan
```

---

## Worker Concurrency (Empirical Findings — 2026-05-04)

Tested `claude -p` subprocess concurrency on VPS (Claude Code 2.1.126):

| Concurrent workers | Result |
|---|---|
| 3 | ✅ 0 failures, ~7s total |
| 5 | ✅ 0 failures, ~8s total |
| 7 | ✅ 0 failures, ~9s total |
| 10 | ✅ 0 failures, ~10s total |
| 15 | ✅ 0 failures, ~16s total |

**Production cap: 5** — conservative vs the empirical safe max of 15. Raise only if needed.

The previous concern about `claude -p` hanging at low concurrency (documented in Step 9 revert, 2026-04-30) does not apply to Claude Code 2.1.126. The hang was version-specific.

---

## Rate Limits

### Subscription mode (`claude -p`)

- Subject to Max plan rolling window (approximately 60 minutes)
- When hit: workers exit non-zero with rate-limit message in stderr
- The dispatch script detects this and prints a retry notice
- **No silent failure** — the ticket comment must be updated with a rate-limit block notice

### API key mode (`WORKER_API_KEY`)

| Tier | Requirement | Sonnet TPM |
|---|---|---|
| Tier 1 (free) | Just signed up | 30,000 |
| Tier 2 | $40+ cumulative spend | 50,000 |
| Tier 3 | $200+ cumulative spend | 200,000 |

**Having API credits ≠ higher tier.** Tier is based on cumulative spend, not balance.

**Claude.ai Max subscription ≠ API credits.** Completely separate billing systems.

---

## Troubleshooting

**Agent wakes up but immediately fails with 429** — `ANTHROPIC_API_KEY` is still set. Remove it so the agent uses subscription auth.

**Workers produce empty output** — Check if `claude -p` is producing a stderr warning. Run `claude -p "hello"` manually as the `paperclip` user to confirm subscription credentials are valid.

**`claude login` doesn't persist between server restarts** — Credentials are stored per Unix user. Make sure you ran `claude login` as the `paperclip` user (not root or santoso).

**Subscription rate limit hit mid-pipeline** — Set `USE_API_KEY=true` and `WORKER_API_KEY=sk-ant-...` on SWE Lead to temporarily switch to API key mode. Restore to subscription after the window resets.

**API key workers fail with auth error** — `WORKER_API_KEY` is not set, or `USE_API_KEY` is not `true`.
