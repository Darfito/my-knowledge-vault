---
title: "Shannon Company Templates — v2 vs v3"
type: reference
status: current
tags: [paperclip, shannon, company-template, roundtable, v2, v3, mad-asimetris]
date: 2026-05-05
---

# Shannon Company Templates — v2 vs v3

> **Why two templates?** v3 (Roundtable Debate Flow) is a structurally different spec stage — SWE Lead becomes a validation gate that can push back on PM briefs before execution. v2 (the sequential CEO → PM → UI/UX → SWE handoff that has shipped to date) is preserved unchanged so existing companies and any project that prefers the simpler flow can continue to use it. Companies bootstrap from the template they were imported with; switching variants requires re-import.

---

## At a glance

| | v2 — `shannon-company.md` | v3 — `shannon-v3-company.md` | v4 — `shannon-v4-company.md` |
|---|---|---|---|
| Spec-stage routing | Linear handoff via `NEXT_COMMAND:` markers | Tagged-signal debate; server `debate-router` reassigns automatically | Same as v3 — unchanged |
| Validation gate | None | SWE Lead emits `[CONFIRMED]` / `[CONCERNS]` (≥2 items, server-enforced) | Same as v3 — unchanged |
| UI/UX trigger | Always at `pipeline_stage: design` | Only when CEO sets `[NEEDS_DESIGN: yes]` | Same as v3 — unchanged |
| Auto-wakeup after assignment | ❌ Manual comment needed | ❌ Manual comment needed | ✅ Server posts `[System — Pipeline]` comment |
| URS-first kickoff lane | ❌ | ❌ | ✅ CEO drafts URS → PM compiles + creates Sprint 0 issues |
| Deadlock handling | None | 2 revisions → CEO `[DECISION]`; 5 round-trips → human checkpoint | Same as v3 — unchanged |
| Pipeline advance after spec | Each agent posts `NEXT_COMMAND:` | `[LGTM]` from SWE Lead triggers automatic advance | Same as v3 — unchanged |
| Reporting line | SWE → UI/UX (`reportsTo: designer`) | SWE → CEO (`reportsTo: ceo`) | Same as v3 — unchanged |
| Pattern reference | Sequential pipeline | MAD Asimetris — [[Roundtable Discussion Architecture]] | MAD Asimetris + URS-first ingestion |
| Server requirement | Pre-Step-12 | `debate-router.ts` (Step 12) | Step 12 + system comment insert (Step 13) |
| New skills | — | — | 5 URS skills + `--from-urs` mode on `shape-spec` |

Outside the spec stage (`design → implementation → sandboxed → tested → shipped`) both variants behave identically and use `NEXT_COMMAND:` for pipeline advancement.

---

## Where the templates live

```
paperclip-shannon/
└── .agents/skills/company-creator/
    ├── SKILL.md                                 # asks variant, then reads matching template
    └── references/
        ├── shannon-company.md                   # v2 — sequential (default)
        └── shannon-v3-company.md                # v3 — roundtable / debate-aware
```

The `company-creator` skill prompts the operator to choose `v2` or `v3` at bootstrap time.

---

## How to import

Both variants are imported the same way:

```bash
# Generate the package via the company-creator skill (interactively choose v2 or v3)
# Then import:
paperclipai company import --from companies/<project-slug>/
```

Re-importing with the opposite template into the same `<project-slug>` overwrites the company package files. Existing issues keep their pipeline state — only the agents' instruction files change. Switching from v2 → v3 mid-issue is supported but the in-flight `spec` stage may produce mixed signals; safer to switch on a fresh issue.

---

## When to use which

- **Start with v2** when: the PM brief is unlikely to need technical pushback, you want zero new server-side surface area, or you want the simplest possible handoff for a one-off project.
- **Use v3** when: SWE Lead should validate PM briefs before execution, you want server-enforced sycophancy guards, or the project benefits from documented `[CONCERNS]` round-trips — but you don't have a URS and don't need URS-first flow.
- **Use v4** when: you have a URS document (or a brief CEO can structure into one), and you want the full automated pipeline from URS ingestion → sprint planning → issue creation → spec → debate → implementation.

v3 and v4 are opinionated about how disagreements surface — `[CONCERNS]` requires ≥2 enumerated items, and server-side validation will reject malformed comments with HTTP 400. Tradeoff: more rigorous spec, more comment volume.

For the full v4 design, see [[Shannon Company Templates — v4]].

---

## Server changes that v3 depends on

v3 only works on a paperclip-shannon server with the Step 12 roundtable code. Specifically:

- `server/src/services/debate-router.ts` — pure helpers + routing logic
- `server/src/routes/issues.ts` — comment route validation gate + routing wiring

If the agents post tagged signals on a pre-Step-12 server, the comments save normally but no automatic reassignment happens — the operator must manually advance via the issue assignee dropdown.

---

## Related

- [[Shannon Company Templates — v4]] — v4 detailed design
- [[Shannon Company — Evolution v2 to v4]] — full evolution history with rationale
- [[Roundtable Discussion Architecture]] — the MAD Asimetris spec v3/v4 implements
- [[Paperclip Shannon — Modification Plan Phase 2]] — Step 12 history, Steps 13 & 14 plan
- [[Paperclip Shannon — Current Condition 2026-05-10]] — current state baseline
- [[Single-Agent vs Multi-Agent — Patterns & Tradeoffs]] — pattern background
