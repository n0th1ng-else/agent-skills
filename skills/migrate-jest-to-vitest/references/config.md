# jest.config → vitest.config mapping

Reference for porting per-package Jest configs. Most per-package configs in a monorepo are no-ops over a shared base —
delete those and let the package inherit the shared Vitest config. Use this mapping when there are real overrides; the
full option list is in the [Vitest config reference](https://vitest.dev/config/).

## Option mapping

| Jest                                      | Vitest                                                                                                                                               |
| ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `testEnvironment: 'jsdom'` / `'node'`     | `environment: 'jsdom'` / `'node'` (if your shared base already sets one, delete the override instead)                                                |
| `testEnvironmentOptions.url`              | `environmentOptions.jsdom.url`                                                                                                                       |
| `setupFiles`                              | `setupFiles` (merged with / after any shared-base setup files — verify ordering)                                                                     |
| `setupFilesAfterEach`                     | `setupFiles` (Vitest doesn't distinguish; registration order is preserved)                                                                           |
| `moduleNameMapper`                        | `resolve.alias` — or rely on tsconfig paths via `vite-tsconfig-paths` if wired                                                                       |
| `testMatch` / `testRegex`                 | `include` (Vitest default `**/*.{test,spec}.?(c\|m)[jt]s?(x)` matches any depth — overrides widening the match to extra dirs can usually be dropped) |
| `testPathIgnorePatterns`                  | `exclude`                                                                                                                                            |
| `transform` for ESM-only deps             | usually drop — Vitest handles ESM natively. If a real error surfaces: `server.deps.inline: ['<package>']`                                            |
| `transformIgnorePatterns`                 | usually drop. If needed: `server.deps.inline`                                                                                                        |
| `globalSetup`                             | `globalSetup` (same shape)                                                                                                                           |
| `snapshotSerializers`                     | `snapshotSerializers`                                                                                                                                |
| `clearMocks` / `restoreMocks` / `globals` | same names under `test:` — if the shared base sets them, don't re-set                                                                                |
| `coveragePathIgnorePatterns`              | `coverage.exclude`                                                                                                                                   |
| `fakeTimers`                              | `fakeTimers` (sinon-based; different default `toFake` set — see patterns.md §3)                                                                      |
| `testTimeout` / `jest.setTimeout(ms)`     | `testTimeout` in config, or `vi.setConfig({testTimeout})` in the file — first try deleting it                                                        |
| `testEnvironmentOptions.globalsCleanup`   | not needed — Vitest resets between tests                                                                                                             |

A minimal per-package config extending a shared base:

```ts
import {defineProject, mergeConfig} from 'vitest/config'
import baseConfig from '<your-shared-base>/vitest.config'

export default mergeConfig(
	baseConfig,
	defineProject({
		test: {
			setupFiles: ['./setupVitest.ts'],
		},
	})
)
```

## Shared-config-first principle

A local per-package config should only contain things ported from that package's old Jest config. Anything else is
almost certainly a global issue that will hit many packages — fix it upstream:

- Missing polyfill / browser global → the shared polyfills/setup file
- ESM-only dep failing to resolve → `server.deps.inline` in the shared base
- jsdom env tweak (URL, options) → shared jsdom config
- A default useful everywhere (alias, `define`, exclude pattern) → shared base

If the same fix would help package N+1, it belongs upstream. Before adding any per-package polyfill, check what the
shared setup already provides (`matchMedia`, `ResizeObserver`, `IntersectionObserver`, `PointerEvent`, fetch/undici
globals) — a Jest-era setup file whose only job was such a shim should be deleted, not ported.

## When to delete the local config entirely

- All overrides happen to match the shared base exactly.
- The old `jest.config.js` was a no-op spread of the shared preset.

No local config is the preferred end state — in monorepos that aggregate config-less packages into one Vitest project,
it also dedupes transforms across packages.

## Setup files

- Rename files with "jest" in the name (`setupJest.ts` → `setupVitest.ts`) and re-check their content — much of it is
  often covered by shared polyfills now.
- Port the content, applying the same `jest.*` → `vi.*` renames as test files.
- MSW packages: only the server lifecycle (`listen`/`resetHandlers`/`close`) needs porting. Check that the shared setup
  already shims the fetch/undici globals, and only then delete the Jest-era `jest.polyfills.js` local overrides.

## Validating with the typechecker

Repo-wide typecheck scripts often **exclude** `*.test.ts` files and will report clean even when a migrated test file has
a type error. Validate with a command that includes tests (usually the package's own `tsc -p tsconfig.json`). If that
command pulls a large project graph with pre-existing errors, baseline: capture the same command's output on the
pre-migration branch and compare error counts + codes per file (line numbers shift when imports are added), acting only
on newly introduced issues.
