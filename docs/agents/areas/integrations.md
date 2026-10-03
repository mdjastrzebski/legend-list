<!-- index-codebase: generated from 3d80d167 on 2026-10-03. Code wins when it disagrees with this doc. -->
# Integrations

**Purpose:** optional subpath entrypoints layered on top of the core list: Animated, Reanimated, keyboard-controller (current and legacy), and a React Native-compatible `SectionList`. Each one wraps the public `LegendList` and plugs into a few internal props. None of them changes virtualization, layout, or scroll logic. That lives in core (see [virtualization](virtualization.md), [scrolling](scrolling.md)).
**Paths:** `src/integrations/animated.tsx`, `src/integrations/reanimated.tsx`, `src/integrations/keyboard.tsx`, `src/integrations/keyboard-legacy.tsx`, `src/section-list/`, the `internal` bridge in `src/entrypoints/shared.ts`, `package.json` `"exports"`, `tsup.config.ts`, `prebuild.ts`, `postbuild.ts`
**Depends on:** [public-api](public-api.md) (`LegendList`, `LegendListRef`, props), [state](state.md) (`useArr$`, `peek$`, `useStateContext`), [rendering-and-platform](rendering-and-platform.md) (`Container`, `PositionView*`, `ListComponent`), [scrolling](scrolling.md) (`reportContentInset`, `setScrollProcessingEnabled`, `anchoredEndSpace`) · **Used by:** app consumers only. Nothing in core imports these.

## What subpaths does the package publish?

`package.json` has **no root `"."` export** and no `main`/`module`/`types` fields. Commit `c34fcb89` ("feat: remove root package entrypoint") deleted them along with `src/index.ts` and `src/types.root.ts`. For users this means `import … from "@legendapp/list"` fails to resolve (`ERR_PACKAGE_PATH_NOT_EXPORTED` under Node, and an unresolved module under Metro or bundlers). Every import must name a subpath. `prebuild.ts` (`findDirectRootPackageImports`) also fails the build if anything in `src/` imports the bare `"@legendapp/list"`.

| Subpath | Source | Exports | Runtime deps it `require`s | Platform |
|---|---|---|---|---|
| `@legendapp/list/react-native` | `src/react-native.ts` | `LegendList`, item hooks, types, `internal` | — | native, plus a `browser` condition that maps to the web build (`6aba4b2e`) |
| `@legendapp/list/react` | `src/react.ts` | same surface, DOM types | — | web |
| `@legendapp/list/animated` | `src/integrations/animated.tsx` | `AnimatedLegendList` (RN `Animated`) | `react-native` | native only |
| `@legendapp/list/reanimated` | `src/integrations/reanimated.tsx` | `AnimatedLegendList`, `AnimatedLegendListProps` | `react-native-reanimated` | native only |
| `@legendapp/list/keyboard` | `src/integrations/keyboard.tsx` | `KeyboardAwareLegendList`, `useKeyboardChatComposerInset`, `useKeyboardScrollToEnd` | `react-native-keyboard-controller` (≥1.21.7, which has `KeyboardChatScrollView`), `react-native-reanimated`, `@legendapp/list/reanimated` | native only |
| `@legendapp/list/keyboard-legacy` | `src/integrations/keyboard-legacy.tsx` | `KeyboardAvoidingLegendList` | `react-native-keyboard-controller` (`useKeyboardHandler`), `react-native-reanimated`, `@legendapp/list/reanimated` | native only |
| `@legendapp/list/section-list` | `src/section-list/index.ts` | `SectionList`, flattening types | `react-native` | built as native, uses `Platform` |

**Peer dependencies:** `package.json` `peerDependencies` lists only `react`. `react-dom` and `react-native` are optional. `react-native-reanimated` and `react-native-keyboard-controller` are **not declared as peers at all**. They appear only in `devDependencies` (`~4.2.1` and `^1.22.5`). An app that imports an integration subpath must install them itself. Nothing warns the app at install time. The only runtime hint is the `__DEV__` `console.warn` in `keyboard.tsx` when `KeyboardChatScrollView` is missing.

Integration exports have no `browser`/`react-native` conditions. They are compiled only in the `nativeEntryPoints` tsup config, which resolves `.native.ts(x)` files first.

## How it works

### The import boundary: integrations consume core through the published subpath

Integrations must import core **only** through `@legendapp/list/react-native` (or `/reanimated` and type-only `/react`). Local `@/…` and relative imports are not allowed. Anything internal they need goes through the frozen `internal` object exported from `src/entrypoints/shared.ts` and re-exported by `src/react-native.ts` and `src/react.ts`.

Three layers enforce this:

| Layer | File / symbol | What it checks |
|---|---|---|
| Source | `prebuild.ts` `findLocalIntegrationImports` | Fails on any `@/`, `./`, `../`, `/`, or `src/` specifier in `src/integrations/*.ts(x)`. Covers static, dynamic, `require`, and `import()` type imports. |
| Bundle | `tsup.config.ts` `external` | Lists `"@legendapp/list"`. tsup treats subpaths as external too, so `dist/*.js` keep `require('@legendapp/list/react-native')`. That was verified in a local `dist/`. |
| Output | `postbuild.ts` `assertIntegrationBundleIntegrity` | For `animated`, `keyboard`, `keyboard-legacy`, and `reanimated` `.js`/`.mjs`, the bundle must reference a `@legendapp/list/(animated\|react-native\|reanimated)` subpath. It must not contain `src/state/state.tsx`, `function useArr$`, `LegendListInner`, `src/core/`, or `ContextState = React.createContext(null)` ("duplicate ContextState detected"). |

`postbuild.ts` also checks that the core bundles (`react-native*.js`, `react*.js`) never mention `react-native-reanimated` or `react-native-keyboard-controller` ("Integration dependency leaked into core entrypoint bundle").

**Why:** integration code runs *inside* the core tree. Reanimated's position components are rendered by core's `Container` and call `useStateContext()` and `useArr$()`. If an integration bundle carried its own copy of `src/state/state.tsx`, it would create a second `ContextState` with `React.createContext`. Its hooks would then read a context that no provider fills, and every module-level singleton would be duplicated. Importing through the package subpath means the app's resolver returns the same core module instance to both the app and the integration.

Evidence:
- Commit `591c769f` ("share integration internals through react-native entrypoint") replaced `@/state/state`, `@/hooks/useCombinedRef`, and similar imports in `keyboard.tsx`/`reanimated.tsx` with `internal`.
- The `shared.ts` comment: "Internal bridge exports used by integration entrypoints to avoid duplicating local modules".
- `d3956843` and `72542d28` added the prebuild and postbuild guards.
- `e8dab833` ("section-list should import from @legendapp/list") and `88825b14` ("change integrations to import from the correct subpath") moved SectionList and the others off local imports.

**`internal` is not public API.** It is tagged `/** @internal */`, and `tsconfig.json` sets `stripInternal: true`, so it does not appear in `dist/*.d.ts`. Its members and their consumers:

| Member | Used by |
|---|---|
| `typedForwardRef`, `useCombinedRef` | keyboard, keyboard-legacy, section-list (`typedForwardRef`), reanimated (`useCombinedRef`) |
| `typedMemo` | reanimated, section-list |
| `useArr$`, `peek$`, `useStateContext`, `getStickyPushLimit`, `POSITION_OUT_OF_VIEW`, `IsNewArchitecture`, `getComponent`, `useLatestRef`, `useStableRenderComponent` | reanimated |

### Core hook points the integrations use

Core reads these props through `(props as any)` in `src/components/LegendList.tsx` and strips them before spreading the rest onto the scroll view. They are deliberately absent from public types.

| Internal prop | Read by core in | Effect | Set by |
|---|---|---|---|
| `renderScrollComponent` (public) | `ListComponent` via `useStableRenderComponent` | Replaces the ScrollView | reanimated (`ReanimatedScrollBridge`), keyboard (`KeyboardChatScrollView`) |
| `positionComponentInternal` | `Container` (picks the `PositionComponent`), `findAvailableContainers` (`shouldAvoidAssignedContainerReuse`) | Replaces `PositionView` for non-sticky rows | reanimated, only when `itemLayoutAnimation` is set |
| `stickyPositionComponentInternal` | `Container`, `useStickyScrollHandler.native.ts` (`shouldUseRnAnimatedEngine`) | Replaces `PositionViewSticky` and turns off the RN `Animated.event` sticky engine | reanimated, **new architecture only** |
| `anchoredEndSpaceOwnerInternal` (`"list"` default \| `"scroll"`) | `ListComponent` (`shouldMaterializeAnchoredEndSpace`) | With `"scroll"`, core does not pad the content and the scroll view owns the blank space | keyboard |
| `animatedPropsInternal` | `LegendList` → `state.props.animatedProps` | **Inferred: vestigial.** Nothing reads `state.props.animatedProps` after `cf342118`. Reanimated `animatedProps` actually reach the ScrollView because `AnimatedLegendList` also forwards `animatedProps` to `Reanimated.createAnimatedComponent(LegendListForwardedRef)`, whose ref is the scroll view. | reanimated |
| `refScrollView` (public) | `LegendList` `useCombinedRef(refScroller, refScrollView)` | Gives the wrapper the native scroll view | reanimated, keyboard-legacy |

Imperative ref methods used: `reportContentInset` (keyboard, keyboard-legacy) sets `contentInsetOverride`, which `src/state/getContentInsetEnd.ts` merges. `setScrollProcessingEnabled` (keyboard-legacy) pauses scroll processing during keyboard animation. `getState().listen` (reanimated) is also used.

### Traced flow: `AnimatedLegendList` with `sharedValues` and sticky headers (reanimated)

1. `AnimatedLegendList` (`reanimated.tsx`) captures the `LegendListRef` into React state through a combined ref. It then runs `useAnimatedLegendListSharedValuesSync`, which seeds each shared value from `getState()` and subscribes with `state.listen("isAtEnd" | "isNearEnd" | "activeStickyIndex" …)`. Writes go through `setSharedValueValue`, which prefers `.set()`. These updates hop over the JS thread, so they trail the UI thread by a tick.
2. It renders `AnimatedLegendListComponent = Reanimated.createAnimatedComponent(LegendListForwardedRef)`, passing `ref={refScrollView}`. Animated styles and props therefore target the scroll view.
3. `LegendListForwardedRef` builds `renderReanimatedScrollComponent` → `ReanimatedScrollBridge`, which renders `Reanimated.ScrollView` or wraps a user or keyboard `renderScrollComponent`. On the new architecture it also builds `stickyPositionComponentInternal` → `ReanimatedPositionViewSticky`. The UI-thread `scrollOffset` is tracked through `ReanimatedScrollOffsetTracker` (`useScrollViewOffset`) **only** when `sharedValues.scrollOffset` is supplied or `stickyHeaderIndices` is non-empty (CHANGELOG 3.3.3 perf entry).
4. Core `LegendList` stores the internal props in `state.props`. `Container` chooses `ReanimatedPositionViewSticky` for sticky containers. That component reads `containerPosition{id}`, `headerSize`, `containerItemIndex{id}`, and other values via `internal.useArr$`, then computes `getStickyPushLimit`. A `useAnimatedStyle` translates the header from `stickyScrollOffset` on the UI thread.

### Per-integration notes

**animated** is just `Animated.createAnimatedComponent(LegendList)`. It depends on `LegendListRef.getAnimatableRef` (`src/utils/createImperativeHandle.ts`). Commit `b7bdc0b6` added that method because the ref was missing it and wrapping broke.

**reanimated** adds three things:
- `sharedValues`: `scrollOffset`, `isAtStart`, `isAtEnd`, `isNearStart`, `isNearEnd`, `isWithinMaintainScrollAtEndThreshold`, `activeStickyIndex`.
- `animatedProps` for the ScrollView.
- `itemLayoutAnimation`: a Reanimated `layout` transition applied per row.

When `itemLayoutAnimation` is set, `ReanimatedPositionView` positions rows with `top`/`left` instead of `transform`, because "Layout transitions require positional layout props". It also skips one transition when a recycled container switches item key. Core's `findAvailableContainers` avoids reusing already-assigned containers when recycling is on and `positionComponentInternal` is set (`fa441898`). The Reanimated ScrollView is used on old architecture too (`9809d277`), but on old arch sticky headers fall back to core's RN `Animated` engine.

**keyboard** (`KeyboardAwareLegendList`, the current API) wraps the reanimated `AnimatedLegendList` and uses `react-native-keyboard-controller`'s `KeyboardChatScrollView` as `renderScrollComponent`. The library handles the keyboard lift (`keyboardLiftBehavior`, `freeze`, `offset` = `keyboardOffset`). The list:
- Forwards `KeyboardChatScrollView.onContentInsetChange` to `ref.reportContentInset`.
- Bridges `anchoredEndSpace.onSizeChanged` into the `blankSpace` shared value and sets `anchoredEndSpaceOwnerInternal="scroll"`, so the scroll view owns the blank space and core doesn't pad (`b71d9ed8`).
- Treats `contentInsetEndAdjustment` as a **SharedValue** here (a number in core) and passes it as `extraContentPadding`. Core ignores `contentInsetEndAdjustment` on native anyway (`contentInsetEndAdjustmentResolved` is web-only in `LegendList.tsx`).

`useKeyboardChatComposerInset(listRef, composerRef, initialHeight)` only updates that shared value, from a layout-effect `measure` plus `onComposerLayout`. Its `listRef` parameter is kept for compatibility. `KeyboardChatScrollView` stays the *only* reporter of the combined keyboard + composer inset (`785f99ae`). `useKeyboardScrollToEnd` freezes keyboard lift while `scrollToEnd` runs, optionally together with `KeyboardController.dismiss()`.

**keyboard-legacy** (`KeyboardAvoidingLegendList`) is the previous hand-rolled implementation. Commit `8696f901` moved it here when `/keyboard` was switched to the `KeyboardChatLegendList`-based component (`9a28ec0b`). `example-oldarch/` still uses it. It drives a `useKeyboardHandler` worklet state machine (`onStart`/`onInteractive`/`onMove`/`onEnd`):
- Animates `contentOffset` through `animatedProps`.
- On iOS, adds the keyboard height to `contentInset.bottom`. On Android it uses an animated `marginBottom`, because Android ignores `contentInset`.
- Calls `setScrollProcessingEnabled(false)` while the keyboard is animating.
- Calls `reportContentInset({ bottom })` on `onEnd`, since reading the animated inset didn't work well (`cf342118`).
- On iOS with `alignItemsAtEnd`, sets a `minHeight` on the content container.
- Takes a `safeAreaInsetBottom` prop.
- Composes the user's `onScroll` (JS function, worklet, or `useAnimatedScrollHandler` result) with its own handler.

Hard-won fixes: ignore the first close event after mount (`fd936aaf`), recover from a cancelled interactive dismissal (`a17fdb10`), clamp `progress` above 1 on iOS (code comment).

**section-list** (`SectionList`) is an RN `SectionList`-compatible adapter (`plans/1-sectionlist.md`). It keeps the core bundle unchanged by shipping as its own subpath.
- `buildSectionListData` (`flattenSections.ts`) flattens `sections` into one `data` array of tagged rows: `header`, `item`, `item-separator`, `footer`, and `section-separator` (only between sections). It also returns `sectionMeta[]` (flat indices of header/items/footer per section) and `stickyHeaderIndices` (the header indices when sticky is enabled).
- Header rows exist only when `renderSectionHeader` is a function, and footer rows only when `renderSectionFooter` is.
- Item separators are **real rows**, emitted when a top-level or per-section `ItemSeparatorComponent` exists. They are not LegendList's `ItemSeparatorComponent`.
- Mapping to `LegendList` props: `data` = flat rows, `keyExtractor = item.key`, `getItemType = item.kind` (so containers recycle per kind), `numColumns = 1`, `columnWrapperStyle = undefined`, and `stickyHeaderIndices` (cleared when `horizontal`).
- `renderItem` dispatches by kind, preferring `section.renderItem` and `section.ItemSeparatorComponent`.
- `getFixedItemSize` receives SectionList-shaped info with a `type` field (`f697b262`).
- `onViewableItemsChanged` filters out non-item rows and remaps `index` to the in-section index. `start`/`end` stay *flat* indices.
- The ref adds `scrollToLocation({ sectionIndex, itemIndex, viewOffset, viewPosition, viewPositionFallback, animated })`. It maps through `sectionMeta`, where `itemIndex === -1` targets the header (falling back to the first item, then the footer), and calls `scrollToIndex`.

## Invariants and gotchas

- **Never import `@/…` or relative paths into core from `src/integrations/`.** Use `@legendapp/list/react-native` plus `internal`. If you need a new internal, add it to `internal` in `src/entrypoints/shared.ts`. Both `react.ts` and `react-native.ts` re-export it.
- **`src/section-list/` is outside both guards.** The prebuild check scans only `src/integrations/`, and `postbuild.ts` `INTEGRATION_ENTRYPOINTS` omits `section-list`. It currently follows the same rule by convention: its only relative import is `./flattenSections`, which has no core imports. An `@/` import there would silently bundle a second copy of core.
- New integration entrypoints need updates in four places: `tsup.config.ts` `nativeEntryPoints`, `package.json` `"exports"`, `postbuild.ts` `INTEGRATION_ENTRYPOINTS`, and, for local dev, `tsconfig.json` `paths` plus `example/metro.config.js`/`example/tsconfig.json` aliases. Root `tsconfig.json` `paths` currently has no entries for `keyboard-legacy` or `section-list`. Tests import those from `src/` directly.
- Every `.tsx` file in `src/` must contain `import * as React from "react";`, enforced by `prebuild.ts` `findMissingReactImports` ("Leaving this out makes it crash in some environments").
- Internal props travel untyped (`props as any`). Renaming one in core won't break the integrations' type check. Grep both sides.
- `SectionList` default `stickySectionHeadersEnabled` is `Platform.OS === "ios"`, matching RN. Sticky headers are disabled whenever `horizontal` is set.
- `SectionList` row keys are `${section.key ?? "section-" + index}:item:${itemKey}`. Without `section.key`, inserting a section before others changes every later key and remounts those rows. The default item key is `item.key ?? item.id ?? index`.
- Flat indices ≠ RN indices: headers, footers, and separators occupy slots. `SectionListRef` spreads the `LegendListRef` methods, so a direct `scrollToIndex` uses flat indices. Use `scrollToLocation` for section-relative indices.
- `SectionList`'s `useImperativeHandle` spreads `legendListRef.current` and depends only on `scrollToLocation`. **Inferred:** it relies on `LegendListRef` method identities being stable.
- Sticky positioning must read `containerItemIndex{id}`, not a render-time `index` prop. Same-key section headers that move to a new flat index after a data change used to stay detached (issue #445, `issues/issue-445-sectionlist-sticky-headers-data-change.md`). The fix added the signal and removed the `index` prop from position components, including the Reanimated ones.
- In tests, integrations are exercised with `mock.module("@legendapp/list/react-native" | "@legendapp/list/reanimated" | "react-native-keyboard-controller", …)`, and those mocks leak across files in bun's module registry. `reanimated.isolation.test.ts` re-runs two suites in a subprocess with both random orders (seeds 1 and 3) to catch an incomplete mocked Reanimated export shape.

## Design decisions

| Decision | Reason / evidence |
|---|---|
| Integrations are separate subpaths, not part of core | Keeps optional native deps out of core bundles. `postbuild.ts` asserts core never mentions them. SectionList ships separately "to keep core bundle size unchanged" (`plans/1-sectionlist.md`). |
| Core is consumed via package subpath + `internal`, never bundled | One core module instance, so one `ContextState` and one set of singletons (`591c769f`, `postbuild.ts` patterns). |
| Root `@legendapp/list` export removed | `c34fcb89`. Previously "remains supported, but uses looser typings for broad compatibility" (removed README line). Platform-specific subpaths give strict typings. |
| `/keyboard` built on `KeyboardChatScrollView`, old code kept as `/keyboard-legacy` | `42033988` → reverted `1b0ae212` → reapplied `9a28ec0b`, then `8696f901`. **Inferred:** offload keyboard tracking to the library while keeping old-arch and old-library users working. |
| Reanimated sticky headers use a UI-thread transform | Avoids JS-thread lag. Used only on the new architecture (`IsNewArchitecture` gate in `LegendListForwardedRef`). |
| Scroll-offset tracker mounted conditionally as a child | Avoids calling a hook conditionally, and skips `useScrollViewOffset` when it isn't needed (comment above `ReanimatedScrollOffsetTracker`). |
| `useStableRenderComponent` for custom scroll components | Prevents remounting the scroll subtree when a `renderScrollComponent` callback changes identity (test "keeps the custom scroll subtree mounted…"). |

## Where to change things

| To… | Start at |
|---|---|
| Expose another core helper to integrations | `internal` in `src/entrypoints/shared.ts` |
| Add a new integration subpath | `tsup.config.ts` `nativeEntryPoints` + `external`, `package.json` `"exports"`, `postbuild.ts` `INTEGRATION_ENTRYPOINTS`, `tsconfig.json` `paths`, `example/metro.config.js` |
| Change Reanimated shared values | `AnimatedLegendListSharedValues` + `useAnimatedLegendListSharedValuesSync` in `src/integrations/reanimated.tsx`. The state must also be `listen`-able (`LegendListListenerType` in `src/state/state.tsx`). |
| Change item layout animations or recycled reuse | `ReanimatedPositionView` (reanimated.tsx), `shouldAvoidAssignedContainerReuse` in `src/utils/findAvailableContainers.ts` |
| Change Reanimated sticky headers | `ReanimatedPositionViewSticky`; core counterpart `PositionViewSticky` in `src/components/PositionView.native.tsx`, `getStickyPushLimit` |
| Change chat keyboard spacing or composer inset | `KeyboardAwareLegendList` / `useKeyboardChatComposerInset` in `src/integrations/keyboard.tsx`; core side `reportContentInset` in `src/utils/createImperativeHandle.ts`, `src/state/getContentInsetEnd.ts`, `src/core/updateAnchoredEndSpace.ts` |
| Fix legacy keyboard animation | `useKeyboardHandler` block in `src/integrations/keyboard-legacy.tsx` |
| Change section flattening, keys, or sticky indices | `buildSectionListData` in `src/section-list/flattenSections.ts` |
| Change SectionList prop mapping, viewability, or `scrollToLocation` | `SectionList` in `src/section-list/SectionList.tsx` |
| Change build guards | `prebuild.ts` (`findLocalIntegrationImports`, `findDirectRootPackageImports`), `postbuild.ts` (`INTEGRATION_REPACKAGED_CORE_PATTERNS`) |

## Tests

- `__tests__/integrations/`:
  - `reanimated.sharedValues.test.tsx`: seeding, `listen` updates, when the offset is tracked.
  - `reanimated.itemLayoutAnimation.test.tsx`: `positionComponentInternal` stability, recycle skip, old-arch bridge, sticky transform.
  - `keyboard.behavior.test.tsx`: `blankSpace` and `anchoredEndSpace`, `reportContentInset`, composer inset.
  - `keyboard.interactive-state.test.tsx`: prop forwarding; legacy cancelled-dismiss and initial-close handling.
  - `keyboard.onScroll.test.tsx`: export surface of `/keyboard` vs `/keyboard-legacy`.
  - `reanimated.isolation.test.ts`: mock-order isolation.
- `__tests__/section-list/SectionList.test.tsx`: flattening and sticky indices, `scrollToLocation` mapping, viewability translation, `getFixedItemSize`/`renderItem` info. `LegendList` is mocked, so this does not cover real virtualization. Sticky-header data-change coverage lives in `__tests__/components/PositionView.*.test.tsx` and `__tests__/core/calculateItemsInView.test.ts`.
- Run: `bun test __tests__/integrations __tests__/section-list` (35 pass at 3d80d167, ~0.3s).
- Build guards run only via `bun run build` (`prebuild` → `tsup` → `postbuild`). `bun test` does not run them.
- Visual checks: `example/screens/fixtures/` (`chat-keyboard*.tsx`, `ai-chat-keyboard.tsx`, `layout-animation.tsx`, `section-list-fixed-size.tsx`, `countries-with-headers-sticky.tsx`) and `example-oldarch/app/chat-keyboard*/` for keyboard-legacy.
