# Jest → Vitest pattern catalog

Symptom-indexed. When triaging, search this file for the error message text first. Ordered by migration phase:
mechanical mapping → module mocking → async/timers → jsdom environment → TypeScript → assertions. Baseline API
differences are documented in Vitest's [Migrating from Jest](https://vitest.dev/guide/migration.html#jest) guide; this
catalog indexes the failures hit in practice, by symptom.

---

## 1. Mechanical API mapping

### Import test APIs explicitly — Vitest globals are off by default

Jest injects `describe`/`it`/`expect`/`jest` as globals. Vitest does not unless `globals: true` is set — and even then,
prefer explicit imports (portable files, working go-to-definition, no surprise if globals are turned off):

```ts
import {describe, it, expect, vi, beforeEach, afterEach, type Mock} from 'vitest'
```

Direct equivalents that need only the rename: `jest.fn` → `vi.fn`, `jest.spyOn` → `vi.spyOn`, `jest.clearAllMocks` →
`vi.clearAllMocks`, `jest.useFakeTimers`/`useRealTimers`/`advanceTimersByTime` → same on `vi`, `jest.mocked` →
`vi.mocked`, `jest.stubGlobal` (jest: manual assignment) → `vi.stubGlobal`.

### `jest.mock` → `vi.mock` — the rename is mandatory even under a compatibility shim

A `globalThis.jest = vi` shim makes `jest.fn()`/`jest.spyOn()` work unchanged, but **not** `jest.mock()`: Vitest hoists
only literal `vi.mock(...)` calls at parse time. A shimmed `jest.mock(...)` executes after the target module has already
been imported, so the mock is silently never applied.

### `__mocks__` folder mocks are not auto-applied

Jest automatically applies `__mocks__/<module>` for node-module specifiers in every test. Vitest never auto-applies a
manual mock — you must call `vi.mock('<module>')` in the test file, or replicate the old auto-application in a global
setup file:

```ts
// setupVitest.ts — replicate jest's root __mocks__ auto-application for chosen modules
vi.mock('heavy-sdk', () => import('./__mocks__/heavy-sdk'))
```

### `jest.requireActual` → `await vi.importActual` (factory must become async)

`jest.requireActual` is synchronous; `vi.importActual` returns a Promise. Inside a `vi.mock` factory, make the factory
async and parenthesize the spread:

```ts
// BEFORE (jest)
jest.mock('./i18n', () => ({
	...jest.requireActual('./i18n'),
	useTranslator: () => ({t}),
}))

// AFTER (vitest)
vi.mock('./i18n', async () => ({
	...(await vi.importActual('./i18n')),
	useTranslator: () => ({t}),
}))
```

Don't add a generic type argument to `vi.importActual()` when spreading. Add one only when reading a named/default
member off the result — member access on the default `unknown` return needs a type.

If the `requireActual` was inside a **sync** helper function, the mechanical rewrite produces
`SyntaxError: Unexpected reserved word` (an `await` in a sync body) — make the helper async:

```ts
// BEFORE (jest)
function flushPromises(): Promise<void> {
	return new Promise(jest.requireActual('timers').setImmediate)
}

// AFTER (vitest)
async function flushPromises(): Promise<void> {
	const {setImmediate} = await vi.importActual<{setImmediate: typeof globalThis.setImmediate}>('timers')
	return new Promise(setImmediate)
}
```

Why not capture `setImmediate` at module load? `vi.useFakeTimers()` replaces the global; `vi.importActual('timers')`
bypasses the fake-timer patching, so the returned `setImmediate` is always real regardless of fake-timer state.

### `jest.requireMock(module)` → static import + `vi.mocked`

`vi.importMock` is async, which is awkward at module top level. When the file already has `vi.mock('mod')`, a plain
import resolves to the hoisted mock:

```ts
// BEFORE (jest)
jest.mock('../api/users')
const {fetchUser} = jest.requireMock('../api/users')

// AFTER (vitest)
import {fetchUser} from '../api/users'
vi.mock('../api/users')
const mockFetchUser = vi.mocked(fetchUser)
```

### `require('./local-module')` in a test → static import

Symptom: `Error: Cannot find module './x'` under Vitest's ESM runtime. The Jest idiom existed to read a module _after_
`jest.mock` ran; since `vi.mock` is hoisted above static imports, a plain top-level `import` gives the same result.
Convert, and delete now-dead `eslint-disable @typescript-eslint/no-var-requires` pragmas.

### `jest.setTimeout(ms)` — remove, or move to config

There is no `vi.setTimeout`. First try deleting it (Vitest's default is often enough). If genuinely needed:
`vi.setConfig({testTimeout: ms})` in the file, or `testTimeout` in the config.

### `jest.replaceProperty(obj, prop, value)` — no Vitest equivalent

For env vars use `vi.stubEnv` (next section). For non-accessor data properties, define and restore manually:

```ts
const original = browserInfo.mobile
beforeEach(() => {
	Object.defineProperty(browserInfo, 'mobile', {
		value: false,
		configurable: true,
		writable: true,
	})
})
afterEach(() => {
	Object.defineProperty(browserInfo, 'mobile', {
		value: original,
		configurable: true,
		writable: true,
	})
})
```

For getter/setter properties, `vi.spyOn(obj, prop, 'get')` is cleaner.

### Environment variables → `vi.stubEnv` / `vi.unstubAllEnvs`

Instead of mutating `process.env` and restoring manually — works for both `process.env` and `import.meta.env`:

```ts
beforeEach(() => {
	vi.stubEnv('MY_FLAG', 'true')
})
afterEach(() => {
	vi.unstubAllEnvs()
})
```

Clear a variable with `vi.stubEnv('MY_FLAG', undefined)`.

### `xit` / `xdescribe` / `fit` / `fdescribe` → `.skip` / `.only`

Vitest does not define Jest's skip/focus aliases, so the suite fails to **collect** with
`ReferenceError: xit is not defined` — the whole file shows as a failed suite while individual tests look fine, which
can mask the real cause of an adjacent failure. Convert mechanically: `xit`/`xtest` → `it.skip`, `xdescribe` →
`describe.skip`, `fit` → `it.only`, `fdescribe` → `describe.only`. Check for these first whenever a suite fails to load.

### `jest.isolateModulesAsync(fn)` → `vi.resetModules()` + **dynamic** `await import`

Vitest has no `vi.isolateModulesAsync` (and `vi.isolateModules` is sync-callback only). Port to:

```ts
beforeEach(() => {
	vi.resetModules()
})

it('...', async () => {
	const {getThing} = await import('./mod') // fresh module instance — module-scope state reset
})
```

**The import MUST be dynamic and happen after `resetModules()`.** A static top-level import does NOT work:
`vi.resetModules()` only changes what a _future_ `import()` returns; an already-imported static binding stays pinned to
the original module instance. Symptom: the first test passes, later tests inherit stale module-scope state (an
`initialized` flag still `true`, a cached singleton) and fail with confusing assertions.

To assert on a mocked dependency's spy across re-imports, don't re-grab it from a fresh import — create it once with
`vi.hoisted()` so its identity is stable across `resetModules()`:

```ts
const {mockLog} = vi.hoisted(() => ({mockLog: vi.fn()}))
vi.mock('./logger', () => ({logger: {log: mockLog}}))

beforeEach(() => {
	mockLog.mockClear()
	vi.resetModules()
})
```

### `jest.mock(path, factory, {virtual: true})` → drop the third argument

`vi.mock` has no `virtual` option and no third positional argument. For any module that physically resolves, the flag
was never needed — delete it. If the module genuinely does not exist on disk, follow
[Mocking a non-existing module](https://vitest.dev/guide/mocking/modules.html#mocking-non-existing-module).

### `it('...', done => {...})` → async test with `Promise.withResolvers`

Vitest rejects the `done` callback (`done() callback is deprecated`), and even where it appears to run, assertions
inside later callbacks are never awaited — the test passes vacuously.

```ts
// BEFORE (jest)
it('emits ready when initialized', done => {
	emitter.on('ready', payload => {
		expect(payload).toEqual({ok: true})
		done()
	})
	emitter.init()
})

// AFTER (vitest)
it('emits ready when initialized', async () => {
	const {promise, resolve, reject} = Promise.withResolvers<void>()
	emitter.on('ready', payload => {
		try {
			expect(payload).toEqual({ok: true})
			resolve()
		} catch (err) {
			reject(err)
		}
	})
	emitter.init()
	await promise
})
```

The `try`/`reject` wrapper matters: an `expect` that throws inside the callback is otherwise swallowed by the emitter
and the `await promise` hangs until timeout, hiding the real assertion failure.

### Remove local `import '@testing-library/jest-dom'`

If the global setup already registers DOM matchers (check it), delete per-file imports of the matcher package.

---

## 2. Module mocking semantics

### Variables referenced inside `vi.mock` factories → `vi.hoisted`

The most common migration failure. Both runners hoist `mock()` calls to the top of the file, but Vitest's hoisting has
stricter scoping — module-scope variables are **not** available inside the factory when it runs. Symptom:
`ReferenceError: Cannot access 'mockX' before initialization`, or the factory reads `undefined`.

```ts
// BROKEN
const mockFetch = vi.fn()
vi.mock('../api/users', () => ({
	fetchUser: mockFetch, // ❌ not yet initialized when the hoisted factory runs
}))

// FIXED
const {mockFetch} = vi.hoisted(() => ({mockFetch: vi.fn()}))
vi.mock('../api/users', () => ({
	fetchUser: mockFetch, // ✅
}))
```

Apply only when an actual error surfaces — most factories don't reference outer variables.

### Never register two mocks for the same module — it flakes

A module must have **exactly one** mock registration per file. This covers both:

- two top-level `vi.mock('mod', factoryA)` / `vi.mock('mod', factoryB)` for the same path (often a stray copy left
  behind when editing), and
- a hoisted `vi.mock('mod', …)` paired with `vi.doMock('mod', …)` (typically added to vary the value per test alongside
  `vi.resetModules()`).

The registrations **race** and resolution is timing-dependent: locally one factory usually wins; under CI load the other
sometimes wins, and the module-under-test silently reads the wrong value. Do **not** rely on "last `vi.mock` wins" — it
is not reliable across environments for competing top-level factories.

Fix: register once, backed by a mutable `vi.hoisted` holder, and mutate per test. The holder's identity is stable across
`vi.resetModules()`, so a freshly re-imported module always reads the current value:

```ts
// BROKEN — doMock races the hoisted mock
vi.mock('./browser-info', () => ({browserInfo: {mobileApp: false}}))
function mockMobileApp(isMobile: boolean) {
	vi.doMock('./browser-info', () => ({browserInfo: {mobileApp: isMobile}})) // ❌ flaky
}

// FIXED — single registration, mutate the holder
const {browserInfoMock} = vi.hoisted(() => ({browserInfoMock: {mobileApp: false}}))
vi.mock('./browser-info', () => ({browserInfo: browserInfoMock}))

beforeEach(() => {
	vi.resetModules()
	browserInfoMock.mobileApp = false
})

it('returns false on mobile', async () => {
	browserInfoMock.mobileApp = true
	const {isEnabled} = await import('./mod')
	expect(isEnabled()).toBe(false) // ✅ deterministic
})
```

### Nested `vi.mock('mod')` inside `beforeAll`/`describe` — delete it

`vi.mock` calls hoist to the top of the file regardless of where they are written (Vitest warns about this), so a nested
call does not "mock for this block only". When the file also has a top-level `vi.mock('mod', factory)`, the nested bare
auto-mock hoists after it and clobbers the factory — nested accessors then return `undefined`
(`Cannot read properties of undefined`). The same clobbering happened under Jest, so the factory was already dead code
there. Delete the nested call and keep the single top-level mock.

### `vi.unmock` in a `beforeAll` is hoisted file-wide

Jest scoped `unmock` calls; Vitest hoists them, disabling the mock for the **whole file**. Delete or restructure.

### Partial mock omitting an export → `No "X" export is defined on the "mod" mock`

Jest silently left non-mocked exports as `undefined`. Vitest errors when the source imports an export the factory didn't
return. Spread `importActual` so non-overridden exports stay real:

```ts
// BROKEN — source also imports DEFAULT_TITLE, which the factory omits
vi.mock('./utils', () => ({
	setPageTitle: vi.fn(),
}))

// FIXED
vi.mock('./utils', async () => ({
	...(await vi.importActual('./utils')),
	setPageTitle: vi.fn(),
}))
```

For a heavy external module where `importActual` is undesirable, add the missing names as `vi.fn()` stubs instead.

### CJS default-export module (e.g. `lodash/debounce`) → return `{default: ...}`, read `.default`

For a CJS module whose `module.exports` is the value itself, Jest treated the factory return as the export. Under
Vitest's ESM interop the factory return is the module **namespace**, and the consumer's default import reads
`namespace.default`. Symptoms:

- `TypeError: vi.mock("lodash/debounce", factory) is not returning an object. Did you mean to return an object with a "default" key?`
- the value from `importActual` is `{default: fn}`, not `fn` — calling it directly throws.

```ts
// BEFORE (jest)
jest.mock('lodash/debounce', () => (fn: unknown) => fn)

// AFTER (vitest)
vi.mock('lodash/debounce', () => ({default: (fn: unknown) => fn}))
```

Same applies to `jest.mock('m', () => vi.fn(...))` / `() => SomeComponent` — wrap as `{__esModule: true, default: ...}`.
Watch for the broken mechanical wrap `({default: vi.fn()}).mockReturnValue(...)` — it must be
`{default: vi.fn().mockReturnValue(...)}`.

### A `vi.mock` factory must NOT return a `new Proxy(...)`

A Jest idiom returns a `Proxy` so every un-listed export auto-resolves to a fresh mock. Under Vitest this throws
**`TypeError: Cannot create proxy with a non-object as target or handler`** at suite load (no stack — it happens while
the mock registry wraps the factory return into a module namespace).

Fix: return a plain object listing the exports explicitly (grep the source for what it imports from the module).
**Preserve the Proxy's behavior**: it returned a bare `vi.fn()` (returning `undefined`) for auto-stubbed names — map
them to `vi.fn()`, **not** to the real implementation. A test asserting a value derived from a stubbed hook may only
pass because the stub returned `undefined`; substituting the real implementation "to be helpful" silently changes the
assertion.

### Auto-mock of a singleton with getter/setter properties

`vi.mock('mod')` without a factory auto-mocks like Jest, but diverges for module-level singletons exported with accessor
properties (`export const Session = new SessionImpl()` with a `get user()`). Jest's auto-mock turned the property into a
plain assignable value; Vitest keeps the getter intact (always returning `undefined`), so a test's assignment is
silently ignored by the source.

Symptom: test sets `Session.user = {...}`, the assertion sees `undefined`, no error anywhere.

```ts
// FIXED — explicit factory exposing the property as a plain field
vi.mock('./session', () => ({
	Session: {user: undefined},
}))
```

### Barrel `__mocks__/index.ts` re-export breaks mock-binding identity

When a manual mock is a barrel re-exporting leaf modules (`export {useX} from './useX'` inside
`hooks/__mocks__/index.ts`), Jest gave the test's `jest.mocked(useX)` and the source's call site the same function.
Under Vitest the re-export indirection splits them: the test mutates one binding, the source calls another. Symptom: no
error, but `mock.calls` is empty and the source sees the default implementation. Spreading `importActual` of the barrel
does **not** fix it.

Fix: mock and import the **leaf** module directly, bypassing the barrel:

```ts
// BEFORE — barrel auto-mock; binding identity splits
import {useScanSettings} from '../hooks'
vi.mock('../hooks')

// AFTER
import {useScanSettings} from '../hooks/useScanSettings'
vi.mock('../hooks/useScanSettings')
```

### Same-module self-spying — extract the spied function into its own module

`vi.spyOn(moduleNs, 'fn')` does not intercept calls made from inside the same module. In Jest with CJS transforms,
internal references went via `exports.fn()` and the spy worked; Vitest preserves ESM direct bindings, so the internal
call bypasses the spy. This is a documented limitation — see
[Mocking modules pitfalls](https://vitest.dev/guide/mocking/modules.html#mocking-modules-pitfalls).

Fix: move the spied function into its own leaf module so the caller imports it across a module boundary (ESM imports are
live bindings, so the spy is visible to consumers):

```ts
// closest-color.ts (NEW leaf module)
export const getClosestColor = (...) => ...

// color.ts (AFTER — imports across a boundary; re-export preserves the external API)
import {getClosestColor} from './closest-color'
export {getClosestColor} from './closest-color'
export const getColorFromMatrix = (...) => getClosestColor(...)

// color.test.ts
import * as closestColorModule from './closest-color'
vi.spyOn(closestColorModule, 'getClosestColor').mockImplementation(...)
```

Heuristics: extract the function with the smaller diff; move its helper deps with it so the new module is
self-contained; watch for import cycles (if the new module imports back from the original, the cycle re-introduces the
problem). Do **not** try `import * as self from './self'` — ESM namespace bindings are spec-immutable and the self-spy
pattern is unreliable.

This is the one pattern where touching source files is sanctioned — it moves code without changing behavior.

### `vi.fn()` used as a constructor — implementation must be `function`, not arrow

When a mock is invoked with `new` (directly, via a factory returning a class-like export, or stubbing globals like
`ResizeObserver`), an arrow-function implementation throws `is not a constructor`:

```ts
// BEFORE — jest tolerated; vitest throws on `new`
vi.mock('./StyleService', () => ({
	StyleService: vi.fn(() => styleServiceMock),
}))

// AFTER — function declarations are constructible
vi.mock('./StyleService', () => ({
	StyleService: vi.fn(function MockStyleService() {
		return styleServiceMock
	}),
}))
vi.stubGlobal(
	'ResizeObserver',
	vi.fn(function MockResizeObserver(callback) {
		/* ... */
	})
)
```

### `.mockImplementation()` with no arguments does not silence

Jest's no-arg `mockImplementation()` replaced the method with a `() => undefined` no-op. Vitest's no-arg call does
**not** replace the original — the underlying method still runs. Especially visible with console silencers in setups
that fail tests on console output:

```ts
// BEFORE — silently fails to replace
vi.spyOn(console, 'warn').mockImplementation()
// AFTER
vi.spyOn(console, 'warn').mockImplementation(() => {})
```

Where Jest's no-op behavior is needed but typing fights back, `(() => undefined) as never` works.

### `vi.resetAllMocks()` restores spy originals — prefer `vi.clearAllMocks()`

Jest's `resetAllMocks` kept spies installed (with implementation cleared); Vitest's restores the original
implementation. Test suites that relied on a `beforeEach(resetAllMocks)` keeping spies alive start calling real code.
When migrating, prefer `vi.clearAllMocks()` for jest-parity unless the suite genuinely wants restoration.

### `jest-when` default value leaks across tests under Jest but not Vitest

`when(fn).mockReturnValue(x)` (a default, no `.calledWith`) stored state that Jest's `resetAllMocks` did not clear, so a
default registered in one test silently leaked into the next. Vitest clears it, so an unmatched `when(fn).calledWith(a)`
call returns `undefined` where Jest returned the leaked default.

Symptom: a test that never configures the mock for some input passes on Jest but throws on Vitest
(`Cannot read properties of undefined (reading 'map')`). Fix: make the failing test self-contained — register the
default it was implicitly relying on, at the top of that test. Don't add `resetAllWhenMocks()` globally; just make the
under-specified test explicit.

---

## 3. Async, timers, and scheduling

### Un-awaited `expect(...).resolves` / `.rejects` — always await

Jest auto-awaited hanging promise assertions silently. Vitest still auto-awaits at end-of-test but emits a deprecation
`console.warn` ("Promise returned by `expect(actual).resolves...` was not awaited") — and in setups that fail tests on
console output, that warn fails a test. The warn for one un-awaited assertion often trips a **different** test that
happens to be running when it fires, so fix every `.resolves`/`.rejects` in the file, not just the one on the failing
line:

```ts
// BEFORE
it('rejects when X', () => {
	expect(doThing()).rejects.toEqual(new Error('Done'))
})
// AFTER
it('rejects when X', async () => {
	await expect(doThing()).rejects.toEqual(new Error('Done'))
})
```

### Un-awaited `waitFor` / `findBy*`-in-`waitFor` → `MutationObserver is not a constructor`

Tests like this "passed" on Jest while asserting nothing:

```ts
waitFor(() => userEvent.hover(badge))
waitFor(() => expect(screen.findByText(tooltip)).toBeInTheDocument())
```

Two stacked bugs: the `waitFor` promises are abandoned (nothing awaited them), and `expect(screen.findByText(...))`
matches against a **Promise**, which trivially "exists". Under Vitest, after the test returns, jsdom tears down while
the abandoned `waitFor` polls keep running on real timers; the next poll hits `new MutationObserver(...)` after the
global is gone and throws `TypeError: MutationObserver is not a constructor` as an unhandled rejection that fails the
run.

```ts
// AFTER
userEvent.hover(badge)
expect(await screen.findByText(tooltip)).toBeInTheDocument()
userEvent.unhover(badge)
await waitFor(() => expect(screen.queryByText(tooltip)).not.toBeInTheDocument())
```

Idioms: `findBy*` already polls — don't wrap it in `waitFor`; use `await screen.findByText(...)` for "eventually
appears"; use `waitFor(() => expect(queryBy*(...)).not.toBeInTheDocument())` for "eventually disappears"; always `await`
the `waitFor` itself.

### `await waitFor(() => triggerAction())` — trigger ≠ observable outcome

`waitFor`'s contract is "retry until the callback doesn't throw", not "wait for the side-effect to be observable". If
the callback is a state setter (returns `undefined`, never throws), `waitFor` resolves after the **first** invocation,
before React commits the re-render. Jest's scheduler usually committed in time anyway; Vitest's ordering differs and the
following assertion catches the pre-commit DOM — flake-prone, so it survives locally and trips CI.

```ts
// BEFORE — assertion races the commit
await waitFor(() => onChangeCallback(newValue))
expect(saveButton).toBeEnabled()

// AFTER — act flushes the resulting commit before returning
await act(async () => {
	onChangeCallback(newValue)
})
expect(saveButton).toBeEnabled()
```

Reserve `waitFor(() => expect(...))` for genuinely async changes (network, animation, microtask chains).

### `vi.useFakeTimers()` fakes more than Jest did — scope with `toFake`

Vitest's fake timers (sinon) also fake `Intl.DateTimeFormat` and friends — a
`vi.spyOn(Intl.DateTimeFormat.prototype, 'resolvedOptions')` installed after `useFakeTimers()` never fires. Scope the
fakes:

```ts
vi.useFakeTimers({
	toFake: ['setTimeout', 'clearTimeout', 'setInterval', 'clearInterval', 'Date'],
})
```

For flushing real promises while fake timers are on, see the `flushPromises` recipe in §1 (`vi.importActual('timers')`).

### Product-code timers firing after teardown

A source-code `setTimeout` scheduled during the test can fire after per-test teardown (DI cleared, DOM gone) and throw
as an unhandled error that fails the shard. Use fake timers and drain (`vi.runOnlyPendingTimers()`) before teardown, or
stub the scheduling module.

---

## 4. jsdom / environment differences

Vitest runs test code in the **Node realm** with jsdom attached, whereas Jest ran tests inside the jsdom realm. Several
failures trace to this realm split.

### Expected render errors fire jsdom's `window.error` event → uncaught exception after a green test

When a test asserts that a component throws during render (caught by an ErrorBoundary), jsdom **also** dispatches a
synchronous `error` event on `window`. Vitest surfaces these as uncaught exceptions failing the run _after_ the test
passes. Stack signature: `jsdom/.../runtime-script-errors.js` → `EventTarget-impl.dispatchEvent` →
`react-dom/.../invokeGuardedCallback`. Silencing `console.error` is not enough — that stops React's log, not jsdom's
event dispatch:

```ts
describe('<PageX />', () => {
	const suppressWindowError = (e: Event) => e.preventDefault()
	beforeAll(() => window.addEventListener('error', suppressWindowError))
	afterAll(() => window.removeEventListener('error', suppressWindowError))
	beforeEach(() => {
		vi.spyOn(console, 'error').mockImplementation(() => {}) // React's log of the caught error
	})
})
```

### `new MouseEvent('click', {view: window, ...})` → drop `view: window`

jsdom under Vitest rejects the `view` member
(`TypeError: Failed to construct 'MouseEvent': member view is not of type Window`) because the Node-realm `window`
reference isn't recognized as the jsdom `Window`. `view` is almost never relevant — remove it.

### `window.event` is never set

Jest-in-jsdom populated `window.event` during dispatch; with Node-realm listeners it stays undefined. Source code
reading `window.event` needs the test to track the current event manually (capture-phase document listener storing the
event, cleared in `queueMicrotask`).

### jsdom `DOMException` is not `instanceof Error`

`new DOMException('x', 'AbortError')` created in a test fails source-side `err instanceof Error` checks. Use
`Object.assign(new Error('x'), {name: 'AbortError'})` in tests instead.

### `element.blur()` / focus dispatch → `Failed to execute 'contains' on 'Node'`

jsdom retargets focus events at the Window and cross-realm `event.target === window` comparisons differ. Dispatch
`fireEvent.focusIn(<element>)` / `fireEvent.focusOut(<element>)` instead of calling `.focus()`/`.blur()` when this
surfaces.

---

## 5. TypeScript typing differences

Vitest's mock types are systematically **stricter** than `@types/jest`. Typecheck failures after the mechanical rename
are usually pre-existing gaps that Jest's loose `any`-heavy types masked. Note: many repo-wide typecheck scripts exclude
`*.test.ts` — validate with a command that includes tests.

### `jest.SpyInstance` / `vi.SpyInstance` → named type imports

`vi` is a value, not a namespace — `vi.SpyInstance` fails with `Cannot find namespace 'vi'`:

```ts
import {type MockInstance, type Mock, type MockedFunction, type Mocked} from 'vitest'
let logSpy: MockInstance
const fn = some as Mock
```

### `vi.fn()`'s narrower `Mock<Procedure>` exposes incomplete mock objects

`jest.fn()` returned loose `jest.Mock<any, any>`, masking missing required members in object literals that "satisfied"
an interface. After the rename, the typecheck flags the gap
(`Property 'enabled' is missing in type ... but required in type 'ILogger'`). Complete the mock object — this is
test-fixture code:

```ts
// BEFORE — typechecked under jest only because of loose types
const logger = {log: vi.fn(), getLogger: vi.fn()}
// AFTER
const logger = {log: vi.fn(), getLogger: vi.fn(), enabled: vi.fn(() => false)}
```

### `jest.Mock<Return, [Args]>` two-type-arg generic → `Mock<Obj['method']>`

Mapping literally to `Mock<(...args: [A, B]) => R>` triggers `TS2322`/`TS2352` because `vi.fn()` infers
`Mock<Interface[K]>`. Pass the whole method's function type:

```ts
// BEFORE (jest)
let move: jest.Mock<void, [WidgetId, Point]>
// AFTER (vitest)
let move: Mock<IWidgetsAPI['move']>
```

### `Mocked<T>` does not deep-mock nested members — wrap call sites with `vi.mocked`

`jest.Mocked<Engine>` typed nested members (`engine.context.getFoo`) as mocks; Vitest's `Mocked<T>` maps only top-level
methods, so `.mockReturnValue` on a nested member fails (`Property 'mockReturnValue' does not exist`). Keep the
annotation, wrap the call site:

```ts
vi.mocked(mockEngine.context.getFormulaDefinition).mockReturnValue(formula)
```

Similarly, `jest.MockedObjectDeep<T>` / `jest.MockedFunctionDeep<T>` → use `vi.mocked(obj, true)` (deep) instead of
casting.

### `.mockImplementation()` / `.mockReturnValue()` with no args on a typed-return mock → `TS2554`

Beyond the runtime silencer issue (§2), the no-arg call fails typechecking when the signature has a non-void return.
Pass an explicit value:

```ts
vi.spyOn(console, 'info').mockReturnValue(undefined)
createObject: vi.fn().mockImplementation(() => undefined),
```

### `.mockImplementation(maybeUndefined)` → `TS2345`

Passing a possibly-`undefined` callback (an optional prop) fails where Jest's typing accepted it. Coalesce:
`handler.mockImplementation(onSubmit ?? (() => {}))`.

### `vi.fn` passed **uncalled** as a callback → `TS2322`

`{reset: vi.fn}` (the factory itself, not a mock instance) was assignable to `(x) => void` under Jest's loose types;
Vitest's generic `vi.fn` is not. The intent was almost always a mock instance — add parens: `{reset: vi.fn()}`.

### Custom matcher types → augment Vitest's `Assertion` via module augmentation

Hand-rolled interfaces extending `jest.Matchers<void>` / `jest.InverseAsymmetricMatchers` with a
`jest.Expect & CustomMatcher` cast: don't keep the `jest.*` namespaces and don't degrade to bare `as unknown as` casts.
Use module augmentation so `expect(x)`, `.not`, and `expect.extend` are typed:

```ts
interface CustomMatchers<R = unknown> {
	toBeFoo: () => R
}

declare module 'vitest' {
	// `Assertion` is what `expect()` returns; augmentation must keep the identical signature.
	// eslint-disable-next-line @typescript-eslint/no-explicit-any -- must match Vitest's `Assertion<T = any>`
	interface Assertion<T = any> extends CustomMatchers<T> {}
}
```

`.not` works for free. Add `interface AsymmetricMatchersContaining extends CustomMatchers {}` in the same block only if
the matcher is used asymmetrically (`expect.toBeFoo()`).

---

## 6. Assertions and snapshots

### `toHaveBeenCalledWith(new Error(...))` also compares `Error.name`

Jest's argument equality for `Error` objects effectively compared only `message`; Vitest compares `name` too. If source
code mutates `error.name` (or throws a subclass), the assertion fails despite matching messages. Assert on the captured
argument:

```ts
expect(errorSpy.mock.calls[0][0].message).toBe('Failed')
```

Same technique when the asserted argument is a live array/object the source mutates after the call —
`toHaveBeenCalledWith(liveRef)` compares final state, `spy.mock.calls[0][i]` snapshots identity.

### `Error`-subclass `toEqual` / `rejects.toEqual` phantom diff (Zod especially)

Vitest's `toEqual` compares `Error` instances by their own properties — including `stack` and instance-bound methods.
Two `ZodError`s with byte-identical `.issues` fail with _"Compared values have no visual difference"_. Assert on the
meaningful payload:

```ts
// BEFORE — passes on jest, fails on vitest
await expect(parse(bad)).rejects.toEqual(new ZodError([...]))
// AFTER
await expect(parse(bad)).rejects.toMatchObject({issues: expectedError.issues})
```

Also replace catch-and-assert blocks (`const err = await parse(bad).catch(e => e)`) with the `rejects.toMatchObject`
form.

### Snapshot key format change

Vitest snapshot keys use `>` between describe/test names; Jest used a space. A run writes the new key and leaves the old
one as "obsolete". Run once with `-u` to prune, and verify old and new snapshot contents match — a divergence means
behavior changed, not just the key.

### `toMatchInlineSnapshot(dynamicValue)` inside `it.each` → convert to `toBe`/`toEqual`

An inline snapshot compiles to a single call site; every `it.each` iteration hits the same one, and Vitest
cross-contaminates the iterations (one row fails with another row's expected/received). When the "snapshot" argument is
really an expected string from the table, assert directly and strip the snapshot-format quotes:

```ts
// BEFORE
it.each([{radius: 0, svg: '"<rect ... />"'}])('...', ({radius, svg}) => {
	expect(getSVGMask(rect, radius)).toMatchInlineSnapshot(svg)
})
// AFTER
it.each([{radius: 0, svg: '<rect ... />'}])('...', ({radius, svg}) => {
	expect(getSVGMask(rect, radius)).toBe(svg)
})
```

### Empty `describe` blocks are an error

`No test found in suite` — usually bare `expect(...)` calls sitting directly in the `describe` body (a latent bug: under
Jest they ran at collection time and never failed anything). Wrap them in a real `it(...)`; delete genuinely empty or
fully-commented describes. Beware: local runs with `passWithNoTests: true` hide this while raw CI runs fail — see
[ci.md](ci.md).
