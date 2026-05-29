## Why

Today the scheduled-dose horizon is materialized only at schedule **create** and **edit** time: rows are synchronously inserted for the first 14 days, and `GenerateScheduledDosesJob` fills out to `min(ends_on, starts_on + 365 days)`. After that, the horizon never advances — a year-long schedule created in January 2026 will simply run out of materialized doses in January 2027, and a year+ open-ended schedule will silently stop surfacing in the patient's daily and "next dose" views well before its actual end. We need a periodic job that **keeps the rolling year of pre-materialized doses topped up** for every live schedule, without re-running on schedules that don't need it.

## What Changes

- Add a new console command `medication-tracking:regenerate-doses` that fans out per-schedule work to the queue once a month.
- The command **reuses** the existing `App\Jobs\MedicationTracking\GenerateScheduledDosesJob` — no new job class. For each eligible schedule the command computes the per-schedule window as `localWindowStart = max(starts_on_local, last_generated_dose_local_date + 1 day)` and `maxDays = (today + ASYNC_HORIZON_DAYS) - localWindowStart`, then dispatches the existing job with `(scheduleId, localWindowStartIso, maxDays)`. Resuming from the day AFTER the last generated dose keeps each per-schedule run's work proportional to the actual gap (typically ~30 days) instead of re-evaluating a full year of mostly-already-materialized doses. The job's existing defensive behavior (null/`as_needed`/empty-window no-ops via the generator + window resolver) covers cron race conditions for free.
- The command's eligibility filter is a single SQL statement: `WHERE EXISTS (SELECT 1 FROM orders WHERE patient_id = mts.patient_id AND status IN OrderStatus::active())` as a semi-join (short-circuits per patient, no dedup phase), plus a correlated subquery returning `MAX(scheduled_at)` per schedule so the per-schedule window math needs no extra round-trip. Only schedules whose patient has at least one active order get a dispatch (custom and order-linked schedules both go through the gate uniformly). Schedules of churned patients (no active orders) are skipped to avoid unnecessary queue work at scale.
- Wire the command into `App\Console\Kernel::schedule()` with a monthly cadence (1st day of month, 02:00 UTC), `onOneServer()` + `withoutOverlapping()`.
- Extend the `medication-tracking-scheduler` spec with a new requirement: **"Periodic top-up of the scheduled-dose horizon"**, covering cadence, the eligibility filter (including the patient-active gate), idempotency, and the rolling-window semantics.
- No DB migrations: we reuse `medication_tracking_scheduled_doses` and its partial unique index `(intake_id, scheduled_at) WHERE type = 'scheduled'`, which already makes re-runs harmless. No new config keys either — we reuse `MedicationTrackingConfig::ASYNC_HORIZON_DAYS` (365) as the rolling horizon.

## Capabilities

### New Capabilities

(none — this is a behavioural extension of the existing scheduler capability, not a new surface)

### Modified Capabilities

- `medication-tracking-scheduler`: add a new "Periodic top-up of the scheduled-dose horizon" requirement so the scheduled-dose materialization stays current after create/edit, alongside the existing sync/async on-write horizons.

## Impact

- **New files**:
  - `app/Console/Commands/MedicationTracking/RegenerateScheduledDosesCommand.php`
  - `tests/Feature/Console/Commands/MedicationTracking/RegenerateScheduledDosesCommandTest.php`
  - `tests/Feature/Console/Schedule/RegenerateScheduledDosesScheduleTest.php`
- **Modified files**:
  - `app/Console/Kernel.php` — register the monthly schedule.
- **Affected systems**: the default queue (already used by `GenerateScheduledDosesJob`); Horizon supervisors that watch that queue.
- **No API changes**, no FE-visible changes, no migrations, no new config env vars (only adds an existing-queue load).
- **Safe by design**: re-running the job is idempotent thanks to the partial unique index; running `php artisan medication-tracking:regenerate-doses` manually performs the same full-batch run as the cron and is the recommended catch-up tool.
