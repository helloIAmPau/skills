---
name: react-native-expo
description: Develop selected React Native and Expo applications with native styling, font startup, containerized Android development tooling and host-native E2E drivers. Backend bundles retain their separate conventions.
---

# React Native and Expo

Apply when a native application is selected; do not add a mobile workspace to an
unrelated repository. Resolve workspace paths, package names and toolchain from
its configuration. In the npm workspace architecture, install from the effective
root with one lockfile. Mobile packages use runtime `dependencies` for Expo,
React Native, React and the development client; `devDependencies` hold build/test
utilities. Backend bundles keep their separate dependency convention.

## Application structure

- Follow [javascript](../javascript/SKILL.md) and [react](../react/SKILL.md) for
  language, JSX, state, callbacks and the responsibility-based extraction rule.
  Every component has one minimal responsibility, even with a single caller;
  separate presentation from lifecycle/data orchestration in focused hooks.
- Workspace-root `index.js` imports `expo-router/entry`. Keep thin default route
  exports in `app/`; `_layout.js` composes navigation/startup, while actual screen
  components live under `components/`. Keep route presentation out of the root layout.
- Use native `View`, `Text` and other native elements, not DOM or CSS modules.
  Declare each component's `StyleSheet.create(...)` above that component.
- Keep shared native colors, typography, spacing, radii and font definitions in
  workspace-root `theme.js`. Install needed `@expo-google-fonts/*` families in
  runtime dependencies and register them through `expo-font`; do not keep copied
  font binaries in a mobile `assets/fonts/` directory.
- Compute conditional children, transformations and derived values with `useMemo`
  above JSX. Use `useCallback` handlers; no inline functions/rendering logic in JSX.
  Always use `useLayoutEffect` for effects, rather than `useEffect`.
- `app.json` owns static Expo configuration. Derive app identifiers, schemes,
  screen titles and routes from it and the application; these do not authorize
  store/signing setup. Implement only requested screens and operations.
- Use SDK-compatible Expo/React/React Native versions. Run `expo install --check`
  and Expo Doctor when changing dependencies; do not force cross-SDK audit downgrades.

## Font startup

Hold the native splash with `expo-splash-screen` before mounting. Keep safe-area/
background layout, error presentation and font/splash orchestration focused.
Consume both results from `useFonts`: a caught loading failure may not throw to
an error boundary. The startup contract is:

| State | Result |
| --- | --- |
| Pending | Keep the splash/loading behavior |
| Loaded | Hide the splash and render the normal application |
| Failed | Hide the splash and render the application's error view |

The error view must use system/default fonts and be outside any font-success-only
layout. Never wait for `loaded === true` after failure or silently render the
normal app instead. Use `useLayoutEffect` for splash release, not an `onLayout`
callback. For example, a focused startup hook with configured `fontDefinitions`:

```js
export function useFontStartup() {
  const [ loaded, error ] = useFonts(fontDefinitions);

  useLayoutEffect(function() {
    if (error == null && loaded === false) {
      return;
    }

    SplashScreen.hideAsync();
  }, [ loaded, error ]);

  return { loaded, error };
}
```

A focused gate composes the application's existing error view and children:

```js
export function FontStartup({ children }) {
  const { loaded, error } = useFontStartup();

  if (error != null) {
    return <StartupError message={ error.message }/>;
  }

  if (loaded === false) {
    return null;
  }

  return children;
}
```

`StartupError` is a native presentation component whose styles omit custom
`fontFamily`; place themed success-only layouts inside the gate's children.
Simulate loader failure and verify a visible error with a released splash.

## Native builds and development

- Expo/Metro bundles the app. Backend CommonJS output, port-80 listeners and the
  backend Dockerfile do not apply. JS bundle exports do not build/install a native app.
- Use Expo's automatic monorepo configuration; add Metro overrides only for an
  established need. Ignore generated `android/`, `ios/`, `.expo/` and `dist/`.
- When containerized Android is selected, root `npm run develop` includes emulator
  and Metro tooling in the development Compose override. Keep them in containers
  while running test drivers on the host. Follow the shared
  [post-mount install barrier](../npm-workspace-services/SKILL.md#post-mount-installation-barrier):
  mounts, successful installation, native preparation/build, Metro/watchers,
  installed app launch, then actual device/application readiness.
- Run `expo prebuild --platform <android|ios> --clean --no-install` before native
  builds, after dependencies are installed. Preserve mobile runtime dependencies
  and the root lockfile without a manifest-restoring wrapper. Native scripts use
  `--no-bundler` when Metro is separately managed.
- Source/styles use Fast Refresh. Static app configuration changes rebuild/reinstall
  the native client. Manifest/lockfile changes stop affected processes, complete
  serialized installation, then rebuild/restart; watch all discovered workspace
  manifests, including sibling manifests and atomic replacements.
- Clear readiness during rebuild/shutdown and leave it absent on installation,
  build or launch failure. Check emulator/device, Metro and app readiness before
  tests; a container starting or a log line alone is insufficient.
- iOS builds require macOS/Xcode, compatible host dependencies and separately
  managed Metro. Android or JS export success does not establish iOS verification.
- Docker-socket development shells must resolve source mounts to Docker-host paths.
  Do not bake machine-specific checkout aliases into reusable configuration.
- Do not introduce EAS accounts, cloud builds, signing or store publication implicitly.

## Shared mobile tooling

Use a separately maintained generic tooling image when that architecture is
selected, pinned by published digest, for example
`<registry>/<namespace>/<tooling-image>@sha256:<digest>`. Its Dockerfile and lifecycle
script live in the tooling project; avoid duplicate application-local lifecycle
implementations. Resolve Node/npm, Java and Android SDK versions from the selected
Expo/Gradle toolchain. Use a glibc-compatible native image, such as Debian.

- Keep application source/dependencies out of the generic image. It installs
  consuming dependencies after mounts using the shared barrier.
- Configure a workspace path relative to the effective root, illustrated as
  `MOBILE_WORKSPACE=workspaces/@scope/mobile`. Discover manifests from the root
  workspace patterns. Read identifiers and launch scheme from static `app.json`;
  dynamic configuration needs an explicitly selected resolution contract.
- Application scripts own `develop` (direct `expo start --dev-client --port 8081`)
  and `android` (generate/build/install with separately managed Metro). The tooling
  lifecycle coordinates them. Derive public API settings from consuming environment
  files, illustrated as `EXPO_PUBLIC_API_URL`; never put server credentials in bundles.
- Keep native tooling in the development override, with compatible platform/KVM,
  `init: true` and a sufficient stop grace period. Forward signals and stop child
  process groups. Do not mount the Docker socket or use host networking.
- Retain writable workspace sources at matching paths under `/source`. A read-only
  checkout directory such as `.:/project:ro` can supply root manifests via links;
  directory mounts preserve atomic replacements. Bind persistent device/home/Gradle
  caches under `./data/<tooling-service>/`. No suite mount or database network/
  credentials is required merely because host tests exercise the mobile app.
- Root-private dependencies and workspace-local dependencies have different mount
  visibility. Writable workspace `node_modules` may reside in the host checkout;
  coordinate installs and platform/ABI compatibility with host and backend runtimes.

## Host ADB and public addressing

Follow the shared
[host/public-interface contract](../npm-workspace-services/SKILL.md#host-testing-and-public-interfaces).
Publish the emulator container's ADB server to the host, for example
`127.0.0.1:5038:5037`, and start it listening on an interface that publication can
reach (`adb -a` server mode). Choose actual ports from configuration.

Configure host ADB, Maestro and scrcpy consistently for that endpoint; do not let
one tool use an unrelated default host ADB server. For compatible tools, an
endpoint such as `ADB_SERVER_SOCKET=tcp:127.0.0.1:5038` can select it; verify each
tool's supported endpoint option and confirm it sees the intended device. Install
ADB, Maestro and its Java prerequisite on the host; install scrcpy/recording tools
there when recording is required. Image-installed binaries do not satisfy this.
Publish additional recording/tooling endpoints only when the selected setup needs them.

Document the host gateway URL, host Metro endpoint and device URLs separately.
Publish Metro to the host where the driver needs it; configure a reachable advertised
address with `REACT_NATIVE_PACKAGER_HOSTNAME` when needed. A reverse tunnel such as
`adb reverse tcp:8081 tcp:8081` connects device localhost to the selected ADB server's
network context; when that server is inside the emulator container, ensure Metro
is reachable there. It does not inherently connect device localhost to the host.

Use public gateway addresses that the host and device can each resolve, with
routing configured for those actual URLs. Do not discover gateway container IPs
through Docker DNS or manually set Host headers. If host/device Metro URLs differ,
configure both explicitly instead of assuming one `localhost` URL works everywhere.
Read the launch scheme from configuration and use Expo's SDK-compatible documented
development-client URL with the device-reachable Metro address.

## Verification and publication

- Follow the shared [E2E lifecycle](../npm-workspace-services/SKILL.md#e2e-run-lifecycle).
  Root `npm test`, Node tests, Maestro, recording and migration verification drivers
  all run on the host. Keep tests flat under `tests/e2e/` and run the complete suite.
- Select elements by visible text or meaningful accessibility labels; use relational
  selectors for repeats. Do not add `testID`, test-only labels or production branches
  just for selectors. Labels must help screen-reader users.
- Exercise a real installed app/device and public API/UI. Cover launch/relaunch,
  source refresh, configuration rebuilds and atomic manifest changes where relevant;
  restore temporary edits in `finally`. Avoid simultaneous edits during that suite.
  Native HTTP health checks use the documented device-reachable gateway URL.
- Missing host tools/devices/services fail or block. Report native platform evidence
  separately from JS exports, and distinguish candidate images from published images.
- Tooling changes require the tooling project's complete host-run suite and relevant
  consuming-stack verification; an SDK fixture alone does not prove Expo integration.
  Keep fixture apps in test-only image stages and publish the runtime stage.
- Honor existing publication authorization and the applicable
  [GitHub workflow](../github-feature-workflow/SKILL.md). Resolve registry/CI settings
  from the tooling project. Never put registry credentials in image/source; use
  scoped CI credentials such as `GITHUB_TOKEN` where configured.
- After authorized image publication, pin its actual digest, verify a registry pull,
  then verify the consuming stack again. Cached/local image success is not registry
  evidence. Tooling publication does not authorize a consuming-project commit/PR.

Apply [workspace-ownership](../workspace-ownership/SKILL.md) to authored changes
and touched metadata before handoff, using owner UID `1000`.

## References

- [Expo monorepos](https://docs.expo.dev/guides/monorepos/)
- [Expo development builds](https://docs.expo.dev/develop/development-builds/use-development-builds/)
- [Development-client launch URLs](https://docs.expo.dev/develop/development-builds/development-workflows/)
- [Maestro CLI installation](https://docs.maestro.dev/maestro-cli/how-to-install-maestro-cli)
