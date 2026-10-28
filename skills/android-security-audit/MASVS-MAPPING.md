# MASVS Mapping

#### When to use this file

Phase 6 uses this file to tag each confirmed finding and each hardening note with an OWASP MASVS (Mobile Application Security Verification Standard) control group, so owners can map the audit to a recognized Android/mobile baseline. This is a reporting aid, not an evidence source: a MASVS tag never substitutes for the boundary, trace, and observed result a finding requires. Use the MASVS group tags below; cite a specific MASTG test number only when you have verified it against the current MASTG.

## MASVS groups and this skill's companions

| MASVS group | Scope | Primary companion(s) |
|---|---|---|
| **MASVS-STORAGE** | Sensitive data at rest, logs, backups, clipboard, leakage surfaces | `ANDROID-STORAGE-AND-SECRETS.md` |
| **MASVS-CRYPTO** | Cryptographic primitives, keys, randomness, Keystore use | `ANDROID-NETWORK-AND-CRYPTO.md`, `ANDROID-STORAGE-AND-SECRETS.md` (keys at rest) |
| **MASVS-AUTH** | Authentication, session lifecycle, biometric and credential gates, entitlement | `ANDROID-AUTH-SESSION-AND-STATE.md` |
| **MASVS-NETWORK** | Transport security, TLS trust, certificate validation and pinning | `ANDROID-NETWORK-AND-CRYPTO.md` |
| **MASVS-PLATFORM** | IPC, components, deep links, WebView, intents, providers, clipboard/notification surfaces | `ANDROID-COMPONENTS-AND-IPC.md`, `ANDROID-DEEPLINKS-AND-WEBVIEW.md` |
| **MASVS-CODE** | Input handling, deserialization, dependencies, build config, platform API misuse | `ANDROID-KOTLIN-RUNTIME-AND-INPUT.md`, `ANDROID-SUPPLY-CHAIN-AND-BUILD.md` |
| **MASVS-RESILIENCE** | Anti-tampering, root/debugger/emulator detection, obfuscation, integrity | Hardening only in this skill unless the threat model makes tier A6 a boundary |

## How to tag

- A confirmed finding gets the group that names the broken control. A finding may touch two groups (for example, a session token in plaintext preferences is MASVS-STORAGE for the at-rest exposure and MASVS-AUTH if logout fails to clear it); list the primary group first.
- A hardening note gets the group it would strengthen. Most tier-A6 resilience gaps (missing root or tamper detection, weak obfuscation) are MASVS-RESILIENCE hardening, not findings, unless the product's threat model makes device-owner resistance a requirement (DRM, anti-fraud, payment certification) and source shows the gap defeating that requirement.
- In `REPORT.md`, group the confirmed-findings table and the hardening section by MASVS group so the owner sees coverage across the standard.

## Important boundary

MASVS-RESILIENCE controls protect against the device owner or someone who fully controls the device (tier A6). Under this skill's default threat model, their absence is hardening, not a vulnerability. Promote a MASVS-RESILIENCE item to a finding only when: the product explicitly commits to resisting A6 (stated DRM, anti-fraud, or certification requirement), and source shows the control missing or defeatable in a way that breaks that specific commitment. Record the commitment you relied on in the finding's conditions.

## Do not over-claim

The MASVS tag describes which control area a finding belongs to; it does not grade the app against the full standard. Unless the run's scope and coverage actually exercised every MASVS group, `REPORT.md` must not present the mapping as a MASVS compliance verdict. State the groups the run covered and the ones left `deferred` or `out_of_scope`, consistent with the coverage ledger.
