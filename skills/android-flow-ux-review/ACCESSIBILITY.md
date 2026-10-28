# Accessibility Review

#### When to use this file

Phase 3 alongside UIUX-REVIEW for every screen on an in-scope flow. Severity follows the control's role: a core-flow control unusable with TalkBack is **major**; the same gap on a decorative element is **polish**.

## Labels and semantics (source-verified)

- Every functional `ImageView`/`ImageButton`/icon-only control has a `contentDescription` (or Compose `semantics`); decorative images set `contentDescription="@null"` / `importantForAccessibility="no"` so they are skipped, not announced as "unlabeled".
- `EditText` fields have `android:hint` or a paired label via `labelFor`; custom controls expose role and state (`AccessibilityDelegate`, Compose `role`/`stateDescription`).
- Dynamic state changes that matter (item added, error shown, loading finished) are announced (`announceForAccessibility`, live regions) on screens where the result is otherwise invisible to a non-sighted user.
- Grep starters: `<ImageButton` and clickable `<ImageView` without `contentDescription`; `contentDescription="@string/app_name"` (lazy label); clickable `View`/`FrameLayout` wrappers whose child holds the label but the wrapper gets focus.

## Focus and navigation (device-verified)

- TalkBack swipe order follows the visual/logical order; no focus traps (dialogs return focus on dismiss, bottom sheets contain focus while open).
- Custom gestures and drag interactions have an accessible alternative (a button or menu action).
- Keyboard/d-pad: focusable order sane, no unreachable interactive views (matters for TV/foldable keyboards).

## Visual accessibility

- **Contrast**: body text 4.5:1, large text 3:1 against its real background. From source, flag literal low-contrast pairs (light gray on white, hint-colored body text); confirm ratios on device/screenshot.
- **Not color-alone**: error/success/selection states carry a second cue (icon, text, underline), not only a color change.
- **Text scaling**: pairs with UIUX-REVIEW — at font scale 2.0 every label is still readable and no control disappears; `sp` everywhere for text.
- **Touch targets**: 48x48dp minimum (shared check with UIUX-REVIEW; count it once).

## Media and timing

- Autoplaying video/animation on core screens can be paused; no flashing above safe thresholds.
- Timed UI (auto-dismissing snackbars carrying actions, countdown-gated buttons) has enough time or an alternative; TalkBack users get longer or persistent affordances.

## Reporting

Record per screen: labeled / unlabeled functional controls (count), focus issues observed, contrast suspects, and the single worst user outcome ("a TalkBack user cannot start a workout from Home" beats ten unlabeled icons in a list). Aggregate counts go in the coverage table; only outcome-bearing defects become findings.
