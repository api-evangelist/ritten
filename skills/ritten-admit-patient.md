---
name: Admit a patient to a Ritten program
description: Create or resolve a patient record in a Ritten clinic tenant, admit them to a program, and record the intake encounter and vitals.
api: openapi/ritten-external-api-openapi.yaml
operations: [postOAuthToken, getPatientByExternalId, createPatient, listPrograms, listEncounterTypes, postEncounter, postPatientVitals]
---

# Admit a patient to a Ritten program

Ritten's External API is a multi-tenant behavioral-health EMR. This flow handles PHI under HIPAA
and 42 CFR Part 2 — do not log request or response bodies.

## 1. Get an access token

`postOAuthToken` — `POST https://api.ritten.io/v1/oauth/token`

Body: `{"client_id": …, "client_secret": …, "audience": "https://external-api.ritten.io", "grant_type": "client_credentials"}`

Tokens last 24 hours (`expires_in: 86400`). **Cache the token.** You are limited to **2 token mints per
hour and 3 per day** per application — minting per request will lock you out. The endpoint caches
server-side, so one mint per day is sufficient at any traffic volume.

## 2. Set the tenant header on every call

`X-Ritten-Tenant: <clinic-id>` is **required on every request** and selects the clinic instance.
Also send `Authorization: Bearer <access_token>`.

## 3. Resolve the patient before creating one

`getPatientByExternalId` — `GET /patients/external/{externalId}` — look up by your own system's id first.

- `200` → the patient already exists; skip to step 4.
- `404` → not found; create with `createPatient` (`POST /patients`).

**There is no idempotency key on this API.** `createPatient` is not idempotent — a retry after a
timeout will create a duplicate patient. Always re-run `getPatientByExternalId` before retrying a
create, and set `externalId` on creation so the lookup works next time.

## 4. Pick the program

`listPrograms` — `GET /programs` (supports `limit`/`offset`, `programType`, `facilityId`).
Take the program `id` for the level of care being admitted to.

## 5. Record the intake encounter

`listEncounterTypes` — `GET /encounter-types` — encounter types are tenant-configured; resolve the
type id rather than hardcoding it. Archived types are treated as not found.

`postEncounter` — `POST /encounters` — creates the visit against the patient and encounter type.

## 6. Record intake vitals (optional)

`postPatientVitals` — `POST /patients/{id}/vitals`.

## Error handling

- `400` — validation error. Bodies for non-OAuth errors carry **no declared schema**; read the message defensively.
- `401` — token expired or rejected. Re-mint **at most** within your 2/hour quota.
- `403` — the integration has not been provisioned for this feature or clinic. Contact Ritten; do not retry.
- `404` — patient, program, or encounter type not found (archived encounter types return 404).
- `429` — rate limited (50 req/s sustained, 100 burst) **or** token mint quota exhausted. No `Retry-After`
  header is returned, so back off exponentially.
