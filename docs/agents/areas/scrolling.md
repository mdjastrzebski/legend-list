<!-- index-codebase: generated from 3d80d167 on 2026-10-03. Code wins when it disagrees with this doc. -->
# Scrolling

**Purpose:** turns platform scroll events into list state (`state.scroll`, velocity, edge flags, `onStartReached`/`onEndReached`) and runs imperative scrolls (`scrollToIndex`/`scrollToOffset`/`scrollToItem`/`scrollToEnd`/`scrollIndexIntoView`) from request to resolved promise. It does **not** own the first scroll after mount (`bootstrapInitialScroll`, `initialScroll*`, `finishInitialScroll`, see [initial-scroll](initial-scroll.md)), MVCP / `maintainScrollAtEnd` / `ScrollAdjust` (see [scroll-stability](scroll-stability.md)), or deciding which items render (see [virtualization](virtualization.md)).
**Paths:** `src/core/onScroll.ts`, `src/core/updateScroll.ts`, `src/core/deferredPublicOnScroll.ts`, `src/core/scrollTo.ts`, `src/core/scrollToIndex.ts`, `src/core/scrollToEnd.ts`, `src/core/doScrollTo.ts` (web), `src/core/doScrollTo.native.ts`, `src/core/finishScrollTo.ts`, `src/core/checkFinishedScroll.ts`, `src/core/cancelImperativeScroll.ts`, `src/core/scrollRequestTracker.ts`, `src/core/recalculateSettledScroll.ts`, `src/core/calculateOffset*.ts`, `src/core/clampScrollOffset.ts`, `src/core/handleLayout.ts`, `src/utils/checkThreshold(s).ts`, `src/utils/checkAtTop.ts`, `src/utils/checkAtBottom.ts`, `src/utils/edgeReachedGate.ts`, `src/utils/getScrollVelocity.ts`, `src/utils/throttledOnScroll.ts`, `src/utils/updateSnapToOffsets.ts`, scroll methods in `src/utils/createImperativeHandle.ts`.
**Depends on:** [state](state.md) (`state`, `set$`/`peek$`, `ScheduledWork`), [virtualization](virtualization.md) (`triggerCalculateItemsInView`, `positions`, `getItemSize`), [scroll-stability](scroll-stability.md) (`ScrollAdjustHandler`, `mvcp.ts`, `doMaintainScrollAtEnd.ts`), [rendering-and-platform](rendering-and-platform.md) (`ListComponentScrollView*.tsx`, `Platform`). · **Used by:** [public-api](public-api.md) (ref methods, `onScroll`, reached callbacks), [initial-scroll](initial-scroll.md) (reuses `scrollTo`/`finishScrollTo`/`checkFinishedScroll`), [integrations](integrations.md).

## How it works

### Scroll event to state (user scroll)
1. **Platform event.** Native: the RN `ScrollView` `onScroll`, wrapped by `useStickyScrollHandler`, calls `fns.onScroll` in `src/components/LegendList.tsx`. Web: `ListComponentScrollView.tsx` listens to DOM `scroll` on the element, `window`, or the `scrollElement` owner. It builds an RN-shaped event and coalesces events to one per animation frame (`useRafCoalescer`). It flushes immediately while `state.scrollingTo`, an initial scroll, or MVCP active mode is in progress, because completion checks need fresh offsets.
2. **`onScroll(ctx, event)`** (`src/core/onScroll.ts`) returns early if `setScrollProcessingEnabled(false)` was called or contentSize is 0×0. It records `nativeContentInset` changes and converts horizontal RTL offsets to logical ones. If an imperative target exists and the event is past the clamped max offset, it re-issues a clamped `scrollTo(... noScrollingTo, forceScroll)` and stops. Otherwise it sets `scrollPending` and calls `updateScroll`. If `scrollingTo` is set it then calls `checkFinishedScroll`, and finally the public `onScroll`. On web, the public `onScroll` is held back during a bootstrap initial scroll and released later by `releaseDeferredPublicOnScroll` with the resolved offset (commit cc31a95d).
3. **`updateScroll`** (`src/core/updateScroll.ts`):
   - Pushes into `scrollHistory`, capped at 5 entries. It skips the push while `scrollingTo` is set or while `scrollAdjust` is changing, since those samples are synthetic.
   - Drops stale pre-MVCP events through `ignoreScrollFromMVCP`.
   - Sets `scroll`/`scrollPrev`/`scrollTime` and computes `scrollBufferDirection`.
   - On web, cancels end-following when the user scrolls up.
   - Opens the reached-edge gate for this gesture (`beginReachedEdgeUserScroll`) and updates adaptive render.
   - Runs `triggerCalculateItemsInView` and then `checkThresholds`, but only if the scroll moved >2px since the last calculation, or one of several forcing conditions holds (MVCP active, inset change, edge allowed, etc.).
   - Treats a jump larger than one viewport (no `scrollingTo`) as a "large user scroll jump". It drops MVCP anchor locks, runs the calculation under `flushSync` with `drawDistanceMode: "visible-first"`, then schedules a full draw-distance prewarm (commits b6c3f7c6, 698d8efa).
4. **Velocity**: `getScrollVelocity` returns an exponentially weighted (200ms half-life) average over the newest same-direction samples within 1s. It feeds `calculateItemsInView` buffer prediction, adaptive render and `getState().scrollVelocity`.

### Imperative scroll lifecycle (traced: `ref.scrollToIndex({ index, viewPosition })`)
1. **Request + ownership.** `createImperativeHandle.scrollToIndex` calls `runScrollWithPromise`:
   - `getScrollRequestTracker(ctx).start(resolve)` bumps a token, resolves the previous request's promise (`settlePendingImperativeScroll`), stores `pendingScrollResolve`, and ends `maintainScrollAtEnd` following.
   - `supersedeInitialScroll` cancels an unfinished initial scroll so it can't land after the explicit one (commit 07b3edee).
2. **Readiness wait.** If the list is settling after a data change (`didDataChange`, `didColumnsChange`, a scheduled `mvcpRecalculate`, `ignoreScrollFromMVCP`), or the target is in the `anchoredEndSpace` tail and its sizes are unknown, `runWhenReady` polls once per frame under key `imperativeScrollReady`. It runs after 2 stable frames or 800ms, and each tick checks `isCurrent(token)`.
3. **Target precompute.** `scrollToIndex` (`src/core/scrollToIndex.ts`) clamps the index and reads `positions[index]` through `calculateOffsetForIndex`. It defaults `viewPosition` to 1 for the last item and 0 otherwise, and defaults `animated` to true. `scrollTo` (`src/core/scrollTo.ts`) then:
   - cancels any previous completion work (`cancelScrollCompletionChecks`);
   - applies `viewOffset`, header/start adjustment and `viewPosition`/`viewPositionFallback` (`calculateOffsetWithOffsetPosition`), then clamps (`clampScrollOffset`);
   - clears `scrollHistory`;
   - stores `state.scrollingTo` with `targetOffset` and an `averageSizeSnapshot`;
   - pins the target viewport's index range (`scrollTargetPinnedRange`) so the destination rows mount before the jump (commit 2124474a).

   For non-animated requests it sets `scrollPending` and calls `updateScroll(targetOffset)` right away, so items at the destination are calculated before the native scroll. For animated requests it leaves `scrollPending` alone, so a layout-triggered completion check can't treat the destination as already reached (commit b35aa376).
4. **Dispatch.** `doScrollTo` calls `refScroller.current.scrollTo({x,y,animated})`, converting horizontal RTL offsets with `toNativeHorizontalOffset`. See the platform table below for how each platform detects completion.
5. **Finish detection.** `checkFinishedScroll` schedules `checkFinishedScrollFrame` one frame later, because a `requestAdjust` may still follow. The scroll counts as finished when `scrollPending` is within 1px of the current target, or of target + applied `scrollAdjust` (only for non-animated or iOS). `checkFinishedScrollFallback` is the timeout safety net:
   - starts after 100ms, or 500ms before containers have laid out;
   - re-checks up to 5 times;
   - finishes once `hasScrolled` is true (for non-initial scrolls).
6. **Retargeting while in flight.** Positions keep changing while the scroll moves. Several things keep the target correct:
   - **MVCP anchors on the target index.** `prepareMVCP` in `src/core/mvcp.ts` uses `scrollingTo.index` as the anchor. When the target's own size changes it adds `(newSize - prevSize) * viewPosition` (or the fallback alignment delta) and updates `scrollingTo.itemSize`.
   - **Deferred adjust where adjusting breaks the scroll.** Where `PlatformAdjustBreaksScroll` is true (web, Android), adjusts during an animated scroll go to `pendingAdjust`. `finishScrollTo` commits them by recomputing the target from the item's current position (`ScrollAdjustHandler.commitPendingAdjust`).
   - **End targets are recomputed on every completion check** (`getCurrentTargetOffset`). On native, an unaligned end target is re-scrolled from the fallback loop (commits eefb9c5b, 7c227fa6).
   - **Viewport length changes.** With `viewPositionFallback`, `handleLayout` re-issues `scrollToIndex(...forceScroll)` on the active target when the viewport length changes (commit ba0399a7).
   - **Frozen averages.** While `scrollingTo` is set, `getItemSize` uses the average-size snapshot captured at the start of the scroll, not live averages, so estimates don't drift mid-scroll (commit 69d7d616).
7. **Settle.** `finishScrollTo` cancels completion work and clears `scrollingTo`/`scrollTargetPinnedRange`/`scrollHistory`. It applies `pendingTotalSize`, commits the pending adjust, and calls `recalculateSettledScroll` (full position recalculation plus `checkThresholds`). It resolves the promise, unless a newer request is still waiting on readiness (`isWaitingToRun`), and replays any `pendingMaintainScrollAtEnd`. If the target was an initial scroll, it hands off to `finishInitialScroll` instead.

### Cancellation
| Trigger | What happens |
|---|---|
| New imperative request | `scrollRequestTracker.start` resolves the old promise. `scrollTo` cancels old completion keys. Every delayed completion callback is bound to the `scrollingTo` object it was created for, so stale timers do nothing (commit ad1d20dc). |
| Unmount | The `LegendList` effect cleanup calls `cancelImperativeScroll(state)`, which clears the target and **resolves** the promise without running `finishScrollTo`, then `state.scheduledWork.dispose()` (issue #508, commit fcd1e730). |
| Fresh dataset | `initialScrollLifecycle.ts` calls `cancelImperativeScroll` (commit f4f25d0e). |
| User drag (native) / wheel up (web) | Only **end-following** is interrupted (`interruptMaintainScrollAtEnd`). It cancels the active scroll only when that scroll is an end-follow or `scrollToEnd` driven by `maintainScrollAtEnd`. A plain `scrollToIndex` is **not** cancelled by a drag. It ends through the normal or fallback completion path once `hasScrolled` is observed. |

Promises from ref scroll methods always resolve and never reject, whether the scroll was superseded, cancelled, or finished.

### Web vs native
| Aspect | Web (`doScrollTo.ts`, `ListComponentScrollView.tsx`) | Native (`doScrollTo.native.ts`, `ListComponentScrollView.native.tsx`) |
|---|---|---|
| Animated finish | DOM `scrollend` (ignored if not within 1px of target), else 80ms scroll-idle, max 1500ms. Browsers without `scrollend` use a 320ms idle wait. The listener group is registered as `platformScrollCompletion`. | `onScroll` → `checkFinishedScroll`, plus `onMomentumScrollEnd` → `checkFinishedScrollFallback`. If the scroller is already at the target, iOS emits no momentum event, so the scroll finishes on the alignment frame (commit d2f4e777). |
| Already-aligned animated request | Finished on the next frame (the DOM fires no events). | Finished through `checkFinishedScroll`. |
| Non-animated finish | 100ms timeout, token-bound. | `state.scroll = offset` plus `checkFinishedScrollFallback`. |
| Adjust during animated scroll | Deferred (`PlatformAdjustBreaksScroll = true`). | Deferred on Android only. |
| Scroll owner | Own div, `useWindowScroll`, or the `scrollElement` ancestor (3.5.0, commit 10f42e34). With external scroll (`props.hasExternalScroll`), offsets are list-relative: `getRawScrollOffset` subtracts the list's position in the owner, and `scrollTo` adds it back (`resolveExternalScrollOffset`). | RN ScrollView. |
| Reached-edge gate re-arm | DOM `scrollend` or a 200ms fallback (`onInternalScrollEnd`). | `onScrollBeginDrag`. |

`scrollElement` and `useWindowScroll` are ignored when `renderScrollComponent` is set, and `useDevChecks` warns about it. `onScroll` skips the past-end clamp when `hasExternalScroll` is set, because the owner can legitimately scroll beyond the list.

### Reached-edge callbacks
`checkThresholds` runs `checkAtBottom` and then `checkAtTop`. Each one:
- publishes `isAtEnd`/`isNearEnd`/`isWithinMaintainScrollAtEndThreshold` (or the `isAtStart`/`isNearStart` equivalents) through `set$`;
- calls `checkThreshold`, which fires once on entering the window `threshold × scrollLength` and latches (`isEndReached`/`isStartReached`, plus a snapshot of contentSize and dataLength);
- unlatches only after the user moves 1.3× the threshold away (hysteresis). With a threshold ≤ 0, any movement away unlatches.

`edgeReachedGate.ts` adds a shared gate across both edges (commits 420ffec3, 5e02da59):
- Any dispatch closes the gate (`"closed"`).
- A user gesture boundary sets `"prepared"`. The next real user scroll event (`isUserScrollEvent` in `updateScroll`) then allows only the edge in the scroll direction and resets that edge's latch.
- The gate clears entirely once scroll is outside both hysteresis windows.

The gate stops data changes, MVCP shifts and leftover scroll events from bouncing start→end→start within one gesture.

**Gating asymmetry:** `checkAtTop` skips dispatch while `scrollingTo` is set, while an initial scroll is active, and before `queuedInitialLayout`. `checkAtBottom` skips only during an active initial scroll or `maintainingScrollAtEnd`, so an in-flight `scrollToEnd` can fire `onEndReached`. Settled scrolls re-check through `recalculateSettledScroll`.

## Key types and entry points
| Symbol | File | Role |
|---|---|---|
| `onScroll` | `src/core/onScroll.ts` | Single entry for platform scroll events. |
| `updateScroll` | `src/core/updateScroll.ts` | Commits a scroll value, triggers item calculation and thresholds. Also used imperatively with `markHasScrolled: false`. |
| `scrollTo` | `src/core/scrollTo.ts` | Core imperative primitive: precomputes and clamps, sets `state.scrollingTo`, pins the range, dispatches. |
| `scrollToIndex` / `scrollToEnd` | `src/core/scrollToIndex.ts`, `src/core/scrollToEnd.ts` | Index → offset. `scrollToEnd` adds padding and footer through `getEndAlignedViewOffset` and marks `isScrollToEnd`. |
| `state.scrollingTo` (`InternalScrollTarget`) | `src/types.internal.ts` | The active target. Its object identity is the completion token. |
| `getScrollRequestTracker` | `src/core/scrollRequestTracker.ts` | Token-based supersession and promise ownership. `runNowIfIdle` is used by end maintenance so it never pre-empts an explicit request. |
| `checkFinishedScroll` / `checkFinishedScrollFallback` | `src/core/checkFinishedScroll.ts` | Frame-delayed alignment check and timeout/retry loop (includes Android silent initial-scroll "jiggle" retries). |
| `finishScrollTo` / `cancelImperativeScroll` | `src/core/finishScrollTo.ts`, `src/core/cancelImperativeScroll.ts` | Settle with recalculation, or tear down without it. |
| `getViewPositionOffset` / `calculateOffsetWithOffsetPosition` | `src/core/calculateOffsetWithOffsetPosition.ts` | `viewPosition` math. When the item is larger than the viewport (`availableSpace < 0`), `viewPositionFallback` replaces the alignment with 0 (`"start"`) or 1 (`"end"`). |
| `clampScrollOffset` | `src/core/clampScrollOffset.ts` | Clamps to `[0, contentSize - scrollLength]`. A negative `viewOffset` extends the max. On Android it doesn't clamp the max before the first layout. |
| `handleLayout` | `src/core/handleLayout.ts` | Viewport size changes: container allocation, recalculation, `viewPositionFallback` retarget, `maintainScrollAtEnd.onLayout`, `checkThresholds`. |
| `ScheduledWork` keys | `src/core/ScheduledWork.ts` | `checkFinishedScrollFrame`, `checkFinishedScrollRetryFrame`, `checkFinishedScrollFallback`, `platformScrollCompletion`, `imperativeScrollReady`. |

## Invariants and gotchas
- **Every delayed completion callback must check ownership.** It must compare against the `scrollingTo` it captured, or hold a `ScheduledWork` key that `cancelScrollCompletionChecks` cancels. Adding a new timer without one reintroduces issue #508 or stale-finish bugs.
- **Clear or finish only through the existing paths.** Never set `scrollingTo = undefined` directly outside `finishScrollTo`/`cancelImperativeScroll`/`finishInitialScroll`: pinned ranges, the pending adjust and the promise would leak.
- **Teardown never calls `finishScrollTo`.** It recalculates layout against a list that is unmounting (issue #508).
- **`scrollToOffset` defaults to non-animated.** `scrollTo` uses `!!animated`, although the JSDoc in `src/types.base.ts` says "Default: true". `scrollToIndex`/`scrollToItem`/`scrollToEnd`/`scrollIndexIntoView` default to animated.
- **`viewPosition` and `viewPositionFallback` apply only to index targets.** `scrollToOffset` only takes `viewOffset` into account (inside `calculateOffsetWithOffsetPosition`, when `index` is undefined).
- **`getState()` returns a snapshot.** `scroll`, `scrollLength`, `start`/`end`, `isAtEnd` etc. are copied when it is called. Only its function members read live data.
- **The public `onScroll` is throttled separately.** `scrollEventThrottle` throttles only the user's callback (`useThrottledOnScroll`, trailing edge). Internal processing always receives every event (`ListComponent` gets `scrollEventThrottle={0}`).
- **`handleLayout` ignores transient 0 lengths.** It keeps the previous non-zero `scrollLength`, which is common on web during flex settle.
- **Two consecutive scroll requests in one tick are allowed.** The first promise resolves when the second starts. Don't assume a resolved promise means the target was reached.
- **`scrollToEnd` is deferred by design.** It is stored in `pendingScrollToEnd` and run from a `useLayoutEffect` after a forced re-render (`scheduleImperativeScrollCommit`), so calling it right after `setData` targets the new last item (commit 1a2bbb5c). On web, data-change MVCP is skipped while it is pending (`mvcp.ts`).

## Design decisions
| Decision | Why | Evidence |
|---|---|---|
| Precompute and pin the target range; update scroll synchronously for non-animated requests | Destination rows mount before the jump, so there are no blank frames | commits 9b855310, 2124474a |
| Animated requests keep `scrollPending` unchanged | Stops layout-triggered completion before any movement | commit b35aa376 |
| Completion check waits one frame | A `requestAdjust` can still arrive after the scroll event | comment in `checkFinishedScroll` |
| Timeout fallback alongside event-driven completion | Native momentum or `scrollend` events are unreliable (zero-distance scroll on iOS, early `scrollend` in some browsers) | commits d2f4e777, `doScrollTo.ts` comments |
| Snapshot average sizes during a scroll | Live averages moved estimated positions under the target | commit 69d7d616 |
| Defer scroll adjusts on web/Android during animated scroll | `scrollBy` during a programmatic smooth scroll breaks it | `ScrollAdjustHandler.commitPendingAdjust` comment |
| Resolve, never reject, on supersede or cancel | One settlement policy for supersession and teardown | issue #508 |
| Shared reached-edge gate per gesture | Data changes or MVCP could fire start and end callbacks repeatedly | commits 420ffec3, 5e02da59 |

## Where to change things
| To… | Start at |
|---|---|
| Change how a scroll event updates state or when items recalculate | `updateScroll` (`shouldUpdate` and the 2px threshold, the large-jump branch) |
| Add or modify an imperative ref method | `src/utils/createImperativeHandle.ts` (always go through `runScrollWithPromise` or `scrollRequestTracker`), type in `LegendListRef` in `src/types.base.ts` |
| Change offset math (`viewOffset`, `viewPosition`, headers, insets) | `calculateOffsetWithOffsetPosition`, `clampScrollOffset`, `getStartOffsetAdjustment`, `src/core/endOfContentTarget.ts` |
| Fix "scroll never resolves", early resolve or stale resolve | `checkFinishedScroll.ts` (`getResolvedScrollCompletionState`, fallback loop), `doScrollTo.ts` (web `listenForScrollEnd`), `finishScrollTo.ts` |
| Fix target drift while items measure during a scroll | `mvcp.ts` (`scrollTarget` anchor branch), `ScrollAdjustHandler.commitPendingAdjust`, `getCurrentTargetOffset` |
| Change readiness waiting before a scroll | `runWhenReady` / `isSettlingAfterDataChange` / `isScrollToIndexReady` in `createImperativeHandle.ts` |
| Change `onStartReached`/`onEndReached` behavior | `checkAtTop.ts`, `checkAtBottom.ts`, `checkThreshold.ts` (hysteresis), `edgeReachedGate.ts`; gate arming in `LegendList.tsx` `fns` and `updateScroll` |
| Change web scroll event delivery or external scroll owners | `src/components/ListComponentScrollView.tsx`, `src/components/webScrollUtils.ts` |
| Change native drag/momentum hooks | `fns` in `src/components/LegendList.tsx` |
| Change `snapToIndices` offsets | `updateSnapToOffsets` (called from `LegendList` and `updateItemPositions`) |

## Tests
- Unit, under `__tests__/core/`: `scrollTo.test.ts`, `scrollTo.precompute.test.ts`, `scrollToIndex.test.ts`, `checkFinishedScroll.test.ts`, `finishScrollTo.test.ts`, `doScrollTo.web.test.ts`, `doScrollTo.native.test.ts`, `onScroll.test.ts`, `updateScroll.test.ts`, `handleLayout.test.ts`, `viewPositionFallback.test.ts`, `calculateOffsetWithOffsetPosition.test.ts`, `clampScrollOffset.test.ts`.
- Unit, under `__tests__/utils/`: `createImperativeHandle.test.ts` (supersession, cancellation, readiness), `checkThreshold*.test.ts`, `checkAtTop/Bottom.test.ts`, `edgeReachedGate.test.ts`, `reachedEdges.*.test.ts`, `getScrollVelocity.test.ts`, `updateSnapToOffsets.test.ts`.
- Integration, under `__tests__/components/`: `LegendList.imperativeScroll.test.tsx`, `LegendList.reachedEdges.integration.test.tsx`, `ListComponentScrollView.web.test.tsx`, `webScrollUtils.test.ts`.
- Pattern: build state with `createMockContext` from `__tests__/__mocks__/createMockContext`, mock `refScroller.current.scrollTo`, and switch platforms by assigning `Platform.OS` (restore it in `afterEach`). Import `doScrollTo.native` explicitly to test the native branch, since the default import resolves to the web file.
- Verified: `bun test __tests__/core/checkFinishedScroll.test.ts __tests__/core/doScrollTo.web.test.ts __tests__/core/viewPositionFallback.test.ts __tests__/utils/edgeReachedGate.test.ts` passed 90 tests in ~60ms.
- Visual checks: `cd example-web && bun run dev:fixtures` (`VITE_LEGEND_LIST_MODE=fixtures`, routes in `example-web/src/fixtures/routes.tsx`). Scroll-relevant fixtures: `/accurate-scrollto`, `/accurate-scrollto-huge`, `/external-scroll` (`scrollElement`), `/window-scroll`, `/snap-to-indices`, `/bidirectional-infinite-list` (reached edges), `/chat-end-follow`. These are manual visual checks. The only automated test that loads a fixture is `__tests__/components/ChatEndFollow.fixture.test.ts` (with `.cases.tsx`). It imports the example-web chat fixture with a mocked `LegendList` to check message identity, not scroll behavior. Native fixtures: `cd example && bun run ios:fixtures`.
