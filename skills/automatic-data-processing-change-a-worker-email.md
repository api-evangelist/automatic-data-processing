---
name: Change a worker's personal email in ADP
description: The canonical ADP write — discover the per-tenant rules with /meta, POST the named event, then re-read the worker to confirm. Shows why ADP writes are events, not PATCHes, and why they are not idempotent.
api: openapi/automatic-data-processing-hr-workers-personal-communication-management-v2-openapi.yml
operations:
  - 6bd32219-6a80-47af-a923-f1ff57f33922   # POST /events/hr/v1/worker.personal-communication.email.add
  - 01616688-5568-4b35-9a69-19a094fd3717   # POST /events/hr/v1/worker.personal-communication.email.change
  - db880d66-662c-492c-b03f-6318f1a67adc   # POST /events/hr/v1/worker.personal-communication.email.remove
  - 6501f53d-1c15-4ea9-902c-52a49970e0d9   # GET /hr/v2/workers/{aoid} — verify
---

# Change a worker's personal email in ADP

**There is no PATCH on a worker.** ADP's write model is CQRS: you POST a named *event* to
`/events/{domain}/v1/{eventName}`. Looking for `PATCH /hr/v2/workers/{aoid}` and not finding it is the
single most common way to conclude, wrongly, that ADP has no write API.

## 1. Read the rules first — `/meta` is not optional

```
GET https://api.adp.com/events/hr/v1/worker.personal-communication.email.add/meta
```

ADP returns the validation and presentation rules **for this client, for this actor, right now**:
which fields are required, which are `readOnly` or `hidden`, `maxLength` (256 for `emailUri`), regex
`pattern`s, `minItems`/`maxItems`, and enumerated `codeList` values. ADP states plainly that an API
user must use the meta response together with the event schema to post a valid body. The OpenAPI
alone will not tell you what this tenant requires.

## 2. Post the event

`POST /events/hr/v1/worker.personal-communication.email.add` (`6bd32219-6a80-47af-a923-f1ff57f33922`)

```json
{ "events": [ {
  "serviceCategoryCode": { "codeValue": "hr" },
  "eventNameCode": { "codeValue": "worker.personalCommunication.email.add" },
  "originator": { "associateOID": "<acting associate>" },
  "actor":      { "associateOID": "<acting associate>" },
  "data": {
    "eventContext": { "worker": { "associateOID": "<target aoid>" } },
    "transform": {
      "eventStatusCode": { "codeValue": "submit" },
      "worker": { "person": { "communication": { "email": { "emailUri": "<new address>" } } } }
    }
  }
} ] }
```

A success is `201` with `eventStatusCode.codeValue = "complete"` and the resulting value echoed under
`data.output`.

Use `.change` (`01616688-5568-4b35-9a69-19a094fd3717`) when an address already exists and `.add` when
it does not — posting `.add` over an existing entry is not the same operation.

## 3. Do NOT blind-retry

ADP publishes **no idempotency key**. There is no `Idempotency-Key` header and no client request id
that de-duplicates a replay, so a retry after a timeout can create a second entry. On a timeout:
re-read the worker with `GET /hr/v2/workers/{aoid}` (`6501f53d-1c15-4ea9-902c-52a49970e0d9`) and only
re-post if the change is genuinely absent.

The `ETag` / `If-Match` / `412 Precondition Failed` machinery ADP does provide is optimistic
concurrency — protection against a lost update, not against a duplicate submission. They are not the
same guarantee.

## 4. Reversing it

`POST /events/hr/v1/worker.personal-communication.email.remove` (`db880d66-662c-492c-b03f-6318f1a67adc`)
is the paired reversal. **ADP publishes no time window for reversals** — the constraint is
effective-dating, which is per-client and surfaced through `/meta`, not a stated policy. Never promise
a user that a write can be undone within N days; ADP does not say that.

## Personal data

`emailUri` is employee PII. Redact it from logs and never reuse a live value as a sample.
