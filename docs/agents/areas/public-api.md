<!-- index-codebase: generated from 3d80d167 on 2026-10-03. Code wins when it disagrees with this doc. -->
# Public API: LegendList component, props, types and entrypoints

**Purpose:** the package surface (subpath entrypoints, public types, ref handle, exported hooks) and the `LegendList` hub component that turns props into `InternalState` and wires React lifecycle to core functions. It does not implement virtualization, scrolling math or rendering; it calls into those areas.
**Paths:** `src/react.ts`, `src/react-native.ts`, `src/entrypoints/shared.ts`, `src/types.base.ts`, `src/types.web.ts`, `src/types.react-native.ts`, `src/types.internal.ts`, `src/typesInternal.ts`, `src/constants.ts`, `src/constants-platform*.ts`, `src/components/LegendList.tsx`, `src/components/useDevChecks.ts`, `src/utils/createImperativeHandle.ts`, `src/utils/normalize*.ts`, `tsup.config.ts`, `prebuild.ts`, `postbuild.ts`, `package.json` (`exports`)
**Depends on:** [state](state.md), [virtualization](virtualization.md), [scrolling](scrolling.md), [initial-scroll](initial-scroll.md), [scroll-stability](scroll-stability.md), [rendering-and-platform](rendering-and-platform.md) · **Used by:** [integrations](integrations.md) (and `src/section-list`), apps

## Which import path ships what?

There is no root `@legendapp/list` entry. It was removed in c34fcb89 ("feat: remove root package entrypoint", which also deleted `src/index.ts` and `src/types.root.ts`). Consumers import a subpath.

| Subpath | Source | tsup build | Notes |
|---|---|---|---|
| `@legendapp/list/react` | `src/react.ts` | web config (plain `.ts/.tsx` resolution) | DOM-typed props/ref from `types.web.ts` |
| `@legendapp/list/react-native` | `src/react-native.ts` (native) / `src/react.ts` (web) | native config emits `react-native.js`; web config emits `react-native.web.js` from `src/react.ts` | `package.json` `exports` picks `react-native.*` under the `react-native` condition and `react-native.web.*` for `browser` and default. Types always come from `react-native.d.ts` (RN types). This is how react-native-web gets the web runtime (6aba4b2e) |
| `/animated`, `/reanimated`, `/keyboard`, `/keyboard-legacy` | `src/integrations/*.tsx` | native config | See [integrations](integrations.md) |
| `/section-list` | `src/section-list/index.ts` | native config only | No web variant in `exports` |

Platform selection is purely by file extension. The native tsup config sets `esbuildOptions.resolveExtensions = [".native.tsx", ".native.ts", ".tsx", ".ts", ".json"]`, so `foo.native.ts` shadows `foo.ts` for every `@/...` import. The web config uses default resolution, so it only ever sees the non-`.native` files. Example: `IsNewArchitecture` is hard-coded `true` in `src/constants-platform.ts` (web has synchronous layout, like Fabric) and detected from `global.nativeFabricUIManager` in `src/constants-platform.native.ts`.

`.d.ts` files are produced by separate per-entry `dtsConfigs` (`dts: { only: true }`, `splitting: false`) so declaration files are not hash-split into shared chunks (dac0fd2d).

Both `src/react.ts` and `src/react-native.ts` are thin barrels over `src/entrypoints/shared.ts`. Each one:
1. re-exports `LegendListRuntime` cast to its platform's `LegendListComponent` type (same runtime component, different typing),
2. re-exports the item hooks from `src/state/ContextContainer.ts`: `useAdaptiveRender`, `useAdaptiveRenderChange`, `useIsLastItem`, `useListScrollSize`, `useRecyclingEffect`, `useRecyclingState`, `useSyncLayout`, `useViewability`, `useViewabilityAmount` (documented in [state](state.md)),
3. exports `internal`, a bag of core helpers (`peek$`, `useArr$`, `useStateContext`, `typedForwardRef`, `getComponent`, `POSITION_OUT_OF_VIEW`, ...) marked `/** @internal */`,
4. `export *` from its platform types file.

## How it works

### Type layering

```
types.base.ts        platform-agnostic: LegendListPropsBase<ItemT, TScrollViewProps, TItemType>, LegendListRef,
                     LegendListState, viewability types, config objects (MaintainScrollAtEndOptions, ...)
   |                 (imports LegendListListenerType/ListenerTypeValueMap from @/state/state)
   v
types.internal.ts    re-exports LegendListPropsBase; adds InternalState, *Normalized configs, ScrollTarget,
                     typedForwardRef / typedMemo
   |
   +--> types.web.ts           Omit<...> + DOM overrides (CSSProperties, HTMLAttributes, scrollElement, HTMLElement refs)
   +--> types.react-native.ts  Omit<...> + RN overrides (ScrollViewProps, StyleProp<ViewStyle>, ScrollView refs, Insets)
```

`LegendListPropsBase` = `BaseScrollViewProps<TScrollViewProps>` (the platform scroll props minus `contentOffset`, `maintainVisibleContentPosition`, `stickyHeaderIndices`, `removeClippedSubviews`, `children`, `onScroll`) & `LegendListSpecificProps` & (`DataModeProps` | `ChildrenModeProps`). The union makes `data`+`renderItem` and `children` mutually exclusive at the type level.

Each platform file builds `LegendListProps` by `Omit`-ing the platform-sensitive keys from `LegendListPropsBase` and re-adding them with concrete types, then does the same for `LegendListRef` (`getNativeScrollRef`, `getScrollResponder`, etc.) and `LegendListState.elementAtIndex`. Shared types are re-exported by an explicit named list, never `export *` (prebuild enforces this, see below).

`src/typesInternal.ts` (note: no dot) only holds `BaseSharedValue` / `StylesAsSharedValue`, used to type `animatedPropsInternal` coming from the reanimated integration.

### The LegendList hub (`src/components/LegendList.tsx`)

Two layers:

- **Outer `LegendList`** (`typedMemo(typedForwardRef(...))`): converts children mode to data mode (`children` flattened into `data`, `renderItem` returns the child, `childrenMode: true`), defaults `data` to `[]`, and wraps everything in `<StateProvider>`. One provider per list instance means one `StateContext` per list.
- **`LegendListInner`**: everything else, in this order on every render:
  1. Destructure props with defaults (`drawDistance = 250`, `estimatedItemSize = 100`, `recycleItems = false`, `maintainScrollAtEndThreshold = 0.1`, `onEndReachedThreshold = 0.5`, `numColumns = 1`, ...). Warns once via `warnDevOnce("recycleItems-omitted", ...)` if `recycleItems` is undefined (177ed578).
  2. Strip private integration props off `rest` via `as any`: `animatedPropsInternal`, `anchoredEndSpaceOwnerInternal`, `positionComponentInternal`, `stickyPositionComponentInternal`. Everything left in `restProps` is passed through to `ListComponent` and on to the scroll view.
  3. Normalize: `StyleSheet.flatten` style and contentContainerStyle, `extractPadding` for each side, `normalizeMaintainScrollAtEnd`, `normalizeMaintainVisibleContentPosition`, build `initialScrollProp` from `initialScrollAtEnd` / `initialScrollIndex` (number or object) / `initialScrollOffset` / horizontal RTL.
  4. Create `InternalState` once and store it on `ctx.state` (not only a ref) so StrictMode double-render reuses it (c839edd2). The same block late-binds `ctx.scrollToEnd`, `state.triggerCalculateItemsInView` and `state.reprocessCurrentScroll` to avoid import cycles.
  5. Detect data change against the previous `state.props` (see flow below).
  6. Overwrite `state.props` wholesale with the normalized snapshot. Callbacks such as `keyExtractor`, `getFixedItemSize` and `getItemType` are wrapped by `useWrapIfItem`, which returns `undefined` when called with an undefined item/index.
  7. On first render only: `initializeStateVars`, `resetLayoutCachesForDataChange`, `updateItemPositions(ctx, true)` synchronously, so positions exist before the first commit.
  8. Layout effects call core: `initializeInitialScrollOnMount`, `handleInitialScrollDataChange`, `maybeUpdateAnchoredEndSpace`, `updateContentInsetEndAdjustment`, `updateSnapToOffsets`, `initializeStateVars(true)`, container allocation/reset, `runPendingScrollToEnd`. Passive effects: `onMetricsChange` listener, `setupViewability`, unmount cleanup (`cancelImperativeScroll`, `state.scheduledWork.dispose()`, fcd1e730).
  9. Render `<ListComponent>` with stable handlers from a `useMemo(..., [])` `fns` object (`onScroll -> core/onScroll`, drag begin/end, momentum end, `getRenderedItem`). These read `state.props.*` at call time, so they never go stale.

### Traced flow: `data` changes

1. Parent re-renders `LegendList` with a new `data` array.
2. `LegendListInner` render: `didDataChangeLocal` is true if `dataKey` or `dataVersion` changed, or if the reference changed **and** `checkStructuralDataChange(state, dataProp, state.props.data)` says so. That function returns `false` when lengths match and each slot is either `===`, or has the same key and `itemsAreEqual` returns true. It caches per-index verdicts in `state.pendingDataComparison` for container re-render decisions.
3. On change: `state.dataChangeEpoch += 1`, `dataChangeNeedsScrollUpdate = true`, `didDataChange = true`, `previousData = state.props.data`. If `dataKey` changed (or data went from empty to non-empty after having had data) `freshDataTransitionEpoch += 1`. If there is no `keyExtractor`, `sizes`, `positions` and `totalSize` are cleared, because index keys give no size continuity.
4. `state.props = {...}` now holds the new data.
5. Layout effect keyed on the epochs consumes them via `state.handledDataChangeEpoch` / `handledFreshDataTransitionEpoch`. It calls `resetInitialRenderState` for fresh datasets, then `handleInitialScrollDataChange` ([initial-scroll](initial-scroll.md)).
6. Layout effect keyed on `[dataProp, dataKey, dataVersion, numColumnsProp, nextScrollAxisGap]` either does `doInitialAllocateContainers(ctx)` (first non-empty data) or `checkResetContainers(ctx, data, { didColumnsChange })`. That runs `calculateItemsInView(ctx, { dataChanged: true, doMVCP: true })` ([virtualization](virtualization.md), [scroll-stability](scroll-stability.md)), then `doMaintainScrollAtEnd` if `maintainScrollAtEnd.onDataChange`, else `checkThresholds`. Afterwards the flags `didDataChange`, `didColumnsChange`, `isFirst` and `pendingDataComparison` are reset.

### Ref API (`createImperativeHandle`)

`useImperativeHandle(forwardedRef, () => createImperativeHandle(ctx, scheduleImperativeScrollCommit), [])` builds the handle once. Every method reads `ctx.state` lazily.

- Scroll methods (`scrollToIndex`, `scrollToOffset`, `scrollToItem`, `scrollIndexIntoView`, `scrollItemIntoView`) return Promises tracked by `getScrollRequestTracker(ctx)`. A newer request supersedes an older one. Each call first calls `supersedeInitialScroll(ctx)`. If the list is still settling after a data change (`didDataChange`, `didColumnsChange`, pending `mvcpRecalculate` work, `ignoreScrollFromMVCP`) or the target is not ready (`anchoredEndSpace` sizes unknown), it polls on `state.scheduledWork.frame(..., "imperativeScrollReady")` until 2 stable frames or 800 ms.
- `scrollToEnd` stores `state.pendingScrollToEnd` and forces a re-render through a `useReducer` counter (`scheduleImperativeScrollCommit`). The unconditional `useLayoutEffect(() => state.runPendingScrollToEnd?.())` then runs it after the new props/data have committed (1a2bbb5c, "defer scrollToEnd until data commits").
- `getState()` returns a fresh snapshot. `listen(signal, cb)` exposes the subset of state signals in `LegendListListenerType` (defined in `src/state/state.tsx`, not in this area).
- Also exposes `clearCaches({ mode: "sizes" | "full" })`, `setItemSize`, `setScrollProcessingEnabled`, `setVisibleContentAnchorOffset`, and `reportContentInset` (RN typing narrows `inset` to `Insets`).

## Key types and entry points

| Symbol | File | Role |
|---|---|---|
| `LegendList` (outer) / `LegendListInner` | `src/components/LegendList.tsx` | Children-mode normalization + `StateProvider`; prop normalization, state init, effects |
| `LegendListRuntime`, `internal` | `src/entrypoints/shared.ts` | Single runtime component shared by both barrels; helper bridge for integrations |
| `LegendListComponent`, `LegendListProps`, `LegendListRef`, `LegendListState` | `src/types.web.ts`, `src/types.react-native.ts` | Platform-typed public surface |
| `LegendListPropsBase`, `BaseScrollViewProps` | `src/types.base.ts` | Shared prop contract (data/children union) |
| `InternalState` | `src/types.internal.ts` | Mutable per-list state; `state.props` is the normalized prop snapshot core reads |
| `MaintainScrollAtEndNormalized`, `MaintainVisibleContentPositionNormalized` | `src/types.internal.ts` | Normalized config shapes consumed by core |
| `normalizeMaintainScrollAtEnd`, `normalizeMaintainVisibleContentPosition` | `src/utils/normalize*.ts` | Prop shorthand to config object |
| `createImperativeHandle` | `src/utils/createImperativeHandle.ts` | Builds the `LegendListRef` object |
| `checkStructuralDataChange` | `src/core/checkStructuralDataChange.ts` | Decides whether a new `data` reference is a real change |
| `useDevChecks` | `src/components/useDevChecks.ts` | Dev-only warnings (anchoredEndSpace + numColumns, missing keyExtractor, web unbounded height, ...) |
| `typedForwardRef`, `typedMemo` | `src/types.internal.ts` | Casts of `React.forwardRef` / `React.memo` that keep the generic `ItemT` |

## Invariants and gotchas

- **Defaults that matter:** `maintainVisibleContentPosition` undefined normalizes to `{ data: false, size: true }`, so size-change anchoring is on by default; `true` gives `{ data: true, size: true }`. `maintainScrollAtEnd: true`, or an object without `on`, enables all four triggers (`onLayout`, `onItemLayout`, `onDataChange`, `onFooterLayout`). An object with `on` enables only the listed ones. `animated` defaults to false.
- **Data change detection is not reference-only.** A new array with the same items (`===`), or same keys plus `itemsAreEqual` true, is *not* a data change. To force one with a stable reference, bump `dataVersion`. Use `dataKey` to signal a different logical dataset; it resets layout readiness and initial scroll without a remount (0007ef77, f6692c65).
- **Render-phase mutation + epochs.** Change detection mutates `state` during render. React may discard and retry that render. So effects consume monotonic `dataChangeEpoch` / `freshDataTransitionEpoch` against `handled*Epoch` stored on shared state, instead of trusting local booleans (660f67a4, 3c0f5d9c). The container-allocation effect reads `didDataChange` / `didColumnsChange` / `isFirst` from `state`, not locals, for the same reason (see the comment there about react-dom double render).
- **Most prop changes do not trigger recomputation by themselves.** `state.props` is just replaced and core reads it on the next scroll/layout/data pass. Only props that appear in effect dependency lists cause immediate work: data/`dataKey`/`dataVersion`, `numColumns`, gap, padding, `snapToIndices`, `extraData`, `overrideItemLayout` with `numColumns > 1`, viewability config, `anchoredEndSpace.*`, `contentInsetEndAdjustment`, `experimental_adaptiveRender` / `experimental_hideItemsUntilMeasured` turned off. **Inferred:** changing e.g. `drawDistance` alone applies on the next scroll. If a new prop must take effect immediately, add an explicit effect.
- **Changing `numColumns` or the scroll-axis gap clears all size caches** (`checkResetContainers` with `didColumnsChange`, e6c3ae19).
- **No `keyExtractor` means data changes wipe `sizes` / `positions` / `totalSize`**, and dev warns via `useDevChecks`.
- **Web-only props are silently dropped on native:** `contentInsetEndAdjustment`, `scrollElement`, `useWindowScroll` are resolved through `Platform.OS === "web"` checks. `scrollElement` and `useWindowScroll` are also ignored when `renderScrollComponent` is set. `scrollElement` takes precedence over `useWindowScroll`.
- **Initial `scrollLength`** is `estimatedListSize`, else `{0,0}` on new arch/web, else window size on old arch. Old arch also allocates containers during render via `useInit`; new arch waits for the layout effect. `canRender` starts `!IsNewArchitecture`.
- **Internal props are untyped contracts.** `*Internal` props are read with `(props as any)` and are not in any public type. Integrations pass them by spreading. Renaming one breaks integrations silently, with no type error.
- **The handle is created once with `[]` deps.** Never capture props in `createImperativeHandle` closures. Read `state.props` at call time.
- **The public `listen` surface is defined in state.** Adding a signal to `LegendListListenerType` in `src/state/state.tsx` changes the public `LegendListState.listen` type, because `types.base.ts` imports it.
- **`IS_DEV` gates a hook call** (`if (IS_DEV) useDevChecks(props)`). This is safe only because `IS_DEV` is a module constant.
- **Tests run native files.** `__tests__/setup.ts` sets `global.nativeFabricUIManager` and maps many `@/...` modules to their `.native` files via `nativeModuleOverrides`. A new `.native.ts` split must be added there, or tests exercise the web file.

## Build-time guards (fail `bun run build`)

`prebuild` = `bun run lint && bun run prebuild.ts`. `prebuild.ts` throws if:

| Check | Why |
|---|---|
| A `.tsx` file in `src` lacks the exact line `import * as React from "react";` | Classic JSX runtime needs `React` in scope (984dbfba) |
| `console.log` anywhere in `src` | Stray debug logs shipped before (490f1a20) |
| Any `src` import of bare `@legendapp/list` | Root entry no longer exists; use subpaths |
| `src/integrations/*` imports a local path (`@/`, `./`, `../`, `src/`) | Integrations must go through `@legendapp/list/*` subpaths, so their bundles reuse the core instead of inlining a second copy with its own `StateContext` (d3956843) |
| `types.web.ts` / `types.react-native.ts` contains `export * from "@/types.base"` | Each public type is listed explicitly |
| Those files re-export from `@/types.internal`, or an exported type in them references an identifier imported from `@/types.internal` | Internal types must not appear in the public `.d.ts`. Note that `LegendListPropsBase` is imported from `@/types.internal` but only used inside a non-exported `LegendListPropsOverrides`, which is how the check is satisfied |

`postbuild.ts` throws if `dist/react.d.ts` mentions `react-native` (the web types must be RN-free, plan `plans/5-types-2.md`); if any core runtime bundle (`react*.js/.mjs`, `react-native*.js/.mjs`) mentions `react-native-reanimated` or `react-native-keyboard-controller`; or if integration bundles do not import `@legendapp/list/(animated|react-native|reanimated)` or contain inlined `state.tsx`, `LegendListInner`, `src/core/`, or a second `createContext(null)` (72542d28). It then copies README/LICENSE/CHANGELOG and writes a stripped `dist/package.json`.

## Design decisions

- **Subpath-only entrypoints with platform-split types** (plans `plans/4-types.md`, `plans/5-types-2.md`; 30ab16dc, c34fcb89). One runtime, two type views: RN users get `ScrollViewProps` / `StyleProp`, DOM users get `CSSProperties` / `HTMLElement` without pulling `react-native` types. The deprecated root entry and `types.root.ts` were deleted in c34fcb89.
- **`react-native` subpath resolves to the web build for browsers** (6aba4b2e) so react-native-web apps importing `@legendapp/list/react-native` get DOM-based rendering rather than the mobile build.
- **`internal` bridge object** (6aba4b2e): integrations and section-list need core helpers but must import through the external `@legendapp/list/react-native` specifier to share one module instance (enforced by prebuild/postbuild). `tsconfig.json` has `stripInternal: true`, so `internal` is absent from the built `dist/*.d.ts`; verified against a 3d80d167 build. It is a runtime-only contract.
- **State lives on context, created once** (c839edd2): fixes web StrictMode double-invoke and lets children read state.
- **Structural data comparison + `itemsAreEqual`** (62e9f6df, c5cfdf0d): avoid resetting containers when callers recreate arrays with unchanged rows.
- **`dataKey` vs `dataVersion`** (0007ef77): `dataVersion` means "same dataset, mutated in place"; `dataKey` means "a different dataset, treat like a fresh mount" (resets initial render/scroll without remounting).

## Where to change things

| To… | Start at |
|---|---|
| Add a cross-platform prop | Declare in `LegendListSpecificProps` (`src/types.base.ts`), destructure + add to `state.props` in `LegendListInner`, add field to `InternalState["props"]` (`src/types.internal.ts`), add an effect if it must apply immediately |
| Add a platform-specific prop or retype one | `Omit` it and re-add in `src/types.web.ts` and/or `src/types.react-native.ts` |
| Export a new public type | Add to `types.base.ts`, then to the explicit `export type {...}` list in **both** platform type files |
| Export a new hook | Implement in `src/state/ContextContainer.ts` (or elsewhere), add to `src/entrypoints/shared.ts` and to both `src/react.ts` and `src/react-native.ts` lists |
| Add a ref method | `LegendListRef` in `src/types.base.ts` (override in platform files if the return type is platform-specific) + `createImperativeHandle`; add a test in `__tests__/utils/createImperativeHandle.test.ts` |
| Change a prop default | Destructuring defaults in `LegendListInner`, or the `normalize*` helpers for config objects; update JSDoc `@default` in `types.base.ts` |
| Change what counts as a data change | `src/core/checkStructuralDataChange.ts` and the `didDataChangeLocal` block in `LegendListInner` |
| Expose a helper to integrations | `internal` in `src/entrypoints/shared.ts` (never import `@/...` from `src/integrations`) |
| Pass integration-only behavior into the list | A `*Internal` prop stripped in `LegendListInner` (see `positionComponentInternal`) |
| Add a new package subpath | `tsup.config.ts` entry (web vs native config), `package.json` `exports`, `tsconfig.json` `paths`, and the postbuild lists if it's an integration |
| Add a dev warning | `src/components/useDevChecks.ts` using `warnDevOnce` |

## Tests

- `__tests__/components/LegendList.props.test.tsx`: prop wiring, defaults, dev warnings, initial-scroll prop shapes, RefreshControl offset, unmount cleanup. It mocks `@/components/ListComponent` to capture props and `@/core/ScrollAdjustHandler` to reach `ctx.state`.
- `__tests__/components/LegendList.dataVersion.test.tsx`: `dataVersion`, `itemsAreEqual`, keyExtractor-less data change detection.
- `__tests__/components/LegendList.imperativeScroll.test.tsx`, `__tests__/utils/createImperativeHandle.test.ts`: ref API and promise settling.
- `__tests__/utils/normalizeMaintainScrollAtEnd.test.ts`. There is no dedicated test for `normalizeMaintainVisibleContentPosition`.
- Also relevant: `LegendList.numColumnsReset`, `LegendList.dataRefresh.integration`, `LegendList.liveUpdates.integration`, `LegendList.totalSizeNotifications`.
- No runtime test covers the entrypoint barrels or type surface. Those are guarded by `bun run tsc`, `prebuild.ts` and `postbuild.ts` (run via `bun run build`).
- Run: `bun test __tests__/components/LegendList.props.test.tsx __tests__/components/LegendList.dataVersion.test.tsx __tests__/utils/createImperativeHandle.test.ts __tests__/utils/normalizeMaintainScrollAtEnd.test.ts` (99 pass at 3d80d167).
