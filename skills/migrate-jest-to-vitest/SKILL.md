---
name: migrate-jest-to-vitest
description: |
  Portable, repo-agnostic playbook for migrating a codebase (or one package of a monorepo) from Jest to
  Vitest, plus a symptom-indexed catalog of every known Jest→Vitest behavior difference.

  TRIGGER when user asks about:
  - Migrating a project or package from Jest to Vitest
  - Porting a `jest.config.js` to `vitest.config.ts`
  - Triaging test failures that appeared after switching to Vitest
  - Jest APIs with no direct Vitest equivalent (`done` callbacks, `requireActual`, `replaceProperty`,
    `isolateModulesAsync`, `jest-when`, custom matcher types)

  Trigger keywords: jest to vitest, vitest migration, migrate to vitest, vi.mock, importActual, vi.hoisted
---

# Jest → Vitest Migration

A migration is three kinds of work: **config porting** (jest.config → vitest.config), **mechanical renames** (`jest.*` →
`vi.*` plus a handful of API-shape changes), and **triage** (tests that were silently wrong or relied on Jest-specific
semantics and now fail honestly). The first two are fast and boring; almost all of the elapsed time goes to triage, and
almost every triage failure matches a known pattern in [references/patterns.md](references/patterns.md). Vitest's
official [Migrating from Jest](https://vitest.dev/guide/migration.html#jest) guide covers the baseline API differences;
this playbook layers the process and a field-tested failure catalog on top.

## Ground rules

1. **Migrate one package (or one bounded directory) at a time.** A migration diff should be reviewable in one sitting.
   Never run the whole repo's test suite to validate one package — scope every run.
2. **Renames are mechanical and confined to test infrastructure.** Test files, fixtures, mocks, and setup files change;
   business/source logic does not. If making a test pass seems to require editing application code, stop — that is a
   triage finding (usually a latent test bug Jest tolerated), not a rename.
3. **A migration is not done while any uncaught exception or unhandled rejection remains in the run output.** Vitest
   reports these in a separate section after the test results — a suite can be all green and still leave errors that
   fail CI or flake later runs. Address every one; they almost always trace back to a pattern in the catalog. The same
   applies to error noise inside the test logs themselves (`Error`, `Uncaught error`, `console.error` output) that fails
   nothing and doesn't reach the post-run section either — a green run that floods the logs is a maintenance burden;
   triage those too.
4. **Don't change test logic or assertions** unless a documented pattern forces a real fix. Snapshot diffs get
   investigated before re-recording.
5. **Fixes default upstream.** If a fix (polyfill, alias, `server.deps.inline` entry, global setup) would help the next
   package too, it belongs in the shared base config, not in a per-package config. Per-package configs are for genuinely
   per-package overrides only.
6. **Before inventing a Vitest-side fix for a config-level failure, read how the old Jest setup solved the same
   problem** (`moduleNameMapper`, `transform`, `setupFiles`, custom resolvers). Port the existing solution — it stays
   consistent with the Jest path and is easy to delete once Jest is gone.

## Repo adapter — fill this in for your repository

Everything below this table is generic. These are the repo-specific facts the playbook needs; fill them in once and keep
them current.

| Question                                                          | Answer for this repo                         |
| ----------------------------------------------------------------- | -------------------------------------------- |
| Run one package's tests                                           | _e.g. `yarn workspace <pkg> test`_           |
| Run a single test file                                            | _e.g. `npx vitest run path/to/file.test.ts`_ |
| Shared/base vitest config location and layering                   | _path(s)_                                    |
| Global setup / polyfills file (what's already shimmed globally?)  | _path_                                       |
| Does console output (`console.error`/`warn`) fail tests?          | _yes/no + mechanism_                         |
| Lint/scan that catches leftover `jest.*` usages                   | _command_                                    |
| Typecheck command that includes test files                        | _command_                                    |
| Does local runner differ from CI runner? (`passWithNoTests` etc.) | _details_                                    |
| Suites that must stay on Jest (holdouts)                          | _list_                                       |
| Is there a `jest = vi` compatibility shim?                        | _yes/no_                                     |

## Per-package workflow

### 1. Switch the test scripts

Point the package's `test` script at Vitest. Update **all** Jest-invoking scripts, not just `test` — stale ones
(`test:debug`, `test:cov`, `test:coverage`, `jest:watch`, a bare `jest` script) keep running Jest silently after the
migration. If the package has zero tests, this step and cleanup (step 6) may be all there is.

### 2. Port or delete the Jest config

Most per-package Jest configs are no-ops over a shared base — delete those; the package should inherit the shared Vitest
config. Port a local `vitest.config.ts` only when the Jest config had real overrides (package-specific `setupFiles`,
`environmentOptions`, `globalSetup`). Option-by-option mapping: [references/config.md](references/config.md).

### 3. Mechanical renames

- `jest.%name%` → `vi.%name%` in test files **and** in fixtures/mocks/setup files that exist to support tests.
  `jest.mock` → `vi.mock` is mandatory even where a compatibility shim exists: Vitest only hoists literal `vi.mock(...)`
  calls at parse time, so a shimmed `jest.mock(...)` runs after the target module has already been imported.
- **Import test APIs explicitly** — `import {describe, it, expect, vi, beforeEach, type Mock} from 'vitest'`. Explicit
  imports keep files portable and make go-to-definition work. Same for helper/fixture files.
- **One mock registration per module per file.** Never two top-level `vi.mock('mod', …)` for the same path, and never a
  hoisted `vi.mock` paired with a `vi.doMock` for the same module — the registrations race and the test flakes on CI
  while passing locally. To vary a mocked value per test, register once with a mutable `vi.hoisted` holder (see the
  catalog).
- Files that put `jest.mock(...)` above their imports (often with `eslint-disable import/first` pragmas): move the
  `vi.mock()` calls below the static imports and delete the pragmas — Vitest hoists the mocks automatically.
- If a setup file has "jest" in its name, rename it; check whether its content is still needed at all (shared polyfills
  may already cover it).

### 4. Run, scoped to the package

Run the package's tests. Read the **entire** output, not just the pass/fail summary: the post-run unhandled-errors
section **and** the per-test logs — errors logged mid-run that fail nothing still count as triage findings (ground rule
3).

### 5. Triage failures

Classify each failure and look it up:

| Class            | Symptoms                                                                                   | Reference                                        |
| ---------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------ |
| **Config-level** | `window`/`document` undefined, module-resolution errors, missing globals, transform errors | [references/config.md](references/config.md)     |
| **Test-level**   | mock factory errors, hoisting errors, timing/async flakes, typing errors, snapshot diffs   | [references/patterns.md](references/patterns.md) |
| **CI-level**     | package tests green but build/lint/CI jobs fail                                            | [references/ci.md](references/ci.md)             |

The patterns catalog is symptom-indexed — search it for the error message text first.

### 6. Cleanup

- Delete the package's `jest.config.js` and any obsolete Jest snapshots.
- Keep a local `vitest.config.ts` only if it contributes something beyond the shared config.
- Typecheck with a command that **includes test files** — there may be a separate typecheck script that covers test
  files (repo adapter). Baseline against the pre-migration branch so you only act on newly introduced errors.
- Run whatever lint/scan catches leftover `jest.*` usages (repo adapter).

### 7. Verify the diff

The diff should contain: test-script changes in `package.json`, deleted `jest.config.js`, optional `vitest.config.ts`,
`jest.*` → `vi.*` renames in test infrastructure, and specific triage fixes. **No edits to business/source logic** — the
one sanctioned exception is extracting a function into its own module to make it spyable under ESM (see "Same-module
self-spying" in the catalog), which moves code without changing behavior. Holdout suites (repo adapter) must be
untouched and absent from the Vitest run output.

### 8. Update this skill

**Always do this last.** If anything surfaced that the catalog doesn't cover — a new failure pattern, a new config
mapping, a wrong assumption in this playbook — record it before closing the task: test-level patterns in
[references/patterns.md](references/patterns.md), config mappings in [references/config.md](references/config.md),
CI-level traps in [references/ci.md](references/ci.md). This skill is designed to accrete: it gets its value from every
migration writing back what it learned, and a finding left unrecorded is a tax on the next package.
