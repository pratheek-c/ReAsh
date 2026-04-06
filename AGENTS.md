<!-- GSD:project-start source:PROJECT.md -->
## Project

**ReAsh AI**

ReAsh AI is a fully open-source, TUI-based AI research agent for developers with GitHub Copilot subscriptions. It authenticates via OAuth Device Flow (no stored PAT), fetches available Copilot models dynamically, lets users conduct deep research through a chat interface, and generates structured output in five formats: DOCX, PPTX, XLSX, GSD Markdown, and PDF.

**Core Value:** A developer with GitHub Copilot can run `npx reash-ai` (or `bunx reash-ai`), authenticate once, and have a full research agent — with memory, sessions, and multi-format document generation — running entirely from their terminal with no external services beyond Copilot.

### Constraints

- **Runtime**: Node.js >=20 (for ESM, top-level await, `crypto.randomUUID`)
- **Bun**: Primary dev/install runtime; `bun run dev` for local development
- **npm compatibility**: dist/ must work with `node`, `npx`, `npm install -g`; no Bun-only APIs in source
- **No .env secrets**: Copilot token obtained at runtime via Device Flow; only `GITHUB_CLIENT_ID`, `TAVILY_API_KEY`, optional `GITHUB_TOKEN`, optional `WORKSPACE_PATH` in `.env`
- **Python optional**: PDF/XLSX require Python + `openpyxl pandas reportlab`; DOCX/PPTX/GSD work without Python (non-fatal warning at startup)
- **Open source**: MIT license; no proprietary deps; all skillsets from anthropics/skills (freely redistributable)
- **TypeScript strict**: `exactOptionalPropertyTypes`, `noUncheckedIndexedAccess`, `noImplicitOverride`
<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->
## Technology Stack

## ⚠️ Critical Version Alerts
### ink is at v6 — not v5
### @mastra/core requires Node >=22.13.0
### tavily package is deprecated
### @inkjs/ui v2 is NOT compatible with ink v6 / React 19 out of the box
## Recommended Stack
### Core Runtime & Build
| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| **Bun** | latest (1.2+) | Primary dev runtime | 10–20× faster install than npm; `bun run dev`, `bun test`; native TS execution without transpile step |
| **Node.js** | >=22.13.0 | Dist runtime target | Required by @mastra/core@1.22.0; npx/node users need this version |
| **TypeScript** | `^6.0.0` | Language | TS 6 is current; strict mode with `exactOptionalPropertyTypes`, `noUncheckedIndexedAccess`, `noImplicitOverride` as specified |
| **tsup** | `^8.5.1` | Build bundler | ESM output, treeshaking, handles CJS compat shim; esbuild-powered; no config needed for most cases |
| **`#!/usr/bin/env node`** | — | Shebang | Ensures `npx reash-ai` and `node dist/cli.js` both work; Bun respects this shebang too |
### Agent Framework (Mastra)
| Package | Version | Purpose | Why |
|---------|---------|---------|-----|
| `@mastra/core` | `^1.22.0` | Agent, Workflow, Mastra instance | Latest stable; includes `Agent`, `Workflow`, `createTool`, `Mastra` class; Apache-2.0 license |
| `@mastra/memory` | `^1.13.1` | ObservationalMemory + WorkingMemory | `Memory` class with `observationalMemory: true` and `workingMemory.enabled: true`; resource-scoped by default |
| `@mastra/libsql` | `^1.7.4` | Persistent storage | `LibSQLStore` with `url: 'file:~/.config/reash-ai/storage.db'`; WAL mode; zero-config SQLite |
### AI SDK / Copilot Gateway
| Package | Version | Purpose | Why |
|---------|---------|---------|-----|
| `@ai-sdk/openai-compatible` | `^2.0.39` | Base for CopilotGateway | Extends Vercel AI SDK v6 OpenAI-compatible provider; accepts custom baseURL + auth headers; `createOpenAICompatible()` factory |
| `zod` | `^4.3.6` | Schema validation | Mastra peer dep; structured output schemas; runtime validation for tool inputs/outputs |
### TUI
| Package | Version | Purpose | Why |
|---------|---------|---------|-----|
| `ink` | `^6.8.0` | React for terminal | Current stable (v5 does not exist on npm); React 19 peer dep; includes `alternateScreen`, `incrementalRendering` (v6.5+), `maxFps` (v6.3+) |
| `react` | `^19.0.0` | ink peer dep | Required by ink v6 |
| `@types/react` | `^19.0.0` | TypeScript types | Peer dep of ink v6 |
| `@inkjs/ui` | `^2.0.0` | Spinner, TextInput, SelectInput, etc. | Peer dep says `ink >= 5`, works at runtime with ink v6 + `--legacy-peer-deps`; if peer conflict is blocky, vendor the 3 components needed (~200 LoC) |
### Authentication
| Package | Version | Purpose | Why |
|---------|---------|---------|-----|
| `conf` | `^15.1.0` | Token persistence | Stores `~/.config/reash-ai/token.json`; JSON schema validation; atomic writes; cross-platform XDG-aware paths; Node >=20, pure ESM |
### Research Tools
| Package | Version | Purpose | Why |
|---------|---------|---------|-----|
| `@tavily/core` | `^0.7.2` | Web search | Official Tavily SDK (the `tavily` package is deprecated and redirects here); Returns structured search results suitable for agent consumption |
| `cheerio` | `^1.2.0` | HTML parsing / scraping | Fast jQuery-like DOM parse for static pages; Node >=20; ESM; pure JS (no native deps = Bun compatible) |
| `playwright` | `^1.59.1` | JS-rendered page scraping | For pages requiring JavaScript execution; spawned as child process, not inline (heavy dep, lazy-loaded) |
| `@octokit/rest` | `^22.0.1` | GitHub API | Official GitHub REST client; ESM; Node >=20; paginate + rate-limit built in |
| `pdf-parse` | `^2.4.5` | Local PDF text extraction | Pure TS, cross-platform, pdfjs-based; Node >=20; ESM; no native deps — Bun compatible |
### Document Generation
| Package | Version | Purpose | Why |
|---------|---------|---------|-----|
| `docx` | `^9.6.1` | DOCX generation | Pure JS, declarative API, no native deps, ESM + CJS dual export; Bun compatible |
| `pptxgenjs` | `^4.0.1` | PPTX generation | Pure JS, no native deps; ESM + CJS; Bun compatible; chart support |
| Python subprocess | — | XLSX + PDF generation | `gen_xlsx.py` (openpyxl + pandas) and `gen_pdf.py` (reportlab Platypus); invoked via `pythonBridge.ts`; optional — non-fatal warning at startup if Python missing |
### Storage
| Package | Version | Purpose | Why |
|---------|---------|---------|-----|
| `@mastra/libsql` | `^1.7.4` | SQLite storage via libSQL | Backed by `@libsql/client`; WAL mode; zero-config; sessions, threads, workflow snapshots all in one file |
### Linting & Formatting
| Package | Version | Purpose | Why |
|---------|---------|---------|-----|
| `@biomejs/biome` | `^2.4.10` | Linter + formatter | Replaces ESLint + Prettier in one binary; zero config; Rust-based (10–100× faster); `biome check --write .` |
### Testing
| Package | Version | Purpose | Why |
|---------|---------|---------|-----|
| `vitest` | `^4.1.2` | Unit + integration testing | Vite-based; native ESM; fastest watch mode; compatible with both `bun test` passthrough and `vitest run` |
| `ink-testing-library` | `^4.0.0` | TUI component testing | Renders ink components in-memory, checks output text; peer dep `ink >= 5` (works with v6 at runtime) |
## Installation
# Core runtime deps
# Dev deps
## Alternatives Considered
| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Agent framework | `@mastra/core` | LangChain JS | LangChain: bloated, weak TS types, 40MB+ dep tree, no first-class workflow |
| Agent framework | `@mastra/core` | LangGraph JS | LangGraph: graph-first, too much boilerplate for linear pipeline, same dep weight |
| TUI | `ink@6` | blessed / neo-blessed | Imperative API, no React model, poor TS support, unmaintained |
| TUI | `ink@6` | Charm Bubbletea | Go-only; different language ecosystem |
| XLSX | Python openpyxl | exceljs | exceljs: lower Excel compatibility, no pivot tables, JS-side formula limits |
| PDF | Python reportlab | jsPDF | jsPDF: canvas-based, no rich layout, poor text reflow |
| PDF | Python reportlab | Puppeteer print-to-PDF | 200MB+ chromium dep, overkill for document generation |
| Linter | Biome | ESLint + Prettier | Two tools, slower, more config surface area |
| Storage | `@mastra/libsql` | better-sqlite3 | Native binding, breaks npx cold-install |
| Storage | `@mastra/libsql` | nedb / lowdb | No WAL, no schema, no concurrent reads |
| Token storage | conf | keytar | Native binding, breaks bunx/npx on CI |
| Web search | `@tavily/core` | `tavily` (deprecated) | `tavily` npm package is deprecated, redirects to @tavily/core |
| HTTP | native fetch() | axios / node-fetch | Node 22 ships stable fetch(); no extra dep needed |
| TypeScript | v6.0.2 | v5.x | TS6 is current; no reason to pin old version |
## Bun Compatibility Summary
| Package | Bun Compatible? | Notes |
|---------|-----------------|-------|
| `@mastra/core` | ✅ Yes | Pure ESM, no native bindings |
| `@mastra/memory` | ✅ Yes | Pure ESM |
| `@mastra/libsql` | ✅ Yes | Uses WASM libSQL client, no native compile |
| `ink` | ✅ Yes | Pure ESM/JS |
| `react` | ✅ Yes | Pure JS |
| `@inkjs/ui` | ✅ Yes | Pure JS (peer dep conflict, not runtime issue) |
| `conf` | ✅ Yes | Pure ESM; uses fs |
| `@tavily/core` | ✅ Yes | HTTP-based, no native deps |
| `cheerio` | ✅ Yes | Pure JS |
| `playwright` | ⚠️ Partial | Browser binaries run as child_process; Bun can spawn playwright CLI fine |
| `@octokit/rest` | ✅ Yes | ESM, fetch-based |
| `pdf-parse` | ✅ Yes | Pure TS, WASM pdfjs, no native deps |
| `docx` | ✅ Yes | Pure JS, ESM |
| `pptxgenjs` | ✅ Yes | Pure JS, CJS+ESM |
| `@biomejs/biome` | ✅ Yes | Rust binary, runs independently |
| `vitest` | ✅ Yes | Runs via `bun run vitest` |
| Python bridge | ✅ Yes | `child_process.spawn('python3', ...)` works in Bun |
## Version Constraints Summary
## Sources
| Claim | Source | Confidence |
|-------|--------|------------|
| ink@6.8.0 is current; v5 does not exist | `registry.npmjs.org/ink/latest` + GitHub releases | HIGH |
| ink v6: React 19 peer dep, incrementalRendering in v6.5, maxFps in v6.3 | ink GitHub releases page | HIGH |
| `@mastra/core@1.22.0` requires Node >=22.13.0 | `registry.npmjs.org/@mastra/core/latest` | HIGH |
| `@mastra/memory@1.13.1` latest | `registry.npmjs.org/@mastra/memory/latest` | HIGH |
| `@mastra/libsql@1.7.4` latest | `registry.npmjs.org/@mastra/libsql/latest` | HIGH |
| `tavily` deprecated → `@tavily/core` | `registry.npmjs.org/tavily/latest` | HIGH |
| `@tavily/core@0.7.2` latest | `registry.npmjs.org/@tavily/core/latest` | HIGH |
| `@inkjs/ui@2.0.0` peer dep `ink >= 5`, React 18 devDep | `registry.npmjs.org/@inkjs/ui/latest` | HIGH |
| `@ai-sdk/openai-compatible@2.0.39` latest | `registry.npmjs.org/@ai-sdk/openai-compatible/latest` | HIGH |
| `docx@9.6.1`, `pptxgenjs@4.0.1`, `cheerio@1.2.0` | npm registry live data | HIGH |
| `conf@15.1.0`, `vitest@4.1.2`, `tsup@8.5.1` | npm registry live data | HIGH |
| `@biomejs/biome@2.4.10`, `typescript@6.0.2` | npm registry live data | HIGH |
| `@octokit/rest@22.0.1`, `playwright@1.59.1`, `pdf-parse@2.4.5` | npm registry live data | HIGH |
| Mastra storage docs (LibSQLStore, Memory) | mastra.ai official docs via MCP | HIGH |
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->
## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, or `.github/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
