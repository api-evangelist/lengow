---
name: Sync products into a Lengow catalogue
description: >-
  Find the right Lengow catalogue, learn its product key and attribute vocabulary, then create, update
  or delete products in batches — reading the per-item 207 results rather than trusting the HTTP status.
api: openapi/lengow-channel-execution-openapi.yml
base_url: https://api.lengow.io
operations:
  - list_catalogues_public_catalogues_get
  - get_catalogue_public_catalogues__catalogue_id__get
  - list_attributes_public_catalogues__catalogue_id__attributes_get
  - list_products_public_catalogues__catalogue_id__products_get
  - get_product_public_catalogues__catalogue_id__products__product_id_value__get
  - upsert_products_public_catalogues__catalogue_id__products_post
  - update_products_public_catalogues__catalogue_id__products_patch
  - update_product_public_catalogues__catalogue_id__products__product_id_value__patch
  - delete_products_public_catalogues__catalogue_id__products_delete
  - delete_product_public_catalogues__catalogue_id__products__product_id_value__delete
generated: '2026-08-17'
method: generated
source: openapi/lengow-channel-execution-openapi.yml
---

# Sync products into a Lengow catalogue

This is the **v1.0 Catalogues API**, and every operation in it is marked `[BETA]` in Lengow's own
published spec. Treat shapes as unstable and pin nothing you cannot re-derive.

Authenticate first — see `lengow-authenticate-and-check-limits.md`. Every call below also needs
`account_id` as a **query parameter**, not just a token.

## 1. Find the catalogue — `list_catalogues_public_catalogues_get`

```
GET /v1.0/catalogues?account_id={account_id}&page=1&limit=50
```

Paginated with `page` / `limit`; the response carries `data`, `page`, `limit`, `total` and `next`.
Each item gives you `catalogue_id`, `name`, `status`, `product_id_key`, `channels_count`, `role`,
`used_by` and `additional_sources`.

## 2. Learn the catalogue's contract before writing anything

- `get_catalogue_public_catalogues__catalogue_id__get` —
  `GET /v1.0/catalogues/{catalogue_id}?account_id=...` returns `product_id_key`, `products_count`,
  `source`, `status`, `created_at`, `updated_at`, `indexed_at`.
- `list_attributes_public_catalogues__catalogue_id__attributes_get` —
  `GET /v1.0/catalogues/{catalogue_id}/attributes?account_id=...` returns the attribute keys that
  catalogue actually uses.

**Two things decide whether your write will work:**

1. `product_id_key` is **per catalogue**. Every product is addressed by the *value* of that key
   (`product_id_value`), so you must read the key before you can build a path or a batch payload.
2. `status` must not be `draft`. Writes against a draft catalogue are rejected with
   `409 Catalogue is in draft status; operation forbidden`.

## 3. Read what is there — `list_products_public_catalogues__catalogue_id__products_get`

```
GET /v1.0/catalogues/{catalogue_id}/products?account_id=...&page=1&limit=100
```

Each item is `{product_id_value, attributes}`. For one product use
`get_product_public_catalogues__catalogue_id__products__product_id_value__get`.

## 4. Write

| Intent | Operation | Method + path |
|---|---|---|
| Create or replace a batch | `upsert_products_public_catalogues__catalogue_id__products_post` | `POST /v1.0/catalogues/{catalogue_id}/products` |
| Patch a batch | `update_products_public_catalogues__catalogue_id__products_patch` | `PATCH /v1.0/catalogues/{catalogue_id}/products` |
| Patch one product | `update_product_public_catalogues__catalogue_id__products__product_id_value__patch` | `PATCH /v1.0/catalogues/{catalogue_id}/products/{product_id_value}` |
| Delete a batch | `delete_products_public_catalogues__catalogue_id__products_delete` | `DELETE /v1.0/catalogues/{catalogue_id}/products?product_ids=...` |
| Delete one product | `delete_product_public_catalogues__catalogue_id__products__product_id_value__delete` | `DELETE /v1.0/catalogues/{catalogue_id}/products/{product_id_value}` |

`POST` is **create-or-replace**, `PATCH` is a partial update. Choosing `POST` when you meant `PATCH`
silently drops the attributes you left out.

## 5. Read the 207 — this is the step agents get wrong

The three batch operations return **HTTP 207 Multi-Status**, not 200. A 207 is *not* "it worked". The
body is an array of per-product results (`product_id_value`, `status`, `errors`). You must:

1. Walk every item.
2. Collect the failures by `product_id_value`.
3. Retry only those, never the whole batch.

## Error rules

- `404` — catalogue not found for this account, or catalogue/product pair not found. Check `account_id`
  before assuming the product is missing.
- `409` — catalogue is in draft. Publish it; retrying will not help.
- `422` — validation error, body is `{"detail": [{loc, msg, type}]}`. `loc` points at the offending
  field; fix the payload, do not retry unchanged.
- `400` on a single-product PATCH — you tried to update an attribute the catalogue does not accept. Go
  back to step 2 and re-read the attribute list.
- `429` — bucket exhausted. Honour `Retry-After`.

## Idempotency

**There is none.** Lengow publishes no `Idempotency-Key` header or parameter on any operation. A retried
`POST` upsert is a second real write. Before retrying a batch whose response you never saw, re-read the
affected products with `list_products_public_catalogues__catalogue_id__products_get` and reconcile —
do not blind-retry.
