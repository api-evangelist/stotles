---
name: Profile a public sector buyer and its procurement history
description: >-
  Resolve a buying authority by name, read its organizational profile and type classification, then
  reconstruct everything it has tendered and awarded by filtering the notices hub on its id.
api: openapi/stotles-public-api-openapi.yml
operations: [searchBuyers, getBuyer, searchNotices]
generated: '2026-08-14'
method: generated
source: >-
  Grounded in operationIds verified verbatim in openapi/stotles-public-api-openapi.yml (harvested
  from https://api.stotles.com/v1/openapi.json, 2026-08-14). Traversal semantics from
  ../data-model/stotles-data-model.yml.
---

# Profile a public sector buyer and its procurement history

Account planning for B2G sales: given a buyer name like "Department for Work and Pensions" or
"NHS Greater Manchester", produce a profile plus their full procurement footprint.

## Steps

### 1. Resolve the buyer with `searchBuyers`

`GET /v1/buyers/search` (operationId `searchBuyers`).

```bash
curl -G https://api.stotles.com/v1/buyers/search \
  -H "x-api-key: $STOTLES_API_KEY" \
  --data-urlencode "query=Department for Work and Pensions" \
  --data-urlencode "country_code=GB" \
  --data-urlencode "limit=50"
```

**`query` is REQUIRED here** — a single string, 2–200 characters. This differs from
`searchNotices`, where `query` is an optional array. Same parameter name, different type. Getting
this wrong is the most common 400 on this API.

Narrow with the classification filters rather than with more keywords:

- `type_group` — six values: `central`, `local`, `healthcare`, `blue_light`, `education`, `other`.
- `type` — 23 finer values including `ministerial_department`, `local_authority`, `nhs_trusts`,
  `nhs_ccg_stp_ics`, `police`, `fire`, `ambulance`, `schools`,
  `higher_and_further_education`, `housing_associations`, `purchasing_organization`, `utilities`.

Both are repeated-value arrays. `?type_group=healthcare&type_group=blue_light` matches either.

Results are "most relevant first". Match on `name` plus `location.locality` — public sector names
collide heavily (dozens of "St Mary's" schools, many similarly-named NHS bodies).

### 2. Read the profile with `getBuyer`

`GET /v1/buyers/{id}` (operationId `getBuyer`), using the UUID from step 1.

The record carries `id`, `name`, `location {country_code, locality}`, `employee_count_band`,
`website_url`, and `types[]` where each type is `{id, name, group: {id, name}}`.

Null handling here is a live demonstration of the API's stated policy, and the two fields behave
differently on the same object:

- `location.country_code` is an **empty string** when unknown.
- `location.locality` is **null** when unknown.
- `employee_count_band` and `website_url` are **null** when unknown.

A buyer can carry **several** types, each rolling up to exactly one group. Do not assume one type.

### 3. Reconstruct the procurement history

There is no `/v1/buyers/{id}/notices` sub-resource. **Every relationship on this API is navigated by
filtering the notices hub.**

```bash
curl -G https://api.stotles.com/v1/notices/search \
  -H "x-api-key: $STOTLES_API_KEY" \
  --data-urlencode "buyer_id=3c6e0b8a-9c15-4f2d-8b7a-2e5d1c9f4a60" \
  --data-urlencode "publish_date_gte=2023-01-01" \
  --data-urlencode "sort=publish_date" \
  --data-urlencode "order=desc" \
  --data-urlencode "limit=50"
```

`buyer_id` is a repeated-value array accepting up to 100 UUIDs, so you can profile a whole group —
every NHS trust in a region, say — in one query. Page to `next_cursor === null`.

Slice the history by stage:

- **What they are buying now** — `stage=open_tender`
- **What is coming** — `stage=pipeline` and `stage=pre_tender`
- **Who they have bought from** — `stage=awarded_contract`, then read `contracts[].suppliers[]`
- **What is up for renewal** — `stage=expired_contract`, or filter live contracts on
  `expiry_date_lte` for the next 12 months

Remember: `expired_contract` can carry a future `expiry_date`. Filter on the date field, not on the
stage, when you care about timing.

### 4. Derive the numbers

From the notice set you can compute incumbent share (count and value of `contracts[].suppliers[]`
per supplier), category mix (`cpv_codes[]`), typical contract size (`value.amount`), and renewal
windows (`contracts[].contract_period.end_date` versus `max_extent_date` — the earliest expected end
versus the latest date it can be extended to).

**Before summing any value, exclude framework double-counting.** Add
`framework_activity=exclude_framework_agreements` so a framework's headline value is not added on
top of the call-offs made under it.

## Pacing

Profiling a group of buyers is the request-hungriest pattern on this API, because every traversal is
another `searchNotices` call. Stay under 3 requests/second and budget against 1,000/hour. Page at
`limit=50`. Batch up to 100 `buyer_id` values per request instead of looping one buyer at a time.

## Errors

400 on a malformed UUID (`errors[]: {"parameter":"id","detail":"Invalid uuid"}`) — fix, do not
retry. 404 from `getBuyer` means the id is well-formed but resolves to nothing. An empty
`searchNotices` result is 200 with `items: []`, not 404.

## See also

- `../data-model/stotles-data-model.yml` — buyer type and group vocabularies in full
- `../conventions/stotles-conventions.yml` — null-vs-empty-string policy, filter grammar
- `stotles-analyse-a-competitor.md` — the same traversal from the supplier side
