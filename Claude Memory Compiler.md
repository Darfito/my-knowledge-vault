---
title: "Claude Memory Compiler"
type: tool-reference
tags: [institutional-memory, phase-6, feedback-os, hooks, shannon-framework]
source: "D:\\Side-Mission\\project-2\\refrensi\\claude-memory-compiler"
author: "coleam00 (adapted from Andrej Karpathy's LLM KB architecture)"
analyzed: 2026-04-18
---

# Claude Memory Compiler
*Technical reference for the AI Software Factory institutional memory layer.*

> **Relation to Index Utama:** This tool directly addresses [[Main Index]] Gap #17 (*"Reflections do not aggregate across projects"*) and Gap #20 (*"No commands for the knowledge layer itself"*). It is the mechanical implementation of Shannon Framework Phase 6: Feedback OS.

---

## 1. Executive Summary

The claude-memory-compiler is a **self-compiling, hook-driven knowledge base** that intercepts Claude Code sessions and converts raw conversation transcripts into structured, queryable institutional memory — automatically, without any human action.

### Why it is vital for the AI Software Factory

The Shannon Framework's Phase 6 (Feedback OS) has always been the *weakest link in the chain*. The `reflect-task` SKILL writes one reflection markdown per task, per project. Those reflections are:
- **Not aggregated** — they pile up per-project, never synthesised across the factory
- **Not injected** — a future session on `webapp-gunawan` has no awareness of lessons learned in `santoso-protocol`
- **Not queryable** — no mechanism exists to answer "what pattern did we use for RLS last time?"

The memory compiler solves all three in one system. It turns every Claude Code session into a contribution to a living, cross-referenced knowledge base. For the Gunawan AI Company, this means:

- Every Santoso Protocol task, every Quinn build, every Reviewer finding accumulates as institutional knowledge
- Dharmawan scraping XLSMART today informs the next Researcher agent working on a different product
- Hendrawan's bug pattern library grows automatically — the QA OS becomes smarter with each run
- The `reflect-task` skill output flows into a compiled knowledge base instead of being lost in per-project files

This is the difference between an AI operation with a **working memory** and one with **institutional memory**. The factory scales when agents can learn from prior agents' work without human curation.

---

## 2. Mechanism — Technical Deconstruction

### 2.1 System Architecture

```
Claude Code Session (any project)
        │
        ▼ (on SessionStart)
  session-start.py ──────────────── injects knowledge/index.md + recent daily log
        │                           into session as additional context (≤20,000 chars)
        │
  [Claude Code session runs]
        │
        ▼ (on PreCompact OR SessionEnd)
  session-end.py / pre-compact.py
        │  1. Recursion guard: exits if CLAUDE_INVOKED_BY is set
        │  2. Reads JSONL transcript from transcript_path
        │  3. Extracts last 30 user/assistant turns as markdown (≤15,000 chars)
        │  4. Writes context to temp file: scripts/session-flush-{id}-{ts}.md
        │  5. Spawns flush.py as background process (CREATE_NO_WINDOW on Windows)
        │
        ▼ (background, detached)
  flush.py
        │  1. Sets CLAUDE_INVOKED_BY=memory_flush (recursion prevention)
        │  2. Deduplicates: skips if same session_id flushed < 60s ago
        │  3. Calls Claude Agent SDK (allowed_tools=[], max_turns=2)
        │  4. LLM decides what's worth saving → structured daily log entry
        │  5. Appends to daily/YYYY-MM-DD.md
        │  6. If hour ≥ 18 AND today's log hash changed → spawns compile.py
        │
        ▼ (triggered after 6 PM, or manually)
  compile.py
        │  Uses Claude Agent SDK (allowed_tools=[Read,Write,Edit,Glob,Grep],
        │  permission_mode="acceptEdits", max_turns=30)
        │
        │  Prompt context: AGENTS.md schema + knowledge/index.md +
        │                  all existing wiki articles + today's daily log
        │
        │  LLM output (writes files directly):
        ├── knowledge/concepts/{slug}.md      ← new atomic knowledge articles
        ├── knowledge/connections/{slug}.md   ← cross-concept synthesis
        ├── knowledge/index.md               ← updated master catalog
        └── knowledge/log.md                 ← append-only build log
        │
        ▼ (next SessionStart)
  Knowledge injected into session → cycle repeats
```

### 2.2 The Three Tiers of Storage

#### Tier 1: `daily/` — Immutable Source (Raw Conversations)
```
daily/2026-04-18.md
```
Append-only. Written by `flush.py`. Each entry is timestamped and structured:
```markdown
### Session (22:37)

**Context:** Building Index Utama for the AI Software Factory vault.

**Key Exchanges:**
- Discovered that blueprint-analysis-proposal.pdf directly maps to Shannon Phases 4-5

**Decisions Made:**
- Wikilinks use note name only (Obsidian resolves by name, not path)

**Lessons Learned:**
- Windows mv fails on .git dirs; use PowerShell Move-Item instead
```
This is **the compiler's raw material** — equivalent to `data/xlsmart-products.json` in Dharmawan's output.

#### Tier 2: `knowledge/` — Compiled Knowledge (LLM-Owned)
```
knowledge/
├── index.md              ← primary retrieval mechanism (read at every SessionStart)
├── log.md                ← append-only build history
├── concepts/             ← atomic knowledge articles (YAML frontmatter + wiki)
├── connections/          ← cross-concept synthesis articles
└── qa/                   ← filed query answers (compounding value)
```
The LLM owns this directory. It creates, updates, and cross-references articles. A concept article looks like:
```yaml
---
title: "RLS Migration Pattern"
aliases: [row-level-security-migration]
tags: [supabase, security, database]
sources:
  - "daily/2026-04-18.md"
created: 2026-04-18
updated: 2026-04-18
---
```

#### Tier 3: `state.json` — Incremental Compilation Tracker
SHA-256 hashes of compiled daily logs prevent redundant compilation. Cost tracked per log ($0.45–0.65/compile). The compiler is **incremental by default** — only changed daily logs are reprocessed.

### 2.3 The Four Scripts

| Script | Claude Agent SDK? | Tools | Max turns | Role |
|--------|------------------|-------|-----------|------|
| `flush.py` | Yes | `[]` (no tools) | 2 | LLM triage: is this worth saving? |
| `compile.py` | Yes | Read, Write, Edit, Glob, Grep | 30 | LLM compiler: writes files directly |
| `query.py` | Yes | Read, Glob, Grep | 10 | Index-guided retrieval, no RAG |
| `lint.py` | Partially | (structural checks free) | — | 7 health checks |

### 2.4 The JSONL Transcript Pipeline (`session-end.py` internals)

Claude Code stores sessions as `.jsonl` files. The hook parses each line:
```python
entry = json.loads(line)
msg = entry.get("message", {})
role = msg.get("role", "")       # "user" or "assistant"
content = msg.get("content", "") # string or list of content blocks
```
Content blocks of type `"text"` are extracted; tool calls (`tool_use`, `tool_result`) are silently dropped. The result is a clean markdown conversation that `flush.py` can reason over in 2 LLM turns.

### 2.5 The Recursion Guard

The most architecturally important detail: when `compile.py` calls the Claude Agent SDK, the SDK internally spawns a Claude Code process. That process would normally fire `session-end.py` → spawning `flush.py` → spawning `compile.py` again — infinite recursion.

The guard is a single env var:
```python
# flush.py — set BEFORE any imports
os.environ["CLAUDE_INVOKED_BY"] = "memory_flush"
```
```python
# session-end.py — checked at top of file
if os.environ.get("CLAUDE_INVOKED_BY"):
    sys.exit(0)
```
This is the same pattern Shannon's `newborn-gate` SKILL uses to prevent agents from spawning agents past approved scope — *guard at the boundary*.

### 2.6 Why No RAG (Karpathy's Insight)

At 50–500 articles (the scale of a single AI Software Factory instance), the master `knowledge/index.md` fits easily in context. The LLM reading this index — understanding semantic relationships, synonyms, intent — outperforms cosine similarity on keyword overlap. RAG adds infrastructure complexity (vector DB, embeddings pipeline, embedding cost) for a quality degradation at this scale.

**The crossover point is ~2,000 articles / ~2M tokens.** The factory will not hit this for years.

---

## 3. Integration Strategy — Wiring into the Shannon Framework

### 3.1 Current State: Two Disconnected Feedback Mechanisms

```
TODAY:
  Task ends
    │
    ├── reflect-task SKILL runs (manual)
    │     └── writes docs/knowledge/reflections/REFLECTION-YYYY-MM-DD-slug.md
    │           (per-project, not aggregated, not injected back)
    │
    └── Session ends → nothing captured
```

```
TARGET:
  Task ends
    │
    ├── reflect-task SKILL runs (unchanged — still creates the ADR/reflection artifact)
    │     └── writes docs/knowledge/reflections/REFLECTION-YYYY-MM-DD-slug.md
    │           (per-project spec artifact, lives with the code)
    │
    └── Session ends → session-end.py fires (automatic)
          └── flush.py extracts + synthesises → daily/YYYY-MM-DD.md
                └── compile.py (after 6 PM) → knowledge/concepts/, connections/
                      └── session-start.py injects index into next session (automatic)
```

The two mechanisms are **complementary, not competing**:
- `reflect-task` → **project-scoped artifact** (goes in the repo, tracked in git, tied to the code that caused the learning)
- Memory compiler → **cross-project institutional memory** (aggregates patterns across all projects, injected into every future session)

### 3.2 Three Integration Tiers (Choose by Urgency)

#### Tier A — Zero-code integration (immediate, today)
Install the memory compiler in `D:\Side-Mission\project-2\refrensi\claude-memory-compiler\` and add its `.claude/settings.json` hooks to the **workspace-level** Claude Code settings. Every session in any project under this workspace then auto-captures.

```json
// Merge into: D:\Side-Mission\knowledge-vault\Claude-Knowledge\.claude\settings.json
{
  "hooks": {
    "SessionStart": [{ "matcher": "", "hooks": [{ "type": "command",
      "command": "uv run --directory D:\\Side-Mission\\project-2\\refrensi\\claude-memory-compiler python hooks/session-start.py",
      "timeout": 15 }] }],
    "PreCompact": [{ "matcher": "", "hooks": [{ "type": "command",
      "command": "uv run --directory D:\\Side-Mission\\project-2\\refrensi\\claude-memory-compiler python hooks/pre-compact.py",
      "timeout": 10 }] }],
    "SessionEnd": [{ "matcher": "", "hooks": [{ "type": "command",
      "command": "uv run --directory D:\\Side-Mission\\project-2\\refrensi\\claude-memory-compiler python hooks/session-end.py",
      "timeout": 10 }] }]
  }
}
```
**Effect:** Every Claude Code session anywhere in the workspace starts contributing to `daily/` and eventually `knowledge/` — zero changes to any agent manifest or skill.

#### Tier B — reflect-task bridge (recommended, 1 hour of work)
Extend the `reflect-task` SKILL (`mobile-gunawan/.claude/skills/reflect-task/SKILL.md`) to append a compiler-formatted summary to the day's daily log after writing the per-project reflection artifact:

```markdown
## Compiler Bridge (add to reflect-task SKILL)

After writing the REFLECTION-*.md file, append a structured entry to the
shared memory compiler daily log:

File: D:\Side-Mission\project-2\refrensi\claude-memory-compiler\daily\YYYY-MM-DD.md

Entry format:
### Reflect-Task Bridge (HH:MM) — {task-slug}

**Context:** {one-line task description} — {project} repo, {role} agent

**Decisions Made:**
- {each item from "Key Decisions Made" in the reflection}

**Lessons Learned:**
- {each item from "What failed or was missing"}
- {each item from "Proposed foundation improvement"}

**Action Items:**
- {open questions that remain unresolved}
```

This makes `reflect-task` the **write path** into the compiler and the `session-start.py` hook the **read path** back, closing the Feedback OS loop across all projects.

#### Tier C — Named agent scoping (future, when factory has 3+ live products)
As additional Santoso-Protocol-style products come online, each will have 7 named agents running in separate sessions. The compiler's daily log will have entries from Dharmawan, Quinn, Hendrawan, etc. across multiple products.

Extend `compile.py`'s prompt to tag articles by **Gunawan role**:
```yaml
# In a concept article frontmatter:
tags: [pattern, qa-pattern, hendrawan-qa]
contributing_agents: ["hendrawan-qa", "quinn-engineer"]
```
This allows `query.py` to answer: *"What QA patterns has Hendrawan discovered across all projects?"*

### 3.3 The `newborn-gate` Connection

The memory compiler's `knowledge/index.md`, when injected at `SessionStart`, acts as an extension of the **context loading order** defined in the `newborn-gate` SKILL:

```
Current context loading order (from newborn-gate SKILL):
  1. CLAUDE.md
  2. foundation/human-intent-os/
  3. foundation/agent-foundation-os/
  4. foundation/role-definition-os/
  5-9. docs/knowledge/ ADRs, patterns, anti-patterns

Proposed extension (after compiler is active):
  0. [SessionStart injection] knowledge/index.md + recent daily log   ← NEW
  1. CLAUDE.md
  ... (existing order unchanged)
```

Layer 0 arrives automatically via the hook — no change to the gate checklist needed. The gate already validates `docs/knowledge/README.md` (step 7) — the compiler's `knowledge/index.md` is the cross-project equivalent.

### 3.4 Operational Costs

At the Gunawan AI Company's current scale (3 active projects, ~5 sessions/day):

| Operation | Frequency | Est. cost |
|-----------|-----------|-----------|
| Flush (per session) | ~5/day | ~$0.10–0.25/day |
| Compile (once/day after 6 PM) | 1/day | ~$0.45–0.65/day |
| Query (on-demand) | ~2–3/week | ~$0.30–0.75/week |
| Lint (weekly) | 1/week | ~$0.15–0.25/week |
| **Total estimate** | | **~$17–30/month** |

Runs on Claude Max subscription — no separate API billing required. The Claude Agent SDK uses `~/.claude/.credentials.json` directly.

---

## 4. Phase 6 Mapping — Shannon Framework Alignment

### Shannon Phase 6: Feedback OS

Defined in `webapp-gunawan/.gunawan/foundation/feedback-os/` (contents not yet written — another gap). The reflect-task SKILL is the only concrete Phase 6 artifact that exists today.

The memory compiler is **what Phase 6 should look like when fully operational**:

| Shannon Phase 6 concept | Memory Compiler implementation |
|-------------------------|-------------------------------|
| Task reflection | `flush.py` — LLM decides what's worth extracting |
| Pattern capture | `compile.py` → `knowledge/concepts/` |
| Anti-pattern library | `compile.py` → `knowledge/connections/` (cross-concept contradictions) |
| Institutional memory | `knowledge/index.md` injected at every SessionStart |
| Feedback loop closure | `session-start.py` → knowledge context → better decisions → `session-end.py` → richer daily log → better articles |
| Cross-agent learning | All agents' sessions compile into the same `daily/` directory |
| Health checks | `lint.py` — broken links, orphans, contradictions, staleness |

The AGENTS.md file in the compiler is the *compiler specification* — the "schema" that tells the LLM-as-compiler how to turn raw daily logs into structured knowledge. This is architecturally equivalent to Shannon's six `.gunawan/foundation/*/` directories, which tell each agent how to behave within their phase.

---

## 5. File Reference

| Path | Purpose |
|------|---------|
| `claude-memory-compiler/AGENTS.md` | Schema + complete technical reference (the "compiler spec") |
| `claude-memory-compiler/hooks/session-start.py` | SessionStart hook — injects `knowledge/index.md` + recent daily log |
| `claude-memory-compiler/hooks/session-end.py` | SessionEnd hook — extracts JSONL transcript → spawns flush.py |
| `claude-memory-compiler/hooks/pre-compact.py` | PreCompact safety net — same as session-end, fires before context compaction |
| `claude-memory-compiler/scripts/flush.py` | Background process — LLM triage (2 turns, no tools) → appends to daily/ |
| `claude-memory-compiler/scripts/compile.py` | Background compiler — LLM writes concept articles directly (30 turns, file tools) |
| `claude-memory-compiler/scripts/query.py` | Index-guided Q&A — no RAG, reads index + relevant articles |
| `claude-memory-compiler/scripts/lint.py` | 7 health checks: broken links, orphans, stale, contradictions |
| `claude-memory-compiler/scripts/config.py` | Path constants (`DAILY_DIR`, `KNOWLEDGE_DIR`, `CONCEPTS_DIR`, etc.) |
| `claude-memory-compiler/scripts/utils.py` | Shared helpers: `file_hash`, `slugify`, `extract_wikilinks`, `list_wiki_articles` |
| `claude-memory-compiler/.claude/settings.json` | Hook configuration (merge into project or workspace settings) |
| `claude-memory-compiler/pyproject.toml` | `claude-agent-sdk>=0.1.29`, `python-dotenv>=1.0.0`, Python 3.12+ |

---

## 6. Related Notes

- [[Main Index]] — Full AI Software Factory knowledge base index (Gaps §5.4 addressed here)
- [[00 - Project Overview]] — Shannon Framework phases; Phase 6 (Feedback OS) is what this compiler operationalises
- [[Repo - webapp-gunawan]] — Source of `reflect-task` SKILL and `newborn-gate` SKILL to be bridged
- [[Repo - santoso-protocol]] — First live product whose agent sessions will contribute to daily/ logs
- [[Santoso Protocol - VPS Memory Export]] — Active session context (VPS agents run compile targets)
