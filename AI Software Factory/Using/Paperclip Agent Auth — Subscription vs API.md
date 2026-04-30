---
title: "Paperclip Agent Auth — Subscription vs API"
type: reference
tags: [paperclip, auth, api, subscription, workers, rate-limits]
date: 2026-04-28
---

# Paperclip Agent Auth — Subscription vs API

Paperclip supports two authentication modes for running agents. The right choice depends on what the agent needs to do.

---

## Two Auth Modes

### 1. Claude Code CLI (Subscription)

When the adapter is `claude_local` and **no `ANTHROPIC_API_KEY` is set**, Claude Code CLI authenticates via `claude login` — your Claude Max/Pro subscription. This means:

- No per-token billing
- No API rate limits (30k TPM)
- Requires `claude login` on the server running Paperclip
- Only works for the orchestrator agent itself (the Claude Code session)

### 2. Direct API (`ANTHROPIC_API_KEY`)

When `ANTHROPIC_API_KEY` is set in the agent's environment, Claude Code uses that key instead of subscription credentials. The adapter warning tells you which mode is active:

> `ANTHROPIC_API_KEY is set. Claude will use API-key auth instead of subscription credentials.`

Direct API is required for:
- Worker dispatch (Python `messages.create()` calls)
- Any code that calls the Anthropic API directly (not via Claude Code CLI)

---

## Shannon Factory Split

The Shannon Factory uses both modes together:

| Role | Auth | Why |
|---|---|---|
| CEO | Subscription (`claude login`) | Orchestrator — judgment and coordination only |
| PM | Subscription (`claude login`) | Orchestrator — spec writing and handoff |
| UI/UX Lead | Subscription (`claude login`) | Orchestrator — design spec + dispatches workers |
| SWE Lead | Subscription (`claude login`) | Orchestrator — reviews output + dispatches workers |
| **Code Workers** | **API key (`WORKER_API_KEY`)** | Direct `messages.create()` calls — Haiku |
| **Design Workers** | **API key (`WORKER_API_KEY`)** | Direct `messages.create()` calls — Haiku |
| **Test Workers** | **API key (`WORKER_API_KEY`)** | Direct `messages.create()` calls — Haiku |

Workers are NOT Claude Code sessions — they are raw API calls dispatched by the `worker-dispatch` skill. They always need an API key.

---

## Setup

### Step 1 — Log in Claude Code on the VPS

```bash
su - paperclip
claude login
# Follow the browser auth flow with your Max account
```

This stores credentials at `~/.claude/` for the `paperclip` user.

### Step 2 — Remove `ANTHROPIC_API_KEY` from orchestrator agents

In each agent's Configuration tab → Environment variables → delete `ANTHROPIC_API_KEY`.

Or remove it from `~/paperclip-shannon/.env` if set globally.

### Step 3 — Add `WORKER_API_KEY` to SWE Lead and UI/UX Lead only

In the Paperclip UI → SWE Lead → Configuration → Environment variables:
- `WORKER_API_KEY` = `sk-ant-...` (your Anthropic API key)

Same for UI/UX Lead. CEO and PM do not need this.

The `worker-dispatch` skill reads `WORKER_API_KEY` (not `ANTHROPIC_API_KEY`) specifically to avoid Claude Code picking it up as API auth.

### Step 4 — Restart the dev server

```bash
pnpm dev:stop && pnpm dev --bind lan
```

---

## Rate Limits

| Tier | Requirement | Sonnet TPM |
|---|---|---|
| Tier 1 (free) | Just signed up | 30,000 |
| Tier 2 | $40+ cumulative spend | 50,000 |
| Tier 3 | $200+ cumulative spend | 200,000 |

**Having API credits ≠ higher tier.** Tier is based on cumulative spend, not balance. A new account with $20 in credits is still Tier 1.

**Claude.ai Max subscription ≠ API credits.** These are completely separate billing systems. Max gives unlimited claude.ai chat; it does not affect API rate limits.

With subscription auth for orchestrators, the 30k TPM limit only applies to worker API calls — which use Haiku (~200 tokens per call) and rarely hit the cap.

---

## Worker Mode: `claude -p` vs API Key

Workers can run in two modes. Choose based on your situation:

### Current setup — API key with Sonnet

Workers call `anthropic.messages.create()` directly using `WORKER_API_KEY` and `claude-sonnet-4-6`. **This is the default in paperclip-shannon.**

- `WORKER_API_KEY` is separate from `ANTHROPIC_API_KEY` — Claude Code ignores it, orchestrators stay on subscription
- The key can come from a completely different Anthropic account than the orchestrators
- Model: `claude-sonnet-4-6`
- Speed: ~0.5-2s per worker (direct HTTP)
- Full cost reporting to Paperclip dashboard (Sonnet: $3/$15 per MTok)
- Safe at 10+ parallel workers
- Set via Paperclip UI → SWE Lead → Configuration → Environment variables: `WORKER_API_KEY=sk-ant-...`

---

## Troubleshooting

**Agent wakes up but immediately fails with 429** — `ANTHROPIC_API_KEY` is still set. Remove it so the agent uses subscription auth.

**Workers fail with auth error** — `WORKER_API_KEY` is not set on the SWE Lead / UI/UX Lead agent (API key mode only).

**`claude login` doesn't persist between server restarts** — Credentials are stored per Unix user. Make sure you ran `claude login` as the `paperclip` user (not root or santoso).

**`claude -p` workers hang** — Too many concurrent `claude -p` processes. Reduce `asyncio.gather()` concurrency to 3-5 workers.
