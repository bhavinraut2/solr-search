# IFB Solr Search — Technical Flow Documentation

**Solr version:** 8.11.4 &nbsp;|&nbsp; **Core:** `IFB` &nbsp;|&nbsp; **Config files:** `solrconfig.xml`, `managed-schema`, `synonyms.txt`, `stopwords.txt`

This document describes exactly how the `IFB` core is built and how a query flows through it today, after the fixes applied during the last review cycle. It is meant for engineers maintaining or extending the search config.

---

## 1. Architecture Overview

```
Client (web/app/GraphQL layer)
        │
        ▼
 Solr HTTP endpoints (this doc, section 3)
        │
        ▼
 edismax query parser  →  qf / pf / mm / bq / boost  →  BM25 scoring (SchemaSimilarity)
        │
        ▼
 Response (docs + score) ── SKUs handed back to GraphQL/CSR for live price & stock (pricing is NOT done in Solr)
```

Solr holds **catalog + content search data only**: product titles, features, categories, capacity, spec text, plus Support/Page/Blog content. Real-time price and stock-by-pincode are resolved downstream via GraphQL using the SKU/id returned by Solr — Solr's own `price`/`productPrice` fields are not used for filtering or sorting (confirmed intentional, not a gap).

### Data snapshot (current index)
- **1999 docs** total: `contentType` = Product (665), Support (736), Page (228), Blog (370)
- `productType` (Product docs only): Products (565), Accessories (47), Essentials (53)
- `productCategory`: Air Conditioners, Washing Machine, Refrigerators, Microwave, Dishwasher, Chimney, Stabilizer, Built in Hob/Oven, Clothes Dryer, Fabric/Dish/Kitchen/Laundry/Machine Care, etc.
- Field population health: `productCapacity` ~82%, `productSmallFeature` ~99%, `productSubCategory` ~70%, `productType`/`productSegment` 100% (Product docs).

---

## 2. Data Model — Key Fields & Field Types

| Field | Type | Notes |
|---|---|---|
| `id` | string | unique key |
| `contentType` | string (exact, case-sensitive) | `Product` / `Support` / `Page` / `Blog` |
| `title` | text_autocomplete | primary display/search text |
| `name` | text_autocomplete | SKU code for real products, but **descriptive text for many accessories** (e.g. `"Descal - Washing Machine"`) — a known ranking risk, mitigated by boost weighting (§5) |
| `sku` | text_general_singleval | |
| `productCategory` | **string** (case-sensitive, exact) | e.g. `"Stabilizer"` — will **not** match a lowercase query directly |
| `productCategory_search` | text_autocomplete (copyField of `productCategory`) | the analyzed/lowercased version — this is the field that actually powers category matching |
| `productSubCategory` | text_general_singleval | e.g. `"Top Load Washing Machine"` (~70% populated) |
| `productSubCategory_search` | text_autocomplete (copyField) | analyzed version |
| `productCapacity` / `productCapacity_value` | text_capacity_exact / pdoubles | e.g. `"1 Ton"`, `1700 W` — highest qf/pf weight |
| `productFeature`, `productSmallFeature` | text_autocomplete | marketing feature text, moderate weight |
| `productDescription` | text_no_edgegram | long-form spec text, low weight (noise-prone) |
| `productType` | **string** | `Products` / `Accessories` / `Essentials` (Product docs only) |
| `productParentCategory` | string | `Living Solutions`, `Kitchen Solutions`, `Laundry Solutions`, `Accessories`, `Essentials` |
| `spellWords` | spellcheck_text | dedicated field feeding the spellchecker dictionary |
| `productPrice` | text_general_singleval, **indexed=false** | stored/display only, not searchable |
| `price` | pdoubles | present in schema but unpopulated — not used (pricing handled externally) |

### Why `productCategory` boosts in `qf`/`pf` are inert
`productCategory` is a `string` field — no analyzer, case-sensitive exact match. A user query like `stabilizer` (lowercase) will **never** match the stored value `Stabilizer`. Verified: `productCategory:stabilizer` → 0 hits, `productCategory_search:stabilizer` → 30 hits. The `productCategory`/`productSubCategory` (raw) qf/pf entries are left in place for completeness but contribute nothing for normal queries — all real category-match relevance flows through the `_search` copy fields.

---

## 3. Analyzer Chains — And Why Filter Order Matters

Three field types carry almost all of the search weight: `text_autocomplete`, `text_no_edgegram`, `text_capacity_exact`. Their analyzer chains combine **graph-producing filters** (`SynonymGraphFilterFactory`, `WordDelimiterGraphFilterFactory`) that must be ordered correctly, or the resulting boolean query silently corrupts.

**`text_autocomplete` (title, name, productFeature, productSmallFeature, `_search` copy fields):**
```
INDEX: Standard → Stop → Lower → WordDelimiterGraph → EdgeNGram(min=3, max=15, preserveOriginal=true) → FlattenGraph
QUERY: Standard → Stop → WordDelimiterGraph → SynonymGraph(expand=true) → Lower → FlattenGraph
```
- `preserveOriginal=true` on `EdgeNGramFilterFactory` was added so literal short words (e.g. `"TV"`, 2 chars) aren't silently dropped (EdgeNGram's default minGram=3 drops anything shorter). **This requires a full reindex to take effect on existing documents** — a core reload alone only affects newly indexed docs.
- **Golden rule learned this cycle:** any graph-producing filter (`SynonymGraphFilterFactory`, `WordDelimiterGraphFilterFactory`) that is followed by another filter must either (a) be immediately followed by `FlattenGraphFilterFactory`, or (b) be the last filter in the chain. Putting `WordDelimiterGraphFilterFactory` *after* `SynonymGraphFilterFactory` without flattening produced nonsensical AND-heavy boolean queries instead of clean OR alternatives (see §8, Bug #1).

**`text_capacity_exact` (productCapacity):** the reference-correct pattern — `WordDelimiterGraph → SynonymGraph → Lower → FlattenGraph`, both index and query side.

**`suggest_text` (dedicated, added this cycle):**
```
Standard → Lower   (no stopwords, no synonyms)
```
Used only by the `/suggest` component. Synonyms/stopwords were deliberately excluded — see §8, Bug #10.

---

## 4. Endpoints Reference

| Endpoint | Purpose | Key params |
|---|---|---|
| `GET /solr/IFB/select` | Main relevance search (edismax) | `q`, `rows`, `fq`, `facet.field`, `sort` |
| `GET /solr/IFB/query` | Plain JSON query handler, `df=_text_` | rarely used directly |
| `GET /solr/IFB/spell` | Dedicated spellcheck-only lookup (no full search overhead) | `q`, `spellcheck.count` |
| `GET /solr/IFB/suggest` | Autocomplete-as-you-type | `suggest.q`, `suggest.dictionary`, `suggest.build` |
| `GET /solr/IFB/terms` | Raw term/document-frequency introspection | `terms.fl` |
| `POST /solr/admin/cores?action=RELOAD&core=IFB` | Apply `solrconfig.xml`/`managed-schema` changes without restart (query-time changes only) | |

### `/select` — defaults baked into `solrconfig.xml`
```
defType=edismax
qf = productCapacity^100 productSubCategory^100 productCategory_search^50 title^15 name^5
     productCategory^10 productFeature^8 productSmallFeature^6 sku^6
     productSubCategory_search^25 pageContent^1 productDescription^1
pf = productCapacity^200 productSubCategory^180 productCategory_search^80 title^30 name^10
     productCategory^20 productFeature^15 productSmallFeature^10 productSubCategory_search^40
ps = 0
mm = 60%
bq = contentType:Product^50
     productType:Products^50
boost = mul(if(termfreq(contentType,'Product'),3,1), if(termfreq(productType,'Products'),2,1))
facet = true, facet.mincount=1, facet.limit=20   (⚠ no facet.field default — see note below)
spellcheck = true (last-components: spellcheckDirect)
```
> **Note:** `facet=true` is on by default but no `facet.field` is set, so `facet_counts.facet_fields` is empty unless the caller explicitly passes `facet.field=...` (e.g. `facet.field=productType`). This is intentional/left as-is per the last review — pass `facet.field` explicitly per request.

### `/spell` — `spellcheckDirect` component
- `classname = solr.DirectSolrSpellChecker`, dictionary field = `spellWords`
- `filterQuery = contentType:Product OR contentType:Support` — Blog/Page content is excluded from spelling suggestions
- `queryAnalyzerFieldType = spellcheck_text`

### `/suggest` — 4 dictionaries
| Dictionary | Field | lookupImpl | Notes |
|---|---|---|---|
| `mySuggester` | `name` | AnalyzingInfixLookupFactory | contextField=`contentType` |
| `titleSuggester` | `title` | AnalyzingInfixLookupFactory | contextField=`contentType` |
| `categorySuggester` | `productCategory` | FuzzyLookupFactory | case-insensitive via `lowercase` analyzer; does **not** support contextField |
| `subCategorySuggester` | `productSubCategory` | AnalyzingInfixLookupFactory | contextField=`contentType`; **not** in the handler's default dictionary list — must be requested explicitly with `suggest.dictionary=subCategorySuggester`, and needs one manual `suggest.build=true` the first time it's used |

Default request params: `suggest.count=10`, `suggest.onlyMorePopular=true`, `suggest.cfq=Product`.

> **Critical syntax note:** `suggest.cfq` must be the **raw context value** (`Product`), *not* a query expression (`contentType:Product`). Using query syntax silently returns zero results for every dictionary that has a `contextField` configured.

All three `AnalyzingInfixLookupFactory`/`FuzzyLookupFactory` suggesters currently report `weight=0` for every suggestion (no `weightField` is configured — none of the source fields are numeric), so result order is effectively insertion order, not popularity. `buildOnCommit=true` keeps dictionaries in sync automatically after real commits; a bare core `RELOAD` does **not** rebuild them — use `suggest.build=true` on a request to force a rebuild after a config change.

---

## 5. Relevance / Ranking Pipeline — Step by Step

For a query like `q=refrigerator`:

1. **edismax** splits/analyzes the query per field listed in `qf`, builds a `DisjunctionMaxQuery` per query term across all `qf` fields (max, not sum, of per-field scores — classic "dismax" tie-breaking).
2. **`mm=60%`** determines how many of the query's optional term-clauses must match. For a 1-word query this reduces to "the term must match somewhere"; for 2 words it typically requires all/most clauses depending on Solr's rounding rules.
3. **`pf`** phrase fields add extra score when the query terms appear **adjacent** in a field (no separate matching requirement — purely a score boost).
4. **`bq`** clauses (`contentType:Product^50`, `productType:Products^50`) add a flat, non-required score bonus — they don't filter, they bias ranking toward real product listings over Blog/Support/Page and over Accessories/Essentials.
5. **`boost` function** (`mul(...)`) is a **multiplicative** post-processing step applied to the final additive score — real Products get up to 6× (3× contentType × 2× productType), Accessories/Essentials get 3×, Blog/Support/Page get 1×.
6. **BM25** (`SchemaSimilarity`, Solr 8 default) scores each individual field match using term frequency + inverse document frequency + field-length normalization. This is why very rare terms in short fields (e.g. an accessory's `name` field, or a feature phrase appearing in only 6 documents) can occasionally outscore a common, contextually-correct match in a longer field — BM25 rewards rarity aggressively. See §8, Bugs #3, #7 for real examples this caused and how they were mitigated (never fully eliminated — it's an inherent property of BM25, not a bug to "fix" outright).

---

## 6. Search Condition Walkthroughs (verified against the live index)

| Condition | Example query | Result |
|---|---|---|
| Exact category word | `refrigerator` | Real refrigerators rank top; 438 total matches |
| Multi-word exact | `washing machine` | Real washing machines rank top; 502 matches |
| Partial/prefix word | `stab` | Matches "Stabilizer" via `productCategory_search` boost (fixed — previously matched an unrelated refrigerator's "Stable Cooling" feature text) |
| Common misspelling | `refridgerator`, `stablizer`, `dishwaser` | Correctly resolved via `synonyms.txt` typo-variant entries |
| Joined/no-space words | `washingmachine`, `airconditioner` | Resolved via explicit `=>` synonym mappings (WordDelimiter cannot split plain lowercase compounds on its own) |
| 2-letter abbreviation | `TV` | **Still 0 today** — fixed in schema (`preserveOriginal=true`) but requires a full reindex to take effect on existing docs |
| Capacity search | `1.5 ton ac`, `7kg`, `331l` | Matches via `productCapacity`/`productCapacity_value` (highest qf/pf weight); works with or without space/unit variants via capacity-unit synonyms |
| Exact SKU/model | `CI145GD21RGN1` | Direct match via `name` field |
| Accessory vs. real product | `descaler` | Finds the accessory directly when searched by name; does not leak into generic category searches like `washing machine` |
| Content-type demotion | any generic term | Product content ranks above Support/Page/Blog via `bq`+`boost`; Accessories/Essentials rank below core Products via `productType` boost |

---

## 7. Change Log — Fixes Applied This Review Cycle

1. **Synonym+WordDelimiter graph corruption** in `text_autocomplete`/`text_no_edgegram` query analyzers — reordered filters + added `FlattenGraphFilterFactory`. (Root cause of "Air Conditioner" returning 1 result instead of 181.)
2. **`productType` boost was dead weight** (field unpopulated at the time) — temporarily switched to `contentType`, then restored `productType` alongside `contentType` once real data arrived with proper values.
3. **Accessories outranking real appliances** — lowered `name` field boost; later fully resolved by boosting `productCategory_search`/`productSubCategory_search` (see #7).
4. **`"ref"` synonym token** collided with EdgeNGram prefixes of unrelated words (e.g. "Refresher") — removed from `synonyms.txt`.
5. **`/suggest` returned zero results by default** — missing `contextField` on infix suggesters, and `suggest.cfq` used invalid query syntax instead of a raw context value. Fixed both; also fixed `categorySuggester` case-sensitivity (`string` → `lowercase` analyzer).
6. **`"voltage"` synonym token** collided with AC/refrigerator "voltage protection" spec text, causing `stabilizer` searches to surface refrigerators — removed from `synonyms.txt`.
7. **`productCategory`/`productSubCategory` boosts were inert** (case-sensitive `string` fields never match lowercase queries) while the real working fields (`_search` copies) were boosted only `^1` — raised `productCategory_search` to `^50`/`^80` and `productSubCategory_search` to `^25`/`^40`. This was the single highest-impact fix of the cycle.
8. **Joined/compound-word queries returned 0** (`washingmachine`, `airconditioner`) — added explicit synonym mappings.
9. **2-letter tokens (`"TV"`) silently unindexed** — added `preserveOriginal="true"` to `EdgeNGramFilterFactory` (requires reindex).
10. **Suggester silently failed for words that are also synonym entries** (`front`, `top` returned 0 even as complete, correctly-spelled words) — root cause: `text_general`'s synonym-expanding query analyzer was reused to *build* the suggester dictionary, producing a token graph `AnalyzingInfixSuggester` can't resolve. Fixed with a dedicated, synonym-free `suggest_text` analyzer. Also added `subCategorySuggester`.

---

## 8. Operational Runbook

- **Changed `solrconfig.xml` only** (qf/pf/bq/boost, request handler defaults, suggester config) → `POST /solr/admin/cores?action=RELOAD&core=IFB` is sufficient.
- **Changed `synonyms.txt` / `stopwords.txt`** → core RELOAD is sufficient (loaded fresh per reload).
- **Changed `managed-schema` field *type* definitions that affect INDEX-time analysis** (e.g. `EdgeNGramFilterFactory` `preserveOriginal`) → core RELOAD applies to *new* documents only; existing documents need a **full reindex** to gain the new tokens.
- **After adding/changing a suggester** → RELOAD, then explicitly call `/suggest?suggest.q=...&suggest.dictionary=<name>&suggest.build=true` once to force the initial build. After that, `buildOnCommit=true` keeps it current automatically on real commits (not on bare reloads).
