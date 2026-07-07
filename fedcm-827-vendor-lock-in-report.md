# FedCM Issue #827 — Vendor Lock-In via the Single-Provider Well-Known Constraint

**Reference:** [w3c-fedid/FedCM#827 — "FedCM facilitates vendor lock-in"](https://github.com/w3c-fedid/FedCM/issues/827)
**Related tracked issues:** [#333 (relax provider_urls size)](https://github.com/fedidcg/FedCM/issues/333), [#552 (multiple config files within an eTLD+1)](https://github.com/w3c-fedid/FedCM/issues/552)

---

## 1. The Issue — Stated Exactly

FedCM requires an Identity Provider (IdP) to expose a **single** `/.well-known/web-identity`
file at its **registrable domain (eTLD+1)**, and that file may name **exactly one** provider
(one config URL, or one `accounts_endpoint` + `login_url` pair). This makes it structurally
impossible to run **two independent FedCM IdPs on subdomains of the same registrable domain
at the same time**, which is the normal shape of a gradual **vendor-to-vendor migration**.

### Motivating real-world scenario

Companies frequently use **white-label auth vendors** (Okta, Microsoft Entra, etc.) hosted on
**CNAME'd subdomains of the company's own domain**, so the vendor's brand never appears in the
URL bar or certificate:

- Old vendor → `login.contoso.example`
- New vendor → `auth.contoso.example`

Both subdomains roll up to the same registrable domain, **`contoso.example`**.

- **With OAuth:** there is no protocol-level restriction. The company can stand up the new
  vendor on `auth.contoso.example`, keep the old vendor on `login.contoso.example`, and cut
  applications over gradually (percentage-based / canary rollout across many product apps).
- **With FedCM:** the single well-known file at `contoso.example` can point to only **one**
  accounts endpoint. The two vendors must share that one file, so only one of them can be the
  valid FedCM IdP at any moment. Parallel coexistence during migration is **blocked**.

The net effect the reporter names: **FedCM raises the switching cost between identity vendors
compared to OAuth — i.e., it facilitates vendor lock-in.**

---

## 2. Where the Spec Fails to Address It

The failure is baked into two spec facts working together.

### Fact A — The well-known dictionary cannot express more than one provider

```webidl
dictionary IdentityProviderWellKnown {
  sequence<USVString> provider_urls;   // config URLs — but capped at 1
  USVString accounts_endpoint;         // single scalar (not a list)
  USVString login_url;                 // single scalar (not a list)
};
```

- `accounts_endpoint` and `login_url` are **single scalar URLs** — you cannot list two.
- `provider_urls` is an array, but *fetch the config file* sets `wellKnown` to **failure** when
  its size is **greater than 1**. Spec semantics: *"provider_urls: A list containing exactly one
  URL."*
- *"Either `provider_urls` **or** both `accounts_endpoint` and `login_url` are required"* — you
  supply one provider description, never a set.

### Fact B — Config validation rejects any provider not named by that one well-known

Inside *fetch the config file* (well-known is fetched at the IdP's eTLD+1):

- **Check accounts and login url step** — if the well-known declares `accounts_endpoint` +
  `login_url`: require `well_known_accounts_url == accounts_url` **and**
  `well_known_login_url == login_url`, else **return failure**.
- **Otherwise** — require `provider_urls[0] == configURL`, else **return failure**.

Because the cap in Fact A is enforced **per registrable domain**, and Fact B rejects anything
the single file does not name, two IdPs under one eTLD+1 can never both validate. The one FedCM
feature that genuinely supports many IdPs — the RP-side `providers` list in
`navigator.credentials.get()` — only works because **each listed IdP lives on a *different*
eTLD+1 with its *own* well-known file**. It does not help two vendors that share a domain.

> The spec authors already flag this limitation: an inline Issue note at the size check reads
> *"relax the size of the provider_urls array"* (#333), and the changelog tracks
> #552 "Allow IDPs to use multiple config files within an eTLD+1". Issue #827 is the
> vendor-lock-in argument for actually doing so.

### Diagram — happy path vs. the #827 failure path

```mermaid
sequenceDiagram
    actor User
    participant RP as "RP page"
    participant CF as "Browser: fetch the config file"
    participant WK as "well-known at eTLD+1 (contoso.example)"
    participant IdP as "IdP config (new vendor)"
    participant UI as "Account chooser"

    User->>RP: "navigator.credentials.get() with configURL"
    RP->>CF: "request identity"
    CF->>WK: "GET /.well-known/web-identity"
    WK-->>CF: "1 provider only (points to OLD vendor)"
    CF->>IdP: "GET config (accounts_endpoint, login_url)"
    IdP-->>CF: "config for auth.contoso.example (NEW vendor)"
    CF->>CF: "Check accounts and login url step"

    alt Happy path — config matches the single well-known entry
        CF->>UI: "show accounts"
        UI-->>User: "pick account, sign in"
    else #827 failure — second vendor on same eTLD+1
        CF-->>RP: "return failure (URLs not equal / not in provider_urls)"
        RP-->>User: "FedCM fails — migration blocked"
    end
```

### Diagram — why "multi-IdP" does not rescue this

```mermaid
flowchart LR
    subgraph OK["Multi-IdP that FedCM allows — DIFFERENT eTLD+1s"]
        A["idp-a.example<br/>own well-known · 1 provider"]
        B["idp-b.example<br/>own well-known · 1 provider"]
    end
    subgraph BAD["#827 — SAME eTLD+1, two vendors"]
        W["contoso.example/.well-known/web-identity<br/>(exactly 1 provider)"]
        V1["login.contoso.example<br/>(old vendor)"] --> W
        V2["auth.contoso.example<br/>(new vendor)"] -.->|"rejected: not the 1 allowed provider"| W
    end
```

### Diagram — the exact validation branch that fails

```mermaid
flowchart TD
    S["well-known + config fetched<br/>(well-known at eTLD+1)"] --> C{"well-known declares<br/>accounts_endpoint AND login_url?"}
    C -- "yes (strict-match mode)" --> D{"config.accounts == well-known.accounts<br/>AND login_url matches?"}
    D -- yes --> OK["accept → account chooser"]
    D -- no --> F1["return failure"]
    C -- "no (provider_urls mode)" --> E{"provider_urls size is 1 or fewer?"}
    E -- no --> F2["return failure (too big)"]
    E -- yes --> G{"provider_urls[0] == configURL?"}
    G -- yes --> OK
    G -- no --> F3["return failure (not in well-known)"]
    F1 --> Z["Second-vendor subdomain rejected<br/>→ vendor migration blocked"]
    F2 --> Z
    F3 --> Z
```

---

## 3. Proposed Solution — Relax `provider_urls` to a Small Bounded N

Change the well-known cap from **exactly 1** to a **small bounded number** (e.g. **N = 2 or 3**)
of provider config URLs per registrable domain. During a migration the file lists both vendors:

```json
{ "provider_urls": [
    "https://login.contoso.example/fedcm.json",
    "https://auth.contoso.example/fedcm.json"
] }
```

- **Spec change:** replace the `size > 1 -> failure` rule with `size > N -> failure`, and change
  the "Otherwise" branch from checking only `provider_urls[0]` to checking **membership** of
  `configURL` in `provider_urls`.

```mermaid
flowchart TD
    S["config fetched (configURL)"] --> E{"provider_urls size is N or fewer?"}
    E -- no --> F["failure (too big)"]
    E -- yes --> M{"configURL is a MEMBER of provider_urls?"}
    M -- yes --> OK["accept - both vendors can coexist"]
    M -- no --> F2["failure"]
```
