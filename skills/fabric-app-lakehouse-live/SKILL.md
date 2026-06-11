---
name: fabric-app-lakehouse-live
description: Use when a web app (especially a Fabric App SPA) must read EXISTING Lakehouse data live with no backend — "query the Lakehouse from the browser", "live data without a server", "call Fabric GraphQL from my SPA", or when debugging that path: 403 InsufficientPrivileges on a Fabric GraphQL API, MSAL sign-in returning to a blank page, popup "tela preta", token works but queries fail, rows silently missing. Read-only by design — the Lakehouse SQL endpoint has no mutations and syncs lazily. If the app writes data back, use fabric-app-sqldb-writeback instead. For creating/deploying the app shell, use fabric-app-bootstrap.
license: MIT
metadata:
  author: Gus Bavia
  version: 0.1.0
  category: microsoft-fabric
  tags: [fabric, graphql, lakehouse, msal, entra, sso, delegated-token, spa, live-data, read-only, GraphQLApi-Execute-All, cors, jwt]
  companions: [fabric-app-bootstrap, fabric-app-sqldb-writeback]
  support: https://linkedin.com/in/gusbavia
---

# Fabric App · Lakehouse Live Bridge

The browser acquires a **delegated Fabric token** (the signed-in user's own identity) and POSTs straight to a Lakehouse "API for GraphQL" endpoint. Live data, per-user permissions and RLS, zero infrastructure, no secrets in the bundle.

```
browser → MSAL (Entra SSO, delegated) → POST → GraphQL API over the Lakehouse
```

A working reference implementation exists in the `fabric-apps-starter` repo, `template-lakehouse-live/` (Vite + React + MSAL, query console + token inspector).

## The recipe — order matters, each step gates the next

**1. Validate CORS before writing code.** Preflight the GraphQL endpoint with the app origin:

```bash
curl -X OPTIONS <graphql-endpoint> -H "Origin: https://<your-app-url>" \
  -H "Access-Control-Request-Method: POST" -i
# want: 200 + Access-Control-Allow-Origin echoing your origin
```

Fabric allows Fabric Apps origins by default. If this fails, nothing downstream can work.

**2. MSAL singleton, initialized BEFORE render.** When the browser returns from `login.microsoftonline.com`, the SPA boots again and MSAL must parse the auth response on that load:

```ts
// lib/msal.ts
export const msal = new PublicClientApplication({
  auth: { clientId, authority: `https://login.microsoftonline.com/${tenantId}`,
          redirectUri: window.location.origin },
  cache: { cacheLocation: "sessionStorage" },
});
export async function bootstrapMsal() {
  await msal.initialize();
  await msal.handleRedirectPromise();   // skip this → sign-in silently never lands
}
// main.tsx: bootstrapMsal() runs before createRoot(...).render(...)
```

Prefer **`loginRedirect` over popups** for SPAs: no blocked windows, no blank popup pages, works embedded and standalone.

**3. App registration (Entra portal):**
- Authentication → add platform → **Single-page application** → redirect URI = the app URL (plus `http://localhost:5173` for dev). Authorization Code + PKCE; leave both implicit-grant checkboxes UNchecked.
- API permissions → Power BI Service → **Delegated** → **`GraphQLApi.Execute.All`** → grant admin consent. This is THE permission that gates query execution; `Tenant/Workspace/Dataset/Item.Read.All` all return 403 without it.

**4. Token + query:**

```ts
let tok;
try {
  tok = await msal.acquireTokenSilent({
    account, scopes: ["https://api.fabric.microsoft.com/.default"],
  });
} catch (e) {
  if (e instanceof InteractionRequiredAuthError) {
    tok = await msal.acquireTokenPopup({ account, scopes: [...] }); // fallback
  } else throw e;
}
const res = await fetch(GRAPHQL_ENDPOINT, {
  method: "POST",
  headers: { Authorization: `Bearer ${tok.accessToken}`,
             "Content-Type": "application/json" },
  body: JSON.stringify({ query }),
});
```

**5. Workspace access:** the signed-in user needs access to the GraphQL API item (direct "Run Queries and Mutations" grant, or a workspace role).

## Query shape (Fabric GraphQL over a Lakehouse)

Types are **pluralized**, fields **snake_case**, rows under `items`:

```graphql
{ dim_employees(first: 500) { items { employee_id employee_name } } }
```

Introspection is usually disabled — get the schema from the GraphQL item's editor in the portal, not from an introspection query.

**Always set `first` explicitly and size it generously.** Omitted or small `first` caps the result and sorted reads drop the tail with NO error. Raising `first` well past the table size (e.g. `first: 50000` on a 5k-row table) is the proven simple fix; after any data growth, verify returned row counts against the source.

## Debug pattern: decode the token on screen

A 403 is almost always a missing scope. Decode the JWT payload and surface `aud` + `scp` in the UI; the diagnosis becomes one glance:

```ts
JSON.parse(atob(token.split(".")[1].replace(/-/g, "+").replace(/_/g, "/")));
// want: aud = https://api.fabric.microsoft.com, scp includes GraphQLApi.Execute.All
```

## Gotchas

| Symptom | Cause / fix |
|---|---|
| Sign-in completes but no session | `handleRedirectPromise()` not called before render. |
| `AADSTS50011` redirect URI error | App URL not registered as an **SPA** platform (not "Web"). |
| 403 InsufficientPrivileges with a valid token | Missing delegated `GraphQLApi.Execute.All` or missing admin consent. If the decoded `scp` ALREADY shows the scope, the cause is item access instead — grant the user "Run Queries and Mutations" on the GraphQL API item or a workspace role. |
| Fixed the permission, still 403 | The cached token predates the consent. Sign out (or clear sessionStorage) and sign in again so the fresh token carries the new `scp`. |
| Rows silently missing at scale | `first: N` caps the result; sorted reads drop the tail QUIETLY. Raise `first` or paginate — verify row counts against the source. |
| Saved data takes minutes-to-a-day to appear | Lakehouse SQL endpoint syncs lazily. This path is read-only by design; write-back belongs in fabric-app-sqldb-writeback. |

## When NOT to use

Any transactional write-back. The endpoint is read-only, has no mutations, and its lazy sync makes "write somewhere else, read it back here" feel broken to users.
