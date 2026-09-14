---
name: Submit pay data to ADP and retrieve pay statements
description: Push payroll input for a pay period, then read back the produced pay statements and payroll outputs — the money-moving half of ADP, with the checks that belong in front of it.
api: openapi/automatic-data-processing-payroll-pay-data-input-v1-openapi.yml
operations:
  - f149b8b5-443d-484f-9c04-a60da170f90a   # GET /events/payroll/v1/pay-data-input.modify/meta
  - 1616325c-2826-44d6-876b-a806a99700a5   # POST /events/payroll/v1/pay-data-input.modify
  - 880a38af-1ece-4606-b028-6d7d319e2ec9   # GET /payroll/v1/workers/{aoid}/organizational-pay-statements
  - f4f4fd27-1035-42a0-b8eb-802667e6385b   # GET /payroll/v1/workers/{aoid}/organizational-pay-statements/{pay-statement-id}
  - 5f32da43-e14c-4690-b147-359ed5d5ef9b   # GET .../images/{image-id}.{image-extension}
---

# Submit pay data to ADP and retrieve pay statements

This is the highest-consequence surface in the ADP catalogue: it changes what people are paid. An
agent should require explicit, per-run human confirmation before step 2.

## 1. Read the rules for this client's pay data

`GET /events/payroll/v1/pay-data-input.modify/meta` (`f149b8b5-443d-484f-9c04-a60da170f90a`)

ADP's meta for payroll events is unusually rich — it carries regex `pattern`s (effective dates), item
count bounds (`minItems`/`maxItems`, e.g. at most three distribution instructions), field lengths
(`accountNumber` max 17, `routingTransitID` max 9) and enumerated `codeList` values for deposit types.
Validate your payload against this response before posting; ADP will reject on the same rules.

## 2. Post the pay data input event

`POST /events/payroll/v1/pay-data-input.modify` (`1616325c-2826-44d6-876b-a806a99700a5`)

Standard event envelope: `data.eventContext` names the worker/payroll group, `data.transform` carries
the earnings, hours and deduction lines. Success is `eventStatusCode.codeValue = "complete"`.

**No idempotency key exists.** A re-post after a timeout can double-enter pay data. On timeout, read
the current pay data or the resulting statement before retrying — never blind-retry a payroll write.

## 3. Read the produced statements

- `GET /payroll/v1/workers/{aoid}/organizational-pay-statements` (`880a38af-1ece-4606-b028-6d7d319e2ec9`)
- `GET /payroll/v1/workers/{aoid}/organizational-pay-statements/{pay-statement-id}` (`f4f4fd27-1035-42a0-b8eb-802667e6385b`)
- The rendered statement image: `.../images/{image-id}.{image-extension}` (`5f32da43-e14c-4690-b147-359ed5d5ef9b`) — this one returns `image/*` or `application/pdf`, not JSON.

`payroll/v2/payroll-outputs` carries the run-level outputs (associate payment allocations and
summaries) when you need the employer view rather than the employee view.

## 4. Reconciliation

Pay statements are the source of truth; the event you posted is not. Re-read after every run and
reconcile against your own ledger before reporting success.

## Timing

ADP Marketplace maintenance runs Tuesday, Wednesday and Thursday 21:00–06:00 US Eastern and API calls
fail during it. Do not schedule a payroll submission inside that window; ADP's own guidance is to stop
30 minutes before it opens and resume 60 minutes after it closes.

## Personal and financial data

Bank routing numbers, account numbers and pay amounts are in scope here. Never log bodies, never
sample from a live response, and keep the data inside the system the client consented to.
