---
name: das-technology-account-hierarchy-reporting
description: Walk a dealer group's account hierarchy in the DAS Web API — parent, children and descendants of a rooftop — then roll enterprise reputation, survey and employee statistics up across the group.
api: DAS Web API (DASWebAPI v1 + v2)
base_url: https://api.digitalairstrike.com
generated: '2026-08-12'
method: generated
source: openapi/_original/das-technology-daswebapi-v1-swagger.json, openapi/_original/das-technology-daswebapi-v2-swagger.json
operations:
  - Account_GetAccount
  - Account_GetParent
  - Account_GetChild
  - Account_GetDescendant
  - Account_GetHeirarchy
  - AccountV2_GetHierarchyAsync
  - HierarchyV2_List
  - Stats_GetEnterpriseStatsAsync
  - StatsV2_GetEnterpriseStatsAsync
  - StatsV2_GetReviewSummaryByIntervalAsync
  - Stats_GetEmployeeStatsAsync
---

# Roll a dealer group up from its account hierarchy

Grounded in operations published in the DAS Web API Swagger documents at
`https://api.digitalairstrike.com/swagger`.

## Before you start

- `Authorization: Bearer <token>` on every call.
- The whole model hangs off `accountGuid`. Confirm which rooftop you hold before walking up.
- This flow is entirely read-only. Nothing here writes.

## Steps

### 1. Resolve the account

    GET /v1/account/{accountGuid}   → Account_GetAccount

Returns `DAS.Models.PortalAccountDetailed`. The v2 shape is `DAS.Models.PortalAccountV2`.

### 2. Walk the hierarchy

    GET /v1/account/{accountGuid}/parent      → Account_GetParent
    GET /v1/account/{accountGuid}/child       → Account_GetChild
    GET /v1/account/{accountGuid}/descendant  → Account_GetDescendant
    GET /v1/account/{accountGuid}/heirarchy   → Account_GetHeirarchy

Note the v1 hierarchy path is spelled `heirarchy` in the published contract (and the operationId
is `Account_GetHeirarchy`). That misspelling is part of the live URL — do not "correct" it.

v2 gives you a cleaner entry point:

    GET  /v2/account/{accountGuid}/hierarchy  → AccountV2_GetHierarchyAsync
    POST /v2/hierarchy/list                   → HierarchyV2_List

`HierarchyV2_List` is a POST that returns a list of available hierarchies. It is a query, not a
mutation — but because it is a POST it will still be blocked by any read-only policy you apply
by HTTP method, so allow it explicitly.

### 3. Roll statistics up across the group

The `/stats` family accepts a list of account identifiers, so you can score the whole group in
one call rather than fanning out per rooftop:

    GET /v1/stats/enterprise   → Stats_GetEnterpriseStatsAsync   (survey + review stats)
    GET /v1/stats/review       → Stats_GetReviewStatsAsync
    GET /v1/stats/survey       → Stats_GetSurveyStatsAsync
    GET /v1/stats/employee     → Stats_GetEmployeeStatsAsync

v2 equivalents, with interval summaries:

    GET /v2/stats/enterprise                 → StatsV2_GetEnterpriseStatsAsync
    GET /v2/stats/review/summary/byinterval  → StatsV2_GetReviewSummaryByIntervalAsync
    GET /v2/stats/survey/summary/byinterval  → StatsV2_GetSurveySummaryByIntervalAsync

Date range is `query.fromDate` / `query.toDate` (ISO 8601 date-time).

### 4. Per-account rollups when you need the rooftop view

    GET /v1/account/{accountGuid}/enterpriseStats    → Account_GetEnterpriseStatsAsync
    GET /v1/account/{accountGuid}/enterpriseStatsV2  → Account_GetEnterpriseStatsAsyncV2
    GET /v1/account/{accountGuid}/employeeStats      → Account_GetEmployeeStatsAsync

## Pagination

v1 list operations return bare JSON arrays with no envelope and no total. v2 collections return
`DAS.Models.PagedResult[T]` with `data` and `count`. There is no cursor and no `Link` header —
page arithmetic is yours. See `conventions/das-technology-conventions.yml`.

## Errors

Only `200` is documented. Expect `401` (XML), `404` (HTML) and `500` (XML with a stack trace).
See `errors/das-technology-problem-types.yml`.
