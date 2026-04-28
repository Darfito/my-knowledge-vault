# Shannon Company Setup Guide

Complete guide for setting up the 4-agent Shannon Factory company on a fresh Paperclip Shannon instance.

---

## What the Shannon Company Is

The Shannon company is an AI Software Factory organized as a **sequential pipeline**. One ticket moves through four agents, each doing their part before handing off to the next.

```
User brief / PRD
      │
      ▼
   CEO Agent          ← parses brief, writes structured work plan
      │ NEXT_COMMAND: stage=spec
      ▼
   PM Agent           ← shapes spec, resolves ambiguities
      │ NEXT_COMMAND: stage=design
      ▼
  UI/UX Lead          ← design spec + component inventory (dispatches Design Workers)
      │ NEXT_COMMAND: stage=implementation
      ▼
  SWE Lead            ← implementation (dispatches Code + Test Workers in parallel)
      │ → Human review checkpoint (request_confirmation)
      │ NEXT_COMMAND: stage=tested → shipped
      ▼
    Shipped
```

**Pipeline stages per ticket:** `spec → design → implementation → sandboxed → tested → shipped`

**Handoff mechanism:** Each agent ends its work by posting a comment containing `NEXT_COMMAND: stage=<next_stage>`. Paperclip reads this marker, updates `pipeline_stage` on the issue, and reassigns it to the next agent automatically. *(Requires Steps 1 & 2 from the Modification Plan — see "Status" below.)*

**Workers (Code / Design / Test):** These are NOT Paperclip agents. They are single `messages.create()` API calls dispatched in parallel by the SWE Lead via the `worker--dispatch` skill. Paperclip has no visibility into them — their cost is reported separately via `POST /companies/:id/cost-events`.

---

## Prerequisites

- [ ] VPS Shannon dev server running (`pnpm dev --bind lan` as user `paperclip`)
- [ ] Instance admin account exists (created during `pnpm paperclipai onboard`)
- [ ] `ANTHROPIC_API_KEY` set in the environment for each agent (or per-agent secret in Paperclip UI)
- [ ] *(For full pipeline automation)* Steps 1 & 2 from the Modification Plan are implemented — see "Implementation Status" below

---

## Implementation Status

| Feature | Status | What it enables |
|---|---|---|
| `pipeline_stage` column on issues | ⚠️ **Not committed** | Stage tracking per ticket |
| `NEXT_COMMAND:` server parsing | ⚠️ **Not committed** | Auto-handoff between agents |
| Shannon company package (`shannon-factory/`) | ✅ **Created** — `/home/paperclip/shannon-factory/` | Company to import |
| `worker--dispatch` skill | ⚠️ **Not committed** | SWE Lead parallel worker calls |
| `request_confirmation` checkpoint | ✅ **Native Paperclip** | Human review gate |

The company can be **imported and used now**. Without the pipeline_stage + NEXT_COMMAND changes, stage advancement and reassignment must be done manually (edit the issue stage, reassign to next agent). Everything else works.

To implement the missing steps, see [[Paperclip Shannon — Modification Plan]].

---

## Step 1 — Generate the Shannon Company Package

The pre-built package lives at `/home/paperclip/shannon-factory/`. Skip this step if it already exists.

If you need to regenerate it, run the `company-creator` skill from inside the `paperclip-shannon` repo:

```
# In Claude Code at /home/paperclip/paperclip-shannon
/company-creator
```

The skill produces a directory conforming to `agentcompanies/v1` spec with the 4-agent Shannon structure.

---

## Step 2 — Archive the Old Company (if replacing)

If a company already exists (e.g. "Bonyo Corp") and you want to hide it before importing the Shannon company:

1. Sign in as the instance admin (`darfito.jkt2003@gmail.com`)
2. Go to `/<company-prefix>/company/settings`
3. Find the **Archive Company** button — click it to archive

Archiving hides the company from the sidebar and all dropdowns but keeps all data in the database. The company can be unarchived later. This is different from deletion — no data is lost.

---

## Step 3 — Import the Shannon Company

```bash
# As user paperclip on the VPS:
cd ~/paperclip-shannon
pnpm paperclipai company import --from /home/paperclip/shannon-factory
```

If the import succeeds, the company and all 4 agents will appear in the Paperclip UI.

---

## Step 4 — Grant User Access

New accounts created from the UI don't get company access automatically.

1. Sign in as instance admin
2. Open `/instance/settings/access`
3. Search for the user by email, select them
4. Tick the Shannon Factory checkbox under **Company access** → **Save**

See also: [[paperclip-dev-setup]] → "No company access" troubleshooting section.

---

## Step 5 — Configure Agent API Keys

Each agent needs an Anthropic API key to run.

1. Sign in as instance admin, open the company
2. Go to **Settings → Secrets** (or per-agent settings)
3. Add `ANTHROPIC_API_KEY` with your key

Alternatively, set it in the VPS environment before starting the dev server:

```bash
echo "ANTHROPIC_API_KEY=sk-ant-..." >> ~/paperclip-shannon/.env
```

---

## Day-to-Day Usage

### Starting a new feature

1. Create a new issue in the Shannon Factory company
2. Set **pipeline_stage** to `spec` *(manual until NEXT_COMMAND is implemented)*
3. Assign to **CEO** agent
4. Add a comment describing the feature or paste the PRD
5. CEO processes and hands off (auto if NEXT_COMMAND is active, manual otherwise)

### Manual stage advancement (while NEXT_COMMAND is not yet implemented)

After each agent finishes:
1. Open the issue
2. Edit → set **pipeline_stage** to the next stage
3. Reassign to the next agent

| Stage | Assigned agent |
|---|---|
| `spec` | PM |
| `design` | UI/UX Lead |
| `implementation` | SWE Lead |
| `sandboxed` | SWE Lead (running sandbox + checkpoint) |
| `tested` | SWE Lead (after human approval) |
| `shipped` | Done |

### The human review checkpoint

When SWE Lead finishes implementation, it creates a `request_confirmation` interaction on the issue. You'll see an **Accept / Reject** card in the issue thread. Accept to let the pipeline advance to `tested`; reject with a reason to send it back for fixes.

---

## Org Chart

```
CEO (ceo)
├── PM (pm)
├── UI/UX Lead (designer)
└── SWE Lead (engineer)
        └── [Code Workers, Design Workers, Test Workers — invisible to Paperclip]
```

All agents report to CEO. CEO has no manager (`reportsTo: null`).

---

## Troubleshooting

**Agent not picking up the issue** — Check that the agent's status is `idle` (not `paused`). Check that `ANTHROPIC_API_KEY` is set.

**NEXT_COMMAND comment posted but stage didn't advance** — The server-side NEXT_COMMAND parser isn't implemented yet. Advance the stage manually (see Day-to-Day Usage above).

**Import fails with "unknown adapter type"** — Check `.paperclip.yaml` — only valid adapter types are `claude_local`, `codex_local`, `opencode_local`, `pi_local`, `cursor`, `gemini_local`, `openclaw_gateway`.

**Company already exists error** — Delete the existing company first (Step 2), then re-import.
