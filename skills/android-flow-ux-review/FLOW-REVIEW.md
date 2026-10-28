# Flow Review

#### When to use this file

Phase 1 (inventory) and Phase 2 (flow review) of the full workflow, and any guidance-mode question about navigation, state, back behavior, or flow completeness.

## Flow inventory (Phase 1 output)

Map the app before judging it. From the manifest, Navigation graphs (`res/navigation/*.xml`), `startActivity`/`navigate` call sites, deep links, and notification/widget/shortcut entries, produce one row per user flow:

```
flow_id | name | priority (core/secondary/edge) | entry (launcher/deeplink/notification/in-app) | screens in order | exit(s) | preconditions (login, permission, network)
```

- **Core**: the reason the app exists (for a workout app: pick plan → do workout → see progress; for a dialer: receive/place call). A new user must be able to complete these.
- **Secondary**: settings, profile, language, history, premium purchase.
- **Edge**: uninstall flows, error recovery, re-onboarding, notification-driven re-entry.

Every exported entry point (launcher, deep link, notification tap, widget, shortcut, `BOOT_COMPLETED` re-entry) must appear as an entry of some flow, or be flagged as an orphan entry. Every screen in the codebase must appear on some flow, or be flagged unreachable (dead code or a gap in the inventory).

## Per-flow checklist (Phase 2)

Walk each flow in priority order. For each, check every item and record covered / defect / needs-device-check:

**1. Happy path completes.** Each step's success action is wired (button handlers non-empty, navigation target exists, result propagated). The final state is observable (confirmation, updated list, persisted record).

**2. Every exit works.** Back, Up, gesture-back, dismiss, and system Home mid-flow. Predictive back (targetSdk 34+) does not break custom back handling; `OnBackPressedDispatcher`/`onBackPressedCallback` used instead of overriding `onBackPressed` alone. Back from the first screen of a flow returns to where the flow started, not to a blank or dead screen. No trap: a screen with `setCancelable(false)` or disabled back must have another exit.

**3. Interruption survives.**
- **Rotation / configuration change**: input text, selections, scroll position, and in-flight work survive. Portrait-locked apps: on targetSdk 35+ large screens the lock is ignored, so this still applies.
- **Process death**: kill mid-flow (see DYNAMIC-UI-CHECKS) and restore. Multi-step progress and user input must come back; `ViewModel` without `SavedStateHandle` or persistence is a source-verified lead here.
- **Overlay interruptions**: permission sheet, incoming call, ad, biometric prompt — the flow resumes where it was, no double-start, no duplicate dialog stack.

**4. Failure paths exist and recover.** For each step that loads or submits data: a loading state (and it cannot get stuck — timeout or cancel), an error state with retry that actually retries, an empty state that explains and offers the next action, and offline behavior (clear message or cached content, not an infinite spinner, not a crash). Source lead: a `catch`/error branch that only logs, or a sealed UI state with an Error type nothing renders.

**5. No double-execution.** Submit/purchase/save buttons are guarded against double-tap and against re-trigger after rotation (event-vs-state in the UI layer: a one-shot event replayed by LiveData/StateFlow on re-subscribe is a classic duplicate-navigation/duplicate-dialog source).

**6. Re-entry and state staleness.** Entering the flow again after completing it, from a notification, or via deep link into the middle: preconditions are re-checked (login, permission), stale drafts are handled, and the back stack built for deep entry is sane (parent stack, not an orphan screen).

**7. Cross-flow consistency.** The same action (e.g., opening an item) behaves the same from list, search, notification, and widget. Permission denial in one flow does not silently disable an unrelated flow.

## Source signals worth grepping first

- `onSaveInstanceState` / `SavedStateHandle` usage versus screens with user input (gap = process-death lead).
- `onBackPressed()` overrides without `OnBackPressedDispatcher` (predictive-back lead).
- `LiveData`/`StateFlow` carrying navigation or one-shot events (replay lead).
- `isEnabled = false` / debounce around submit handlers (absence = double-tap lead).
- Error types in sealed UI states never referenced by any renderer (dead error path).
- `navigate(` targets that do not exist in the graph, and graph destinations never navigated to (unreachable screen).
- `finish()` right after `startActivity` (back-stack shape), `FLAG_ACTIVITY_CLEAR_TASK` mid-flow (destroys return path).
- TODO/FIXME in ui/ and feature/ packages mentioning state, back, or restore.

## Verdicts

A flow defect is **confirmed** when source proves the wrong outcome (the error branch is empty; the event replays by construction; the screen has input and no saved state) or it was observed on the emulator. A rendering- or timing-dependent suspicion is **needs-device-check** with the exact reproduction (screen, configuration, steps). Rank severity by the flow's priority: a blocker on a core flow is the headline; the same defect on an edge flow is minor.
