# Search or Ask: reference

Search or ask lets one search field answer questions. On each typing pause a **judge** (Jev, or OpenAI's Decisions API) reads the typed text and the top results and scores it: a lookup, a question, or on a catalogue a description of what to list. The page shows an offer to ask; when the visitor asks, an **answer model** writes the answer from the dataset's records only, citing them. Indx Server only; the C# library does not have it.

Human docs: [docs.indx.co/guides/search-or-ask](https://docs.indx.co/guides/search-or-ask). API: the *Search or Ask* section of the HTTP reference.

---

## Decide whether it fits

| Kind (`kind`) | Records | Answers |
|---|---|---|
| `documentation` | Written to explain: help, manuals, policies, long descriptions | Combine what several pages say |
| `catalogue` | Facts about things: products, places, films | A description becomes filters; a question about one item is answered from its fields |

Do **not** propose it for a dataset of identifiers or codes. Low question volume in the statistics is weak evidence against it: a search that cannot answer teaches visitors to type keywords.

---

## Turn it on: three switches

1. **Instance:** a model key. Console *Admin > Settings > Experimental* (show Search or ask, the keys), or `Indx:SearchOrAsk:ApiKey`. One OpenRouter key serves the judge and the answers; Anthropic or any OpenAI-compatible endpoint can serve the answers, with a judge key beside it. Without a key the routes answer `409 searchOrAskUnavailable`.
2. **Dataset shown:** a team admin, the dataset's Options tab, or `PUT …/search-or-ask/shown { "shown": true }`. Until then every route answers `404`.
3. **Dataset enabled:** an editor, `"enabled": true` in the settings (`PUT …/search-or-ask`, Full key). Off: `409 searchOrAskDisabled`.

**Tell the user** that typed text then goes to the model providers, and their privacy notice must say so.

---

## Settings (`GET`/`PUT /api/teams/{team}/datasets/{ds}/search-or-ask`)

`PUT` replaces the whole object, so read, change, write back. Applies from the next call; no reindex.

| Setting | Default | Notes |
|---|---|---|
| `enabled` | `false` | |
| `kind` | `documentation` | `documentation` or `catalogue` |
| `siteDescription` | dataset description | One or two sentences: who reads, what the records hold. Every judgement and answer reads it |
| `titleField`, `sectionFields`, `snippetFields` | automatic | What the judge reads of a result. Snippets are prose, never identifiers or URLs |
| `filterNotes` | none | Catalogue: what a value means, one per line (`barneskole: grades 1 to 7`) |
| `knowledge` | none | Markdown facts no record holds (scope, what a rating means, VAT, delivery). Answers may state it as "the site says"; never read by the judge; 8,000 characters max. A Search key reads the settings **without** it |
| `answers` | `best` | `best` (stronger model; on a catalogue an agent that searches, sorts and filters) or `fast` (about 1/20 of the cost) |
| `answerInstructions` | none | Voice and house rules; cannot loosen grounding. 1,500 characters max |
| `answerLanguage` | visitor's | One language for every answer |
| `askPhrases` | English | `{ leadIns, frames, links, articles }`, see below |
| `refocusWhileTyping`, `inviteOnAskPhrase` | `true`, `true` | |
| `thresholds` | 0.65 / 0.45 / 0.40 / 0.30 | `strong`, `strongHide`, `invite`, `inviteHide` |
| `cutWhileAsking`, `cutShareOfTop`, `cutTopFloor` | `true`, 0.5, 0.2 | Keep only strong matches while the full offer is up |
| `strictCoverageWhileAsking` | `true` | Whole words only on the page's searches while asking |
| `filtersWhileTyping` | `off` | Catalogue: `off`, `facets` (instant), `model` (about 1.5 s) |
| `dailyCallBudget` | 5000 | Calls per UTC day; `0` stops all calls |

`GET` also returns `effective` (what it runs with after defaults, the Ask phrases in effect) and `available` (the instance has a key).

### Ask phrases

While a question is typed, the list searches what it is about: "help me find the movie about the green ogre" searches `green ogre`. Groups, matched in order from the start: lead-ins ("help me find"), frames, i.e. what is asked for ("the movie"), links ("about", "with"), articles. Nothing is taken unless the start has a question's shape and a word of 3+ letters is left. A frame also comes off the end ("show me green ogre movie" → `green ogre`) when two words stand before it.

For a catalogue, **add the records' own names as frames** ("the movie", "a film", "movie", "the product") and lead-ins in the site's language. With all four lists empty the English defaults apply; once any list has a phrase, the four are the dataset's own (an empty list then takes nothing of its kind).

### Catalogue fields

The judge's filters and the agent work from **Filterable** fields. A text field becomes a filter only when it is also **Facetable** (its values come from the facets) and has at most 500 values; a filterable field with more (a director, a writer) is not a filter, but the agent reads it from every record found and searches its words. Make the fields questions are about Filterable, and Sortable where a question asks for an order ("the newest", "the largest").

---

## Front end with intrface (`@indxsearch/intrface` 3.13+)

```tsx
import { SearchProvider, SearchInput, SearchResults, AskPanel, useSearchOrAsk, useFilterSuggestion } from '@indxsearch/intrface';

function Search() {
  const sor = useSearchOrAsk();                         // settings, judge on pause, Ask phrases on the searches
  const suggestion = useFilterSuggestion({ auto: sor }); // a catalogue's filters on the list
  const [question, setQuestion] = useState<string | null>(null);
  return (
    <>
      <SearchInput onAsk={setQuestion} searchOrAsk={sor} />
      {question && <AskPanel question={question} followUps onFilters={suggestion.apply} />}
      <SearchResults fields={['title', 'description']} searchOrAsk={sor} />
    </>
  );
}
// inside <SearchProvider url team dataset preAuthenticatedToken={SEARCH_KEY}>
```

- Where Search or ask is off or unavailable, `useSearchOrAsk` stays quiet and the page is an ordinary search: safe to ship before it is enabled.
- `AskPanel fill` takes its container's height (a panel beside the results); `SearchInput askOffer="below"` puts the offer under the field.
- **Use a Search key in the browser.** Never a Read or Full key.

---

## Front end of your own (HTTP)

All under `/api/teams/{team}/datasets/{ds}/search-or-ask`, Search key:

- `POST /intent { text }` on a **typing pause**, not every keystroke: `{ p, answerable, criteria, offer: none|invite|strong, searchText, reason?, … }`. `p` 0 with a `reason` when the judge is slow or failing; it never fails the call. Keep an offer up until `p` falls below the `…Hide` threshold.
- **Search `searchText`**, not the typed text, when refocus is on, and send the typed text as `?typed=` on that search so the statistics count the typing once.
- `POST /ask { question, history? }` → server-sent events `status`, `sources` (`[{ n, documentKey, title }]`), `filters` (catalogue), `text` (`{ delta }`, markdown citing `[n]`), `done`, or `error` (`nothingFound`, `upstreamBusy`, `failed`). Header `Indx-Ask-Id`: send it as the `queryId` of a select when a cited source is opened. `history`: up to three `{ question, answer }` turns.
- `POST /filters { text }` (catalogue, when `criteria` is strong): `{ filters: [{ field, value | min, max }], text, sort? }`.
- Pass `?session=` (one id per page load), `?source=` and `?queryId=` as on searches.

Limits: text 1 to 300 characters, question 1 to 1,000. `429` with `Retry-After` over the per-address limit or the daily budget (`searchOrAskBudgetSpent`, resets at midnight UTC).

---

## Costs and statistics

A judgement is about $0.00005; a `best` answer about $0.035 on a film catalogue, a `fast` one about $0.0015. The answers are the bill. `GET …/statistics/search-or-ask?days=30` (Read key) returns offers, questions, how they ended (unanswered first), sources opened, cost and calls against the budget; the console's Statistics tab shows the same.

## Pitfalls

- Do not put a Full key in a `NEXT_PUBLIC_` (or any browser) variable to reach these routes; a Search key reaches them all.
- Do not expect the judge to read `knowledge`: put a site's scope in `siteDescription` too.
- A catalogue with no Filterable, Facetable fields gets no filters, and its agent can only search words.
- The tester on the console's Search or ask tab runs any settings, saved or not, against the budget: use it to tune before switching on.
