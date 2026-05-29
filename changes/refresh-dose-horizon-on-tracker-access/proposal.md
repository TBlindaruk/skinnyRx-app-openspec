## Why

The `medication-tracking:regenerate-doses` monthly cron tops up the year-long `scheduled`-type dose horizon for **every** schedule, then tries to guess (via order-status / activity proxies) which patients are still engaged so it doesn't waste work on churned ones. But nothing other than the patient's **own** tracker endpoints ever reads materialized scheduled doses — there is no proactive reminder or notification over them. So the engagement guess is solving a problem that disappears entirely if we generate **lazily, on access**: top up a patient's horizon when they actually interact with the medication tracker. Engagement stops being a heuristic and becomes a fact — the request itself.

## What Changes

- **BREAKING (internal):** Remove the `medication-tracking:regenerate-doses` artisan command and its `App\Console\Kernel::schedule()` registration. Horizon maintenance is no longer periodic.
- Add a `EnsureDoseHorizon` middleware on the `/api/v1/patient/medication-tracking` route group (alongside the existing `TrackActivity`). On each request it reads the authenticated patient's `dose_horizon_refreshed_at` (already hydrated by the guard — no extra query) and, when that is `NULL` or older than a 7-day throttle window, dispatches one queued top-up job. Otherwise it does nothing. It never blocks the response.
- Add a `RefreshPatientDoseHorizon` that iterates the patient's eligible schedules (not soft-deleted, `schedule_type != as_needed`, `ends_on` open), reuses the existing per-schedule window computation + `ScheduledDoseGenerator` to extend the horizon idempotently, and stamps `patient.dose_horizon_refreshed_at = now`.
- Drop the patient-engagement gate entirely (no `active order` / `last_active_at >= now - 90 days` predicate). On-access generation makes it redundant.
- Add a `dose_horizon_refreshed_at` nullable timestamp column to `patients` (the throttle marker), with a cast + getter on the `Patient` model.
- Add a `HORIZON_REFRESH_THROTTLE_DAYS = 7` knob to `MedicationTrackingConfig`.

## Capabilities

### New Capabilities

_None._

### Modified Capabilities

- `medication-tracking-scheduler`: **remove** the "Periodic top-up of the scheduled-dose horizon" requirement (the cron) and **add** an "On-access top-up of the scheduled-dose horizon" requirement (middleware-triggered, throttled, per-patient job). The per-schedule window math and `GenerateScheduledDosesJob` idempotency are preserved; only the trigger and the patient-eligibility model change.

## Impact

- **Removed:** `app/Console/Commands/MedicationTracking/RegenerateScheduledDosesCommand.php` and its `Kernel::schedule()` entry; the command's feature test.
- **Added:** `app/Http/Middleware/EnsureDoseHorizon.php`; `app/Jobs/MedicationTracking/RefreshPatientDoseHorizon.php`; a migration adding `patients.dose_horizon_refreshed_at`; `Patient` cast + `getDoseHorizonRefreshedAt()` getter; `MedicationTrackingConfig::HORIZON_REFRESH_THROTTLE_DAYS`.
- **Changed:** `routes/api/api.php` — attach the new middleware to the `medication-tracking` group; `app/Http/Kernel.php` if the middleware needs an alias.
- No change to create-time / edit-time dose materialization, to `GenerateScheduledDosesJob`, or to the dose read endpoints. Existing materialized doses remain valid — no data migration. The DB write pressure of the monthly full-table sweep is replaced by throttled, demand-driven per-patient top-ups spread across real traffic.
