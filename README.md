# Air Gesture

Recreates a Galaxy-S7-style "air gesture" Home Screen swipe on a POCO M5s
(MIUI 14, Android 13) using the front camera, on-device MediaPipe hand
tracking, and an AccessibilityService that dispatches a synthetic swipe.

Package name: `com.airgesture.app`

## ⚠️ Build status — read this first

This sandbox has **no Android SDK, no Gradle/Kotlin compiler, and no network
access** (every Maven/Gradle/GitHub host is blocked at the proxy level), so
`./gradlew assembleDebug` could not actually be run here, and I will not
claim it was. What I *did* do instead:

- Wrote every file against real, current Android/CameraX/MediaPipe/DataStore
  APIs, verified against Google's official docs (not invented).
- Ported the core `GestureDetector` state machine to plain Java and actually
  **compiled and ran it** with the JDK available in this sandbox
  (`java GestureCheck.java`, JEP 330 single-file execution) against all 10
  required scenarios. This caught and fixed one real bug (a test-harness bug
  that reported only the last frame's result instead of any trigger during a
  sweep) before finalizing `GestureDetectorTest.kt`. Final result: **14/14
  checks passed.**
- I could not execute the Kotlin/Android-specific files (CameraX, MediaPipe,
  AccessibilityService, Compose UI, DataStore) at all, since those need the
  Android SDK and real device/emulator APIs that don't exist as plain JVM
  classes. You must build this in Android Studio or Claude Code on a machine
  with SDK + network access, and fix anything that comes up there.

Two things are guaranteed to need your attention before the first build:

1. **`gradle-wrapper.jar` is missing.** Open this folder in Android Studio
   (it will offer to generate the wrapper and sync automatically), or run
   `gradle wrapper --gradle-version 8.7 --distribution-type bin` once from
   the project root if you have Gradle installed locally.
2. **The MediaPipe model file is missing**
   (`app/src/main/assets/hand_landmarker.task`) — see
   `app/src/main/assets/PUT_HAND_LANDMARKER_MODEL_HERE.txt` for the exact
   download URL. The app will fail to initialize hand detection (with a
   logged error, not a crash) until this file is in place.

Everything else — Gradle config, manifest, all Kotlin source, the
AccessibilityService config, and unit tests — is complete.

## Project structure

```
AirGesture/
├── settings.gradle.kts
├── build.gradle.kts
├── gradle.properties
├── gradle/wrapper/gradle-wrapper.properties      (jar missing, see above)
└── app/
    ├── build.gradle.kts
    ├── proguard-rules.pro
    └── src/
        ├── main/
        │   ├── AndroidManifest.xml
        │   ├── assets/                            (put hand_landmarker.task here)
        │   ├── java/com/airgesture/app/
        │   │   ├── AirGestureApp.kt
        │   │   ├── MainActivity.kt
        │   │   ├── camera/FrontCameraManager.kt
        │   │   ├── detection/GestureDetector.kt
        │   │   ├── detection/HandDetector.kt
        │   │   ├── accessibility/AirGestureAccessibilityService.kt
        │   │   ├── service/AirGestureForegroundService.kt
        │   │   ├── settings/SettingsRepository.kt
        │   │   └── ui/MainScreen.kt
        │   └── res/
        │       ├── values/ (strings, themes, colors)
        │       ├── xml/accessibility_service_config.xml
        │       ├── drawable/ic_launcher_foreground.xml
        │       └── mipmap-anydpi-v26/ic_launcher.xml
        └── test/java/com/airgesture/app/detection/GestureDetectorTest.kt
```

## Versions used

- Android Gradle Plugin 8.6.1, Gradle 8.7, Kotlin 1.9.24, Java 17
- compileSdk/targetSdk 34, minSdk 24
- Jetpack Compose (BOM 2024.06.00) + Material3 for the UI
- CameraX 1.3.4 (front camera, headless `ImageAnalysis` only — no preview)
- `com.google.mediapipe:tasks-vision:latest.release` (HandLandmarker, LIVE_STREAM mode)
- `androidx.datastore:datastore-preferences:1.1.1` for persisted settings

## Permissions required

- `CAMERA` — runtime-requested, to see your hand
- `FOREGROUND_SERVICE` + `FOREGROUND_SERVICE_CAMERA` — Android 13/14 require a
  typed foreground service to keep using the camera while the app isn't in
  the foreground
- `POST_NOTIFICATIONS` — Android 13+ requires this to show the "Air Gesture is
  active" persistent notification
- Accessibility Service access — a system-level permission the user must
  grant manually in Settings; it is not a manifest `<uses-permission>`

No internet, storage, or media permissions are requested. No images/video are
ever saved or uploaded — frames are processed in memory and discarded.

## How the pipeline works

```
Front camera (CameraX ImageAnalysis)
    -> HandDetector (MediaPipe HandLandmarker, wrist landmark, LIVE_STREAM)
    -> GestureDetector (pure Kotlin state machine, see below)
    -> AirGestureAccessibilityService.requestSwipe()
    -> dispatchGesture() -> MIUI Home Screen
```

`GestureDetector` keeps a short rolling time-window of normalized hand-center
samples. On each new sample it:
1. Discards the sample if a hand isn't present (aborts any in-progress
   gesture rather than bridging the gap).
2. Discards accumulated history if the frame-to-frame jump is larger than a
   configured "physically implausible" threshold (tracking glitch).
3. Drops samples older than the history window.
4. Requires the window to span a minimum duration before evaluating.
5. Requires horizontal displacement to exceed a minimum threshold **and**
   dominate vertical displacement by a configurable ratio.
6. On trigger, clears history and starts a cooldown so a fresh, distinct
   movement is required before the next gesture can register.

A single "Sensitivity" slider in the UI maps to concrete thresholds via
`GestureConfig.fromSensitivity()`; "Cooldown" is a direct millisecond value.

## Build steps (on a machine with Android Studio / SDK)

1. Open the `AirGesture/` folder in Android Studio, let it generate the
   Gradle wrapper and sync (or run `gradle wrapper` yourself first).
2. Download `hand_landmarker.task` (URL above) into `app/src/main/assets/`.
3. `./gradlew clean test` — run the gesture-detector unit tests.
4. `./gradlew assembleDebug` — build the APK.
5. APK output path: `app/build/outputs/apk/debug/app-debug.apk`

## Installing on the POCO M5s

1. Enable Developer Options + USB debugging, connect via USB, and run
   `adb install -r app/build/outputs/apk/debug/app-debug.apk`
   (or copy the APK to the phone and install it directly; MIUI will ask you
   to confirm "install from unknown sources" the first time).
2. Open **Air Gesture**.

## Enabling Accessibility Service

1. In the app, tap **"Open Accessibility settings."**
2. Find **Air Gesture** in the list, enable it, and confirm the MIUI dialog.
3. Return to the app — the "Accessibility Service" status should flip to OK.

## Enabling Air Gesture

1. Grant the camera permission when prompted (or via the button in the app).
2. Once both status rows show OK, the master switch becomes enabled — turn
   it on. Android 13+ will also prompt for the notification permission so
   the "Air Gesture is active" notification can be shown.
3. A persistent notification confirms detection is running. Move your open
   hand horizontally in front of the front camera at a normal distance.

## MIUI/POCO recommendations

MIUI aggressively kills background work. If gestures stop firing after a
while:
- Settings → Apps → Manage apps → Air Gesture → **Autostart**: enable
- Settings → Battery & performance → App battery saver → Air Gesture: set to
  **No restrictions**
- Settings → Apps → Air Gesture → **Battery saver / Background activity**:
  allow
- Re-check Accessibility access hasn't been revoked (MIUI occasionally resets
  this after major updates)

No root, bootloader unlock, Shizuku, or ADB is required for normal use.

## Known limitations

- Unverified by an actual Gradle/Android build — see the status section
  above. Expect to fix minor issues (dependency resolution, resource IDs) on
  first real build, though the code was written carefully against current
  APIs.
- MediaPipe model file must be added manually (multi-MB binary, not
  something that should be fabricated).
- `gradle-wrapper.jar` must be generated locally (binary, not fetchable from
  this sandbox).
- Front-camera mirroring is corrected by flipping the analyzed bitmap before
  it reaches MediaPipe; this is the standard approach but is worth a quick
  visual sanity check on-device (move your hand right and confirm the swipe
  direction matches expectation, not its mirror image).
- MIUI's aggressive background-service killing is a platform limitation that
  cannot be fully bypassed without root — the in-app guidance is the
  practical mitigation.

## Confirmation

- Gradle build: **not run** (no SDK/network in this sandbox) — must be run
  by you
- Unit tests: **algorithm logic verified** via a plain-Java port executed
  directly in this sandbox (14/14 scenario checks passed); the actual
  `./gradlew test` run against the Kotlin file still needs to happen on your
  machine
- Lint: not run
- APK: not generated
