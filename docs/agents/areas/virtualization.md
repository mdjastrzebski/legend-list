<!-- index-codebase: generated from 3d80d167 on 2026-10-03. Code wins when it disagrees with this doc. -->
# Virtualization

**Purpose:** decides which items are rendered and where. It covers item sizes (estimate, then average, then measured), item positions and total content size, the buffered render window, which physical container each item gets (allocation and recycling), and viewability callbacks. It does **not** own scroll-offset corrections (MVCP, maintainScrollAtEnd, anchored end space), initial-scroll bootstrapping, or the React components that draw containers. It calls into those areas only through narrow hooks.
**Paths:** `src/core/calculateItemsInView.ts`, `src/core/updateItemPositions.ts`, `src/core/updateItemSizes.ts`, `src/core/{setSize,addTotalSize,updateTotalSize,prepareColumnStartState}.ts`, `src/core/{checkStructuralDataChange,resetLayoutCachesForDataChange,doInitialAllocateContainers,checkResetContainers}.ts`, `src/core/{syncMountedContainer,containerItemMetadata,containerLayoutBaseline,containerLayoutReady,scheduleContainerLayout,ScheduledWork}.ts`, `src/core/measureContainersInLayoutEffect(.native).ts`, `src/core/{viewability,adaptiveRender,useWrapIfItem}.ts`, `src/utils/{findAvailableContainers,containerPool,getItemSize,getEffectiveDrawDistance,getAlwaysRenderIndices,getId,checkAllSizesKnown,getRenderedItem,useRafCoalescer,layoutMeasurement}.ts`
**Depends on:** [state](state.md) (`peek$`/`set$` signals, `InternalState`) · [scroll-stability](scroll-stability.md) (`prepareMVCP`, `doMaintainScrollAtEnd`, `maybeUpdateAnchoredEndSpace`) · [initial-scroll](initial-scroll.md) (`evaluateBootstrapInitialScroll`, `handleInitialScrollLayoutReady`)
**Used by:** [scrolling](scrolling.md) (`updateScroll` → `triggerCalculateItemsInView`) · [rendering-and-platform](rendering-and-platform.md) (`Containers`, `ContainerSlot`, `Container`, `ContainerLayoutCoordinator`, `useContainerMeasurement`) · [public-api](public-api.md) (`LegendList.tsx` wiring, `clearCaches`, `getState`)

## How it works

### Core state (all on `ctx.state`, i.e. `InternalState` in `src/types.internal.ts`)
| Field | Meaning |
|---|---|
| `idCache[i]`, `indexByKey` | index→key and key→index. Keys come from `getId` (`keyExtractor`, which `LegendList` defaults to `index.toString()`). |
| `sizesKnown: Map<key, number>` | **Authoritative** sizes: measured, or resolved from `getFixedItemSize`. Includes `ctx.scrollAxisGap`. |
| `sizes: Map<key, number>` | The size currently **counted in `totalSize`**. It may be an estimate. `setSize` keeps `totalSize` in sync by adding the diff. |
| `averageSizes[itemType]` | Running `{avg, num}` per `getItemType` value (`""` when there is no `getItemType`). Updated only for measured, non-fixed, size>0 items. |
| `positions[i]`, `columns[i]`, `columnSpans[i]` | Item top offsets in item space (header/padding excluded), plus the 1-based column and span for grids. |
| `startBuffered/endBuffered`, `startNoBuffer/endNoBuffer`, `idsInView` | Render window and visible window from the last pass. |
| `containerItemKeys: Map<key, containerIndex>` | Reverse map of current assignments. |
| `positionRecalculationStartIndex` | Lowest index whose size changed since the last position pass. |
| `positionsAreCurrent` | The single-column position reuse flag (see Design decisions). |
| `scrollForNextCalculateItemsInView {top,bottom}` | Cached window bounds. Scrolling inside them skips the whole pass. |

Per-container reactive signals (in `ctx.values`, read by components): `containerItemKey${i}`, `containerItemIndex${i}`, `containerItemData${i}`, `containerPosition${i}` (`POSITION_OUT_OF_VIEW` = -10000000 when unused), `containerColumn${i}`, `containerSpan${i}`, `containerSticky${i}`, `containerLayoutReady${i}`. List-wide: `numContainers` (slots in use), `numContainersPooled` (slots rendered), `totalSize`, `containerLayoutEpoch`, `adaptiveRender`.

### Size resolution order (`getItemSize` in `src/utils/getItemSize.ts`)
1. `sizesKnown` (measured or fixed).
2. If `preferCachedSize`: last value in `sizes`.
3. `getFixedItemSize(item, index, type)`. The result plus the gap is written into `sizesKnown`, so it counts as known.
4. If `useAverageSize` and not `scrollingTo`: `averageSizes[type].avg`.
5. Last value in `sizes`.
6. While `scrollingTo`: `scrollingTo.averageSizeSnapshot[type]`, frozen when the scroll starts.
7. `estimatedItemSize + scrollAxisGap`. `LegendList` defaults `estimatedItemSize` to **100**.

Every path except 1 and 2 calls `setSize`, so asking for a size also registers it in `totalSize`. `getEstimatedItemSize` **no longer exists**; it was removed in c00f0469. `itemsAreEqual` has nothing to do with sizes. It only decides whether a changed data reference counts as a structural change (`checkStructuralDataChange`) and whether a mounted container gets new `containerItemData` (`syncMountedContainer`).

### Flow A: scroll event → items placed in containers
1. `updateScroll` ([scrolling](scrolling.md)) records history and sets `state.scrollBufferDirection`. It calls `updateAdaptiveRender`, then, once the offset has moved more than 2px since the last calculation (or a forcing condition holds), calls `state.triggerCalculateItemsInView({ doMVCP: scrollingTo !== undefined, scrollVelocity })`. A user jump larger than one viewport uses `drawDistanceMode: "visible-first"` inside `flushSync`, then `scheduleFullDrawDistancePrewarm`.
2. `calculateItemsInView` (inside `batchedUpdates`):
   - `getEffectiveDrawDistance` returns the `drawDistance` prop (default 250), capped at `INITIAL_DRAW_DISTANCE` = 50 before `readyToRender` or in `"visible-first"` mode.
   - **Directional buffer:** `drawDistance*0.5` behind and `drawDistance*1.5` ahead of `scrollBufferDirection`. At the top with no direction, the buffer leans forward.
   - **Velocity projection:** both buffered edges shift toward the scroll direction by `min(top,bottom buffer) * min(1, |v|/4 px/ms)`. A `renderRangeProjection` timeout (100ms) clears `scrollHistory` and recalculates once scrolling settles.
   - **Fast exit:** if `scrollForNextCalculateItemsInView` still contains the buffered window, only viewability and `onFirstVisibleItemChanged` are updated, and the pass returns. On web this exit is skipped while MVCP is active.
   - `prepareMVCP` (if `doMVCP`). On `dataChanged`, `resetLayoutCachesForDataChange` runs.
   - `updateItemPositions(startIndex = positionRecalculationStartIndex ?? startBuffered ?? 0)`. If MVCP moved the scroll, the range is recomputed.
   - The window loop walks backward from the previous `startBufferedId` to find the first item whose bottom is past the buffered top. For grids it snaps back to column 1. It then walks forward to fill `startBuffered/endBuffered`, the no-buffer range, and the next cache bounds.
   - **Allocation:** collect indices in the buffered range with no container, plus `alwaysRender` indices, plus `scrollTargetPinnedRange` (set by `scrollTo`), plus active sticky headers. `findAvailableContainers` returns `{containerIndex, itemIndex, itemType}`. For each allocation: write `containerItemMetadata` **first**, bump `containerItemGenerations[c]`, set `containerLayoutReady` (only with `experimental_hideItemsUntilMeasured`), then `containerItemKey/Index/Data`. Grow `numContainers`, and `numContainersPooled` through `getExpandedContainerPoolSize` (1.5×).
   - `handleStickyRecycling` frees sticky or pinned containers that are far away. On data change, containers whose key disappeared are cleared to `POSITION_OUT_OF_VIEW`.
   - Every mounted container goes through `syncMountedContainer`, which writes position (minus `scrollAdjustPending`, RTL-mapped), column, span, index, and data only when it changed.
   - Changed container ids go to `scheduleContainerLayout` (new arch and web).
   - The first time `checkAllSizesKnown` passes for the visible or buffered range, `setDidLayout` and `handleInitialScrollLayoutReady` run.
   - `updateViewableItems`, then the sticky header change callback.
3. `ContainerSlot` subscribes to `containerItemKey${id}` and renders `Container`. `Container` calls `getRenderedItem(key, id)` → `renderItem`.

### Flow B: item measured → sizes, positions, total size
1. New arch (`IsNewArchitecture`) and web: every `Container` commit runs a layout effect in `useContainerMeasurement` that calls `scheduleContainerLayout(ctx, id)`. This merges ids into `ctx.pendingContainerIds` (`null` = all) and bumps `containerLayoutEpoch` once per commit. `ContainerLayoutCoordinator` is the parent, so its layout effect runs after all child effects. It calls `getContainerLayoutEffectScope` → `measureContainersInLayoutEffect`.
   - Native (`.native.ts`): `view.measure` with a per-container **generation token**, so late callbacks are dropped. Fixed-size items whose `sizesKnown` already matches skip measurement.
   - Web: `getBoundingClientRect`, recorded with `setContainerLayoutBaseline` so the following ResizeObserver delivery reads as a confirmation, not a new change.
   - Old arch: `onLayout` → `processContainerLayout` → `updateItemSizes`. A 16ms timeout replays the last size because RN may not fire `onLayout` for an equal-sized recycled view. Web ResizeObserver and imperative sizes go through `batchItemSizeUpdates`. Web shrinks of more than 1px are confirmed one frame later unless MVCP is active.
2. `updateItemSizesBatch` drops measurements whose container no longer holds `itemKey`. For each remaining one, `applyItemSize` → `updateOneItemSize`:
   - Native deltas within `NATIVE_LAYOUT_MEASUREMENT_EPSILON` (1/PixelRatio + 0.01) are ignored.
   - Sizes are rounded (`Math.round` on web, `roundSize` on native), stored in `sizesKnown`, and folded into `averageSizes[type]`.
   - `setSize` → `addTotalSize(diff)`.
3. A nonzero diff lowers `positionRecalculationStartIndex` to the item's index, fires `onItemSizeChanged`, and marks `needsRecalculate` if the item is in the buffered range or mounted.
4. After the batch: `set$("totalSize")` once. Then `flushItemSizeUpdates` runs `runOrScheduleMVCPRecalculate` → `calculateItemsInView({ doMVCP: true, mvcpAdjustmentSource: "item-size" })`, deferred to one RAF on web while `mvcpAnchorLock` is set. That recomputes positions from the invalidated index with `forceFullUpdate` and re-syncs containers. `maybeUpdateAnchoredEndSpace` and `doMaintainScrollAtEnd` follow ([scroll-stability](scroll-stability.md)).

### Positions (`updateItemPositions`)
A linear sweep from `startIndex`. Single-column: `top += size`. Grid (`numColumns > 1`): row-major. The row height is the max item size in the row. `overrideItemLayout` can set `span` (clamped to 1..numColumns), and an item that doesn't fit wraps to a new row. `prepareColumnStartState` rewinds a mid-list start to its row start. There is **no masonry**: columns are uniform rows, not independent stacks. With non-zero velocity (or a web jump larger than the viewport) and no forced update, the sweep stops about `numColumns + 10` items past `scrollBottomBuffered + 1000`. `totalSize` is recomputed by `updateTotalSize` only if the sweep did not break early.

### Containers and recycling
- **Initial count** (`doInitialAllocateContainers`): `ceil((scrollLength + 2*drawDistance) / avgSize * numColumns)`. `avgSize` is the average `getFixedItemSize` over the first 20 items, or `estimatedItemSize`. The rendered pool `numContainersPooled` = `getInitialContainerPoolSize`: roughly 3× the count, capped at count+64, at least min(data,32), at most max(data,count). Slots beyond `numContainers` render `null` until assigned. This was made lazy in 09cbb79b.
- **Physical slots are always reused**, whatever `recycleItems` is. `findAvailableContainers` allocates the whole batch at once:
  - Candidates are unassigned or pending-removal slots (infinite distance), then assigned slots **outside** the buffered range, farthest first.
  - Matching runs in passes over the whole batch: exact `getItemType` match, then untyped slot, then any type (cross-type retype, normal pool only). Sticky slots stay in their own pool and never cross-type.
  - The pool grows only when no candidate is left. In dev a warning blames a too-large `estimatedItemSize`.
  - Finally, slots are reordered within each type group so React child order follows render priority. When scrolling up, or for initial `viewPosition: 1`, the order is reversed (176afe2c) so images ahead of the scroll start loading first.
- **What `recycleItems` actually does:** `Container` renders its `PositionComponent` with `key={recycleItems ? undefined : itemKey}`. With `false` (the default, with a dev warning when omitted), a reassigned slot remounts the item subtree. With `true`, the subtree stays mounted and receives a new item, so item-local state must be reset with `useRecyclingState`/`useRecyclingEffect` (`src/state/ContextContainer.ts`). With `recycleItems` and a custom `positionComponentInternal` (Reanimated layout animations), assigned slots are never stolen, because the outgoing view may still be animating.
- `containerItemMetadata` (per slot) snapshots `data`, `dataChangeEpoch`, `itemIndex`, `itemData`, `itemType`, and the resolved `getFixedItemSize`. `getRenderedItem` renders from that snapshot when the epoch is stale, so a retained slot cannot render a just-prepended item before signals sync (6d423f31).

### alwaysRender, adaptive render, viewability
- `alwaysRender {top, bottom, indices, keys}` → `getAlwaysRenderIndices`, memoized in `LegendList.tsx`. It also adds every index from `anchoredEndSpace.anchorIndex` to the end. Pinned containers live in `stickyContainerPool`, which protects them from reuse even when they are not sticky.
- `experimental_adaptiveRender`: `updateAdaptiveRender` switches the `adaptiveRender` signal to `"light"` when |velocity| > `enterVelocity` (3 native / 6 web, px/ms) or on a large jump. It switches back to `"normal"` once below `exitVelocity` (1 / 3), after `exitDelay` (250ms). Items read it with `useAdaptiveRender`/`useAdaptiveRenderChange`. The library itself renders nothing differently.
- Viewability: `setupViewability` merges `viewabilityConfig` + `onViewableItemsChanged` into `viewabilityConfigCallbackPairs` (default `viewAreaCoveragePercentThreshold: 0`). `updateViewableItems` runs over `startNoBuffer..endNoBuffer`, computing coverage from `positions` and `sizes` (current sizes, not only known ones) minus top padding/header. `minimumViewTime` defers through an unkeyed `scheduledWork.timeout`. Per-container hooks `useViewability`/`useViewabilityAmount` read `ctx.mapViewability*`.

## Key types and entry points
| Symbol | File | Role |
|---|---|---|
| `calculateItemsInView(ctx, {dataChanged, doMVCP, forceFullItemPositions, drawDistanceMode, scrollVelocity, mvcpAdjustmentSource})` | `src/core/calculateItemsInView.ts` | The main pass. Usually called as `state.triggerCalculateItemsInView`. |
| `updateItemPositions` | `src/core/updateItemPositions.ts` | Position, column, and span sweep, plus `indexByKey` rebuild after data changes. |
| `updateItemSizes` / `updateItemSizesBatch` / `batchItemSizeUpdates` / `updateOneItemSize` | `src/core/updateItemSizes.ts` | Measurement ingestion and averages. |
| `getItemSize`, `getKnownOrFixedItemSize` | `src/utils/getItemSize.ts` | Size resolution chain. |
| `setSize`, `addTotalSize`, `updateTotalSize` | `src/core/` | Keep `sizes` and `totalSize` consistent. |
| `findAvailableContainers` | `src/utils/findAvailableContainers.ts` | Batch container allocation. |
| `syncMountedContainer` | `src/core/syncMountedContainer.ts` | Push one slot's position, column, and data to signals. |
| `scheduleContainerLayout`, `getContainerLayoutEffectScope` | `src/core/scheduleContainerLayout.ts` | Coalesce post-commit measurement scope. |
| `measureContainersInLayoutEffect` | `src/core/measureContainersInLayoutEffect(.native).ts` | One measurement pass per commit (new arch / web). |
| `doInitialAllocateContainers`, `checkResetContainers` | `src/core/` | First allocation; data or `numColumns` reset. |
| `checkStructuralDataChange` | `src/core/checkStructuralDataChange.ts` | Decides if a new `data` reference really changed (keys plus `itemsAreEqual`). Caches per-index results in `pendingDataComparison`. |
| `ScheduledWork` | `src/core/ScheduledWork.ts` | Keyed, cancellable microtask/timeout/RAF work, disposed on unmount. New keys go in `ScheduledWorkKey`. |
| `updateViewableItems`, `setupViewability` | `src/core/viewability.ts` | Viewability. |

## Invariants and gotchas
- **`sizes` vs `sizesKnown`.** `totalSize` equals the sum of `sizes` (maintained by diff in `setSize`) and is recomputed exactly as `positions[last] + size(last)` by `updateTotalSize`. Never write `sizes` directly. Use `setSize` or `updateOneItemSize`, or `totalSize` drifts.
- **Sizes include `scrollAxisGap`** (`columnWrapperStyle` row/column gap). A `getFixedItemSize` result does not, so comparisons must add `ctx.scrollAxisGap` (see `canSkipMeasurement` in the native measurer; 59805d0a).
- **Positions are in item space.** `calculateItemsInView` converts scroll with `scroll + scrollAdjustPending - (stylePaddingTop + alignItemsAtEndPadding + headerSize)`. `syncMountedContainer` subtracts `scrollAdjustPending` again when publishing. Viewability uses its own top-pad subtraction.
- **After a size change, all later positions must be recomputed.** Any `positionRecalculationStartIndex` forces `forceFullUpdate`, which disables the velocity early break. Otherwise the suffix stays in the old coordinate space (783a889d).
- **Metadata before signals.** Always set `state.containerItemMetadata` before `set$(containerItemKey…)`. A reactive render can happen as soon as the key signal changes (6d423f31).
- **Measurements can be stale.** Check that `containerItemKey${containerId}` still equals the measured key (as `updateItemSizesBatch` does), and on native the generation token too. Imperative `updateItemSizes` without `containerId` skips the ownership check.
- **Bump `containerItemGenerations[c]` on every reassignment or removal.** Native measure callbacks and the hide-until-measured reveal depend on it.
- **Recycled slot ≠ same item.** With `recycleItems`, a mounted item component can receive a different item. `Container`'s `renderedItemInfo` memo depends on `[itemKey, data, extraData]`.
- **`keyExtractor` is always defined in production.** `LegendList` defaults it to the index string and wraps it with `useWrapIfItem`, which returns `undefined` for an undefined item. The `!keyExtractor` branches only run in hand-built test states. With index keys, any insertion shifts every key, so measured sizes are attributed by index, not by identity.
- **Duplicate keys** cause gaps and missing items. In dev, `updateItemPositions` logs "Detected overlapping key".
- **`checkAllSizesKnown` returns false when no mounted container is in range.** The first layout waits until a container is assigned and measured.
- **Multi-column lists never reuse positions** (`positionsAreCurrent = !hasColumns`). After a size change they recompute from the invalidated row with `optimizeForVisibleWindow`. Changing `numColumns` (or the gap) clears `sizes`, `sizesKnown`, and averages in `checkResetContainers`.
- **Averages ignore size 0** (conditionally empty rows) and freeze during `scrollTo` through `averageSizeSnapshot` (69d7d616). Live averages would shift the target while the scroll is in flight.
- **Viewability config id.** `maybeUpdateViewabilityCallback` keys callbacks by `containerId + configId`, and `useViewability` uses `containerId + (configId ?? "")`. **Inferred:** a `viewabilityConfig` with no `id` never reaches `useViewability` (`0 + undefined` is `NaN`). The dev warning in `ContextContainer.ts` asks for an id.
- `estimatedItemSize` JSDoc in `src/types.base.ts` says `@default undefined`, but `LegendList.tsx` defaults it to 100.

## Design decisions
| Decision | Reason / evidence |
|---|---|
| Single-column positions are reused when nothing changed (`positionsAreCurrent`). Any size or data change clears it (`setSize`, `updateOneItemSize`, fixed-size resolution in `getKnownOrFixedSize`, `resetLayoutCachesForDataChange`). Reuse is disabled for columns, `snapToIndices`, `scrollingTo`, a pending invalidation, or when MVCP needs refreshed averages. | Expanding the range alone doesn't move single-column items. The 200-row swap mount went from 1,200 to 400 position visits (301a9659). `doInitialAllocateContainers` also reuses positions from the first render instead of recomputing with `dataChanged: true`. |
| Measurement is one parent pass per commit, not per-row `onLayout` (new arch / web). | Recycled rows recalculated against partially updated sizes. Batching produces one position update per list (bbad389e, 5013ed25, ddf1ca6e). |
| Allocate the whole batch at once, type-first, and recycle before growing. | Greedy per-item assignment let early requests steal exact-type slots. Long-tail types grew the pool without bound (b7a24dd3, 0675cb42, a7adc9de). |
| Distant assigned containers are reused farthest-first. | Avoids stealing a container that is about to scroll back in (comment in `findAvailableContainers`). |
| Directional buffer, 0.5× behind and 1.5× ahead, and direction survives the settle. | Render ahead of motion. Clearing velocity on settle used to flip the buffer backward (2bd8e539). |
| Velocity-projected window. A speed-based `scrollExtra` exists but is disabled. | The old `scrollExtra` caused blank areas (comment in `calculateItemsInView`). The projection is bounded by the trailing buffer and settles after 100ms. |
| Draw distance starts at 50 and is prewarmed to full a frame later. Large jumps use `"visible-first"`. | Faster first render and faster jump response (a85e6ef4, e29d733d, 698d8efa, 47c3454a). |
| Lazy pooled slots (`numContainersPooled` > `numContainers`, empty slots render `null`). | Growing the pool doesn't remount siblings, and unused slots cost little (09cbb79b). |
| `getEstimatedItemSize` removed; per-type averages replace it. | c00f0469. `getItemType` drives both pooling and averages. |
| Native sub-pixel noise filter. | Fabric re-measures within 1 physical pixel. Feeding those values back caused position jitter. **Inferred** from `layoutMeasurement.ts`. |
| `experimental_hideItemsUntilMeasured` hides newly assigned unknown-size rows until measured. | Prepended rows otherwise flash at provisional recycled positions (d59c44c9). |

## Where to change things
| To… | Start at |
|---|---|
| Change how far ahead or behind items render | `getEffectiveDrawDistance`; the buffer split and `getProjectedBufferAdjustment` in `calculateItemsInView.ts` |
| Change which items count as visible or buffered | Window loop and `trackVisibleRange` in `calculateItemsInView.ts` |
| Change container reuse policy or type matching | `findAvailableContainers` (+ `__tests__/utils/findAvailableContainers.test.ts`) |
| Change initial or expanded pool sizes | `src/utils/containerPool.ts`, `doInitialAllocateContainers` |
| Change size estimation | `getItemSize` (resolution chain), `updateOneItemSize` (averages) |
| Change grid / spanning layout | `updateItemPositions` (column loop), `prepareColumnStartState`, `updateTotalSize` (last row) |
| Make a new state change invalidate positions | Set `positionRecalculationStartIndex` (min index) and/or `positionsAreCurrent = false`; clear `scrollForNextCalculateItemsInView` |
| Add a new per-container signal | `set$` in the allocation and removal blocks of `calculateItemsInView` and in `syncMountedContainer`; type it in `src/state/state.tsx` |
| Change measurement timing or batching | `scheduleContainerLayout`, `ContainerLayoutCoordinator`, `measureContainersInLayoutEffect(.native)`, `useContainerMeasurement` |
| React to size changes (scroll correction) | `flushItemSizeUpdates` / `runOrScheduleMVCPRecalculate` → [scroll-stability](scroll-stability.md) |
| Pin extra items mounted | `getAlwaysRenderIndices`, or `scrollTargetPinnedRange` (set in `scrollTo.ts`) |
| Change viewability math or callbacks | `computeViewability` / `updateViewableItemsWithConfig` in `viewability.ts`; hooks in `src/state/ContextContainer.ts` |
| Add deferred work | Add a key to `ScheduledWorkKey`, then use `state.scheduledWork.frame/timeout/microtask` |
| Decide whether new `data` triggers a reset | `checkStructuralDataChange`; wiring in `LegendList.tsx` (`didDataChangeLocal`, `checkResetContainers`) |

## Tests
- Core: `__tests__/core/calculateItemsInView.test.ts`, `updateItemPositions.test.ts` (includes position-reuse cases from 301a9659), `updateItemSizes.test.ts`, `updateItemSize.test.ts`, `updateTotalSize.test.ts`, `doInitialAllocateContainers.test.ts`, `checkResetContainers.test.ts`, `checkStructuralDataChange.test.ts`, `syncMountedContainer.test.ts`, `scheduleContainerLayout.test.ts`, `measureContainersInLayoutEffect.{native,web}.test.ts`, `viewability.test.ts`, `updateAdaptiveRender.test.ts`, `ScheduledWork.test.ts`, `handleLayout.test.ts`.
- Utils: `__tests__/utils/{findAvailableContainers,containerPool,getItemSize,getEffectiveDrawDistance,getAlwaysRenderIndices,getId,checkAllSizesKnown,getRenderedItem,useRafCoalescer}.test.ts(x)`.
- Components: `__tests__/components/Containers.layoutScheduler.native.test.tsx`, `Containers.{native,web}.test.*`, `ContainerSlot.test.tsx`, `LegendList.numColumnsReset.test.tsx`, `LegendList.totalSizeNotifications.test.tsx`.
- Pattern: unit tests build state with `createMockContext` (`__tests__/__mocks__/createMockContext.ts`), seed `idCache`/`indexByKey`/`positions`/`sizes` directly (helpers in `__tests__/helpers/layoutArrays.ts`), set `numContainers`, then call the core function and assert on `ctx.values` signals.
- Example run: `bun test __tests__/core/updateItemPositions.test.ts __tests__/utils/findAvailableContainers.test.ts __tests__/core/calculateItemsInView.test.ts` → 191 pass, 0 fail (~0.2s at 3d80d167).
- Visible behavior: `example-web` fixtures (e.g. the slow-image directional allocation fixture added in 176afe2c); run with `cd example-web && bun run dev:fixtures`.
