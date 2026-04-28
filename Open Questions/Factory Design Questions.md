---
title: "Factory Design Questions"
type: open-question
status: mixed
priority: medium
tags: [open-question, factory-design, strategy]
date: 2026-04-22
---

# Factory Design Questions

Broader design questions about the factory itself — not Paperclip-specific integration details.

---

## 1. When does the factory become headless?

Right now every Claude Code session is interactive — a human opens a terminal and runs commands. The factory is a **ritual**, not a **service**.

The Product Vision (PLP Software Development Module) requires a headless execution layer: a job queue, a status protocol, and a Verification Dashboard. This turns the factory into something you call via an API.

**The question:** What is the minimum headless layer design? What does a "job" look like? What events does it emit?

**Why it matters:** This is the prerequisite for the PLP integration. Without it, the factory can't be embedded in NexusPLM or sold to pharma/fintech clients.

**Status:** Not yet designed. Deferred to Phase 3 (Index Utama Gap #12).

---

## 2. What is the mobile-specific Shannon Protocol?

The `noahs-ark.md` file is the web Shannon Protocol reference. It gets injected into mobile sub-agents unmodified — but mobile has different rules (e.g., "no `useEffect+fetch`") that currently live only in the repo overview note, not in an injectable manifest.

**The question:** What are the mobile-specific bans and conventions? What would `noahs-ark-mobile.md` contain?

**Status:** Undocumented. Mobile-specific rules live only in the `00 - Project Overview` narrative note.

---

## 3. How should `webapp-gunawan` and `mobile-gunawan` diverge and converge?

Both repos mirror the same core agents/commands/skills, but with documented drift. There's no automated diff or convergence target.

**The question:** Which divergence is intentional (platform-specific rules) vs accidental (copy-paste drift)? How do we keep them aligned without manual sync?

**Status:** Undocumented. Noted as Gap #18 in Index Utama.

---

## 4. What does a fully automated factory session look like end-to-end?

From a human submitting a product idea → to a deployed, tested artifact — what does each step look like? Who does what? What human touchpoints remain by design?

**The question:** Map the full happy path with specific Paperclip events, agent handoffs, skill calls, and human gates.

**Status:** The per-feature pipeline is defined ([[Lifecycle - Project vs Feature]]) but the full end-to-end flow including the CEO → PM kickoff and the final deployment gate has not been written out.

---

## Related
- [[Main Index]] — Section 5 (Gaps & Opportunities) contains the canonical gap list
- [[AI Software Factory/Vision/Vision & Roadmap]] — phase roadmap
- [[Paperclip Integration Unknowns]] — Paperclip-specific blocking questions
