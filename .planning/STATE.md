# STATE: ReAsh AI

**Last updated:** 2026-04-07
**Milestone:** v1 — Terminal-native AI research agent for GitHub Copilot subscribers

---

## Project Reference

**Core Value:** GitHub Copilot subscribers can run `/research <topic>` from the terminal and get a structured, multi-format research document — no extra API keys, no setup beyond `npm install -g reash-ai`

**Stack:** Bun-primary (dev) + tsup ESM (dist) + Ink v6 + Mastra (@mastra/core@1.22.0, Node >=22.13.0) + LibSQL/SQLite + Python subprocess for XLSX/PDF

**Repository:** D:\Pratheek\reash-ai

---

## Current Position

**Current phase:** Phase 1 — Project Scaffolding & Distribution Bootstrap
**Current plan:** None (not started)
**Status:** Not started

**Progress:**
```
Phase 1 [          ]   0% — Not started
Phase 2 [          ]   0% — Not started
Phase 3 [          ]   0% — Not started
Phase 4 [          ]   0% — Not started
Phase 5 [          ]   0% — Not started
Phase 6 [          ]   0% — Not started
─────────────────────────────────────────
Overall [          ]   0% — 0/6 phases complete
```

---

## Performance Metrics

| Metric | Value |
|--------|-------|
| Phases defined | 6 |
| Requirements mapped | 38/38 |
| Plans created | 0 |
| Plans completed | 0 |
| Phases completed | 0 |

---

## Accumulated Context

### Key Decisions (from research)

| Decision | Rationale |
|----------|-----------|
| Ink v6 (not v5) | ink@5 does not exist on npm; current stable is v6 (React 19) |
| Node >=22.13.0 (not 20) | @mastra/core@1.22.0 requires Node >=22.13.0 |
| `observationalMemory` requires explicit model config | Defaults to google/gemini-2.5-flash, not CopilotGateway — would silently break "no extra API keys" value prop |
| pythonBridge.ts over workspace sandbox for workflow | Workflow steps need strict stdin/stdout JSON protocol, 60s timeout, SIGTERM, structured error result |
| LibSQL WAL mode | May require explicit `PRAGMA journal_mode=WAL` — verify during Phase 4 |
| `conf@15` Windows path | Verify `cwd` override forces `~/.config/reash-ai/` on Windows before Phase 2 implementation |

### Todos (carry forward into planning)

- [ ] Phase 2: Verify `conf@15` supports `cwd` override to `~/.config/reash-ai/` on Windows (Pitfall 20)
- [ ] Phase 2: Spike CopilotGateway model string for Observational Memory background agents
- [ ] Phase 3: Verify Ink v6.8.0 option names (`incrementalRendering`, `maxFps`) before implementing TUI
- [ ] Phase 3: Verify Bun 1.2+ Windows `setRawMode` fix status on dev machine
- [ ] Phase 4: Verify `@mastra/libsql` WAL mode auto-enable vs explicit PRAGMA
- [ ] Phase 5: Test `.parallel()` + `.map()` schema keying with a minimal workflow spike before full research pipeline
- [ ] Phase 6: Test Python PATH detection on Windows dev machine; verify `child.kill()` SIGTERM behavior

### Blockers

*None at project start*

### Phase Transition Notes

*Populated at each phase transition*

---

## Session Continuity

**How to resume:**
1. Read this file to understand current position
2. Read `.planning/ROADMAP.md` for full phase structure and success criteria
3. Read `.planning/REQUIREMENTS.md` for requirement details and traceability
4. Start next action: `/gsd-plan-phase 1` to plan Phase 1

**Next action:** `/gsd-plan-phase 1` — Plan Phase 1: Project Scaffolding & Distribution Bootstrap

---
*STATE initialized: 2026-04-07*
