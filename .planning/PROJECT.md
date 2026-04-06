# ReAsh AI

## What This Is

ReAsh AI is a fully open-source, TUI-based AI research agent for developers with GitHub Copilot subscriptions. It authenticates via OAuth Device Flow (no stored PAT), fetches available Copilot models dynamically, lets users conduct deep research through a chat interface, and generates structured output in five formats: DOCX, PPTX, XLSX, GSD Markdown, and PDF.

## Core Value

A developer with GitHub Copilot can run `npx reash-ai` (or `bunx reash-ai`), authenticate once, and have a full research agent — with memory, sessions, and multi-format document generation — running entirely from their terminal with no external services beyond Copilot.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] GitHub Copilot OAuth Device Flow authentication (no PAT stored in .env)
- [ ] Token persistence to `~/.config/reash-ai/token.json` via `conf`
- [ ] Dynamic model fetching from `GET https://api.githubcopilot.com/models`
- [ ] Ink v5 TUI with chat, progress, files, sessions, and model selector panels
- [ ] `/research <task>` command triggers Mastra workflow
- [ ] Observational Memory (thread-scoped) + Working Memory (resource-scoped) on researchAgent
- [ ] Session threads — list, resume, create new
- [ ] Research workflow: plan → parallel research (web + github) → synthesize → generate files
- [ ] Web search via Tavily, web scraping via Cheerio + Playwright
- [ ] GitHub research via Octokit
- [ ] Local file reading (txt, md, pdf) via pdf-parse
- [ ] Output generators: DOCX (`docx` npm), PPTX (`pptxgenjs`), XLSX (Python/openpyxl), GSD Markdown (native fs), PDF (Python/reportlab)
- [ ] Mastra Workspace (LocalFilesystem + LocalSandbox + skills/) for agent file/shell access
- [ ] MIT license, GitHub Actions CI (lint + typecheck + test), CONTRIBUTING.md, CHANGELOG.md
- [ ] Published as npm package: `npx reash-ai` works; `bunx reash-ai` preferred

### Out of Scope

- OAuth providers other than GitHub Copilot — v1 targets Copilot-only auth
- Web UI / Electron app — TUI-only for v1
- Team/multi-user features — single-user local tool
- Real-time collaboration — not relevant for research CLI
- Cloud sync of sessions — LibSQL local file only
- Mobile app — CLI tool, terminal-first

## Context

- **Framework**: Mastra (`@mastra/core`, `@mastra/memory`, `@mastra/libsql`) for agent orchestration, memory, and workflows
- **TUI**: Ink v5 + `@inkjs/ui`, `alternateScreen: true`, `incrementalRendering: true`, `maxFps: 60`
- **Auth API**: GitHub Copilot OAuth — `POST https://github.com/login/device/code`, scope `copilot`
- **Models API**: `GET https://api.githubcopilot.com/models` with headers `Copilot-Integration-Id: vscode-chat`, `Editor-Version: vscode/1.90.0`
- **Gateway**: Custom `CopilotGateway extends MastraModelGateway` using `@ai-sdk/openai-compatible-v5`; model string format `copilot/github-copilot/<model-id>`
- **Storage**: `LibSQLStore` (SQLite file via `@mastra/libsql`), WAL mode
- **Workspace**: `Workspace` from `@mastra/core/workspace` — `LocalFilesystem` + `LocalSandbox` pointed at `./output/`; skills auto-loaded from `./skills/`
- **Workflow**: Mastra workflow with `.parallel()` for research fan-in and file generation fan-out; `.map()` to propagate initial input through parallel steps
- **Document generation**: DOCX/PPTX via Node.js npm packages; XLSX/PDF via Python subprocess (`gen_xlsx.py` using openpyxl+pandas, `gen_pdf.py` using reportlab Platypus); Python bridge via `pythonBridge.ts`; skills from `anthropics/skills`
- **Build runtime**: **Bun-primary** — `bun run dev`, `bun install`; dist/ ships as ESM with `#!/usr/bin/env node` shebang, compatible with `node`, `npm`, `npx`, `bunx`
- **Linting**: Biome (replaces ESLint + Prettier)
- **Build**: `tsup` (ESM, treeshake)
- **Testing**: `vitest` + `ink-testing-library`
- **Error strategy**: Workflow steps never throw — return `{ failed: boolean, error?: string }` pattern

## Constraints

- **Runtime**: Node.js >=20 (for ESM, top-level await, `crypto.randomUUID`)
- **Bun**: Primary dev/install runtime; `bun run dev` for local development
- **npm compatibility**: dist/ must work with `node`, `npx`, `npm install -g`; no Bun-only APIs in source
- **No .env secrets**: Copilot token obtained at runtime via Device Flow; only `GITHUB_CLIENT_ID`, `TAVILY_API_KEY`, optional `GITHUB_TOKEN`, optional `WORKSPACE_PATH` in `.env`
- **Python optional**: PDF/XLSX require Python + `openpyxl pandas reportlab`; DOCX/PPTX/GSD work without Python (non-fatal warning at startup)
- **Open source**: MIT license; no proprietary deps; all skillsets from anthropics/skills (freely redistributable)
- **TypeScript strict**: `exactOptionalPropertyTypes`, `noUncheckedIndexedAccess`, `noImplicitOverride`

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Mastra for agent orchestration | First-class workflow, memory, structured output, gateway — avoids LangChain complexity | — Pending |
| GitHub Copilot via Device Flow | No PAT storage; works for any Copilot subscriber without additional API keys | — Pending |
| Ink v5 TUI over web UI | True CLI tool; zero browser dependency; distributes as single npm binary | — Pending |
| Bun-primary, npm-compatible | Faster dev loop; `bun install` is 10-20× faster; dist/ stays node-compatible for npx users | — Pending |
| Python subprocess for XLSX/PDF | anthropics/skills prescribes openpyxl+pandas and reportlab; better than JS ports; optional dep | — Pending |
| pythonBridge.ts over workspace sandbox for workflow | Workflow steps need strict stdin/stdout JSON protocol, 60s timeout, SIGTERM, structured error result — cannot use workspace tool calls from within step execute() | — Pending |
| LocalFilesystem + LocalSandbox workspace | Agents can use file tools and execute Python scripts ad-hoc; skills/ SKILL.md auto-loaded | — Pending |
| Biome over ESLint+Prettier | Single tool, zero config, faster; same quality output | — Pending |
| LibSQL (SQLite) storage | Zero-config local DB; WAL mode for concurrent reads; no external DB server needed | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-04-07 after initialization*
