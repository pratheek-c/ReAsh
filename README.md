# 🔥 ReAsh AI

> **Your terminal. Your Copilot. Your research superpower.**

ReAsh AI is a fully open-source, TUI-based AI research agent that runs entirely in your terminal — powered by your existing **GitHub Copilot** subscription. No extra API keys. No browser tabs. Just type `/research` and get a full structured research document delivered to your machine. ⚡

---

## ✨ What It Does

```
$ reash
```

That's it. One command and you're in. 🎉

- 🔐 **Authenticates** with GitHub Copilot via OAuth Device Flow — no PAT, no `.env` secrets
- 🤖 **Fetches available Copilot models** dynamically — pick the one you want, switch anytime
- 💬 **Chat interface** with streaming responses, multi-line input, and slash commands
- 🔍 **`/research <topic>`** — triggers a parallel Mastra workflow that searches the web, GitHub, and your local files simultaneously
- 📁 **Generates output documents** in 5 formats: DOCX, PPTX, XLSX, GSD Markdown, and PDF
- 🧠 **Remembers your sessions** — resume any past research thread where you left off
- 📺 **Live task tree** — watch every parallel research branch in real time as it runs

---

## 🚀 Quick Start

```bash
# Install globally
npm install -g reash-ai

# or with Bun (recommended — way faster 🐰)
bun add -g reash-ai

# Launch!
reash
```

First run walks you through a one-time GitHub Copilot auth. After that, you're in for every future run. 🔓

---

## 🎮 Commands

| Command | What it does |
|---|---|
| `/research <topic>` | 🔍 Kick off a full parallel research workflow |
| `/model` | 🤖 Switch between available Copilot models |
| `/sessions` | 📋 Browse your past research sessions |
| `/resume <id>` | ⏪ Pick up any session where you left off |
| `/clear` | 🧹 Clear the current chat |
| `/help` | ❓ Show all commands |
| `/exit` | 👋 Quit |
| `!<command>` | 🐚 Run a shell command without leaving the TUI |
| `@<path>` | 📂 Inject a local file or directory as research context |

---

## 📦 Output Formats

After every `/research` run, your results land in `./output/` in up to **5 formats**:

| Format | Requires |
|---|---|
| 📝 GSD Markdown | Nothing — always works |
| 📄 DOCX | Nothing — always works |
| 📊 PPTX | Nothing — always works |
| 🗃️ XLSX | Python + `openpyxl` |
| 📑 PDF | Python + `reportlab` |

> 💡 No Python? No problem. ReAsh gracefully skips XLSX/PDF and generates everything else without complaint.

---

## 🧱 Built With

| Thing | Why |
|---|---|
| 🟦 **TypeScript** (strict mode) | Type safety everywhere — no surprises at runtime |
| ⚡ **Bun** | 10–20× faster installs than npm |
| 🎨 **Ink v6** | React-in-the-terminal TUI framework |
| 🤖 **Mastra** | Agent orchestration, memory, and workflow engine |
| 🗄️ **LibSQL (SQLite)** | Zero-config local session storage |
| 🔎 **Tavily** | Web search for research workflows |
| 🐙 **Octokit** | GitHub repository deep-research |
| 📄 **docx + pptxgenjs** | Pure JS document generation |
| 🐍 **Python bridge** | XLSX + PDF via openpyxl, pandas, reportlab |
| 🧹 **Biome** | Linting + formatting in one fast Rust binary |

---

## 🏗️ Roadmap

- [x] 📐 Architecture & planning
- [x] 📋 Requirements (38 defined)
- [ ] 🔧 **Phase 1** — Project scaffolding & global `reash` binary
- [ ] 🔐 **Phase 2** — GitHub Copilot Device Flow authentication
- [ ] 💬 **Phase 3** — Ink TUI, streaming chat, model management
- [ ] 🧠 **Phase 4** — Sessions, memory, and session browser
- [ ] 🔍 **Phase 5** — `/research` Mastra parallel workflow
- [ ] 📁 **Phase 6** — Multi-format output generation

---

## 🤝 Contributing

This project is MIT-licensed and fully open source. Contributions, issues, and ideas are welcome! 🙌

---

## 📜 License

MIT © ReAsh AI Contributors

---

<div align="center">
  <strong>Built for developers. Runs in your terminal. Powered by Copilot. 🚀</strong>
</div>
