# Network and Cryptography Hunting

#### When to use this file

Reach for this file when the app makes network calls (Retrofit/OkHttp/Ktor, raw sockets, WebSocket), configures TLS trust or pinning, parses server or push responses, or uses cryptographic primitives for confidentiality, integrity, or authentication. The realistic attacker is tier A1 (on-path MITM, rogue Wi-Fi, malicious or compromised server, malicious push). Use `ANDROID-STORAGE-AND-SECRETS.md` for keys at rest and `ANDROID-AUTH-SESSION-AND-STATE.md` for token exchange and session logic.

## Core discipline (include in every agent prompt for this domain)

```
- Establish the tier-A1 attacker: on-path network, hostile Wi-Fi, a compromised or malicious endpoint the app talks to, or a malicious push payload. Traffic to the app's own first-party backend over correctly validated TLS is not, by itself, attacker-controlled.
- TLS trust is decided by the release network_security_config plus any custom TrustManager/HostnameVerifier/pinner in code. A debug-only relaxation (debug-overrides, user CA trust in a debug config) is not a finding.
- Pinning and user-CA rejection are controls against A1; their absence is a finding only when authenticated or sensitive traffic is exposed and the product's threat model expects protection beyond platform TLS. Absence of pinning alone, with valid platform TLS, is usually hardening, not a vulnerability.
- A response body is untrusted input. Deserialization, polymorphic type handling, and code paths that trust server-chosen types, URLs, or flags belong here and in ANDROID-KOTLIN-RUNTIME-AND-INPUT.md.
- Crypto findings need misuse plus a reachable consequence: forgeable integrity, recoverable confidentiality, or a bypassed check. A primitive choice with no reachable attacker is hardening.
- Use confirmed for a source-visible trust or crypto failure plus a bounded local observation (the app's own behavior against a local mock). Use needs_validation when the real server's TLS, the deployed config, or a CA-store fact is decisive and unobservable.
```

## TLS-trust attack classes (subagent_type: `general`)

**Trust-all and weak verification**
A custom `X509TrustManager` whose `checkServerTrusted` does nothing, a `HostnameVerifier` returning `true`, an OkHttp client built without certificate validation, or a Ktor/engine config disabling verification, present in the release variant. These fully defeat TLS for tier A1. Confirm the code path is used by the release client, not a test-only or debug-only instance.

**Network-security-config weaknesses**
The release `network_security_config` sets `cleartextTrafficPermitted="true"` for first-party domains, trusts user-added CAs in the release `base-config`/`domain-config`, or applies a relaxed `debug-overrides` that is not actually confined to debug. Check `usesCleartextTraffic`, per-domain configs, and which config the shipped variant references. Cleartext to a first-party authenticated host is a finding; cleartext to an intended plaintext public resource is not.

**Pinning gaps and failures**
Where the threat model expects pinning (payments, credentials, anti-MITM requirements), check whether a `CertificatePinner` or config `pin-set` exists, is applied to the right client and domains, fails closed on mismatch, and is not silently bypassed by a fallback client. A pin that is expired, applied to the wrong client, or caught-and-ignored is a gap. Framing: report the exposed authenticated traffic, not "pinning can be bypassed on a rooted device" (tier A6).

## Response-handling attack classes (subagent_type: `general`)

**Server-trusted response data**
The app trusts a server-chosen URL, redirect, filename, type discriminator, HTML snippet, deep link, or executable path from a response because "it is our backend", then reaches a dangerous sink (WebView load, file write, intent, dynamic code, SQL). Under tier A1 the endpoint may be malicious or compromised. Check deserialization of polymorphic types, `@Json`/`@SerialName` type handling, and values that flow into sinks covered by `ANDROID-KOTLIN-RUNTIME-AND-INPUT.md`.

**Push and realtime payload trust**
An FCM `onMessageReceived`, WebSocket, or SSE handler acts on payload fields (navigation, state change, notification content, deep link) without validating them against the session and expected shape. A push payload originates outside the app and may be attacker-influenced. Check what the handler does with each field and whether it authenticates the payload.

## Cryptography attack classes (subagent_type: `general`)

**Confidentiality misuse**
`Cipher.getInstance("AES")` or `"AES/ECB"`, a static or reused IV/nonce with CTR/GCM, a key derived from a device identifier or constant, or unauthenticated encryption where integrity matters. Trace to the protected data and the attacker who benefits. Report the recoverable or malleable data, not the primitive name alone.

**Integrity and authentication misuse**
Non-constant-time comparison of MACs or tokens, missing MAC/signature verification before use, local JWT or receipt decoding treated as verification, or a signature check that can be skipped by a null/empty value. Trace to the forgeable decision (entitlement, auth, integrity gate).

**Randomness and key-derivation misuse**
`java.util.Random`, `Math.random`, or time-seeded generators for tokens, keys, nonces, or IVs; weak PBKDF iteration counts or missing salt for password-derived keys. Trace to the security-critical value that becomes predictable.

## Universal moves (apply across the above)

- Inventory every HTTP/socket client, its interceptors, its TrustManager/HostnameVerifier/pinner, and the config it uses, per variant. Identify which client the release app actually uses.
- For a crypto finding, trace the primitive to the protected asset and the attacker; for a trust finding, trace to the authenticated or sensitive traffic exposed.
- Validate by observing the app's own TLS behavior against a local mock server on the emulator with the debug build (does it accept an untrusted cert, does it send cleartext), and crypto with a JVM harness around the function. Do not intercept real endpoints or real user traffic.

## Validation rules (apply before reporting ANY finding here)

1. Name the tier-A1 attacker, the exposed traffic or data, the missing or broken control, and the concrete consequence (credential capture, response forgery, recoverable confidentiality).
2. Confirm the weakness is in the release variant and used by the real client, not a debug or test instance.
3. For pinning, frame the finding as exposed authenticated traffic against the product's stated threat model, never as device-owner bypass (A6). For crypto, cite misuse plus a reachable consequence.
4. Keep local checks to the app's own behavior against a local mock, or a function harness with dummy keys. Stop at the first boundary result.
5. Return `confirmed` with a source-visible trust/crypto failure plus bounded local evidence. Return `needs_validation` when the real server's TLS, deployed config, or CA-store fact is decisive and unobservable.
