# Android Components and IPC Hunting

#### When to use this file

Reach for this file when the app exposes activities, services, broadcast receivers, or content providers to other apps, builds or forwards intents and `PendingIntent`s, binds services across apps, or defines custom permissions. The realistic attacker is usually tier A2 (a co-installed app with no permissions) or A3 (one with a user-granted permission). Use `ANDROID-DEEPLINKS-AND-WEBVIEW.md` for intent-filter URIs and WebView bridges, and `ANDROID-STORAGE-AND-SECRETS.md` for provider-backed files.

## Core discipline (include in every agent prompt for this domain)

```
- Establish the attacker tier (A2 no-permission app, A3 permissioned app) and its real capability. A component the app invokes on itself is not a boundary.
- Exported state and permission are decided by the MERGED manifest, including library-contributed components and activity-alias entries, not by the source manifest alone. A conclusion from source manifests only is needs_validation.
- A package name or claimed sender is not caller authentication. Use the Binder calling UID, a verified signing certificate, or a signature-level permission.
- A component must authorize each operation and final resource after parsing, not only on entry. A trusted-looking caller does not make attacker-controlled extras trusted.
- Framework defaults depend on targetSdk (exported default for components with intent filters; PendingIntent mutability; implicit-intent delivery restrictions). Apply the shipped targetSdk.
- Use confirmed for source evidence plus a bounded local observation. Use needs_validation when the merged manifest, signing certificate, or a device fact is required but unobservable.
```

## Exported-component attack classes (subagent_type: `general`)

**Exported component overreach**
An activity, service, or receiver reachable by another app performs an operation meant for the app itself: changes state, imports or exports data, completes a flow, or discloses information. Check the merged-manifest `exported` and `permission` attributes, the `protectionLevel` of any gating permission (a `normal` permission is not a real barrier against a co-installed app), and whether the handler re-checks authorization for the action and the specific resource. Reaching a component is only a finding when a sensitive operation or disclosure sits behind it.

**Caller-identity and signature-check gaps**
A component trusts `getCallingPackage()`, a package name in an extra, or the presence of any caller rather than verifying the caller's signing certificate or a signature-level permission. Check for package-name string matching that another app can satisfy by declaring the same name is not possible, but spoofing claimed-sender fields in extras is; verify that identity comes from `Binder.getCallingUid()`/`PackageManager` signature APIs, and that the check runs before the sensitive operation.

**Implicit-intent and broadcast exposure**
The app sends a sensitive action or data through an implicit intent or an unprotected broadcast that any app can receive, or registers a receiver for an action any app can send. Check implicit `sendBroadcast` of private data, ordered broadcasts whose result another app can alter, dynamically registered receivers without a permission, and `startActivity`/`startService` with an implicit intent carrying private content URIs. For exported receivers, check that the handler validates the action and extras.

## PendingIntent and intent-forwarding attack classes (subagent_type: `general`)

**Mutable PendingIntent and implicit base intent**
A `PendingIntent` is created mutable (or without `FLAG_IMMUTABLE` where the targetSdk still allows it) over an implicit base intent, letting another app fill in the component or data and have it sent with the app's identity and permissions. Check every `PendingIntent.getActivity/getBroadcast/getService` site, the flags, and whether the base intent is explicit and fully specified.

**Intent redirection (confused deputy)**
A component extracts a nested `Intent` (or a URI parsed with `Intent.parseUri`) from an attacker-reachable extra and forwards it with `startActivity`, `startService`, or `setResult` using the app's own authority, reaching internal, non-exported components or granting URI permissions. Check that any forwarded intent is validated for component, action, and flags, and that `FLAG_GRANT_*_URI_PERMISSION` is never propagated from untrusted input.

**Task and launch-mode confusion**
`taskAffinity`, `singleTask`/`singleInstance`, or `allowTaskReparenting` let another app place or observe an activity in a shared task (UI redress, back-stack confusion). Check affinities on sensitive activities and whether a strandhogg-style overlay or reparenting changes what the user is actually interacting with. This is a finding only when a sensitive action can complete in the confused state.

## Content-provider and bound-service attack classes (subagent_type: `general`)

**Provider exposure and argument trust**
An exported content provider (or one with weak `protectionLevel`) exposes `query`/`insert`/`update`/`delete`/`openFile` without binding the operation to an authorized caller and resource. Check selection and projection built from caller input (SQL injection into `rawQuery`/selection), `openFile` path handling (traversal into private files via `..` or encoded separators), `grantUriPermissions` scope, and `path-permission` coverage of every URI the provider serves.

**Bound-service command surface**
An exported bound service exposes Messenger or AIDL methods that perform privileged operations without per-call authorization of the caller UID and the requested resource. Check the message/transaction handlers, whether each method re-checks authority, and whether returned data exceeds the caller's authority.

## Universal moves (apply across the above)

- Enumerate every component from the merged manifest, including `activity-alias` and library-contributed entries, and record exported state, permission, `protectionLevel`, intent filters, `taskAffinity`, `launchMode`, and `grantUriPermissions`.
- For each exported, sensitive component, state the attacker tier and trace one dummy input from entry to the operation or disclosure, noting the authorization check and the resource it binds to.
- Confirm exported state with `ANDROID-DYNAMIC-CHECKS.md` (merged manifest via offline build, `apkanalyzer`, or `dumpsys package`) before assigning impact.

## Validation rules (apply before reporting ANY finding here)

1. Name the attacker tier, the component and entry, the accepted extras or arguments, the missing or wrong authorization, and the unauthorized operation or disclosure.
2. Base exported state and permission on the merged manifest for the shipped variant. If only source manifests were available, report `needs_validation` with that exact blocker.
3. For caller-identity findings, cite the spoofable field and the signature or UID check that is missing. For PendingIntent and redirection, cite the mutable/implicit construction and the internal sink reached.
4. Keep the local check to one dummy invocation observed on the emulator; confirm the tier from the manifest, not from adb reachability. Stop at the first boundary result.
5. Return `confirmed` only with a complete source and local chain. Return `needs_validation` with the exact merged-manifest, signing, or device fact required.
