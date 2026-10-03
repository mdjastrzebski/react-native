# maintainVisibleContentPosition Design and Architecture

## 1. Overview

### 1.1 Purpose

`maintainVisibleContentPosition` (MVCP) is a React Native prop on `ScrollView`
and `FlatList` that prevents unwanted scroll jumps when items are prepended to a
list. When new items are added at the beginning of a list, MVCP computes a
scroll delta correction to keep the previously visible content at the same
screen position.

### 1.2 What Problem It Solves

Without MVCP, prepending items to a scrollable list causes all existing content
to shift downward visually — the scroll offset remains the same but the user's
visible content changes. This creates a jarring UX where the user loses their
place in the list. MVCP compensates by adjusting `contentOffset` by the same
amount that the anchor view shifted, keeping the visible content locked to the
same screen position.

### 1.3 Scope

- **Components:** `ScrollView` (iOS Fabric, Android), `FlatList` (via
  ScrollView)
- **Operations handled:** Prepends, appends, deletes, item updates, size changes
- **Axes:** Vertical and horizontal lists
- **Modes:** Normal and inverted lists
- **Layers:** JS (detection), Native iOS Fabric, Native Android

**Note:** iOS Paper (Legacy) MVCP implementation was removed in June 2026
(commit `86350ab9884`, PR #56831). Only iOS Fabric and Android retain MVCP
support.

---

## 2. Architecture Overview

### 2.1 High-Level Design

MVCP follows a **three-layer split** between JS and native:

```text
┌──────────────────────────────────────────────────────────┐
│  JS Layer (VirtualizedList / ScrollView)                 │
│  - Detects prepends via firstVisibleItemKey comparison   │
│  - Tracks pendingScrollUpdateCount                       │
│  - Suppresses render window updates during correction    │
│  - Forwards maintainVisibleContentPosition prop to native│
└────────────────────────┬─────────────────────────────────┘
                         │ prop: {minIndexForVisible, autoscrollToTopThreshold}
┌────────────────────────▼────────────────────────────────┐
│  Native Layer (per-platform)                            │
│  ┌─────────────┐  ┌────────────────┐                    │
│  │ iOS Fabric  │  │   Android      │                    │
│  │ Mounting    │  │ UIManager      │                    │
│  │ Transaction │  │ Listener       │                    │
│  │ Callbacks   │  │ Callbacks      │                    │
│  └─────────────┘  └────────────────┘                    │
│                                                         │
│  Core algorithm (all platforms):                        │
│  1. Capture anchor view frame before mount              │
│  2. Apply mutations (insert, remove, update)            │
│  3. Compute delta = newFrame - oldFrame                 │
│  4. Apply delta to contentOffset                        │
│                                                         │
│  Anchor recomputation happens in the next cycle's       │
│  WILL_MOUNT phase, not in DID_MOUNT.                    │
└─────────────────────────────────────────────────────────┘
```

### 2.2 Core Algorithm: Frame Delta Correction

All platforms implement the same fundamental algorithm:

```text
delta = anchorFrameAfterMount - anchorFrameBeforeMount
contentOffset += delta
```

Frames are **snapshots**, not live references. Once captured, frame values don't
change even if the view is later moved or removed. This makes the approach safe
across all platforms.

---

## 3. Key Components

### 3.1 JS Layer

#### 3.1.1 VirtualizedList (`packages/virtualized-lists/Lists/VirtualizedList.js`)

**Responsibilities:**

- Detects prepends by comparing `firstVisibleItemKey` across renders
- Computes `maintainVisibleContentPositionAdjustment` (prepend delta in item
  count)
- Sets `pendingScrollUpdateCount` to `1` to suppress render window updates
- Adjusts `cellsAroundViewport` render window by the adjustment amount

**Key state field:** `pendingScrollUpdateCount` — dual-purpose:

1. Initial scroll index tracking (set to `1` when `initialScrollIndex > 0`)
2. MVCP adjustment tracking (set to `1` on prepend detection, reset to `0` on
   the next scroll event)

**Detection flow (in `getDerivedStateFromProps`):**

```js
// When maintainVisibleContentPosition != null and the item count changed:
if (firstVisibleItemKey changed between renders) {
    // Item was prepended — find where the previous anchor is now
    newAdjustment = firstVisibleItemIndex - minIndexForVisible
    cellsAroundViewport shifted by adjustment
    pendingScrollUpdateCount = 1
}
```

**Guard interactions:**

- `_adjustCellsAroundViewport`: Returns early when
  `pendingScrollUpdateCount > 0`, preventing render window updates during MVCP
  corrections
- `_maybeCallOnEdgeReached`: Suppresses edge callbacks while
  `pendingScrollUpdateCount > 0` (`_updateViewableItems` is gated the same way)

#### 3.1.2 ScrollView (`packages/react-native/Libraries/Components/ScrollView/ScrollView.js`)

**Responsibilities:**

- Passes `maintainVisibleContentPosition` prop through to native component
- Sets `collapsableChildren={false}` on the content container when MVCP is
  active (via `preserveChildren`), so Fabric does not flatten its children —
  critical for stable native view references

**Prop type:**

```js
maintainVisibleContentPosition?: ?{
  minIndexForVisible: number,
  autoscrollToTopThreshold?: ?number,
}
```

#### 3.1.3 ListMetricsAggregator (`packages/virtualized-lists/Lists/ListMetricsAggregator.js`)

**Responsibilities:**

- Tracks cell layout metrics for approximate sizing
- Clears metrics on orientation change (prevents stale metric corruption)
- Guards against divide-by-zero in `_averageCellLength` computation

**Key state:**

- `_cellMetrics: Map<string, CellMetrics>` — per-cell layout info
- `_measuredCellsCount: number` — count of measured cells
- `_averageCellLength: number` — computed average, guarded by `if (count > 0)`

### 3.2 iOS Fabric Layer

#### 3.2.1 RCTScrollViewComponentView (`RCTScrollViewComponentView.mm`)

**Mounting transaction callbacks:**

- `mountingTransactionWillMount:` — triggers
  `_prepareForMaintainVisibleScrollPosition`
- `mountingTransactionDidMount:` — triggers `_remountChildren` then
  `_adjustForMaintainVisibleContentPosition`

**Core methods:**

- `_prepareForMaintainVisibleScrollPosition` — recomputes anchor before mount;
  scans subviews to find first visible view
- `_adjustForMaintainVisibleContentPosition` — computes delta, applies
  correction

**State variables:**

| Variable                                            | Type              | Purpose                                                                                                             |
| --------------------------------------------------- | ----------------- | ------------------------------------------------------------------------------------------------------------------- |
| `_prevFirstVisibleFrame`                            | `CGRect`          | Captured frame of anchor before mount                                                                               |
| `_firstVisibleView`                                 | `__weak UIView *` | Reference to current first visible subview                                                                          |
| `_firstVisibleViewTag`                              | `NSInteger`       | Tag for recycle detection                                                                                           |
| `_avoidAdjustmentForMaintainVisibleContentPosition` | `BOOL`            | Skip gate set to `enableImmediateUpdateModeForContentOffsetChanges` only while `_updateStateWithContentOffset` runs |

**Tag comparison safeguard:**

```objc
// Abort if the first visible view has been recycled for a different item.
// The tag was captured in _prepareForMaintainVisibleScrollPosition (before
// mounting), and RCTComponentViewRegistry assigns new tags during dequeue
// (mounting) and resets them to 0 during enqueue (unmounting). When items
// are removed and re-added, recycled views get new tags based on their
// position, so the view at position 0 may have a different tag than before.
// If the tag changed, we bail out to avoid applying the MVCP delta to the
// wrong view, which would produce incorrect scroll offsets.
if (_firstVisibleView.tag != _firstVisibleViewTag) {
    return;  // View was recycled - abort correction
}
```

**Status:** Always active. `RCTComponentViewRegistry` assigns tags during
dequeue (`componentViewDescriptor.view.tag = tag`) and resets to 0 during
enqueue (`componentViewDescriptor.view.tag = 0`). When items are removed and
re-added, recycled UIViews get the React tag of the new ShadowView they now
host, so the check must always run.

#### 3.2.2 RCTComponentViewRegistry (`RCTComponentViewRegistry.mm`)

**Recycle pool mechanics:**

- Pool size: 1024 views per component type
- **Enqueue:** Delete mutations -> `prepareForRecycle()` -> push to pool
- **Dequeue:** Create mutations -> pop from pool -> set new tag -> register
- **Memory pressure:** Clears entire pool on `didReceiveMemoryWarning`

### 3.3 Android Layer

#### 3.3.1 MaintainVisibleScrollPositionHelper (`MaintainVisibleScrollPositionHelper.kt`)

**Class signature:**

```kotlin
internal class MaintainVisibleScrollPositionHelper<ScrollViewT>(
    private val scrollView: ScrollViewT,
    private val horizontal: Boolean,
) : UIManagerListener
    where
        ScrollViewT : HasScrollEventThrottle?,
        ScrollViewT : HasSmoothScroll?,
        ScrollViewT : ViewGroup?
```

**State variables:**

| Variable                | Type                   | Purpose                                        |
| ----------------------- | ---------------------- | ---------------------------------------------- |
| `config`                | `Config?`              | MVCP configuration                             |
| `firstVisibleViewRef`   | `WeakReference<View>?` | Anchor view reference (auto-nullifies if GC'd) |
| `prevFirstVisibleFrame` | `Rect?`                | Captured frame of anchor                       |
| `isListening`           | `Boolean`              | Whether listener is active                     |

**Lifecycle callbacks:**

- `willDispatchViewUpdates` — Paper-only `UIManagerListener` callback; never
  invoked under Fabric
- `willMountItems` — calls `computeTargetView()` (pre-layout capture)
- `didMountItems` — calls `updateScrollPositionInternal()`

**`computeTargetView`:**

- Iterates from `config.minIndexForVisible` through `contentView.childCount`
- Selects first child where `position > currentScroll` or the last child
- Stores `WeakReference(child)` in `firstVisibleViewRef`
- Captures `child.getHitRect(frame)` into `prevFirstVisibleFrame`

**`updateScrollPositionInternal`:**

- Retrieves cached `firstVisibleViewRef` and `prevFirstVisibleFrame` (captured
  by `willMountItems`)
- Computes delta on `left` (horizontal) or `top` (vertical) coordinates
- `scrollToPreservingMomentum()`
- Updates `prevFirstVisibleFrame` to new frame after correction
- Early return if `firstVisibleViewRef.get()` is null (view GC'd)
- **Threshold:** Uses `delta != 0` (vs iOS `ABS(delta) > 0.5`)

#### 3.3.2 ReactScrollView (`ReactScrollView.kt`)

**MVCP field:**

```kotlin
private var maintainVisibleContentPositionHelper: MaintainVisibleScrollPositionHelper<ReactScrollView>? = null
```

**`setMaintainVisibleContentPosition`:**

- `config != null && helper == null`: creates new helper with
  `horizontal = false`, calls `start()`
- `config == null && helper != null`: calls `stop()`, sets helper to `null`
- Helper exists: updates its `config`

**`horizontal` flag:** Hardcoded to `false` — `ReactScrollView` only supports
vertical scrolling (`ReactHorizontalScrollView` passes `true`).

**Lifecycle integration:**

- `onAttachedToWindow`: calls `helper.start()`
- `onDetachedFromWindow`: calls `helper.stop()`

#### 3.3.3 ReactViewGroup Content Culling (`ReactViewGroup.kt`)

**Culling mechanism:**

- `allChildren` array: stores ALL children (visible + culled) for O(1)
  re-addition
- `removeClippedSubviews` boolean: enables culling
- Off-screen children: `removeViewInLayout()` — detached but kept in
  `allChildren`
- On-screen children: `addViewInLayout()` — re-attached from `allChildren`

**MVCP interaction:** `computeTargetView` iterates `contentView.childCount`
(visible children only, not `allChildrenCount`), meaning culling affects anchor
candidate selection.

---

## 4. Events & Lifecycle

### 4.1 Mount Cycle Events

Both active platforms follow the same high-level pattern:

```text
1. WILL_MOUNT (before mutations):
    - Capture anchor view's frame -> prevFirstVisibleFrame
   - Store anchor view reference -> firstVisibleView

2. MOUNT (mutations applied):
   - New items inserted, existing items shifted
   - Layout computed, frames updated

3. DID_MOUNT (after mutations):
   - Compute delta = (anchor's frame now) - (captured frame)
   - Apply delta to contentOffset

Anchor recomputation happens in the next cycle's WILL_MOUNT phase, not in DID_MOUNT.
```

#### 4.1.1 iOS Fabric Event Flow

```text
RCTMountingManager.performTransaction:
  |
  +-- _observerCoordinator.notifyObserversMountingTransactionWillMount
  |      -> RCTScrollViewComponentView.mountingTransactionWillMount
  |         -> _prepareForMaintainVisibleScrollPosition
  |            -> Scan _contentView.subviews from minIndexForVisible
  |            -> Find first partially visible subview
  |            -> Store: _firstVisibleView, _firstVisibleViewTag, _prevFirstVisibleFrame
  |
  +-- RCTPerformMountInstructions (mutations applied)
  |      Create: dequeue from RCTComponentViewRegistry
  |      Delete: enqueue to RCTComponentViewRegistry
  |      Update: update existing views
  |
  +-- _observerCoordinator.notifyObserversMountingTransactionDidMount
  |      -> RCTScrollViewComponentView.mountingTransactionDidMount
  |         -> _remountChildren (no-op when enableViewCulling is true;
  |            calls updateClippedSubviewsWithClipRect when false)
  |         -> _adjustForMaintainVisibleContentPosition
  |            -> Tag comparison check (always active)
  |            -> delta = _firstVisibleView.frame - _prevFirstVisibleFrame
  |            -> Abort if ABS(delta) <= 0.5
  |            -> contentOffset += delta
  |            -> autoscrollToTopThreshold check (animate to start if near top)
```

#### 4.1.2 Android Event Flow

```text
MountItemDispatcher.dispatchMountItems:
  |
  +-- FabricUIManager.MountItemDispatchListener.willMountItems
  |      -> MaintainVisibleScrollPositionHelper.willMountItems
  |         -> computeTargetView() [pre-layout capture]
  |
  +-- View mount / layout updates
  |      Children added/removed from contentView
  |      UPDATE_LAYOUT: view.measure() + view.layout() — frames set here
  |      Culling: off-screen children removed from children (kept in allChildren)
  |
  +-- FabricUIManager.MountItemDispatchListener.didMountItems
  |      -> MaintainVisibleScrollPositionHelper.didMountItems
  |         -> updateScrollPositionInternal()
  |            -> firstVisibleView = firstVisibleViewRef.get()
  |            -> if firstVisibleView != null:
  |            ->    delta = firstVisibleView.frame - prevFirstVisibleFrame
  |            ->    if delta != 0: scrollToPreservingMomentum(currentScroll + delta)
  |            ->    Update prevFirstVisibleFrame to new frame
```

### 4.2 Scroll Events

**JS-side scroll event handling (`_onScroll` in VirtualizedList):**

```js
if (this.state.pendingScrollUpdateCount > 0) {
  this.setState({pendingScrollUpdateCount: 0});
}
```

Any scroll event resets `pendingScrollUpdateCount` to 0, after which render
window updates resume and edge callbacks are re-enabled. It is reset rather than
decremented because native `onScroll` events are coalesced
(`dispatchUniqueEvent`), so N native corrections can arrive in JS as a single
event (commit `5cb65244dc5`).

### 4.3 Observer Registration Lifecycle

| Platform   | Registration Trigger                                                                  | Deregistration Trigger                                               |
| ---------- | ------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| iOS Fabric | Create mutation (`RCTMountingTransactionObserverCoordinator`, for every scroll view)  | Delete mutation                                                      |
| Android    | `setMaintainVisibleContentPosition:` config != null (creates helper, calls `start()`) | `setMaintainVisibleContentPosition:` config == null (calls `stop()`) |

---

## 5. Code Flows

### 5.1 Normal Operation — Single Prepend

```text
User scrolls to item 5 (offset = 500)
  |
  v
JS renders: [X, A, B, C, D, E, F, G, H]  (prepend 1 item)
  |
  v
VirtualizedList detects firstVisibleItemKey changed
  |
  v
JS computes adjustment = 1 (one item prepended above minIndexForVisible)
JS sets pendingScrollUpdateCount = 1
JS shifts cellsAroundViewport by 1
  |
  v
Native: Capture anchor (first visible view at offset 500)
Native: _prevFirstVisibleFrame = {y: 500}
  |
  v
Native: Mount mutations applied
Native: X inserted at index 0, all items shift down by item height
Native: Anchor now at y = 550
  |
  v
Native: delta = 550 - 500 = 50
Native: contentOffset += 50 -> offset = 550
Native: Anchor stays at same screen position (550 - 550 = 0, top of viewport)
  |
  v
Next cycle's WILL_MOUNT: Recompute anchor for next correction
JS: Scroll event fires -> pendingScrollUpdateCount reset to 0
JS: Render window updates resume
```

### 5.2 Rapid Consecutive Prepends

```text
First prepend:
  _prepare (pre-mount):   capture A at y=0  [stale frame, pre-layout]
  mount + layout:         A moves to y=100  [frames updated]
  _adjust:                delta = 100 - 0 = 100, offset = 100

Second prepend:
  _prepare (pre-mount):   capture A at y=100  [stale frame, but from correct layout pass]
  mount + layout:         A moves to y=200
  _adjust:                delta = 200 - 100 = 100, offset = 200

Why this works:
  _prepare runs BEFORE layout blocks fire, so it always captures
  a stale frame from the previous layout pass. The delta is computed
  as (post-layout frame) - (pre-layout frame), which correctly
  represents the frame shift caused by the mount.

  The _prepare capture for batch N+1 uses the frame that was captured
  by _prepare in batch N (which was stale from N-1's layout). But
  since the delta is computed from the ACTUAL post-layout frame
  minus that captured frame, the delta is still correct.

Additional role: anchor re-selection
  If the correction pushed A off-screen, the next _prepare finds the new
  first visible view (e.g., B). The next _prepare then anchors to B instead of
  the now-invisible A. Without recomputation, stale frame data from the wrong view would
  be used for the next correction.
```

### 5.3 View Recycling — iOS Fabric

```text
Initial state: [A, B, C, D, E], anchor = B at y=100
  _firstVisibleView = vB, _firstVisibleViewTag = 101
  _prevFirstVisibleFrame = {y: 100}

User removes A, adds X at top: [X, B, C, D, E]
  Differ generates: Delete A, Create X, Update B,C,D,E

Delete A: vA.tag = 0 -> enqueue to recycle pool
Create X: dequeue vA from pool -> set vA.tag = 200 (X's tag)
          SAME UIView object, NEW tag

Mount: B moves to index 1, frame.y = 150
  _firstVisibleView still points to vB (same object, tag unchanged)

Tag check:
  _firstVisibleView.tag (101) != _firstVisibleViewTag (101) -> PASS
  (Tag check would fail if _firstVisibleView was recycled)

delta = 150 - 100 = 50
contentOffset += 50
```

**Bug scenario when anchor is recycled:** If the anchor view itself happens to
be recycled (deleted, then reused with a new tag), the tag comparison detects
the mismatch and aborts the correction. The next batch will recompute and
correct from fresh data.

> **Important:** The tag check is **always active** (no feature flag gate).
> `RCTComponentViewRegistry` assigns tags during dequeue and resets to 0 during
> enqueue. When items are removed and re-added, recycled UIViews get the React
> tag of the new ShadowView they now host, so the check must always run. This
> was confirmed by the `flatlist-inverted-recycle-maintainvisible` maestro test,
> which failed when the tag check was gated behind `enableViewCulling()` (which
> returns false in RNTester).

### 5.4 Empty List / Data Reset

**iOS Fabric (safe):** When the list becomes empty,
`_prepareForMaintainVisibleScrollPosition` finds no candidate (its loop body
doesn't run), leaving the previous `_firstVisibleView` in place.
`_adjustForMaintainVisibleContentPosition` then aborts: the nil check catches a
deallocated anchor, and the tag check catches one that was deleted (tag reset
to 0) or recycled.

**Android (safe):** `updateScrollPositionInternal` checks
`firstVisibleViewRef.get() ?: return` — early return if view is null. No
incorrect correction.

---

## 6. State Management

### 6.1 State Variables by Platform

| Variable              | iOS Fabric                                          | Android                               |
| --------------------- | --------------------------------------------------- | ------------------------------------- |
| Anchor view reference | `_firstVisibleView` (UIView\*)                      | `firstVisibleViewRef` (WeakReference) |
| Anchor view tag       | `_firstVisibleViewTag` (NSInteger)                  | N/A                                   |
| Captured frame        | `_prevFirstVisibleFrame` (CGRect)                   | `prevFirstVisibleFrame` (Rect)        |
| Config                | `props.maintainVisibleContentPosition`              | `config` (Config object)              |
| Skip gate             | `_avoidAdjustmentForMaintainVisibleContentPosition` | N/A                                   |

### 6.2 Recomputation Pattern Detail

The recomputation happens at the start of each mount transaction:

```text
Phase 1: _prepareForMaintainVisibleScrollPosition / willMountItems
  Purpose: Capture anchor that reflects the current (post-layout, pre-mount) state
  Executes: Before mount mutations are applied
  Result: Fresh _firstVisibleView, _firstVisibleViewTag, _prevFirstVisibleFrame

Phase 2: _adjustForMaintainVisibleContentPosition / didMountItems
  Purpose: Compute and apply scroll correction
  Step 1: delta = newFrame - prevFirstVisibleFrame
  Step 2: contentOffset += delta
  Step 3: (Android only) Update prevFirstVisibleFrame to new frame
```

### 6.3 JS-Side pendingScrollUpdateCount

The `pendingScrollUpdateCount` field in VirtualizedList state serves dual
purposes:

1. **Initial scroll index:** Set to `1` when `initialScrollIndex > 0`,
   preventing render window updates until a valid scroll offset is received from
   native.

2. **MVCP adjustment tracking:** Set to `1` when a prepend is detected
   (JS-side), reset to `0` on the next scroll event. While > 0:
   - `_adjustCellsAroundViewport` returns early (no render window updates)
   - `_maybeCallOnEdgeReached` is suppressed (edge callbacks don't fire on stale
     metrics)

This prevents the list from adjusting its render window while native-side MVCP
corrections are still settling.

---

## 7. Safeguards & Edge Cases

### 7.1 Tag Comparison Safeguard (iOS Fabric)

**Purpose:** Detect when the anchor view was recycled (deleted and recreated
with a new tag) during mount.

**Implementation:**

```objc
// Abort if the first visible view has been recycled for a different item.
// The tag was captured in _prepareForMaintainVisibleScrollPosition (before
// mounting), and RCTComponentViewRegistry assigns new tags during dequeue
// (mounting) and resets them to 0 during enqueue (unmounting). When items
// are removed and re-added, recycled views get new tags based on their
// position, so the view at position 0 may have a different tag than before.
// If the tag changed, we bail out to avoid applying the MVCP delta to the
// wrong view, which would produce incorrect scroll offsets.
if (_firstVisibleView.tag != _firstVisibleViewTag) {
    return;  // View was recycled - abort correction
}
```

**How it works:**

- `_prepareForMaintainVisibleScrollPosition` captures `_firstVisibleView` and
  `_firstVisibleViewTag` (the view's React tag)
- During mount, `RCTComponentViewRegistry` dequeues views from the recycle pool
  and assigns new tags (`componentViewDescriptor.view.tag = tag`), or resets
  tags to 0 during enqueue
- When items are removed and re-added, the same UIView objects may be reused for
  different items with new tags (the React tag of the ShadowView they now host)
- `_adjustForMaintainVisibleContentPosition` compares the current tag with the
  captured tag
- If tags differ → view was recycled → abort correction (avoids applying delta
  to wrong view)

**Why the check is always active:** `RCTComponentViewRegistry` assigns tags
during dequeue and resets to 0 during enqueue, regardless of culling state. When
items are removed and re-added, recycled UIViews get the React tag of the new
ShadowView they now host, so the check must always run.

**Impact:** When the anchor view is recycled, MVCP correctly aborts and waits
for the next batch to recompute from fresh data. Without this check, MVCP would
apply an incorrect delta to the wrong view, producing incorrect scroll offsets.

### 7.2 Deletion Check (iOS Fabric)

**Purpose:** Detect when the anchor view was removed from `_contentView` during
mount without being deleted (e.g. reparented). Deleted anchors, such as during a
`setData([])` + `scrollToOffset(0)` reset, are already caught by the tag check,
because every Delete enqueues the view and resets its tag to 0.

**Implementation:**

```objc
if (_firstVisibleView.superview != _contentView) {
    return;  // View was deleted - abort correction
}
```

**When it triggers:**

- Anchor view removed from `_contentView` by a Remove mutation that is not
  followed by a Delete
- `_firstVisibleView.superview != _contentView` → abort

**Why it's needed:** Without this check, MVCP would compute a delta from the
stale view's frame and apply it to `scrollToOffset(0)`, resulting in incorrect
offset (e.g., offset ~3876 instead of 0).

**Two abort conditions compared:**

| Scenario             | Tag changed? | Superview changed? | First check (tag) | Second check (superview) |
| -------------------- | ------------ | ------------------ | ----------------- | ------------------------ |
| Normal prepend       | No           | No                 | False             | False → **proceed**      |
| View recycled        | Yes          | No                 | True → **abort**  | -                        |
| View deleted (reset) | Yes (→ 0)    | Yes                | True → **abort**  | -                        |
| View reparented      | No           | Yes                | False             | True → **abort**         |

Deletion resets the tag to 0, so deleted or recycled anchors are caught by the
tag check. The superview check catches an anchor that was removed from
`_contentView` but not deleted.

### 7.3 Scroll Skip Guards

**Purpose:** Skip MVCP correction during user dragging or momentum scroll to
avoid conflicting with user gestures.

**Current status:**

| Platform   | Scroll Skip Guard                                                                                                                                                             |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| iOS Fabric | **Not present** in MVCP code. `_avoidAdjustmentForMaintainVisibleContentPosition` is driven by the `enableImmediateUpdateModeForContentOffsetChanges` flag, not scroll state. |
| Android    | **Not present**. No scroll skip guard in `updateScrollPositionInternal`.                                                                                                      |

### 7.4 Divide-by-Zero Guard (JS)

**Location:** `ListMetricsAggregator.js`

```js
if (this._measuredCellsCount > 0) {
  this._averageCellLength =
    this._measuredCellsLength / this._measuredCellsCount;
}
```

**Purpose:** Prevents `_averageCellLength` from becoming `Infinity` or `NaN`
when no cells have been measured yet.

**Related fix:** `_invalidateIfOrientationChanged` clears `_cellMetrics` when
orientation changes (horizontal/vertical or RTL), preventing stale metrics from
corrupting new measurements.

### 7.5 Empty List Handling

| Platform   | Behavior                                                 |
| ---------- | -------------------------------------------------------- |
| iOS Fabric | Safe: nil / tag / superview checks abort the correction  |
| Android    | Safe: `firstVisibleViewRef.get() ?: return` early return |

### 7.6 Frame Delta Threshold

| Platform   | Threshold          |
| ---------- | ------------------ |
| iOS Fabric | `ABS(delta) > 0.5` |
| Android    | `delta != 0`       |

**Purpose:** Prevents sub-pixel noise from triggering unnecessary scroll
corrections. The threshold filters out floating-point rounding errors. iOS uses
0.5px while Android uses exact zero comparison.

### 7.7 Autoscroll to Top Threshold

**Prop:** `autoscrollToTopThreshold` (optional, number)

**Behavior:** When a correction is applied and the scroll offset _before_ the
correction was `<= autoscrollToTopThreshold`, the list then animates to the
start, so a user who was at the top sees the newly prepended items.

### 7.8 Scroll Event Throttle (Android)

**The throttle mechanism:**

```kotlin
if (!skipThrottle && scrollEventType == SCROLL &&
    scrollView.scrollEventThrottle >= max(17, now - scrollView.lastScrollDispatchTime)) {
    return  // throttled
}
```

**Purpose:** Limits `onScroll` event frequency to reduce JS bridge traffic
during rapid scrolling. With `scrollEventThrottle = 500`, events are only
dispatched once per 500ms window.

**Problem:** The throttle blocks MVCP-adjusted scroll events, causing JS state
to be stale:

- During scroll animation: events are throttled, JS state doesn't update
- After animation: throttle window hasn't expired, MVCP event is blocked
- Result: JS offset is stale when MVCP computes delta

**Fix:** Added `emitScrollEventNoThrottle()` that bypasses the throttle check,
called when a fling animation finishes (`registerFlingAnimator`, in both
`onAnimationEnd` and `onAnimationCancel`). This ensures JS state is updated
immediately when the animation completes. The MVCP correction itself
(`scrollToPreservingMomentum`) goes through the normal, throttled scroll event
path.

**Why this is correct:**

- Throttle still applies during active scrolling (reduces traffic as intended)
- Unthrottled events only fire after fling animations end or are cancelled
- JS state is current when needed for delta calculations

**Platform difference:** iOS throttles all `scrollViewDidScroll:` events too,
but MVCP calls `_forceDispatchNextScrollEvent` before changing `contentOffset`,
so the adjusted offset is always emitted.

---

## 8. Design Details and Trade-offs

This section documents specific design choices and the trade-offs that shaped
the current implementation.

### 8.1 JS Cell Metrics — Orientation Change Handling

The `_cellMetrics` Map stores per-cell layout info keyed by cell ID. When
orientation changes, the metric coordinate system flips (horizontal ↔ vertical),
making all stored metrics invalid.

**Why the Map must be cleared (not just counters):** Clearing `_cellMetrics` is
necessary because the counters alone don't prevent stale entries from being
found on subsequent `notifyCellLayout` calls. The Map acts as a set of "known
cells" — if not cleared, old entries persist and interfere with new
measurements.

**Why the division is guarded:** The `_averageCellLength` computation uses
`if (count > 0)` rather than relying solely on the orientation change
invalidation. This is defense-in-depth: even if the invalidation is missed or
delayed (e.g., rapid orientation changes), the division won't produce `NaN`.

### 8.2 iOS Fabric — Anchor View Abort Conditions

MVCP uses three abort conditions in `_adjustForMaintainVisibleContentPosition`
to handle views that are no longer valid anchors:

1. **Nil check** (`!_firstVisibleView`): Catches the case where the list was
   empty during mount and no anchor was captured.
2. **Tag check** (`_firstVisibleView.tag != _firstVisibleViewTag`): Detects when
   the anchor view was recycled from the pool and reassigned to a different
   item. `RCTComponentViewRegistry` assigns tags during dequeue and resets to 0
   during enqueue, so a tag mismatch means the view no longer represents the
   same item.
3. **Superview check** (`_firstVisibleView.superview != _contentView`): Detects
   when the anchor view was removed from the scroll view's hierarchy without
   being deleted (a deleted view already fails the tag check).

**Ordering rationale:** The nil check is first (cheapest, catches empty list).
The tag check is second (catches recycling). The superview check is last
(catches deletion). This ordering minimizes unnecessary checks in the common
case (normal prepend where all three pass).

**Why the tag check is always active:** `RCTComponentViewRegistry` assigns tags
during dequeue and resets to 0 during enqueue regardless of culling state. When
items are removed and re-added (even without culling), recycled UIViews receive
the React tag of the new ShadowView they host. The tag check must always run to
avoid applying a delta to the wrong view.

### 8.3 Android — Scroll Event Throttle Design

`scrollEventThrottle` limits `onScroll` event frequency to reduce JS bridge
traffic during active scrolling. This creates a trade-off: MVCP adjustments that
occur during or immediately after a scroll animation may find stale JS offset
state if the throttle blocks the adjustment event.

**Resolution: selective unthrottling.** The `emitScrollEventNoThrottle()`
function bypasses the throttle check but is only called when a fling animation
ends or is cancelled (`registerFlingAnimator`). This preserves the throttle's
purpose (reducing JS bridge traffic during active scrolling) while ensuring JS
state is current once a scroll animation completes. MVCP corrections themselves
are emitted through the normal throttled path.

**Platform difference:** iOS throttles `scrollViewDidScroll:` events as well,
but MVCP calls `_forceDispatchNextScrollEvent` before adjusting `contentOffset`,
so the corrected offset is always dispatched.

---

## 9. Appendix: Key Code References

### iOS Fabric

| File                            | Description                                                                          |
| ------------------------------- | ------------------------------------------------------------------------------------ |
| `RCTScrollViewComponentView.mm` | State variables (\_prevFirstVisibleFrame, \_firstVisibleView, \_firstVisibleViewTag) |
| `RCTScrollViewComponentView.mm` | Mounting transaction callbacks (willMount/didMount)                                  |
| `RCTScrollViewComponentView.mm` | `_prepareForMaintainVisibleScrollPosition` — pre-mount recomputation                 |
| `RCTScrollViewComponentView.mm` | `_adjustForMaintainVisibleContentPosition` — delta computation + correction          |
| `RCTComponentViewRegistry.mm`   | Recycle pool max size constant (1024)                                                |
| `RCTComponentViewRegistry.mm`   | `_dequeueComponentViewWithComponentHandle` — pool dequeue                            |
| `RCTComponentViewRegistry.mm`   | `_enqueueComponentViewWithComponentHandle:componentViewDescriptor:` — pool enqueue   |
| `RCTMountingManager.mm`         | `performTransaction` — three-phase mount lifecycle                                   |

### Android

| File                                     | Description                                                           |
| ---------------------------------------- | --------------------------------------------------------------------- |
| `MaintainVisibleScrollPositionHelper.kt` | Class signature and state variables                                   |
| `MaintainVisibleScrollPositionHelper.kt` | `updateScrollPositionInternal` — correction logic                     |
| `MaintainVisibleScrollPositionHelper.kt` | `computeTargetView` — anchor scan with WeakReference                  |
| `MaintainVisibleScrollPositionHelper.kt` | `willMountItems` / `didMountItems` — UIManagerListener callbacks      |
| `ReactScrollView.kt`                     | `maintainVisibleContentPositionHelper` field                          |
| `ReactScrollView.kt`                     | `setMaintainVisibleContentPosition` — helper creation/update/teardown |
| `ReactViewGroup.kt`                      | Culling state (\_removeClippedSubviews, allChildren, clippingRect)    |
| `ReactViewGroup.kt`                      | `updateClippingToRect` — culling implementation                       |

### JS / VirtualizedLists

| File                       | Description                                                                |
| -------------------------- | -------------------------------------------------------------------------- |
| `VirtualizedList.js`       | State shape (renderMask, cellsAroundViewport, pendingScrollUpdateCount)    |
| `VirtualizedList.js`       | `getDerivedStateFromProps` — MVCP prepend detection                        |
| `VirtualizedList.js`       | `pendingScrollUpdateCount` increment on prepend                            |
| `VirtualizedList.js`       | `_adjustCellsAroundViewport` — guard when pendingScrollUpdateCount > 0     |
| `VirtualizedList.js`       | `_onScroll` — pendingScrollUpdateCount decrement                           |
| `ListMetricsAggregator.js` | State variables (\_averageCellLength, \_cellMetrics, \_measuredCellsCount) |
| `ListMetricsAggregator.js` | `notifyCellLayout` — cell measurement tracking                             |
| `ListMetricsAggregator.js` | Divide-by-zero guard                                                       |
| `ListMetricsAggregator.js` | `_invalidateIfOrientationChanged` — metrics clear on orientation change    |
| `ScrollView.js`            | MVCP prop type definition                                                  |
| `ScrollView.js`            | `preserveChildren` logic — collapsableChildren when MVCP active            |
