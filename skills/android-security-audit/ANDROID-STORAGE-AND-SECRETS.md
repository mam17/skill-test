# Storage and Secrets Hunting

#### When to use this file

Reach for this file when the app persists data in SharedPreferences, DataStore, Room/SQLite, files, or the Keystore, writes logs, uses the clipboard, allows backup, or embeds secrets. The realistic attacker is tier A2/A3 (another app, possibly with storage access), A5 (physical access, backup extraction, lock-screen surfaces), or A1 for secrets that reach the wire. Use `ANDROID-NETWORK-AND-CRYPTO.md` for crypto primitives on the wire and `ANDROID-AUTH-SESSION-AND-STATE.md` for token lifecycle across logout.

## Core discipline (include in every agent prompt for this domain)

```
- Name the lower-trust reader: another app (A2/A3), the backup/ADB extraction path (A5), a lock-screen surface (A5), or a log/analytics consumer. Data in the app's own private storage, readable only by the same app, is not a finding without such a reader.
- Private app storage (filesDir, internal databases, MODE_PRIVATE) is isolated by the OS. A finding requires a path out: allowBackup, a world-readable/writeable mode, an exported provider, external storage, a FileProvider, or run-as on a debuggable release.
- EncryptedSharedPreferences/Keystore is only as strong as the key parameters. Check user-authentication binding, setUserAuthenticationRequired, StrongBox/hardware backing where required, and whether the key survives credential changes.
- A hardcoded string is a secret only if it authorizes something beyond what a client may hold. Public client IDs and API keys with server-side restriction are not secrets; a signing key, server token, or symmetric key is.
- Framework defaults depend on targetSdk and OS version: scoped storage, allowBackup default, auto-backup inclusion, clipboard access restrictions. Apply the shipped targetSdk and minSdk.
- Use confirmed for a source-visible storage path plus a bounded local observation. Use needs_validation when backup inclusion, OEM keystore behavior, or device-policy facts are required but unobservable.
```

## Local-storage exposure attack classes (subagent_type: `general`)

**World-accessible and external storage**
Sensitive data is written with `MODE_WORLD_READABLE`/`MODE_WORLD_WRITEABLE` (legacy), to shared/external storage, or to a path another app can read with storage permission. Check `getExternalFilesDir`/`getExternalStorageDirectory` writes of private data, media-store entries, and cache directories exposed through a provider. State the reader and what the data authorizes.

**Backup and extraction exposure**
`android:allowBackup="true"` (or the default) with no `fullBackupContent`/`dataExtractionRules` excludes, so tokens, keys, or personal data leave the device through cloud or adb backup (tier A5). Check the backup rules for the release variant and whether sensitive files or preferences are excluded. On a debuggable release build, note `run-as` access, but confirm the release variant is actually debuggable before reporting.

**Provider-backed file disclosure**
A `FileProvider` or custom provider exposes a root wider than intended, or `openFile` resolves attacker-influenced names into private files (traversal). Check `filepaths.xml` roots, `grantUriPermissions`, and path normalization. Cross-reference `ANDROID-COMPONENTS-AND-IPC.md#Provider exposure and argument trust`.

## Keystore and secret-handling attack classes (subagent_type: `general`)

**Keystore binding mismatch**
A key that should require user authentication, or should be invalidated on biometric/credential change, is created without `setUserAuthenticationRequired`, without `setInvalidatedByBiometricEnrollment`, or with an unbounded auth validity window, so a protected secret is usable without the intended gate. Check the `KeyGenParameterSpec` against the use case (payments, local vault, at-rest tokens).

**At-rest secret without a real boundary, or with a weak one**
Tokens or personal data stored in plaintext SharedPreferences/DataStore/files reachable by a lower-trust reader (see classes above), or "encrypted" with a key derived from a device identifier, package name, or a hardcoded constant rather than the Keystore. Check the key source and the reader. Plaintext in strictly private storage with no reader is hardening, not a finding.

**Hardcoded and embedded secrets**
A server-side secret, signing key, symmetric key, or privileged token embedded in `strings.xml`, `BuildConfig`, resources, assets, or native strings, or committed files (`*.jks`, `google-services.json` with sensitive fields, `local.properties`). For each, state what it authorizes and who can extract it (any installed build, tier A2+). A public, restricted client identifier is not a finding.

## Leakage-surface attack classes (subagent_type: `general`)

**Logs and analytics**
`Log.*`, Timber, `printStackTrace`, or an analytics/crash SDK records tokens, credentials, PII, or full requests/responses. Logcat is readable by the same app and, on older APIs or with the right permission, by others; crash SDKs ship data off-device. Check that logging of secrets is confined to a debug-only tree and that SDK payloads are scrubbed. Confirm the release variant.

**Lock-screen and task-switcher surfaces**
Notifications, app widgets, or the recent-apps snapshot reveal protected content on a locked device (tier A5). Check notification visibility on secret content, `FLAG_SECURE` on sensitive screens, and `setRecentsScreenshotEnabled`/secure-flag handling. A finding requires actually sensitive content on the surface.

**Clipboard and autofill**
The app copies secrets to the clipboard (readable by other apps on older APIs, or via a clipboard-access permission), marks sensitive clipboard data without `EXTRA_IS_SENSITIVE`, or exposes secret fields to autofill/keyboard suggestions. State the reader and the secret.

## Universal moves (apply across the above)

- Inventory every persistence site (preferences, DataStore, databases, files, Keystore) and every leakage surface (logs, analytics, clipboard, notifications, widgets, backup). For each, record who writes, who can read, and when it is cleared.
- For each candidate, name the lower-trust reader and trace the data from write to that reader. Confirm the release-variant backup rules, storage mode, and logging tree.
- Validate on the emulator by observing the app's own write with dummy data (a dummy token in a preference file, a dummy value in logcat). Do not read other apps' data.

## Validation rules (apply before reporting ANY finding here)

1. Name the lower-trust reader (A2/A3 app, A5 backup/physical, log/analytics consumer), the stored item, what it authorizes, and the path that exposes it.
2. Confirm there is a real path out of private storage (backup, world mode, provider, external storage, debuggable release). Private-only storage with no reader is hardening.
3. For Keystore findings, cite the missing binding parameter and the use case that requires it. For hardcoded secrets, state what the secret authorizes beyond a client's legitimate holding.
4. Confirm the leakage or relaxation is present in the release variant, not debug-only. Keep local checks to dummy data.
5. Return `confirmed` only with a source path plus a lower-trust reader. Return `needs_validation` when backup inclusion, OEM keystore, or device-policy facts are decisive and unobservable.
