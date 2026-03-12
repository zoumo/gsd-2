---
id: T01
parent: S02
milestone: M002
provides:
  - checkExistingEnvKeys utility function (exported)
  - detectDestination utility function (exported)
  - optional destination parameter in TypeBox schema
  - guidance field on key items in TypeBox schema
  - auto-detection of destination in execute()
key_files:
  - src/resources/extensions/get-secrets-from-user.ts
  - src/resources/extensions/gsd/tests/secure-env-collect.test.ts
key_decisions:
  - Used `key in process.env` check (not `process.env[key] !== undefined`) to match empty-string-as-existing semantics
  - detectDestination is synchronous (existsSync/statSync) — fine for one-shot project detection
patterns_established:
  - Test file naming: `secure-env-collect.test.ts` uses node:test + node:assert/strict with temp dirs for fs isolation
observability_surfaces:
  - ToolResultDetails.detectedDestination — present when destination was auto-inferred
  - ToolResultDetails.existingSkipped — ready for T02 to populate with silently skipped keys
duration: 12m
verification_result: passed
completed_at: 2026-03-12
blocker_discovered: false
---

# T01: Add utility functions with tests and schema changes

**Added `checkExistingEnvKeys` and `detectDestination` utilities with 12 unit tests, made `destination` optional with auto-detection, and added `guidance` field to key schema.**

## What Happened

Created two exported utility functions in `get-secrets-from-user.ts`:

1. **`checkExistingEnvKeys(keys, envFilePath)`** — reads `.env` file content (ENOENT-safe), then for each key checks if it exists in the file via regex (`^KEY\s*=`) or in `process.env` (including empty-string values). Returns the subset of keys that already exist.

2. **`detectDestination(basePath)`** — checks for `vercel.json` (→ "vercel"), then `convex/` directory (→ "convex"), fallback → "dotenv". Uses `existsSync`/`statSync` for synchronous one-shot detection.

Updated the TypeBox schema: `destination` is now `Type.Optional(Type.Union([...]))`, and keys gain `guidance: Type.Optional(Type.Array(Type.String()))`.

Updated `execute()` to auto-detect destination when not provided, tracking whether detection occurred in the result details. The `ToolResultDetails` interface gained `existingSkipped` and `detectedDestination` fields.

Created 12 unit tests covering both functions (7 for `checkExistingEnvKeys`, 5 for `detectDestination`) using `node:test` + `node:assert/strict` with temp directory isolation.

## Verification

- `npm run build` — clean compilation, no type errors
- `npm run test -- --test-name-pattern "secure_env_collect"` — all 12 tests pass
- `npm run test` — 66 pass, 2 fail (pre-existing AGENTS.md failures in app-smoke.test.ts)

### Slice-level verification status (T01 of 2):
- ✅ `npm run test -- --test-name-pattern "secure_env_collect"` — all new tests pass
- ✅ `npm run build` — clean compilation
- ✅ `npm run test` — 66 pass, 2 pre-existing failures only

## Diagnostics

- `ToolResultDetails.detectedDestination` surfaces in tool result when destination was auto-inferred (agent can inspect this)
- `ToolResultDetails.existingSkipped` field is defined but not yet populated — T02 will wire `checkExistingEnvKeys` into the execute flow to populate it
- `checkExistingEnvKeys` silently handles ENOENT (no error thrown, returns subset based on process.env)
- `detectDestination` always returns a valid value (fallback "dotenv")

## Deviations

- Added a bonus test: `detectDestination — convex file (not dir) does not trigger convex` — verifies that a regular file named `convex` doesn't falsely trigger convex detection
- `renderCall` updated to show "auto" when destination is not provided (minor backward-compat improvement not in plan)

## Known Issues

None.

## Files Created/Modified

- `src/resources/extensions/get-secrets-from-user.ts` — added `checkExistingEnvKeys()`, `detectDestination()`, updated TypeBox schema (optional destination, guidance field), updated execute() with auto-detection, updated ToolResultDetails interface
- `src/resources/extensions/gsd/tests/secure-env-collect.test.ts` — new test file with 12 test cases for both utility functions
