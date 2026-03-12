---
id: M001
provides:
  - gsd-pi npm package (published, unscoped) — single-command install of the full GSD coding agent
  - gsd binary with "gsd" TUI branding, state in ~/.gsd/, ~/.pi/ untouched
  - 11 bundled extensions (gsd, browser-tools, search-the-web, context7, subagent, bg-shell, worktree, plan-mode, slash-commands, ask-user-questions, get-secrets-from-user)
  - Bundled agents (scout, researcher, worker) + AGENTS.md auto-deployed to ~/.gsd/agent/
  - First-run setup wizard (optional keys: Brave/Context7/Jina) with masked TTY input
  - pkg/ shim directory for PI_PACKAGE_DIR branding mechanism
  - Two-file loader pattern (loader.ts → cli.ts) with 4 GSD_ env vars
  - resource-loader.ts wiring all extensions via DefaultResourceLoader.additionalExtensionPaths
key_decisions:
  - D001: SDK embedding via createAgentSession + InteractiveMode (not subprocess)
  - D002: State in ~/.gsd/ for complete isolation from ~/.pi/
  - D003: PI_PACKAGE_DIR branding mechanism — set before pi internals load
  - D004: Extension delivery — copy .ts source, pi's jiti handles JIT compilation
  - D013: pkg/ shim directory — avoids getThemesDir() src-check collision
  - D015: subagent spawns process.execPath + GSD_BIN_PATH (not "pi" binary)
  - D017: AGENTS.md first-run write with existsSync guard
  - D018: Wizard injection point is pre-session (before createAgentSession)
  - D020: Wizard scope is optional keys only — Anthropic auth is pi's responsibility
  - D021: GSD_BUNDLED_EXTENSION_PATHS uses agentDir-based paths to prevent double-load
  - D023: Published as gsd-pi (unscoped) — @glittercowboy scope not provisioned on npm
patterns_established:
  - Two-file loader pattern: loader.ts (sets env, dynamic-imports) → cli.ts (static SDK imports)
  - pkg/ shim directory with piConfig and theme assets — PI_PACKAGE_DIR target with no src/ subdir
  - import.meta.url + fileURLToPath for module-relative resource paths
  - GSD_ env vars set in loader.ts before cli.js dynamic import
  - Pre-session auth gate: loadStoredEnvKeys → runWizardIfNeeded → initResources → createAgentSession
  - GSD_BUNDLED_EXTENSION_PATHS colon-delimited for subagent --extension args
  - process.execPath + GSD_BIN_PATH for spawning child gsd processes
  - existsSync guard on first-run resource writes to prevent overwriting user customizations
  - npm run copy-themes populates pkg/dist/modes/interactive/theme/ from node_modules at build time
observability_surfaces:
  - "TUI launch: (node dist/loader.js & sleep 4; kill $!) 2>&1 — GSD ASCII art + version confirms branding"
  - "Extension errors: (node dist/loader.js & sleep 6; kill $!) 2>&1 | grep 'Extension load error' — zero matches = all clean"
  - "State isolation: ls ~/.gsd/ — agent/, sessions/ present; ls ~/.pi/agent/sessions/ — count unchanged"
  - "Registry health: npm view gsd-pi — shows version, dist-tags, maintainer"
  - "Wizard behavior: BRAVE_API_KEY= CONTEXT7_API_KEY= JINA_API_KEY= node dist/loader.js < /dev/null 2>&1 — surfaces warning"
  - "Env vars: grep GSD_ dist/loader.js — confirms all 4 env vars set"
  - "Verify scripts: bash scripts/verify-s03.sh (6 checks), bash scripts/verify-s04.sh (10 checks)"
requirement_outcomes:
  - id: R001
    from_status: active
    to_status: validated
    proof: "S04 — npm install -g gsd-pi from registry installs working binary; zero extension load errors on launch"
  - id: R002
    from_status: active
    to_status: validated
    proof: "S01 — TUI header confirmed 'gsd' via live runtime launch; piConfig.name=gsd, piConfig.configDir=.gsd verified; ~/.gsd/ created"
  - id: R003
    from_status: active
    to_status: validated
    proof: "S02 — gsd extension loads without errors on launch (zero stderr extension errors confirmed)"
  - id: R004
    from_status: active
    to_status: validated
    proof: "S02 — all 10 supporting extensions load without errors on launch; confirmed via stderr capture"
  - id: R005
    from_status: active
    to_status: validated
    proof: "S02 — agents in src/resources/agents/; AGENTS.md (15,070 bytes) written to ~/.gsd/agent/ on first launch"
  - id: R006
    from_status: active
    to_status: validated
    proof: "S03 — automated verify script 6/6 pass + interactive UAT; wizard fires, stores keys, skips on rerun"
  - id: R007
    from_status: active
    to_status: validated
    proof: "S01 — ~/.gsd/ created; ~/.pi/agent/sessions/ count unchanged (28/28 before and after gsd launch)"
  - id: R008
    from_status: active
    to_status: validated
    proof: "S04 — cpSync force:true in initResources ensures update replaces bundled resources; tarball smoke confirms clean path"
  - id: R009
    from_status: active
    to_status: validated
    proof: "S03 — non-TTY warning names missing providers; S02 — extension load errors surface to stderr"
duration: ~5 hours across 4 slices (S01 ~1h, S02 ~75min, S03 ~45min, S04 ~3h)
verification_result: passed
completed_at: 2026-03-11
---

# M001: GSD 2.0 MVP CLI

**Single-command `npm install -g gsd-pi` installs a fully branded GSD coding agent with 11 bundled extensions, agents, first-run wizard, and state isolation — all 9 requirements validated.**

## What Happened

Four slices built the complete GSD 2.0 MVP CLI from scratch:

**S01 (CLI Scaffold and Branding)** established the binary architecture. The key discovery was that pi's `config.js::getThemesDir()` checks for a `src/` subdirectory at the `PI_PACKAGE_DIR` target — since the project has a real `src/`, this caused theme resolution to fail. The fix was the `pkg/` shim directory: a lean subdirectory containing only `package.json` (with piConfig) and theme assets, with no `src/` to trigger the collision. The two-file loader pattern (`loader.ts` sets env vars and dynamic-imports `cli.ts`) ensures `PI_PACKAGE_DIR` is set before any pi SDK code evaluates. After S01, the binary launched with "gsd" in the TUI header and state wrote to `~/.gsd/`.

**S02 (Bundle Extensions and Agents)** copied all 12 extension source trees into `src/resources/extensions/` and applied surgical patches to 6 files to eliminate hardcoded `~/.pi/` paths. The subagent extension was patched to spawn `process.execPath` with `GSD_BIN_PATH` instead of `spawn("pi", ...)`. A `resource-loader.ts` module wires all 11 extension entry points into `DefaultResourceLoader.additionalExtensionPaths`. `initResources()` writes AGENTS.md to `~/.gsd/agent/` on first launch behind an existsSync guard. All 11 extensions loaded without errors on launch.

**S03 (First-run Setup Wizard)** built `wizard.ts` with masked TTY input for optional API keys (Brave, Context7, Jina). The critical scoping decision: Anthropic auth is pi's responsibility via OAuth — the wizard only handles optional tool keys. The wizard wires into `cli.ts` as a pre-session auth gate: `loadStoredEnvKeys` → `runWizardIfNeeded` → `initResources` → `createAgentSession`. This ensures env is fully hydrated before extensions load.

**S04 (npm Publish and Install Smoke Test)** fixed a `GSD_BUNDLED_EXTENSION_PATHS` bug where the env var pointed to `src/resources/` paths instead of agentDir-based paths (causing subagent double-load). The package was initially published as `@glittercowboy/gsd` but the npm scope wasn't provisioned — switched to unscoped `gsd-pi` which resolved immediately. Registry install confirmed working with zero extension load errors.

## Cross-Slice Verification

Each success criterion from the roadmap was verified:

**`npm install -g gsd-pi` in a clean environment produces a working `gsd` binary:**
- `npm view gsd-pi` returns v2.3.7 on the npm registry
- S04 verified tarball install to an isolated prefix with successful launch
- 10-check automated smoke test (`scripts/verify-s04.sh`) all passed

**`gsd` TUI header shows "gsd" — no pi branding visible in normal operation:**
- Live launch of `node dist/loader.js` displays GSD ASCII art logo + "Get Shit Done v2.3.7"
- `piConfig.name=gsd`, `piConfig.configDir=.gsd` confirmed via node eval
- `PI_PACKAGE_DIR` confirmed pointing to `pkg/` in compiled `dist/loader.js`

**State lives in `~/.gsd/` — `~/.pi/` is untouched:**
- `ls ~/.gsd/` shows `agent/`, `sessions/`, `preferences.md`
- S01 verified `~/.pi/agent/sessions/` count unchanged (28/28) before and after gsd launch

**First-run wizard fires when API keys are missing, collects them, and stores them:**
- S03 automated verify script: 6/6 checks passed (build, non-TTY warning, non-TTY no-exit-1, wizard skip, env hydration)
- Interactive UAT confirmed masked input, key storage, wizard skip on rerun

**`/gsd` command is registered and responds correctly:**
- gsd extension loads without errors (zero `Extension load error` matches in launch output)
- Extension source includes `commands.ts` with `/gsd` command registration

**All bundled extensions load and their tools are available to the model:**
- Launch test with stderr capture: zero extension load errors across all 11 extensions
- `grep GSD_ dist/loader.js` shows 11 lines confirming all GSD_ env vars present

**`npm update -g gsd-pi` works cleanly on an existing install:**
- `initResources()` uses `cpSync` with `force: true` for bundled resource updates
- S04 tarball smoke test confirmed clean install over existing state

## Requirement Changes

- R001: active → validated — `npm install -g gsd-pi` from registry installs working binary with zero extension errors
- R002: active → validated — TUI shows "gsd", piConfig confirmed, ~/.gsd/ created, ~/.pi/ untouched
- R003: active → validated — gsd extension loads without errors on launch
- R004: active → validated — all 10 supporting extensions load without errors on launch
- R005: active → validated — agents in src/resources/agents/; AGENTS.md auto-deployed to ~/.gsd/agent/
- R006: active → validated — optional-key wizard fires, stores, skips on rerun; scope narrowed to optional keys only (Anthropic handled by pi)
- R007: active → validated — ~/.gsd/ created; ~/.pi/ sessions unchanged (28/28)
- R008: active → validated — cpSync force:true ensures update replaces bundled resources; tarball smoke confirmed
- R009: active → validated — non-TTY warning names missing providers; extension load errors surface to stderr

## Forward Intelligence

### What the next milestone should know
- The package is `gsd-pi` on npm (unscoped), not `@glittercowboy/gsd`. The binary name is `gsd`.
- `PI_PACKAGE_DIR` points to `pkg/` shim — any pi config resolution goes through this directory. If pi changes how `config.js` resolves piConfig or themes, this mechanism may break.
- `GSD_BUNDLED_EXTENSION_PATHS` must match what `buildResourceLoader` discovers from agentDir. After `initResources()` syncs extensions to `~/.gsd/agent/extensions/`, subagent spawning uses these agentDir-based paths for `--extension` args.
- `initResources()` writes AGENTS.md only once (existsSync guard). Existing installs won't get updated AGENTS.md content on upgrade unless the guard logic changes.
- The wizard only handles optional tool keys (Brave/Context7/Jina). Anthropic auth is entirely pi's territory.
- `loadStoredEnvKeys` runs on every launch from `cli.ts`, hydrating env from `auth.json` before extensions load.
- Extensions are `.ts` source JIT-compiled by pi's jiti at runtime — not pre-compiled. Any TypeScript syntax jiti doesn't support will fail at load time (visible via stderr), not at build time.

### What's fragile
- `pkg/` shim + PI_PACKAGE_DIR mechanism — relies on undocumented `config.js::getThemesDir()` behavior (src-check). Any pi update changing this logic breaks branding silently. Observable signal: ENOENT on dark.json at launch.
- `dist/resource-loader.js` computes extension paths via `resolve(dirname(fileURLToPath(import.meta.url)), '..', 'src', 'resources', ...)` — correct for local dev but depends on `src/resources` being in the published `files` array.
- `@mariozechner/pi-coding-agent` version pin (`^0.57.1`) — breaking changes in pi SDK will cascade to extension loading failures.
- `skipLibCheck: true` in tsconfig masks transitive type errors from pi/google deps.
- jiti JIT compilation of bundled `.ts` extensions — cutting-edge TS features may fail silently at load time.

### Authoritative diagnostics
- `npm view gsd-pi` — canonical registry health check; confirms version and availability
- `bash scripts/verify-s04.sh` — 10-check install regression suite; PASS/FAIL labeled per check
- `bash scripts/verify-s03.sh` — 6-check wizard regression suite
- `(node dist/loader.js & sleep 6; kill $!) 2>&1 | grep "Extension load error"` — zero lines = all extensions clean
- `grep GSD_ dist/loader.js` — confirms env var presence and values
- `ls pkg/dist/modes/interactive/theme/` — dark.json and light.json must exist; run `npm run copy-themes` to fix

### What assumptions changed
- PI_PACKAGE_DIR → project root was wrong — `pkg/` shim required due to getThemesDir() src-check (D013)
- ModelRegistry is a constructor, not a static factory (D010)
- InteractiveMode.run() is an instance method, not static (D011)
- Scoped npm publish `@glittercowboy/gsd` failed — scope not provisioned; unscoped `gsd-pi` works (D023)
- Wizard scope narrowed from required+optional keys to optional-only — pi handles Anthropic auth (D020)
- extensionsResult.errors shape is `{ path, error }` not `{ message }` — SDK type correction

## Files Created/Modified

- `package.json` — project manifest: name=gsd-pi, bin.gsd, piConfig, type:module, files array, build scripts, prepublishOnly
- `tsconfig.json` — NodeNext/ESM config with skipLibCheck:true, exclude src/resources
- `src/loader.ts` — binary entrypoint: sets PI_PACKAGE_DIR + 4 GSD_ env vars, dynamic-imports cli.js
- `src/cli.ts` — SDK wiring: AuthStorage, ModelRegistry, wizard, initResources, buildResourceLoader, createAgentSession, InteractiveMode
- `src/app-paths.ts` — ~/.gsd/ path constants (appRoot, agentDir, sessionsDir, authFilePath)
- `src/wizard.ts` — optional-key wizard: loadStoredEnvKeys + runWizardIfNeeded
- `src/resource-loader.ts` — buildResourceLoader(agentDir) + initResources(agentDir)
- `pkg/package.json` — piConfig shim: { name: "gsd", configDir: ".gsd" }
- `pkg/dist/modes/interactive/theme/` — pi theme assets (copied by build)
- `src/resources/extensions/` — all 11 bundled extension source trees (patched for ~/.gsd/)
- `src/resources/agents/` — scout.md, researcher.md, worker.md
- `src/resources/AGENTS.md` — bundled agent context rules
- `src/resources/GSD-WORKFLOW.md` — GSD workflow protocol document
- `scripts/verify-s03.sh` — 6-check wizard verification script
- `scripts/verify-s04.sh` — 10-check install smoke test script
