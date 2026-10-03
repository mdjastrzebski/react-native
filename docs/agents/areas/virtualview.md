<!-- index-codebase: generated from 024b474ce92 on 2026-10-03. Code wins when it disagrees with this doc. -->
# VirtualView

**Purpose:** Experimental host-component virtualization primitive (internal codename "Fling"). A `VirtualView` renders its children only while native code says it is inside, or near, its nearest ancestor ScrollView's viewport. When it is far away, JS unmounts the children and keeps a placeholder size. The visibility decision is made natively, by a per-ScrollView "container state", and sent to JS as `onModeChange` events. It does **not** handle windowing, item recycling, or list data. Those belong to [virtualcollection.md](virtualcollection.md) (VirtualColumn/VirtualRow, built on this) and to the JS-driven lists in [virtualized-list.md](virtualized-list.md) and [flatlist-sectionlist.md](flatlist-sectionlist.md). Viewability callbacks are a separate mechanism: [viewability.md](viewability.md).

**Paths:**
- JS: `packages/react-native/src/private/components/virtualview/` (`VirtualView.js`, `VirtualViewNativeComponent.js`, `VirtualViewExperimentalNativeComponent.js`, `logger/`, `__tests__/VirtualView-itest.js`)
- C++: `packages/react-native/ReactCommon/react/renderer/components/virtualview/` (`VirtualViewShadowNode.h`, `VirtualViewComponentDescriptor.h`, header-only)
- iOS: `packages/react-native/React/Fabric/Mounting/ComponentViews/VirtualView/*`, `.../ScrollView/RCTVirtualViewContainerState.{h,mm}`, `RCTVirtualViewContainerProtocol.h`, `RCTVirtualViewProtocol.h`, hooks in `RCTScrollViewComponentView.mm`
- Android: `packages/react-native/ReactAndroid/src/main/java/com/facebook/react/views/virtual/**`, `views/scroll/VirtualViewContainer.kt`, `VirtualViewContainerStateClassic.kt`, `VirtualViewContainerStateExperimental.kt`, hooks in `ReactScrollView.kt` / `ReactHorizontalScrollView.kt` / `ReactNestedScrollView.kt`

**Depends on:** ScrollView (native), Fabric event emitters, Codegen (`FBReactNativeSpec`), feature flags · **Used by:** [virtualcollection.md](virtualcollection.md) (`VirtualCollectionView.js`, `VirtualColumnGenerator.js`, `VirtualRowGenerator.js`)

## How it works

### Concepts
| Concept | Meaning |
| --- | --- |
| **Mode** (`VirtualViewMode`) | Decided natively. `Visible`=0: overlaps the viewport. `Prerender`=1: overlaps the prerender rect. `Hidden`=2: neither. The numbers are shared by JS, ObjC (`RCTVirtualViewMode.h`) and Kotlin (`VirtualViewMode.kt`). |
| **Render state** (`VirtualViewRenderState`) | A **plain prop** that JS sends back to native: `Rendered`=1 if children are in the last committed tree, `None`=2 if they are not, `Unknown`=0. Native reads it to drop redundant events. It is not C++ state; the ShadowNode has no custom state. |
| **Container state** | One object per ScrollView, created lazily when the first VirtualView under it asks for it. It holds the registered VirtualViews and computes their modes on every scroll and layout change. |
| **Prerender rect** | The visible rect inset by `-width*ratio` and `-height*ratio` on **each side**, where ratio = `virtualViewPrerenderRatio` (default 5). On each axis that is (1 + 2·5) = 11 viewport lengths in total. |
| **targetRect / thresholdRect** | Sent in the event. `targetRect` is the view's rect relative to the scroll content. `thresholdRect` is the visible rect (Visible), the prerender rect (Prerender), or an empty rect (Hidden). VirtualCollection's spacer uses it to decide how many more items to render. |
| **hiddenStyle** | `(targetRect) => style`, merged into `style` while hidden. The default is `{minHeight, minWidth}` of the last `targetRect`, so the placeholder keeps the content's size. |

### Flow: user scrolls an item out of range and back (Android, Classic)
1. **Mount.** `VirtualView.js` `createVirtualView(NotHidden)` renders `VirtualViewNativeComponent` with `initialHidden=false` and `renderState=Rendered`. `ReactVirtualViewManager.setInitialHidden` sets `ReactVirtualView.mode = Visible`, but only if `mode` is null. `addEventEmitters` installs a `VirtualViewEventEmitter`.
2. **Register.** `ReactVirtualView.onAttachedToWindow` → `traverseParentStack`. This walks up to the first `VirtualViewContainer` (one of the three ScrollViews), adds an `OnLayoutChangeListener` to every ancestor on the way, and stops at `ReactRoot`. `onLayout` / `onSizeChanged` compute `containerRelativeRect` (own left/top plus the summed ancestor offsets). If that rect changed, `reportRectChangeToContainer` calls `virtualViewContainerState.onChange(this)`, which adds the view and runs `updateModes(this)`.
3. **Scroll.** `ReactScrollView.onScrollChanged` (also `onLayout` and `onSizeChanged`) → `VirtualViewContainerState.updateState()` → `VirtualViewContainerStateClassic.updateModes()`. It reads `getDrawingRect(visibleRect)`, builds `prerenderRect`, runs `rectsOverlap` for **every** view, and calls `vv.onModeChange(mode, thresholdRect)`.
4. **Filter.** `ReactVirtualView.onModeChange` drops events where the mode is unchanged. It also drops `Visible→Prerender` (children are already rendered). It drops `→Visible` when the old mode was `Prerender` and `renderState == Rendered`, because the prerender result is already committed. `Visible` is emitted with `synchronous = true` (`VirtualViewModeChangeEvent.experimental_isSynchronous`); `Prerender` and `Hidden` are emitted asynchronously.
5. **JS.** `handleModeChange` in `VirtualView.js` handles each mode. `Visible` calls `setState(NotHidden)` at the event's (discrete, synchronous) priority. `Prerender` calls `setState(NotHidden)` inside `startTransition`. `Hidden` calls `setState(hiddenStyle(targetRect))` inside `startTransition`, so the children become `null` and the style is composed with the hidden style. `onModeChange` is called in the same priority context. Its `renderState` field is the state **before** this event is applied.
6. **Commit.** The new `renderState` prop reaches native through `setRenderState`, which closes the loop for step 4.

iOS follows the same steps with different hooks. `RCTVirtualViewComponentView.didMoveToWindow` finds the container through `_getParentVirtualViewContainer`, which walks superviews until one responds to `virtualViewContainerState`. `updateLayoutMetrics` → `[containerState onChange:self]`. `RCTVirtualViewContainerState` registers itself as a scroll listener (`addScrollListener:`) and recomputes every view in `scrollViewDidScroll:`. The rect comes from `convertRect:toView:scrollView`. A synchronous `Visible` is sent by `_dispatchSyncModeChange`, which uses `experimental_flushSync` and `RawEvent::Category::Discrete` and rebuilds the JSI payload by hand. `Prerender` and `Hidden` go through the codegen `emitter.onModeChange`.

### Which native component JS uses
`VirtualView.js` picks the component once, at module load. It uses `VirtualViewNativeComponent` ("VirtualView") if `UIManager.hasViewManagerConfig('VirtualView') && !hasViewManagerConfig('VirtualViewExperimental')`. Otherwise it uses `VirtualViewExperimentalNativeComponent` ("VirtualViewExperimental"). Both specs are identical. OSS native code registers only `VirtualView`: iOS through `RCTFabricComponentsPlugins.mm` `{"VirtualView", VirtualViewCls}`, Android through `MainReactPackage.kt` `ReactVirtualViewManager` (`REACT_CLASS = "VirtualView"`) and `CoreComponentsRegistry.cpp` `VirtualViewComponentDescriptor`. The `VirtualViewExperimental` spec still exists so the JS keeps working against native builds that still ship the old name (9c4b92f8220, 1c94a36a83b). Codegen still generates `VirtualViewExperimentalProps` and `VirtualViewExperimentalManagerDelegate/Interface`, but no ShadowNode or view uses them.

## Key types and entry points
| Symbol | File | Role |
| --- | --- | --- |
| `VirtualView` (default, exported as `unstable_VirtualView`), `VirtualViewMode`, `ModeChangeEvent` | `src/private/components/virtualview/VirtualView.js` | JS component, mode→state mapping, `hiddenStyle` |
| `createHiddenVirtualView(style)` | `VirtualView.js` | Factory for a variant that starts hidden (`initialHidden=true`). Used by the VirtualCollection spacer. |
| `_logs.states` | `VirtualView.js` | Records state values in `__DEV__` for tests. Only filled if a test sets it to an array. |
| `NativeModeChangeEvent`, codegen spec | `VirtualViewNativeComponent.js` | Props `initialHidden`, `renderState`, `removeClippedSubviews`; event `onModeChange`. `interfaceOnly: true`. |
| `useVirtualViewLogging`, `IVirtualViewLogger` | `logger/VirtualViewLogger.js`, `logger/VirtualViewLoggerTypes.js` | Logging hook. In OSS it is a stub that returns `useRef(null)`. |
| `VirtualViewShadowNode` | `ReactCommon/.../virtualview/VirtualViewShadowNode.h` | `ConcreteViewShadowNode<"VirtualView", VirtualViewProps, VirtualViewEventEmitter>` plus the `Unstable_uncullableView` trait |
| `RCTVirtualViewComponentView` | `React/Fabric/.../VirtualView/RCTVirtualViewComponentView.mm` | iOS view: mode filtering, sync/async dispatch |
| `RCTVirtualViewContainerState` | `React/Fabric/.../ScrollView/RCTVirtualViewContainerState.mm` | iOS container. O(n) scan per scroll. |
| `RCTScrollViewComponentView -virtualViewContainerState` | `RCTScrollViewComponentView.mm` | Lazily creates the iOS container. Reset to nil in `prepareForRecycle`. |
| `ReactVirtualView`, `ReactVirtualViewManager`, `VirtualViewEventEmitter` | `ReactAndroid/.../views/virtual/view/` | Android view, manager and emitter |
| `VirtualViewContainer`, `VirtualView` (interface), `VirtualViewContainerState.create` | `ReactAndroid/.../views/scroll/VirtualViewContainer.kt` | Android contracts. `create` picks Classic or Experimental from the flag. |
| `VirtualViewContainerStateClassic` | `.../scroll/VirtualViewContainerStateClassic.kt` | O(n) scan; notifies every view |
| `VirtualViewContainerStateExperimental`, `IntervalTree` | `.../scroll/VirtualViewContainerStateExperimental.kt` | AVL interval tree plus V/P/PV sets; notifies only views whose mode changed |

## Invariants and gotchas
- **A ScrollView ancestor is required.** Without one, no mode events are ever sent. iOS `_parentVirtualViewContainer` stays nil. On Android `traverseParentStack` returns null and `onModeChange` exits early. A plain `VirtualView` then always renders its children. A `createHiddenVirtualView` instance **never shows** anything.
- **Nearest container wins.** Only the nearest ScrollView ancestor counts. Android stops searching at `ReactRoot`. Containers are `ReactScrollView`, `ReactHorizontalScrollView`, `ReactNestedScrollView`, and on iOS `RCTScrollViewComponentView`.
- **Mode transitions are filtered on purpose.** `Visible→Prerender` emits nothing, so children stay rendered until the view is `Hidden`. Do not "fix" this unless you also change the JS state machine. The Fantom test "changes mode from prerender to visible" checks that no extra state update happens.
- **Priority asymmetry.** `Visible` is synchronous and discrete, so it never shows a blank on screen. Prerender and Hidden are transitions and can be interrupted. If you add a mode, keep both platforms' sync/async split and the JS `startTransition` choice consistent.
- **`renderState` lags by one commit.** Native only knows what JS last committed. That is why `→Visible` is still emitted after `Hidden`, even if `renderState` reads `Rendered`: the Hidden update may still be pending (see the comment in both `onModeChange` implementations).
- **Uncullable.** `VirtualViewShadowNode::BaseTraits` sets `Unstable_uncullableView`. The code comment: "It must not be culled, otherwise Fling will not work." **Inferred:** view culling (`enableViewCulling`) would leave off-screen VirtualViews unmounted, so they could never measure themselves or receive scroll-driven events.
- **Hidden = unmounted children.** Children become `null`, which frees fibers, state and shadow nodes (Fantom "memory management" tests). Wrapping them in `Activity` was tried and dropped, because it costs memory and raises OOM risk (b9294a70fb3).
- **The default hidden size is `minHeight` + `minWidth`.** It does not use `flexBasis`, because `hiddenStyle` cannot know the parent's flex direction (d6ed32f8d64). It is a *min* size, so a placeholder can still grow.
- **iOS does not recompute on ScrollView resize.** It recomputes only on `scrollViewDidScroll:` and on per-VirtualView `updateLayoutMetrics` / `didMoveToWindow`. `TODO(T202601695)`: content outside the VirtualViews that changes size is not handled. Android also recomputes on ScrollView `onLayout` and `onSizeChanged`.
- **Rect math differs by platform.** iOS uses `convertRect:toView:` (includes transforms). **Inferred:** Android sums ancestor `left`/`top` in `updateParentOffset`, so ancestor transforms and translations are ignored.
- **Android Experimental is 1-D.** `IntervalTree` keys on the main axis only: `ReactHorizontalScrollView` uses horizontal intervals, everything else (including `ReactNestedScrollView`) uses vertical ones. The cross axis is ignored.
- **Empty viewport.** Android Classic returns early with no events when `visibleRect.isEmpty()` (the ScrollView content is not ready). **Inferred:** Experimental continues after `updateRects`, so already-tracked views can be sent to `Hidden`.
- **Clipping (Android).** `ReactVirtualView.updateClippingRect` intersects the ScrollView's clipping rect, or its drawing rect, with its own rect. This clips the VirtualView's *children*, never the VirtualView itself. It happens even when the ScrollView has `removeClippedSubviews` off (b0e754bc7f7, 025e0e47ef6). `removeClippedSubviews` is declared explicitly in the spec because the generated delegate did not call the setter for the spread `ViewProps` (TODO in the spec). **Inferred:** Experimental notifies only views whose mode changed, so a view that stays `Visible` does not get the `updateClippingRect(null)` that Classic triggers on every scroll.
- **Detach resets everything (Android).** `onDetachedFromWindow` → `recycleView()` clears `mode`, `hadLayout`, and **`modeChangeEmitter`**. The emitter is only set in `addEventEmitters`, at creation.
- **IDs.** iOS `virtualViewID` is `nativeID` or a UUID. Android uses `"${nativeId ?: "unknown"}:::${viewTag}"`. Experimental keys its sets and tree by this ID.
- **iOS leftovers.** `sIsAccessibilityUsed` is set but nothing acts on it, and `_unhideIfNeeded` only clears `hidden`. Both are left over from the removed `hideOffscreenVirtualViewsOnIOS` flag (3233436177c). The `accessibilityElementCount` and `focusItemsInRect:` overrides remain.
- **Accessibility naming collision.** `getVirtualViewAt`, `onPopulateNodeForVirtualView` and related methods in `ReactAccessibilityDelegate.kt` and `ReactTextViewAccessibilityDelegate.kt` belong to Android's `ExploreByTouchHelper` "virtual views". They are **unrelated** to this component.
- **Fantom output.** The Fantom snapshots render `<rn-virtualViewExperimental>`, so in Fantom the selector picks the Experimental spec. **Inferred:** the Fantom tester does not report `VirtualView` from `__nativeComponentRegistry__hasComponent`.

## Design decisions
| Decision | Reason / evidence |
| --- | --- |
| Intersection logic lives in the ScrollView container, not in each VirtualView | One scroll listener instead of N (b3f397f343b, e40c10b9f61). The legacy per-view implementation was deleted in 74b8ddeb357. |
| Native decides the mode; JS only renders | No JS round trip on every scroll. JS gets an event only when the mode changes. **Inferred** from the architecture. |
| `renderState` prop fed back to native | Avoids redundant synchronous `Visible` events once a Prerender has been committed (08a59c59e09) |
| Visible is sync and discrete; Prerender and Hidden use transitions | Content entering the viewport must render this frame; off-screen work can be deferred. iOS hand-copies the codegen payload to use `experimental_flushSync` (TODO: custom event emitter). |
| Experimental interval-tree container (Android only) | `updateModes` goes from O(n) to O(m + log n). Add and update become O(log n) and memory grows by O(m) (328d07a30de). Fixes: 6f8432330c0 (AVL rotation), f2898d4b162 (set leak in `remove`). "iOS may follow after experimentation." |
| Configurable `hiddenStyle` | Lets row and grid layouts be virtualized, not just columns (d6ed32f8d64) |
| Experiments that never shipped were removed | Hysteresis window (bb980b0c084), window-focus detection (d339c6a2c9e), Activity (b9294a70fb3), debug overlay (083fd99ba4f), iOS `hidden` for offscreen views (3233436177c). Do not reintroduce them without new data. |
| Logger is a hook with a null ref in OSS | A seam for lifecycle and mode-change logging (`logMount`, `logModeChange`, `logUnmount`) (6515ada02f8). **Inferred:** Meta supplies a real implementation internally. |

## Where to change things
| To… | Start at |
| --- | --- |
| Change how JS reacts to modes (priority, placeholder, children) | `VirtualView.js` `handleModeChange` |
| Change props or the event payload | Both `VirtualViewNativeComponent.js` and `VirtualViewExperimentalNativeComponent.js`, then iOS `_dispatchSyncModeChange` (hand-written payload), Android `VirtualViewModeChangeEvent.getEventData`, the manager setters, and the API snapshots |
| Change the visible/prerender/hidden calculation | iOS `RCTVirtualViewContainerState._updateModes`; Android `VirtualViewContainerStateClassic.updateModes` **and** `VirtualViewContainerStateExperimental.updateMode`/`updateModesAll` |
| Change event deduplication | `onModeChange` in `RCTVirtualViewComponentView.mm` and `ReactVirtualView.kt` (keep them in sync) |
| Hook a new scroll container | Implement `RCTVirtualViewContainerProtocol` (iOS) or `VirtualViewContainer` (Android), and call `updateState()` on scroll and layout |
| Tune the prerender distance or switch the container algorithm | Flags in `packages/react-native/scripts/featureflags/ReactNativeFeatureFlags.config.js`, then `yarn featureflags` (never edit the generated accessors) |
| Public API impact | `ReactNativeApi.d.ts` contains `VirtualViewMode`, `ModeChangeEvent`, `NativeModeChangeEvent`, `VirtualViewRenderState` (`unstable_VirtualView` is stripped by `stripUnstableApis`); regenerate with `yarn build-types`. `ReactAndroid/api/ReactAndroid.api` contains `views/virtual/*`, `views/scroll/VirtualView*`, and the generated `VirtualView{,Experimental}Manager{Delegate,Interface}`. The C++ snapshots in `scripts/cxx-api/api-snapshots/*.api` contain `VirtualViewShadowNode` and `VirtualViewComponentDescriptor` (`yarn cxx-api-build`). |
| Build wiring | iOS `React-FabricComponents.podspec` subspec `virtualview`, `Package.swift` `virtualViewPath`. Android `CoreComponentsRegistry.cpp`, `MainReactPackage.kt`. |

## Feature flags
| Flag | Default | Platforms read | Effect |
| --- | --- | --- | --- |
| `enableVirtualViewContainerStateExperimental` | `false` | Android (`VirtualViewContainerState.create`) | Interval-tree container instead of Classic. iOS has no Experimental variant. |
| `virtualViewPrerenderRatio` | `5` | iOS and Android (read once, at container creation) | Prerender inset multiplier, per side |

Both flags are `common`, with `ossReleaseStage: 'none'`.

## Tests
- Fantom: `yarn fantom packages/react-native/src/private/components/virtualview/__tests__/VirtualView-itest.js` (**not run**; builds a native tester on first run). It covers JS mode→render transitions, hidden `minHeight`/`minWidth`, and that children's instances, state and shadow nodes are released when hidden. It sends events by hand (`Fantom.dispatchNativeEvent(..., 'onModeChange', ...)`), so it does **not** exercise the native container logic.
- No native unit tests exist for the container states, `IntervalTree`, or `ReactVirtualView` (the legacy `ReactVirtualViewTest.kt` was deleted in 74b8ddeb357).
- RNTester has no VirtualView example. Exercise it through the VirtualCollection consumers ([virtualcollection.md](virtualcollection.md)).
