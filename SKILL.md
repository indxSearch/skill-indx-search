# Indx Search — Agent Skill

Indx is a high-performance search engine for structured and unstructured text. It uses pattern recognition instead of tokenizers, stemmers, or analyzers — handling typos, formatting variations, and messy input without configuration.

## Choosing Your Integration Path

**C# / .NET project** → Use the [IndxSearchLib NuGet package](https://www.nuget.org/packages/IndxSearchLib/) directly. Embed search into your application with no external dependencies.

**Any other tech stack** (Node.js, Python, Java, etc.) → Deploy the [IndxCloudApi](https://github.com/indxSearch/IndxCloudApi) HTTP API server and interact via REST. Recommended deployment target: Azure App Service (works with zero config).

## Core Concepts

### Two-Step Search Model

Every search executes in two phases:

1. **Pattern Matching** — Scans all documents for textual and structural patterns. Produces candidate results with strong recall and built-in typo tolerance. No query preprocessing needed.

2. **Coverage** (enabled by default) — A collection of algorithms that detect exact and near-exact token matches (whole words, fuzzy words, joined/split words, prefixes/suffixes) in the top-K candidates (default: 500). Confirmed matches are scored 0–255 and promoted above pure pattern matches. A truncation index marks where coverage-confirmed results end.

### Field Configuration

Fields must be explicitly marked with their roles before indexing:

| Role | Purpose | Notes |
|------|---------|-------|
| **Searchable** | Included in matching and scoring | At least one required. Supports weight for relative importance |
| **Filterable** | Available for filter operations | Used with value filters and range filters |
| **Facetable** | Used for aggregations | Returns value counts (histograms) |
| **Sortable** | Enables result ordering | Works on numbers and strings. See sorting behavior below |
| **WordIndexing** | Enables word-level index | Improves performance for large text fields |

**Weight priority** — Searchable fields support a weight to control importance:
- C# API: `Weight.High`, `Weight.Med`, `Weight.Low` (enum)
- HTTP API: `0` (High), `1` (Medium), `2` (Low) (integer)

Note: Weights affect pattern recognition directly. A short text pattern in a longer string will not necessarily rank higher than the same pattern in a shorter string, even if the longer field has higher weight.

### Search Behavior Guidance

- **Human-facing search**: Disable coverage (`EnableCoverage = false`) for broader results from pattern matching. Good for exploration and browsing.
- **Agent/tool usage**: Keep coverage enabled (the default). Reduces noise and false positives for programmatic consumption.
- **Empty search**: Supported with empty/null query text. Requires facets enabled and at least one facetable field. Returns all documents, ignores `CoverageDepth`.

### Sorting Behavior

- **With search text**: Sorting is **2nd-order** — search relevancy (score) is always the primary sort. Sorting only applies to results included in the coverage step.
- **Empty search** (no text): Sorting becomes the **primary** ordering function.
- Works on both numbers and strings (A–Z, 1–9). Default: descending. Set `SortAscending = true` to invert.
- To reset: set `query.SortBy = null`.

### Facets Tip

When implementing search-as-you-type with a large dataset, consider only fetching facets (`EnableFacets = true`) after a small delay, not on every keystroke.

### Coverage Tuning

- `EnableCoverage` (default: `true`) — Toggle the coverage refinement step.
- `CoverageDepth` (default: `500`) — Number of top-K pattern-match candidates to evaluate. Higher = better recall, more latency. Auto-increases if `MaxNumberOfRecordsToReturn > CoverageDepth`. Set to `engine.Status.DocumentCount` for full-dataset coverage.
- `CoverageSetup` — Fine-grained control (see advanced sections below).

---

## C# NuGet Integration

### Install

```bash
dotnet add package IndxSearchLib
```

Targets **.NET 9.0**. Current version: **4.1.2**.

### SearchEngine Constructor

```csharp
// Default — works for most cases (config 400)
var engine = new SearchEngine();

// With license file
var engine = new SearchEngine("indx-developer.license");

// With logging and custom config (400 is recommended for most use cases)
var engine = new SearchEngine(logPrefix: "MyApp", factory: loggerFactory, configurationNumber: 400, licenseFileName: "indx-developer.license");
```

### Basic Usage

```csharp
using Indx.Api;

var engine = new SearchEngine();

// 1. Analyze JSON structure
FileStream fstream = File.Open("products.json", FileMode.Open, FileAccess.Read);
engine.Init(fstream);

// 2. Configure fields
engine.GetField("name")!.Searchable = true;
engine.GetField("name")!.Weight = Weight.High;
engine.GetField("description")!.Searchable = true;
engine.GetField("description")!.Weight = Weight.Med;
engine.GetField("category")!.Filterable = true;
engine.GetField("category")!.Facetable = true;
engine.GetField("price")!.Filterable = true;
engine.GetField("price")!.Sortable = true;

// 3. Load and index
fstream.Position = 0;
engine.Load(fstream);
fstream.Close();
engine.Index();

// 4. Search
var result = engine.Search(new Query("wireless headphones", 20));
```

### SearchEngine Lifecycle

```
Init(stream) → Configure fields → Load(stream) → Index() → Search(query)
```

**Core methods:**

| Method | Description |
|--------|-------------|
| `Init(Stream)` / `Init(Stream, ProcessMonitor?)` | Analyze JSON structure, discover fields. Blocks if monitor is null |
| `GetField(name)` | Get a `Field` object to configure (Searchable, Filterable, Facetable, Sortable, WordIndexing, Weight) |
| `Load(Stream)` / `Load(Stream, ProcessMonitor?)` | Load JSON documents into memory. Blocks if monitor is null |
| `Index()` / `Index(ProcessMonitor)` | Build the search index asynchronously. Check `Status.SystemState` for progress |
| `Search(Query)` | Execute a search, returns `Result` |
| `GetJsonDataOfKey(long)` | Retrieve the full JSON string for a document key |
| `GetAllDocuments()` | Returns `List<Document>` of all loaded documents |
| `DeleteJsonRecord(long)` | Delete a single document by key |
| `DeleteRecordsInFilter(Filter)` | Bulk delete all documents matching a filter |

**Memory management:**

| Method | Description |
|--------|-------------|
| `Hibernate(out string)` | Dispose indexed data to free memory. Searches will timeout. Can still configure fields and create filters |
| `WakeUp(ProcessMonitor?)` | Exit hibernation, re-index and resume |
| `Unload(out string)` | Delete all indexes and documents. Must be in Loaded or Ready state |
| `Dispose()` | Free all resources. Use for hot-swapping engine instances |

### ProcessMonitor (Async Operations)

`Init`, `Load`, and `Index` all accept an optional `ProcessMonitor` for non-blocking execution:

```csharp
var monitor = new ProcessMonitor();
monitor.TimeoutSeconds = 120;

engine.Load(fstream, monitor);

// Poll progress
while (monitor.IsRunning)
{
    Console.WriteLine($"Progress: {monitor.ProgressPercent}%");
    Thread.Sleep(200);
}

// Or wait synchronously / asynchronously
monitor.WaitForCompletion();
await monitor.WaitForCompletionAsync();

if (!monitor.Succeeded)
    Console.WriteLine($"Error: {monitor.ErrorMessage}");
```

Key properties: `IsRunning`, `ProgressPercent` (0–100), `Succeeded`, `ErrorMessage`, `DidTimeOut`, `IsCompleted`.
Supports `Cancel()` and configurable `ThreadPriority` (default: `Normal`).

Using ProcessMonitor allows you to run loading and indexing in parallel.

### Analyzing Fields

After `Init()`, inspect the discovered field structure:

```csharp
engine.Init(fstream);

var fields = engine.DocumentFields.GetFieldList();
fields.Sort((x, y) => x.Name.CompareTo(y.Name));
foreach (var field in fields)
{
    Console.WriteLine($"{field.Name} ({field.Type}) {(field.IsArray ? "IsArray" : "")} {(field.Optional ? "Optional" : "")}");
}
```

Field properties: `Name`, `Type` (JsonValueKind), `IsArray`, `Optional`, `Searchable`, `Filterable`, `Facetable`, `Sortable`, `WordIndexing`, `Weight` (enum), `PreloadFilters`.

`PreloadFilters` — when `true`, the engine creates and caches filters for each distinct value of this field during `Load`. Access via `SearchEngine.GetPreloadedFilters`.

### Retrieving Field Configuration

```csharp
// Facetable fields
List<Field> facetableFields = engine.DocumentFields.GetFacetableFieldList();

// Filterable fields
List<Field> filterableFields = engine.DocumentFields.GetFilterableFieldList();

// All fields (check individual properties for Searchable, Sortable, etc.)
List<Field> allFields = engine.DocumentFields.GetFieldList();
```

### Saving and Loading Field Configuration

After running `Init()` once, save the field configuration for faster subsequent loads:

```csharp
// First run: analyze, configure, save
engine.Init(fstream);
engine.GetField("title")!.Searchable = true;
engine.GetField("title")!.Weight = Weight.High;
engine.GetField("category")!.Facetable = true;
engine.DocumentFields.SaveToFile("fieldconfig.json");

// Subsequent runs: load config directly (skip Init)
engine.LoadDocumentFields("fieldconfig.json");
```

### Query Object

```csharp
var query = new Query("search text", maxResults)
{
    EnableCoverage = true,              // default: true
    CoverageDepth = 500,                // default: 500 (auto-increases if maxResults > this)
    CoverageSetup = coverageSetup,      // CoverageSetup object, default: null (uses defaults)
    EnableFacets = false,               // default: false
    EnableBoost = false,                // default: false
    SortBy = engine.GetField("rating"), // Field object, default: null
    SortAscending = false,              // default: false (descending)
    RemoveDuplicates = true,            // default: true
    Filter = filter,                    // Filter object, default: null
    Boosts = boostArray,                // Boost[], default: null
    KeyExcludeFilter = excludeFilter,   // KeyFilter — exclude specific documents
    KeyIncludeFilter = includeFilter,   // KeyFilter — only include specific documents
    TimeOutLimitMilliseconds = 1000     // default: 1000, max: 10000
};
```

### Handling Results

Search returns `Result` with document keys and scores — not full JSON:

```csharp
var result = engine.Search(query);
if (result != null)
{
    // Iterate scored results (ordered by score descending)
    foreach (var rec in result.Records)
    {
        long key = rec.DocumentKey;
        int score = rec.Score;
        string json = engine.GetJsonDataOfKey(key);
        Console.WriteLine($"[{score}] {json}");
    }

    // Access facets (when enableFacets = true)
    if (result.Facets != null)
    {
        foreach (var facet in result.Facets)
        {
            Console.WriteLine($"Facet: {facet.Key}");
            foreach (var bucket in facet.Value)
                Console.WriteLine($"  {bucket.Key}: {bucket.Value}");
        }
    }
}
```

Result properties:
- `Records` — `ScoreEntry[]` with `DocumentKey` (long) and `Score` (byte, 0–255). Score 255 = identical similarity. Coverage overrides pattern scores upward for near-exact matches. Boost reduces scores for non-boosted documents.
- `Facets` — `Dictionary<string, KeyValuePair<string, int>[]>` (field → value/count pairs)
- `TruncationIndex` — index in the results where coverage truncation occurred (> -1 if coverage detected exact/near-exact matches)
- `TruncationScore` — the score value at the truncation point
- `DidTimeOut` — `true` if not enough CPU resources to complete within the timeout

### Filters

Create filters on fields marked as Filterable. Precondition: `Init()` and `Load()` must be completed.

```csharp
// Value filter (equality) — filterValue must have ToString() or be string
Filter categoryFilter = engine.CreateValueFilter("category", "electronics")!;

// Range filter (numeric, inclusive on both ends)
Filter priceFilter = engine.CreateRangeFilter("price", 10.0, 100.0)!;

// Combine with operators: & (AND), | (OR), ! (NOT)
Filter combined = categoryFilter & priceFilter;   // electronics AND price 10-100
Filter either = categoryFilter | priceFilter;      // electronics OR price 10-100
Filter excluded = !categoryFilter;                  // everything EXCEPT electronics

// Check how many documents match
int count = combined.NumberOfDocumentsInFilter;

// Use in query
query.Filter = combined;

// Reset filter: pass null
query.Filter = null;
```

Preload filters for faster first search:
```csharp
engine.LoadFilters(new[] { categoryFilter, priceFilter }, threadCount: 2);
// Or preload all: engine.LoadAllFilters(maxThreadCount: 4);
```

### Boosts

Boost results matching certain criteria without excluding non-matching results. Boosting only affects results where coverage detects exact or near-exact word hits. Must be created after `Load()` completes. `CreateBoost` pre-calculates all score values of affected documents.

```csharp
// BoostStrength enum: Low = 1, Med = 2, High = 3
Filter yearFilter = engine.CreateRangeFilter("year", 1980, 2025)!;
Filter genreFilter = engine.CreateValueFilter("genre", "Documentary")!;

// CreateBoost accepts either Filter or KeyFilter
var boosts = new List<Boost>();
Boost b = engine.CreateBoost(yearFilter & genreFilter, BoostStrength.Med);
Console.WriteLine($"Documents boosted: {b.DocumentsBoosted}");
boosts.Add(b);

query.Boosts = boosts.ToArray();
query.EnableBoost = true;
```

### Personalized Boosting with KeyFilter

For user-specific boosting across many items, use `KeyFilter` with the `|` operator:

```csharp
// Build a KeyFilter from multiple value filters
KeyFilter frequentPurchases = new KeyFilter();
foreach (long itemId in userFrequentItemIds)
{
    Filter f = engine.CreateValueFilter("item_id", itemId)!;
    frequentPurchases = frequentPurchases | f.KeyFilter;
}

var userBoosts = new List<Boost>();
userBoosts.Add(engine.CreateBoost(frequentPurchases, BoostStrength.Med));
// Can boost hundreds of thousands of items without significant performance impact

query.Boosts = userBoosts.ToArray();
query.EnableBoost = true;
```

### Coverage Control

Coverage is a collection of algorithms that detect exact and near-exact token matches in the query, lifting them to the top of the result list. It complements the core pattern recognition which always runs first. For most applications the default values work well.

**How it works**: After pattern matching produces ranked candidates, coverage analyzes the top-K results (controlled by `CoverageDepth`) and scores each 0–255 based on how well the query tokens are represented. Results confirmed by coverage are promoted above pure pattern matches. A `TruncationIndex` in the result marks where coverage-confirmed results end.

```csharp
CoverageSetup cov = new CoverageSetup();

// Strict mode: only whole words (disable fuzzy, prefix/suffix)
cov.CoverFuzzyWords = false;
cov.CoverPrefixSuffix = false;
cov.CoverWholeQuery = false;
cov.CoverJoinedWords = true;
cov.CoverWholeWords = true;
cov.MinWordSize = 3;

query.CoverageSetup = cov;
```

**Detection algorithms** (all default `true`):

| Property | Description |
|----------|-------------|
| `CoverWholeQuery` | Detect the whole search query as a single string |
| `CoverWholeWords` | Detect individual whole words from the query. Handles multiple words |
| `CoverFuzzyWords` | Detect words with minor error tolerance (edit distance) |
| `CoverJoinedWords` | Detect words that are either joined together or split apart. Both forms returned in same query |
| `CoverPrefixSuffix` | Detect incomplete strings as prefix or suffix of a bigger word |

**Result control**:

| Property | Default | Description |
|----------|---------|-------------|
| `Truncate` | `true` | Cut the result list at the truncation index |
| `IncludePatternMatches` | `true` | Include pure pattern matches not detected by coverage. Set to `false` to only return near-exact matches (requires coverage enabled) |
| `TruncationScore` | `255` | Score threshold: always truncate at or above this score |
| `TruncateWordHitLimit` | `1` | Minimum number of query words that must match for a result to survive truncation. Effective limit may be higher due to `TruncateWordHitTolerance` |
| `TruncateWordHitTolerance` | `0` | Maximum difference in word hit count from the best result (`maxWordHits`) to still include in truncated list |

**Token control**:

| Property | Default | Description |
|----------|---------|-------------|
| `MinWordSize` | `2` | Minimum token length to detect. Values 2–5 recommended |
| `LevenshteinMaxWordSize` | `20` | Longest word for edit distance calculation. Does not affect pattern recognition fault tolerance. Max: 64 |

### Common C# Patterns

**Near-exact hits only** — exclude pattern-match results, keep only coverage-confirmed matches:
```csharp
query.EnableCoverage = true;
CoverageSetup cov = new CoverageSetup();
cov.IncludePatternMatches = false; // only near-exact matches from coverage
query.CoverageSetup = cov;
```

**Pattern matching only** (disable coverage):
```csharp
query.EnableCoverage = false;
```

**Run coverage but keep all results** (don't truncate):
```csharp
query.EnableCoverage = true;
CoverageSetup cov = new CoverageSetup();
cov.Truncate = false;
query.CoverageSetup = cov;
```

**Deep coverage** (evaluate all documents):
```csharp
query.CoverageDepth = engine.Status.DocumentCount;
```

**Empty search with sorting and facets** (browse mode):
```csharp
query.Text = ""; // or null
query.SortBy = engine.GetField("rating")!; // descending by default
query.EnableFacets = true;
```

**Get min/max from facets** (for range filter UI):
```csharp
query.EnableFacets = true;
var result = engine.Search(query);
if (result.Facets != null && result.Facets.TryGetValue("price", out var histogram))
{
    var values = histogram.Select(item => int.Parse(item.Key)).ToList();
    int min = values.Min();
    int max = values.Max();
    Filter priceRange = engine.CreateRangeFilter("price", min, max)!;
}
```

### Licensing

- **No license**: 100,000 document limit
- **Extended license (free)**: Unlimited — register at [indx.co](https://indx.co)
- **Company license (paid)**: Unlimited + SLA and support

Pass the license path to `SearchEngine` constructor or place `.license` file in the working directory.

---

## HTTP API Integration

For non-.NET tech stacks, deploy the IndxCloudApi server and interact via REST.

### Setup

```bash
git clone https://github.com/indxSearch/IndxCloudApi
cd IndxCloudApi
dotnet run
```

Requires **.NET 9.0 SDK**. Access at `https://localhost:5001`. Register at `/Account/Register`. Swagger at `/swagger`.

**OpenAPI spec**: `https://localhost:5001/swagger/v1/swagger.json`

### Deploy to Azure (Recommended for Production)

```bash
# Create App Service with .NET 9 runtime, then:
dotnet publish -c Release
az webapp up --name your-app-name --resource-group your-resource-group
```

Works on Azure without any additional configuration. SQLite databases auto-create in persistent storage. See the [IndxCloudApi README](https://github.com/indxSearch/IndxCloudApi) for full Azure deployment guide.

### Authentication

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

### API Endpoints

All endpoints prefixed with `/api/`, JWT Bearer auth required.

#### Dataset Lifecycle

| Method | Endpoint | Description |
|--------|----------|-------------|
| PUT | `CreateOrOpen/{dataSetName}` | Create or open a dataset (default config) |
| PUT | `CreateOrOpen/{dataSetName}/{configuration}` | Create with explicit config (int) |
| DELETE | `DeleteDataSet/{dataSetName}` | Delete dataset permanently |
| GET | `GetUserDatasets` | List your datasets → `string[]` |
| GET | `GetStatus/{dataSetName}` | Get dataset status → `SystemStatus` |
| GET | `GetNumberOfJsonRecordsInDb/{dataSetName}` | Get document count → `int` |

#### Data Loading

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| POST | `AnalyzeString/{dataSetName}` | JSON as string | Analyze JSON structure, discover fields |
| POST | `AnalyzeStreamAsync/{dataSetName}` | JSON stream | Analyze via stream (large files) |
| PUT | `LoadString/{dataSetName}` | JSON as string | Load JSON documents |
| PUT | `LoadStream/{dataSetName}` | JSON stream body | Load via stream (large files) |
| GET | `LoadFromDatabase/{dataSetName}` | — | Reload persisted data into memory |

#### Field Configuration

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

#### Indexing and Search

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| GET | `IndexDataSet/{dataSetName}` | — | Trigger indexing (async) → `SystemStatus` |
| POST | `Search/{dataSetName}` | `CloudQuery` | Execute search → `Result` |
| POST | `GetJson/{dataSetName}` | `long[]` (document keys) | Retrieve full JSON records → `string[]` |

#### Filters and Boosts

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| PUT | `CreateValueFilter/{dataSetName}` | `ValueFilterProxy` | Create equality filter → `FilterProxy` |
| PUT | `CreateRangeFilter/{dataSetName}` | `RangeFilterProxy` | Create numeric range filter → `FilterProxy` |
| PUT | `CombineFilters/{dataSetName}` | `CombinedFilterProxy` | Combine with AND/OR → `FilterProxy` |
| PUT | `CreateBoost/{dataSetName}` | `BoostProxy` | Create boost rule → `BoostProxy` |

### Schemas

#### CloudQuery (Search Request)

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

#### Result (Search Response)

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

#### Filter and Boost Models

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

#### SystemStatus

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

### HTTP API Workflow

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

### Common HTTP Patterns

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

### Self-Host Production Configuration

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

## Data Format

Indx accepts JSON arrays of objects. Nested fields are supported (schemaless):

```json
[
  { "id": 1, "title": "Product A", "specs": { "weight": 1.2, "color": "red" } },
  { "id": 2, "title": "Product B", "specs": { "weight": 0.8, "color": "blue" } }
]
```

Nested fields use dot notation: `specs.weight`, `specs.color`.

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

---

## Key Design Properties

- **No language configuration** — works across languages without tokenizers, stemmers, or stop words
- **Built-in typo tolerance** — pattern matching handles misspellings automatically
- **In-memory indexing** — all search indexes live in memory for speed; persistence is metadata-only
- **Linear coverage scaling** — coverage cost scales linearly with `coverageDepth`
- **Schemaless JSON** — nested objects supported, fields discovered automatically via `Init`/`Analyze`
- **Two-step retrieval** — search returns keys + scores; fetch full documents separately (`GetJsonDataOfKey` in C#, `GetJson` endpoint in HTTP)

## Resources

- [Indx Home](https://indx.co) — registration and licensing
- [API Documentation](https://docs.indx.co/api-41) — full C# API reference with How-To guides
- **C# / .NET**
  - [IndxSearchLib NuGet](https://www.nuget.org/packages/IndxSearchLib/) — core search engine (.NET 9)
  - [IndxCloudLoader](https://github.com/indxSearch/IndxCloudLoader) — C# data loading reference
- **HTTP API**
  - [IndxCloudApi](https://github.com/indxSearch/IndxCloudApi) — self-host server template (ASP.NET Core)
  - [OpenAPI spec](https://cloud.indx.co/swagger/v1/swagger.json) — machine-readable API definition
- **Node.js / TypeScript**
  - [@indxsearch/indx-types](https://www.npmjs.com/package/@indxsearch/indx-types) — TypeScript type definitions
  - [IndxNodeLoader](https://github.com/indxSearch/IndxNodeLoader) — Node.js data loading reference
- **Frontend**
  - [indx-intrface](https://github.com/indxSearch/indx-intrface) — React search UI components (@indxsearch/intrface)
