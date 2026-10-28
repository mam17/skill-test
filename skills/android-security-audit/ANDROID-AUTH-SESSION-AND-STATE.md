# Authentication, Session, and State Hunting

#### When to use this file

Reach for this file when the app logs users in, holds sessions or tokens, offers biometric or device-credential gates, handles OTP or PKCE, uses `AccountManager` or Credential Manager, or carries security decisions across process death, configuration change, logout, and account switch. The attacker varies: A5 (physical access to a gate), A2 (another app reaching a session surface), A1 (token on the wire, covered mainly in `ANDROID-NETWORK-AND-CRYPTO.md`). Use `ANDROID-DEEPLINKS-AND-WEBVIEW.md` for OAuth-callback interception and `ANDROID-STORAGE-AND-SECRETS.md` for token storage.

## Core discipline (include in every agent prompt for this domain)

```
- Name the attacker and the gated asset. A local biometric or PIN gate protects against A5 (someone holding the device); defeating it with a debugger on your own device is A6 and usually hardening.
- Client-side gates and client-held state are advisory unless the server enforces the same decision. If the real enforcement is server-side and not in source, the decisive fact is needs_validation; do not confirm a client-only bypass as a server bypass.
- A biometric prompt only protects what it is cryptographically bound to. A BiometricPrompt whose success merely sets a boolean, versus one that unlocks a Keystore key via a CryptoObject, are different controls.
- Session lifecycle is a contract: check login, refresh, logout, account switch, and process death. State that outlives logout, or leaks across accounts, is the finding.
- Framework and API-level behavior matters: SavedStateHandle/onSaveInstanceState persistence, Credential Manager vs legacy flows, autofill, and keyguard APIs across minSdk..targetSdk.
- Use confirmed for a source-visible local gate/lifecycle failure plus a bounded local observation. Use needs_validation when server-side enforcement, the IdP, or OEM biometric behavior is decisive and unobservable.
```

## Local-gate attack classes (subagent_type: `general`)

**Biometric and device-credential binding gaps**
A biometric or device-credential prompt gates a sensitive action but its success is not bound to a cryptographic operation: it sets a flag, returns a boolean, or unlocks data without a Keystore `CryptoObject`, so the gate can be satisfied by reaching the success path another way (a deep link into the post-auth screen, a restored state, a skipped step). Check what the prompt actually unlocks and whether the protected asset depends on it cryptographically. The attacker is A5 unless the only way to bypass is A6.

**PIN/passcode and retry-counter logic**
A local PIN or passcode check compares in a way that leaks, stores the PIN recoverably, or keeps the failed-attempt counter and lockout only in memory or in a client-writable store, so it resets on restart or process death. Check the comparison, the counter's durability, and whether lockout is enforced where it matters.

**Post-auth screen reachability**
A screen behind a login or gate is reachable directly by deep link, exported activity, or restored back-stack without re-establishing the session. Cross-reference `ANDROID-COMPONENTS-AND-IPC.md` and `ANDROID-DEEPLINKS-AND-WEBVIEW.md`; the finding here is that the auth state is not re-checked on entry.

## Token and session-lifecycle attack classes (subagent_type: `general`)

**Token exchange and PKCE/state binding**
The OAuth/OIDC flow omits PKCE on a public client, fails to generate and verify `state`/`nonce`, or accepts a token/code not bound to the request, enabling injection or interception (pair with the callback-hijack class in `ANDROID-DEEPLINKS-AND-WEBVIEW.md`). Check code generation, storage bound to the session, and verification on return.

**Logout and revocation completeness**
Logout clears the UI but leaves tokens, cookies (`CookieManager`, WebView storage), cached responses, databases, in-memory singletons, background `WorkManager` jobs, or widgets holding the prior session, so data appears under the next user or after "logout". Check every store the session touched and every background task. This is a frequent real Android finding.

**Account-switch and multi-user leakage**
Switching accounts (or device multi-user/work-profile) leaves cached data, keys, notifications, or pending operations from the previous account visible to the new one. Check cache keys and file paths for account scoping and whether per-account state is cleared on switch.

## State-carrying attack classes (subagent_type: `general`)

**Saved-state and process-death decision replay**
A security decision (authenticated flag, entitlement, verification step, amount, recipient) is carried in `onSaveInstanceState`, `SavedStateHandle`, a `ViewModel`, or a persisted draft and restored without re-validation, letting a user skip a step or restore a stale decision via process death or configuration change. Check what the restore path trusts versus what it re-checks.

**Client-authoritative entitlement**
A purchase, subscription, role, or feature entitlement is decided locally (a flag, a decoded receipt, a remote-config value treated as authorization) without server verification. If the server is authoritative but not in source, record the server fact as `needs_validation`; if source shows the server trusting the client, it is a finding.

## Universal moves (apply across the above)

- Map the full session lifecycle: where the token is obtained, stored, refreshed, used, and cleared; every gate and what it is bound to; every place a security decision is persisted and restored.
- For each candidate, name the attacker and the gated asset, and trace one path that reaches the asset without satisfying the intended control.
- Validate on the emulator with dummy accounts: observe logout leaving a dummy token behind, a gate's success path reached without the prompt, or a restored decision. Do not use real credentials or defeat gates on a device you control as "proof".

## Validation rules (apply before reporting ANY finding here)

1. Name the attacker (A5 physical, A2 app, A1 network), the gated asset, the control that should bind it, and the path that reaches the asset without the control.
2. Distinguish a client-only gate from server enforcement. A client-side bypass with authoritative server enforcement elsewhere is hardening or `needs_validation`, not a confirmed auth bypass.
3. For biometric findings, cite whether success is cryptographically bound (CryptoObject/Keystore) or merely a flag. For logout, name each store or task that retains the session.
4. Keep local checks to dummy accounts and the app's own behavior. Stop at the first boundary result; never defeat a gate on your own device as the finding (A6).
5. Return `confirmed` for a source-visible local failure plus bounded local evidence. Return `needs_validation` when server-side enforcement, the IdP, or OEM biometric behavior is decisive and unobservable.
