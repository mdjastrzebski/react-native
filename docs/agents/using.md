<!-- index-codebase: generated from 024b474ce92 on 2026-10-03. Code wins when it disagrees with this doc. -->

# Lists and Virtualization: Using and Extending

Scope: the list stack indexed in [index.md](index.md). For the documented props, see the public docs ([FlatList](https://reactnative.dev/docs/flatlist), [VirtualizedList](https://reactnative.dev/docs/virtualizedlist), [SectionList](https://reactnative.dev/docs/sectionlist)). This page records what those docs leave out.

## Platforms and integrations

| Piece | Platforms | Where it lives |
| --- | --- | --- |
| FlatList / SectionList / VirtualizedList | Any platform with `ScrollView` and `View` (pure JS) | `packages/react-native/Libraries/Lists/`, `packages/virtualized-lists/` |
| `@react-native/virtualized-lists` package | Published separately, at the same version as `react-native`, so that React Native for Web can share it (PR #35406). Its README says "don't depend on it directly". | `packages/virtualized-lists/package.json` |
| MVCP native half | iOS Fabric and Android only. The iOS Paper implementation was removed in 86350ab9884. | `RCTScrollViewComponentView.mm`, `MaintainVisibleScrollPositionHelper.kt` |
| VirtualView / VirtualCollection | New Architecture (Fabric) only, iOS and Android. They need DOM APIs (`ReactNativeElement`) and a native ScrollView ancestor (`ReactScrollView`, `ReactHorizontalScrollView`, `ReactNestedScrollView`, `RCTScrollViewComponentView`). | [areas/virtualview.md](areas/virtualview.md) |

## Configuration

Most configuration is done through props. These are the defaults that are easy to get wrong:

| Setting | Where it's read | Default | Notes |
| --- | --- | --- | --- |
| `windowSize`, `initialNumToRender`, `maxToRenderPerBatch` | `*OrDefault` helpers in `VirtualizedListProps.js` | 21, 10, 10 | `windowSize` is in viewport lengths |
| `updateCellsBatchingPeriod` | `VirtualizedList._scheduleCellsToRenderUpdate` | 50 ms | Applies to the low-priority path only |
| `onEndReachedThreshold` / `onStartReachedThreshold` | `VirtualizedListProps.js`; `_maybeCallOnEdgeReached` | 2 viewports for windowing, **2 px** for the callback when unset | TODO T121172172 |
| `removeClippedSubviews` (FlatList) | `removeClippedSubviewsOrDefault` in `FlatList.js` | `true` on Android, `false` on iOS (`true` everywhere when the `shouldUseRemoveClippedSubviewsAsDefaultOnIOS` flag is on) | |
| `stickySectionHeadersEnabled` (SectionList) | `SectionList.render` | `true` on iOS only | |
| `strictMode` (FlatList) | `FlatList.render` | `false` | When true, the renderer is memoized |
| `numColumns`, `viewabilityConfig`, `onViewableItemsChanged`, `viewabilityConfigCallbackPairs` | FlatList / VirtualizedList constructors | | **Cannot change after mount.** FlatList throws when they change, while a raw VirtualizedList silently ignores the change. See [viewability.md](areas/viewability.md). |
| `maintainVisibleContentPosition` | `VirtualizedList.render` → `ScrollView` | off | `minIndexForVisible` is shifted by +1 when there is a `ListHeaderComponent` |
| `virtualViewPrerenderRatio` (feature flag) | iOS and Android container state, read once at creation | 5 | Prerender rect = viewport inflated by ratio × viewport size on each side |
| `enableVirtualViewContainerStateExperimental` (flag) | Android `VirtualViewContainerState.create` | false | Interval-tree container |
| `fixVirtualizeListCollapseWindowSize`, `deferFlatListFocusChangeRenderUpdate` (flags) | `VirtualizeUtils.js`, `VirtualizedList._onCellFocusCapture` | false | JS-only |

All the flags above have `ossReleaseStage: 'none'`. Apps override them through the feature-flags API (`packages/react-native/src/private/featureflags/`).

## Extension points

- **`renderScrollComponent`** replaces the `ScrollView`, for example with `Animated` or a gesture-handler ScrollView. It must forward `onScroll`, `onLayout` and `onContentSizeChange`, plus `maintainVisibleContentPosition` if MVCP is used.
- **`CellRendererComponent`** replaces the cell wrapper. It must forward `onLayout` and `onFocusCapture`, or measurement and focus retention break silently.
- **`getItemLayout`** skips measurement entirely. It enables accurate `scrollToIndex` and edge checks after updates.
- **`getItem` / `getItemCount`** on `VirtualizedList` accept any data source, such as immutable lists or lazily computed data.
- **`VirtualizedListContextResetter`** stops a nested list from attaching to an outer list (for example, `Modal` uses it).
- **`createVirtualCollectionView(Layout, generator)`** builds custom VirtualView-based layouts, such as grids or wrapped rows. `VirtualColumnGenerator` is exported for reuse; `VirtualRowGenerator` is not. A custom `VirtualCollection` (`size` + `at`) avoids materialising an array.
- **`hiddenStyle` on `VirtualView`** controls the placeholder size while hidden. The default is `minHeight`/`minWidth` of the last rect.
- **Needs a fork or patch:** the windowing math (`computeWindowedRenderLimits`), the viewability rules, and the native visibility computation have no plugin hooks.

## Limitations and gotchas

- With `numColumns > 1`, `scrollToItem` silently does nothing. `FlatList._getItem` builds a new row array each time, and `VirtualizedList.scrollToItem` compares with `===`.
- In a multi-column list, row keys join all item keys, so inserting one item re-keys every later row.
- `viewabilityConfigCallbackPairs` on `SectionList` receive raw flat indices, not section/item indices.
- Edge callbacks and viewability stay frozen while `pendingScrollUpdateCount > 0`. That happens after `initialScrollIndex > 0`, or after an MVCP prepend, until a native scroll event arrives. If native never scrolls, they stay frozen (a known leak per 5cb65244dc5).
- Without `getItemLayout`, the tail spacer is clamped to the highest measured cell. The scrollbar size is therefore approximate, and `scrollToIndex` past the measured range requires `onScrollToIndexFailed`.
- VirtualCollection has no `scrollToIndex`, viewability, `onEndReached`, headers or footers, sticky headers, `inverted`, or `getItemLayout`. Items are never unmounted, only their VirtualView children. The default `itemToKey` reads `item.key`, although its error message says `id`.
- VirtualView on iOS does not recompute modes when the ScrollView is resized (`TODO(T202601695)`).

## Security and trust

This stack has no network access, telemetry endpoints, or file I/O. `FillRateHelper` and the VirtualView logger (`useVirtualViewLogging`) are app-level hooks that are disabled or stubbed in OSS. There is no `SECURITY.md` in this repository.
