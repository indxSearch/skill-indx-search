# HTTP API Integration

For non-.NET tech stacks, deploy the IndxCloudApi server and interact via REST.

## Setup

```bash
git clone https://github.com/indxSearch/IndxCloudApi
cd IndxCloudApi
dotnet run
```

Requires **.NET 9.0 SDK**. Access at `https://localhost:5001`. Register at `/Account/Register`. Swagger at `/swagger`.

**OpenAPI spec**: `https://localhost:5001/swagger/v1/swagger.json`

## Deploy to Azure (Recommended for Production)

```bash
# Create App Service with .NET 9 runtime, then:
dotnet publish -c Release
az webapp up --name your-app-name --resource-group your-resource-group
```

Works on Azure without any additional configuration. SQLite databases auto-create in persistent storage. See the [IndxCloudApi README](https://github.com/indxSearch/IndxCloudApi) for full Azure deployment guide.

## Authentication

```bash
# 1. Login to get JWT token
curl -X POST https://localhost:5001/api/Login \
  -H "Content-Type: application/json" \
  -d '{"userEmail": "you@example.com", "userPassWord": "YourPass1!"}'
# → returns JWT token

# 2. Use token in all requests
curl -H "Authorization: Bearer <token>" https://localhost:5001/api/...
```

Long-lived API keys (30-365 days) can be generated from the web UI at `/Account/Manage`.

## API Endpoints

All endpoints prefixed with `/api/`, JWT Bearer auth required.

### Dataset Lifecycle

| Method | Endpoint | Description |
|--------|----------|-------------|
| PUT | `CreateOrOpen/{dataSetName}` | Create or open a dataset (default config) |
| PUT | `CreateOrOpen/{dataSetName}/{configuration}` | Create with explicit config (int) |
| DELETE | `DeleteDataSet/{dataSetName}` | Delete dataset permanently |
| GET | `GetUserDatasets` | List your datasets → `string[]` |
| GET | `GetStatus/{dataSetName}` | Get dataset status → `SystemStatus` |
| GET | `GetNumberOfJsonRecordsInDb/{dataSetName}` | Get document count → `int` |

### Data Loading

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| POST | `AnalyzeString/{dataSetName}` | JSON as string | Analyze JSON structure, discover fields |
| POST | `AnalyzeStreamAsync/{dataSetName}` | JSON stream | Analyze via stream (large files) |
| PUT | `LoadString/{dataSetName}` | JSON as string | Load JSON documents |
| PUT | `LoadStream/{dataSetName}` | JSON stream body | Load via stream (large files) |
| GET | `LoadFromDatabase/{dataSetName}` | — | Reload persisted data into memory |

### Field Configuration

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| PUT | `SetSearchableFields/{dataSetName}` | `[{"Item1":"title","Item2":0}]` | Set searchable fields with weights (0=High, 1=Med, 2=Low) |
| PUT | `SetFilterableFields/{dataSetName}` | `["field1","field2"]` | Mark fields as filterable |
| PUT | `SetFacetableFields/{dataSetName}` | `["field1","field2"]` | Mark fields as facetable |
| PUT | `SetSortableFields/{dataSetName}` | `["field1","field2"]` | Mark fields as sortable |
| PUT | `SetWordIndexingFields/{dataSetName}` | `["field1","field2"]` | Mark fields for word indexing |
| GET | `GetSearchableFields/{dataSetName}` | — | → `string[]` |
| GET | `GetFilterableFields/{dataSetName}` | — | → `string[]` |
| GET | `GetFacetableFields/{dataSetName}` | — | → `string[]` |
| GET | `GetSortableFields/{dataSetName}` | — | → `string[]` |
| GET | `GetWordIndexingFields/{dataSetName}` | — | → `string[]` |
| GET | `GetallFields/{dataSetName}` | — | All discovered fields → `string[]` |

### Indexing and Search

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| GET | `IndexDataSet/{dataSetName}` | — | Trigger indexing (async) → `SystemStatus` |
| POST | `Search/{dataSetName}` | `CloudQuery` | Execute search → `Result` |
| POST | `GetJson/{dataSetName}` | `long[]` (document keys) | Retrieve full JSON records → `string[]` |

### Filters and Boosts

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| PUT | `CreateValueFilter/{dataSetName}` | `ValueFilterProxy` | Create equality filter → `FilterProxy` |
| PUT | `CreateRangeFilter/{dataSetName}` | `RangeFilterProxy` | Create numeric range filter → `FilterProxy` |
| PUT | `CombineFilters/{dataSetName}` | `CombinedFilterProxy` | Combine with AND/OR → `FilterProxy` |
| PUT | `CreateBoost/{dataSetName}` | `BoostProxy` | Create boost rule → `BoostProxy` |

## Schemas

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
  "coverageSetup": null
}
```

Full with CoverageSetup (defaults shown):
```json
{
  "text": "search terms",
  "maxNumberOfRecordsToReturn": 30,
  "enableCoverage": true,
  "coverageDepth": 1000,
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
    { "documentKey": 42, "score": 890 },
    { "documentKey": 17, "score": 650 }
  ],
  "facets": {
    "category": [
      { "key": "electronics", "value": 12 },
      { "key": "accessories", "value": 5 }
    ]
  },
  "truncationIndex": 15,
  "truncationScore": 255,
  "didTimeOut": false
}
```

- `records` — Array of `{ documentKey (int64), score (int32) }`. Use `GetJson` to retrieve full documents.
- `facets` — Map of field name → `[{key, value}]` pairs. Only populated when `enableFacets: true`.
- `truncationIndex` — Where coverage truncation occurred in the results.
- `didTimeOut` — Whether the search hit the timeout limit.

### Filter and Boost Models

```json
// ValueFilterProxy — equality match on a filterable field
{ "fieldName": "category", "value": "electronics" }

// RangeFilterProxy — numeric range on a filterable field
{ "fieldName": "price", "lowerLimit": 10.0, "upperLimit": 100.0 }

// FilterProxy — returned by filter creation endpoints, referenced in queries
{ "hashString": "<filter-hash>" }

// CombinedFilterProxy — combine two existing filters
{
  "a": { "hashString": "<filter-a-hash>" },
  "b": { "hashString": "<filter-b-hash>" },
  "useAndOperation": true
}

// BoostProxy — boost results matching a filter
// boostStrength: 1 (Low), 2 (Med), 3 (High)
{ "boostStrength": 2, "filterProxy": { "hashString": "<filter-hash>" } }
```

### SystemStatus

```json
{
  "systemState": 4,
  "documentCount": 5000,
  "searchCounter": 42,
  "secondsToIndex": 2,
  "reIndexRequired": false,
  "version": "4.1.2",
  "errorMessage": null,
  "invalidDataSetName": false,
  "invalidState": false
}
```

`systemState`: `-1`=Hibernated, `0`=Created, `1`=Loading, `2`=Loaded, `3`=Indexing, `4`=Ready, `255`=Error.

## HTTP API Workflow

```
1. POST  Login                              → get JWT token
2. PUT   CreateOrOpen/{dataSetName}         → create dataset
3. POST  AnalyzeString/{dataSetName}        → discover fields (JSON string as body)
4. PUT   SetSearchableFields/{dataSetName}  → [{"Item1":"title","Item2":0}]
5. PUT   Set*Fields/{dataSetName}           → configure filterable/facetable/sortable
6. PUT   LoadString/{dataSetName}           → load JSON data
7. GET   IndexDataSet/{dataSetName}         → trigger indexing
8. GET   GetStatus/{dataSetName}            → poll until systemState = 4 (Ready)
9. POST  Search/{dataSetName}              → returns document keys + scores
10. POST GetJson/{dataSetName}             → fetch full JSON by document keys
```

## Common HTTP Patterns

**Agent-optimized search** (precise, tight results):
```json
{ "text": "exact product name XYZ-123", "maxNumberOfRecordsToReturn": 10 }
```

**Human-facing exploratory search** (broad results with facets):
```json
{
  "text": "headphones",
  "maxNumberOfRecordsToReturn": 50,
  "enableCoverage": false,
  "enableFacets": true
}
```

**Empty search** (browse mode with sorting):
```json
{
  "text": "",
  "maxNumberOfRecordsToReturn": 50,
  "sortBy": "rating",
  "enableFacets": true
}
```

**Filtered search:**
```bash
# 1. Create filters
curl -X PUT .../api/CreateValueFilter/products \
  -H "Content-Type: application/json" \
  -d '{"fieldName":"category","value":"electronics"}'
# → {"hashString":"abc123..."}

curl -X PUT .../api/CreateRangeFilter/products \
  -H "Content-Type: application/json" \
  -d '{"fieldName":"price","lowerLimit":10,"upperLimit":100}'
# → {"hashString":"def456..."}

# 2. Combine filters (AND)
curl -X PUT .../api/CombineFilters/products \
  -H "Content-Type: application/json" \
  -d '{"a":{"hashString":"abc123..."},"b":{"hashString":"def456..."},"useAndOperation":true}'
# → {"hashString":"combined789..."}

# 3. Search with filter
curl -X POST .../api/Search/products \
  -H "Content-Type: application/json" \
  -d '{"text":"wireless","maxNumberOfRecordsToReturn":20,"filter":{"hashString":"combined789..."}}'
```

**Boosted search:**
```bash
# 1. Create a boost (boostStrength: 1=Low, 2=Med, 3=High)
curl -X PUT .../api/CreateBoost/products \
  -H "Content-Type: application/json" \
  -d '{"boostStrength":3,"filterProxy":{"hashString":"<filter-hash>"}}'

# 2. Search with boost
curl -X POST .../api/Search/products \
  -H "Content-Type: application/json" \
  -d '{"text":"headphones","maxNumberOfRecordsToReturn":20,"enableBoost":true,"boosts":[{"boostStrength":3,"filterProxy":{"hashString":"<filter-hash>"}}]}'
```

**Retrieve full documents:**
```bash
curl -X POST .../api/GetJson/products \
  -H "Content-Type: application/json" \
  -d '[42, 17]'
# → ["{ \"id\": 42, \"title\": \"Wireless Headphones\", ... }", "..."]
```

## Self-Host Production Configuration

JWT security:
```bash
dotnet user-secrets set "Jwt:Key" "your-secret-key-minimum-32-characters"
```

Registration control:
```json
{ "Registration": { "Mode": "Open|EmailDomain|Closed", "AllowedDomains": ["yourcompany.com"] } }
```

License: place `.license` file in `./IndxData/` directory.

---

## Data Loading Reference

Loading data into Indx via the HTTP API follows a consistent pattern regardless of your tech stack. Reference implementations exist for both Node.js and C#.

### TypeScript / Node.js

Install the types package for full TypeScript support:

```bash
pnpm add @indxsearch/indx-types axios
```

The `@indxsearch/indx-types` package exports: `SystemStatus`, `SystemState`, `CloudQuery`, `Result`, `FilterProxy`, `RangeFilterProxy`, `ValueFilterProxy`, `CombinedFilterProxy`, `BoostProxy`, `BoostStrength`.

**Complete load-and-index workflow (Node.js):**

```typescript
import axios from 'axios';
import * as fs from 'fs';
import { SystemState, CloudQuery } from '@indxsearch/indx-types';

const API = 'https://localhost:5001/api';

// Authenticate
const loginRes = await axios.post(`${API}/Login`, {
  userEmail: 'you@example.com',
  userPassWord: 'YourPass1!'
});
const token = loginRes.data.token;
const client = axios.create({
  baseURL: API,
  headers: { Authorization: `Bearer ${token}` }
});

const dataset = 'products';

// 1. Create dataset
await client.put(`CreateOrOpen/${dataset}`, '');

// 2. Analyze JSON structure (discover fields)
const jsonData = fs.readFileSync('products.json', 'utf-8');
await client.post(`AnalyzeString/${dataset}`, jsonData, {
  headers: { 'Content-Type': 'text/plain' }
});

// 3. Configure fields
//    Searchable: Item1 = field name, Item2 = weight (0=High, 1=Med, 2=Low)
await client.put(`SetSearchableFields/${dataset}`, [
  { Item1: 'name', Item2: 0 },
  { Item1: 'description', Item2: 1 }
]);
await client.put(`SetFilterableFields/${dataset}`, ['category', 'price']);
await client.put(`SetFacetableFields/${dataset}`, ['category']);
await client.put(`SetSortableFields/${dataset}`, ['price']);

// 4. Load data via stream (recommended for large files)
const fileStream = fs.createReadStream('products.json');
const fileStats = fs.statSync('products.json');
await client.put(`LoadStream/${dataset}`, fileStream, {
  headers: {
    'Content-Type': 'text/plain',
    'Content-Length': fileStats.size.toString()
  },
  maxBodyLength: Infinity
});

// 5. Index and wait for ready
await client.get(`IndexDataSet/${dataset}`);
let status;
do {
  await new Promise(r => setTimeout(r, 200));
  const res = await client.get(`GetStatus/${dataset}`);
  status = res.data;
} while (status.systemState !== 4); // 4 = Ready

// 6. Search
const query: CloudQuery = {
  text: 'wireless headphones',
  maxNumberOfRecordsToReturn: 10
};
const searchRes = await client.post(`Search/${dataset}`, query);
const result = searchRes.data;

// 7. Fetch full documents
const keys = result.records.map((r: any) => r.documentKey);
const docsRes = await client.post(`GetJson/${dataset}`, keys);
console.log(docsRes.data); // string[] of JSON records
```

### Dataset Configuration Pattern

Structure your field configuration as a reusable config object:

```typescript
// TypeScript
interface DatasetConfig {
  name: string;
  filePath: string;
  searchableFields: { name: string; weight: number }[];  // 0=High, 1=Med, 2=Low
  wordIndexingFields: string[];
  filterableFields: string[];
  facetableFields: string[];
  sortableFields: string[];
}

const moviesConfig: DatasetConfig = {
  name: 'tmdb',
  filePath: 'data/tmdb_top10k.json',
  searchableFields: [
    { name: 'title', weight: 0 },          // High
    { name: 'original_title', weight: 1 },  // Med
    { name: 'description', weight: 1 },     // Med
    { name: 'actors', weight: 2 }           // Low
  ],
  wordIndexingFields: ['title'],
  filterableFields: ['release_year', 'vote_average', 'genres', 'actors'],
  facetableFields: ['release_year', 'vote_average', 'genres', 'actors'],
  sortableFields: ['popularity', 'vote_average']
};
```

```csharp
// C# equivalent — note: weights use (string, int) tuples
var config = new {
    Name = "tmdb",
    FilePath = "data/tmdb_top10k.json",
    SearchableFields = new[] {
        ("title", (int)Weight.High),
        ("original_title", (int)Weight.Med),
        ("description", (int)Weight.Med),
        ("actors", (int)Weight.Low)
    },
    WordIndexingFields = new[] { "title" },
    FilterableFields = new[] { "release_year", "vote_average", "genres", "actors" },
    FacetableFields = new[] { "release_year", "vote_average", "genres", "actors" },
    SortableFields = new[] { "popularity", "vote_average" }
};
```

### Status Polling

After calling `IndexDataSet`, poll `GetStatus` until `systemState` reaches `4` (Ready):

```typescript
// Node.js
let ready = false;
while (!ready) {
  await new Promise(r => setTimeout(r, 200));
  const res = await client.get(`GetStatus/${dataset}`);
  ready = res.data.systemState === 4; // SystemState.Ready
}
```

States you may observe during loading: `0`→Created, `1`→Analyzing, `2`→Loaded, `3`→Indexing, `4`→Ready.

### Searchable Fields: The Item1/Item2 Format

When calling `SetSearchableFields` via the HTTP API, the server expects C# `ValueTuple` serialization format. You must use `Item1` (field name) and `Item2` (weight integer), **not** `name`/`weight`:

```json
// CORRECT
[{ "Item1": "title", "Item2": 0 }, { "Item1": "description", "Item2": 1 }]

// WRONG — will silently fail
[{ "name": "title", "weight": 0 }, { "name": "description", "weight": 1 }]
```

### Full Reference Implementations

For complete working examples with CLI menus, progress monitoring, error handling, and test searches:

- **Node.js / TypeScript**: [IndxNodeLoader](https://github.com/indxSearch/IndxNodeLoader) — uses `axios`, `@indxsearch/indx-types`
- **C# / .NET**: [IndxCloudLoader](https://github.com/indxSearch/IndxCloudLoader) — uses `HttpClient`, `System.Text.Json`

Both loaders include sample datasets (TMDB movies, Pokedex) for testing.
