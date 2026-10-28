# Reporting

#### When to use this file

Phase 5 of full review mode. Output directory mirrors the security skill: a new directory outside the target, default `~/android-flow-ux-review/<repo-name>/run-<N>`, holding `REPORT.md`, `FLOWS.md`, `findings.json`, an `evidence/` directory for screenshots/recordings, and `API-TEST-PLAN.md` when [API-FIRST-TESTING.md](API-FIRST-TESTING.md) applies.

## `findings.json` shape

A top-level JSON array; one object per finding:

```json
{
  "id": "flow/workout-session/process-death-loses-progress",
  "dimension": "flow | ui | localization | accessibility | api-contract",
  "severity": "blocker | major | minor | polish",
  "flow": "flow_id from the inventory, or null for a global issue",
  "flow_priority": "core | secondary | edge",
  "title": "one sentence, the user outcome first",
  "scenario": "concrete steps a real user takes and what goes wrong",
  "evidence": [
    {"type": "source", "ref": "app/src/main/java/.../WorkoutSessionFragment.kt:88", "note": "no SavedStateHandle/onSaveInstanceState for step index"},
    {"type": "device", "ref": "evidence/session_processdeath_pixel6_api36.png", "config": "Pixel 6, API 36, don't-keep-activities", "note": "restored at step 1 of 8"}
  ],
  "status": "confirmed | needs-device-check",
  "fix": "smallest source change that repairs the user outcome, with file references",
  "regression_guard": "the test that keeps it fixed (unit/espresso/manual matrix entry)"
}
```

Rules: `confirmed` requires at least one evidence entry proving the outcome (source proof or device observation per SKILL.md's evidence discipline). `needs-device-check` must carry the exact configuration that would settle it and never carries `severity: blocker` (unverified claims do not headline). One root cause = one finding; list additional affected screens inside the finding, do not duplicate it.

## `FLOWS.md` — the coverage ledger of this skill

The flow inventory table from [FLOW-REVIEW.md](FLOW-REVIEW.md), each flow annotated with its checklist outcome:

```
flow | priority | happy path | exits/back | interruption | failure states | re-entry | screens reviewed (UI/a11y) | disposition
```

Disposition is `covered`, `defects (ids)`, `needs-device-check (ids)`, or `not reviewed`. Unreviewed flows and screens are listed, never silently dropped. If no emulator was available, the interruption and rendering columns say `source-only` rather than implying device coverage.

## `REPORT.md` structure

1. **Run metadata**: target, variant, minSdk/targetSdk, UI stack (Views/Compose), locales shipped, profile (source-only or device matrix used), what was NOT reviewed.
2. **Verdict paragraph**: can a new user complete each core flow, and what is the single worst user outcome found.
3. **Findings table** ranked by severity then flow priority: id, severity, dimension, flow, title, status.
4. **Finding details** for blocker/major: scenario, evidence, fix, regression guard. Minor/polish may be grouped.
5. **Needs-device-check list** with configurations, when the run was source-only.
6. **API test plan summary** (when applicable): the Step-1-first execution table from [API-FIRST-TESTING.md](API-FIRST-TESTING.md), and contract findings folded into the findings table with dimension `api-contract`.
7. **Coverage**: the FLOWS.md summary line counts (covered / defects / needs-check / not reviewed), the device matrix actually run, and known coverage limits.

Severity ranking in prose must match `findings.json` exactly. A clean review states what was verified clean and what remains unchecked — zero findings with honest coverage beats padded polish lists.
