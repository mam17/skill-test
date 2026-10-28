# UI/UX Screen Review

#### When to use this file

Phase 3 (screen review) for every screen on an in-scope flow, and guidance-mode questions about layout, theming, localization, or visual quality. Rendering claims (overflow, overlap, contrast) found here are **needs-device-check** until seen on a device; resource and code facts are source-verified.

## Layout and resilience (per screen)

- **Text resilience**: every user-visible text container survives long content — translated strings up to ~1.5x English length, user data of arbitrary length, and font scale 2.0. Source leads: fixed `layout_height` in dp on text, `maxLines=1` without `ellipsize`, `textSize` in `dp` instead of `sp` (confirmed defect from source), `sp` on icon-sized boxes that clip at large scale.
- **Small and large screens**: the layout works at ~320dp width and on tablets/foldables. Fixed large dp widths, absolute positioning, and non-scrolling screens with tall content are leads. Portrait-locked apps still need this on targetSdk 35+ large screens.
- **Edge-to-edge and insets** (targetSdk 35+ enforced): every screen handles status/navigation-bar insets (`ViewCompat.setOnApplyWindowInsetsListener`, `fitsSystemWindows`, or Compose `WindowInsets`). Source lead: no inset handling anywhere plus full-screen content = content under system bars; bottom buttons under the gesture bar are the classic case.
- **Scrolling and keyboards**: input screens scroll when the IME opens (`windowSoftInputMode` or inset animation); the focused field and the submit button stay reachable.
- **State visuals**: loading, empty, error, and offline each have a designed state on screens that load data (pairs with FLOW-REVIEW item 4 — flow checks the behavior, this checks a state exists and is distinguishable).

## Theming

- **Dark mode**: if the theme is `DayNight` or `values-night` exists, every screen must be checked in both modes. Source leads: hardcoded `#FFFFFF`/`#000000` or literal colors in layouts instead of theme attributes/color resources, drawables without night variants holding text, `values-night` present but missing keys that `values` defines.
- **Deliberate opt-out** is acceptable: a forced light/dark theme via `AppCompatDelegate.setDefaultNightMode` or a non-DayNight theme is a product choice, not a defect — but then `values-night` leftovers that partially apply are a defect (half-themed screens).
- **Consistency**: one source of truth for colors and type (theme attributes / design tokens); per-screen literal colors diverging from the palette are polish findings.

## Localization

- **No hardcoded user-visible strings** in layouts (`android:text="Some text"`) or code (`setText("...")`, string literals in dialogs, toasts, notifications). `tools:text` is fine.
- **Translation completeness**: for each `values-<locale>/strings.xml`, diff keys against the default. Missing keys fall back to the default language mid-UI — on a main screen that is major. Report per-locale missing counts and name the main-screen keys.
- **Format safety**: positional args (`%1$s`) in every translation that reorders; plurals via `<plurals>`, not `"%d items"`; dates/numbers via locale-aware formatters, not string concatenation.
- **Expansion**: German/Russian/Hindi length versus fixed-width buttons and tabs — needs-device-check per screen unless the layout provably wraps/scales.
- **RTL**: if `supportsRtl="true"`, grep for `layout_marginLeft|Right`, `paddingLeft|Right`, `gravity="left|right"`, `layout_alignParentLeft|Right` — each is a source-verified RTL lead; directional icons (back arrows, progress chevrons) need `autoMirrored` or RTL variants. If the app does not support RTL, verify `supportsRtl` is false so the system does not mirror it anyway.

## Interaction quality

- **Touch targets**: interactive views at least 48x48dp including padding (`minWidth/minHeight` or measured size). Small icon buttons without padding are source-verified leads; severity by the control's role.
- **Feedback**: every tap has a visible response (ripple/selected state); destructive actions confirm or offer undo; long operations show progress, not a frozen UI.
- **Consistency**: the same concept uses the same control and wording across screens (one "Save" pattern, one back affordance, dialogs from one style). Divergences are polish unless they mislead.
- **System conventions**: standard status/navigation bar behavior, standard share/sheet patterns, notifications use channels with sensible importance, full-screen intents only for truly urgent interruptions (alarm, call).

## What is a product choice, not a defect

Locked portrait on phones, a custom non-Material visual style, an opinionated onboarding, forced theme — none of these are findings by themselves. They become findings only when they produce a concrete bad outcome in evidence (portrait lock + targetSdk 35 tablet = broken layout; custom style + 2.0 font scale = clipped CTA). State the outcome, not the preference.
