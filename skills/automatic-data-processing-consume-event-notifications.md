---
name: Consume ADP event notifications
description: Keep a downstream system in sync with ADP by draining the event-notification queue or receiving signed webhooks — including the acknowledgement contract and the signature check most integrations get wrong.
api: openapi/automatic-data-processing-hr-workers-v2-openapi.yml
operations:
  - 6501f53d-1c15-4ea9-902c-52a49970e0d9   # GET /hr/v2/workers/{aoid} — re-read after a notification
  - 2aa32939-9059-4187-8785-0eb593ece43b   # GET /hr/v2/workers
---

# Consume ADP event notifications

ADP offers two delivery mechanisms for the same event stream. Choose one.

## Prerequisite: subscribe

An event you have not subscribed to is never delivered, and the polling endpoint answers `403` for it.
API Central clients add events at *Projects > Select a Project > APIs > Events > Add Events*;
Marketplace partners go through their Marketplace Technical Advisor.

## Option A — polling

```
GET  https://api.adp.com/core/v1/event-notification-messages
DELETE https://api.adp.com/core/v1/event-notification-messages/{adp-msg-msgid}
```

- One message per GET, FIFO.
- The message id arrives as the **response header** `adp-msg-msgid`, not in the body.
- `200` = a message. `204` = the queue is empty — this is the normal idle response, not an error.
- You **must** DELETE the message before the next one is served. Loop: GET → process → DELETE → repeat
  until `204`.
- Deleting twice returns `404`; treat it as already-done.

## Option B — webhooks

ADP added webhook delivery in **October 2025**. ADP watches the queue and POSTs to an endpoint you
host.

Requirements ADP states:

- a publicly reachable endpoint hosted in the **United States**
- secured with one of: API-key bearer, HTTP basic, or OAuth 2.0 client credentials
- an acknowledgement of `200`, `201` or `202` **with** this body:

```json
{ "status": "success", "timestamp": "YYYY-MM-DDTHH:MM:SSZ" }
```

Anything else marks the attempt failed and starts ADP's retry mechanism.

### Verify the signature — do this before you parse the body

Each delivery carries `adpx-messageauthentication`: an **HMAC-SHA256** whose *message* is your data
connector **client ID** and whose *key* is your data connector **client secret**. Both come from
*Project Details > Credentials* in API Central or Partner Self-Service. Reject any delivery whose
computed HMAC does not match.

### The webhook body is not the polling body

Webhook deliveries wrap the polling event in extra metadata:

| Path | Meaning |
|---|---|
| `/meta/messageId` | queue message id, unique per event |
| `/events/eventNameCode/codeValue` | the event that fired — switch on this |
| `/events/data/eventContext/worker/associateOID` | the worker that changed |
| `/events/transform/` | the new value (product- and event-dependent) |

Code written against the polling shape will not read a webhook body unchanged.

## Treat the notification as a trigger, not as truth

ADP says this explicitly: there is always lag between when a notification is generated and when you
process it, so re-read the resource — `GET /hr/v2/workers/{aoid}`
(`6501f53d-1c15-4ea9-902c-52a49970e0d9`) — rather than trusting the payload.

And on effective dating: unless noted otherwise the notification fires when the change is **issued**,
not on its effective date. Read the effective date out of the payload before acting on it.

## Testing

Save a webhook configuration and select **Test webhook** in API Central or Partner Self-Service. ADP
reports success or failure before you enable it.

## Event naming

`<object>.<facet>.<verb>` — `worker.hire`, `worker.rehire`,
`worker.personalCommunication.email.add`, `worker.leave.cancel`,
`worker.work-assignment.terminate`. The per-product data dictionary is the catalogue; ADP publishes no
single machine-readable event list and no AsyncAPI document.
