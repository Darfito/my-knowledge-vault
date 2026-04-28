---
title: "Action Items — MVP Sprint"
type: action-items
tags: [action-items, mvp, factory-building, paperclip]
date: 2026-04-22
updated: 2026-04-22
---

# MVP Sprint — Action Items

Small, solvable tasks. Each one can be done in a single session. Prioritized by dependency order — unblock the top items first.

**Legend:** `[can do now]` = no dependencies | `[blocked by X]` = needs something else first | `[done]` = completed

---

## 0 — Blockers to verify first

- [ ] **Verify Paperclip ticket reassignment** `[can do now]`
  - Read `D:\Side-Mission\project-2\refrensi\paperclip\` source or API docs
  - Answer: can a ticket's assignee be changed after creation?
  - **If yes** → Option 2 (single ticket, sequential stages) is confirmed. Proceed.
  - **If no** → fall back to Option 1 (dependency tickets). Update [[Pipeline Model & Ticket Strategy]].
  - *Resolves: [[Paperclip Integration Unknowns]] Q1*

- [ ] **Find and import Q3 & Q4 from the conversation** `[can do now]`
  - Search `D:\Side-Mission\project-2\refrensi\` for other conversation files
  - Extract decisions and add notes to `AI Software Factory/Building/`
  - *Resolves: [[Missing Q3 & Q4 from Conversation]]*

- [ ] **Map the Paperclip milestone structure (M1, M1.x)** `[can do now]`
  - Find the milestone map in refrensi/paperclip docs
  - Add a note: `AI Software Factory/Building/Paperclip Milestone Map.md`
  - *Resolves: [[Paperclip Integration Unknowns]] Q5*

---

## 1 — Zero-code quick wins (do today)

- [ ] **Install memory compiler hooks workspace-wide** `[can do now]`
  - Merge hook config into workspace-level Claude Code settings
  - Source: `D:\Side-Mission\project-2\refrensi\claude-memory-compiler\.claude\settings.json`
  - Effect: every Claude Code session starts auto-capturing to `daily/`
  - Effort: ~10 minutes
  - *From: [[Claude Memory Compiler]] Tier A integration*

- [ ] **Fork `agents/` directory into `webapp-gunawan/`** `[can do now]`
  - Copy `mobile-gunawan/.claude/agents/` → `webapp-gunawan/.claude/agents/`
  - Five-minute fix. Prevents next product from breaking on Quinn's bootstrap.
  - *From: [[Index Utama]] Gap #1*

---

## 2 — Simple skill/script work

- [ ] **Write `sandbox:test` skill** `[can do now]`
  - Convention: read `scripts.test` from `package.json` → `docker exec <container> pnpm test`
  - If missing: post ticket comment asking Engineer to add test script
  - *From: [[Test Command Convention]]*

- [ ] **Write `.env.sandbox` MVP secret injection** `[can do now]`
  - Agents write `.env.sandbox` to `/data/workspaces/<slug>/`
  - Orchestrator reads it on container start
  - Simple file write + env load. No encryption needed for MVP.
  - *From: [[Secret Injection Strategy]]*

- [ ] **Add `NEXT_COMMAND:` markers to each persona's skill end-steps** `[blocked by Paperclip reassignment verify]`
  - Update each skill in `mobile-gunawan/.claude/skills/` to end with correct `NEXT_COMMAND:`
  - Format TBD based on what Paperclip recognizes
  - *From: [[Pipeline Model & Ticket Strategy]] enforcement mechanism*

- [ ] **Add skill collision-check to `publish-skills` script** `[can do now]`
  - Script: list all names in `~/.claude/skills/` and `.claude/skills/`, detect duplicates, fail loud
  - Mark as M1.8 acceptance criterion
  - *From: [[Skill Naming Convention]]*

---

## 3 — Design work needed first, then implement

- [ ] **Design `sandbox:up` skill** `[needs design]`
  - NEW step in per-feature pipeline (it wasn't there before the conversation)
  - Needs: port assignment strategy, URL output format, container name convention
  - Resolves: [[Paperclip Integration Unknowns]] Q4
  - After design: implement `sandbox:up`, `sandbox:down`, `sandbox:status`

- [ ] **Write the per-feature pipeline as a Paperclip workflow** `[blocked by reassignment verify + sandbox:up design]`
  - Map each stage to a Paperclip ticket state
  - Define the `stage` field values: `spec → design → implementation → sandboxed → tested → shipped`
  - Define which persona is assignee at each stage

---

## 4 — Santoso operational fixes (unblock first)

- [ ] **Investigate Paperclip `status=error`** `[can do now]`
  - SSH to VPS, check Paperclip run logs
  - Fix or document root cause
  - *From: [[Index Utama]] Gap #2 — nothing else matters until orchestrator is stable*

- [ ] **Write systemd unit for Telegram bridge** `[can do now after gap 2]`
  - Automate bridge start on boot, add health check + auto-restart
  - *From: [[Index Utama]] Gap #3*

- [ ] **Write script to update `WEBHOOK_URL` from ngrok** `[can do now]`
  - Read new ngrok URL on start → rewrite `.env`
  - *From: [[Index Utama]] Gap #5*

- [ ] **Automate Paperclip enum patches** `[can do now]`
  - Script that re-applies `analyst` and `marketing` AGENT_ROLES patches after reinstall
  - *From: [[Index Utama]] Gap #4*

---

## Completed
*(move items here when done, with date)*

---

## Related
- [[Development Log/2026-04-22]] — context for today's action items
- [[Paperclip Integration Unknowns]] — questions behind the blocked items
- [[Action Items/Phase 2 Backlog]] — items that are not MVP
