# Changelog

## 0.1.0 — 2026-06-11

First public release. Three skills, extracted from a real end-to-end build (Next.js team dashboard → Fabric-native app with transactional write-back), each application-tested with agent scenarios before release.

- `fabric-app-bootstrap` — platform constraints, `rayfin` CLI recipe, headless service-principal deploys, Next.js → Vite port pattern, deploy gotchas.
- `fabric-app-lakehouse-live` — browser → delegated SSO → Lakehouse GraphQL, in dependency order, with the debugging patterns.
- `fabric-app-sqldb-writeback` — operational-data-store architecture: mirrors + write-back entities, TDS/pyodbc seed-and-refresh recipe, wire/SDK gotchas.

Verified against: `@microsoft/rayfin-cli` 1.33.x, `@azure/msal-browser` 5.12.x, Fabric Apps preview as of June 2026.
