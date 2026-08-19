---
name: Work the Ritten CRM admissions pipeline
description: Read and advance admissions-pipeline cases in Ritten's CRM, adding notes and action items as a referral moves toward admission.
api: openapi/ritten-external-api-openapi.yaml
operations: [postOAuthToken, listCases, createCase, getCase, updateCase, createCaseNote, createCaseActionItem, listOrganizations, listOrganizationMembers]
---

# Work the Ritten CRM admissions pipeline

A Ritten "case" is a referral in the admissions pipeline, before or alongside a patient record.

> **Provisioning gate:** CRM endpoints require CRM to be enabled for the target clinic **and** the
> integration to be explicitly granted organization access by Ritten. A `403` here means "not
> provisioned" — it is not retryable. Contact Ritten to enable access.

## 1. Authenticate

`postOAuthToken` → cache the token (24h; 2 mints/hour, 3/day). Send `X-Ritten-Tenant` on every call.

## 2. List and read cases

- `listCases` — `GET /cases` (`limit`, `offset`, `statuses`, `assignedUserIds`, `sortBy`, date-range filters)
- `getCase` — `GET /cases/{id}`

## 3. Create and advance a case

- `createCase` — `POST /cases`. Returns `400` on an unknown tag or case source — resolve those first.
- `updateCase` — `PATCH /cases/{id}`. Returns `400` if the case is **archived**; archived cases cannot be advanced.

## 4. Annotate

- `createCaseNote` — `POST /cases/{id}/notes`
- `createCaseActionItem` — `POST /cases/{id}/action-items`

## 5. Referral sources

- `listOrganizations` — `GET /organizations` (referring organizations)
- `listOrganizationMembers` — `GET /organizations/{id}/members`

## Events

`case.created` and `case.status.update` webhooks fire on pipeline movement — see
`asyncapi/ritten-webhooks.yml`. Payloads are unsigned; verify by re-reading `getCase` before acting on
anything consequential.

## Errors

`400` validation / archived case / unknown tag · `403` CRM or organization access not provisioned ·
`404` case or organization not found · `429` rate limited, no `Retry-After` header.
