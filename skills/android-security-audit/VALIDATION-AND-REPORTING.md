# Validation, Structured Output, Verification, and Reporting

### Phase 3: Independently validate every candidate

After the clean critic pass or a recorded early stop, consolidate Phase 2 candidates and carried same-source prior confirmations by fingerprint and root cause. Give every unique proposed `confirmed` and `needs_validation` candidate to a fresh `general` verifier that did not hunt it. The verifier may read hunter artifacts but must re-read every cited current source location and independently reproduce any decisive bounded check it can perform safely.

Assign each verifier a canonical lowercase ID with `scratch/` and parent-owned `artifacts/`. It receives only: the candidate, its linked ledger checks and artifact paths, the architecture facts needed to interpret the path, the exact relevant `ANDROID-*.md` `Validation rules` blocks, the execution and promotion rules block from `HUNTING.md`, `ANDROID-DYNAMIC-CHECKS.md`, the `confirmed`, `needs_validation`, and `rejected` branches of `report-schema.json` verbatim, and prior records with the same fingerprint. It must not receive another verifier's conclusion.

#### Candidate-verifier prompt

```text
You did not write this candidate. Try to refute it from repository source and bounded local
evidence under the execution policy in this prompt. Use a fresh emulator, a debug build of
the target, dummy accounts, and a local or owner-designated non-production backend only.

1. Verify every trace and evidence file, line, scope, and description. Confirm the first
   entry is a real entry point reachable by the stated attacker tier (A1-A6) and the last
   is the claimed sink or boundary effect.
2. Reconstruct the strongest source-visible controls on the path: merged-manifest exported
   state and permissions, caller identity checks, intent and URI validation, WebView
   navigation and bridge policy, network_security_config for the shipped variant, Keystore
   parameters, biometric binding, framework defaults at the shipped targetSdk, and minSdk.
   If the candidate depends on a manifest fact, state whether you read the merged manifest
   or source manifests only.
3. For a proposed confirmed candidate, independently reproduce the minimum observed result
   when possible (unit harness, one-shot component observation, local page, mock server).
   Note what the chosen method proves: an adb result does not by itself establish a
   no-permission app's capability unless the component is exported without an effective
   permission in the merged manifest. Do not infer a stronger result or continue after it.
4. Verify that likelihood, impact, confidence, and the proposed fix match only what the
   evidence establishes, that the finding is not confined to the debug variant, and that
   tier A6 is not the only attacker.
5. For a proposed needs_validation candidate, decide whether the blocker is genuinely
   outside source and local observation (backend, Play Console, Firebase rules, OEM, shipped
   manifest you cannot build). If source refutes the trace, reject it. If the missing fact
   remains decisive, keep needs_validation and make the local and owner-observed plans exact
   and non-destructive.
6. Preserve the fingerprint for the same source-derived root cause across every state.

Return exactly one JSON object and no surrounding prose:
{"decision": "confirmed|needs_validation|rejected", "record": { ... }}
where record exactly matches the decision's verdict branch of the schema in this prompt.
A corrected record replaces the hunter's wording.
```

The parent copies the execution and promotion rules block from `HUNTING.md` verbatim into every verifier prompt.

A malformed, prose-wrapped, or schema-violating verifier result is discarded without repair; re-run with a fresh verifier when the budget permits. If no budget remains, the candidate stays an unresolved ledger `candidate` and the run becomes `incomplete` with reason `validation_budget_exhausted`.

### Phase 4: Write and validate structured output

The parent writes `<output-dir>/findings.json` as a top-level JSON array containing every final `confirmed`, `needs_validation`, and `rejected` record, each exactly matching its branch of `report-schema.json`. A rejected record keeps the fingerprint, the trace that was checked, and the `reason` it failed.

Android conventions inside records:

- `execution.attacker_perspective` begins with the tier, for example `A2 (co-installed app, no permissions): ...`.
- `conditions` is an array of `{kind, description}` objects, where `kind` is one of the schema enum values (`authentication_level`, `authorization_role`, `user_interaction`, `system_configuration`, `network_routing`, `environmental_dependency`, `data_state`, `timing_dependency`, `third_party_dependency`). Map Android conditions onto these: the shipped variant and API range go under `system_configuration`; required permissions under `authentication_level` or `authorization_role`; required taps or navigation under `user_interaction`; a bundled-SDK requirement under `third_party_dependency`. Example: `{"kind":"system_configuration","description":"prodRelease variant, minSdk 24..targetSdk 34"}`.
- `execution.payloads` hold the dummy intent, URI, local HTML page, fixture JSON, or snippet actually used to observe the behavior.
- `remediation.code_changes` name repository files: Kotlin, `AndroidManifest.xml`, `network_security_config.xml`, `build.gradle.kts`, R8/ProGuard rules.

Run:

```bash
node <skill-dir>/validate-findings.cjs <output-dir>/findings.json
node <skill-dir>/validate-coverage-ledger.cjs <output-dir>/coverage-ledger.json
```

Both must pass before Phase 5. Fix records, never the validator, unless the validator itself is wrong.

### Phase 5: Verify the final records with fresh eyes

Launch one fresh `research` verifier per final `confirmed` and `needs_validation` record, in parallel. In a `quick` run, Phase 3 and Phase 5 merge into one independent reviewer per candidate. Never skip independent review of a `confirmed` record.

For `confirmed`, the verifier checks: every repository-relative trace and evidence location; the real entry interface and exact dummy input; every condition, control, and the observed result; the affected principal and demonstrated impact; severity separation (overall not above impact; A6-only attackers downgraded to hardening); remediation that enforces the invariant rather than moving trust (for example, replacing `getCallingPackage()` string matching with signature verification, not with a different string).

For `needs_validation`, the verifier checks: the source path is real and supports only the `claimed_root_cause`; every blocker is decisive and not answerable locally (a merged manifest the parent could have produced offline is answerable locally); a concrete boundary and result are named; at least one validation-plan field is exact; the fingerprint matches prior records.

Each verifier returns exactly `{"decision":"verified","fingerprint":"..."}` or `{"decision":"replace","reason":"...","record":{...}}`. A replacement that promotes a verdict, or materially changes root cause, trace, input, observed result, impact, or severity, goes to another fresh verifier before it is applied. Non-material wording and line corrections may be applied directly. After every applied replacement, rerun both validators and update linked ledger decisions. Set `run_status: "complete"` only when every ledger candidate has an independent final disposition and every retained record passes Phase 5.

### Phase 6: Produce target-neutral reports from final records

Only after Phase 5 passes for every retained record, derive prose from the final records, the ledger, and retained `hardening` notes. Prose never changes a verdict, severity, blocker, or impact.

#### `REPORT.md`

1. Run profile, scope, budget spent versus planned, source ref, shipped variant, `minSdk`/`targetSdk`, `manifest_source`, execution policy, prior-run use, and explicit deferred and out-of-scope coverage. A `quick`, scoped, budget-limited, or incomplete run states plainly that it is a partial pass and lists unvalidated fingerprints.
2. A short security posture summary with the realistic attacker tiers.
3. A confirmed-findings table: severity, title, attacker tier, affected boundary, MASVS control(s) from [MASVS-MAPPING.md](MASVS-MAPPING.md), one-line observed result.
4. Each confirmed finding: source location, tier, bounded emulator reproduction with dummy data, conditions, actual result, impact, priority rationale, smallest source fix.
5. A separate `NEEDS VALIDATION` table: title, trace, exact blocker, bounded local next step, owner-observed check (backend, Play Console, Firebase, shipped build). No severity.
6. Hardening notes grouped by MASVS category (resilience items go here unless the threat model made A6 a boundary) and positive source patterns observed.
7. Coverage summary from the ledger: covered, candidate, blocked, deferred counts, important exclusions, final critic result, and any JNI handoffs left `uncovered`.

#### `FINDINGS-DETAIL.md`

For each confirmed `medium`, `high`, or `critical` record: ordered trace and evidence; dummy attacker and affected dummy resource; exact intent, URI, page, fixture, or snippet used; emulator image and commands; observed output and the invariant it proves; conditions and containment; source-level remediation and regression test.

#### `NEEDS-VALIDATION.md`

For every unresolved record: trace, verified evidence, exact blocker, affected boundary, and each bounded local or owner-observed resolution plan. Prioritized leads without severity; never live-test guidance against production.

A clean run may have zero confirmed records. State that result and the remaining coverage limits without inventing low findings.
