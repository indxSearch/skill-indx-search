# Migrating Indx v5 → v6

Use this when a project is on **Indx 5** and moves to **Indx 6**:

| | Indx 5 | Indx 6 |
|---|---|---|
| C# library | `IndxSearchLib` 5.0.x | `IndxSearchLib` 6.0 |
| Indx server | 2.0.0-beta.1 to beta.3 | 2.0.0-beta.4 and later (same routes, `/api/teams/…`) |
| npm | `@indxsearch/indx-types` 3.x, `@indxsearch/intrface` 3.x | 4.0 of both |

Every name below that is marked as Indx 5 is in IndxSearchLib 5.0.6, the last 5.0 release, so
the guide covers any 5.0.x project. v6 is a breaking release on purpose, without compatibility
layers: an old name gives a compile error, and over HTTP an old member is ignored. Every change is
listed with what to write instead.

What changes, in order of how much code it touches:

1. **The query.** Coverage, its depth and its truncation are gone. A search's settings live in
   three parts (`Match`, `Candidates`, `Ranking`), and *how records are fetched* (`Query.Mode`) is a
   separate choice from *whether they are reranked* (`Ranking.Rerank`).
2. **The result.** It says how many records answer (`Count`) and how they spread over the typo
   levels (`ToleranceCounts`), in place of a truncation index. The list is what was counted.
3. **Fields.** `Searchable` is two roles: matched by whole words (`WordMatching`) and matched
   inside words (`PatternMatching`). The index sizes move from the configuration to the field.
4. **The configuration.** `WordMatcherSetup` is gone (fuzzy matching is set per search), and the
   scoring mode is chosen instead of inferred.

Loading, documents, filters, facets, synonym lists, vector search and the engine's lifecycle keep
their calls. Lists change by design: re-judge relevance after upgrading, and do not diff v6 lists
against v5 lists as a regression test.

---

## 1. Determine the version

- **C# / NuGet:** the `IndxSearchLib` version in the project file. On v6 `Query.Match`,
  `Query.Candidates` and `Query.Ranking` exist and `Query.EnableCoverage` does not.
- **HTTP:** a v6 server answers a search with `count`, `ranked` and `toleranceCounts` and without
  `truncationIndex`, and writes enums by name (`"systemState": "Ready"`, where v5 wrote `4`).
- **npm:** `@indxsearch/indx-types` and `@indxsearch/intrface` 4.0 and later speak v6; 3.x speaks v5.

If the project is on v5 and the user does not want to upgrade, write v5 code: do not apply
anything below to it.

---

## 2. The query

### 2a. Fetching and reranking are two choices

| Indx 5 | Indx 6 |
|----|----|
| `Query.EnableCoverage = true` (the default) | nothing to write: `Mode = SearchMode.EditDistance` and `Ranking.Rerank = true` are the defaults |
| `Query.EnableCoverage = false` | `Mode = SearchMode.Bm25f` **and** `Ranking.Rerank = false` |
| (new) | `Mode = SearchMode.Bm25f`, `Rerank = true`: BM25F's best, reranked |
| (new) | `Mode = SearchMode.EditDistance`, `Rerank = false`: the counted records in BM25F order |

`SearchMode.Bm25f` alone leaves `Rerank` on, so code that meant "Coverage off" must say both:

```csharp
var query = new Query(text, 20) { Mode = SearchMode.Bm25f, Ranking = { Rerank = false } };
```

`EditDistance` is the counting search. Each query word is looked up exactly, then within one edit,
then two; the first of those levels that finds records holding every word is the result, counted,
faceted and listed alike. `Bm25f` ranks by relevance and does not count. The reranker (called
Coverage in v5) orders the records inside either.

### 2b. Where each Indx 5 setting went

Every search setting that is not the query's own (`Text`, `Mode`, `Filter`, `SortBy`,
`EnableFacets`, the rows, the timeout) is on one of three parts, each never null: `Query.Match`
(`MatchSetup`, what counts as a hit), `Query.Candidates` (`CandidateSetup`, which records come in)
and `Query.Ranking` (`RankingSetup`, their order).

| Indx 5 | Indx 6 |
|----|----|
| `Query.CoverageDepth` (500 by default) | `Candidates.MaxCandidates` (`int?`). Null, the default, is the rows × `Candidates.CandidatesPerRow` (4), at least 100 and at most 5 000. Set it to keep a depth you relied on |
| `Query.CoverageSetup` | gone; its members are below |
| `CoverageSetup.IncludePatternMatches` | `Candidates.Fallback` (what is listed when no record holds every word) |
| `CoverageSetup.CoverJoinedWords` | `Match.JoinedWords` (on by default) |
| `CoverageSetup.CoverWholeQuery` | `Ranking.WholeQueryFloor` |
| `CoverageSetup.CoverFuzzyWords = false` | `Match.Tolerance = Tolerance.Exact` |
| `CoverageSetup.MinWordSize` | `TokenizerSetup.MinimumWordSize` (2), in the configuration |
| `CoverageSetup.LevenshteinMaxWordSize` | gone: how many edits a word may have is `Match.OneEditFrom` (3 letters), `TwoEditsFrom` (6), `ThreeEditsFrom` (9) |
| `CoverageSetup.Truncate`, `TruncationScore`, `TruncateWordHitLimit`, `TruncateWordHitTolerance`, the `TruncationStrictness` enum | gone: nothing is cut from the list (section 3) |
| `CoverageSetup.CoverWholeWords`, `CoverPrefixSuffix` | gone: always as their defaults were |
| `Query.Boosts` with `EnableBoost = true` | `Ranking.Boosts` (`Boost[]`); a list that is not empty is on |
| `Query.EnableBoost` | gone |
| `Query.MaxBoost` | gone. The strengths of a search's boosts may sum to at most 255; past it the search is refused with a `Result.Reason` |
| `Query.FieldBoosts` | `Ranking.FieldBoosts` |

```csharp
// Indx 5
var q = new Query("melk", 20) { CoverageDepth = 1000, EnableBoost = true, Boosts = [boost] };
q.CoverageSetup.CoverFuzzyWords = false;
q.CoverageSetup.IncludePatternMatches = false;

// Indx 6
var q = new Query("melk", 20)
{
    Match = { Tolerance = Tolerance.Exact },
    Candidates = { MaxCandidates = 1000, Fallback = false },
    Ranking = { Boosts = [boost] },
};
```

New settings, all optional: `Match.Tolerance` (`Exact`, `OneEdit`, `TwoEdits`, the default,
`ThreeEdits`), `Match.Rungs` (`Fewest`, the default, stops at the first level with hits; `All`
counts every level), `Match.CursorAt` and `TypingPrefixFrom` for search-as-you-type,
`Candidates.Rest`, `FillReach`, `FillWithMostWords` and `IndexTerms`, `Ranking.Share`,
`FieldWeights` and `Buries` (the counterpart of boosts, `ISearchEngine.CreateBury`), and
`Query.EnableMatchReport` and `MeasureFit`. [csharp.md](csharp.md) describes each.

---

## 3. The result

| Indx 5 | Indx 6 |
|----|----|
| `Result.TruncationIndex`, `TruncationScore` | gone: the list is what was counted, cut only to the rows |
| (new) | `Result.Count`: how many records answer (the level counted; in `Bm25f` the records with a score) |
| (new) | `Result.Ranked`: how many were ranked. `Count > Ranked` means the candidate bound was too small |
| (new) | `Result.ToleranceCounts` (`Exact`, `OneEdit`, `TwoEdits`, `ThreeEdits`, `Rest`, `ThroughSynonym`), each level with those below it; null in `Bm25f` |
| (new) | `Result.CountedTolerance`: the level the list was taken from |
| (new) | `Result.IsFallback`: no record held every word; the list is the records holding the most of them |
| (new) | `Result.Reranked`, `Result.Fill`, `Result.MatchReport` (with `Query.EnableMatchReport`) |

A client that showed "N hits" or paged with the truncation index reads `Count`, and can split it by
level: "140 hits, 45 of them with a typo" from `ToleranceCounts`.

### 3a. What the list is (compiles either way)

- In `EditDistance` the list is **the counted records and nothing else**. A record that does not
  hold every query word within the tolerance is not listed, where v5 listed pattern matches below
  the good ones and cut the list with truncation. `Candidates.Rest = true` adds BM25F's best of the
  others below the counted ones.
- **Facets count what the search counted**, every record of the level, not only the rows returned,
  and fold case: `Fiction` and `FICTION` are one entry, shown as first written. Count, facets and
  list agree.
- When nothing is counted, the fallback lists the records holding the most of the words
  (`IsFallback`). `Candidates.Fallback = false` gives an empty list instead.
- A query with no word as long as `TokenizerSetup.MinimumWordSize` (2) is searched as `Bm25f`.
- **Synonyms are alternatives.** An entry of the `SynonymList` widens the counted set where v5's
  appended terms narrowed it: "war film" with film / movie also counts the war movies. The list's
  format is unchanged.
- A dataset whose fields had differing `BM25k1` scores with BM25F unless built with
  `ScoringMode.BM25PerField` (section 5).

---

## 4. Fields

| Indx 5 | Indx 6 |
|----|----|
| `Field.Searchable = true` | `Field.WordMatching = true` **and** `Field.PatternMatching = PatternMatchingSetup.Default`, which is how v5 searched every searchable field |
| `Field.Searchable = false` | neither (both are off by default) |
| `Field.WordIndexing` | gone: `WordMatching` puts the field's words in the word lookup |
| `Field.HighResolution`, `TokenizerSetup.HighResolutionMode` | gone: words typed together or apart are matched by `Match.JoinedWords`, on by default |
| `IndexerSetup.IndexSizes`, `NumberOfStartCharacters`, `NumberOfStopCharacters`, `ArrayIndexSizes`, `ArrayNumberOfStartCharacters` | per field: `new PatternMatchingSetup(indexSizes, numberOfStartCharacters, numberOfStopCharacters)`. An array field is indexed like any other |
| `IndexerSetup.Dense` | `new PatternMatchingSetup([2, 3])` on the fields that need it |
| `IndexerSetup.SingleWord` (codes, SKUs) | `new PatternMatchingSetup([2, 3], 2, 1)`, or `WordMatching` alone for whole codes |
| `Field.BM25k1` differing between fields | chooses nothing any more: see `ScoringSetup` (section 5) |

**To keep v5's matching**, every field that was `Searchable = true` becomes:

```csharp
field.WordMatching = true;
field.PatternMatching = PatternMatchingSetup.Default;
```

`WordMatching` alone matches whole words only, within the field's `Tolerance` (`OneEdit` by
default): the right choice for codes, identifiers and tags. `PatternMatching` alone matches inside
words and stays out of the word lookup, so it orders records but does not count them.

- **A configuration made with `Init` searches nothing until one of the two is set.** Code that set
  `Searchable` after `Init` must set the new roles instead, or no field is searched.
- **A field configuration saved by v5** (`SaveFieldConfiguration`, an Indx server's stored
  datasets) is read as it was searched: a `Searchable` field is matched by pattern, and by its words
  only where v5's `WordIndexing` was on, which it was not by default. **Turn on `WordMatching` after
  upgrading** on the fields visitors type (titles, names, codes): until then no field is matched by
  its words, searches order records by score and do not narrow the list to the ones that match, and
  counts and facets cover nearly the whole dataset. The server's field configuration page says so.
- Which way a searched field is matched is a reindex (`Index()`); whether it is searched at all is a
  reload (`Load`), as `Searchable` was.
- The shortest word is `TokenizerSetup.MinimumWordSize` (2) in every field: one-letter words are no
  longer matched as words, and two-letter words ("øl", "te") now are, anywhere in a field.
- New per field, all optional: `Tolerance`, `RerankWeight` (0 leaves the field out of the reranker),
  `TypingPrefixFrom`, `CaseSensitive`, `TextNormalizer`.

---

## 5. Configuration

| Indx 5 | Indx 6 |
|----|----|
| `WordMatcherSetup` (the fourth argument of `ConfigurationParameters`) | gone. That argument is now `CompoundSetup`; pass `CompoundSetup.Default` |
| `new WordMatcherSetup()` (fuzzy matching off) | per search `Match.Tolerance = Tolerance.Exact`, or per field `Field.Tolerance = Tolerance.Exact` |
| `WordMatcherSetup.Default` (one edit and affixes) | the defaults: up to two edits by length, capped by each field's `Tolerance` (one edit by default) |
| `WordMatcherSetup.SupportAffix` ("fish" finds "fishing") | the word being typed is matched as the start of a longer word (`Match.TypingPrefixFrom`, 2); a finished word is not. Compound words are linked to their parts by `CompoundSetup` ("ost" finds "brunost") |
| `ConfigurationParameters.With(wordMatcherSetup: …)` | `With(compoundSetup: …)`, `With(scoringSetup: …)` |
| `new IndexerSetup(indexSizes, numberOfStartCharacters, numberOfStopCharacters, stopTermLimit, doCaseSensitiveSearching, maxIndexTextLength, maxSearchTextLength, …)` | `new IndexerSetup(stopTermLimit, doCaseSensitiveSearching, maxIndexTextLength, maxSearchTextLength)`; the sizes are per field (section 4) |
| `new TokenizerSetup(delimiters, highResolutionMode, minimumWordSize)` | `new TokenizerSetup(delimiters, minimumWordSize)` |
| the scoring mode chosen by differing `Field.BM25k1` | `ScoringSetup` in the configuration: `ScoringMode.BM25F` (the default, with `BM25FK1` 1.2) or `ScoringMode.BM25PerField`. In BM25F a field's `BM25k1` is ignored |
| `TokenizerSetup.MaxSizeOfWordToken` 20 | 100, so long compounds are indexed as words |

```csharp
// Indx 5: fuzzy matching off for the whole engine
var config = new ConfigurationParameters(IndexerSetup.Default, TokenizerSetup.Default,
    TextNormalizer.Default, new WordMatcherSetup());

// Indx 6: the configuration has no fuzzy switch; say it per search (or per field)
var config = new ConfigurationParameters(IndexerSetup.Default, TokenizerSetup.Default,
    TextNormalizer.Default, CompoundSetup.Default);
var query = new Query(text, 20) { Match = { Tolerance = Tolerance.Exact } };
```

A configuration file written by v5 (`ConfigurationParameters.Save`) still loads: the members that
are gone are ignored, and the scoring reads as BM25F with k1 1.2.

---

## 6. Filters and boosts

| Indx 5 | Indx 6 |
|----|----|
| `CreateBoost(filter, strength)` | `CreateBoost(filter, strength, out string? error)`, returning `Boost?` (null with the reason for a null filter or an unknown strength) |
| `GetFilterFromKey(key)` | `GetFilterFromKey(key, out string? error)` |
| `CreateRangeFilter(field, min, max, CultureInfo, out error)`, `RangeFilter.Culture` | gone: limits are invariant culture. A key with a culture segment is refused |
| `CombinedFilterProxy(a, b, useAnd)` with `A` and `B` | `new CombinedFilterProxy(useAnd, filters…)` with `Filters`; over HTTP `{ "filters": [...], "useAndOperation": true }` |

A filter key (a `hashString`) is read exactly as the engine writes it, brackets included; a
combination spelled any other way is refused with the key to use. Keys an engine handed out work as
before. New, optional: `CreateFilter(expression)` (`brand IN ["nike", "puma"] AND price BETWEEN 100
AND 2000`), `CreateInFilter`, `CreatePositionFilter`, `CreateCustomFilter`, `CreateDateFilter`,
`Filter.And` / `Filter.Or` over any number of filters, and `FilterKeyReader`.

---

## 7. HTTP API (Indx server)

The search body (`QueryProxy`) follows the library. Every member of `match`, `candidates` and
`ranking` is optional: one left out takes the dataset's query parameters, and where the dataset
sets none, the default.

```json
{
  "text": "melk",
  "maxNumberOfRecordsToReturn": 20,
  "mode": "EditDistance",
  "match": { "tolerance": "OneEdit" },
  "candidates": { "maxCandidates": 1000, "fallback": false },
  "ranking": { "boosts": [{ "boostStrength": "High", "filterProxy": { "hashString": "…" } }] }
}
```

- **Gone from the search body:** `enableCoverage`, `coverageDepth`, `coverageSetup`, `enableBoost`,
  `boosts` and `fieldBoosts` at the top level. **They are ignored if sent**, so a v5 client keeps
  getting answers but silently loses those settings: move each into its part as in section 2.
- **The result:** `count`, `ranked`, `toleranceCounts`, `countedTolerance`, `isFallback`,
  `reranked`, `fill`; `truncationIndex` and `truncationScore` are gone.
- **Enums are written by name** in every response: `"systemState": "Ready"`, `"boostStrength":
  "Med"`. v5 wrote numbers, so a client that compares one to a number
  breaks. A number is still read in a request.
- **Fields (`PUT …/fields/configuration`):** `wordMatching`, `patternMatching` (with
  `indexSizes`, `numberOfStartCharacters`, `numberOfStopCharacters`), `tolerance`, `rerankWeight`,
  `typingPrefixFrom`, `caseSensitive`. `searchable` from an older client is read as `wordMatching`
  only, so the field loses pattern matching: send `"wordMatching": true, "patternMatching": true`
  to keep v5's matching. `wordIndexing` and `highResolution` bind to nothing.
- **Routes gone:** `GET …/fields/searchable` and `GET …/fields/word-indexing`; read the roles from
  `GET …/fields/configuration`.
- **Routes new:** `…/buries` (GET, PUT, DELETE, `POST …/buries/from-filter`), `…/fields/profiles`
  (GET, PUT), `POST …/filters/expression`, `POST …/filters/in`, `POST …/filters/position`. A
  search can also take `filterExpression` with `filterParameters` in place of a `filter` token.
- **`POST …/filters/combine` takes a list:** `{ "filters": [...], "useAndOperation": true }` in place
  of `{ "a", "b", "useAndOperation" }`.
- **A dataset's query parameters (`…/query-parameters`)** are `{ mode, match, candidates, ranking }`
  in place of `{ coverageDepth, coverageSetup }`, and `GET` answers `{ parameters, effective }`.
  **Parameters saved by a v5 server read as none set after the upgrade**: set them again, a
  coverage depth as `candidates.maxCandidates`.
- **A dataset's stored boost and bury rules apply to every search**, with no flag in the body.
- **MCP:** `search` returns `total` (the matches in all) beside `count` (the hits returned), and
  `broaden` turns on the rest and the fallback.

---

## 8. intrface and indx-types (npm)

Upgrade both to 4.0 together; they need a v6 server.

- **`SearchProvider`:** `mode`, `match`, `candidates` and `ranking` in place of `enableCoverage`,
  `coverageDepth` (now `candidates={{ maxCandidates }}`) and `initialCoverageSetup`. A part the page
  leaves out is the dataset's.
- **`SearchSettings`** holds the same four; `setSearchSettings` merges a part member by member, and
  `setSearchSettings({ match: undefined })` hands a part back to the dataset.
- **The state's `truncationIndex` is `countedTotal`** (the result's `count`), with `isFallback`.
- **Search or ask:** the Ask phrases are gone; `stripAskPhrases` and the `AskPhrases` type are no
  longer exported. The list under an offer is no longer cut to the strong matches: `cutResults`,
  `cutRule`, `visibleResults`, `DEFAULT_CUT`, `CutRule` and `SCORE_SCALE` are gone, and
  `SearchResults` dims the list instead when the judge reads the answer as not in it (`notFound`).
  `held` says asking is held (the day's budget spent, the provider refusing answers), when nothing
  is offered in any mode. `SearchOrAskMode` and `pageMode` are exported.
- **indx-types:** `QueryProxy`, `MatchSetup`, `CandidateSetup`, `RankingProxy`, `Result` and
  `FieldProxy` as the server has them, `CombinedFilterProxy` with `filters`, and string enums
  (`SearchMode`, `Tolerance`, `Rungs`, `IndexTerms`, `BoostStrength` with `Med` where it had
  `Medium`). `CoverageSetup`, `CloudQuery` and `CoverageStrictness` are gone.

---

## 9. Checklist

**C# (IndxSearchLib):**
1. Replace `EnableCoverage = false` with `Mode = SearchMode.Bm25f, Ranking = { Rerank = false }`;
   delete `EnableCoverage = true` (2a).
2. Move `CoverageDepth`, every `CoverageSetup` member, `Boosts`/`EnableBoost` and `FieldBoosts` into
   `Match`, `Candidates` or `Ranking`; delete the truncation settings (2b).
3. Read `Count`, `Ranked` and `ToleranceCounts` where you read `TruncationIndex` (3).
4. Replace `Searchable = true` with `WordMatching = true` and `PatternMatching =
   PatternMatchingSetup.Default`; drop `WordIndexing` and `HighResolution`; move `IndexerSetup`'s
   sizes to `PatternMatchingSetup` on the fields (4).
5. Replace the `WordMatcherSetup` argument with `CompoundSetup.Default`, and a fuzzy-off
   `new WordMatcherSetup()` with `Match.Tolerance = Tolerance.Exact`; add `CreateBoost`'s
   `out error` (5, 6).
6. If fields had differing `BM25k1`, decide on `ScoringSetup` (5).
7. Build, then re-judge relevance on real queries (3a).

**HTTP:**
1. Move the search body's flat settings into `mode`, `match`, `candidates`, `ranking` (7).
2. Read `count`, `ranked`, `toleranceCounts` and `isFallback`; compare enums by name.
3. Send `wordMatching` and `patternMatching` in field configurations, never `searchable`.
4. Send `filters/combine` a list.
5. Set each dataset's query parameters again after upgrading the server.
6. Smoke-test a search with and without a typo, and one with no hit (`isFallback`).

**intrface:** upgrade `@indxsearch/indx-types` and `@indxsearch/intrface` to 4.0 together and
rename the provider's props (8).
