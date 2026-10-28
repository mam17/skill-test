---
name: android-security-audit
description: Security guidance and vulnerability review for Android applications and libraries written in Kotlin (and Java). Use for Android security questions, focused reviews of a feature or module, mobile pen-test planning, or full security audits with report artifacts. Run the complete six-phase workflow only for explicit audit, pen-test, full/comprehensive review, or report requests.
---

# Android Security Audit

Find vulnerabilities in an Android codebase that violate a real trust boundary, then give owners the source evidence, a safe emulator reproduction, a priority, and the smallest effective fix. This is a defensive, source-first workflow adapted from Cloudflare's `security-audit` skill for Android and Kotlin targets. A candidate without a concrete attacker tier, a crossed boundary, and an observable result is not a confirmed finding.

## Operating modes

This skill is guidance by default. Loading it does not authorize the complete audit workflow or file creation.

- **Guidance mode**: for Android security questions, review of one feature, module, component, or pull request, triage of a scanner result, or methodology help, use only the relevant parts of this skill. Do not run all six phases, create an output directory, or write audit artifacts. You may launch focused agents; they return results to the current task.
- **Full audit mode**: when the user explicitly asks to audit or pen-test the app, asks for a full, comprehensive, or end-to-end security review, or requests report artifacts. Run all six phases and write the files defined below.

If the request could mean either mode, ask one focused question before creating files or starting the complete workflow.

## Platform terminology

- **Parent** is the agent that coordinates the run and owns shared state.
- **Task tool** is the platform's delegation or sub-agent mechanism.
- **`research` agent** is a delegated agent for focused source exploration and factual verification. It never runs target code.
- **`general` agent** is a delegated agent for broad investigation and bounded local execution (Gradle offline tasks, unit tests, emulator checks).
- **`subagent_type:`** in a heading names which role runs that work.

## Android attacker tiers

Every candidate names one starting capability from this list. The tier is part of the boundary claim and drives severity.

| Tier | Attacker | Typical entry |
|---|---|---|
| **A1** | Remote network attacker: on-path MITM, rogue Wi-Fi, malicious or compromised third-party server, malicious push payload | TLS trust, response parsing, FCM `onMessageReceived`, WebSocket |
| **A2** | Malicious co-installed app with **no** permissions | Exported components, implicit intents, deep links, content URIs, broadcasts, pending intents, task affinity |
| **A3** | Malicious co-installed app with a user-granted permission (state which: storage, `QUERY_ALL_PACKAGES`, notification listener, accessibility, etc.) | External storage, notification content, accessibility tree, clipboard on older APIs |
| **A4** | Malicious web content: a page in the app's WebView, a Custom Tab, or a browser page invoking a deep link | WebView bridges, `shouldOverrideUrlLoading`, App Links, OAuth redirect |
| **A5** | Physical access: locked, or briefly unlocked; lost device with backup extraction | Lock screen bypass, `FLAG_SECURE`, backup rules, Keystore user-auth binding, screenshots, notification previews |
| **A6** | The device owner: rooted device, debugger, instrumentation, repackaging | Only a boundary when the product threat model states it (DRM, anti-fraud, payment certification). Otherwise resilience is hardening, not a vulnerability |

The backend is out of scope unless its code is in the repository. Client-side enforcement of a server-authoritative decision is a finding only when source shows the server trusting the client; otherwise record it as `needs_validation` with the exact server fact an owner must observe.

## Universal execution safety

These rules apply in both modes. Source inspection is read-only. Target-controlled code includes Gradle build scripts, `buildSrc`, included builds, annotation processors, unit tests, instrumentation tests, and the app itself.

- **Build and unit tests** run only with `--offline`, from a clean environment that holds no real signing keys, API tokens, or developer credentials, inside the strongest sandbox available. Never let a build fetch dependencies or publish anything.
- **App execution** happens only on a fresh emulator (AVD) or dedicated test device with no personal Google account, using a **debug build of the target** and dummy accounts. The app's backend is a local mock (`MockWebServer`, WireMock, or the repository's own fake) or a non-production environment the owner explicitly designates. Never point the app at production, never use a real user's credentials, never trigger payments, SMS, calls, or push to real recipients.
- **Attacker simulation stays bounded and observational.** Confirm a boundary by observing the target's own behavior with one dummy input: whether a component accepts an external invocation, whether a guard returns the wrong value, whether a dummy token appears in logcat. Stop at the first unauthorized effect. Do not build persistence, data exfiltration, or chained offensive payloads, and do not touch other installed apps' data.
- **Dependencies and SDKs**: use only toolchains already installed (Android SDK, `adb`, `apkanalyzer`, `aapt2`, Gradle wrapper with a populated cache). Do not install new tools during a run.

When the user confirms the repository is their own first-party code that they already build locally, the parent may record `execution_policy: "trusted-first-party-offline"` in `run-metadata.json` and run Gradle offline tasks on the host. Otherwise use `execution_policy: "sandboxed-source-and-local-only"` and treat any build that cannot run inside a sandbox as a `needs_validation` blocker.

If the decisive fact lives in the backend, Play Console, Firebase rules, a device-vendor behavior, or an OEM ROM, report it as `needs_validation` with the exact owner-observed check.

## Full audit setup

Resolve these values before reconnaissance:

- **Skill directory**: the absolute directory containing this `SKILL.md`.
- **Target**: the absolute repository root (the Gradle root project).
- **Repo name**: a stable identifier from the directory or Git remote.
- **Output directory**: a new writable directory outside the target, defaulting to `~/android-security-audit/<repo-name>/run-<N>`. A directory inside the target is allowed only when the user selects it and the parent verifies version control ignores it.
- **Source ref**: the reviewed commit and whether the worktree is dirty.
- **Build variant under review**: the release-like variant the owner ships (for example `prodRelease`). Manifest, `network_security_config`, `BuildConfig`, and R8 facts are read for that variant; a debug-only relaxation is not a finding.
- **Merged manifest**: `app/build/intermediates/merged_manifests/<variant>/AndroidManifest.xml` when an offline build is permitted; otherwise source manifests plus every library manifest the parent can locate in the Gradle cache. Record which was used. A conclusion about exported state from source manifests alone is `needs_validation` when any dependency may contribute components.

### Write isolation

The parent creates and is the only writer of shared run files: `run-metadata.json`, `architecture.md`, `coverage-ledger.json`, `findings.json`, `REPORT.md`, `FINDINGS-DETAIL.md`, `NEEDS-VALIDATION.md`.

Each hunter or verifier receives a unique root under `<output-dir>/agents/<agent-id>/` with separate `scratch/` and `artifacts/`. Agent IDs match `^[a-z0-9][a-z0-9_-]{0,63}$` and are never Windows device names (`con`, `prn`, `aux`, `nul`, `com1`-`com9`, `lpt1`-`lpt9`). Agents and target-controlled processes write only to `scratch/`. `artifacts/` is parent-owned: after the agent finishes, the parent promotes only predeclared, regular, non-symlink files under the recorded size limits from `scratch/` to `artifacts/`, one file at a time, never recursively and never from an archive. Anything that fails a check is discarded; decisive evidence that cannot be promoted stays `needs_validation` with the promotion blocker. Do not use the system temp directory or the host home directory as a fallback.

For a reproduced check, record the exact command, emulator image, dummy input, and only the allowlisted environment variable names with safe values. Never capture the ambient environment, `local.properties`, keystore passwords, or `~/.gradle/gradle.properties`.

Before delegation, write `run-metadata.json` with at least `run_id`, `repo`, `target`, `source_ref`, `variant`, `min_sdk`, `target_sdk`, `profile`, `scope_paths`, `budget` (null if unset), `execution_policy`, `manifest_source` (`merged` or `source-only`), selected companion files, prior-run paths, shared-file owners, and `run_status: "in_progress"`.

## Full audit planning

### Coverage and prior runs

Build a deterministic coverage plan before hunting and update it after every agent result. [RECONNAISSANCE.md](RECONNAISSANCE.md) defines the coverage units and [HUNTING.md](HUNTING.md) defines coverage-critic waves. The parent alone updates the ledger.

If prior runs exist, read every compatible `coverage-ledger.json` and `findings.json` first. Carry an unchanged prior `confirmed` record only when its source, manifest, variant, and `minSdk`/`targetSdk` conditions are unchanged; a changed `targetSdk` alone can change exported defaults, cleartext defaults, and permission behavior, so treat an SDK bump as changed source for every manifest-dependent unit. Prior `needs_validation`, `deferred`, `blocked`, and `out_of_scope` states become current work. A prior `rejected` record suppresses only its unchanged claim. If no prior ledger exists, say so in the final coverage statement.

### Run profiles and scope

Pick `quick`, `standard` (default), or `deep`:

- **`quick`**: one hunter wave, one final critic, one verifier per candidate, ledger units coarsened to surface x boundary x attack class with subsystem `profile/quick/all-in-scope-subsystems`.
- **`standard`**: the workflow as written.
- **`deep`**: split units per Gradle module and lifecycle mode (install, upgrade, logout, account switch, backup/restore, process death), critic waves to a clean pass, separate candidate and final verifiers.

A **scoped run** audits a subset: one module, one feature, one companion domain, or the diff between two refs. Everything else is `out_of_scope`, never `covered`. Profiles change breadth, never the evidence bar.

### Cost budget

When a budget is set, record it as a maximum number of agent invocations, reserve the four reconnaissance calls, the mandatory critics, and at least one verifier before any hunter, and follow the strict gate, ordering, and incomplete-run rules in [HUNTING.md](HUNTING.md). Never exceed a user-set budget silently; set `run_status: "incomplete"` with an exact reason instead.

## Core principles

### Require a boundary and result

For every candidate name: the attacker tier, the accepted input or action, the intended control, the crossed boundary, the affected principal or resource, and the concrete observed result. A missing best practice, an obfuscation gap, a crash from a malformed extra, or a debug-variant relaxation is not a finding by itself.

### Use bounded local evidence

Static analysis establishes the source path. Bounded emulator or unit evidence settles behavior: an existing unit test, a Robolectric or JVM harness around the parser or guard, a one-shot observation of how a component handles a dummy input, or a local mock server. Stop at the wrong return value, the unauthorized dummy record, the leaked dummy token in logcat, or the bridge method reached from a local page. See [ANDROID-DYNAMIC-CHECKS.md](ANDROID-DYNAMIC-CHECKS.md) for what each check does and does not prove.

### Respect source visibility

The shipped manifest, Play App Signing, Play Console settings, backend authorization, Firebase/Firestore rules, OEM behavior, and the user's device policy are real controls or real gaps that source may not show. Do not assume either way. Use `needs_validation` with the exact missing fact and a safe owner-observed check.

### Separate priority from certainty

Only `confirmed` records receive severity. Anchors for Android:

- **critical**: tier A1, A2, or A4 gains code execution in the app process, or reads or forges any user's credentials or session at scale, or a remote page drives a privileged native bridge.
- **high**: tier A2 with no permissions reads the current user's session token or personal data, or performs an authenticated action through an exported component, deep link, or pending intent; an authentication or biometric gate for a protected action is bypassed; TLS trust is fully defeated for authenticated traffic; an OAuth or App Link callback can be captured by another app.
- **medium**: a real boundary violation that needs a user-granted permission, specific user interaction, an older API level still inside `minSdk`, or yields limited data.
- **low**: disclosure of non-secret internals, or an app-only crash another app can trigger once.
- **informational**: a confirmed but minimal observation, mainly a prerequisite inside a larger finding.

`needs_validation` has no severity. Resilience gaps (tier A6) are hardening unless the product threat model makes them boundaries.

### Recommend the smallest effective source fix

Name the invariant and the narrowest Kotlin, manifest, resource, or Gradle change that enforces it at the last trusted decision point, with a regression test (unit, Robolectric, or instrumentation). The audit describes fixes; it does not modify target source.

## Full audit workflow

1. **Reconnaissance**: map modules, variants, merged manifest, principals, entry surfaces, and the initial coverage ledger with [RECONNAISSANCE.md](RECONNAISSANCE.md).
2. **Coverage-led hunting waves**: assign hunters from the ledger with [HUNTING.md](HUNTING.md), [ATTACK-CLASSES.md](ATTACK-CLASSES.md), and the selected `ANDROID-*.md` companions.
3. **Candidate validation**: a fresh verifier tries to disprove every candidate as defined in [VALIDATION-AND-REPORTING.md](VALIDATION-AND-REPORTING.md).
4. **Structured output**: write `findings.json`; validate it with `report-schema.json` and `validate-findings.cjs`; validate the ledger with `validate-coverage-ledger.cjs`.
5. **Independent record verification**: fresh agents verify final records; material replacements get another verifier.
6. **Target-neutral report**: derive `REPORT.md`, `FINDINGS-DETAIL.md`, and `NEEDS-VALIDATION.md`, with the MASVS mapping from [MASVS-MAPPING.md](MASVS-MAPPING.md).

Do not end the run before one of two terminal states: all Phase 6 artifacts written with both validators passing, or `run_status: "incomplete"` recorded with its reason and disclosed in the report.

## Android anti-patterns

1. Reporting missing root, emulator, debugger, or tamper detection, or missing obfuscation, as a vulnerability when the threat model does not make tier A6 a boundary.
2. Reporting `android:debuggable`, cleartext, `debug-overrides`, or logging that is provably confined to the debug variant.
3. Reporting an exported component, deep link, or broadcast receiver without a sensitive operation or disclosure behind it.
4. "Pinning can be bypassed with instrumentation" or "the APK can be decompiled" as findings.
5. Treating a public client identifier (Maps key, Firebase API key, OAuth client ID) as a leaked secret without checking what it authorizes.
6. Rating a null-pointer crash from a malformed extra as high; it is low unless it is persistent or crosses a boundary.
7. Treating plaintext data in the app's private directory as a finding when no lower-trust reader exists (no backup, no world permissions, no provider, no `run-as` in release).
8. Applying a class that only matters below the app's `minSdk` without checking `minSdk`.
9. Concluding exported state from source manifests when library manifests may merge in components or permissions.
10. Writing the report before independent verification, or letting prose and `findings.json` disagree.
