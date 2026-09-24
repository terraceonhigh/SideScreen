# AGENTS.md — terraceonhigh/SideScreen (personal fork)

Fork of tranvuongquocdat/SideScreen (MIT): Mac host (Swift, `MacHost/`) streams
a virtual display to an Android client (Kotlin, `AndroidClient/`). Primary
device: **Pixel 9 Pro Fold** (inner screen 2076×2152, ~1:1.04, 390 dpi,
Android 17), used as a wireless second screen for a MacBook (macOS 27, arm64).

## Goal

Duet-grade **wireless** experience on the Fold. Wireless mode already exists
(QR pair, token auth, no adb; adb is only for USB mode via `adb reverse`).
The work is reliability and polish, not a rewrite.

## Branches and remotes

- `main` tracks `upstream/main`. Never commit to it; `git pull upstream main`
  to sync.
- `labs` is the integration branch. One branch per task off `labs`:
  `labs/<issue#>-<slug>` or `labs/<slug>`. Merge into `labs` only when the
  build + tests pass.
- `origin` = the fork. Push task branches there freely. **Never push to
  `upstream`, never open PRs upstream** without the owner saying so.

## Build and test

Mac host (from `MacHost/`):
```
swift build --build-system native
swift test  --build-system native     # 49 XCTest tests, all passing at fork time
```
The default (swift-build) build system fails on Swift 6.4 / macOS 27 SDK:
`Package.swift` passes `-fmodule-map-file=Sources/module.modulemap` as a
relative path and the new build system runs from the repo root, so the
`CGVirtualDisplayBridge` module and then Cocoa etc. fail to resolve. Fixing
that is task 0 below. Release bundle: `scripts/build_mac.sh`.

Android client (from `AndroidClient/`): AGP 8.4, Gradle 8.6, Kotlin 1.9.22,
compileSdk 34, needs **JDK 17** and an Android SDK (`ANDROID_HOME`).
```
./gradlew assembleDebug testDebugUnitTest
adb install -r app/build/outputs/apk/debug/app-debug.apk
```
The release installed on the phone (0.11.3, package `com.sidescreen.app`) is
tracked by Obtainium against **upstream** releases; debug builds use the same
package id, so installing one replaces it until the next Obtainium update.

## Hardware-in-the-loop rules

- A Pixel is plugged in / on the same Wi-Fi only when the owner says so.
  Don't assume it; `adb devices` first. The phone has a second user profile
  (user 12) adb can't access; ignore it.
- Anything that needs eyes on the screen (picture quality, latency feel,
  fold/unfold) goes in the task's notes as "needs manual check", not faked.
- Never leave a virtual display running at the end of a task: the owner has
  lost their cursor to it once. `pkill -x SideScreen`;
  `adb shell am force-stop com.sidescreen.app`.
- Wireless single-screen use can lock you out of the Mac (upstream #39).
  Never disable or mirror the built-in display.

## Task queue (pick one, claim it by creating its branch)

0. **Fix the default SwiftPM build** (see above): absolute modulemap path
   (e.g. from `#filePath`) or a proper `.systemLibrary`/C target for the
   bridge header. Done = `swift build` and `swift test` pass without
   `--build-system native`, and CI still works.
1. **#72** dead wireless link shows frozen frame + green "Connected" for ~15
   min → heartbeat/timeout, visible "reconnecting" state.
2. **#71** left mouse button stuck down on the Mac after mid-drag disconnect
   → release all held buttons on disconnect.
3. **#69** any inbound TCP connection drops the current client before the
   new peer authenticates → admit only after handshake.
4. **#74** backgrounded app keeps receiving frames it doesn't decode, logs
   and allocates per frame → pause stream on background.
5. **#55** washed-out picture: full-range BT.709 rendered with limited-range
   matrix → match range flags encoder↔decoder.
6. **Fold support**: resolution preset matching 2076×2152 (and the cover
   screen); handle fold/unfold (configuration change) without dropping the
   session.
7. **#45** touch/stylus drag → mouse drag, for drawing.
8. **#59** cursor disappears after Mac display sleep.
9. **#35** code-based pairing (no camera) — only if 1–4 are done.

Upstream issue text: `gh issue view <n> -R tranvuongquocdat/SideScreen`.

## Done means

Build + tests pass on both sides; new logic has at least one unit test
(existing tests live in `MacHost/Tests/SideScreenTests` and
`AndroidClient/app/src/test`); commit messages say what was verified on
hardware vs. only in tests; `CHANGELOG.md` "Unreleased" gets a line.
