# Roadmap: ReAsh AI

**Milestone:** v1 — Terminal-native AI research agent for GitHub Copilot subscribers
**Created:** 2026-04-07
**Granularity:** Standard (5-8 phases)
**Coverage:** 38/38 v1 requirements mapped

---

## Phases

- [ ] **Phase 1: Project Scaffolding & Distribution Bootstrap** — Installable binary that boots, prints version/help, and exits cleanly
- [ ] **Phase 2: Authentication** — GitHub Copilot Device Flow OAuth with persistent token storage
- [ ] **Phase 3: Model Management & Chat Interface** — Live Ink v6 TUI with model selection, streaming chat, slash commands, and always-visible workspace path
- [ ] **Phase 4: Sessions & Memory** — LibSQL-backed session persistence, session browser, observational and working memory on the research agent
- [ ] **Phase 5: Research Workflow** — `/research <topic>` triggers Mastra plan→parallel-research→synthesize pipeline with graceful cancellation and actionable errors
- [ ] **Phase 6: Output Generation** — Research results fan out to GSD Markdown, DOCX, PPTX, XLSX, and PDF with Python detection and configurable output path

---

## Phase Details

### Phase 1: Project Scaffolding & Distribution Bootstrap
**Goal**: A developer can install and invoke `reash` as a global binary and confirm it works before any features exist
**Depends on**: Nothing (first phase)
**Requirements**: DIST-01, DIST-02, DIST-03, DIST-04
**Success Criteria** (what must be TRUE):
  1. `npm install -g reash-ai` and `bun add -g reash-ai` complete without errors and place a `reash` binary on PATH
  2. Running `reash` in any terminal launches the application (even if it shows only a placeholder shell)
  3. `reash --version` prints the correct semver version from `package.json`
  4. `reash --help` prints all available commands and flags without error
**Plans**: TBD
**UI hint**: yes

### Phase 2: Authentication
**Goal**: Users can authenticate once with GitHub Copilot via Device Flow and stay authenticated across sessions
**Depends on**: Phase 1
**Requirements**: AUTH-01, AUTH-02, AUTH-03, AUTH-04
**Success Criteria** (what must be TRUE):
  1. On first launch, the TUI presents a Device Flow `user_code` and a browser URL prominently, prompting the user to authorize
  2. After authorizing in the browser, the CLI detects approval and continues without manual input
  3. On subsequent launches, the user is NOT prompted to authenticate again (token loaded from `~/.config/reash-ai/token.json`)
  4. When a token expires mid-session, the app silently refreshes and continues the interrupted operation without user action
**Plans**: TBD

### Phase 3: Model Management & Chat Interface
**Goal**: Users can select a Copilot model and conduct a streaming multi-line chat conversation in the TUI
**Depends on**: Phase 2
**Requirements**: MODL-01, MODL-02, MODL-03, CHAT-01, CHAT-02, CHAT-03, CHAT-04, CHAT-05, CHAT-06, CHAT-07, RELY-03
**Success Criteria** (what must be TRUE):
  1. At startup, available Copilot models are fetched dynamically and the default model is shown in the UI; the last-used model is remembered across restarts
  2. Typing `/model` opens a model selector; picking a model immediately updates the active model shown in the UI
  3. User can type multi-line messages using Shift+Enter and send with Enter; responses stream token-by-token into the chat panel
  4. Chat auto-scrolls to the latest message unless the user has manually scrolled up
  5. `/help`, `/exit`, `/clear`, and `!<command>` all function correctly; working directory and output path are visible in the UI at all times
**Plans**: TBD
**UI hint**: yes

### Phase 4: Sessions & Memory
**Goal**: Every research conversation is persisted in a local database and users can browse, search, and resume past sessions
**Depends on**: Phase 3
**Requirements**: SESS-01, SESS-02, SESS-03, SESS-04, SESS-05
**Success Criteria** (what must be TRUE):
  1. After a chat conversation, closing and reopening `reash` shows the session in `/sessions` list with a timestamp and inferred topic
  2. Running `/resume <id>` reopens a past session and restores the full conversation history
  3. The session browser panel is keyboard-navigable and allows searching past sessions without leaving the TUI
  4. The research agent surfaces relevant context from past threads (observational memory) when a new topic overlaps with previous research
**Plans**: TBD
**UI hint**: yes

### Phase 5: Research Workflow
**Goal**: Users can run `/research <topic>` and get a synthesized research result produced by a parallel Mastra workflow with live status and resilient error handling
**Depends on**: Phase 4
**Requirements**: RSCH-01, RSCH-02, RSCH-03, RSCH-04, RSCH-05, RELY-01, RELY-02
**Success Criteria** (what must be TRUE):
  1. Typing `/research <topic>` starts a structured workflow: the TUI shows a live task tree with plan, parallel web/GitHub/file branches, and synthesis status updating in real time
  2. All three research branches (Tavily web search, Octokit GitHub research, local file reading) run in parallel and their results are merged in the synthesis step
  3. Prefixing a research prompt with `@<path>` injects the referenced file or directory contents as context into the research task
  4. Pressing Ctrl+C during any workflow step cancels gracefully and displays whatever partial results have been collected
  5. Every failure surfaces a message that names what failed, why it failed, and what the user can try next
**Plans**: TBD

### Phase 6: Output Generation
**Goal**: Completed research is automatically saved to multiple structured document formats in a configurable output directory, with graceful degradation when Python is unavailable
**Depends on**: Phase 5
**Requirements**: OUTP-01, OUTP-02, OUTP-03, OUTP-04, OUTP-05, OUTP-06, OUTP-07
**Success Criteria** (what must be TRUE):
  1. After a research workflow completes, GSD Markdown and DOCX files appear in the output directory automatically
  2. PPTX is generated and saved to the output directory as part of the same fan-out step
  3. If Python is available, XLSX and PDF files are also generated; if Python is absent, the app shows a non-fatal warning at startup and marks those formats as unavailable without blocking other outputs
  4. Setting `WORKSPACE_PATH` to a custom directory causes all output files to appear in that directory instead of the default `./output/`
**Plans**: TBD

---

## Progress

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Project Scaffolding & Distribution Bootstrap | 0/? | Not started | - |
| 2. Authentication | 0/? | Not started | - |
| 3. Model Management & Chat Interface | 0/? | Not started | - |
| 4. Sessions & Memory | 0/? | Not started | - |
| 5. Research Workflow | 0/? | Not started | - |
| 6. Output Generation | 0/? | Not started | - |

---

## Coverage Map

| Requirement | Phase |
|-------------|-------|
| DIST-01 | Phase 1 |
| DIST-02 | Phase 1 |
| DIST-03 | Phase 1 |
| DIST-04 | Phase 1 |
| AUTH-01 | Phase 2 |
| AUTH-02 | Phase 2 |
| AUTH-03 | Phase 2 |
| AUTH-04 | Phase 2 |
| MODL-01 | Phase 3 |
| MODL-02 | Phase 3 |
| MODL-03 | Phase 3 |
| CHAT-01 | Phase 3 |
| CHAT-02 | Phase 3 |
| CHAT-03 | Phase 3 |
| CHAT-04 | Phase 3 |
| CHAT-05 | Phase 3 |
| CHAT-06 | Phase 3 |
| CHAT-07 | Phase 3 |
| RELY-03 | Phase 3 |
| SESS-01 | Phase 4 |
| SESS-02 | Phase 4 |
| SESS-03 | Phase 4 |
| SESS-04 | Phase 4 |
| SESS-05 | Phase 4 |
| RSCH-01 | Phase 5 |
| RSCH-02 | Phase 5 |
| RSCH-03 | Phase 5 |
| RSCH-04 | Phase 5 |
| RSCH-05 | Phase 5 |
| RELY-01 | Phase 5 |
| RELY-02 | Phase 5 |
| OUTP-01 | Phase 6 |
| OUTP-02 | Phase 6 |
| OUTP-03 | Phase 6 |
| OUTP-04 | Phase 6 |
| OUTP-05 | Phase 6 |
| OUTP-06 | Phase 6 |
| OUTP-07 | Phase 6 |

**Coverage: 38/38 ✓**

---
*Roadmap created: 2026-04-07*
*Last updated: 2026-04-07 after initial creation*
