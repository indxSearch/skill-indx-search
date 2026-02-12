# C# NuGet Integration

## Install

```bash
dotnet add package IndxSearchLib
```

Targets **.NET 9.0**. Current version: **4.1.2**.

## SearchEngine Constructor

```csharp
// Default — works for most cases (config 400)
var engine = new SearchEngine();

// With license file
var engine = new SearchEngine("indx-developer.license");

// With logging and custom config (400 is recommended for most use cases)
var engine = new SearchEngine(logPrefix: "MyApp", factory: loggerFactory, configurationNumber: 400, licenseFileName: "indx-developer.license");
```

## Basic Usage

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

## SearchEngine Lifecycle

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

## ProcessMonitor (Async Operations)

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

## Analyzing Fields

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

## Retrieving Field Configuration

```csharp
// Facetable fields
List<Field> facetableFields = engine.DocumentFields.GetFacetableFieldList();

// Filterable fields
List<Field> filterableFields = engine.DocumentFields.GetFilterableFieldList();

// All fields (check individual properties for Searchable, Sortable, etc.)
List<Field> allFields = engine.DocumentFields.GetFieldList();
```

## Saving and Loading Field Configuration

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

## Query Object

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

## Handling Results

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

## Filters

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

Preload filters for faster first search. Useful for large datasets where many filters are applied simultaneously (e.g. e-commerce with hundreds of warehouse/store availability filters per query):
```csharp
engine.LoadFilters(new[] { categoryFilter, priceFilter }, threadCount: 2);
// Or preload all: engine.LoadAllFilters(maxThreadCount: 4);
```

## Boosts

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

## Personalized Boosting with KeyFilter

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

## Coverage Control

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

## Common C# Patterns

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

## Licensing

- **No license**: 100,000 document limit
- **Extended license (free)**: Unlimited — register at [indx.co](https://indx.co)
- **Company license (paid)**: Unlimited + SLA and support

Pass the license path to `SearchEngine` constructor or place `.license` file in the working directory.
