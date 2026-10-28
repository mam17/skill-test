# API-First Test Planning

#### When to use this file

Whenever the app has a remote API (Retrofit/Ktor interfaces, an OpenAPI spec such as `openapitools.json`, Firebase callable functions, GraphQL). This file produces the fast-feedback test plan the user asked for: **validate the API contract first, then the layers that consume it, from the outside in.** API tests are the cheapest to run and the first to invalidate everything downstream, so they run before unit tests of internal functions and long before UI flows.

Execution safety: contract tests run against a local mock (`MockWebServer`, WireMock, the repo's own fakes) or an owner-designated non-production environment. Never run the plan against production, with real accounts, or with real payments/push.

## Step 0 — Discover the API surface

From source, inventory every remote call before writing any test:

- Retrofit/Ktor interfaces: each function = method, path, query/body model, response model, auth header (interceptors), timeout.
- Generated clients: an OpenAPI config (`openapitools.json`) means the spec is the contract source of truth — cite the spec file and whether the generated code is committed or generated at build.
- Other channels: Firebase (Firestore/Functions/Remote Config/FCM payload shapes), WebSocket/SSE message types.
- For each endpoint record: calling repository/usecase, DTO models, error mapping, and which flows depend on it. An endpoint no flow depends on is dead surface — note it, don't test it first.

Produce the inventory table:

```
endpoint | method | req/resp models | auth | consumed by (repo -> usecase -> screen) | flow priority | spec source
```

## Step 1 — API contract tests (fastest, run first)

JVM-only tests with `MockWebServer` (no emulator, seconds to run). For each endpoint in flow-priority order:

1. **Request shape**: the client sends the right method, path, query, headers (auth interceptor attaches the key/token), and serialized body. Assert on `mockWebServer.takeRequest()`.
2. **Response happy path**: a recorded/spec-derived JSON body deserializes into the DTO with every field the app actually reads (nullability mismatches between JSON and Kotlin models are the top crash source — a non-null Kotlin field the server can omit is an immediate finding).
3. **Contract tolerance**: unknown extra fields are ignored (forward compatibility); enum/discriminator values outside the known set fail soft, not crash.
4. **Error paths**: 400/401/403/404/429/500 and malformed JSON each map to the app's error type, not an unhandled exception. Timeouts and connection failures map to the offline path (ties into FLOW-REVIEW item 4).
5. **Spec drift** (when an OpenAPI spec exists): regenerate or diff models against the spec; committed generated code that drifted from the spec is a finding.

Exit criterion: every core-flow endpoint has request-shape + happy-path + error-path tests green against the mock. Only then move inward.

## Step 2 — Inward: the functions that consume the API

Now test the internal call chain, still JVM-first:

1. **Repositories**: with the API mocked at the interface level, assert mapping DTO→domain (unit conversions, date parsing, null→default policies), caching behavior, and error propagation (the repository does not swallow errors the UI needs).
2. **UseCases/domain**: business rules with fake repositories — thresholds, combination logic, retry/backoff decisions.
3. **ViewModels**: state emission per scenario (loading→success, loading→error→retry), one-shot event behavior, `SavedStateHandle` restore. Use the error scenarios proven in Step 1 as the fixture set so the layers are tested against the same contract.
4. **Serialization round-trips** for anything persisted (Room entities, DataStore protos) that mirrors API models.

## Step 3 — Then the flows

Only after Steps 1–2 are green, run the UI flows per [FLOW-REVIEW.md](FLOW-REVIEW.md) with the mock server providing the same fixtures — instrumentation/Espresso for core flows, manual emulator passes per [DYNAMIC-UI-CHECKS.md](DYNAMIC-UI-CHECKS.md). A UI failure at this point is a UI defect, not an API ambiguity, because the contract is already pinned.

## Plan output format

Deliver the plan as a table the team can execute top-down:

```
# | layer (API/repo/usecase/VM/UI) | target | scenarios | tooling | depends on | est. runtime
```

ordered so that everything above a row must pass before the row is worth running. Flag endpoints with no error mapping, DTOs with risky nullability, and spec drift as findings (severity by the consuming flow's priority), not just as test gaps.

## When the backend is not reachable from the repo

If `BASE_URL` is a placeholder or the spec is absent, say so: the plan is still written (Step 0 inventory + Step 1 test skeletons against the mock), but contract fidelity against the real server is **needs-owner-input** — ask the owner for the spec or a staging base URL. Never guess the contract from field names alone without flagging it.
