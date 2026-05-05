---
title: "Shannon V3 — Test Playbook with Rollback"
type: playbook
status: current
tags: [paperclip, shannon, v3, roundtable, testing, rollback, playbook]
date: 2026-05-05
---

# Shannon V3 — Test Playbook with Rollback

> **Purpose.** Walk through testing the v3 (roundtable / debate-aware) Shannon company without disrupting the existing v2 production company (`shannon-factory-2`). Includes a rollback table for every step so you can pull the plug at any point.

For background on what v3 is and how it differs from v2, see [[Shannon Company Templates — v2 vs v3]] and [[Roundtable Discussion Architecture]].

---

## Strategy at a glance

Run **both companies in parallel under the same Paperclip server**:

- v2 (`shannon-factory-2`) — existing production company, **untouched**
- v3 (`shannon-v3-test`) — newly imported, isolated by company id

Switch between them in the dashboard sidebar. To roll back v3, you stop using it (or delete the company); v2 is never modified.

The Step 12 server code (`debate-router.ts` + `routes/issues.ts` updates) is signal-gated: it only fires when the comment body contains a Roundtable tag like `[TASK]` / `[PM BRIEF v<n>]` / `[CONFIRMED]` / `[CONCERNS]` / `[DECISION]` / `[UI/UX SPEC]` / `[LGTM]`. v2 agents emit `NEXT_COMMAND:` instead, so the router never engages on v2 issues — confirmed in `server/src/services/debate-router.test.ts` (22 unit tests).

---

## Pre-flight checklist

| Item | Check |
|---|---|
| Server has Step 12 code in working tree | `git diff server/src/services/index.ts` shows debate-router exports |
| Step 12 typecheck passes | `pnpm -r typecheck` exit 0 |
| Step 12 tests pass | `npx vitest run src/services/debate-router.test.ts` — 22 ✅ |
| `claude login` completed under `paperclip` user | `claude -p "test"` returns output |
| Project workspace dir for v3 created | `mkdir -p /home/paperclip/workspaces/shannon-v3-test` |
| v2 dev server stopped | `pnpm dev:stop` (avoid port collisions on restart) |

---

## Phase 1 — Bring up the v3 company

### 1.1 Generate the v3 package

In Claude Code on the VPS, invoke the `company-creator` skill. When prompted:

| Question | Answer |
|---|---|
| Variant | `v3` |
| Project name | `Shannon V3 Test` (or any other) |
| Project path | `/home/paperclip/workspaces/shannon-v3-test` (NEW dir, not the v2 one) |
| Output directory | `companies/shannon-v3-test/` |

This produces `companies/shannon-v3-test/{COMPANY.md, agents/..., .paperclip.yaml, README.md}` from `references/shannon-v3-company.md`.

**Rollback at this point:** `rm -rf companies/shannon-v3-test/` — file system only, nothing touched server-side.

### 1.2 Restart the dev server with Step 12 code

```bash
cd /home/paperclip/paperclip-shannon
pnpm dev:stop 2>/dev/null
pnpm dev --bind lan
```

Confirms in the log: `listening on http://0.0.0.0:3100`. The Step 12 server code loads on startup.

**Rollback at this point:** `git checkout server/src/services/index.ts server/src/routes/issues.ts && rm -f server/src/services/debate-router.ts server/src/services/debate-router.test.ts && pnpm dev:stop && pnpm dev --bind lan` — server reverts to pre-Step-12 behavior. `shannon-factory-2` keeps working.

### 1.3 Import v3 as a NEW company

```bash
cd /home/paperclip/paperclip-shannon
node cli/dist/index.js company import companies/shannon-v3-test \
  --target new \
  --new-company-name "Shannon V3 Test" \
  --yes \
  --paperclip-url http://127.0.0.1:3100
```

Critical flag: `--target new` — creates a fresh company row instead of merging into `shannon-factory-2`. After import the dashboard sidebar shows two companies.

**Rollback at this point:** delete the v3 company entirely:
```bash
node cli/dist/index.js company list --paperclip-url http://127.0.0.1:3100
# copy the v3 company id from the listing
node cli/dist/index.js company delete --company-id <V3_COMPANY_ID> \
  --paperclip-url http://127.0.0.1:3100
```
Removes v3 from DB (issues, agents, history). v2 untouched.

---

## Phase 2 — Drive a test issue through v3

In the dashboard (http://84.247.150.72:3100), click `Shannon V3 Test` in the sidebar → create an issue → assign to its CEO.

You can either let the agents run autonomously, or post comments manually impersonating each agent for a deterministic smoke test:

| Step | Comment by | Body | Expected effect |
|---|---|---|---|
| 1 | CEO | `[TASK: build login screen]`<br>`Goal: ...`<br><br>`[NEEDS_DESIGN: yes]` | Issue auto-reassigns to PM |
| 2 | PM | `[PM BRIEF v1]`<br>`Stories: ...`<br>`Scope: ...`<br>`Estimate: ...` | Auto-reassigns to SWE Lead |
| 3a | SWE Lead | `[CONFIRMED]`<br>`1. Did PM confirm migration ordering?`<br>`2. Did PM verify rate-limit constraint?` | Auto-reassigns to UI/UX (because `NEEDS_DESIGN: yes`) |
| 3b *(alt — concerns path)* | SWE Lead | `[CONCERNS]`<br>`1. Token storage approach unclear`<br>`2. Auth scope ambiguous` | Auto-reassigns back to PM for revision |
| 4 | UI/UX | `[UI/UX SPEC]`<br>`Components: ...` | Auto-reassigns back to SWE Lead for `[LGTM]` |
| 5 | SWE Lead | `[LGTM]`<br>`ready to dispatch` | `pipeline_stage` advances to `design`, SWE Lead reassigned |

After each comment, refresh the issue — assignee should change automatically. Look in `activity_log` for `issue.debate_routed` entries; each one explains the routing decision (`reason`, `latestTag`, `assigneeAgentId`).

### Validation gate spot-check

Have the SWE Lead post `[CONFIRMED]\n1. only one item` — server should respond:

```
HTTP 400 {"error":"SWE Lead [CONFIRMED] requires ≥2 enumerated lines (e.g. \"1. ...\" / \"2. ...\")"}
```

Also confirm the comment is **not** persisted in `issue_comments` for that issue (the validator runs before insert).

### Round-trip cap spot-check

Force 3+ PM brief revisions with persistent SWE concerns. After the 5th round-trip, the router should create a `request_confirmation` interaction titled "Debate round-trip cap reached" instead of routing further.

---

## Phase 3 — Confirm v2 still works

Open an issue in `shannon-factory-2` and post a `NEXT_COMMAND: stage=design,assignee=<designer-agent-id>` comment as one of its agents. It should advance exactly as before.

Why this works: the debate router only fires when (a) actor is an agent, (b) `pipeline_stage === "spec"` or null, AND (c) the comment contains a debate signal tag. v2 agents emit `NEXT_COMMAND:` (not a debate tag), so the router stays silent.

If v2 *does* break, that's a regression — open an issue and revert per the rollback table below.

---

## Rollback matrix

Pick the lightest rollback that solves your problem.

| Rollback | Action | Reverses | Side effects on v2 |
|---|---|---|---|
| **Stop using v3** | Click `shannon-factory-2` in the sidebar; ignore v3 | Everything | None — v2 was never modified |
| **Pause v3** | Dashboard → v3 company → Pause | New issues + agent runs on v3 only | None |
| **Wipe v3 company** | `node cli/dist/index.js company delete --company-id <v3-id>` | Removes v3 entirely (issues, agents, history) | None |
| **Revert Step 12 server code** | `git checkout server/src/services/index.ts server/src/routes/issues.ts && rm -f server/src/services/debate-router.ts server/src/services/debate-router.test.ts` then restart server | Removes the debate router; v3 agents will post tagged signals that don't auto-route, but the server accepts the comments | None — v2 keeps working |
| **Re-import v2 prompts onto v3 company** *(advanced)* | `node cli/dist/index.js company import companies/shannon-factory-2 --target existing --company-id <v3-id> --collision replace --include company,agents --yes ...` | Switches v3's agents to v2 prompts in-place; existing issues stay but mid-flight spec stages may produce mixed signals | None |

The first three are dashboard / CLI actions, no code changes. The fourth is the nuclear option for a Step 12 regression. The fifth is a way to keep the company shell but undo the v3-ness of its agents.

---

## Acceptance — when is v3 "validated"?

You can mark v3 as ready to migrate `shannon-factory-2` onto when all of these hold for at least one full issue end-to-end:

- [ ] CEO `[TASK]` + `[NEEDS_DESIGN]` block routes to PM automatically
- [ ] PM `[PM BRIEF v1]` routes to SWE Lead automatically
- [ ] SWE `[CONFIRMED]` with ≥2 items routes correctly (UI/UX or back to SWE for LGTM, depending on `NEEDS_DESIGN`)
- [ ] SWE `[CONFIRMED]` with <2 items returns HTTP 400 and is not persisted
- [ ] SWE `[CONCERNS]` routes back to PM
- [ ] After 2 PM revisions with persistent concerns, escalation to CEO fires
- [ ] After 5 round-trips, human checkpoint via `request_confirmation` fires
- [ ] `[LGTM]` advances `pipeline_stage` to `design` (if NEEDS_DESIGN: yes) or `implementation` (if no)
- [ ] `shannon-factory-2` (v2) is unaffected throughout — `NEXT_COMMAND:` still works, no router activity in its activity log

If any of these fail, open a follow-up issue against Step 12 before rolling out to v2.

---

## Switching v2 → v3 in production (later, after acceptance)

Once v3 is validated, you can migrate `shannon-factory-2` in-place without losing issue history. See "Migrate the existing v2 company to v3 prompts in-place" guidance below — keep this step until v3 has run cleanly through several real features.

```bash
# 1. Generate a v3 package targeted at shannon-factory-2's existing project path
#    (use company-creator skill, pick v3, point to the v2 project path,
#     output to companies/shannon-factory-2-v3-staging/)

# 2. Find the v2 company id
node cli/dist/index.js company list --paperclip-url http://127.0.0.1:3100

# 3. Re-import into the existing company with replace collision
node cli/dist/index.js company import companies/shannon-factory-2-v3-staging \
  --target existing \
  --company-id <SHANNON_FACTORY_2_ID> \
  --collision replace \
  --include company,agents \
  --yes \
  --paperclip-url http://127.0.0.1:3100
```

Run a `--dry-run` first (drop `--yes`, add `--dry-run`) to preview the diff. `--include company,agents` keeps issues/projects/skills untouched — only swaps agent prompts + `COMPANY.md`.

To roll back from this state to v2 prompts: same command, point at the original v2 package (`companies/shannon-factory-2/`).

---

## Related

- [[Shannon Company Templates — v2 vs v3]] — what v3 changes vs v2
- [[Roundtable Discussion Architecture]] — the MAD Asimetris pattern v3 implements
- [[Paperclip Shannon — Modification Plan Phase 2]] — Step 12 sub-tasks and acceptance criteria
- [[Paperclip Agents vs Workers — Shared Subscription Model]] — auth context for v3 agents and workers
- [[Paperclip Agent Auth — Subscription vs API]] — fallback configuration if subscription rate-limit is hit during v3 testing
