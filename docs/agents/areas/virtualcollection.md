<!-- index-codebase: generated from 024b474ce92 on 2026-10-03. Code wins when it disagrees with this doc. -->
# VirtualCollection (VirtualColumn / VirtualRow)

**Purpose:** Experimental list primitives (internal codename "Fling") that render a collection as a sequence of `VirtualView`s plus one hidden "spacer" `VirtualView` standing in for the items not yet mounted. Native code decides visibility for each item; this layer only decides *how many* items to mount and memoizes their elements. It is **not** a scroll container, and it does not handle viewability, scroll-to, or headers/footers.
**Paths:** `packages/react-native/src/private/components/virtualcollection/` (`Virtual.js`, `VirtualCollectionView.js`, `FlingConstants.js`, `column/VirtualColumn.js`, `column/VirtualColumnGenerator.js`, `row/VirtualRow.js`, `row/VirtualRowGenerator.js`)
**Depends on:** [virtualview.md](virtualview.md) (`VirtualView`, `createHiddenVirtualView`, `VirtualViewMode`, `ModeChangeEvent`), DOM node APIs (`ReactNativeElement`, `previousElementSibling`, `getBoundingClientRect`), `Dimensions` · **Used by:** nothing in this repo. Consumers are Meta-internal (commit 231d53ac13e mentions "40 consumer files"). Contrast with [virtualized-list.md](virtualized-list.md), [flatlist-sectionlist.md](flatlist-sectionlist.md), [viewability.md](viewability.md).

## How it works

### Three layers
1. **Data: `VirtualCollection<T>`** (`Virtual.js`): an interface with only `size` and `at(index)`. It is used instead of an array so a consumer can back it with lazily computed or paged data, and no array of N items has to be allocated (doc comment: "without requiring that each item be eagerly (or lazily) allocated"). Contract: `size` stays constant for the object's lifetime, `at(i)` returns the **same item object** every time and throws for out-of-range indexes. `VirtualArray` is the convenience implementation: it copies its input (`[...input]`) and throws `RangeError`. Its comment says it is "not recommended for larger arrays" because elements are allocated eagerly.
2. **Factory: `createVirtualCollectionView(VirtualLayout, generator)`** (`VirtualCollectionView.js`) returns a component. You configure it along two axes:
   - `VirtualLayout`: a component that receives `children` (mounted item nodes) and `spacer` and must render both. Extra props on the resulting component pass through to it (`...layoutProps`).
   - `generator: VirtualCollectionGenerator`: `{initial: {itemCount, spacerStyle}, next(event) => {itemCount, spacerStyle}}`. It controls how many items to mount first and how many more to mount each time the spacer comes into range. **"Generator" means this pagination strategy object. It is not a JS `function*` and has nothing to do with infinite or streamed data.**
3. **Instances: `VirtualColumn` / `VirtualRow`**: thin calls to the factory. Their layouts (`VirtualColumnLayout`, `VirtualRowLayout`) are identical fragments (`<>{children}{spacer}</>`). The only difference is the generator: `VirtualColumnGenerator` measures height/`y`, `VirtualRowGenerator` measures width/`x`. The scroll direction comes entirely from the enclosing ScrollView.

### Flow: initial mount, then growth
1. `VirtualCollectionView` keeps `desiredItemCount` in state, starting at `Math.ceil(initial.itemCount)`. For both built-ins that is `INITIAL_NUM_TO_RENDER` = 7 (`FlingConstants.js`).
2. It mounts `min(desiredItemCount, items.size)` items by calling `items.at(i)` only for those indexes. Each item goes through `renderItem`, which returns `<VirtualView key={key} nativeID={key} removeClippedSubviews>{children(item, key)}</VirtualView>`.
3. If any items are left, it renders a single `VirtualCollectionSpacer` with `virtualItemCount = items.size - mounted`. The spacer renders `SpacerView`, a `createHiddenVirtualView(spacerStyle(virtualItemCount))`: a VirtualView that **starts hidden**, has no children, and is sized as an estimate. For example, the column's initial estimate is `height = n * FALLBACK_ESTIMATED_HEIGHT`, where that constant is window height / 7.
4. Native tracks the spacer like any VirtualView. When its rect overlaps the prerender or visible region of the parent ScrollView, native fires `onModeChange` with `mode` = `Prerender` or `Visible`, plus `targetRect` (the spacer) and `thresholdRect` (the visible or prerender rect). See [virtualview.md](virtualview.md). VirtualView wraps `Prerender` in `startTransition` and handles `Visible` synchronously, so pagination is low-priority unless the spacer is actually on screen.
5. `VirtualCollectionSpacer.handleModeChange` calls `next(event)`. `VirtualColumnGenerator.next`:
   - computes `heightToFill`, the overlap of `targetRect` and `thresholdRect` along y;
   - estimates item height by walking up to 3 `previousElementSibling`s whose `nodeName` starts with `RN:VirtualView`, and averaging `getBoundingClientRect().top` deltas. With no siblings it falls back to `FALLBACK_ESTIMATED_HEIGHT`;
   - returns `itemCount = heightToFill / itemHeight` plus a `spacerStyle` that uses the measured height.
6. The spacer replaces its `SpacerView` with one built from the refined `spacerStyle`, and calls `onRenderMoreItems(min(ceil(itemCount), virtualItemCount))`. That adds to `desiredItemCount`.
7. The re-render mounts more items. `virtualItemCount` shrinks, so `SpacerView` is a new component type (also re-memoized on `itemCount`). The spacer therefore **remounts as a fresh hidden VirtualView**, resized from the new estimate. If it still overlaps the threshold, native fires again and the loop repeats. It stops when the spacer moves out of range or every item is mounted, at which point the spacer renders `null`.

### Who unmounts content when scrolling away?
Not this layer. `desiredItemCount` **only grows**: the collection never removes a mounted item's `VirtualView` shell. Off-screen items are virtualized per item by `VirtualView` itself. In `Hidden` mode it drops its children and keeps the size through `hiddenStyle` (`minHeight`/`minWidth` of the last `targetRect`). The cost per item that stays mounted is therefore one native view plus one memoized element.

### Memoization
`renderItem` is wrapped in `useMemoCallback`, which uses a `WeakMap` keyed by **item object identity** (the `memoize` helper). The same item gets back the same React element, so React skips reconciling that item when the list re-renders after pagination. The cache is rebuilt whenever `children`, `itemToKey`, or `removeClippedSubviews` changes identity.

## Key types and entry points

| Symbol | File | Role |
| --- | --- | --- |
| `Item`, `VirtualCollection<T>` | `Virtual.js` | Data interface (`size`, `at`). Exported as types `unstable_VirtualItem`, `unstable_VirtualCollection` |
| `VirtualArray` | `Virtual.js` | Array-backed `VirtualCollection`. Exported as `unstable_VirtualArray` |
| `createVirtualCollectionView` | `VirtualCollectionView.js` | Factory: layout + generator → list component. `unstable_createVirtualCollectionView` |
| `VirtualCollectionGenerator`, `VirtualCollectionLayoutComponent`, `VirtualCollectionViewComponent` | `VirtualCollectionView.js` | Factory types (exported with the `unstable_` prefix in `index.js.flow`) |
| `VirtualCollectionSpacer` / `createSpacerView` | `VirtualCollectionView.js` (internal) | Hidden VirtualView that triggers pagination |
| `VirtualColumn`, `VirtualColumnGenerator` | `column/` | Vertical instance and its strategy. Both exported (`unstable_VirtualColumn`, `unstable_VirtualColumnGenerator`) |
| `VirtualRow`, `VirtualRowGenerator` | `row/` | Horizontal instance (`unstable_VirtualRow`). **`VirtualRowGenerator` is not exported** from `index.js` |
| `DEFAULT_INITIAL_NUM_TO_RENDER` (7), `INITIAL_NUM_TO_RENDER`, `FALLBACK_ESTIMATED_HEIGHT/WIDTH` | `FlingConstants.js` | Initial count and fallback size estimates. Only the default is exported (`unstable_DEFAULT_INITIAL_NUM_TO_RENDER`) |

Public API: `index.js` exposes lazy getters and `index.js.flow` exposes the exports and types. None of this appears in `ReactNativeApi.d.ts`, because `scripts/js-api/build-types/transforms/typescript/stripUnstableApis.js` removes `unstable_*`/`experimental_*` symbols from the generated TS. Exports from these modules are allowlisted in `packages/eslint-plugin-react-native/utils.js` (the deep-import map of public modules).

## Invariants and gotchas
- **Put it inside a ScrollView.** The component renders a fragment, not a scroll container. Native VirtualViews find their container by walking superviews for one that responds to `virtualViewContainerState` (iOS `RCTVirtualViewComponentView._getParentVirtualViewContainer`; Android uses `VirtualViewContainer`). **Inferred:** without such an ancestor no mode change fires, so only the first 7 items ever mount. `VirtualRow` needs a horizontal ScrollView, or at least a row layout, because its layout fragment does not set direction.
- **Requires DOM APIs / Fabric.** `next()` throws `"Expected target to be a ReactNativeElement…"` if the event target isn't a `ReactNativeElement`.
- **Default key reads `item.key`**, not `id`. `defaultItemToKey` throws a `TypeError` if `item.key` isn't a string. Its message says `'id'`, and so does the `Item` doc comment in `Virtual.js`; both are misleading. Pass `itemToKey` if items have no `key: string`. The key also becomes each item VirtualView's `nativeID`.
- **Item identity matters.** `at(i)` must return the same object on every call (part of the `VirtualCollection` contract), or the WeakMap memo misses and the item re-renders. Items must be objects (the `WeakMap` key / `interface {}` bound), not primitives.
- **Pass a stable `children` render function** (e.g. via `useCallback`). An inline arrow invalidates the whole memo cache on every parent render.
- **Don't mutate a collection; replace it.** `size` must stay constant for the collection object's lifetime. When the list grows, pass a new `VirtualCollection`. `desiredItemCount` survives the swap, because it is state on the same component instance.
- **`testID` is only used for the spacer's `nativeID` (`${testID}:Spacer`).** No view on the component itself receives it.
- **Size estimates are coarse.** `FALLBACK_ESTIMATED_*` is computed once at module load from `Dimensions.get('window')` and does not update on rotation. Later estimates average only the last ≤3 mounted siblings that are VirtualViews. Variable-height items make the spacer size, and so the scrollbar, imprecise until everything is mounted.
- **Native gap:** `RCTVirtualViewComponentView.mm` has `TODO(T202601695)`: elements in the ScrollView *outside* a VirtualColumn that change size are not handled when recomputing modes.
- **Not supported (verified: absent from the component's props and code):** `scrollToIndex`/`scrollToOffset` (no imperative handle), viewability callbacks, `onEndReached` (pagination only covers items already in `items`), headers/footers/separators/empty component, sticky headers, `inverted`, `getItemLayout`/known sizes, multi-column grids (possible only through a custom layout + generator), and unmounting items far from the viewport.
- The `Hidden` branch in `handleModeChange` is a no-op by design ("This should never happen; this starts hidden and otherwise unmounts"), because the spacer remounts after every pagination.

## Design decisions
- **Native visibility instead of JS windowing.** VirtualizedList/FlatList compute a render window in JS from `onScroll` offsets and `onLayout` measurements, and use spacer views for unrendered ranges ([virtualized-list.md](virtualized-list.md)). Here native computes each VirtualView's mode against visible/prerender rects (`VirtualViewContainerStateClassic.kt`; prerender rect = visible rect inflated by feature flag `virtualViewPrerenderRatio`, default 5). JS reacts only through `onModeChange`. **Inferred:** this removes the scroll-event round-trip and lets prerendering go through React transitions.
- **Interface, not array** (`Virtual.js` doc comments): avoids eager allocation and lets callers adapt any indexed source. `VirtualArray` is explicitly a convenience.
- **Factory plus strategy** (`createVirtualCollectionView` docblock): layout and pagination are separate so column, row, and custom layouts share the mounting and memo logic. `VirtualColumnGenerator` is exported so custom layouts can reuse the vertical strategy (commit 231d53ac13e lists it among the public re-exports).
- **`SpacerView` kept in a wrapper object** (inline NOTE): `useState` would otherwise treat a component function as an updater.
- **`VirtualArray` stores `at` as a closure** (inline NOTE): Flow does not allow `input` as a read-only instance property.
- **`VirtualColumn` / `VirtualRow` repeat the component type annotation** rather than using `VirtualCollectionViewComponent<...>`, because of a Flow generic-resolution limitation (`TODO: Figure out component generic resolution`).
- **History:** moved from Meta-internal `RKJSModules/Libraries/List/` into OSS in 231d53ac13e (#56256). Debug overlay `FlingItemOverlay` and flag `enableVirtualViewDebugFeatures` were removed in 083fd99ba4f (#56946) because the flag was dead. `getScrollParent`/`isScrollableNode` were removed in 3818b20eeda (#58768): "scroll-parent discovery depends on application-specific scrollability semantics and should be implemented in userland."
- **"Fling" is a codename, not physics.** `FlingConstants.js` has no velocity or deceleration logic, only the initial count and fallback sizes. The name matches commit titles such as "Fling: Rename VirtualView…" (1c94a36a83b) and the removed `rn_fling` MobileConfig. **Inferred:** `INITIAL_NUM_TO_RENDER` exists apart from `DEFAULT_INITIAL_NUM_TO_RENDER` as a leftover hook for an internal override.

## Where to change things

| To… | Start at |
| --- | --- |
| Change the initial item count or fallback size estimates | `FlingConstants.js` |
| Change how many items each pagination adds, or how item size is estimated | `column/VirtualColumnGenerator.js`, `row/VirtualRowGenerator.js` (keep the two in sync) |
| Change mounting, keys, memoization, or spacer behavior | `VirtualCollectionView.js` (`VirtualCollectionView`, `VirtualCollectionSpacer`, `memoize`) |
| Add a new layout (grid, wrapped row) | Call `createVirtualCollectionView` with a custom layout component and generator |
| Change the data contract | `Virtual.js`, then the type exports in `index.js.flow` |
| Change per-item visibility, prerender range, or hidden placeholder size | VirtualView: `src/private/components/virtualview/VirtualView.js` and native `VirtualViewContainer*` ([virtualview.md](virtualview.md)) |
| Add or rename a public export | `packages/react-native/index.js`, `index.js.flow`, `packages/eslint-plugin-react-native/utils.js` |

## Tests
There are **no tests or examples** for VirtualCollection: no `__tests__` under `virtualcollection/`, and no references in `packages/rn-tester` or `private/`. The closest coverage is the VirtualView Fantom test, `yarn fantom packages/react-native/src/private/components/virtualview/__tests__/VirtualView-itest.js`. A new Fantom itest beside it, under `virtualcollection/__tests__/`, would be the natural place for a pagination test.
