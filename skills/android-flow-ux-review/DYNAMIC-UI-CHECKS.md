# Dynamic UI Checks

#### When to use this file

Phase 4, and whenever a needs-device-check lead must be settled. Every check runs the target's own debug build on a fresh emulator with dummy data. Record the device profile, API level, and the exact configuration for every piece of evidence; screenshots go in the run's evidence directory with the screen + configuration in the filename.

## Configuration matrix

Verify core flows under each axis (combine where cheap). Minimum worthwhile set:

| Axis | Values | Settles |
|---|---|---|
| Device profile | small phone (~320dp wide), default phone, tablet/foldable | layout resilience, large-screen orientation unlock on targetSdk 35+ |
| Font scale | 1.0 and 2.0 (`adb shell settings put system font_scale 2.0`) | clipping, disappearing controls |
| Dark mode | `adb shell cmd uimode night yes` / `no` | theming, hardcoded colors |
| Locale / RTL | longest shipped locale (de/ru/hi) and `ar` or force-RTL dev option | expansion, mirroring |
| Display size | largest display zoom | compound scaling |

## Interruption drills (per core flow)

- **Rotation** (or window resize on large screen): perform mid-input and mid-loading; state and in-flight work survive.
- **Process death**: background the app mid-flow, then kill it the way the system would and relaunch from Recents:
  `adb shell am kill <package>` (only while backgrounded — this simulates real reclaim; a force-stop is NOT equivalent, it clears the task). Enable "Don't keep activities" in developer options for the per-screen sweep. Input, step progress, and scroll position must restore.
- **Permission sheet / interruption overlay**: trigger a runtime permission or system dialog mid-flow; resume cleanly, no duplicate dialogs.
- **Offline**: `adb shell svc wifi disable` + `svc data disable` before and during a load; verify the designed offline state and recovery on re-enable.

## Accessibility sweep

- TalkBack on (`adb shell settings put secure enabled_accessibility_services com.google.android.marvin.talkback/com.google.android.marvin.talkback.TalkBackService` on images that bundle it, or enable in Settings): traverse each core screen start to finish; every functional control announces a sensible label/role; complete one core flow end-to-end by swipe navigation alone.
- Accessibility Scanner (if installed) or `adb shell uiautomator dump` for programmatic label/bounds checks: parse the dump for clickable nodes with empty `content-desc` and bounds smaller than 48dp — this turns label/touch-target review into evidence.

## Evidence capture

- Screenshot: `adb exec-out screencap -p > <screen>_<config>.png`; record video for flow defects: `adb shell screenrecord /sdcard/flow.mp4` (pull, then delete from the device).
- Layout bounds overlay for overlap/inset claims: `adb shell setprop debug.layout true` (visualizes margins under system bars).
- The claim in a finding cites its artifact: configuration + file. An undocumented screenshot is not evidence.

## What each check proves

Device checks prove rendering and behavior **on that configuration**. They do not prove OEM-specific behavior (gesture nav variants, aggressive task killers) — note those as device-coverage limits in the report, not as cleared. Espresso/instrumentation tests added for regressions prove the scripted path only; manual sweeps still own visual judgment.
