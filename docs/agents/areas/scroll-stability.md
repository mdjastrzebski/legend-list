<!-- index-codebase: generated from 3d80d167 on 2026-10-03. Code wins when it disagrees with this doc. -->
# Scroll stability

**Purpose:** keeps what the user is looking at from jumping when sizes or data change: maintainVisibleContentPosition (MVCP), following the end of the list (`maintainScrollAtEnd`), bottom-aligned short lists (`alignItemsAtEnd`), trailing space that keeps a row at the top (`anchoredEndSpace`), and the shared "scroll adjust" channel that moves the native or DOM scroll offset without an imperative `scrollTo`. Not covered here: `scrollTo*` / `onScroll` / thresholds ([scrolling](scrolling.md)), initial scroll ([initial-scroll](initial-scroll.md)), measuring and positioning items ([virtualization](virtualization.md)).
**Paths:** `src/core/mvcp.ts`, `src/utils/requestAdjust.ts`, `src/core/ScrollAdjustHandler.ts`, `src/components/ScrollAdjust.tsx` / `ScrollAdjust.native.tsx`, `src/core/doMaintainScrollAtEnd.ts`, `src/core/updateAnchoredEndSpace.ts`, `src/core/updateContentMetrics.ts`, `src/core/updateContentMetricsState.ts`, `src/core/updateContentInsetEndAdjustment.ts`, `src/core/endOfContentTarget.ts`, `src/core/getStartOffsetAdjustment.ts`, `src/state/getContentInsetEnd.ts`, `src/utils/{normalizeMaintainScrollAtEnd,normalizeMaintainVisibleContentPosition,hasActiveMVCPAnchorLock,isInMVCPActiveMode(.native),setPaddingTop}.ts`, `plans/6-anchored-end-space.md`
**Depends on:** [state](state.md), [virtualization](virtualization.md) (`calculateItemsInView`, `updateItemPositions`, `updateItemSizes`) · **Used by:** [scrolling](scrolling.md) (`scrollToEnd`, `finishScrollTo`), [initial-scroll](initial-scroll.md), [integrations](integrations.md) (keyboard)

## Glossary

| Term | Meaning |
|---|---|
| MVCP | maintainVisibleContentPosition. Two independent switches: `size` (compensate for size changes and scroll targets, **on by default**) and `data` (compensate when the `data` array changes, e.g. prepend, **off by default**). |
| anchor | The item whose position MVCP keeps fixed. Its position delta becomes the scroll adjustment. |
| scroll adjust | A relative scroll correction published through the `scrollAdjust` signal. `ScrollAdjust` turns it into a native or DOM scroll move. |
| anchor lock (web) | `state.mvcpAnchorLock`. Keeps the same anchor across the frames after a data change while measurements settle. |
| pending native MVCP adjust | `state.pendingNativeMVCPAdjust`. A data-change correction that native will partly do itself by clamping at the end of content. |
| maintain-at-end state | `state.maintainingScrollAtEnd`: `"pending-instant" \| "pending-animated" \| "instant" \| "animated" \| undefined`. |
| end threshold | Signal `isWithinMaintainScrollAtEndThreshold`, set by `checkAtBottom` (`src/utils/checkAtBottom.ts`): distance from end ≤ `maintainScrollAtEndThreshold` (default `0.1`) × viewport, or content shorter than the viewport, or a maintain is pending or active. |

## Config defaults

| Prop | Normalizer | Result |
|---|---|---|
| `maintainVisibleContentPosition` omitted | `normalizeMaintainVisibleContentPosition` | `{ data: false, size: true }` |
| `true` / `false` | same | `{ data: true, size: true }` / `{ data: false, size: false }` |
| `{ ... }` | same | `data ?? false`, `size ?? true`, plus `shouldRestorePosition` |
| `maintainScrollAtEnd: true` | `normalizeMaintainScrollAtEnd` | every trigger on (`onDataChange`, `onLayout`, `onItemLayout`, `onFooterLayout`), `animated: false` |
| `{ animated }` without `on` | same | every trigger on |
| `{ on: {...} }` | same | only the keys set to `true` are on |
| `maintainScrollAtEndThreshold` | - | `0.1` (fraction of the viewport) |

Commit `bbca7164` made `size` MVCP always on. Before that, the prop had to be set to keep items steady when sizes changed above the viewport; the prop now only opts in to data-change anchoring. Commit `15a7bda3` added `size: false` as a way to turn it off.

## How it works

### The adjust pipeline (all platforms)

Every correction goes through `requestAdjust(ctx, diff, source?)` in `src/utils/requestAdjust.ts`:
1. It ignores corrections of 0.1px or less, adds `diff` to `state.scroll` right away (JS state now holds the target offset), and clears `scrollForNextCalculateItemsInView`.
2. If the list is not `readyToRender`, it bumps `adjustingFromInitialMount` and applies the correction on the next RAF.
3. Otherwise `state.scrollAdjustHandler.requestAdjust(diff)` (`ScrollAdjustHandler`) adds to `appliedAdjust` and publishes signal `scrollAdjust`. During an **animated** non-initial `scrollTo` on platforms where `PlatformAdjustBreaksScroll` is true (web, Android: `src/platform/Platform(.native).ts`), it adds to `pendingAdjust` / `scrollAdjustPending` instead. `finishScrollTo` → `commitPendingAdjust` then recomputes the target's current offset and applies the difference.
4. On native, for any source other than `"item-size"`, it sets `state.ignoreScrollFromMVCP = { lt | gt: halfway threshold }` for 100ms (250ms with the Android old-arch workaround). `updateScroll` drops scroll events on the stale side of that threshold, which would otherwise undo the adjustment. Item-size corrections skip this guard because they keep the native coordinate space and can arrive during real momentum (`c2aa8725`).

### How the adjust reaches the scroll view

| Platform | Mechanism |
|---|---|
| Native (`ScrollAdjust.native.tsx`) | Renders a 0×0 absolutely positioned `View` as the **first child** of the scroll content, at `top = 10_000_000 + scrollAdjust + scrollAdjustUserOffset`. `ListComponent` passes RN `maintainVisibleContentPosition={{ minIndexForVisible: 0 }}` whenever `size` or `data` is on. **Inferred:** RN's native MVCP picks the first subview below the current offset as its anchor, which is always this far-away view, so moving it by Δ makes the platform shift `contentOffset` by Δ in the same layout commit. That avoids a JS `scrollTo` round trip. The 10M bias keeps the offset from going negative (origin: `cf942815`, "a simpler way to do maintainVisibleContentPosition"). |
| Native, Android old arch, `"data"` source, `state.scroll <= diff` | `requestAdjust` calls `doScrollTo(offset: state.scroll)` instead (`9440891d`). Covered by `__tests__/utils/requestAdjust.oldarch.cases.ts`. |
| Web (`ScrollAdjust.tsx`) | Renders nothing. It listens to `scrollAdjust` / `scrollAdjustUserOffset` and calls `el.scrollBy` with `intendedScroll - liveDOMScroll`, using `state.scroll` as the intended offset. It reconciles against the live DOM scroll so that browser clamping or native scroll anchoring is not applied twice (`f2d4543a`, #463). When the move would go past the current scroll range, it temporarily adds end padding (`(overshoot) * 2`) to the content node and restores it on the next RAF, but only if the padding it installed is still there (`57faa00e`, `aeebba57`, `e2fc84b5`). It skips the move when a shared or external scroller is outside the list (`isScrollInRange() === false`). |

`scrollAdjustUserOffset` is a separate channel for the public `setVisibleContentAnchorOffset` imperative method (`createImperativeHandle.ts`).

### Traced flow: an item above the viewport grows (size MVCP)

1. A container measures a new size → `updateItemSizes` / `updateItemSizesBatch` (`src/core/updateItemSizes.ts`) → `applyItemSize` → `updateOneItemSize` → `setSize` → `addTotalSize`. It records `positionRecalculationStartIndex = min(index)` and sets `needsRecalculate` if the item is in the buffered range or mounted in a container. Native layout noise is ignored (`isNativeLayoutNoise`).
2. `flushItemSizeUpdates` publishes `totalSize`, then calls `runOrScheduleMVCPRecalculate`. On native this calls `calculateItemsInView(ctx, { doMVCP: true, mvcpAdjustmentSource: "item-size" })` right away. On web it does the same, unless an anchor lock is active; then it coalesces to one RAF (`scheduledWork` key `"mvcpRecalculate"`) to avoid oscillating corrections.
3. In `calculateItemsInView`, `prepareMVCP` runs **before** `updateItemPositions` and captures the anchor's old position. Anchor choice, in priority order:
   - web anchor lock (if no scroll target);
   - `state.scrollingTo.index` (MVCP keeps an in-flight scroll target steady; on old arch it skips the initial scroll, `f00bf085`);
   - first id in `state.idsInView` still present in `indexByKey`. Only for non-data passes, after `didContainersLayout`.
4. `updateItemPositions` recomputes positions from the invalidated index. Then `checkMVCP()` computes `newPosition - prevPosition` and calls `requestAdjust(ctx, diff, "item-size")`. If scroll changed, `calculateItemsInView` refreshes its local scroll range in the same pass (`eca611e0`).
5. `flushItemSizeUpdates` then calls `maybeUpdateAnchoredEndSpace` and, if `onItemLayout` is on and the size changed by more than 5px (or was the first measurement), `doMaintainScrollAtEnd`.

### Data changes (prepend, insert, remove)

`LegendList` sets `state.dataChangeNeedsScrollUpdate`. `checkResetContainers` calls `calculateItemsInView(ctx, { dataChanged: true, doMVCP: true })` and then, if `onDataChange` is on, `doMaintainScrollAtEnd`. In `prepareMVCP` with `dataChanged`:
- MVCP runs only if `maintainVisibleContentPosition.data` is on. With the default config, prepending does **not** keep content steady.
- It snapshots the old positions of all `idsInView`. After positions update, it uses the first snapshot item that still exists and passes `shouldRestorePosition(item, index, data)` as a fallback anchor. A web anchor lock or a scroll target takes priority if still valid.
- `calculateItemsInView` also protects the visible rows' containers from recycling during that pass (`protectedContainerKeys`).
- **Web:** `updateAnchorLock` keeps that anchor for up to `MVCP_ANCHOR_LOCK_TTL_MS = 300`ms. It releases the lock after 2 quiet passes (diff ≤ 0.1) (`d24a9d93`). Estimated sizes of prepended rows are corrected frame by frame against the same anchor. `isInMVCPActiveMode` (web: data change pending or lock active; native: data change pending only) forces `updateScroll` and `calculateItemsInView` to recalculate on every scroll instead of using the cached range.
- **Web:** if a `scrollToEnd` is queued but not yet active (`state.pendingScrollToEnd`), data MVCP is skipped so it doesn't instantly undo the animated end scroll (`6b851736`).
- **Native, content shrinks near the end:** `shouldQueueNativeMVCPAdjust` detects that the native scroll view will clamp the offset itself, because the distance to the end is smaller than the negative diff. It records `pendingNativeMVCPAdjust`, applies only the part the platform will not do (`maybeApplyPredictedNativeMVCPAdjust`), and `resolvePendingNativeMVCPAdjust` (called from `updateScroll`) watches native scroll events until the platform settles. It then sends any remainder or abandons the adjust if native moved the wrong way. While this is pending, later MVCP passes and `doMaintainScrollAtEnd` wait (`ed1cac94`).

### maintainScrollAtEnd (following the end)

Following is **not a sticky mode**. Each trigger checks the end threshold at that moment. Being near the end re-enables following; scrolling away disables it.

| Trigger | Caller |
|---|---|
| `dataChange` | `checkResetContainers`; also replayed by `updateScroll` after a pending native MVCP adjust resolves |
| `layout` (viewport resize) | `handleLayout` with `{ immediate: true }`. On web, a non-animated maintain runs synchronously inside the ResizeObserver callback, before paint. |
| `itemLayout` | `flushItemSizeUpdates` (size change > 5px or first measurement; first measurements count so prepended rows' growth is followed) |
| `footerLayout` | `ListComponent` footer `onLayout` |
| replay of pending | `setDidLayout`, `finishScrollTo`, the scroll-request promise in `doMaintainScrollAtEnd` |

`doMaintainScrollAtEnd` proceeds only after `didFinishInitialScroll`, while no other scroll request is waiting, and only if no `scrollingTo` is active (or the active one is a scroll-to-end). It sets `maintainingScrollAtEnd = "pending-*"`, waits one RAF (or runs right away for immediate web), then runs `ctx.scrollToEnd({ animated })` through the scroll request tracker (`runNowIfIdle`) and moves to the active state. When the promise resolves, it either replays a pending follow or calls `finishMaintainScrollAtEnd`. Triggers that arrive while a follow is in flight are merged via `pendingMaintainScrollAtEnd`.

**Cancellation by the user:**
- Native: `onScrollBeginDrag` (`LegendList.tsx`) → `isDragging = true` + `interruptMaintainScrollAtEnd` (also cancels the in-flight imperative scroll). `doMaintainScrollAtEnd` refuses to start while dragging. Native scroll offsets are deliberately **not** used to detect user intent, because native measurement and MVCP events arrive late (comment in `updateScroll`).
- Web: an upward scroll event (`newScroll < prevScroll`, not synthetic) in `updateScroll` → `finishMaintainScrollAtEnd`. A wheel event with negative delta (`ListComponentScrollView`) → `interruptMaintainScrollAtEnd` and pins the browser at its current offset. Inside the pending RAF, a web scroll since the request that also left the threshold cancels the follow.
- Either platform: a shared or external scroller scrolled outside the list (`isScrollInRange() === false`).

**MVCP interplay** (`a9a08415`): while `maintainingScrollAtEnd` is `pending-animated` or `animated` and the list is within the threshold, `prepareMVCP` skips its adjust so it doesn't fight the animation. Instant pins still get MVCP compensation.

### alignItemsAtEnd

`LegendList` chooses one of two implementations:
- **Padding mode** (`alignItemsAtEndPaddingEnabled`): vertical, `contentContainerStyle.minHeight` not set, data not empty. `getAlignItemsAtEndPadding` = `max(0, viewport - (header + footer + totalSize + paddingTop + paddingBottom) - contentInsetEnd)`. It is stored in signal `alignItemsAtEndPadding` and rendered by `AlignItemsAtEndSpacer` before the containers in `ListComponent`. It is also added to every item offset via `getStartOffsetAdjustment`, so scroll targets account for it. `updateContentMetricsState` recomputes it whenever header, footer, total size or insets change. During an animated maintain-at-end, it keeps the old padding as "animation runway" and lets `finishMaintainScrollAtEnd` shrink it and `requestAdjust` the difference afterwards. Padding mode subtracts the end inset, which is what makes it work with keyboard insets (`458862ff`).
- **Flex mode** (horizontal, or `minHeight` set, or empty data): `contentContainerStyle` gets `flexGrow: 1, justifyContent: "flex-end"`.

### anchoredEndSpace

`anchoredEndSpace = { anchorIndex, anchorOffset?, anchorMaxSize?, onSizeChanged?, onReady? }` adds trailing space so that the row at `anchorIndex` can sit `anchorOffset` from the viewport's start edge even when the content below it is shorter than the viewport. This is the "new chat turn scrolls to the top" pattern. `maybeUpdateAnchoredEndSpace` (`src/core/updateAnchoredEndSpace.ts`):
- size = `max(0, viewport - tailContent - anchorOffset)`. `tailContent` = known or fixed sizes from `anchorIndex` to the end, minus the last scroll-axis gap, plus footer and axis- and RTL-aware end padding (`87152b5c`). The anchor's contribution is capped by `anchorMaxSize`.
- While any tail size is unknown, it can only **shrink** toward the known-size bound, never grow, and `onReady` is withheld (`345b9e3a`, `63c440fa`).
- It is recomputed from a `LegendList` layout effect (data, anchor props, padding, axis, RTL), after each item-size batch, and after footer size changes (`d5a159f8`).
- The resolved size is part of the logical end inset: `getContentInsetEnd` returns `max(baseEndInset + contentInsetEndAdjustment, anchoredEndSpaceSize)`. The space and the inset overlap rather than add up.
- One layer renders it (`b71d9ed8`): owner `"list"` adds it to the content container's end padding (`ListComponent`). The keyboard integration sets `anchoredEndSpaceOwnerInternal="scroll"` and renders it as animated blank space. The `includeInEndInset` option in `plans/6-anchored-end-space.md` was later removed, so that plan is history, not the current API.
- `getAlwaysRenderIndices` keeps the anchor row mounted so it gets measured (`a975bd47`). Imperative `scrollToIndex` at or after the anchor waits until tail sizes are known (`isScrollToIndexReady` in `createImperativeHandle.ts`, `fa4cb2be`).
- **Single-column only.** The tail sum treats every item as its own row. `useDevChecks` warns once (`anchoredEndSpaceNumColumns`) when `numColumns > 1`. Row geometry was deliberately not added (`62d0ab73`, CHANGELOG 3.3.x).

### Other content-edge adjustments

| Change | Handler | Behavior |
|---|---|---|
| Header size (web only) | `setHeaderSize` in `updateContentMetrics.ts` | `requestAdjust(newSize - oldSize)` if `size` MVCP is on, initial scroll has finished, there is no scroll target, the viewport is past the header, and the header was already measured once (`f46e6dda`, #468). Native has no equivalent here (see open questions). |
| `contentInsetEndAdjustment` (web-only prop) | `updateContentInsetEndAdjustment` | If the list was within the end threshold, `requestAdjust(insetDiff)`. On web only for growth, since the browser clamps on shrink. It first retargets an active initial scroll-at-end. |
| `contentContainerStyle.paddingTop` changes (iOS) | `initializeStateVars` in `LegendList.tsx` + `setPaddingTop` | `requestAdjust(paddingDiff)`. When padding shrinks, total size is temporarily inflated for 16ms so the ScrollView doesn't clamp. |
| Keyboard insets | [integrations](integrations.md) | `reportContentInset` → `setContentInsetOverride` → `updateContentMetricsState`. Feeds `getContentInsetEnd`. |

## Invariants and gotchas

- **`state.scroll` is updated before the platform moves.** `requestAdjust` writes the target immediately. Code that reads `state.scroll` after an adjust sees the intended offset, not the observed one. Web `ScrollAdjust` depends on this (`intendedScroll = state.scroll`).
- **Always use `requestAdjust`.** Calling `scrollAdjustHandler.requestAdjust` or setting the `scrollAdjust` signal directly skips the `state.scroll` update, the stale-event guard and the pre-render deferral.
- `prepareMVCP` must run before `updateItemPositions` in the same `calculateItemsInView` pass, because it captures old positions. Adding a position update before it breaks MVCP silently.
- **Inferred:** on native, the adjust channel only works while RN's `maintainVisibleContentPosition` prop is set, which needs `size || data`. With `maintainVisibleContentPosition={false}`, any adjust (maintain-at-end padding changes, inset changes) updates JS state but may not move the native view.
- `PlatformAdjustBreaksScroll`: on web and Android, an adjust during an animated `scrollTo` would break the animation, so it is held in `scrollAdjustPending` until `finishScrollTo`.
- The type doc for `maintainVisibleContentPosition.data` says passing true "also sets the RN maintainVisibleContentPosition prop". In practice the default `size: true` already sets it (`ListComponent`).
- All epsilons are 0.1px (`MVCP_POSITION_EPSILON`, `requestAdjust`). The native end-clamp tolerance is 1px.
- Large user jumps (> one viewport, `updateScroll`) drop the anchor lock and the pending native adjust, and create `state.userScrollAnchorReset`. Item-size recalculation then runs **without** MVCP until the newly mounted rows have been measured (`a97482f7`).
- `doMaintainScrollAtEnd` sets `state.scroll = 0` when the content fits the viewport. That keeps `checkAtTop` and `checkAtBottom` correct.
- `checkAtBottom` reports "within threshold" while a maintain is pending or active, so the threshold check doesn't cancel an in-flight follow.
- Old architecture: `addTotalSize` holds a shrinking total in `pendingTotalSize` during initial scroll so the platform does not clamp, and MVCP skips the initial scroll target.

## Design decisions

- **Relative adjust through a layout-driven view instead of `scrollTo`** (native). It lands in the same commit as the content change, so no frame shows the jump. **Inferred** from `cf942815` and the bias comment.
- **Web reconciles against the live DOM offset** rather than sending blind deltas, because the browser's own clamping and anchoring had been applied twice (#463, `f2d4543a`).
- **Web anchor lock with TTL and quiet passes.** Web layout settles across several frames after a data change. Re-choosing the "first visible" anchor each frame made corrections oscillate (`d24a9d93`).
- **Native shrink prediction.** Native clamps at max scroll on its own. Requesting the full diff on top of that double-corrected (`ed1cac94`, `eca611e0`).
- **Item-size adjusts do not suppress scroll events** so momentum keeps going (`c2aa8725`).
- **Maintain-at-end is modelled as pending and active, instant and animated states** so MVCP can keep compensating during instant pins but stays out of animated ones (`a9a08415`).
- **Chat without `inverted`** (README "Chat UIs without inverted"). The recommended setup is `alignItemsAtEnd` + `initialScrollAtEnd` + `maintainScrollAtEnd` + `maintainVisibleContentPosition` (`data` for prepending history). See `example/screens/fixtures/chat-example.tsx` and `example-web/src/fixtures/ChatEndFollowExample.tsx`. Add `anchoredEndSpace` for "pin the latest turn to the top" AI-chat layouts (`example/screens/fixtures/anchored-end-space.tsx`).

## Where to change things

| To… | Start at |
|---|---|
| Change which item MVCP anchors to | `prepareMVCP` in `src/core/mvcp.ts` (anchor priority block + fallback loop) |
| Tune web anchor-lock lifetime | `MVCP_ANCHOR_LOCK_*` constants, `resolveAnchorLock` / `updateAnchorLock` in `mvcp.ts`; `runOrScheduleMVCPRecalculate` in `updateItemSizes.ts` |
| Fix native end-clamp after a data change | `shouldQueueNativeMVCPAdjust`, `maybeApplyPredictedNativeMVCPAdjust`, `resolvePendingNativeMVCPAdjust` in `mvcp.ts`; caller in `updateScroll.ts` |
| Change how an adjust reaches the scroll view | `requestAdjust.ts` → `ScrollAdjustHandler.ts` → `ScrollAdjust.tsx` (web) / `ScrollAdjust.native.tsx` |
| Adjusts during animated `scrollTo` | `ScrollAdjustHandler.requestAdjust` / `commitPendingAdjust`, `finishScrollTo.ts` |
| Add a maintain-at-end trigger or change cancellation | `normalizeMaintainScrollAtEnd.ts` (+ `MaintainScrollAtEndOnOptions` in `types.base.ts`), `doMaintainScrollAtEnd.ts`, drag handler in `LegendList.tsx`, `updateScroll.ts`, wheel handler in `ListComponentScrollView.tsx` |
| Change the end threshold logic | `checkAtBottom.ts` |
| Bottom alignment math | `getAlignItemsAtEndPadding` / `updateContentMetricsState` in `updateContentMetricsState.ts`; mode choice in `LegendList.tsx`; spacer in `ListComponent.tsx` |
| anchoredEndSpace math or readiness | `maybeUpdateAnchoredEndSpace`; rendering in `ListComponent.tsx` (list owner) / `src/integrations/keyboard.tsx` (scroll owner); inset merge in `getContentInsetEnd.ts` |
| Header, footer or inset driven adjustments | `updateContentMetrics.ts`, `updateContentInsetEndAdjustment.ts`, `setPaddingTop.ts` |
| Whether trailing space counts in an end target | `getEndAlignedViewOffset` / `isEndOfContentTarget` in `endOfContentTarget.ts` (shared with [initial-scroll](initial-scroll.md)) |

## Tests

| Behavior | Tests |
|---|---|
| Anchor choice, data and size MVCP, web lock, native clamp prediction | `__tests__/core/prepareMVCP.test.ts`, `__tests__/core/mvcp.test.ts` |
| Adjust channel | `__tests__/utils/requestAdjust.test.ts`, `__tests__/utils/requestAdjust.oldarch.test.ts`, `__tests__/core/ScrollAdjustHandler.test.ts`, `__tests__/components/ScrollAdjust.web.test.ts`, `__tests__/components/ScrollAdjust.native.test.tsx` |
| Maintain at end | `__tests__/core/doMaintainScrollAtEnd.test.ts`, `doMaintainScrollAtEnd.deferredRequest.test.ts`, `__tests__/utils/normalizeMaintainScrollAtEnd.test.ts`, `__tests__/utils/checkAtBottom.test.ts`, `__tests__/core/endAnchorResize.test.ts`, `__tests__/components/ChatEndFollow.fixture.test.ts` |
| Anchored space, content metrics, insets | `__tests__/core/updateAnchoredEndSpace.test.ts`, `__tests__/components/AnchoredEndSpace.web-usage.test.tsx`, `__tests__/core/contentMetrics.test.ts`, `__tests__/core/updateContentInsetEndAdjustment.test.ts` |
| Integration-level | `__tests__/components/LegendList.liveUpdates.integration.test.tsx`, `LegendList.dataRefresh.integration.test.tsx`, `__tests__/integrations/keyboard.behavior.test.tsx` |

- Several tests need a **fresh process** because they mock module constants: `requestAdjust.oldarch.test.ts` mocks `IsNewArchitecture: false`, and `ChatEndFollow.fixture.test.ts` isolates the web example's React 18. Each test spawns `bun test <*.cases.ts(x)>`. Follow this pattern when a new test needs `IsNewArchitecture` or a different React.
- Quick check: `bun test __tests__/core/mvcp.test.ts __tests__/core/prepareMVCP.test.ts __tests__/core/doMaintainScrollAtEnd.test.ts __tests__/utils/requestAdjust.oldarch.test.ts __tests__/core/updateAnchoredEndSpace.test.ts` (145 tests passed at 3d80d167, about 0.1s).
- Manual fixtures: web `example-web/src/fixtures/{MVCPTestExample,HeaderMvcpExample,PrependLargeItemsJumpExample,BidirectionalInfiniteListExample,ChatEndFollowExample,EndResizeExample,ChatFloatingComposerExample}.tsx`; native `example/screens/fixtures/{anchored-end-space,end-anchor-resize,chat-*,add-to-end,scroll-regression-qa}.tsx`.
- `bun run test:visualize` does **not** help with scroll offsets. `visualize-tests.js` runs the whole suite twice (plain and `--coverage`), regex-parses pass/fail counts, runtime and coverage, and writes a summary page to the tracked `test-visualization.html` (last committed 2025-07-21). It contains no scroll or offset data. Use semantic assertions on `state.scroll`, the `scrollAdjust` signal and `positions` instead.

## Open questions

- Native header size changes above the viewport: `setHeaderSize` compensates only on web. It is unverified whether native RN MVCP covers this case or it is simply not handled.
- The native anchoring trick (the 10M-offset view acting as RN's MVCP anchor) is inferred from the code. No commit or comment states the mechanism.
