---
title: "Skill Naming Convention"
type: architecture-decision
status: decided
tags: [factory-building, skills, naming, paperclip]
source: ai-software-factory-conversation.md
date: 2026-04-22
---

# Skill Naming Convention

Skill names must be **globally unique** within a skills directory (global or project). A collision causes one skill to silently shadow another — no error, no warning.

---

## Convention

| Skill type | Format | Example |
|---|---|---|
| AI Software Factory skills | `namespace--skill` (double-dash) | `architecture--new-feature`, `foundation--init`, `qa--fix` |
| Anthropic catalog skills | Keep as-is (short, generic) | `docx`, `pdf`, `xlsx`, `pptx` |
| Sandbox skills | `sandbox--<action>` | `sandbox--up`, `sandbox--test`, `sandbox--down`, `sandbox--status` |
| Community skills (from skills.sh) | `community--<original-name>` | `community--git-review` |

---

## Why Double-dash, Not Single Colon

- Single colon (`:`) is reserved on **Windows filesystems** and in many tooling environments.
- Double-dash (`--`) is universally valid, rare enough in filenames that collisions with external skills are very unlikely.
- Visually similar to the colon-delimited slash command names (`architecture:new-feature`) — easy to mentally map between them.

---

## Collision Check Step

Add to `publish-skills` script:
1. List all names in `~/.claude/skills/` (global) and `.claude/skills/` (project)
2. Detect duplicates
3. Fail loud with a clear error message

**Add as M1.8 acceptance criterion.**

---

## Related
- [[Personas & Skills Matrix]] — which personas load which skills
- [[Main Index]] — Section 3.3 (Skills) for the existing `newborn-gate` and `reflect-task` skills
