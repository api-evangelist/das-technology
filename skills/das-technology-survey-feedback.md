---
name: das-technology-survey-feedback
description: Read customer survey responses for a dealership from the DAS Web API, reply to a specific response, and request a new SMS/mobile survey — the closed-loop CX flow behind DAS Technology's survey and NPS reporting.
api: DAS Web API (DASWebAPI v1 + v2)
base_url: https://api.digitalairstrike.com
generated: '2026-08-12'
method: generated
source: openapi/_original/das-technology-daswebapi-v1-swagger.json, openapi/_original/das-technology-daswebapi-v2-swagger.json
operations:
  - AccountV2_GetSurveyResponsesV2Async
  - Account_GetSurveyResponsesAsync
  - SurveysV2_GetSurveyResponses
  - SurveyResponse_GetSurveyResponse
  - SurveyResponse_GetSurveyResponseResponses
  - SurveyResponse_PostSurveyResponseResponse
  - SurveyResponse_PostSurveyDealerResponse
  - Survey_RequestSurvey
  - Account_SendSurveyEmail
  - StatsV2_GetNpsSurveyStatsAsync
---

# Close the loop on a survey response

Grounded in operations published in the DAS Web API Swagger documents at
`https://api.digitalairstrike.com/swagger`.

## Before you start

- `Authorization: Bearer <token>` on every call; anonymous requests return `401`.
- Resources are scoped by `accountGuid` (rooftop UUID). Survey sends are scoped by `clientId`.
- Send `Accept: application/json`; errors come back as XML regardless.

## Steps

### 1. Read survey responses for the account

    GET /v2/account/{accountGuid}/survey/responsesV2  → AccountV2_GetSurveyResponsesV2Async

Returns `DAS.Models.PagedResult[DAS.Models.SurveyResponseV2]` — `data` plus a `count` total.
Filter with the bound query object: `query.fromDate`, `query.toDate`, `query.q`,
`query.orderBy`, `query.page`, `query.pageSize`.

Alternatives:
- `GET /v1/account/{accountGuid}/survey/responsesV2` → `Account_GetSurveyResponsesV2Async`
- `GET /v1/survey/{surveyId}/response` → `Survey_GetSurveyResponsesAsync` (by survey, not account)
- `GET /v2/surveys/responses` → `SurveysV2_GetSurveyResponses`

### 2. Open one response and its reply thread

    GET /v1/surveyResponse/{surveyResponseGuid}           → SurveyResponse_GetSurveyResponse
    GET /v1/surveyResponse/{surveyResponse}/response      → SurveyResponse_GetSurveyResponseResponses

Read the thread first — the write in step 3 has no idempotency key.

### 3. Reply

    POST /v1/surveyResponse/{surveyResponseGuid}/response        → SurveyResponse_PostSurveyResponseResponse
    POST /v1/surveyResponse/{surveyResponseGuid}/DealerResponse  → SurveyResponse_PostSurveyDealerResponse

The first posts a response on the thread; the second creates or updates a *pending* dealer
response for the store. **Neither is idempotent.** On a timeout, re-read the thread before
retrying.

### 4. Request a new survey (write — sends a real message)

    POST /v1/survey/accounts/{clientId}/smssurveys           → Survey_RequestSurvey
    POST /v1/account/{dealerGuid}/surveySample/{email}       → Account_SendSurveyEmail

Both operations dispatch a message to a real consumer. There is no idempotency key, no dry-run
mode and no sandbox — DAS Technology publishes no test environment. Treat every call as
production and gate it behind an explicit human confirmation.

### 5. Read the reporting rollup

    GET /v1/account/{accountGuid}/surveyStats             → Account_GetSurveyStatsAsync
    GET /v1/account/{accountGuid}/surveyStatsByInterval   → Account_GetSurveyStatsByIntervalAsync
    GET /v2/stats/survey                                  → StatsV2_GetSurveyStatsAsync
    GET /v2/stats/surveymethod                            → StatsV2_GetSurveyMethodStatsAsync
    GET /v2/stats/npsStats                                → StatsV2_GetNpsSurveyStatsAsync

## Errors and limits

Only `200` is documented; `401` (XML), `404` (HTML) and `500` (XML, with a stack trace) are
what you will actually meet. See `errors/das-technology-problem-types.yml`. No rate limits are
published or signaled — see `rate-limits/das-technology-rate-limits.yml`.
