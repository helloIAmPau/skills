---
name: react-native
description: How the native app is built, run and tested in these repositories — an Expo app with Expo Router in the `app` workspace, run in Expo Go on the phone plugged in over USB, or on an emulator when none is. Covers the pinned versions, why a route is a default export, why a StyleSheet sits in the component's own file, how the device reaches Metro and the API over `adb reverse`, the one `android` service that owns adb, Metro and the emulator, that the phone is a person's own, and the device e2e test. Load before scaffolding or changing the app, adding a screen or a route, styling a native component, running the app on a device, touching the android service, or writing or running a test of the app.
---

# React Native

The phone app. **Everything in the `javascript` skill holds here unchanged** —
no arrows, no `else`, no ternary, no `!`, a component taking one destructured
object, nothing decided inside JSX, every handler and held child memoised, the
comment density. What follows is only what a native app changes, and how it is
run and tested.

Read out of Hybrid's app scaffold (helloIAmPau/hybrid#10). Where a project's
AGENTS.md says otherwise, it is right.

## The stack

- **Expo, with Expo Router, run in Expo Go** during development. A development
  build comes only when a native module Expo Go lacks is needed: until then Expo
  Go is the whole native side, and there is no Android project to keep.
- **Pinned exactly, to the SDK's own versions.** Expo's template writes `~`
  ranges; the repositories pin exact versions (`.npmrc` has `save-exact`). Read
  the version for each module out of `expo/bundledNativeModules.json` of the
  pinned `expo`, and take the newest patch inside that range. The SDK decides
  React Native's version — SDK 57 is 0.86.3 — not the latest React Native.
- **Expo Go on the device must serve the app's SDK.** The Play Store's serves
  only the newest; `api.expo.dev/v2/versions/latest` names the APK for each
  SDK under `sdkVersions["<n>.0.0"].androidClientUrl`.
- JavaScript, not TypeScript, like every other workspace. Only the modules the
  scaffold uses: `expo`, `expo-router`, `expo-constants`, `expo-linking`,
  `expo-status-bar`, `react`, `react-native`,
  `react-native-safe-area-context`, `react-native-screens`.

## The workspace

**The app is `workspaces/@<project>/app`, a workspace and not a service.** No
image, no ingress route, nothing in the root Dockerfile. Its `main` is
`expo-router/entry`, its `develop` is `expo start --localhost`, and it has no
`build`: Metro bundles on request.

**The root Dockerfile installs only the service's own workspace** —
`npm install --workspace=@<project>/${SERVICE}` — which still brings in the
libraries the service links to. A plain `npm install` at the root would put the
app's Expo and React Native, about 450MB, into every service image.

## Routes

- **A screen is a file under `app/`**, and `app/_layout.js` is the one frame
  around them all. A `Stack` with the header off until a design says otherwise:
  it is the shape that commits to the least.
- **A route is `export default function`**, where everything else in these
  repositories is a named export. Expo Router takes a route file's default
  export as its component, so the name is not ours to choose — and each route
  file says so in a comment, or the next reader "fixes" it.

## Styles

**A `StyleSheet.create` in the component's own file, just above the
component.** No styles file beside it and no CSS module; an inline style in
the JSX is for a computed value only, as on the web:

```js
import { StyleSheet, Text, View } from 'react-native';

const styles = StyleSheet.create({
  screen: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center'
  }
});

export default function Home() {
  return (
    <View style={ styles.screen }>
      <Text>Hybrid</Text>
    </View>
  );
}
```

The `javascript` skill's CSS module beside each component is the web's rule:
there is no CSS on a phone. The file then reads top to bottom as imports,
constants, styles, component — everything the component needs is above it.

## The API

**Only through the ingress, at the device's own `localhost`.** `adb reverse`
opens the device's localhost on the machine running the android service, so
the app never needs an address on the LAN, or the LAN at all:

- `adb reverse tcp:8081 tcp:8081` — Metro.
- `adb reverse tcp:8080 tcp:80` — the ingress. **8080 on the device, not 80**:
  a port under 1024 is privileged there, and the emulator refuses to open one
  where a Pixel does not mind. Using 8080 everywhere keeps the two alike.

The base URL is **one constant**, `http://localhost:8080/graphql`, so that a
deployed API and a build that talks to it make it configuration in one place.

A fetch that is refused turns into the sentence on screen: a non-2xx from the
ingress (Caddy answers 502 while the service is down) is thrown as an `Error`
naming the status, before anything is parsed as JSON — otherwise the screen
shows a parser's complaint instead of what happened.

## Running it on a device

**One `android` service in the develop compose file holds everything
Android**: the only adb server, Metro, and an emulator booted on demand.

- **Built on `budtmo/docker-android:emulator_<version>` for its Android SDK
  only**: emulator, system image, adb, a node for Metro. Its own launcher is
  not used — on its way up it runs `sed -i '1d' /etc/passwd` and chowns
  `/dev/kvm`. The inline Dockerfile is the base, `user root`, `entrypoint []`,
  `shell [ "bash", "-c" ]` and the start command; nothing of the repository is
  copied in. The repository is bind-mounted at `/source`, so Metro runs on the
  host's `node_modules`.
- **Root, for the USB nodes**, which belong to root on the host.
  `/dev/bus/usb` is mounted as a directory with `device_cgroup_rules:
  'c 189:* rmw'`, so a replugged phone's new node is covered — and
  **`ADB_LIBUSB=0`**, because libusb learns of a new device from udev and a
  container hears no udev events: without it a replugged phone stays unseen
  until the container restarts. The native backend scans `/dev/bus/usb`
  itself.
- **One adb server, and it owns the phone.** Everything that talks to a device
  — the script, the test, a person, the agent — asks this server:
  `ADB_SERVER_SOCKET=tcp:172.17.0.1:5037`, or `docker compose exec android
  adb`. A second server on the host fights it for the phone.
- **A host with IPv6 off in its kernel** makes `adb -a` die binding `[::]`, and
  this adb takes no other address. The server listens on its own localhost and
  `socat` relays it to 5038, published only on the Docker bridge
  (`172.17.0.1:5037:5038`) — adb on the LAN is a shell on the phone for anyone
  there. A second `socat` hands the container's `127.0.0.1:80` to the ingress,
  which is where `adb reverse` lands. The same IPv6 fact breaks
  `adb emu kill`; power the emulator off with `adb shell reboot -p`.
- **Everything it starts is waited on with `wait -n`**, so if one dies the
  container does, and `restart: unless-stopped` brings them all back.
- **`data/android` is bind-mounted at `/root/.android`**: the key the phone
  trusts, so a new container is not asked again, and the emulator itself —
  kept on purpose, about 1.7GB, so after its first cold boot it starts in
  seconds with Expo Go still on it. The image's `avdmanager` ignores
  `ANDROID_AVD_HOME`; an AVD always lands in `$HOME/.android/avd`.

**`npm run android` picks the device**: a phone listed by adb, else a running
`emulator-5554`, else it boots a headless emulator of the same shape as the
phone and waits for `sys.boot_completed`. A phone listed as `unauthorized`
stops it with a message instead — falling back would hide the prompt on the
phone's screen. Then both `adb reverse` rules, Expo Go if missing, and
`exp://localhost:8081`. The compose file carries `name: <project>`, so the
script reaches the running stack whatever path npm starts it from.

## The phone is a person's own

**A screenshot or a read of the screen happens only with the app in front and
the device awake and unlocked**: `dumpsys power` shows `mWakefulness=Awake`,
and `dumpsys window` shows Expo Go's `ExperienceActivity` in `mCurrentFocus`
and `isKeyguardShowing=false`. A locked phone draws its notifications over the
app, and a screen gone dark but not yet locked still says it is unlocked —
which is why being awake is asked separately. Nothing else that appears on it is described, kept or printed; a stray
screenshot is deleted. The emulator is nobody's, and has no such limit.

## Testing the app

**A device e2e in `tests/app/`, `node:test`, nothing mocked**, run with
`npm run test:app`; `npm test` stays the backend suite (`tests/*.test.mjs`) and
needs no device.

- It runs `npm run android` first and reads the serial off its `using <serial>`
  line, so it tests whichever device a person would get.
- **A launch re-sends the link until the app is in front.** Expo Go started
  cold, just after `am force-stop`, drops the link about half the time on a
  Pixel 8a: its `LauncherActivity` shows and closes and the project never
  opens. A second link to a running Expo Go gets through.
- A dark or locked device **fails the test at once** with a sentence saying
  so; waiting would not wake it, and a deadline spent on it reads as a bug.
- It reads the screen with `uiautomator dump`, the `text` and `bounds` of each
  node, and taps a button at the centre of its bounds — never at fixed
  coordinates. A dump fails while anything is drawing, so reads are polled
  with a deadline **in seconds**, not a count of polls (each poll is several
  adb calls: three hundred took twenty minutes on a runner).
- **A fresh Expo Go covers the app twice**: an introduction to its developer
  menu, whose Continue then opens the menu itself. Tap Continue, then send
  Back while the menu's own "Go home" is on screen — and only then, since Back
  anywhere else leaves the app. **Back is not always enough**: on a runner the
  menu stayed through five minutes of it, while a relaunch came up without
  it. After a few polls in a row still showing the menu, relaunch the app.
- On an emulator only, a failure names what was in front and the last text
  the app's screen showed; that is how the two covers above were found.
- **Every check about the screen compares a boolean.** An assertion on the
  `dumpsys` output itself — `assert.match(window, …)` — prints all of it when
  it fails, every window on the device. It happened once, on the emulator.
- To see the app refuse and recover, **stop and start the ingress**, not the
  service: it breaks the same path, and it starts cleanly. The ingress is
  started again in a `finally`.
- In CI it is **its own job**, beside the backend one: free the runner's disk
  (the emulator image is about 11GB), open KVM with GitHub's udev rule,
  `npm ci`, the develop stack, `npm run test:app` — a cold boot, so minutes,
  in parallel with the fast job. **A cold first boot on a runner brings up
  "System UI isn't responding" over the app**, which is fine behind it; the
  script sets `hide_error_dialogs 1` and the animations to 0 on an emulator it
  booted, never on a phone. **An emulator's screen also times out into a lock
  screen**, so whenever the script uses an emulator it sets `svc power stayon
  true`, wakes it and runs `wm dismiss-keyguard` — an emulator has nobody to
  press anything. A phone is never touched. On failure the job uploads a screenshot of the
  emulator, and the test names what was in front — on an emulator only.
