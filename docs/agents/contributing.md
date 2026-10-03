<!-- index-codebase: generated from 3d80d167 on 2026-10-03. Code wins when it disagrees with this doc. -->
# Legend List: Contributing

Maintainer policy (commit format, plan execution cadence, lint-before-done, question-only turns) lives in [`AGENTS.md`](../../AGENTS.md). This file covers the mechanics behind it.

## Setup

- `bun install` at the repo root (Bun 1.3.x was used when this doc was written; tests, scripts, and release tooling are Bun-specific). A fresh checkout has no `node_modules`, so `bun test` fails with 130 load errors and `tsc` is "command not found" until you install.
- `bun install` may rewrite the `peerDependencies` block in `bun.lock`, because `package.json` lists `react-dom`/`react-native` only in `peerDependenciesMeta`. Don't commit that churn unless you mean to.
- The example apps have their own `package.json` and lockfile: `cd example && bun install`, `cd example-web && bun install`. They resolve `@legendapp/list/*` straight to `../src` (Metro `resolveRequest` in `example/metro.config.js`, Vite aliases in `example-web/vite.config.ts`), so source edits show up live without a build.

## Feedback loop

| To check… | Run | Notes |
|---|---|---|
| Whole unit/regression suite | `bun test` | About 7 s, 2200+ tests. Preloads `__tests__/setup.ts` (via `bunfig.toml`). |
| One file | `bun test __tests__/core/mvcp.test.ts` | |
| One test by name | `bun test -t "pattern"` | |
| Lint and format | `bun run lint` / `bun run lint:fix` | Biome, scoped to `src`, `__tests__`, `example/app`, `example/screens`, `example-web`. `lint:fix-unsafe` also removes unused imports. |
| Types (library only) | `bun run tsc:src` | `bun run tsc` also checks `example` and `example-web`, which need their own installs. `tsc:go` uses `tsgo` from `@typescript/native-preview` (the Go port of TypeScript), not a go-to-definition tool. |
| Build and bundle guards | `bun run build` | Runs `prebuild` (lint plus `prebuild.ts` source guards), then `tsup` and `postbuild.ts` (bundle guards plus writing `dist/package.json`). About 11 s. |
| Native behavior on device | `cd example && bun run ios:fixtures` (or `android:fixtures`) | New architecture only. Use `example-oldarch/` for old-architecture checks. |
| Web behavior in a browser | `cd example-web && bun run dev:fixtures` | Fixture routes live in `example-web/src/fixtures/`. `bun run dev` shows the curated example catalog. |
| Repeatable device QA gestures | `.argent/flows/*.yaml` | Scripted flows for the `argent` tool (MCP server in `.vscode/mcp.json`). They drive fixture screens in the native example app by test ID. |
| Test-result HTML report | `bun run test:visualize` | Runs the suite twice (plain and `--coverage`), regex-parses the counts, rewrites the **tracked** `test-visualization.html` (last committed 2025-07), then `open`s it (macOS only). It has no scroll-offset data, so it doesn't help debug offsets. |

There is no CI in this repo (`.github/` only has `FUNDING.yml`). Every check above runs locally, and `bun run publish` re-runs tests and build before anything ships.

### How the test harness fakes the platform

- **Native by default.** `__tests__/setup.ts` uses `mock.module` to point every `@/…` specifier that has a `.native` variant at that variant (the `nativeModuleOverrides` list), replaces `react-native` with `__tests__/__mocks__/react-native.ts`, and sets `global.nativeFabricUIManager` so `IsNewArchitecture` is true. **If you add a new `.native.ts(x)` file, add it to `nativeModuleOverrides`**, or native tests will silently exercise the web file.
- **Web tests** (`*.web.test.ts(x)`) use one of three patterns:
  - Import the plain file with a unique query suffix, e.g. `await import("../../src/core/doScrollTo?web-stale-no-op")`. That gives a fresh module the mock doesn't intercept, but its own `@/` imports still resolve native.
  - Flip `Platform.OS = "web"` on the mock and restore it in `afterEach`.
  - Run a `.cases` file in a subprocess.
- **Not overridden:** `@/core/doScrollTo` and `@/utils/isInMVCPActiveMode` are missing from `nativeModuleOverrides`. Default tests therefore run their web variants, and `doScrollTo.native.test.ts` imports the `.native` file explicitly.
- **Fresh-process cases.** A `Foo.test.ts` that spawns `bun test ./Foo.cases.tsx` (for example `LegendList.bootstrapInitialScroll.oldarch.*`, `requestAdjust.oldarch.*`) exists because `mock.module` can't be undone within a process. Use this pattern for old-architecture or module-isolation scenarios.
- **Old architecture.** `*.oldarch.cases.*` files mock `@/constants-platform` to `{ IsNewArchitecture: false }` inside the subprocess.
- **Isolation.** `mock.module` replacements can leak across files (`__tests__/integrations/reanimated.isolation.test.ts` re-runs suites in a subprocess to catch this). The global `afterEach` restores timers and rAF, re-registers base module mocks, and resets the `Platform`/`I18nManager` mocks. `__tests__/helpers/testIsolation.test.ts` guards that this works.
- **Building blocks.** `__tests__/__mocks__/createMockState.ts` and `createMockContext.ts` build a list's state for direct unit tests of `src/core` functions. `__tests__/helpers/testRenderer.ts` and `testingLibrary.ts` handle component rendering.

`TEST_PLAN.md` is an old coverage checklist and isn't kept current. Use it as background only.

## Rules

Enforced by tooling (all at build time; there are no git hooks):

- **Biome** (`biome.json`): 4-space indent, 120 columns, double quotes, sorted imports in groups (React and Expo first, then `@legendapp/**`, then `@/**`), sorted object keys and JSX attributes (`useSortedKeys`, `useSortedAttributes`), and `noShadow` as an error.
- **`prebuild.ts`** fails the build if:
  - any `src/**/*.tsx` lacks the exact line `import * as React from "react";`
  - any `console.log` appears in `src`
  - anything in `src` imports the bare `@legendapp/list` (use subpaths)
  - `src/integrations/*` uses local imports (`@/`, `./`, `../`); integrations may only import `@legendapp/list/...` subpaths
  - `src/types.web.ts` or `src/types.react-native.ts` does `export *` from `@/types.base`, re-exports `@/types.internal`, or exports a declaration that references `@/types.internal` types
- **`postbuild.ts`** fails if `dist/react.d.ts` mentions `react-native`, if a core bundle contains `react-native-reanimated` or `react-native-keyboard-controller`, or if an integration bundle inlines core code (state module, `LegendList`, `src/core/`, or a second `ContextState`). See [areas/integrations.md](areas/integrations.md) for why.

Written down but not enforced:

- Conventional Commits, `type: subject` with no scope. `package.json` has a `commitlint` config, but nothing installs or runs commitlint.
- Commit cadence for plan files and PR expectations: see `AGENTS.md`.
- Tests mirror `src` paths under `__tests__/`. Names are `*.test.ts` or `*.test.tsx`, plus `.web.`/`.native.` infixes and `.cases.` companions.
- Plans live in `plans/<n>-<topic>.md`. `issues/` holds GitHub issue snapshots used as working context for a fix.

No CLA or DCO, and no `CONTRIBUTING.md`.

## Release process

Releases are cut by a maintainer from a local machine:

1. Bump `version` in `package.json` by hand.
2. `bun run publish`, which runs in order:
   - `testsilently`: `bun test` with output hidden, so it only reports pass or fail.
   - `prep-changelog` (`prep-changelog.ts`): refuses `-beta`/`-next` versions. If `CHANGELOG.md` already has `## <version>`, it does nothing. Otherwise it runs the first agent CLI it finds on PATH (`codex exec --ask-for-approval never --sandbox workspace-write`, then `claude -p`) with a fixed prompt to write the entry, following `.cursor/rules/changelog.mdc`. It then asks for y/N in the terminal.
   - `build`: lint, source guards, tsup, bundle guards.
   - `cd dist && npm publish`. The published package is the flat `dist/` folder, with a `package.json` that `postbuild.ts` has stripped of `scripts`, `devDependencies`, and `engines`.
   - `create-release.ts`: `git add .`, commits `Version <x.y.z>`, `git push`, then `gh release create v<x.y.z> --latest` with the top `CHANGELOG.md` section as notes.
3. Prereleases: `bun run publish:next` and `publish:beta` test, build, and `npm publish --tag next|beta`. They don't touch the changelog or git.

Changelog style (`.cursor/rules/changelog.mdc` and the prompt in `prep-changelog.ts`): one user-facing sentence per change, prefixed `Feat:`/`Fix:`/`Perf:`/`Improvement:`, with refactors and chores left out and issue numbers kept.

Watch out: `create-release.ts` stages everything with `git add .`, so a dirty worktree ends up in the version commit.
