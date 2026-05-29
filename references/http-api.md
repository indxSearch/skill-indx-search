# HTTP API Integration

For non-.NET tech stacks, deploy the IndxCloudApi server and interact via REST. For setup, deployment, and configuration see [cloudapi-setup.md](cloudapi-setup.md).

**OpenAPI spec**: `https://localhost:5001/swagger/v1/swagger.json`

## API Endpoints

All endpoints prefixed with `/api/`, JWT Bearer auth required.

### Dataset Lifecycle

| Method | Endpoint | Description |
|--------|----------|-------------|
| PUT | `CreateOrOpen/{dataSetName}` | Create or open a dataset (default config) |
| PUT | `CreateOrOpen/{dataSetName}/{configuration}` | Create with explicit config (int) |
| DELETE | `DeleteDataSet/{dataSetName}` | Delete dataset permanently |
| GET | `GetUserDatasets` | List your datasets → `DataSetListDto[]` |
| GET | `GetStatus/{dataSetName}` | Get dataset status → `SystemStatus` |
| GET | `GetNumberOfJsonRecordsInDb/{dataSetName}` | Get document count → `int` |

### Data Loading

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| POST | `AnalyzeStreamAsync/{dataSetName}` | JSON body | Analyze JSON structure, discover fields |
| POST | `AnalyzeString/{dataSetName}` | JSON as plain text string | Analyze from string |
| PUT | `LoadString/{dataSetName}` | JSON as plain text string | Load JSON documents |
| PUT | `LoadStream/{dataSetName}` | JSON body | Load via stream (large files) |
| GET | `LoadFromDatabase/{dataSetName}` | — | Reload persisted data into memory |

### Field Configuration

**Recommended — single unified endpoint:**

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| PUT | `SetFieldConfiguration/{dataSetName}` | `FieldProxy[]` | Set all field roles and weights in one call |
| GET | `GetFieldConfiguration/{dataSetName}` | — | Get current field configuration → `FieldProxy[]` |

**Legacy separate endpoints (still available):**

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| PUT | `SetSearchableFields/{dataSetName}` | `[{"Item1":"field","Item2":0}]` | Weights: 0=High, 1=Med, 2=Low |
| PUT | `SetFilterableFields/{dataSetName}` | `["field1","field2"]` | |
| PUT | `SetFacetableFields/{dataSetName}` | `["field1","field2"]` | |
| PUT | `SetSortableFields/{dataSetName}` | `["field1","field2"]` | |
| PUT | `SetWordIndexingFields/{dataSetName}` | `["field1","field2"]` | |

### Indexing and Search

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| GET | `IndexDataSet/{dataSetName}` | — | Trigger indexing → `SystemStatus` |
| POST | `Search/{dataSetName}` | `CloudQuery` | Execute search → `Result` |
| POST | `GetJson/{dataSetName}` | `long[]` (document keys) | Retrieve full JSON records → `string[]` |

### Dynamic Document Operations

Insert, update, and delete without rebuilding the index. The dataset stays ready throughout.

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| POST | `{dataSetName}/insert` | `string[]` (JSON objects) | Insert multiple documents |
| POST | `{dataSetName}/insert/{documentKey}` | JSON string | Insert single document |
| PUT | `{dataSetName}/update` | `string[]` (JSON objects) | Update multiple documents (must include key field) |
| PUT | `{dataSetName}/update/{documentKey}` | JSON string | Update single document |
| PUT | `{dataSetName}/field/{documentKey}` | `UpdateFieldProxy` | Partial field update on one document |
| DELETE | `{dataSetName}/{documentKey}` | — | Delete single document |
| DELETE | `{dataSetName}` | `long[]` | Delete multiple documents by key |
| DELETE | `DeleteRecordsInFilter/{dataSetName}` | `FilterProxy` | Delete all documents matching a filter |
| PUT | `UpdateFieldInFilter/{dataSetName}` | `FilterFieldUpdateProxy` | Batch update a field on all matching documents |

### Filters and Boosts

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| PUT | `CreateValueFilter/{dataSetName}` | `ValueFilterProxy` | Create equality filter → `FilterProxy` |
| PUT | `CreateRangeFilter/{dataSetName}` | `RangeFilterProxy` | Create numeric range filter → `FilterProxy` |
| PUT | `CombineFilters/{dataSetName}` | `CombinedFilterProxy` | Combine with AND/OR → `FilterProxy` |
| PUT | `CreateBoost/{dataSetName}` | `BoostProxy` | Create boost rule → `BoostProxy` |

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
`fieldName`, `fieldType`, `isArray`, `searchable`, `filterable`, `facetable`, `sortable`, `wordIndexing`, `embeddable`, `weight` (float), `bm25b` (float, 0–1), `bm25k1` (float, 1–2), `preloadFilters`.

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
    "truncationScore": 255,
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

Use `POST GetJson/{dataSetName}` with the array of `documentKey` values to retrieve full JSON documents.

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

### SystemStatus

```json
{
  "systemState": 4,
  "documentCount": 5000,
  "searchCounter": 42,
  "secondsToIndex": 2,
  "reIndexRequired": false,
  "version": "5.0.0",
  "errorMessage": null,
  "invalidDataSetName": false,
  "invalidState": false
}
```

`systemState`: `-1`=Hibernated, `0`=Created, `1`=Loading, `2`=Loaded, `3`=Indexing, `4`=Ready, `255`=Error.

## HTTP API Workflow

```
1. POST  Login                                   → get JWT token
2. PUT   CreateOrOpen/{dataSetName}              → create dataset
3. POST  AnalyzeStreamAsync/{dataSetName}        → discover fields
4. PUT   SetFieldConfiguration/{dataSetName}     → configure all fields in one call
5. PUT   LoadStream/{dataSetName}                → load JSON data
6. GET   IndexDataSet/{dataSetName}              → trigger indexing
7. GET   GetStatus/{dataSetName}                 → poll until systemState = 4 (Ready)
8. POST  Search/{dataSetName}                    → returns records with documentKey + score
9. POST  GetJson/{dataSetName}                   → fetch full JSON by document keys
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

**Filtered search:**
```bash
# 1. Create filters
curl -X PUT .../api/CreateValueFilter/products \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"fieldName":"category","value":"electronics"}'
# → {"hashString":"abc123..."}

curl -X PUT .../api/CreateRangeFilter/products \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"fieldName":"price","lowerLimit":10,"upperLimit":100}'
# → {"hashString":"def456..."}

# 2. Combine (AND)
curl -X PUT .../api/CombineFilters/products \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"a":{"hashString":"abc123"},"b":{"hashString":"def456"},"useAndOperation":true}'
# → {"hashString":"combined789..."}

# 3. Search with filter
curl -X POST .../api/Search/products \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"text":"wireless","maxNumberOfRecordsToReturn":20,"filter":{"hashString":"combined789..."}}'
```

**Boosted search:**
```bash
curl -X PUT .../api/CreateBoost/products \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"boostStrength":3,"filterProxy":{"hashString":"<filter-hash>"}}'

curl -X POST .../api/Search/products \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"text":"headphones","maxNumberOfRecordsToReturn":20,"enableBoost":true,"boosts":[{"boostStrength":3,"filterProxy":{"hashString":"<filter-hash>"}}]}'
```

**Dynamic insert:**
```bash
curl -X POST .../api/products/insert \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '["{\"id\":999,\"name\":\"New Product\",\"price\":29.99}"]'
```

**Partial field update:**
```bash
curl -X PUT .../api/products/field/42 \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"fieldName":"price","value":24.99}'
```

**Retrieve full documents:**
```bash
curl -X POST .../api/GetJson/products \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '[42, 17]'
# → ["{ \"id\": 42, ... }", "{ \"id\": 17, ... }"]
```

## Data Loading Reference (TypeScript / Node.js)

```bash
pnpm add @indxsearch/indx-types axios
```

```typescript
import axios from 'axios';
import * as fs from 'fs';

const API = 'https://localhost:5001/api';

// Authenticate
const { data: { token } } = await axios.post(`${API}/Login`, {
  userEmail: 'you@example.com',
  userPassWord: 'YourPass1!'
});
const client = axios.create({
  baseURL: API,
  headers: { Authorization: `Bearer ${token}` }
});

const dataset = 'products';

// 1. Create dataset
await client.put(`CreateOrOpen/${dataset}`, '');

// 2. Analyze
const jsonData = fs.readFileSync('products.json', 'utf-8');
await client.post(`AnalyzeStreamAsync/${dataset}`, jsonData, {
  headers: { 'Content-Type': 'application/json' }
});

// 3. Configure fields — single call replaces all the old Set* endpoints
await client.put(`SetFieldConfiguration/${dataset}`, [
  { fieldName: 'name',        searchable: true, weight: 2.0 },
  { fieldName: 'description', searchable: true, weight: 1.0 },
  { fieldName: 'category',    filterable: true, facetable: true },
  { fieldName: 'price',       filterable: true, sortable: true },
]);

// 4. Load
const fileStream = fs.createReadStream('products.json');
const fileStats = fs.statSync('products.json');
await client.put(`LoadStream/${dataset}`, fileStream, {
  headers: { 'Content-Type': 'application/json', 'Content-Length': fileStats.size },
  maxBodyLength: Infinity
});

// 5. Index and wait
await client.get(`IndexDataSet/${dataset}`);
let ready = false;
while (!ready) {
  await new Promise(r => setTimeout(r, 200));
  const { data } = await client.get(`GetStatus/${dataset}`);
  ready = data.systemState === 4;
}

// 6. Search
const { data: result } = await client.post(`Search/${dataset}`, {
  text: 'wireless headphones',
  maxNumberOfRecordsToReturn: 10
});

// 7. Fetch full documents
const keys = result.records.map((r: any) => r.documentKey);
const { data: docs } = await client.post(`GetJson/${dataset}`, keys);
```

**Reference implementations:**
- [IndxNodeLoader](https://github.com/indxSearch/IndxNodeLoader) — Node.js/TypeScript
- [IndxCloudLoader](https://github.com/indxSearch/IndxCloudLoader) — C#/.NET
