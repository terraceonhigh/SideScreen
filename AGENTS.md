# AGENTS.md — terraceonhigh/SideScreen (personal fork)

Fork of tranvuongquocdat/SideScreen (MIT): Mac host (Swift, `MacHost/`) streams
a virtual display to an Android client (Kotlin, `AndroidClient/`). The human owner is **the Architect** (house convention). Primary
device: **Pixel 9 Pro Fold** (inner screen 2076×2152, ~1:1.04, 390 dpi,
Android 17), used as a wireless second screen for a MacBook (macOS 27, arm64).

## Goal

Duet-grade **wireless** experience on the Fold. Wireless mode already exists
(QR pair, token auth, no adb; adb is only for USB mode via `adb reverse`).
The work is reliability and polish, not a rewrite.

## The bar: Sidecar-grade invisibility (house standard: `~/Labs/Comprador`)

Read `~/Labs/AGENTS.md` (house law) first, then study Comprador as the model
for what "done" feels like here: `Comprador/README.md` ("Plug in. … That's
it."), `Comprador/CLAUDE.md` (decision records, icon states, the "Check your
phone" UX), and `Comprador/docs/INVISIBILITY.md` (the thumb-drive baseline
and a ranked gap list). SideScreen's equivalent baseline is **Sidecar**:

1. **Open the lid near the phone → the second screen is there.** No clicking
   Start, no picking a mode, no QR after the first pair. Headless/auto-start
   and auto-reconnect should be the default, not a settings excursion.
2. **No developer ceremony, ever, in the default path.** Wireless is the
   product; USB/adb is a power-user option and must never be a prerequisite,
   a red status row, or the first thing the settings window shows.
3. **One expected moment of friction, then silence.** First launch primes
   every permission (Screen Recording, Local Network, Accessibility, camera
   on the phone) in one guided pass; after that, no alerts in steady state.
   Stale-grant problems (upstream #77, the tccutil workaround) are detected
   and fixed for the user, not documented at them.
4. **Failures say what to do, in one line, and then self-heal.** Menu bar
   icon states map to real states (idle / searching / connected /
   error-with-reason); a dead link visibly reconnects instead of freezing
   (#72); nothing is left stuck (#71: held mouse button).
5. **No collateral damage.** Your cursor never gets lost on a screen you
   can't see; the built-in display is never disabled (#39); stopping the app
   removes the virtual display cleanly; other apps behave (#65).
6. **Signed and notarized**, so first launch is a double-click. The fork is
   ad-hoc signed today. Comprador's build identity setup
   (`Comprador/docs/PLAN-BUILD-IDENTITY.md`) is the reference; signing
   credentials are the Architect's to operate, per house secrets rules.

Write a `docs/INVISIBILITY.md` for this project early (Sidecar baseline →
where we match → gaps ranked by leverage) and keep it current; it should
drive the queue below. Keep a `docs/DECISIONS.md` for anything that departs
from upstream's design.

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

- A Pixel is plugged in / on the same Wi-Fi only when the Architect says so.
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

Order is a starting point; re-rank once `docs/INVISIBILITY.md` exists.

- **Invisibility pass** (do first, in parallel with 0): write
  `docs/INVISIBILITY.md` from the bar above by actually walking the
  first-run and daily flows on both devices and logging every click,
  prompt, and wait. Upstream's settings window (`SettingsWindow.swift`,
  1.7k lines) and `MainActivity.kt` (1.8k lines) are where most of the
  friction lives.

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
