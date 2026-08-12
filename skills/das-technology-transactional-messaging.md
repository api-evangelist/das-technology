---
name: das-technology-transactional-messaging
description: Send a consumer message through the DAS Web API mail surface — asynchronously or synchronously — then poll delivery status and events by token, and provision or deprovision the SMS number behind an account.
api: DAS Web API (DASWebAPI v1)
base_url: https://api.digitalairstrike.com
generated: '2026-08-12'
method: generated
source: openapi/_original/das-technology-daswebapi-v1-swagger.json
operations:
  - Mail_PostMessageAsynchronous
  - Mail_PostMessageSynchronous
  - Mail_GetStatus
  - Mail_GetEventsAsync
  - Notification_ProvisionSmsNumber
  - Notification_DeprovisionSmsNumber
  - Account_GetSMSProvisionDetails
---

# Send a message and track it

Grounded in operations published in the DAS Web API v1 Swagger document at
`https://api.digitalairstrike.com/swagger/docs/v1`.

**This flow writes to the real world.** Every send reaches a real consumer. There is no sandbox,
no test mode and no dry-run — DAS Technology publishes no test environment. Gate every call in
this skill behind an explicit human confirmation.

## Before you start

- `Authorization: Bearer <token>` on every call.
- Send `Accept: application/json`.

## Steps

### 1. Send

    POST /v1/mail/send      → Mail_PostMessageAsynchronous   (queued)
    POST /v1/mail/sendsync  → Mail_PostMessageSynchronous    (blocking)

Both take a `DAS.Models.Envelope` body. The envelope carries the message and its
`DAS.Models.EnvelopeAttachment` list. The response is an untyped object in the published
contract; the async send is the one that yields the tracking `token` used below.

**No idempotency key.** The API accepts no `Idempotency-Key` header and returns no request
identifier, so a retried send is a second real message. If a send times out, poll status by the
token you already have rather than re-posting. If you never received a token, escalate to a
human — there is no safe automatic recovery.

### 2. Poll delivery status

    GET /v1/mail/status/{token}   → Mail_GetStatus

### 3. Read delivery events

    GET /v1/mail/events/{token}   → Mail_GetEventsAsync

This is a **polling** surface. DAS Technology publishes no webhooks, no callbacks and no
AsyncAPI document, so there is no way to be notified of a delivery event — you must poll by
token. Pace your polling yourself; no rate limits are published or signaled.

### 4. SMS number provisioning (rarely run)

    GET    /v1/account/smsprovisiondetails/{accountGuid}          → Account_GetSMSProvisionDetails
    POST   /v1/notification/{clientId}/sms/phonenumbers/new       → Notification_ProvisionSmsNumber
    DELETE /v1/notification/{clientId}/sms/phonenumbers/{smsNumber} → Notification_DeprovisionSmsNumber

Read the provisioning details first — note it is keyed by `accountGuid` while the two write
operations are keyed by `clientId`, so you need both identifiers. Provisioning allocates a real
phone number; deprovisioning releases it. Neither is idempotent and neither is reversible by re-running the other — treat
both as one-way operations requiring human approval.

## Errors

Only `200` is documented across the whole API. Expect `401` (application/xml), `404` (text/html)
and `500` (application/xml, currently including a stack trace). Full catalog:
`errors/das-technology-problem-types.yml`.
