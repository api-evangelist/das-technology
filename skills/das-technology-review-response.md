---
name: das-technology-review-response
description: Pull a dealership's online reviews from the DAS Web API, read a single review and its existing responses, and post a dealer response back — the core reputation-management loop behind DAS Technology's Retain product line.
api: DAS Web API (DASWebAPI v1 + v2)
base_url: https://api.digitalairstrike.com
generated: '2026-08-12'
method: generated
source: openapi/_original/das-technology-daswebapi-v1-swagger.json, openapi/_original/das-technology-daswebapi-v2-swagger.json
operations:
  - AccountV2_GetReviewsV2
  - Account_GetReviewsV2
  - Review_GetReview
  - Review_GetReviewResponses
  - Review_PostReviewResponse
  - Review_DepartmentDisposition
  - Account_GetReviewStatsAsync
---

# Respond to dealership reviews

Every step below is grounded in an operation that exists in the Swagger documents DAS
Technology publishes at `https://api.digitalairstrike.com/swagger`. Nothing here is invented.

## Before you start

- **Auth.** Every call needs `Authorization: Bearer <token>`. The specification declares no
  security scheme, but the live API returns `401` with `WWW-Authenticate: Bearer` on any
  anonymous request. There is no public token endpoint — credentials come from the DAS
  dealer/OEM/partner relationship. See `authentication/das-technology-authentication.yml`.
- **Scope.** Almost everything is scoped by `accountGuid` (a UUID identifying a rooftop).
- **Accept.** Send `Accept: application/json`. The API also serves XML and will return XML by
  default on the error path even when you asked for JSON.

## Steps

### 1. List reviews for the account

Prefer v2 — it returns a paged envelope with a total.

    GET /v2/account/{accountGuid}/reviewV2
    → AccountV2_GetReviewsV2

Filters are a bound query object flattened onto the query string: `query.fromDate`,
`query.toDate`, `query.source`, `query.reviewType`, `query.siteName`, `query.state`,
`query.hasResponse`, `query.disputedReview`, `query.closedLoop`, `query.categories`,
`query.brands`, `query.orderBy`, `query.page`, `query.pageSize`.

The response is `DAS.Models.PagedResult[DAS.Models.ReviewV2]` — `data` (the page) and `count`
(the total). There is no next-page link; compute pages yourself from `count` and `pageSize`.

To find reviews still needing a reply, filter on `query.hasResponse=false`.

The v1 equivalent, `Account_GetReviewsV2` at `GET /v1/account/{accountGuid}/reviewV2`, returns
the same records without the paged envelope.

### 2. Read one review and what has already been said

    GET /v1/review/{reviewGuid}              → Review_GetReview
    GET /v1/review/{reviewGuid}/response     → Review_GetReviewResponses

Always run the second call before writing. There is no idempotency key on the write below, so
reading existing responses is the only way to avoid posting a duplicate reply.

### 3. Post the dealer response

    POST /v1/review/{reviewGuid}/response    → Review_PostReviewResponse

**Not idempotent.** The API accepts no `Idempotency-Key` header and returns no request
identifier. If the call times out, do NOT blind-retry — re-run `Review_GetReviewResponses`
and check whether your response landed before trying again.

### 4. Optionally route the review to a department

    POST /v1/review/{reviewGuid}/DepartmentDisposition  → Review_DepartmentDisposition

### 5. Confirm the effect on the account's reputation numbers

    GET /v1/account/{accountGuid}/reviewStats            → Account_GetReviewStatsAsync
    GET /v1/account/{accountGuid}/reviewStatsByInterval  → Account_GetReviewStatsByIntervalAsync
    GET /v2/stats/review                                 → StatsV2_GetReviewStatsAsync

## Errors

The specification documents only `200` on all 136 operations. What the API really returns:

| Status | Media type | Meaning |
|---|---|---|
| 401 | application/xml | Missing/rejected bearer token (`WWW-Authenticate: Bearer`) |
| 404 | text/html | Unrouted path — an ASP.NET page, not the XML error envelope |
| 500 | application/xml | Server fault; body currently includes a stack trace |

Parse defensively: the error envelope is `<Error><Message>…</Message></Error>`, and the 404 is
not in that shape at all. Full catalog: `errors/das-technology-problem-types.yml`.

## Rate limits

None published and none signaled — no `RateLimit-*` or `Retry-After` header appears on any
response. Pace your own calls. See `rate-limits/das-technology-rate-limits.yml`.
