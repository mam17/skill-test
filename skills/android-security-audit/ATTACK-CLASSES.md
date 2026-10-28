# Attack Classes

#### Attack classes — choose and split based on Phase 1

Select attack classes relevant to the app. Not every class applies to every codebase. Keep validation to source review plus bounded emulator or unit evidence with dummy data; never probe production backends, other users, or other apps' data. Use `confirmed` only when source evidence and bounded validation establish the full boundary and a meaningful result. Use `needs_validation` when a backend, Play Console, device, OEM, or shipped-manifest fact is required but unavailable.

## Android companion routing

> **Exported components, intents, broadcasts, providers, pending intents, bound services, task affinity, app widgets, notification actions**: use [ANDROID-COMPONENTS-AND-IPC.md](ANDROID-COMPONENTS-AND-IPC.md).
>
> **Custom schemes, App Links, OAuth redirects, Custom Tabs, WebView, JavaScript bridges, file and universal access, SSL error handling in WebView**: use [ANDROID-DEEPLINKS-AND-WEBVIEW.md](ANDROID-DEEPLINKS-AND-WEBVIEW.md).
>
> **SharedPreferences, DataStore, Room/SQLite, files, external storage, backup rules, Keystore, logs, clipboard, screenshots, hardcoded secrets**: use [ANDROID-STORAGE-AND-SECRETS.md](ANDROID-STORAGE-AND-SECRETS.md).
>
> **TLS trust, network security config, pinning, OkHttp/Retrofit/Ktor, response deserialization, crypto primitives, token handling on the wire**: use [ANDROID-NETWORK-AND-CRYPTO.md](ANDROID-NETWORK-AND-CRYPTO.md).
>
> **Login, session, logout, account switch, biometric and device-credential gates, OTP, PKCE, AccountManager, Credential Manager, process death and saved state**: use [ANDROID-AUTH-SESSION-AND-STATE.md](ANDROID-AUTH-SESSION-AND-STATE.md).
>
> **Kotlin and runtime input handling: Parcelable/Serializable extras, kotlinx.serialization and Gson polymorphism, path and zip handling, SQL building, reflection and dynamic code loading, coroutine state races, JNI handoff**: use [ANDROID-KOTLIN-RUNTIME-AND-INPUT.md](ANDROID-KOTLIN-RUNTIME-AND-INPUT.md).
>
> **Gradle dependencies, plugins, manifest merger, build types and flavors, signing, ProGuard/R8 rules, third-party SDK initialization, CI**: use [ANDROID-SUPPLY-CHAIN-AND-BUILD.md](ANDROID-SUPPLY-CHAIN-AND-BUILD.md).
>
> **Bounded emulator, adb, and harness checks for any of the above**: hunters and verifiers read [ANDROID-DYNAMIC-CHECKS.md](ANDROID-DYNAMIC-CHECKS.md) for what each check proves.
>
> **Native code (`.so`, JNI, NDK)**: this skill traces inputs to the JNI boundary only. If native parsing of untrusted bytes matters, record the handoff as `uncovered` with the native entry point so the parent can run a memory-safety review with a dedicated native-safety companion.

## Ordinary attack classes (Android framing)

**Injection** (subagent_type: `general`)
Trace untrusted input from an Android entry point to a dangerous sink. Entry points: intent extras and data URIs, deep-link query parameters, content-provider URIs and selection arguments, broadcast extras, push payloads, WebView messages, files opened via `ACTION_VIEW` or share, NFC/Bluetooth payloads, server responses. Sinks: raw SQL (`execSQL`, `rawQuery`, Room `@RawQuery`, `SupportSQLiteDatabase`), `File` paths and `ContentResolver.openFile`, `Runtime.exec`/`ProcessBuilder`, `WebView.loadUrl`/`loadData`/`evaluateJavascript`, `Intent.parseUri` and `startActivity` with attacker-built intents, `String.format`, logging, notification text, `Uri.parse` into HTTP clients, reflection (`Class.forName`). Look for second-order paths: a value stored safely, then later used as a file name, URL, intent, or SQL fragment by different code.

**Access control** (subagent_type: `general`)
Verify that a caller cannot do something outside its authority. On Android the caller is often another app or a web page, not a user: check `Binder.getCallingUid()`/`getCallingPackage()` use, signature verification of callers, `checkCallingPermission`, permission `protectionLevel`, `exported` state in the merged manifest, `grantUriPermissions` scope, and whether a component gates on "is the user logged in" but forgets "did a trusted caller ask". Inside the app, check that feature gates (premium, admin, parental) are enforced where the action executes, not only in the UI.

**Resource and file handling** (subagent_type: `general`)
Path traversal from intent, provider, or server-controlled names into `filesDir`, `cacheDir`, or `FileProvider` roots (including `..`, encoded separators, and symlinks on external storage); zip slip in downloaded bundles or themes; temp files in world-readable locations; TOCTOU on external storage; unbounded downloads into app storage; `ContentResolver` queries without projection or selection control.

**Cryptography and secrets** (subagent_type: `general`)
`Cipher.getInstance("AES")` (defaults to ECB), static or reused IVs and nonces, keys derived from device identifiers or package names, `Random` instead of `SecureRandom`, weak PBKDF iteration counts, non-constant-time comparison on MACs, custom crypto, Keystore keys without the binding the use case requires, local JWT decoding treated as verification, secrets in `strings.xml`, `BuildConfig`, assets, or native strings. A hardcoded value alone is not a finding: state what the value authorizes.

**Business logic** (subagent_type: `general`)
Client-side state machines: onboarding, KYC, purchase, transfer, PIN retry counters, offline mode. Can a step be skipped by deep link or process-death restore? Does a retry counter live only in memory or SharedPreferences? Is a purchase or entitlement validated locally only (then the server fact is `needs_validation`)? Do `onSaveInstanceState`/`ViewModel`/`SavedStateHandle` carry a decision that should be re-checked? Do feature flags from remote config act as authorization?

**Feature abuse and data leakage** (subagent_type: `general`)
Share and export features that leak more than selected, `FileProvider` roots wider than intended, notification previews and app widgets showing protected data on the lock screen, screenshots and recent-apps thumbnails (`FLAG_SECURE`), clipboard writes of secrets, analytics and crash SDKs capturing PII or tokens, `Intent.ACTION_SEND` with private content URIs to any chooser target, autofill and keyboard suggestions on secret fields.

**Chained vulnerabilities and trust boundaries** (subagent_type: `general`)
Intent redirection (a nested `Intent` extra forwarded with the app's identity), `PendingIntent` with mutable flags and an implicit base intent, a deep link that opens a WebView that exposes a bridge, a content URI leak that reaches a provider that grants file access. Confirm each prerequisite; do not assume the downstream step.

**Wildcard** (subagent_type: `general`)
No category. Read boring, experimental, "legacy", and debug-adjacent code: hidden settings screens, QA menus toggled by taps or deep links, `BuildConfig` flags consulted at runtime, flavor-specific source sets, backup/restore helpers, migration code, `else` branches that fall back to insecure behavior, test doubles that ship in the release classpath. What can another app or page do that the UI never does?

**Obvious things** (subagent_type: `general`)
Literal, thorough checks on the release variant: `android:debuggable`, `android:allowBackup` with sensitive data and no rules, `usesCleartextTraffic`, user-CA trust in release `network_security_config`, `exported="true"` components without permissions, `Log.*`/Timber with tokens or PII outside debug-only trees, `setWebContentsDebuggingEnabled`, `addJavascriptInterface` with any remote content, `setJavaScriptEnabled` plus `setAllowFileAccess`/`setAllowUniversalAccessFromFileURLs`, `MODE_WORLD_READABLE`/`WORLD_WRITEABLE`, trust-all `X509TrustManager`/`HostnameVerifier`, `onReceivedSslError` calling `proceed()`, `google-services.json` and keystore files in the repo, `local.properties` or `*.jks` committed, `signingConfigs` with literal passwords, dependency versions with `+`, TODO/FIXME/HACK referencing auth or validation. For every hit, trace the impact before reporting; a flag is not a finding.
