<!-- index-codebase: generated from 3d80d167 on 2026-10-03. Code wins when it disagrees with this doc. -->
# State

**Purpose:** The per-list reactive signal store (`StateContext.values` + listeners, read with `peek$`, written with `set$`, subscribed with `useArr$`), the big mutable non-reactive `InternalState` object hung off it as `ctx.state`, the per-container `ContextContainer` and the public item hooks built on both. It does not decide *what* to render or where. That is [virtualization](virtualization.md) (calculateItemsInView, positions) and [scrolling](scrolling.md).
**Paths:** `src/state/state.tsx`, `src/state/ContextContainer.ts`, `src/state/getContentSize.ts`, `src/state/getContentInsetEnd.ts`, `src/hooks/useValue$.ts`, `src/hooks/useValueListener$.ts`, `src/hooks/useAnimatedValue.ts`, the `InternalState` type in `src/types.internal.ts`, and its initializer in `src/components/LegendList.tsx` (`LegendListInner`)
**Depends on:** platform (`createAnimatedValue`), `use-sync-external-store/shim` · **Used by:** every other area: virtualization, scrolling, initial-scroll, scroll-stability, rendering-and-platform, integrations, public-api

## How it works

### Two kinds of state per list
Each `LegendList` gets one `StateContext` object (`ctx`). `LegendList` wraps `LegendListInner` in `<StateProvider>`, and `StateProvider` creates `ctx` once with `useState`. The object has two parts:

| Part | Where | Reactive? | Who reads it |
|---|---|---|---|
| **Signals**: `ctx.values: Map<ListenerType, any>` + `ctx.listeners: Map<ListenerType, Set<cb>>` | `src/state/state.tsx` | Yes. `set$` notifies listeners synchronously, and `useArr$` re-renders subscribers | Rendering components (containers, `ListComponent`, `Containers`, `ScrollAdjust`), public hooks, `getState().listen` |
| **Internal state**: `ctx.state: InternalState` | type in `src/types.internal.ts`, built in `LegendListInner` | No. Plain mutation, nothing re-renders | All `src/core/*` and `src/utils/*` algorithms (positions, sizes, scroll, MVCP, initial scroll) |
| **Other mutable ctx fields**: `viewRefs`, `containerLayoutTriggers`, `pendingContainerIds`, `mapViewability*`, `positionListeners`, `scrollAxisGap`, `columnWrapperStyle`, `scrollToEnd`, `scrollRequestTracker`, `animatedScrollY` | `StateContext` in `src/state/state.tsx` | No | Registries and side channels between components and core |

**The rule:** a value goes in signals only if some component has to re-render (or an external listener has to fire) when it changes. Anything else goes on `ctx.state`. Commit `7f65cf0a` ("move scrollingTo from state values to state") and `621f42b2` ("activeStickyIndex was in both types of state") applied this rule.

### Why this shape
- `LegendListInner` subscribes to no signals at all (`src/components/LegendList.tsx` has no `useArr$`). Scrolling, measuring and recycling all run through `set$` on fine-grained keys, so only the affected `ContainerSlot`/`Container`/`PositionView` re-renders. The whole list never does. History: commits `d674440c` and `952bbb5b` ("Create a state signaling system… instead of Legend State"), `4215406f` ("Move ScrollView rendering into a separate memoized component so that it never re-renders") and `19d8eb74` ("reduce the size of the re-render while scrolling").
- Subscriptions use `useSyncExternalStore` (`f8e30dac`, which replaced `useReducer`). The import is from the `use-sync-external-store/shim` package so older React versions still work (`e3446adf`). `getServerSnapshot` is passed (`54d4902f`, issue #349).
- `ctx.state` lives on the context object rather than in a `useRef`. That way StrictMode's double render cannot create it twice ("all our functions expect it to be created once", comment in `LegendListInner`).

### Signal primitives (`src/state/state.tsx`)
| Function | Behavior |
|---|---|
| `peek$(ctx, key)` | `values.get(key)`. Non-reactive read, used everywhere in core |
| `set$(ctx, key, v)` | No-op if `values.get(key) === v` (reference equality). Otherwise stores the value and calls every listener **synchronously** |
| `listen$(ctx, key, cb)` | Adds `cb` to a Set and returns an unsubscribe function. Used by `useValue$`, `useValueListener$`, `useAdaptiveRenderChange`, `getState().listen`, header/footer metrics in `LegendList.tsx` |
| `useArr$([k1, k2, …])` | `useSyncExternalStore` over 1–8 keys. Returns a tuple that keeps the same reference until one value changes. Re-renders only when one of *these* keys changes |
| `useSelector$(key, selector)` | Single key, and re-renders only when `selector(value)` changes (used by `useIsLastItem`) |
| `listenPosition$` / `notifyPosition$` | Separate per-item-key channel on `ctx.positionListeners`. Exposed only as `getState().listenToPosition`. `updateItemPositions` checks `positionListeners.size` and pays nothing when nobody listens |
| `useValue$(key)` | Mirrors a signal into an RN `Animated.Value` without re-rendering. Only native code uses it (`PositionViewAnimated` on old arch, `Containers.native.tsx` for `totalSize`) |
| `useValueListener$(key, cb)` | `listen$` in a layout effect. Used by web `ScrollAdjust.tsx` to apply `scrollAdjust` to the DOM without rendering |

### Signal keys: who writes, who renders
List-level keys (defaults seeded in `StateProvider`: `alignItemsAtEndPadding`, `containerLayoutEpoch`, `stylePaddingTop`, `headerSize`, `numContainers`, `activeStickyIndex=-1`, `isAt*/isNear*=false`, `adaptiveRender="normal"`, `totalSize`, `scrollAdjustPending`):

| Key(s) | Writer | Reactive reader |
|---|---|---|
| `numContainers`, `numContainersPooled` | `calculateItemsInView`, `doInitialAllocateContainers` | `Containers` (renders `numContainersPooled` slots) |
| `totalSize` | `addTotalSize`, `updateItemSizesBatch`, `setPaddingTop`, `clearCaches`, `LegendListInner` | `ContainersInner` (height), `Containers.native` (Animated) |
| `readyToRender` | `setInitialRenderState` | `ContainersInner` (opacity) |
| `scrollAdjust`, `scrollAdjustPending` | `ScrollAdjustHandler` | `ScrollAdjust(.native)`. Core peeks `scrollAdjustPending` |
| `scrollAdjustUserOffset` | `createImperativeHandle` | `ScrollAdjust` |
| `headerSize`, `footerSize` | `updateContentMetrics` | `getContentSize`, metrics listener in `LegendList` |
| `alignItemsAtEndPadding` | `updateContentMetricsState`, `doMaintainScrollAtEnd` | `ListComponent` |
| `anchoredEndSpaceSize` | `updateAnchoredEndSpace` | `ListComponent`, `ListComponentScrollView`, `getContentInsetEnd` |
| `otherAxisSize` | `updateItemSizes` (max of measured cross-axis) | `ContainersInner`, `ListComponent` |
| `isAtEnd/isAtStart/isNearEnd/isNearStart/isWithinMaintainScrollAtEndThreshold` | `checkAtBottom`, `checkAtTop` | public `getState()`/`listen`, reanimated `sharedValues` |
| `activeStickyIndex` | `calculateItemsInView` | `PositionViewSticky` |
| `lastItemKeys`, `numColumns`, `extraData`, `maintainVisibleContentPosition` | `LegendListInner` (render phase) | `useIsLastItem`, `Container`, `Containers` |
| `snapToOffsets` | `updateSnapToOffsets` | `SnapWrapper` |
| `scrollSize` | `handleLayout` | `useListScrollSize` |
| `adaptiveRender` | `core/adaptiveRender.ts` (initial value via `ctx.values.set` in `LegendListInner`) | `useAdaptiveRender`, `useAdaptiveRenderChange` |
| `containerLayoutEpoch` | `scheduleContainerLayout` | `ContainerLayoutCoordinator` (runs the post-commit measurement pass) |
| `lastPositionUpdate`, `debugRawScroll`, `debugComputedScroll` | `calculateItemsInView` | `DebugView`, public `listen` |

Per-container keys (template literals with the container id appended, for example `containerItemKey3`):

| Key | Meaning | Rendered by |
|---|---|---|
| `containerItemKey${id}` | Item key currently assigned to the slot. `undefined` means the slot is empty | `ContainerSlot` (returns null when undefined), `PositionView.native`, recycling hooks, `useIsLastItem` |
| `containerItemIndex${id}` / `containerItemData${id}` | Index and data for the assigned item | `Container` (data), sticky views, `useRecyclingState`/`Effect` |
| `containerPosition${id}` | Main-axis offset (`POSITION_OUT_OF_VIEW` when parked) | `PositionView*` |
| `containerColumn${id}` / `containerSpan${id}` | Grid column and span | `Container` (cross-axis style) |
| `containerSticky${id}` | Slot holds a sticky header | `Container` (picks `PositionViewSticky`) |
| `containerLayoutReady${id}` | Only published when `experimental_hideItemsUntilMeasured` is on. `undefined` means ready | `PositionView.native`, reanimated position view |

Public subset: `LegendListListenerType` (in `state.tsx`) is what `ref.getState().listen` accepts (`src/types.base.ts`, `3f3c1af1`). Renaming those keys is a breaking API change.

### Traced flow: a container receives a new item
1. Scrolling or a data change calls `state.triggerCalculateItemsInView` → `calculateItemsInView(ctx)` (`src/core/calculateItemsInView.ts`).
2. `findAvailableContainers` chooses physical slots. For each allocation it:
   - removes the old key from `state.containerItemKeys` (a non-reactive `itemKey → containerId` map),
   - writes `state.containerItemMetadata.set(containerIndex, …)` **before** any signal, and bumps `state.containerItemGenerations[containerIndex]`,
   - optionally `set$(containerLayoutReady${i}, …)`, then `set$` `containerItemKey`, `containerItemIndex`, `containerItemData`, `containerSticky`, and `numContainers`/`numContainersPooled` when they grow.
3. A loop over all containers calls `syncMountedContainer(ctx, i, itemIndex)` (`src/core/syncMountedContainer.ts`). It reads `state.positions/columns/columnSpans` and calls `set$` on `containerPosition/Column/Span` only when the value differs (`peek$` compare). It refreshes `containerItemData` only when `itemsAreEqual` says the item changed. The result is cached per index in `state.pendingDataComparison`. Released slots get `containerItemKey/Index/Data = undefined` and `containerPosition = POSITION_OUT_OF_VIEW`.
4. Every `set$` calls the `useSyncExternalStore` subscribers of that key. `ContainerSlot` (subscribed to `containerItemKey${id}`) re-renders with the new `itemKey` prop. `Container` (subscribed to column/span/data/sticky plus `numColumns`/`extraData`) recomputes `getRenderedItem(itemKey, id)` (`src/utils/getRenderedItem.ts`), which reads `ctx.state.containerItemMetadata` and `state.props.renderItem`. `PositionView` (subscribed to `containerPosition${id}`) moves only its own view. On old-arch native, `PositionViewAnimated` uses `useValue$`, so position changes skip React entirely.
5. After commit, `useContainerMeasurement` and `ContainerLayoutCoordinator` measure and feed sizes back through `updateItemSizesBatch`. That function ignores a measurement unless `peek$(containerItemKey${id}) === measurement.itemKey` (stale-recycle guard).

## Key types and entry points
| Symbol | File | Role |
|---|---|---|
| `StateContext`, `StateProvider`, `useStateContext` | `src/state/state.tsx` | Per-list context object and provider |
| `ListenerType`, `ListenerTypeValueMap`, `LegendListListenerType` | `src/state/state.tsx` | Signal key union, value types, public subset |
| `InternalState` | `src/types.internal.ts` | Mutable algorithm state, including `props` (a normalized snapshot of props) |
| `ContainerItemMetadata` | `src/types.internal.ts` | Per-slot assignment snapshot (data generation, index, type, fixed size) |
| `ContextContainer`, `ContextContainerType` | `src/state/ContextContainer.ts` | `{ containerId, triggerLayout }`. Item identity is deliberately absent |
| `useRecyclingState`, `useRecyclingEffect`, `useViewability`, `useViewabilityAmount`, `useIsLastItem`, `useListScrollSize`, `useSyncLayout`, `useAdaptiveRender`, `useAdaptiveRenderChange` | `src/state/ContextContainer.ts` | Public item hooks, re-exported from `src/entrypoints/shared.ts` |
| `getContentSize(ctx)` | `src/state/getContentSize.ts` | Header + footer + layout size (minus trailing `scrollAxisGap`) + paddings + `alignItemsAtEndPadding` + end inset. Used for max-scroll and edge checks across core |
| `getContentInsetEnd(ctx)` | `src/state/getContentInsetEnd.ts` | End inset = max(prop/native inset + `contentInsetEndAdjustment`, anchored end space). Honors `state.contentInsetOverride` |
| `internal.{peek$, useArr$, useStateContext}` | `src/entrypoints/shared.ts` | Bridge so integration entrypoints share the same store module |

## Invariants and gotchas
- **`set$` compares by reference.** If you mutate an array or object in place and `set$` the same reference, nothing fires. Build a new value instead (as with `lastItemKeys`, `snapToOffsets`, `scrollSize`).
- **`set$` is synchronous and can render.** Listeners run inside the call, so anything a subscriber's render reads from `ctx.state` must be written *before* the `set$`. That is why `containerItemMetadata` is published before `containerItemKey` (`6d423f31`, comment "Publish coherent assignment metadata before reactive signals can render the container").
- **`ctx.values.set` bypasses listeners.** Production does this only to seed values before anyone subscribes (`adaptiveRender` in `LegendListInner`). Tests also use it to seed container signals.
- **`useArr$` keys its memo by the joined names** (`"\0"` separator), so inline arrays like `useArr$([\`containerItemKey${id}\`])` stay subscribed across renders (`5dc6db41`). The `get` snapshot must keep returning the same array reference while values are unchanged, or `useSyncExternalStore` loops.
- **Hooks must survive running without a list.** `createSelectorFunctionsArr(null, …)` returns empty values and `ContextContainer` hooks no-op without a container (`333ce86c`, `4d958e31`). Outside a container the hooks use `NO_CONTAINER_ID = -1`, so they never subscribe to a real slot's signals.
- **The stale header comment in `state.tsx`** ("use$ is only ever called once for each unique name… don't need to manage a Set of listeners") no longer describes the code. Listeners are now a `Set` with real unsubscribe.
- **Render-phase writes.** `LegendListInner` writes `state.props = {…}`, `ctx.columnWrapperStyle`, `ctx.scrollAxisGap` and some signals during render. React can restart that render, so data-change detection uses monotonic epochs (`dataChangeEpoch` / `handledDataChangeEpoch`, consumed in a layout effect) to avoid losing a change.
- **`totalSize` exists twice.** `state.totalSize` (and `state.pendingTotalSize` on old arch while the list shrinks during initial scroll) is the computed truth. The `totalSize` signal is what gets rendered and may lag or be padded on purpose: `setPaddingTop` inflates it for 16ms, and `addTotalSize(…, notifyTotalSize=false)` defers publishing. Read `state.pendingTotalSize ?? state.totalSize` in math, as `getContentSize` does.
- **Recycled containers.** A `Container` with a given `id` stays mounted across items. When `recycleItems` is on, the item subtree persists too (`Container` sets `key={recycleItems ? undefined : itemKey}` on the position view), so `useState` inside an item leaks between items. Use `useRecyclingState`, which resets synchronously during render when `containerItemKey` changes and calls `triggerLayout` on set, or `useRecyclingEffect`.
- **Measurement and identity guards.** Compare `peek$(containerItemKey${id})` against the expected key, and check `state.containerItemGenerations[id]`, before trusting async callbacks for a container (`bbad389e`, `d59c44c9`).
- **`containerLayoutReady` treats `undefined` as ready.** Lists that never opt in never publish it. `resetContainerLayoutReady` walks `ctx.values.keys()` because `numContainers` can lag.
- `contextNum` is a debug counter and nothing reads it.

## Design decisions
- **Container context carries only `{containerId, triggerLayout}`.** Item key, index and data moved into signals so that recycling does not change the context value and propagate broadly through React (`154a371d`). Recycling hooks subscribe to `containerItem*${id}` instead.
- **Pool rendered at a larger size than used.** `Containers` renders `numContainersPooled` slots, more than `numContainers`, so it seldom has to re-render to add slots (`e3bb614d`).
- **Animated positions only on old arch.** "The Animated version is better on old arch but worse on new arch… would make position updates not synchronous with the rest of the state updates" (comment in `PositionView.native.tsx`).
- **Measurement batching through a signal.** `scheduleContainerLayout` accumulates ids in `ctx.pendingContainerIds` and bumps `containerLayoutEpoch` once, so one parent layout effect measures every changed slot (`bbad389e`).
- **Inferred:** a hand-rolled store instead of Legend State or another library keeps the package free of runtime dependencies and allows per-key Sets with no selector overhead on the scroll hot path.

## Where to change things
| To… | Start at |
|---|---|
| Add a list-level signal | Add to `ListenerType` and `ListenerTypeValueMap` in `src/state/state.tsx`. Seed a default in `StateProvider` if readers assume one, and add it to `createMockContext` defaults |
| Expose a signal publicly via `getState().listen` | Add to `LegendListListenerType`. It becomes public API (see [public-api](public-api.md)) |
| Add a per-container signal | Add a template-literal member and a mapped-type block in `state.tsx`. Write it in `calculateItemsInView` (assignment and release paths) and `doInitialAllocateContainers`. Read it in `Container`/`PositionView*` |
| Add non-reactive bookkeeping | Add a field to `InternalState` (`src/types.internal.ts`), initialize it in the `ctx.state = {…}` literal in `LegendListInner`, and add a default in `__tests__/__mocks__/createMockState.ts` |
| Pass a new prop to core | Add to `InternalState["props"]` and the `state.props = {…}` assignment in `LegendListInner` |
| Add a public per-item hook | `src/state/ContextContainer.ts`, then export from `src/entrypoints/shared.ts`. Subscribe with `useArr$` on `container*${containerId}` and no-op when there is no container |
| Change how content length or end inset is computed | `src/state/getContentSize.ts`, `src/state/getContentInsetEnd.ts` (many core callers, see [scrolling](scrolling.md)) |
| Drive a native style without re-render | `useValue$` (old arch only, otherwise prefer `useArr$`) |

## Tests
- Run `bun test __tests__/state/ContextContainer.test.tsx __tests__/hooks/useValue$.test.tsx __tests__/core/syncMountedContainer.test.ts` (43 pass at 3d80d167). Also see `__tests__/hooks/useRecyclingEffect.test.tsx` (hooks outside a list) and `__tests__/components/ContainerSlot.test.tsx`.
- **Core unit tests** (64 files) build `ctx` with `createMockContext(initialValues, stateOverrides)` from `__tests__/__mocks__/createMockContext.ts`, which calls `createMockState(overrides)` (`__tests__/__mocks__/createMockState.ts`). How the mocks differ from production:
  - `values.set("totalSize")` also writes `state.totalSize` (production writes them separately).
  - `state.isAtEnd/isAtStart/isNear*/activeStickyIndex` are accessors that proxy to `values`. Production `InternalState` has no such fields.
  - `scrollAdjustHandler` is a stub.
  - Defaults are `totalSize: 1000`, `scrollLength: 300`, `keyExtractor: item_${index}`, and `stylePaddingTop: 0`.
  - `positions/columns/columnSpans` setters copy arrays.
- **Component and hook tests** render the real `StateProvider` and seed signals with `set$` (which re-renders) or `ctx.values.set` (which does not), wrapping children in `ContextContainer.Provider` (see `SignalBackedContainerProvider` in `ContextContainer.test.tsx`).
