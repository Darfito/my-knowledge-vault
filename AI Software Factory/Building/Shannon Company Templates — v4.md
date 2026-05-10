---
title: "Shannon Company Templates — v4"
type: reference
status: planned
tags: [paperclip, shannon, company-template, v4, urs-first, auto-wakeup]
date: 2026-05-10
---

# Shannon Company Templates — v4

> **Why v4?** v3 introduced the roundtable debate flow (Step 12), but two gaps remain: (1) agents don't auto-wake after reassignment — a human must post a manual comment each time; (2) there is no URS-first ingestion lane — kickoff always starts from a freeform task description. v4 closes both gaps. It is a superset of v3: the debate router is unchanged, but the server now posts system comments as wakeup triggers (Step 13), and PM has five new URS skills (Step 14).

---

## At a Glance

| | v3 — `shannon-v3-company.md` | v4 — `shannon-v4-company.md` |
|---|---|---|
| Spec-stage routing | Roundtable debate via `debate-router.ts` | Same — unchanged |
| Auto-wakeup after assignment | ❌ Human must post manual comment | ✅ Server posts `[System — Pipeline]` comment automatically |
| URS-first kickoff lane | ❌ Not available | ✅ CEO runs `foundation--urs-draft`; PM compiles + creates Sprint 0 issues |
| New skills | — | `foundation--urs-draft`, `foundation--urs`, `foundation--sprint-plan`, `urs--create-issues`, `foundation--shape-spec --from-urs` (extension) |
| Agent instruction changes | — | CEO: URS-First Kickoff section; all agents: Autonomous Continuation section |
| Server requirement | `debate-router.ts` (Step 12) | Step 12 + system comment insert (Step 13) |

---

## What Changes from v3

### 1. Server: System Comment as Wakeup Trigger (Step 13)

Every time `debate-router.ts` completes a routing decision and updates `issues.assignee_agent_id`, the server immediately inserts a row in `issue_comments` with:

```
author_type: "system"
body: [System — Pipeline] <tailored instruction for the next agent>
```

This comment is what Paperclip uses to send the wakeup event to the newly assigned agent. The 8 routing paths each have distinct system comment text — see [[Step 13 — Auto-Wakeup Fix Plan]] for the full message designs.

### 2. Agent Instructions: Autonomous Continuation (Step 13)

All four agents (CEO, PM, SWE Lead, UI/UX Lead) receive an `## Autonomous Continuation` section in their `AGENTS.md`:

```markdown
## Autonomous Continuation

Kamu akan diassign ke issue ini bersama sebuah [System — Pipeline] comment yang menjelaskan apa yang harus kamu lakukan. Baca comment tersebut, baca thread discussion board di atas, lalu langsung mulai bekerja tanpa menunggu input tambahan dari human.

Jika kamu butuh klarifikasi yang tidak bisa kamu resolve sendiri:
- Post pertanyaanmu sebagai bagian dari signal tag yang sesuai (contoh: "Open questions for SWE: ..." di [PM BRIEF])
- Gunakan request_confirmation HANYA jika pipeline tidak bisa lanjut tanpa jawaban human
```

### 3. New Skills: URS-First Lane (Step 14)

Five skills are added to the v4 skills set:

| Skill | Agent | When used |
|---|---|---|
| `foundation--urs-draft` | CEO | Kickoff issue has a brief (not yet a structured URS) → CEO structures it into `urs/main.md` |
| `foundation--urs` | PM | Compile `urs/main.md` → `urs/index.json`, `urs/applies-to.json`, `urs/main.tex` |
| `foundation--sprint-plan` | PM | Read `urs/index.json` → compute complexity → pack FRs into sprints → write `urs/sprint-plan.md` |
| `urs--create-issues` | PM | Read Sprint 0 from `urs/sprint-plan.md` → POST one Paperclip issue per FR via API |
| `foundation--shape-spec` (extend) | PM | `--from-urs FR-XX` mode reads spec from `urs/index.json` instead of prompting user |

### 4. Agent Instructions: URS-First Kickoff (Step 14)

**CEO addition:**

```markdown
## URS-First Kickoff

When assigned to a kickoff issue (title contains "Kickoff" or "URS"):
1. Check if the description contains structured URS tables (columns: URS ID, Type, Requirement, Rank).
   - YES → copy content to `urs/main.md` in the project cwd, then post [TASK] for PM.
   - NO  → run the `foundation--urs-draft` skill to structure it first, then post [TASK] for PM.
2. Your [TASK] comment must instruct PM to run: foundation--urs → foundation--sprint-plan → urs--create-issues.
3. Do not create FR issues yourself — that is PM's responsibility via urs--create-issues.
```

**PM addition:**

```markdown
## URS Compilation & Sprint Planning

When assigned to an issue where CEO has posted [TASK] referencing URS ingestion:
1. Run `foundation--urs` to compile urs/main.md → index.json + applies-to.json
2. Run `foundation--sprint-plan` to generate sprint-plan.md
3. Run `urs--create-issues` to create Sprint 0 FR issues in Paperclip
4. Mark the kickoff issue pipeline_stage = shipped via NEXT_COMMAND

When assigned to a FR issue (title starts with "FR-XX"):
- Run `foundation--shape-spec --from-urs FR-XX` immediately.
- Do not ask for clarification — all data is in urs/index.json.
```

---

## End-to-End Flow (v4)

```
Human creates "Project Kickoff" issue
  Description: [paste URS OR paste brief for CEO to structure]
  Assign to: CEO
      │
      ▼ (auto-wakeup via Step 13 system comment)
  CEO Agent
    ├── If brief (no FR tables): run foundation--urs-draft → writes urs/main.md
    └── If already structured URS: copy to urs/main.md directly
    Post [TASK] comment → debate-router reassigns PM → system comment wakes PM
      │
      ▼ (auto-wakeup via Step 13 system comment)
  PM Agent
    ├── Run foundation--urs          → urs/index.json, applies-to.json, main.tex
    ├── Run foundation--sprint-plan  → clusters.json, sprint-plan.md
    └── Run urs--create-issues       → N issues created for Sprint 0 FRs
    Post summary comment on kickoff issue
    Kickoff issue → pipeline_stage = shipped
      │
      ▼
  [N new Paperclip issues, one per Sprint 0 FR]
  Each issue: pipeline_stage = spec, assignee = PM
  Description: "Run: foundation--shape-spec --from-urs FR-XX"
      │
      ▼ (auto-wakeup triggers PM on each FR issue)
  PM runs foundation--shape-spec --from-urs FR-XX
  → writes specs/{fr}.md + urs/tasks/FR-XX.json
  → NEXT_COMMAND: stage=design or implementation → debate begins
      │
      ▼
  Normal v3 roundtable debate per FR issue
  (CEO [TASK] → PM brief → SWE confirm → UI/UX → [LGTM] → workers)
```

---

## When to Use v4 vs v3 vs v2

| Scenario | Recommended |
|---|---|
| Project starts with a formal URS document | **v4** — URS-first lane fully automated |
| Project starts with a rough brief that CEO should structure | **v4** — `foundation--urs-draft` handles it |
| Project is a one-off with no URS, wants spec debate | **v3** — roundtable without URS overhead |
| Simplest possible sequential handoff, no debate | **v2** — zero server-side complexity |
| Existing v3 company mid-flight (active issues) | Stay on **v3** — switching mid-issue risks mixed signals |

---

## Template Location (After Step 14 Complete)

```
paperclip-shannon/
└── .agents/skills/company-creator/
    ├── SKILL.md                           # presents v2/v3/v4 choice at bootstrap
    └── references/
        ├── shannon-company.md             # v2 — sequential
        ├── shannon-v3-company.md          # v3 — roundtable debate
        └── shannon-v4-company.md          # v4 — roundtable + auto-wakeup + URS-first
```

The `company-creator` skill will be updated (in Step 14.7) to offer v4 as a third option.

---

## What v4 Does NOT Change

- `debate-router.ts` routing logic — all signal → next-agent mappings unchanged
- `[CONFIRMED]` validation gate (≥2 items, server-enforced) — unchanged
- `NEXT_COMMAND:` mechanism for v2 companies — unchanged
- Pipeline stages after `spec` (`design → implementation → sandboxed → tested → shipped`) — unchanged
- `worker--dispatch` skill — unchanged
- All 22 existing standalone skills — unchanged except `foundation--shape-spec` (extension only, backward compatible)
- v2 company operation — system comments only appear in issues that have debate signal tags; v2 issues have none

---

## Related

- [[Step 13 — Auto-Wakeup Fix Plan]] — implementation plan for system comment wakeup
- [[Step 14 — URS Skills Port Plan]] — implementation plan for URS-first skills
- [[Shannon Company Templates — v2 vs v3]] — comparison of all three variants
- [[Shannon Company — Evolution v2 to v4]] — full evolution history with rationale
- [[Roundtable Discussion Architecture]] — v3/v4 debate spec (MAD Asimetris)
- [[Paperclip Shannon — Current Condition 2026-05-10]] — current server state
