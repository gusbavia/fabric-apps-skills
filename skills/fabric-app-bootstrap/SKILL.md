---
name: fabric-app-bootstrap
description: Use when creating or deploying a Microsoft Fabric App (preview, Rayfin CLI) — "build my first Fabric App", "host this SPA inside Fabric", "deploy to Fabric Apps", "rayfin up fails", "what can Fabric Apps host", or when a deploy needs to run headless/CI with a service principal and zero portal clicks. Also use BEFORE porting an existing app into Fabric, to check what survives static hosting (no Node SSR, no server secrets, no Python runtime, 100 MB zip cap). Do NOT use for the data layer — that is fabric-app-lakehouse-live (read-only) or fabric-app-sqldb-writeback (write-back).
license: MIT
metadata:
  author: Gus Bavia
  version: 0.1.0
  category: microsoft-fabric
  tags: [fabric-apps, rayfin, deploy, service-principal, vite, spa, static-hosting, headless, ci, preview]
  companions: [fabric-app-lakehouse-live, fabric-app-sqldb-writeback]
  support: https://linkedin.com/in/gusbavia
---

# Fabric App Bootstrap

Stand up and deploy a Microsoft Fabric App (preview) from the CLI, including fully headless with a service principal. Distilled from a real production-shaped build; every gotcha here cost real hours.

## What a Fabric App is (and is not)

One workspace item ("App", backed by an AppBackend) with three managed children:

| Service | What it gives you |
|---|---|
| **Static hosting** | your built SPA on a public URL (served from OneLake) |
| **SQL Database** | schema generated from TypeScript `@entity()` decorators |
| **GraphQL API** | data plane the client SDK talks to, behind Entra SSO |

**It will NOT host:** Node SSR (no Next.js server), server-held secrets (no BFF with a client secret), Python or any custom runtime. The deploy is a zip of `dist/`, capped at **100 MB compressed**. End-user auth on a deployed app is **Entra SSO only**.

**Porting check:** if the existing app fetches server-side with a secret, that code cannot board. Visual/client code survives intact. Route the data layer through one of the companion skills.

**Porting from Next.js (proven pattern, ~30 min/page):** rebuild the shell as a Vite SPA, then alias `next/link` and `next/navigation` to two ~25-line shims in `vite.config.ts` (`resolve.alias`) so page components, CSS modules and shared components copy over VERBATIM with zero import rewrites. Wrap async server components in a client component with `useState` + `useEffect`. Re-export the old data layer's types and swap only the implementation.

**Honest scoping of "zero clicks":** steady-state deploys are fully headless. First-run is not: the App item is created in the portal, two tenant settings are admin toggles, and if the app has a database the schema apply needs one interactive owner login (see gotchas). Say this up front when planning CI.

## Prerequisites

- Tenant setting **"Fabric App items (preview)"** enabled (Fabric admin portal)
- A workspace + an App item created in it (portal: + New item → App)
- Node 18+, a Vite SPA (React/Vue/Svelte — templates are Vite)
- Headless path only: an Entra service principal with **Contributor** on the workspace and tenant setting "Service principals can use Fabric APIs" = On

## The recipe

```bash
# 1. one-time: bind the local project to the EXISTING Fabric item
#    (records the item id in rayfin/.deployments.json so it never duplicates)
npx rayfin -y init . --project-name <app> \
  --services auth --auth-methods fabric \
  --workspace-id <ws-guid> --item-id <item-guid>

# 2a. interactive login (item OWNER). If the app has a DATABASE this is
#     required at least once — schema applies are owner-only, the SP can't.
npx rayfin login

# 2b. headless login for CI / automation (static deploys + data plane).
#     Not either/or with 2a: DB apps need 2a once, then 2b for every deploy.
#     In CI, read the secret from an env var — a literal on the command line
#     leaks into logs and process lists.
npx rayfin login --service-principal \
  --client-id "$SP_CLIENT_ID" --client-secret "$SP_CLIENT_SECRET" --tenant "$TENANT_ID"

# 3. full deploy: build + zip + upload, prints the live URL
npx rayfin up -y

# 4. iterate (static tier only, much faster)
npx rayfin up staticapp deploy -y
```

## Vite config that survives Fabric hosting

```ts
// vite.config.ts
export default defineConfig({
  plugins: [react()],
  base: "./",            // relative assets — works under any URL prefix
  build: { outDir: "dist" },
});
```

Use **HashRouter** (or equivalent) for routing: static hosting serves one `index.html`, so `#/routes` survive deep links and refreshes; path routes 404.

## Gotchas (each one is a scar)

| Symptom | Cause / fix |
|---|---|
| `rayfin up` schema step fails 403 "Only AppBackend artifact owner" | DB schema applies are **owner-only**. SP cannot do it. Owner runs `npx rayfin login` once interactively; SP handles everything else. |
| Deploy balloons / rejected | 100 MB zip cap. Convert images to WebP (`sharp`, quality ~82) before they bite. |
| `curl -I` on the live URL returns 500 | Harmless platform quirk on HEAD. Verify with GET or a browser. |
| Re-deploy says login expired | SP token lasts ~1h. Re-run the same `rayfin login --service-principal`. |
| `rayfin up db apply` compile error TS5096 | Root tsconfig has `allowImportingTsExtensions: true`; the `rayfin/tsconfig.json` must override it to `false` (it needs to emit). |
| Functions tier 404 on `__private/functions/deploy` | `rayfin functions` is experimental and not provisioned in every region. Keep `services.functions: false` until it deploys. |

## Verify the deploy

GET the printed URL in a browser. Static content is publicly reachable; data calls gate on SSO. Test SDK auth flows **embedded in the Fabric portal first** — standalone browser tabs have flakier token refresh.
