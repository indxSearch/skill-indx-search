# C# NuGet Integration

## Install

```bash
dotnet add package IndxSearchLib --prerelease
```

Targets **.NET 10.0**. This page is for **IndxSearchLib 6.0** (while 6.0 is in preview, `--prerelease` or an explicit version). On 5.0.x, see [migration-v5-to-v6.md](migration-v5-to-v6.md) before using anything here.

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
    new FieldProxy { FieldName = "name",        WordMatching = true, PatternMatching = true, Weight = 2.0f },
    new FieldProxy { FieldName = "description", WordMatching = true, PatternMatching = true },
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
| `new SearchEngine(license, filterCacheMB: 256)` | How many megabytes the filters kept between searches may take; the least recently used go past it, `0` keeps none. `Status.FilterCacheFilters`, `FilterCacheBytes`, `FilterCacheLimitBytes` say what they take |
| `Hibernate(out string)` | Dispose indexed data to free memory. Searches will timeout. Filters remain usable. Runs no garbage collection: call `GC.Collect(2, GCCollectionMode.Aggressive, true, true)` when it suits, to give the memory back. It stops every thread of the process while it runs, which on a large heap can be over a second: not while searches are served |
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

`FieldProxy` properties: `FieldName`, `FieldType` ("String"/"Number"/"Boolean"), `IsArray`, `Optional`, `SampleValue`, the roles `WordMatching`, `PatternMatching` (on or off, with `IndexSizes`, `NumberOfStartCharacters` and `NumberOfStopCharacters`, the setup's when on), `Filterable`, `Facetable`, `Sortable`, `Embeddable`, and `Weight`, `BM25b`, `BM25k1`, `Tolerance`, `RerankWeight`, `TypingPrefixFrom`, `CaseSensitive`. A null property on a set call leaves the value as it is.

**A configuration from `Init` searches nothing**: every field starts with neither `WordMatching` nor `PatternMatching`. Give at least one field `WordMatching`, or no search narrows.

## Field Configuration

### Applying Configuration

Use `TrySetFieldConfiguration` to configure fields in a single call. Only set the properties that matter — null properties are left unchanged.
Every refusal (an unknown field, a negative weight, a vector field given another role) returns
`false` with the reason in `error`, and nothing is applied.

```csharp
engine.TrySetFieldConfiguration([
    new FieldProxy { FieldName = "title",       WordMatching = true, PatternMatching = true, Weight = 2.0f },
    new FieldProxy { FieldName = "description", WordMatching = true, PatternMatching = true },
    new FieldProxy { FieldName = "sku",         WordMatching = true, Tolerance = Tolerance.Exact },
    new FieldProxy { FieldName = "category",    Filterable = true, Facetable = true },
    new FieldProxy { FieldName = "price",       Filterable = true, Sortable = true },
    new FieldProxy { FieldName = "rating",      Sortable = true },
], out string error);
```

How a searched field is matched:

| Setting | What it does |
|---|---|
| `WordMatching = true` | Matched by its whole words, exactly and within the edits its `Tolerance` allows; its words are in the word lookup, so the field **counts**. Every word from `TokenizerSetup.MinimumWordSize` (2) is indexed ("A4", "XL") |
| `PatternMatching` | Matched inside its words: on a `FieldProxy` true or false (true takes `PatternMatchingSetup.Default` unless `IndexSizes` and the start and stop characters say otherwise); on a `Field` a `PatternMatchingSetup?`. `Default` is sizes `[3]`, two start characters, no stop character. `new PatternMatchingSetup([2, 3])` matches short values (tags, codes) by two characters; a stop character anchors the end. Alone, a field orders records but cannot narrow them |
| `Tolerance` | The most edits the field allows: `Exact`, `OneEdit` (default), `TwoEdits`, `ThreeEdits`. Capped by the search's `Match.Tolerance` |
| `RerankWeight` | The field's weight in the reranker; 0 leaves it out of the reranker and the word lookup (it still counts through BM25F) |
| `TypingPrefixFrom` | From how many letters the word being typed matches the start of a longer word in this field |
| `CaseSensitive` | Matched with case |

Both `WordMatching` and `PatternMatching` is the usual choice for titles, names and descriptions, and is how every searchable field was matched in 5.0. `WordMatching` alone suits codes and identifiers.

`Weight` is a `float` (default 1.0). Higher values increase the field's influence on BM25F scoring relative to other searched fields. Typical range: 0.5 to 3.0.

`Filterable` and `Sortable` are refused on a field with no type: `TrySetFieldConfiguration` returns
`false` with the reason, and the setters on `Field` throw `ArgumentException`. A field
that was null in every document analyzed has no type, so a filter cannot choose between a numeric and
a categorical index and a sort cannot pick a comparator. The matching roles and `Facetable` do not
read the type and stay available: setting `WordMatching` on such a field prepares it, so a record
inserted later that does carry a value is indexed and found. Clearing a role is always allowed.

### Changing Configuration on a Loaded Engine

Ask before applying, because a change needs one of three things:

```csharp
var proposed = new[] { new FieldProxy { FieldName = "category", Facetable = true } };

if (engine.DocumentFields.RequiresReload(proposed))
{
    // Whether a field is searched at all (WordMatching or PatternMatching on or off),
    // Embeddable. Only a Load consumes these. Applying one in place returns the engine to Created,
    // and searches then fail with a reason rather than answering as if the field were empty.
}
else if (engine.DocumentFields.RequiresReindex(proposed))
{
    engine.TrySetFieldConfiguration(proposed, out string error);
    engine.Index();   // which way a searched field is matched, a k1 change, a field's FIRST role
}
else
{
    engine.TrySetFieldConfiguration(proposed, out string error);   // takes effect at query time
}
```

The alternative to a reload, and what a server does to keep answering while it rebuilds, is
`CreateInMemoryClone(monitor, fieldOverrides)`: the clone applies the change before it loads, then
you swap it in.

### Scoring Mode and BM25 Tuning (Advanced)

The scoring mode is the engine's, chosen in the configuration with `ScoringSetup`:
`ScoringMode.BM25F`, the default, one index across the fields with one k1 (`BM25FK1`, 1.2), or
`ScoringMode.BM25PerField`, one BM25F index per field with each field's own k1 and b, merged by
`Weight`. Differing `BM25k1` on the fields no longer chooses the mode. `engine.Scoring` says which.

```csharp
var config = ConfigurationParameters.Default.With(
    scoringSetup: new ScoringSetup(ScoringMode.BM25PerField));
var engine = new SearchEngine(null, null, config, "indx-developer.license");

// per field: b always, k1 only in BM25PerField
new FieldProxy { FieldName = "title",       WordMatching = true, BM25b = 0.5f,  BM25k1 = 1.5f },
new FieldProxy { FieldName = "description", WordMatching = true, BM25b = 0.75f, BM25k1 = 1.2f },
```

`BM25b` controls length normalization (range 0 to 1, default 0.75). `BM25k1` controls term frequency
saturation (range 1.0 to 2.0, default 1.2). In BM25F a field's `BM25k1` is ignored
(`Status.FieldK1Ignored` says so) and `TrySetFieldConfiguration` refuses a changed value; set
`ScoringSetup.BM25FK1` instead. A k1 change is a reindex.

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
        Rerank = true,                      // default: the reranker orders the candidates, in either mode
        Share = 0.2f,                       // the reranker's share of the blend
        Boosts = boostArray,                // Boost[]; a list that is not empty is on
        Buries = buryArray,                 // Bury[]; the same, scored lower
        FieldBoosts = new Dictionary<string, float> { ["title"] = 2.0f }, // BM25F per-query boost
    },
    EnableFacets = false,                   // default: false
    SortBy = engine.GetField("rating"),     // Field object, default: null
    SortAscending = false,                  // default: false
    Filter = filter,                        // Filter object, default: null
    TimeOutLimitMilliseconds = 1000         // default: 1000, max: 10000
};
```

The three parts are never null. On 5.0 their settings were `CoverageSetup`, `CoverageDepth`,
`EnableBoost` and members of the query itself; [migration-v5-to-v6.md](migration-v5-to-v6.md) maps
each.

`Ranking.FieldBoosts`: per-query multipliers on each field's `Weight` (`Weight = 3.0f` × boost `2.0f` = 6×; fields not listed keep their `Weight`). Works in both scoring modes. **Every key must be a searched field**: an unknown key, or a field no longer searched, makes `Search` return an empty `Result` with its `Reason` naming the field. `Ranking.FieldWeights` is the reranker's counterpart, a field's reranking weight for this search.

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
- `Records` — `ScoreEntry16[]` with `DocumentKey` (long), `Score` (ushort, 0–65535) and, after 6.0.0-rc1, `Source` (`RecordSource`: `Counted`, `Fill`, `MostWords`, `Rest`), in rung order (exact before one edit before two), reranked inside each rung.
- `Facets` — `Dictionary<string, KeyValuePair<string, int>[]>` (field → value/count pairs), counted
  over every record the search counted, not only those returned. A value is counted
  case-insensitively, as a value filter compares, and shown as first written, so a corpus that spells
  it both ways lists it once, and a default value filter on it returns the facet's count
- `ToleranceCounts` — how many documents hold every query word `Exact`ly, within `OneEdit`, within
  `TwoEdits`, within `ThreeEdits` (each with those before it; a rung not worked out is null), with
  `Candidates.Rest` the `Rest`, and `ThroughSynonym`, the records holding a word only through a
  synonym. Null in `Bm25f` mode
- `CountedTolerance` — the rung the list was taken from; null when nothing was counted
- `Count` — how many documents answer, in every mode: the rung counted (with `Candidates.Rest` the
  rest too), the fallback's most-words documents, every document with a score in `Bm25f`, every
  document the filter lets through in an empty search
- `Ranked` — how many were ranked, at most the candidate bound. `Count > Ranked`: the bound was too
  small; `Ranked > Records.Length`: more were ranked than the rows asked for
- `IsFallback` — no document held every word: the most words first, then pattern matches
- `Fill` (`PageFill`: `NextRungs`, `MostWords`) — how many listed records came from
  `Candidates.FillReach` and `FillWithMostWords`, below the counted ones
- `Reranked` — whether the reranker ordered the list
- `MatchReport` — with `Query.EnableMatchReport`: for each returned record how each query word was
  met (`Whole`, `Prefix`, `Suffix`, `Joined`, `RunTogether`, `OneEdit`, `TwoEdits`, `ThreeEdits`,
  `Synonym`, `NotFound`), in which field it counted, and in `Fields` every field that met it; for
  `Synonym`, after 6.0.0-rc1, `Synonym` is the one the record holds (null, as the field, in rc1)
- `Reason` — set when the search was refused (a bad request, a disposed engine), null on a normal
  search including one that matched nothing
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
// non-filterable field, wrong kind for the type, unparseable bound) and names the
// alternative. Null means REJECTED, never "no filter": passing it on widens the
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

// One of a set of values: one filter, far faster than an OR of value filters
// (100 tags on 500 000 documents: 0.1 s as IN, 5.7 s as an OR). Numbers compare as numbers.
Filter brands = engine.CreateInFilter("brand", ["nike", "puma", "adidas"], out _)!;

// Within a distance, in metres, of a position. The field is an object {"latitude": 59.91,
// "longitude": 10.75}; mark location.latitude and location.longitude Filterable
Filter nearby = engine.CreatePositionFilter("location", new GeoPosition(59.91f, 10.75f), 5000, out _)!;

// Dates on a field holding Unix time
Filter recent = engine.CreateDateRangeFilter("published", DateTime.UtcNow.AddDays(-30), DateTime.UtcNow, out _)!;

// Combine with & (AND), | (OR), ! (NOT), or Filter.And / Filter.Or over any number
Filter combined  = categoryFilter & priceFilter;
Filter either    = categoryFilter | priceFilter;
Filter excluded  = !categoryFilter;
Filter all       = Filter.And(categoryFilter, priceFilter, brands);

// Or the whole condition as one expression. $name takes a filter you made; MATCHCASE matches
// case; within(field, lat, lon, meters) is the position filter. Null with the errors when
// the expression makes no filter, each with its kind, message and place.
Filter? shop = engine.CreateFilter(
    "brand IN [\"nike\", \"puma\"] AND price BETWEEN 100 AND 2000 AND NOT category = \"outlet\"",
    out IReadOnlyList<FilterExpressionError> errors);
string expression = combined.ToExpression();   // any filter but a custom one, back as text

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

Boost results matching certain criteria without excluding non-matching results. A boost adds 257 × its strength to the score of each document its filter holds, so it reorders records that score close together within a rung; it never moves a record across rungs. `CreateBoost` returns `Boost?`: null with the reason in `error` for a null filter or an unknown strength. The strengths of one search's boosts may sum to at most 255; past it the search is refused with a `Reason`.

```csharp
Filter yearFilter  = engine.CreateRangeFilter("year", 1980, 2025, out _)!;
Filter genreFilter = engine.CreateValueFilter("genre", "Documentary", out _)!;

var boosts = new List<Boost>();
boosts.Add(engine.CreateBoost(yearFilter & genreFilter, BoostStrength.Med, out _)!);  // BoostStrength: Low, Med, High

query.Ranking.Boosts = boosts.ToArray(); // a list that is not empty is on
```

**Personalized boosting**: boost a user's own items with one IN filter over their keys:

```csharp
Filter? frequentPurchases = engine.CreateInFilter("item_id", userFrequentItemIds.Cast<object>(), out _);
if (frequentPurchases is not null)
    boosts.Add(engine.CreateBoost(frequentPurchases, BoostStrength.Med, out _)!);
```

**Bury**: the counterpart. A bury scores the documents its filter holds lower by its strength
(`BuryStrength`: Low, Med, High) and moves nothing else; a buried document still appears, below the
rest. Use it for out-of-stock items that should stay findable.

```csharp
Filter outOfStock = engine.CreateRangeFilter("stock", 0, 0, out _)!;
query.Ranking.Buries = [engine.CreateBury(outOfStock, BuryStrength.High, out _)!];
```

## Synonyms

Attach a `SynonymList` to widen searches: a query word, or run of words, that matches an entry is held through the words as typed **or** any of the entry's terms. A synonym therefore widens the counted set ("war film" with *film ↔ movie* also counts the war movies), and a record holding a word only through a synonym ranks below those holding the typed word on its rung (`ToleranceCounts.ThroughSynonym`). Search-time only: nothing is re-indexed, and the caller's `Query` is never modified.

```csharp
engine.SynonymList = new SynonymList
{
    Entries =
    [
        // Multidirectional: all terms equivalent — matching any of them pulls in the group.
        new SynonymEntry { Direction = SynonymDirection.Multidirectional, Terms = ["legemiddel", "medisin"] },
        // OneWay: only Source expands, into Terms (acronyms) — matching a term does NOT pull in Source.
        new SynonymEntry { Direction = SynonymDirection.OneWay, Source = "hms", Terms = ["helse, miljø og sikkerhet"] },
    ]
};

engine.SaveSynonyms("synonyms.json");   // serialize the current list to a file
engine.LoadSynonyms("synonyms.json");   // load and apply a saved list
engine.SynonymList = null;              // search without synonyms again
```

- Multi-word terms are matched as whole phrases, and matching splits on the engine's own delimiters, so punctuation (`medisin,`) still triggers an entry.
- The reranker reads the query as typed, so a list does not dilute the typed words' scores. Do not add typos, spelling variants or translations: the matcher finds misspellings itself.
- After mutating `Entries` in place, call `SynonymList.Invalidate()`; assigning a new list or a new `Entries` collection handles it automatically.

## Dynamic Document Operations

Insert, update, and delete documents without rebuilding the index. The engine stays in Ready state throughout.

```csharp
// Insert single document
engine.InsertJsonRecord(jsonString, out string error);

// Insert multiple
engine.InsertJsonRecords(new[] { json1, json2 }, monitor: null, out error);
engine.InsertJsonRecords(jsonStream, monitor, out error);

// Keys are text: the key field's value ("42", "sku-17"). A result's ScoreEntry16.Key is it;
// GetJsonDataOfKey, DeleteJsonRecord and UpdateField take it, or the long DocumentKey.
string json = engine.GetJsonDataOfKey("sku-17");
engine.DeleteJsonRecord("sku-17");

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

## Match Control

The search counts the documents holding every query word (exactly, else within one edit, else
two) and the reranker orders them, blended with BM25F. A finished word is matched whole; the word
being typed (the one `Match.CursorAt` is in, else the last word when no delimiter follows it) is also
matched as the start of a longer word, from `TypingPrefixFrom` letters. The shortest word is
`TokenizerSetup.MinimumWordSize` (2). Compound words are linked to their parts when the index is
built (`CompoundSetup` in the configuration, on by default): "ost" finds "brunost", and
"hvitløksmør" finds records holding "hvitløk smør".

```csharp
var query = new Query("search text", 20)
{
    Match = {
        Tolerance = Tolerance.TwoEdits,  // default — the fields' own Field.Tolerance; Exact, OneEdit cap it
        OneEditFrom = 3, TwoEditsFrom = 6, ThreeEditsFrom = 9,   // letters a word needs for each
        CursorAt = text.Length,          // the caret, for search-as-you-type
        JoinedWords = true,              // default — words written together or apart
    },
    Candidates = {
        Fallback = true,                 // default — false for exact-only results
        FillReach = 1,                   // a short page is filled from the next rung (default 0)
    },
    Ranking = { Share = 0.2f, WholeQueryFloor = true },
};
```

## Common Patterns

**Near-exact hits only:**
```csharp
query.Candidates.Fallback = false;
```

**BM25F only (no word count, no reranker):**
```csharp
query.Mode = SearchMode.Bm25f;
query.Ranking.Rerank = false;
```

**BM25F's best, reranked (no word count):**
```csharp
query.Mode = SearchMode.Bm25f;   // Rerank is true by default
```

**Words within an edit, exact first, with a count, in BM25F order:**
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
