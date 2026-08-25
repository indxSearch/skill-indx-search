# C# NuGet Integration

## Install

```bash
dotnet add package IndxSearchLib
```

Targets **.NET 10.0**. Current version: **5.0.0**.

## SearchEngine Constructor

```csharp
// Default — works for most cases
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
using var stream = File.OpenRead("products.json");
engine.Init(stream);

// 2. Configure fields
engine.SetFieldConfiguration([
    new FieldProxy { FieldName = "name",        Searchable = true, Weight = 2.0f },
    new FieldProxy { FieldName = "description", Searchable = true, Weight = 1.0f },
    new FieldProxy { FieldName = "category",    Filterable = true, Facetable = true },
    new FieldProxy { FieldName = "price",       Filterable = true, Sortable = true },
]);

// 3. Load and index
stream.Position = 0;
engine.Load(stream);
engine.Index();

// 4. Search
var result = engine.Search(new Query("wireless headphones", 20));
```

## SearchEngine Lifecycle

```
Init(stream) → SetFieldConfiguration(...) → Load(stream) → Index() → Search(query)
```

**Core methods:**

| Method | Description |
|--------|-------------|
| `Init(Stream)` / `Init(Stream, ProcessMonitor?)` | Analyze JSON structure, discover fields. Blocks if monitor is null |
| `GetFieldConfiguration()` | Returns `FieldProxy[]` of all discovered fields with their current settings |
| `SetFieldConfiguration(FieldProxy[])` | Apply field roles and weights. Returns null on success, or the name of the first unknown field |
| `Load(Stream)` / `Load(Stream, ProcessMonitor?)` | Load JSON documents into memory. Blocks if monitor is null |
| `Index()` / `Index(ProcessMonitor)` | Build the search index. Check `Status.SystemState` for progress |
| `Search(Query)` | Execute a search, returns `Result` |
| `GetJsonDataOfKey(long key)` | Retrieve the full JSON string for a document key |

**Memory management:**

| Method | Description |
|--------|-------------|
| `Hibernate(out string)` | Dispose indexed data to free memory. Searches will timeout. Filters remain usable |
| `WakeUp()` / `WakeUp(int maxThreadCount)` | Exit hibernation, re-index and resume |
| `Dispose()` | Free all resources |

## ProcessMonitor (Async Operations)

`Init`, `Load`, and `Index` all accept an optional `ProcessMonitor` for non-blocking execution:

```csharp
var monitor = new ProcessMonitor();
monitor.TimeoutSeconds = 120;

engine.Load(fstream, monitor);

// Poll or wait
while (monitor.IsRunning)
    Thread.Sleep(200);

monitor.WaitForCompletion();
// or: await monitor.WaitForCompletionAsync();

if (!monitor.Succeeded)
    Console.WriteLine($"Error: {monitor.ErrorMessage}");
```

Key properties: `IsRunning`, `ProgressPercent` (0–100), `Succeeded`, `ErrorMessage`, `DidTimeOut`, `IsCompleted`.

## Analyzing Fields

After `Init()`, inspect discovered fields:

```csharp
engine.Init(stream);

foreach (var field in engine.GetFieldConfiguration())
{
    Console.WriteLine($"{field.FieldName} ({field.FieldType})" +
        $"{(field.IsArray == true ? " IsArray" : "")}");
}
```

`FieldProxy` properties: `FieldName`, `FieldType` ("String"/"Number"/"Boolean"), `IsArray`, `Searchable`, `Filterable`, `Facetable`, `Sortable`, `WordIndexing`, `Embeddable`, `Weight`, `BM25b`, `BM25k1`, `PreloadFilters`, `HighResolution` (also index/query N-grams with delimiters removed, so a run-together or split query matches across them).

## Field Configuration

### Applying Configuration

Use `SetFieldConfiguration` to configure fields in a single call. Only set the properties that matter — null properties are left unchanged.

```csharp
engine.SetFieldConfiguration([
    new FieldProxy { FieldName = "title",       Searchable = true, Weight = 2.0f },
    new FieldProxy { FieldName = "description", Searchable = true, Weight = 1.0f },
    new FieldProxy { FieldName = "category",    Filterable = true, Facetable = true },
    new FieldProxy { FieldName = "price",       Filterable = true, Sortable = true },
    new FieldProxy { FieldName = "rating",      Sortable = true },
]);
```

`Weight` is a `float`. Higher values increase the field's influence on BM25 scoring relative to other searchable fields. Typical range: 0.5–3.0.

### Per-Field BM25 Tuning (Advanced)

By default all searchable fields share the same `BM25k1` value (1.2), which activates BM25F scoring (single unified index). Setting different `BM25k1` values across fields switches to per-field BM25 scoring:

```csharp
new FieldProxy { FieldName = "title",       Searchable = true, BM25k1 = 1.5f, BM25b = 0.5f },
new FieldProxy { FieldName = "description", Searchable = true, BM25k1 = 1.2f, BM25b = 0.75f },
```

`BM25b` controls length normalization (range 0–1, default 0.75). `BM25k1` controls term frequency saturation (range 1.0–2.0, default 1.2).

### Saving and Loading Field Configuration

After configuring once, save to skip `Init` on subsequent loads:

```csharp
// First run: analyze, configure, save
engine.Init(stream);
engine.SetFieldConfiguration([...]);
engine.SaveFieldConfiguration("fieldconfig.json");

// Subsequent runs: load config (skip Init)
engine.LoadFieldConfiguration("fieldconfig.json");
```

## Query Object

```csharp
var query = new Query("search text", maxResults)
{
    EnableCoverage = true,                  // default: true
    CoverageDepth = 500,                    // default: 500
    CoverageSetup = coverageSetup,          // CoverageSetup object, default: null
    EnableFacets = false,                   // default: false
    EnableBoost = false,                    // default: false
    SortBy = engine.GetField("rating"),     // Field object, default: null
    SortAscending = false,                  // default: false
    RemoveDuplicates = true,                // default: true
    Filter = filter,                        // Filter object, default: null
    Boosts = boostArray,                    // Boost[], default: null
    FieldBoosts = new Dictionary<string, float> { ["title"] = 2.0f }, // BM25F per-query boost
    TimeOutLimitMilliseconds = 1000         // default: 1000, max: 10000
};
```

`FieldBoosts` — per-query field boost multipliers for the BM25F scoring path. Only has effect when `ScoringMode` is `BM25F`. Fields not listed default to 1.0.

## Handling Results

```csharp
var result = engine.Search(query);

foreach (var entry in result.Records)
{
    long key    = entry.DocumentKey;
    ushort score = entry.Score;          // 0–65535; higher = better match
    string json = engine.GetJsonDataOfKey(key);
    Console.WriteLine($"[{score}] {json}");
}

// Access facets
if (result.Facets != null)
{
    foreach (var facet in result.Facets)
        foreach (var bucket in facet.Value)
            Console.WriteLine($"{facet.Key}: {bucket.Key} ({bucket.Value})");
}
```

Result properties:
- `Records` — `ScoreEntry16[]` with `DocumentKey` (long) and `Score` (ushort, 0–65535). Coverage-confirmed matches always score higher than pure pattern matches.
- `Facets` — `Dictionary<string, KeyValuePair<string, int>[]>` (field → value/count pairs)
- `TruncationIndex` — index where coverage truncation occurred (–1 if no truncation)
- `TruncationScore` — score value at the truncation point
- `DidTimeOut` — `true` if search exceeded the timeout

## Filters

```csharp
// Value filter — equality match on a filterable field
// The out parameter carries the reason when the call returns null (unknown field,
// non-filterable field, type mismatch). Use `out var error` where you want to surface it.
Filter categoryFilter = engine.CreateValueFilter("category", "electronics", out _)!;

// Range filter — inclusive numeric range
Filter priceFilter = engine.CreateRangeFilter("price", 10.0, 100.0, out _)!;

// Combine with & (AND), | (OR), ! (NOT)
Filter combined  = categoryFilter & priceFilter;
Filter either    = categoryFilter | priceFilter;
Filter excluded  = !categoryFilter;

// Check match count
int count = combined.NumberOfDocumentsInFilter;

// Use in query
query.Filter = combined;
```

Preload filters for faster first search on large datasets:
```csharp
engine.LoadFilters(new[] { categoryFilter, priceFilter }, maxThreadCount: 2);
// Or: engine.LoadAllFilters(maxThreadCount: 4);
```

## Boosts

Boost results matching certain criteria without excluding non-matching results. Only affects results where coverage confirms a near-exact match. `CreateBoost` pre-calculates score adjustments for all matching documents.

```csharp
Filter yearFilter  = engine.CreateRangeFilter("year", 1980, 2025, out _)!;
Filter genreFilter = engine.CreateValueFilter("genre", "Documentary", out _)!;

var boosts = new List<Boost>();
boosts.Add(engine.CreateBoost(yearFilter & genreFilter, BoostStrength.Med));  // BoostStrength: Low, Med, High

query.Boosts = boostArray;
query.EnableBoost = true;
```

**Personalized boosting** — boost a user's own items by OR-ing per-key value filters into one filter, then boosting that filter:

```csharp
Filter? frequentPurchases = null;
foreach (long itemId in userFrequentItemIds)
{
    Filter f = engine.CreateValueFilter("item_id", itemId, out _)!;
    frequentPurchases = frequentPurchases is null ? f : frequentPurchases | f;
}
if (frequentPurchases is not null)
    boosts.Add(engine.CreateBoost(frequentPurchases, BoostStrength.Med));
```

## Dynamic Document Operations

Insert, update, and delete documents without rebuilding the index. The engine stays in Ready state throughout.

```csharp
// Insert single document
engine.InsertJsonRecord(jsonString, out string error);

// Insert multiple
engine.InsertJsonRecords(new[] { json1, json2 }, monitor: null, out error);
engine.InsertJsonRecords(jsonStream, monitor, out error);

// Update single (replaces entire document; must include the key field)
engine.UpdateJsonRecord(jsonString, out error);

// Update multiple
engine.UpdateJsonRecords(new[] { json1, json2 }, monitor: null, out error);

// Partial field update
engine.UpdateField(documentKey, "price", 49.99, out error);

// Batch field update on a filter
engine.UpdateFieldInFilter(priceFilter, "on_sale", true, out error);

// Delete by key
engine.DeleteJsonRecord(documentKey);

// Delete multiple
engine.DeleteJsonRecords(new[] { key1, key2 }, monitor: null);

// Delete all matching a filter
engine.DeleteRecordsInFilter(priceFilter);
```

A small `avgdl` drift accumulates over many incremental operations. A full `Index()` re-normalises it when needed.

## Coverage Control

Coverage re-evaluates the top-K pattern-match candidates for exact and near-exact token matches, promoting confirmed matches above pure pattern results.

```csharp
var cov = new CoverageSetup
{
    CoverWholeQuery    = true,    // default true — detect whole query as a string
    CoverWholeWords    = true,    // default true — detect individual words
    CoverFuzzyWords    = true,    // default true — edit-distance tolerance
    CoverJoinedWords   = true,    // default true — handle split/joined words
    CoverPrefixSuffix  = true,    // default true — detect partial words
    IncludePatternMatches = true, // default true — set false for exact-only results
    Truncate           = true,    // default true — cut results at coverage boundary
    TruncationScore    = 65024,   // default 65024
    TruncateWordHitLimit    = 1,  // default 1
    TruncateWordHitTolerance = 0, // default 0
    MinWordSize        = 2,       // default 2
    LevenshteinMaxWordSize = 20,  // default 20
};
query.CoverageSetup = cov;
```

## Common Patterns

**Near-exact hits only:**
```csharp
query.CoverageSetup = new CoverageSetup { IncludePatternMatches = false };
```

**Pattern matching only (disable coverage):**
```csharp
query.EnableCoverage = false;
```

**Keep all results, don't truncate:**
```csharp
query.CoverageSetup = new CoverageSetup { Truncate = false };
```

**Deep coverage (all documents):**
```csharp
query.CoverageDepth = engine.Status.DocumentCount;
```

**Empty search — browse mode with sorting and facets:**
```csharp
var query = new Query("", 50)
{
    SortBy = engine.GetField("rating"),
    EnableFacets = true
};
```

**Get min/max from facets for range filter UI:**
```csharp
var query = new Query("", 1) { EnableFacets = true };
var result = engine.Search(query);
if (result.Facets != null && result.Facets.TryGetValue("price", out var histogram))
{
    var values = histogram.Select(p => double.Parse(p.Key)).ToList();
    var priceRange = engine.CreateRangeFilter("price", values.Min(), values.Max(), out _);
}
```

## Licensing

- **No license**: 100,000 document limit
- **Extended license (free)**: Unlimited — register at [indx.co](https://indx.co)
- **Company license (paid)**: Unlimited + SLA and support

Pass the license path to the `SearchEngine` constructor, or place the `.license` file in the working directory.
