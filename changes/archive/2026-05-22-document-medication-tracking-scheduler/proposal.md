## Why

The medication-tracking scheduler — shipped under SRD-1233/SRD-1257/SRD-1260/SRD-1154 — is in production but its behavior lives only in `docs/features/SRD-1233-medication-tracking-api.md` and the code itself. There is no OpenSpec capability that future changes can reference as a baseline, so any follow-up work (the planned `PUT/DELETE` on schedules, the async dose-generation job, retroactive intake quantity edits) has nothing to delta against.

This change establishes that baseline by reverse-engineering the current behavior into a `medication-tracking-scheduler` capability spec. No code changes.

## What Changes

- Add a new `medication-tracking-scheduler` capability spec describing the seven endpoints that are live today:
  1. `POST /api/v1/patient/medication-tracking/schedules`
  2. `GET  /api/v1/patient/medication-tracking/schedules`
  3. `GET  /api/v1/patient/medication-tracking/scheduled-doses`
  4. `GET  /api/v1/patient/medication-tracking/scheduled-doses/next`
  5. `PATCH /api/v1/patient/medication-tracking/schedules/{schedule}/intakes/{intake}/planned-quantity`
  6. `POST /api/v1/patient/medication-tracking/tracking-logs/by-dose`
  7. `POST /api/v1/patient/medication-tracking/tracking-logs/by-schedule`
- Capture the cross-cutting invariants: order-product locking, schedule-per-order uniqueness (409), per-patient schedule cap (422), synchronous dose materialization, forward-only quantity edits, off-schedule window guards.
- No requirements are added or removed — every requirement reflects behavior already shipping on `master`.

## Capabilities

### New Capabilities

- `medication-tracking-scheduler`: patient-facing API for creating and viewing medication schedules (Rx-linked or custom), materializing scheduled doses across a horizon, exposing a daily/next dose view, updating the planned quantity of an intake forward in time, and logging intake events (by-dose for planned slots, by-schedule for off-schedule events).

### Modified Capabilities

_None — there is no pre-existing `openspec/specs/` content for this area._

## Impact

- **Specs added**: `openspec/specs/medication-tracking-scheduler/spec.md` (after archive).
- **Code**: no changes. The tasks in this change are verification-only — read the controllers, services, requests and resources and confirm each requirement matches.
- **Out of scope (explicitly deferred to follow-up changes)**:
  - `PUT` / `DELETE /schedules/{schedule}` — not implemented yet.
  - Async background dose generation — currently synchronous inside the create request.
  - Retroactive intake quantity edits — current rule is forward-only.
  - The legacy `/medication-tracking/medication-schedules` routes (older `MedicationSchedule` model with `apply_times`) — separate capability if/when documented.
  - The plain `/medication-tracking/tracking-logs` index/create/update/delete and the `MedicationDoseReminder` reminder pipeline.