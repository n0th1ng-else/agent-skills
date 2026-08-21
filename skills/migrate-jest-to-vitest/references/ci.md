# Jest → Vitest CI-level failures

Patterns where the migration is correct from a test-runner perspective but breaks an adjacent CI job (build, lint)
because of how the migrated package interacts with the rest of the repo.

## Vitest leaks into the production bundle via a re-exported test fixture

**Symptom.** The CI build job fails (often commits after a green local test run) with bundler errors like:

```
⚠ Critical dependency: the request of a dependency is an expression       (vitest's dynamic `import(filepath)`)
⚠ Critical dependency: Accessing import.meta directly is unsupported      (vitest's `import.meta.resolve(...)`)
```

**Cause.** A test fixture in the migrated package (typically `*Fake.ts`, `*TestUtils.ts`, `*Mock.ts`) does
`import {vi} from 'vitest'` AND is re-exported from the package's barrel (`src/index.ts`). Production bundling resolves
the barrel, pulls Vitest's source in, and the bundler chokes on Vitest's dynamic `import(filepath)` /
`import.meta.resolve(...)` patterns.

Reverting the fixture to `jest.fn()` under a `globalThis.jest = vi` shim just trades this for a lint failure wherever
leftover-`jest.*` scanning is enforced — fix the export boundary instead.

**Fix.** Move the fixture behind a `/testing` subpath export:

1. `package.json` — add an `exports` field:

   ```json
   "exports": {
     ".": "./src/index.ts",
     "./testing": "./src/ThingFake.ts"
   }
   ```

2. `src/index.ts` — remove the fixture's barrel re-export.

3. Each consumer — split the import:

   ```ts
   import {RealThing} from '<pkg>'
   import {ThingFake} from '<pkg>/testing'
   ```

**Verify** with a production build, not just the test run.

## Empty `describe` block fails CI but passes locally (`passWithNoTests` divergence)

**Symptom.** CI fails with:

```
Error: No test found in suite #subSuite
```

even though the package's local test run passed.

**Cause.** Two compounding things:

1. A `describe(...)` registers **no `it`/`test`** — usually bare `expect(...)` calls sitting directly in the `describe`
   body (a latent bug: under Jest those ran at collection time and never failed the suite). Vitest treats a zero-test
   suite as a hard error.
2. The local runner hides it: a wrapper invoking Vitest with `passWithNoTests: true` suppresses the empty-suite error,
   while CI runs raw `vitest` where `passWithNoTests` defaults to `false`.

So a green local per-package run is not a sufficient gate for this class of bug. Check whether your local wrapper and CI
invocation differ (repo adapter in SKILL.md).

**Fix.** Wrap the stray assertions in a real test — don't delete them, they're the intended coverage:

```ts
// BEFORE — bare expects in the describe body; suite registers no tests
describe('#querySelector', () => {
	const engine = new SelectorEngine()
	expect(engine.querySelector(root, 'Text')).toEqual([...]) // ❌ runs at collection, not a test
})

// AFTER
describe('#querySelector', () => {
	const engine = new SelectorEngine()
	it('returns the first matching element', () => {
		expect(engine.querySelector(root, 'Text')).toEqual([...])
	})
})
```

If a `describe` is genuinely meant to be empty, delete it.

## Local wrapper vs raw CI invocation — general note

Both patterns above share a root cause worth checking once per repo: the local dev command and the CI command may not
run Vitest the same way (flags like `passWithNoTests`, sharding, project aggregation, reporter differences that hide the
unhandled-errors section). Diff the two invocations at the start of a migration effort and record the differences in the
repo adapter table.
