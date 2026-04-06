# Feature Landscape

**Domain:** TUI-based AI research agent CLI (GitHub Copilot-exclusive)
**Researched:** 2026-04-07
**Reference class:** Claude Code, Gemini CLI, Aider, OpenAI Agents SDK patterns

---

## Table Stakes

Features users expect from any serious AI CLI tool in 2025. Missing = product feels incomplete or amateurish compared to Claude Code / Gemini CLI.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| OAuth Device Flow auth with browser prompt | GitHub Copilot standard; zero PAT storage; users have seen this pattern in `gh auth login` | Low | Show `user_code` prominently; open browser with `open`/`start`; poll with backoff; expire gracefully |
| Token persistence across sessions | Re-auth on every run is unacceptable for a daily-driver tool | Low | `~/.config/reash-ai/token.json` via `conf`; refresh on 401 |
| Dynamic model listing and switching | Copilot exposes multiple models; hiding this is a regression vs VSCode | Medium | `GET https://api.githubcopilot.com/models`; `/model` command; persist selection |
| Chat input with multi-line support | Single-line input breaks pasting research prompts | Low | `@inkjs/ui` `TextInput`; Shift+Enter for newline |
| Slash commands (`/help`, `/exit`, `/clear`) | Every AI CLI has these; absence signals immaturity | Low | At minimum: `/help`, `/exit`, `/clear`, `/research`, `/model` |
| Streaming response rendering | Non-streaming feels broken and slow for long outputs | Medium | Ink `<Text>` updated per token via `onChunk`; preserve scroll position |
| Session persistence and resume | Users cannot redo an hour-long research thread from scratch | Medium | LibSQL threads; `/sessions` list with timestamps; `/resume <id>` |
| Keyboard interrupt safety (Ctrl+C) | Abrupt exit mid-research with no recovery = data loss | Low | Catch SIGINT; gracefully cancel in-flight workflow step; show partial result |
| Output file generation | Core value prop — research → document; without it the tool is just a chatbot | High | DOCX, PPTX, GSD Markdown at minimum (PDF/XLSX can degrade gracefully) |
| Progress visibility during research | Long-running workflows with no feedback feel broken | Medium | `tasuku` task tree or Ink `<Spinner>` per workflow step with step name |
| Error messages that are actionable | "Error: undefined" destroys trust | Low | Every error surfaces: what failed, why, what to do next |
| `--version` and `--help` flags | Standard CLI contract; missing = unprofessional | Low | Wire via `meow` or `yargs`; include in startup |
| Graceful Python-absent degradation | Python is optional; silently failing XLSX/PDF is unacceptable | Low | Detect at startup; warn once; mark PDF/XLSX as unavailable in UI |
| Auto-scroll chat to latest message | Basic UX; stale scroll = confused users | Low | Track scroll state; auto-scroll on new message unless user scrolled up |
| Working directory / output path config | Users need to control where files land | Low | `WORKSPACE_PATH` env var; default `./output/`; show in UI |

---

## Differentiators

Features that make ReAsh AI stand out against the reference class. Not table stakes, but they shift the "this is special" perception.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Parallel research fan-out visibility | Gemini/Claude don't show concurrent sub-agent work in a structured task tree — ReAsh can | Medium | `tasuku` parallel task group: Web Search ‖ GitHub Research ‖ Local Files; live status per branch |
| GitHub repository deep-research via Octokit | No competitor targets developer + GitHub as first-class research source | High | Search issues, PRs, README, code, commits via Octokit; surface repo insights in synthesis |
| Five output formats from a single `/research` command | Claude Code generates files; Gemini doesn't; none generate PPTX/XLSX | High | DOCX + PPTX + XLSX + GSD Markdown + PDF fan-out in one workflow; format picker in UI |
| Copilot-native (no extra API keys for core function) | Zero friction for the 50M+ Copilot subscribers; no OpenAI key, no Anthropic key | Low (design) | Headline differentiator; must be front and center in README and first-run UX |
| `@file` / `@dir` context injection into research | Aider does file context; no research-focused CLI does it cleanly | Medium | Parse `@path` tokens in input; attach file contents to research plan step |
| Research memory across sessions (observational) | Most CLI tools clear context on exit; ReAsh retains research style/preferences per thread | Medium | Mastra observational memory on `researchAgent`; surfaces "you previously researched X" |
| `/research` as a first-class workflow trigger | AI CLIs treat everything as chat; ReAsh has a named command that activates a structured multi-step workflow | Low (design) | Naming matters: `/research <topic>` is intentional, not implicit |
| GSD Markdown output format | Structured, milestone-ready research doc output format aligned with developer workflow tooling | Low | Native `fs` write; no dependencies; pairs with GSD project planning tooling |
| Session browser with search | Gemini has `/resume`; Claude has implicit memory; ReAsh can do both with searchable session list | Medium | Fuzzy search over session titles/topics; keyboard-navigable list panel |
| `!shell` passthrough for power users | Aider has it; developers expect it; avoids context switching | Low | `!<cmd>` prefix executes in child_process.exec; streams stdout back into chat |

---

## Anti-Features

Things to deliberately NOT build in v1. Each has a clear reason and a "what to do instead."

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| PAT / API key auth for Copilot | Defeats the "zero extra setup" value prop; stores sensitive credentials | Device Flow only; token in `~/.config/reash-ai/token.json` |
| Web UI / Electron app | Scope creep; loses the CLI-native identity; bloated distribution | TUI-only; Ink v5; distribute as npm binary |
| Cloud session sync | Requires backend infrastructure; violates local-first promise; adds GDPR surface | LibSQL local file; user owns their data |
| Multi-user / team features | Research CLI is a single-developer tool; team features add auth/RBAC complexity out of proportion | Single-user; no sharing; no workspaces |
| Plugin / extension marketplace | Maintenance burden disproportionate to v1 benefit; Claude Code's MCP integration took years to mature | Hard-code research workflow; open source encourages forks |
| Voice input | Not a research-oriented interaction model; adds significant dep weight | Text-only; paste-friendly multi-line input |
| Real-time collaboration / shared sessions | Not the use case; live cursors + conflict resolution are a separate product | Local, single-user sessions |
| Auto-git-commit of outputs | Aider's killer feature is code-centric; research output to git is presumptuous | Write to `./output/`; user decides what to commit |
| Built-in LLM model fine-tuning or training | Wildly out of scope; Copilot doesn't expose fine-tuning API | Use Copilot models as-is; no custom training |
| GUI file picker | Breaks terminal-only contract; adds cross-platform complexity | `@file` text injection; `$EDITOR` for multi-file context |
| Streaming to remote endpoint / webhook | Adds network surface; not needed for local research tool | Write files locally; user exports manually |
| Custom theming system | Gemini has `/theme`; it's nice but low-value for a research tool | Single clean dark theme; respect terminal colors |

---

## Feature Dependencies

```
Device Flow Auth
  └─→ Token Persistence
        └─→ Dynamic Model Listing
              └─→ Chat Input + Streaming
                    └─→ Session Persistence
                          └─→ Session Browser (search/resume)

/research command
  └─→ Parallel Research Workflow
        ├─→ Web Search (Tavily + Cheerio/Playwright)
        ├─→ GitHub Research (Octokit)
        └─→ Local File Reading (@file injection)
              └─→ Synthesis Step
                    └─→ Output Fan-out
                          ├─→ DOCX (docx npm)
                          ├─→ PPTX (pptxgenjs)
                          ├─→ GSD Markdown (fs)
                          ├─→ XLSX (Python / openpyxl) [optional]
                          └─→ PDF  (Python / reportlab)  [optional]

Progress Visibility
  └─→ tasuku task tree (depends on: Parallel Research Workflow)

Observational Memory
  └─→ Session Persistence (depends on: LibSQL store)

!shell passthrough
  └─→ Chat Input (slash/bang command parser)
```

---

## MVP Recommendation

**Prioritize (must ship in v1):**

1. Device Flow auth + token persistence — without auth, nothing works
2. Dynamic model listing + selection — Copilot exposes multiple models; hiding them is a downgrade
3. Chat input with streaming — core interaction loop
4. `/research <topic>` → parallel workflow → synthesis — the whole reason the tool exists
5. GSD Markdown + DOCX output — minimum viable document generation (no Python required)
6. Session persistence + basic resume — users need to continue work across runs
7. Progress visibility (tasuku task tree) — long workflows need feedback or feel broken
8. Ctrl+C safety + graceful error messages — trust-building fundamentals

**Defer to v1.x:**

| Feature | Reason to Defer |
|---------|----------------|
| XLSX / PDF output | Requires Python; optional dep; can ship as v1.1 after validating Python bridge |
| Session browser with fuzzy search | Nice-to-have; basic `/sessions` list is sufficient for v1 |
| `@file` / `@dir` context injection | Adds parser complexity; validate core research loop first |
| `!shell` passthrough | Power-user feature; validate core audience first |
| Observational memory (cross-session) | Working memory (thread-scoped) sufficient for v1; cross-session memory is v1.1 |
| GitHub repository deep-research | High complexity; validate web research workflow first |

---

## Sources

- Claude Code docs: https://docs.anthropic.com/en/docs/claude-code/overview (HIGH confidence — official)
- Gemini CLI commands: https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/commands.md (HIGH confidence — official repo)
- Aider slash commands: https://aider.chat/docs/usage/commands.html (HIGH confidence — official)
- Ink v5 README: https://github.com/vadimdemedes/ink (HIGH confidence — official)
- GitHub Device Flow: https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps#device-flow (HIGH confidence — official)
- tasuku: https://github.com/privatenumber/tasuku (MEDIUM confidence — official repo)
- OpenAI Agents SDK: https://openai.github.io/openai-agents-python/ (MEDIUM confidence — official)
