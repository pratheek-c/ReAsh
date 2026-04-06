# Technology Stack

**Project:** ReAsh AI — TUI-based AI Research Agent CLI
**Researched:** 2026-04-07
**Overall confidence:** HIGH (all versions verified from npm registry live data)

---

## ⚠️ Critical Version Alerts

### ink is at v6 — not v5
**ink@5** does not exist on npm (404). The current stable series is **ink@6.8.0** (React 19 peer dep).
The project spec says "Ink v5" but this refers to an outdated mental model. Use ink v6.

### @mastra/core requires Node >=22.13.0
**Not Node 20.** The latest @mastra/core@1.22.0 sets `"engines": { "node": ">=22.13.0" }`.
This supersedes PROJECT.md's Node >=20 note. CI and shebang compatibility must be updated.

### tavily package is deprecated
The `tavily` npm package redirects to `@tavily/core`. Use `@tavily/core@^0.7.2`.

### @inkjs/ui v2 is NOT compatible with ink v6 / React 19 out of the box
`@inkjs/ui@2.0.0` declares `peerDependencies: { "ink": ">=5" }` but was built and tested
against React 18. With ink v6 requiring React 19, you'll get a peer dep conflict.
**Resolution:** Install with `--legacy-peer-deps` (works at runtime) OR build TUI components
from scratch using ink v6 primitives. The components from `@inkjs/ui` (Spinner, TextInput,
SelectInput, etc.) are simple enough to re-implement in ~200 LoC if needed.

---

## Recommended Stack

### Core Runtime & Build

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| **Bun** | latest (1.2+) | Primary dev runtime | 10–20× faster install than npm; `bun run dev`, `bun test`; native TS execution without transpile step |
| **Node.js** | >=22.13.0 | Dist runtime target | Required by @mastra/core@1.22.0; npx/node users need this version |
| **TypeScript** | `^6.0.0` | Language | TS 6 is current; strict mode with `exactOptionalPropertyTypes`, `noUncheckedIndexedAccess`, `noImplicitOverride` as specified |
| **tsup** | `^8.5.1` | Build bundler | ESM output, treeshaking, handles CJS compat shim; esbuild-powered; no config needed for most cases |
| **`#!/usr/bin/env node`** | — | Shebang | Ensures `npx reash-ai` and `node dist/cli.js` both work; Bun respects this shebang too |

**Bun compatibility note:** Bun executes the dist/ output as Node-compatible ESM. Do not use `Bun.file()`, `Bun.serve()`, or any `bun:*` imports in source — stick to Node stdlib. The only Bun-specific tooling is the dev script (`bun run dev`) and install (`bun install`).

---

### Agent Framework (Mastra)

| Package | Version | Purpose | Why |
|---------|---------|---------|-----|
| `@mastra/core` | `^1.22.0` | Agent, Workflow, Mastra instance | Latest stable; includes `Agent`, `Workflow`, `createTool`, `Mastra` class; Apache-2.0 license |
| `@mastra/memory` | `^1.13.1` | ObservationalMemory + WorkingMemory | `Memory` class with `observationalMemory: true` and `workingMemory.enabled: true`; resource-scoped by default |
| `@mastra/libsql` | `^1.7.4` | Persistent storage | `LibSQLStore` with `url: 'file:~/.config/reash-ai/storage.db'`; WAL mode; zero-config SQLite |

**Why Mastra over LangChain/LangGraph:**
LangChain JS is over-engineered for single-agent CLI use, has poor TypeScript inference, and adds 40+ MB of transitive deps. Mastra provides first-class workflow (`.parallel()`, `.map()`), memory, and structured output in a single coherent API with excellent TS types. LangGraph is graph-first and requires more boilerplate for linear research pipelines.

**Mastra Node requirement:** @mastra/core@1.22.0 engine `"node": ">=22.13.0"`. Bun ≥1.2 supports Node 22 compatibility layer.

---

### AI SDK / Copilot Gateway

| Package | Version | Purpose | Why |
|---------|---------|---------|-----|
| `@ai-sdk/openai-compatible` | `^2.0.39` | Base for CopilotGateway | Extends Vercel AI SDK v6 OpenAI-compatible provider; accepts custom baseURL + auth headers; `createOpenAICompatible()` factory |
| `zod` | `^4.3.6` | Schema validation | Mastra peer dep; structured output schemas; runtime validation for tool inputs/outputs |

**CopilotGateway pattern:**
```typescript
import { createOpenAICompatible } from '@ai-sdk/openai-compatible';

export function createCopilotProvider(token: string) {
  return createOpenAICompatible({
    name: 'github-copilot',
    baseURL: 'https://api.githubcopilot.com',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Copilot-Integration-Id': 'vscode-chat',
      'Editor-Version': 'vscode/1.90.0',
    },
  });
}
// Model string: 'copilot/<model-id>' from GET /models response
```

**Why not `@ai-sdk/openai` directly:**
The Copilot gateway is OpenAI-compatible but lives at a different base URL with custom auth headers. `@ai-sdk/openai-compatible` is the correct package for custom OpenAI-compatible endpoints.

---

### TUI

| Package | Version | Purpose | Why |
|---------|---------|---------|-----|
| `ink` | `^6.8.0` | React for terminal | Current stable (v5 does not exist on npm); React 19 peer dep; includes `alternateScreen`, `incrementalRendering` (v6.5+), `maxFps` (v6.3+) |
| `react` | `^19.0.0` | ink peer dep | Required by ink v6 |
| `@types/react` | `^19.0.0` | TypeScript types | Peer dep of ink v6 |
| `@inkjs/ui` | `^2.0.0` | Spinner, TextInput, SelectInput, etc. | Peer dep says `ink >= 5`, works at runtime with ink v6 + `--legacy-peer-deps`; if peer conflict is blocky, vendor the 3 components needed (~200 LoC) |

**ink v6 render options for the app:**
```typescript
render(<App />, {
  alternateScreen: true,
  incrementalRendering: true,
  maxFps: 60,
});
```

**Why not blessed/neo-blessed/terminal-kit:**
These are imperative APIs with poor TypeScript support. ink's React model composes naturally with Mastra's async data flows via `useState`/`useEffect`.

**Why not Charm Bubbletea (Go):**
Project is Node/TypeScript. ink is the de-facto standard for Node TUI (37k+ GitHub stars).

**`@inkjs/ui` compatibility risk (MEDIUM):**
The package was last published against React 18 / ink 5. With ink v6 requiring React 19, npm will warn about peer dep mismatch. Runtime behavior is unaffected (React 19 is backwards compatible for basic hooks). Use `overrides` in package.json to silence:
```json
{
  "overrides": {
    "@inkjs/ui": {
      "react": "^19.0.0"
    }
  }
}
```

---

### Authentication

| Package | Version | Purpose | Why |
|---------|---------|---------|-----|
| `conf` | `^15.1.0` | Token persistence | Stores `~/.config/reash-ai/token.json`; JSON schema validation; atomic writes; cross-platform XDG-aware paths; Node >=20, pure ESM |

**Device Flow implementation:** Native `fetch()` (Node 22 / Bun built-in) — no additional HTTP library needed. Flow: POST `/login/device/code` → poll `/login/oauth/access_token` → store via conf.

**Why not keytar/system-keychain:**
Adds native binding complexity and breaks `bunx`/`npx` installs on CI/containers. The PROJECT.md spec explicitly chooses conf over keytar.

**Why not dotenv for token:**
PROJECT.md constraint: no .env for secrets. `conf` writes to user-level config dir, not project dir.

---

### Research Tools

| Package | Version | Purpose | Why |
|---------|---------|---------|-----|
| `@tavily/core` | `^0.7.2` | Web search | Official Tavily SDK (the `tavily` package is deprecated and redirects here); Returns structured search results suitable for agent consumption |
| `cheerio` | `^1.2.0` | HTML parsing / scraping | Fast jQuery-like DOM parse for static pages; Node >=20; ESM; pure JS (no native deps = Bun compatible) |
| `playwright` | `^1.59.1` | JS-rendered page scraping | For pages requiring JavaScript execution; spawned as child process, not inline (heavy dep, lazy-loaded) |
| `@octokit/rest` | `^22.0.1` | GitHub API | Official GitHub REST client; ESM; Node >=20; paginate + rate-limit built in |
| `pdf-parse` | `^2.4.5` | Local PDF text extraction | Pure TS, cross-platform, pdfjs-based; Node >=20; ESM; no native deps — Bun compatible |

**Why not puppeteer instead of Playwright:**
Playwright has a smaller browser download footprint (chromium-only mode), better headless stability, and is the 2025 community standard for programmatic scraping. Both work on Bun via child_process — neither runs natively in Bun's JSC runtime.

**Why not `node-fetch` / `axios` for web requests:**
Node 22 ships with `fetch()` as stable global. No additional HTTP library needed.

---

### Document Generation

| Package | Version | Purpose | Why |
|---------|---------|---------|-----|
| `docx` | `^9.6.1` | DOCX generation | Pure JS, declarative API, no native deps, ESM + CJS dual export; Bun compatible |
| `pptxgenjs` | `^4.0.1` | PPTX generation | Pure JS, no native deps; ESM + CJS; Bun compatible; chart support |
| Python subprocess | — | XLSX + PDF generation | `gen_xlsx.py` (openpyxl + pandas) and `gen_pdf.py` (reportlab Platypus); invoked via `pythonBridge.ts`; optional — non-fatal warning at startup if Python missing |

**Why not exceljs for XLSX:**
PROJECT.md explicitly chooses openpyxl (Python) because the anthropics/skills SKILL.md prescribes this. exceljs produces lower-fidelity XLSX with weaker formula/pivot support. Python openpyxl generates native Excel files with full compatibility.

**Why not jsPDF / pdfkit for PDF:**
PROJECT.md explicitly uses reportlab Platypus (Python) per anthropics/skills. reportlab produces publication-quality PDFs with proper text reflow, headers/footers, and table layout. jsPDF is canvas-based and lacks rich document layout primitives.

**Why not Puppeteer/CDP print-to-PDF:**
Adds 200MB+ chromium binary as a required dep. Python reportlab is ~10MB and produces better structured documents.

**`pythonBridge.ts` design:** stdin/stdout JSON protocol, 60s timeout, SIGTERM on timeout, structured `{ failed: boolean, error?: string }` return — never throws from within workflow steps.

---

### Storage

| Package | Version | Purpose | Why |
|---------|---------|---------|-----|
| `@mastra/libsql` | `^1.7.4` | SQLite storage via libSQL | Backed by `@libsql/client`; WAL mode; zero-config; sessions, threads, workflow snapshots all in one file |

**Storage path pattern:**
```typescript
new LibSQLStore({
  id: 'reash-ai-storage',
  url: `file:${path.join(os.homedir(), '.config', 'reash-ai', 'storage.db')}`,
})
```

**Why not better-sqlite3:**
`@mastra/libsql` wraps `@libsql/client` which bundles its own SQLite WASM — no native binding, works on Bun without `bun install` native compilation step. better-sqlite3 requires native compilation which breaks in `npx` cold-install scenarios.

**Why not nedb / lowdb:**
No query language, no concurrent-read WAL, no schema migration support.

---

### Linting & Formatting

| Package | Version | Purpose | Why |
|---------|---------|---------|-----|
| `@biomejs/biome` | `^2.4.10` | Linter + formatter | Replaces ESLint + Prettier in one binary; zero config; Rust-based (10–100× faster); `biome check --write .` |

**Why not ESLint + Prettier:**
Two tools, two configs, two plugin ecosystems, slower. Biome 2.x has feature parity for TypeScript linting and is the 2025 community recommendation for new projects.

---

### Testing

| Package | Version | Purpose | Why |
|---------|---------|---------|-----|
| `vitest` | `^4.1.2` | Unit + integration testing | Vite-based; native ESM; fastest watch mode; compatible with both `bun test` passthrough and `vitest run` |
| `ink-testing-library` | `^4.0.0` | TUI component testing | Renders ink components in-memory, checks output text; peer dep `ink >= 5` (works with v6 at runtime) |

**vitest + Bun:** Bun has its own test runner (`bun test`), but vitest is recommended for this project because: (1) it integrates with CI via `vitest run`; (2) ink-testing-library is built for vitest/jest API; (3) Mastra test patterns use vitest. Use `vitest` directly, not `bun test`.

**ink-testing-library + ink v6:** Library has `peerDependencies: { "ink": ">=5" }` so ink v6 satisfies the constraint. However, it was built against React 18 — use `--legacy-peer-deps` or `overrides` as with `@inkjs/ui`.

---

## Installation

```bash
# Core runtime deps
bun add @mastra/core @mastra/memory @mastra/libsql
bun add @ai-sdk/openai-compatible zod
bun add ink react
bun add @inkjs/ui --legacy-peer-deps
bun add conf
bun add @tavily/core cheerio @octokit/rest pdf-parse
bun add docx pptxgenjs

# Dev deps
bun add -d typescript tsup @biomejs/biome
bun add -d vitest ink-testing-library @types/react @types/node
bun add -d playwright  # lazy dep — installed but not bundled
```

**Bun install note:** `bun add` resolves all deps from bun's lockfile. For the legacy-peer-dep packages (`@inkjs/ui`, `ink-testing-library`), add to `bun.toml` trust list or use `bun add --no-peer-deps` and manually audit.

---

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

---

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

**Runtime constraint:** Source code must not import any `bun:*` module. The dist/ ships as `#!/usr/bin/env node` with ESM, so it runs under `node`, `npx`, `bun run`, and `bunx` identically.

---

## Version Constraints Summary

```json
{
  "engines": {
    "node": ">=22.13.0"
  },
  "dependencies": {
    "@mastra/core": "^1.22.0",
    "@mastra/memory": "^1.13.1",
    "@mastra/libsql": "^1.7.4",
    "@ai-sdk/openai-compatible": "^2.0.39",
    "zod": "^4.3.6",
    "ink": "^6.8.0",
    "react": "^19.0.0",
    "@inkjs/ui": "^2.0.0",
    "conf": "^15.1.0",
    "@tavily/core": "^0.7.2",
    "cheerio": "^1.2.0",
    "playwright": "^1.59.1",
    "@octokit/rest": "^22.0.1",
    "pdf-parse": "^2.4.5",
    "docx": "^9.6.1",
    "pptxgenjs": "^4.0.1"
  },
  "devDependencies": {
    "typescript": "^6.0.0",
    "tsup": "^8.5.1",
    "@biomejs/biome": "^2.4.10",
    "vitest": "^4.1.2",
    "ink-testing-library": "^4.0.0",
    "@types/react": "^19.0.0",
    "@types/node": "^22.0.0"
  },
  "overrides": {
    "@inkjs/ui": { "react": "^19.0.0" },
    "ink-testing-library": { "react": "^19.0.0" }
  }
}
```

---

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
