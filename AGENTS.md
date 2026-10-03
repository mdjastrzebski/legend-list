# Repository Guidelines

## Guidance Source
`AGENTS.md` is the repo-local source of truth for agent guidance. Do not add parallel tool-specific guidance files such as `CLAUDE.md`; keep shared instructions here so they stay current for every coding agent.

## Plans
When I ask you to execute a plan file, after completing each step: check it off in the plan and immediately make a commit for that step. The first commit should add the plan, and each additional commit should check off one step.

## Project Structure & Module Organization
Legend List is a TypeScript React Native and React DOM list library. Core source lives in `src/`, split into feature folders such as `components` (public list primitives and rendering containers), `core` (scroll orchestration, layout, and viewability logic), `state` (shared state and context machinery), `entrypoints` (shared runtime behind `src/react.ts` and `src/react-native.ts`), `hooks`, `utils`, `platform`, `integrations`, and `section-list`. Shared base types live in `src/types.base.ts`, with public entrypoint wrappers in `src/types.web.ts` and `src/types.react-native.ts`; internal-only types such as `InternalState` live in `src/types.internal.ts`. Unit and regression tests mirror this layout in `__tests__/`. `example/` contains the Expo React Native showcase app (new architecture only; `example-oldarch/` covers the old architecture), and `example-web/` houses the Vite playground used for web fixtures and comparisons. Built artifacts land in `dist/` after `bun run build` and should not be edited manually.

## Build, Test, and Development Commands
Install dependencies with `bun install`. Use `bun run build` to generate production bundles: it runs lint plus the `prebuild.ts` source guards, then `tsup`, then the `postbuild.ts` bundle guards and `dist/package.json` generation. Run `bun test` for the full suite, `bun test --watch` during local iteration, and `bun test --coverage` before publishing. `bun run lint` executes Biome checks across `src`, `__tests__`, `example/app`, `example/screens`, and `example-web`. Type-level regressions are caught with `bun run tsc`; use `bun run tsc:src`, `bun run tsc:example`, or `bun run tsc:example-web` for narrower checks. `bun run tsc:go` runs the same source typecheck with `tsgo` (the `@typescript/native-preview` Go port of TypeScript).

Example app commands live in their own packages. For native scenarios, use `cd example && bun run ios`, `bun run android`, or the `*:fixtures` variants. For web scenarios, use `cd example-web && bun run dev` or `bun run dev:fixtures`.

## Architecture Notes
`LegendList` wraps the public component in a state provider and coordinates virtualization, scroll state, anchor tracking, and item positioning. The container system in `src/components/Container*.tsx` always reuses container slots (allocation lives in `src/core/calculateItemsInView.ts` and `src/utils/findAvailableContainers.ts`); `recycleItems` only decides whether the item component stays mounted across items instead of remounting by key. When it is enabled, treat local state inside recycled item components carefully because the same mounted component can represent different items over time.

Dynamic item sizing is estimate-first, then measurement-corrected. Prefer `estimatedItemSize` for initial size hints, or `getFixedItemSize` when item sizes are known exactly. Size changes feed back into total size, item positions, viewability, and maintain-visible-content-position behavior, so layout changes should include tests around scroll offsets and anchor stability.

The scroll adjustment paths are intentionally sensitive. Changes touching `bootstrapInitialScroll`, `maintainScrollAtEnd`, `maintainVisibleContentPosition`, imperative scroll completion (`scrollTo`, `checkFinishedScroll`, `doScrollTo`), item measurement, container allocation, or viewability should include focused regression tests and, when useful, an `example-web` fixture that exercises the visible behavior.

## Coding Style & Naming Conventions
We rely on Biome for formatting; always run the lint script before pushing to ensure consistent 4-space indentation, trailing commas, and sorted imports. Components and classes use PascalCase, hooks use the `useFeature` camelCase pattern, and helper functions stay camelCase. Constants defined in `constants.ts` and similar modules use SCREAMING_SNAKE_CASE. Prefer explicit exports through the entrypoint barrels (`src/react.ts`, `src/react-native.ts`, over `src/entrypoints/shared.ts`; there is no root `@legendapp/list` entry) to keep the public surface predictable.

## Lint / Formatting
- For any task that changes code, do not call the work done until formatting and lint autofix has been handled.
- Prefer writing changes in Biome-conforming style from the start, but still run `bun run lint:fix` before declaring completion when it is safe to do so.
- If repo-wide autofix would create unrelated churn in a dirty worktree, limit fixes to the touched files when possible; otherwise explicitly report that lint autofix was not run and why.
- If unused-import cleanup requires it, use `bun run lint:fix-unsafe` deliberately rather than leaving that cleanup for the user.

## Testing Guidelines
Unit tests live alongside features inside `__tests__` and should mirror file names from `src` with a `.test.ts` or `.test.tsx` suffix. Use the helpers in `__tests__/helpers` (`testRenderer.ts`, `testingLibrary.ts`) for rendering scenarios and `__tests__/__mocks__/createMockContext.ts` for core unit tests; avoid snapshot churn by relying on semantic assertions. New behavior must include coverage. (`bun run test:visualize` only regenerates the pass/coverage summary in `test-visualization.html`; it contains no scroll-offset data.)

Prefer focused tests near the changed behavior before broad suite runs. Core logic changes usually belong under `__tests__/core`, component/rendering behavior under `__tests__/components`, and small shared helpers under a matching utility test. For web-only behavior, validate through `example-web` or web-specific tests rather than assuming native behavior covers it.

## Commit & Pull Request Guidelines
Follow Conventional Commits (`fix:`, `feat:`, `chore:`) as seen in recent history. Commit titles should be imperative, concrete, and descriptive enough to explain the changed behavior or artifact; avoid overly abstract titles even if they are short. Prefer detailed commit messages for behavioral changes: use a concise Conventional Commit subject, then add a body explaining what changed, why it changed, and the key invariant or regression covered. One-line commits are fine only for trivial mechanical changes. Commit messages must use `type: subject` format with no scope parentheses; use `fix: handle null state`, not `fix(core): handle null state`. Do not add `Co-authored-by: Codex <noreply@openai.com>` to commits. Every PR should link to the relevant issue, describe the user-facing impact, and include reproduction steps or screenshots when UI is affected. Before requesting review of behavioral or release changes, confirm `bun run lint`, `bun test`, and `bun run build` have passed against the current change; reuse unchanged results. For documentation or mechanical-only changes, use checks relevant to the affected files, and update the changelog with `bun run prep-changelog` when shipping user-visible changes.

## Task Completion
After completing the authorized work, run relevant checks, fix failures caused by the change, and report remaining gaps. For plan execution, use the commit cadence above and check the plan again after committing; continue any newly added tasks. The question-triggered pause below still applies.

## Communication Guidelines
- If the user asks any questions, do not make any changes, only answer the questions.

## Agent docs

- [docs/agents/index.md](docs/agents/index.md): read before exploring the code. Architecture, area map, where to look, usage and extension, contributing. Generated from an earlier commit; verify details against the code.
- [docs/agents/project-health.md](docs/agents/project-health.md): maintenance activity, backing, license, alternatives, contribution norms. Snapshot from 2026-10-03; rerun index-github for current numbers.
