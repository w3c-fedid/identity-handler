# Identity Handler for FedCM

## Authors:

- [@sureshpotti](https://github.com/sureshpotti)
- [@Brandr0id](https://github.com/Brandr0id) (Microsoft)

## Participate
- [Issue tracker — FedCM Issue #80](https://github.com/w3c-fedid/FedCM/issues/80)
- [Identity Handler repository](https://github.com/w3c-fedid/identity-handler)
- Discussion: [W3C Federated Identity Community/Working Group](https://github.com/w3c-fedid/FedCM/discussions)

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->

- [Introduction](#introduction)
- [User-Facing Problem](#user-facing-problem)
  - [Goals](#goals)
  - [Non-goals](#non-goals)
- [User research](#user-research)
- [Proposed Approach](#proposed-approach)
  - [Service Worker registration (UA / FedCM-managed)](#service-worker-registration-ua--fedcm-managed)
  - [Dispatch model](#dispatch-model)
  - [Flow (on each FedCM invocation)](#flow-on-each-fedcm-invocation)
  - [Selective endpoint policy](#selective-endpoint-policy)
  - [Fallback vs. failure](#fallback-vs-failure)
  - [Dependencies on non-stable features](#dependencies-on-non-stable-features)
  - [Solving token-binding with this approach](#solving-token-binding-with-this-approach)
  - [Solving outage resiliency with this approach](#solving-outage-resiliency-with-this-approach)
  - [Solving sovereignty-aware routing with this approach](#solving-sovereignty-aware-routing-with-this-approach)
  - [Solving non-FedCM backend bridging with this approach](#solving-non-fedcm-backend-bridging-with-this-approach)
- [Alternatives considered](#alternatives-considered)
- [Accessibility, Internationalization, Privacy, and Security Considerations](#accessibility-internationalization-privacy-and-security-considerations)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

[FedCM](https://fedidcg.github.io/FedCM/) lets a user sign in to a website (the Relying Party, or RP) with a federated identity provider (IDP) without third-party cookies. To do this, the browser makes a set of credentialed network requests to the IDP — `accounts`, `id-assertion`, and `disconnect` — on the user's behalf. Today these requests are made with `service-workers mode: "none"`, so they go straight to the network and the IDP has no programmable seam between FedCM and its own network stack.

This proposal adds an **opt-in Identity Handler Service Worker**, hosted at the IDP origin, that the browser dispatches FedCM's credentialed calls to before they reach the network. The handler can attach a fresh proof-of-possession header, serve cached account data during an outage, route a request to the correct region, or translate between FedCM and a non-FedCM identity backend. The feature is purely additive: RPs need no changes, configuration/discovery endpoints are never exposed to the handler, and when no handler is registered or it declines to handle a request, FedCM transparently uses the existing direct-network path.

## User-Facing Problem

**An end user wants to sign in to a website with their identity provider and have it _just work_ — securely, and even when their provider is partly degraded or operates under regional data-residency rules.** Today, four gaps in FedCM make that harder than it should be.

The connection to the end user is partly indirect (these are security and reliability properties of the sign-in network path), so each problem below states the user-visible consequence.

**1. Bearer-token theft on the IDP cookie.** The cookies FedCM sends on `accounts` and `id-assertion` are **bearer credentials** — anyone who copies the bytes can replay them, often for the lifetime of the session (months). Modern enterprise IDPs increasingly require the cookie to be accompanied by a fresh, device-bound proof-of-possession assertion. Microsoft Entra, for instance, only honors its `ESTSAUTH*` cookies when an `x-ms-RefreshTokenCredential` JWT — signed by a TPM-backed device key — is attached. FedCM as specified gives no place to attach that proof, so today this only works via a browser extension (Chrome) or a non-standard API (Edge). _User impact: a stolen session cookie can impersonate the user._

**2. Hard failures during IDP outages.** When the IDP backend is degraded — a regional incident, a partial accounts-service outage, a maintenance window — the FedCM call fails outright. The IDP has no chance to apply the resiliency patterns (regional failover, last-known-good cached account data) it routinely uses on its own surface. _User impact: sign-in fails even though the user is already authenticated._

**3. Geographic / sovereignty-aware routing.** Some IDPs must route a user's authentication traffic to a specific regional service for data-residency reasons. `config.json` declares a single set of endpoint URLs per origin and is fetched without user context, so there is no point at which the IDP can pick a region for the actual signed-in user before the `accounts` call goes out. _User impact: sign-in can violate the user's data-residency expectations, or simply fail._

**4. Bridging non-FedCM identity backends.** An IDP whose existing backend wire format does not match FedCM's `accounts` / `id-assertion` shape must deploy a server-side translation layer in front of every endpoint. This is operationally heavy and slows adoption. _User impact: fewer providers offer the privacy-preserving FedCM sign-in experience._

### Goals

- Let an IDP attach a fresh, device-bound **proof-of-possession header** (e.g., DPoP) to credentialed FedCM requests, so a stolen cookie alone is not enough to impersonate the user.
- Let an IDP apply **outage resiliency** (failover, last-known-good cache) to FedCM's credentialed calls.
- Let an IDP perform **user-context-aware regional routing** before the credentialed call goes to the network.
- Let an IDP **bridge a non-FedCM backend** into FedCM's request/response shape without a separate server-side translation tier.
- Require **zero changes from RPs** and preserve FedCM's existing privacy boundary.
- **Fail safe**: sign-in never breaks because a handler is absent or declines a request; those cases fall back to today's direct-network FedCM flow.

### Non-goals

- Exposing FedCM's **configuration / discovery** endpoints (`/.well-known/web-identity`, `config.json`, `client_metadata`) to the handler. These stay UA-direct to preserve the privacy boundary.
- Allowing the handler to **bypass or alter the FedCM consent UI**. The handler operates only on the network layer; user consent is still required before any token is issued.
- Cross-origin interception. A handler can only ever see requests destined for **its own IDP origin**.

## User research

No formal end-user research was conducted: this is a network-layer capability with no UI surface of its own, so its user-facing effect (sign-in reliability and session-theft resistance) is indirect. The problem framing is instead grounded in identity-provider and enterprise requirements.

## Proposed Approach

We introduce an opt-in **Identity Handler** Service Worker, registered at the IDP origin, that the user agent invokes for FedCM's credentialed endpoints before going to the network. The handler is a normal Service Worker that listens for a single new event, `identityrequest`, and responds with a `Response` the way a `fetch` handler does.

### Service Worker registration (UA / FedCM-managed)

Registration is **UA-managed (FedCM-managed)**: the user agent — not a script on an IDP page — registers and maintains the handler. This matters because a signed-in IDP session can stay valid for weeks or months (Entra allows up to ~90 days), during which FedCM keeps making credentialed calls without the user ever revisiting an IDP page. A page-script registration would not reliably exist when needed.

The IDP declares the handler in **`/.well-known/web-identity`** using an `identity_handler` member with a `service_worker` member:

```json
{
  "provider_urls": [
    "https://idp.example/fedcm/config.json"
  ],
  "identity_handler": {
    "service_worker": "/fedcm/sw.js"
  }
}
```

- The presence of `identity_handler` is the IDP's **opt-in** signal.
- `service_worker` is the script URL the UA registers; it MUST be same-origin with the well-known file, so that an attacker who controls a different subdomain of the IDP cannot register a handler for this origin. The UA derives the registration scope from it (the script URL with its last path segment removed).
- There is no separate `enabled` switch: an IDP disables interception by **removing the `identity_handler` member**, which causes the UA to unregister the handler and use the direct-network path.

UA/FedCM-managed registration via the well-known `identity_handler` member is the current direction.

### Dispatch model

When the browser needs to fetch a credentialed IDP endpoint (`accounts`, `id-assertion`, `disconnect`), it dispatches a purpose-built `identityrequest` event to the registered handler using the Service Worker spec's [_Fire Functional Event … on registration_](https://www.w3.org/TR/service-workers/#fire-functional-event) primitive (a UA-initiated request, a destination-origin SW, and **no controlling client**). Because the request is sent over this trusted, FedCM-only path — not with the RP's [request client](https://fetch.spec.whatwg.org/#concept-request-client) — it does not reintroduce the generic cross-origin SW invocation that "foreign fetch" was removed to prevent.

The handler reads `event.endpoint` (one of `"accounts"`, `"id-assertion"`, or `"disconnect"`) to learn which credentialed call this is, inspects `event.request`, and calls `event.respondWith()` with a `Promise<Response>`. FedCM processes that response in a FedCM-specific way (e.g., parsing the accounts JSON) and feeds it into the existing FedCM internals.

Dispatch uses an **explicit, UA-only association** between the IDP and the registration the UA created for it (keyed by the isolated FedCM `StorageKey`), rather than the implicit _Match Service Worker Registration_ path used for client-controlled fetches. Because a credentialed FedCM request has no controlling client, this ensures a functional event reaches only a Service Worker the UA registered for that specific IDP.

### Flow (on each FedCM invocation)

On each FedCM invocation the UA:

1. Fetches the well-known and reads the `identity_handler` declaration. If the member is absent, the UA unregisters any existing handler and uses the direct-network path.
2. Looks up the registration under a **FedCM-managed, isolated `StorageKey`** that is derived deterministically from the IDP origin, so the FedCM registration can never collide with a page-initiated `navigator.serviceWorker.register()` for the same path, and a later flow can recompute the key to look up or unregister the handler without persisted state.
3. If a matching registration exists for the declared origin and `service_worker` URL, it dispatches the `identityrequest` event to it and schedules a [Soft Update](https://www.w3.org/TR/service-workers/#soft-update) **in parallel** (governed by HTTP cache freshness, so the IDP keeps its normal staged-rollout/rollback strategy).
4. Otherwise it registers and activates the declared handler, then dispatches.

Because the isolated `StorageKey` is keyed to the IDP origin, clearing the IDP's site data (e.g., via `Clear-Site-Data`) is expected to also clear the isolated FedCM partition, though the spec text should mandate this explicitly since the partition is otherwise UA-only. No FedCM-specific unregistration mechanism is required.

To keep handler add/remove latency low, IDPs SHOULD serve `/.well-known/web-identity` with a short `Cache-Control: max-age` (e.g., no more than 5 minutes). Because Step 1 re-reads the well-known on each FedCM invocation subject to normal HTTP caching, this bounds how quickly adding or removing the `identity_handler` member takes effect. It complements the SW Soft Update flow (which recovers a broken script) by covering the case of pulling the handler entirely.

### Selective endpoint policy

Only **credentialed** endpoints are dispatched to the handler. Configuration / discovery endpoints stay UA-direct to preserve the existing privacy boundary.

| Endpoint | Dispatched to handler | Reason |
|----------|:--:|--------|
| `/accounts` (GET) | ✅ | User account data — benefits from caching and augmentation |
| `/token` (id-assertion, POST) | ✅ | Token generation — can attach DPoP, custom logic |
| `/disconnect` (POST) | ✅ | Account management — graceful error handling, backend bridging |
| `/.well-known/web-identity` | ❌ | IDP discovery — privacy protection |
| `/config.json` | ❌ | Configuration — privacy protection |
| `/client_metadata` | ❌ | RP metadata — privacy protection |

This credentialed-endpoints-only scoping is the result of explicit attack analysis on [Issue #80](https://github.com/w3c-fedid/FedCM/issues/80) and cleared WG privacy review.

### Fallback vs. failure

The UA distinguishes two outcomes:

- **Not handled → transparent fallback.** If no handler is registered, the worker cannot be started, or the handler returns without calling `respondWith()`, the browser falls back to a normal network fetch using a "skip flag" pattern: emit a console warning, set a `skip_identity_handler` flag, re-issue the same request UA-direct, then clear the flag. This is the expected path for endpoints a handler chooses not to intercept.
- **Handled but failed → hard failure (no fallback).** If the handler calls `respondWith()` but the promise rejects, resolves with an invalid or non-OK response, exceeds the response-body cap, or does not settle within an implementation-defined timeout (the prototype uses 10 seconds), the FedCM request **fails without falling back**. This mirrors `FetchEvent.respondWith()` in the Service Worker spec and ensures a handler that has claimed a request cannot be silently bypassed; the timeout is required (its duration is implementation-defined) so a handler that never settles its promise cannot hang the FedCM flow indefinitely.

### Dependencies on non-stable features

- The dispatch primitive ([_Fire Functional Event … on registration_](https://www.w3.org/TR/service-workers/#fire-functional-event)) is a stable Service Worker concept.
- The new `identityrequest` event and the UA-managed registration model are **new and not yet shipped**.

### Solving token-binding with this approach

A stolen IDP cookie is no longer enough on its own: the handler attaches a fresh, device-bound proof to the `id-assertion` call.

```js
// /fedcm/sw.js — IDP's Identity Handler Service Worker
self.addEventListener('identityrequest', (event) => {
  if (event.endpoint === 'id-assertion') {
    event.respondWith((async () => {
      const proof = await generateDPoPProof(event.request.url, 'POST');
      const headers = new Headers(event.request.headers);
      headers.set('DPoP', proof);
      const augmented = new Request(event.request, { headers });
      return fetch(augmented); // same-origin → carries Sec-Fetch-Site: same-origin
    })());
  }
});
```

Here `generateDPoPProof` stands in for however the IDP mints its proof; for hardware-bound credentials that value can be a [DBSC(E)](https://github.com/w3c/webappsec-dbsc/tree/main/DBSCE) proof (a separate, enterprise-gated track).

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

### Foreign Fetch (removed from the platform)

[Foreign fetch](https://github.com/whatwg/fetch/issues/506) was a Service Worker mechanism that let a SW intercept cross-origin requests initiated by *other* origins (i.e., dispatch to a SW with no initiating client of its own).

#### Pros
- Superficially matches the shape we need: a UA-initiated, client-less request reaching an origin's SW.

#### Cons
- It was **removed from the web platform** precisely because a client-less, cross-origin SW-invocation surface proved hard to reason about and secure (Ben Kelly, on the [Chromium service-worker-discuss thread](https://groups.google.com/a/chromium.org/g/service-worker-discuss/c/t9d33x6l718)).

#### Reason for rejection
Reviving a client-less *cross-origin* invocation path is a non-starter. Identity Handler deliberately avoids it: the intercepted request is **same-origin** with the IDP (never cross-origin), and the trigger for dispatch is a UA-initiated FedCM call for that specific IDP, not an arbitrary cross-origin fetch that any origin can issue. Dispatch is a UA-only, FedCM-specific path keyed to an isolated registration, not a general-purpose surface. This distinction is exactly why a dedicated `identityrequest` event, rather than a reused `FetchEvent`, is the chosen design.

### A dedicated worklet or one-off worker

Run the augmentation logic in a worklet (à la Audio/Paint worklets, or a hypothetical "auth worklet") or a freshly spawned dedicated worker, instead of a Service Worker.

#### Pros
- Smaller, more constrained execution surface than a full Service Worker; no persistent registration to manage.

#### Cons
- Worklets and dedicated workers have **no registration, lifecycle, or request-interception model**, and do not exist unless a page spins them up. FedCM's defining requirement is augmenting credentialed calls **weeks or months after the IDP page was last open**, when there is no page to create the worklet/worker.
- Worklets further lack `fetch()`, the Cache API, and IndexedDB, which the handler needs for the outage-resiliency (Cache API) and non-FedCM backend-bridging (`fetch()` to internal endpoints) use cases described above.
- No standard, isolated place to persist the IDP's augmentation code across sessions.

#### Reason for rejection
Only a Service Worker provides UA-managed, persistent registration plus a dispatchable, request-scoped event that works with **no page open**. Worklets and one-off workers cannot satisfy the long-lived-session requirement.

### Registration shape — SW URL in `config.json` (backed out)

Declaring the handler script inside `config.json` (e.g., `"identity_handler": { "service_worker": "/sw.js" }`).

#### Cons
- `config.json` is fetched with RP context, so the IDP could encode RP-specific values into the URL (`/sw-for-rp-A.js` vs. `/sw-for-rp-B.js`); the SW (and the server serving it) would learn **which RP** the user is interacting with.

#### Reason for rejection
Fails privacy. Registration is therefore declared in `/.well-known/web-identity` (fetched without RP context) via the `identity_handler` member and managed by the UA.

### Registration isolation: page-side opt-in via `acceptsFedCM` or `registration.identity` (not adopted)

Two page-side opt-in shapes were considered for keeping FedCM's Service Worker isolated from the IDP's ordinary Service Workers: a boolean `acceptsFedCM` flag on `navigator.serviceWorker.register()`, and a dedicated `registration.identity` manager (`registration.identity.register(configURL)`) modeled on the Periodic Background Sync and Notification APIs.

#### Pros
- Explicit, page-controlled opt-in with a clear `register()` / `unregister()` lifecycle.
- `registration.identity` gives a clean one-to-one `configURL`→registration mapping and avoids dispatching through the implicit _Match Service Worker Registration_ path for a client-less request.

#### Cons
- Both require a **page** to call `register()`. FedCM's core requirement is that credentialed calls keep working for weeks or months **without the IDP page being revisited**; a page-side registration would not reliably exist when needed.
- More API surface for IDPs to adopt, and it re-introduces the "page must be visited" fragility the UA-managed model exists to remove.

#### Reason for rejection
The long-lived-session requirement is decisive: registration must be UA-managed. The isolation these proposals sought is instead achieved by storing the FedCM registration under a **UA-only, deterministically-derived, isolated `StorageKey`** that page script cannot target, and by framing dispatch as an explicit UA-only IDP→registration association (which addresses the client-less _Match_ concern). The clean lifecycle is preserved: the UA unregisters when the `identity_handler` member is removed, and `Clear-Site-Data` clears the partition.

## Accessibility, Internationalization, Privacy, and Security Considerations

### Accessibility & Internationalization

This is a network-layer feature with **no UI surface of its own** and no user-visible strings; it does not change the FedCM account chooser or consent UI. There are therefore no direct accessibility or internationalization implications. By improving sign-in reliability during IDP degradation, it indirectly benefits all users, including those relying on assistive technology.

### Privacy

- **Configuration endpoints stay protected.** `/.well-known`, `config.json`, and `client_metadata` are **not** dispatched to the handler. They are fetched with privacy-preserving properties (`credentials: "omit"`, opaque origin, no referrer). If the handler could intercept them, it could correlate user identity (via its own-origin cookies) with RP identity (from `client_metadata` parameters).
- **No new information to the IDP.** The handler runs at the IDP origin and sees only what the IDP server already sees: `/accounts` (GET) reveals no RP identity (opaque origin); `/token` and `/disconnect` (POST) carry the RP `client_id` in the body, which the IDP server already receives.
- **Consent is preserved.** The handler cannot bypass the FedCM consent UI; it only augments the network layer, and consent is required before any token is issued.
- **Registration cannot leak the RP.** Declaring the handler in the RP-context-free well-known (not `config.json`) prevents encoding RP identity into the SW URL.

### Security

- **Origin isolation.** The handler is registered at the IDP origin and can only intercept requests **to that origin**. Cross-origin interception is architecturally impossible.
- **Registration isolation.** The handler lives under a UA-only, isolated `StorageKey`, so page script cannot register, enumerate, hijack, or unregister it, and the IDP's ordinary first-party Service Workers are never enrolled as identity handlers.
- **Cookie scope.** Reusing standard SW `fetch` semantics means the same-origin call carries the full IDP cookie jar (including `SameSite=Lax`/`Strict`), not only `SameSite=None`. This is acceptable because the cookies are already scoped to the IDP origin; FedCM-specific cookie exceptions were rejected as too invasive.
- **Response-size cap.** The response a handler provides is read with the same cap as the direct FedCM network path, enforced while reading, so a buggy or hostile handler cannot force an unbounded allocation.
- **No forged or deferred responses.** `respondWith()` throws `InvalidStateError` on an untrusted (script-constructed) event, after dispatch has completed, or if called twice, matching `FetchEvent.respondWith()`.
- **Fail-closed on claimed requests.** A handler that calls `respondWith()` but fails (rejection, invalid/non-OK response, oversize body, or timeout) fails the request rather than silently falling back, so a broken or hostile handler cannot mask a security-relevant failure. Not-registered / not-handled cases fall back (see [Fallback vs. failure](#fallback-vs-failure)).

## Stakeholder Feedback / Opposition

- **FedCM editors / WG**: Engaged. Resolved: dedicated event, relaxed response-type/cookie-scope invariants, `Sec-Fetch-Site` CSRF marker, `client_metadata` exclusion, and agreement that FedCM handler registrations live in a **UA-managed, isolated partition** (not colliding with, nor reachable by, the IDP's other Service Workers). Open: formally closing the registration-model alternatives (`acceptsFedCM` / `registration.identity`) in favor of the UA-only isolated key (see [Alternatives considered](#alternatives-considered)).
- **Service Worker editors**: Cautious. Rejected reusing a client-less `FetchEvent` (foreign-fetch concerns) and asked for formal spec/security/privacy/TAG review, which shaped the dedicated-event direction. The "no controlling client" concern is addressed by framing dispatch as an explicit UA-only IDP→registration association rather than an implicit _Match Service Worker Registration_.
- **Other browser engines**: No public signals yet.

## References & acknowledgements

Many thanks for valuable feedback and advice from the FedCM editors, the Service Worker editors, and the WG privacy reviewer.

Thanks to the following prior work that influenced this proposal (in alphabetical order):

- [aaronpk's OAuth profile for FedCM](https://github.com/aaronpk/oauth-fedcm-profile), which framed the scope of the IDP-cookie bearer problem this proposal targets.
- [Device Bound Session Credentials (DBSC)](https://github.com/w3c/webappsec-dbsc) and its enterprise variant [DBSC(E)](https://github.com/w3c/webappsec-dbsc/tree/main/DBSCE): a complementary standard for binding sessions/cookies to a device-held key; a natural pairing for the bearer-token-theft problem this proposal targets (a handler could attach DBSC(E) proofs, or DBSC(E) could bind the cookie the handler refreshes).
- [DPoP (RFC 9449)](https://www.rfc-editor.org/rfc/rfc9449).
- [Platform Authentication explainer (Microsoft Edge)](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/PlatformAuthentication/explainer.md).

Specifications:

- [Fetch Specification](https://fetch.spec.whatwg.org/)
- [FedCM Specification](https://fedidcg.github.io/FedCM/)
- [Service Worker Specification](https://w3c.github.io/ServiceWorker/)
