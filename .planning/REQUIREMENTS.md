# Requirements: ReAsh AI

**Defined:** 2026-04-07
**Core Value:** GitHub Copilot subscribers can run `/research <topic>` from the terminal and get a structured, multi-format research document — no extra API keys, no setup beyond `npm install -g reash-ai`

## v1 Requirements

### Distribution

- [ ] **DIST-01**: User can install globally via `npm install -g reash-ai` or `bun add -g reash-ai`
- [ ] **DIST-02**: User can start the application by entering `reash` in any terminal
- [ ] **DIST-03**: User can run `reash --version` to see the installed version
- [ ] **DIST-04**: User can run `reash --help` to see available commands and flags

### Authentication

- [ ] **AUTH-01**: User can authenticate via GitHub Copilot OAuth Device Flow (browser-based; no PAT required)
- [ ] **AUTH-02**: User sees a `user_code` prominently during Device Flow with a browser prompt
- [ ] **AUTH-03**: User's auth token persists across sessions (`~/.config/reash-ai/token.json`)
- [ ] **AUTH-04**: User is transparently re-authenticated on token expiry (401 refresh flow)

### Model Management

- [ ] **MODL-01**: App fetches available Copilot models dynamically at startup from the Copilot API
- [ ] **MODL-02**: User can switch active model via `/model` command in the chat interface
- [ ] **MODL-03**: User's selected model persists across sessions

### Chat Interface

- [ ] **CHAT-01**: User can type multi-line messages (Shift+Enter for newline) in the chat input
- [ ] **CHAT-02**: User sees AI responses rendered token-by-token as they stream in
- [ ] **CHAT-03**: Chat auto-scrolls to the latest message unless the user has scrolled up
- [ ] **CHAT-04**: User can run `/help` to see all available slash commands
- [ ] **CHAT-05**: User can run `/exit` to quit the application cleanly
- [ ] **CHAT-06**: User can run `/clear` to clear the current chat history
- [ ] **CHAT-07**: User can run `!<command>` to execute a shell command and see output in chat

### Research Workflow

- [ ] **RSCH-01**: User can trigger a structured research workflow via `/research <topic>`
- [ ] **RSCH-02**: Research workflow runs web search (Tavily), GitHub research (Octokit), and local file reading in parallel
- [ ] **RSCH-03**: User sees a live task tree (tasuku) showing each parallel research branch's status during execution
- [ ] **RSCH-04**: User can inject local file/directory context into a research prompt via `@<path>` tokens
- [ ] **RSCH-05**: Research workflow follows plan → parallel research → synthesis → output fan-out steps orchestrated by Mastra

### Output Generation

- [ ] **OUTP-01**: Research results are saved as a GSD Markdown file in the output directory
- [ ] **OUTP-02**: Research results are saved as a DOCX file in the output directory
- [ ] **OUTP-03**: Research results are saved as a PPTX file in the output directory
- [ ] **OUTP-04**: Research results are saved as an XLSX file in the output directory (requires Python)
- [ ] **OUTP-05**: Research results are saved as a PDF file in the output directory (requires Python)
- [ ] **OUTP-06**: App detects Python availability at startup and gracefully marks XLSX/PDF as unavailable if Python is absent
- [ ] **OUTP-07**: User can configure the output directory via `WORKSPACE_PATH` env var (default `./output/`)

### Sessions

- [ ] **SESS-01**: Each research conversation is saved as a named session in a local LibSQL database
- [ ] **SESS-02**: User can list past sessions via `/sessions` showing timestamps and topics
- [ ] **SESS-03**: User can resume a past session via `/resume <id>` continuing from where they left off
- [ ] **SESS-04**: User can browse and search sessions via a keyboard-navigable session browser panel
- [ ] **SESS-05**: Observational memory retains research style/preferences per thread and surfaces relevant past context

### Reliability & UX

- [ ] **RELY-01**: Pressing Ctrl+C cancels any in-flight workflow step gracefully and shows partial results
- [ ] **RELY-02**: Every error message includes what failed, why it failed, and what the user can do next
- [ ] **RELY-03**: Working directory and output path are visible in the UI at all times

---

## v2 Requirements

### Output Generation

- **OUTP-V2-01**: XLSX/PDF output without Python (pure JS fallback)

### Research Workflow

- **RSCH-V2-01**: GitHub repository deep-research: search issues, PRs, README, code, and commits via Octokit and surface structured repo insights
- **RSCH-V2-02**: Observational memory surfaces cross-session research patterns (e.g., "you previously researched topic X in session Y")

### Chat Interface

- **CHAT-V2-01**: Session browser with fuzzy search over session titles and topics

### Distribution

- **DIST-V2-01**: Published to npm with automated CI/CD release pipeline

---

## Out of Scope

| Feature | Reason |
|---------|--------|
| PAT / API key auth for Copilot | Defeats zero-setup value prop; OAuth Device Flow only |
| Web UI / Electron app | Scope creep; loses CLI-native identity; bloated distribution |
| Cloud session sync | Requires backend; violates local-first promise; adds GDPR surface |
| Multi-user / team features | Single-developer tool; RBAC complexity out of proportion |
| Plugin / extension marketplace | Maintenance burden disproportionate to v1 benefit |
| Voice input | Not a research-oriented interaction model; adds dep weight |
| Real-time collaboration / shared sessions | Not the use case; local single-user sessions only |
| Auto-git-commit of outputs | Presumptuous; user decides what to commit |
| Built-in LLM fine-tuning | Out of scope; Copilot doesn't expose fine-tuning API |
| GUI file picker | Breaks terminal-only contract |
| Streaming to remote endpoint / webhook | Adds network surface; not needed for local tool |
| Custom theming system | Low-value for a research tool; single clean dark theme |

---

## Traceability

*Populated by roadmap creation*

| Requirement | Phase | Status |
|-------------|-------|--------|
| DIST-01 | — | Pending |
| DIST-02 | — | Pending |
| DIST-03 | — | Pending |
| DIST-04 | — | Pending |
| AUTH-01 | — | Pending |
| AUTH-02 | — | Pending |
| AUTH-03 | — | Pending |
| AUTH-04 | — | Pending |
| MODL-01 | — | Pending |
| MODL-02 | — | Pending |
| MODL-03 | — | Pending |
| CHAT-01 | — | Pending |
| CHAT-02 | — | Pending |
| CHAT-03 | — | Pending |
| CHAT-04 | — | Pending |
| CHAT-05 | — | Pending |
| CHAT-06 | — | Pending |
| CHAT-07 | — | Pending |
| RSCH-01 | — | Pending |
| RSCH-02 | — | Pending |
| RSCH-03 | — | Pending |
| RSCH-04 | — | Pending |
| RSCH-05 | — | Pending |
| OUTP-01 | — | Pending |
| OUTP-02 | — | Pending |
| OUTP-03 | — | Pending |
| OUTP-04 | — | Pending |
| OUTP-05 | — | Pending |
| OUTP-06 | — | Pending |
| OUTP-07 | — | Pending |
| SESS-01 | — | Pending |
| SESS-02 | — | Pending |
| SESS-03 | — | Pending |
| SESS-04 | — | Pending |
| SESS-05 | — | Pending |
| RELY-01 | — | Pending |
| RELY-02 | — | Pending |
| RELY-03 | — | Pending |

**Coverage:**
- v1 requirements: 38 total
- Mapped to phases: 0 (pending roadmap creation)
- Unmapped: 38

---
*Requirements defined: 2026-04-07*
*Last updated: 2026-04-07 after initial definition*
