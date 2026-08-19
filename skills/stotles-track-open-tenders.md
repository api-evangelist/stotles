---
name: Track open tenders in your market
description: >-
  Build and page through a filtered feed of currently-open UK/Ireland public sector tenders using
  keywords, CPV codes, contract value and closing dates, without double-counting framework values
  or truncating results.
api: openapi/stotles-public-api-openapi.yml
operations: [searchNotices, getNotice]
generated: '2026-08-14'
method: generated
source: >-
  Grounded in operationIds verified verbatim in openapi/stotles-public-api-openapi.yml (harvested
  from https://api.stotles.com/v1/openapi.json, 2026-08-14). Cross-cutting rules from
  ../conventions/stotles-conventions.yml, ../errors/stotles-problem-types.yml and
  ../rate-limits/stotles-rate-limits.yml.
---

# Track open tenders in your market

The single most common Stotles API job: turn a market definition (keywords, CPV categories, value
band, geography) into a reliable list of tenders that are open for bids right now.

## Prerequisites

- An API key issued by your Stotles Customer Success Manager. There is no self-serve signup.
- Send it on every request as `x-api-key`. Never in a query string, never from a browser — CORS is
  pinned to `https://app.stotles.com`, so this must run server-side.

## Steps

### 1. Define the market with `searchNotices`

`GET /v1/notices/search` (operationId `searchNotices`). Everything is optional here, so start
narrow and widen.

```bash
curl -G https://api.stotles.com/v1/notices/search \
  -H "x-api-key: $STOTLES_API_KEY" \
  --data-urlencode "query=cyber security" \
  --data-urlencode "query=security operations centre" \
  --data-urlencode "query_operator=or" \
  --data-urlencode "cpv_code=72500000" \
  --data-urlencode "stage=open_tender" \
  --data-urlencode "country_code=GB" \
  --data-urlencode "value_gte=100000" \
  --data-urlencode "sort=close_date" \
  --data-urlencode "order=asc" \
  --data-urlencode "limit=50"
```

Rules that matter:

- **Repeat a parameter for multiple values — never comma-join.** `?stage=open_tender&stage=closed_tender`
  matches either. `?stage=open_tender,closed_tender` is a 400.
- `query` here is an **array** of terms combined by `query_operator` (`and` | `or`, default `or`).
  Each term is 2–500 chars, max 50 terms.
- Dates are calendar dates, `YYYY-MM-DD`. Range filters take an explicit suffix and are inclusive:
  `close_date_gte`, `close_date_lte`, `publish_date_gte`, `value_lte`, and so on.
- `limit` defaults to 20 but accepts up to **50**. Always ask for 50 — it cuts your request count
  by 60% against the hourly allowance.

### 2. Pick the right stage

`stage` is the field that decides whether something is actually biddable. The spec documents all ten
values; the ones that matter for a live feed:

| stage | meaning |
|---|---|
| `pipeline` | Spotted in a buyer's forward plan. No tender documents exist yet. |
| `pre_tender` | Formal early-market notice published. Not open yet. |
| `open_tender` | **Accepting bids — `close_date` has not passed.** |
| `closed_tender` | Deadline passed, no award published yet. |
| `awarded_contract` | Awarded; see `contracts[]` for suppliers and values. |

For a "what can I bid on today" feed use `stage=open_tender`. For an early-warning feed add
`stage=pipeline` and `stage=pre_tender` as separate repeated values.

Do **not** infer timing from stage. The provider warns explicitly that an `expired_contract` can
still carry a **future** `expiry_date`.

### 3. Avoid double-counting framework value

A framework-establishing notice and every call-off under it both appear in results, and both carry
a `value`. Summing `value` across unfiltered results overstates your market.

- Totalling addressable spend? Add `framework_activity=only_call_offs` or
  `framework_activity=exclude_framework_agreements`.
- Looking for the framework vehicles themselves? Use `framework_activity=only_framework_agreements`.

### 4. Page to the end — correctly

Each response is `{ "items": [...], "next_cursor": "..." }`.

```
cursor = null
loop:
  call searchNotices with cursor (omit on first call)
  append response.items
  cursor = response.next_cursor
  until cursor == null
```

Two traps, both stated by the provider:

- **A short page is not the last page.** `items.length < limit` does **not** mean you are done. Only
  `next_cursor === null` ends the loop. Stopping early silently truncates your pipeline.
- **Cursors are opaque.** Pass them back byte-for-byte. Do not decode, construct or persist them.

### 5. Read the money correctly

`value` is `{ "amount": 4500000, "currency": "GBP" }` — a decimal amount plus a currency, **not**
integer minor units. Contract values reach nine figures, so parse `amount` with a big-decimal type;
IEEE-754 floats lose precision. `value` is `null` when no amount is known, and `currency` can be
`null` on its own when the source did not state one. Handle both.

### 6. Pull the full record with `getNotice`

Search results already carry the full notice shape. Use `GET /v1/notices/{id}` (operationId
`getNotice`) when you are refreshing a single tracked opportunity rather than re-running the search.

```bash
curl https://api.stotles.com/v1/notices/8f14e45f-ea0d-4a3b-9c2e-1d7b6a5c4e30 \
  -H "x-api-key: $STOTLES_API_KEY"
```

`source_url` on the record links back to the originating portal (e.g. Find a Tender) — use it
whenever a human needs the primary document.

## Pacing

1,000 requests per hour **and** 3 requests per second, both per API key. The per-second ceiling
bites first. Cap in-flight requests at 3. Sequential paging at `limit=50` is comfortably inside both.

## Error handling

Errors are RFC 9457 `application/problem+json`. **Branch on `type`, never on `title` or `detail`.**

| status | `type` | what to do |
|---|---|---|
| 400 | `.../problems/validation` | Read `errors[].parameter`, fix the request. Never retry as-is. |
| 401 | `.../problems/unauthenticated` | Key missing or invalid. Escalate to a human. |
| 429 | `.../problems/rate-limited` | Sleep exactly `Retry-After` seconds, then retry. |
| 500 | `.../problems/internal` | Exponential backoff. `detail` is deliberately omitted; log the `request-id` response header. |

An empty result set is **200 with `items: []`**, not a 404. Never treat "no tenders" as an error.

## See also

- `../conventions/stotles-conventions.yml` — casing, dates, money, filters, pagination
- `../errors/stotles-problem-types.yml` — full problem catalog and retry matrix
- `../data-model/stotles-data-model.yml` — entity graph and enum vocabularies
