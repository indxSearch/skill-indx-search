# Search or Ask: reference

Search or ask lets one search field answer questions. On each typing pause a **judge** (Jev, or OpenAI's Decisions API) reads the typed text and the top results and scores it: a lookup, a question, or on a catalogue a description of what to list. The page shows an offer to ask; when the visitor asks, an **answer model** writes the answer from the dataset's records only, citing them. Indx Server only; the C# library does not have it.

Human docs: [v6.docs.indx.co/guides/search-or-ask](https://v6.docs.indx.co/guides/search-or-ask). API: the *Search or Ask* section of the HTTP reference.

---

## Decide whether it fits

| Kind (`kind`) | Records | Answers |
|---|---|---|
| `documentation` | Written to explain: help, manuals, policies, long descriptions | Combine what several pages say |
| `catalogue` | Facts about things: products, places, films | A description becomes filters; a question about one item is answered from its fields |

Do **not** propose it for a dataset of identifiers or codes. Low question volume in the statistics is weak evidence against it: a search that cannot answer teaches visitors to type keywords.

---

## Turn it on: three switches

1. **Instance:** model keys, one per provider. Console *Admin > Settings > Experimental* (show Search or ask, the keys), or `Indx:SearchOrAsk:Keys:OpenRouter` / `Keys:OpenAi` / `Keys:Anthropic` / `Keys:TypeSafe` (the older `Indx:SearchOrAsk:ApiKey` still works). The judge and the answers each name a provider and use its key. One OpenRouter key serves both; so does one OpenAI key (OpenAI answers plus the Decisions API as judge, with EU data residency and an Ultrafast answer speed). Anthropic or any OpenAI-compatible endpoint can serve the answers. Without a key the routes answer `409 searchOrAskUnavailable`.
2. **Dataset shown:** a team admin, the dataset's Options tab, or `PUT …/search-or-ask/shown { "shown": true }`. Until then `intent`, `filters` and `ask` answer `404`, and `GET` says `available: false`.
3. **Dataset enabled:** an editor, `"enabled": true` in the settings (`PUT …/search-or-ask`, Full key). Off: `409 searchOrAskDisabled`.

**Tell the user** that typed text then goes to the model providers, and their privacy notice must say so.

---

## Settings (`GET`/`PUT /api/teams/{team}/datasets/{ds}/search-or-ask`)

`PUT` replaces the whole object, so read, change, write back. Applies from the next call; no reindex.

| Setting | Default | Notes |
|---|---|---|
| `enabled` | `false` | |
| `kind` | `documentation` | `documentation` or `catalogue` |
| `siteDescription` | a sentence naming the dataset | One or two sentences: who reads, what the records hold. Every judgement and answer reads it |
| `titleField`, `sectionFields`, `snippetFields` | chosen from a sample of the records (`effective`) | What the judge reads of a result. Automatic follows the rule in *What the judge reads* below; override it when visitors ask about a field it left out |
| `filterNotes` | none | Catalogue: what a value means, one per line (`barneskole: grades 1 to 7`) |
| `knowledge` | none | Markdown facts no record holds (scope, what a rating means, VAT, delivery). Answers may state it as "the site says"; never read by the judge; 8,000 characters max. A Search key reads the settings **without** it |
| `answers` | `best` | `best` (the instance's stronger model) or `fast` (about 1/20 of the cost) |
| `answerModel` | none | Another model the instance offers (`effective.answerModels`), in place of the one `answers` names; any other is refused with `400` |
| `searches` | follows `answers` | Catalogue only: `several` (an agent that searches, sorts and filters) or `one` (one search, a summary). Unset: `best` several, `fast` one. The agent filters by expression (OR, NOT, IN), is told whether its words matched exactly or within edits, reads records whole and reads their `public` and `answer` cold storage, never `private` |
| `answerInstructions` | none | Voice and house rules; cannot loosen grounding. 1,500 characters max |
| `answerLanguage` | visitor's | One language for every answer |
| `mode` | `soft` | How forward the offer is, see below |
| `dailyProactiveAnswers` | 500 | Answers `proactive` starts without a click per UTC day; past it the mode reports `inviting` until midnight |
| `thresholds` | 0.65 / 0.45 / 0.40 / 0.30 | `strong`, `strongHide`, `invite`, `inviteHide`; also `hint` (null: the judge's own, Jev 0.25 catalogue / 0.31 documentation, the Decisions API 0.05) `finished` (0.5: the full offer waits for a whole question, the invitation until then) and `found` (0.3: under the full offer, dim the list when `found` is below it; 0 is off) |
| `strictCoverageWhileAsking` | `true` | Whole words only on the page's searches while asking |
| `filtersWhileTyping` | `off` | Catalogue: `off`, `facets` (instant), `model` (about 1.5 s) |
| `dailyCallBudget` | 5000 | Calls per UTC day; `0` stops all calls |

`GET` also returns `effective` (what it runs with after defaults, `answerModel`, `searches`, and the offered `answerModels`) and `available` (the instance has a key and the dataset shows it).

### Mode

| `mode` | What the visitor sees |
|---|---|
| `strict` | Only the full offer, once the judge is sure. Where a wrong offer costs more than a missed one |
| `soft` (default) | The quiet invitation early, the full offer once sure |
| `open` | An Ask button whenever something is typed; Enter asks whatever the judge says |
| `inviting` | Open, plus "For a quick answer, type a bit more" from `thresholds.hint` up, and on a sentence (three words, or a `?`) before the judge has read it. Teaches visitors the field answers |
| `proactive` | Inviting, and the answer starts on its own once the offer is strong, `finished` passes its threshold and the visitor has been still a second |

Recommend `soft` unless the user asks otherwise. `proactive` spends an answer per finished question nobody clicked for: suggest it for documentation with a budget to match, not for a catalogue. A page can choose its own mode (`useSearchOrAsk({ mode })`), such as `strict` on a checkout page; `proactive` applies only where the dataset is set to it.

### What the judge reads of a result

The judge sees five results, each as its title, its section (the section fields' values joined by `; `, a list's items by `, `, never cut) and its snippet (the snippet fields as one line, cut at 240 characters). It can tell a list that holds what the visitor means from one that only shares the words only when what the visitor names is in front of it.

**Automatic** (the fields left unset) reads a sample of the records: the title is the highest-weighted searched field, the snippet the prose (long sentences, at most two, none when there is none), the section up to six short facts ranked by their roles, never codes, a number only as a year, nothing nearly every record has, nothing another field already says. Measured as good as fields chosen by hand on films, furniture and schools. Read `effective` to see what it chose, and set the fields only to change that.

**The rule for `sectionFields`, Automatic's or yours: the short facts visitors name besides the item's name.** A field belongs there when visitors' requests name its values ("who wrote", "gray", "private schools") and the title does not already show them.

- **Find them in the search statistics** where the dataset has real searches: run the most frequent searches with `enableMatchReport` and see which fields their words land on, besides the title. Without statistics, imagine the visitors and what they would ask.
- **When unsure, include the field.** A missing field costs a lot, an unneeded one little or nothing, and the judge's time does not change measurably. Tested on films, furniture and schools (Nov 2026): on furniture, colour, material and style in the section raised how well the judge told right lists from wrong ones from 0.72 to 0.94 (area under the curve).
- **Values that read on their own.** The judge sees values without field names, so codes and ids tell it nothing. A true/false field is shown as its name when true and left out when false, so it works when the name reads (`isPrivateSchool`, `upholstered`), not when it is cryptic (`flag2`).
- **Not the same thing twice** (a category and a class holding the same value).
- **A list of short items can go in whole** (ten names); leave out a list of hundreds.
- **In the section, not the snippet**: the snippet is cut at 240, so a field after a long description is not read.

**`snippetFields`: the prose**, longest first. Never identifiers, URLs or codes. **`titleField`**: the name a visitor would recognise.

For example, on a film site where visitors ask who made a film and who is in it: section `genres`, `release_year`, `director`, `writers`, `actors`, snippet `description`. On a shop where they search by colour and material: section `class`, `color`, `material`, `style`, snippet `description`.

### Catalogue fields

The judge's filters and the agent work from **Filterable** fields. A text field becomes a filter only when it is also **Facetable** (its values come from the facets) and has at most 500 values; a filterable field with more (a director, a writer) is not a filter, but the agent reads it from every record found and searches its words. Make the fields questions are about Filterable, and Sortable where a question asks for an order ("the newest", "the largest").

---

## Front end with intrface (`@indxsearch/intrface` 4.0+, for the Indx 6 server)

Three ready-made shapes; recommend one before wiring parts by hand. All go inside `<SearchProvider url team dataset preAuthenticatedToken={SEARCH_KEY}>`, follow the dataset's mode, and are an ordinary search where Search or ask is off.

| Shape | For | Asking |
|---|---|---|
| `SearchPage` | a whole search page: title, filters left, field + results right, no background | switches to a full-width ask mode (field and filters hidden, picks under the answer); *Back to search* puts the question back in the field |
| `SearchOrAsk` | a search component inside the customer's page | a column beside the results, two buttons switching Filters / Assistant; the list becomes the picks |
| `AskChat` | a help or docs site | a launcher in the corner opens a chat window; `actions` for "Talk to a person" |

```tsx
<SearchPage title="Films" filters={<ValueFilterPanel field="genres" label="Genres" />}
            renderResult={item => <h3>{item.title}</h3>} examples={['Films about grief']} />
```

`SearchPage`/`SearchOrAsk` share `filters`, `renderResult`, `examples`, `results` (SearchResults' other props), `restore` (the page's own start, rerun when Clear takes off an order the assistant set), `words`, `labels`, `mode`.

A page of its own, from the parts:

```tsx
const sor = useSearchOrAsk();                    // settings, judge on pause
const flow = useAskFlow({ searchOrAsk: sor });   // asking, the wait, typing back, Clear
<SearchInput askOffer="door" {...flow.inputProps} />
{flow.offerShown && <AskOffer searchOrAsk={sor} onAsk={flow.ask} />}
{flow.question && <AskPanel key={flow.run} {...flow.panelProps} followUps />}
{!flow.waiting && <SearchResults fields={['title', 'description']} searchOrAsk={sor} />}
```

- Where Search or ask is off or unavailable, `useSearchOrAsk` stays quiet and the page is an ordinary search: safe to ship before it is enabled.
- Two surfaces, one job each: `SearchInput askOffer="door"` is a quiet *Ask* in the field wherever asking is allowed, same words always; `AskOffer` is the row above the results, only for the early hint, the sure offer (`offerKind`: a question, not among the results, a description to explore) or *Thinking* before a proactive answer. `words` rewords it per offer. Enter asks on the sure offer only, otherwise searches. Default `askOffer="chip"` keeps the whole offer in the field, for a page without `AskOffer`.
- `useAskFlow` waits after asking for the answer and then for the search with the filters it read, until the list settles, so the question's own list does not flash first. Without it, wire `onAsk`, `proactive` and `useFilterSuggestion` yourself.
- `useSearchOrAsk` exposes `door`, `offerRow`, `offerKind`, `enterAsks` for a page drawing its own surfaces.
- `AskPanel fill` takes its container's height; `showSearched={false}` leaves out what the answer searched with.
- In `proactive` the field starts the answer on its own; `useAskFlow` passes `proactive` on so the statistics count those answers apart.
- On a catalogue with several searches, the answer's picks (the records it chose, each with `why`) become the list by themselves: `AskPanel` puts them on the provider, `SearchResults` lists them, `SearchInput` drops Nearest and No results. Typing, Clear or a new conversation gives the list back. Not for a proactive answer; `listPicks={false}` keeps them in the panel, `onPicks` reports them.
- `SearchResults searchOrAsk={sor}` never cuts the list; it dims it, with a line above, when the judge reads what is asked about as not in it (`sor.notFound`).
- While asking is held (`sor.held`: the day's budget spent, or the provider refusing answers, reason `paused`) nothing is offered in any mode and Enter searches. `AskPanel` shows nothing for a refused proactive start, and "No more answers today" for a spent budget.
- Deprecated, still working: `QuickAnswer` (use `AskOffer`), `askOffer="below"` (use `"door"`), `sor.quick` and `sor.askAnytime`.
- **Use a Search key in the browser.** Never a Read or Full key.

---

## Front end of your own (HTTP)

All under `/api/teams/{team}/datasets/{ds}/search-or-ask`, Search key:

- `POST /intent { text }` on a **typing pause**, not every keystroke: `{ p, answerable, found, criteria, finished, offer: none|invite|strong, mode, reason?, … }`. `found`: whether what is asked about is in the results, answer or not; dim the list under a strong offer when it is below `thresholds.found` (null from the Decisions API: dim nothing). Follow the returned `mode`, not the one you read at load: `proactive` comes back `inviting` once the day's proactive answers are spent. `p` 0 with a `reason` when the judge is slow or failing; it never fails the call. Reason `paused`: the provider refuses answers, so offer nothing at all, not even open's Ask button, until a judgement comes back without it. Keep an offer up until `p` falls below the `…Hide` threshold.
- `POST /ask { question, history? }` → server-sent events `status` (`{ step, kind, search?, count? }`: `step` an English sentence, `kind` `read`/`search`/`pick` with the parts, `search: { filters, filter, text, sort, found, more }`, `count` picked; show your own wording from the parts), `sources` (`[{ n, documentKey, title }]`), `filters` (catalogue, one search), `picks` (catalogue, several searches: `{ picks: [{ n, documentKey, title, why }], search }`, the records the answer chose; list them as the page's results while the conversation is open), `text` (`{ delta }`, markdown citing `[n]`), `done`, or `error` (`nothingFound`, `upstreamBusy`, `failed`). Header `Indx-Ask-Id`: send it as the `queryId` of a select when a cited source is opened. `history`: up to three `{ question, answer, shown }` turns, `shown` being what the turn put on the page (`{ filters, text, sort }` or `{ picks: [{ documentKey, title }] }`). Send `"proactive": true` when the page starts the answer itself; outside the `proactive` mode, or past the day's proactive answers, that answers `429 searchOrAskBudgetSpent`: show nothing, nobody asked.
- `POST /filters { text }` (catalogue, when `criteria` is strong): `{ filters: [{ field, value | min, max }], text, sort? }`.
- Pass `?session=` (one id per page load), `?source=` and `?queryId=` as on searches.

Limits: text 1 to 300 characters, question 1 to 1,000. `429` with `Retry-After` over the per-address limit or the daily budget (`searchOrAskBudgetSpent`, resets at midnight UTC: offer nothing until then). Show a visitor your own words for a `409`; the problem's `detail` is written for the owner.

---

## Costs and statistics

A judgement is about $0.00005; a `best` answer about $0.035 on a film catalogue, a `fast` one about $0.0015. The answers are the bill. `GET …/statistics/search-or-ask?days=30` (Read key) returns offers, questions, how they ended (unanswered first), sources opened, questions asked again in other words within five minutes (`askedAgain`, and the pairs in `rephrased`; needs `?session=`), the needs those bundle into (`needs`: wordings, how visits ended, records landed on), answers proactive wrote on its own (`proactiveAnswers`, apart from `asks`), cost and calls against the budget; the console's Statistics tab shows the same.

## Pitfalls

- Do not put a Full key in a `NEXT_PUBLIC_` (or any browser) variable to reach these routes; a Search key reaches them all.
- Do not expect the judge to read `knowledge`: put a site's scope in `siteDescription` too.
- A catalogue with no Filterable, Facetable fields gets no filters, and its agent can only search words.
- The tester on the console's Search or ask tab runs any settings, saved or not, against the budget: use it to tune before switching on.
