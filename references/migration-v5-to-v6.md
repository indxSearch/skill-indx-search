# Migrating Indx v5 → v6

Use this when a user is on **Indx v5** (IndxSearchLib 5.0.x, the Indx server's v2 API) and moves to
**v6** (the search that counts what it finds). v6 is a breaking release, and on purpose without
compatibility layers: old names give a compile error or a `400`, never a silent default. Every
change below is listed with what to write instead.

The single biggest change is **the query**: its settings live in three parts (`Match`,
`Candidates`, `Ranking`), and *how documents are fetched* (`Query.Mode`) is a separate choice from
*whether they are reranked* (`Ranking.Rerank`). The second is **the result**: it says how many
documents answer (`Count`) and how they are spread over the rungs (`ToleranceCounts`), in place of
a truncation index.

> v6 is being prepared on the `fastpath` branch; the library's changelog lists it under its next
> version. This guide covers everything that breaks code coming from 5.0.x.

---

## 1. Determine the version

- **C# / NuGet:** the `IndxSearchLib` version. On v6, `Query.Match`, `Query.Candidates` and
  `Query.Ranking` exist and `Query.EnableCoverage` does not.
- **HTTP:** a v6 server answers a search body with `"enableCoverage"` or `"coverageDepth"` the same
  as without it (unknown members are ignored), and `"mode": "Coverage"` with `400`. The result has
  `count`, `ranked` and `toleranceCounts`, and no `truncationIndex`.
- **npm:** `@indxsearch/indx-types` and `@indxsearch/intrface` at the major version after the
  `query-in-three-parts` changeset.

---

## 2. The query

### 2a. Fetching and reranking are two choices

| v5 | v6 |
|----|----|
| `Query.EnableCoverage = true` (the default) | `Query.Mode = SearchMode.EditDistance` (the default), `Ranking.Rerank = true` (the default): nothing to write |
| `Query.EnableCoverage = false` | `Query.Mode = SearchMode.Bm25f` **and** `Ranking.Rerank = false`: BM25F alone |
| – | `Mode = SearchMode.Bm25f`, `Rerank = true`: BM25F's best reranked, new |
| – | `Mode = SearchMode.EditDistance`, `Rerank = false`: the counted set ordered by BM25F, new |

`SearchMode.Bm25f` alone leaves `Rerank` on. Code that meant "Coverage off" must say both:

```csharp
var query = new Query(text, 20) { Mode = SearchMode.Bm25f, Ranking = { Rerank = false } };
```

`EditDistance` is the counting search: every document holding each query word exactly, else
within one edit, else two, counted, ordered rung by rung (exact first). `Bm25f` is ranking by
relevance with no count. Reranking (until v6 "Coverage") orders inside either. For relevance
ranking, `Bm25f` reranked reads best on the product benchmarks; for a shop-like, counted search,
`EditDistance` (the default).

### 2b. The settings in three parts

Every setting that is not the query's own (`Text`, `Mode`, filter, sort, facets, rows, timeout)
moved to one of three parts, each never null.

| v5 | v6 |
|----|----|
| `Query.CoverageDepth` | `Candidates.MaxCandidates` (`int?`, outright; null: the rows × `CandidatesPerRow`, 4, at least 100, at most 5 000) |
| `CoverageSetup.IncludePatternMatches` | `Candidates.Fallback` |
| `Query.IndexTerms` | `Candidates.IndexTerms` |
| `CoverageSetup.Strictness` | `Match.Tolerance` (`Exact`, `OneEdit`, `TwoEdits`) |
| `CoverageSetup.OneEditFrom`, `TwoEditsFrom` | `Match.OneEditFrom`, `Match.TwoEditsFrom` |
| `CoverageSetup.CoverJoinedWords`, `CoverRunTogetherWords` | `Match.JoinedWords` |
| `WordLookupSetup.PrefixFrom`, `PrefixOneEditFrom` | `Match.TypingPrefixFrom`, `Match.TypingPrefixOneEditFrom` |
| `Query.CursorAt`, `Query.Rungs` | `Match.CursorAt`, `Match.Rungs` |
| `CoverageShare` / `CoverageSetup.TypingShift` | `Ranking.Share`, `Ranking.TypingShift` |
| `CoverageSetup.CoverWholeQuery` | `Ranking.WholeQueryFloor` |
| `CoverageSetup.LongFieldFrom`, `LongFieldWeightFloor` | `Ranking.LongFieldFrom`, `Ranking.LongFieldWeightFloor` |
| `CoverageSetup.Fields[].Weight` | `Ranking.FieldWeights` (a dictionary by field name) |
| `Query.FieldBoosts` | `Ranking.FieldBoosts` |
| `Query.Boosts` + `EnableBoost`, `Query.Buries` + `EnableBury` | `Ranking.Boosts`, `Ranking.Buries` (a list not empty is on) |
| `Query.EnableCoverageReport`, `Result.CoverageReport` | `Query.EnableRerankReport`, `Result.RerankReport` |

**Gone without a replacement** (fixed at the defaults they had): `CoverWholeWords`,
`CoverPrefixSuffix`, `PrefixSuffixWhileTyping`, `CompoundRestFrom`, `WholeWordByLength`,
`CoverageSetup.MinWordSize` (the shortest word is `TokenizerSetup.MinimumWordSize`), and the cut:
`Truncate`, `TruncationScore`, `TruncationHardness`, `TruncateWordHitLimit`,
`TruncateWordHitTolerance`, `TruncationFilter`, `ScoreCut`. The types `CoverageSetup`,
`WordLookupSetup` and `FieldCoverage` are gone. `SearchEngine.ValidateCoverage(CoverageSetup)` is
`ValidateRanking(RankingSetup)`.

```csharp
// v5
var q = new Query("melk", 20) { CoverageDepth = 500, EnableBoost = true, Boosts = [boost] };
q.CoverageSetup.Strictness = CoverageStrictness.Ldd0;
// v6
var q = new Query("melk", 20)
{
    Match = { Tolerance = Tolerance.Exact },
    Candidates = { MaxCandidates = 500 },
    Ranking = { Boosts = [boost] },
};
```

---

## 3. The result

| v5 | v6 |
|----|----|
| `Result.TruncationIndex`, `TruncationScore` | gone: the list is what was counted, cut to the rows |
| – | `Result.Count`: how many documents answer (the rung counted; in `Bm25f` the documents with a score) |
| – | `Result.Ranked`: how many were ranked, at most the bound. `Count > Ranked`: the bound was too small |
| – | `Result.ToleranceCounts` (`Exact`, `OneEdit`, `TwoEdits`, `Rest`), cumulative; null in `Bm25f` |
| – | `Result.CountedTolerance`: the rung the list was taken from |
| – | `Result.IsFallback`: no document held every word; the list is the documents holding the most |
| `Result.CoverageReport` | `Result.RerankReport` |

A client that paged with `truncationIndex` ("load results 1–20 of N") reads `Count` for N. A
client that showed "N hits" when the truncation index was set shows `Count`, and can split it by
rung: "140 hits, 45 with a typo" from `ToleranceCounts`.

### 3a. What the list is (behaviour, compiles either way)

- In `EditDistance` the list is **the counted set and nothing else**: documents that do not hold
  every query word within the tolerance are not shown. With `Candidates.Rest = true` the best of
  them by BM25F come below the counted ones, counted in `ToleranceCounts.Rest` and in the facets.
- **The facets are counted over the counted set**, the whole rung, not over the rows returned.
  Count, facets and list agree.
- With nothing counted, the fallback: the documents holding the most of the words first
  (`IsFallback`). `Candidates.Fallback = false` gives an empty list instead.
- A query with no word as long as `TokenizerSetup.MinimumWordSize` (2): "m" is searched as
  `Bm25f`, with its count and facets.
- Rankings changed: the rung outermost (exact before one edit), reranking inside. Do not compare
  v6 lists to v5 lists as a regression test; compare against judged results.

---

## 4. Fields

| v5 | v6 |
|----|----|
| `Field.Searchable` | `Field.WordMatching` (by whole words) and/or `Field.PatternMatching` (inside words). Neither: not searched |
| `Field.IndexSizes`, `NumberOfStartCharacters`, `NumberOfStopCharacters` | `Field.PatternMatching = new PatternMatchingSetup(sizes, start, stop)`; `PatternMatchingSetup.Default` is what every searched field had |
| `Field.CoverageStrictness` (`Ldd0`, `Ldd1`, `Ldd2`) | `Field.Tolerance` (`Exact`, `OneEdit`, `TwoEdits`) |
| `CoverageStrictness.None` | `Field.RerankWeight = 0` (out of reranking and the word lookup) |
| `Field.CoverageWeight` | `Field.RerankWeight` |
| `Field.HighResolution` | gone |
| `Field.BM25k1` in BM25F | the engine's `ScoringSetup.BM25FK1`; the field's is read in `BM25PerField` only |

**To keep v5's matching**, a field that was `Searchable = true` becomes:

```csharp
field.WordMatching = true;
field.PatternMatching = PatternMatchingSetup.Default;
```

`WordMatching` alone matches by whole words only (codes, identifiers, tags). A field configuration
saved by a v5 engine (`DocumentFields`, the server's stored datasets) is read as it was searched:
`Searchable`, `CoverageStrictness` and `CoverageWeight` are translated when it is loaded.

---

## 5. Configuration

| v5 | v6 |
|----|----|
| `WordTrieSetup` (`Affixes`, `MinimumAffixLength`, `MaximumAffixWordLength`) | `CompoundSetup` (`Compounds`, `MinimumPartLength`, `LinkingLetters`): compounds linked to their parts |
| `WordLookupSetup` in the configuration | per search, `Query.Match` |
| `IndexerSetup.IndexSizes`, `NumberOfStartCharacters`, … | per field, `Field.PatternMatching` |
| the scoring mode inferred from differing k1 | `ConfigurationParameters.Scoring` (`ScoringSetup(Mode, BM25FK1)`) |
| `TokenizerSetup.MaxSizeOfWordToken` 20 | 100 |

---

## 6. HTTP API

The search body (`QueryProxy`) follows the library:

```json
{
  "text": "melk",
  "maxNumberOfRecordsToReturn": 20,
  "mode": "EditDistance",
  "match": { "tolerance": "OneEdit" },
  "candidates": { "maxCandidates": 500, "rest": false },
  "ranking": { "rerank": true, "share": 0.2, "boosts": [{ "boostStrength": "High", "filterProxy": { "hashString": "…" } }] }
}
```

- **Gone from the body:** `enableCoverage`, `coverageDepth`, `coverageSetup`, `enableBoost`,
  `enableBury`, top-level `boosts`, `buries` and `fieldBoosts`. They are ignored if sent, so a v5
  client keeps working but loses those settings: move them into the parts.
- **Enums are written by name** in requests and responses: `"mode": "Bm25f"`,
  `"tolerance": "Exact"`, `"boostStrength": "Med"` (was `Medium`), `"systemState": "Ready"`.
- **`mode: "Coverage"` is refused** (`400`): write `EditDistance` (with `ranking.rerank` true, the
  default).
- **The result:** `count`, `ranked`, `toleranceCounts`, `countedTolerance`, `isFallback`,
  `rerankReport`; `truncationIndex`, `truncationScore` and `overflow` are gone.
- **Fields:** `wordMatching`, `patternMatching` (+ `indexSizes`, `numberOfStartCharacters`,
  `numberOfStopCharacters`), `tolerance`, `rerankWeight` in a field's configuration
  (`PUT …/fields/configuration`). `searchable` from an older client is still read (true as
  `wordMatching`, false as neither) and said in the dataset's status; it is never written back.
  `coverageStrictness` and `coverageWeight` are ignored if sent.
- **A dataset's stored boost and bury rules apply to every search**, without a flag.
- **MCP:** `search` returns `total` (the matches in all) beside `count` (the hits returned);
  `broaden` turns on the rest and the fallback.

---

## 7. intrface and indx-types (npm)

- **`SearchProvider`:** `mode`, `match`, `candidates`, `ranking` in place of `enableCoverage`,
  `coverageDepth` (now `candidates={{ maxCandidates }}`) and `initialCoverageSetup`.
- **`SearchSettings`** holds the same four; `setSearchSettings` merges a part member by member.
- **The state's `truncationIndex` is `countedTotal`** (the result's `count`), with `isFallback`.
- **indx-types:** `QueryProxy`, `MatchSetup`, `CandidateSetup`, `RankingProxy`, `Result`
  (`count`, `ranked`, `toleranceCounts`, …), `FieldProxy` (`wordMatching`, `tolerance`,
  `rerankWeight`) as the server has them; the enums are string enums (`SearchMode`, `Tolerance`,
  `Rungs`, `IndexTerms`, `BoostStrength` …). `CoverageSetup`, `CloudQuery` and the
  `CoverageStrictness` enum are gone.

---

## 8. Other changes since 5.0

These came in the same line and break code that used them:

- **Internal now:** `CoreSearchEngine`, `Document`, `CoreDocument`, `JsonParser`,
  `EmbeddingIndex`. `SearchEngine.GetDocumentJsonOfKey` is gone (`GetJsonDataOfKey` returns the
  record). A vector field is read through `IEmbeddingIndex`.
- **Filters:** `CreateBoost` returns `Boost?` and refuses a null filter; `CreateRangeFilter(…,
  CultureInfo, …)` and `RangeFilter.Culture` are gone (limits are invariant culture); a filter key
  is read with its brackets (`FilterKeyReader.Operands` to read one); a range filter needs a Number
  field and a value filter a non-Number one.
- **`TrySetFieldConfiguration`** returns every refusal instead of throwing for some.
- **Vector search** was rebuilt (the engine's own HNSW graph): scores and order of vector and
  hybrid results change, and the `HNSW` package is no longer a dependency.

The library's changelog has every entry with its reason.

---

## 9. Checklist

**C# embedded:**
1. Replace `EnableCoverage = false` with `Mode = SearchMode.Bm25f, Ranking = { Rerank = false }`;
   delete `EnableCoverage = true` (2a).
2. Move every `CoverageSetup`, `WordLookupSetup`, `CoverageDepth`, `FieldBoosts`, `Boosts`/`Buries`
   setting into `Match`, `Candidates` or `Ranking` (2b).
3. Replace `TruncationIndex`/`TruncationScore` with `Count`, `Ranked` and `ToleranceCounts` (3).
4. Replace `Field.Searchable` with `WordMatching` + `PatternMatching`, `CoverageStrictness` with
   `Tolerance`, `CoverageWeight` with `RerankWeight` (4).
5. Replace `WordTrieSetup` with `CompoundSetup` (5).
6. Rebuild; check against [csharp.md](csharp.md). Re-judge relevance, do not diff old lists (3a).

**HTTP:**
1. Move the body's flat settings into `mode`, `match`, `candidates`, `ranking`, enums by name (6).
2. Read `count`, `ranked`, `toleranceCounts`, `isFallback` from the result.
3. Rename the field configuration's members (6).
4. Smoke-test a search with and without a typo, and a search with no hit (`isFallback`).

**intrface:** upgrade both packages together and rename the provider props (7).
