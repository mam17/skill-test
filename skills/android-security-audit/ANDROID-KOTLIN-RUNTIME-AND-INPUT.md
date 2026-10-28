# Kotlin Runtime and Input Handling

#### When to use this file

Reach for this file when untrusted bytes or objects are parsed or dispatched inside the app: `Parcelable`/`Serializable` extras, `kotlinx.serialization`/Gson/Moshi polymorphic decoding, path and archive handling, SQL built from input, reflection and dynamic code loading, coroutine and concurrency state, and handoffs to native (JNI). The attacker tier is whatever reaches the input (A1 response, A2 intent/provider, A4 WebView). Use `ANDROID-NETWORK-AND-CRYPTO.md` for the transport and `ANDROID-COMPONENTS-AND-IPC.md` for the component that delivers the input.

## Core discipline (include in every agent prompt for this domain)

```
- Identify the untrusted input and the tier that controls it (A1 response, A2 intent/provider/broadcast, A4 WebView message). Trace it to the parser and then to the sink.
- Kotlin null-safety and types are not input validation. A crash from a malformed extra is availability of the app's own process (usually low) unless it is persistent or crosses a boundary.
- Deserialization risk on Android is mostly type and field confusion, not classic Java gadget RCE, but Serializable extras, dynamic class loading, and polymorphic decoders can still select unexpected types or reach dangerous constructors. Establish the concrete consequence, not the pattern alone.
- Concurrency findings need a realistic interleaving an attacker can induce and a security-relevant wrong state (double-spend, lost auth check, TOCTOU), not a theoretical race.
- A JNI handoff of untrusted bytes leaves this skill's scope: record it as uncovered with the native symbol so a memory-safety review can follow.
- Use confirmed for a source path plus a bounded local harness result. Use needs_validation when reachability depends on a manifest, minSdk, or runtime fact that is unobservable.
```

## Input-parsing attack classes (subagent_type: `general`)

**Unsafe deserialization and type confusion**
`getSerializableExtra`, a `Parcelable` with a hand-written `createFromParcel`, or a polymorphic `kotlinx.serialization`/Gson/Moshi decoder trusts attacker-chosen types, class names, or field values from an intent, provider, or response. Check class allowlists, `@JsonClassDiscriminator`/registered subtypes, default-value handling, and whether a decoded type reaches a dangerous sink or a security decision. Report the concrete consequence (type reached, field injected), not the decode alone.

**Path and archive handling**
A filename, entry name, or path from input is joined to a private directory without canonicalizing and confirming containment, allowing traversal (`..`, encoded separators, absolute paths) or zip slip during extraction of a downloaded bundle, theme, or import. Check `File`/`Path` construction, `ZipInputStream`/`ZipFile` entry handling, and the final resolved path versus the intended root.

**Injection into local interpreters**
Input reaches `rawQuery`/`execSQL`/selection without parameterization, a `ProcessBuilder`/`Runtime.exec` command, a `String.format` pattern, a regex compiled from input (ReDoS), or a template. Trace the input to the interpreter and the effect. Cross-reference `ATTACK-CLASSES.md#Injection`.

## Code-loading and reflection attack classes (subagent_type: `general`)

**Dynamic code and class loading**
`DexClassLoader`/`PathClassLoader` over a file an attacker can influence (external storage, a downloaded module, an unverified dynamic-feature or plugin), `Class.forName`/reflection with an input-derived class name, or loading native libraries from a non-system path. Check the integrity and location of what is loaded and who can write it. Loading attacker-writable code is a high-impact class.

**Reflection-driven control**
Input selects a method, field, or class via reflection and thereby reaches functionality the UI never exposes or bypasses a check. Check the mapping from input to reflected target and the authority gained.

## Concurrency and lifecycle attack classes (subagent_type: `general`)

**Check-then-act and TOCTOU**
A security check and the guarded action are not atomic across coroutines, threads, `WorkManager` jobs, or file operations, so a concurrent request or a swapped file changes the decision between check and use (double-spend, double-redeem, stale authorization). Check synchronization around the guarded state and whether an attacker can realistically trigger the interleaving.

**Shared mutable state and cancellation**
A singleton, cached credential, or coroutine scope shares state across users or requests without isolation, or a cancellation/partial-failure path leaves state half-updated (a step committed without its guard). Check per-request isolation and rollback on failure.

## Universal moves (apply across the above)

- Start from each untrusted input identified in reconnaissance, name the tier, and trace through the parser to the sink or the security decision.
- For deserialization and reflection, state the exact type/class/field the attacker controls and the dangerous thing it reaches. For concurrency, state the interleaving and the wrong state.
- Validate with a JVM or Robolectric harness that feeds the parser a dummy malformed input and observes the wrong result, or a deterministic concurrency test. A JNI handoff is returned as `uncovered`, not tested here.

## Validation rules (apply before reporting ANY finding here)

1. Name the untrusted input and its tier, the parser or dispatcher, the missing validation, and the concrete sink or decision reached.
2. Do not escalate a crash into code execution or a parse quirk into a stronger effect than observed. A malformed-extra crash is low unless persistent or boundary-crossing.
3. For dynamic code loading, cite who can write the loaded artifact and why it is reachable; this is only high when attacker-writable code executes in the app.
4. Keep local checks to a bounded harness with dummy input. Stop at the first wrong result. Return a native handoff as `uncovered` with the symbol.
5. Return `confirmed` for a source path plus a bounded harness result. Return `needs_validation` when reachability depends on an unobservable manifest, minSdk, or runtime fact.
