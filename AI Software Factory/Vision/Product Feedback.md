---
title: "Product Feedback — AI Software Factory"
type: feedback-log
tags: [factory-vision, feedback, ux]
date: 2026-04-22
---

# Product Feedback — AI Software Factory

Feedback collected during real usage sessions. Each item is a signal for factory improvement.

---

## [2026-04-22] Token visibility during Q&A sessions

**Feedback:** The factory has no way for users to track input/output token consumption during or after a command session. Users can't see how much context each command burns or estimate costs.

**Why it matters:** AI Software Factory commands involve heavy context loading (CLAUDE.md + project-state.md + spec files + standards docs). Token costs add up fast. Without visibility, users can't optimize their workflow or know when they're approaching limits.

**Root cause:** The JSONL session transcripts already contain full token data (`input_tokens`, `output_tokens`, `cache_read_input_tokens`, `cache_creation_input_tokens`) per API call — but the session-end hook never extracts it. Factory commands (markdown files) have no mechanism to report token counts since Claude doesn't know its own token count during generation.

**Solution implemented (2026-04-22):**
- Modified `session-end.py` to extract cumulative token totals from JSONL and write to `tokens-latest.json`
- Modified `flush.py` to include a Token Usage section in every daily log entry
- Modified `session-start.py` to inject previous session's token totals at the start of each new session
- Effect: users see token usage from the previous session at the top of every new session context, and token totals appear in every daily log entry

**Remaining gap:** Real-time token count during a session is not possible from within commands (Claude doesn't have access to its own API usage metadata during generation). The end-of-session reporting is the closest approximation.

---
