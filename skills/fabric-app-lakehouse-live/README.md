# fabric-app-lakehouse-live

Read EXISTING Lakehouse data live from the browser with the signed-in user's own delegated token — no backend, no secrets in the bundle, per-user permissions for free. Covers the full chain in dependency order: CORS preflight, MSAL singleton (initialize before render), SPA app registration + PKCE, the one permission that matters (`GraphQLApi.Execute.All`), token acquisition, Fabric GraphQL query shape, and the on-screen JWT decoder debug pattern.

Read-only by design. Companion to **fabric-app-bootstrap** (app shell + deploy) and **fabric-app-sqldb-writeback** (when users write data back).

A working reference app ships in the `fabric-apps-starter` repo (`template-lakehouse-live/`).

## Quick start

```text
@fabric-app-lakehouse-live  wire my SPA to query my Lakehouse GraphQL API as the signed-in user
```
