# Identity Handler for FedCM

## Authors:

- [@sureshpotti](https://github.com/sureshpotti)

## Participate
- [Issue tracker — FedCM Issue #80](https://github.com/w3c-fedid/FedCM/issues/80)

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- END doctoc generated TOC please keep comment here to allow auto update -->

- [Introduction](#introduction)
- [User-Facing Problem](#user-facing-problem)
  - [Goals](#goals)
  - [Non-goals](#non-goals)
- [Proposed Approach](#proposed-approach)
  - [Service Worker registration (UA / FedCM-managed)](#service-worker-registration-ua--fedcm-managed)
  - [Dispatch model](#dispatch-model)
  - [Flow (on each FedCM invocation)](#flow-on-each-fedcm-invocation)
  - [Selective endpoint policy](#selective-endpoint-policy)
  - [Transparent fallback](#transparent-fallback)
  - [Dependencies on non-stable features](#dependencies-on-non-stable-features)
  - [Solving token-binding with this approach](#solving-token-binding-with-this-approach)
  - [Solving outage resiliency with this approach](#solving-outage-resiliency-with-this-approach)
  - [Solving sovereignty-aware routing with this approach](#solving-sovereignty-aware-routing-with-this-approach)
  - [Solving non-FedCM backend bridging with this approach](#solving-non-fedcm-backend-bridging-with-this-approach)
- [Alternatives considered](#alternatives-considered)
- [Accessibility, Internationalization, Privacy, and Security Considerations](#accessibility-internationalization-privacy-and-security-considerations)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)
- [Appendix A: WebIDL (Chromium / Blink form)](#appendix-a-webidl-chromium--blink-form)

## Introduction

[FedCM](https://fedidcg.github.io/FedCM/) lets a user sign in to a website (the Relying Party, or RP) with a federated identity provider (IDP) without third-party cookies. To do this, the browser makes a set of credentialed network requests to the IDP — `accounts`, `id_assertion`, and `disconnect` — on the user's behalf. Today these requests are made with `service-workers mode: "none"`, so they go straight to the network and the IDP has no programmable seam between FedCM and its own network stack.

This proposal adds an **opt-in Identity Handler Service Worker**, hosted at the IDP origin, that the browser dispatches FedCM's credentialed calls to before they reach the network. The handler can attach a fresh proof-of-possession header, serve cached account data during an outage, route a request to the correct region, or translate between FedCM and a non-FedCM identity backend. The feature is purely additive: RPs need no changes, configuration/discovery endpoints are never exposed to the handler, and any handler failure transparently falls back to the existing direct-network path.

## User-Facing Problem

**An end user wants to sign in to a website with their identity provider and have it _just work_ — securely, and even when their provider is partly degraded or operates under regional data-residency rules.** Today, four gaps in FedCM make that harder than it should be.

The connection to the end user is partly indirect (these are security and reliability properties of the sign-in network path), so each problem below states the user-visible consequence.

**1. Bearer-token theft on the IDP cookie.** The cookies FedCM sends on `accounts` and `id_assertion` are **bearer credentials** — anyone who copies the bytes can replay them, often for the lifetime of the session (months). Modern enterprise IDPs increasingly require the cookie to be accompanied by a fresh, device-bound proof-of-possession assertion. Microsoft Entra, for instance, only honors its `ESTSAUTH*` cookies when an `x-ms-RefreshTokenCredential` JWT — signed by a TPM-backed device key — is attached. FedCM as specified gives no place to attach that proof, so today this only works via a browser extension (Chrome) or a non-standard API (Edge). _User impact: a stolen session cookie can impersonate the user._

**2. Hard failures during IDP outages.** When the IDP backend is degraded — a regional incident, a partial accounts-service outage, a maintenance window — the FedCM call fails outright. The IDP has no chance to apply the resiliency patterns (regional failover, last-known-good cached account data) it routinely uses on its own surface. _User impact: sign-in fails even though the user is already authenticated._

**3. Geographic / sovereignty-aware routing.** Some IDPs must route a user's authentication traffic to a specific regional service for data-residency reasons. `config.json` declares a single set of endpoint URLs per origin and is fetched without user context, so there is no point at which the IDP can pick a region for the actual signed-in user before the `accounts` call goes out. _User impact: sign-in can violate the user's data-residency expectations, or simply fail._

**4. Bridging non-FedCM identity backends.** An IDP whose existing backend wire format does not match FedCM's `accounts` / `id_assertion` shape must deploy a server-side translation layer in front of every endpoint. This is operationally heavy and slows adoption. _User impact: fewer providers offer the privacy-preserving FedCM sign-in experience._

### Goals

- Let an IDP attach a fresh, device-bound **proof-of-possession header** (e.g., DPoP) to credentialed FedCM requests, so a stolen cookie alone is not enough to impersonate the user.
- Let an IDP apply **outage resiliency** (failover, last-known-good cache) to FedCM's credentialed calls.
- Let an IDP perform **user-context-aware regional routing** before the credentialed call goes to the network.
- Let an IDP **bridge a non-FedCM backend** into FedCM's request/response shape without a separate server-side translation tier.
- Require **zero changes from RPs** and preserve FedCM's existing privacy boundary.
- **Fail safe**: any handler error degrades transparently to today's direct-network FedCM flow.

### Non-goals

- Exposing FedCM's **configuration / discovery** endpoints (`/.well-known/web-identity`, `config.json`, `client_metadata`) to the handler. These stay UA-direct to preserve the privacy boundary.
- Allowing the handler to **bypass or alter the FedCM consent UI**. The handler operates only on the network layer; user consent is still required before any token is issued.
- Cross-origin interception. A handler can only ever see requests destined for **its own IDP origin**.
- Replacing RP-side token-binding mechanisms (e.g., DPoP-bound RP tokens obtainable via an OAuth profile); this proposal targets the **IDP-cookie** bearer problem.

## Proposed Approach

We introduce an opt-in **Identity Handler** Service Worker, registered at the IDP origin, that the user agent invokes for FedCM's credentialed endpoints before going to the network. The handler is a normal Service Worker that listens for a single new event, `identityrequest`, and responds with a `Response` the way a `fetch` handler does. WebIDL for the new event lives in [Appendix A](#appendix-a-webidl-chromium--blink-form) so this section stays focused on behavior and example code.

### Service Worker registration (UA / FedCM-managed)

Registration is **UA-managed (FedCM-managed)**: the user agent — not a script on an IDP page — registers and maintains the handler. This matters because a signed-in IDP session can stay valid for weeks or months (Entra allows up to ~90 days), during which FedCM keeps making credentialed calls without the user ever revisiting an IDP page. A page-script registration would not reliably exist when needed.

The IDP declares the handler in **`/.well-known/web-identity`** using an `identity-handler` member with a `service-worker` member:

```json
{
  "provider_urls": [
    "https://idp.example/fedcm/config.json"
  ],
  "identity-handler": {
    "service-worker": "/fedcm/sw.js"
  }
}
```

- The presence of `identity-handler` is the IDP's **opt-in** signal.
- `service-worker` is the script URL the UA registers.

UA/FedCM-managed registration via the well-known `identity-handler` member is the current direction. A handful of operational sub-questions — onboarding already-signed-in users, forced bad-rollout recovery, behavior after storage eviction, and well-known fetch reliability — remain to be finalized.

### Dispatch model

When the browser needs to fetch a credentialed IDP endpoint (`accounts`, `id_assertion`, `disconnect`), it dispatches a purpose-built `identityrequest` event to the registered handler using the Service Worker spec's [_Fire Functional Event … on registration_](https://www.w3.org/TR/service-workers/#fire-functional-event) primitive (a UA-initiated request, a destination-origin SW, and **no controlling client**). Because the request is sent over this trusted, FedCM-only path — not with the RP's [request client](https://fetch.spec.whatwg.org/#concept-request-client) — it does not reintroduce the generic cross-origin SW invocation that "foreign fetch" was removed to prevent.

The handler reads `event.endpoint` to learn which credentialed call this is, inspects `event.request`, and calls `event.respondWith()` with a `Promise<Response>`. FedCM processes that response in a FedCM-specific way (e.g., parsing the accounts JSON) and feeds it into the existing FedCM internals.

### Flow (on each FedCM invocation)

On each FedCM invocation the UA:

1. Fetches the well-known and reads the `identity-handler` declaration.
2. Runs _Match Service Worker Registration_ against a **FedCM-managed, isolated `StorageKey`** (nonce-based) so the FedCM registration can never collide with a page-initiated `navigator.serviceWorker.register()` for the same path.
3. If a matching registration exists for the declared origin and `service-worker` URL, it dispatches the `identityrequest` event to it and schedules a [Soft Update](https://www.w3.org/TR/service-workers/#soft-update) **in parallel** (governed by HTTP cache freshness, so the IDP keeps its normal staged-rollout/rollback strategy).
4. Otherwise it registers and activates the declared handler, then dispatches.

### Selective endpoint policy

Only **credentialed** endpoints are dispatched to the handler. Configuration / discovery endpoints stay UA-direct to preserve the existing privacy boundary.

| Endpoint | Dispatched to handler | Reason |
|----------|:--:|--------|
| `/accounts` (GET) | ✅ | User account data — benefits from caching and augmentation |
| `/token` (id_assertion, POST) | ✅ | Token generation — can attach DPoP, custom logic |
| `/disconnect` (POST) | ✅ | Account management — graceful error handling, backend bridging |
| `/.well-known/web-identity` | ❌ | IDP discovery — privacy protection |
| `/config.json` | ❌ | Configuration — privacy protection |
| `/client_metadata` | ❌ | RP metadata — privacy protection |

This credentialed-endpoints-only scoping is the result of explicit attack analysis on [Issue #80](https://github.com/w3c-fedid/FedCM/issues/80) and cleared WG privacy review.

### Transparent fallback

If the handler does not call `respondWith()`, rejects the promise, returns a non-OK response, or times out (default 10 seconds), the browser **transparently falls back** to a normal network fetch using a "skip flag" pattern: emit a console warning, set a `skip_identity_handler` flag, re-issue the same request UA-direct, then clear the flag. Handler failures never block authentication.

### Dependencies on non-stable features

- The dispatch primitive ([_Fire Functional Event … on registration_](https://www.w3.org/TR/service-workers/#fire-functional-event)) is a stable Service Worker concept.
- The new `identityrequest` event ([Appendix A](#appendix-a-webidl-chromium--blink-form)) and the UA-managed registration model are **new and not yet shipped**.

### Solving token-binding with this approach

A stolen IDP cookie is no longer enough on its own: the handler attaches a fresh, device-bound proof to the `id_assertion` call.

```js
// /fedcm/sw.js — IDP's Identity Handler Service Worker
self.addEventListener('identityrequest', (event) => {
  if (event.endpoint === 'id_assertion') {
    event.respondWith((async () => {
      const proof = await generateDPoPProof(event.request.url, 'POST');
      const augmented = new Request(event.request, {
        headers: { ...event.request.headers, 'DPoP': proof }
      });
      return fetch(augmented); // same-origin → carries Sec-Fetch-Site: same-origin
    })());
  }
});
```

### Solving outage resiliency with this approach

The handler tries the network and falls back to a recent cache, so a degraded IDP backend no longer means a sign-in failure.

```js
self.addEventListener('identityrequest', (event) => {
  if (event.endpoint === 'accounts') {
    event.respondWith(
      fetch(event.request)
        .then(response => {
          if (response.ok) {
            caches.open('fedcm').then(c => c.put(event.request, response.clone()));
          }
          return response;
        })
        .catch(() => caches.match(event.request))
    );
  }
});
```

### Solving sovereignty-aware routing with this approach

Because the handler runs with the user's IDP context, it can pick the correct regional endpoint for the actual signed-in user before the request leaves the device.

```js
self.addEventListener('identityrequest', (event) => {
  if (event.endpoint === 'accounts') {
    event.respondWith((async () => {
      const region = await resolveUserRegion();         // e.g., "eu"
      const url = new URL(event.request.url);
      url.hostname = `${region}.idp.example`;            // same-origin family, regional cell
      return fetch(new Request(url, event.request));
    })());
  }
});
```

### Solving non-FedCM backend bridging with this approach

The handler can synthesize a FedCM-shaped response from a non-FedCM backend, avoiding a separate server-side translation tier.

```js
self.addEventListener('identityrequest', (event) => {
  if (event.endpoint === 'disconnect') {
    event.respondWith((async () => {
      const body = await event.request.clone().text();
      const accountId = new URLSearchParams(body).get('account_hint');
      const upstream = await fetch('/internal/revoke', { method: 'POST', body });
      if (!upstream.ok) return upstream;
      return Response.json({ account_id: accountId }); // FedCM-shaped response
    })());
  }
});
```

**RPs require zero changes** — the FedCM call is unchanged:

```js
const credential = await navigator.credentials.get({
  identity: {
    providers: [{
      configURL: "https://idp.example/fedcm/config.json",
      clientId: "rp_client_123",
      nonce: "random_nonce_value"
    }]
  }
});
```

## Alternatives considered

### Approach A — Flip `service-workers mode` to `"all"`

The most conservative change: replace `service-workers mode: "none"` with `"all"` on FedCM's credentialed requests and let standard SW dispatch handle it.

#### Pros
- Smallest apparent spec delta; reuses existing `FetchEvent`.

#### Cons
- The Service Worker spec's [_Handle Fetch_](https://www.w3.org/TR/service-workers/#handle-fetch) dispatches via the request's **client**, but every FedCM credentialed endpoint sets `client = null`.

#### Reason for rejection
_Handle Fetch_ has nothing to dispatch through, so the request goes straight to the network. **It cannot work as a spec-only change.**

### Approach B — Reuse `FetchEvent` via a browser-side dispatch layer

Ship a FedCM-specific dispatch layer that bypasses _Handle Fetch_: look up the IDP's SW by URL scope, construct an internal request, and invoke its `fetch` event directly.

#### Pros
- Prototyped and implementable today; no new event type to learn.

#### Cons
- Spec direction (extending Fetch/SW to dispatch a `FetchEvent` to a SW with no controlling client) was **not endorsed** on the [Chromium service-worker-discuss thread](https://groups.google.com/a/chromium.org/g/service-worker-discuss/c/t9d33x6l718): it echoes the deprecated [foreign fetch](https://github.com/whatwg/fetch/issues/506) "no initiating client" shape (Ben Kelly).
- Any pre-existing IDP SW with a generic `fetch` handler would **suddenly receive FedCM requests** it was never written to handle.
- Formal spec design, security/privacy review, cross-vendor review, and TAG review were called out as prerequisites (Dominic Farolino).

#### Reason for rejection
Silently overloading existing `fetch` handlers is risky, and the spec extension needed to dispatch a client-less `FetchEvent` was not endorsed.

### Registration shape — SW URL in `config.json` (backed out)

Declaring the handler script inside `config.json` (e.g., `"identity_handler": { "service_worker": "/sw.js" }`).

#### Cons
- `config.json` is fetched with RP context, so the IDP could encode RP-specific values into the URL (`/sw-for-rp-A.js` vs. `/sw-for-rp-B.js`); the SW (and the server serving it) would learn **which RP** the user is interacting with.

#### Reason for rejection
Fails privacy. Registration is therefore declared in `/.well-known/web-identity` (fetched without RP context) via the `identity-handler` member and managed by the UA.

## Accessibility, Internationalization, and Privacy & Security Considerations

### Accessibility & Internationalization

This is a network-layer feature with **no UI surface of its own** and no user-visible strings; it does not change the FedCM account chooser or consent UI. There are therefore no direct accessibility or internationalization implications. By improving sign-in reliability during IDP degradation, it indirectly benefits all users, including those relying on assistive technology.

### Privacy

- **Configuration endpoints stay protected.** `/.well-known`, `config.json`, and `client_metadata` are **not** dispatched to the handler. They are fetched with privacy-preserving properties (`credentials: "omit"`, opaque origin, no referrer). If the handler could intercept them, it could correlate user identity (via its own-origin cookies) with RP identity (from `client_metadata` parameters).
- **No new information to the IDP.** The handler runs at the IDP origin and sees only what the IDP server already sees: `/accounts` (GET) reveals no RP identity (opaque origin); `/token` and `/disconnect` (POST) carry the RP `client_id` in the body, which the IDP server already receives.
- **Consent is preserved.** The handler cannot bypass the FedCM consent UI; it only augments the network layer, and consent is required before any token is issued.
- **Registration cannot leak the RP.** Declaring the handler in the RP-context-free well-known (not `config.json`) prevents encoding RP identity into the SW URL.

### Security

- **Origin isolation.** The handler is registered at the IDP origin and can only intercept requests **to that origin**. Cross-origin interception is architecturally impossible.
- **CSRF marker.** The handler-initiated `fetch` does **not** propagate `Sec-Fetch-Dest: webidentity` (which would require an invasive Fetch-spec exception). IDP servers instead verify the source via `Sec-Fetch-Site: same-origin`, which the same-origin handler `fetch` carries reliably.
- **Redirect prevention.** FedCM requests use `redirect mode: "error"`. Requiring the handler to issue **same-origin** requests gives the same guarantee at the server layer, so a separate UA-side response-type filter is not required.
- **Cookie scope.** Reusing standard SW `fetch` semantics means the same-origin call carries the full IDP cookie jar (including `SameSite=Lax`/`Strict`), not only `SameSite=None`. This is acceptable because the cookies are already scoped to the IDP origin; FedCM-specific cookie exceptions were rejected as too invasive.
- **Augmenting headers.** The handler may add headers (e.g., `DPoP`) but must **not** override UA-set headers (`Accept`, `Sec-Fetch-*`, `Cookie`, `Origin`, `Referer`). The exact allowlist and the process for growing it are still being finalized.
- **Robust fallback.** The skip-flag pattern guarantees a handler failure degrades to the direct-network path rather than breaking sign-in.

## Stakeholder Feedback / Opposition

- **FedCM editors / WG** : Engaged. Several design questions have been **resolved** (dedicated event, relaxed response-type/cookie-scope invariants, `Sec-Fetch-Site` CSRF marker, `client_metadata` exclusion). The SW registration model remains under active discussion.
- **Service Worker editors** : Cautious. Rejected reusing a client-less `FetchEvent` (foreign-fetch concerns) and asked for formal spec/security/privacy/TAG review — shaping the dedicated-event direction.
- **Other browser engines** : No public signals yet.

## References & acknowledgements

Many thanks for valuable feedback and advice from the FedCM editors, the Service Worker editors, and the WG privacy reviewer.

Thanks to the following prior work that influenced this proposal:

- [DPoP (RFC 9449)](https://www.rfc-editor.org/rfc/rfc9449)
- [Platform Authentication explainer (Microsoft Edge)](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/PlatformAuthentication/explainer.md)
- [aaronpk's OAuth profile for FedCM](https://github.com/aaronpk/oauth-fedcm-profile), which framed the scope of the IDP-cookie bearer problem this proposal targets

Specifications:

- [FedCM Specification](https://fedidcg.github.io/FedCM/)
- [Service Worker Specification](https://w3c.github.io/ServiceWorker/)
- [Fetch Specification](https://fetch.spec.whatwg.org/)

## Appendix A: WebIDL (Chromium / Blink form)

The dedicated event is defined below in the Blink `.idl` form used in the Chromium tree — one definition per file, with the standard license header, a spec-link comment, an extended-attribute block (`RuntimeEnabled`, `Exposed`), `[CallWith=ScriptState]` on the constructor, `[SameObject]` on the `Request` attribute, and `[CallWith=ScriptState, RaisesException] void respondWith(...)` — modeled directly on [`fetch_event.idl`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/modules/service_worker/fetch_event.idl).

`third_party/blink/renderer/modules/credentialmanagement/identity_request_event.idl`

```webidl
// Copyright 2026 The Chromium Authors
// Use of this source code is governed by a BSD-style license that can be
// found in the LICENSE file.

// https://fedidcg.github.io/FedCM/
[
    RuntimeEnabled=FedCmIdentityHandler,
    Exposed=ServiceWorker
] interface IdentityRequestEvent : ExtendableEvent {
    [CallWith=ScriptState] constructor(DOMString type, IdentityRequestEventInit eventInitDict);
    readonly attribute IdentityRequestEndpoint endpoint;
    [SameObject] readonly attribute Request request;

    [CallWith=ScriptState, RaisesException] void respondWith(Promise<Response> r);
};
```

`third_party/blink/renderer/modules/credentialmanagement/identity_request_event_init.idl`

```webidl
// Copyright 2026 The Chromium Authors
// Use of this source code is governed by a BSD-style license that can be
// found in the LICENSE file.

// https://fedidcg.github.io/FedCM/
dictionary IdentityRequestEventInit : ExtendableEventInit {
    required IdentityRequestEndpoint endpoint;
    required Request request;
};
```

`third_party/blink/renderer/modules/credentialmanagement/identity_request_endpoint.idl`

```webidl
// Copyright 2026 The Chromium Authors
// Use of this source code is governed by a BSD-style license that can be
// found in the LICENSE file.

// https://fedidcg.github.io/FedCM/
enum IdentityRequestEndpoint {
    "accounts",
    "id_assertion",
    "disconnect"
};
```

> Notes
> - `respondWith()` mirrors `FetchEvent`'s signature, but its `Promise<Response>` is processed in a **FedCM-specific** way (e.g., the accounts response is parsed as JSON and passed to FedCM internals).
> - `[CallWith=ScriptState]` and `[RaisesException]` are **Blink IDL extended attributes** required by the Chromium implementation; they are not part of standard [Web IDL](https://webidl.spec.whatwg.org/) and are therefore dropped from the spec-facing IDL. Blink still uses `void` rather than the Web IDL `undefined` return type, which is reflected above.
> - `RuntimeEnabled=FedCmIdentityHandler` gates the feature behind a flag declared in `runtime_enabled_features.json5`.
