<!-- index-codebase: generated from 024b474ce92 on 2026-10-03. Code wins when it disagrees with this doc. -->

# VirtualizedList core

**Purpose:** The windowing engine behind `FlatList` and `SectionList`. It decides which item cells are mounted (the render window), keeps the rest as sized blank spacers, measures cells, calls the edge-reached callbacks, and coordinates nested lists. It does **not** decide viewability (see [viewability.md](viewability.md)), the column and section shaping (see [flatlist-sectionlist.md](flatlist-sectionlist.md)), or the native scroll anchoring for `maintainVisibleContentPosition` (see [MVCP DESIGN.md](../../../packages/virtualized-lists/__docs__/DESIGN.md)). The newer native-driven virtualization primitives are separate: [virtualview.md](virtualview.md), [virtualcollection.md](virtualcollection.md).

**Paths:** `packages/virtualized-lists/` (published as `@react-native/virtualized-lists`): `Lists/VirtualizedList.js` (the hotspot), `Lists/VirtualizeUtils.js`, `Lists/CellRenderMask.js`, `Lists/ListMetricsAggregator.js`, `Lists/VirtualizedListCellRenderer.js`, `Lists/VirtualizedListContext.js`, `Lists/ChildListCollection.js`, `Lists/StateSafePureComponent.js`, `Lists/VirtualizedListProps.js`, `Utilities/`, `index.js`, `index.d.ts`, `Lists/VirtualizedList.d.ts`, `types_generated/`. Re-exported by `packages/react-native/Libraries/Lists/VirtualizedList.js`.

**Depends on:** `react-native` (as a peer dependency: `ScrollView`, `View`, `RefreshControl`, `I18nManager`, plus feature flags through `react-native/react-private-interface`), and the ViewabilityHelper and FillRateHelper ([viewability.md](viewability.md)). **Used by:** [flatlist-sectionlist.md](flatlist-sectionlist.md) (`FlatList`, `VirtualizedSectionList`/`SectionList`), and `Modal` (uses `VirtualizedListContextResetter`).

## How it works

### State model
`VirtualizedList` is a class component (it extends `StateSafePureComponent`). Its React state (`State` in `VirtualizedList.js`) is:

| Field | Meaning |
| --- | --- |
| `cellsAroundViewport: {first, last}` | Inclusive range of item indices around the viewport, including overscan. `{first: 0, last: -1}` means an empty range. |
| `renderMask: CellRenderMask` | Sparse list of `{first, last, isSpacer}` regions covering `[0, itemCount)`. It is the union of `cellsAroundViewport`, the initial region, the closest sticky header, and the focused-cell region. `render()` turns it into cells and spacers. |
| `firstVisibleItemKey` | Key of the item at `maintainVisibleContentPosition.minIndexForVisible` (default 0). Used to detect prepends. |
| `pendingScrollUpdateCount` | When `> 0`, the JS scroll offset is stale. The window, viewability, and edge callbacks are frozen until the next scroll event. |

Everything else is instance-local and mutable: `_scrollMetrics` (offset, visibleLength, velocity, dt, zoomScale, crossAxisLength), `_listMetrics: ListMetricsAggregator`, `_cellRefs`, `_indicesToKeys`, `_nestedChildLists`, `_lastFocusedCellKey`, `_hiPriInProgress`, `_updateCellsToRenderTimeoutID`, `_sentStart/EndForContentLength`, `_hasMore`.

### Flow: scroll event → new render window
1. The native `ScrollView` fires `onScroll` → `VirtualizedList._onScroll`. The list sets `scrollEventThrottle ?? 0.0001`, because iOS needs a non-zero throttle to send more than one event.
2. `_onScroll` forwards the event to the nested child lists (`_nestedChildLists.forEach(child => child._onScroll(e))`), then calls the user's `onScroll`. It computes `offset` with `_offsetFromScrollEvent`, which flips the offset for horizontal RTL. A nested list with the same orientation converts the parent's metrics with `_convertParentScrollMetrics`. The method then computes `dt` and `velocity` and replaces `_scrollMetrics`.
3. If `pendingScrollUpdateCount > 0`, it calls `setState({pendingScrollUpdateCount: 0})` (a reset, not a decrement).
4. It calls `_updateViewableItems` (handled by ViewabilityHelper), `_maybeCallOnEdgeReached`, and `_computeBlankness` (handled by FillRateHelper), then `_scheduleCellsToRenderUpdate()`.
5. `_scheduleCellsToRenderUpdate` takes one of two paths:
   - **High priority (no timer):** used when the cell sizes are known (`getAverageCellLength() > 0` or `getItemLayout` is set), `_shouldRenderWithPriority()` holds, and `!_hiPriInProgress`. `_shouldRenderWithPriority()` holds when the viewport has passed the edge of the rendered window, or is moving fast toward that edge and is within `threshold*visibleLength/2` of it. On this path the method clears any pending timer and calls `_updateCellsToRender()` directly.
   - **Low priority:** a single debounced `setTimeout(_updateCellsToRender, updateCellsBatchingPeriod ?? 50)`. There is no `Batchinator` any more; the timeout ID is the only batching mechanism.
6. `_updateCellsToRender` calls `setState(updater)`. Inside the updater, `_adjustCellsAroundViewport(props, state.cellsAroundViewport, state.pendingScrollUpdateCount)` does one of the following:
   - If `visibleLength` or `contentLength` is still unknown, it keeps the current window (trusting `initialNumToRender`).
   - If `disableVirtualization` is set, it returns `{first: 0, last: last + renderAhead}`. The window grows by `maxToRenderPerBatch` only when the end is within `onEndReachedThreshold` viewports.
   - If `pendingScrollUpdateCount > 0`, it returns the current window unchanged.
   - Otherwise it calls `computeWindowedRenderLimits(props, maxToRenderPerBatch, windowSize, prev, listMetrics, scrollMetrics)`.
   - If there are nested child lists, it caps `last` at the first cell whose child list still `hasMore()` (`_findFirstChildWithMore`).
   The updater then builds a new `CellRenderMask` with `_createRenderMask(props, cells, _getNonViewportRenderRegions(props))`, and returns `null` (no update) if neither the window nor the mask changed.
7. `componentDidUpdate` calls `_scheduleCellsToRenderUpdate()` again. Each committed batch therefore schedules the next one, until `computeWindowedRenderLimits` reaches a fixed point. `_hiPriInProgress` ensures that a high-priority update cannot trigger another high-priority update from its own `componentDidUpdate`.

### `computeWindowedRenderLimits` (`VirtualizeUtils.js`)
- It starts from the visible range. The overscan budget is `(windowSize - 1) * visibleLength`, split evenly before and after the viewport (`leadFactor = 0.5`). A velocity-based lead exists but is commented out: "Considering velocity seems to introduce more churn than it's worth."
- It grows `first` and `last` one cell at a time, up to the overscan bounds, and counts the cells that are new compared with `prev` (`newRangeCount`). It stops once `maxToRenderPerBatch` new cells have been added and growing further would add more. Cells that are already rendered are kept without counting against the budget. The sign of the velocity (`>1` / `<-1`) sets which side grows first.
- It uses binary search over `listMetrics.getCellMetricsApprox` (`elementsThatOverlapOffsets`), scaled by `zoomScale`.
- It throws `Bad window calculation` if the result does not contain the visible range or falls outside the overscan bounds.
- If the whole list ends before the overscan window, it returns the last `maxToRenderPerBatch + 1` items.

### Rendering (`render()`)
Children of the scroll view, in order: header cell (`$header`), the empty component (when `itemCount === 0`), then for each region of `renderMask.enumerateRegions()`, either a spacer `<View style={{height|width: n}}>` or `CellRenderer`s via `_pushCells`, then the footer (`$footer`).
- Spacer size comes from `getCellMetricsApprox(first).offset` to `getCellMetricsApprox(last)` end. Without `getItemLayout`, the **tail** spacer is clamped to `getHighestMeasuredCellIndex()`, which stops the user from scrolling into estimated space where content would jump.
- When `disableVirtualization` is set, no spacers are rendered (legacy behavior).
- `stickyHeaderIndices` from props are in "item + header" space (`+1` when `ListHeaderComponent` exists). `render()` remaps them to positions in the `cells` array, which includes spacers, before passing them to `ScrollView`.
- With `inverted`, `styles.verticallyInverted` or `styles.horizontallyInverted` is applied to the scroll view and to every cell, the header, the footer, and the empty component. `invertStickyHeaders` defaults to `inverted`, and `isInvertedVirtualizedList` is forwarded to native.

### Mask regions that are always rendered (`_createRenderMask`)
- **Initial region** `[initialScrollIndex ?? 0, +initialNumToRender-1]`. It is kept permanently, but only when `initialScrollIndex` is null or `<= 0`. This is the "scroll-to-top optimization" documented on `initialNumToRender` in `VirtualizedListProps.js`.
- **Closest sticky header** before `cellsAroundViewport.first` (`_ensureClosestStickyHeader`). Its layout can be offscreen while the header is visually pinned. The scan only runs when `stickyHeaderIndices` is non-empty (fe53279889b, #57210, a perf fix for long lists).
- **Focus region:** one viewport before and after `_lastFocusedCellKey` (`_getNonViewportRenderRegions`), so that keyboard and a11y focus can move without blanking (479053cb3ce). `getDerivedStateFromProps` does not add this region; the next `_updateCellsToRender` does.

### Layout measurement (`ListMetricsAggregator`)
- `CellRenderer` attaches `onLayout` only when `getItemLayout == null || debug || fillRateHelper.enabled()`. With `getItemLayout`, cells never report their layout and all metrics come from `getItemLayout`.
- `_onCellLayout` → `notifyCellLayout({cellIndex, cellKey, layout, orientation})` stores `{index, length, offset, isMounted}` keyed by **cellKey**, and updates `_measuredCellsLength/Count`, `_averageCellLength`, and `_highestMeasuredCellIndex`. If the layout changed, it schedules a window update. It also remeasures child lists in that cell, recomputes blankness, and updates viewability.
- `getCellMetrics(index)` returns a stored frame only if `frame.index === index`, which handles reordering. Otherwise it falls back to `getItemLayout`. `getCellMetricsApprox` extrapolates from the end of the highest measured cell plus `averageCellLength * gap`, which accounts for header offset.
- `notifyCellUnmounted` only sets `isMounted = false`. The metrics are kept, so spacers stay exact for cells that were measured earlier.
- `_onContentSizeChange` / `measureLayoutRelativeToContainingList` → `notifyListContentLayout` sets `_contentLength`.
- **Horizontal RTL:** offsets are flow-relative. `flowRelativeOffset` returns `contentLength - (x + width)`, and `cartesianOffset` converts back. This requires `_contentLength`, so `scrollToOffset` warns and does nothing in RTL before the content is laid out. `_scrollToParamsFromOffset` adds `visibleLength` so the offset is right-aligned.
- **Orientation change** (`_invalidateIfOrientationChanged`): a change of `rtl` clears only `_cellMetrics`. A change of `horizontal` resets everything, including `_contentLength` (e59a1d252c9, #57871).
- **Inverted is not modelled:** inversion is purely a transform. See the TODO on `ListOrientation`.

### initialScrollIndex
1. The constructor sets `cellsAroundViewport = _initialRenderRegion(props)` (it starts at the index), and `pendingScrollUpdateCount = 1` if `initialScrollIndex > 0`.
2. The first `_onContentSizeChange` → `_maybeScrollToInitialScrollIndex` runs once (`_hasTriggeredInitialScrollToIndex`): `scrollToIndex({animated: false})`, or `scrollToEnd` if the index is out of range. This step is skipped if the `contentOffset` prop is set.
3. The resulting native scroll event resets the pending count to 0, and windowing resumes. `scrollToIndex` beyond the highest measured cell without `getItemLayout` requires `onScrollToIndexFailed`, otherwise an invariant fails.

### Prop changes (`getDerivedStateFromProps`)
- If `itemCount === prevState.renderMask.numCells()`, it returns `prevState` unchanged. All recomputation, including MVCP prepend detection, happens **only when the item count changes**. Other data changes reach the window through `componentDidUpdate` → `_scheduleCellsToRenderUpdate`.
- Otherwise it recomputes `firstVisibleItemKey`. With MVCP, if the key changed, it finds the old key's new index (using the hint `itemCount - prevNumCells + minIndexForVisible`), shifts `cellsAroundViewport` by the delta, sets `pendingScrollUpdateCount = 1`, clamps with `_constrainToItemCount` (which widens `first` so the window holds at least `maxToRenderPerBatch` cells), and rebuilds the mask.
- MVCP details: [DESIGN.md](../../../packages/virtualized-lists/__docs__/DESIGN.md). `render()` also adds `+1` to `minIndexForVisible` when there is a `ListHeaderComponent`, because native counts the header as a child.

### Nested lists (`VirtualizedListContext`, `ChildListCollection`)
- Each list provides `VirtualizedListContext` (`getScrollMetrics`, `horizontal`, `getOutermostParentListRef`, `registerAsNestedChild`, `unregisterAsNestedChild`). Each cell wraps its content in `VirtualizedListCellContextProvider` to add `cellKey`.
- A child with the **same orientation** (`_isNestedWithSameOrientation`):
  - registers in `componentDidMount` under its parent's `cellKey`. The parent stores children in a `ChildListCollection`, a two-way map from cellKey to a set of lists.
  - renders a plain `<View>` instead of a `ScrollView` (`_defaultRenderScrollComponent`), and strips `onContentSizeChange` so the event does not bubble into the parent.
  - gets scroll events forwarded from the parent.
  - learns its offset through `measureLayout` against the outermost list's scroll ref (`measureLayoutRelativeToContainingList`), which is triggered by the parent's `_onCellLayout` / `_onLayoutFooter`.
  - ignores scroll events until its content length is known.
- A parent caps its window at the first cell whose child `hasMore()`, so nested lists fill before the parent mounts more cells. Per the comment, this avoids churn from many child lists mounting.
- A child with a different orientation scrolls on its own. `VirtualizedListContextResetter` (used by `Modal`) breaks the chain.
- In `__DEV__`, a same-orientation `VirtualizedList` inside a plain `ScrollView` logs an error, unless `scrollEnabled={false}`.

### Edge callbacks (`_maybeCallOnEdgeReached`)
- Called from `_onScroll`, `_onLayout`, `_onContentSizeChange`, and, only with `getItemLayout`, from `componentDidUpdate`, because you can scroll past rendered cells without a layout event.
- Requires real metrics, `pendingScrollUpdateCount === 0`, and the window to touch the edge (`last === itemCount-1` / `first === 0`).
- Fires at most once per content length (`_sentEndForContentLength`), and re-arms after the user leaves the threshold. Distances below `0.001` are rounded to 0.

## Key types and entry points

| Symbol | File | Role |
| --- | --- | --- |
| `VirtualizedList` | `Lists/VirtualizedList.js` | The component: state, render, event handlers, imperative `scrollTo*` |
| `_onScroll`, `_scheduleCellsToRenderUpdate`, `_updateCellsToRender`, `_adjustCellsAroundViewport` | `Lists/VirtualizedList.js` | Window update pipeline |
| `getDerivedStateFromProps`, `_createRenderMask`, `_constrainToItemCount`, `_initialRenderRegion` | `Lists/VirtualizedList.js` | State derivation on item-count changes |
| `computeWindowedRenderLimits`, `elementsThatOverlapOffsets`, `newRangeCount`, `keyExtractor` | `Lists/VirtualizeUtils.js` | Pure windowing math, default key extraction (`key` → `id` → index) |
| `CellRenderMask` | `Lists/CellRenderMask.js` | Sparse region set; `addCells`, `enumerateRegions`, `equals` |
| `ListMetricsAggregator`, `CellMetricProps`, `ListOrientation` | `Lists/ListMetricsAggregator.js` | Cell and content measurement cache and estimates |
| `CellRenderer` | `Lists/VirtualizedListCellRenderer.js` | Per-cell PureComponent: onLayout, focus capture, separators, unmount notification |
| `VirtualizedListContext`, `VirtualizedListContextResetter` | `Lists/VirtualizedListContext.js` | Nested-list wiring |
| `StateSafePureComponent` | `Lists/StateSafePureComponent.js` | Throws if `this.props`/`this.state` is read inside a functional `setState` updater |
| `*OrDefault` helpers | `Lists/VirtualizedListProps.js` | Defaults: `windowSize` 21, `initialNumToRender` 10, `maxToRenderPerBatch` 10, `on{Start,End}ReachedThreshold` 2 (viewports). `updateCellsBatchingPeriod` defaults inline to 50 ms. |

## Invariants and gotchas
- **`renderMask.numCells() === itemCount`** at all times. `getDerivedStateFromProps` uses this to detect count changes. `CellRenderMask.addCells` and `_createRenderMask` enforce `0 <= first`, `last >= first - 1`, and `last < itemCount` with invariants.
- **No `this.props`/`this.state` inside `setState` updaters:** `StateSafePureComponent` makes this throw. Pass `props`/`state` from the updater arguments, which is why `_adjustCellsAroundViewport` takes `pendingScrollUpdateCount` as a parameter (051a67784f2). Instance fields such as `_scrollMetrics` and `_cellRefs` are allowed.
- **`pendingScrollUpdateCount` is effectively a boolean.** `getDerivedStateFromProps` sets it to `1` and any scroll event resets it to `0`, because native scroll events coalesce (`dispatchUniqueEvent`), so N prepends deliver only one scroll event (5cb65244dc5, #57955). If native never sends a scroll event, for example when MVCP declines to adjust or a no-op `initialScrollIndex` scroll occurs, the window, viewability, and edge callbacks stay frozen. The commit message names this a known leak; the follow-up escape hatch is not at HEAD.
- **The pending guard does not apply with `disableVirtualization`.** That branch runs before the `pendingScrollUpdateCount` check in `_adjustCellsAroundViewport`.
- **Metrics are keyed by `keyExtractor` output.** Unstable or duplicate keys corrupt spacer sizes and estimates. Without a `key`/`id`/`keyExtractor`, index keys are used and a warning is logged.
- **Two different default thresholds:** windowing priority uses `on*ReachedThreshold ?? 2` viewports, but `_maybeCallOnEdgeReached` uses **2 px** when the prop is unset (`DEFAULT_THRESHOLD_PX`, TODO T121172172).
- **`_onScroll` with an `Animated` native-driver handler** must go through `Animated.createAnimatedComponent`; `_checkProps` has an invariant for this. Props are checked on every `render()`.
- **`CellRendererComponent` must forward `onLayout` and `onFocusCapture`.** Otherwise measurement and focus retention silently break.
- **`I18nManager.isRTL` is read on every `_orientation()` call** and is treated as constant. Commit 731ae45ae6e's message says `_orientation()` was cached, but the landed diff only routed hot paths through `_isHorizontalRTL()`/`horizontalOrDefault`. `_orientation()` still allocates a new object on each call.
- **The RTL flip in `_invalidateIfOrientationChanged`** clears `_cellMetrics` but keeps `_measuredCellsCount/Length`. The average then drifts instead of dividing by zero; the `count > 0` guard is the defense-in-depth that its comment describes.
- **`componentWillUnmount`** clears the batch timer, disposes the viewability helpers, and flushes the FillRateHelper.

## Design decisions
| Decision | Reason / evidence |
| --- | --- |
| Sparse `CellRenderMask` state instead of a contiguous `{first,last}` | Allows rendering discontiguous regions (focus retention for keyboard/a11y on desktop, sticky headers, initial region). It was shipped as `VirtualizedList_EXPERIMENTAL` and then made the default (479053cb3ce, 971599317b7). |
| Keep the initial `initialNumToRender` cells mounted | Gives instant scroll-to-top. Disabled when `initialScrollIndex > 0` (`VirtualizedListProps.js` docs). |
| High- vs low-priority updates; skip high priority until sizes are known | Without a size estimate, the list would render zero-size cells and starve layout (comment in `_scheduleCellsToRenderUpdate`). |
| Cap new cells per batch (`maxToRenderPerBatch`), keep rendered cells | Fills the visible content first and reduces churn (comments in `computeWindowedRenderLimits`). |
| Clamp the tail spacer to the highest measured cell | Prevents hyperscrolling into estimated space where content would jump (comment in `render()`). |
| Android inverted uses `scale: -1`, not `scaleY: -1` | `scaleY:-1` causes ANRs on API 33+. Native moves the scrollbar via `isInvertedVirtualizedList` (#38071, #38073 / 3dd816c6b7b). |
| Same-orientation nested lists render a `View` and share the parent's scroll | Only one scrollable. The child windows itself against translated parent metrics (`_convertParentScrollMetrics`). |
| Flags read via `react-native/react-private-interface` | The package is published separately, and the deep import `react-native/src/private/...` was not in `exports`, which made Metro warn (a506ed66cc5, #57940). |
| `getDerivedStateFromProps` shifts the window on prepend (MVCP) | Keeps the previously visible cells mounted so native can anchor them (69b22c97991, #35993). See [DESIGN.md](../../../packages/virtualized-lists/__docs__/DESIGN.md). |
| Defer focus-driven window updates (flagged) | Focus changes skipped the batch/timeout path that scroll uses (9253fc3b420, #52380). |

### Feature flags
Both flags are in the `jsOnly` section of `packages/react-native/scripts/featureflags/ReactNativeFeatureFlags.config.js`, with `defaultValue: false` and `ossReleaseStage: 'none'` (off in OSS). Both are imported as `import {ReactNativeFeatureFlags} from 'react-native/react-private-interface'`. Tests override them with `ReactNativeFeatureFlags.override({...})` (see `VirtualizeUtils-test.js`). Regenerate the accessors with `yarn featureflags`.

| Flag | Where | Effect |
| --- | --- | --- |
| `fixVirtualizeListCollapseWindowSize` | `computeWindowedRenderLimits` | When on, `first`/`last` count as "adding more" only relative to `prev.first`/`prev.last`, not when the window jumps past `prev`. This fixes the window collapsing to one cell on fast scroll (df7b6ae092d, #47965). |
| `deferFlatListFocusChangeRenderUpdate` | `_onCellFocusCapture` | When on, uses `_scheduleCellsToRenderUpdate` instead of the immediate `_updateCellsToRender`. |

## Types: hand-written vs generated
- `types_generated/**` is **generated and gitignored, do not edit** (`@generated SignedSource`). `yarn build-types` (`scripts/js-api/build-types`) walks the Flow import graph from `packages/react-native/index.js.flow` and writes `types_generated/` into every package it reaches. Only modules reachable through types appear there; for example, there is no `VirtualizedListCellRenderer.d.ts`. These files are what `exports["."].types` serves. Public API changes also update `packages/react-native/ReactNativeApi.d.ts`. Check with `yarn test-generated-typescript`.
- `index.d.ts` + `Lists/VirtualizedList.d.ts` are **hand-written legacy types**, served under the `react-native-legacy-deep-imports` export condition and consumed by `packages/react-native/types_DEPRECATED`. Keep them in sync by hand when the props change (for example 5306e9229c2 added `ListItemComponent`).
- `Lists/__tests__/__snapshots__/*.snap` are Jest snapshots. Regenerate them with `-u`; do not hand-edit.

## Where to change things
| To… | Start at |
| --- | --- |
| Change how far ahead or behind cells render, or the batch sizing | `computeWindowedRenderLimits` in `VirtualizeUtils.js` |
| Change when a window update fires, or its priority | `_scheduleCellsToRenderUpdate`, `_shouldRenderWithPriority` in `VirtualizedList.js` |
| Keep extra cells mounted (a new always-rendered region) | `_createRenderMask` or `_getNonViewportRenderRegions` |
| Fix spacer size or scroll-to-index estimates | `ListMetricsAggregator.getCellMetricsApprox`, spacer loop in `render()` |
| Fix RTL or orientation measurement | `ListMetricsAggregator.flowRelativeOffset`/`cartesianOffset`/`_invalidateIfOrientationChanged`, `_offsetFromScrollEvent` |
| Fix onEndReached/onStartReached | `_maybeCallOnEdgeReached` |
| Fix initialScrollIndex | constructor, `_maybeScrollToInitialScrollIndex`, `_initialRenderRegion` |
| Fix MVCP / prepend window shifting | `getDerivedStateFromProps`; native side per [DESIGN.md](../../../packages/virtualized-lists/__docs__/DESIGN.md) |
| Fix nested-list behavior | `measureLayoutRelativeToContainingList`, `_convertParentScrollMetrics`, `_findFirstChildWithMore`, `VirtualizedListContext.js` |
| Change cell wrapper, separators, or focus capture | `VirtualizedListCellRenderer.js` |
| Add or change a prop | `VirtualizedListProps.js` + hand-written `Lists/VirtualizedList.d.ts`, then `yarn build-types` |

## Tests
- `yarn test packages/virtualized-lists`: 9 suites. At 024b474ce92: 185 passed, 1 skipped (`calls onStartReached when near the start`), 69 snapshots, about 5 s.
- `Lists/__tests__/VirtualizedList-test.js` uses `react-test-renderer` and `jest.useFakeTimers()`. It drives private handlers directly: `simulateLayout`/`simulateCellLayout`/`simulateScroll` call `_onLayout`, `_onContentSizeChange`, `_onCellLayout`, and `_onScroll`. It steps batches with `performNextBatch` (one timer) or `performAllBatches`. To wait for the window, use `advanceUntilRenderAreaChanged`/`advanceUntilLastCellIndexRendered` rather than fixed timer counts, which were flaky under React 19 (9b966d1d8f7, c0bf1549c2b). Most tests assert the rendered regions in snapshots.
- Unit suites: `VirtualizeUtils-test.js` (window math, including the flag override), `CellRenderMask-test.js`, `ListMetricsAggregator-test.js`, `ChildListCollection-test.js`, `Utilities/__tests__/clamp-test.js`.
- Integration ([Fantom](../../../private/react-native-fantom/__docs__/README.md), `yarn fantom <path>`): `packages/react-native/Libraries/Lists/__tests__/{FlatList,SectionList}-itest.js`, `VirtualizedList-stickyHeaders-benchmark-itest.js`, `Libraries/Components/ScrollView/__tests__/ScrollView-maintainVisibleContentPosition-itest.js`.
