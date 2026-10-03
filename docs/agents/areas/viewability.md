<!-- index-codebase: generated from 024b474ce92 on 2026-10-03. Code wins when it disagrees with this doc. -->
# Viewability, edge callbacks and fill rate

**Purpose:** Turns a `VirtualizedList`'s scroll metrics and cell metrics into three kinds of user-facing signal: `onViewableItemsChanged` (through `ViewabilityHelper`), `onStartReached`/`onEndReached` (through `VirtualizedList._maybeCallOnEdgeReached`), and opt-in blankness telemetry (through `FillRateHelper`). It does not choose which cells render. That is the windowing engine (`computeWindowedRenderLimits`, `_adjustCellsAroundViewport`), covered in [virtualized-list.md](virtualized-list.md). This area only reads the window it produces (`state.cellsAroundViewport`).

**Paths:**
- `packages/virtualized-lists/Lists/ViewabilityHelper.js`, `FillRateHelper.js`
- `packages/virtualized-lists/Lists/VirtualizedList.js`: the viewability, edge and fill-rate members listed under "Key types and entry points"
- `packages/react-native/Libraries/Lists/FlatList.js` (`numColumns` adaptation), `packages/virtualized-lists/Lists/VirtualizedSectionList.js` (section index mapping)
- `packages/react-native/Libraries/Lists/ViewabilityHelper.js`, `FillRateHelper.js`: re-export shims of the package classes
- RNTester: `packages/rn-tester/js/examples/FlatList/FlatList-BaseOnViewableItemsChanged.js` and its `FlatList-onViewableItemsChanged*.js` variants, `FlatList-onEndReached.js`, `FlatList-onStartReached.js`, plus the matching `SectionList-*` examples
- Maestro: `packages/rn-tester/.maestro/flatlist-viewability.yml`, `sectionlist-viewability.yml`

**Depends on:** [virtualized-list.md](virtualized-list.md) (`_scrollMetrics`, `ListMetricsAggregator`, `state.cellsAroundViewport`, `pendingScrollUpdateCount`) · **Used by:** [flatlist-sectionlist.md](flatlist-sectionlist.md). Not used by [virtualview.md](virtualview.md) or [virtualcollection.md](virtualcollection.md): nothing outside `virtualized-lists` and the `Libraries/Lists` shims imports `ViewabilityHelper` or `FillRateHelper`.

## How it works

### Viewability: concepts
- **Tuple.** The `VirtualizedList` constructor builds `_viewabilityTuples` once. It makes one `{viewabilityHelper, onViewableItemsChanged}` per entry in `viewabilityConfigCallbackPairs`. If that prop is absent and `onViewableItemsChanged` is set, it makes a single tuple from `viewabilityConfig`. When both are passed, pairs win silently; only `FlatList._checkProps` throws on that combination.
- **Two exclusive modes.** `ViewabilityHelper.computeViewableItems` requires exactly one of `itemVisiblePercentThreshold` (visible pixels ÷ item length) and `viewAreaCoveragePercentThreshold` (visible pixels ÷ viewport length). Anything else fails an invariant. An item that is entirely inside the viewport is always viewable (`_isEntirelyVisible`). With no config at all, the default is `{viewAreaCoveragePercentThreshold: 0}`, which means any visible pixel counts.
- **Scan.** Only `renderRange` (the current `cellsAroundViewport`) is scanned. Cells without metrics are skipped. The scan stops at the first non-visible cell after a visible one. `top` and `bottom` are `Math.floor`ed.
- **Interaction gate.** With `waitForInteraction`, `onUpdate` returns early until `recordInteraction()` sets `_hasInteracted`. Only `VirtualizedList._onScrollBeginDrag` (a user drag) and the public `recordInteraction()` call it. Programmatic scrolls (`scrollToIndex`, `scrollToEnd`) do not count.
- **Dwell timer.** With `minimumViewTime`, each change in the visible-index set schedules a `setTimeout`. When it fires, it drops itself if `_viewableIndices` is no longer the same array instance. It then keeps only the indices that are still viewable.
- **Diff by key.** `_onUpdateSync` builds `ViewToken`s through `createViewToken`, which is `VirtualizedList._createViewToken`, and diffs them against `_viewableItems` by **key**. `changed` holds the newly viewable tokens (`isViewable: true`) plus the tokens that left (`isViewable: false`). The callback fires only if `changed` is non-empty. The info object also carries `viewabilityConfig`.

### One flow: scroll → `onViewableItemsChanged`
1. Native `onScroll` reaches `VirtualizedList._onScroll`. It first forwards the event to `_nestedChildLists` and to `props.onScroll`. Then it computes `visibleLength`, `offset` (from `_offsetFromScrollEvent`, which flips the offset for horizontal RTL), `dOffset`, `velocity` and `crossAxisLength`, and writes them to `this._scrollMetrics`.
2. If `state.pendingScrollUpdateCount > 0`, it calls `setState({pendingScrollUpdateCount: 0})`.
3. `_updateViewableItems(props, state.cellsAroundViewport)` bails while `pendingScrollUpdateCount > 0`. It passes `visibleLength` as 0 when `crossAxisLength` is 0, so a zero-sized list reports nothing. Then it calls `viewabilityHelper.onUpdate(...)` on each tuple.
4. `onUpdate` bails in four cases: `waitForInteraction && !_hasInteracted`, `itemCount === 0`, no metrics for item 0, or a visible-index set identical to the last one ("lots of scroll events where visibility doesn't change"). Otherwise it stores the new array and either calls `_onUpdateSync` directly or after `minimumViewTime`.
5. `_onUpdateSync` sends `{viewableItems, changed, viewabilityConfig}` to the tuple's callback. The callback is often a wrapper:
   - **FlatList** (`_createOnViewableItemsChanged`): when `numColumns > 1`, each row token carries an array `item`. `_pushMultiColumnViewable` expands it into one token per column, with `index = row * numColumns + col` and a per-item `key` from `keyExtractor`. This path drops `viewabilityConfig` from the info.
   - **VirtualizedSectionList** (`_onViewableItemsChanged` → `_convertViewable`): it maps the flat index through `_subExtractor`. The result has the index within the section, `section` set, and the key from `section.keyExtractor` or `props.keyExtractor`. Section header and footer slots appear as tokens with `index: null` and the **section object** as `item`. The `viewabilityConfig` field is dropped here too.
6. `_onScroll` then runs `_maybeCallOnEdgeReached()`, `_fillRateHelper.activate()` (when `velocity !== 0`), `_computeBlankness()` and `_scheduleCellsToRenderUpdate()`.

Other callers of `_updateViewableItems`:
- `_onCellLayout`, so layout changes alone can change viewability
- `recordInteraction()`
- `_updateCellsToRender`, called **before** its `setState`. Commit 62a0640e4a8 moved it there so user callbacks that call imperative list methods don't run inside a state update.

`componentDidUpdate` calls `resetViewableIndices()` when `data` or `extraData` changes. The next update then re-diffs. Because the diff is by key, a data change that keeps the keys produces no callback.

### Horizontal, inverted, nested
- **Horizontal:** `_selectLength` and `_selectOffset` pick the axis. The helper is axis-agnostic: it only sees `offset`, `length` and `viewportHeight` (really "viewport length").
- **Inverted:** the code never reads `inverted`. Viewability and edges are computed in content coordinates (index 0 at offset 0). For an inverted list, "start" is the visual bottom, so `onStartReached` and `onEndReached` refer to data order, not screen position.
- **Horizontal RTL:** `_offsetFromScrollEvent` measures from the right edge. Commit a2fb46ec0d1 fixed an invariant violation in viewability on RTL.
- **Nested, same orientation:** the child gets the parent's scroll events through `_nestedChildLists`. `_convertParentScrollMetrics` rebases the offset by `_offsetFromParentVirtualizedList`, and the visible length is the parent's. So child viewability and edge callbacks are relative to the **outer** viewport. The child ignores scroll events until it has a content length. `recordInteraction()` and `_onScrollBeginDrag` propagate to children. A child that registers after the parent saw a drag (`_hasInteracted`) gets `recordInteraction()` in `_registerAsNestedChild`.
- **Nested, different orientation** (for example a horizontal row inside a vertical list): this is a separate scroll root. Viewability only considers its own viewport, not whether the row is on screen. The `*-offScreen` RNTester examples cover the case where the whole list sits below a 1000px spacer. **Inferred:** a horizontal row scrolled off-screen vertically still reports its items as viewable.

### Edge callbacks (`_maybeCallOnEdgeReached`)
Called from `_onScroll`, `_onLayout` and `_onContentSizeChange`. `componentDidUpdate` also calls it, but only when `getItemLayout != null`: you can scroll past the render window without a new layout event (commits 62b7396bf45 and 3485e9ed871).

Steps:
1. Bail until `hasContentLength()` is true and `visibleLength > 0`, and while `state.pendingScrollUpdateCount > 0`.
2. Compute `distanceFromStart = offset` and `distanceFromEnd = contentLength - visibleLength - offset`. Values below `ON_EDGE_REACHED_EPSILON` (0.001) are floored to 0.
3. The threshold is `prop * visibleLength`, or **2px** when the prop is null (`DEFAULT_THRESHOLD_PX`).
4. Fire `onEndReached` when the last cell is in the window (`cellsAroundViewport.last === itemCount - 1`), the end is within the threshold, and `contentLength !== _sentEndForContentLength`. Firing records the content length. `onStartReached` mirrors this with `first === 0` and `_sentStartForContentLength`. Both can fire in one call (commit 4dcc1d3efbd).
5. **Re-arm:** leaving the threshold resets the matching `_sent*ForContentLength` to 0. A content-length change while still inside the threshold also allows another fire.

### FillRateHelper (blankness telemetry)
- **What it measures:** the pixels of viewport not covered by mounted cells, sampled per scroll event. `blankTop` is ignored when the first mounted cell is item 0, and `blankBottom` when the last is the final item, so the header and footer don't count as blank. The `Info` counters include `any_blank_count`/`_ms`, `mostly_blank_*` (blankness > 0.5), `pixels_blank`, `pixels_sampled`, `pixels_scrolled` and `sample_count`.
- **Sampling:** these are global module state.
  - `FillRateHelper.setSampleRate(r)` must be called **before** lists mount. Each instance decides `_enabled = r > Math.random()` once, in its constructor.
  - `addListener(cb)` warns if the sample rate was never set.
  - `setMinSampleCount` (default 10) discards under-sampled sessions.
- **Lifecycle:** `activate()` runs on a scroll with velocity. `computeBlankness` runs on scroll, cell layout, drag end and momentum end. It auto-flushes when the viewport is not blank and scrolling has nearly stopped. `deactivateAndFlush()` also runs on unmount and delivers `Info` plus `total_time_spent` to the listeners.
- **Status:** it is still wired up but inert by default (`_sampleRate` is `null` unless the internal `DEBUG` constant is true). One side effect remains when sampled: `_pushCells` sets `shouldListenForLayout` when `_fillRateHelper.enabled()`, so cells attach `onLayout` even with `getItemLayout`. It is public as `FillRateHelper` on the `@react-native/virtualized-lists` default export, and its type is exported as `FillRateInfo`. Nothing in the repo calls `setSampleRate`. **Inferred:** it is kept for app-level (Meta) perf logging.

## Key types and entry points
| Symbol | File | Role |
| --- | --- | --- |
| `ViewabilityConfig`, `ViewToken`, `ViewabilityConfigCallbackPair(s)` | `ViewabilityHelper.js` | Public types, re-exported as `ListViewToken` etc. from `virtualized-lists/index.js` |
| `ViewabilityHelper.onUpdate` / `computeViewableItems` / `_onUpdateSync` | `ViewabilityHelper.js` | Gating → index scan → dwell → key diff → callback |
| `recordInteraction`, `resetViewableIndices`, `dispose` | `ViewabilityHelper.js` | Interaction latch, cache clear on data change, timer cleanup on unmount |
| `_viewabilityTuples`, `_updateViewableItems`, `_createViewToken` | `VirtualizedList.js` | Wiring from list to helpers |
| `recordInteraction()` (public), `_onScrollBeginDrag` | `VirtualizedList.js` | The only sources of "interaction" |
| `_maybeCallOnEdgeReached`, `_sentStartForContentLength`, `_sentEndForContentLength`, `ON_EDGE_REACHED_EPSILON` | `VirtualizedList.js` | Edge callbacks and re-arm |
| `onStartReachedThresholdOrDefault` / `onEndReachedThresholdOrDefault` (default 2) | `VirtualizedListProps.js` | Used by the windowing engine's priority logic (`_shouldRenderWithPriority`, `_adjustCellsAroundViewport`), **not** by `_maybeCallOnEdgeReached` |
| `FillRateHelper` (`setSampleRate`, `addListener`, `computeBlankness`, `activate`, `deactivateAndFlush`) | `FillRateHelper.js` | Blankness telemetry |
| `_createOnViewableItemsChanged`, `_pushMultiColumnViewable` | `FlatList.js` | `numColumns` row→item token expansion |
| `_onViewableItemsChanged`, `_convertViewable`, `_subExtractor` | `VirtualizedSectionList.js` | Flat index → section-relative token |

## Invariants and gotchas
- **Config changes after mount are ignored or rejected.** Tuples are built only in the `VirtualizedList` constructor. On a raw `VirtualizedList`, later changes to `viewabilityConfig`, `onViewableItemsChanged` or the pairs are **silently ignored**, and the original callback keeps firing. `FlatList.componentDidUpdate` turns these into invariants:
  - "Changing viewabilityConfig on the fly is not supported" (checked with `deepDiffer`, so a new object with the same contents is fine)
  - "Changing viewabilityConfigCallbackPairs on the fly is not supported" (checked by **identity**, so pass a stable array)
  - "Changing onViewableItemsChanged nullability on the fly is not supported"

  Changing the *function* is allowed in FlatList: its wrapper reads `this.props.onViewableItemsChanged` at call time. Verified.
- **SectionList:** `_onViewableItemsChanged` reads `this.props` at call time, so the function can change. Going from null to a function after mount never creates a tuple, so callbacks silently never fire. `viewabilityConfigCallbackPairs` passes through to `VirtualizedList` **unconverted**. Its tokens carry flat indices that count header and footer slots, have no `section`, and use the SectionList-internal keys (for example `"0:header"`).
- **Threshold config must name a mode.** A `viewabilityConfig` with neither threshold, such as `{waitForInteraction: true}` alone, throws "Must set exactly one of itemVisiblePercentThreshold or viewAreaCoveragePercentThreshold" on the first update.
- **No metrics for item 0, no viewability.** `onUpdate` bails while item 0 has no metrics, and cells are keyed by key, not index. **Inferred:** with `initialScrollIndex > 0` and no `getItemLayout`, item 0 is not in the initial render mask (`_createRenderMask` only keeps the initial region when there is no `initialScrollIndex`). Viewability then stays silent until item 0 is measured.
- **Suppressed while scroll metrics are stale:** `_updateViewableItems` and `_maybeCallOnEdgeReached` both bail while `pendingScrollUpdateCount > 0`. This happens after an MVCP prepend, or before the initial scroll to `initialScrollIndex`. **Inferred:** `_onScroll` resets the count with a batched `setState` but then reads `this.state` synchronously, so the event that clears it is itself still suppressed. Viewability catches up via `componentDidUpdate` → `_scheduleCellsToRenderUpdate`. Edge callbacks wait for the next scroll, layout or content-size event, unless `getItemLayout` is set.
- **Default edge threshold is 2px.** It is not 2 viewports, despite `onEndReachedThresholdOrDefault` returning 2 (`TODO: T121172172`). Edges also require the first or last *cell* to be inside `cellsAroundViewport`.
- **`changed` is key-based.** Index shifts with stable keys don't fire. `index` in a token reflects the latest computation. With `minimumViewTime`, the timer uses the `props` captured when it was scheduled.
- **Programmatic scroll is not an interaction.** `VirtualizedList.recordInteraction()` does not set the list's own `_hasInteracted`; only a drag does. **Inferred:** a nested child that mounts after a programmatic `recordInteraction()` on the parent stays gated.
- `ViewabilityHelper` holds timers. `VirtualizedList.componentWillUnmount` calls `dispose()`, so anyone using the helper standalone must call it too.
- `_pendingViewabilityUpdate` is declared on `VirtualizedList` but never read or written. It is dead.

## Design decisions
| Decision | Reason / evidence |
| --- | --- |
| Discard a dwell timer if `_viewableIndices` changed identity | Intermediate snapshots during initial layout fired late and overwrote the correct set (1c4a46f4e31, test "minimumViewTime ignores stale viewability updates") |
| `visibleLength = 0` when `crossAxisLength` is 0 | A zero-height horizontal list reported viewable items (c057b1fa016; an earlier attempt, 57c8285d40a, was reverted) |
| `Math.floor` on `top`/`bottom` | Sub-pixel measurement imprecision made fully visible items look partial (824c1c6d073, test "should account for imprecision…") |
| `_updateViewableItems` before `setState` in `_updateCellsToRender` | User callbacks calling imperative methods mid-update read inconsistent state (62a0640e4a8, issue #36329) |
| Fire edge callbacks once per content length, re-arm on leaving the threshold | Avoids repeated `onEndReached` per scroll event while still firing after new data grows the content (comments in `_maybeCallOnEdgeReached`) |
| Fire start and end independently | Small lists fired only `onEndReached` on mount (4dcc1d3efbd) |
| Edge check in `componentDidUpdate` only with `getItemLayout` | You can scroll past rendered cells without a layout event (3485e9ed871) |
| `pendingScrollUpdateCount` clamped to 1 and drained to 0 by any scroll | Native coalesces unique scroll events, so N prepends produced only one event, and viewability and `onEndReached` stayed blocked forever (5cb65244dc5) |
| FlatList wraps `onViewableItemsChanged` in a stable closure | "allow the actual callback to change while still keeping the function provided to native to be stable" (comment in the `FlatList` constructor) |
| FillRate is global and sampled | "Listeners and sample rate are global… typical usage will combine with the active scene/route" (class doc) |

## Where to change things
| To… | Start at |
| --- | --- |
| Change how "viewable" is computed | `ViewabilityHelper.computeViewableItems`, `_isViewable` |
| Change dwell or interaction semantics | `ViewabilityHelper.onUpdate`, `recordInteraction`; interaction sources in `VirtualizedList._onScrollBeginDrag` |
| Change what triggers a viewability recompute | `VirtualizedList._updateViewableItems` and its callers (`_onScroll`, `_onCellLayout`, `_updateCellsToRender`, `recordInteraction`) |
| Change token shape for grids or sections | `FlatList._pushMultiColumnViewable`, `VirtualizedSectionList._convertViewable` |
| Change edge callback timing or thresholds | `VirtualizedList._maybeCallOnEdgeReached` |
| Add a blankness metric | `FillRateHelper.computeBlankness`, `Info` |
| Accept config changes after mount | `VirtualizedList` constructor (tuple creation), `FlatList.componentDidUpdate` invariants |

## Tests
- `yarn test packages/virtualized-lists/Lists/__tests__/ViewabilityHelper-test.js packages/virtualized-lists/Lists/__tests__/FillRateHelper-test.js`: 2 suites, 18 tests, passing at 024b474ce92. They cover the threshold modes, dwell (including stale timers), `waitForInteraction`, data changes and measurement imprecision; for fill rate, blankness, sampling and listeners.
- `packages/virtualized-lists/Lists/__tests__/VirtualizedList-test.js`: viewable items after a data change, an empty cross-axis viewport, `onStartReached`/`onEndReached` near the edges and initially, and `onEndReached` after `onContentSizeChange`.
- Fantom (`yarn fantom`): `packages/react-native/Libraries/Lists/__tests__/FlatList-itest.js` covers `onEndReached`, `onViewableItemsChanged`, pairs, `waitForInteraction`, `recordInteraction` and `numColumns`.
- Maestro (on device, through RNTester deep links):
  - `flatlist-viewability.yml`: horizontal no-wait, horizontal wait and vertical wait
  - `sectionlist-viewability.yml`: `onEndReached`, horizontal variants, and vertical wait **on Android only** (no reason recorded in eba89ccc99e)

  The examples use `minimumViewTime: 1000` and `viewAreaCoveragePercentThreshold: 100` (`FlatList-BaseOnViewableItemsChanged.js`).
