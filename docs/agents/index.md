<!-- index-codebase: generated from 3d80d167 on 2026-10-03. Code wins when it disagrees with this doc. -->
# Legend List: Agent Index

## What this is
`@legendapp/list` is a virtualized list component for React Native (old and new architecture) and React DOM. It is a FlatList/FlashList replacement written in pure TypeScript with no native code. It handles dynamically sized items, two-way infinite lists, and chat UIs without `inverted`. It ships as a library of subpath entrypoints (`/react`, `/react-native`, plus optional `/reanimated`, `/keyboard`, `/keyboard-legacy`, `/animated`, `/section-list`) built with tsup. Development uses Bun. User docs: README and https://www.legendapp.com/open-source/list.

## Architecture in brief
- **Two kinds of state per list.**
  - `StateProvider` (`src/state/state.tsx`) gives each list a `ctx` with a **signal store**: `set$`/`peek$`/`useArr$` on keys such as `containerPosition3`. Only small subscribers re-render.
  - `ctx` also holds a mutable **`InternalState`** (`ctx.state`, typed in `src/types.internal.ts`) that every `src/core` function reads and writes.
  - `LegendListInner` itself subscribes to nothing.
- **Containers.** A pool of absolutely positioned slots (`Containers` → `ContainerSlot` → `Container` → `PositionView`). `calculateItemsInView` assigns items to slots, writes per-slot signals, and slots move or re-render independently.
- **Sizing is estimate-first.** Positions come from estimates (or `getFixedItemSize`), then one post-commit measurement pass corrects sizes. Corrections flow into positions, total size, MVCP, and end-following.
- **Scroll corrections** never call `scrollTo`. They go through `requestAdjust` → the `scrollAdjust` signal → `ScrollAdjust` (a native anchor-view trick, or DOM `scrollBy` on web).
- **Platform split by file suffix.** `X.native.ts(x)` vs `X.ts(x)`, with one runtime behind both entrypoints.

Primary flows:
1. **Scroll:** platform `onScroll` → `core/onScroll` → `updateScroll` → `calculateItemsInView` (window, slot allocation, `syncMountedContainer`) → `checkThresholds` (start/end reached).
2. **Measure:** `Container` commit → `scheduleContainerLayout` → `ContainerLayoutCoordinator` → `measureContainersInLayoutEffect` → `updateItemSizesBatch` → `calculateItemsInView({ doMVCP })` → `prepareMVCP` and `requestAdjust` → `doMaintainScrollAtEnd`.
3. **Mount or new dataset:** `LegendListInner` render (normalize props, detect data change by epoch) → layout effects → `initializeInitialScrollOnMount` / bootstrap convergence → `finishInitialScroll` → `readyToRender` (content becomes visible, `onReady`).

## Map
| Area | Responsibility | Paths | Doc |
|---|---|---|---|
| public-api | Entrypoints, public types, `LegendList` hub (props → state → effects), ref API, build guards | `src/react*.ts`, `src/entrypoints/`, `src/types.*.ts`, `src/components/LegendList.tsx`, `src/utils/createImperativeHandle.ts` | [areas/public-api.md](areas/public-api.md) |
| state | Signal store, `InternalState`, container context, public item hooks | `src/state/`, `src/hooks/useValue*$.ts` | [areas/state.md](areas/state.md) |
| virtualization | Sizes, positions, render window, container allocation and recycling, viewability | `src/core/calculateItemsInView.ts`, `updateItem*.ts`, `src/utils/findAvailableContainers.ts`, `src/core/viewability.ts` | [areas/virtualization.md](areas/virtualization.md) |
| scrolling | Scroll events, imperative `scrollTo*` lifecycle, reached-edge callbacks | `src/core/{onScroll,updateScroll,scrollTo*,doScrollTo*,finishScrollTo,checkFinishedScroll}.ts`, `src/utils/checkAt*.ts` | [areas/scrolling.md](areas/scrolling.md) |
| initial-scroll | First placement, bootstrap convergence, dataKey restarts, `readyToRender`/`onReady` | `src/core/{bootstrapInitialScroll,initialScroll*,finishInitialScroll}.ts` | [areas/initial-scroll.md](areas/initial-scroll.md) |
| scroll-stability | MVCP, `maintainScrollAtEnd`, `alignItemsAtEnd`, `anchoredEndSpace`, the adjust channel | `src/core/mvcp.ts`, `src/utils/requestAdjust.ts`, `src/components/ScrollAdjust*.tsx`, `src/core/doMaintainScrollAtEnd.ts` | [areas/scroll-stability.md](areas/scroll-stability.md) |
| rendering-and-platform | React tree, measurement hooks, web/native shims, DOM order, sticky, snap, RTL | `src/components/`, `src/platform/`, `src/hooks/` | [areas/rendering-and-platform.md](areas/rendering-and-platform.md) |
| integrations | Reanimated, keyboard, Animated, SectionList subpaths and their build boundary | `src/integrations/`, `src/section-list/` | [areas/integrations.md](areas/integrations.md) |

Outside `src/`:
- `__tests__/` mirrors `src`.
- `example/` (Expo, new arch), `example-oldarch/` and `example-web/` (Vite) run against `../src` live. They share data from `examples-shared/`.
- `plans/` and `issues/` are working notes.
- `.argent/flows/` holds device QA scripts.

## Where to look
| Task or question | Start at |
|---|---|
| Add or change a prop, or its default | `LegendListInner` in `src/components/LegendList.tsx`, `src/types.base.ts` → [public-api](areas/public-api.md) |
| Add a ref method | `src/utils/createImperativeHandle.ts`, `LegendListRef` in `src/types.base.ts` |
| Items render too late, too early, or blank while scrolling | `getEffectiveDrawDistance`, buffer logic in `calculateItemsInView` → [virtualization](areas/virtualization.md) |
| Wrong item shown in a recycled row / stale state | `findAvailableContainers`, `containerItemMetadata`, `useRecyclingState` → [virtualization](areas/virtualization.md), [state](areas/state.md) |
| Item sizes, estimates, grid layout | `getItemSize`, `updateItemSizes.ts`, `updateItemPositions.ts` |
| Content jumps when items resize or data is prepended | `prepareMVCP` in `src/core/mvcp.ts`, `requestAdjust` → [scroll-stability](areas/scroll-stability.md) |
| Chat list doesn't stay at the bottom | `doMaintainScrollAtEnd.ts`, `checkAtBottom.ts`, `alignItemsAtEnd` → [scroll-stability](areas/scroll-stability.md) |
| `scrollToIndex` lands wrong or its promise never resolves | `scrollTo.ts`, `checkFinishedScroll.ts`, `doScrollTo*.ts` → [scrolling](areas/scrolling.md) |
| `onEndReached` fires twice or never | `checkThreshold.ts`, `edgeReachedGate.ts` → [scrolling](areas/scrolling.md) |
| `initialScrollIndex` lands wrong, list invisible, `onReady` timing | `bootstrapInitialScroll.ts`, `setInitialRenderState.ts` → [initial-scroll](areas/initial-scroll.md) |
| Web-only bug (DOM scroll, ResizeObserver, focus order) | `ListComponentScrollView.tsx`, `createResizeObserver.ts`, `useDOMOrder.ts` → [rendering-and-platform](areas/rendering-and-platform.md) |
| Sticky headers | `PositionView(.native).tsx`, `stickyPositionUtils.ts`, `activeStickyIndex` in `calculateItemsInView` |
| Keyboard / Reanimated / SectionList behavior | `src/integrations/*.tsx`, `src/section-list/` → [integrations](areas/integrations.md) |
| Add a platform shim | `src/platform/X.ts` + `X.native.ts`, then `nativeModuleOverrides` in `__tests__/setup.ts` |
| Build fails in prebuild/postbuild | `prebuild.ts`, `postbuild.ts` → [contributing](contributing.md#rules) |
| Why was it designed this way? | [decisions.md](decisions.md) |

## Dev loop
```bash
bun install                       # required first; tests fail to load without it
bun test                          # ~7 s, 2200+ tests; bun test <file> or -t "<name>" for one
bun run lint:fix                  # Biome (format, sorted imports/keys)
bun run tsc:src                   # library typecheck
bun run build                     # lint + source guards + tsup + bundle guards
cd example-web && bun run dev:fixtures   # visual web fixtures (native: cd example && bun run ios:fixtures)
```
There is no CI, so run these locally. Full details: [contributing.md](contributing.md).

## Conventions and traps
- **Tests default to native.** `__tests__/setup.ts` remaps `@/x` to `x.native` through `nativeModuleOverrides`. Add every new `.native` file there. Web tests import with a `?web-…` query suffix or flip `Platform.OS`. Old-arch and module-isolated tests run `*.cases.ts(x)` in a subprocess.
- **Build guards** (`prebuild.ts`, `postbuild.ts`):
  - every `src/**/*.tsx` needs `import * as React from "react";`
  - no `console.log` in `src`
  - no bare `@legendapp/list` imports
  - integrations import core only through `@legendapp/list/*` subpaths
  - platform type files don't `export *` or leak `types.internal`
- **`set$` is synchronous and compares by reference.** Write `ctx.state` data that subscribers read *before* the `set$`, and never mutate a value in place and set it again.
- **Never assign `state.scroll`, `scrollingTo`, `didFinishInitialScroll` or `sizes` directly.** Go through `requestAdjust`, `finishScrollTo`/`cancelImperativeScroll`, `finishInitialScroll`, `setSize`/`updateOneItemSize`.
- **Async callbacks per container must re-check ownership:** `peek$(containerItemKey${id})` and `state.containerItemGenerations`.
- **Untyped `*Internal` props** link core and the integrations. Grep both sides when renaming.
- **Generated or tracked artifacts:** `dist/` (build output, gitignored), `test-visualization.html` (rewritten by `test:visualize`), `CHANGELOG.md` entries (drafted by `prep-changelog`). Don't edit `dist/`.
- **Changing a sensitive path needs both a regression test and a fixture.** The sensitive paths are initial scroll, MVCP, maintain-at-end, measurement, allocation, viewability, and imperative-scroll completion. Add a focused regression test plus, when visible, an `example-web` or `example` fixture (`AGENTS.md`).

## Glossary
| Term | Meaning | Defined in |
|---|---|---|
| container / slot | A recycled, absolutely positioned render slot with a numeric id. Not an item. | `src/components/Container.tsx` |
| signal, `$` suffix | A key in `ctx.values`; `$` functions operate on the store (named after Legend State) | `src/state/state.tsx` |
| `ctx` / `state` | Per-list context / its mutable `InternalState` | `src/state/state.tsx`, `src/types.internal.ts` |
| `numContainers` vs `numContainersPooled` | Slots in use vs slots rendered (empty ones render null) | [virtualization](areas/virtualization.md) |
| `sizes` vs `sizesKnown` | Size counted in `totalSize` (maybe estimated) vs measured or fixed | `src/utils/getItemSize.ts` |
| buffered vs no-buffer range | Render window vs visible window | `calculateItemsInView.ts` |
| MVCP | maintainVisibleContentPosition: `size` (default on) and `data` (opt-in) anchoring | `src/core/mvcp.ts` |
| scroll adjust | Relative scroll correction delivered through the `scrollAdjust` signal | `src/utils/requestAdjust.ts` |
| `scrollingTo` | The active imperative or initial scroll target; its identity is the completion token | `src/types.internal.ts` |
| bootstrap session | Hidden convergence loop for index or end initial targets | `src/core/bootstrapInitialScroll.ts` |
| `readyToRender` | `didContainersLayout && didFinishInitialScroll`; gates content opacity and `onReady` | `src/utils/setInitialRenderState.ts` |
| `dataKey` / `dataVersion` | New dataset (restarts initial scroll) / same dataset mutated | `src/components/LegendList.tsx` |
| epoch | Monotonic counter (`dataChangeEpoch`, `freshDataTransitionEpoch`) that survives render retries | `src/types.internal.ts` |
| old arch / new arch | RN Paper vs Fabric (`IsNewArchitecture`); web counts as new arch | `src/constants-platform*.ts` |
| fixture | Internal validation screen in the example apps (`*:fixtures` scripts) | `example-web/src/fixtures/`, `example/screens/fixtures/` |

## Further reading
- [`AGENTS.md`](../../AGENTS.md): maintainer policy for agents (commits, plans, lint, question-only turns).
- [using.md](using.md): platforms, config defaults, extension points, limitations, security.
- [contributing.md](contributing.md): setup, test harness, rules, release.
- [decisions.md](decisions.md): cross-cutting design decisions.
- [project-health.md](project-health.md): maintenance and project-level facts (from `index-github`).
- `.cursor/rules/changelog.mdc`: changelog style. `TEST_PLAN.md`: an older coverage checklist that isn't kept current.
- README and https://www.legendapp.com/open-source/list for the public API.
