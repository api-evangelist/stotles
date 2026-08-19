---
name: Map a framework agreement and the call-offs made under it
description: >-
  Find a framework agreement or dynamic purchasing system, read its lifecycle stage and term, and
  separate the notice that established it from the individual call-off awards made under it.
api: openapi/stotles-public-api-openapi.yml
operations: [searchFrameworks, getFramework, searchNotices]
generated: '2026-08-14'
method: generated
source: >-
  Grounded in operationIds verified verbatim in openapi/stotles-public-api-openapi.yml (harvested
  from https://api.stotles.com/v1/openapi.json, 2026-08-14). Framework vocabularies and the
  double-counting hazard from ../data-model/stotles-data-model.yml.
---

# Map a framework agreement and the call-offs made under it

Frameworks are how most UK public sector spend actually reaches suppliers: a buyer pre-competes a
vehicle, then awards individual call-offs under it. Getting on the right framework at the right
moment is a route-to-market decision, and this API models it explicitly.

## Steps

### 1. Find the framework with `searchFrameworks`

`GET /v1/frameworks/search` (operationId `searchFrameworks`).

```bash
curl -G https://api.stotles.com/v1/frameworks/search \
  -H "x-api-key: $STOTLES_API_KEY" \
  --data-urlencode "query=G-Cloud" \
  --data-urlencode "stage=live" \
  --data-urlencode "stage=upcoming" \
  --data-urlencode "sort=end_date" \
  --data-urlencode "order=asc" \
  --data-urlencode "limit=50"
```

`query` is optional here (2–500 chars) — unlike buyer and supplier search — so you can browse by
stage and date alone.

**Framework `stage` is a DIFFERENT vocabulary from notice `stage`.** Do not share an enum type
between them. The eight framework values are:

`stale`, `upcoming`, `tendering`, `closed`, `awarded`, `live`, `expired`, `canceled`

The two that matter for a go-to-market decision:

- `tendering` — the framework itself is accepting applications. **This is your window to get on it.**
- `live` — awarded and in use; call-offs are flowing, but you cannot join unless it is a dynamic
  purchasing system that reopens.

Date filters use the framework's own term: `start_date_gte`, `start_date_lte`, `end_date_gte`,
`end_date_lte`. To find vehicles you should be preparing for, filter `end_date_lte` to the next
12–18 months and watch for the successor to appear as `upcoming` or `tendering`.

### 2. Read the framework with `getFramework`

`GET /v1/frameworks/{id}` (operationId `getFramework`) returns `id`, `title`, `description`,
`service_provider {id, name}`, `stage`, `procedure_type`, `value`, `start_date`, `end_date`.

`procedure_type` distinguishes the two vehicle types and drives your strategy:

- `framework` — a framework agreement. Closed once awarded; you must be on it at award time.
- `dynamic_purchasing_system` — a DPS. Typically stays open for new suppliers throughout its life,
  so a late entry is possible.
- `null` — the source did not say. Do not assume; check `source_url` on a related notice.

`service_provider` is the organization operating the framework (often a purchasing organization
such as a central buying body) — it is an organisation reference `{id, name}`, and that id is a
buyer id, so you can feed it to `getBuyer`.

`description` is an **empty string** when unknown, not null.

### 3. Separate the framework notice from its call-offs

This is the whole point of the exercise, and it is where analyses usually go wrong.

Filter the notices hub on `framework_id`, then use `framework_activity` to choose which side you
want. The four values:

| `framework_activity` | returns |
|---|---|
| `only_framework_agreements` | Only the notices that **establish** a framework. |
| `only_call_offs` | Only the individual awards **made under** an existing framework. |
| `exclude_framework_agreements` | Everything except framework-establishing notices — keeps call-offs and non-framework work. |
| `exclude_all` | Only procurements unrelated to any framework. |

```bash
# Who is actually winning work through this framework?
curl -G https://api.stotles.com/v1/notices/search \
  -H "x-api-key: $STOTLES_API_KEY" \
  --data-urlencode "framework_id=<framework-uuid>" \
  --data-urlencode "framework_activity=only_call_offs" \
  --data-urlencode "stage=awarded_contract" \
  --data-urlencode "sort=award_date" \
  --data-urlencode "order=desc" \
  --data-urlencode "limit=50"
```

Each returned notice also carries `framework: { id, relationship }` where `relationship` is
`framework_agreement` (this notice establishes the framework) or `call_off` (this notice is an
award under it). Use it to verify what you are holding.

### 4. Do not double-count

**A framework's headline `value` and the values of its call-offs describe the same money.** Summing
both inflates market size, sometimes by an order of magnitude. Pick one lens:

- *How big is the vehicle?* — read `Framework.value` from `getFramework`.
- *How much has actually been called off, and by whom?* — sum `contracts[].value` across
  `framework_activity=only_call_offs` results.

Never add the two together. When totalling a whole market rather than one framework, pass
`framework_activity=exclude_framework_agreements` on your `searchNotices` calls.

### 5. Build the supplier and buyer picture

From the call-off notices: `contracts[].suppliers[]` gives you who is winning under the vehicle;
`buyers[]` gives you which authorities are actually using it. A framework with a long supplier list
but few active buyers is a poor route to market regardless of its headline value.

## Pacing

Stay at or under 3 requests/second, budget against 1,000 requests/hour, page at `limit=50`, and
terminate only on `next_cursor === null`.

## See also

- `../data-model/stotles-data-model.yml` — framework stage, procedure_type and framework_activity vocabularies
- `stotles-analyse-a-competitor.md` — who is winning the call-offs
- `stotles-track-open-tenders.md` — the general notice feed and its filters
