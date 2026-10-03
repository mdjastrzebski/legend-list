<!-- index-codebase: generated from 3d80d167 on 2026-10-03. Code wins when it disagrees with this doc. -->
# Legend List: Using and Extending

The user-facing API reference is the docs site (https://www.legendapp.com/open-source/list) and the README. This file covers what an agent needs when changing or embedding the library: where each surface lives, defaults as the code sets them, and the limits visible in code.

## Platforms and integrations

| Platform / integration | Versions in use | Where it lives |
|---|---|---|
| React Native, new architecture (Fabric) | `example/`: Expo ~54, RN 0.81.5, React 19.1 | `.native.ts(x)` files; `IsNewArchitecture` from `global.nativeFabricUIManager` in `src/constants-platform.native.ts` |
| React Native, old architecture (Paper) | `example-oldarch/`: RN 0.79.3 | Same files, branching on `IsNewArchitecture` (Animated positions, `onLayout`-only measurement, render-time allocation) |
| React DOM (web) | `example-web/`: React 18.2 with Vite | Plain `.ts(x)` files. Web counts as "new architecture" (`src/constants-platform.ts`) |
| react-native-web | via the `browser` condition | `@legendapp/list/react-native` resolves to the DOM build (`react-native.web.js`), not RN-Web views |
| Reanimated | dev dep `~4.2.1` | `@legendapp/list/reanimated` → `src/integrations/reanimated.tsx` |
| react-native-keyboard-controller | dev dep `^1.22.5`; `/keyboard` needs ≥1.21.7 | `@legendapp/list/keyboard`, `/keyboard-legacy` → `src/integrations/keyboard*.tsx` (both also need Reanimated) |
| RN `Animated` | — | `@legendapp/list/animated` |
| SectionList API | — | `@legendapp/list/section-list` → `src/section-list/` |

Package facts:
- There is **no root `@legendapp/list` import**. Use `/react` or `/react-native` (commit c34fcb89).
- `peerDependencies` lists only `react`. Reanimated and keyboard-controller aren't declared, even as optional peers, so apps must install them.
- The integration subpaths are native-only builds with no `browser` condition.
- The only runtime dependency is `use-sync-external-store`, through its shim for older React.

Details: [areas/public-api.md](areas/public-api.md), [areas/integrations.md](areas/integrations.md).

## Configuration

Everything is configured through props. `LegendListInner` in `src/components/LegendList.tsx` sets the defaults and normalizes the config objects (`src/utils/normalize*.ts`). Before you change one, note that some JSDoc `@default` tags in `src/types.base.ts` disagree with the code (see "Limitations and gotchas" below).

| Setting | Default (code) | Notes |
|---|---|---|
| `drawDistance` | 250 | Render buffer in px: 0.5× behind and 1.5× ahead of the scroll direction. Capped at 50 until the list is ready, then prewarmed. Changing it applies on the next pass. |
| `estimatedItemSize` | 100 | Superseded by per-`getItemType` running averages once items are measured. `getEstimatedItemSize` was removed. |
| `getFixedItemSize` | — | Exact sizes. They skip measurement and count as known. |
| `recycleItems` | `false` (dev warning if omitted) | Only decides whether the item subtree remounts. Container slots are always reused. |
| `keyExtractor` | index string | Index keys tie measured sizes to positions, not items. A data change without a `keyExtractor` wipes sizes. |
| `numColumns` | 1 | Row-major grid with row height = tallest item, no masonry. Changing it clears every size cache. |
| `maintainVisibleContentPosition` | `{ data: false, size: true }` | Size anchoring is on by default. Use `true` to also anchor on data changes (prepend). |
| `maintainScrollAtEnd` | `false` | `true` (or an object without `on`) enables the `dataChange`, `layout`, `itemLayout` and `footerLayout` triggers. `animated` defaults to false. |
| `maintainScrollAtEndThreshold` | 0.1 | Fraction of the viewport. |
| `onStartReachedThreshold` / `onEndReachedThreshold` | 0.5 | Fraction of the viewport. 1.3× hysteresis plus a per-gesture gate. |
| `alignItemsAtEnd` | `false` | Uses padding when vertical with data, flex `justify-content: flex-end` otherwise. |
| `initialScrollIndex` / `initialScrollAtEnd` / `initialScrollOffset` | — | Index and end targets use the hidden "bootstrap" convergence. A bare offset scrolls directly. |
| `dataKey` / `dataVersion` / `itemsAreEqual` | — | Change `dataKey` for a different dataset (resets initial scroll and readiness, fires `onReady` again) and `dataVersion` for in-place mutation. `itemsAreEqual` suppresses spurious data changes. |
| `anchoredEndSpace` | — | Trailing space that lets an anchor row sit at the top. Single column only. |
| `alwaysRender` | — | `{ top, bottom, indices, keys }` stay mounted. |
| `viewabilityConfig` | `viewAreaCoveragePercentThreshold: 0` | Give each config an `id`. |
| `useWindowScroll`, `scrollElement`, `contentInsetEndAdjustment` | — | Web-only, silently ignored on native. The first two are also ignored when `renderScrollComponent` is set. |
| `experimental_adaptiveRender`, `experimental_hideItemsUntilMeasured` | off | Experimental. |
| `ENABLE_DEVMODE`, `ENABLE_DEBUG_VIEW` | `IS_DEV && false` | Compile-time debug overlays in `src/constants.ts`. Flip them locally only. |

There are no config files, environment variables, or global settings. `IS_DEV` reads `process.env.NODE_ENV`/`MODE`, then `__DEV__` (`src/utils/devEnvironment.ts`).

## Extension points

- **Render hooks for items:**
  - `useRecyclingState` and `useRecyclingEffect` reset per item when recycling.
  - `useViewability` and `useViewabilityAmount` report viewability.
  - `useIsLastItem`, `useListScrollSize` and `useSyncLayout`.
  - `useAdaptiveRender` and `useAdaptiveRenderChange`.

  All live in `src/state/ContextContainer.ts` and are exported from both platform entrypoints.
- **Ref API** (`src/utils/createImperativeHandle.ts`):
  - scroll methods that return promises; they always resolve, and a newer request supersedes an older one;
  - `getState()` (a snapshot) with `listen(signal)`/`listenToPosition`;
  - `clearCaches`, `setItemSize`, `reportContentInset`, `setScrollProcessingEnabled`, `setVisibleContentAnchorOffset`, `getAnimatableRef`/`getNativeScrollRef`.
- **`renderScrollComponent`:** swaps the scroll view. Wrapped by `useStableRenderComponent`, so an inline function doesn't remount it. Native sticky headers expect an `Animated`-compatible `onScroll` (**Inferred**).
- **`overrideItemLayout`:** per-item `span` in grids. **`getItemType`:** separate container pools and size averages per type.
- **Writing a new integration:** import only `@legendapp/list/react-native` plus its `internal` bridge (never `@/…`). Pass behavior in through untyped `*Internal` props (`positionComponentInternal`, `stickyPositionComponentInternal`, `anchoredEndSpaceOwnerInternal`). Then register the subpath in `tsup.config.ts`, `package.json` `exports`, and `postbuild.ts`. See [areas/integrations.md](areas/integrations.md).
- **What needs a fork:** the measurement pipeline, container allocation policy, the MVCP or adjust channel, and draw-buffer heuristics have no plugin hooks. Changing them means editing `src/core`.

## Limitations and gotchas

- No masonry. `anchoredEndSpace` is single-column only. Multi-column lists never reuse cached positions.
- Web: `RefreshControl` renders nothing (no pull-to-refresh), `I18nManager.isRTL` is always false (use the `rtl` prop), web sticky headers don't push each other off, snapping is disabled with an external scroll owner, Tailwind `gap-*` classes are unsupported (use numeric gaps in `contentContainerStyle`), and an unbounded list height gets a dev warning.
- Header-resize compensation above the viewport exists only on web (`setHeaderSize`).
- Reanimated UI-thread sticky headers need the new architecture. Old architecture falls back to RN `Animated`.
- SectionList flattens headers, footers, and separators into real rows, so flat indices differ from RN's. Use `scrollToLocation`.
- With the default MVCP, prepending data does **not** keep content steady. Opt in with `maintainVisibleContentPosition: true` or `{ data: true }`.
- Doc vs code mismatches found at 3d80d167:
  - `estimatedItemSize` JSDoc says `@default undefined`, but the code uses 100.
  - `scrollToOffset` JSDoc says `animated` defaults to true, but the code treats undefined as false.
  - `maintainVisibleContentPosition.data` JSDoc implies only `data` sets the RN prop, but the default `size: true` already does.
- Behaviors that may be unintended:
  - A user drag doesn't cancel an in-flight `scrollToIndex`; only end-following is interrupted.
  - `onEndReached` can fire during an in-flight `scrollToEnd`, while `onStartReached` is suppressed during scrolls.
  - A `viewabilityConfig` without `id` may never reach `useViewability` (**Inferred**).

## Security and trust

- **Runtime:** the library makes no network calls, has no telemetry, and writes no files. It only reads and mutates the scroll view or DOM it renders. On web it injects one `<style id="legend-list-scrollbar-axis-hidden-style">` and observes layout with `ResizeObserver`/`MutationObserver`.
- **Install:** no `postinstall` or other lifecycle scripts reach users. The published `dist/package.json` has `scripts` stripped.
- **Maintainer tooling** (local only, never shipped):
  - `prep-changelog.ts` runs `codex exec --ask-for-approval never --sandbox workspace-write` or `claude -p` on the repo to draft `CHANGELOG.md`.
  - `create-release.ts` runs `git add .`, commits, pushes, and creates a GitHub release with `gh`.
  - `example-web/tests/recycle-image-priority.mjs` drives Chrome via Playwright against a local Vite server.
  - `.vscode/mcp.json` registers the `argent` MCP server for device QA flows.
- **Vulnerability reporting:** no `SECURITY.md`. Issues go to https://github.com/LegendApp/legend-list/issues (`package.json` `bugs`).
