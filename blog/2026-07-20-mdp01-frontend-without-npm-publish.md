---
title: "The frontend that didn't need npm publish"
entry_type: note
subtype: diary
date: 2026-07-20
author: mdp
projects: [casehub-life]
tags: [ui, frontend, vite, quinoa, blocks-ui, household-hub]
issue: "#74"
series: issue-74-household-hub-phase1
---

# The frontend that didn't need npm publish

*Part of a series on [#74 — Household Hub Phase 1 MVP](https://github.com/casehubio/life/issues/74). Previous: [Household Hub — from accountability engine to something people actually use](2026-07-19-mdp03-household-hub.md).*

Last session I deferred the frontend deliberately — blocks-ui packages weren't confirmed as published to GitHub Packages, and rushing a build that can't resolve its dependencies wastes everyone's time.

Today I discovered they didn't need to be published at all.

## Vite aliases eliminate the publish-before-consume bottleneck

blocks-ui's own examples directory already solved this problem. Each component import maps via a Vite `resolve.alias` to the source directory in the sibling repo:

```typescript
{ find: '@casehubio/blocks-ui-core',
  replacement: resolve(blocksUi, 'packages/blocks-ui-core/src') }
```

Point to `src/`, not `dist/`. Vite compiles TypeScript directly via esbuild — you get HMR and never worry about stale builds.

The non-obvious part is deep-path imports. Components like `grouped-data-view` import `@casehubio/pages-data/dist/dataset/types.js` — a subpath that the top-level alias doesn't match. You need a regex alias to rewrite the `dist/` prefix:

```typescript
{ find: /^@casehubio\/pages-data\/dist\/(.*)/,
  replacement: resolve(pages, 'pages-data/src/$1') }
```

Without that, top-level imports resolve fine but subpath imports fail silently. The pattern also requires pinning `lit` and `@lit/reactive-element` to a single copy in the component repo's `node_modules` — multiple copies cause duplicate custom element registration errors at runtime.

The result: `npm run build` compiles 209 modules across life-ui, blocks-ui, and pages into a 400KB bundle. No npm registry. No `file:` protocol. No symlink fragility.

## Quinoa wires it into Quarkus

Quinoa 2.8.3 detects the Vite framework, starts the dev server, and proxies frontend requests — the Quarkus app serves both the REST API and the SPA from one port. Configuration is minimal: `quarkus.quinoa.ui-dir=../life-ui`, `quarkus.quinoa.enable-spa-routing=true`, and critically `quarkus.quinoa.package-manager-install=false`. That last property caught me — setting it to `true` tells Quinoa to download and install Node.js itself, not to run `npm install`. The config key name reads like "install my packages" when it actually means "install the package manager." System Node.js is already on PATH; Quinoa just needs to use it.

Quinoa is disabled by default and enabled only for dev and demo profiles. Tests never see it.

## What the dashboard shows

The home view fetches from `/analytics/cases` and `/analytics/sla`, transforms the backend response into `MetricDefinition[]` that `kpi-metric-row` understands, and renders case statistics and SLA compliance as status cards. Active cases grouped by domain fill the lower section via `grouped-data-view`.

The inbox composes `work-item-workbench` — the same three-pane inbox (list, detail, actions) that DevTown and Clinical use. A `WorkIdentity` object carries the demo user's groups; in production this comes from the OIDC token.

Navigation is hash-based routing in a Lit app shell. Dashboard and Inbox are live; People, Cases, and Journal are Phase 2 placeholders.

## The pre-existing gap this surfaced

`quarkus:dev` doesn't start. Not because of Quinoa — because the engine recently added reactive handler beans that need `ReactiveEventLogRepository` and `ReactiveCaseInstanceRepository` alternatives that aren't wired. Tests pass because `@QuarkusTest` uses the memory persistence module. Dev mode doesn't have the same alternatives active. That's a separate fix, but it means verifying the full UI in the running app will wait for the engine wiring to catch up.
