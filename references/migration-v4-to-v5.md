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

Diffed from the public surface of `IndxSearchLib` 4.1.2 against 5.0.0. Everything below is a
compile-time break unless marked otherwise, so a project that builds after these changes has
dealt with the surface. Behaviour changes that do not break the build are called out separately,
because they are the ones that pass review and surprise someone later.

### 3a. Target framework and logging

- **`net9.0` to `net10.0`.** Update the TFM.
- **`ILoggerFactory` is now the standard one.** v4 shipped its own `Indx.Utilities.ILoggerFactory`,
  which is gone. The constructor takes `Microsoft.Extensions.Logging.ILoggerFactory`. Add the
  `Microsoft.Extensions.Logging.Abstractions` reference and drop any adapter written for the old
  interface.

### 3b. Field weight is a number, not three levels

v4 had `Field.Weight` as an enum with `Low`, `Med` and `High`, plus a separate
`Field.WeightAsFloat` for finer control. v5 has one property:

```csharp
// v4
field.Weight = Weight.High;
field.WeightAsFloat = 2.5f;

// v5
field.Weight = 2.5f;          // float, default 1.0
```

The `Weight` enum and `WeightAsFloat` are both removed. Pick the number the old level meant to you
rather than looking for a mapping table: there is no official one, because the enum was never
backed by fixed constants a caller could rely on.

### 3c. Scores are 16-bit

This is the change most likely to pass the compiler in spirit and be wrong in practice.

- `Result.Records` is `ScoreEntry16[]`, not `ScoreEntry[]`. The `ScoreEntry` struct is gone.
- `ScoreEntry16.Score` is a `ushort` (0 to 65535). v4's `ScoreEntry.Score` was a `byte` (0 to 255).
- `Result.TruncationScore` is a `ushort` for the same reason.

Any threshold, percentage or bucket computed against 255 is now wrong by a factor of 257. Search
the codebase for score arithmetic, not just for the type name: `score / 255.0`, `score > 200`, a
cast to `byte`, a progress bar scaled to 255.

### 3d. Filter creation reports the reason instead of throwing

```csharp
// v4
Filter f = engine.CreateValueFilter("brand", "acme", isCaseSensitive: false);

// v5
if (engine.CreateValueFilter("brand", "acme", isCaseSensitive: false, out string error) is { } f)
    query.Filter = f;
else
    // error says why: unknown field, not Filterable, unparseable bound or culture
```

The overloads without the `out string error` parameter are removed, on `SearchEngine` and on
`ISearchEngine`, for both `CreateValueFilter` and `CreateRangeFilter`. A rejected filter returns
null and fills `error`. **Null means rejected, never "no filter":** assigning it onward widens the
search to the whole dataset.

### 3e. Configuration is an object, not a number

```csharp
// v4
var engine = new SearchEngine(logPrefix, loggerFactory, 400, "indx-developer.license");

// v5
var engine = new SearchEngine(logPrefix, loggerFactory, ConfigurationParameters.Default,
                              "indx-developer.license");
```

`ConfigurationParameters.Default` is the production configuration and is what config 400 was.
Vary it with `ConfigurationParameters.Default.With(...)` rather than hand-copying; three
`IndexerSetup` presets ship (`Dense`, `Default`, `SingleWord`). The numbered profiles and the
lookup that resolved them are gone, so there is no number to pass and none to get wrong.

`new SearchEngine(licenseFileName)` is unchanged and still the shortest way in.

- `Field.ConfigNumber` is removed.
- The `Index(monitor, taskStartDelayMs, batchDelayMs, batchSize)` tuning overload is removed.
  `Index(monitor)` remains.

### 3f. An empty result now says why

```csharp
// v4
Result.MakeEmptyResult(timedOut);

// v5
Result.MakeEmptyResult(timedOut, reason, systemState);
```

`Result` gains `Reason` (a `ProcessError`) and `SystemState`, so a caller can tell a timeout from a
refused query from an engine that is not ready. Worth reading even if you never construct one:
an empty result in v4 was silent about its cause.

### 3g. Behaviour changes that do not break the build

- **Segmentation is gone.** `AutoSegmentationSetup` is removed, and so is the behaviour: v4 split a
  document longer than `MaxIndexTextLength` into overlapping segments sharing one key, so the tail
  was still searchable. v5 cuts at the ceiling instead and reports it as
  `SystemStatus.IndexedTextTruncated`. BM25F's per-field length normalisation is what replaced it.
  If you index long documents, this changes which of them match.
- **Query text over `MaxSearchTextLength` (default 300) is refused**, not truncated. The result is
  empty with a `Reason` of `TooLongSearchText`. In v4 the query and index ceilings were one number.
- **Out-of-range field values throw.** `Field.Weight` and `Field.BM25k1` reject a negative,
  `Field.BM25b` anything outside [0, 1], with `ArgumentOutOfRangeException` (since 5.0.0-RC170926).
  v4 accepted them and produced quietly wrong rankings. If you set these from configuration or user
  input, validate before assigning.
- **Utility types left the public surface**: `SpanAlloc`, `GaussianGenerator`, `Mean`, `Uniform`,
  `UniformDiscrete`, `OrderPreservingHashSet<T>`, `RandomEnum<T>`, `ByteAsFloat`,
  `FileNameValidity`, `VirtualMachineArithmetics`, `ConnectionStringHelper`. They were never part
  of the search API, but a project that reached for one will not compile.

### 3h. What v5 adds

Not required for the upgrade, but this is usually why someone is doing it:

- **Dynamic indexing.** `InsertJsonRecord(s)`, `UpdateJsonRecord(s)`, `DeleteJsonRecords`,
  `UpdateField`, `UpdateFieldInFilter`. No `Index()` needed after a mutation.
- **BM25F multi-field scoring**, with per-field `Field.BM25b` / `Field.BM25k1`, per-query
  `Query.FieldBoosts`, and `SearchEngine.ScoringMode` reporting which path resolved.
- **Synonyms.** `SynonymList`, `SynonymEntry`, `LoadSynonyms` / `SaveSynonyms`.
- **Embeddings and vector search.** `Indx.Embeddings`, `Field.Embeddable`,
  `Field.EmbeddingDimensions`, `SearchEngine.EmbeddingFields`.
- **Field configuration as data.** `GetFieldConfiguration()` / `SetFieldConfiguration(FieldProxy[])`,
  plus `DocumentFields.RequiresReindex(proposed)` to tell a flag change from a rebuild.
- **`CreateInMemoryClone`**, the basis for building a new index beside a live one and swapping.
- **`Field.HighResolution`**, and `Field.SampleValue` for showing what a field actually holds.
- **HTTP proxy types** under `Indx.Http` (`QueryProxy`, `FilterProxy`, and the rest) for talking to
  an Indx server. The old `Indx.CloudApi` names still compile as obsolete aliases.

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
2. Swap `Indx.Utilities.ILoggerFactory` for `Microsoft.Extensions.Logging.ILoggerFactory` (3a).
3. Replace `Weight.High` / `WeightAsFloat` with a float `Field.Weight` (3b).
4. Fix every score computation for `ushort` instead of `byte` (3c). This one compiles either way
   wherever a score is assigned to a wider type, so grep for the arithmetic.
5. Add the `out string error` argument to filter creation, and handle null as rejected (3d).
6. Replace the configuration number with `ConfigurationParameters.Default` (3e).
7. Re-check long-document behaviour if you relied on segmentation, and query length limits (3g).
8. Rebuild; resolve what is left against [csharp.md](csharp.md).

---

## 5. If the user must stay on v4 for now

That's fine — just be explicit about it and answer with v4 (flat-route, password-login) semantics, **not** the v5 guidance in the rest of this skill. Note that new tooling (the v5 npm packages, team-scoped clients) targets v5, so plan the upgrade when practical.
