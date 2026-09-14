---
name: Terminate and rehire a worker in ADP
description: The highest-consequence ADP write pair — end a work assignment and bring the same associate back — with the meta pre-check, the delegation headers, and an explicit statement of what ADP does and does not promise about reversal.
api: openapi/automatic-data-processing-hr-workers-work-assignment-management-v2-openapi.yml
operations:
  - 9b1be338-a34e-4ae2-8255-f7ae20aec06d   # POST /events/hr/v1/worker.work-assignment.terminate
  - f3355703-f654-47ee-a98a-f67b74c4b364   # POST /events/hr/v1/worker.rehire
  - df6fbf20-b396-4db9-b76c-cd7bbd2bef8b   # GET /events/hr/v1/worker.rehire/meta
  - b451d2d2-c440-423a-97a8-4012fb88d6cc   # GET /events/hr/v1/worker.work-assignment.modify/meta
  - 6501f53d-1c15-4ea9-902c-52a49970e0d9   # GET /hr/v2/workers/{aoid}
---

# Terminate and rehire a worker in ADP

This flow ends someone's employment record and their pay. Treat every step as requiring a human
decision that is already made and recorded — an agent executes it, an agent does not decide it.

## 0. Establish who you are acting for

Every ADP operation carries a caller context, not just a token:

| Header | Purpose |
|---|---|
| `orgoid` | the ADP client organization |
| `associateoid` | the acting associate |
| `roleCode` | the ADP product role being exercised — **missing or unassigned returns 400 "Invalid / Missing Role code"** |
| `ADP-Act-As-AssociateOID` / `ADP-Act-As-OrgOID` | act-as delegation |
| `ADP-On-Behalf-Of-AssociateOID` / `ADP-On-Behalf-Of-OrgOID` | on-behalf-of delegation |

An agent acting for an HR administrator sets the act-as/on-behalf-of pair. Do not act as the worker
being terminated.

## 1. Rehearse with `/meta`

`GET /events/hr/v1/worker.work-assignment.modify/meta` (`b451d2d2-c440-423a-97a8-4012fb88d6cc`) and
`GET /events/hr/v1/worker.rehire/meta` (`df6fbf20-b396-4db9-b76c-cd7bbd2bef8b`) return the exact
field rules, required reasons and permitted code-list values for this tenant, and mutate nothing.
This is the closest thing ADP offers to a dry run. Run it before every attempt.

## 2. Terminate the work assignment

`POST /events/hr/v1/worker.work-assignment.terminate` (`9b1be338-a34e-4ae2-8255-f7ae20aec06d`)

The event body follows the standard shape: `eventContext.worker.associateOID` names the worker,
`transform` carries the termination date and reason permitted by `/meta`. Note the `ADP-Action-Reason`
header ADP declares on some lifecycle operations — populate it when `/meta` marks it required.

**Effective dating matters more than wall-clock time here.** ADP generates the notification when the
change is *issued*, not on its effective date, so downstream systems will see the event before it
takes effect. Read the effective date out of the payload rather than assuming "now".

## 3. Rehire

`POST /events/hr/v1/worker.rehire` (`f3355703-f654-47ee-a98a-f67b74c4b364`) re-employs the same
associateOID. The associateOID is permanent — a rehire is not a new worker and must not be created as
one.

## 4. Confirm

`GET /hr/v2/workers/{aoid}` (`6501f53d-1c15-4ea9-902c-52a49970e0d9`) and read
`workAssignments[].assignmentStatus.statusCode.codeValue`.

## Reversal: what is and is not promised

- A reversal **operation** exists: `worker.rehire` reverses `worker.work-assignment.terminate`.
- A reversal **window** does not. ADP publishes no deadline inside which a termination can be undone,
  and payroll consequences (a final pay run, tax filings) are not undone by a rehire event.
- Therefore: never tell a user a termination is reversible "within N days". Say a rehire event exists
  and that the payroll effects must be reconciled separately.

## Not idempotent

No idempotency key exists on this surface. A retried terminate can post twice. On timeout, re-read the
worker before re-posting.
