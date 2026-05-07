---
description: "Use when the user asks about search queries, bool filters, aggregations, async search, runtime fields, or cross-cluster search."
allowed-tools: eep-setup-search-lab-tool, eep-grade-search-query-tool
---

# Searching Data

Exam objectives this skill covers:

- Write and execute a search query for terms and/or phrases in one or more fields
- Boolean combinations of queries and filters
- Asynchronous search
- Metric and bucket aggregations, with sub-aggregations
- Cross-cluster search
- Search using runtime fields

---

## Beat 1 — Explain

Give a 2-sentence summary of each topic, then ask if they want to set up a lab:

**Query DSL and bool logic**
> All searches are JSON sent to `_search`. The `bool` query composes four clauses: `must` (scored full-text), `filter` (exact, cached, no scoring), `should` (boost), `must_not` (exclude). Prefer `filter` over `must` for exact values — it's faster and uses the bitset cache.

**Search templates**
> Store parameterized queries with `PUT _scripts/<template-id>` using Mustache syntax. Execute with `POST <index>/_search/template { "id": "<template-id>", "params": {...} }`. Templates decouple query logic from application code — change the template without redeploying.

**Aggregations**
> Aggs run in parallel with the query on the same shard data. `terms` buckets by value, `date_histogram` buckets by time, `avg`/`sum`/`max` are metrics. Nest a metric agg inside a bucket agg to get "average score per category".

**Async search**
> Submit long queries with `POST index/_async_search`. You get an `id` immediately. Poll with `GET /_async_search/<id>` — the response has `is_partial` and `is_running` flags. Delete with `DELETE /_async_search/<id>` when done.

**Runtime fields**
> Add `"runtime_mappings"` to any search request to define a Painless-computed field that never touches the index. Emit with `emit(value)`. Reference it in `fields`, `sort`, or `aggs` just like a real mapped field.

**Cross-cluster search**
> Prefix the index with `cluster_alias:` — e.g., `GET cluster_two:logs-*,local-logs-*/_search`. The remote cluster must be registered in `cluster.remote` settings first.

Close with: _"Want me to set up a search lab so you can practice building queries and search templates in Dev Tools?"_

---

## Beat 2 — Setup and canonical template exercise

Call **`eep-setup-search-lab-tool`** with no parameters.

It creates `eep-search-lab-demo` with:
- A custom `title_english` analyzer (standard tokenizer + lowercase + porter_stem)
- Strict mappings: `title` (text, analyzed), `category` (keyword), `difficulty` (keyword), `points` (integer), `published` (boolean)
- 5 seed docs across search/ingest/mappings categories, beginner/intermediate/advanced difficulty

After the tool returns, tell the learner:

> Lab is ready. Your first exercise: create a **search template** that finds published docs in a given category matching a search term, sorted by points.
>
> Here's the scaffold — fill in the three `___FILL IN___` placeholders, then run it in **Dev Tools Console**:
>
> ```
> PUT _scripts/eep-canonical-search
> {
>   "script": {
>     "lang": "mustache",
>     "source": {
>       "query": {
>         "bool": {
>           "must": [
>             ___FILL_IN: match query on "title" using the search_term param___
>           ],
>           "filter": [
>             ___FILL_IN: term filter where published = true___,
>             ___FILL_IN: term filter on "category" using the category param___
>           ]
>         }
>       },
>       "sort": [{ "points": "desc" }],
>       "_source": ["title", "points"]
>     }
>   }
> }
> ```
>
> After you've stored the template, test it:
>
> ```
> POST eep-search-lab-demo/_search/template
> {
>   "id": "eep-canonical-search",
>   "params": {
>     "search_term": "query",
>     "category": "search"
>   }
> }
> ```
>
> Tell me when you've run both and I'll grade your template.

When they confirm, call **`eep-grade-search-query-tool`** with `mode: canonical`.

The grade tool executes the learner's stored template directly from the cluster with the same params.

- If the template doesn't exist or errors: report the `error_snippet` and ask them to check their `PUT _scripts/eep-canonical-search` command ran without errors.
- If `passed: true`: confirm — "Your template returned 1 hit: 'Optimize bool query relevance tuning', 35 pts. The `match` on the English-analyzed field matched 'query' via porter stemming ('querying' → 'queri'). `filter` clauses restrict without affecting the relevance score. And because the query is stored as a template, you can reuse it with different params without rewriting it."
- If `passed: false, hits_count: 0`: the template executed but returned no hits. Check their `must` and `filter` clauses — likely `published` or `category` filter is wrong, or the `match` field name is incorrect.

---

## Beat 3 — Challenge template

Say:

> Now create a second template on your own. Store it as **`eep-beginner-filter`** — it should accept a `difficulty` parameter and return only published docs at that difficulty level, sorted by `points` descending.
>
> Here's just the outer structure — you decide what goes inside:
>
> ```
> PUT _scripts/eep-beginner-filter
> {
>   "script": {
>     "lang": "mustache",
>     "source": {
>       "query": { ... },
>       "sort": [ ... ],
>       "_source": ["title", "points"]
>     }
>   }
> }
> ```
>
> Store the template, then test it:
>
> ```
> POST eep-search-lab-demo/_search/template
> {
>   "id": "eep-beginner-filter",
>   "params": { "difficulty": "beginner" }
> }
> ```
>
> Tell me when you have results and I'll grade it.

When they confirm, call **`eep-grade-search-query-tool`** with `mode: challenge`.

The grade tool executes the learner's `eep-beginner-filter` template directly from the cluster.

- `passed: true, hits_count: 2, first_points: 20` → "Your template returned 2 hits: 'Configure text analysis for product search' (20 pts), then 'Compare term versus match behavior' (15 pts). Using `filter` instead of `must` is correct here — `difficulty` is a keyword field, so exact matching with `term` is the right choice. No scoring needed."
- `passed: false, hits_count != 2` → use the hint. Likely filtering on the wrong field, missing the `published: true` filter, or using `match` instead of `term` on a keyword field.
- `passed: false, first_points != 20` → sort is wrong — they have the right docs but ascending order.
- Template not found / error → "Template `eep-beginner-filter` doesn't exist yet or has a syntax error. Run `GET _scripts/eep-beginner-filter` to check, and re-run the `PUT _scripts` command if needed."

---

## Beat 4 — Reference exercises (inline, no grade tool)

Work through these one at a time based on what the learner asks about next. Ask them to run each in Console and paste the response; verify the key fields inline.

### Aggregations

```
GET eep-search-lab-demo/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category" },
      "aggs": {
        "avg_points": { "avg": { "field": "points" } }
      }
    }
  }
}
```

Verify: `aggregations.by_category.buckets` has 3 entries (search, ingest, mappings). Each bucket has `avg_points.value`. Explain: `size: 0` skips hits since we only want agg results.

### Async search

```
POST eep-search-lab-demo/_async_search
{
  "query": { "match_all": {} }
}
```

Then poll: `GET /_async_search/<id from response>`

Verify the initial response has an `id` field. The poll response has `is_running: false` and `response.hits.total.value: 5` when complete.

### Runtime field

```
GET eep-search-lab-demo/_search
{
  "runtime_mappings": {
    "points_tier": {
      "type":   "keyword",
      "script": "emit(doc['points'].value >= 30 ? 'high' : 'low')"
    }
  },
  "query":   { "term": { "points_tier": "high" } },
  "fields":  [ "title", "points_tier" ],
  "_source": false
}
```

Verify: hits have `fields.points_tier: ["high"]` and `points >= 30`. Explain: the runtime field exists only for this request — nothing is written to the index.

### Cross-cluster search (reference only — requires a registered remote)

```
GET cluster_two:eep-search-lab-demo,eep-search-lab-demo/_search
{
  "query": { "match_all": {} }
}
```

No live lab for CCS. Walk through what each part means: `cluster_two:` is the remote alias registered via `PUT /_cluster/settings`. The local index comes after the comma with no prefix.

---

## Hard rules

- All queries go in **Dev Tools Console** — never Discover or ES|QL.
- Call the grade tool after **every** exercise that has a graded mode; don't skip it.
- Do not paste raw workflow JSON to the user.
- Do not show the learner the correct template source — give scaffolds and hints, not answers.
- One query block per turn is enough — don't front-load all patterns.
