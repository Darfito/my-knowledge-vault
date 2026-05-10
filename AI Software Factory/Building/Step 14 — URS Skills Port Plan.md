---
title: "Step 14 — URS Skills Port Plan"
type: modification-plan
status: draft
tags: [paperclip, shannon, urs, skills, step-14, planning, urs-first]
date: 2026-05-10
---

# Step 14 — URS Skills Port Plan

> **Goal.** Enable the Shannon pipeline to start from a published URS document instead of a freeform brief. CEO receives a kickoff issue containing the URS, runs the ingestion skills, and PM automatically creates one Paperclip issue per FR in Sprint 0 — no human prompting required after the initial kickoff.
>
> **Prerequisite.** Step 13 (auto-wakeup fix) must be complete and verified. Without auto-wakeup, the CEO → PM → issue creation chain will stall and require human intervention.

---

## Context: What URS-First Means Here

The AI Software Factory (`ai-software-factory/`) has a URS-first ingestion lane defined in ADR 0003 and `docs/workflows.md`. The factory's commands compile a Markdown URS into a machine-readable `index.json`, cluster FRs into sprints, and lazy-expand each FR into tasks on demand.

In Paperclip Shannon, the same flow maps as follows:

| Factory command | Shannon equivalent | Agent |
|---|---|---|
| `/foundation:urs:draft` | `foundation--urs-draft` skill | CEO |
| `/foundation:urs` | `foundation--urs` skill | PM |
| `/foundation:plan --from-urs` | `foundation--plan` skill (add `--from-urs` mode) | PM |
| `/foundation:sprint-plan` | `foundation--sprint-plan` skill | PM |
| `/foundation:shape-spec --from-urs FR-XX` | `foundation--shape-spec` skill (add `--from-urs` mode) | PM |
| *(no factory equivalent)* | `urs--create-issues` skill | PM |

---

## Files to Read First

### In `ai-software-factory/` (source to port from)

```
.claude/commands/foundation/urs-draft.md      ← source for foundation--urs-draft
.claude/commands/foundation/urs.md            ← source for foundation--urs
.claude/commands/foundation/sprint-plan.md    ← source for foundation--sprint-plan
.claude/commands/foundation/shape-spec.md     ← source for --from-urs mode addition
.claude/docs/adr/0002-urs-source-format.md   ← urs/main.md required structure
.claude/docs/adr/0003-urs-first-ingestion-and-sprint-planning.md ← complexity scoring formula
```

### In `paperclip-shannon/` (target — understand existing patterns)

```
skills/foundation--plan/SKILL.md             ← reference: how existing plan skill is structured
skills/foundation--shape-spec/SKILL.md       ← existing skill to extend with --from-urs
skills/foundation--discover/SKILL.md         ← reference: how CEO/PM hand-off is written

.agents/skills/company-creator/references/shannon-v3-company.md   ← CEO + PM agent instructions
                                                                     (where to add skill invocation docs)

server/src/routes/issues.ts                  ← POST /companies/:id/issues endpoint
                                               (needed for urs--create-issues API call)
packages/db/src/schema/issues.ts             ← issues table schema (fields for new issues)
```

---

## Skills to Build

### Skill 1 — `foundation--urs-draft` (Port, CEO)

**Purpose.** When the user has a product brief or rough requirements prose (not a structured URS), CEO runs this skill to author a properly structured `urs/main.md` from that input.

**Source file to port:** `.claude/commands/foundation/urs-draft.md`

**Conversion rules (same as all ports):**
- Add `SKILL.md` frontmatter: `name: foundation--urs-draft`, `description: ...`
- Replace "Ask the user: ..." → "Read from issue description or attached content"
- Replace "Tell the user / confirm before writing" → "Write directly, post summary comment on issue"
- Remove any `COMMAND_COMPLETE` blocks
- Replace "Run /foundation:urs next" → "Post comment: 'URS draft complete at urs/main.md. PM: run foundation--urs to compile.'"

**Output:**
- `urs/main.md` — structured Markdown URS following ADR 0002 format
- Issue comment summarising sections created and FR count

**When to skip this skill:**
If the kickoff issue already contains a structured URS (has front-matter + section headings + tables), CEO skips `foundation--urs-draft` and posts directly to PM to run `foundation--urs`.

---

### Skill 2 — `foundation--urs` (Port, PM)

**Purpose.** Compile `urs/main.md` into machine-readable artifacts consumed by all downstream skills. This is the core of URS-first ingestion.

**Source file to port:** `.claude/commands/foundation/urs.md`

**Output files (write all three):**
- `urs/index.json` — full requirement list, schema per ADR 0002:
  ```json
  {
    "project": { "name": "...", "code": "...", "version": "...", "status": "draft" },
    "requirements": [
      { "id": "FR-01", "class": "FR", "type": "Functional", "rank": "C",
        "title": "...", "text": "...", "section_anchor": "...", "risk_zone": 1 }
    ],
    "user_matrix": [...],
    "roles": [...],
    "glossary": [...]
  }
  ```
- `urs/applies-to.json` — NFR/UR/VR → FR cross-cutting map
- `urs/main.tex` — LaTeX formal artifact (compile from template if available, else emit structured LaTeX)

**Also updates:**
- `.claude/docs/project-state.md` — adds backlog rows per FR (id, urs_ref, risk_zone, maturity=spec, last_updated). **Never duplicates requirement text** — IDs only.

**Conversion notes:**
- Replace "Ask the user to review the compiled output" → "Post comment with compile summary: FR count, NFR count, any warnings (wildcard applies_to, missing sections)"
- The compiler must parse Markdown tables: strict on column count and section headings; lenient on whitespace per ADR 0002

**Key invariants to enforce:**
- Every `requirements[]` entry must have `id`, `rank` (C/I/D), `title`, `text`
- Rank C → `risk_zone: 1`, Rank I → `risk_zone: 2`, Rank D → `risk_zone: 3`
- URS IDs must be unique across the document
- Warn (not error) if any NFR/UR/VR row has `applies_to: ["*"]` (wildcard — regex found no FR mentions)

---

### Skill 3 — `foundation--sprint-plan` (Port, PM)

**Purpose.** Read `urs/index.json`, compute complexity points per FR, cluster FRs by section/persona, pack clusters into sprints, and emit a sprint timeline. Sprint 0 is always a walking skeleton.

**Source file to port:** `.claude/commands/foundation/sprint-plan.md`

**Complexity formula (from ADR 0003):**
```
points = (tables_touched × 2)
       + (applies_to_count × 1)   ← capped at 10, wildcards excluded
       + (risk_zone × 1)          ← Z1=3, Z2=2, Z3=1 → so rank C adds 3
       + (dep_count × 1)          ← FR depends on other FRs
```

Default sprint budget: **13 points**. Auto-shrink if total points < 2 × budget.

**Sprint 0 selection (deterministic, NOT topo-driven):**
Pick one Zone-1 FR matching this priority order:
1. auth / login / registration
2. case / submission / application
3. Fallback: first Zone-1 FR alphabetically

**Bin-packing rule:** Operates on FRs individually, not on whole clusters. A cluster that exceeds the sprint budget spans multiple sprints. No "oversized sprint" for oversized clusters.

**Output files:**
- `urs/clusters.json` — FR groupings by section/persona
- `urs/sprint-plan.md` — human-readable sprint timeline:
  ```markdown
  ## Sprint 0 — Walking Skeleton
  | FR | Title | Points | Zone |
  |---|---|---|---|
  | FR-01 | Auth: Login | 5 | Z1 |

  ## Sprint 1
  ...
  ```

**Conversion notes:**
- Post comment on issue with sprint summary: total FRs, total sprints, Sprint 0 FR list
- If < 5 FRs total: skip clustering, one FR per sprint in topological order

---

### Skill 4 — `foundation--shape-spec` (Extend existing skill, PM)

**Purpose.** The existing `foundation--shape-spec` skill generates a feature spec from user intent. Add a `--from-urs FR-XX` mode that reads the spec directly from `urs/index.json` instead.

**Source file to read:** `skills/foundation--shape-spec/SKILL.md` (existing)

**Change: add `--from-urs` detection at the top of the skill:**
```
If the issue description or invocation contains "--from-urs FR-XX":
  1. Read urs/index.json — extract the row where id == "FR-XX"
  2. Read urs/applies-to.json — extract constraints that reference FR-XX
  3. Populate the spec template from the FR row data
  4. Write to urs/tasks/FR-XX.json (task expansion) AND specs/{fr-title}.md
  5. Do NOT ask for user input — all data comes from index.json
Else:
  [existing shape-spec flow unchanged]
```

**Output (--from-urs mode):**
- `specs/{fr-title-slug}.md` — feature spec in the standard template
- `urs/tasks/FR-XX.json` — lazy task expansion record:
  ```json
  {
    "fr_id": "FR-01",
    "tasks": ["spec", "migration", "rpc", "action", "component", "tests"],
    "completed": [],
    "spec_path": "specs/submit-registration.md",
    "generated_at": "2026-05-10T..."
  }
  ```

---

### Skill 5 — `urs--create-issues` (New, Shannon-specific, PM)

**Purpose.** Read `urs/sprint-plan.md` and create one Paperclip issue per FR in Sprint 0 via the Paperclip API. This is the only skill with no factory equivalent — it bridges the URS world and the Paperclip ticket world.

**No source to port** — write from scratch.

**Algorithm:**
```
1. Read urs/sprint-plan.md — extract Sprint 0 FR list
2. Read urs/index.json — get title + text for each Sprint 0 FR
3. Read .paperclip/config.json (or env) — get company_id, pm_agent_id, base_url
4. For each FR in Sprint 0:
   a. POST {base_url}/api/companies/{company_id}/issues
      Body:
        title: "FR-XX — {title}"
        description: "{text}\n\n---\n@spec: FR-XX\nRun: foundation--shape-spec --from-urs FR-XX"
        pipeline_stage: "spec"
        assignee_agent_id: {pm_agent_id}
   b. Log created issue ID
5. Post summary comment on kickoff issue:
   "Sprint 0 tickets created: [FR-01 #123], [FR-02 #124], ..."
   "PM is now assigned to each. Auto-wakeup will trigger spec work."
6. Mark kickoff issue pipeline_stage = "shipped"
```

**Config resolution order for base_url, company_id, pm_agent_id:**
1. `.paperclip/config.json` in project cwd (written during company import)
2. Environment variables: `PAPERCLIP_URL`, `PAPERCLIP_COMPANY_ID`, `PAPERCLIP_PM_AGENT_ID`
3. Fallback: post comment asking PM to set these values and retry

**API call details (verify exact endpoint against `server/src/routes/issues.ts`):**
```bash
curl -X POST http://localhost:3100/api/companies/{company_id}/issues \
  -H "Content-Type: application/json" \
  -d '{
    "title": "FR-01 — Submit Registration",
    "description": "...",
    "pipeline_stage": "spec",
    "assignee_agent_id": "pm-agent-uuid"
  }'
```

**Error handling:**
- If API call fails: log the failed FR, continue with remaining FRs, report failures in summary comment
- If company_id / pm_agent_id not resolvable: post comment asking human to set config, stop

---

## End-to-End Flow After Step 14

```
Human creates "Project Kickoff" issue
  Description: [paste URS content OR paste brief for CEO to structure]
  Assign to: CEO
        │
        ▼ (auto-wakeup via Step 13)
    CEO Agent
      ├── If description is a brief (no FR tables):
      │     Run foundation--urs-draft → writes urs/main.md
      └── If description is already a structured URS:
            Skip urs-draft, copy content to urs/main.md directly
      Post [TASK] comment → NEXT_COMMAND: stage=spec → PM wakes up
        │
        ▼ (auto-wakeup via Step 13)
    PM Agent
      ├── Run foundation--urs          → urs/index.json, applies-to.json, main.tex
      ├── Run foundation--sprint-plan  → clusters.json, sprint-plan.md
      └── Run urs--create-issues       → N issues created for Sprint 0 FRs
      Post summary comment on kickoff issue
      Kickoff issue → pipeline_stage = shipped
        │
        ▼
    [N new Paperclip issues, one per Sprint 0 FR]
    Each issue:
      pipeline_stage = spec
      assignee = PM
      description includes: "Run foundation--shape-spec --from-urs FR-XX"
        │
        ▼ (auto-wakeup via Step 13 triggers PM for each new issue)
    PM runs foundation--shape-spec --from-urs FR-XX
    → writes specs/{fr}.md + urs/tasks/FR-XX.json
    → NEXT_COMMAND: stage=design → CEO wakes up for [TASK] framing
        │
        ▼
    Normal v3 roundtable debate per FR issue
    (CEO [TASK] → PM brief → SWE confirm → UI/UX → [LGTM] → workers)
```

---

## Agent Instruction Updates

After porting the skills, update `shannon-v3-company.md` (and `shannon-company.md` for v2) to tell CEO and PM about the URS-first lane.

### CEO addition to `AGENTS.md`:

```markdown
## URS-First Kickoff

When assigned to a kickoff issue (title contains "Kickoff" or "URS"):
1. Check if the description contains structured URS tables (columns: URS ID, Type, Requirement, Rank).
   - YES → copy content to `urs/main.md` in the project cwd, then post [TASK] for PM.
   - NO  → run the `foundation--urs-draft` skill to structure it first, then post [TASK] for PM.
2. Your [TASK] comment must instruct PM to run: foundation--urs → foundation--sprint-plan → urs--create-issues.
3. Do not create FR issues yourself — that is PM's responsibility via urs--create-issues.
```

### PM addition to `AGENTS.md`:

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

## Implementation Order

```
Step 14.1  Read source files in ai-software-factory
           (.claude/commands/foundation/urs-draft.md, urs.md, sprint-plan.md, shape-spec.md)
           Understand each command's structure before porting.

Step 14.2  Port foundation--urs-draft
           Create: skills/foundation--urs-draft/SKILL.md
           Test: invoke manually in a test project with a brief in the issue description

Step 14.3  Port foundation--urs
           Create: skills/foundation--urs/SKILL.md
           Test: invoke with a pre-written urs/main.md — verify index.json, applies-to.json emitted correctly

Step 14.4  Port foundation--sprint-plan
           Create: skills/foundation--sprint-plan/SKILL.md
           Test: verify Sprint 0 selection is deterministic (auth FR picked first)
                 verify bin-packing — no oversized sprints

Step 14.5  Extend foundation--shape-spec with --from-urs mode
           Edit: skills/foundation--shape-spec/SKILL.md
           Test: invoke with --from-urs FR-01 — verify reads from index.json, writes urs/tasks/FR-01.json

Step 14.6  Build urs--create-issues (new, Shannon-specific)
           Create: skills/urs--create-issues/SKILL.md
           First: read server/src/routes/issues.ts to confirm the exact POST endpoint + body schema
           Test: create a test company, run skill, verify issues appear in Paperclip UI

Step 14.7  Update agent instructions in company templates
           Edit: .agents/skills/company-creator/references/shannon-v3-company.md (CEO + PM sections)
           Edit: .agents/skills/company-creator/references/shannon-company.md (CEO + PM sections)

Step 14.8  End-to-end test
           Create kickoff issue with a sample 5-FR URS → verify full auto-flow:
           CEO structures → PM compiles + sprint plan → PM creates 5 issues → each issue specs itself
```

---

## Checklist

### Pre-conditions
- [ ] Step 13 (auto-wakeup) is complete and verified end-to-end
- [ ] `pnpm dev --bind lan` running on VPS (port 3100)
- [ ] `claude login` active under `paperclip` user

### Reading (do before coding)
- [ ] Read `.claude/commands/foundation/urs-draft.md` in ai-software-factory
- [ ] Read `.claude/commands/foundation/urs.md` in ai-software-factory
- [ ] Read `.claude/commands/foundation/sprint-plan.md` in ai-software-factory
- [ ] Read `skills/foundation--shape-spec/SKILL.md` (existing) in paperclip-shannon
- [ ] Read `server/src/routes/issues.ts` — confirm POST issues endpoint + required body fields
- [ ] Read `packages/db/src/schema/issues.ts` — confirm assignee_agent_id column name

### Skill 1 — foundation--urs-draft
- [ ] `skills/foundation--urs-draft/SKILL.md` created
- [ ] Writes `urs/main.md` with correct ADR 0002 structure (front-matter + all 5 section headings)
- [ ] Posts summary comment with FR count
- [ ] Skipped gracefully when URS is already structured

### Skill 2 — foundation--urs
- [ ] `skills/foundation--urs/SKILL.md` created
- [ ] Emits `urs/index.json` with correct schema (id, class, rank, title, text, risk_zone)
- [ ] Emits `urs/applies-to.json` with NFR/UR/VR → FR map
- [ ] Emits `urs/main.tex` (even if basic LaTeX wrapper)
- [ ] Updates `.claude/docs/project-state.md` with FR backlog rows (no requirement text duplicated)
- [ ] Warns on wildcard `applies_to: ["*"]`
- [ ] Rank C → risk_zone 1, I → 2, D → 3

### Skill 3 — foundation--sprint-plan
- [ ] `skills/foundation--sprint-plan/SKILL.md` created
- [ ] Complexity formula applied: (tables×2) + (applies_to, capped 10) + risk_zone + dep_count
- [ ] Sprint 0 selects auth/registration FR first (deterministic, not topo-driven)
- [ ] Bin-packing operates on FRs, not clusters — no oversized sprints
- [ ] Auto-shrinks budget when total points < 2 × budget
- [ ] Emits `urs/clusters.json` + `urs/sprint-plan.md`
- [ ] Posts sprint summary comment (total FRs, total sprints, Sprint 0 list)

### Skill 4 — foundation--shape-spec (extend)
- [ ] `--from-urs FR-XX` mode detected at top of skill
- [ ] Reads FR data from `urs/index.json` by ID — no LLM hallucination of requirements
- [ ] Reads constraints from `urs/applies-to.json` for the FR
- [ ] Writes `specs/{fr-slug}.md` using standard spec template
- [ ] Writes `urs/tasks/FR-XX.json` with task expansion record
- [ ] Existing non-URS flow unchanged

### Skill 5 — urs--create-issues
- [ ] `skills/urs--create-issues/SKILL.md` created
- [ ] Reads Sprint 0 FRs from `urs/sprint-plan.md`
- [ ] Reads FR titles + text from `urs/index.json`
- [ ] Resolves company_id + pm_agent_id from config or env
- [ ] Creates one Paperclip issue per FR via POST API
- [ ] Issue description includes `@spec: FR-XX` traceability tag
- [ ] Issue description includes `Run: foundation--shape-spec --from-urs FR-XX`
- [ ] Sets `pipeline_stage: "spec"` and `assignee_agent_id: pm_agent_id`
- [ ] Marks kickoff issue `pipeline_stage = shipped` after all issues created
- [ ] Posts summary comment with created issue IDs
- [ ] Handles API failures gracefully (log + continue + report)

### Agent instructions
- [ ] `shannon-v3-company.md` CEO section updated with URS-first kickoff instructions
- [ ] `shannon-v3-company.md` PM section updated with URS compilation + urs--create-issues instructions
- [ ] `shannon-company.md` (v2) updated same way

### Integration test
- [ ] Create kickoff issue with a 5-FR sample URS
- [ ] CEO automatically structures URS (or skips if pre-structured) → posts [TASK]
- [ ] PM automatically compiles URS → generates sprint plan → creates 5 issues
- [ ] Each FR issue appears in Paperclip UI with correct title, description, stage, assignee
- [ ] Auto-wakeup (Step 13) triggers PM on each FR issue → spec runs without human prompting
- [ ] `pnpm -r typecheck` exits 0
- [ ] Existing v2 company unaffected

---

## What Does NOT Change in Step 14

- `debate-router.ts` routing logic — unchanged
- Pipeline stages after `spec` (design → implementation → sandboxed → tested → shipped) — unchanged
- `worker--dispatch` skill — unchanged
- Existing 13 ported skills — unchanged except `foundation--shape-spec` (extension only, no breaking changes)
- v2 company pipeline for non-URS issues — unchanged

---

## Related

- [[Step 13 — Auto-Wakeup Fix Plan]] — prerequisite: auto-wakeup must work first
- [[Roundtable Discussion Architecture]] — v3 debate flow that FR issues will use
- [[AI Software Factory/Building/Skill Port Analysis — ai-software-factory ke paperclip-shannon]] — porting methodology
- [[VPS-Shannon/paperclip-dev-setup]] — how to access the VPS and run the dev server
- [[VPS-Shannon/shannon-company-setup]] — how to import and use the Shannon company
