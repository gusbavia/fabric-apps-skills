# fabric-app-bootstrap

Create and deploy a Microsoft Fabric App (preview) from the CLI — including fully headless with a service principal, zero portal clicks. Covers what Fabric Apps will and will not host, the `rayfin init/login/up` recipe, the Vite config that survives static hosting, and the deploy gotchas (owner-only schema applies, 100 MB cap, HEAD-500 quirk).

Companion to **fabric-app-lakehouse-live** (read-only data layer) and **fabric-app-sqldb-writeback** (transactional write-back).

## Quick start

```text
@fabric-app-bootstrap  deploy this Vite app to my Fabric workspace, headless with my SP
```

You provide: workspace id, App item id, and (headless path) SP client id/secret/tenant. Never hardcode them — env vars or a local gitignored file.
