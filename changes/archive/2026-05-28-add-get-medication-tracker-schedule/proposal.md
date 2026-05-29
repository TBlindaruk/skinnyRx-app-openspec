## Why

The FE can list a patient's medication schedules but has no way to fetch a single schedule by id without re-paginating through `GET /schedules` and filtering client-side with `?id[]=`. A deep-link or refresh-on-detail view needs a direct `GET /schedules/{schedule}` returning the same `ScheduleResource` shape as the list items.

## What Changes

- Add `GET /api/v1/patient/medication-tracking/schedules/{schedule}` — single-item lookup, authenticated patient only.
- Authorization via the existing `can:manage,schedule` policy (returns `403` for other patients' schedules).
- Implicit route binding excludes soft-deleted rows, so a deleted schedule returns `404` — consistent with how `DELETE` resolves the same path.
- Response body is a single `ScheduleResource` — identical fields to one item from `GET /schedules` (intakes eager-loaded, `image_path` resolved from product).

## Capabilities

### New Capabilities
- _(none)_

### Modified Capabilities
- `medication-tracking-scheduler`: add a new "Show a single medication schedule" requirement alongside the existing list requirement.

## Impact

- `routes/api/api.php` — one new route declaration inside the existing `{schedule}` group (so it sits behind `can:manage,schedule`).
- `app/Http/Controllers/Api/Patient/MedicationTracking/ScheduleController.php` — one new `get(MedicationTrackingSchedule $schedule): JsonResponse` action.
- `tests/Feature` — new test file covering happy path, not-owner `403`, soft-deleted `404`, unauthenticated `401`.
- No DB migrations, no DTOs, no new resources — reuses `ScheduleResource`.
