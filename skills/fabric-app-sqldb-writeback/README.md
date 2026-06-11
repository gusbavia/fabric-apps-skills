# fabric-app-sqldb-writeback

The architecture for Fabric Apps that WRITE: the app owns its SQL Database, a ~30-second job mirrors Lakehouse Gold into it (headless, service principal over TDS/pyodbc), and write-back entities live alongside the mirrors. One database, one SDK, instant read-your-own-writes — no lazy-sync pain, no dual-source merge logic.

Covers: entity modeling (mirrors + write-back), owner-only schema applies, the TDS access-token connection recipe, wire gotchas (pluralized tables, reserved words, `fast_executemany` truncation), SDK gotchas (date auto-conversion on text columns), and end-to-end verification.

Companion to **fabric-app-bootstrap** (app shell + deploy) and **fabric-app-lakehouse-live** (read-only apps).

## Quick start

```text
@fabric-app-sqldb-writeback  add write-back to my Fabric app: mirror these Gold tables + a Comment entity
```
