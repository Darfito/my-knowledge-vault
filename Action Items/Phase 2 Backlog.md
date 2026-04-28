---
title: "Action Items — Phase 2 Backlog"
type: action-items
tags: [action-items, phase-2, factory-building]
date: 2026-04-22
---

# Phase 2 Backlog

Items that are designed but intentionally deferred past MVP. Do not start these until the MVP sprint is complete.

---

## Secrets & Security
- [ ] **Design `project_secrets` table + pgsodium encryption**
  - Schema: `project_slug`, `key`, `value_encrypted` (pgsodium), timestamps
  - Two orchestrator functions: encrypt (write) + decrypt (read on container start)
  - *From: [[Secret Injection Strategy]]*

- [ ] **Build `secrets:set` skill**
  - Agents POST to orchestrator → writes to `project_secrets` table via service role
  - *From: [[Secret Injection Strategy]]*

- [ ] **Build admin dashboard for human-managed secrets**
  - Humans manage production secrets via UI — agents never touch real prod keys
  - *From: [[Secret Injection Strategy]]*

---

## Ticket & Pipeline Improvements
- [ ] **Back-edge support in Paperclip**
  - QA/Engineer/UI-UX can bounce a ticket back one stage with a reason comment
  - Change stage back to `implementation` + reassign
  - *From: [[Pipeline Model & Ticket Strategy]] deferred section*

- [ ] **DevOps persona**
  - Skills: `deployment:release → deployment:k8s-config`
  - Scope: production deployments (currently out of Paperclip scope)
  - *From: [[Personas & Skills Matrix]] deferred section*

---

## Blueprint Analysis Governance (not yet implemented)
- [ ] **Risk Zones** — map paths → governance zones in `.claude/project-config.json`
  - *From: [[Index Utama]] Gap #7*

- [ ] **VCM (Verification Coverage Metric)** — aggregate, trackable quality signal
  - *From: [[Index Utama]] Gap #8*

- [ ] **GMM (Governance Maturity Metric)** — track newborn-gate history over time
  - *From: [[Index Utama]] Gap #9*

- [ ] **AAI tracking** — enforce `AAI ≤ GMM × VCM` invariant
  - *From: [[Index Utama]] Gap #10*

---

## Memory Compiler Integrations
- [ ] **Tier B: reflect-task bridge** — append structured entry to `daily/YYYY-MM-DD.md` after writing per-project reflection
  - Effort: ~1 hour
  - *From: [[Claude Memory Compiler]] Tier B integration*

- [ ] **Tier C: Named-agent scoping** — tag compiled articles by Gunawan role
  - Trigger: when factory has 3+ live products
  - *From: [[Claude Memory Compiler]] Tier C integration*

---

## Related
- [[Action Items/MVP Sprint]] — what to finish first
- [[AI Software Factory/Vision/Vision & Roadmap]] — strategic context for Phase 2
