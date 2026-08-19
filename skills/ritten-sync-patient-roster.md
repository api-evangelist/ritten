---
name: Sync a Ritten patient roster
description: Page the full patient roster out of a Ritten clinic tenant and keep an external system in sync, including relationships and documents.
api: openapi/ritten-external-api-openapi.yaml
operations: [postOAuthToken, listPatients, getPatientById, getPatientByExternalId, patchPatient, listPatientRelationships, createPatientRelationship, attachDocument]
---

# Sync a Ritten patient roster

## 1. Authenticate once

`postOAuthToken` → cache the 24-hour token. Send `Authorization: Bearer …` and
`X-Ritten-Tenant: <clinic-id>` on every request.

## 2. Page the roster

`listPatients` — `GET /patients` with `limit` and `offset`.

Pagination is **offset-based** and the response carries **no total or next-cursor field**. Page until
a short page comes back. Useful filters: `search`, `programStatus`, `tagIds` + `matchAllTags`,
`clinical` (includes clinical detail), `createdAfter` for incremental pulls.

Respect **50 requests/second sustained, 100 burst**. There are no `RateLimit-*` response headers, so
you cannot read remaining quota — pace yourself and treat `429` as a hard backoff signal.

## 3. Fetch detail as needed

`getPatientById` — `GET /patients/{id}`.

## 4. Write back changes

`patchPatient` — `PATCH /patients/{id}`. A `400` is returned when the payload is invalid **or when no
fields are set** — do not send empty patches.

## 5. Relationships and documents

- `listPatientRelationships` / `createPatientRelationship` — `GET|POST /patients/{id}/relationships`
- `attachDocument` — `POST /patients/{id}/attachments`

## Incremental sync notes

- Store the Ritten patient `id` (UUID) alongside your own id, and set `externalId` on Ritten records so
  `getPatientByExternalId` can resolve them later.
- There is **no idempotency key and no webhook signature** on this API. Reconcile by re-reading rather
  than assuming a write landed.
- Subscribe to the `patient.created`, `patient.admit`, `patient.transfer` and `patient.discharge`
  webhooks (see `asyncapi/ritten-webhooks.yml`) to avoid full-table polling. Subscription is provisioned
  by Ritten, not self-service.

## PHI handling

Patient records are PHI under HIPAA and, for substance-use treatment, 42 CFR Part 2. Do not write
payloads to logs, and do not pass them into a model context without a BAA and Part 2 consent.
