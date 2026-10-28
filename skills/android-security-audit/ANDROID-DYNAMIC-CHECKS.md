# Android Dynamic Checks

#### When to use this file

Hunters and verifiers read this file before running anything. It lists the bounded, observational checks allowed under the execution policy, and — just as important — what each one does and does not prove. This file is not an exploitation guide: every check confirms a source-grounded candidate by observing the target's own behavior with dummy data, and stops at the first boundary result. It does not develop attack chains, defeat the target's own protections, or touch other apps or real users.

Every check uses a fresh emulator (or a dedicated test device with no personal account), a **debug build of the target**, dummy accounts, and a local or owner-designated non-production backend. Record the emulator image (for example `system-images;android-<api>;google_apis;x86_64`), the exact command, and the minimum observed output. If any control required by the execution policy is unavailable, do not run the check: keep the candidate as `needs_validation` with the missing capability.

## What establishes ground truth for the manifest

The exported state and permissions that decide a tier-A2 boundary come from the **merged** manifest for the shipped variant, not the source manifest. Produce or read it with ordinary developer tooling:

```bash
# Offline merged manifest for the shipped variant (preferred, under the execution policy)
./gradlew :app:process<Variant>MainManifest --offline
#  -> app/build/intermediates/merged_manifests/<variant>/AndroidManifest.xml

# From a built APK
apkanalyzer manifest print app-<variant>.apk
aapt2 dump xmltree --file AndroidManifest.xml app-<variant>.apk
aapt2 dump permissions app-<variant>.apk

# From the installed debug build on the emulator
adb shell dumpsys package <applicationId>
```

Proves: exported state, permissions and their `protectionLevel`, intent filters, `taskAffinity`, `launchMode`, and provider authorities and grants as actually merged, including library-contributed components. Does not prove: release-variant behavior unless you inspected the release-like variant; Play App Signing's final certificate (an owner-observed fact).

## Observing how a component handles a dummy input

The Android developer tools `am` and `content` let you send one dummy, non-sensitive input to a component and observe the app's reaction (a logcat line, a returned row, a screen). Use them to confirm reachability and behavior, then establish the attacker tier from the merged manifest.

- `adb shell am start` / `am broadcast` / `am start-foreground-service`: deliver one dummy intent to an activity, receiver, or service and observe what the handler does with it.
- `adb shell content query` / `content read`: observe what a provider returns for a dummy URI.
- `adb logcat` around the single action: capture whether a dummy token, PII, or error is logged.

**What this does and does not prove.** These commands run from the `adb` shell, which holds privileges an ordinary no-permission app does not. Reaching a component from `adb` therefore shows the handler's behavior, but it does **not** by itself prove a tier-A2 (no-permission app) boundary. Establish the tier separately from the merged manifest: a component is reachable by A2 when it is `exported` with no permission, or gated only by a `normal`-protection permission. If the manifest does not settle it, the candidate is `needs_validation`.

## Confirming a tier-A2 (co-installed app) boundary without offensive tooling

Prefer manifest evidence over building an attacker. A tier-A2 claim is confirmed when the merged manifest shows the component `exported` without an effective permission **and** a source trace shows a sensitive operation or disclosure behind it. You do not need to build a malicious app to confirm this; the manifest fact plus the source sink is the boundary. If reachability genuinely cannot be settled from the manifest and source, record it as `needs_validation` with the exact missing fact and an owner-observed plan, rather than constructing an exploit. Do not write or install a second app that attacks the target's data; this skill confirms boundaries by observation, not by exploitation.

## Unit, Robolectric, and instrumentation harnesses

The narrowest and most reliable check is usually a JVM or Robolectric test around the specific guard, parser, or validator, fed one dummy input, asserting the wrong result the candidate claims.

- JVM unit test (`./gradlew :module:test --offline`): parsers, validators, crypto functions, URL/path logic, serialization.
- Robolectric: intent parsing, `Uri` handling, provider logic, preference storage, without an emulator.
- Instrumentation (`./gradlew :app:connected<Variant>AndroidTest --offline` on a fresh emulator): real framework behavior for components, WebView, Keystore, and biometric flows.

Proves: the guard returns the wrong value, the parser accepts malformed input, the store retains a dummy token after logout. Does not prove: anything about the shipped release config unless the test runs against it, or anything about the real backend.

## Observing network trust behavior

To confirm a TLS-trust candidate, observe the target's **own** debug build against a **local mock** on the emulator (for example `MockWebServer` presenting an untrusted certificate, or a plain-HTTP endpoint): does the client accept an untrusted certificate, or send cleartext to a first-party host. This observes the app's trust decision with dummy traffic. Do not intercept real endpoints, real user traffic, or production, and do not defeat the app's own pinning on a device you control as "proof" — that is tier A6 and is not a finding.

## APK inspection

`apkanalyzer`, `aapt2`, and `dumpsys` read a built artifact offline:

- `apkanalyzer manifest print|permissions`, `aapt2 dump badging|permissions|xmltree`: shipped manifest facts.
- `apkanalyzer files list` / `apkanalyzer dex packages`: what shipped, bundled assets, dynamic-feature modules.
- Grep decompiled resources and `strings` for committed secrets, but confirm what any hit authorizes before reporting (see `ANDROID-STORAGE-AND-SECRETS.md`).

## Evidence and promotion

Each check writes its output only inside the agent's `scratch/`. The parent promotes predeclared, bounded, regular files into `artifacts/` per the promotion rules in `HUNTING.md`; capture only the minimum dummy output needed to prove the invariant (one logcat line, one returned row, one manifest excerpt), never the ambient environment, credentials, or unrelated host data. Record the emulator image and exact command in the check's `result`.
