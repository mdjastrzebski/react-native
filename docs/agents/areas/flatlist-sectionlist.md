<!-- index-codebase: generated from 024b474ce92 on 2026-10-03. Code wins when it disagrees with this doc. -->

# FlatList and SectionList

**Purpose:** The public list components. `FlatList` adapts an array, optionally laid out in a grid, into `VirtualizedList`. `SectionList` adapts `sections` into `VirtualizedSectionList`, which flattens them into one stream that `VirtualizedList` renders. The area also covers the `react-native` re-export shims for the `@react-native/virtualized-lists` package, the Animated wrappers, and how the public types are produced. It does **not** cover windowing, cell rendering, batching, or scroll metrics (see [virtualized-list.md](virtualized-list.md)), viewability computation (see [viewability.md](viewability.md)), or MVCP ([DESIGN.md](../../../packages/virtualized-lists/__docs__/DESIGN.md)). See also [virtualview.md](virtualview.md) and [virtualcollection.md](virtualcollection.md).

**Paths:**
- `packages/react-native/Libraries/Lists/FlatList.js`, `SectionList.js`: the public components.
- `packages/react-native/Libraries/Lists/{VirtualizedList,VirtualizedSectionList,ViewabilityHelper,FillRateHelper,VirtualizedListContext,VirtualizeUtils}.js`: thin re-export shims.
- `packages/virtualized-lists/Lists/VirtualizedSectionList.js`: the engine behind SectionList. It lives in the other package.
- `packages/virtualized-lists/index.js`: the package's default-export object, made of lazy getters.
- `packages/react-native/Libraries/Animated/components/Animated{Flat,Section}List.js`
- `packages/react-native/index.js` (runtime getters), `index.js.flow` (type and value exports), `ReactNativeApi.d.ts` (API snapshot), `types_DEPRECATED/Libraries/Lists/*.d.ts` (legacy hand-written types).
- Tests: `Libraries/Lists/__tests__/*-itest.js` (Fantom), `Libraries/Lists/__flowtests__/`, `packages/virtualized-lists/Lists/__tests__/VirtualizedSectionList-test.js`.
- Manual and e2e: `packages/rn-tester/js/examples/{FlatList,SectionList}/`, `packages/rn-tester/.maestro/{flatlist,sectionlist}*.yml`.

**Depends on:** [virtualized-list.md](virtualized-list.md) (`VirtualizedList`, its `getItem`/`getItemCount`/`keyExtractor`/`renderItem` contract, `__getListMetrics`), [viewability.md](viewability.md) (`ViewToken`, `ViewabilityConfigCallbackPair`), ScrollView, Animated (`createAnimatedComponent`), feature flags. · **Used by:** apps through `react-native` (`FlatList`, `SectionList`, `Animated.FlatList`, `Animated.SectionList`), RNTester. Nothing else inside `Libraries/` or `src/` imports FlatList or SectionList.

## How it works

### The two-package split
- `VirtualizedList`, `VirtualizedSectionList`, `ViewabilityHelper`, `FillRateHelper`, `VirtualizedListContext`, and `VirtualizeUtils` live in `@react-native/virtualized-lists` (`packages/virtualized-lists`), which `react-native` depends on at the same version.
- `FlatList` and `SectionList` stayed in `react-native/Libraries/Lists`.
- Each `Libraries/Lists/Virtualized*.js` (and related) file is a typed re-export of a lazy getter on the package's default export, for example `VirtualizedLists.VirtualizedList`. The shims exist so `react-native` keeps the old import paths and `index.js` getters working.
- Why the package exists: PR #35406 (commit `2e3dbe9c2fb`) implemented issue #35263, "npm packages for react-native-web". The goal is to let React Native and React Native for Web depend on the same list implementation instead of RNWeb copy-pasting it.
  - The issue also proposed a separate `@react-native/flat-lists` package for FlatList and SectionList (PR #35423). That move never landed, which is why the split falls where it does.
  - The move was reverted and re-applied as file moves (`ebaa00e3274`, `0daf83ac51e`, "Reconnect VirtualizedList Source History") so that `git log --follow` works across it.
- `packages/virtualized-lists/README.md`: "internal dependency of React Native. Please don't depend on it directly."
  - Its `exports` map still allows `./*` deep imports. A subpath restriction was reverted in `dc494bb3416` (#50846) "after … consultation with framework authors".

### FlatList → VirtualizedList (`FlatList.render`)
FlatList is a `PureComponent`. It never passes `numColumns` or `columnWrapperStyle` through, and it supplies its own:

| VirtualizedList prop | FlatList supplies | Notes |
| --- | --- | --- |
| `getItem` | `_getItem` | With `numColumns > 1`, returns a **fresh array** with the row's 1..N items. Otherwise returns `data[index]`. |
| `getItemCount` | `_getItemCount` | `ceil(length / numColumns)`. Returns 0 for non-array-like `data`, keeping a legacy behaviour (see the comment in `_getItemCount`). |
| `keyExtractor` | `_keyExtractor` | Multi-column: the row key is the per-item keys (from the user's `keyExtractor` or the default `item.key` → `item.id` → index) joined with `':'`. |
| `renderItem` / `ListItemComponent` | `_renderer(...)` | Wraps the user renderer. Multi-column: renders a `View` with `styles.row` (`flexDirection: 'row'`) composed with `columnWrapperStyle`, one child per item with `index = row * cols + kk`, and the row's `separators` object shared across its items. |
| `viewabilityConfigCallbackPairs` | `_virtualizedListPairs` | Always passed, even as `[]`. VirtualizedList prefers pairs whenever the prop is truthy, so the raw `onViewableItemsChanged` that leaks through `restProps` is ignored. |
| `removeClippedSubviews` | `removeClippedSubviewsOrDefault` | Default is `true` on Android and `false` elsewhere. When the `shouldUseRemoveClippedSubviewsAsDefaultOnIOS` flag is on, the default is `true` everywhere (the flag is off by default, `ossReleaseStage: 'none'`). |
| `ref` | `_captureRef` | Sets `_listRef`. |

All other props, including `getItemLayout`, `initialScrollIndex`, and `ItemSeparatorComponent`, pass through **unchanged**, so they operate on VirtualizedList's index space (rows when `numColumns > 1`).

**Viewability wrapping.** The constructor builds `_virtualizedListPairs` once, from either `viewabilityConfigCallbackPairs` or `onViewableItemsChanged`. Each callback is wrapped by `_createOnViewableItemsChanged`. For `numColumns > 1`, `_pushMultiColumnViewable` expands each row token into one token per item with a recomputed `index` and `key`. Every column copies the row's `isViewable`. The single-callback path also wraps the callback in a closure that reads `this.props.onViewableItemsChanged` on each call. The code comment says this keeps "the function provided to native … stable" while the user callback changes.

**Renderer memoization (`strictMode`).** `_renderer` takes `(ListItemComponent, renderItem, columnWrapperStyle, numColumns, extraData)`.
- With `strictMode`, FlatList uses `_memoizedRenderer`, which is `memoizeOne(_renderer)`, so the wrapper keeps its identity until one of those arguments changes.
- Without `strictMode` (the default), every FlatList render creates a new `renderItem`. Commit `c231d5e371c` lists this as one cause of every `CellRenderer` re-rendering whenever the list does.

**Imperative API.** `scrollToEnd`, `scrollToIndex`, `scrollToItem`, `scrollToOffset`, `recordInteraction`, `flashScrollIndicators`, `getScrollResponder`, `getNativeScrollRef` (→ `getScrollRef`), `getScrollableNode`, and `setNativeProps` all forward unchanged to `_listRef`. If the list isn't mounted yet they silently do nothing.

### SectionList → VirtualizedSectionList → VirtualizedList
- `SectionList` (a `PureComponent`):
  - Defaults `stickySectionHeadersEnabled` to `Platform.OS === 'ios'`. `VirtualizedSectionList` has no default.
  - Supplies array `getItem` and `getItemCount` as inline lambdas.
  - Forwards `scrollToLocation`. The other methods go through `getListRef()` to the inner `VirtualizedList`.
  - Has no `scrollToIndex`, `scrollToItem`, or `scrollToEnd`. Use `getListRef()` on the inner list if you need them.
- **Flattening** (`VirtualizedSectionList.render`): each section contributes `1 header + N items + 1 footer` cells, always, even when `renderSectionHeader` or `renderSectionFooter` is absent (the cell renders `null`). The flat item count is computed in `render`, and `getItemCount={() => itemCount}`. `data` is the `sections` array itself.
- **Index mapping:**
  - `_getItem(props, sections, flatIndex)` returns the section object for header and footer cells, and `props.getItem(section.data, i)` for item cells.
  - `_subExtractor(flatIndex)` is the single source of truth for what a flat index represents. It returns `{section, key, index (null for header/footer), header, leading/trailing item/section}`, using a linear scan over sections.
- **Keys** (`_keyExtractor` → `_subExtractor`):
  - Headers and footers: `${sectionKey}:header` and `${sectionKey}:footer`.
  - Items: `${sectionKey}:${extractor(item, i)}`, where the extractor is the section's `keyExtractor`, then the list's, then the default.
  - `sectionKey` is `section.key`, falling back to `String(sectionIndex)`.
- **Sticky headers:** when enabled, `stickyHeaderIndices` gets each section's header flat index, plus 1 if `ListHeaderComponent` exists. The +1 is needed because VirtualizedList's sticky indices count the list header as a child.
- **Rendering** (`_renderItem(itemCount)` returns a new function on every render):
  - Header and footer cells call `renderSectionHeader` / `renderSectionFooter({section})`.
  - Item cells render `ItemWithSeparator` with the section's `renderItem`, falling back to the list's.
- **Separators** are drawn *inside item cells* by `ItemWithSeparator`, not by VirtualizedList. `ItemSeparatorComponent` is deliberately stripped from the pass-through props.
  - The trailing separator comes from `_getSeparatorComponent`. On the last item of a section it is `SectionSeparatorComponent`. Otherwise it is the section's or list's `ItemSeparatorComponent`, except after the last item of the list.
  - The leading separator is `SectionSeparatorComponent` on item 0 of each section. `inverted` swaps their order.
- **`separators.highlight` / `unhighlight` / `updateProps`:**
  - Each `ItemWithSeparator` registers its own setters in `_updateHighlightMap` and `_updatePropsMap` under its `cellKey`, in an effect.
  - To change the separator *above* it, an item calls `updateHighlightFor(prevCellKey)` or `updatePropsFor(prevCellKey)`. `prevCellKey` is `_subExtractor(index - 1).key`. This is how a cell reaches its neighbour's trailing separator without re-rendering the list.
  - `updateProps('leading')` updates its own leading separator if it has one, and otherwise the previous cell's trailing separator.
- **Viewability:** `_onViewableItemsChanged` passes tokens through `_convertViewable`. That function sets `index` to the in-section index (`null` for header and footer tokens, whose `item` is the section object), recomputes `key` with the user's `keyExtractor`, and adds `section`.
- **`scrollToLocation({sectionIndex, itemIndex, viewOffset})`:**
  - The flat index is `itemIndex + 1 + Σ(previous sections' count + 2)`.
  - When `stickySectionHeadersEnabled`, it adds the section header's length to `viewOffset`, read from `listRef.__getListMetrics().getCellMetricsApprox(headerIndex, …)`. That value is an estimate when the header hasn't been measured.
  - It then calls `scrollToIndex`.
  - Commit `0c35f574474` (#58329, marked `[Breaking]`) fixed an off-by-one where `itemIndex: 0` targeted the header, and a missing sticky offset for item 0. Callers that compensated with `itemIndex + 1` must remove that.

### Traced flow: `<SectionList sections={[A(2 items), B(1 item)]} stickySectionHeadersEnabled />`
1. `SectionList.render` passes `getItemCount = items => items.length` and the default sticky setting to `VirtualizedSectionList`.
2. `VirtualizedSectionList.render` gets `itemCount = (2+2) + (1+2) = 7` and `stickyHeaderIndices = [0, 4]`. It renders `VirtualizedList` with `data=sections`.
3. VirtualizedList asks for cell 2. `_getItem` returns `A.data[1]`, and `_subExtractor(2)` returns `{key: 'A:<k>', index: 1, trailingItem: undefined, …}`.
4. The `_renderItem` closure renders `ItemWithSeparator`. Item 1 is the last item in section A, so its trailing separator is `SectionSeparatorComponent`, if set.
5. `ref.scrollToLocation({sectionIndex: 1, itemIndex: 0})` computes flat index `0 + 1 + (2 + 2) = 5`, adds header B's length (cell 4) to `viewOffset`, and calls `VirtualizedList.scrollToIndex`.

### How the public types are produced
- The source of truth is the Flow types in `FlatList.js`, `SectionList.js`, and `packages/virtualized-lists`.
- `yarn build-types` (`scripts/js-api/build-types`) translates them into per-package `types_generated/` directories (gitignored and published). It also regenerates the committed `packages/react-native/ReactNativeApi.d.ts` snapshot from `index.js.flow`.
  - The snapshot strips doc comments and adds an 8-character shape hash after each export, for example `FlatList, // 76dfb3dc`. The hash changes when any type the export depends on changes shape (`scripts/js-api/README.md`).
  - In the snapshot, `FlatList` and `SectionList` appear as `declare class … extends React.PureComponent`. `Animated.FlatList` and `Animated.SectionList` appear as `$$AnimatedFlatList` and `$$AnimatedSectionList`.
- `FlatListProps` and `SectionListProps` carry `/** @build-types emit-as-interface … */`. `scripts/js-api/build-types/transforms/typescript/convertTypeAliasesToInterfaces.js` then emits them as TS `interface`s, because Nativewind, Uniwind, and Expo/RNW use module augmentation, which needs open interfaces (commits `db89600b562`, `25748636428`).
- `package.json` `exports`: the `types` condition resolves to `types_generated/index.d.ts`, so **the generated types are the real types by default**.
  - `types_DEPRECATED/` (moved there from `types/` in `7cdac2a15c1`, #57513, as part of RFC0894, which removes deep imports) holds the hand-written legacy `.d.ts` files. They resolve only under the opt-in `react-native-legacy-deep-imports` condition (`packages/react-native/__typetests__/tsconfig.legacy.json`, `packages/typescript-config/README.md`).
  - Edit them only to keep the legacy surface in sync. They are feature-locked.
- `@react-native/virtualized-lists` follows the same pattern: `types` resolves to `types_generated`, and the legacy condition resolves to the hand-written `index.d.ts` and `Lists/VirtualizedList.d.ts`.

## Key types and entry points

| Symbol | File | Role |
| --- | --- | --- |
| `FlatList` (class, `FlatListProps<ItemT>`, `FlatListInstance`) | `Libraries/Lists/FlatList.js` | Array/grid adapter over VirtualizedList |
| `_getItem` / `_getItemCount` / `_keyExtractor` / `_renderer` | `FlatList.js` | Row grouping for `numColumns` |
| `_createOnViewableItemsChanged` / `_pushMultiColumnViewable` | `FlatList.js` | Expand row viewability tokens into per-item tokens |
| `_checkProps`, `componentDidUpdate` invariants | `FlatList.js` | Prop validation and the "can't change on the fly" rules |
| `removeClippedSubviewsOrDefault` | `FlatList.js` | Platform and feature-flag default |
| `SectionList` (`SectionListProps`, `SectionListRenderItem[Info]`, `SectionListData`, `SectionBase`) | `Libraries/Lists/SectionList.js` | Public section list with iOS sticky default |
| `VirtualizedSectionList` (`_subExtractor`, `_getItem`, `_renderItem`, `scrollToLocation`, `_convertViewable`) | `packages/virtualized-lists/Lists/VirtualizedSectionList.js` | Flattens sections and maps indices back |
| `ItemWithSeparator` | same | Item cell with leading and trailing separators and highlight/updateProps state |
| `ScrollToLocationParamsType` (= `SectionListScrollParams`) | same | `scrollToLocation` params |
| default export object with lazy getters | `packages/virtualized-lists/index.js` | What the `Libraries/Lists/*` shims re-export |
| `AnimatedFlatList`, `AnimatedSectionList` | `Libraries/Animated/components/` | `createAnimatedComponent(FlatList / SectionList)` |

## Invariants and gotchas
- **These props are fixed at mount; changing them throws an invariant in `componentDidUpdate`:**
  - `numColumns`.
  - Whether `onViewableItemsChanged` is null.
  - `viewabilityConfig` (deep-compared).
  - `viewabilityConfigCallbackPairs` (compared by identity).
  - To change `numColumns`, change FlatList's `key`. That is the remedy the invariant message itself gives (commit `46d6766a53f`).
  - **Inferred** reason: both FlatList's `_virtualizedListPairs` and VirtualizedList's `ViewabilityHelper`s are built once in their constructors, and `numColumns` changes the shape of every item and key that VirtualizedList has cached.
- **`_checkProps` rejects:**
  - `getItem` / `getItemCount` (FlatList doesn't support custom data formats; use VirtualizedList).
  - `numColumns > 1` together with `horizontal`.
  - `columnWrapperStyle` with a single column.
  - Both `onViewableItemsChanged` and `viewabilityConfigCallbackPairs`.
- **With `numColumns > 1`, indices are row indices** for everything passed through: `scrollToIndex`, `getItemLayout`, `initialScrollIndex`, `ItemSeparatorComponent` placement, and `onEndReached` math. Only `renderItem`'s `index` and viewability tokens are per item.
- **`scrollToItem` silently does nothing when `numColumns > 1`.** `VirtualizedList.scrollToItem` compares `getItem(data, i) === item`, and `_getItem` returns a new array for every row. (Found by reading the code; no test covers it.)
- **Row keys combine all of the row's item keys.** Inserting one item near the top re-keys every following row, so those rows remount. (**Inferred** from `_keyExtractor`.)
- **`ListItemComponent` without `strictMode`:** FlatList passes a new `renderProp` function as `ListItemComponent` on every render. VirtualizedList renders it as `<ListItemComponent/>`, so React sees a new component type and **remounts** item subtrees whenever FlatList re-renders. Use `renderItem` or `strictMode`. (**Inferred** from `_renderer` together with `VirtualizedListCellRenderer`.)
- **`strictMode` memoizes on `extraData`, `renderItem`, and the other `_renderer` arguments.** A `renderItem` that reads mutable outside state without a changed `extraData` will render stale items. That is the documented contract of `extraData`.
- **SectionList `getItemLayout` and `onViewableItemsChanged` index spaces:**
  - `getItemLayout` receives `(sections, flatIndex)`, where `flatIndex` counts a header and a footer cell for every section, including empty ones and ones without a renderer.
  - `onViewableItemsChanged` tokens are remapped to in-section indices.
  - **`viewabilityConfigCallbackPairs` on SectionList is not remapped.** It passes through to VirtualizedList, which then ignores the remapped `onViewableItemsChanged`, so those callbacks get raw flat tokens. (Found by reading the code.)
- **SectionList separators sit inside item cells.** Their height is part of the item cell's measured length, which matters for `getItemLayout`. Empty sections get no `SectionSeparatorComponent`.
- **`ListEmptyComponent` on SectionList renders only when `sections` is empty.** A section with empty `data` still produces header and footer cells (Fantom tests "renders section footer when there is no data" and "renders ListEmptyComponent when sections is empty").
- **Sections without `key` fall back to their array index.** Reordering sections then re-keys every cell.
- **`VirtualizedSectionList` re-renders all cells on every render.** It creates new `getItem` and `renderItem` closures each time, and `SectionList` passes new `getItem`/`getItemCount` lambdas each time. Both components are `PureComponent`s, so this happens only when their own props change. (**Inferred**.)
- **Bug in `_setUpdateHighlightFor` (null branch):** it deletes from `this._updateHighlightFor` (the method) instead of `this._updateHighlightMap`. Highlight setters for unmounted cells are never removed from the map. Harmless but leaky.
- **The shims must stay thin.** `Libraries/Lists/Virtualized*.js` and related files only re-type and re-export. Put logic in `packages/virtualized-lists`. That package reaches back into `react-native` only through its peer dependency: public `react-native` imports, plus `react-native/react-private-interface` for feature flags. It never uses relative paths into `packages/react-native`.

## Design decisions

| Decision | Reason / evidence |
| --- | --- |
| VirtualizedList and VirtualizedSectionList live in their own package. FlatList and SectionList stay in `react-native`. | To share one implementation with React Native for Web: #35263, PR #35406, `2e3dbe9c2fb`. Moving FlatList (PR #35423) never landed. |
| Sections are flattened into one VirtualizedList | Class comment in `VirtualizedSectionList`: "Right now this just flattens everything into one list… should be plenty fast for up to ~10,000 items." |
| Every section always gets header and footer cells | Keeps the index math (`+2` per section) uniform in `_getItem`, `_subExtractor`, and `scrollToLocation`. **Inferred**. |
| Sticky section headers default to on only for iOS | Prop doc: "Only enabled by default on iOS because that is the platform standard there." |
| `removeClippedSubviews` defaults to `true` on Android | Prop doc ("default value is true for Android"). The iOS flag added in `fbc090d070c` (#42836) is described as "removeClippedSubviews prop will be used as the default in FlatList on iOS to match Android" (experimentation). |
| Renderer memoization is opt-in through `strictMode` | `c231d5e371c` added `memoizeOne(_renderer)` to stop every cell re-rendering. It is gated, presumably to avoid breaking `renderItem` closures that read state without `extraData` (**Inferred**). |
| Multiple viewability configs | `ad733ad430a` "Extend FlatList to support multiple viewability configs". |
| SectionList highlight and updateProps work through key-indexed callback maps | `76307f47b9b` (separator highlighting for SectionList) and `ad21ad25590` ("Fix stale separator props"). A cell updates its neighbour's separator without re-rendering the list. |
| `scrollToLocation` changed in a breaking way to skip the header and always apply the sticky offset | `0c35f574474` (#58329), diagnosis in issue #50143. |
| Props types are emitted as TS interfaces | Module augmentation by Nativewind and Uniwind (`convertTypeAliasesToInterfaces.js`, `db89600b562`, `25748636428`). |
| Hand-written `.d.ts` files moved to `types_DEPRECATED` | RFC0894 removes deep imports. The legacy surface is feature-locked (`7cdac2a15c1`). |
| List unit tests moved from Jest to Fantom | They now render through real Fabric and Yoga instead of a mocked renderer (`f5c7510a661`, `de82d61f3b0`). |

## Where to change things

| To… | Start at |
| --- | --- |
| Change grid/`numColumns` behaviour, `columnWrapperStyle`, or row keys | `FlatList._getItem`, `_getItemCount`, `_keyExtractor`, `_renderer` |
| Change multi-column viewability tokens | `FlatList._createOnViewableItemsChanged` / `_pushMultiColumnViewable` |
| Change FlatList prop defaults | `removeClippedSubviewsOrDefault`, `numColumnsOrDefault` in `FlatList.js`. For the flag, `ReactNativeFeatureFlags.config.js`, then run `yarn featureflags`. |
| Add a FlatList imperative method | `FlatList.js` (forward to `_listRef`). The method must exist on `VirtualizedList`. |
| Change section flattening, keys, or index mapping | `VirtualizedSectionList._subExtractor` and `_getItem` (keep them in sync with `scrollToLocation` and the `render` count) |
| Change section separators or highlighting | `VirtualizedSectionList._getSeparatorComponent`, `ItemWithSeparator` |
| Change `scrollToLocation` | `VirtualizedSectionList.scrollToLocation`, plus tests in `VirtualizedSectionList-test.js` |
| Change sticky header defaults | `SectionList.render` |
| Change public prop types | The Flow types in `FlatList.js` / `SectionList.js` / `VirtualizedSectionList.js`. Then run `yarn build-types` (which updates `ReactNativeApi.d.ts`) and `yarn test-generated-typescript`. Update `types_DEPRECATED/Libraries/Lists/*.d.ts` only for the legacy surface, then run `yarn test-typescript-legacy`. |
| Export a new list type from `react-native` | `packages/react-native/index.js.flow` (types and values) and `index.js` (runtime getter) |
| Windowing, cell rendering, metrics, sticky header rendering | Not here: see [virtualized-list.md](virtualized-list.md) |

## Tests

| What | Command | Notes |
| --- | --- | --- |
| VirtualizedSectionList unit tests: rendering, separators, nested lists, `scrollToLocation` | `yarn test packages/virtualized-lists/Lists/__tests__/VirtualizedSectionList-test.js` | Jest with `react-test-renderer` snapshots. **Ran at 024b474ce92: 1 suite, 18 tests, 10 snapshots passed (0.33 s).** CI job `test_js`. |
| FlatList integration (props propagated to Fabric, numColumns, viewability, imperative methods) | `yarn fantom FlatList-itest` | Fantom. Builds the native tester the first time. Not run for this doc. |
| SectionList integration (headers, footers, sticky wrappers, separators, empty state) | `yarn fantom SectionList-itest` | Not run. |
| Sticky header render-mask benchmark (100k to 1M rows) | `yarn fantom VirtualizedList-stickyHeaders-benchmark` | `@fantom_mode dev`. Guards the fix in `fe53279889b`. It has no verify function, so CI runs it as a single-iteration smoke test (`private/react-native-fantom/src/Benchmark.js`, `isTestOnly`). Not run. |
| Flow type tests | `yarn flow-check` | Covers `Libraries/Lists/__flowtests__/{FlatList,SectionList}-flowtest.js` (`$FlowExpectedError` cases). Excluded from type generation. |
| Generated and legacy TS types | `yarn test-generated-typescript`, `yarn test-typescript-legacy` | `packages/react-native/__typetests__/` |
| Manual | RNTester → FlatList / SectionList | `FlatListExampleIndex.js`, `SectionListIndex.js` (multi-column, inverted, sticky, separators, viewability variants, MVCP) |

**How CI runs them**
- **Jest:** the `test_js` job.
- **Fantom:** in `.github/workflows/test-all.yml`, `build_fantom_runner` builds the runner, then `run_fantom_tests` runs `fantom-tests.yml` → `.github/actions/run-fantom-tests`, which runs plain `yarn fantom` (the whole suite) up to 3 times. Two more workflow-level retry jobs follow (`run_fantom_tests_retry_1`, `_retry_2`).
- **Maestro:** the flows live in `packages/rn-tester/.maestro/`: `flatlist.yml`, `flatlist-viewability.yml`, `sectionlist-viewability.yml`, and 24 `flatlist-*-maintainvisible.yml` flows tagged `android-release-only`. They run against RNTester builds, never individually:
  - `test_e2e_ios_rntester` → `e2e-ios-rntester.yml` (Debug and Release matrix) → `.github/actions/maestro-ios` → `.github/workflow-scripts/maestro-ios.js`. This runs every `.yml` in the directory in sorted order, skipping `helpers/`, and does **no tag filtering**.
  - `test_e2e_android_rntester` → `e2e-android-rntester.yml` (debug and release) → `.github/actions/maestro-android` → `.github/workflow-scripts/maestro-android.js`. The debug flavor passes `exclude-tags: android-release-only`, so the MVCP flows run only on the release APK ("timing-sensitive" comment in the workflow). Per-flow state is persisted so retries skip flows that already passed.
  - **Maestro Cloud** (`maestro-cloud-rntester.yml`) runs on pushes to main and on same-repo PRs. iOS excludes `android-only,local-screenshot-baseline`, and Android excludes `ios-only,local-screenshot-baseline`.
  - Locally: `yarn --cwd packages/rn-tester e2e-test-android` or `e2e-test-ios` (the iOS script excludes `android-only`). To run a single flow: `maestro test packages/rn-tester/.maestro/flatlist.yml -e APP_ID=…`.
