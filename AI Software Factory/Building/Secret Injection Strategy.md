---
title: "Secret Injection Strategy"
type: architecture-decision
status: mvp-done / phase-2-designed
tags: [factory-building, secrets, security, supabase, paperclip]
source: ai-software-factory-conversation.md
date: 2026-04-22
---

# Secret Injection Strategy

How Paperclip injects secrets (API keys, DB credentials) into sandbox containers for dev/test use.

---

## MVP (Phase 1) — File-based

Agents write a `.env.sandbox` file in `/data/workspaces/<slug>/`. Orchestrator reads it on container start. Good enough for development and test secrets. Graduates to the encrypted store in Phase 2.

---

## Phase 2 Design — Supabase-backed Encrypted Store

**Schema:**
```sql
-- project_secrets table
project_slug   TEXT NOT NULL,
key            TEXT NOT NULL,
value_encrypted BYTEA NOT NULL,   -- encrypted via pgsodium
created_at     TIMESTAMPTZ,
updated_at     TIMESTAMPTZ
```

**Read path:** Orchestrator reads secrets when starting a sandbox container → decrypts via pgsodium → injects as environment variables.

**Write path:** Agents write secrets via a `secrets:set` skill → POSTs to orchestrator → orchestrator writes to the table via service role key.

**Human path:** You manage sensitive secrets via the admin dashboard — agents never handle real production keys.

**Why pgsodium:** Already in the ai-software-factory baseline migration. No new infrastructure — one table and two orchestrator functions (encrypt + decrypt).

---

## Hard Rule

> Never store production secrets in this system.

Paperclip builds things; **shipping is a human act** via existing CI/CD and Kubernetes Secrets. The `project_secrets` table is strictly for **development and sandbox environments**.

---

## Related
- [[Pipeline Model & Ticket Strategy]] — orchestrator context where secrets are consumed
- [[Main Index]] — Gap #6: `WEBHOOK_URL` rotating on ngrok restart (related secret-management problem in Santoso)
