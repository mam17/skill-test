# Reconnaissance

### Phase 1: Map the app and plan coverage

The parent initializes `run-metadata.json`, applies the budget gate in `SKILL.md`, creates agent roots, and seeds the ledger before hunting. Reconnaissance reads the repository, the Gradle cache already on disk, and build outputs that already exist or that an offline build under the recorded execution policy produced. It does not contact backends, Play Console, Firebase, Maven repositories, or other services.

Launch these `research` agents in parallel. They return structured facts with repository-relative `file:line` references and do not write files.

**Agent 1a: Product, stack, and local operation**

```text
Read the Android project at <target>. Do not use network access. Return:
1. Product type, users, and the trust-sensitive actions (login, payments, messaging, health/finance data, device control).
2. Gradle modules and their types (application, library, dynamic feature, Wear, Auto, TV), the application ID(s), build types and product flavors, and which variant ships. Report minSdk, targetSdk, compileSdk per application module.
3. Languages and frameworks: Kotlin/Java mix, Compose or Views, Navigation component, Hilt/Koin/Dagger, Room/SQLDelight/Realm, DataStore, Retrofit/OkHttp/Ktor, kotlinx.serialization/Moshi/Gson, WorkManager, Firebase products, analytics and crash SDKs, payment SDKs, WebView usage, native libraries (.so, NDK, Rust).
4. Exact offline commands that could build the shipped variant, print the merged manifest, run JVM unit tests, and assemble instrumentation tests, with their expected write locations. Do not run them.
5. Whether a merged manifest already exists under build/intermediates/merged_manifests/, and the location of every library manifest the Gradle cache holds for declared dependencies that could contribute components, permissions, or network config.
6. Missing toolchains (SDK, emulator image, adb, apkanalyzer) that limit bounded execution.
Return only source facts with repository-relative file:line references.
```

**Agent 1b: Principals, authority, and controls**

```text
Read all source that establishes identity, authorization, isolation, and privilege for the Android app. Map:
1. Which attacker tiers from SKILL.md are realistic for this product and why (A1 remote, A2 no-permission app, A3 permissioned app, A4 web content, A5 physical, A6 device owner only if the product states it).
2. Permissions declared, requested at runtime, and defined (custom permissions with protectionLevel), plus uses-permission entries added by libraries if visible.
3. Every component (activity, activity-alias, service, receiver, provider) with its exported state, intent filters, permission attributes, taskAffinity, launchMode, grantUriPermissions, and path-permission entries. State whether this comes from the merged manifest or source manifests only.
4. Caller checks in code: getCallingUid/getCallingPackage, signature checks, checkCallingPermission, Binder identity, PendingIntent creation flags, intent validation helpers.
5. Authentication and session controls: login flows, token storage, refresh, logout, biometric and device-credential prompts and what they gate, Keystore key parameters.
6. Network trust: network_security_config per variant, TrustManager/HostnameVerifier implementations, CertificatePinner, WebView SSL handlers.
7. For each control, the strongest source-visible guarantee and the facts it depends on that are not in source (backend behavior, Play settings, OEM).
Return only source facts with repository-relative file:line references.
```

**Agent 1c: Entry surfaces and sinks**

```text
Enumerate every path by which data or control enters the app and the dangerous operations it can reach. For each, give the component or function, the attacker tier that can reach it, the parsing or validation applied, and the sinks downstream:
1. Intents: exported components, implicit intents the app receives, extras and data URIs read, nested Intent or PendingIntent extras, broadcast receivers (manifest and dynamic, with or without permission), sticky or ordered broadcasts.
2. Deep links and App Links: schemes, hosts, pathPrefix/pathPattern, autoVerify, which activity handles them and what it does with each parameter, OAuth/SSO redirect handling, Custom Tabs.
3. Content providers: URIs, query/insert/update/delete/openFile implementations, FileProvider paths, exported and permission state.
4. WebViews: where created, what loads (local assets, remote), JavaScript enabled, addJavascriptInterface methods, shouldOverrideUrlLoading/shouldInterceptRequest, file and universal access settings, onReceivedSslError, WebMessagePort or postMessage use, cookie handling.
5. Files and media: ACTION_VIEW/ACTION_SEND handlers, downloads, imports, zip or bundle extraction, external storage reads and writes, backup rules.
6. Remote inputs: API clients and response models, push (FCM) handlers, WebSocket or SSE, remote config consumed for behavior.
7. Device inputs: NFC, Bluetooth, USB, QR scanning, SMS retriever, accessibility, clipboard, notification actions, app widgets, shortcuts, Wear/Auto/TV surfaces.
8. Storage sinks: SharedPreferences, DataStore, Room/SQLite, files, Keystore, logs, analytics.
Return a table of surface -> tier -> validation -> sinks with repository-relative file:line references.
```

**Agent 1d: Prior evidence and test posture**

```text
Read tests, lint configuration, and history for security-relevant evidence:
1. Existing unit, Robolectric, and instrumentation tests that exercise intent handling, deep links, providers, WebView, auth, storage, or network trust. Name what they prove and what edge cases they omit.
2. lint.xml, lint-baseline.xml, and @Suppress/@SuppressLint entries for security checks (ExportedActivity, ExportedReceiver, ExportedService, ExportedContentProvider, SetJavaScriptEnabled, JavascriptInterface, TrustAllX509TrustManager, BadHostnameVerifier, AllowBackup, HardcodedDebugMode, UnsafeProtectedBroadcastReceiver, UnspecifiedImmutableFlag, WorldReadableFiles, WorldWriteableFiles, UnsafeImplicitIntentLaunch, SecureRandom, TrulyRandom, InsecureBaseConfiguration, AcceptsUserCertificates). Each suppression is a lead, not a finding.
3. ProGuard/R8 rules that keep classes with security relevance, and any rule disabling obfuscation of bridge or model classes.
4. TODO/FIXME/HACK comments and git history entries that mention auth, validation, pinning, exported, or security.
5. Third-party SDK versions declared in Gradle files and version catalogs, listed verbatim for offline comparison; do not look up advisories online.
Return only source facts with repository-relative file:line references.
```

Launch focused follow-up `research` agents only when a surface, library, or lifecycle path remains unclear, and only after re-checking the budget gate for each extra call.

## Prior-run intake

Before selecting work, the parent reads every available prior `coverage-ledger.json` and `findings.json` for the same repo and applies the carry-forward rules in `SKILL.md`. For Android, also compare: the shipped variant, `minSdk`/`targetSdk`, the merged-manifest source (`merged` versus `source-only`), and the dependency list. Any change in these marks every manifest-dependent or dependency-dependent unit as changed source.

## Architecture summary and companion selection

The parent synthesizes `<output-dir>/architecture.md` (about 1,000 words maximum):

1. Product, realistic attacker tiers, protected resources, and ordinary authority.
2. Modules, shipped variant, SDK levels, frameworks, and offline build/test limits.
3. Entry surfaces with tier and the important source-to-sink paths.
4. Trust boundaries and the strongest source-visible control on each, with the external fact each depends on.
5. Repository-relative starting paths per module.
6. Prior coverage gaps, changed-source targets, and same-source confirmed exclusions.
7. Companion selection derived from [ATTACK-CLASSES.md](ATTACK-CLASSES.md): selected `ANDROID-*.md` files and the boundaries that require them. Select a companion because reconnaissance found its boundary, not because a library name appears.

Keep per-unit block maps in the ledger, not in `architecture.md`.

## Deterministic coverage ledger

The parent writes `<output-dir>/coverage-ledger.json` as a top-level JSON array. Derive one unit per material combination of entry surface, trust boundary, subsystem (Gradle module), and attack class, at the granularity the profile sets. For a scoped run, seed in-scope surfaces and keep discovered out-of-scope surfaces as `out_of_scope` units.

Each dimension has a human label and a stable source-derived value in `canonical_refs`. Android conventions:

- `surface`: the manifest entry or code entry that receives input, for example `app/src/main/AndroidManifest.xml#com.example.app.DeepLinkActivity`, `core/network/src/main/kotlin/com/example/net/Api.kt#AuthApi`, `app/src/main/kotlin/com/example/push/PushService.kt#onMessageReceived`.
- `boundary`: the control that should stop the attack, for example `app/src/main/kotlin/com/example/link/LinkValidator.kt#validate`, `app/src/main/res/xml/network_security_config.xml#base-config`, `app/src/main/AndroidManifest.xml#permission com.example.app.INTERNAL`.
- `subsystem`: the Gradle module path, for example `:app`, `:feature:payments`, `:core:storage`.
- `attack_class`: `FILE.md#` plus the exact class name as written in bold or as a heading in that file, for example `ANDROID-COMPONENTS-AND-IPC.md#Exported component overreach`.
- `lifecycle` (when material): `install`, `upgrade`, `logout`, `account-switch`, `backup-restore`, `process-death`, `multi-user`.

Derive `coverage_id` exactly as the validator expects: NFC-normalized visible references, RFC 3986 percent-encoding leaving only `A-Z a-z 0-9 - . _ ~` unescaped, joined with `::` in the order surface, boundary, subsystem, attack_class, then lifecycle when present. Use `profile/quick/all-in-scope-subsystems` as the quick-profile subsystem. Never include wave, agent, verdict, severity, or line numbers. Sort by `coverage_id` before each assignment and fail on duplicates.

Each unit records:

```json
{
  "coverage_id": "...",
  "canonical_refs": {
    "surface": "app/src/main/AndroidManifest.xml#com.example.app.DeepLinkActivity",
    "boundary": "app/src/main/kotlin/com/example/link/LinkValidator.kt#validate",
    "subsystem": ":app",
    "attack_class": "ANDROID-DEEPLINKS-AND-WEBVIEW.md#Deep-link parameter trust"
  },
  "surface": "DeepLinkActivity (VIEW, https://example.com/*)",
  "boundary": "LinkValidator.validate",
  "subsystem": ":app",
  "attack_class": "Deep-link parameter trust",
  "starting_paths": ["app/src/main/kotlin/com/example/link"],
  "ordinary_attack_class_block": "ATTACK-CLASSES.md#Injection",
  "selected_companion_blocks": [
    "ANDROID-DEEPLINKS-AND-WEBVIEW.md#Core discipline",
    "ANDROID-DEEPLINKS-AND-WEBVIEW.md#Deep-link parameter trust",
    "ANDROID-DEEPLINKS-AND-WEBVIEW.md#Universal moves",
    "ANDROID-DEEPLINKS-AND-WEBVIEW.md#Validation rules"
  ],
  "excluded_blocks": [{"block": "ANDROID-DEEPLINKS-AND-WEBVIEW.md#JavaScript bridge exposure", "reason": "no WebView in this module (recon 1c)"}],
  "prior_status": "none",
  "attempts": [],
  "wave": 1,
  "status": "planned",
  "agent_id": null,
  "reviewed_paths": [],
  "local_checks": [],
  "result_fingerprints": [],
  "unresolved": []
}
```

`selected_companion_blocks` includes each chosen class plus that companion's `Core discipline`, `Universal moves`, and `Validation rules`. `excluded_blocks` records every considered but unselected block with the source fact that excludes it. The parent may add bookkeeping fields but keeps the semantic fields above stable.

`prior_status` values, the `attempts` archive, and the state table are identical to the upstream skill:

| Status | `agent_id` | `reviewed_paths` / `local_checks` | `result_fingerprints` | `unresolved` |
|---|---|---|---|---|
| `planned` | null | empty | empty | empty |
| `not_applicable`, `out_of_scope`, `deferred` | null | empty | empty | nonempty reason |
| `in_progress` | canonical owner | empty | empty | empty |
| `blocked` | canonical owner | both nonempty | empty | nonempty blocker |
| `covered` | canonical owner | both nonempty | empty | empty |
| `candidate` | canonical owner | both nonempty | nonempty | optional |

Every check records its own `agent_id` and nonempty `reviewed_paths`; the unit-level `reviewed_paths` is their union. A source-only check uses `artifact: null`. A local check (unit-test output, logcat capture, `dumpsys` output, harness result) requires a regular file promoted by the parent under exactly `agents/<check.agent_id>/artifacts/`.

Run `node <skill-dir>/validate-coverage-ledger.cjs <output-dir>/coverage-ledger.json` after seeding, after every parent update, and before Phase 6. Fix every error before assigning work or making a coverage claim.
