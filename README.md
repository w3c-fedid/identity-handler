# Identity Handler

A proposal to extend FedCM to allow an IDP-controlled Service Worker to intercept FedCM's credentialed requests (accounts, id_assertion, disconnect), so the IDP can attach proof-of-possession headers, apply outage resiliency, perform sovereignty-aware routing, and bridge non-FedCM identity backends.

## Stage

This is a [Stage 1](https://github.com/w3c-fedid/Administration/blob/main/proposals-CG-WG.md) proposal.

## Champions

- @sureshpotti

## Participate

- https://github.com/w3c-fedid/FedCM/issues/80
- https://github.com/w3c-fedid/FedCM/pull/815 (spec PR)

# The Problem

The credentialed FedCM endpoints (accounts, id_assertion, disconnect) authenticate the user to the IDP using cookies. Today these requests have `service-workers mode: "none"`, so the IDP has no programmable seam between FedCM and its own network stack. Four concrete problems result.

## 1. Bearer-token theft on the IDP cookie

The cookies sent on the accounts and id_assertion calls are **bearer credentials** — anyone in possession of the bytes can replay them, often for the lifetime of the session (months). Modern enterprise IDPs increasingly require the cookie to be accompanied by a fresh, device-bound, proof-of-possession assertion before they will honor it. Microsoft Entra, for example, only accepts its `ESTSAUTH*` cookies if an `x-ms-RefreshTokenCredential` JWT — containing the device-wide SSO token and a server-issued nonce, signed by a TPM-backed device key — is attached to the request. Today the only way to get that header onto a FedCM-style call is via a browser extension (Chrome) or a non-standard browser API (Edge); see the [Platform Authentication explainer](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/PlatformAuthentication/explainer.md). FedCM as specified gives no place for that proof to be attached.

> **Why not just "OAuth profile + DPoP at code redemption"?** [aaronpk's OAuth profile for FedCM](https://github.com/aaronpk/oauth-fedcm-profile) shows that DPoP-bound *RP* tokens can already be obtained without browser changes by treating the FedCM response as an authorization code and DPoP-binding the subsequent code→token exchange. That solves RP-side token binding but does **not** address the IDP-cookie bearer problem (the cookie sent on the accounts call is still a plain bearer), and the extra round trip is costly for bundle apps that talk to many IDP-issued resources (e.g., Teams).

## 2. Hard failures during IDP outages

When the IDP backend is degraded — a regional incident, a partial accounts-service outage, a brief maintenance window — the FedCM call fails outright and the user sees a sign-in failure. The IDP has no opportunity to apply the resiliency patterns (regional failover, last-known-good cached account data) it routinely uses on its own surface.

## 3. Geographic / sovereignty-aware routing

Some IDPs are required (by data-residency regulations or by tenant configuration) to route a given user's authentication traffic to a specific regional service. The FedCM `config.json` declares a single set of endpoint URLs per IDP origin and is fetched without user context, so there is no point at which the IDP can pick a region based on the actual signed-in user before the accounts call goes out.

## 4. Bridging non-FedCM identity backends

An IDP with an existing identity backend whose wire format does not exactly match FedCM's accounts / id_assertion shape would otherwise have to deploy a server-side translation layer in front of every FedCM endpoint. This is operationally heavy and slows adoption — especially for IDPs experimenting with FedCM alongside an existing OIDC / SAML stack.

# The Proposal

Introduce an opt-in **Identity Handler** Service Worker, hosted at the IDP origin, that the user agent dispatches FedCM's credentialed calls to before they go to the network. The SW can attach proof-of-possession headers, serve cached responses during outages, rewrite outbound URLs for regional routing, or synthesize a FedCM-shaped response from a non-FedCM backend.

## Dispatch model

When the browser needs to fetch a credentialed IDP endpoint (accounts, id_assertion, disconnect), it dispatches a purpose-built `IdentityRequestEvent` to the registered SW via the Service Worker spec's [*Fire Functional Event ... on registration*](https://www.w3.org/TR/service-workers/#fire-functional-event) primitive — the same primitive [Payment Handler](https://w3c.github.io/web-based-payment-handler/) uses for the same shape of problem (UA-initiated request, destination-origin SW, no controlling client).

```webidl
enum IdentityRequestEndpoint {
    "accounts",
    "id_assertion",
    "disconnect"
};

[Exposed=ServiceWorker]
interface IdentityRequestEvent : ExtendableEvent {
    constructor(DOMString type, IdentityRequestEventInit eventInitDict);

    readonly attribute IdentityRequestEndpoint endpoint;
    readonly attribute Request request;

    [RaisesException] undefined respondWith(Promise<Response> r);
};

dictionary IdentityRequestEventInit : ExtendableEventInit {
    required IdentityRequestEndpoint endpoint;
    required Request request;
};
```

## SW handles the event

```javascript
// /idp-sw.js — IDP's Identity Handler Service Worker

self.addEventListener('identityrequest', (event) => {
  switch (event.endpoint) {
    case 'accounts':
      // Outage resiliency: try the network, fall back to a recent cache.
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
      break;

    case 'id_assertion':
      // Token binding: attach a fresh DPoP proof.
      event.respondWith((async () => {
        const proof = await generateDPoPProof(event.request.url, 'POST');
        const augmented = new Request(event.request, {
          headers: { ...event.request.headers, 'DPoP': proof }
        });
        return fetch(augmented);
      })());
      break;

    case 'disconnect':
      // Bridge a non-FedCM revocation backend.
      event.respondWith((async () => {
        const body = await event.request.clone().text();
        const accountId = new URLSearchParams(body).get('account_hint');
        const upstream = await fetch('/internal/revoke', { method: 'POST', body });
        if (!upstream.ok) return upstream;
        return Response.json({ account_id: accountId });
      })());
      break;
  }
});
```

If the SW does not call `respondWith()`, rejects the promise, returns a non-OK response, returns a response of a disallowed type, or times out (default 10 seconds), the browser **transparently falls back** to a normal network fetch. The feature is purely additive — failures never break the FedCM flow.

## Selective endpoint policy

Only credentialed endpoints are dispatched to the SW. Configuration endpoints stay on the UA-direct path to preserve the existing privacy boundary.

| Endpoint | SW Dispatch | Reason |
|----------|-------------|--------|
| `/accounts` | ✅ Yes | User account data — benefits from caching and augmentation |
| `/token` (id_assertion) | ✅ Yes | Token generation — can use DPoP, custom logic |
| `/disconnect` | ✅ Yes | Account management — graceful error handling |
| `/.well-known/web-identity` | ❌ No | IDP discovery — privacy protection |
| `/config.json` | ❌ No | Configuration — privacy protection |
| `/client_metadata` | ❌ No | RP metadata — privacy protection |

The credentialed-endpoints-only scoping is the result of explicit attack analysis on Issue #80 and cleared privacy review in the WG.

## RP transparency

RPs require **zero changes**. The FedCM API contract is unchanged:

```javascript
const credential = await navigator.credentials.get({
  identity: {
    providers: [{
      configURL: "https://idp.example/config.json",
      clientId: "rp_client_123",
      nonce: "random_nonce_value"
    }]
  }
});
```

# Alternatives Considered

## Approach A — Flip `service-workers mode` to `"all"`

The most conservative spec change: replace `service-workers mode: "none"` with `"all"` on FedCM's credentialed requests and let standard SW dispatch handle it. This **cannot work as a spec-only change** because the Service Worker spec's [*Handle Fetch*](https://www.w3.org/TR/service-workers/#handle-fetch) dispatches via the request's **client** — and per the [FedCM spec](https://fedidcg.github.io/FedCM/), every credentialed endpoint sets `client = null`. *Handle Fetch* has nothing to dispatch through, and the request goes straight to the network.

## Approach B — Reuse `FetchEvent` via a browser-side dispatch layer

An alternative would be for the browser to ship a FedCM-specific dispatch layer that bypasses standard *Handle Fetch* entirely — look up the IDP's SW registration by URL scope, construct an internal request, invoke the SW's `fetch` event directly. This was prototyped and is implementable today, but the corresponding **spec direction** — extending Fetch / Service Worker so the *spec* can dispatch a `FetchEvent` to a SW with no controlling client — was not endorsed on the [Chromium service-worker-discuss thread](https://groups.google.com/a/chromium.org/g/service-worker-discuss/c/t9d33x6l718):

- **No initiating client** (Ben Kelly). The dispatch target is the *destination* origin's SW — the same dispatch shape used by the deprecated [foreign fetch](https://github.com/whatwg/fetch/issues/506), which the SW WG removed.
- **Risk to deployed SWs.** Any IDP SW with a generic `fetch` handler would suddenly receive FedCM requests it was not written to handle.
- **Procedural asks** (Dominic Farolino). A formal spec design section, security / privacy review, cross-vendor review, and TAG review were called out as prerequisites.

## Approach C — Dedicated event (current direction)

Introduce a purpose-built `IdentityRequestEvent` that the SW must explicitly listen for, dispatched via *Fire Functional Event ... on registration* — the same primitive used by Payment Handler for the same shape of problem. This makes the unusual dispatch model **explicit at the API surface** rather than overloading `FetchEvent`, leaves pre-existing IDP SWs unaffected, and sidesteps the "foreign fetch in disguise" critique.

## Backed-out registration shape

**SW URL declared in the IDP `config.json`** (e.g., `"identity_handler": { "service_worker": "/sw.js" }` inside `config.json`) **fails privacy** — the IDP could encode RP-specific values into the URL (`/sw-for-rp-A.js` vs. `/sw-for-rp-B.js`), so the SW (and the server serving the SW script) would learn which RP the user is interacting with. The registration surface is therefore being designed outside `config.json`; see Open Questions.

# Open Questions

A full explainer with the design discussion is at [identity_handler.md](https://github.com/MicrosoftEdge/MSEdgeExplainers/) (source). Three areas are still being worked out:

- **a) Dispatch shape — `FetchEvent` vs. a dedicated event.** Reuse `FetchEvent` (smallest spec surface, but risks silently invoking deployed IDP SWs) vs. dedicated `IdentityRequestEvent` (larger IDL surface, but invisible to existing handlers).
- **b) Service Worker registration.** How does an IDP register a SW for FedCM, and how does the UA know which registration to dispatch to? Options under consideration: SW path/scope declared in `/.well-known/web-identity` plus standard `navigator.serviceWorker.register()`, or a dedicated `registration.identity` manager extending `ServiceWorkerRegistration` (modeled on Payment Handler's `registration.paymentManager`).
- **c) Lock-down enforcement when `fetch` is called from the SW.** Whether the spec should mechanically enforce that the UA-stamped request properties (`Sec-Fetch-Dest: webidentity`, opaque/RP `Origin`, `redirect mode: "error"`, etc.) survive the SW forwarding round-trip — via a dedicated forwarding helper on `IdentityRequestEvent`, or via broader Fetch / Request spec changes.

# Acknowledgements

This proposal builds on discussion in [Issue #80](https://github.com/w3c-fedid/FedCM/issues/80), the [WG meeting on 23 September 2025](https://github.com/w3c-fedid/meetings/blob/main/2025/2025-09-23-FedCM-notes.md), and the [Chromium service-worker-discuss thread](https://groups.google.com/a/chromium.org/g/service-worker-discuss/c/t9d33x6l718), with feedback from FedCM editors, service-worker editors (Ben Kelly, Dominic Farolino), the WG privacy reviewer, and [aaronpk's OAuth profile for FedCM](https://github.com/aaronpk/oauth-fedcm-profile) which framed the scope of the IDP-cookie bearer problem this proposal targets.

# References

- [FedCM Specification](https://fedidcg.github.io/FedCM/)
- [Spec PR #815 — Enabling IDP Interception in FedCM Request](https://github.com/w3c-fedid/FedCM/pull/815)
- [GitHub Issue #80 — SW interception for FedCM](https://github.com/w3c-fedid/FedCM/issues/80)
- [WG meeting notes (23 Sept 2025)](https://github.com/w3c-fedid/meetings/blob/main/2025/2025-09-23-FedCM-notes.md)
- [Chromium service-worker-discuss thread](https://groups.google.com/a/chromium.org/g/service-worker-discuss/c/t9d33x6l718)
- [Service Worker Specification](https://w3c.github.io/ServiceWorker/)
- [Fetch Specification](https://fetch.spec.whatwg.org/)
- [Payment Handler API](https://w3c.github.io/web-based-payment-handler/)
- [DPoP (RFC 9449)](https://www.rfc-editor.org/rfc/rfc9449)
