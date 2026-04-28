---
title: "Lifecycle - Project vs Feature"
type: architecture-decision
status: decided
tags: [factory-building, lifecycle, workflow, paperclip]
source: ai-software-factory-conversation.md
date: 2026-04-22
---

# Lifecycle — Project vs Feature

The factory uses a **two-tier cycle**: a per-project chain (run once at creation) and a per-feature chain (repeated for every feature on that project).

---

## Per-Project (once, at project creation)

```
foundation:init       ← scaffold the project, run the baseline migration
foundation:discover   ← document the product (users, use cases, scope)
foundation:plan       ← create the feature backlog
design:system         ← document the design system (optional)
```

## Per-Feature (repeated for every feature)

```
foundation:shape-spec       ← write the spec
architecture:new-feature    ← migration + RPC (only if schema changes)
implementation:new-feature  ← scaffold code
sandbox:up                  ← stand up preview
qa:new-tests                ← scaffold tests
qa:fix                      ← run tests, iterate until green
```

---

## Key Insight

- The CEO kicks off the project-level chain **once** when a company starts.
- Every ticket after that is a feature and flows through the per-feature chain only.
- This matches `workflows.md`: *"Per feature (one session each): foundation:status → foundation:shape-spec → architecture:new-feature → implementation:new-feature → qa:new-tests → qa:fix."*
- In Paperclip terms: Paperclip handles per-feature chains as ticket workflows; the project-level chain is a one-time bootstrap.

---

## Related
- [[Pipeline Model & Ticket Strategy]] — how the per-feature pipeline is enforced in Paperclip tickets
- [[Personas & Skills Matrix]] — who executes each step in the per-feature chain
- [[Main Index]] — Section 3.2 (Commands by OS module) maps commands to Shannon phases
