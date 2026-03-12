# M002: Branded Installer & Onboarding Experience

**Vision:** Transform the entire first-contact experience — from `npm install` through first working session — into a polished, guided, trust-building flow that gets users from zero to productive with no friction.

## Success Criteria

- After `npm install -g gsd-pi`, the terminal shows a clean branded postinstall flow with the GSD ASCII logo, spinners, staged progress, and boxed summary
- On first `gsd` launch, a unified onboarding wizard guides the user through LLM provider auth (OAuth or API key) and optional tool API keys before the TUI opens
- After completing onboarding, the user drops straight into a working TUI session with an authenticated LLM — no need to discover `/login`
- Users who skip onboarding or already have auth configured go straight to the TUI with no friction
- The entire flow is visually polished — comparable to openclaw's onboarding or vercel-labs/skills installer

## Key Risks / Unknowns

- Spinner animation during synchronous subprocess execution — clack's spinner may not animate while `execSync` blocks the event loop → **retired in S01**
- OAuth flows outside TUI — the pi-ai OAuth providers (`loginAnthropic`, etc.) require browser opening + user pasting an auth code back. These are currently wired to the TUI's `LoginDialogComponent`. Need to prove we can drive the same flow from a standalone clack-based wizard using `p.text()` for code input and `exec('open <url>')` for browser opening.
- Clack inside pre-TUI context — `@clack/prompts` writes to stdout. The wizard runs before `InteractiveMode` takes over the terminal. Need to verify that clack's raw mode cleanup (cursor visibility, etc.) doesn't corrupt the subsequent TUI session.

## Proof Strategy

- Spinner + async subprocess → **retired in S01** by proving the spinner animates during Playwright download
- OAuth outside TUI → retire in S03 by proving Anthropic OAuth login works end-to-end from the clack-based onboarding wizard (browser opens, user pastes code, credentials are stored). Originally planned for S02, but S02 was scoped to logo work only.
- Clack → TUI handoff → retire in S03 by proving the TUI starts cleanly after the wizard completes. Originally planned for S02, but S02 was scoped to logo work only.

## Verification Classes

- Contract verification: postinstall and wizard run to completion, produce expected output
- Integration verification: full flow from `npm install -g` → `gsd` → onboarding → working TUI session
- Operational verification: works in TTY and non-TTY, handles failures, respects skip/existing-auth
- UAT / human verification: visual quality judgment, LLM auth actually works for a real chat

## Milestone Definition of Done

This milestone is complete only when all are true:

- All slices are complete and verified
- `npm install -g gsd-pi` produces branded postinstall with ASCII logo
- First `gsd` launch shows the onboarding wizard which guides through LLM auth + optional keys
- After onboarding, the TUI session has a working authenticated LLM
- Returning users (already authed) skip the wizard and go straight to TUI
- The visual quality bar is met for both postinstall and onboarding
- Final integrated acceptance: a fresh install → onboarding → send a real message → get a response

## Requirement Coverage

- Covers: R008 (npm install experience)
- New: R012 (first-run onboarding — LLM auth before TUI)
- Partially covers: none
- Leaves for later: none
- Orphan risks: none

## Slices

- [x] **S01: Branded postinstall with clack** `risk:medium` `depends:[]`
  > After this: `npm install -g gsd-pi` shows a structured, branded installer flow with spinners, staged progress, and boxed summary instead of raw ASCII dump

- [x] **S02: ASCII logo in postinstall + first-launch banner** `risk:low` `depends:[S01]`
  > After this: postinstall shows the GSD block-letter logo before the clack flow; the existing first-launch banner in loader.ts also uses the shared logo constant

- [ ] **S03: Unified first-run onboarding wizard** `risk:high` `depends:[S01]`
  > After this: first `gsd` launch walks the user through LLM provider selection (Anthropic OAuth / API key / OpenAI / others / skip), runs the auth flow, collects optional tool keys, and drops into a working TUI session

## Boundary Map

### S01 → S02

Produces:
- `@clack/prompts` and `picocolors` available as production dependencies
- Postinstall script pattern using clack intro/spinner/note/outro

Consumes:
- nothing (first slice)

### S01 → S03

Produces:
- `@clack/prompts` and `picocolors` available as production dependencies
- Pattern for structured CLI output with clack

Consumes:
- nothing (first slice)

### S02 → S03

Produces:
- Shared ASCII logo constant importable from `src/logo.ts`
- Logo rendering pattern with picocolors

### S03

Consumes:
- `@clack/prompts` and `picocolors` (from S01)
- Shared ASCII logo (from S02)
- `AuthStorage` API: `.set()`, `.has()`, `.login()` (from pi-coding-agent)
- OAuth provider functions: `loginAnthropic`, `loginGitHubCopilot`, etc. (from pi-ai)
- Existing wizard: `runWizardIfNeeded()` in `src/wizard.ts` (to be replaced/absorbed)
