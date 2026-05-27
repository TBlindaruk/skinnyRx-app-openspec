## Why

`pain_level` is collected on every medication tracking log today — both `POST /api/v1/patient/medication-tracking/tracking-logs/by-dose` and `POST .../by-schedule` declare it `required`. The pain rating is only clinically meaningful for injectable SkinnyRx prescriptions (subcutaneous injections, e.g. GLP-1s) where the patient may feel injection-site pain. For custom (patient-defined) schedules and for oral Rx products (PILLS, DROPS), the field is asked for but adds no value, clutters the UX, and pollutes the data set with values the UI is about to hide. Mobile is updating the form to drop the field for non-injection schedules; the API must do the same so the contract matches what's actually sent.

## What Changes

- `pain_level` becomes conditionally required on `POST /api/v1/patient/medication-tracking/tracking-logs/by-dose`:
  - **Required** (`integer`, `between:0,10`) when the resolved scheduled dose belongs to a schedule whose product is of type `INJECTION`.
  - **Forbidden** (must not be present, or must be `null`) for every other schedule — custom schedules (`product_id IS NULL`) and Rx schedules whose product type is `PILLS` or `DROPS`.
- Same conditional rule applied to `POST /api/v1/patient/medication-tracking/tracking-logs/by-schedule` (off-schedule logging), keyed off the schedule's product type.
- `tracking_logs.pain_level` SHALL be persisted as `NULL` whenever the request does not (or cannot) carry the field. The DB column already permits `NULL`; no migration needed.
- The OpenAPI schemas for both `CreateByDoseRequest` and `CreateByScheduleRequest` drop `pain_level` from `required`, document it as `nullable`, and describe the conditional rule.
- **BREAKING** for any client that currently sends `pain_level` for non-injection schedules: the field will now trigger a `422` "must not be present" error. The mobile and web clients are being updated in lockstep — no third-party consumers exist for these endpoints.

## Capabilities

### New Capabilities
<!-- none -->

### Modified Capabilities
- `medication-tracking-scheduler`: the two "Log a tracking event" requirements (`Log a tracking event against a scheduled dose`, `Log an off-schedule intake against a schedule`) change their `pain_level` rule from unconditionally required to conditional on the schedule's product type being `INJECTION`.

## Impact

- **Validation layer**: `app/Http/Requests/Api/Patient/MedicationTracking/TrackingLog/CreateByDoseRequest.php` and `CreateByScheduleRequest.php` switch `pain_level` from `required` to a conditional rule and become nullable on the getter side.
- **Controllers**: `CreateByDoseController` and `CreateByScheduleController` already resolve the schedule/dose (and load `product`) before dispatching to the action — they now decide whether to forward `pain_level` based on the product type.
- **DTOs**: `CreateDoseTrackingLogDto::$painLevel` and `CreateOffScheduleTrackingLogDto::$painLevel` become `?int`.
- **Actions**: `CreateDoseTrackingLogAction` and `CreateOffScheduleTrackingLogAction` continue to write the value verbatim; no branching there.
- **OpenAPI / docs**: schema attributes on the two request classes updated; spec deltas applied to `openspec/specs/medication-tracking-scheduler/spec.md`.
- **Tests**: feature tests under `tests/Feature/Api/Patient/MedicationTracking/TrackingLog/` extended to cover (a) injection schedules still require pain_level, (b) custom/PILLS/DROPS schedules reject pain_level and persist `NULL`.
- **DB / migrations**: none — `pain_level` is already nullable.
- **Consumers**: mobile and web clients (in flight). No external/B2B consumers.
