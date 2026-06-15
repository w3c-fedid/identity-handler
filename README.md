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

# Privacy Considerations

## Configuration endpoints are protected

Configuration endpoints (`/.well-known`, `/config.json`, `/client_metadata`) are **not** dispatched to the SW for privacy reasons:

- These endpoints are fetched with privacy-preserving properties (`credentials: "omit"`, opaque origin, no referrer).
- If intercepted, the SW could correlate user identity (via cookies from its own origin) with RP identity (from `client_metadata` URL parameters).
- Keeping these endpoints out of SW scope preserves the privacy boundary.

## IDP trust model

The Identity Handler SW runs at the IDP's origin and sees the same information the IDP server already sees:

- `/accounts` (GET) — no RP identity visible (opaque origin).
- `/token` (POST) — RP `client_id` in the POST body (already visible to IDP server).
- `/disconnect` (POST) — RP `client_id` in the POST body (already visible to IDP server).

No new information is exposed beyond what the IDP server already receives.

## User consent

User consent is still required before any tokens are issued. The SW cannot bypass the FedCM consent UI — it only augments the network layer between the browser and the IDP server.

## Attacks considered

The credentialed-endpoints-only scoping was the result of explicit attack analysis on Issue #80 and cleared privacy review in the WG:

- **Attack A — RP iframes IDP, then invokes FedCM.** The RP embeds the IDP in an iframe so the IDP's SW (or storage) sees the RP's identity, then invokes FedCM; the IDP SW correlates the RP with the user on the accounts call. **Conclusion:** does not introduce new surface — browsers already partition IndexedDB and other SW-accessible storage by top-frame origin, so an iframed IDP cannot stash RP context that a top-frame FedCM SW can read.
- **Attack B — uncredentialed endpoint logs the RP, credentialed endpoint reads it.** If SW interception were enabled on uncredentialed endpoints (e.g., `client_metadata`), the IDP SW could record the RP identity from that call and read it back during the subsequent accounts call — combining user identity (via cookies) and RP identity in the same SW context, before the FedCM consent UI fires. **Conclusion:** real attack. Resolved by restricting SW interception to credentialed endpoints only (accounts, id_assertion, disconnect) and continuing to fetch configuration / metadata / well-known endpoints UA-direct with the existing privacy properties.

# Security Considerations

## Origin isolation

The Identity Handler SW is registered at the IDP origin and can only intercept requests destined for that origin. Cross-origin Service Worker interception is architecturally impossible:

```
✅ idp.example's SW intercepts: requests TO idp.example/accounts
❌ idp.example's SW cannot intercept: requests TO other-idp.com/accounts
❌ rp.com's SW cannot intercept: requests TO idp.example/accounts
```

## CSRF marker

On the UA-direct path, the browser stamps `Sec-Fetch-Dest: webidentity` — a forbidden header JavaScript cannot set — so the IDP can confirm the request came from the FedCM API, not from page-initiated script. In the SW path, `event.request.destination` is read-only and equal to `"webidentity"`.

When the SW forwards the request via `fetch(event.request)`, the WG aligned (in [Issue #833](https://github.com/w3c-fedid/FedCM/issues/833)) on **not** propagating `Sec-Fetch-Dest: webidentity` on the SW-initiated call. Instead, IDP servers verify the request source using the standard `Sec-Fetch-Site: same-origin` header — the SW-initiated `fetch` to the same IDP origin reliably carries that signal, and the IDP backend treats it as the FedCM CSRF marker on the SW path.

## Redirect prevention

FedCM requests use `redirect mode: "error"` — the SW cannot redirect requests to different URLs. The WG agreed in [Issue #833](https://github.com/w3c-fedid/FedCM/issues/833) that **requiring same-origin requests from the SW** effectively prevents cross-origin redirects as a risk vector, so an additional response-type filter is not required on the SW path (see [Open Questions](#open-questions)).

## Augmenting headers

When the SW forwards a FedCM request, it may attach additional headers (e.g., a `DPoP` proof). Headers UA-set on the FedCM internal request (`Accept`, `Sec-Fetch-*`, `Cookie`, `Origin`, `Referer`) must **not** be overridden by the SW. The exact set of header names the SW is permitted to add, and the process for growing that set, is still being worked out.

## Cookie scope

The WG aligned in [Issue #833](https://github.com/w3c-fedid/FedCM/issues/833) that the cookie-scope invariant can be relaxed: when the SW forwards via `fetch(event.request)`, the IDP receives the full cookie jar — including `SameSite=Lax` / `Strict` cookies — rather than only `SameSite=None`. This is acceptable because the SW runs at the IDP origin and the cookies are scoped to that origin already; the relaxation is a consequence of reusing standard Service Worker `fetch` semantics rather than defining FedCM-specific exceptions.

## Robust fallback

The implementation uses a "skip flag" pattern for fallback:

1. If SW dispatch fails → emit console warning.
2. Set `skip_identity_handler` flag → bypass SW on retry.
3. Re-issue the same request via normal network fetch.
4. Clear the flag for future requests.

This ensures that SW failures never block authentication — the flow degrades gracefully to the non-SW path.

# Developer Experience

**IDP developers** opt in by:

1. Registering a Service Worker that handles FedCM (registration mechanism is being finalized — see [Open Questions](#open-questions)).
2. Implementing an `identityrequest` event handler in their SW.
3. Using standard web APIs (Fetch, Cache, Crypto) within the handler, subject to the augmenting-header allowlist.

**RP developers**: No changes required. Completely transparent.

**Debugging:** Console warnings are emitted when the SW fails — e.g., "FedCM: No identity handler service worker registration found for the provider. Falling back to network."

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

The full design discussion is in the [explainer](https://github.com/w3c-fedid/FedCM/blob/main/explorations/identity_handler.md). Several questions raised on [Issue #833](https://github.com/w3c-fedid/FedCM/issues/833) have been resolved by the WG; a few remain open.

## Resolved

- **a) Dispatch shape — dedicated event.** RESOLVED in favor of a purpose-built `IdentityRequestEvent` rather than reusing `FetchEvent`. The semantics are fundamentally different — FedCM requests are UA-initiated rather than document-bound — and a dedicated interface avoids silently invoking pre-existing IDP `fetch` handlers. Reusing `respondWith()` as a method name is acceptable since the FedCM-specific processing logic is unambiguous in this context. ([Issue #833](https://github.com/w3c-fedid/FedCM/issues/833))
- **Response-type restriction — relaxed.** RESOLVED: the response-type check on `respondWith()` is not required. Requiring the SW to issue same-origin requests when it forwards the FedCM call gives the same `redirect mode: "error"` guarantee at the server layer, so an additional UA-side response-type filter would be redundant.
- **Cookie-scope invariant — relaxed.** RESOLVED: the SW-mediated request carries the full IDP cookie jar (including `SameSite=Lax` / `Strict`), not only `SameSite=None`. This is a consequence of reusing standard Service Worker `fetch` semantics; introducing FedCM-specific cookie exceptions was rejected as too invasive for the platform.
- **CSRF marker on the SW path.** RESOLVED: the SW-initiated `fetch` does not propagate `Sec-Fetch-Dest: webidentity`. IDP servers should instead verify the request source via `Sec-Fetch-Site: same-origin`, which the SW-initiated call to the same IDP origin carries reliably.
- **`client_metadata` exclusion — confirmed.** RESOLVED: `client_metadata` (an uncredentialed request that reveals `rp_client_id`) must not be dispatched through the SW, since combining it with a credentialed accounts call in the same SW context would let the IDP join user identity with RP identity before consent. Enforcement is by filtering — FedCM-initiated requests bypass the standard fetch handler by default.

## Still open

- **b) Service Worker registration mechanism.** Two approaches are still on the table; the most recent direction is captured in [PR #834](https://github.com/w3c-fedid/FedCM/pull/834) and the [service_worker_model.md](https://github.com/w3c-fedid/FedCM/blob/main/explorations/service_worker_model.md) exploration:
  - **IDP-managed registration via `configURL`.** The IDP page calls a JS API to bind a service worker registration to a FedCM `configURL`. No SW path in the well-known file.
  - **UA-managed registration via `/.well-known/web-identity`** (current direction in PR #834). The well-known file declares an `identity_handler` with a `service_worker` script URL and `scope`. The UA performs the registration itself, into an **isolated `StorageKey`** (nonce-based) so it cannot collide with any page-initiated `navigator.serviceWorker.register()` for the same path. The UA fetches the well-known on every FedCM call, runs Match Service Worker Registration against the FedCM-managed `StorageKey`, and either dispatches to the matched registration (scheduling Soft Update in parallel) or registers + activates and then dispatches.

  Sub-questions under the UA-managed direction:
  - **Onboarding existing users.** How do already-signed-in users get the SW registered without a fresh visit to the IDP page?
  - **Bad-rollout recovery.** Soft Update is governed by HTTP cache freshness, so a broken SW can persist until the cache expires. A control knob (e.g., `sw_version` in the well-known) that forces a hard update bypassing `Cache-Control` is under discussion.
  - **Storage pressure.** Cookies survive storage eviction but the SW registration does not — leaving a dangling FedCM registration ID. Whether the UA re-registers on the next FedCM call, or relies on the user revisiting the IDP page, is open.
  - **Well-known reliability.** Fetching the well-known on every FedCM call makes the flow sensitive to transient network errors. Aligning the well-known check cadence with the Soft Update cadence is one option under discussion.

## Backed out

- **SW URL declared in `config.json`.** Fails privacy — the IDP could encode RP-specific values into the URL (`/sw-for-rp-A.js` vs. `/sw-for-rp-B.js`), so the SW (and the server serving the SW script) would learn which RP the user is interacting with.
- **Propagating `Sec-Fetch-Dest: webidentity` on SW-initiated `fetch`.** Would require Fetch-spec exceptions to let a service worker set a UA-reserved `destination` value — rejected as too invasive for the platform. Replaced by the `Sec-Fetch-Site: same-origin` server-side check above.

# Acknowledgements

This proposal builds on discussion in [Issue #80](https://github.com/w3c-fedid/FedCM/issues/80), [Issue #833](https://github.com/w3c-fedid/FedCM/issues/833), the [WG meeting on 23 September 2025](https://github.com/w3c-fedid/meetings/blob/main/2025/2025-09-23-FedCM-notes.md), follow-up FedCM calls on [2 June 2026](https://github.com/w3c-fedid/meetings/blob/main/2026/2026-06-02-FedCM-notes.md#enabling-idp-interception-in-fedcm-request-815) and [9 June 2026](https://github.com/w3c-fedid/meetings/blob/main/2026/2026-06-09-FedCM-notes.md), and the [Chromium service-worker-discuss thread](https://groups.google.com/a/chromium.org/g/service-worker-discuss/c/t9d33x6l718), with feedback from FedCM editors, service-worker editors (Ben Kelly, Dominic Farolino, Yoshisato Yanagisawa), the WG privacy reviewer, and [aaronpk's OAuth profile for FedCM](https://github.com/aaronpk/oauth-fedcm-profile) which framed the scope of the IDP-cookie bearer problem this proposal targets.

# References

- [FedCM Specification](https://fedidcg.github.io/FedCM/)
- [Spec PR #815 — Enabling IDP Interception in FedCM Request](https://github.com/w3c-fedid/FedCM/pull/815)
- [Spec PR #834 — Service Worker registration strategy](https://github.com/w3c-fedid/FedCM/pull/834)
- [GitHub Issue #80 — SW interception for FedCM](https://github.com/w3c-fedid/FedCM/issues/80)
- [GitHub Issue #833 — Resolutions on dispatch, response type, cookie scope, and CSRF marker](https://github.com/w3c-fedid/FedCM/issues/833)
- [Explainer — identity_handler.md](https://github.com/w3c-fedid/FedCM/blob/main/explorations/identity_handler.md)
- [Service Worker model exploration](https://github.com/w3c-fedid/FedCM/blob/main/explorations/service_worker_model.md)
- [WG meeting notes (23 Sept 2025)](https://github.com/w3c-fedid/meetings/blob/main/2025/2025-09-23-FedCM-notes.md)
- [Chromium service-worker-discuss thread](https://groups.google.com/a/chromium.org/g/service-worker-discuss/c/t9d33x6l718)
- [Service Worker Specification](https://w3c.github.io/ServiceWorker/)
- [Fetch Specification](https://fetch.spec.whatwg.org/)
- [Payment Handler API](https://w3c.github.io/web-based-payment-handler/)
- [DPoP (RFC 9449)](https://www.rfc-editor.org/rfc/rfc9449)
