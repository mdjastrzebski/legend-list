<!-- index-codebase: generated from 3d80d167 on 2026-10-03. Code wins when it disagrees with this doc. -->
# Rendering and platform

**Purpose:** the React tree between `LegendList` and the user's `renderItem` output (scroll view, container layer, recycled containers, positioning, separators, sticky headers, snapping, debug overlays), plus the web/native split: `.native.ts(x)` shims, DOM-specific scroll and measurement code, and how measured sizes get back to core. It does **not** decide which items render or where they sit. Positions, container allocation and sizes live in core (see [virtualization](virtualization.md)), scroll handling in [scrolling](scrolling.md), and MVCP/ScrollAdjust in [scroll-stability](scroll-stability.md).
**Paths:** `src/components/*` except `LegendList.tsx` and `ScrollAdjust*`; `src/platform/*`; `src/constants-platform(.native).ts`; `src/hooks/{useOnLayoutSync,useContainerMeasurement,createResizeObserver,useDOMOrder,useStableRenderComponent,useCombinedRef,useLatestRef,useThrottleDebounce}`; `src/utils/{reordering,rtl,getComponent,createColumnWrapperStyle,devEnvironment}.ts(x)`.
**Depends on:** [state](state.md) (`useArr$`, `set$`, `peek$`, `listen$`, per-container signals), core measurement entry points (`src/core/updateItemSizes.ts`, `src/core/scheduleContainerLayout.ts`, `src/core/measureContainersInLayoutEffect(.native).ts`, `src/core/containerLayoutBaseline.ts`) · **Used by:** [public-api](public-api.md) (`LegendList.tsx` renders `ListComponent`), [integrations](integrations.md) (reanimated injects `positionComponentInternal`), `src/section-list`.

## What does the component tree look like?

```
LegendList (public-api)
└─ ListComponent                      src/components/ListComponent.tsx
   └─ SnapOrScroll = SnapWrapper (if snapToIndices) wrapping ScrollComponent,
      or ScrollComponent directly:
        renderScrollComponent → useStableRenderComponent (stable forwardRef)
        else ListComponentScrollView  (native: Animated.ScrollView; web: <div> emulation)
      ├─ ScrollAdjust                  (scroll-stability area)
      ├─ LayoutView(ListHeaderComponent) → setHeaderSize
      ├─ ListEmptyComponent            (LegendList passes it only when data is empty)
      ├─ AlignItemsAtEndSpacer         (only if alignItemsAtEndPaddingEnabled)
      ├─ Containers   (only when canRender && no ListEmptyComponent)
      │  └─ ContainersInner <div> (web) | ContainersLayer Animated.View (native), sized to totalSize
      │     └─ ContainerLayoutCoordinator  (one layout effect for all slots)
      │        └─ ContainerSlot × numContainersPooled   (renders null until containerItemKey${id} set)
      │           └─ Container            (key={recycleItems ? undefined : itemKey})
      │              └─ PositionView | PositionViewSticky | positionComponentInternal
      │                 └─ ContextContainer.Provider {containerId, triggerLayout}
      │                    ├─ renderItem output   (via getRenderedItem in src/utils/getRenderedItem.ts)
      │                    └─ Separator (null for last item, via useIsLastItem)
      ├─ LayoutView(ListFooterComponent) → setFooterSize (+ maintainScrollAtEnd.onFooterLayout)
      └─ DevNumbers (IS_DEV && ENABLE_DEVMODE)
DebugView is a sibling of ListComponent (IS_DEV && ENABLE_DEBUG_VIEW)
```

Every container is absolutely positioned inside the container layer. `Container` computes the cross-axis placement (`getContainerPositionStyle`: column % offset/width, gap padding), and the `PositionView` variant applies the main-axis `top`/`left` from the `containerPosition${id}` signal. The default `POSITION_OUT_OF_VIEW` is `-10000000`. Container re-renders are driven by per-id signals (`containerItemKey${id}`, `containerItemData${id}`, `containerColumn${id}`, `containerSpan${id}`, `containerSticky${id}`, `containerPosition${id}`, `containerItemIndex${id}`), so a scroll that only moves rows re-renders only `PositionView`s.

## How does a container measure its item and report size to core?

Measurement starts in `useContainerMeasurement` (`src/hooks/useContainerMeasurement.tsx`), which `Container` calls with its `ref`. The path then depends on the platform and React Native architecture:

| Runtime | Synchronous pass (before paint) | Asynchronous / follow-up path |
|---|---|---|
| Native, new arch (Fabric) | Each `Container` commit runs `scheduleContainerLayout(ctx, id)` in a dep-less `useLayoutEffect`. This adds the id to `ctx.pendingContainerIds` and bumps `containerLayoutEpoch` once. `ContainerLayoutCoordinator`'s layout effect consumes the scope (`getContainerLayoutEffectScope`) and calls `measureContainersInLayoutEffect` (native), which runs a synchronous `view.measure` per container and then one `updateItemSizesBatch` | RN `onLayout` events from `useOnLayoutSync.native` → `processContainerLayout` → `updateItemSizes` |
| Native, old arch (Paper) | None. The coordinator skips its pass, and `useOnLayoutSync.native` registers its layout-effect `measure` only when `IsNewArchitecture` | `onLayout` only. A 16 ms timeout replays `lastSize` if a recycled view got an equal-sized item and emitted no `onLayout`. A size of 0 falls back to `ref.measure` |
| Web | Same coordinator pass. Web `measureContainersInLayoutEffect` reads `getBoundingClientRect()` and records it with `setContainerLayoutBaseline` | One shared `ResizeObserver` (`createResizeObserver`, `box: "border-box"`) → `useOnLayoutSync` (web) → `processContainerLayout` |

`processContainerLayout` is the single funnel for event-driven sizes. It takes the scroll-axis size, then calls `updateItemSizes(ctx, {containerId, itemKey, size})`. That function queues into `batchItemSizeUpdates` when a batch is open, otherwise it applies `updateItemSizesBatch` right away. `updateItemSizesBatch` belongs to core. It rejects measurements for containers that have since been reassigned, recalculates positions once per list, and triggers MVCP and maintain-at-end.

`triggerLayout` is exposed through `ContextContainer` and used by `useRecyclingState` and `useSyncLayout`. `useSyncLayout` returns a no-op on old arch. `triggerLayout` schedules a targeted coordinator pass on new arch and web, and forces a re-render (`layoutRenderCount`) on old arch, which re-runs the `useOnLayoutSync` effect deps.

Headers and footers use `LayoutView` (`src/platform/LayoutView(.native).tsx`), which is also built on `useOnLayoutSync`. On web with the default `measureInLayoutEffect = true`, the header is measured synchronously in a layout effect and then by ResizeObserver. On native new arch it is measured synchronously and by `onLayout`.

## What differs between web and native?

| Concern | Native (`*.native.ts(x)`) | Web (plain file) |
|---|---|---|
| Scroll view | `Animated.ScrollView` (`ListComponentScrollView.native.tsx`) | `<div overflow:auto>` plus an inner `.legend-list-content-container` div that emulates the RN ScrollView API (`ScrollViewMethods`) and emits RN-shaped `onScroll` events |
| `IsNewArchitecture` | `global.nativeFabricUIManager != null` | always `true`. Web layout reads are synchronous, so it follows the new-arch paths |
| Container layer | `Animated.View` whose height/width is the animated `totalSize` (`useValue$`) | `<div position:relative>` with `height: totalSize`, plus `useDOMOrder` |
| Row position | new arch: plain `View` with `top`/`left` from state, hidden (`opacity 0`) while `containerLayoutReady${id}` is false. Old arch: `PositionViewAnimated` (Animated value) | `<div>` with `top`/`left` and `contain: paint layout style`. When `#react-native-stylesheet` exists (RN-Web), it also gets `display:flex; flex-direction:column` |
| Sticky headers | `PositionViewSticky` with `translateX/Y` interpolated from `ctx.animatedScrollY`. `useStickyScrollHandler.native` swaps `onScroll` for `Animated.event(..., {useNativeDriver: true})`. Push-off by the next sticky comes from `getStickyPushLimit` | CSS `position: sticky` only on the container whose `containerItemIndex${id}` equals `activeStickyIndex` (set in `calculateItemsInView`); the rest stay absolute. There is no push-limit interpolation |
| Snapping | `snapToOffsets` (from `updateSnapToOffsets` → `snapToOffsets` signal) is passed through `SnapWrapper` to the RN ScrollView | the same prop becomes 1px `scroll-snap-align: start` anchor divs and `scroll-snap-type: x/y mandatory`. Disabled with external scroll |
| Pull to refresh | RN `RefreshControl` | `RefreshControl` returns `null` |
| RTL default | `I18nManager.isRTL` | stub always `false`, so web RTL needs the `rtl` prop |
| Misc shims | RN `StyleSheet`, `PixelRatio`, `Dimensions`, `unstable_batchedUpdates`. `flushSync` just calls `fn()` | `StyleSheet.flatten` = `Object.assign` of the array; `PixelRatio.get()` = 1; `window.inner*`; `react-dom` `unstable_batchedUpdates`/`flushSync`. `PlatformAdjustBreaksScroll` = true on web, `OS === "android"` on native |
| MVCP active gate | `isInMVCPActiveMode.native` = `dataChangeNeedsScrollUpdate` only | also active while an anchor lock holds (`hasActiveMVCPAnchorLock`) |

### Web-only behaviors worth knowing

- **ResizeObserver is a module singleton.** `createResizeObserver` keeps one `ResizeObserver` for all lists, with a `WeakMap<Element, Set<cb>>`, and wraps every delivery in `batchItemSizeUpdates` so each list flushes once per delivery. It observes `border-box` so its results match the coordinator's `getBoundingClientRect` reads. A delivery equal to the stored baseline (`getContainerLayoutBaseline`) counts as confirmation and is dropped, unless `webLayoutResync()` (MVCP active) forces a re-emit. Without `ResizeObserver` (tests, SSR) it is a no-op.
- **Shrinks are deferred one frame.** In `processContainerLayout`, a web shrink of more than 1px compared with core's `sizesKnown` (not the container's local `lastSize`, because the physical container may be recycled) waits one `requestAnimationFrame`. The element is then re-read with `getBoundingClientRect` and all pending shrinks are applied in one batch. This is skipped while MVCP is active. Reason, from the code comment: brief observer shrinks occur while web content settles.
- **DOM reordering for accessibility and focus.** Recycled containers keep their DOM position, so DOM order (tab order, screen readers) drifts away from visual order. `useDOMOrder` listens to `lastPositionUpdate` and calls `sortDOMElements` (`src/utils/reordering.ts`), which moves only the elements outside the longest increasing subsequence. It uses `Element.moveBefore` when available so focus, animations and iframes keep their state, and falls back to `insertBefore`. Delay is 0 when the list is quiet (no update for 1s, no scroll for 500ms, `moveBefore` supported, connected), otherwise 500ms debounced (commit 4e203bb3). The index map comes from `containerItemIndex${id}` signals, not `data-index` attributes (059e8906).
- **External scroll owners.** `useWindowScroll` and `scrollElement` (web only, and ignored when `renderScrollComponent` is set: `LegendList.tsx` resolves both to `undefined`/`false`, and `useDevChecks` warns). The list div loses `overflow`. Offsets are list-relative (`getElementScrollPosition`, `resolveExternalScrollOffset`) and clamped to `[0, max]`. `observeScrollLayout` watches the list, its ancestors up to the owner, and preceding siblings (ResizeObserver plus a `childList` MutationObserver), so content above the list moving it triggers a fresh `onLayout` (10f42e34). `scrollElement === null` means "owner not mounted yet", so nothing is observed.
- **Scroll event shaping.** DOM `scroll` events are rAF-coalesced (`useRafCoalescer`). They flush immediately during `scrollingTo`, unfinished initial scroll, or MVCP-active mode. `scrollend` (or a 200ms fallback timer) fires `onInternalScrollEnd`. Scrolling the wheel toward the start interrupts `maintainScrollAtEnd`. With MVCP enabled, `overflow-anchor: none` keeps Chrome's native scroll anchoring from compensating a second time. `contentOffset` is applied on mount and again on the next frame.
- **Hidden scrollbars** inject one `<style id="legend-list-scrollbar-axis-hidden-style">` with per-axis classes (`webConstants.ts`). Tailwind-style `gap-*` classes on the list trigger a dev warning, because gaps have to be exact numbers (see columns below).

## Platform file resolution: who picks `.native`?

| Consumer | Rule | Evidence |
|---|---|---|
| Published native bundles (`react-native`, `animated`, `keyboard*`, `reanimated`, `section-list`) | esbuild `resolveExtensions: [".native.tsx", ".native.ts", ".tsx", ".ts", ".json"]` | `tsup.config.ts` second config |
| Published web bundles (`react`, `react-native.web`) | default resolution, plain files only. Both come from `src/react.ts` | `tsup.config.ts` `webEntryPoints` |
| `@legendapp/list/react-native` import under the `browser` condition (RN-Web, bundlers) | resolves to `react-native.web.*`, so RN-Web users get the DOM implementation, not RN-Web views. There is no root `"."` export | `package.json` `exports` |
| `example/` (Metro) | normal Metro platform extensions with a `resolveRequest` alias to `../src` | `example/metro.config.js` |
| `example-web/` (Vite) | `@` → `../src`, extensions `.tsx` first, so web files win | `example-web/vite.config.ts` |
| `tsc` | `@/*` path alias resolves to plain (web) files | `tsconfig.json` |
| `bun test` | explicit `mock.module` remapping in `__tests__/setup.ts` (next section) | `bunfig.toml` `preload` |

**Inferred:** because the typechecker and the web build both import the plain file, each `.native` variant has to export the same names with compatible types, or native-only call sites will drift without any type error.

## How do bun tests choose web vs native?

The default is native, new arch, iOS. `bunfig.toml` preloads `__tests__/setup.ts`. That file:
1. sets `global.nativeFabricUIManager = {}`, so `IsNewArchitecture` is `true`;
2. `mock.module("react-native", …)` points to `__tests__/__mocks__/react-native.ts` (`Platform.OS = "ios"`, `I18nManager`);
3. for every pair in `nativeModuleOverrides`, `mock.module("@/x", () => require("../src/x.native.ts(x)"))`, then snapshots those exports and re-registers them after `mock.restore()` in `afterEach`. It also restores `Platform` and `I18nManager` fields.

To exercise web code, existing tests use one of these patterns:
- **Query-suffixed import** such as `await import("../../src/components/Containers?web-render")` or `"…/useOnLayoutSync.tsx?web-layout-baseline"`. The `@/x` mock intercepts the plain file's path, and the unique `?query` gives a fresh, un-mocked module instance. That module's own transitive `@/…` imports still get the native overrides, so mock any you depend on (as `useOnLayoutSync.web.test.tsx` does with `@/hooks/createResizeObserver`).
- **Flip `Platform.OS = "web"`** on the shared mock object for code that branches at runtime (`processContainerLayout`, `useDOMOrder`). `afterEach` resets it.
- **Subprocess `.cases.ts(x)` files.** A `*.test.ts` wrapper runs `Bun.spawnSync(["bun","test","./__tests__/…cases.tsx"])`. Bun only auto-discovers `*.test.*`, so cases files run only through their wrapper. Use this when module-level state must be fresh: the global ResizeObserver, `mock.module("@/constants-platform", () => ({IsNewArchitecture: false}))` for old arch (`*.oldarch.cases.*`), or DOM-order timers.
- File names carry the intent: `*.web.test.ts`, `*.native.test.tsx`, `*.oldarch.test.ts`. Of 130 test files, 92 use `.test.ts` and 38 use `.test.tsx`.

**Gotcha:** `nativeModuleOverrides` does **not** list `@/core/doScrollTo` or `@/utils/isInMVCPActiveMode`. Default tests therefore run native components together with the **web** `doScrollTo` and the **web** `isInMVCPActiveMode`. `doScrollTo.native.test.ts` imports `@/core/doScrollTo.native` explicitly. When you add a new `.native` file, add it to `nativeModuleOverrides` or tests will silently use the web variant.

## Invariants and gotchas

- **Hooks branch on module constants.** `useOnLayoutSync.native` calls `useLayoutEffect` only `if (IsNewArchitecture)`, and `PositionView` is chosen at module load. This works because the constant never changes at runtime. In tests, set `IsNewArchitecture` before the module is imported (hence subprocesses).
- **Recycled containers hold stale local state.** The same `Container` id moves between item keys. Do not trust `useContainerMeasurement`'s `lastSize` for the current item on web; core's `sizesKnown` is authoritative. Measurement callbacks check `state.itemKey === currentItemKey`, and the native coordinator uses `containerItemGenerations` tokens so a late `measure` callback cannot overwrite a newer measurement (bbad389e).
- **Measurement order is commit order.** Child `Container` layout effects only add ids. The parent `ContainerLayoutCoordinator` measures all of them in one pass, so positions are never recalculated against partially updated sizes. Do not move measurement back into each `Container`'s own effect.
- **`scheduleContainerLayout(ctx)` with no target** means "all containers" (`pendingContainerIds = null`). It overrides targeted requests made later in the same commit.
- **Web drops zero-size layouts.** `useOnLayoutSync` (web) ignores 0×0 rects, which fixed lists breaking on 0-height layouts (8c3a7d3d).
- **Gaps.** `createColumnWrapperStyle` strips `gap`/`rowGap`/`columnGap` from the (copied) `contentContainerStyle` into `ctx.columnWrapperStyle`. Gaps are then drawn as per-container padding (`getContainerPositionStyle`, with `boxSizing: border-box` on web) plus negative margins on the container layer, so measured sizes include the gap (`ctx.scrollAxisGap`).
- **Horizontal RTL.** Core computes physical positions (`toPhysicalHorizontalItemPosition`). The web container layer and each horizontal container force `direction: ltr` so `left` stays physical. `toLogicalHorizontalOffset` detects the platform's RTL scroll offset convention (`negative` if raw < 0, otherwise `normal` vs `inverted` by distance to the last scroll; the default before any scroll is `normal` on web and `inverted` on native) and caches it in `state.horizontalRTLScrollType` (3e808df5, c87bd84e).
- **Non-recycling remounts.** `Container` passes `key={recycleItems ? undefined : itemKey}` to the position component, so without `recycleItems` the item subtree remounts when a slot changes item. With recycling it re-renders, and `renderedItemInfo` is memoized on `[itemKey, containerItemData, extraData]`.
- **Sticky on native requires the Animated `onScroll`.** `useStickyScrollHandler.native` replaces `onScroll` with a native-driver `Animated.event` unless an integration provides `stickyPositionComponentInternal` (reanimated, new arch only). **Inferred:** a custom `renderScrollComponent` must accept Animated events for sticky headers to move.
- **Debug overlays are compiled off.** `ENABLE_DEVMODE` and `ENABLE_DEBUG_VIEW` in `src/constants.ts` are `IS_DEV && false`, so flip them locally. `IS_DEV` (`devEnvironment.ts`) prefers `process.env.NODE_ENV ?? MODE` and falls back to `__DEV__`.

## Design decisions

| Decision | Reason / evidence |
|---|---|
| Native rows positioned with `top`/`left`, not `transform` | "fix a myriad of problems (android input, macos issues, keyboard animations), as it wasn't really much of a performance improvement" (d1f38ef2). Sticky headers went back to `transform` (0a234ee2) because they follow `animatedScrollY` on the native driver |
| New-arch `PositionView` is a plain `View`, not Animated | comment in `PositionView.native.tsx`: Animated is better on old arch but would make position updates asynchronous with other state updates on new arch |
| One coordinated, post-commit measurement pass with batched size application | "so recycled rows cannot recalculate against partially updated sizes" (bbad389e) |
| Web treated as "new architecture" | `src/constants-platform.ts`: "Web is closer to react native new architecture because it has synchronous layout" |
| Hide unmeasured rows (`experimental_hideItemsUntilMeasured`, native new arch) | Fabric cannot re-enter the current commit when measurement corrects an estimated position (comment in `PositionView.native.tsx`, d59c44c9) |
| Reorder the DOM with LIS plus `moveBefore` | keeps focus, animation and iframe state while the DOM follows visual order (4e203bb3) |
| A single external-scroll implementation for window and element owners | `scrollElement` reuses the `useWindowScroll` path with one observation path (10f42e34) |
| `useStableRenderComponent` for `renderScrollComponent` | wraps the callback in a stable `forwardRef` component (refs via `useLatestRef`), so an inline `renderScrollComponent` does not remount the scroll view on every render. **Inferred** from its implementation |

## Where to change things

| To… | Start at |
|---|---|
| Change how rows are absolutely positioned or styled across columns | `getContainerPositionStyle` in `src/components/Container.tsx`; `PositionView(.native).tsx` |
| Change what a container renders around the item (separator, context) | `Container` in `src/components/Container.tsx`, `Separator.tsx`, `src/state/ContextContainer.ts` |
| Change when or how item sizes are read | `src/hooks/useContainerMeasurement.tsx` (`processContainerLayout`), `src/core/measureContainersInLayoutEffect(.native).ts`, `ContainerLayoutCoordinator.tsx` |
| Change ResizeObserver behavior on web | `src/hooks/createResizeObserver.ts`, `src/hooks/useOnLayoutSync.tsx`, `src/core/containerLayoutBaseline.ts` |
| Change native `onLayout` handling | `src/hooks/useOnLayoutSync.native.tsx` |
| Change the web scroll view (events, imperative API, snapping, scrollbars) | `src/components/ListComponentScrollView.tsx`, `webScrollUtils.ts` |
| Change window or shared-element scroll | `webScrollUtils.ts`, `observeScrollLayout.ts`, the props resolution in `LegendList.tsx` |
| Change sticky headers | native: `PositionViewSticky` in `PositionView.native.tsx`, `stickyPositionUtils.ts`, `src/platform/useStickyScrollHandler.native.ts`; web: `PositionViewSticky` in `PositionView.tsx`; active index: `calculateItemsInView` |
| Change DOM order / accessibility order | `src/hooks/useDOMOrder.ts`, `src/utils/reordering.ts` |
| Change RTL offset mapping | `src/utils/rtl.ts` |
| Add a platform shim | add `src/platform/X.ts` and `X.native.ts(x)` with identical exports, then add the pair to `nativeModuleOverrides` in `__tests__/setup.ts` |
| Add header/footer/empty-state layout | `ListComponent.tsx`, `src/platform/LayoutView(.native).tsx` |
| Add a dev-time warning | `src/components/useDevChecks.ts` (no-op in production) |
| Turn on visual debugging | `ENABLE_DEVMODE` / `ENABLE_DEBUG_VIEW` in `src/constants.ts` (`DevNumbers`, `DebugView`) |

## Tests

| Topic | Files |
|---|---|
| Native containers, coordinator scheduling | `__tests__/components/Containers.native.test.tsx`, `Containers.layoutScheduler.native.test.tsx`, `__tests__/core/scheduleContainerLayout.test.ts`, `measureContainersInLayoutEffect.native.test.ts` |
| Web containers, measurement | `Containers.web.test.ts`, `Container.web.test.ts`, `__tests__/hooks/useOnLayoutSync.web.test.tsx`, `useContainerMeasurement.web.test.ts` (+ `.cases`), `createResizeObserver.web.test.ts` (+ `.cases`), `__tests__/core/measureContainersInLayoutEffect.web.test.ts` |
| Position views, sticky | `PositionView.native.test.tsx`, `PositionView.web.test.tsx`, `stickyPositionUtils.test.ts`, `__tests__/platform/useStickyScrollHandler.native.test.tsx` |
| Web scroll view and external scroll | `ListComponentScrollView.web.test.tsx`, `webScrollUtils.test.ts`, `observeScrollLayout.test.ts` |
| DOM order | `__tests__/hooks/useDOMOrder.test.tsx` (+ `.cases.tsx`) |
| Misc | `SnapWrapper.test.tsx`, `ContainerSlot.test.tsx`, `ListComponent.test.ts`, `ListComponent.renderScrollComponent.test.tsx`, `__tests__/hooks/useOnLayoutSync.native.test.tsx` |

Run a focused set with `bun test __tests__/hooks/useOnLayoutSync.web.test.tsx __tests__/components/Containers.native.test.tsx` (verified at 3d80d167: 20 pass across 5 files in about 0.2s). For behavior visible in a browser, `cd example-web && bun run dev:fixtures` has fixtures such as `WindowScrollExample`, `ExternalScrollExample`, `CountriesWithHeadersStickyExample`, `SnapToIndicesExample` and `ColumnsExample` (`example-web/src/fixtures/`). The test helpers are `__tests__/helpers/testRenderer.ts` (a react-test-renderer re-export, used by about 30 files) and `__tests__/helpers/testingLibrary.ts` (`render`/`act` wrappers over `@testing-library/react-native` internals, used by about 12 files).
