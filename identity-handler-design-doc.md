# FedCM Identity Handler Service Worker

## Design Doc

**This Document is Public**

Authors: sureshpotti@ (Microsoft Edge) — June 2026

CL: [chromium-review #7974037](https://chromium-review.googlesource.com/c/chromium/src/+/7974037) — *[FedCM]: Add Identity Handler Service Worker interception*

---

## One-page overview

### Summary

The FedCM Identity Handler lets an IDP declare a Service Worker that the browser
dispatches FedCM's **credentialed** requests to — `accounts` (GET),
`id_assertion`/`token` (POST), and `disconnect` (POST) — *before* they reach the
network. The handler is a normal Service Worker that listens for a single new
event, `identityrequest`, reads `event.endpoint` and `event.request`, and calls
`event.respondWith(Promise<Response>)`, just like a `fetch` handler. FedCM then
processes that response through its existing internals.

This gives the IDP a programmable seam on FedCM's network path so it can attach a
fresh device-bound proof-of-possession header (e.g., DPoP), serve last-known-good
account data during an outage, route to a region for data-residency, or bridge a
non-FedCM backend into FedCM's request/response shape. The feature is purely
**additive and opt-in**: RPs need no changes, the configuration/discovery
endpoints (`/.well-known/web-identity`, `config.json`, `client_metadata`) are
never dispatched to the handler, and **any handler failure transparently falls
back** to the existing direct-network FedCM flow.

### Platforms

All Blink platforms: Mac, Windows, Linux, Chrome OS, Fuchsia, Android, Android
WebView. Not iOS (non-Blink). No `//chrome` code required; the feature lives in
`//content` (browser) and `//third_party/blink` (renderer). Gated behind the
`FedCmIdentityHandler` runtime feature and the `kFedCmIdentityHandler` base
feature (off by default).

### Team

sureshpotti@ (Microsoft Edge), with the FedCM editors and the Service Worker
editors.

### Bug

[crbug.com/526074797](https://issues.chromium.org/issues/526074797)
(Blink>Identity>FedCM).

### Code affected

* **Blink renderer — new `IdentityRequestEvent`**
  (`third_party/blink/renderer/modules/credentialmanagement/identity_request_event.{idl,h,cc}`,
  `identity_request_event_init.idl`, `identity_request_endpoint.idl`,
  `identity_request_service_worker_global_scope.{idl,h}`,
  `identity_request_respond_with_observer.{h,cc}`): the new event exposed on the
  Service Worker global scope and its `respondWith()` observer.
* **Blink Service Worker plumbing**
  (`service_worker_global_scope.{h,cc}`, `event_type_names.json5`,
  `wait_until_observer.{h,cc}`, `service_worker.mojom`): a new
  `DispatchIdentityRequestEvent` entry point that constructs and fires the event
  inside the SW global scope.
* **Browser — registration & dispatch** (`content/browser/webid/`):
  `identity_handler_service_worker_manager.{h,cc}` (register/unregister the SW
  under an isolated, recomputable FedCM-managed `StorageKey`),
  `identity_request_event_dispatcher.{h,cc}` (dispatch the event to the SW and
  apply the timeout/fallback policy), `idp_network_request_manager.{h,cc}`
  (per-IDP-origin handler state and SW-first-then-network routing of each
  credentialed endpoint), and `accounts_fetcher.{h,cc}`.
* **Mojo** (`fedcm_identity_request.mojom`): `IdentityRequestEndpoint` enum,
  `IdentityRequestEventData`, and the `IdentityRequestResponseCallback`
  interface (`OnResponse` / `OnError` / `OnNotHandled`).
* **Flags & metrics**: `content/public/common/content_features.{h,cc}`,
  `content/browser/webid/flags.{h,cc}`, `runtime_enabled_features.json5`,
  `content/browser/webid/metrics.{h,cc}` and the histograms metadata.

---

## Design

### Background & motivation

FedCM lets a user sign in to an RP with a federated IDP without third-party
cookies by making credentialed network requests (`accounts`, `id_assertion`,
`disconnect`) on the user's behalf. Today these are made with
`service-workers mode: "none"`, so they go straight to the network and the IDP
has **no programmable seam** between FedCM and its own network stack. Four gaps
follow:

1. **Bearer-token theft.** FedCM's IDP cookies are bearer credentials. Modern
   enterprise IDPs require the cookie to be accompanied by a fresh, device-bound
   proof-of-possession assertion (e.g., Entra's `x-ms-RefreshTokenCredential`
   JWT). FedCM has no place to attach that proof, so today it only works via a
   browser extension or a non-standard API.
2. **Hard failures during IDP outages.** When the IDP backend is degraded, the
   FedCM call fails outright; the IDP cannot apply its usual resiliency patterns
   (regional failover, last-known-good cache) to FedCM's calls.
3. **Sovereignty-aware routing.** `config.json` declares one set of endpoint URLs
   per origin and is fetched without user context, so the IDP cannot pick a
   regional cell for the actual signed-in user before `accounts` goes out.
4. **Bridging non-FedCM backends.** An IDP whose backend wire format differs from
   FedCM's must deploy a server-side translation tier in front of every endpoint.

The Identity Handler closes all four by reusing the Service Worker model the
platform already ships — a normal SW responding to an event with a `Response` —
restricted to FedCM's credentialed endpoints and to the IDP's own origin.

### Links

* Explainer: `identity-handler-explainer.md` (this repo).
* Spec draft (Bikeshed): `index.bs` (this repo).
* Discussion: [FedCM Issue #80](https://github.com/w3c-fedid/FedCM/issues/80).
* FedCM: <https://fedidcg.github.io/FedCM/> · Service Worker:
  <https://w3c.github.io/ServiceWorker/> · Fetch:
  <https://fetch.spec.whatwg.org/>.

### The dispatch model (the "handshake")

```
IDP config.json:
  { ...,
    "identity_handler": { "service_worker": "/fedcm/sw.js" } }

Opt-in signal:  presence of identity_handler.service_worker

On a FedCM call, for each credentialed endpoint:
  Browser → IdentityRequestEvent { endpoint, request } → IDP Service Worker
  Service Worker → event.respondWith(Promise<Response>)
  Browser ← OnResponse(status, body, headers) → FedCM internals

Outcomes (per endpoint):
  (a) respondWith() resolves with a valid Response  → FedCM uses it.
  (b) respondWith() rejects / times out / invalid   → request FAILS (no fallback).
  (c) handler never calls respondWith()             → fall back to credentialed
                                                       network fetch.
  (d) no/!ready registration, worker fails to start → fall back to network.
```

Only **credentialed** endpoints are dispatched. Configuration/discovery
endpoints (`/.well-known/web-identity`, `config.json`, `client_metadata`) stay
UA-direct to preserve FedCM's privacy boundary.

> **Note (current patchset):** registration is driven from the IDP's
> `config.json` `identity_handler.service_worker` member. The explainer's
> longer-term direction is UA/FedCM-managed registration declared in
> `/.well-known/web-identity` (RP-context-free); that migration is tracked
> separately and does not change the dispatch/runtime design below.

### Data flow (threads / processes / network)

**In the renderer (the IDP's Service Worker process).** A new
`IdentityRequestEvent` (gated on the `FedCmIdentityHandler` runtime feature,
`Exposed=ServiceWorker`, extends `ExtendableEvent`) carries a read-only
`endpoint` (`IdentityRequestEndpoint`) and a read-only `Request`, and exposes
`[CallWith=ScriptState, RaisesException] respondWith(Promise<Response>)`. It is
modeled directly on `FetchEvent`. `ServiceWorkerGlobalScope::DispatchIdentityRequestEvent`
constructs the `Request` from the browser-supplied `IdentityRequestEventData`
(url/method/body), fires the event, and an `IdentityRequestRespondWithObserver`
(a `WaitUntilObserver`-style observer) tracks the `respondWith()` promise. When
it settles, the resolved `Response` (status, body, headers) is sent back over the
`IdentityRequestResponseCallback` Mojo interface — `OnResponse` on success,
`OnError` on rejection/invalid response, `OnNotHandled` if `respondWith()` was
never called.

**In the browser process (the trust boundary; all decisions live here, on the UI
thread).**

1. **Registration.** `IdentityHandlerServiceWorkerManager::Register(sw_url)`
   registers the SW with scope `sw_url.GetWithoutFilename()` under an
   **isolated, FedCM-managed `StorageKey`** returned by
   `GetFedCmStorageKey(idp_origin)` — a nonce-based key whose nonce is derived
   deterministically (SHA-1 over `"fedcm-identity-handler-v1:" + origin` → a
   `base::UnguessableToken` → `StorageKey::CreateWithNonce`). Because page
   JavaScript can only register into a first-party key, this FedCM partition is
   unreachable from the page, so the FedCM registration can never collide with —
   or clobber — the IDP's ordinary site-wide Service Workers; and because the
   key is recomputed from the IDP origin, no per-instance state has to be
   persisted to find it again later. On success it finds the ready
   registration and calls
   `IdpNetworkRequestManager::SetIdentityHandlerDispatcher(idp_origin,
   dispatcher, sw_context, registration_id)`, recording
   `Blink.FedCm.IdentityHandler.Registered`. Registration failure simply
   proceeds without interception (graceful no-op).
2. **Per-IDP routing state.** `IdpNetworkRequestManager` keeps a
   `flat_map<url::Origin, IdentityHandlerState>` where each
   `IdentityHandlerState` holds `{ dispatcher, sw_context, registration_id }`.
   FedCM supports multiple IDPs per request, so handler state is **keyed by IDP
   origin**; an IDP without a handler falls through to the network.
3. **Dispatch.** For a credentialed endpoint to `idp_origin`, if a handler state
   exists, `IdentityRequestEventDispatcher::DispatchIdentityRequestEvent` calls
   `FindReadyRegistrationForIdOnly(registration_id)`, then
   `RunAfterStartWorker(IDENTITY_REQUEST, …)`, then
   `active_version->endpoint()->DispatchIdentityRequestEvent(event_data,
   response_callback, …)`. The response callback is a self-owned Mojo receiver.
4. **Timeout & fallback.** A delayed task (default **10 s**) closes the Mojo pipe
   if `respondWith()` has not settled; closing destroys the response-callback
   impl, whose destructor fires `kRespondWithFailed` and adds a console warning.
   The dispatcher maps outcomes to a `DispatchFailureReason`:
   `kNotHandled` / `kNoRegistration` / `kNoActiveVersion` / `kWorkerStartFailed`
   → **fall back** to a credentialed network fetch
   (`Blink.FedCm.IdentityHandler.FellBackToNetwork`); `kRespondWithFailed` →
   **fail** the request without falling back. Successful dispatch records
   `Blink.FedCm.IdentityHandler.Dispatched`.
5. **Unregistration.** `Unregister(idp_origin)` is fire-and-forget: it
   recomputes the FedCM-managed `StorageKey` from `idp_origin`, enumerates the
   registrations under that key (`GetRegistrationsForStorageKey`), and removes
   them all. Every registration in that partition was created by FedCM
   (page-initiated registrations live in the IDP's first-party key and can never
   land there), so legitimate non-FedCM Service Workers are never touched — and
   because the key is recomputed rather than read from per-instance state,
   cleanup works even from a freshly constructed manager in a later FedCM flow.

**Over Mojo.** `IdentityRequestEventData` (endpoint, request_url, optional
request_body, request_method) flows browser→renderer;
`IdentityRequestResponseCallback` (`OnResponse`/`OnError`/`OnNotHandled`) flows
renderer→browser.

### Component responsibilities

* **Renderer / Service Worker** only constructs and dispatches the event and
  reports the `respondWith()` outcome; it makes **no** trust or fallback
  decision and cannot reach configuration/discovery endpoints.
* **`IdentityHandlerServiceWorkerManager`** owns SW lifecycle (register on
  opt-in under an isolated FedCM-managed `StorageKey`, stateless recomputable
  unregister) and wires the dispatcher into the network manager keyed by IDP
  origin.
* **`IdpNetworkRequestManager` + `IdentityRequestEventDispatcher`** own routing
  (per-origin state), worker startup, the timeout, and the
  fail-vs-fall-back-vs-network decision for every credentialed endpoint.

### Key property: handler failures never block authentication

Sign-in must not become *less* reliable than today. Every path that is not a
clean valid `Response` either falls back to the existing direct-network FedCM
fetch (no registration, worker won't start, `respondWith()` not called) or fails
the single request deterministically (`respondWith()` rejected/timed-out/invalid)
— and the configuration/discovery endpoints bypass the handler entirely. The
handler is therefore strictly additive to FedCM's existing flow.

---

## Metrics

### Success metrics

* `Blink.FedCm.IdentityHandler.Registered` (boolean) — handler registration
  succeeded vs. failed, sizing IDP adoption and registration health.
* `Blink.FedCm.IdentityHandler.Dispatched` (boolean) — a credentialed request
  was successfully dispatched to the handler, confirming the mechanism fires.
* `Blink.FedCm.IdentityHandler.FellBackToNetwork` (boolean) — a dispatch
  degraded to the direct-network path, confirming the fail-safe works and
  watching for unexpectedly high fallback (a handler-quality signal).

### Regression metrics

* Existing FedCM funnel/latency metrics on pages whose IDP does **not** declare a
  handler — should stay flat, since all work is behind the feature flag and only
  runs when `identity_handler` is present.
* Fallback rate (above) watched as a proxy for handler-induced sign-in friction.

### Experiments

None at the prototype stage. Behind the `FedCmIdentityHandler` runtime feature
and the `kFedCmIdentityHandler` base feature; no Finch experiment is defined yet.

---

## Rollout plan

Prototype behind flags (off by default). Not targeting a milestone yet. The
registration mechanism is expected to migrate from `config.json` to a
UA/FedCM-managed `/.well-known/web-identity` declaration before any broader
rollout. Any future rollout would be experiment-controlled (Origin Trial or other
staged mechanism).

---

## Core principle considerations

### Speed

Work is added only when an IDP declares `identity_handler` and the flag is on:
one SW registration lookup, worker start (amortized by Soft Update / SW
lifetime), and a per-endpoint event dispatch with a bounded 10 s timeout. Zero
added work on the hot path of a FedCM flow with no handler or when the flag is
off. A handler that augments headers and re-fetches adds one same-origin network
round trip versus the direct fetch it replaces.

### Simplicity

No new end-user UI and no new user decisions — the FedCM account chooser and
consent UI are unchanged. The surface is developer-facing and intentionally
mirrors `FetchEvent`: an `identityrequest` event with `endpoint` / `request` and
`respondWith(Promise<Response>)`, so authors who know Service Workers already
understand the model. The feature is opt-in for the IDP (declare
`identity_handler`) and invisible to RPs (`navigator.credentials.get` is
unchanged).

### Security

Threat model: the goal is to give the IDP a seam on **its own** credentialed
FedCM path without weakening FedCM's privacy boundary or sign-in reliability.

* **Origin isolation.** The handler is registered at the IDP origin under an
  isolated, FedCM-managed (nonce-based) `StorageKey` and can only ever see
  requests destined for **its own origin**; cross-origin interception is
  architecturally impossible. The isolated key keeps FedCM's registration in a
  partition page JavaScript cannot reach, so it never collides with or clobbers
  the IDP's first-party Service Workers. State is keyed per IDP origin so one
  IDP's handler cannot observe another's requests.
* **Selective scope.** Only credentialed endpoints (`accounts`, `id_assertion`,
  `disconnect`) are dispatched; `/.well-known`, `config.json`, and
  `client_metadata` are never exposed to the handler, so it cannot correlate
  user identity with RP identity.
* **CSRF / redirect.** The handler-initiated `fetch` is same-origin and carries
  `Sec-Fetch-Site: same-origin`, which IDP servers verify; requiring same-origin
  requests preserves FedCM's `redirect mode: "error"` guarantee at the server
  layer.
* **Header augmentation.** The handler may add headers (e.g., `DPoP`) but must
  not override UA-set headers (`Accept`, `Sec-Fetch-*`, `Cookie`, `Origin`,
  `Referer`); the allowlist is being finalized.
* **No consent bypass.** The handler operates only on the network layer; FedCM
  consent is still required before any token is issued.
* **Robust fallback.** The timeout + skip/fallback pattern guarantees a handler
  failure degrades to the direct-network path (or fails one request) rather than
  breaking sign-in.

### Privacy considerations

No new persistent state and no new fingerprinting surface. The handler runs at
the IDP origin and sees only what the IDP server already sees: `accounts` (GET)
reveals no RP identity (opaque origin); `id_assertion`/`disconnect` (POST) carry
the RP `client_id` in the body, which the IDP server already receives. Excluding
the configuration/discovery endpoints prevents the handler from learning *which
RP* the user is interacting with. Driving registration from an RP-context-free
source (the well-known, in the target design) prevents encoding RP identity into
the SW URL.

---

## Testing plan

* **Renderer unit tests** —
  `identity_request_event_test.cc`: `IdentityRequestEvent` construction,
  `endpoint`/`request` accessors, and `respondWith()` behavior.
* **Browser unit tests** —
  `identity_handler_service_worker_manager_unittest.cc`: registration success/
  failure, dispatcher wiring, isolated-`StorageKey` registration/unregistration
  (FedCM SWs partitioned from the IDP's first-party SWs, which are left intact),
  and the dispatch failure-reason → fail/fallback/network mapping.
* **WPT** (`external/wpt/fedcm/fedcm-identity-handler/`):
  `identity-request-event-constructor.https.*` (event construction in a SW),
  `same-origin.https.html` (handler intercepts and responds),
  `cross-origin.https.html` (cross-origin requests are not intercepted),
  `respond-with-failure.https.html` (rejection/timeout fail without fallback),
  plus the `support/` manifest, registration page, and sample handler SW. A
  `virtual/fedcm-identity-handler/` suite runs these with the feature enabled.

---

## Followup work

* Migrate registration to UA/FedCM-managed `/.well-known/web-identity`
  declaration and resolve the open operational sub-questions (onboarding
  already-signed-in users, forced bad-rollout recovery, behavior after storage
  eviction, well-known fetch reliability).
* Finalize the header-augmentation allowlist and the process for growing it.
* Standards work: dedicated-event spec text, security/privacy and TAG review, and
  cross-vendor signals.

**Success assessment:** correctness of the dispatch/fallback contract (covered by
unit + WPT tests), implementer/standards feedback on the FedCM and Service Worker
threads, and — before any ship — adoption (`Registered`/`Dispatched`) and
fallback-rate metrics.
