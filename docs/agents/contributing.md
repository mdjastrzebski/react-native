<!-- index-codebase: generated from 024b474ce92 on 2026-10-03. Code wins when it disagrees with this doc. -->

# Lists and Virtualization: Contributing

Repo-wide setup, commands, and PR rules are in [`AGENTS.md`](../../AGENTS.md) and on [reactnative.dev/contributing](https://reactnative.dev/contributing/overview). This page covers only what is specific to the list stack.

## Setup

JS-only work, which is most list work, needs only Node and Yarn: run `yarn install` from the root. Fantom tests and RNTester also need native toolchains (see `AGENTS.md`). Metro bundling of RNTester requires `yarn --cwd packages/react-native-codegen build` once.

## Feedback loop

| To check… | Run | Notes |
| --- | --- | --- |
| VirtualizedList core, VirtualizedSectionList, ViewabilityHelper, FillRateHelper, metrics, render mask | `yarn test packages/virtualized-lists` | Jest + `react-test-renderer` + fake timers. At 024b474ce92: 185 passed, 1 skipped, 69 snapshots, about 5 s. CI job `test_js`. |
| One suite | `yarn test packages/virtualized-lists/Lists/__tests__/VirtualizeUtils-test.js` | Use `-u` to update snapshots; never hand-edit `.snap` |
| FlatList / SectionList integration | `yarn fantom packages/react-native/Libraries/Lists/__tests__/FlatList-itest.js` (or `SectionList-itest.js`) | Builds a native tester on first run. Not run for this doc. |
| Sticky-header perf | `yarn fantom packages/react-native/Libraries/Lists/__tests__/VirtualizedList-stickyHeaders-benchmark-itest.js` | CI runs it once as a smoke test |
| MVCP (JS + Fabric, no real native scroll) | `yarn fantom packages/react-native/Libraries/Components/ScrollView/__tests__/ScrollView-maintainVisibleContentPosition-itest.js` | Real native behavior is covered only by Maestro |
| VirtualView JS state machine | `yarn fantom packages/react-native/src/private/components/virtualview/__tests__/VirtualView-itest.js` | Dispatches `onModeChange` by hand, so the native container logic is **not** tested. There are no native unit tests and no VirtualCollection tests. |
| Flow | `yarn flow-check` | Includes `Libraries/Lists/__flowtests__/` |
| Public types | `yarn build-types`, then `yarn test-generated-typescript`; legacy: `yarn test-typescript-legacy` | `build-types` regenerates `packages/virtualized-lists/types_generated/` and `packages/react-native/ReactNativeApi.d.ts` |
| Feature flags | `yarn featureflags` after editing `ReactNativeFeatureFlags.config.js` | Regenerates the JS, Kotlin, and C++ accessors |
| Native API snapshots (VirtualView) | `yarn cxx-api-build`; Android `ReactAndroid/api/ReactAndroid.api` | VirtualView ShadowNode, ComponentDescriptor, and Android classes are in the snapshots |
| End-to-end | `yarn --cwd packages/rn-tester e2e-test-android` / `e2e-test-ios`, or `maestro test packages/rn-tester/.maestro/flatlist.yml -e APP_ID=…` | Flows: `flatlist.yml`, `flatlist-viewability.yml`, `sectionlist-viewability.yml`, 24 × `flatlist-*-maintainvisible.yml` |
| Manual | `yarn start` + RNTester → FlatList / SectionList | VirtualView has no RNTester example |

### How CI runs these

- **Jest:** the `test_js` job in `.github/workflows/test-all.yml`.
- **Fantom:** `build_fantom_runner` builds the runner, then `run_fantom_tests` runs `.github/actions/run-fantom-tests`, which executes the whole `yarn fantom` suite. It retries up to 3 times inside the action, and the workflow adds 2 more retry jobs.
- **Maestro:**
  - iOS: `e2e-ios-rntester.yml` runs every flow in sorted order, with no tag filtering.
  - Android: `e2e-android-rntester.yml`. The debug flavor excludes the `android-release-only` tag, so the MVCP flows run only against the release APK because they are timing-sensitive.
  - Maestro Cloud (`maestro-cloud-rntester.yml`) also runs on pushes to main and on same-repo PRs.

## Rules

- Change the Flow source and regenerate. Never hand-edit `types_generated/`, `ReactNativeApi.d.ts`, feature-flag accessors, codegen output, or `.snap` files.
- When a list prop changes, update by hand: `packages/virtualized-lists/Lists/VirtualizedList.d.ts` / `index.d.ts` and `packages/react-native/types_DEPRECATED/Libraries/Lists/*.d.ts`. These legacy types are served under the `react-native-legacy-deep-imports` condition.
- When a VirtualView prop or event changes, keep both specs identical (`VirtualViewNativeComponent.js`, `VirtualViewExperimentalNativeComponent.js`), and update the hand-written iOS sync payload (`_dispatchSyncModeChange`) and the Android `VirtualViewModeChangeEvent.getEventData`.
- When an MVCP behavior changes, update [`packages/virtualized-lists/__docs__/DESIGN.md`](../../packages/virtualized-lists/__docs__/DESIGN.md) as well.
- Code in `packages/virtualized-lists` must import from `react-native` only through public entry points. Feature flags go through `react-native/react-private-interface` (a506ed66cc5).
- PR template, changelog tags, and test plan are covered in [`AGENTS.md`](../../AGENTS.md#contributing-guidelines). List fixes usually take `[GENERAL] [FIXED]`.

## Release process

`@react-native/virtualized-lists` is versioned in lockstep with `react-native`. On `main` it is `0.87.0-main`, and `react-native` pins the exact version. It is published by the monorepo release tooling (`scripts/releases/`), together with the other public packages. `CHANGELOG.md` is compiled at release time from PR descriptions. The `unstable_*` VirtualView and VirtualCollection exports have no stability guarantee and are not part of the TypeScript API snapshot.
