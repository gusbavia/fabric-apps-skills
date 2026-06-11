---
name: fabric-app-sqldb-writeback
description: Use when a Fabric App needs transactional WRITE-BACK (comments, edits, requests, approvals) on top of Lakehouse data — "users edit data in my Fabric app", "write back to the source", "my saved comment takes forever to appear", "read-your-own-writes", "mirror Gold into the app database", "seed the Rayfin SQL DB headless", or when the app is juggling multiple data tiers/auth flows and needs consolidation. Encodes the operational-data-store architecture: app owns its SQL DB, a copy job mirrors Lakehouse Gold into it, one SDK for everything. For read-only apps use fabric-app-lakehouse-live; for app creation/deploy use fabric-app-bootstrap.
license: MIT
metadata:
  author: Gus Bavia
  version: 0.1.0
  category: microsoft-fabric
  tags: [fabric-apps, rayfin, sql-database, write-back, tds, pyodbc, service-principal, mirror, operational-data-store, entities, dab, read-your-own-writes, seed, refresh]
  companions: [fabric-app-bootstrap, fabric-app-lakehouse-live]
  support: https://linkedin.com/in/gusbavia
---

# Fabric App · SQL DB Write-back

The architecture for Fabric Apps that write: the app owns its **SQL Database** (a managed child of the App item), a small job **mirrors Lakehouse Gold into it**, and write-back entities live alongside the mirrors. One database, one SDK, instant read-your-own-writes.

```
Lakehouse (canonical, untouched)
      ↓ copy job (~30s, headless via SP over TDS)
App SQL Database
   ├─ mirror entities (DimX, FactY...)      ← refreshed by re-running the job
   └─ write-back entities (Comment, Edit…)  ← transactional, instant
      ↓ ONE SDK (RayfinClient)
SPA — every read and write
```

## Why not write to the Lakehouse directly (the scars)

1. Its SQL endpoint is **read-only** — the GraphQL over it has no mutations.
2. Its sync is **lazy** — a write landed in OneLake in seconds and surfaced through the API a day later. Users will not wait a day to see their own comment.
3. Two sources = two auth flows + dual-source merge logic + optimistic-cache band-aids in the frontend, forever.

The copy job moves that complexity out of the frontend into one script. Microsoft's name for this serving pattern: **operational data store**.

## Step 1 · Model entities (TypeScript decorators)

Mirror entities for the Gold tables the app reads + the app's own write-back entities. Keep mirror columns **text-friendly** (no enum constraints — source data will surprise you). Register everything in the schema the client SDK is typed against.

```ts
// rayfin/data/DimCustomer.ts — a mirror entity, minimal shape
@entity()
export class DimCustomer {
  @field({ type: "uuid", primaryKey: true }) id!: string;
  @field({ type: "text", unique: true }) customer_id!: string;
  @field({ type: "text" }) customer_name!: string;
  @field({ type: "text", optional: true }) segment?: string;
}
// register it in rayfin/data/schema.ts so the client SDK is typed
```

**Schema apply is OWNER-ONLY:** `npx rayfin up db apply` needs one interactive login by the item owner. The SP cannot apply schema (403 "Only AppBackend artifact owner") — but handles everything else headless.

**Never DROP entities, only ADD.** Dropping requires a force flag the server currently ignores; a blocked apply costs a night, a retired entity costs nothing.

## Step 2 · Seed + refresh over TDS (the automation unlock)

The app's GraphQL **data plane accepts delegated user tokens only** (SP tokens are rejected regardless of headers). The unlock: the SQL Database child has a real **TDS endpoint**, and a workspace-Contributor SP connects with plain pyodbc — read AND write, fully headless:

```python
tok = credential.get_token("https://database.windows.net/.default")
packed = struct.pack(f"<I{len(t)}s", len(t), t) if (t := tok.token.encode("utf-16-le")) else None
conn = pyodbc.connect(
    "Driver={ODBC Driver 17 for SQL Server};"
    "Server=<db-host>.database.fabric.microsoft.com,1433;"
    "Database=<db-name>;",
    attrs_before={1256: packed},   # SQL_COPT_SS_ACCESS_TOKEN
)
# mirrors: DELETE + reload (idempotent). write-back tables: insert-only with dedup.
```

Find the server/database names on the SQL Database child item in the portal. **Re-running this script IS the refresh** — schedule it (cron/Actions/pipeline) or trigger it from the UI via a delegated pipeline-run REST call.

**Reading the SOURCE side (Lakehouse Gold) in the same script — proven path:** read the Delta tables straight from OneLake with `deltalake` (delta-rs), no Spark:

```python
tok = credential.get_token("https://storage.azure.com/.default")  # data-plane scope
df = DeltaTable(
    f"abfss://{WORKSPACE_ID}@onelake.dfs.fabric.microsoft.com/{LAKEHOUSE_ID}/Tables/dim_customer",
    storage_options={"bearer_token": tok.token, "use_fabric_endpoint": "true"},
).to_pyarrow_table()
```

Same SP, two tokens: `storage.azure.com` scope to READ Gold, `database.windows.net` scope to WRITE the app DB.

**One-time backfill of existing write-back rows** (e.g. comments previously written to the Lakehouse): same script, read the old table from Gold, insert into the new entity with the dedup rule below.

### TDS wire gotchas

| Gotcha | Detail |
|---|---|
| Tables are **pluralized** | entity `DimEmployee` → `dbo.DimEmployees` on the wire (the data-API layer names them). Don't guess irregular cases — confirm with `SELECT name FROM sys.tables` |
| Reserved words | columns named `plan`, `usage` etc. need `[brackets]` in T-SQL |
| `fast_executemany` truncates | string buffer sized from the FIRST row; longer strings later get cut. Use it only on all-numeric facts |
| Dedup key for insert-only tables | dedup write-back rows on a natural composite key (e.g. `author|text|created_at`), never on the surrogate id — re-runs must not duplicate history |

## Step 3 · One SDK in the frontend

Every read and write goes through the client SDK against the app DB. Short cache (~60s) on mirror reads; write-back reads always fresh. Auth: silent SSO bootstrap when embedded in the Fabric portal; explicit sign-in gate standalone.

### SDK gotchas

| Gotcha | Detail |
|---|---|
| **Date auto-conversion** | the SDK converts ISO-looking strings to `Date` objects EVEN ON TEXT COLUMNS. Guard every date-ish field back to string at the fetch boundary or `.slice()` crashes a page |
| Endpoint is `/graphql` | docs say `/api/graphql`; the SDK source is the truth |
| Embedded vs standalone | test flows embedded in the portal first; standalone tabs have flaky token refresh |
| Pagination | use the SDK's cursor loop for >1k rows; a flat `first(1000)` silently caps |

## Step 4 · Verify end-to-end

Write from the UI → row visible in the DB inspector/SQL immediately → re-read in the app shows it **instantly** (same DB, no sync lag). If a "saved" item lags, you are accidentally still reading the Lakehouse path.

## Bonus finding

"API for GraphQL" exists on the SQL Database item too — same menu item as the Lakehouse one, but **transactional with mutations**. Useful as an external-consumer API over the app DB (scripts, Power Automate, a second app) without TDS. Note the distinction: this is a separate Fabric item over the same DB, NOT the app's own data plane (which stays delegated-user-only). For headless automation, TDS is the proven path; treat the SQL DB GraphQL as plan-B and verify SP auth on it before depending on it.
