---
name: Read the ADP workforce
description: Authenticate against ADP, page through a client's worker roster with OData query options, and read one worker in full — the safe, read-only entry point to every other ADP flow.
api: openapi/automatic-data-processing-hr-workers-v2-openapi.yml
operations:
  - 2aa32939-9059-4187-8785-0eb593ece43b   # GET /hr/v2/workers — Workers Data Collection
  - 6501f53d-1c15-4ea9-902c-52a49970e0d9   # GET /hr/v2/workers/{aoid} — Individual Worker Data
  - ff6cbf8f-dd8b-4d22-8086-1caa805fe384   # GET /hr/v2/workers/meta — Worker Meta
---

# Read the ADP workforce

Read-only. Nothing in this skill mutates ADP data.

## 1. Get a token

ADP will not talk to you over ordinary TLS. Every call — including the token call — needs an ADP-issued
X.509 client certificate presented on the handshake.

```
POST https://accounts.adp.com/auth/oauth/v2/token
  (client certificate + key attached)
  grant_type=client_credentials&client_id=<id>&client_secret=<secret>
```

Tokens live 60 minutes. Cache one and re-use it; ADP explicitly says not to request a token per call.
UAT uses `uat-accounts.adp.com` and `uat-api.adp.com` with a separate certificate.

## 2. Ask what you are allowed to query

`GET /hr/v2/workers/meta` (`ff6cbf8f-dd8b-4d22-8086-1caa805fe384`) returns the OData options this
operation supports **for this client**. Supported `$filter` paths vary by ADP product, so read
`queryOptionCode` before composing a filter rather than guessing.

## 3. Page the roster

`GET /hr/v2/workers` (`2aa32939-9059-4187-8785-0eb593ece43b`)

```
GET https://api.adp.com/hr/v2/workers?$top=20&$skip=0
Authorization: Bearer <token>
orgoid: <client org oid>
associateoid: <acting associate oid>
roleCode: <your registered role>
```

- `$select` trims the payload: `?$select=workers/associateOID,workers/person/legalName`
- `$filter` narrows it: `?$filter=workers/workAssignments/assignmentStatus/statusCode/codeValue eq 'A'`
- Walk forward with `$skip += $top`.

**Stop condition is a 400, not an empty 200.** When `$skip` passes the end of the collection ADP
returns `400 Bad Request` with a ConfirmMessage. Treat that specific 400 at the paging boundary as
"no more rows"; do not retry it as an error.

## 4. Read one worker

`GET /hr/v2/workers/{aoid}` (`6501f53d-1c15-4ea9-902c-52a49970e0d9`). The `aoid` is the
**associateOID** — ADP's permanent, opaque worker key. Carry it, not a name or an employee number.

## 5. Cache correctly

200 responses carry an `ETag` (346 of 353 in the harvested contracts). Send it back on `If-None-Match`
and handle `304 Not Modified` rather than re-downloading the roster.

## Errors you will actually hit

| Status | Meaning | Do this |
|---|---|---|
| 401 `invalid_token` | token expired (60 min) or certificate not presented | re-token; confirm the client cert is installed for **both** hosts |
| 403 `invalid_scope` | this API is not registered to your application | ADP must add it — contact your ADP rep / Marketplace Technical Advisor |
| 403 `access_denied` | the client withdrew consent or the subscription is suspended | ask the client to re-consent; do not retry |
| 400 "Invalid / Missing Role code" | `roleCode` header absent or not assigned | have the permission assigned in the ADP product |
| 429 | throttled — ADP publishes no number | exponential backoff with jitter, honour `Retry-After` |
| 503 | usually a maintenance window (Tue/Wed/Thu 21:00–06:00 ET) | stop 30 min before, resume 60 min after |

## Personal data

Every response here is employee PII. Do not log bodies, do not use a live response as an example, and
do not persist worker records outside the system the client consented to.
