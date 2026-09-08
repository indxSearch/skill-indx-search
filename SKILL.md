---
name: indx-search
description: Indx Search integration skill for AI coding agents. Use when building or evaluating search with Indx — a high-performance, typo-tolerant search engine that matches at the character-pattern level instead of relying on tokenizers or per-language stemmers. Covers C# NuGet (IndxSearchLib) and HTTP API (IndxCloudApi) integration, a built-in MCP server, field configuration and weighting, search-as-you-type, filters and facets, boost rules (incl. scheduled campaigns and per-user personalisation), synonyms, vector and hybrid search, real-time document updates, zero-downtime reindexing, server-side (SSR) usage, and search UX patterns.
---

# Indx Search — Agent Skill

Indx is a high-performance search engine for structured and unstructured text. It matches at the character-pattern level rather than through tokenizers and per-language stemmers, so typos, inflections, compound words and messy input are handled without language configuration. See [Language handling](#language-handling) for exactly what that covers and where synonyms come in.

If you are **evaluating** Indx against a feature list, read [Capability summary](#capability-summary) first — it answers the usual comparison questions directly.

## Versions & Compatibility — read first

This skill targets **Indx v5**: IndxSearchLib **5.x** (.NET 10) and IndxCloudApi **v2** (team-scoped HTTP API, token-only auth). Everything below assumes v5.

**Before giving any integration guidance, establish which version the user is on.** v4 and v5 differ in ways that silently break copy-pasted code — handing a v4 user v5 routes is a common, confusing failure. Detection signals:

| Signal | v4 (legacy) | v5 (this skill) |
|--------|-------------|-----------------|
| C# `IndxSearchLib` NuGet | `4.x` (older .NET target) | `5.x`, .NET 10 |
| HTTP route shape | flat — `/api/Search/{dataset}` | team-scoped — `/api/teams/{team}/datasets/{dataset}/search` |
| List-datasets endpoint | `/api/GetUserDatasets` → `string[]` | `/api/me/datasets` → objects with `teamName` + `role` |
| Teams | none — datasets owned by the user | datasets owned by **teams**; every endpoint is team-scoped |
| Auth | API password login (`/api/Login`) available | **token-only** — portal-issued JWT, no login API |

If the user is on **v5**, proceed normally. If on **v4**, do **not** hand them v5 routes/APIs — either answer for their v4 setup, or (recommended) help them upgrade: see [references/migration-v4-to-v5.md](references/migration-v4-to-v5.md). When the version is ambiguous, ask one quick question (NuGet version, or whether their API URLs contain `/teams/`) before proceeding.

## When to Use Indx

- Up to millions of documents where you need fast, typo-tolerant search
- When you don't want to configure tokenizers, stemmers, analyzers, or language settings
- When you want embedded search with no external dependencies (NuGet) or a lightweight self-hosted API
- Unmatched on speed — in-memory indexes and a fast vector model mean Indx outperforms Elastic/Solr/Algolia in most scenarios

**When Indx is not the right fit:**
- Tens of millions+ of documents (log aggregation, large-scale analytics)
- Pure exact-match queries (database-style lookups)
- Schemas that change every few minutes — field *values* update in real time (see [Dynamic document operations](#dynamic-document-operations)), but adding or retyping *fields* needs a reload, which `replace` does with zero downtime

## Choosing Your Integration Path

**C# / .NET project** → Use the [IndxSearchLib NuGet package](https://www.nuget.org/packages/IndxSearchLib/) directly (.NET 10, v5.0.0). Embed search into your application with no external dependencies. See [references/csharp.md](references/csharp.md) for full API reference.

**Any other tech stack** (Node.js, Python, Java, etc.) → Deploy the [IndxCloudApi](https://github.com/indxSearch/IndxCloudApi) HTTP API server and interact via REST (.NET 10). Recommended deployment target: Azure App Service. See [references/cloudapi-setup.md](references/cloudapi-setup.md) for setup and deployment, and [references/http-api.md](references/http-api.md) for endpoints, schemas, and data loading.

**Connecting an AI agent to a running instance** → IndxCloudApi has a built-in **MCP server** — see below.

## AI Agents over MCP

IndxCloudApi exposes a read-only **MCP server** at `/mcp` (Streamable HTTP), so any MCP client (Claude Desktop/Code, agent frameworks) can search a running instance with no glue code. It runs in-process, so **saved boost rules apply** to agent searches and hibernated datasets auto-wake. Authenticate with an **API key as a `Bearer` token** (generate under Account → API keys; admins toggle the server under Admin → Settings).

Tools:

- **`list_datasets`** — datasets the token can reach, with document counts and state.
- **`describe_dataset(team, dataset)`** — configured fields with capabilities, plus **value hints** (distinct values for facetable fields, numeric ranges), an owner description, and a sample document. Call first so the agent filters with real values.
- **`search(team, dataset, query, filters?, limit?, fields?, broaden?, facets?)`** — ranked hits with scores. Declarative AND filters (`{field, value}` or `{field, min, max}`). **Precise by default** (`includePatternMatches=false`) so an empty result is a trustworthy no-match; pass `broaden: true` for fuzzy recall.
- **`get_document(team, dataset, key)`** — full JSON for a key.
- **`get_synonyms(team, dataset)`** — the dataset's synonym list (an empty `entries` array when it has none). Explains why a search matched more than its literal words (queries expand through the list before scoring).

Connect with the endpoint URL + API key. Clients without a custom-header field use the `mcp-remote` bridge:

```json
{ "mcpServers": { "indx": { "command": "npx",
  "args": ["mcp-remote", "https://<your-host>/mcp", "--header", "Authorization: Bearer <your-api-key>"] } } }
```

Typical flow: `describe_dataset` → `search` (text + filters) → `get_document`.

## Core Concepts

### Two-Step Search Model

Every search executes in two phases:

1. **Pattern Matching** — Scans all documents for textual and structural patterns. Produces candidate results with strong recall and built-in typo tolerance. No query preprocessing needed.

2. **Coverage** (enabled by default) — A collection of algorithms that detect exact and near-exact token matches (whole words, fuzzy words, joined/split words, prefixes/suffixes) in the top-K candidates (default: 500). Confirmed matches are scored 0–65535 and promoted above pure pattern matches. A truncation index marks where coverage-confirmed results end.

### Field Configuration

Fields must be explicitly marked with their roles before indexing:

| Role | Purpose | Notes |
|------|---------|-------|
| **Searchable** | Included in matching and scoring | At least one required. Supports weight for relative importance |
| **Filterable** | Available for filter operations | Used with value filters and range filters |
| **Facetable** | Used for aggregations | Returns value counts (histograms) |
| **Sortable** | Enables result ordering | Works on numbers and strings. See sorting behavior below |
| **WordIndexing** | Indexes entire words | Useful on fields with many repeating words. See below |
| **HighResolution** | Also indexes the text with delimiters removed | Sharper matching of compound and run-together words (`"hvis feks"` also indexes `"hvisfeks"`). Use on key fields such as titles |
| **Embeddable** | Field carries a vector for semantic search | See [Vector and hybrid search](#vector-and-hybrid-search) |

**WordIndexing** — indexes entire words in a field. Useful for large datasets where many documents share the same words (e.g. a `category` field across thousands of products). Complements Searchable and must be combined with it on the same field. Only affects single-word queries.

**Weight** — Searchable fields have a `float` weight (default 1.25) to control importance. Higher = more influence on BM25 scoring. Typical range 0.5–3.0.
- C# API: `new FieldProxy { FieldName = "title", Searchable = true, Weight = 2.0f }`
- HTTP API: `{ "fieldName": "title", "searchable": true, "weight": 2.0 }`

Note: Weights affect pattern recognition directly. A short text pattern in a longer string will not necessarily rank higher than the same pattern in a shorter string, even if the longer field has higher weight.

**Per-query field boosts** — override field importance for one search without reconfiguring: `query.FieldBoosts = { ["title"] = 2.0f, ["description"] = 1.0f }` in C#, `"fieldBoosts": { "title": 2.0 }` in the HTTP `search` body. Use this for "title matches first" ranking, or to A/B two weightings from the same index.

### Filters Must Be Server-Side

Never filter results client-side after a search. The search only returns a limited number of results (`maxNumberOfRecordsToReturn`), so client-side filtering on that subset will miss documents. Always create filters server-side (`CreateValueFilter` / `CreateRangeFilter` / `CombineFilters` in C#; `POST filters/value` / `filters/range` / `filters/combine` over HTTP) and pass the filter in the query (`query.Filter` in C#, `CloudQuery.filter` in HTTP) so the server applies the filter during search.

### Search Behavior Guidance

- **Keep coverage enabled** (the default). This is the recommended setting for nearly all use cases — both human-facing and programmatic. Only disable coverage in edge cases where you search a single field and only care about top-K fuzzy matches (e.g. name lookup). With coverage disabled, truncation is unreliable and results degrade when searching across multiple fields (title + description + category, etc.).
- **Agent/tool usage**: Enable coverage and set `IncludePatternMatches = false` (`coverageSetup.includePatternMatches` in HTTP). This returns only exact and near-exact matches (within ~1 typo), filtering out loose pattern hits.
- **Empty search**: Supported with empty/null query text. Requires facets enabled and at least one facetable field. Returns all documents, ignores `CoverageDepth`.
- **No debounce needed on search**: Indx is fast enough that debouncing search requests is unnecessary. Fire on every keystroke.

### Sorting Behavior

- **With search text**: Sorting is **2nd-order** — search relevancy (score) is always the primary sort. Sorting only applies to results included in the coverage step.
- **Empty search** (no text): Sorting becomes the **primary** ordering function.
- Works on both numbers and strings (A–Z, 1–9). Default: descending. Set `SortAscending = true` to invert.
- To reset: set `query.SortBy = null`.

### Measuring performance

Two different numbers, both true: **latency** (one search at a time — what a user waits; the Cloud UI's search preview reports the median and p95 of 100 sequential searches after warm-up) and **throughput** (searches per second with all cores busy — what a server sustains; the same tab reports it, and `SpectreJsonClient`'s "response time" is this figure divided per search). Compare like with like: a throughput-derived per-search time is typically 2–3× lower than latency on a multi-core machine.

### Facets Tip

When implementing search-as-you-type with a large dataset, consider only fetching facets (`EnableFacets = true`) after a small delay, not on every keystroke.

### Coverage Tuning

- `EnableCoverage` (default: `true`) — Toggle the coverage refinement step.
- `CoverageDepth` (default: `500`) — Number of top-K pattern-match candidates to evaluate. Higher = better recall, more latency. Auto-increases if `MaxNumberOfRecordsToReturn > CoverageDepth`. Set to `engine.Status.DocumentCount` for full-dataset coverage.
- `CoverageSetup` — Fine-grained control (see advanced sections below).

### Language handling

Indx has no per-language tokenizer, stemmer or stop-word list, and does not need one to work in Norwegian, English or mixed-language data. What replaces them:

- **Typos and spelling variants** — pattern matching with one-edit tolerance (`Coverage.CoverFuzzyWords`).
- **Inflections and prefixes** — a query word that is a prefix of a document word matches (`CoverPrefixSuffix`): *run* finds *running*, *bok* finds *bokhandel*. The reverse direction (query *running*, document *run*) is covered by pattern matching but scores lower under Coverage; add a one-way synonym (*running → run*) when that direction matters for a term.
- **Compound words (sammensatte ord)** — *spesialpedagogikk* matches *spesial pedagogikk* and vice versa without rules (`CoverJoinedWords`); mark title-like fields `HighResolution` for the sharpest compound matching. Decomposition and recomposition both work; no dictionary is involved.
- **Domain vocabulary** — synonyms, editable by editors without code (next section).

So on a "stemming" checklist: Indx does not ship a stemmer because the matcher already handles the common cases; the remaining case (long inflected query, short stored form) is a synonym entry.

### Synonyms

A per-dataset synonym list expands the query text before scoring — no re-indexing, effective on the next search. Entries are **multidirectional** (every term pulls in the others: *sofa ↔ couch*) or **one-way** (*hms → his majesty's ship*: searching the short form expands, searching the long form does not).

- **Cloud UI**: the dataset's **Synonyms** tab — create, edit, filter, import/export JSON. Editors need no code access.
- **HTTP**: `GET …/synonyms`, `PUT …/synonyms` (editor role).
- **C#**: `engine.SynonymList` / `LoadSynonyms(path)` / `SaveSynonyms(path)`.

Expansion lengthens the query, which lowers Coverage scores proportionally — measure before shipping a very large list. Details: [references/csharp.md](references/csharp.md#synonyms), [references/http-api.md](references/http-api.md#synonyms).

### Dynamic document operations

Insert, update, partially update and delete documents while the engine stays Ready. Changes are **searchable immediately** — there is no queue, no rebuild and no eventual consistency: a document inserted on publish is in the next search.

| Operation | C# | HTTP |
|---|---|---|
| Insert | `InsertJsonRecord(s)` | `POST …/documents`, `POST …/documents/{key}` |
| Replace a document | `UpdateJsonRecord(s)` | `PUT …/documents`, `PUT …/documents/{key}` |
| Update one field | `UpdateField(key, field, value)` | `PATCH …/documents/{key}` |
| Update a field on a filter | `UpdateFieldInFilter(filter, field, value)` | `POST …/documents/update-by-filter` |
| Delete | `DeleteJsonRecord(s)`, `DeleteRecordsInFilter` | `DELETE …/documents/{key}`, `DELETE …/documents`, `POST …/documents/delete-by-filter` |

For a whole-catalogue reload use `POST …/replace` (IndxCloudApi): the old index keeps serving until the new one is built, then swaps atomically — **zero downtime**, and field configuration, boost rules, synonyms and the key field carry over. Adding new fields needs this path; changing values does not.

### Boost rules, campaigns and personalisation

Boosts lift matching documents when a search runs with `enableBoost`. Two layers:

- **Saved boost rules** (Cloud UI **Boost rules** tab; `GET/PUT/DELETE …/boosts`): a rule has a name, an `enabled` flag, conditions on filterable fields (`{field, value}` or `{field, min, max}`, joined with AND/OR), a strength (Low/Med/High) and an optional schedule (`activeFrom`, `activeUntil` dates) — campaign windows without code changes. Rules stack.
- **Ad-hoc boosts per query** (`Query.Boosts` / `POST …/boosts/from-filter`): boost any filter, including an OR of many value filters. This is how **per-user personalisation** works — build a boost list from the user's history or segment and pass it with the query; hundreds of thousands of boosted documents cost little. See [references/csharp.md](references/csharp.md#boosts).

Scheduled rules that pass their end date stop applying but are kept; the Cloud UI flags them (chip, summary, tab badge) and notifies the team's editors. Import/export the rule list as JSON from the tab, the same array as `GET/PUT …/boosts`.

Popularity or sales-based ranking: store a `popularity` number on each document and boost on ranges of it (rule or ad-hoc). Indx does not collect behavioural signals itself. There is no negative boost ("bury") and no pinned positions — relevance stays the primary order; a High boost is the strongest lift.

### Vector and hybrid search

Mark a field `Embeddable`, load vectors with the documents, and use:

- `POST …/search/vector` — `{ fieldName, vector, maxResults, filter? }`: pure semantic nearest-neighbour (HNSW). Best for **"more like this"**, related items, cross-sell rails.
- `POST …/search/hybrid` — `{ text, vector, alpha, maxNumberOfRecordsToReturn, filter? }`: `alpha · vectorScore + (1 − alpha) · textScore`. Best for meaning-plus-keyword queries (*grisebok* finding *Peppa Gris*).

Indx **stores and searches** vectors; it does not generate them — bring embeddings from your own model. Filters apply to both. Details: [references/http-api.md](references/http-api.md#vector--hybrid-models).

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

### Arrays of objects

Dot notation continues straight through an array — the elements are addressed by
position, not by name, so they add nothing to the field name:

```json
[
  { "id": 1, "title": "Product A",
    "prices": [ { "amount": 299, "currency": { "code": "NOK" } },
                { "amount": 25,  "currency": { "code": "EUR" } } ] }
]
```

That gives the fields `prices.amount` and `prices.currency.code`, each holding
one value per array element. This is the usual shape for prices, variants and
localized strings in commerce exports.

### Files that wrap their documents

Many exporters wrap the documents in an object carrying metadata rather than
handing you a bare array:

```json
{ "exportedAt": "2026-05-11T09:18:09Z", "count": 16761,
  "products": [ { "id": 1, "title": "Product A" }, … ] }
```

Feed it in as it is. `Init`/`Analyze` locates the documents itself — the
outermost array of objects wins, and the envelope's own keys (`exportedAt`,
`count`) are not treated as fields. There is nothing to configure and no
conversion step; a plain array is of course still read exactly as before.

One thing to know: a *single* document handed over on its own, whose only array
is an array of objects, looks the same from outside — `{ "orderId": 1, "lines":
[…] }` will be read as its lines. Wrap it in an array if you mean it as one
document.

---

## Search UX Patterns

These patterns come from [indx-intrface](https://github.com/indxSearch/indx-intrface), a React component library for Indx Search. Even if you're not using the library, these are good principles to follow when building search UI on top of Indx.

### Preserving empty facets

When a user applies filters, some facet values may drop to 0 hits and disappear from the facets response. Don't remove them from the UI — keep showing them (greyed out, with count 0) so the user can still see and deselect them. Removing filter options mid-interaction is disorienting.

### Empty search as browse mode

Support an empty search state (`allowEmptySearch`) that returns all documents with facets and sorting. This lets users browse and filter before typing anything — useful for catalogue-style interfaces. Requires `enableFacets: true` and at least one facetable field.

### Facet debouncing

When doing search-as-you-type, search results need no debounce (Indx is fast enough to fire on every keystroke). But facet counts jumping on every character is noisy — debounce facet requests (e.g. 500ms) while keeping result updates immediate.

### Range filters from facet data

Use facet histograms to derive min/max bounds for range filter UI (sliders, inputs). This way the range controls automatically reflect the actual data spread rather than hardcoded limits.

### Active filter summary

Show all active filters (value filters and range filters) as removable chips above the results. Include a "reset all" action. This gives the user a clear picture of what's narrowing their results.

### Two-step result display (C# only)

The C# NuGet API returns document keys and scores (not full documents) in `result.Records`. Fetch full JSON separately via `engine.GetJsonDataOfKey(key)`. The HTTP API returns document keys too — use `POST …/documents/lookup` with the array of keys to retrieve full JSON. Only fetch the fields you need for display — keep the result list lightweight.

### React component library

For React 19+ projects, [@indxsearch/intrface](https://github.com/indxSearch/indx-intrface) implements all of the above patterns as drop-in components: `SearchProvider`, `SearchInput`, `SearchResults`, `ValueFilterPanel`, `RangeFilterPanel`, `ActiveFiltersPanel`, `SortByPanel`, and more. See the [indx-intrface README](https://github.com/indxSearch/indx-intrface) for full component API.

---

## Integration notes

### Server-side use (Next.js, React Server Components, any backend)

The HTTP API is plain REST over `fetch`; nothing about it needs a browser. For SSR or server components call `POST …/search` from the server with the API key in `Authorization: Bearer …` — the key stays server-side and never reaches the client. `@indxsearch/indx-types` is types-only (no runtime, safe anywhere); `@indxsearch/intrface` is React *client* components for the interactive parts. A typical Next.js split: server renders the first result page with `fetch` + `indx-types`, the client takes over typing with `intrface`.

### Caching responses

Search responses are deterministic for a given (query body, filter, index build). Cache them in your server or CDN layer as you like; invalidate when the dataset is reloaded or documents change. Nothing in the API forbids caching, and empty-search (browse) responses are the usual candidates.

### Several content types in one index

Put books, articles, events and authors in **one dataset** with a `contentType` field (mark it Filterable + Facetable). One search then covers all of them; the facet on `contentType` gives per-type counts in the same request, and a value filter narrows to one type. Fields that only some types have are fine — absent fields simply do not match.

### Variants and editions

Model one document per work with editions as an array (`editions[].isbn`, `editions[].format`, `editions[].price`). Array fields index one value per element and a filter matches if **any** element matches, so "paperback under 200" works at variant level while results stay one card per work. Note that `removeDuplicates` is per-document-key dedupe, not a group-by.

## Capability summary

Quick answers for feature comparisons. ✅ built in · 🟠 achievable with the noted pattern · ❌ not offered.

| Capability | | How |
|---|---|---|
| Field weighting, per-query field boosts | ✅ | `Weight`, `FieldBoosts` |
| Typo tolerance | ✅ | built in, no config |
| Search-as-you-type / autocomplete | ✅ | full ranked search per keystroke; no separate suggest endpoint needed |
| Stemming | 🟠 | prefix/compound matching built in; reverse inflection via one-way synonyms |
| Compound-word decomposition | ✅ | no rules needed |
| Synonyms, editor-editable | ✅ | Cloud UI tab, HTTP, C# |
| Facets with counts, per-type counts | ✅ | `Facetable`; zero-count values drop from the response — keep them greyed in the UI |
| Value / range / boolean filters, AND/OR | ✅ | server-side filters; NOT is C#-only (`!filter`) |
| Sorting by number/date/string | ✅ | `Sortable` |
| Variant-level filtering | ✅ | array fields, any-element match |
| Collapse variants to one card | 🟠 | model one document per work |
| Real-time updates on publish | ✅ | dynamic operations, immediate |
| Zero-downtime full reload | ✅ | `replace` |
| Scheduled campaign boosts | ✅ | boost rules with `activeFrom`/`activeUntil` |
| Per-user personalisation | ✅ | per-user boost lists |
| Popularity ranking | 🟠 | boost on a popularity field you supply |
| Bury / pin | ❌ | boosts lift only |
| Semantic / vector / hybrid search | ✅ | bring your own embeddings |
| Related items / more-like-this | 🟠 | vector search on the item's embedding |
| Spell-check "did you mean" | ❌ | not needed: the corrected match is returned directly |
| Search analytics (null results, popular queries) | ❌ | only a search counter; log `query.LogPrefix` into your own pipeline |
| Behavioural tracking / consent events | ❌ | none by design — no telemetry in the engine |
| A/B testing of relevance | 🟠 | `FieldBoosts` per query, or a second engine via `CreateInMemoryClone` |
| Client timeout | ✅ | `TimeOutLimitMilliseconds` + `DidTimeOut`; abort the HTTP request for hard cancel |
| SSR / server-side SDK | ✅ | REST + `indx-types`; see Integration notes |
| EU/EEA hosting, DPA | ✅ | self-host anywhere; the Azure Managed App deploys into your own subscription in the region you choose (e.g. Norway East); DPA on request |
| Self-hosting / source | 🟠 | IndxCloudApi host is open source on GitHub; the engine is a proprietary library with a free tier to 100k documents |

## Key Design Properties

- **No language configuration** — inflections, compounds and typos are handled by character-level matching, so there are no tokenizers, stemmers or stop-word lists to maintain per language; synonyms cover domain vocabulary
- **Built-in typo tolerance** — pattern matching handles misspellings automatically
- **In-memory indexing** — all search indexes live in memory for speed; persistence is metadata-only
- **Linear coverage scaling** — coverage cost scales linearly with `coverageDepth`
- **Schemaless JSON** — nested objects supported, fields discovered automatically via `Init`/`Analyze`
- **Two-step retrieval** — search returns keys + scores; fetch full documents separately: `GetJsonDataOfKey` in the C# NuGet API, `POST …/documents/lookup` in the HTTP API

## Resources

- [Indx Home](https://indx.co) — registration and licensing
- [API Documentation](https://v5.docs.indx.co) — C# and HTTP API reference with How-To guides (v4 docs remain at docs.indx.co)
- [Privacy & hosting](https://v5.docs.indx.co/gdpr) — data residency, GDPR, DPA
- **C# / .NET**
  - [IndxSearchLib NuGet](https://www.nuget.org/packages/IndxSearchLib/) — core search engine (.NET 10, v5.0.0)
- **HTTP API**
  - [IndxCloudApi](https://github.com/indxSearch/IndxCloudApi) — self-host server template (ASP.NET Core)
  - [OpenAPI spec](https://v5.cloud.indx.co/swagger/v2.0-beta/swagger.json) — machine-readable API definition
- **Node.js / TypeScript**
  - [@indxsearch/indx-types](https://www.npmjs.com/package/@indxsearch/indx-types) — TypeScript type definitions
- **Frontend**
  - [indx-intrface](https://github.com/indxSearch/indx-intrface) — React search UI components (@indxsearch/intrface)
