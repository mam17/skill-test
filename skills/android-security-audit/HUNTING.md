# Vulnerability Hunting

### Phase 2: Run coverage-led hunting waves

The parent assigns `planned` ledger units to `general` agents. One hunter may own closely related units in one module; no unit may be silently unassigned. A unit the budget cannot reach is explicitly `deferred` with reason `budget_cannot_reserve_critics_and_validation`.

When a budget or profile caps hunter count, rank units: (1) surfaces reachable by tier A1, A2, or A4 before those needing permissions or physical access; (2) boundaries protecting credentials, session tokens, payments, personal data, or code execution; (3) prior gaps, changed source, and SDK-level changes before same-source re-passes; (4) classes that historically yield confirmed Android findings (exported components, deep links, WebView bridges, TLS trust, intent redirection) before speculative ones. Ties break lexicographically by `coverage_id`.

Before launch, the parent sets assigned units to `in_progress`, assigns a canonical lowercase `agent_id`, and creates that agent's `scratch/` and parent-owned `artifacts/`. Hunters read source and parent-provided context, write only to `scratch/`, and return one structured result. They never edit target source, shared files, retained artifacts, or another agent's files.

## Required hunter prompt

Every hunter prompt contains, in order:

1. A two-sentence role preamble: find source-grounded security invariant failures in the assigned units and return exactly one JSON object matching the structured-result contract.
2. `architecture.md` verbatim.
3. Assigned coverage IDs, module, boundary, starting paths, and each unit's block map from the ledger.
4. The exact selected blocks copied verbatim: each ordinary block from `ATTACK-CLASSES.md`, and from each selected `ANDROID-*.md` companion its `Core discipline`, each chosen class subsection, `Universal moves`, and `Validation rules`. Never send block names alone.
5. Explicit excluded blocks with reasons.
6. The core hunting method below, then the execution and promotion rules block.
7. The core validation rules below.
8. Carried same-source prior confirmed exclusions (fingerprint, title, root cause only) and peer-owned coverage IDs the hunter must not duplicate.
9. The hunter's scratch/artifact paths, agent ID, predeclared promotion allowlist and byte limits, the execution policy from `run-metadata.json`, the structured-result contract below, and the `confirmed` and `needs_validation` branches of `report-schema.json` copied verbatim.

If an unexpected different boundary appears, the hunter returns it under `uncovered` so the parent creates a ledger unit for the next wave. A JNI entry that parses untrusted bytes is always returned as `uncovered` with the native symbol.

#### Core hunting method — include in every hunter prompt

```text
## Defensive vulnerability-finding method (Android)

Your goal is to find source-grounded security invariant failures and the smallest fix, not
to expand harm beyond the boundary result. Stay within source review and bounded local
execution on a fresh emulator with a debug build and dummy data. Do not contact production
backends, Play services for real accounts, other users, or other installed apps' data.

READ THE CODE AT DEPTH. Follow each assigned input from its Android entry point (intent,
URI, provider call, WebView message, response body, file) through parsing, caller identity,
authorization, normalization, state, derived copies, and the final sink. Read sibling
components, activity-aliases, legacy and flavor-specific source sets, retry, cancellation,
migration, and error paths that produce the same effect. Compare what one component
guarantees with what the next assumes.

WORK FROM A CONCRETE INVARIANT:
1. Name the attacker tier (A1-A6) and starting capability.
2. Name the accepted value, action, state transition, or resource selector.
3. Locate the control that should reject, bind, isolate, limit, or revoke it, and whether
   it comes from the merged manifest, code, a resource, or an external fact.
4. Trace the exact source path after that decision.
5. Stop at the smallest affected dummy record, wrong return value, leaked dummy token,
   unauthorized component invocation, or bridge call from a local page.
6. State a source-level change and a regression test that enforce the invariant.

CHECK THE API LEVEL. A class that only applies below minSdk is excluded; a class whose
default changed at the target's targetSdk (exported defaults, PendingIntent mutability,
cleartext, scoped storage, implicit-intent restrictions for exported components) applies as
the shipped targetSdk defines it.

DEPTH BOUND: trace only paths that can reach your assigned boundary or whose guarantees it
relies on. Stop a line of investigation as soon as the invariant is settled either way and
record the result as covered, candidate, blocked, or uncovered.

TEST SAD PATHS. Absent, empty, duplicate, and oversized extras; null Uri; mixed-case
schemes and hosts; encoded separators; nested intents; mutable PendingIntents; process
death and restore; logout and account switch; an older API level inside minSdk; a
library manifest that adds a component or permission.

When a proposed high or critical candidate reveals a reusable root cause, search paths owned
by your coverage IDs for variants. Consolidate the same root cause; establish each variant
independently. Return a variant with no current unit as uncovered.

USE THE NARROWEST LOCAL CHECK THAT SETTLES THE CLAIM, following ANDROID-DYNAMIC-CHECKS.md:
prefer an existing unit test, a JVM or Robolectric harness around the guard, a one-shot
observation of how the target handles a dummy input, or a local mock server. Record the
exact command, emulator image, input, and minimum result. Remember what each method proves:
an adb invocation runs with shell privileges a third-party app lacks, so establish the
attacker tier from the merged manifest, not from adb reachability alone. Never stress
availability, use a real credential, send real SMS/push/payments, or continue past the
minimum observed effect.

A backend, Play Console, Firebase rule, OEM, or shipped-manifest fact outside source is
not proof either way. If one such fact is decisive, return needs_validation with the exact
missing observation and a safe owner-observed check.
```

#### Execution and promotion rules — include verbatim in every hunter and verifier prompt

```text
Execution policy: <copied from run-metadata.json>.

Builds and unit tests run offline (--offline), from an environment that holds no signing
keys, API tokens, or developer credentials, inside the strongest sandbox available under the
execution policy. App execution happens only on a fresh emulator or dedicated test device
with a debug build of the target, dummy accounts, and a local or owner-designated
non-production backend. You write only inside your scratch/ directory.

Artifact promotion (parent-side only; you never perform it): after you finish, the parent
promotes each predeclared scratch-relative file separately. It rejects absolute paths,
empty or dot components, symlinks, directories, archives, special files, files over the
recorded per-file or cumulative byte limit, and anything not on the allowlist. It never
copies recursively or globs scratch. A file that fails any check is discarded; if it was
decisive evidence, you keep needs_validation with the exact promotion blocker.
```

#### Core validation rules — include in every hunter prompt

```text
## Candidate gate

1. A candidate needs a complete repository-relative source trace and evidence for the claimed
   root cause, including the strongest source-visible control and the attacker tier.
2. A proposed confirmed record needs a bounded local observed result, meaningful impact across
   the stated boundary, complete conditions (variant, API range, permissions, user action),
   and no visible preventing layer.
3. Do not strengthen a crash into code execution, a debug-only relaxation into a release
   finding, a device-owner (A6) action into a boundary violation, or an exported component
   with no sensitive behavior into a finding.
4. If a required fact is not source-visible or locally observable, use needs_validation. Name
   exact blockers; do not give it severity.
5. A missing best practice with no affected principal or resource is hardening, not a finding.
   A candidate disproved by source is not needs_validation.
6. Use the same source-derived fingerprint for the same root cause in every state. It must
   match ^[A-Za-z0-9][A-Za-z0-9._:/@+-]*$ and must not include a line, wave, agent,
   severity, or verdict.
7. Return an empty candidate array when nothing survives these gates.
```

## Local validation boundaries

Allowed under the recorded execution policy: offline Gradle builds with a populated cache; JVM unit and Robolectric tests; instrumentation tests on a fresh emulator; one-shot observations of how an exported component or provider handles a dummy input; manifest and APK inspection (`dumpsys package`, `apkanalyzer`, `aapt2`); logcat capture around one action; a local `MockWebServer` or WireMock; observing the app's own TLS trust behavior against a local mock on the emulator with the debug build.

Disallowed: production or shared backends; real accounts, SIMs, payment methods, or push recipients; Play Store installs; other installed apps' data; dependency downloads; repackaging or signing with real keys; stress or saturation; defeating the target's own protections on a device you control as "evidence"; any work after the minimum dummy-data boundary result.

## Structured hunter result

Return exactly one JSON object, no surrounding prose:

```json
{
  "units": [
    {
      "coverage_id": "one assigned ID",
      "disposition": "covered|candidate|blocked",
      "reviewed_paths": ["repo/relative/path"],
      "checks": [
        {
          "agent_id": "canonical owner of this check",
          "reviewed_paths": ["repo/relative/path owned by this check"],
          "invariant": "specific control checked for this unit",
          "method": "source|local",
          "result": "what source or the bounded check established",
          "artifact": "agents/<agent-id>/artifacts/file for local, null for source"
        }
      ],
      "candidate_fingerprints": [],
      "unresolved": []
    }
  ],
  "candidates": [],
  "hardening": ["concrete non-finding note, with MASVS control id when known"],
  "uncovered": [
    {
      "surface": "...",
      "boundary": "...",
      "subsystem": ":module",
      "attack_class": "...",
      "starting_paths": ["repo/relative/path"],
      "reason": "why this needs its own deterministic coverage unit"
    }
  ]
}
```

Each `candidates` entry is schema-shaped except that it uses `proposed_verdict` in place of `verdict`:

- `proposed_verdict: "confirmed"`: include every field the `confirmed` branch requires other than `verdict`: `fingerprint`, `title`, `description`, `root_cause`, `intended_behavior`, ordered `trace`, `evidence`, `conditions`, target-neutral `execution`, `remediation`, `severity`, `confidence`. `conditions` is an array of `{kind, description}` objects whose `kind` is one of the schema enum values (`system_configuration` for variant and API range, `authentication_level`/`authorization_role` for required permissions, `user_interaction`, `third_party_dependency` for a bundled SDK, etc.). `execution.attacker_perspective` names the tier. `execution.payloads` holds the minimum dummy intent, URI, page, or fixture. `execution.instructions` describe the emulator steps already performed. `execution.observed_result` records actual output (logcat line, returned cursor row, bridge method reached). Overall severity must not exceed observed impact.
- `proposed_verdict: "needs_validation"`: include `fingerprint`, `title`, `description`, `claimed_root_cause`, ordered `trace`, `evidence`, nonempty `blockers`, and `validation_plan` with at least one `local` or `deployment` step. `deployment` here means an owner-observed check of the backend, Play Console, Firebase rules, shipped manifest, or device policy; never a request to probe production.

Every assigned coverage ID appears exactly once in `units`. Trace entries begin at `entrypoint`, end at `sink`, and label intermediate steps `propagation`. All paths are repository-relative. A `source` check uses `artifact: null`; a `local` check uses one parent-promoted regular file beneath `agents/<check.agent_id>/artifacts/`.

## Parent consolidation and ledger update

The parent validates each unit result, maps it to exactly one assigned `coverage_id`, and updates only that ledger unit: `reviewed_paths`, `checks` into `local_checks`, artifact paths, candidate fingerprints, unresolved facts. Retain `hardening` lists in parent bookkeeping fields for Phase 6. A malformed result leaves the unit `planned` for reassignment. Untouched budget units become `deferred` with a reason. Run `validate-coverage-ledger.cjs` after every update.

Consolidate candidates by fingerprint, then by root cause. One root cause reachable from several components or links is one candidate with the strongest complete trace. Independent missing controls use separate fingerprints.

## Coverage-critic waves

Immediately after each hunter wave, spend the reserved invocation on one fresh `research` coverage critic. It receives `architecture.md`, the full ledger with block maps, current candidate fingerprints and states, and the prior-ledger gap summary. It reads source but never runs targets. Require exactly this JSON:

```json
{
  "missing_units": [
    {
      "surface": "...",
      "boundary": "...",
      "subsystem": ":module",
      "attack_class": "...",
      "starting_paths": ["repo/relative/path"],
      "selected_companion_blocks": ["FILE.md#section"],
      "excluded_blocks": [{"block": "FILE.md#section", "reason": "..."}],
      "reason": "source-backed coverage gap"
    }
  ],
  "reassign_ids": ["existing-id-that-did-not-close"],
  "resolved_prior_leads": ["fingerprint"],
  "stop": false
}
```

The Android critic specifically checks for: components present in the merged manifest but absent from the ledger (including library-contributed ones and `activity-alias` entries); deep-link hosts or paths without a unit; WebViews without bridge and navigation units; each `PendingIntent` creation site; each `onMessageReceived` or WebSocket handler; each `FileProvider` and provider path; each `TrustManager`, `HostnameVerifier`, and `network_security_config` per variant; logout and account-switch lifecycle for storage units; flavor-specific source sets; JNI handoffs returned as `uncovered` but not yet ledgered. It proposes coverage, not findings.

Loop, archive, `reassign_ids`, `quick`-profile, scoped-run, and budget behavior match the upstream skill: in `standard` and `deep`, complete coverage requires a post-wave critic with no accepted work followed by a distinct final-clean critic with no accepted work. `quick` runs one wave and one final critic and records accepted work as `deferred` with reason `quick_profile_final_critic`. A scoped run records out-of-scope gaps as `out_of_scope`. If reserves cannot be met, launch no hunters, defer the wave, and disclose the gap. Never use a silent cap as evidence of complete coverage.
