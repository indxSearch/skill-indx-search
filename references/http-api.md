# HTTP API Integration

For non-.NET tech stacks, deploy the Indx server and interact via REST. For setup, deployment, and configuration see [cloudapi-setup.md](cloudapi-setup.md).

**OpenAPI spec**: `https://localhost:5001/swagger/v1/swagger.json`

## Authentication

All endpoints require a **Bearer token** — an API key from the console (Account → API keys, `/account/api-key`). Every key is limited to **one team**, optionally to **some of its datasets**, and to an **access level**:

| Level | Can call | Use it for |
|---|---|---|
| **Search only** | `search`, `search/vector`, `search/hybrid`, `documents/lookup`, `filters/value`, `filters/range`, `filters/combine`, `filters/not`, `boosts/from-filter`, `fields/facetable`, `fields/filterable`, `fields/sortable`, `fields/searchable`, `GET fields/configuration`, `status`, and `PUT` on an existing dataset | Websites and apps — the key is visible in the browser, and this is all it can do |
| **Read only** | Every read: adds `export`, `fields/configuration`, `synonyms`, `boosts`, counts, dataset lists | Exports, reporting, agents, back-ups — keep it on a server |
| **Full access** | Everything your team role allows, including loading, changing and deleting data | Your own servers and pipelines |

A key never exceeds its owner's role in the team: a Viewer can create Search and Read keys, not Full ones. Keys cannot be changed after they are created — create a new one and revoke the old. A key outside its limits gets the same answer as a missing team or dataset (`404 teamNotFound` / `datasetNotFound`); a call above its level gets `403 insufficientKeyScope`. Keys created before access levels existed are unscoped and reach every team you belong to — replace them.

**Personal keys and team keys.** A personal key (Account → API keys) acts as its owner and never exceeds their team role; it stops working if they leave the team. A team key (Manage team → API keys, created by a team Admin) belongs to the team: its rights are its access level alone (Full access writes as an Editor; deleting datasets and managing the team give `403 insufficientRole`), it keeps working when its creator leaves, and every Admin of the team can see and revoke it. **For anything deployed (a production site, a shared server, CI), recommend a team key.** Personal keys suit a developer's own scripts and AI agents.

A Search only key can be shown again from the key list; Read only and Full access keys are shown once, so a lost one means a new key. Keys are listed with their last four characters (`…a1b2`). Keys last 30 to 365 days; a team Search only key may never expire, and the team's Admins are warned 30 and 7 days before a team key expires. Rotate without downtime: create the new key, deploy it, then revoke the old one (revocation is immediate).

**For a browser front-end, always a Search only key** limited to the datasets it searches: the key is readable by every visitor. Send it on every request:

```bash
curl -H "Authorization: Bearer <token>" https://your-host/api/...
```

The integration flow is token-only: you create the token in the portal, not via an API call.

## Teams and dataset scoping

Datasets are owned by **teams**. A user can belong to multiple teams with a role on each. **Every dataset endpoint is scoped to a team and a dataset:**

```
/api/teams/{teamName}/datasets/{dataSetName}/{operation}
```

The tables below list just the `{operation}`. Endpoints that aren't dataset-scoped (listing datasets) show their full path.

| Role | Search | Modify data & fields | Delete / transfer |
|------|:------:|:--------------------:|:-----------------:|
| `Admin` | ✓ | ✓ | ✓ |
| `Editor` | ✓ | ✓ | — |
| `Viewer` | ✓ | — | — |

Team membership and roles are managed in the account portal, not via this API.

## API Endpoints

All endpoints prefixed with `/api/`, JWT Bearer auth required.

Success codes follow standard REST semantics: `201 Created` when a resource is created, `202 Accepted` for the asynchronous index build (poll `GET status`), `204 No Content` (empty body) for mutations, and `200` with a body for reads and searches. Count endpoints return a `{"count": n}` envelope, never a naked number.

## Error responses

Every error is an [RFC 9457 ProblemDetails](https://www.rfc-editor.org/rfc/rfc9457) served as `application/problem+json`, carrying a machine-readable **`code`** extension — branch on `code` (or the status), never on the English `detail` text.

| Status | `code` | Meaning | What to do |
|--------|--------|---------|------------|
| `400` | `invalidArgument` | Missing/invalid body or argument | Fix the request; don't retry as-is |
| `400` | `invalidDatasetName` | The dataset name itself is not allowed | Use letters/digits (no path characters) |
| `400` | `loadFailed` | The JSON payload could not be loaded/analyzed (parse error, key-field violation) | Fix the data; the dataset's previous documents are untouched on `replace` |
| `400` | `operationFailed` | A server-side step failed cleanly (index build, shadow build, wake-up) | Read `detail`; retrying may work after fixing the cause |
| `400` | `unknownFilter` | A referenced filter `hashString` could not be resolved | Re-create the filter and retry with the new token — never retry without the filter |

A `hashString` is **opaque**: obtain it from `filters/value|range|combine|not`, pass it back, and do
not parse or construct it. It is durable — it survives cache eviction and a restart — so it can
be held for the life of a dataset's field configuration. There is no NOT endpoint; exclude a
value by filtering on the complementary values instead.
| `401` | — | Missing/expired/invalid token (body-less) | Refresh the bearer token |
| `401` | `invalidCredentials` / `userNotFound` | Login failed | Fix the credentials |
| `403` | `insufficientRole` | You are a member, but your team role is too low | Get a higher role (Editor/Admin) |
| `403` | `insufficientKeyScope` | The API key's access level is below the operation (Search key calling a write) | Use a key with a higher level — on a server, never in a browser |
| `404` | `teamNotFound` | The team doesn't exist **or you are not a member** (identical on purpose — team names can't be enumerated) | Verify via `GET /api/me/datasets` |
| `404` | `datasetNotFound` | The dataset doesn't exist in this team | Create it, or fix the name |
| `404` | `documentNotFound` | The addressed document key(s) don't exist | Fix the keys; batch deletes name every missing key and apply nothing |
| `404` | `statisticsDisabled` | Statistics are switched off on this instance | Don't retry; the events/statistics routes are unavailable by operator choice |
| `409` | `invalidState` | **Wrong lifecycle state** — the dataset can't serve this operation right now | See below |
| `409` | `shadowBusy` | A background rebuild (replace / re-index / field-config) is already running | Retry after it completes |
| `429` | `rateLimited` | Too many attempts from this address on an anonymous auth endpoint; this API key exceeded its requests-per-second budget (where the operator enabled it); or too many calls from this address carried no bearer token that validates (missing, expired or invalid) | Wait `Retry-After` seconds (also `retryAfterSeconds` in the body); do not retry sooner. For the per-key limit, spread calls out or batch them (`documents` endpoints take arrays). For the tokenless limit, authenticate — every one of those calls was going to 401 |
| `500` | `internalError` | Unexpected server error; `traceId` included | Report the `traceId` |

A `409 invalidState` is returned when an operation is valid but the dataset's `systemState` can't serve it (e.g. `POST search` before the dataset is `Ready`, `POST wakeup` when the dataset is not hibernated):

```json
{
  "status": 409,
  "detail": "Search cannot run on dataset 'products' because it is currently Indexing. Indexing is in progress — retry once the dataset reaches Ready.",
  "code": "invalidState",
  "currentState": "Indexing",
  "allowedStates": ["Ready"],
  "retryable": true
}
```

**Agent guidance:**
- If `retryable` is `true` (states `Loading`/`Indexing`), poll `GET status` until `systemState` is `Ready` — respecting the **`Retry-After`** response header (seconds) — then retry the call.
- If `retryable` is `false` (e.g. `Created`, `Error`), don't spin: take the corrective action in `detail`/`allowedStates` first. `Created` with `recordsOnDisk > 0` is a hibernated dataset: `POST wakeup`. `Created` without → `POST load` then `POST index`. `Error` → read the included error and re-create/re-load.
- A 409 is **never** fixed by resending the same request immediately — change the state, not the payload.

### Datasets

| Method | Path / Operation | Description |
|--------|------------------|-------------|
| GET | `/api/me/datasets` | List every dataset across your teams → `DataSetListDto[]` (each with `teamName` + your `role`) |
| GET | `/api/teams/{teamName}/datasets` | List datasets owned by one team → `string[]` |
| PUT | `/api/teams/{teamName}/datasets/{dataSetName}` | Create or open a dataset — `201` created, `200` already existed |
| GET | `status` | Get dataset status → `CloudSystemStatus` |
| GET | `documents/count` | Get document count → `{"count": n}` |
| DELETE | `/api/teams/{teamName}/datasets/{dataSetName}` | Delete dataset permanently (team Admin) → `204` |

### Data Loading

| Method | Operation | Body | Description |
|--------|-----------|------|-------------|
| POST | `analyze` | JSON body (stream) | Analyze JSON structure, discover fields → `200` (`SystemStatus`) |
| POST | `analyze/text` | JSON as plain text string | Analyze from string → `200` |
| POST | `load` | JSON body (stream) | Load JSON documents (large files) → `204` |
| POST | `load/text` | JSON as plain text string | Load from string → `204` |
| POST | `load/from-database` | — | Reload persisted data into memory → `204` |
| POST | `replace` | JSON body (stream) | **Zero-downtime full reload**: builds a new engine on the side (analyze → carry over field config, key field → load → index), then swaps it in; the old data serves until the swap, and a failed build leaves it untouched. Editor role → `200` with a schema-change summary |

`replace` response:

```json
{
  "added":       ["newField"],        // in the new JSON, not previously configured — loads unconfigured
  "removed":     ["oldField"],        // previously configured, absent now — its config is dropped
  "typeChanged": ["price"],           // type changed — that field's roles were reset
  "lostRoles":   ["price (filterable, sortable)"],  // WARNING: removed/retyped fields that had a role; searches and filters on them now silently miss
  "keyFieldFallback": null            // WARNING when set: the declared key field was absent, so documents were keyed by the engine default ("id" if present, else auto)
}
```

Surface `lostRoles` and `keyFieldFallback` to the user — they are the changes that alter search behaviour. Boost rules referencing a removed field go dormant (not deleted) and re-apply if the field returns.

### Field Configuration

Roles are written by one endpoint; the role lists are read-only.

| Method | Operation | Body | Description |
|--------|-----------|------|-------------|
| PUT | `fields/configuration` | `FieldProxy[]` | Set field roles, weights and BM25 parameters; only the properties sent are changed → `204` applied, or `202` + status when a rebuild was started |
| GET | `fields/configuration` | — | Get current field configuration → `FieldProxy[]` |
| GET | `fields` | — | List all discovered field names → `string[]` |
| GET | `fields/searchable` / `fields/filterable` / `fields/facetable` / `fields/sortable` / `fields/word-indexing` / `fields/embeddable` | — | List field names by role → `string[]` |

`PUT fields/configuration` checks the whole request first; a refused one is `400` and changes nothing. Before the dataset is indexed it always answers `204`. On a `Ready` dataset a change that needs the index rebuilt answers `202`: the rebuild runs in the background on a shadow engine and the old configuration serves until the swap. Poll `GET status` until `shadowBuildInProgress` is `false`, then check `shadowBuildError` (`null` = the new configuration is in use). During a rebuild another configuration change, `POST index` or a bulk document change gets `409 shadowBusy`.

A vector field (`embeddable: true`) must be an array of numbers and can have no other role.

The individual role setters (`PUT fields/searchable` / `fields/filterable` / `fields/facetable` / `fields/sortable` / `fields/word-indexing` / `fields/embeddable`) were removed in October 2026 and answer `405` — use `PUT fields/configuration`.

### Indexing and Search

| Method | Operation | Body | Description |
|--------|-----------|------|-------------|
| POST | `index` | — | Start indexing → `202 Accepted` at once with the status body; the build runs in the background. First build: poll `GET status` until Ready. On a Ready dataset it is a rebuild on a shadow engine: poll until `shadowBuildInProgress` is `false`, then check `shadowBuildError` |
| POST | `search` | `QueryProxy` | Full-text search → `Result` |
| POST | `search/vector` | `VectorQueryProxy` | Embedding nearest-neighbour search → `EmbeddingResultEntry[]` |
| POST | `search/hybrid` | `HybridQueryProxy` | Blended text + vector search → `EmbeddingResultEntry[]` |
| POST | `documents/lookup` | `long[]` (document keys) | Retrieve full JSON records → `string[]` |

Search responses carry an **`Indx-Query-Id`** header (CORS-exposed): an opaque id minted per
search when statistics are enabled. Hold it briefly on the client and echo it in the
select/convert events below so they join the search they belong to. Search also accepts an
optional **`?subject=`** query parameter — see Statistics and Events.

### Dynamic Document Operations

Insert, update, and delete without rebuilding the index. The dataset stays ready throughout.

**Unknown fields are refused.** An insert or update carrying a field that is not in the field configuration returns `400 invalidArgument` naming the field, and stores nothing: one such document fails its whole batch. Add fields with `replace` (see Data Loading). Missing non-key fields are treated as null; a missing key field rejects the whole batch.

| Method | Operation | Body | Description |
|--------|-----------|------|-------------|
| POST | `documents` | `string[]` (JSON objects) | Insert multiple documents → `201` |
| POST | `documents/{documentKey}` | JSON string | Insert single document — the route key must equal the body's key field (mismatch → 400, nothing inserted) → `201` |
| PUT | `documents` | `string[]` (JSON objects) | Update multiple documents (must include key field) — all-or-nothing: one bad record rejects the whole batch → `204` |
| PUT | `documents/{documentKey}` | JSON string | Update single document — route key must exist (404 otherwise) and equal the body's key field (mismatch → 400) → `204` |
| PATCH | `documents/{documentKey}` | `UpdateFieldProxy` | Partial single-field update on one document → `204` |
| DELETE | `documents/{documentKey}` | — | Delete single document (404 if the key doesn't exist) → `204` |
| DELETE | `documents` | `long[]` | Delete multiple documents by key — all-or-nothing: missing keys → 404 naming all of them, nothing deleted → `204` |
| POST | `documents/delete-by-filter` | `FilterProxy` | Delete all documents matching a filter → `204` |
| POST | `documents/update-by-filter` | `FilterFieldUpdateProxy` | Batch update a field on all matching documents → `200` (`{"count": n}` — documents updated) |

### Filters and Boosts

| Method | Operation | Body | Description |
|--------|-----------|------|-------------|
| POST | `filters/value` | `ValueFilterProxy` | Create equality filter on a **non-numeric** field → `FilterProxy`. On a numeric field: `400` naming `filters/range` instead |
| POST | `filters/range` | `RangeFilterProxy` | Create range filter on a **numeric** field → `FilterProxy`. Equal limits = exact match. On a text field: `400` (a number stored as a JSON string is text) |
| POST | `filters/combine` | `CombinedFilterProxy` | Combine with AND/OR → `FilterProxy` |
| POST | `filters/not` | `FilterProxy` | Negate: every document the filter does not match → `FilterProxy` |
| POST | `boosts/from-filter` | `BoostProxy` | Create boost rule → `BoostProxy` |
| POST | `filters/load` | — | Pre-load all registered filters into memory → `204` |
| GET | `filters/count` | — | Count cached filters → `{"count": n}` |
| POST | `filters/delete` | `FilterProxy` | Release one cached filter → `204` (POST, not DELETE, because the filter key travels in the body) |
| DELETE | `filters` | — | Release all cached filters → `204` |

### Synonyms

| Method | Operation | Body | Description |
|--------|-----------|------|-------------|
| GET | `synonyms` | — | The dataset's synonym list → `200` (a literal `null` when it has none) |
| PUT | `synonyms` | `SynonymList` or `null` | Replace the list; `null` removes it; editor role → `204` |

Synonyms expand queries at search time (matching an entry appends its terms to the query text before scoring), so a `PUT` applies on the very next search — no re-indexing, no state change. Body shape:

```json
{
  "name": "medical-terms",
  "entries": [
    { "direction": 0, "terms": ["geriatri", "geriatrisk", "geriatriske"] },
    { "direction": 1, "source": "hms", "terms": ["helse, miljø og sikkerhet"] }
  ]
}
```

`direction`: `0` = Multidirectional (all terms equivalent — any of them triggers the group), `1` = OneWay (only `source` expands, into `terms` — for acronyms). Multi-word terms match as whole phrases. Note that expansion lengthens the query text, which lowers Coverage scores proportionally.

### Statistics and Events

The server counts every HTTP search with text itself (query text, hit count — that is what
makes the zero-hit report exist), and the storefront reports what happened next. An empty
search with a filter counts as browsing (per filter value, `statistics/filters`); without a
filter it is not counted, and neither is the web console's search preview. The `Indx-Query-Id`
header is sent either way. **Send `?session=<per-page-load random id>` on every search-as-you-type
request**: only the search the visitor settled on is then counted, not each keystroke's prefix, and the same
search re-sent within 30 s (facets after a debounce, a new sort, load more) counts once; keep the
`Indx-Query-Id` of the latest response for the click. A request for zero records is not counted. **Name the surface with `?source=<name>`** when a dataset has several (a header search box, a product finder, an app): it is stored with the search and counted as usual. **A search that is not a visitor's** (an agent checking what a query returns, a monitor, a test) **sends `?count=false`**, usually with its own `?source=`: it is stored but left out of every statistic, like the console's preview, so whoever reads the numbers does not read their own searches back. A select or convert on such a search is not counted either; an event can also say `"count": false` itself. Event posts answer
`202 Accepted` (queued, batch-written); an unknown or expired `queryId` is still accepted and
counts on the document — it just finds no search to join. All statistics routes answer
`404 statisticsDisabled` when the operator switched the feature off.

| Method | Operation | Body / Query | Description |
|--------|-----------|--------------|-------------|
| POST | `events/select` | `{queryId?, documentKey, position, subject?, count?}` | The user chose a result; `position` is 1-based in the list. Search key suffices → `202` |
| POST | `events/convert` | `{queryId?, documentKey, type, subject?, count?}` | Something valuable happened — `type` is your own name for it (`"order"`, `"addToCart"`). Counted per document, the signal for personalised boosting; `value`/`currency`/`quantity` are accepted but not reported. `queryId` optional: an order may come long after the search. Search key → `202` |
| GET | `statistics/overview` | `?days=30` | The window's totals with the rates computed server-side: `uncoveredRate` (searches coverage confirmed nothing for: typos too large for an exact match), `uncoveredChosen` (of those, the ones a result was chosen from anyway: fuzzy finds), `clickThroughRate` (searches with ≥1 select over ALL searches), `averageClickPosition` — `null` when the denominator is zero, never 0%. `zeroHitRate` stays for compatibility. Read key |
| GET | `statistics/timeseries` | `?days=30` | One row per UTC day (`date` as `yyyy-MM-dd`), zero rows included so charts have no holes. Read key |
| GET | `statistics/filters` | `?days&limit` | Browsing: uses and zero-hit count per filter value (`field`, `value`; a range has an empty `value`), with or without text. Read key |
| GET | `statistics/changes` | `?days=30` | What the dataset's owners changed, beside the numbers: `changes` (`at`, `kind`: synonyms, boostRules, fields, reindex, replace, documents, hibernate, wake, rename, delete; `summary` of counts and field names, never content or who) and `documents` (inserted/updated/deleted per day). A single request changing ≥1 000 documents (or a tenth of the dataset) and every by-filter operation is a `documents` change of its own. Read key |
| GET / PUT | `statistics/settings` | `{recordFilters}` | Whether each search's filter is stored (Browsing). On by default; switch off on datasets of people, where a filter value can be a name. Off stops the recording; values already recorded stay until they age out or the statistics are erased. PUT needs a Full key and team admin |
| GET | `statistics/queries` | `?days&limit&uncoveredOnly&order` | Top queries with searches, selects, clicked-search count and position sum. `uncoveredOnly=true` lists the misspelled searches (coverage confirmed nothing) with `uncoveredChosen` (a result chosen anyway: fuzzy search found it) and `mostChosenDocument` (the result chosen most). `order`: `searches` (default), `lowestClickThrough` (searched often, chosen from rarely: missing what people want, or ranking it too low), `highestClickThrough`, `chosenAnyway` (with `uncoveredOnly`); the two click-through orders leave out queries searched fewer than five times. Read key |
| GET | `statistics/queries/documents` | `?text&days&limit` | For one query (`text`, lowercased like the query list): every document chosen from its results with `selects`, `positionSum` and the `converts` from the same searches, most chosen first. Tells whether a rarely chosen query found the thing itself or a substitute. From the raw rows, so up to 90 days back. `text` missing → `400`. Read key |
| GET | `statistics/documents` | `?days&limit` | Top documents: selects and conversions. Read key |
| GET | `statistics/subjects/{subject}` | `?limit=10` | One subject's lifetime top documents — the personalization read. Read key |
| DELETE | `statistics` | — | Purge the dataset's statistics (Full key + team Admin) → `204` |
| DELETE | `statistics/subjects/{subject}` | — | GDPR erasure of one subject: its edge rows are deleted and its raw rows anonymised, so aggregates stay true (Full key + team Admin) → `204` |

**`subject`** is one optional opaque string you define: an end-user id, a session id, or a
customer segment. Send it on search (`?subject=`) and on the events; omitted means anonymous
(never invent a placeholder like "noname" — it would pollute every distinct count). It powers
the per-subject top-N and is what the GDPR erasure targets.

**Statistics survive dataset deletion and recreation** — they are keyed on the team and the
dataset *name*, matching how document keys are yours. Deleting the dataset does not touch them;
`DELETE statistics` is the one explicit way to start clean. Raw events are kept 90 days
(configurable) and rolled up into daily aggregates that are kept until deleted.

### Lifecycle

| Method | Operation | Description |
|--------|-----------|-------------|
| POST | `hibernate` | Free in-memory structures while retaining persisted data → `204` |
| POST | `wakeup` | Restore a hibernated dataset from persisted state → `204` |

## Schemas

### FieldProxy (Field Configuration)

```json
[
  { "fieldName": "title",       "searchable": true,  "weight": 2.0 },
  { "fieldName": "description", "searchable": true,  "weight": 1.0 },
  { "fieldName": "category",    "filterable": true,  "facetable": true },
  { "fieldName": "price",       "filterable": true,  "sortable": true },
  { "fieldName": "rating",      "sortable": true }
]
```

All fields are optional (null = unchanged). Available properties:
`fieldName`, `fieldType`, `isArray`, `searchable`, `filterable`, `facetable`, `sortable`, `wordIndexing`, `embeddable`, `weight` (float), `bM25b` (float, 0–1), `bM25k1` (float, 1–2), `preloadFilters`, `highResolution` (also index/query N-grams with delimiters removed, so a run-together or split query matches across them).

`fieldType` and `isArray` are read-only — the server fills them in on `GET fields/configuration` and ignores them on `PUT fields/configuration`.

### QueryProxy (Search Request)

Minimal:
```json
{ "text": "search terms", "maxNumberOfRecordsToReturn": 30 }
```

Full (with defaults shown):
```json
{
  "text": "search terms",
  "maxNumberOfRecordsToReturn": 30,
  "enableCoverage": true,
  "coverageDepth": 500,
  "enableFacets": false,
  "enableBoost": false,
  "removeDuplicates": true,
  "sortBy": null,
  "sortAscending": false,
  "timeOutLimitMilliseconds": 1000,
  "filter": null,
  "boosts": null,
  "coverageSetup": null,
  "fieldBoosts": {}
}
```

- `coverageDepth` (default **500**) — how many pattern-match candidates coverage confirms. Hits past
  the depth are never evaluated, so a broad query reports a total landing exactly on the depth: **a
  count equal to `coverageDepth` is the cap, not the data.** Raise it to the document count on small
  and mid-sized datasets (cost is linear in depth), and use one depth for the list and its facet counts.
- `enableFacets` — also required for an **empty** `text`. Empty text with `enableFacets: false` is
  refused: `200`, no records, `reason` set. The dataset also needs one facetable field.

`sortBy` — field name as a **string** (e.g. `"price"`).

`fieldBoosts` — per-query multipliers on each field's configured `weight` (BM25F mode only):
```json
{ "fieldBoosts": { "title": 2.0, "description": 0.5 } }
```
Every key must be a **searchable** field. A key that is not (unknown, or made non-searchable since) empties the whole search: `200`, no records, and a `reason` naming the field. A boost multiplies the weight — `weight: 3.0` × boost `2.0` = 6×; omitted fields keep their weight. Ignored when `bM25k1` differs between searchable fields (per-field scoring).

Full with CoverageSetup (defaults shown):
```json
{
  "text": "search terms",
  "maxNumberOfRecordsToReturn": 30,
  "coverageSetup": {
    "coverWholeQuery": true,
    "coverWholeWords": true,
    "coverFuzzyWords": true,
    "coverJoinedWords": true,
    "coverPrefixSuffix": true,
    "includePatternMatches": true,
    "truncate": true,
    "truncationScore": 65024,
    "minWordSize": 2,
    "levenshteinMaxWordSize": 20,
    "truncateWordHitLimit": 1,
    "truncateWordHitTolerance": 0
  }
}
```

### Result (Search Response)

```json
{
  "records": [
    { "documentKey": 42, "score": 52000 },
    { "documentKey": 17, "score": 31000 }
  ],
  "facets": {
    "category": [
      { "key": "electronics", "value": 12 },
      { "key": "accessories", "value": 5 }
    ]
  },
  "truncationIndex": 15,
  "truncationScore": 52000,
  "didTimeOut": false
}
```

- `records` — `[{ documentKey, score }]`. Score is 0–65535 (ushort); coverage hits always score higher than pattern-only matches.
- `facets` — only populated when `enableFacets: true`.
- `reason` — set when the engine **refused** the query instead of finding nothing: empty `text`
  without `enableFacets`, text over the 300-character limit, a dataset that is not ready. Records are
  empty either way, so check it alongside `didTimeOut` and never render a refusal as "no results".
- `truncationIndex` — where coverage truncation occurred (–1 if none).
- `didTimeOut` — the search could not be *served*, as opposed to serving and finding nothing: either it
  hit the timeout limit, or the engine was not `Ready`. Over HTTP the not-Ready case normally surfaces
  as `409` with the current state before it gets this far, so on this surface the flag almost always
  means the timeout. Either way, do not render an empty result with this flag set as "no results".

Use `POST .../documents/lookup` with the array of `documentKey` values to retrieve full JSON documents.

### Filter and Boost Models

```json
// ValueFilterProxy — equality match on a field that does NOT hold numbers.
// "value" may be a string, number or boolean. "isCaseSensitive" is optional and off by
// default; a facet count equals a filter count only for a case-sensitive filter, and a
// case-sensitive one is answered by walking the documents rather than by the index.
{ "fieldName": "category", "value": "electronics", "isCaseSensitive": false }

// RangeFilterProxy — inclusive range on a field that DOES hold numbers.
// Equal limits are numeric equality: that is how to match one value on a numeric field.
{ "fieldName": "price", "lowerLimit": 10.0, "upperLimit": 100.0 }

// FilterProxy — handle returned by filter creation endpoints
{ "hashString": "<filter-hash>" }

// CombinedFilterProxy — AND/OR two existing filters
{
  "a": { "hashString": "<filter-a-hash>" },
  "b": { "hashString": "<filter-b-hash>" },
  "useAndOperation": true
}

// BoostProxy — boost results matching a filter
// boostStrength: 1 (Low), 2 (Med), 3 (High)
{ "boostStrength": 2, "filterProxy": { "hashString": "<filter-hash>" } }

// UpdateFieldProxy — partial field update
{ "fieldName": "price", "value": 49.99 }
```

### Vector / Hybrid Models

```json
// VectorQueryProxy — embedding nearest-neighbour search
{ "fieldName": "embedding", "vector": [0.12, -0.04, ...], "maxResults": 10, "filter": null }

// HybridQueryProxy — blended text + vector (alpha: 0 = all text, 1 = all vector)
{ "text": "wireless headphones", "embeddingField": "embedding", "vector": [0.12, ...], "alpha": 0.5, "maxNumberOfRecordsToReturn": 10 }

// EmbeddingResultEntry — returned by search/vector and search/hybrid
{ "documentKey": 42, "score": 0.87 }
```

### SystemStatus

```json
{
  "systemState": 4,
  "documentCount": 5000,
  "searchCounter": 42,
  "secondsToIndex": 2,
  "version": "5.0.0",
  "errorMessage": null,
  "invalidDataSetName": false,
  "invalidState": false
}
```

`systemState`: `0`=Created, `1`=Loading, `2`=Loaded, `3`=Indexing, `4`=Ready, `255`=Error. There is no hibernated state: a hibernated dataset reports `0`=Created with `recordsOnDisk > 0` (the old `-1`=Hibernated is gone).

`GET status` returns `CloudSystemStatus`, which adds cloud-layer fields — most usefully `shadowBuildInProgress` (true while a background rebuild runs), `shadowBuildFinishedUtc` (when the last one ended) and `shadowBuildError` (why it failed, or `null`).

## HTTP API Workflow

All steps below are under `/api/teams/{team}/datasets/{dataset}/`.

```
1. Create a token on the Indx Dashboard (Account → API Key)
2. PUT   (the dataset route itself) → create dataset (201 created / 200 existed)
3. POST  analyze                   → discover fields
4. PUT   fields/configuration      → configure all fields in one call (204)
5. POST  load                      → load JSON data (204)
6. POST  index                     → start indexing (202, SystemStatus body)
7. GET   status                    → poll until systemState = 4 (Ready)
8. POST  search                    → returns records with documentKey + score
9. POST  documents/lookup          → fetch full JSON by document keys
```

## Common HTTP Patterns

**Agent-optimized search (exact and near-exact only):**
```json
{
  "text": "exact product name XYZ-123",
  "maxNumberOfRecordsToReturn": 10,
  "coverageSetup": { "includePatternMatches": false }
}
```

**Human-facing search with facets:**
```json
{
  "text": "headphones",
  "maxNumberOfRecordsToReturn": 50,
  "enableFacets": true
}
```

**Empty search — browse mode with sorting:**
```json
{
  "text": "",
  "maxNumberOfRecordsToReturn": 50,
  "sortBy": "rating",
  "sortAscending": false,
  "enableFacets": true
}
```

In the curl examples below, `BASE` is `https://your-host/api/teams/<team>/datasets/products`.

**Filtered search:**
```bash
# 1. Create filters
curl -X POST "$BASE/filters/value" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"fieldName":"category","value":"electronics"}'
# → {"hashString":"abc123..."}

curl -X POST "$BASE/filters/range" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"fieldName":"price","lowerLimit":10,"upperLimit":100}'
# → {"hashString":"def456..."}

# 2. Combine (AND)
curl -X POST "$BASE/filters/combine" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"a":{"hashString":"abc123"},"b":{"hashString":"def456"},"useAndOperation":true}'
# → {"hashString":"combined789..."}

# 3. Search with filter
curl -X POST "$BASE/search" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"text":"wireless","maxNumberOfRecordsToReturn":20,"filter":{"hashString":"combined789..."}}'
```

**Boosted search:**
```bash
curl -X POST "$BASE/boosts/from-filter" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"boostStrength":3,"filterProxy":{"hashString":"<filter-hash>"}}'

curl -X POST "$BASE/search" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"text":"headphones","maxNumberOfRecordsToReturn":20,"enableBoost":true,"boosts":[{"boostStrength":3,"filterProxy":{"hashString":"<filter-hash>"}}]}'
```

**Dynamic insert:**
```bash
curl -X POST "$BASE/documents" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '["{\"id\":999,\"name\":\"New Product\",\"price\":29.99}"]'
# → 201 Created
```

**Partial field update:**
```bash
curl -X PATCH "$BASE/documents/42" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"fieldName":"price","value":24.99}'
# → 204 No Content
```

**Retrieve full documents:**
```bash
curl -X POST "$BASE/documents/lookup" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '[42, 17]'
# → ["{ \"id\": 42, ... }", "{ \"id\": 17, ... }"]
```

## Data Loading Reference (TypeScript / Node.js)

The token comes from the portal (Account → API Key); put it and your team in env vars.

```bash
pnpm add @indxsearch/indx-types axios
```

```typescript
import axios from 'axios';
import * as fs from 'fs';

const HOST = 'https://localhost:5001';
const TOKEN = process.env.INDX_TOKEN!;   // created in the portal (Account → API Key)
const TEAM = process.env.INDX_TEAM!;     // team that owns the dataset
const dataset = 'products';

const client = axios.create({
  baseURL: `${HOST}/api/teams/${TEAM}/datasets/${dataset}`,
  headers: { Authorization: `Bearer ${TOKEN}` }
});

// 1. Create dataset — PUT the dataset route itself (201 created / 200 existed)
await client.put('', '');

// 2. Analyze
const jsonData = fs.readFileSync('products.json', 'utf-8');
await client.post('analyze', jsonData, {
  headers: { 'Content-Type': 'application/json' }
});

// 3. Configure fields — single call replaces the per-role setters (204)
await client.put('fields/configuration', [
  { fieldName: 'name',        searchable: true, weight: 2.0 },
  { fieldName: 'description', searchable: true, weight: 1.0 },
  { fieldName: 'category',    filterable: true, facetable: true },
  { fieldName: 'price',       filterable: true, sortable: true },
]);

// 4. Load (204)
const fileStream = fs.createReadStream('products.json');
const fileStats = fs.statSync('products.json');
await client.post('load', fileStream, {
  headers: { 'Content-Type': 'application/json', 'Content-Length': fileStats.size },
  maxBodyLength: Infinity
});

// 5. Index (202 Accepted) and poll status until Ready
await client.post('index');
let ready = false;
while (!ready) {
  await new Promise(r => setTimeout(r, 200));
  const { data } = await client.get('status');
  ready = data.systemState === 4;
}

// 6. Search
const { data: result } = await client.post('search', {
  text: 'wireless headphones',
  maxNumberOfRecordsToReturn: 10
});

// 7. Fetch full documents
const keys = result.records.map((r: any) => r.documentKey);
const { data: docs } = await client.post('documents/lookup', keys);
```
