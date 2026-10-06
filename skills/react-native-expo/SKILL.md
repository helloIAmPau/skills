---
name: react-native-expo
description: Develop Hybrid's React Native and Expo mobile workspace, including native styling, Metro startup, development builds, and native E2E tests. Use for the mobile app; backend Express services retain their separate workspace and Docker conventions.
---

# React Native and Expo

Follow the owner-approved scaffold in [issue #5](https://github.com/helloIAmPau/hybrid/issues/5). The app lives in `workspaces/@hybrid/mobile`, named `@hybrid/mobile`. Install from the repository root with one npm lockfile and Node 24/npm 11. Mobile workspaces may use `dependencies`: declare Expo, React Native, React, and the development client there. Reserve `devDependencies` for build/test-only tooling; backend workspaces keep their separate convention.

## Application structure

- Workspace-root `index.js` imports `expo-router/entry`. Keep thin default route exports in `app/`, with `_layout.js` exporting the Navigation component and `index.js` exporting App. Navigation wraps the router stack in the shared font/splash Layout; screen components remain under `components/`. Keep route-specific presentation out of the root layout.
- Use JavaScript and JSX following [javascript](../javascript/SKILL.md) for functions, control flow, formatting, and components. This does not choose a language for future unrelated workspaces.
- Use native `View`, `Text`, and other React Native components, not DOM elements or CSS modules. Declare `StyleSheet.create(...)` above the component that uses it, as explicitly requested by the owner.
- Keep native colors, typography, spacing, radii and font definitions in workspace-root `theme.js`, not a `design/` directory. Install the needed `@expo-google-fonts/*` families in mobile runtime dependencies, and register them through `expo-font` before rendering themed components. Do not maintain mobile `assets/fonts/` copies; the separate HTML blueprint assets remain under `docs/`.
- The used `Layout` component owns safe-area/background styling and startup. Hold the native splash with `expo-splash-screen` before mounting; return `null` while fonts load and hide the splash in `useLayoutEffect` when fonts are loaded, following Expo’s font-loading lifecycle rather than an `onLayout` callback. Keep Layout free of custom font-error handling, as requested by the owner. Introduce other components when they are connected to a feature.
- Do not define inline functions or rendering logic in JSX. Define handlers with `useCallback`; use `useMemo` to build conditional elements, mapped children and derived values, then insert the resulting variables into the JSX body. Keep conditions, formatting calls and list transformations above the JSX they supply. Always use `useLayoutEffect` for effects, never `useEffect`.
- Every native component must perform exactly one scope of responsibility and be as minimal as possible, following [JavaScript's mandatory component rule](../javascript/SKILL.md#one-scope-per-component-always). Keep only the props, state, handlers, effects, markup and styles needed for that scope. Split independent presentation concerns into focused child components, compose them in the screen and put separate data/lifecycle orchestration in focused hooks. Extract these units even with only one caller; create only units that are actually used. During implementation and review, split components that mix responsibilities before considering the work complete.
- Implement only approved screens and product operations. The day screen uses real GraphQL reads through Caddy; later writers extend its persisted sections. Keep authentication and navigation within their own approved scope.
- `app.json` owns Expo configuration. `io.helloiampau.hybrid` is the development app identifier for Android and iOS; `hybrid` is its launch scheme. These are not store/signing decisions.
- Use the SDK-compatible Expo/React/React Native versions. Run `expo install --check` and Expo Doctor when changing dependencies. Do not force a dependency audit downgrade across SDK versions.

## Native builds and development

- Expo/Metro bundles the app. Do not apply the Express CommonJS build contract, port-80 listener, or backend Dockerfile to the mobile workspace. Its `export` script validates Android/iOS JS bundles; it does not build or install a native app.
- Use Expo's automatic monorepo configuration. Add Metro overrides only for a demonstrated requirement, not as scaffolding defaults.
- Keep generated `android/`, `ios/`, `.expo/`, and `dist/` outputs ignored. Expo generates native projects from configuration; rebuild the development client after changing native dependencies or app configuration.
- Root `npm run develop` starts the full Compose development stack, including the default `mobile-tools` service in `docker-compose.develop.yml`. It boots a headless Android emulator, starts Metro, builds/installs the app, and opens the current bundle. JavaScript/styles use Fast Refresh; changes to `app.json`, root/workspace manifests, or the lockfile trigger native regeneration and reinstall. Wait for `Mobile ready` before testing. Compose stops companion containers on exit.
- Use the separately maintained tooling image and its default startup command as defined below. Keep backend Docker stages on Alpine; native tooling uses Debian for Android's glibc requirement.
- The default Android build/install is automatic. The workspace `android` script remains available for manual native builds. iOS still requires macOS/Xcode, host Metro (`npm run develop --workspace=@hybrid/mobile`), and `npm run ios --workspace=@hybrid/mobile`; the Linux container does not build iOS. Native scripts use `--no-bundler` because Metro is managed separately.
- Run Expo native generation directly with `expo prebuild --platform <android|ios> --clean --no-install` before the native build. The owner allows mobile runtime `dependencies`, so do not maintain a wrapper that restores the manifest after prebuild. Preserve the single root lockfile.
- A device's localhost is not necessarily the development host. For a locally attached Android device, `adb reverse tcp:8081 tcp:8081` makes Metro reachable at device localhost. Otherwise use an address reachable from both the device and test runner. Set `REACT_NATIVE_PACKAGER_HOSTNAME` before startup when Expo's advertised address needs to differ from its automatic selection.
- A Docker-socket development shell must resolve bind mounts to paths on the Docker host. Use a checkout alias matching the host path where needed; do not replace repository mount paths with machine-specific paths.
- Do not introduce EAS accounts, cloud builds, signing, store publishing, or web hosting as implicit requirements.

## Shared mobile tooling image

- Maintain the Dockerfile and generic `develop.sh` in [helloIAmPau/mobile-tools](https://github.com/helloIAmPau/mobile-tools). The image installs the script as `mobile-develop` and uses it as its default command. Do not keep another mobile Dockerfile or lifecycle script in Hybrid, or override the image command with an application-local script.
- Keep the image application-independent: Node 24/npm 11, Java 17, Android SDK/emulator and Maestro, with no bundled application source or application npm dependencies. Install consuming-project dependencies at container startup. Keep Hybrid package names, paths, identifiers and scheme in the consuming repository.
- Require `MOBILE_WORKSPACE` as a workspace path relative to `/source`; Hybrid supplies `workspaces/@hybrid/mobile`. Discover dependency manifests from the root package.json workspace patterns rather than hardcoding a scope or directory glob. Read `expo.android.package` and the string `expo.scheme` from the selected workspace's static `app.json`; dynamic Expo configuration is outside the current startup contract.
- The selected workspace supplies `develop` and `android` npm scripts. The image-owned `mobile-environment` command resolves the configured hosts for the Android emulator before launching Metro; host development supplies a device-reachable public host. Configure `EXPO_PUBLIC_HYBRID_HOST` and `MOBILE_RESOLVE_URLS` in `.env`/`.env.develop`, with safe examples in `.env.example`, and pass them through Compose interpolation. Keep only the host (scheme included, no trailing slash); the client uses `process.env.EXPO_PUBLIC_HYBRID_HOST + '/graphql'`. Do not keep a mobile `develop.js` wrapper. The workspace `develop` script calls `expo start --dev-client --port 8081` directly; the image adds `--localhost`. `android` generates, builds and installs the app with `--no-bundler`. Keep these application scripts in the workspace while the image manages their lifecycle.
- Keep `mobile-tools` a default service in `docker-compose.develop.yml`, with `platform: linux/amd64`, `/dev/kvm:/dev/kvm`, `init: true`, and a 30-second stop grace period. The init process forwards signals and reaps orphaned children; the lifecycle script stops its build, Metro and emulator process groups. Do not mount the Docker socket or use host networking.
- Metro and the emulator run inside this container, with `adb reverse tcp:8081 tcp:8081`. Set `REACT_NATIVE_PACKAGER_HOSTNAME=localhost`, `EXPO_METRO_URL=http://localhost:8081`, `EXPO_NO_TELEMETRY=1` and `EXPO_NO_DEV_MENU=1` for Hybrid's default workflow. Join `frontend` for gateway access and `backend` for the E2E runner’s database fixtures. Development tooling publishes only `127.0.0.1:5038:5037` for ADB and `127.0.0.1:27183:27183` for scrcpy; Caddy owns the HTTP ports. `scrcpy.sh` runs on the Linux x86_64 Docker host, downloads the pinned release when `data/scrcpy/` is absent, and runs it against localhost; the container ADB server must start with `-a`. ClickHouse credentials remain tooling-only; native requests always use Caddy.

Use these consumer bind mounts; do not add Docker-managed dependency volumes:

| Host source | Container destination | Purpose |
| --- | --- | --- |
| `./workspaces` | `/source/workspaces` | Writable application source and generated native projects |
| `.` | `/project:ro` | Checkout directory containing root manifests |
| `./tests` | `/source/tests:ro` | Flat E2E suite |
| `./data/mobile-tools` | `/state` | Home, device state and Gradle caches |

The script links `/source/package.json` and `/source/package-lock.json` from `/project`. Mount the directory, not individual root manifest files, so atomic replacements remain visible. Container-installed node_modules stay isolated from the host. Other consumers mount their workspace directories at matching paths under `/source`.

- Run `npm ci` from `/source` on initial installation and when root/workspace manifests or the root lockfile change. It installs the locked dependency tree; use npm install when intentionally changing dependencies and updating the lockfile, not as a startup substitute.
- Source/style edits use Metro Fast Refresh. Changes to static app configuration rebuild/reinstall the native client. Manifest or lockfile changes also stop Metro, reinstall dependencies and restart Metro before rebuilding the client. Watch all discovered workspace manifests, including sibling workspaces and atomic replacements.
- Clear `/state/mobile-ready` during rebuilds and shutdown. Successful installation and launch recreate it and log `Mobile ready`. Native build failures leave readiness absent; corrections to watched configuration/manifests trigger retry. Confirm the container is running, Metro responds and the app is ready before testing; startup ordering alone is insufficient.

## Tooling image verification and publication

- Run the tooling repository's complete `npm test`: verify missing workspace configuration fails, then compile/install a real Android APK and check launch/relaunch through Maestro. Test fixtures belong only in its test image stage; publish the runtime stage.
- For lifecycle changes, also run Hybrid's entire development stack and complete E2E suite. The tooling SDK fixture alone does not prove Expo startup, source refresh, configuration rebuilds or manifest-triggered reinstallation.
- Local candidate images may be used while implementing and reviewing a tooling change. Record the candidate clearly; passing tests against it do not prove a published GHCR image works. Keep task status and test evidence on GitHub, not in the skills.
- Follow the owner approval workflow before publishing new feature commits/PRs. The tooling repository's GitHub Actions tests pull requests; its main workflow publishes `ghcr.io/helloiampau/mobile-tools` with a full-commit `sha-<commit>` tag and `latest`, using the scoped `GITHUB_TOKEN` with package write permission. Never put registry credentials into the image or source.
- Once published, pin the actual GHCR digest in Hybrid, pull that image, start the full stack with `npm run develop`, confirm readiness and run complete `npm test` again. Verify registry access for consumers; a local tag or cached image is not evidence of a successful registry pull. Tooling publication does not itself authorize committing or opening Hybrid's feature PR.

## E2E tests

- Follow [github-feature-workflow](../github-feature-workflow/SKILL.md): start the entire development stack before every test run and confirm readiness and completed rebuilds.
- Keep all test files directly in `tests/e2e/`. Do not create feature/platform subdirectories.
- Root `npm test` first verifies the migration runner from the host, runs the headless scrcpy recording/cache E2E check from the Linux x86_64 Docker host, then executes every Node E2E file sequentially inside the running Compose `mobile-tools` container. Always run the complete suite with `npm test`; do not add feature-specific test commands. The GraphQL test uses `HYBRID_GATEWAY_URL=http://caddy` as its network destination and the configured `HYBRID_HOST` as its Host header, preserving real gateway routing.
- Select native elements by visible text or meaningful accessibility labels, using relational selectors for repeated controls when needed. Do not add `testID` props, test-only accessibility labels, or production branches solely to support selectors. Accessibility labels must describe the control for screen-reader users.
- Native tests invoke the installed Maestro CLI against a real emulator/device and the installed `io.helloiampau.hybrid` development app. Java 17, Android SDK, and Maestro are installed by the development tooling image, not npm packages. Missing tools or devices must fail, not skip.
- `EXPO_METRO_URL` defaults to `http://localhost:8081`. It must be reachable from both the runner and device. The native wrapper checks Metro availability and passes an explicit development-client URL to Maestro, so the app loads the current server's bundle.
- SDK 57 uses `hybrid://expo-development-client/?url=<encoded Metro URL>`; the test URL disables development-client onboarding and menu overlays. Recheck Expo's documented launch URL when upgrading SDKs.
- `mobile-launch.yaml` verifies the Hybrid title on launch and after stopping and relaunching the app. `mobile-refresh.yaml` observes source edits reaching the already-running native screen. The native configuration test changes Android versionCode and verifies the installed package updates automatically. The manifest test atomically replaces a sibling workspace manifest and checks native reinstallation and the resulting screen, validating the image-owned manifest discovery. Tests restore edited source/configuration in `finally`; avoid concurrent editing during this suite. Assert native UI and installed-app outcomes without mocked transports or application-internal imports.
- `mobile-health.jsx` and `mobile-health.yaml` exercise a real React Native fetch to `/graphql/health` through Caddy. The runner resolves Docker DNS to Caddy's IPv4 address; the app uses normal URL-derived headers against the development HTTP catch-all. Maestro checks HTTP 200, the health status and Caddy's Via header. Load the probe temporarily through Metro, retain the router root Layout during the home-screen swap so native safe-area modules remain registered, and restore the original app in `finally`. Text selectors must distinguish the actual app screen from the development-client launcher. This verifies local HTTP emulator connectivity, not a product GraphQL client or native HTTPS trust.
- Report native device/platform evidence separately from JS bundle exports. Android is the initial required native E2E target. iOS verification requires a macOS/Xcode simulator; do not imply it passed from Android or export results.

## References

- [Expo monorepos](https://docs.expo.dev/guides/monorepos/)
- [Expo development builds](https://docs.expo.dev/develop/development-builds/use-development-builds/)
- [Development-client launch URLs](https://docs.expo.dev/develop/development-builds/development-workflows/)
- [Maestro CLI installation](https://docs.maestro.dev/maestro-cli/how-to-install-maestro-cli)

Apply [workspace-ownership](../workspace-ownership/SKILL.md) after authored file changes and leave implementation uncommitted until publication is approved.
