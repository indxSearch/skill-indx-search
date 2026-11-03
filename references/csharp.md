# C# NuGet Integration

## Install

```bash
dotnet add package IndxSearchLib
```

Targets **.NET 10.0**. Current version: **5.0.2**.

## SearchEngine Constructor

```csharp
// Default — works for most cases
var engine = new SearchEngine();

// With license file
var engine = new SearchEngine("indx-developer.license");

// With logging and a configuration (ConfigurationParameters.Default suits most cases)
var engine = new SearchEngine(logPrefix: "MyApp", factory: loggerFactory,
    configuration: ConfigurationParameters.Default, licenseFileName: "indx-developer.license");
```

## Basic Usage

```csharp
using Indx.Api;

var engine = new SearchEngine();

// 1. Analyze JSON structure
using var stream = File.OpenRead("products.json");
engine.Init(stream);

// 2. Configure fields
if (!engine.TrySetFieldConfiguration([
    new FieldProxy { FieldName = "name",        Searchable = true, Weight = 2.0f },
    new FieldProxy { FieldName = "description", Searchable = true, Weight = 1.0f },
    new FieldProxy { FieldName = "category",    Filterable = true, Facetable = true },
    new FieldProxy { FieldName = "price",       Filterable = true, Sortable = true },
], out string error))
    throw new InvalidOperationException(error);

// 3. Load and index
stream.Position = 0;
engine.Load(stream);
engine.Index();

// 4. Search
var result = engine.Search(new Query("wireless headphones", 20));
```

## SearchEngine Lifecycle

```
Init(stream) → TrySetFieldConfiguration(...) → Load(stream) → Index() → Search(query)
```

**Core methods:**

| Method | Description |
|--------|-------------|
| `Init(Stream)` / `Init(Stream, ProcessMonitor?)` | Analyze JSON structure, discover fields. Blocks if monitor is null |
| `GetFieldConfiguration()` | Returns `FieldProxy[]` of all discovered fields with their current settings |
| `TrySetFieldConfiguration(FieldProxy[], out string error)` | Apply field roles and weights. Returns `true` on success; `false` with the reason (such as an unknown field) in `error` |
| `TryValidateFieldConfiguration(FieldProxy[], out string error)` | Whether `TrySetFieldConfiguration` would accept the same argument, changing nothing. For applying the change later or on a clone |
| `SetFieldConfiguration(FieldProxy[])` | `[Obsolete]`: the same, returning null on success or the name of the first unknown field. Use `TrySetFieldConfiguration` |
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

`Init`, `Load` and `LoadAllFilters` dispatch to a background thread and return
immediately, with the monitor already running — safe to poll or wait on the next line.
`Index` runs on the calling thread. Pick one way to wait; they are alternatives, not steps.

```csharp
var monitor = new ProcessMonitor();
monitor.TimeoutSeconds = 120;

engine.Load(fstream, monitor);

// Either poll for progress...
while (monitor.IsRunning)
    Thread.Sleep(200);

// ...or wait, synchronously or asynchronously - not both.
// monitor.WaitForCompletion();
// await monitor.WaitForCompletionAsync();

if (!monitor.Succeeded)
    Console.WriteLine($"Error: {monitor.ErrorMessage}");
```

One monitor per operation; reusing one while it runs throws. `WaitForCompletionAsync` on a
monitor never given to an operation completes immediately — that means "nothing to wait
for", not "succeeded", so always check `Succeeded`.

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

`FieldProxy` properties: `FieldName`, `FieldType` ("String"/"Number"/"Boolean"), `IsArray`, `Searchable`, `Filterable`, `Facetable`, `Sortable`, `Embeddable`, `Weight`, `BM25b`, `BM25k1`, `PreloadFilters`, `PatternMatching` (on or off) with `IndexSizes`, `NumberOfStartCharacters` and `NumberOfStopCharacters` (the setup's, when on).

## Field Configuration

### Applying Configuration

Use `TrySetFieldConfiguration` to configure fields in a single call. Only set the properties that matter — null properties are left unchanged.
Every refusal (an unknown field, a negative weight, a vector field given another role) returns
`false` with the reason in `error`, and nothing is applied.

```csharp
engine.TrySetFieldConfiguration([
    new FieldProxy { FieldName = "title",       Searchable = true, Weight = 2.0f },
    new FieldProxy { FieldName = "description", Searchable = true, Weight = 1.0f },
    new FieldProxy { FieldName = "category",    Filterable = true, Facetable = true },
    new FieldProxy { FieldName = "price",       Filterable = true, Sortable = true },
    new FieldProxy { FieldName = "rating",      Sortable = true },
], out string error);
```

`Weight` is a `float`. Higher values increase the field's influence on BM25 scoring relative to other searchable fields. Typical range: 0.5–3.0.

`Filterable` and `Sortable` are refused on a field with no type: `TrySetFieldConfiguration` returns
`false` with the reason, and the setters on `Field` throw `ArgumentException`. A field
that was null in every document analyzed has no type, so a filter cannot choose between a numeric and
a categorical index and a sort cannot pick a comparator. `Searchable` and `Facetable` do not read the
type and stay available: setting `Searchable` on such a field prepares it, so a record inserted later
that does carry a value is indexed and found. Clearing either role is always allowed.

### Changing Configuration on a Loaded Engine

Ask before applying, because a change needs one of three things:

```csharp
var proposed = new[] { new FieldProxy { FieldName = "category", Facetable = true } };

if (engine.DocumentFields.RequiresReload(proposed))
{
    // Searchable, Embeddable, PreloadFilters.
    // Only a Load consumes these. Applying one in place returns the engine to Created,
    // and searches then fail with a reason rather than answering as if the field were empty.
}
else if (engine.DocumentFields.RequiresReindex(proposed))
{
    engine.TrySetFieldConfiguration(proposed, out string error);
    engine.Index();   // includes a field getting its FIRST role: its positions are built here
}
else
{
    engine.TrySetFieldConfiguration(proposed, out string error);   // takes effect at query time
}
```

The alternative to a reload, and what a server does to keep answering while it rebuilds, is
`CreateInMemoryClone(monitor, fieldOverrides)`: the clone applies the change before it loads, then
you swap it in.

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
engine.TrySetFieldConfiguration([...], out string error);
engine.SaveFieldConfiguration("fieldconfig.json");

// Subsequent runs: load config (skip Init)
engine.LoadFieldConfiguration("fieldconfig.json");
```

## Query Object

```csharp
var query = new Query("search text", maxResults)
{
    Mode = SearchMode.EditDistance,         // default: EditDistance (or Bm25f)
    Match = { Tolerance = Tolerance.TwoEdits, Rungs = Rungs.Fewest },     // what counts as a hit
    Candidates = { CandidatesPerRow = 4, MaxCandidates = null, Fallback = true, Rest = false }, // who comes in
    Ranking = {                             // the order
        Rerank = true,                      // default: Coverage reranks the candidates, in either mode
        Share = 0.2f,                       // the reranker's share of the blend
        Boosts = boostArray,                // Boost[]; a list that is not empty is on
        FieldBoosts = new Dictionary<string, float> { ["title"] = 2.0f }, // BM25F per-query boost
    },
    EnableFacets = false,                   // default: false
    SortBy = engine.GetField("rating"),     // Field object, default: null
    SortAscending = false,                  // default: false
    RemoveDuplicates = true,                // default: true
    Filter = filter,                        // Filter object, default: null
    TimeOutLimitMilliseconds = 1000         // default: 1000, max: 10000
};
```

The three parts are never null. Until 6.0 they were `CoverageSetup`, `WordLookupSetup`,
`CoverageDepth`, `EnableBoost`/`EnableBury` and members of the query itself.

`Ranking.FieldBoosts` — per-query multipliers on each field's `Weight` for the BM25F scoring path (`Weight = 3.0f` × boost `2.0f` = 6×; fields not listed default to 1.0, i.e. their `Weight`). Only has effect when `ScoringMode` is `BM25F` — every searchable field has the same `BM25k1`. **Every key must be a searchable field**: an unknown or non-searchable key makes `Search` return an empty `Result` with no `Reason`, indistinguishable from no matches; the message is on `engine.Status.ErrorMessage`.

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
- `Facets` — `Dictionary<string, KeyValuePair<string, int>[]>` (field → value/count pairs), counted
  over every record the search counted, not only those returned. A value is counted
  case-insensitively, as a value filter compares, and shown as first written, so a corpus that spells
  it both ways lists it once, and a default value filter on it returns the facet's count
- `ToleranceCounts` — how many documents hold every query word `Exact`ly, within `OneEdit`, within
  `TwoEdits` (each with those before it; a rung not worked out is null), and with `Candidates.Rest`
  the `Rest`: the counted rung and the related documents taken below it. Null in `Bm25f` mode
- `CountedTolerance` — the rung the list was taken from; null when nothing was counted
- `Count` — how many documents answer, in every mode: the rung counted (with `Candidates.Rest` the
  rest too), the fallback's most-words documents, every document with a score in `Bm25f`, every
  document the filter lets through in an empty search
- `Ranked` — how many were ranked, at most the candidate bound. `Count > Ranked`: the bound was too
  small; `Ranked > Records.Length`: more were ranked than the rows asked for
- `IsFallback` — no document held every word: the most words first, then pattern matches
- `DidTimeOut` — `true` when the search could not be **served**, rather than served and found nothing.
  Two causes: it exceeded the timeout, or the engine was not `Ready` (loading, indexing, hibernated).
  Check it before showing "no results" — a dataset being reloaded would otherwise look like a dataset
  with nothing in it. A mistake in the call (null query, empty text with facets off) returns an empty
  result *without* the flag.

## Filters

```csharp
// THE FIELD'S TYPE DECIDES THE FILTER KIND. A range filter needs a field whose values
// are numbers; a value filter needs a field whose values are not. The other way round is
// refused -- a value filter compares text, so on a number it missed 129.0 when asked for
// 129. For an exact match on a numeric field use a range with EQUAL LIMITS, which
// compares as numbers and reads the field's index. A number stored as a JSON *string* is
// a text field, so it takes a value filter and refuses a range.
//
// The out parameter carries the reason when the call returns null (unknown field,
// non-filterable field, wrong kind for the type, unparseable bound or culture) and names
// the alternative. Null means REJECTED, never "no filter": passing it on widens the
// search to the whole dataset. Use `out var error` where you want to surface it.

// Value filter — equality match on a filterable, non-numeric field
Filter categoryFilter = engine.CreateValueFilter("category", "electronics", out _)!;

// Case-sensitive variant. A facet count equals a filter count only for a case-sensitive
// filter, and this one is answered by walking the documents rather than by the index.
Filter exact = engine.CreateValueFilter("category", "Electronics", isCaseSensitive: true, out _)!;

// Range filter — inclusive numeric range
Filter priceFilter = engine.CreateRangeFilter("price", 10.0, 100.0, out _)!;

// Equal limits are numeric equality — how to match one value on a numeric field
Filter oneItem = engine.CreateRangeFilter("item_id", 129, 129, out _)!;

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
boosts.Add(engine.CreateBoost(yearFilter & genreFilter, BoostStrength.Med, out _)!);  // BoostStrength: Low, Med, High

query.Ranking.Boosts = boosts.ToArray(); // a list that is not empty is on
```

**Personalized boosting** — boost a user's own items by OR-ing per-key value filters into one filter, then boosting that filter:

```csharp
Filter? frequentPurchases = null;
foreach (long itemId in userFrequentItemIds)
{
    // item_id holds numbers: a range with equal limits, not a value filter
    Filter f = engine.CreateRangeFilter("item_id", itemId, itemId, out _)!;
    frequentPurchases = frequentPurchases is null ? f : frequentPurchases | f;
}
if (frequentPurchases is not null)
    boosts.Add(engine.CreateBoost(frequentPurchases, BoostStrength.Med, out _)!);
```

## Synonyms

Attach a `SynonymList` to widen searches: when a query matches an entry, the entry's terms are appended to the query text before scoring. Search-time only — nothing is re-indexed, and the engine works on its own copy of the query, so the caller's `Query` is never modified.

```csharp
engine.SynonymList = new SynonymList
{
    Entries =
    [
        // Multidirectional: all terms equivalent — matching any of them pulls in the group.
        new SynonymEntry { Direction = SynonymDirection.Multidirectional, Terms = ["geriatri", "geriatrisk", "geriatriske"] },
        // OneWay: only Source expands, into Terms (acronyms) — matching a term does NOT pull in Source.
        new SynonymEntry { Direction = SynonymDirection.OneWay, Source = "hms", Terms = ["helse, miljø og sikkerhet"] },
    ]
};

engine.SaveSynonyms("synonyms.json");   // serialize the current list to a file
engine.LoadSynonyms("synonyms.json");   // load and apply a saved list
engine.SynonymList = null;              // search without synonyms again
```

- Multi-word terms are matched as whole phrases, and matching splits on the engine's own delimiters, so punctuation (`geriatri,`) still triggers an entry.
- Expansion lengthens the query text, which lowers Coverage scores proportionally — measure the net effect before shipping a large list.
- After mutating `Entries` in place, call `SynonymList.Invalidate()`; assigning a new list or a new `Entries` collection handles it automatically.

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

## Vector Fields

Mark an array-of-numbers field `Embeddable` and load the vectors with the documents. A vector
field has no other role. After `Index()`, search it through `engine.EmbeddingFields["embedding"]`
(`IEmbeddingIndex.Search(vector, maxResults)`), or merge with a text result through
`IEmbeddingIndex.MergeHybrid`.

- **Any length is accepted.** Each vector is brought to unit length when stored and when searched
  for, so a score is the cosine similarity.
- **Insert, update and delete work** on a dataset with vector fields. A removed vector leaves a
  node behind in the graph; the graph is rebuilt in the background once enough have gathered
  (`EmbeddingSetup.AutomaticRebuild`, on by default). `IEmbeddingIndex.RebuildDue` reports it when
  automatic rebuilds are off, and `Index()` rebuilds.
- **Save the graph to skip building it.** Building is the slow part of loading vectors.
  `SaveEmbeddings(path, out error)` writes every vector field's graph; `LoadEmbeddings(path, out
  error)`, called after the field configuration is set and before `Load`, has that `Load` take the
  saved graph instead of building one. The file need not match the documents exactly: the graph is
  brought in line with what is loaded, or built as usual when more than a fifth differs.
  `IEmbeddingIndex.RestoredFromSaved` says which happened.
- **Memory.** `ConfigurationParameters.EmbeddingSetup` decides how vectors are held: `GraphVectors`
  (`VectorStorage.Int16`, the default, or `Float32`) for the search graph, and `DocumentVectors`
  (`Float32` default, `Int16`, `Text`, or `NotKept`) for the document's own copy. `Text` returns
  every document character for character; with `Float32` a number may come back spelled
  differently (`0.10` as `0.1`).

```csharp
var config = ConfigurationParameters.Default.With(
    embeddingSetup: EmbeddingSetup.Default.With(documentVectors: DocumentVectorStorage.Text));
```

## Coverage Control

The search counts the documents holding every query word (exactly, else within one edit, else
two) and Coverage ranks them, blended with pattern recognition. Whole words and a word as the start
or end of a longer one are always detected; the shortest word is `TokenizerSetup.MinimumWordSize`.

```csharp
var query = new Query("search text", 20)
{
    Match = {
        Tolerance = Tolerance.TwoEdits,  // default — the fields' own Field.Tolerance; Exact, OneEdit cap it
        OneEditFrom = 3, TwoEditsFrom = 6,
        JoinedWords = true,              // default — words written together or apart
    },
    Candidates = { Fallback = true },    // default — false for exact-only results
    Ranking = { Share = 0.2f, WholeQueryFloor = true },
};
```

## Common Patterns

**Near-exact hits only:**
```csharp
query.Candidates.Fallback = false;
```

**Pattern matching only (no Coverage):**
```csharp
query.Mode = SearchMode.Bm25f;
query.Ranking.Rerank = false;
```

**Pattern matching, reranked by Coverage (no word count):**
```csharp
query.Mode = SearchMode.Bm25f;   // Rerank is true by default
```

**Words within an edit, exact first, with a count, no Coverage:**
```csharp
query.Mode = SearchMode.EditDistance;   // the default
query.Ranking.Rerank = false;
```

**Every rung up to the tolerance, not only the first with hits:**
```csharp
query.Match.Rungs = Rungs.All;
```

**Rank more candidates:**
```csharp
query.Candidates.MaxCandidates = 5000;
```
The count (`Result.ToleranceCounts`) is never capped; the bound decides how many of the counted
documents are ranked (`Result.Count` above `Result.Ranked` when it was reached). Cost is linear in the candidates.

**Empty search — browse mode with sorting and facets.** `EnableFacets` is what makes empty text
legal: without it the search is refused, returning an empty `Result` with a `Reason`, which is not the
same as matching nothing.

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
