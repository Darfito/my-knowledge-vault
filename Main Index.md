# Main Index — AI Software Factory Knowledge Base
*Initialized: 2026-04-18 · Architect: Senior AI Knowledge Architect · Source library: `D:\Side-Mission\project-2\refrensi`*

> **Purpose.** This index is the entry point to the AI Software Factory reference library. It maps two PDF source documents (the *theory*) onto the working Shannon/Gunawan markdown corpus (the *implementation*) and the Santoso Protocol live deployment (the *demonstration*). Every wikilink below resolves to a concrete artifact in this vault.

---

## 1. Library Topography

```
refrensi/
├── blueprint-analysis-proposal.pdf          ← THEORY: critique of AINE 9-layer Blueprint
├── product-vision-plp-software-module.pdf   ← COMMERCIAL VEHICLE: PLP × Factory fusion
├── ian/                                      ← NARRATIVE LAYER (Obsidian notes)
│   ├── 00 - Project Overview.md
│   ├── 00 - Project Index (v2).md
│   ├── Repo - webapp-gunawan.md
│   ├── Repo - mobile-gunawan.md
│   ├── Repo - santoso-protocol.md
│   ├── Santoso Protocol - VPS Memory Export.md
│   └── Guknowledge/                          ← REPO MIRRORS (raw .claude/ trees)
│       ├── gunawan-agents/
│       │   ├── webapp-gunawan/               ← web governance source-of-truth
│       │   ├── mobile-gunawan/               ← mobile mirror (canonical agents/roles/skills live here)
│       │   └── santoso-protocol/             ← live product (named personas live here)
│       └── santoso-protocol-hq/              ← marketing + xlsmart-advisor product code
├── ai-software-factory/                      ← Russell's reference repo (source of /commands)
├── paperclip/                                ← orchestrator runtime (port 3100)
├── obsidian/, docs/                          ← supporting materials
```

Top-level entry notes:
- [[00 - Project Overview]] — Shannon framework, 6 phases, maturity ladder, working environment, naming convention
- [[00 - Project Index (v2)]] — canonical master index of the `ian/` notes (supersedes `00 - Project Index`)
- [[Santoso Protocol - VPS Memory Export]] — VPS state, agent identity rules, cold-boot order, project facts

---

## 2. Conceptual Map — Why These Documents Are Related

The four document classes form a **theory → methodology → implementation → demonstration → commercialisation** chain. Each link is load-bearing.

```
                       ┌─────────────────────────────────────┐
                       │  blueprint-analysis-proposal.pdf    │
                       │  (Russell Otniel · 2026-04-06)      │
                       │  • Reviews the AINE 9-layer Blueprint
                       │  • Defines AAI / VCM / GMM / ASEM   │
                       │  • Critical invariant:              │
                       │      autonomy ≤ verification × govern.│
                       │  • Proposes 7 design principles     │
                       └────────────────┬────────────────────┘
                                        │ adapts theory into…
                                        ▼
        ┌───────────────────────────────────────────────────────────┐
        │  Shannon Framework (the methodology)  → [[00 - Project Overview]] │
        │  6 phases:                                                │
        │    1. Human Intent OS     (global)                        │
        │    2. Agent Foundation OS (global)                        │
        │    3. Role Definition OS  (global)                        │
        │    4. Design OS           (per-project)                   │
        │    5. Build OS            (per-project)                   │
        │    6. Feedback OS         (per-project)                   │
        └───────────────┬───────────────────────────────────────────┘
                        │ instantiated by…
                        ▼
   ┌───────────────────────────────────────────────────────────────────┐
   │  Gunawan AI Company (the operation)                               │
   │  ├── [[Repo - webapp-gunawan]] — web governance (Next.js 16)      │
   │  ├── [[Repo - mobile-gunawan]] — mobile mirror (Expo + RN)        │
   │  └── [[Repo - santoso-protocol]] — live product                   │
   │         ↓                                                         │
   │  symlink: santoso-protocol/.gunawan → ../webapp-gunawan/.gunawan  │
   └───────────────┬───────────────────────────────────────────────────┘
                   │ proven in production by…
                   ▼
   ┌───────────────────────────────────────────────────────────────────┐
   │  XLSMART Package Advisor (live demo)                              │
   │  • Orchestrated by Santoso (Paperclip on VPS, port 3100)          │
   │  • 7 specialist agents, $5–15 budget per task, Telegram bridge    │
   │  • See: [[Santoso Protocol - VPS Memory Export]] for live state   │
   └───────────────┬───────────────────────────────────────────────────┘
                   │ packaged for sale via…
                   ▼
        ┌─────────────────────────────────────┐
        │  product-vision-plp-software-module.pdf │
        │  (Russell Otniel · 2026-04-06)      │
        │  • Fuses NexusPLM (regulated PLM)   │
        │    with the AI Software Factory     │
        │  • Adds Software Development Module │
        │    URS → Spec → Code → Verified → Deploy │
        │  • Headless Claude execution layer  │
        │  • Target: Kalbe + Pharma / MedDev / Fintech │
        └─────────────────────────────────────┘
```

### Why the Blueprint Analysis is the upstream document
The [[blueprint-analysis-proposal]] PDF isolates the *transferable* governance ideas from the AINE textbook (specification-first development, verification coverage as a metric, risk-proportional governance, traceability) and rejects the *operationally heavy* ones (GraphRAG, formal verification lanes, AAI as continuous decimal, full 9-layer service mesh). Every "non-negotiable rule" in [[Repo - webapp-gunawan]] (RLS-in-same-migration, `safeParse()` only, `ActionResult<T>`, `'use cache'` over `unstable_cache`) is a concrete instantiation of the Blueprint's *Specification* and *Verification & Assurance* layers — adapted for full-stack web at small-team scale.

### Why the Product Vision is the downstream document
The [[product-vision-plp-software-module]] PDF takes the *working* Shannon/Gunawan factory and proposes wrapping it as a Software Development Module bolted onto NexusPLM. Its "URS → Spec → Code → Verified → Deployed" pipeline is **the same lifecycle** as Shannon's Phases 4 → 5 → 6, but exposed through a UI and audit log targeted at GxP-regulated industries. The Verification Dashboard, Governance Gates, and Audit Trail features in the Product Vision map 1:1 to the Reviewer agent, the newborn gate, and the reflect-task skill that already exist in the codebase.

### The Santoso Protocol is the empirical proof
[[Repo - santoso-protocol]] is not a separate concept — it is the **falsification test** for the entire stack. If a 7-agent team coordinated by Paperclip can ship a working product to PT XLSMART in <90 min for <$60, the Blueprint's *risk-proportional governance* thesis is validated and the PLP product becomes a sellable asset. If the bridge fails or agents drift, the Blueprint's invariant (`AAI ≤ VCM × GMM`) explains why and points at the missing control.

---

## 3. Operational Assets

The `ian/Guknowledge/` tree contains **three discoverable kinds of operational asset**: Roles, Commands, and Skills. They live primarily under `mobile-gunawan/.claude/` (the canonical mirror) and `santoso-protocol/.claude/` (the named-persona overrides). The `webapp-gunawan/.claude/` mirror duplicates commands and skills but, per the open items in [[Santoso Protocol - VPS Memory Export]], its `agents/` directory is missing — a known gap.

### 3.1 Roles — Who an agent *is*

Three orthogonal layers define identity. Read them in order: preamble → role manifest → persona.

#### Layer A — Universal preamble (injected into every sub-agent)
- `mobile-gunawan/.claude/agents/_preamble.md` — Non-negotiable rules, Decision Priority Framework (Correctness > Maintainability > Security > Simplicity > Optimisation > Convenience), Output Contract, protected files
- `mobile-gunawan/.claude/agents/noahs-ark.md` — Shannon Protocol reference: YAGNI → KISS → DRY, directory responsibilities, Server Action structure, naming conventions, hard bans

#### Layer B — Generic agent manifests (`mobile-gunawan/.claude/agents/`)
Capability-typed sub-agents that any orchestrator can spawn. Each declares Identity, Scope (allowed / not allowed), and Output Format.

| Manifest | Gunawan Category | Mandate |
|----------|------------------|---------|
| `architect.md` | Thinker + Reviewer | ADRs, API contracts, schema design, implementation plans. **No code.** |
| `builder.md` | Builder | Implements approved plans. Writes/modifies files. No scope creep. |
| `explorer.md` | Thinker (read-only) | Discovery, file search, data-flow tracing. No writes. |
| `reviewer.md` | Reviewer | Audits Builder output against checklist. Approves/rejects. No fixes. |
| `devops.md` | Builder + Reviewer | CI/CD, Kubernetes, deployment readiness. Never touches `src/`. |

#### Layer C — Role state files (`mobile-gunawan/.claude/roles/`)
Persistent state per role: maturity level, gate status, decisions, open questions. The Newborn Team Rule says *all roles default to Level 0 (Born)*; promotion requires Noah's explicit approval via `/roles:promote`.

| Role file | Maps to manifest |
|-----------|------------------|
| `system-architect.md` | `architect.md` |
| `software-engineer.md` | `builder.md` |
| `qa-reviewer.md` | `reviewer.md` |
| `devops-platform.md` | `devops.md` |
| `product-strategist.md` | (Claude Code itself, as orchestrator) |
| `active.md` | Pointer to currently-active role |

#### Layer D — Named personas (`santoso-protocol/.claude/agents/`)
The 7 production agents that orchestrate the XLSMART Package Advisor. Each has a budget, an output directory, and a hand-off chain. Defined in [[Repo - santoso-protocol]].

| Persona file | Role | Budget | Output |
|-------------|------|--------|--------|
| `dharmawan-researcher.md` | Researcher (first agent on every project) | $3–5 | `data/xlsmart-products.json` |
| `gunawan-analyst.md` | Analyst — recommendation matrix + Claude API prompt | $3–5 | `analysis/recommendation-matrix.md` |
| `langston-designer.md` | Designer — markdown UI specs (no code, no Figma) | $3–5 | `design/ui-spec.md` |
| `quinn-engineer.md` | Engineer — bootstraps `.gunawan/`, builds Next.js app | $10–15 | `src/` |
| `hendrawan-qa.md` | QA — 5 scenarios + responsive + edge cases | $3–5 | `test/test-report.md` |
| `alpha-devops.md` | DevOps — Docker + ngrok/Caddy → public HTTPS URL | $3–5 | live URL |
| `linawati-marketing.md` | Marketing — copy, LinkedIn, cold email, SEO, one-pager | $3–5 (Phase 1) + $1–2 (Phase 2) | `marketing-assets/` |

Hand-off chain: Dharmawan → Gunawan → Langston → Quinn → Hendrawan → Alpha; Linawati runs in parallel with Gunawan and again after Alpha.

### 3.2 Commands — What an agent *does*

Slash commands live under `mobile-gunawan/.claude/commands/` (and are mirrored, with minor drift, into `webapp-gunawan/.claude/commands/`). They are organised into **OS modules** that match the Shannon Phases.

| OS Module | Slash Commands | Purpose |
|-----------|----------------|---------|
| `foundation/` | `bootstrap-foundation`, `init`, `discover`, `plan`, `shape-spec`, `inject-standards`, `status`, `validate` | Phase 1–3 scaffolding. `bootstrap-foundation` is the *first* command on any new install. |
| `architecture-os/` | `new-feature`, `review` | Phase 4 — schema design, ADRs, API contracts |
| `design-os/` | `import`, `system` | Phase 4 — design tokens, design system import |
| `implementation-os/` | `new-feature`, `review` | Phase 5 — feature build + Builder review |
| `data-fetching-os/` | `review` | Phase 5 — TanStack Query / Server Action audit |
| `qa-os/` | `new-tests`, `fix` | Phase 6 — test generation + bug-fix loop |
| `deployment-os/` | `k8s-config`, `release` | Phase 6 — k3s manifests, semantic-release tag |
| `roles/` | `promote`, `switch`, `sync` | Maturity ladder management. Only Noah may promote. |

The `ai-software-factory/.claude/commands/` directory at the refrensi root contains an additional canonical set used by Russell's reference repo; the mobile/webapp mirrors are derivatives.

### 3.3 Skills — How an agent *gates and reflects*

Two SKILLs ship with every project. Both are mandatory; both implement explicit phases of the Shannon framework.

| Skill | Phase | Trigger | Purpose |
|-------|-------|---------|---------|
| `mobile-gunawan/.claude/skills/newborn-gate/SKILL.md` | Phase 2 (Agent Foundation OS) | **Before** every workflow that writes/modifies/deletes files | Verifies foundation integrity, declares role, classifies task, lists assumptions, identifies protected files in scope, identifies approval gates, applies Decision Priority Framework. **Outputs `NEWBORN GATE: PASSED` or `BLOCKED`.** |
| `mobile-gunawan/.claude/skills/reflect-task/SKILL.md` | Phase 6 (Feedback OS) | **After** every substantive task | Writes `docs/knowledge/reflections/REFLECTION-YYYY-MM-DD-slug.md`. May also generate ADRs, patterns, or postmortems. Proposes (never auto-applies) foundation improvements. |

Together they form the **bookends of every agent task**: gate → role manifest → command → reflection.

---

## 4. Cross-Cutting Governance Constants

These values appear in every manifest, command, and skill — they are the operating contract.

- **Maturity Ladder.** 0 Born → 1 Infant → 2 Child → 3 Adolescent → 4 Teen/Junior → 5 Adult. Default = 0. Only Noah promotes.
- **14 Non-Negotiables.** RLS-in-same-migration, no `SECURITY DEFINER` outside `private`, no session vars for tenant, no `service_role` in client, `safeParse()` only, typed `ActionResult<T>` returns, no `unstable_cache`, no committed secrets, no direct push to `main`/`dev`, semantic-release tags drive production, newborn gate always runs, no protected file edits, no silent assumptions, no self-promotion.
- **Protected Files.** `CLAUDE.md`, `.gunawan/**`, `.env*`, `supabase/migrations/**`, `.github/workflows/**`, `k8s/**`, `.claude/settings.json`.
- **Address Ian as.** `Ian` (default), `Cao Cao` (commanding), `Zhuge Liang` (tactical). Never "boss/sir/user/operator/human". English-only voices.

---

## 5. Gaps & Opportunities

What is missing today to achieve a *fully automated* AI Software Factory — derived by intersecting the Blueprint Analysis recommendations, the Product Vision MVP feature list, and the open items in [[Santoso Protocol - VPS Memory Export]] / [[Repo - santoso-protocol]].

### 5.1 Implementation gaps in the existing factory
1. **`webapp-gunawan/.claude/agents/` does not exist.** Quinn's bootstrap copies `.gunawan/` but the agents/ directory mirroring is incomplete. The `cp` instruction in Quinn's manifest will fail. Listed as open in [[Santoso Protocol - VPS Memory Export]].
2. **Santoso `status=error` in Paperclip.** The orchestrator itself is degraded. No automation can be trusted until run logs are investigated.
3. **Telegram bridge not running.** Manual `tmux bridge` start required after every reboot; no systemd unit, no health check, no auto-restart.
4. **Paperclip patches not automated.** `analyst` and `marketing` AGENT_ROLES enum patches must be re-applied after every `paperclipai` reinstall — currently a manual `dist/index.js` edit.
5. **ngrok URL rotates on restart.** `WEBHOOK_URL` is updated by hand; no script reads the new ngrok URL and rewrites `.env`.
6. **No `cwd` pinned for 6 of 7 agents.** Only Quinn has a working directory. Non-determinism risk.

### 5.2 Concepts from the Blueprint Analysis that are *not yet* implemented
7. **Risk Zones.** Principle 1 of the Blueprint Analysis proposal. The factory currently applies one governance model uniformly; there is no `.claude/project-config.json` mapping paths → zones, and no per-zone test mandate.
8. **VCM (Verification Coverage Metric) as a measurable.** The Reviewer checklist is binary pass/fail. There is no aggregate verification-coverage number tracked over time — the Blueprint's central quality signal is absent.
9. **GMM (Governance Maturity Metric) tracking.** Newborn-gate runs, but its history is not stored or charted. No way to detect governance regressions.
10. **AAI (Agentic Autonomy Index) tracking.** Maturity levels are per-role and discrete (good — matches the Blueprint's "false precision" critique), but there is no rollup that prevents promoting Role X past `AAI ≤ GMM × VCM`.
11. **Provenance signing / SBOM / structured audit log.** The Blueprint's *Governance & Security* layer; the Product Vision's *Audit Trail* feature. Currently only ad-hoc Paperclip task logs.
12. **Headless execution layer.** Required by the Product Vision (Spec Editor → Code Generation Trigger → Verification Dashboard). Currently every Claude Code session is interactive; no job-queue + status protocol exists.

### 5.3 Bridge-to-product gaps (Product Vision PLP requirements)
13. **No PLP traceability matrix integration.** URS-REQ-NNN ↔ SPEC ↔ IMPL ↔ TEST ↔ REVIEW ↔ DEPLOY chain is described in the Product Vision but no schema, no migration, no `src/lib/traceability.ts` exists.
14. **No Spec Editor / Feature Backlog UI.** The PLP module needs a Kanban-style view backed by `project-state.md`; not yet specified inside any role manifest.
15. **No Risk Zone Configuration UI.** Mentioned in the Product Vision feature table but no design-os or implementation-os command emits it.
16. **No qualified-approver sign-off flow for GxP projects.** The Product Vision lists this as a Deployment Checklist requirement; the existing `roles/promote.md` is the closest analogue but is human-eyeball-only.

### 5.4 Cross-project / institutional-memory gaps
17. ~~**Reflections do not aggregate across projects.**~~ **→ BEING ADDRESSED by [[Claude Memory Compiler]].** The `claude-memory-compiler` tool (hook-driven, Claude Agent SDK, Python 3.12) auto-captures every session via `SessionEnd`/`PreCompact` hooks, extracts knowledge via a 2-turn LLM flush, compiles daily logs into structured `knowledge/concepts/` and `knowledge/connections/` articles after 6 PM, and re-injects `knowledge/index.md` at every `SessionStart`. The remaining integration step is bridging the `reflect-task` SKILL to append structured entries to the compiler's `daily/YYYY-MM-DD.md` after writing per-project reflection artifacts. See [[Claude Memory Compiler]] for the full technical spec and three-tier integration strategy.
18. **Mobile / web framework drift is undocumented.** [[Repo - mobile-gunawan]] lists "Key Differences from webapp-gunawan" but there is no automated diff or convergence target.
19. **No `noahs-ark.md` for mobile-specific rules.** The web Shannon Protocol reference is injected into mobile sub-agents unmodified; mobile-specific bans (e.g., "no `useEffect+fetch`") live only in the repo overview note, not in an injectable manifest.
20. **No commands for the `00 - Project Overview` knowledge layer itself.** The narrative Obsidian notes in `ian/` are human-maintained; no `/foundation:export-vault` command keeps them synchronized with the live repos.

### 5.5 Highest-leverage next moves (synthesis)
- **Unblock Santoso first** (gaps 2, 3, 4, 5) — no automation matters until the orchestrator is stable.
- **Implement Risk Zones + VCM aggregation** (gaps 7, 8) — the lowest-cost Blueprint concepts that unlock the safety invariant.
- **Fork the agents/ directory into `webapp-gunawan/`** (gap 1) — five-minute fix that prevents the next product from breaking on bootstrap.
- **Spec the headless execution layer** (gap 12) — prerequisite for any PLP integration; turns the factory into a callable service rather than an interactive ritual.
- ~~**Stand up a cross-project reflection ingest** (gap 17)~~ → **IN PROGRESS:** [[Claude Memory Compiler]] provides the ingest infrastructure (hooks + flush + compile pipeline). Remaining work: bridge `reflect-task` SKILL to append to compiler's `daily/` log (Tier B integration, ~1 hour). Install workspace-level hooks today (Tier A, zero code).

---

## 6. Source Document Index (Wikilink Reference)

### PDFs (theory & commercialisation)
- [[blueprint-analysis-proposal.pdf]] — *AI-Native Engineering Platform: Analysis & Enhancement Proposal* (Russell Otniel, 2026-04-06)
- [[product-vision-plp-software-module.pdf]] — *Product Vision: PLP Software Development Module* (Russell Otniel, 2026-04-06)

### Tools & Active Solutions (this vault)
- [[Claude Memory Compiler]] — Hook-driven institutional memory system. Implements Shannon Phase 6 (Feedback OS) as running infrastructure. Addresses Gaps #17 and #20. Source at `refrensi/claude-memory-compiler/`. Three-tier integration strategy: (A) workspace hooks today, (B) reflect-task bridge, (C) named-agent tagging in compile.py.

### Architecture Knowledge (this vault — `AI Software Factory/Building/`)
- [[Single-Agent vs Multi-Agent — Patterns & Tradeoffs]] — Practical breakdown of single vs multi-agent design: when to use each, quality tradeoffs, architecture rules, and how the hierarchical pattern maps to the factory's Orchestrator-Worker model. Source: Ida Silfverskiöld / Data Science Collective, Oct 2025.

### Narrative notes (`ian/`)
- [[00 - Project Overview]]
- [[00 - Project Index (v2)]]
- [[Repo - webapp-gunawan]]
- [[Repo - mobile-gunawan]]
- [[Repo - santoso-protocol]]
- [[Santoso Protocol - VPS Memory Export]]

### Operational asset roots (`ian/Guknowledge/gunawan-agents/`)
- `mobile-gunawan/.claude/agents/` — generic agent manifests + preamble
- `mobile-gunawan/.claude/roles/` — role state files
- `mobile-gunawan/.claude/commands/` — slash commands by OS module
- `mobile-gunawan/.claude/skills/newborn-gate/SKILL.md`
- `mobile-gunawan/.claude/skills/reflect-task/SKILL.md`
- `santoso-protocol/.claude/agents/` — 7 named personas
- `webapp-gunawan/.claude/commands/` — web mirror (agents/ missing — see Gap 1)

---

*End of index. Re-run synthesis when any of the source PDFs, the `ian/` narrative notes, or the `mobile-gunawan/.claude/` canonical tree change.*
