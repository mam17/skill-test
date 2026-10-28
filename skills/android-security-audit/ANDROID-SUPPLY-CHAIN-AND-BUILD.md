# Supply Chain and Build Hunting

#### When to use this file

Reach for this file when the app's trust depends on how it is built and assembled: Gradle dependency resolution, plugins and `buildSrc`, the manifest merger, build types and flavors, signing configuration, R8/ProGuard rules, bundled third-party SDKs and their initialization, and CI. The attacker ranges from a malicious dependency or build input to a misconfiguration that ships an insecure artifact. Use `ANDROID-STORAGE-AND-SECRETS.md` for committed secrets and `ANDROID-NETWORK-AND-CRYPTO.md` for SDK network trust.

## Core discipline (include in every agent prompt for this domain)

```
- A dependency being outdated or theoretically vulnerable is not a finding by itself. Show the version in source, the code path that uses the affected functionality, and the reachable consequence, or record it as needs_validation (advisory lookup is an owner-observed, offline-unavailable step here).
- Build logic (Gradle scripts, buildSrc, included builds, plugins) is code that runs on the build machine. Treat an untrusted or mutable build input as a build-integrity concern, distinct from app runtime.
- The manifest merger can add components, permissions, and network config from libraries. Conclusions about the shipped app must use the merged manifest for the release variant.
- A relaxation confined to the debug build type or a non-shipping flavor is not a finding. Confirm the shipped variant.
- Signing determines app identity and signature-permission trust. v2/v3 scheme, key strength, and whether secrets sit in source matter; Play App Signing changes the final certificate and is an owner-observed fact.
- Use confirmed for an in-repo build/config failure with a shipped-variant consequence. Use needs_validation for CI secrets, registry, advisory, or Play-signing facts not in the repo.
```

## Dependency and build-input attack classes (subagent_type: `general`)

**Dynamic and unpinned dependency resolution**
Version declarations with `+` or open ranges, `mavenLocal()` or an untrusted/HTTP repository in the resolution list, a `latest.release`/snapshot dependency, or a plugin resolved without a pinned version, so the build can consume content that changed without review. Check `build.gradle(.kts)`, the version catalog, `settings.gradle` repositories, and `buildscript` classpath. Note HTTP (non-TLS) repositories specifically.

**Mutable or untrusted build logic**
`buildSrc`, included builds, or `apply from:` pulling a script from a remote or mutable location; a Gradle plugin from an untrusted source; a build step that downloads a tool or asset without integrity checking and then executes or bundles it. Check what runs at configuration/build time and whether its input is trusted and pinned.

**Bundled SDK initialization and overreach**
A third-party SDK (analytics, ads, crash, attribution, payment) auto-initializes via a manifest `provider`/`ContentProvider` merge, requests broad permissions, adds exported components, registers URL handlers, or exfiltrates data. Check merged-manifest contributions, the SDK's declared permissions, and what data the app hands it. Cross-reference `ANDROID-STORAGE-AND-SECRETS.md#Logs and analytics`.

## Build-configuration attack classes (subagent_type: `general`)

**Shipped insecure build settings**
The release variant ships `debuggable true`, `minifyShrinkResources` disabled where it was relied on for a control, a test-only or insecure flavor in the release dimension, `BuildConfig` flags that enable debug behavior at runtime, or a `debug` signing config on a release variant. Check the `buildTypes`/`productFlavors` matrix and which variant is published.

**Manifest-merger surprises**
The merger adds an exported component, a permission, `usesCleartextTraffic`, a network-security-config reference, or a `provider` from a library that the app author did not intend. Compare source manifests with the merged manifest and attribute each added element. This feeds `ANDROID-COMPONENTS-AND-IPC.md` and `ANDROID-NETWORK-AND-CRYPTO.md`.

**R8/ProGuard rules weakening a control**
Keep rules that preserve class names relied on by a security check, `-dontobfuscate` on bridge/model classes where obscurity was the control (weak by itself, note as hardening), or rules that strip an assertion or check. More importantly, check that minification does not remove a runtime check the app depends on, and that `-keep` does not expose a reflection surface. Frame obfuscation itself as resilience (A6), not a boundary.

## Signing and release attack classes (subagent_type: `general`)

**Signing-key and credential exposure in the repo**
A keystore (`*.jks`, `*.keystore`), `signingConfigs` with literal `storePassword`/`keyPassword`, or signing secrets in `gradle.properties`/`local.properties` committed to the repo. State what signing with that key authorizes (app identity, update trust, signature permissions). This is high when a real release key is exposed.

**Signature-scheme and update-trust gaps**
The app relies on signature-level permissions or caller signature checks, but the signing scheme or certificate pinning of callers is weak or unverified, or the app trusts an update/patch channel without verifying authenticity. Check the signature-verification code and the scheme; note that Play App Signing's final certificate is an owner-observed fact.

## Universal moves (apply across the above)

- Inventory declared dependencies and plugins with exact versions (verbatim, for offline reference), the repository list, build types and flavors, the published variant, signing configs, and R8 rules.
- Produce or read the merged manifest for the release variant and attribute every component, permission, and network setting to source or a specific library.
- For a consequence, connect the build/config fact to a shipped-variant runtime effect (an added exported component, a shipped debug setting, an exposed key). For advisory-dependent dependency claims, record `needs_validation` with the offline-unavailable lookup.

## Validation rules (apply before reporting ANY finding here)

1. Name the build input or setting, who or what can influence it, the shipped variant it reaches, and the runtime or identity consequence.
2. Confirm the setting ships in the release variant, and attribute merged-manifest additions to their source. A debug-only relaxation is not a finding.
3. For dependency findings, cite the in-source version and a used code path, or record `needs_validation` for the advisory lookup. For signing, state what the exposed key authorizes.
4. Build-integrity checks run offline; do not fetch dependencies, contact registries, or run CI. Keep any local check to offline Gradle inspection.
5. Return `confirmed` for an in-repo failure with a shipped-variant consequence. Return `needs_validation` for CI, registry, advisory, or Play-signing facts outside the repo.
