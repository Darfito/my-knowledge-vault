---
title: "Paperclip Integration Unknowns"
type: open-question
status: unresolved
priority: high
tags: [open-question, paperclip, factory-building, blocking]
date: 2026-04-22
---

# Paperclip Integration Unknowns

Questions that must be answered before the Paperclip integration can be finalized. Blocking items marked **[BLOCKER]**.

---

## 1. Does Paperclip support ticket reassignment? **[BLOCKER for Option 2]**

The chosen ticket model (Option 2 — single ticket, sequential stages) requires that the `assignee` field changes as the ticket moves from `spec` → `design` → `implementation` → etc.

**What needs verification:**
- Can a Paperclip ticket be reassigned to a different agent after creation?
- Does Paperclip's API expose a `PATCH /tickets/:id` with an `assignee` field?
- If not, we fall back to Option 1 (dependency tickets) or Option 3 (epic + sub-tickets)

**Where to check:** Paperclip source at `D:\Side-Mission\project-2\refrensi\paperclip\` or the Paperclip API docs.

---

## 2. How do `NEXT_COMMAND:` markers trigger Paperclip assignment?

The existing ai-software-factory commands end with a `NEXT_COMMAND:` marker. The plan is for Paperclip to read this marker and trigger the next agent assignment automatically.

**What is unclear:**
- Does Paperclip currently parse `NEXT_COMMAND:` from Claude's output? Or does this need to be built?
- Is the trigger synchronous (Claude finishes → Paperclip immediately reassigns) or async (Paperclip polls)?
- What format does `NEXT_COMMAND:` need to follow for Paperclip to recognize it?

---

## 3. What does Paperclip's "skills manager" look like?

The conversation references Paperclip's "skills manager" (from their roadmap) as the mechanism that handles per-persona skill assignment. Each agent loads only the skills assigned to its role.

**What is unclear:**
- Is the skills manager already built or still on the roadmap?
- Is it a Paperclip UI feature, an API feature, or a config file?
- How does a skill get assigned to a persona — via the Paperclip dashboard, a JSON config, or a CLI command?

---

## 4. How does `sandbox:up` know which port to expose?

The per-feature pipeline now includes `sandbox:up` as a new step. Each project gets a preview environment.

**What is unclear:**
- Is there a port registry per project slug, or does Paperclip assign dynamic ports?
- How does the Engineer agent know the URL of the sandbox after it's up?
- How does `sandbox:down` clean up — does it kill the container or just stop it?

---

## 5. What are M1, M1.8 and what is the full milestone map?

The conversation references M1.8 as the milestone for the skill collision-check acceptance criterion. This implies a milestone numbering system.

**What is unclear:**
- What is M1? (presumably MVP complete?)
- What are M1.1 through M1.7?
- Is there a milestone map document in the refrensi directory?

**Where to check:** `D:\Side-Mission\project-2\refrensi\paperclip\` or the main project docs.

---

## Related
- [[Pipeline Model & Ticket Strategy]] — Option 2 depends on Q1 above
- [[Skill Naming Convention]] — M1.8 depends on Q5 above
- [[Action Items/MVP Sprint]] — action items for each of these
