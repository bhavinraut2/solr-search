# IFB Website Search — How It Works (Business Overview)

This document explains, in plain language, how search on the IFB website works today — what happens when a customer types something into the search box, what the "autocomplete" and "did you mean" features do, and what was recently fixed to make search results more accurate.

---

## 1. What Search Does

When a customer types a search term (e.g. *"washing machine"*, *"1.5 ton AC"*, *"refrigirator"*), the request goes to our search engine (Solr), which:

1. Looks across product titles, categories, capacities (like "7 kg" or "1.5 Ton"), features, and descriptions to find matching products.
2. Ranks the matches so the **most relevant, real appliances show up first** — not accessories, spare parts, or unrelated support/blog articles.
3. Returns the matching product IDs, which are then sent to our pricing/stock system (GraphQL) to fetch the **real-time price and availability for the customer's pincode**. Search itself does not decide price or stock — that's intentional, since prices and stock vary by location and change constantly.

---

## 2. The Three Search-Related Features

### a) Main Search (what happens when a customer hits "Search")
The customer's words are matched against product data with different levels of importance:
- **Capacity match** (e.g. "7 kg", "1.5 Ton") — weighted the highest, since it's the most decisive factor for appliance shopping.
- **Category match** (e.g. "Washing Machine", "Air Conditioners") — weighted very high.
- **Product title** — high weight.
- **Feature text, descriptions** — lower weight (used to fill in results, not to drive the top ranking).

Real appliances are also boosted above **accessories and consumables** (like descaling powder, stands, Wi-Fi modules) and above **Support articles, Blog posts, and generic pages**, so a search for "washing machine" shows actual washing machines first, not a "how-to" article or a machine-cleaning product.

### b) Autocomplete ("Suggestions" while typing)
As a customer types, a separate, faster lookup suggests complete product names, titles, and categories — similar to how Google suggests full searches before you finish typing. This is powered by four "suggestion dictionaries": product names, product titles, categories, and (newly added) subcategories (e.g. "Top Load Washing Machine", "Single Door Refrigerator").

### c) Spell Correction ("Did you mean...?")
If a customer mistypes a word (e.g. *"refridgirator"*, *"stablizer"*, *"washng machne"*), the system suggests the correctly spelled word(s), pulled only from real Product and Support content — not Blog posts, so suggestions stay relevant to shopping.

---

## 3. Real Examples — Before and After

| Customer types | What used to happen | What happens now |
|---|---|---|
| `Air Conditioner` | Only **1 result** returned (a config bug) | **181 relevant AC results**, correctly ranked |
| `washing machine` | An unrelated cleaning accessory and two generic help articles ranked **above** actual washing machines | Real washing machines rank on top |
| `1.5 ton ac` | An AC outdoor-unit stand (an accessory, out of stock) ranked **above** actual 1.5-ton ACs | Actual 1.5-ton air conditioners rank on top |
| `refrigerator` | Returned **washing machines** in the top results (a hidden text-matching quirk) | Returns real refrigerators |
| `stabilizer` | Even spelled correctly, showed **refrigerators** above real stabilizers | Real stabilizer products rank on top |
| `washingmachine` (no space) | 0 results | Now finds real washing machines |
| Autocomplete for "front" or "top" | Returned **nothing**, even though many "Front Load"/"Top Load" products exist | Now suggests correctly |

These weren't cosmetic tweaks — each one directly affects whether a customer finds the product they're looking for on the first try, which affects conversion.

---

## 4. What's Working Well Today

- ✅ Exact category searches (washing machine, refrigerator, air conditioner, dishwasher, chimney, stabilizer, microwave)
- ✅ Common misspellings and typo variants
- ✅ Capacity-based searches ("7 kg", "1.5 ton", "331 L") with or without spaces/units
- ✅ Exact model number / SKU search
- ✅ Accessories and consumables are still findable when searched by name, without polluting main category results
- ✅ Autocomplete suggestions across product names, titles, categories, and subcategories
- ✅ Spell-check suggestions scoped to real product/support content

## 5. Known Limitations (Not Bugs — Needs a Data or Process Decision)

- **2-letter product abbreviations** (e.g. searching literally "TV" for "TV Stabilizer") won't find results until the catalog is **fully reindexed** — a configuration fix is already in place, it just needs a full data refresh cycle to apply to existing products.
- **Subcategory data** is only present on ~70% of products. Products missing this data get slightly less precise ranking for subcategory-style searches (e.g. "Top Load" vs "Front Load").
- **Price and stock are intentionally not part of Solr search** — they're resolved per-pincode in real time via the GraphQL layer after search returns matching products, so this is expected behavior, not a gap.
- **Search does not automatically return "filter" counts** (e.g. how many results are Products vs Accessories) unless the storefront explicitly asks for them per request — this can be added if the UI needs filter/facet counts by default.

---

## 6. Glossary (Plain Language)

- **Relevance ranking** — the process of deciding which matching products to show first.
- **Autocomplete/Suggester** — the "suggested searches" dropdown shown while typing.
- **Spellcheck** — the "did you mean...?" correction feature.
- **Facet/Filter counts** — the number of results per category/type (e.g. "Products: 223, Accessories: 18"), used to build filter sidebars.
- **Reindex** — rebuilding the entire search catalog from source data; required when certain structural search-configuration changes are made, as opposed to a quick "reload" which only applies to how new searches are interpreted.
