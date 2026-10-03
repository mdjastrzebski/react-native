<!-- index-codebase: generated from 024b474ce92 on 2026-10-03. Code wins when it disagrees with this doc. -->

# React Native Lists and Virtualization: Agent Index

> **Scope:** this index covers only the list and virtualization stack: `FlatList`, `SectionList`, `VirtualizedList`, viewability, `maintainVisibleContentPosition`, `VirtualView`, and `VirtualColumn`/`VirtualRow`. The rest of React Native is not indexed here. For repo-wide setup, commands and rules, start at [`AGENTS.md`](../../AGENTS.md).

## What this is

React Native ships two separate approaches to virtualization:

1. **JS windowing (stable, public):** `FlatList` and `SectionList` sit on top of `VirtualizedList`. JS measures cells through `onLayout`, follows the scroll offset through `onScroll`, and decides which cells to mount. Unmounted ranges are replaced by sized spacer `View`s. The engine lives in its own npm package, `@react-native/virtualized-lists` (`packages/virtualized-lists`), which `react-native` re-exports.
2. **Native visibility (experimental, `unstable_*`):** `VirtualView` is a host component. A per-ScrollView native "container state" classifies each `VirtualView` as Visible, Prerender or Hidden, and JS mounts or unmounts the children when it receives an `onModeChange` event. `VirtualColumn` and `VirtualRow`, built with `createVirtualCollectionView`, are list components layered on top. Internally this is code-named "Fling". It has no RNTester example and no consumers in this repo.

Everything in both approaches is Flow JavaScript, except the VirtualView native side (Obj-C++, Kotlin, a header-only C++ ShadowNode) and the native half of `maintainVisibleContentPosition` (MVCP).

## Architecture in brief

**JS windowing, per scroll event:** native `onScroll` → `VirtualizedList._onScroll` updates `_scrollMetrics` → `_updateViewableItems` and `_maybeCallOnEdgeReached` run → `_scheduleCellsToRenderUpdate` picks either a high-priority (immediate) or batched (`updateCellsBatchingPeriod`, 50 ms) update → `_adjustCellsAroundViewport` calls `computeWindowedRenderLimits` (`VirtualizeUtils.js`) → a new `cellsAroundViewport` and a sparse `CellRenderMask` → `render()` emits cells and spacers → `componentDidUpdate` schedules the next batch until the window is stable. Cell sizes come from `ListMetricsAggregator`, or from `getItemLayout` when it is provided.

**Shaping layers:** `FlatList` turns `numColumns` into rows (`_getItem` returns an array per row) and remaps viewability tokens back to individual items. `SectionList` → `VirtualizedSectionList` flattens sections into one stream of `[header, items…, footer]` per section and maps indices back with `_subExtractor`.

**MVCP (prepend anchoring):** JS detects a prepend in `getDerivedStateFromProps` (the `firstVisibleItemKey` moved), shifts the window, and freezes windowing and callbacks (`pendingScrollUpdateCount = 1`) until the next scroll event. Native code captures an anchor view before each mount, then shifts `contentOffset` by how far that anchor moved. Full design: [`packages/virtualized-lists/__docs__/DESIGN.md`](../../packages/virtualized-lists/__docs__/DESIGN.md).

**VirtualView, per scroll event:** native ScrollView scroll → `VirtualViewContainerState` (iOS `RCTVirtualViewContainerState`; Android `VirtualViewContainerStateClassic`, or `…Experimental` behind a flag) intersects every registered VirtualView with the viewport and with the prerender rect (`virtualViewPrerenderRatio`, default 5 viewports per side) → it emits `onModeChange`, synchronously for Visible and asynchronously otherwise → `VirtualView.js` renders the children (Prerender via `startTransition`) or replaces them with a `minHeight`/`minWidth` placeholder → the `renderState` prop feeds back so native can drop redundant events. `VirtualColumn` mounts `INITIAL_NUM_TO_RENDER` (7) items plus one hidden spacer VirtualView. Each time the spacer turns Prerender or Visible, the generator estimates how many more items fit and mounts them.

## Map

| Area | Responsibility | Paths | Doc |
| --- | --- | --- | --- |
| FlatList and SectionList | Public list components, `numColumns`, section flattening, re-export shims, public types | `packages/react-native/Libraries/Lists/`, `packages/virtualized-lists/Lists/VirtualizedSectionList.js` | [areas/flatlist-sectionlist.md](areas/flatlist-sectionlist.md) |
| VirtualizedList core | Render window, batching, cell metrics, spacers, nested lists, `initialScrollIndex`, edge callbacks, JS side of MVCP | `packages/virtualized-lists/` | [areas/virtualized-list.md](areas/virtualized-list.md) |
| Viewability and fill rate | `onViewableItemsChanged`, `onStartReached`/`onEndReached`, `FillRateHelper` blankness telemetry | `packages/virtualized-lists/Lists/{ViewabilityHelper,FillRateHelper}.js` + the parts of `VirtualizedList.js` that drive them | [areas/viewability.md](areas/viewability.md) |
| maintainVisibleContentPosition | Keeping the visible content anchored on prepends (JS, iOS Fabric, Android) | `VirtualizedList.js`, `RCTScrollViewComponentView.mm`, `MaintainVisibleScrollPositionHelper.kt` | [DESIGN.md](../../packages/virtualized-lists/__docs__/DESIGN.md) (hand-written) |
| VirtualView | Native-decided per-view visibility: JS component, codegen specs, ShadowNode, iOS and Android container states | `packages/react-native/src/private/components/virtualview/`, `ReactCommon/react/renderer/components/virtualview/`, `React/Fabric/Mounting/ComponentViews/{VirtualView,ScrollView}/`, `ReactAndroid/.../views/{virtual,scroll}/` | [areas/virtualview.md](areas/virtualview.md) |
| VirtualCollection | `VirtualColumn`, `VirtualRow`, `createVirtualCollectionView`, `VirtualArray`, pagination generators | `packages/react-native/src/private/components/virtualcollection/` | [areas/virtualcollection.md](areas/virtualcollection.md) |

## Where to look

| Task or question | Start at |
| --- | --- |
| Cells blank while scrolling fast; tune how far ahead cells render | `computeWindowedRenderLimits` (`VirtualizeUtils.js`), `_scheduleCellsToRenderUpdate`. See [virtualized-list.md](areas/virtualized-list.md) |
| Wrong spacer size, scroll jumps, `scrollToIndex` lands in the wrong place | `ListMetricsAggregator.getCellMetricsApprox`, the spacer loop in `VirtualizedList.render()` |
| Horizontal RTL offsets are wrong | `ListMetricsAggregator.flowRelativeOffset` / `cartesianOffset`, `_offsetFromScrollEvent` |
| `onEndReached` fires too often or never | `_maybeCallOnEdgeReached`, `_sentEndForContentLength`. See [viewability.md](areas/viewability.md) |
| `onViewableItemsChanged` is wrong or silent | `ViewabilityHelper.onUpdate`, `VirtualizedList._updateViewableItems`; multi-column remapping in `FlatList._createOnViewableItemsChanged` |
| Grid / `numColumns` behavior | `FlatList._getItem`, `_keyExtractor`, `_renderer` |
| Section headers, separators, `scrollToLocation` | `VirtualizedSectionList` (`_subExtractor`, `_getSeparatorComponent`, `scrollToLocation`) |
| Prepending makes the list jump (chat UIs) | [DESIGN.md](../../packages/virtualized-lists/__docs__/DESIGN.md); JS: `VirtualizedList.getDerivedStateFromProps`; native: `RCTScrollViewComponentView._adjustForMaintainVisibleContentPosition`, `MaintainVisibleScrollPositionHelper.updateScrollPositionInternal` |
| Nested lists in the same orientation | `VirtualizedListContext.js`, `_convertParentScrollMetrics`, `_findFirstChildWithMore` |
| VirtualView mode is not updating, or the prerender distance is wrong | iOS `RCTVirtualViewContainerState`; Android `VirtualViewContainerStateClassic.updateModes` / `…Experimental`; flag `virtualViewPrerenderRatio` |
| How VirtualView JS reacts to a mode change | `handleModeChange` in `src/private/components/virtualview/VirtualView.js` |
| VirtualColumn mounts too few or too many items | `column/VirtualColumnGenerator.js` `next`, `FlingConstants.js`, `VirtualCollectionSpacer` in `VirtualCollectionView.js` |
| Add a prop to a list | Flow source (`VirtualizedListProps.js` / `FlatList.js`) → hand-written `packages/virtualized-lists/Lists/VirtualizedList.d.ts` → `yarn build-types`. See [contributing.md](contributing.md) |

## Dev loop

| To check… | Run |
| --- | --- |
| VirtualizedList, VirtualizedSectionList, viewability, metrics unit tests | `yarn test packages/virtualized-lists` (9 suites, about 5 s; passed at 024b474ce92) |
| FlatList, SectionList, VirtualView, MVCP integration | `yarn fantom <path>` on `Libraries/Lists/__tests__/*-itest.js`, `src/private/components/virtualview/__tests__/VirtualView-itest.js`, `Libraries/Components/ScrollView/__tests__/ScrollView-maintainVisibleContentPosition-itest.js` (builds a native tester on first run; not run for this index) |
| Types | `yarn flow-check`, `yarn build-types`, `yarn test-generated-typescript` |
| End-to-end behavior | RNTester → FlatList / SectionList examples; Maestro flows in `packages/rn-tester/.maestro/` |

Full details are in [contributing.md](contributing.md).

## Conventions and traps

- **Two packages.** The engine is in `packages/virtualized-lists`, while `FlatList`/`SectionList` are in `packages/react-native/Libraries/Lists`. Files such as `Libraries/Lists/VirtualizedList.js` are re-export shims; edit the package instead.
- **Feature flags** used by the virtualized-lists package come from `react-native/react-private-interface`, not from deep imports. Declare flags in `packages/react-native/scripts/featureflags/ReactNativeFeatureFlags.config.js`, then run `yarn featureflags`, and never edit the generated accessors. Relevant flags: `fixVirtualizeListCollapseWindowSize`, `deferFlatListFocusChangeRenderUpdate`, `enableVirtualViewContainerStateExperimental`, `virtualViewPrerenderRatio`. All of them are `ossReleaseStage: 'none'`.
- **Generated outputs:** `packages/virtualized-lists/types_generated/` (gitignored) and the committed `ReactNativeApi.d.ts` (both from `yarn build-types`), `ReactAndroid.api` and `scripts/cxx-api/api-snapshots/*.api` (which include VirtualView symbols), Jest `__snapshots__` (`-u`). The VirtualView native props and event emitter are generated by codegen from `VirtualViewNativeComponent.js`.
- **Hand-written legacy types** need manual sync: `packages/virtualized-lists/index.d.ts`, `Lists/VirtualizedList.d.ts`, and `packages/react-native/types_DEPRECATED/Libraries/Lists/*.d.ts`.
- **`StateSafePureComponent`** throws if a `setState` updater reads `this.props` or `this.state`, so use the updater's arguments.
- **`pendingScrollUpdateCount` is effectively a boolean**, and while it is set, windowing, viewability and edge callbacks are frozen until a native scroll event arrives.
- **`unstable_*` exports** (VirtualView, VirtualColumn, …) are stripped from `ReactNativeApi.d.ts` by `stripUnstableApis.js`; their types are exposed only through `index.js.flow`.
- **VirtualView needs a native ScrollView ancestor.** Without one, no mode events ever fire.
- **Two VirtualView specs:** keep `VirtualViewNativeComponent.js` and `VirtualViewExperimentalNativeComponent.js` identical. Native code registers only `VirtualView`; the other spec is a fallback for older native builds.

## Glossary

| Term | Meaning | Defined in |
| --- | --- | --- |
| Render window / `cellsAroundViewport` | Contiguous range of item indices mounted around the viewport, including overscan | `VirtualizedList.js` |
| `CellRenderMask` | Sparse set of rendered and spacer regions covering all items (window + initial region + sticky header + focused cell) | `CellRenderMask.js` |
| `windowSize` | Viewport-lengths of content kept mounted (default 21, so 10 above and 10 below) | `VirtualizedListProps.js` |
| Cell | One VirtualizedList item slot. A FlatList row in multi-column mode; in SectionList, also headers and footers | `VirtualizedListCellRenderer.js` |
| Spacer | Sized blank `View` that stands in for unmounted cells. In VirtualCollection, a hidden VirtualView that triggers pagination | `VirtualizedList.render`, `VirtualCollectionView.js` |
| MVCP | `maintainVisibleContentPosition`, anchoring visible content when items are prepended | [DESIGN.md](../../packages/virtualized-lists/__docs__/DESIGN.md) |
| Viewability | Whether an item counts as "viewable" under `viewabilityConfig` thresholds and `minimumViewTime`. Not the same as rendered | `ViewabilityHelper.js` |
| Fill rate / blankness | Telemetry for how much of the viewport was blank (unrendered) | `FillRateHelper.js` |
| VirtualView mode | `Visible` / `Prerender` / `Hidden`, decided by native code | `VirtualView.js`, `RCTVirtualViewMode.h`, `VirtualViewMode.kt` |
| Render state | Prop that JS sends back to native: whether the children are committed (`Rendered`/`None`/`Unknown`) | `VirtualView.js` |
| Container state | Per-ScrollView native object that tracks VirtualViews and computes their modes | `RCTVirtualViewContainerState.mm`, `VirtualViewContainer.kt` |
| Generator (VirtualCollection) | Pagination strategy object `{initial, next(event)}`, **not** a JS `function*` | `VirtualCollectionView.js` |
| Fling | Internal codename for VirtualView/VirtualCollection, unrelated to fling physics | `FlingConstants.js` |
| "Virtual view" in a11y code | Android `ExploreByTouchHelper` virtual nodes, unrelated to `VirtualView` | `ReactAccessibilityDelegate.kt` |

## Further reading

- [using.md](using.md): configuration surfaces, extension points, limitations.
- [contributing.md](contributing.md): tests, CI jobs, type generation, release.
- [`packages/virtualized-lists/__docs__/DESIGN.md`](../../packages/virtualized-lists/__docs__/DESIGN.md): MVCP design (hand-written).
- [`private/react-native-fantom/__docs__/README.md`](../../private/react-native-fantom/__docs__/README.md): Fantom integration tests.
- [`__docs__/README.md`](../../__docs__/README.md): React Native technical docs index (does not yet link the lists docs).
- Public docs: [FlatList](https://reactnative.dev/docs/flatlist), [VirtualizedList](https://reactnative.dev/docs/virtualizedlist), [SectionList](https://reactnative.dev/docs/sectionlist), [optimizing FlatList](https://reactnative.dev/docs/optimizing-flatlist-configuration).
