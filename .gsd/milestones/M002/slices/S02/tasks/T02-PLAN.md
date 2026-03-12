---
estimated_steps: 5
estimated_files: 2
---

# T02: Build summary screen, enhance guidance display, and wire execute flow

**Slice:** S02 — Enhanced Collection UX
**Milestone:** M002

## Description

Delivers the user-facing TUI enhancements: a summary screen showing all needed keys with guidance before collection begins, numbered guidance steps in each per-key collection screen, and the full wired execute flow that skips existing keys and shows the summary. This completes the slice by connecting T01's utilities into the real collection pipeline.

## Steps

1. **Add `showSecretsSummary()` function** in `get-secrets-from-user.ts`. Import `makeUI` from `./shared/ui.js`. Signature: `async function showSecretsSummary(ctx, keys, existingKeys)` where keys is the full array with guidance and existingKeys is the set of already-present key names. Uses `ctx.ui.custom()` following the `confirm-ui.ts` pattern. Render function:
   - `ui.bar()`, `ui.blank()`, `ui.header("  Secret Collection")`, `ui.meta("  N keys needed · M already configured")`
   - `ui.blank()`
   - For each key: `ui.progressItem(key.key, existingKeys.has(key.key) ? "done" : "pending", { detail: key.hint ?? "" })`. If key has guidance: render each step as `ui.progressAnnotation("N. step text")`.
   - `ui.blank()`, `ui.hints(["enter to continue"])`, `ui.bar()`
   - Handle input: enter or escape → `done(null)`. No cursor navigation needed (informational display).

2. **Enhance `collectOneSecret()` to display guidance** — Add `guidance: string[] | undefined` parameter after `hint`. In the render function, after the hint line and before "Preview:", add a numbered guidance list: for each guidance step, render `theme.fg("muted", `  ${i+1}. ${step}`)`. Add a blank line after guidance if present.

3. **Wire existing-key skip into `execute()`** — After destination resolution (from T01), call `checkExistingEnvKeys(params.keys.map(k => k.key), envFilePath)` where envFilePath is `resolve(ctx.cwd, params.envFilePath ?? ".env")` for dotenv destinations (for vercel/convex, only check `process.env` by passing a nonexistent path). Build a `Set<string>` of existing keys. Filter `params.keys` to `remainingKeys` (those not in the existing set). Track `existingSkipped` as the existing key names.

4. **Wire summary screen into `execute()`** — After existing-key detection, if any key in the original `params.keys` array has a `guidance` field with entries, call `showSecretsSummary(ctx, params.keys, existingKeySet)`. This shows ALL keys (existing marked as done, remaining marked as pending) so the user sees the full picture.

5. **Wire guidance into collection loop and update result details** — Change the collection loop to iterate over `remainingKeys` instead of `params.keys`. Pass `item.guidance` to `collectOneSecret()`. Update result details: add `existingSkipped` array and `detectedDestination` string (when auto-detected). Verify the result text includes skipped-existing keys in the summary output (e.g., `⊘ KEY: already configured`).

## Must-Haves

- [ ] `showSecretsSummary()` renders via `ctx.ui.custom()` using `makeUI()` with `progressItem()` and `progressAnnotation()`
- [ ] Summary screen shows existing keys as "done" status and remaining keys as "pending"
- [ ] Summary screen displays numbered guidance steps per key via `progressAnnotation()`
- [ ] `collectOneSecret()` renders numbered guidance steps below hint
- [ ] `execute()` calls `checkExistingEnvKeys` and silently skips existing keys
- [ ] `execute()` shows summary screen when any key has guidance
- [ ] `execute()` only collects remaining (non-existing) keys
- [ ] Result details include `existingSkipped` array and `detectedDestination` string
- [ ] Result text summary includes already-configured keys with distinct marker
- [ ] `npm run build` passes
- [ ] `npm run test` — 54+ pass, 2 pre-existing failures only

## Verification

- `npm run build` — clean compilation (TypeScript catches type mismatches in TUI component render functions, `makeUI()` usage, `progressItem()`/`progressAnnotation()` calls)
- `npm run test` — no new failures (TUI changes don't break existing test suite)
- Manual code review: `showSecretsSummary()` follows `confirm-ui.ts` pattern (render/handleInput/invalidate triple), uses `makeUI()` consistently

## Observability Impact

- Signals added/changed: Result details gain `existingSkipped: string[]` and `detectedDestination: string` — visible in tool result for the calling LLM agent
- How a future agent inspects this: Tool result `details` shows which keys were skipped (existing) vs collected vs user-skipped, and whether destination was auto-detected
- Failure state exposed: If summary screen fails to render, `ctx.ui.custom` will throw — caught by the existing execute error handling. Missing `makeUI` import would be caught at build time.

## Inputs

- `src/resources/extensions/get-secrets-from-user.ts` — after T01 modifications (has `checkExistingEnvKeys`, `detectDestination`, updated schema with `guidance` and optional `destination`)
- `src/resources/extensions/shared/ui.ts` — `makeUI()` with `progressItem()`, `progressAnnotation()`, `bar()`, `header()`, `meta()`, `hints()`, `blank()`
- `src/resources/extensions/shared/confirm-ui.ts` — pattern for `ctx.ui.custom()` render/handleInput/invalidate triple
- S01 forward intelligence: guidance is `string[]`, display as numbered list not paragraph

## Expected Output

- `src/resources/extensions/get-secrets-from-user.ts` — gains `showSecretsSummary()`, enhanced `collectOneSecret()` with guidance display, fully wired `execute()` flow with skip + summary + guidance. All four R004/R005/R006/R010 capabilities working.
