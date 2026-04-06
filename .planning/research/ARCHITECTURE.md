# Architecture Patterns: ReAsh AI

**Domain:** Mastra-powered TUI research agent
**Researched:** 2026-04-07
**Confidence:** HIGH (Mastra docs via official mastra.ai, verified patterns)

---

## Recommended Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Entry Point (bin/cli.ts)                   │
│               #!/usr/bin/env node  →  render(<App />)               │
└───────────────────────────┬─────────────────────────────────────────┘
                            │ bootstraps
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       Bootstrap Layer                               │
│  1. loadToken() from ~/.config/reash-ai/token.json (conf)           │
│  2. If absent → GitHubDeviceFlow OAuth → store token                │
│  3. fetchCopilotModels(token) → model list                          │
│  4. new CopilotGateway(token) → mastra.addGateway()                 │
│  5. mastra instance ready                                           │
└───────────────────────────┬─────────────────────────────────────────┘
                            │ passes {mastra, models, token}
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Ink TUI Layer (React/Ink v5)                 │
│  <App alternateScreen incrementalRendering maxFps={60}>             │
│  ├── <ChatPanel>         ← user input, message history              │
│  ├── <ProgressPanel>     ← workflow step status (streaming events)  │
│  ├── <FilesPanel>        ← generated output files list              │
│  ├── <SessionsPanel>     ← thread list / resume / new               │
│  └── <ModelSelector>     ← copilot model picker                     │
└───────────────────────────┬─────────────────────────────────────────┘
                            │ /research <task> command
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Mastra Core Layer                                 │
│                                                                     │
│  mastra = new Mastra({                                              │
│    gateways:  { copilot: new CopilotGateway() }                     │
│    agents:    { researchAgent }                                     │
│    workflows: { researchWorkflow }                                  │
│    storage:   new LibSQLStore({ url: 'file:~/.config/reash-ai/...'})│
│    workspace: new Workspace({                                        │
│                 filesystem: new LocalFilesystem({ basePath: './output' }) │
│                 sandbox:    new LocalSandbox({ workingDirectory: './output' }) │
│                 skills:     ['./skills']                            │
│               })                                                    │
│  })                                                                 │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
          ┌─────────────────┴──────────────────────┐
          ▼                                        ▼
┌──────────────────────┐              ┌────────────────────────────────┐
│    researchAgent     │              │      researchWorkflow          │
│                      │              │                                │
│  Agent({             │  called      │  .then(planStep)               │
│   model: copilot/...,│  from TUI    │  .parallel([                   │
│   memory: Memory({   │  or workflow │    webSearchStep,              │
│     observational:   │  step        │    githubResearchStep          │
│     true,            │              │  ])                            │
│     workingMemory:   │              │  .then(synthesizeStep)         │
│     { enabled:true } │              │  .parallel([                   │
│   }),                │              │    genDocxStep,                │
│   workspace          │              │    genPptxStep,                │
│  })                  │              │    genXlsxStep,                │
│                      │              │    genPdfStep,                 │
│                      │              │    genGsdStep                  │
└──────────────────────┘              │  ])                            │
                                      │  .commit()                     │
                                      └────────────┬───────────────────┘
                                                   │ streaming events
                                                   ▼
                                         Ink TUI ProgressPanel
                                      (for-await loop on fullStream)
```

---

## Component Boundaries

| Component | Responsibility | Communicates With |
|-----------|---------------|-------------------|
| **bin/cli.ts** | Entry point: shebang, parse argv, call bootstrap, render TUI | Bootstrap, Ink TUI |
| **auth/** | GitHub Device Flow OAuth; `loadToken()`, `storeToken()`, `refreshToken()` | Bootstrap, `conf` (fs) |
| **gateway/CopilotGateway** | Extends `MastraModelGateway`; token→auth header, `fetchProviders()` calls Copilot models API, `resolveLanguageModel()` creates `@ai-sdk/openai-compatible-v5` instance | Mastra Core |
| **mastra/index.ts** | Constructs `Mastra` instance: registers gateway, agent, workflow, storage, workspace | All Mastra subsystems |
| **mastra/agents/researchAgent.ts** | `Agent` with Copilot model, Memory (observational + working), Workspace, tools | researchWorkflow steps, Mastra Core |
| **mastra/workflows/researchWorkflow.ts** | Defines parallel fan-in research + fan-out generation workflow | Steps (tools + agents), TUI via stream |
| **mastra/steps/** | Individual `createStep()` units: plan, webSearch, githubResearch, synthesize, genDocx, genPptx, genXlsx, genPdf, genGsd | Tools, pythonBridge, researchAgent |
| **tools/** | `createTool()` wrappers: Tavily search, Cheerio+Playwright scrape, Octokit GitHub, pdf-parse local files | Workflow steps, researchAgent |
| **generators/** | DOCX/PPTX via npm; XLSX/PDF via pythonBridge.ts; GSD via fs | Workflow steps (genDocx/genPptx/genXlsx/genPdf/genGsd steps) |
| **pythonBridge.ts** | Spawns Python subprocess; JSON stdin/stdout protocol; 60s timeout; SIGTERM on cancel | genXlsx step, genPdf step |
| **workspace/** | `new Workspace(LocalFilesystem, LocalSandbox, skills)` pointed at `./output`; auto-loads `./skills/` SKILL.md files | researchAgent (injected automatically as tools) |
| **TUI/** | Ink v5 React components; subscribes to workflow stream events via React state; displays chat/progress/files/sessions/models | Bootstrap state, workflow stream |
| **storage/** | `LibSQLStore` at `~/.config/reash-ai/db.sqlite`; WAL mode; stores threads, messages, working memory | Memory, researchAgent |

---

## Data Flow

```
User types "/research analyze quantum computing trends"
│
▼
TUI ChatPanel → onCommand('/research', task)
│
▼
sessionManager.getOrCreateThread(resourceId, threadId)
│
▼
const run = await researchWorkflow.createRun()
const stream = run.stream({ inputData: { task, threadId, resourceId } })
│
│  ← TUI subscribes: for await (const chunk of stream) { dispatch(chunk) }
│
│  Step 1: planStep
│    execute({ inputData }) → researchAgent.generate(planPrompt)
│    → returns { plan: string[], queries: string[] }
│    → stream emits: workflow-step-start, workflow-step-finish
│
│  Step 2: .parallel([webSearchStep, githubResearchStep])
│    Both execute concurrently:
│    webSearchStep:
│      → Tavily.search(query) + Cheerio.scrape(urls)
│      → returns { webResults: SearchResult[] }
│    githubResearchStep:
│      → Octokit.search(query)
│      → returns { githubResults: RepoResult[] }
│    stream emits: workflow-step-start × 2, workflow-step-finish × 2
│
│  .map() merges parallel outputs:
│    { webResults, githubResults } → synthesizeStep input
│
│  Step 3: synthesizeStep
│    execute({ inputData }) → researchAgent.generate(synthesizePrompt, {
│      memory: { thread: threadId, resource: resourceId }
│    })
│    → returns { synthesis: string, title: string }
│    → stream emits: workflow-step-start, workflow-step-finish
│
│  Step 4: .parallel([genDocxStep, genPptxStep, genXlsxStep, genPdfStep, genGsdStep])
│    genDocxStep → docx npm package → ./output/<title>.docx
│    genPptxStep → pptxgenjs → ./output/<title>.pptx
│    genXlsxStep → pythonBridge.ts → spawn gen_xlsx.py → ./output/<title>.xlsx
│    genPdfStep  → pythonBridge.ts → spawn gen_pdf.py  → ./output/<title>.pdf
│    genGsdStep  → fs.writeFile → ./output/<title>.md
│    All return: { path: string, failed: boolean, error?: string }
│    stream emits: workflow-step-start × 5, workflow-step-finish × 5
│
▼
stream.result → { status: 'success', result: { files: string[] } }
│
▼
TUI FilesPanel updates with generated file paths
TUI ChatPanel shows final summary from synthesizeStep
Memory stores conversation thread (observational + working memory auto-managed)
```

### Token → Gateway → Agent → Model

```
loadToken() → ~/.config/reash-ai/token.json
│
▼
new CopilotGateway({ token })
  - getApiKey() returns token
  - buildUrl() → 'https://api.githubcopilot.com'
  - resolveLanguageModel() → createOpenAICompatible({
      name: 'github-copilot',
      apiKey: token,
      baseURL: 'https://api.githubcopilot.com',
      headers: {
        'Copilot-Integration-Id': 'vscode-chat',
        'Editor-Version': 'vscode/1.90.0'
      }
    }).chatModel(modelId)
│
▼
mastra.addGateway(copilotGateway)
│
▼
Agent({ model: 'copilot/github-copilot/<model-id>' })
  → Mastra model router resolves 'copilot' prefix → CopilotGateway
  → CopilotGateway.resolveLanguageModel() → LanguageModelV2
  → agent.generate() / agent.stream()
```

### Workflow Streaming → TUI

```
// In TUI (React/Ink component):
const [steps, setSteps] = useState<Record<string, StepStatus>>({})
const [files, setFiles] = useState<string[]>([])

async function runResearch(task: string) {
  const run = await researchWorkflow.createRun()
  const stream = run.stream({ inputData: { task, threadId, resourceId } })

  for await (const chunk of stream) {
    switch (chunk.type) {
      case 'workflow-step-start':
        setSteps(prev => ({
          ...prev,
          [chunk.payload.stepName]: 'running'
        }))
        break
      case 'workflow-step-finish':
        setSteps(prev => ({
          ...prev,
          [chunk.payload.stepName]: chunk.payload.status  // 'success' | 'failed'
        }))
        break
      case 'workflow-start':
        // reset progress
        break
    }
  }

  const result = await stream.result
  if (result.status === 'success') {
    setFiles(result.result.files)
  }
}
```

**Key constraint:** Ink's render loop runs in the same Node.js event loop. The `for await` loop should be started in a `useEffect` or via a command handler, NOT blocking the render thread. Ink's `incrementalRendering` means re-renders can happen while the async loop processes chunks — this works correctly because Ink batches state updates.

### Workspace → Agent Integration

```
const workspace = new Workspace({
  filesystem: new LocalFilesystem({ basePath: './output' }),
  sandbox: new LocalSandbox({ workingDirectory: './output' }),
  skills: ['./skills'],  // auto-loads SKILL.md files
})

// When assigned to agent:
const researchAgent = new Agent({
  id: 'research-agent',
  model: 'copilot/github-copilot/<model-id>',
  workspace,           // adds mastra_workspace_* tools automatically
  memory: new Memory({ ... }),
  tools: { tavilySearch, cheerioScrape, octokit, pdfParse },
})

// Agent gets auto-injected tools:
//   mastra_workspace_read_file
//   mastra_workspace_write_file
//   mastra_workspace_list_files
//   mastra_workspace_execute_command  ← can run Python scripts ad-hoc
//   mastra_workspace_grep
//   skill                             ← activate skills from ./skills/
//   skill_search
//   skill_read

// NOTE: pythonBridge.ts (for workflow steps) is SEPARATE from workspace sandbox.
// Workflow steps use pythonBridge directly for structured JSON protocol.
// The workspace sandbox is for the agent's ad-hoc tool use only.
```

---

## Patterns to Follow

### Pattern 1: Workflow Steps Never Throw

Every step returns a typed result union — never throws to the workflow engine:

```typescript
const genDocxStep = createStep({
  id: 'gen-docx',
  inputSchema: z.object({ synthesis: z.string(), title: z.string() }),
  outputSchema: z.object({
    path: z.string().optional(),
    failed: z.boolean(),
    error: z.string().optional(),
  }),
  execute: async ({ inputData }) => {
    try {
      const path = await generateDocx(inputData.synthesis, inputData.title)
      return { path, failed: false }
    } catch (err) {
      return { failed: true, error: String(err) }
    }
  },
})
```

### Pattern 2: `.parallel()` + `.map()` for Fan-Out Generation

Research parallel fan-in:
```typescript
researchWorkflow
  .then(planStep)
  .parallel([webSearchStep, githubResearchStep])
  .map(async ({ inputData, getInitData }) => {
    // inputData keyed by step id
    const init = getInitData<WorkflowInput>()
    return {
      webResults: inputData['web-search'].results,
      githubResults: inputData['github-research'].results,
      task: init.task,
    }
  })
  .then(synthesizeStep)
  .parallel([genDocxStep, genPptxStep, genXlsxStep, genPdfStep, genGsdStep])
  .then(collectFilesStep)
  .commit()
```

### Pattern 3: Memory Thread + Resource Pattern

```typescript
// TUI session manager
const resourceId = `user-${machineId}`         // stable per-machine
const threadId = session.id                     // per-research-session

// Agent call within workflow step
await researchAgent.generate(prompt, {
  memory: {
    resource: resourceId,
    thread: threadId,
  },
})

// Observational Memory compresses old messages automatically.
// Working Memory persists user research preferences across threads.
```

### Pattern 4: Mastra Instance as Singleton

```typescript
// src/mastra/index.ts — single module, imported everywhere
export const mastra = new Mastra({
  gateways:  { copilot: copilotGateway },
  agents:    { researchAgent },
  workflows: { researchWorkflow },
  storage:   libSQLStore,
  workspace,
})

// In workflow steps, use mastra.getAgent() for type inference:
const agent = mastra.getAgent('researchAgent')
```

### Pattern 5: CopilotGateway Token Refresh

The gateway must handle token expiry. The `getApiKey()` method should check expiry before returning. If expired, surface an error up to the TUI so the user can re-authenticate via Device Flow.

```typescript
class CopilotGateway extends MastraModelGateway {
  readonly id = 'copilot'
  readonly name = 'GitHub Copilot'
  private token: StoredToken

  async getApiKey(modelId: string): Promise<string> {
    if (isTokenExpired(this.token)) {
      throw new CopilotTokenExpiredError('Token expired — re-run auth')
    }
    return this.token.access_token
  }
}
```

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Calling `workspace.executeCommand()` from Workflow Steps

**What:** Using the workspace sandbox tool inside a workflow step's `execute()` to run Python.
**Why bad:** Workspace tools are designed for agent tool calls, not direct step execution. The sandbox has no structured JSON stdin/stdout protocol, no deterministic 60s timeout enforcement, and no clean `{ failed, error }` return contract. Steps cannot invoke tool calls directly via `agent.callTool()`.
**Instead:** Use `pythonBridge.ts` — spawn a subprocess with `child_process.spawn`, JSON protocol over stdin/stdout, 60s SIGTERM, structured `{ failed, error }` result.

### Anti-Pattern 2: Blocking the Ink Render Loop

**What:** Awaiting workflow stream inside a synchronous React render or top-level component body.
**Why bad:** Ink renders are synchronous; a blocking loop would freeze the entire TUI (no spinner, no key events).
**Instead:** Start the `for await` loop inside `useEffect(() => { startResearch() }, [])` or in an event handler. Use `setState` to push updates; Ink's `incrementalRendering` will pick them up.

### Anti-Pattern 3: One Big Monolithic Agent Step

**What:** A single workflow step that calls `researchAgent.generate()` with a 5,000-token prompt asking it to plan, research, and generate all at once.
**Why bad:** No granular progress feedback for the TUI; single-step failure loses all work; memory context explodes; unreliable output structure.
**Instead:** Discrete steps (plan → parallel research → synthesize → parallel generate) with typed `inputSchema`/`outputSchema` per step.

### Anti-Pattern 4: Registering Workspace Globally Without Agent-Level Override

**What:** Setting `workspace` only on the Mastra instance and not the agent.
**Why bad:** If multiple agents exist (e.g., future plan-only agent), they all inherit the same workspace including execute_command, which may be undesirable.
**Instead:** Set workspace directly on `researchAgent`. This is also the v1 approach since there's only one agent.

### Anti-Pattern 5: Storing Copilot Token in `.env`

**What:** Using `COPILOT_TOKEN=xxx` in `.env`.
**Why bad:** Defeats the purpose of Device Flow; security risk; tokens rotate.
**Instead:** `conf`-managed `~/.config/reash-ai/token.json` with expiry check at startup.

---

## Build Order (Dependency Graph)

The following represents the implementation dependency order — each layer can only be built once the layer(s) it depends on are complete.

```
Layer 0 (no deps):
  auth/deviceFlow.ts        — pure HTTP, no Mastra deps
  auth/tokenStore.ts        — conf, no Mastra deps
  types/index.ts            — shared Zod schemas and TypeScript types

Layer 1 (depends on Layer 0):
  gateway/CopilotGateway.ts — depends on auth/tokenStore, @mastra/core/llm
  storage/libsql.ts         — LibSQLStore config wrapper

Layer 2 (depends on Layer 1):
  tools/tavily.ts           — createTool(), pure, no agent deps
  tools/cheerio.ts          — createTool(), Cheerio + Playwright
  tools/octokit.ts          — createTool(), Octokit
  tools/pdfParse.ts         — createTool(), pdf-parse
  generators/pythonBridge.ts — child_process spawn, no Mastra deps
  generators/docx.ts        — docx npm, no Mastra deps
  generators/pptx.ts        — pptxgenjs, no Mastra deps

Layer 3 (depends on Layer 2):
  mastra/agents/researchAgent.ts — Agent with tools, Memory, Workspace
  mastra/steps/*.ts              — createStep(), depends on tools + generators

Layer 4 (depends on Layer 3):
  mastra/workflows/researchWorkflow.ts — composes steps with .parallel()/.map()/.then()

Layer 5 (depends on Layer 1, 3, 4):
  mastra/index.ts            — Mastra({gateway, agent, workflow, storage, workspace})

Layer 6 (depends on Layer 5):
  tui/components/*.tsx       — Ink components, import mastra instance
  tui/hooks/useWorkflowStream.ts — for-await stream subscription

Layer 7 (depends on all):
  bin/cli.ts                 — bootstrap + render(<App />)
```

**Build order for phases:**

1. **Auth + Gateway** (Layer 0-1): Device Flow + CopilotGateway → validate token works against models API
2. **Tools + Generators** (Layer 2): Individual createTool() units + pythonBridge → test each tool in isolation with vitest
3. **Agent + Steps** (Layer 3): researchAgent (no workflow yet) → test agent.generate() with memory
4. **Workflow** (Layer 4): compose steps, test with run.start() before adding streaming
5. **TUI** (Layer 6-7): Ink components with mock/stub data first, then wire to real workflow stream
6. **Integration** (all): End-to-end: auth → model select → /research → stream → files

---

## How Ink TUI Subscribes to Mastra Workflow Streaming Events

Mastra's `run.stream()` returns an async iterable (`MastraWorkflowStream`). The TUI subscribes via a `for await` loop in a non-blocking async context:

```typescript
// src/tui/hooks/useWorkflowStream.ts
export function useWorkflowStream(mastra: Mastra) {
  const [stepStatus, setStepStatus] = useState<Record<string, string>>({})
  const [outputFiles, setOutputFiles] = useState<string[]>([])
  const [isRunning, setIsRunning] = useState(false)
  const [error, setError] = useState<string | null>(null)
  const abortRef = useRef<AbortController | null>(null)

  const startResearch = useCallback(async (task: string, threadId: string, resourceId: string) => {
    setIsRunning(true)
    setStepStatus({})
    setOutputFiles([])
    setError(null)

    abortRef.current = new AbortController()
    const workflow = mastra.getWorkflow('researchWorkflow')
    const run = await workflow.createRun()

    try {
      const stream = run.stream({
        inputData: { task, threadId, resourceId },
      })

      for await (const chunk of stream) {
        if (abortRef.current?.signal.aborted) break

        if (chunk.type === 'workflow-step-start') {
          setStepStatus(prev => ({ ...prev, [chunk.payload.stepName]: 'running' }))
        } else if (chunk.type === 'workflow-step-finish') {
          setStepStatus(prev => ({
            ...prev,
            [chunk.payload.stepName]: chunk.payload.status,
          }))
        }
      }

      const result = await stream.result
      if (result.status === 'success') {
        setOutputFiles(result.result.files ?? [])
      } else if (result.status === 'failed') {
        setError(String(result.error))
      }
    } catch (err) {
      setError(String(err))
    } finally {
      setIsRunning(false)
    }
  }, [mastra])

  const cancel = useCallback(() => {
    abortRef.current?.abort()
  }, [])

  return { startResearch, cancel, stepStatus, outputFiles, isRunning, error }
}
```

**Workflow stream event types for the TUI to handle:**

| Event Type | Payload Key | TUI Action |
|------------|-------------|------------|
| `workflow-start` | `runId` | Reset progress display |
| `workflow-step-start` | `stepName`, `startedAt`, `status: 'running'` | Show step as active (spinner) |
| `workflow-step-finish` | `stepName`, `status` | Mark step as done/failed |
| `workflow-step-progress` | `id`, `completedCount`, `totalCount` | Progress bar for foreach (not used in v1) |
| `finish` / `stream.result` | full result | Display file paths, close progress panel |

---

## Mastra Workspace in the Overall Architecture

The `Workspace` serves two distinct roles:

### Role 1: Agent File/Shell Access (Primary)
The workspace gives `researchAgent` tools to read local files, write output, list directories, and execute commands. This is used when the agent is called with a task that involves local context (e.g., "include this PDF in the research").

```
User: "analyze this file: report.pdf"
  → researchAgent tool call: mastra_workspace_read_file("/path/to/report.pdf")
  → returns file content to agent context
  → agent incorporates into synthesis
```

### Role 2: Skills Auto-Loading (Secondary)
SKILL.md files from `anthropics/skills` (placed in `./skills/`) are auto-discovered by the workspace and exposed as `skill`, `skill_search`, `skill_read` tools to the agent. This gives the agent access to structured instructions for document generation patterns.

```
./skills/
  report-writing/SKILL.md     → activated via: skill("report-writing")
  data-analysis/SKILL.md      → activated via: skill("data-analysis")
```

### Role 3: NOT Used for Python Execution in Workflow Steps
Despite the workspace having `mastra_workspace_execute_command`, the workflow generator steps (genXlsxStep, genPdfStep) use `pythonBridge.ts` directly. This is because:
- Workflow step `execute()` cannot make agent tool calls
- `pythonBridge.ts` provides a controlled stdin/stdout JSON contract
- 60s timeout + SIGTERM is enforced explicitly in `pythonBridge.ts`
- Error result is typed as `{ failed: boolean, error?: string }`

The workspace sandbox remains available to `researchAgent` for ad-hoc Python scripting at the agent level if the user explicitly requests it through chat.

---

## Scalability Considerations

| Concern | v1 (single-user local) | If multi-user later |
|---------|----------------------|---------------------|
| Storage | LibSQL file, WAL mode | Migrate to PostgreSQL (`@mastra/pg`) |
| Concurrency | Single workflow run per session | Add run queue or separate Mastra instances |
| Model auth | Single Copilot token per install | OAuth per user, token per session |
| Workspace | Single `./output/` dir | Per-user directories or E2B sandbox |
| Memory | Single LibSQL, resource-scoped | Shared PgStore, resourceId = userId |

---

## Sources

- Mastra Docs — Workflows Overview: https://mastra.ai/docs/workflows/overview
- Mastra Docs — Control Flow (parallel, map): https://mastra.ai/docs/workflows/control-flow
- Mastra Docs — Streaming Events: https://mastra.ai/docs/streaming/events
- Mastra Docs — Streaming Overview: https://mastra.ai/docs/streaming/overview
- Mastra Docs — Workspace Overview: https://mastra.ai/docs/workspace/overview
- Mastra Reference — Workspace Class: https://mastra.ai/reference/workspace/workspace-class
- Mastra Reference — Agent Class: https://mastra.ai/reference/agents/agent
- Mastra Reference — MastraModelGateway: https://mastra.ai/reference/core/mastra-model-gateway
- Mastra Reference — Custom Gateways: https://mastra.ai/models/gateways/custom-gateways
- Mastra Reference — Memory Class: https://mastra.ai/reference/memory/memory-class
- Mastra Reference — ChunkType: https://mastra.ai/reference/streaming/ChunkType
- Mastra Reference — Run Class: https://mastra.ai/reference/workflows/run
- Mastra Docs — Working Memory: https://mastra.ai/docs/memory/working-memory
- Mastra Docs — Memory Overview: https://mastra.ai/docs/memory/overview
- Mastra Guide — AI SDK UI (streaming to custom UIs): https://mastra.ai/guides/build-your-ui/ai-sdk-ui

**Confidence:** HIGH — all patterns verified against current official Mastra documentation.
