---
name: Create and manage an ADP time-off request
description: Check a worker's balance, create a paid-time-off request, amend it, and cancel the underlying leave — a complete, reversible flow with a named cancel operation.
api: openapi/automatic-data-processing-time-time-off-requests-v2-openapi.yml
operations:
  - 282fb8d5-4746-4857-9432-e59524acc578   # GET /time/v2/workers/{aoid}/time-off-details/time-off-requests
  - f0010334-0978-41c5-9eef-837cc9350ccb   # POST /time/v2/workers/{aoid}/time-off-requests
  - 647e8a1a-602c-4639-8512-74ee148c1954   # PUT /time/v2/workers/{aoid}/time-off-requests/{request-id}
  - 5723b54d-cd30-4fc9-a2f4-f761bb76803b   # POST /events/hr/v1/worker.leave.cancel
  - dd5a01cc-0901-4aa5-b5b3-4f826b021ef7   # GET /hr/v2/workers/{aoid}/leaves
---

# Create and manage an ADP time-off request

One of the few ADP surfaces with conventional REST writes (`POST`/`PUT` on a resource) rather than
event posts — worth knowing, because assuming the event pattern everywhere will send you looking for
an operation that does not exist here.

## 1. Read what the worker already has

- `GET /time/v2/workers/{aoid}/time-off-details/time-off-requests` (`282fb8d5-4746-4857-9432-e59524acc578`) — existing requests
- `GET /hr/v2/workers/{aoid}/leaves` (`dd5a01cc-0901-4aa5-b5b3-4f826b021ef7`) — leaves of absence
- `time/v3/time-off-balances` carries the accrual balances; check the balance before creating a
  request that cannot be granted.

## 2. Create the request

`POST /time/v2/workers/{aoid}/time-off-requests` (`f0010334-0978-41c5-9eef-837cc9350ccb`)

Note that ADP Workforce Now publishes time-off-requests at **both v2 and v3** concurrently, with
different operation sets (v2 has five operations, v3 has three). Neither is marked deprecated. Pin the
version you integrated against and re-check it before a migration — ADP publishes no deprecation or
sunset policy, so a version disappearing will not be announced through a header.

## 3. Amend

`PUT /time/v2/workers/{aoid}/time-off-requests/{request-id}` (`647e8a1a-602c-4639-8512-74ee148c1954`)

Send back the `ETag` you received on the read as `If-Match`. A `412 Precondition Failed` means someone
else changed the request underneath you — re-read and re-apply rather than forcing the write.

## 4. Cancel

For a leave of absence the reversal is an event:
`POST /events/hr/v1/worker.leave.cancel` (`5723b54d-cd30-4fc9-a2f4-f761bb76803b`). Call its `/meta`
first for the per-tenant rules.

**No cancellation window is published.** ADP does not state a deadline after which a leave can no
longer be cancelled. Do not invent one for the user.

## Errors

`403 access_denied` means the client's consent was withdrawn or the subscription is suspended — it is
not a permissions bug in your code and must not be retried. `400` carries a ConfirmMessage whose
`processMessages[].userMessage.messageTxt` is the human-readable reason.
