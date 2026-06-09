# HTTP API Integration

For non-.NET tech stacks, deploy the IndxCloudApi server and interact via REST. For setup, deployment, and configuration see [cloudapi-setup.md](cloudapi-setup.md).

**OpenAPI spec**: `https://localhost:5001/swagger/v1/swagger.json`

## Authentication

All endpoints require a **JWT Bearer token**. Create one on the IndxCloudApi website — the **API Key** page in your account portal (`/Account/ApiKey`) — then send it on every request:

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

### Datasets

| Method | Path / Operation | Description |
|--------|------------------|-------------|
| GET | `/api/me/datasets` | List every dataset across your teams → `DataSetListDto[]` (each with `teamName` + your `role`) |
| GET | `/api/teams/{teamName}/datasets` | List datasets owned by one team → `string[]` |
| PUT | `CreateOrOpen` | Create or open a dataset (default config) |
| PUT | `CreateOrOpen/{configuration}` | Create with explicit config (int) |
| GET | `GetStatus` | Get dataset status → `CloudSystemStatus` |
| GET | `GetNumberOfJsonRecordsInDb` | Get document count → `int` |
| DELETE | `/api/teams/{teamName}/datasets/{dataSetName}` | Delete dataset permanently (team Admin) |

### Data Loading

| Method | Operation | Body | Description |
|--------|-----------|------|-------------|
| POST | `AnalyzeStreamAsync` | JSON body | Analyze JSON structure, discover fields |
| POST | `AnalyzeString` | JSON as plain text string | Analyze from string |
| PUT | `LoadString` | JSON as plain text string | Load JSON documents |
| PUT | `LoadStream` | JSON body | Load via stream (large files) |
| GET | `LoadFromDatabase` | — | Reload persisted data into memory |

### Field Configuration

**Recommended — single unified endpoint:**

| Method | Operation | Body | Description |
|--------|-----------|------|-------------|
| PUT | `SetFieldConfiguration` | `FieldProxy[]` | Set all field roles and weights in one call |
| GET | `GetFieldConfiguration` | — | Get current field configuration → `FieldProxy[]` |
| PUT | `SetEmbeddableFields` | `string[]` | Mark fields embeddable (call after Analyze, before Load) |
| GET | `GetallFields` | — | List all discovered field names → `string[]` |
| GET | `GetSearchableFields` / `GetFilterableFields` / `GetFacetableFields` / `GetSortableFields` / `GetWordIndexingFields` | — | List field names by role → `string[]` |

The individual `SetSearchableFields` / `SetFilterableFields` / `SetFacetableFields` / `SetSortableFields` / `SetWordIndexingFields` helpers still exist but are legacy — prefer `SetFieldConfiguration`.

### Indexing and Search

| Method | Operation | Body | Description |
|--------|-----------|------|-------------|
| GET | `IndexDataSet` | — | Trigger indexing → `SystemStatus` |
| POST | `Search` | `CloudQuery` | Full-text search → `Result` |
| POST | `VectorSearch` | `VectorQueryProxy` | Embedding nearest-neighbour search → `EmbeddingResultEntry[]` |
| POST | `HybridSearch` | `HybridQueryProxy` | Blended text + vector search → `EmbeddingResultEntry[]` |
| POST | `GetJson` | `long[]` (document keys) | Retrieve full JSON records → `string[]` |

### Dynamic Document Operations

Insert, update, and delete without rebuilding the index. The dataset stays ready throughout.

| Method | Operation | Body | Description |
|--------|-----------|------|-------------|
| POST | `insert` | `string[]` (JSON objects) | Insert multiple documents |
| POST | `insert/{documentKey}` | JSON string | Insert single document |
| PUT | `update` | `string[]` (JSON objects) | Update multiple documents (must include key field) |
| PUT | `update/{documentKey}` | JSON string | Update single document |
| PUT | `field/{documentKey}` | `UpdateFieldProxy` | Partial field update on one document |
| DELETE | `documents/{documentKey}` | — | Delete single document |
| DELETE | `documents` | `long[]` | Delete multiple documents by key |
| DELETE | `DeleteRecordsInFilter` | `FilterProxy` | Delete all documents matching a filter |
| PUT | `UpdateFieldInFilter` | `FilterFieldUpdateProxy` | Batch update a field on all matching documents |

### Filters and Boosts

| Method | Operation | Body | Description |
|--------|-----------|------|-------------|
| PUT | `CreateValueFilter` | `ValueFilterProxy` | Create equality filter → `FilterProxy` |
| PUT | `CreateRangeFilter` | `RangeFilterProxy` | Create numeric range filter → `FilterProxy` |
| PUT | `CombineFilters` | `CombinedFilterProxy` | Combine with AND/OR → `FilterProxy` |
| PUT | `CreateBoost` | `BoostProxy` | Create boost rule → `BoostProxy` |
| POST | `LoadAllFilters` | — | Pre-load all registered filters into memory |
| GET | `GetNumberOfFilters` | — | Count cached filters → `int` |
| DELETE | `DeleteFilter` | `FilterProxy` | Release one cached filter |
| DELETE | `DeleteAllFilters` | — | Release all cached filters |

### Lifecycle

| Method | Operation | Description |
|--------|-----------|-------------|
| PUT | `Hibernate` | Free in-memory structures while retaining persisted data |
| PUT | `WakeUp` | Restore a hibernated dataset from persisted state |

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
`fieldName`, `fieldType`, `isArray`, `searchable`, `filterable`, `facetable`, `sortable`, `wordIndexing`, `embeddable`, `weight` (float), `bM25b` (float, 0–1), `bM25k1` (float, 1–2), `preloadFilters`.

`fieldType` and `isArray` are read-only — the server fills them in on `GetFieldConfiguration` and ignores them on `SetFieldConfiguration`.

### CloudQuery (Search Request)

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

`sortBy` — field name as a **string** (e.g. `"price"`).

`fieldBoosts` — per-field BM25F boost multipliers:
```json
{ "fieldBoosts": { "title": 2.0, "description": 1.0 } }
```

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
- `truncationIndex` — where coverage truncation occurred (–1 if none).
- `didTimeOut` — hit the timeout limit.

Use `POST .../GetJson` with the array of `documentKey` values to retrieve full JSON documents.

### Filter and Boost Models

```json
// ValueFilterProxy — equality match
{ "fieldName": "category", "value": "electronics" }

// RangeFilterProxy — inclusive numeric range
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

// EmbeddingResultEntry — returned by VectorSearch / HybridSearch
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

`systemState`: `-1`=Hibernated, `0`=Created, `1`=Loading, `2`=Loaded, `3`=Indexing, `4`=Ready, `255`=Error.

`GetStatus` returns `CloudSystemStatus`, which adds cloud-layer fields — most usefully `shadowBuildInProgress` (true while a background rebuild runs).

## HTTP API Workflow

All steps below are under `/api/teams/{team}/datasets/{dataset}/`.

```
1. Create a token on the IndxCloudApi website (Account → API Key)
2. PUT   CreateOrOpen              → create dataset
3. POST  AnalyzeStreamAsync        → discover fields
4. PUT   SetFieldConfiguration     → configure all fields in one call
5. PUT   LoadStream                → load JSON data
6. GET   IndexDataSet              → trigger indexing
7. GET   GetStatus                 → poll until systemState = 4 (Ready)
8. POST  Search                    → returns records with documentKey + score
9. POST  GetJson                   → fetch full JSON by document keys
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
curl -X PUT "$BASE/CreateValueFilter" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"fieldName":"category","value":"electronics"}'
# → {"hashString":"abc123..."}

curl -X PUT "$BASE/CreateRangeFilter" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"fieldName":"price","lowerLimit":10,"upperLimit":100}'
# → {"hashString":"def456..."}

# 2. Combine (AND)
curl -X PUT "$BASE/CombineFilters" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"a":{"hashString":"abc123"},"b":{"hashString":"def456"},"useAndOperation":true}'
# → {"hashString":"combined789..."}

# 3. Search with filter
curl -X POST "$BASE/Search" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"text":"wireless","maxNumberOfRecordsToReturn":20,"filter":{"hashString":"combined789..."}}'
```

**Boosted search:**
```bash
curl -X PUT "$BASE/CreateBoost" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"boostStrength":3,"filterProxy":{"hashString":"<filter-hash>"}}'

curl -X POST "$BASE/Search" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"text":"headphones","maxNumberOfRecordsToReturn":20,"enableBoost":true,"boosts":[{"boostStrength":3,"filterProxy":{"hashString":"<filter-hash>"}}]}'
```

**Dynamic insert:**
```bash
curl -X POST "$BASE/insert" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '["{\"id\":999,\"name\":\"New Product\",\"price\":29.99}"]'
```

**Partial field update:**
```bash
curl -X PUT "$BASE/field/42" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"fieldName":"price","value":24.99}'
```

**Retrieve full documents:**
```bash
curl -X POST "$BASE/GetJson" \
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

// 1. Create dataset
await client.put('CreateOrOpen', '');

// 2. Analyze
const jsonData = fs.readFileSync('products.json', 'utf-8');
await client.post('AnalyzeStreamAsync', jsonData, {
  headers: { 'Content-Type': 'application/json' }
});

// 3. Configure fields — single call replaces all the old Set* endpoints
await client.put('SetFieldConfiguration', [
  { fieldName: 'name',        searchable: true, weight: 2.0 },
  { fieldName: 'description', searchable: true, weight: 1.0 },
  { fieldName: 'category',    filterable: true, facetable: true },
  { fieldName: 'price',       filterable: true, sortable: true },
]);

// 4. Load
const fileStream = fs.createReadStream('products.json');
const fileStats = fs.statSync('products.json');
await client.put('LoadStream', fileStream, {
  headers: { 'Content-Type': 'application/json', 'Content-Length': fileStats.size },
  maxBodyLength: Infinity
});

// 5. Index and wait
await client.get('IndexDataSet');
let ready = false;
while (!ready) {
  await new Promise(r => setTimeout(r, 200));
  const { data } = await client.get('GetStatus');
  ready = data.systemState === 4;
}

// 6. Search
const { data: result } = await client.post('Search', {
  text: 'wireless headphones',
  maxNumberOfRecordsToReturn: 10
});

// 7. Fetch full documents
const keys = result.records.map((r: any) => r.documentKey);
const { data: docs } = await client.post('GetJson', keys);
```
