# Research Summary: ReAsh AI

**Domain:** TUI-based AI research agent CLI (GitHub Copilot-exclusive, npm/npx distributed)
**Researched:** 2026-04-07
**Overall confidence:** HIGH

---

## Executive Summary

ReAsh AI is a single-user, open-source CLI tool that turns any GitHub Copilot subscription into a full research agent with memory, session persistence, and five output formats — no additional API keys required. The project targets developers who want a terminal-native alternative to ChatGPT/Claude for deep research tasks.

The stack is well-defined in `PROJECT.md` and fully verified against live npm registry data (see `STACK.md`). The most significant version discovery is that **ink@5 does not exist on npm** — the current stable series is ink@6 (React 19), and `@mastra/core@1.22.0` requires Node >=22.13.0 (not Node 20). Both are documented in `STACK.md` with mitigations.

The research workflow (plan → parallel research → synthesize → parallel file generation) maps cleanly onto Mastra's `.parallel()` + `.map()` + `.then()` primitives. The critical implementation discipline is that every parallel step's downstream `inputSchema` must key by each step's `id` string exactly — silent schema mismatches are the #1 avoidable runtime bug in this architecture. The full data flow and component boundary map is documented in `ARCHITECTURE.md`.

The pitfall surface is dominated by four concerns: (1) Mastra observational memory defaulting to `google/gemini-2.5-flash` instead of the CopilotGateway, (2) Ink v6 terminal cleanup on crash, (3) Bun Windows `setRawMode` flag overwrite, and (4) Python PATH detection on Windows. All four are preventable with explicit checks documented in `PITFALLS.md`. The dev machine is Windows, making the Bun/Windows-specific issues higher priority than they would be on macOS/Linux.

---

## Key Findings

**Stack:** Bun-primary (dev) + tsup ESM (dist) + ink@6 + Mastra (@mastra/core@1.22.0 requires Node >=22.13.0) + LibSQL/SQLite + Python subprocess for XLSX/PDF.

**Architecture:** Bootstrap → Ink TUI → Mastra (Agent + Workflow + Storage + Workspace) → stream events back to TUI; pythonBridge.ts for workflow steps calling Python; Workspace for agent ad-hoc tool use. See `ARCHITECTURE.md` for the full component boundary map and streaming pattern.

**Critical pitfall:** `observationalMemory: true` defaults to `google/gemini-2.5-flash` — NOT the CopilotGateway. Always pass an explicit config object with a Copilot-routed model. This would silently break the "no external API keys" value proposition at startup.

---

## Implications for Roadmap

Based on research, suggested phase structure:

1. **Auth + TUI Shell** — Device Flow OAuth, token persistence, bare Ink v6 shell with alternateScreen crash handler
   - Addresses: Device Flow auth, token persistence, model listing, basic chat input
   - Avoids: Ink crash leaving terminal in raw mode (Pitfall 3), Bun Windows setRawMode (Pitfall 4)
   - Foundation for everything else; unblocked by any other phase

2. **Memory & Sessions** — LibSQL storage, Mastra agent with memory, session thread management
   - Addresses: Session persistence, session browser, observational + working memory
   - Avoids: `observationalMemory: true` defaulting to Gemini (Pitfall 2), `mastra_resources` table mis-init (Pitfall 6), `resourceId` immutability (Pitfall 15)
   - Must come before Research Workflow because the workflow synthesize step writes to memory

3. **Research Workflow** — Mastra workflow with parallel fan-in (web + GitHub) → synthesize → DOCX + GSD output
   - Addresses: `/research` command, Tavily + Cheerio, Octokit GitHub research, synthesize, file generation
   - Avoids: `.parallel()` inputSchema keying (Pitfall 1), `{ failed }` pattern not checked downstream (Pitfall 19)
   - Validates the core value proposition before adding optional Python outputs

4. **Python Bridge & Extended Output** — pythonBridge.ts, gen_xlsx.py, gen_pdf.py, PPTX
   - Addresses: XLSX + PDF output, Python PATH detection, pythonBridge JSON protocol
   - Avoids: SIGTERM on Windows (Pitfall 12), Python PATH probe (Pitfall 13), `import.meta.url` in ESM dist (Pitfall 5)
   - Optional dep — non-fatal if Python absent; can ship after Phase 3 validates core workflow

5. **Polish & Publish** — Biome, CI matrix (Node 22+), npm publish, CONTRIBUTING.md, CHANGELOG.md
   - Addresses: tsup shebang (Pitfall 10), npm `files` field (Pitfall 17), GitHub Actions CI
   - Avoids: Publishing source files, missing shebang, wrong Node engine in `package.json`

**Phase ordering rationale:**
- Auth must be first: every other phase depends on a valid Copilot token
- Memory before Workflow: synthesize step requires a Mastra agent with memory configured; avoids rework if memory is wired incorrectly after the workflow is built
- Research Workflow before Python Bridge: validates the primary user journey (plan → research → DOCX/GSD) before adding optional complexity
- Polish last: CI, publishing, and documentation are straightforward once the codebase is stable

**Research flags for phases:**
- Phase 1: Standard patterns. Ink v6 + Device Flow are well-documented. Low research risk.
- Phase 2: Needs validation of `observationalMemory` model config against CopilotGateway in a real Mastra instance. May need a spike to confirm the CopilotGateway model string routes correctly for background agents.
- Phase 3: Parallel step schema keying should be tested with a minimal workflow before building the full research pipeline. The `.map()` helper usage with `getInitData()` for passing the original task through parallel steps needs a vitest proof-of-concept.
- Phase 4: Python PATH detection on Windows needs a real test on the dev machine. `child.kill()` vs `SIGTERM` behavior on Windows should be verified before relying on the 60s timeout.
- Phase 5: `npm pack --dry-run` must be a CI gate before any publish.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All versions verified from live npm registry data; ink version mismatch caught (v5→v6); Node version mismatch caught (20→22.13.0) |
| Features | HIGH | Reference-class analysis (Claude Code, Gemini CLI, Aider); table stakes and anti-features well-defined |
| Architecture | HIGH | All patterns verified against current official Mastra documentation via MCP; streaming event types confirmed |
| Pitfalls | MEDIUM-HIGH | Mastra-specific pitfalls are HIGH (from official docs); Bun/Windows pitfalls are MEDIUM (from issue tracker, may be fixed) |

---

## Gaps to Address

- **CopilotGateway model string for OM:** Whether `'copilot/github-copilot/gpt-4.1'` (or similar) actually routes through a custom `MastraModelGateway` subclass for Observational Memory's background agents needs a Phase 2 spike. The Mastra model router resolution for gateways during OM background processing is not explicitly documented.

- **Ink v6 `incrementalRendering` API contract:** The v6 render option names (`incrementalRendering`, `maxFps`) were confirmed via the STACK.md npm registry research but not tested with the exact Ink v6.8.0 API. Verify the option names haven't changed.

- **Bun 1.2+ Windows `setRawMode` fix:** Issue #25663 was reportedly fixed in Dec 2025. The current Bun version on the dev machine should be checked against the fix revision to confirm the issue is resolved before Phase 1.

- **`@mastra/libsql` WAL mode auto-enable:** Whether `LibSQLStore` enables WAL mode by default or requires explicit `PRAGMA journal_mode=WAL` needs verification during Phase 2 storage setup.

- **`conf@15` with custom `cwd`:** Whether `conf` version 15 supports a `cwd` override to force `~/.config/reash-ai/` on Windows (instead of AppData\Roaming) should be verified before using it in Phase 1. See Pitfall 20 in `PITFALLS.md`.
