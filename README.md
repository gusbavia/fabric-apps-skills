# Fabric Apps Skills

[![License: MIT](https://img.shields.io/badge/License-MIT-c2e000.svg)](./LICENSE)
[![Version](https://img.shields.io/badge/version-0.1.1-201d15.svg)](./CHANGELOG.md)
[![Microsoft Fabric Apps](https://img.shields.io/badge/Microsoft%20Fabric%20Apps-preview-93b000.svg)](https://learn.microsoft.com/fabric)
[![Agent Skills](https://img.shields.io/badge/agent%20skills-3-201d15.svg)](#the-skills)

Three agent skills that teach an AI coding agent (Claude Code and compatible) how to build **Microsoft Fabric Apps** (preview): what the platform hosts, how to deploy headless, how to read Lakehouse data live from the browser, and how to architect transactional write-back.

Every pattern here was hit, broken and fixed on a live workspace before it was written down. The gotcha tables are scars, not speculation.

## The skills

| Skill | Use it when | Encodes |
|---|---|---|
| [`fabric-app-bootstrap`](./skills/fabric-app-bootstrap/) | creating or deploying a Fabric App; porting an existing app; CI deploys | What Fabric Apps will and will not host · the `rayfin init/login/up` recipe · headless service-principal deploys · the Next.js → Vite port pattern · deploy gotchas |
| [`fabric-app-lakehouse-live`](./skills/fabric-app-lakehouse-live/) | a SPA must read EXISTING Lakehouse data with no backend; debugging 403s and MSAL blank pages | CORS-first ordering · MSAL singleton done right · the one permission that matters (`GraphQLApi.Execute.All`) · on-screen JWT debugging · the silent `first: N` row cap |
| [`fabric-app-sqldb-writeback`](./skills/fabric-app-sqldb-writeback/) | users write data back (comments, edits, approvals); "my saved row takes forever to appear" | The operational-data-store architecture · entity modeling rules · the TDS/pyodbc service-principal recipe (both sides) · wire and SDK gotchas · read-your-own-writes verification |

The three are companions: `bootstrap` ships the app, then the decision rule picks the data layer — **read-only → `lakehouse-live`**, **write-back → `sqldb-writeback`**.

## Install

**Claude Code, per project** (recommended): copy the skill folders into your project's `.claude/skills/`:

```bash
git clone https://github.com/gusbavia/fabric-apps-skills.git
cp -r fabric-apps-skills/skills/* your-project/.claude/skills/
```

**Claude Code, global**: copy them into `~/.claude/skills/` instead, and they are available in every project.

Then just describe what you want ("deploy this Vite app to my Fabric workspace, headless") — the agent discovers the right skill from the task, or invoke explicitly with `@fabric-app-bootstrap`.

## What you bring

- A Microsoft Fabric workspace with the **Fabric App items (preview)** tenant setting enabled
- For the data skills: a Lakehouse with a GraphQL API over it, and an Entra app registration or service principal as each skill specifies
- Your own ids and secrets via environment variables. The skills never embed credentials and will tell the agent not to either

## Status and honesty

Fabric Apps is **preview technology**. The CLI surface and platform behavior move; these skills describe what was true and verified at the version noted in the changelog. If something drifts, issues and PRs are welcome — a reproduced gotcha with the fix is the most valuable contribution this repo can receive.

## Provenance

Written by [Gus Bavia](https://www.linkedin.com/in/gusbavia) (GUSBAVIA, Sydney) while porting a complete Next.js team dashboard into a Fabric-native app with its own transactional database. The full build journey is published as a case study on the GUSBAVIA site and LinkedIn.

Companion repo (coming): **fabric-apps-starter** — two runnable templates implementing these recipes end to end.

## License

MIT. Use it, fork it, ship it.
