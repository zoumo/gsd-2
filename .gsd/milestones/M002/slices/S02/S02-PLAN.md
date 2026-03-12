# S02: Enhanced Collection UX

**Goal:** `secure_env_collect` gains four capabilities: multi-line guidance display per key, a summary screen before collection, existing-key detection with silent skip, and automatic destination inference from project context.
**Demo:** Running the enhanced tool with a test manifest (keys with guidance) shows a summary screen listing all needed keys, silently skips keys already in the environment, auto-detects the write destination, and displays step-by-step guidance during per-key collection. Unit tests prove the utility functions. Build passes.

## Must-Haves

- `guidance` field (optional `string[]`) added to the `keys` TypeBox schema — backward compatible with existing callers
- `destination` parameter made optional — auto-detected via `detectDestination()` when omitted
- `checkExistingEnvKeys(keys, envFilePath, cwd)` exported and unit-tested — checks both `.env` file and `process.env`, handles missing `.env` gracefully, treats empty-string values as existing
- `detectDestination(basePath)` exported and unit-tested — checks for `vercel.json` → "vercel", `convex/` dir → "convex", fallback → "dotenv"
- Summary screen TUI component using `makeUI()` — renders before per-key collection when guidance is present, shows all keys with service context and guidance steps, user presses enter to continue
- Per-key collection screen enhanced to show numbered guidance steps below the hint
- `execute()` wired: auto-detect destination → check existing keys → show summary → collect remaining keys with guidance
- All existing callers that provide `destination` and omit `guidance` work identically (backward compat)
- `npm run build` passes
- `npm run test` passes (54 pass, 2 pre-existing failures)

## Proof Level

- This slice proves: contract
- Real runtime required: no (utility functions are unit-tested; TUI components are contract-proven via TypeScript compilation and structural render)
- Human/UAT required: no (TUI visual quality is structure-proven; real UX validation happens in S04 end-to-end)

## Verification

- `npm run test -- --test-name-pattern "secure_env_collect"` — all new tests pass
- `npm run build` — clean compilation
- `npm run test` — 54+ pass, 2 pre-existing failures
- Test file: `src/resources/extensions/gsd/tests/secure-env-collect.test.ts`
  - `checkExistingEnvKeys`: finds keys in `.env` file, finds keys in `process.env`, handles missing `.env`, treats empty values as existing, returns only existing keys from input list
  - `detectDestination`: returns "vercel" when `vercel.json` exists, returns "convex" when `convex/` dir exists, returns "dotenv" as fallback, checks vercel before convex when both exist

## Observability / Diagnostics

- Runtime signals: Summary screen render logs key count and skip count to the result details. The `execute()` result includes `autoDetected: true` when destination was inferred.
- Inspection surfaces: The tool's result `details` object gains `existingSkipped: string[]` listing keys that were silently skipped, and `detectedDestination: string` when auto-detection was used.
- Failure visibility: `checkExistingEnvKeys` silently handles ENOENT (returns empty array for file checks). `detectDestination` always returns a valid value (fallback "dotenv"). Errors during collection are already captured in the result's error array.
- Redaction constraints: Secret values never appear in logs, result details, or summary screen. Only key names are displayed.

## Integration Closure

- Upstream surfaces consumed: `shared/ui.ts` (`makeUI()` design system), `shared/confirm-ui.ts` (pattern reference for `ctx.ui.custom()` components)
- New wiring introduced in this slice: `checkExistingEnvKeys()` and `detectDestination()` exported for S03 consumption. `guidance` field on TypeBox schema available for S03 callers. Summary screen function available as internal module function.
- What remains before the milestone is truly usable end-to-end: S03 (auto-mode dispatches collect-secrets phase, guided flow triggers collection, reads manifest and passes entries to enhanced tool), S04 (end-to-end verification)

## Tasks

- [x] **T01: Add utility functions with tests and schema changes** `est:25m`
  - Why: Establishes the testable foundation — `checkExistingEnvKeys`, `detectDestination`, and the `guidance`/optional-`destination` schema changes. All non-TUI code. Tests prove the utilities work before the TUI integration task.
  - Files: `src/resources/extensions/get-secrets-from-user.ts`, `src/resources/extensions/gsd/tests/secure-env-collect.test.ts`
  - Do: Create test file with assertions for both utility functions. Add `checkExistingEnvKeys(keys, envFilePath, cwd)` — reads `.env` with try/catch, checks `process.env`, returns keys that already exist. Add `detectDestination(basePath)` — checks `vercel.json` (existsSync), `convex/` dir (existsSync), fallback "dotenv". Make `destination` optional in TypeBox schema with `Type.Optional()`. Add `guidance` field as `Type.Optional(Type.Array(Type.String()))` to key items. Update `execute()` to default destination via `detectDestination(ctx.cwd)` when not provided. Handle `params.destination` being `string | undefined`.
  - Verify: `npm run test -- --test-name-pattern "secure_env_collect"` passes; `npm run build` passes; `npm run test` — 54+ pass
  - Done when: Both utility functions are exported, unit-tested, and the schema accepts optional `destination` and `guidance` fields without breaking existing callers

- [ ] **T02: Build summary screen, enhance guidance display, and wire execute flow** `est:30m`
  - Why: Delivers the user-facing UX — the summary screen before collection, numbered guidance in per-key screens, and the full execute flow with existing-key skip and auto-detection wired in. Completes all four R004/R005/R006/R010 requirements.
  - Files: `src/resources/extensions/get-secrets-from-user.ts`, `src/resources/extensions/shared/ui.ts` (reference only)
  - Do: Add `showSecretsSummary()` function using `ctx.ui.custom()` + `makeUI()` — renders key list with `progressItem()` per key (pending status), `progressAnnotation()` for each guidance step, hints line with "enter to continue", returns void on enter/escape. Enhance `collectOneSecret()` to accept `guidance: string[] | undefined` parameter — render numbered guidance steps below hint using `theme.fg("muted")`. Wire `execute()`: (1) resolve destination via `detectDestination` if not provided, (2) call `checkExistingEnvKeys` to get existing keys, (3) filter keys list to only uncollected ones, (4) if any keys have guidance, call `showSecretsSummary()` with all original keys (marking existing ones as done), (5) loop through filtered keys calling `collectOneSecret` with guidance. Update result `details` to include `existingSkipped` and `detectedDestination`.
  - Verify: `npm run build` passes; `npm run test` — 54+ pass (no regressions); summary screen function exists and compiles; execute flow handles all code paths
  - Done when: The full execute flow works — auto-detects destination, skips existing keys, shows summary, displays guidance per key, collects remaining, and reports results with skip/detection metadata

## Files Likely Touched

- `src/resources/extensions/get-secrets-from-user.ts` — main target: utilities, schema, summary screen, execute wiring
- `src/resources/extensions/gsd/tests/secure-env-collect.test.ts` — new test file for utility functions
- `src/resources/extensions/shared/ui.ts` — imported by summary screen (no modifications)
- `src/resources/extensions/shared/confirm-ui.ts` — pattern reference (no modifications)
