---
name: Pull a Lengow feed integration report and triage product errors
description: >-
  Download the last product integration report for a feed as CSV, respect the 10 req/min cap on that
  endpoint, and turn the rows into a prioritised list of products blocked from a marketplace.
api: openapi/lengow-channel-execution-openapi.yml
base_url: https://api.lengow.io
operations:
  - Get product reports
  - Get Rate Limits
generated: '2026-08-17'
method: generated
source: openapi/lengow-channel-execution-openapi.yml + https://docs.lengow.io/
---

# Pull a Lengow feed integration report

This is the answer to "why isn't my product live on the marketplace?" — the report carries both Lengow's
pre-transmission checks and the marketplace's own validation response.

Authenticate first (see `lengow-authenticate-and-check-limits.md`).

## 1. Download the report — `Get product reports`

```
GET https://api.lengow.io/v1.0/report/export?feed_id={feed_id}&nb_days_to_skip=0
```

| Parameter | Required | Meaning |
|---|---|---|
| `feed_id` | yes | integer id of the feed whose report you want |
| `nb_days_to_skip` | no | fetch an earlier day's report — `1` = yesterday |

The response is **`text/csv`, pipe-delimited (`|`)** — not JSON. Parse it as CSV with `|` as the field
separator; a comma parser will produce garbage rows silently.

## 2. Respect the cap

This endpoint is capped at **10 requests per minute** — the only hard numeric limit Lengow publishes.
If you are pulling reports for many feeds, pace at 6 seconds between calls, or read the live budget with
`Get Rate Limits` and schedule against `remaining_credit` / `remaining_time`.

## 3. Handle the failure modes

- `429` — bucket exhausted. Honour `Retry-After` (seconds until the fixed window expires).
- `503` — the report could not be produced. This is expected occasionally: Lengow's docs note that for
  some marketplaces reports are unavailable, either because the marketplace does not provide the
  feature or because the Lengow integration is still in progress. Do not treat a persistent 503 on one
  marketplace as a bug in your integration — check the Help Center for that channel.

## 4. Triage the rows

Report rows correspond to the error model the API publishes: each entry carries a product identifier, a
`severity`, a `message`, the offending `attribute` / `header_slug`, and an `impacted_line_id`. The
report roll-ups Lengow computes over the same data are: `products_in_feed`, `products_published`,
`products_in_error`, `products_with_warning`, `products_healthy`.

Sort work in this order:

1. **Errors** blocking publication — the product is not on the marketplace at all.
2. **Warnings** — the product is live but degraded (missing an attribute that affects ranking or
   eligibility).
3. Group by `attribute` / `header_slug` rather than by product. A single missing attribute usually
   explains hundreds of rows, and one catalogue-level rule fixes all of them at once.

## 5. Close the loop

Fix the offending attributes in the catalogue with
`update_products_public_catalogues__catalogue_id__products_patch` (see
`lengow-sync-catalogue-products.md`), then re-pull the report the **next** day with
`nb_days_to_skip=0` — the export returns the last report issued after transmission, so it will not
reflect a change you made seconds ago.
