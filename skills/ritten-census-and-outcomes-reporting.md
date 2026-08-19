---
name: Pull Ritten census and outcomes reports
description: Extract census, admissions, discharges, length-of-stay and outcomes reporting from Ritten's insights endpoints, as JSON or CSV.
api: openapi/ritten-external-api-openapi.yaml
operations: [postOAuthToken, getCensusReport, getProgramCensusReport, getFacilityCensusReport, getAdmissionsReport, getDischargesReport, getAlosReport, getFormOutcomesReport, getEventAuditReport]
---

# Pull Ritten census and outcomes reports

The `/insights/*` family is Ritten's reporting and data-export surface — 17 read-only operations.

## 1. Authenticate

`postOAuthToken` → cache the 24-hour token. Send `Authorization: Bearer …` and `X-Ritten-Tenant`.

## 2. Choose the report

| Need | Operation |
|---|---|
| Current census | `getCensusReport` — `GET /insights/census-report` |
| Census by program | `getProgramCensusReport` — `GET /insights/program-census` |
| Census by facility | `getFacilityCensusReport` — `GET /insights/facility-census` |
| Admissions over a period | `getAdmissionsReport` — `GET /insights/admissions` |
| Discharges over a period | `getDischargesReport` — `GET /insights/discharges` |
| Average length of stay | `getAlosReport` — `GET /insights/alos` |
| Form/assessment outcomes | `getFormOutcomesReport` — `GET /insights/form-outcomes` |
| Audit trail of events | `getEventAuditReport` — `GET /insights/event-audit` |

## 3. Scope the window

Most insights operations take `startDate` and `endDate`. A `400` is returned for a missing required
date, an invalid date format, or an invalid range — validate before calling.

## 4. Choose the format

Add `csv=true` to export CSV instead of JSON (supported on 16 operations). Use CSV for bulk extracts;
JSON responses are not paginated with a total, so large windows are better pulled as CSV.

## 5. Pace the extract

50 req/s sustained, 100 burst, `429` on exhaustion with **no `Retry-After` header**. Reporting pulls are
the easiest way to trip this — run them serially with backoff, and prefer one wide date range over many
narrow ones.

## PHI

Census and audit reports contain PHI under HIPAA and 42 CFR Part 2. Treat CSV exports as regulated
data at rest.
