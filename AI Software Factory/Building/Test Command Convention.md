---
title: "Test Command Convention"
type: architecture-decision
status: decided
tags: [factory-building, testing, sandbox, paperclip]
source: ai-software-factory-conversation.md
date: 2026-04-22
---

# Test Command Convention

How `sandbox:test` determines which command to run tests inside a container.

---

## Decision

1. Read `scripts.test` from the project's `package.json`
2. If it exists → `docker exec <container> pnpm test`
3. If not → `docker exec <container> pnpm test` (pnpm will fail helpfully with a clear message)
4. If that also fails → post a ticket comment asking the Engineer agent to add a test script

**No configuration needed. One line of logic.**

---

## Why

The ai-software-factory standards mandate `pnpm test` as the project-level test command (documented in `.claude/docs/qa-os/strategy.md`). Every project scaffolded from the factory already has this. For the rare project that doesn't, **failing with a clear message beats silently picking the wrong command**.

---

## Related
- [[Lifecycle - Project vs Feature]] — `qa:fix` step is where `sandbox:test` is called
- [[Personas & Skills Matrix]] — QA persona owns `qa:new-tests` and `qa:fix`
