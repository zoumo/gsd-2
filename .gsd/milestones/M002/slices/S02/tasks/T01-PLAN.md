---
estimated_steps: 5
estimated_files: 2
---

# T01: Add utility functions with tests and schema changes

**Slice:** S02 — Enhanced Collection UX
**Milestone:** M002

## Description

Establishes the non-TUI foundation for S02: two exported utility functions (`checkExistingEnvKeys` and `detectDestination`), their unit tests, and the TypeBox schema changes (optional `destination`, new `guidance` field on keys). The execute function gains destination auto-detection when `destination` is omitted. No TUI changes — that's T02.

## Steps

1. **Create test file** `src/resources/extensions/gsd/tests/secure-env-collect.test.ts` with test groups for `checkExistingEnvKeys` and `detectDestination`. Use Node's built-in test runner (`node:test` + `node:assert`). Tests should initially fail (functions don't exist yet). Test cases:
   - `checkExistingEnvKeys`: key found in `.env` file, key found in `process.env`, key found in both, key not found anywhere, `.env` file doesn't exist (ENOENT → still checks process.env), empty-string value in process.env counts as existing, returns only existing keys from input list
   - `detectDestination`: returns "vercel" when `vercel.json` exists in basePath, returns "convex" when `convex/` dir exists in basePath, returns "dotenv" when neither exists, vercel takes priority when both exist

2. **Implement `checkExistingEnvKeys`** in `get-secrets-from-user.ts`. Export it. Signature: `async function checkExistingEnvKeys(keys: string[], envFilePath: string): Promise<string[]>`. Reads `.env` with try/catch (ENOENT → empty content). For each key: check if regex `^KEY\s*=` matches in file content OR `key in process.env` (including empty string values). Return array of keys that exist. Reuse the regex pattern from `writeEnvKey` for consistency.

3. **Implement `detectDestination`** in `get-secrets-from-user.ts`. Export it. Signature: `function detectDestination(basePath: string): "dotenv" | "vercel" | "convex"`. Uses `existsSync` from `node:fs` (synchronous, fine for one-shot check). Check `resolve(basePath, "vercel.json")` → "vercel". Check `resolve(basePath, "convex")` with `statSync` for directory → "convex". Fallback → "dotenv". Import `existsSync` and `statSync` from `node:fs`.

4. **Update TypeBox schema**: Make `destination` optional with `Type.Optional(Type.Union([...]))`. Add `guidance` to keys items: `Type.Optional(Type.Array(Type.String(), { description: "Step-by-step guidance for finding this key" }))`. Update the `ToolResultDetails` interface to add `existingSkipped?: string[]` and `detectedDestination?: string`.

5. **Update `execute()` destination handling**: At the top of execute, add: `const destination = params.destination ?? detectDestination(ctx.cwd)`. Replace all subsequent `params.destination` references with `destination`. Track whether destination was auto-detected for result details.

## Must-Haves

- [ ] `checkExistingEnvKeys` exported, checks both `.env` and `process.env`, handles ENOENT, treats empty values as existing
- [ ] `detectDestination` exported, checks vercel.json then convex/ dir, fallback dotenv
- [ ] `destination` parameter is optional in TypeBox schema
- [ ] `guidance` field is optional `string[]` on key items in TypeBox schema
- [ ] `execute()` auto-detects destination when not provided
- [ ] All tests in `secure-env-collect.test.ts` pass
- [ ] `npm run build` passes
- [ ] `npm run test` — 54+ pass, 2 pre-existing failures only

## Verification

- `npm run test -- --test-name-pattern "secure_env_collect"` — all new test assertions pass
- `npm run build` — clean TypeScript compilation (catches schema type mismatches)
- `npm run test` — no new failures beyond the 2 pre-existing ones

## Observability Impact

- Signals added/changed: `ToolResultDetails` gains `existingSkipped` and `detectedDestination` fields — these surface in the tool result for agent inspection
- How a future agent inspects this: The tool result `details` object shows which keys were auto-skipped and whether destination was inferred
- Failure state exposed: `checkExistingEnvKeys` silently handles ENOENT (no error thrown, returns subset). `detectDestination` always returns a valid value.

## Inputs

- `src/resources/extensions/get-secrets-from-user.ts` — existing 352-line tool implementation
- `src/resources/extensions/shared/confirm-ui.ts` — pattern reference for imports (not modified)
- D008 — silent skip for existing keys, no confirmation
- D009 — destination inferred from project context, fallback to .env

## Expected Output

- `src/resources/extensions/gsd/tests/secure-env-collect.test.ts` — new test file with ~15-20 test cases for both utility functions
- `src/resources/extensions/get-secrets-from-user.ts` — gains `checkExistingEnvKeys()`, `detectDestination()`, updated TypeBox schema, updated execute() destination handling
