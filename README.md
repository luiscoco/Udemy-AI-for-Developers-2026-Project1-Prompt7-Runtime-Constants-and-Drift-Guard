# Runtime Constants and Drift Guard

This step adds two safety nets around the generated OpenAPI contract in
`packages/contract`:

1. **Runtime constant arrays** (`STATES`, `ACTIONS`, `PRIORITIES`) so the
   frontend/backend can iterate over the valid enum values at runtime, not
   just reference them as compile-time types.
2. **A drift guard script** that fails CI/local checks if `openapi.yaml`
   is ever edited without regenerating `src/types.gen.ts`.

Below are the exact steps followed to build this, in order, so you can
reproduce or extend the pattern yourself.

## 1. Understand what already existed

Before changing anything, the existing files were read:

- `packages/contract/openapi.yaml` — the source of truth for the API shape.
- `packages/contract/src/types.gen.ts` — types generated from the YAML via
  `openapi-typescript` (see the `gen` script in
  `packages/contract/package.json`). This file is committed to the repo,
  **not** generated on every build, so it can silently go stale.
- `packages/contract/src/index.ts` — the package's public entry point,
  which only re-exported *types* (`WorkOrderState`, `WorkOrderAction`,
  `Priority`, ...) derived from the generated file. There was no runtime
  value you could actually loop over (e.g. to render a `<select>` of every
  priority) — TypeScript types disappear at runtime.

## 2. Add runtime constant arrays that can't drift silently

In `packages/contract/src/index.ts`, three `const` arrays were added:

```ts
export const STATES = [
  "reported", "triaged", "scheduled", "in_progress", "completed", "cancelled",
] as const satisfies readonly WorkOrderState[];

export const ACTIONS = [
  "triage", "schedule", "start", "complete", "cancel",
] as const satisfies readonly WorkOrderAction[];

export const PRIORITIES = [
  "low", "medium", "high", "critical",
] as const satisfies readonly Priority[];
```

`as const` gives each array a literal tuple type (not `string[]`), and
`satisfies readonly T[]` asks the compiler to check every element belongs
to the generated union type — so a typo or an extra value that isn't part
of the OpenAPI enum is caught immediately.

### Why `satisfies` alone isn't enough

`satisfies` only checks that the array's members are a **subset** of the
union type. It does **not** fail if the array is *missing* a member that
the union type has (e.g. the OpenAPI spec grows a new `WorkOrderState` but
nobody updates `STATES`). To close that gap, a small compile-time equality
check was added right after the arrays:

```ts
type AssertSame<A, B> = [A] extends [B] ? ([B] extends [A] ? true : never) : never;

type _StatesCheck = AssertSame<(typeof STATES)[number], WorkOrderState>;
type _ActionsCheck = AssertSame<(typeof ACTIONS)[number], WorkOrderAction>;
type _PrioritiesCheck = AssertSame<(typeof PRIORITIES)[number], Priority>;

const _statesCheck: _StatesCheck = true;
const _actionsCheck: _ActionsCheck = true;
const _prioritiesCheck: _PrioritiesCheck = true;
```

`AssertSame<A, B>` only resolves to `true` when `A` and `B` are mutually
assignable (i.e. the exact same set of literals in both directions). If
`STATES` and `WorkOrderState` ever diverge in **either** direction —
extra value or missing value — the `const _xCheck: _XCheck = true;` line
fails to compile with a clear `Type 'true' is not assignable to type
'never'` error, right where the mismatch is.

## 3. Write the drift guard script

Even with the compile-time check above, there's a second failure mode:
someone edits `openapi.yaml` (e.g. renames an enum value) but never reruns
`npm run gen`, so `types.gen.ts` — and therefore `STATES`/`ACTIONS`/
`PRIORITIES`, which were written by hand against the *old* types — quietly
fall out of sync with the spec that's supposed to be the source of truth.

`packages/contract/scripts/verify-contract.mjs` closes this gap:

1. Regenerates types from `openapi.yaml` into a **temporary directory**
   (via `os.tmpdir()`), using the exact same `openapi-typescript` CLI the
   `gen` script uses — invoked directly through `node` against its
   resolved CLI entry point, rather than the `.bin` shim, to avoid a
   Windows-specific `spawnSync EINVAL` error with `.cmd` shims.
2. Reads that freshly generated file and the committed
   `src/types.gen.ts` and compares them byte-for-byte.
3. If they differ, it prints a clear message telling you to run
   `npm run gen -w @equipment-hub/contract` and commit the result, then
   exits with a non-zero status code.
4. Cleans up the temp directory in a `finally` block either way.

## 4. Wire up the npm script

A root-level script was added to `package.json`:

```json
"scripts": {
  "verify:contract": "node packages/contract/scripts/verify-contract.mjs"
}
```

Run it with:

```sh
npm run verify:contract
```

This is the command CI (or a pre-commit hook) should run to catch contract
drift before it reaches `main`.

## 5. Prove it actually catches drift

To demonstrate the guard works, the following was done and then reverted:

1. Changed `packages/contract/openapi.yaml`'s `Priority` enum from
   `critical` to `urgent`, **without** running `npm run gen`.
2. Ran `npm run verify:contract` → it failed with exit code 1 and printed
   the "Contract drift detected" message, because the freshly generated
   types (with `urgent`) didn't match the committed `types.gen.ts` (which
   still had `critical`).
3. Then ran `npm run gen` to regenerate `types.gen.ts` for real (now
   `Priority` includes `urgent` instead of `critical`), but deliberately
   left `PRIORITIES` in `index.ts` unchanged. Running `tsc --noEmit` then
   failed in two places, showing the two compile-time guards work
   independently of the drift-guard script:
   - `Type '"critical"' is not assignable to type '"low" | "medium" |
     "high" | "urgent"'` (the `satisfies` clause on `PRIORITIES`).
   - `Type 'true' is not assignable to type 'never'` (the
     `AssertSame` check).
4. Reverted `openapi.yaml` back to `critical` and reran `npm run gen` to
   restore the committed `types.gen.ts` to its original, in-sync state.

## Running things in Windows Terminal

> **Note for students:** `apps/backend` and `apps/frontend` are currently
> empty stubs (just a `package.json` and `tsconfig.json`, no server or
> Vite source yet) — there is no `npm run dev` app to launch at this
> point in the course. The commands below are what you can actually run
> today: install dependencies and exercise the contract package's checks
> from this prompt.

Open **Windows Terminal**, `cd` into the project root (the folder
containing this `README.md`), then run:

```powershell
# 1. Install all workspace dependencies (only needed once, or after pulling changes)
npm install

# 2. Run the drift guard — fails if openapi.yaml and types.gen.ts are out of sync
npm run verify:contract

# 3. Type-check the contract package (includes the STATES/ACTIONS/PRIORITIES checks)
npx tsc --noEmit -p packages/contract/tsconfig.json

# 4. Lint the whole repo
npm run lint
```

If you ever edit `packages/contract/openapi.yaml`, regenerate the types
before committing:

```powershell
npm run gen -w @equipment-hub/contract
```

Once `apps/backend`/`apps/frontend` gain real source code and a `dev`
script (Fastify server / Vite dev server), `npm run dev` from the root
will become the actual "run the app" command — this section should be
updated then.

## Takeaways

- **Types alone aren't enough** when you need runtime values (arrays to
  iterate, sets to validate against) — but keeping a hand-written runtime
  array in sync with a generated type needs its own safety net, which is
  what the `satisfies` + `AssertSame` combo provides.
- **Generated files that are committed to the repo** (like
  `types.gen.ts`) are a common source of silent drift, because nothing
  forces someone to regenerate them after editing the source spec. A
  small script that regenerates into a temp location and diffs against
  the committed file is a cheap, dependency-free way to guard against
  that — it belongs in CI right next to `lint`, `test`, and `build`.
