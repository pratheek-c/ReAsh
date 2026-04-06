# Domain Pitfalls

**Domain:** Bun-primary ESM CLI with Ink v5 TUI, Mastra agent orchestration, GitHub Copilot OAuth, Python subprocesses
**Researched:** 2026-04-07
**Project:** ReAsh AI

---

## Critical Pitfalls

Mistakes that cause rewrites or major issues.

---

### Pitfall 1: Mastra `.parallel()` — `inputSchema` key must match step `id` exactly

**What goes wrong:** After `.parallel([step1, step2])`, the downstream step's `inputSchema` must define keys using each parallel step's `id` string verbatim (e.g., `'web-researcher': z.object({...})`). If the `id` used at `createStep({ id: 'web-researcher' })` and the key in the next step's `inputSchema` don't match, validation fails at runtime — silently or with a cryptic schema mismatch error rather than a clear "key not found" message.

**Why it happens:** Mastra uses step `id` as the key in the merged output object from `.parallel()`. The workflow system does not coerce or alias these names. Any drift between `id` and schema key breaks the contract.

**Consequences:** Wrong data shape reaching synthesize step; agent receives empty/undefined input; output generation fails with no obvious cause.

**Prevention:**
- Define step `id` as a named constant: `const WEB_STEP_ID = 'web-researcher'` and reference it in both `createStep({ id: WEB_STEP_ID })` and the downstream `inputSchema` key.
- Unit-test the step schema contract: mock `inputData` with the parallel structure and verify the synthesize step executes without schema errors.
- When schemas don't align (e.g., parallel steps return different shapes than downstream expects), use `.map()` between `.parallel()` and the next `.then()` to reshape data explicitly.

**Detection:** Zod validation error in workflow step execute() referencing unexpected field names; `inputData['web-researcher']` being `undefined` at runtime.

**Applies to phase:** Research Workflow (parallel web + github fan-out → synthesize → file generation fan-out)

---

### Pitfall 2: `observationalMemory: true` defaults to `google/gemini-2.5-flash` — not CopilotGateway

**What goes wrong:** Setting `observationalMemory: true` in `Memory` options uses `google/gemini-2.5-flash` for the Observer and Reflector background agents. This model is NOT routed through the `CopilotGateway` and requires a separate Google API key. The project's constraint is Copilot-only auth with no external API keys beyond `TAVILY_API_KEY`.

**Why it happens:** Observational Memory defaults are baked into `@mastra/memory` and resolve through Mastra's model router, not through the custom `CopilotGateway`. The shorthand `observationalMemory: true` bypasses any custom gateway.

**Consequences:** Startup failure if `GOOGLE_GENERATIVE_AI_API_KEY` is not set; or silent fallback to error if the model isn't reachable.

**Prevention:**
- Never use `observationalMemory: true` shorthand. Always pass a config object with an explicit model that routes through CopilotGateway:
  ```typescript
  observationalMemory: {
    model: 'copilot/github-copilot/gpt-4.1', // routed via CopilotGateway
  }
  ```
- Test OM startup against CopilotGateway in CI with a mock Copilot token endpoint.

**Detection:** `Error: GOOGLE_GENERATIVE_AI_API_KEY is not set` at startup; Observer/Reflector API calls going to `generativelanguage.googleapis.com` instead of `api.githubcopilot.com`.

**Applies to phase:** Memory & Sessions (Phase 2)

---

### Pitfall 3: Ink v5 `alternateScreen: true` + crash = terminal left in raw mode

**What goes wrong:** When the process crashes (uncaught exception, unhandled promise rejection, `SIGTERM`) while Ink is rendering with `alternateScreen: true`, the terminal is left in raw mode and/or alternate screen mode. The user's terminal becomes unusable (no echo, garbage characters) until they run `reset` or close the terminal.

**Why it happens:** Ink installs cleanup on `process.exit()` but not on uncaught throws or `SIGKILL`. The alternate screen escape sequence (`\x1b[?1049h`) is never reversed.

**Consequences:** Developer/user experience catastrophe; reproducible on every crash during development.

**Prevention:**
```typescript
// In main entry point, before render()
process.on('uncaughtException', (err) => {
  // Restore terminal before exiting
  process.stdout.write('\x1b[?1049l'); // exit alternate screen
  process.stdout.write('\x1b[?25h');   // show cursor
  process.stdin.setRawMode?.(false);
  console.error(err);
  process.exit(1);
});
process.on('unhandledRejection', (reason) => {
  process.stdout.write('\x1b[?1049l');
  process.stdout.write('\x1b[?25h');
  process.stdin.setRawMode?.(false);
  console.error(reason);
  process.exit(1);
});
```
- Wrap Ink's `render()` in a try-catch for synchronous errors.
- Use the `useApp()` hook's `exit()` function for graceful shutdown rather than `process.exit()` directly.

**Detection:** After a crash, terminal shows no echo; running any command produces garbage; `stty sane` or `reset` required.

**Applies to phase:** TUI Foundation (Phase 1)

---

### Pitfall 4: Bun `setRawMode` on Windows overwrites console mode flags

**What goes wrong:** On Windows, Bun's implementation of `process.stdin.setRawMode(true)` overwrites the console mode flags entirely instead of OR-ing them. This can strip `ENABLE_VIRTUAL_TERMINAL_INPUT` and other flags, breaking ANSI escape processing and causing Ink rendering corruption.

**Why it happens:** Bun issue #25663 (fixed in Dec 2025 but may regress in point releases). The Windows Console API requires carefully masking existing flags; Bun's initial implementation did not do this.

**Consequences:** Ink TUI renders with garbled ANSI sequences on Windows; mouse events may stop working; arrow key navigation broken.

**Prevention:**
- Pin Bun to a version ≥ 1.2.0 (post-fix).
- Add a Windows-specific smoke test in CI that verifies raw mode doesn't corrupt virtual terminal input.
- In `pythonBridge.ts` and any stdin setup code, do not call `setRawMode` from multiple places; centralize it in the Ink initialization path.

**Detection:** Ink components render as raw escape sequences on developer's Windows machine; `[A`, `[B` visible instead of cursor movement.

**Applies to phase:** TUI Foundation (Phase 1), Windows CI

---

### Pitfall 5: `import.meta.url` resolves to `cwd` not binary location in `bun build --compile`

**What goes wrong:** When bundled as a compiled Bun binary (`bun build --compile`), `import.meta.url` and `Bun.resolve(path, import.meta.url)` resolve relative to the current working directory, NOT the binary's location. This breaks any path resolution for bundled assets, skills, or Python scripts.

**Why it happens:** Bun issue #13405. The embedded virtual filesystem root is not exposed via `import.meta.url` in compiled binaries.

**Consequences:** `pythonBridge.ts` cannot find `gen_xlsx.py` / `gen_pdf.py` by relative path when the binary is run from a different directory; workspace skills/ auto-loading breaks.

**Prevention:**
- This project distributes as an `npx`-compatible npm package (dist/ ESM), NOT as a compiled Bun binary. `bun build --compile` is explicitly out of scope for the distribution target.
- For dev (`bun run dev`), `import.meta.url` works correctly — only compiled binaries are affected.
- Python scripts should be resolved relative to `__dirname` (CommonJS compat shim) or use `new URL('./gen_xlsx.py', import.meta.url).pathname` in the ESM dist. Verify this works in both `node` and `bun` ESM execution.

**Detection:** `ENOENT: gen_xlsx.py not found` when running the binary from a different directory; only reproducible with `bun build --compile` output.

**Applies to phase:** Python Bridge & File Generation (Phase 3)

---

### Pitfall 6: Resource-scoped Working Memory requires `mastra_resources` table — not all adapters support it

**What goes wrong:** `workingMemory: { enabled: true, scope: 'resource' }` requires a storage adapter that supports the `mastra_resources` table. If the LibSQLStore is not initialized correctly or the schema migration hasn't run, working memory silently fails to persist or throws a table-not-found error at runtime.

**Why it happens:** Mastra's LibSQLStore auto-creates tables on first use, but only if the `url` is a writable file path (not `:memory:`). If the storage URL is mis-configured (wrong path, permissions issue on Windows), the tables aren't created.

**Consequences:** Working memory data lost between sessions; agent forgets user context; no error surfaced to the user — agent just behaves as if memory is blank.

**Prevention:**
- On startup, verify the LibSQL file path is writable before initializing Mastra. Surface a clear startup warning if the path is read-only.
- Use a test that checks `mastra_resources` table exists after `LibSQLStore` initialization.
- Store the DB at `~/.config/reash-ai/memory.db` (same directory as `token.json`) to centralize all persisted state and reduce path-not-found issues.

**Detection:** Working memory blank on every session restart; `no such table: mastra_resources` in logs.

**Applies to phase:** Memory & Sessions (Phase 2)

---

## Moderate Pitfalls

---

### Pitfall 7: GitHub Device Flow — `device_flow_disabled` if not enabled in OAuth App settings

**What goes wrong:** The Device Flow endpoint (`POST https://github.com/login/device/code`) returns `{"error":"device_flow_disabled"}` if Device Flow hasn't been explicitly enabled in the OAuth App's settings page on GitHub. This is a one-time setup step that is easy to forget and is not enforced during app registration.

**Prevention:**
- Document in CONTRIBUTING.md that Device Flow must be enabled manually in GitHub OAuth App settings.
- The `GITHUB_CLIENT_ID` for the published app must have Device Flow pre-enabled before publishing to npm.
- Handle `device_flow_disabled` error code explicitly in `auth/deviceFlow.ts` with a user-facing message: "Device flow is not enabled for this app. Contact the maintainer."

---

### Pitfall 8: GitHub Device Flow — `slow_down` error accumulates interval additively

**What goes wrong:** Polling too fast (faster than the `interval` returned in Step 1, default 5s) returns a `slow_down` error. Each `slow_down` response adds 5 seconds to the interval — additively, not as a reset. If the polling loop doesn't track and respect the accumulated interval, it will keep triggering `slow_down` indefinitely and never receive the token.

**Prevention:**
```typescript
let interval = initialInterval; // from device/code response
// In polling loop:
if (error === 'slow_down') {
  interval += 5;
  await sleep(interval * 1000);
  continue;
}
```
- Always read the `interval` field from the device/code response (do not hardcode 5s).
- Parse ALL error codes in the polling loop: `authorization_pending` (keep polling), `slow_down` (increase interval), `expired_token` (restart flow), `access_denied` (user cancelled — stop).

---

### Pitfall 9: GitHub Device Flow — 10-token cap per user/app/scope

**What goes wrong:** GitHub allows at most 10 valid tokens per user/application/scope combination. The 11th authorization revokes the oldest token. Users who frequently re-authenticate (e.g., during dev testing with `--reset-auth`) may find older tokens revoked, breaking any CLI session that cached an older token.

**Prevention:**
- Implement token refresh/reuse: check if the stored token in `~/.config/reash-ai/token.json` is still valid before initiating new Device Flow.
- Add a `GET https://api.github.com/user` call as a token validation probe on startup.
- Document the cap in `CONTRIBUTING.md` so developers testing auth repeatedly are aware.

---

### Pitfall 10: `tsup` does not strip shebangs — must configure `banner` for reliable shebang injection

**What goes wrong:** `tsup` with ESM output may or may not preserve the `#!/usr/bin/env node` shebang from the source file depending on version and configuration. Without a shebang in `dist/index.js`, `npm install -g reash-ai` produces a binary that is not directly executable on Unix systems (requires `node dist/index.js` explicitly).

**Prevention:**
```typescript
// tsup.config.ts
export default defineConfig({
  entry: ['src/index.ts'],
  format: ['esm'],
  banner: {
    js: '#!/usr/bin/env node',
  },
  // ...
});
```
- Verify the shebang survives the build: `head -1 dist/index.js` must output `#!/usr/bin/env node`.
- Set `"bin": { "reash-ai": "dist/index.js" }` in `package.json`.
- Add a CI step that checks the first line of the output binary.

---

### Pitfall 11: LibSQL WAL mode on Windows — file locking differences

**What goes wrong:** SQLite WAL mode requires shared memory files (`.db-shm`, `.db-wal`). On Windows, these files use advisory locking which differs from POSIX behavior. If multiple processes (e.g., two terminal windows running `reash-ai`) access the same LibSQL file simultaneously, one may receive `SQLITE_BUSY` errors.

**Prevention:**
- This is a single-user local tool — concurrent access is unlikely but not impossible.
- Add `busy_timeout = 5000` to LibSQL initialization to wait up to 5s before returning `SQLITE_BUSY`.
- Document that running multiple instances simultaneously against the same DB file is unsupported.
- On Windows dev machine: ensure `.db-shm` / `.db-wal` files are excluded from git via `.gitignore`.

---

### Pitfall 12: Python subprocess on Windows — `SIGTERM` does not kill child processes

**What goes wrong:** `process.kill(child.pid, 'SIGTERM')` on Windows does not send SIGTERM to Python processes. Windows does not support Unix signals. Node.js/Bun translates `SIGTERM` to `TerminateProcess()` for the direct child, but any Python child-of-child processes (e.g., if reportlab spawns subprocesses) are orphaned.

**Prevention:**
- In `pythonBridge.ts`, use `child.kill()` (which calls `TerminateProcess` on Windows) rather than `process.kill(pid, 'SIGTERM')`.
- The 60s timeout in `pythonBridge.ts` should call `child.kill()` on timeout expiry.
- Keep Python scripts (`gen_xlsx.py`, `gen_pdf.py`) single-process — no subprocess spawning from within them.
- On Windows, `child_process.spawn` with `shell: false` avoids spawning a `cmd.exe` wrapper that would be harder to kill.

---

### Pitfall 13: Python not on PATH on Windows — non-fatal but must be detected early

**What goes wrong:** On Windows, Python is often installed as `python` but the PATH entry may point to the Microsoft Store stub (`python.exe` that opens the Store instead of running Python). `which python` or `python --version` may succeed but produce unexpected output.

**Prevention:**
- At startup, probe for Python by running `python --version` or `python3 --version` and check output contains `Python 3.x`.
- If Python is not found or the version probe fails, display a non-fatal startup warning: "PDF/XLSX generation unavailable — Python not found. DOCX/PPTX/GSD still work."
- Try `python3` before `python` for cross-platform compatibility; fall back gracefully.
- Document Python install requirements in README with Windows-specific instructions (prefer `python.org` installer over Microsoft Store version).

---

### Pitfall 14: Ink `useFocus` requires explicit opt-in — easy to forget in new components

**What goes wrong:** In Ink v5, keyboard focus is not automatic. Components must call `useFocus()` to receive focus and `useFocusManager()` to programmatically shift focus. Forgetting to add `useFocus` to a new panel means keyboard events never reach it, even if it appears visually active.

**Prevention:**
- Establish a component convention: every interactive panel (chat input, sessions list, model selector) must call `useFocus({ id: 'panel-name' })`.
- Add an integration test using `ink-testing-library` that verifies Tab key cycles focus through all panels.
- Document the focus architecture in `src/tui/README.md`.

---

### Pitfall 15: `threadId` owner (`resourceId`) cannot be changed after creation

**What goes wrong:** Mastra memory threads have an immutable `resourceId` (owner). If a thread is created with one `resourceId` and later queried with a different one, Mastra throws an error. This is easy to hit if the `resourceId` format changes between versions (e.g., changing from email to GitHub username as the resource identifier).

**Why it happens:** Thread ownership is checked on every memory query to prevent cross-user data leakage. The check is strict and non-negotiable.

**Prevention:**
- Decide the `resourceId` format in Phase 1 and commit to it: recommend using the GitHub `user.id` (numeric, stable) from the `/user` API response rather than `user.login` (can change).
- Never change the `resourceId` format once sessions are in production.
- Store the `resourceId` alongside the token in `~/.config/reash-ai/token.json` so it's consistent across invocations.

---

## Minor Pitfalls

---

### Pitfall 16: `noUncheckedIndexedAccess` makes array/object access require null checks

**What goes wrong:** With `noUncheckedIndexedAccess: true` (enabled in this project), every `array[0]` and `obj[key]` returns `T | undefined`. Code that does `const first = results[0]; first.value` will fail to typecheck. Common in workflow step execute functions receiving `inputData` arrays.

**Prevention:**
- Use optional chaining: `results[0]?.value`.
- Use `at(0)` for arrays (same type behavior but more idiomatic).
- Add the `// @ts-expect-error` comment only as a last resort; prefer proper null checks.

---

### Pitfall 17: `npm publish` — `files` field omission leaks source files

**What goes wrong:** Without an explicit `files` field in `package.json`, `npm publish` includes everything not in `.npmignore`, which typically means `src/`, `.planning/`, `tests/`, Python scripts, etc. are published, bloating the package.

**Prevention:**
```json
{
  "files": ["dist/", "skills/", "python/", "README.md", "LICENSE"]
}
```
- Add `npm pack --dry-run` as a CI check to verify published file list.
- Exclude `.planning/`, `src/`, `tests/`, `.env*`, `*.db`, `*.db-shm`, `*.db-wal`.

---

### Pitfall 18: `ink-testing-library` does not emulate alternate screen

**What goes wrong:** `ink-testing-library` renders to a string buffer, not a real terminal. `alternateScreen: true` has no effect in tests, and keyboard input via `stdin.write()` does not go through real raw mode. Tests that depend on terminal-specific behavior (alternate screen cleanup, raw mode state) cannot be validated with `ink-testing-library`.

**Prevention:**
- Use `ink-testing-library` only for component logic tests (rendering, state transitions, key handling via `stdin.write()`).
- Smoke-test terminal cleanup behavior manually or in a separate E2E script that spawns the CLI as a child process and checks exit behavior.

---

### Pitfall 19: Mastra workflow `{ failed: boolean }` pattern — branches must handle both cases

**What goes wrong:** The project's error strategy is for steps to never throw but return `{ failed: true, error?: string }`. However, if the synthesize step's `inputSchema` marks `failed` as optional and the downstream logic doesn't branch on it, partial research failures (e.g., one of the parallel researchers failing) will produce a synthesis with missing data silently.

**Prevention:**
- The synthesize step should always check each parallel result's `failed` field and either:
  - Synthesize with a note about which sources failed, OR
  - Fail loudly if too many parallel steps failed (e.g., both web and github researchers)
- Never silently ignore `failed: true` from an upstream step.

---

### Pitfall 20: `conf` package on Windows stores config in `AppData\Roaming` not `~/.config`

**What goes wrong:** The `conf` npm package uses OS-appropriate config directories: `~/.config/reash-ai/` on Linux/macOS, `C:\Users\<name>\AppData\Roaming\reash-ai\` on Windows. The project spec says `~/.config/reash-ai/token.json`. On Windows, these are different locations, which will confuse users who look for the token file manually.

**Prevention:**
- Use `conf`'s `cwd` option to explicitly set the storage directory to `os.homedir() + '/.config/reash-ai'` on all platforms for consistency.
- Document the actual path in the README for each OS.
- Or, accept the OS-default path and update the spec to say "`conf`-managed config directory" rather than hard-coding `~/.config/reash-ai/`.

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| TUI Foundation | Ink crash leaves terminal in raw mode (Pitfall 3) | Add uncaughtException cleanup handlers before render() |
| TUI Foundation | Bun Windows setRawMode flag overwrite (Pitfall 4) | Pin Bun ≥ 1.2.0; add Windows smoke test |
| TUI Foundation | useFocus opt-in forgotten in new panels (Pitfall 14) | Convention + integration test |
| Auth: GitHub Device Flow | device_flow_disabled (Pitfall 7) | Pre-enable in OAuth App; handle error code explicitly |
| Auth: GitHub Device Flow | slow_down accumulates interval (Pitfall 8) | Track interval additively in polling loop |
| Memory & Sessions | observationalMemory defaults to Gemini (Pitfall 2) | Always pass explicit model via CopilotGateway |
| Memory & Sessions | Working memory requires mastra_resources table (Pitfall 6) | Verify LibSQL path writability at startup |
| Memory & Sessions | threadId resourceId immutable (Pitfall 15) | Use GitHub user.id (numeric) as resourceId from day 1 |
| Research Workflow | .parallel() inputSchema keying (Pitfall 1) | Named constants for step IDs; unit-test schema contract |
| Research Workflow | { failed } pattern not checked downstream (Pitfall 19) | Explicit failure handling in synthesize step |
| Python Bridge & File Gen | Python not on PATH on Windows (Pitfall 13) | Probe at startup; non-fatal warning |
| Python Bridge & File Gen | SIGTERM doesn't kill on Windows (Pitfall 12) | Use child.kill(); single-process Python scripts |
| Python Bridge & File Gen | import.meta.url in dist (Pitfall 5) | Use new URL('./...', import.meta.url) pattern in ESM dist |
| Publishing | npm files field omits dist (Pitfall 17) | Explicit files array + npm pack --dry-run CI |
| Publishing | tsup strips/misses shebang (Pitfall 10) | banner: { js: '#!/usr/bin/env node' } in tsup config |
| All Windows Dev | LibSQL WAL file locking (Pitfall 11) | busy_timeout; document single-instance constraint |
| All TypeScript | noUncheckedIndexedAccess on arrays (Pitfall 16) | Optional chaining everywhere; team awareness |
| All Deployments | conf stores to AppData on Windows (Pitfall 20) | Pin cwd to ~/.config/reash-ai explicitly |

---

## Sources

- Mastra docs: `docs/workflows/control-flow.md` — parallel step id/inputSchema contract (HIGH confidence)
- Mastra docs: `docs/memory/observational-memory.md` — default model `google/gemini-2.5-flash` (HIGH confidence)
- Mastra docs: `docs/memory/working-memory.md` — `mastra_resources` table requirement for resource-scoped (HIGH confidence)
- Mastra docs: `docs/memory/overview.md` — threadId resourceId immutability warning (HIGH confidence)
- GitHub Docs: Device Flow error codes, polling interval, 10-token cap — https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps (HIGH confidence)
- Bun issue tracker: #25663 (Windows setRawMode flag overwrite), #13405 (import.meta.url in compiled binary) — MEDIUM confidence (issues may be partially fixed)
- Ink v5 docs: alternateScreen, useFocus, ink-testing-library limitations — MEDIUM confidence (from training + project context)
- tsup docs: banner option for shebang injection — MEDIUM confidence (verified via tsup documentation)
- Node.js/Bun child_process: SIGTERM behavior on Windows — MEDIUM confidence (cross-verified with Node.js Windows docs)
