<!-- index-codebase: generated from 3d80d167 on 2026-10-03. Code wins when it disagrees with this doc. -->
# Legend List: Design Decisions

These are the cross-cutting choices that shaped the system. Area-specific decisions are in each `areas/*.md` file. Commit bodies in this repo are often detailed, so `git show <sha>` usually says more.

| Decision | Why | Evidence |
|---|---|---|
| **Hand-rolled signal store plus a mutable `InternalState`, not React state** | `LegendListInner` subscribes to no signals. Scrolling, measuring, and recycling `set$` fine-grained keys, so only the affected container or position view re-renders. A value becomes a signal only if something must re-render or a public listener must fire. | d674440c, 952bbb5b (replaced Legend State), 7f65cf0a, 621f42b2. See [areas/state.md](areas/state.md) |
| **`InternalState` lives on the context, created once** | StrictMode double render must not create two states, because all core functions assume one. | c839edd2 |
| **Physical container slots are recycled; `recycleItems` only controls remounting** | A bounded pool of absolutely positioned slots is the virtualization primitive. Remounting is the safe default for stateful items. | `Container.tsx` `key={recycleItems ? undefined : itemKey}`; [areas/virtualization.md](areas/virtualization.md) |
| **Estimate-first sizing, corrected by measurement, with per-type averages** | Works with dynamic sizes without a measurement pass up front. Per-type averages beat a single estimate. | c00f0469 (removed `getEstimatedItemSize`) |
| **One coordinated post-commit measurement pass (new arch and web)** | Per-row `onLayout` let recycled rows recalculate against partly updated sizes. | bbad389e |
| **Positions via `top`/`left`, not `transform`** | Transforms caused Android input, macOS, and keyboard-animation problems for little performance gain. Sticky headers keep `transform` for native-driver animation. | d1f38ef2, 0a234ee2 |
| **All scroll corrections go through `requestAdjust` → a scroll-adjust channel** | Native moves a far-away anchor view inside RN's own `maintainVisibleContentPosition`, so the correction lands in the same commit. Web reconciles `scrollBy` against the live DOM offset. | cf942815, f2d4543a (#463). See [areas/scroll-stability.md](areas/scroll-stability.md) |
| **Size MVCP on by default; data MVCP opt-in** | Items resizing above the viewport should never jump. Anchoring on prepend changes semantics, so users opt in. | bbca7164, 15a7bda3 |
| **Hidden-convergence initial scroll ("bootstrap")** | Visible scroll-measure-rescroll retries caused jumps. Content stays at opacity 0 until two passes agree. | plan 10 (`git show 8ff19f38:plans/10-bootstrap-reveal-initial-scroll.md`), 936f1b20 |
| **Monotonic epochs for data and dataset changes** | Change detection mutates state during render, and React may discard and retry renders, so local booleans got lost. | 660f67a4, 3c0f5d9c |
| **Imperative scrolls: token-owned, promise always resolves, newer supersedes older (including initial scroll)** | One settlement policy for supersession and unmount. Stale timers must not finish a newer scroll. | 07b3edee, ad1d20dc, issue #508 (`issues/issue-508-*.md`) |
| **Subpath-only entrypoints with platform-split types** | RN users get `ScrollViewProps`, DOM users get `CSSProperties` with no `react-native` in their types. The loose root entry was dropped. | `plans/4-types.md`, `plans/5-types-2.md`, c34fcb89 |
| **One runtime for both platforms, chosen by file suffix** | `.native.ts(x)` vs plain files, resolved by the native tsup build, Metro, Vite, or test mocks. No runtime platform detection for structural choices. | `tsup.config.ts`, `__tests__/setup.ts` |
| **Integrations consume core through the published subpath plus `internal`** | A bundled copy of core would create a second `ContextState` and break hooks inside the core tree. Build guards enforce this. | 591c769f, d3956843, 72542d28 |
| **Web treated as "new architecture"** | DOM layout reads are synchronous, like Fabric. | `src/constants-platform.ts` |
| **DOM order follows visual order (LIS plus `moveBefore`)** | Recycling scrambles tab order and screen-reader order. `moveBefore` keeps focus and iframe state. | 4e203bb3 |
| **No native code** | "100% JS" is a product goal (README), so every platform feature is built on ScrollView/DOM primitives. | README |

## Plans and their status

`plans/` holds design plans executed under the `AGENTS.md` plan cadence. Older plans (10, 13, …) were deleted from the tree but are still in git history. Some never fully landed. For example, plan 13's `phase` states for initial scroll were not adopted ([areas/initial-scroll.md](areas/initial-scroll.md)), and `plans/6-anchored-end-space.md` describes an `includeInEndInset` option that was later removed. Treat plans as history, not as a spec of current behavior.
