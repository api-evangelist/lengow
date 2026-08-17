---
name: Authenticate against the Lengow API and check your rate budget
description: >-
  Exchange a Lengow account's access_token and secret for a one-hour session token, verify the session,
  and read the per-route rate-limit budget before doing any real work.
api: openapi/lengow-channel-execution-openapi.yml
base_url: https://api.lengow.io
operations:
  - Get Authentication Token
  - Get Current Session
  - Get Rate Limits
generated: '2026-08-17'
method: generated
source: openapi/lengow-channel-execution-openapi.yml + https://docs.lengow.io/#authentication
---

# Authenticate against the Lengow API

Every Lengow call except the token exchange needs a session token. The token lives **one hour**, so a
long-running agent must plan to refresh it, not fetch it once.

## 1. Get the credentials

The two long-lived keys — `access_token` and `secret` — are issued in the **API section of the Lengow
account** at https://my.lengow.io/. They are **per (sub)account**: if you act for several subaccounts you
hold several key pairs. Generating a new pair in the UI **immediately revokes the old one**, so never
regenerate keys as a "fix" for a failing call without checking what else uses them.

## 2. Exchange them for a session token — `Get Authentication Token`

```
POST https://api.lengow.io/access/get_token
--data "access_token=${ACCESS_TOKEN}&secret=${SECRET}"
```

Returns `{"token": "...", "account_id": 1}`.

- `400` means the pair is wrong. Do not retry with the same values.
- Keep `account_id` — most other endpoints require an `account_id` query parameter, and it must match
  the account the token belongs to.

## 3. Send the token on every request

```
curl https://api.lengow.io/v3.0/orders/?account_id=1 -H "Authorization: ${TOKEN}"
```

The docs' own examples put the **raw token value** in the `Authorization` header with no `Bearer `
prefix, even though the OpenAPI declares an `http`/`bearer` scheme. Follow the docs.

Missing or stale credentials come back as:

- `401 {"error": {"message": "Missing authentication token", "code": 401}}`
- `403 {"error": {"message": "No token in headers to access /orders API", "code": 403}}`

Treat both as "re-authenticate", not as "the resource does not exist".

## 4. Check whether the session is still good — `Get Current Session`

```
GET https://api.lengow.io/me
```

Returns the current session description including the token's expiration date. Call this instead of
blindly re-exchanging keys — the docs are explicit that requesting a token before every request is
unnecessary.

## 5. Read your rate budget before a batch — `Get Rate Limits`

```
GET https://api.lengow.io/rate-limits
```

Returns every route available to this login with its configured fixed-window quota, plus:

- `remaining_credit` — current bucket value, clamped to zero once exhausted
- `remaining_time` — seconds until the window resets
- both `null` when rate limiting is disabled for that route

This is the only way to see your remaining budget: **successful responses carry no `X-RateLimit-*` or
`RateLimit-*` headers.** The only rate-limit header Lengow returns is `Retry-After`, and only on a `429`.

## Rules

- On `429`, honour `Retry-After` (seconds until the fixed window expires). Do not exponential-backoff
  past it blindly — call `Get Rate Limits` and schedule against `remaining_time`.
- One published hard number exists: `GET /v1.0/report/export` is capped at **10 requests per minute**.
  Everything else is per-route configuration you must read at runtime.
- Cache the session token for its full hour and share it across the batch. Re-exchanging keys per
  request wastes budget on a route you also need for real work.
