---
name: Analyse a competitor's public sector wins
description: >-
  Resolve a supplier by name, read its all-time award count, then reconstruct which contracts it has
  won, from which buyers, in which categories, and when those contracts come up for renewal.
api: openapi/stotles-public-api-openapi.yml
operations: [searchSuppliers, getSupplier, searchNotices]
generated: '2026-08-14'
method: generated
source: >-
  Grounded in operationIds verified verbatim in openapi/stotles-public-api-openapi.yml (harvested
  from https://api.stotles.com/v1/openapi.json, 2026-08-14). Traversal semantics from
  ../data-model/stotles-data-model.yml.
---

# Analyse a competitor's public sector wins

Competitive intelligence: given a rival's company name, find every public sector contract they hold
and work out where they are vulnerable.

## Steps

### 1. Resolve the supplier with `searchSuppliers`

`GET /v1/suppliers/search` (operationId `searchSuppliers`).

```bash
curl -G https://api.stotles.com/v1/suppliers/search \
  -H "x-api-key: $STOTLES_API_KEY" \
  --data-urlencode "query=Capita" \
  --data-urlencode "country_code=GB" \
  --data-urlencode "limit=50"
```

**`query` is REQUIRED** — a single string, 2–200 characters (as on `searchBuyers`, and unlike
`searchNotices` where it is an optional array).

Supplier is the thinnest entity on this API — four fields only: `id`, `name`,
`location {country_code}`, and `award_count` (all-time count of contracts awarded). There is no
company number, no website, no parent/subsidiary linkage.

**Large suppliers fragment across many records.** Group companies bid through dozens of legal
entities, and the API gives you no corporate hierarchy to collapse them. Page the full result set,
keep every id whose name plausibly belongs to the group, and treat the analysis as multi-id from
here on. `award_count` is the useful triage signal for which entities actually matter.

### 2. Confirm with `getSupplier`

`GET /v1/suppliers/{id}` (operationId `getSupplier`) returns the same four fields for one UUID. Use
it to refresh a tracked competitor rather than re-running the search.

### 3. Pull their awards from the notices hub

There is no `/v1/suppliers/{id}/contracts`. Filter `searchNotices` on `supplier_id` — it accepts up
to 100 UUIDs, so pass every entity id you collected in step 1 at once.

```bash
curl -G https://api.stotles.com/v1/notices/search \
  -H "x-api-key: $STOTLES_API_KEY" \
  --data-urlencode "supplier_id=<uuid-1>" \
  --data-urlencode "supplier_id=<uuid-2>" \
  --data-urlencode "stage=awarded_contract" \
  --data-urlencode "award_date_gte=2022-01-01" \
  --data-urlencode "sort=award_date" \
  --data-urlencode "order=desc" \
  --data-urlencode "limit=50"
```

Page until `next_cursor === null` — a short page is not the last page.

### 4. Read the awards out of the notice

The supplier linkage lives **inside** the notice, not at the top level. For each notice in the
result, walk `contracts[]`:

```
notice.contracts[] -> { title, description, value, award_date,
                        contract_period: { start_date, end_date, max_extent_date },
                        cpv_codes[], suppliers[] { id, name }, lots[] }
```

Filter `contracts[].suppliers[]` down to the ids you are tracking — a notice can award several
contracts to several different suppliers, and only some of them are your competitor. Attributing the
whole notice `value` to them is the classic error here; attribute `contracts[].value` instead.

Buyers come from the notice's own `buyers[]` array.

### 5. Find the renewal windows

This is the commercially useful output. For each contract your competitor holds:

- `contract_period.end_date` — the **earliest** expected end.
- `contract_period.max_extent_date` — the **latest** date the period can be extended to.

The gap between them is the extension risk. Procurement for a replacement typically starts months
before `end_date`, so sort your target list by `end_date` ascending and work backwards.

Cross-check against forward-looking signals: re-run `searchNotices` with the same `buyer_id` values
and `stage=pipeline` / `stage=pre_tender` to see whether the re-procurement has already surfaced.

### 6. Read the category footprint

Aggregate `cpv_codes[]` across their awards to see which categories they actually win in, and
`buyers[]` to see which parts of government they are strong in. Combine with `type_group` from
`getBuyer` (see `stotles-profile-a-buyer.md`) to say "they own healthcare, they are weak in local".

## Counting money honestly

- `value.amount` is a decimal with a separate `currency`; parse with a big-decimal type, not a float.
- `value` may be `null`; `currency` may be `null` on its own.
- Add `framework_activity=exclude_framework_agreements` before totalling, or a framework's headline
  value gets added on top of the call-offs made under it.
- Values are **estimates** drawn from published notices, not audited spend. Report them as such.

## Pacing

Multi-entity competitor analysis is request-hungry. Batch up to 100 `supplier_id` values per call
rather than looping. Stay at or under 3 requests/second; budget against 1,000 requests/hour. Page at
`limit=50`.

## See also

- `../data-model/stotles-data-model.yml` — Contract and ContractPeriod shapes
- `stotles-profile-a-buyer.md` — the buyer-side traversal
- `stotles-map-a-framework.md` — when the competitor's route to market is a framework
