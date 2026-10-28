---
name: android-flow-ux-review
description: Functional flow testing, UI/UX review, and API-first test planning for Android apps written in Kotlin (Views or Compose). Use for reviewing user flows (navigation, state, back stack, process death, error/empty/offline states), screen-level UI quality (layout, dark mode, RTL, text scaling, insets, localization), accessibility, and for planning fast API-contract tests before testing the internal functions that consume them. Run the full workflow for explicit flow-test, UX-audit, or UI-review requests; answer focused questions in guidance mode.
---

# Android Flow and UI/UX Review

Find defects a real user would hit: broken or dead-end flows, lost state, screens that fall apart under rotation, large fonts, dark mode, RTL, or small devices, and interactions that are confusing or inaccessible. This is the functional/UX sibling of `android-security-audit`: same evidence discipline, different target. A finding without a concrete user scenario and an observable wrong outcome is not a finding.

## Operating modes

- **Guidance mode** (default): for questions about one screen, one flow, or one UX concern, use the relevant companion sections and answer inline. Do not create output directories or write report artifacts.
- **Full review mode**: when the user explicitly asks to test the flows, audit the UX/UI, or requests report artifacts. Map the flow inventory, review every in-scope flow and screen, and write the artifacts in [REPORTING.md](REPORTING.md).

If the request could mean either, ask one focused question first.

## What this skill reviews

| Dimension | Companion | Example defects |
|---|---|---|
| Flow correctness | [FLOW-REVIEW.md](FLOW-REVIEW.md) | dead ends, unreachable screens, broken back stack, state lost on process death, double-submit, stuck loading, no offline path |
| Screen UI quality | [UIUX-REVIEW.md](UIUX-REVIEW.md) | clipped or overlapping text, broken dark mode, RTL mirroring failures, unscalable text, content under system bars, missing empty/error states |
| Localization | [UIUX-REVIEW.md](UIUX-REVIEW.md) | hardcoded strings, missing translations, overflow in long languages, untranslated plurals/formats |
| Accessibility | [ACCESSIBILITY.md](ACCESSIBILITY.md) | missing labels, tiny touch targets, focus traps, contrast, TalkBack-unusable controls |
| Device-verified behavior | [DYNAMIC-UI-CHECKS.md](DYNAMIC-UI-CHECKS.md) | what to run on the emulator and what each check proves |
| API-first test plan | [API-FIRST-TESTING.md](API-FIRST-TESTING.md) | contract tests against a mock before repository/usecase/ViewModel tests, then flows; DTO nullability crashes, unmapped error codes, spec drift |

Out of scope: security boundaries (use `android-security-audit`), backend correctness, visual taste beyond platform/Material conventions, and copywriting style beyond clarity defects.

## Evidence discipline

- **Source-verified**: the defect is provable from code or resources alone (a hardcoded string, a layout with fixed-height text, a fragment that never handles the error branch, an activity that ignores `onSaveInstanceState`). Cite `file:line`.
- **Device-verified**: the defect was observed on an emulator with a stated configuration (device profile, font scale, locale, dark mode, orientation). Cite the configuration and attach or describe the screenshot evidence.
- **Needs-device-check**: source suggests a defect but rendering or timing decides it (possible overflow, suspected jank, suspected inset overlap). Record the exact configuration that would settle it. Never report these as confirmed.

Do not invent findings to fill a report. A clean flow is reported as covered. Do not downgrade honest uncertainty: a suspected overflow stays needs-device-check until seen.

## Severity model (user impact, not security)

- **blocker** — a user cannot complete a core flow, or data is lost: crash on a main path, dead end with no exit, purchase/save silently fails, state wiped by process death mid-task.
- **major** — a flow completes but with real damage: progress lost on rotation, error state with no retry, content unreadable in dark mode or at large font, a screen unusable with TalkBack, a missing translation on a main screen.
- **minor** — friction or breakage off the main path: clipped text in one locale, inconsistent back behavior on a secondary screen, a touch target below guideline on a non-primary control.
- **polish** — visual or wording inconsistency with no functional impact.

Severity reflects the worst user outcome the evidence demonstrates, bounded by how central the flow is (use the flow inventory's priority). A defect on a core flow outranks the same defect on a settings screen.

## API-level and device realities

Check `minSdk`/`targetSdk` before applying any rule:

- **targetSdk 35+**: edge-to-edge is enforced; screens that do not handle window insets draw under status/navigation bars. On large screens the system can ignore `screenOrientation` and resizability locks, so portrait-locked layouts must still survive landscape and multi-window.
- **Font scale** up to 2.0 on modern Android; fixed-dp heights around text are suspects.
- **Dark theme** follows the system since API 29; an app with `DayNight` or `values-night` must be checked in both, and an app that opts out must force a theme deliberately, not accidentally.
- **RTL**: `android:supportsRtl` plus any `left/right` attribute instead of `start/end` is a concrete lead.
- **Process death**: any screen holding user input or multi-step progress must survive it; `ViewModel` alone does not (it dies with the process) — only `SavedStateHandle`/`onSaveInstanceState` or persistence survive.

## Full review workflow

1. **Inventory** — map modules, screens (activities/fragments/composables), navigation edges (Navigation graph, manual intents, deep links), and entry points. Produce the flow inventory: each user flow, its screens, its priority (core / secondary / edge), and its entry conditions. [FLOW-REVIEW.md](FLOW-REVIEW.md) defines the inventory format. If the app has a remote API, also produce the endpoint inventory and the API-first execution plan from [API-FIRST-TESTING.md](API-FIRST-TESTING.md) — contract tests run before anything else because a broken contract invalidates every layer above it.
2. **Flow review** — walk each flow in priority order through the checklists in [FLOW-REVIEW.md](FLOW-REVIEW.md): happy path, every exit, back/up, interruption (rotation, process death, incoming call, permission sheet), failure (offline, error, empty, slow), and re-entry.
3. **Screen review** — for each screen on an in-scope flow, apply [UIUX-REVIEW.md](UIUX-REVIEW.md) and [ACCESSIBILITY.md](ACCESSIBILITY.md) to its layout/composable and resources.
4. **Device verification** — when an emulator is available, verify every needs-device-check lead and the top flows under the configuration matrix in [DYNAMIC-UI-CHECKS.md](DYNAMIC-UI-CHECKS.md). Without an emulator, deliver the source-verified findings and the explicit needs-device-check list; say plainly that rendering-dependent claims are unverified.
5. **Report** — write the artifacts in [REPORTING.md](REPORTING.md): findings ranked by severity x flow priority, the flow coverage table, and what was not reviewed.

Execution safety: run only the target's own debug build on a fresh emulator with dummy data; never a real account, never production. The same offline-build and write-isolation rules as `android-security-audit` apply when both skills run together.

## Anti-patterns

1. Reporting taste ("this blue feels off") as a finding; cite a platform convention, guideline value, or user outcome instead.
2. Confirming a rendering claim (overflow, overlap, contrast) from source alone — those are needs-device-check until seen.
3. Reporting a rotation bug on a portrait-locked phone screen without noting large-screen/multi-window reachability, or ignoring that targetSdk 35+ large screens unlock orientation anyway.
4. Counting every missing `contentDescription` as major; severity follows the control's role in a flow.
5. Flooding the report with polish items while a core flow dead-ends; rank by severity x flow priority.
6. Treating a deliberate product choice (locked portrait, custom back behavior with a confirm dialog) as a defect without stating what user harm the evidence shows.
7. Letting the report disagree with the coverage table; an unreviewed flow is listed as unreviewed, not silently omitted.
