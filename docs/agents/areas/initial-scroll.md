<!-- index-codebase: generated from 3d80d167 on 2026-10-03. Code wins when it disagrees with this doc. -->
# Initial scroll

**Purpose:** gets a freshly mounted list, or a list whose dataset was replaced, to its first scroll position (`initialScrollIndex`, `initialScrollOffset`, `initialScrollAtEnd`). It keeps the items hidden until they are placed, then flips `readyToRender` and fires `onLoad`/`onReady`. After the first placement it keeps end-aligned targets alive for a short time so late footer, viewport or inset changes can still correct them. It does **not** own imperative `scrollTo*` (see [scrolling](scrolling.md)), MVCP or `maintainScrollAtEnd` (see [scroll-stability](scroll-stability.md)), or range/container math (see [virtualization](virtualization.md)). It does dispatch through those systems.
**Paths:** `src/core/bootstrapInitialScroll.ts`, `src/core/initialScroll.ts`, `src/core/initialScrollLifecycle.ts`, `src/core/initialScrollSession.ts`, `src/core/finishInitialScroll.ts`, `src/utils/hasActiveInitialScroll.ts`, `src/utils/setInitialRenderState.ts`, `src/utils/setDidLayout.ts`, `src/hooks/useFreshDataTransitionVisibility.ts`, `src/hooks/useInit.ts`, plus the initial-scroll wiring in `src/components/LegendList.tsx`.
**Depends on:** [state](state.md) (`InternalState`, `set$`/`peek$`), [scrolling](scrolling.md) (`scrollTo`, `checkFinishedScroll`, `finishScrollTo`), [virtualization](virtualization.md) (`calculateItemsInView`, positions, `checkAllSizesKnown`), [rendering-and-platform](rendering-and-platform.md) (containers' opacity, `ListComponent` `contentOffset`). **Used by:** [public-api](public-api.md) (props, `onReady`), [scroll-stability](scroll-stability.md) (preserved end targets, `retargetActiveInitialScrollAtEnd`).

## How it works

### Two session kinds
`LegendList` turns the props into one internal target, `state.initialScroll` (type `InternalInitialScrollTarget` in `src/types.internal.ts`). It also creates `state.initialScrollSession`, whose `kind` is one of two values:

| Props | `kind` | Strategy |
|---|---|---|
| `initialScrollIndex` (number or object) or `initialScrollAtEnd` | `"bootstrap"` | Hidden convergence loop. The target offset is re-resolved from measured sizes until stable, then one final scroll is dispatched. |
| `initialScrollOffset` only (or horizontal RTL with no initial prop, via `shouldInitializeHorizontalRTL`) | `"offset"` | Direct offset. There is nothing to converge, so it scrolls once layout exists. |

`usesBootstrapInitialScroll = initialScrollAtEnd || hasInitialScrollIndex`. When an index is also given, `initialScrollOffset` acts as `viewOffset`.

The target is normalized in `LegendList` at render time:
- `initialScrollAtEnd` → `{index: last, viewPosition: 1, viewOffset: getEndAlignedViewOffset(paddingEnd), preserveForBottomPadding: true}`. `initializeInitialScrollOnMount` adds `preserveForFooterLayout` when a `ListFooterComponent` exists.
- A numeric `initialScrollIndex` that points at the last row (with `data.length > 1` and no `initialScrollOffset`) is bottom-aligned like `initialScrollAtEnd`. This came from 06516b9f and changelog 3.3.6, and a3418a74 includes the footer in the target.
- An object target with `viewPosition: 1` and no explicit `viewOffset`/`viewPositionFallback` gets `preserveForBottomPadding`.

### The bootstrap session
`state.initialScrollSession.bootstrap` (`BootstrapInitialScrollSession`) exists only while convergence runs. It holds `scroll` (the bootstrap's own idea of the offset, kept apart from live `state.scroll`), `seedContentOffset`, `targetIndexSeed`, `passCount`, `mountFrameCount`, `previousResolvedOffset`, `visibleIndices` and an rAF `frameHandle`. While it exists, `calculateItemsInView` sets `suppressInitialScrollSideEffects`. That has four effects: viewport math uses `bootstrap.scroll`; MVCP, render-range projection and threshold or viewability work are skipped; container allocation starts from `targetIndexSeed`; and every pass ends in `evaluateBootstrapInitialScroll` instead of the normal `setDidLayout` and viewability path.

### End-to-end flow: native mount with `initialScrollIndex={50}`
1. **Render (`LegendList`).** The first render builds `ctx.state` with `initialScroll` and `initialScrollSession: {kind: "bootstrap"}`. `updateItemPositions` lays out **estimated** positions from `estimatedItemSize`/`getFixedItemSize`. The `initialContentOffset` memo calls `resolveInitialScrollOffset`, which runs `calculateOffsetForIndex` → `calculateOffsetWithOffsetPosition` → `clampScrollOffset`. `ListComponent` passes that offset as the ScrollView `contentOffset`. On **web** the memo returns `undefined` for bootstrap sessions, so the DOM stays at 0 and the bootstrap loop owns the first scroll.
2. **Mount layout effect.** `initializeInitialScrollOnMount` (`initialScrollLifecycle.ts`) stores the estimated `contentOffset` on the target and calls `startBootstrapInitialScrollOnMount`. That call has two fast paths. A zero-offset top-origin target calls `finishInitialScroll` right away. Empty data with an index target calls `finishInitialScroll({preserveTarget: true})`, which keeps the target for when data arrives. Otherwise it calls `startBootstrapInitialScrollSession` (native: `seedContentOffset = offset`, web: `0`) and `ensureBootstrapInitialScrollFrameTicker`.
3. **Layout and measurement.** `onLayout` → `handleLayout` → `calculateItemsInView` (suppressed) → `evaluateBootstrapInitialScroll`. Each pass increments `passCount`, re-resolves the offset with whatever sizes are now known, and decides:
   - offset moved by more than 1px → record it and queue another pass on the next rAF (`queueBootstrapInitialScrollReevaluation` calls `triggerCalculateItemsInView({forceFullItemPositions: true})`);
   - buffered range or viewport slice not fully in `sizesKnown` → wait for more measurements;
   - otherwise this pass is the baseline. **Two consecutive passes** with the same offset (within 1px) and the same visible indices (`doVisibleIndicesMatch`) count as settled.
4. **Settle.**
   - On iOS, when both the seed and the observed native offset already equal the resolved offset, `finishBootstrapInitialScrollWithoutScroll` finishes without scrolling (fd685c59).
   - Otherwise the session is cleared and `dispatchInitialScroll` sends one `scrollTo({isInitialScroll: true, forceScroll: true})`. On web it adds `waitForInitialScrollCompletionFrame`.
   - Android always dispatches, because it can drop the mount `contentOffset` while content is still materializing (af48bf74).
5. **Normal path takes over.** With the bootstrap session gone, `calculateItemsInView` runs unsuppressed. Before `queuedInitialLayout`, the active target still anchors `scrollState` through `hasActiveInitialScroll`. Once the buffered sizes are known it calls `setDidLayout` (sets `queuedInitialLayout`, `checkThresholds`, `setInitialRenderState({didLayout})`) and then `handleInitialScrollLayoutReady`. That runs `advanceCurrentInitialScrollSession` now and again one rAF later, plus `checkFinishedScroll({onlyIfAligned: true})`.
6. **Completion.** `checkFinishedScroll` → `finishScrollTo`. Because `scrollingTo.isInitialScroll` is set, it calls `finishInitialScroll`. That clears the watchdog, either clears the target or *preserves* it (bottom-aligned targets with data), optionally calls `recalculateSettledScroll`, then calls `setInitialRenderState({didInitialScroll: true})`.
7. **Reveal.** `setInitialRenderState` sets `readyToRender` only when **both** `didContainersLayout` and `didFinishInitialScroll` are true. On that transition it switches adaptive render to `"normal"`, schedules the full-draw-distance prewarm (the first render caps draw distance at `INITIAL_DRAW_DISTANCE`), calls `onLoad` once per mount (`state.didLoad`), and calls `onReady` every time. `Containers.tsx`/`Containers.native.tsx` render at `opacity: 0` (web also sets `pointerEvents: none`) until `useFreshDataTransitionVisibility(readyToRender, freshDataTransitionEpoch)` is true. On web, `finishInitialScroll` also releases the public `onScroll` event held back during bootstrap (`shouldDeferPublicOnScroll` in `onScroll.ts`, `releaseDeferredPublicOnScroll`; cc31a95d).

### Bounded failure
`ensureBootstrapInitialScrollFrameTicker` counts frames. After `DEFAULT_BOOTSTRAP_REVEAL_MAX_FRAMES` (8) frames or `DEFAULT_BOOTSTRAP_REVEAL_MAX_PASSES` (24) passes, `abortBootstrapInitialScroll` warns in DEV and dispatches to the **last resolved** `bootstrap.scroll`. The original plan was to fall back to the origin (git `8ff19f38:plans/10-bootstrap-reveal-initial-scroll.md`); 034c9dfc and 33ebbf6b changed it to keep the best-known target. Re-arming during mount keeps the same `mountFrameCount` budget (`startBootstrapInitialScrollSession`).

### Re-targeting after the first estimate
| Event | Entry point | Effect |
|---|---|---|
| Data length/structure change during bootstrap, or a bottom-aligned target | `handleInitialScrollDataChange` → `handleBootstrapInitialScrollDataChange` | Rebuilds end targets (`createInitialScrollAtEndTarget`, `createRetargetedBottomAlignedInitialScroll`) and calls `rearmBootstrapInitialScroll` (resets `passCount` and snapshot, new rAF pass). |
| Footer measured | `onLayoutFooter` → `handleBootstrapInitialScrollFooterLayout` | If the end target changed, sets `resetDidFinish` and re-arms, which **re-hides the list after it finished**. Otherwise it downgrades `preserveForFooterLayout` (`clearPendingInitialScrollFooterLayout`). |
| Viewport size change (`scrollLength`/`otherAxisSize`) | `onLayoutChange` → `handleBootstrapInitialScrollLayoutChange` | During bootstrap it re-arms. During the final scroll it re-dispatches to the new offset. After finish it calls `schedulePreservedEndAnchorCorrection`, an rAF loop of `requestAdjust` that waits for a native scroll event between corrections. |
| Content inset change (web `contentInsetEndAdjustment`, `reportContentInset`) | `retargetActiveInitialScrollAtEnd` | Same split: advance if active, preserved-end correction if finished. |
| User scrolls away from a finished end target | `onScroll` → `clearFinishedBootstrapInitialScrollTargetIfMovedAway` | Releases the preserved target. It stays alive if `isAtEnd` and the target is retargetable, because the browser may be clamped at the end while positions are still estimated. |
| Imperative `scrollTo*`/`scrollToEnd` | `createImperativeHandle` → `supersedeInitialScroll` | Cancels bootstrap and the initial `scrollingTo`, then finishes the initial scroll so the explicit request wins (07b3edee, changelog 3.3.5 #518). |
| No event at all | `finishInitialScroll({schedulePreservedTargetClear})` | Clears the preserved target after `PRESERVED_INITIAL_SCROLL_FALLBACK_CLEAR_DELAY_MS` (2000 ms). The code calls this a backstop; the main lifecycle is event-driven. The delay was raised from 300 ms in fe42f5b4. |

### dataKey, fresh datasets and `onReady`
During render, `LegendList` computes `shouldResetFreshDataLayout`: the list is not on its first render, the data changed, `hasHadNonEmptyData` is true, either `dataKey` changed or the old length was 0, and the new data is non-empty. When true it bumps `state.freshDataTransitionEpoch`. Any data change also bumps `state.dataChangeEpoch`. The layout effect compares those epochs with `state.handled*Epoch`, which is kept on shared state. It does not compare render-local booleans, because a Strict Mode or concurrent render retry would otherwise drop the reset (660f67a4, 3c0f5d9c, changelog 3.3.10). On a fresh start:
1. `resetInitialRenderState({resetLayout: true, resetInitialScroll: !!initialScrollProp})` clears `didContainersLayout`, `queuedInitialLayout`, edge-reached snapshots, `didFinishInitialScroll` and `readyToRender`, and resets adaptive render.
2. `handleInitialScrollDataChange({didStartFreshData})` cancels the old dataset's imperative scroll and `maintainScrollAtEnd`, then installs the **latest** initial-scroll props with `setInitialScrollTarget` (e9258e31, changelog 3.1.0; dataKey itself added in 0007ef77, changelog 3.3.0).
3. `finishInitialScroll` captures `freshDataTransitionEpoch` and ignores a completion that was queued by an older dataset (f4f25d0e).
4. The cycle re-runs, ends in `setInitialRenderState`, and fires `onReady` again (1669b216, changelog 3.4.0). `onLoad` does not fire again (eb6db50b).

**Inferred:** for a non-bottom-aligned index target, a dataKey reset does not re-arm a bootstrap session. `resetInitialRenderState` has already cleared `didFinishInitialScroll`, so `shouldResetDidFinish` is false, and no session is left to re-arm. The replay runs through the measured path instead: `calculateItemsInView` anchors to `resolveInitialScrollOffset`, then `setDidLayout` → `handleInitialScrollLayoutReady` → `advanceMeasuredInitialScroll`. The rows stay hidden through `readyToRender`. A bootstrap re-arm happens only for end-aligned targets, or when the very first mount had empty data (`shouldResetDidFinish`).

`useFreshDataTransitionVisibility` exists because stale rows from the old dataset flashed for one frame (9d35f6a8, changelog 3.3.4). The readiness reset and its restore can coalesce in one layout-effect pass, so the hook completes the epoch in its own `useLayoutEffect` keyed on the epoch. Before that fix, lists could stay at opacity 0 forever (627200d9, #519, changelog 3.3.5).

## Key types and entry points
| Symbol | File | Role |
|---|---|---|
| `initializeInitialScrollOnMount` | `src/core/initialScrollLifecycle.ts` | Mount entry, called from `LegendList`'s first `useLayoutEffect`. |
| `handleInitialScrollDataChange` | `src/core/initialScrollLifecycle.ts` | Data/dataKey entry. Picks the latest target, then calls the bootstrap handler or the offset replay. |
| `handleInitialScrollLayoutReady` | `src/core/initialScrollLifecycle.ts` | Called from `calculateItemsInView` right after `setDidLayout`. Two-pass advance. |
| `retargetActiveInitialScrollAtEnd` | `src/core/initialScrollLifecycle.ts` | Inset-change hook used by `updateContentInsetEndAdjustment` and `reportContentInset`. |
| `startBootstrapInitialScrollOnMount`, `evaluateBootstrapInitialScroll`, `handleBootstrap*Change`/`FooterLayout` | `src/core/bootstrapInitialScroll.ts` | Bootstrap loop: start, per-pass convergence, re-arm triggers. |
| `schedulePreservedEndAnchorCorrection`, `clearFinishedBootstrapInitialScrollTargetIfMovedAway` | `src/core/bootstrapInitialScroll.ts` | Corrections after finish, and releasing the preserved end target. |
| `resolveInitialScrollOffset`, `dispatchInitialScroll`, `setInitialScrollTarget`, `advanceCurrentInitialScrollSession` | `src/core/initialScroll.ts` | Target to offset math, the one `scrollTo` call site, target mutation, and the non-bootstrap advance. |
| `setInitialScrollSession`, `initialScrollCompletion`, `initialScrollWatchdog` | `src/core/initialScrollSession.ts` | Session object and the native-dispatch, silent-retry and watchdog flags read by `scrollTo`/`onScroll`/`checkFinishedScroll`. |
| `finishInitialScroll`, `supersedeInitialScroll`, `clearPreservedInitialScrollTarget` | `src/core/finishInitialScroll.ts` | The only path that sets `didFinishInitialScroll`. |
| `hasActiveInitialScroll` | `src/utils/hasActiveInitialScroll.ts` | `initialScroll && !didFinishInitialScroll`. Gates thresholds (`checkAtTop`/`checkAtBottom`), render projection, allocation direction and the draw-distance prewarm. |
| `setInitialRenderState` / `resetInitialRenderState` | `src/utils/setInitialRenderState.ts` | `readyToRender` latch, `onLoad`/`onReady`. |
| `setDidLayout` | `src/utils/setDidLayout.ts` | Marks `queuedInitialLayout`, evaluates edge thresholds, may run pending `maintainScrollAtEnd`. |
| `useInit` | `src/hooks/useInit.ts` | Runs once during render. Used only for the old-architecture `doInitialAllocateContainers`. |

## Invariants and gotchas
- **`readyToRender` needs both flags.** Something must set `didContainersLayout` (through `setDidLayout`) and something must set `didFinishInitialScroll` (through `finishInitialScroll`). If a path forgets either one, the list stays invisible. All finishes go through `finishInitialScroll`; do not set `didFinishInitialScroll = true` directly.
- **`onReady` fires on each false→true `readyToRender` edge.** Anything that calls `resetInitialRenderState`, such as a footer re-arm with `resetDidFinish`, can fire `onReady` again and re-hide the list.
- **Bootstrap `scroll` is not live `state.scroll`.** While the session exists, range math uses `bootstrap.scroll`. Do not write the bootstrap target into `state.scroll` (decision recorded in plan 10).
- **Exactly one corrective dispatch per bootstrap.** `evaluateBootstrapInitialScroll` clears the session *before* `dispatchInitialScroll`. Later movement is handled by `advanceMeasuredInitialScroll` and `handleBootstrapInitialScrollLayoutChange`, not by bootstrap.
- **Platform splits are deliberate:** web seeds 0 and dispatches with one extra completion frame; Android always dispatches; iOS may skip the dispatch.
  - Android also has a silent-dispatch "jiggle" retry in `checkFinishedScroll`, tracked by `didRetrySilentInitialScroll` (56648784, b3c65621).
  - The iOS/native watchdog in `initialScrollWatchdog` keeps `hasScrolled` false until the native offset reaches the target, so early scroll events are not read as user scrolls (af1e3b8a, 6b6ed2ad).
- **Epsilons are 1px** (`DEFAULT_BOOTSTRAP_REVEAL_EPSILON`, `INITIAL_SCROLL_MIN_TARGET_OFFSET`). A target at 1px or less counts as "zero" for the watchdog.
- **Footer offset belongs only to end-of-content targets** (`isEndOfContentTarget`, `getEndAlignedViewOffset`). Item-aligned targets must never include the footer, and adding it twice was a past bug (e83ee4e6, a3418a74).
- **Effect ordering in `LegendList` matters.** The mount effect (`initializeInitialScrollOnMount`) is declared before the data-change effect, and the data-change effect is declared before the container-allocation effect.
- **Old architecture differs** (`IsNewArchitecture === false`): `canRender` starts true, `scrollLength` is seeded from the window size, and containers are allocated during render through `useInit`. `doInitialAllocateContainers` defers the first `calculateItemsInView` to rAF when `initialScroll` exists. `addTotalSize` holds a shrinking total in `pendingTotalSize` during initial scroll, and `prepareMVCP` (`src/core/mvcp.ts`) skips MVCP for an initial-scroll target (51fd720a).
- **Not every planned change landed.** Plan 13 (git `1012e5a8:plans/13-initial-scroll-session-architecture.md`) wanted `phase` states and bootstrap policy moved out of `calculateItemsInView`. The current code has neither: `calculateItemsInView` still branches on the bootstrap session, and `finishScrollTo` still decides preserve-versus-clear.

## Design decisions
- **Hidden convergence instead of visible retries** (plan 10 at `8ff19f38`, runtime 936f1b20, April 2026). The older strategy scrolled, measured, then re-scrolled while visible, which caused jumps. Bootstrap keeps the content hidden until two consecutive passes agree on the offset and the visible slice. Measuring the buffered range alone was not enough on web with variable row heights; see the comment in `evaluateBootstrapInitialScroll`.
- **`initialScrollOffset` stays out of bootstrap** (plan 10 non-goals, 9ab77523). A raw offset does not depend on item sizes, so it uses the direct `"offset"` session.
- **One session object instead of scattered flags** (plan 13, about 40 `refactor:` commits on 2026-04-04/05). This explains most of the churn in `bootstrapInitialScroll.ts`: the 84 commits split into one large refactor burst plus a long tail of platform-specific `fix:` commits for footer, viewport, inset, Android and iOS edge cases.
- **Preserve bottom-aligned targets after finish.** Chat-style lists open at the end before the footer, keyboard inset or viewport has measured. Clearing the target at finish would leave the list short of the end (8e08ee61, 05a8e03b, 6a52cea9, d08ea6a6, e83ee4e6). The preservation is limited by user-scroll release, data-change release (80054a97) and a 2 s timer (52440ed1).
- **Epoch counters for dataset resets**, because render-phase flags do not survive React render retries (660f67a4).

## Where to change things
| To… | Start at |
|---|---|
| Change how props become a target (new prop shape, default alignment) | `initialScrollProp` construction in `src/components/LegendList.tsx`, plus `createInitialScrollAtEndTarget` |
| Change offset math for a target | `resolveInitialScrollOffset` (`src/core/initialScroll.ts`); `calculateOffsetWithOffsetPosition` and `clampScrollOffset` are shared with `scrollToIndex` |
| Tune convergence, abort limits or the settle rule | `evaluateBootstrapInitialScroll`, `DEFAULT_BOOTSTRAP_REVEAL_*` in `src/core/bootstrapInitialScroll.ts` |
| React to a new late-layout input (new inset or padding source) | Add a trigger that calls `rearmBootstrapInitialScroll` during bootstrap or `schedulePreservedEndAnchorCorrection` after finish; model it on `handleBootstrapInitialScrollLayoutChange` or `retargetActiveInitialScrollAtEnd` |
| Change when content becomes visible or when `onReady`/`onLoad` fire | `setInitialRenderState`, `useFreshDataTransitionVisibility`, `Containers*.tsx` |
| Change what a dataKey or fresh dataset resets | `shouldResetFreshDataLayout` and the data-change `useLayoutEffect` in `LegendList.tsx`, `resetInitialRenderState`, `handleInitialScrollDataChange` |
| Change completion detection (native silent scroll, watchdog) | `checkFinishedScroll`, `finishScrollTo` ([scrolling](scrolling.md)), `initialScrollSession.ts` flags |
| Let a new imperative API cancel initial scroll | Call `supersedeInitialScroll` from `createImperativeHandle` |
| Change gating of edge callbacks during initial scroll | `hasActiveInitialScroll` call sites in `checkAtTop`/`checkAtBottom`, and `setDidLayout` |

## Tests
| Scope | File |
|---|---|
| Bootstrap unit (start, evaluate, re-arm, abort) | `__tests__/core/bootstrapInitialScroll.test.ts` |
| Target and session helpers | `__tests__/core/initialScroll.test.ts`, `__tests__/core/initialScrollSession.test.ts`, `__tests__/core/initialScrollLifecycle.test.ts` |
| Completion | `__tests__/core/finishScrollTo.test.ts`, `__tests__/core/checkFinishedScroll.test.ts` |
| Readiness, `onReady`, layout | `__tests__/utils/setInitialRenderState.test.ts`, `__tests__/utils/setDidLayout.test.ts`, `__tests__/hooks/useFreshDataTransitionVisibility.test.tsx` |
| Mounted list, platform matrix, dataKey | `__tests__/components/LegendList.bootstrapInitialScroll.test.tsx`, `__tests__/components/LegendList.initialScroll.integration.test.tsx` (Strict Mode dataKey case), `__tests__/components/LegendList.props.test.tsx` ("restarts layout readiness when dataKey changes…"), `__tests__/components/LegendList.initialScrollThresholds.test.tsx` |
| Old architecture | `__tests__/components/LegendList.bootstrapInitialScroll.oldarch.test.ts`, which spawns `bun test …oldarch.cases.tsx` in a fresh process. The cases file calls `mock.module("@/constants-platform", {IsNewArchitecture: false})`. **Inferred:** Bun module mocks are process-global, so the spawn keeps the old-arch mock from leaking into other files (19279c48). `__tests__/utils/requestAdjust.oldarch.test.ts` uses the same pattern. Add old-arch cases to `*.cases.tsx`, not to the wrapper. |
| Visual fixtures | Web: `example-web/src/fixtures/InitialScrollIndexExample.tsx`, `InitialScrollAtEndExample.tsx`, `EndResizeExample.tsx`. Native: `example/screens/fixtures/scroll-regression-qa.tsx` |
| Device QA replay | `.argent/flows/qa-initial-last-index-resize.yaml` and its siblings. They are recorded Argent flows (845a7a72) that launch `com.legendapp.listtest`, tap testIDs in the native QA fixture and wait for a `*-pass` testID. The repo only registers Argent as an MCP server (`argent mcp` in `.vscode/mcp.json`). It does not install the `argent` binary. |

Verified at 3d80d167: `bun test __tests__/core/bootstrapInitialScroll.test.ts __tests__/core/initialScrollLifecycle.test.ts __tests__/components/LegendList.bootstrapInitialScroll.oldarch.test.ts __tests__/hooks/useFreshDataTransitionVisibility.test.tsx` gave 28 passing tests. `bun test __tests__/components/LegendList.bootstrapInitialScroll.test.tsx __tests__/components/LegendList.initialScroll.integration.test.tsx __tests__/components/LegendList.dataRefresh.integration.test.tsx` gave 62 passing tests.
