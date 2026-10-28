# Deep Links and WebView Hunting

#### When to use this file

Reach for this file when the app handles custom-scheme links, App Links (`autoVerify`), OAuth or SSO redirects, Custom Tabs, or embeds a `WebView`. The realistic attacker is tier A4 (malicious web content or another app invoking a link) and, for link interception, tier A2. Use `ANDROID-COMPONENTS-AND-IPC.md` for the intent and activity side of a link handler and `ANDROID-NETWORK-AND-CRYPTO.md` for the TLS trust of what a WebView loads.

## Core discipline (include in every agent prompt for this domain)

```
- Establish the attacker: a web page in the WebView or a Custom Tab (A4), or another app sending a deep link (A2). Self-navigation to your own trusted content is not a boundary.
- A deep link, App Link, or redirect parameter is untrusted input. Bind each state-changing or authenticating link to the current session, a one-time value, and the expected provider/account.
- A WebView bridge intended for packaged content must re-check the frame origin at call time and after every navigation, redirect, subframe, and error page. Initial-load and URL-prefix checks are insufficient.
- Verify App Link ownership (autoVerify plus the hosted assetlinks) before assuming a link cannot be hijacked; an unverified https deep link can be claimed by another app.
- Framework and setting defaults matter: JavaScript, file access, universal access, mixed content, safe-browsing, and content debugging. Apply the shipped targetSdk and the release variant.
- Use confirmed for a source chain plus a bounded local check (a local page or dummy link on the emulator). Use needs_validation when assetlinks hosting, the installed-browser behavior, or an OEM WebView fact is required but unobservable.
```

## Deep-link and redirect attack classes (subagent_type: `general`)

**Deep-link parameter trust**
A deep-link or App Link handler reads a query parameter, path segment, or fragment and uses it to mutate state, select an account or tenant, load a URL, build a file path, or complete authentication without validating it against an allowlist and the current session. Check URI normalization (case, encoding, duplicate query keys, `@` and userinfo tricks in URLs passed onward), the exact scheme/host/path matching, and whether a parameter flows into a WebView, an HTTP client (open redirect/SSRF toward internal hosts), or an intent.

**App Link and scheme hijacking**
A sensitive flow relies on a custom scheme or an unverified https link that another installed app can also register, letting that app intercept the callback (OAuth code, magic link, invite). Check `autoVerify`, whether the redirect carries a secret (authorization code, token) that a competing handler could receive, and whether PKCE and state binding limit the damage. Verified App Links with correct hosted assetlinks are a real control; absence of verification is the gap.

**Auth-callback and account-handoff confusion**
An OAuth/OIDC, SSO, magic-link, invite, or payment callback returns to the wrong account, tenant, or pending operation, or accepts a callback not bound to the request that started it. Check that `state`/`nonce`/PKCE are generated, stored bound to the session, and verified on return, and that the callback cannot be replayed or delivered to a different in-app flow.

## WebView attack classes (subagent_type: `general`)

**JavaScript bridge exposure**
Remote or attacker-influenceable content can reach an `@JavascriptInterface` method, a `WebMessage` handler, or an `evaluateJavascript` round-trip intended only for packaged content, and that method performs a privileged action (file access, token retrieval, IPC, navigation, arbitrary intent). Check what URLs can load in the WebView, whether the origin is re-validated at call time and after redirects and subframes, the method allowlist, and the authority the bridge exposes. On older `minSdk` levels, note `addJavascriptInterface` reflection exposure, but only when `minSdk` makes it reachable.

**Navigation and origin control**
`shouldOverrideUrlLoading`/`shouldInterceptRequest` fail to confine navigation, so the WebView follows attacker-controlled links to remote origins, `javascript:`, `file:`, `content:`, or `intent:` destinations, or renders attacker HTML via `loadData`/`loadDataWithBaseURL` with a trusted base URL. Check the navigation policy, `intent:` URL handling (which can reach internal components), and whether a trusted base URL grants attacker markup same-origin access to app content.

**WebView file and universal access**
`setAllowFileAccess`, `setAllowFileAccessFromFileURLs`, `setAllowUniversalAccessFromFileURLs`, `setAllowContentAccess`, or a custom `WebViewAssetLoader`/content-scheme handler lets remote content read app-local files or privileged origins. Check which settings are enabled in the release variant and whether reachable protected content exists; a relaxed setting with no reachable protected content is hardening.

**WebView TLS and debugging**
`onReceivedSslError` calls `proceed()` (defeating TLS for everything the WebView loads), `setWebContentsDebuggingEnabled(true)` ships in release, or mixed content is allowed on a page that carries a session. Check the release variant specifically.

## Universal moves (apply across the above)

- Enumerate every intent-filter URI (scheme, host, `pathPrefix`/`pathPattern`, `autoVerify`) and every WebView creation site with its settings, loaded URLs, and bridges.
- For a deep-link finding, trace one dummy link from the handler through validation to the sink. For a WebView finding, cite both the navigation/origin control and the privileged bridge or file sink, and show attacker content reaching it.
- Validate with a dummy link or a local HTML page loaded in the target's debug build on the emulator. Do not load attacker infrastructure or real OAuth providers.

## Validation rules (apply before reporting ANY finding here)

1. Name the attacker (A4 page or A2 link sender), the entry (URI or loaded content), the accepted value, the missing binding or origin check, and the resulting action or disclosure.
2. For link hijacking, state whether the link is a verified App Link or an interceptable scheme/unverified link, and what secret or action the callback carries.
3. For a WebView bridge, cite both the origin/navigation control and the privileged sink, and confirm remote content can actually load. Confirm the setting is present in the release variant.
4. Keep the local check to one dummy link or local page. Stop at the first boundary result; do not pivot to real accounts or remote origins.
5. Return `confirmed` only with a complete content-to-sink chain and bounded local evidence. Return `needs_validation` with the exact assetlinks-hosting, browser, or OEM-WebView fact an owner must verify.
