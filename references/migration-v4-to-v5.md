# Migrating Indx v4 → v5

Use this when a user is on **Indx v4** (IndxSearchLib 4.x / Indx v1) and wants to move to **v5** (IndxSearchLib 5.x / Indx v2), or when their existing v4 code is being mistaken for v5.

v5 is the recommended version. The single biggest change is on the **HTTP API**: datasets now belong to **teams**, and every dataset endpoint is **team-scoped**. Flat v4 routes no longer exist on a v5 server.

---

## 1. Determine the current version

Don't assume. Check:

- **C# / NuGet**: the `IndxSearchLib` package version (`4.x` = v4, `5.x` = v5) and the project's target framework (v5 requires **.NET 10**).
- **HTTP API**: the shape of the URLs the user calls.
  - Flat, e.g. `POST /api/Search/{dataset}` → **v4**.
  - Team-scoped, e.g. `POST /api/teams/{team}/datasets/{dataset}/search` → **v5**.
  - Quick probe: `GET /api/me/datasets` returns `200` (with `teamName`/`role` per entry) on v5; it doesn't exist on v4.
- **Auth**: if they authenticate by POSTing email + password to a login endpoint, that's **v4**. v5 is token-only (a JWT created in the portal).

---

## 2. HTTP API (Indx v1 → v2) — the breaking changes

### 2a. Datasets belong to teams; every endpoint is team-scoped

This is the change that breaks the most code. Every dataset operation moved under a team + dataset prefix, and the operation names became lowercase resource routes with proper HTTP verbs and status codes:

```
v4:  /api/{Operation}/{dataSetName}
v5:  /api/teams/{teamName}/datasets/{dataSetName}/{operation}
```

| Operation | v4 route | v5 route |
|-----------|----------|----------|
| Search | `POST /api/Search/{ds}` | `POST /api/teams/{team}/datasets/{ds}/search` |
| Create/open | `PUT /api/CreateOrOpen/{ds}/{cfg}` | `PUT /api/teams/{team}/datasets/{ds}` (the dataset route itself) — `201` created / `200` existed. There is no configuration to choose: a `?configuration=` value is accepted and ignored, so existing clients need no change |
| Load | `PUT /api/LoadString/{ds}` | `POST /api/teams/{team}/datasets/{ds}/load/text` — `204` (`POST …/load` for a JSON stream) |
| Index | `GET /api/IndexDataSet/{ds}` | `POST /api/teams/{team}/datasets/{ds}/index` — `202` (poll `GET …/status`) |
| Status | `GET /api/GetStatus/{ds}` | `GET /api/teams/{team}/datasets/{ds}/status` |
| Get JSON | `POST /api/GetJson/{ds}` | `POST /api/teams/{team}/datasets/{ds}/documents/lookup` |
| Field config | `PUT /api/SetSearchableFields/{ds}` (etc.) | `PUT /api/teams/{team}/datasets/{ds}/fields/searchable` (etc.) — `204`; prefer `PUT …/fields/configuration` |

Note the status-code semantics on v5: `201` on creation, `202` for the async index build, `204` (no body) for mutations, and count endpoints return `{"count": n}` instead of a naked number.

The simplest migration is to compute a per-dataset base once and append the operation:

```
const BASE = `${HOST}/api/teams/${team}/datasets/${dataset}`;
// then BASE + "/search", BASE + "/status", ...
```

> **Pre-modernization v5 routes are gone.** Early v5 servers exposed PascalCase operation names (`CreateOrOpen`, `GetStatus`, `IndexDataSet`, …) under the same team-scoped prefix, answering `200` for everything. They were kept as hidden aliases for a while; they have since been removed, so they now answer `404` (or `405` where only the verb changed, such as `PUT …/replace`). Use the modern routes shown here.

### 2b. Delete routes changed shape

| | v4 | v5 |
|---|----|----|
| Delete dataset | `DELETE /api/DeleteDataSet/{ds}` | `DELETE /api/teams/{team}/datasets/{ds}` (the dataset root) — `204` |
| Delete document(s) | `DELETE /api/{ds}/{key}` / `DELETE /api/{ds}` | `DELETE …/datasets/{ds}/documents/{key}` / `…/documents` — `204` |

### 2c. Listing datasets

`GET /api/GetUserDatasets` (returned a `string[]` of names) is gone. Use:

- `GET /api/me/datasets` → every dataset the caller can see, each with its `name`, `teamName`, and the caller's `role`.
- `GET /api/teams/{team}/datasets` → datasets owned by one team.

A v4 client that expected a flat `string[]` of names must now read `name`/`teamName` from objects.

### 2d. Auth is token-only

v4 allowed an API login (POST email + password). v5 removes it: create a JWT in the account portal (**/account/api-key**) and send `Authorization: Bearer <token>` on every request. There is no login API call to script. Update any code that logged in programmatically to instead read a pre-issued token from config/secret.

### 2e. Pick the team

Because everything is team-scoped, the client needs a team slug. On registration each user gets a personal team; more are created in the portal. Decide whether the team is global to the deployment (one env var / flag) or per-dataset, then thread it into the base URL (2a).

---

## 3. C# library (IndxSearchLib 4.x → 5.x)

The embedded C# API is less affected than the HTTP API, but two things matter:

- **Target framework**: v5 requires **.NET 10**. Update the project's TFM.
- **API surface**: field configuration and scoring are configured via `Field` / `FieldProxy` (searchable/filterable/facetable/sortable/wordIndexing, per-field `Weight`, per-field `BM25b`/`BM25k1`, and `HighResolution`). Verify any v4 field-setup or scoring calls against the current C# reference — see [csharp.md](csharp.md).

> The exact v4→v5 C# method-level deltas are not enumerated here. Confirm them against the IndxSearchLib 5.x release notes / [csharp.md](csharp.md) before promising a specific code change, rather than guessing. (This section is intentionally conservative — fill it in with authoritative deltas when available.)

---

## 4. Upgrade checklist

**HTTP API consumer:**
1. Confirm the server is running Indx v2 (team-scoped routes respond; `/api/me/datasets` works).
2. Replace flat routes with `…/teams/{team}/datasets/{ds}/{op}` (2a) — including the delete-route shapes (2b).
3. Swap any login call for a portal-issued bearer token (2d).
4. Replace `GetUserDatasets` usage with `me/datasets` / `teams/{team}/datasets` and read the object shape (2c).
5. Decide the team slug and thread it through (2e).
6. Smoke-test: `PUT` the dataset route → `POST analyze` / `POST load` → `POST index` (202) → poll `GET status` → `POST search`.

**C# embedded:**
1. Bump the project to .NET 10 and `IndxSearchLib` 5.x.
2. Rebuild; resolve compile errors against [csharp.md](csharp.md).
3. Re-verify field configuration and scoring against the v5 reference.

---

## 5. If the user must stay on v4 for now

That's fine — just be explicit about it and answer with v4 (flat-route, password-login) semantics, **not** the v5 guidance in the rest of this skill. Note that new tooling (the v5 npm packages, team-scoped clients) targets v5, so plan the upgrade when practical.
