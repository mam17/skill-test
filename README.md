# android skill-test suite

Two coding-agent skills for testing Android/Kotlin apps:

- **`skills/android-security-audit/`** — security audit (this file's main subject, below).
- **`skills/android-flow-ux-review/`** — functional flow testing, UI/UX and accessibility review, and API-first test planning (contract tests against a mock before repository/usecase/ViewModel tests, then UI flows). Same evidence discipline as the security skill, user-impact severity model (`blocker/major/minor/polish`), source-verified vs device-verified vs needs-device-check status, and its own report artifacts (`REPORT.md`, `FLOWS.md`, `findings.json`, `API-TEST-PLAN.md`).

# android-security-audit

A coding-agent skill that turns your agent into an Android security auditor for Kotlin (and Java) apps. It is an Android-specific adaptation of Cloudflare's [`security-audit`](https://github.com/cloudflare/security-audit-skill) skill (MIT), keeping that skill's isolated-agent orchestration, coverage ledger, adversarial validation, and schema-checked output, while replacing the attack-class content with Android trust boundaries.

## What changed from the upstream skill

- **Android attacker tiers (A1-A6)** instead of generic principals: remote network, co-installed app (no permission / with permission), malicious web content, physical access, and device owner. The tier is part of every finding and drives severity.
- **Android companions** replace the web/cloud/native companions:
  - `ANDROID-COMPONENTS-AND-IPC.md` — exported components, intents, `PendingIntent`, providers, bound services.
  - `ANDROID-DEEPLINKS-AND-WEBVIEW.md` — deep links, App Links, OAuth callbacks, WebView bridges.
  - `ANDROID-STORAGE-AND-SECRETS.md` — preferences, databases, files, backup, Keystore, logs, leakage surfaces.
  - `ANDROID-NETWORK-AND-CRYPTO.md` — TLS trust, network security config, pinning, response handling, crypto.
  - `ANDROID-AUTH-SESSION-AND-STATE.md` — login, session lifecycle, biometric gates, saved-state replay.
  - `ANDROID-KOTLIN-RUNTIME-AND-INPUT.md` — deserialization, path/zip, injection, reflection, concurrency, JNI handoff.
  - `ANDROID-SUPPLY-CHAIN-AND-BUILD.md` — Gradle, manifest merger, flavors, signing, R8, bundled SDKs.
- **`ANDROID-DYNAMIC-CHECKS.md`** defines bounded, observational checks on a fresh emulator with a debug build and dummy data, and states what each check does and does not prove (notably that `adb` reachability does not by itself prove a no-permission-app boundary).
- **`MASVS-MAPPING.md`** tags findings to OWASP MASVS groups for reporting.
- **Execution safety** is rewritten for Android: offline Gradle, no real signing keys or credentials, emulator-only app execution against a local or owner-designated non-production backend, and resilience (tier A6) treated as hardening unless the threat model says otherwise.

The validators and report schema are carried over from the upstream skill unchanged in behavior:

- `report-schema.json` — the `confirmed` / `needs_validation` / `rejected` record schema.
- `validate-findings.cjs` — zero-dependency validator for `findings.json`.
- `validate-coverage-ledger.cjs` — zero-dependency validator for `coverage-ledger.json`.
- `*.test.cjs` — the upstream tests, passing on this fork.

The only code change to the validators is a Windows compatibility fallback in the file reader: upstream refuses to run where `O_NOFOLLOW` is unavailable (which includes Windows Node builds), so this fork falls back to an `lstat` symlink-and-regular-file check before opening. The behavior is identical on POSIX. See `validate-findings.cjs` / `validate-coverage-ledger.cjs` `openInputDescriptor`.

## Files

| File | Purpose |
|------|---------|
| `skills/android-security-audit/SKILL.md` | Modes, attacker tiers, execution safety, setup, workflow, anti-patterns |
| `skills/android-security-audit/RECONNAISSANCE.md` | Phase 1 recon prompts and the Android coverage ledger |
| `skills/android-security-audit/HUNTING.md` | Phase 2 orchestration, hunter prompt, and coverage critics |
| `skills/android-security-audit/ATTACK-CLASSES.md` | Ordinary classes (Android framing) and companion routing |
| `skills/android-security-audit/ANDROID-*.md` | The seven Android domain companions |
| `skills/android-security-audit/ANDROID-DYNAMIC-CHECKS.md` | Bounded emulator/adb/harness checks and what they prove |
| `skills/android-security-audit/MASVS-MAPPING.md` | MASVS group tagging for the report |
| `skills/android-security-audit/VALIDATION-AND-REPORTING.md` | Phases 3-6: validation, output, verification, reporting |
| `skills/android-security-audit/report-schema.json` | Record schema (from upstream) |
| `skills/android-security-audit/validate-*.cjs` | Validators (from upstream, Windows-compatible) |
| `skills/android-security-audit/validate-*.test.cjs` | Validator tests (from upstream) |

## Install

Copy `skills/android-security-audit/` into your agent's skills directory (for example `~/.claude/skills/android-security-audit/`), or point your Skills tooling at this repo.

## Use

Open your coding agent in the Android project you want to review, then ask:

```
security audit this Android app
```
```
review the deep link and WebView handling in :feature:profile
```
```
do a full Android security review, output to ~/audits/myapp
```

A direct audit or pen-test request runs full audit mode and writes report artifacts. A focused question or a single-module review runs in guidance mode and returns results inline.

## Requirements

- A coding agent with tool use and parallel sub-agents.
- Node.js for the validators.
- For dynamic checks: Android SDK command-line tools, an emulator image, `adb`, `apkanalyzer`/`aapt2`, and a Gradle wrapper with a populated offline cache. Without a sandbox for target-controlled builds and app execution, the workflow keeps blocked leads as `needs_validation` instead of running target code.

## Run the validator tests

```bash
cd skills/android-security-audit
node --test validate-findings.test.cjs validate-coverage-ledger.test.cjs
```

## Scope and ethics

This is a defensive, source-first workflow for auditing code you are authorized to review. It confirms vulnerabilities by observing the target's own behavior with dummy data on an isolated emulator; it does not build exploit chains, attack other apps or real users, or defeat protections on devices it controls. Device-owner (tier A6) resistance is treated as hardening unless the product's threat model makes it a boundary.

## License

MIT, inherited from the upstream Cloudflare skill. See [LICENSE](LICENSE).
