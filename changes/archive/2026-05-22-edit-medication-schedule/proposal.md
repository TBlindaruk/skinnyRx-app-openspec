## Why

[SRD-1169](https://skinnyrx.atlassian.net/browse/SRD-1169) — slice 2 of "Manage and delete medications". The delete slice is already merged ([`openspec/changes/archive/...`/`...-delete-medication-schedule/`](../../specs/medication-tracking-scheduler/spec.md)); now we add **edit**.

The edit endpoint must follow the same history-preservation contract as delete: **past doses and logged doses are immutable, future-unlogged doses are regenerated from the edit moment forward**. The patient changes dose, schedule pattern, or end date; the daily view from yesterday backwards keeps looking exactly the same, and the future starts being driven by the new state.

Doing this on top of the current dose generator is painful — every edit would synchronously materialize up to 730 days of dose rows. We use this slice as the moment to refactor generation: **14 days synchronously inside the create/edit transaction, the rest up to 1 year asynchronously via a queued job**. Both create and edit funnel through the same generator API and the same job; the patient gets a fast response, and the queue worker materializes the long tail.

## What Changes

### New endpoint

- `PUT /api/v1/patient/medication-tracking/schedules/{schedule}` — authorized via `can:manage,schedule`.

**Editable:**
- `ends_on` (nullable, `>= today` in the schedule's timezone),
- `schedule_type` (custom schedules only — order-linked schedules keep the product-mandated type),
- `schedule_rule` (must match the resulting `schedule_type`),
- intakes (full replacement — the new list replaces the old row-set).

**Not editable** (request body MUST reject them with `422`):
- `name`, `starts_on`, `timezone`, `product_id`, `order_id`.

**Side effects, in one DB transaction:**
1. compute `now`;
2. **hard-delete** future, unlogged `scheduled`-type doses (`scheduled_at >= now AND type = 'scheduled' AND tracking_log_id IS NULL`);
3. update the schedule row (`ends_on`, `schedule_type`, `schedule_rule`);
4. replace the intake set;
5. materialize the new **sync window** (`[now, now + sync_horizon_days)`) via the generator.

**After commit:** dispatch `GenerateScheduledDosesJob` to materialize the **async tail** (`[now + sync_horizon_days, min(ends_on, now + async_horizon_days))`).

Past doses keep their `intake_id` even when the original intake row was removed by the replacement (Decision 3 in design.md).

### Modified dose generation — 14 days sync, 365 days async

The existing **"Materialize scheduled doses synchronously on create"** requirement is replaced by a split-horizon model:

- `ScheduledDoseGenerator::generate(...)` becomes a primitive: it takes `windowStart` and `windowEnd` (exclusive) and fills `scheduled`-type dose rows inside that window. It clips at `ends_on` internally. It knows nothing about sync/async horizons — those are caller concerns.
- `ScheduleCreator` materializes the sync window inline (`[starts_on, starts_on + SYNC_HORIZON_DAYS)`), then dispatches `GenerateScheduledDosesJob` (after commit) for the async tail.
- `ScheduleEditor` does the same with `from = now`.
- The new job re-fetches the schedule by id, short-circuits if it's trashed or `as_needed`, then calls `generate(...)` with the async tail window (`[from + SYNC, from + ASYNC)`). Idempotent — the partial unique index `(intake_id, scheduled_at) WHERE type = 'scheduled'` silently drops collisions on re-runs.

### Configuration → constants

The whole config layer for medication-tracking knobs is removed. These values are domain decisions, not environment-specific, and the `env → config → provider → primitive` 4-layer pipeline was cargo-cult — env defaults were identical on stage and prod, no one ever tuned them per environment.

Replaced by a single class `App\Services\MedicationTracking\MedicationTrackingConfig` with `public const` values:

```php
final class MedicationTrackingConfig
{
    public const SYNC_HORIZON_DAYS         = 14;
    public const ASYNC_HORIZON_DAYS        = 365;
    public const SCHEDULE_LIMIT_PER_PATIENT = 50;
}
```

The provider reads the constants instead of `$config->get(...)`, asserts `ASYNC_HORIZON_DAYS > SYNC_HORIZON_DAYS` (a copy-paste guard, free at boot), and passes primitives into service constructors as before. Service signatures are unchanged.

Removed alongside:
- `config/medication_tracking.php`
- `MEDICATION_TRACKING_SYNC_HORIZON_DAYS`, `MEDICATION_TRACKING_ASYNC_HORIZON_DAYS`, `MEDICATION_TRACKING_SCHEDULE_LIMIT_PER_PATIENT`, and the old `MEDICATION_TRACKING_DOSE_GENERATION_HORIZON_DAYS` from `.env.example` and `.env.testing`.

## Capabilities

### New Capabilities

_None._

### Modified Capabilities

- `medication-tracking-scheduler`:
  - **ADDED**: "Edit a medication schedule" requirement.
  - **ADDED**: "Asynchronous dose tail via queued job" requirement.
  - **MODIFIED**: the existing "Materialize scheduled doses synchronously on create" requirement is replaced by a split sync/async version.

## Impact

### Code

- `app/Http/Controllers/Api/Patient/MedicationTracking/ScheduleController.php` — add `update(...)` method.
- `app/Http/Requests/Api/Patient/MedicationTracking/Schedule/UpdateRequest.php` — new FormRequest with `prohibited` rule for immutable fields.
- `app/DTO/MedicationTracking/Schedule/UpdateScheduleDto.php` — new.
- `app/Services/MedicationTracking/Schedule/ScheduleEditor.php` — new; orchestrates the edit transaction.
- `app/Services/MedicationTracking/Schedule/Update/UpdateScheduleValidator.php` — new; sibling of `CreateScheduleValidator`, reuses `ProductConstraintValidator` for order-linked schedules.
- `app/Services/MedicationTracking/Schedule/ScheduleCreator.php` — split into sync-window + after-commit job dispatch.
- `app/Services/MedicationTracking/ScheduledDose/ScheduledDoseGenerator.php` — signature changes to `generate($schedule, CarbonImmutable $windowStart, CarbonImmutable $windowEnd)`; internally clips at `ends_on`. No more horizon constructor params.
- `app/Services/MedicationTracking/ScheduledDose/Strategy/*` — each strategy adapts to the `windowStart / windowEnd` window contract (exclusive end).
- `app/Jobs/MedicationTracking/GenerateScheduledDosesJob.php` — new.
- `app/Services/MedicationTracking/MedicationTrackingConfig.php` — new; holds the three `public const` values.
- `app/Providers/MedicationTrackingServiceProvider.php` — read constants from `MedicationTrackingConfig`, assert `ASYNC > SYNC`, pass primitives to services.
- `config/medication_tracking.php` — **deleted**.
- `.env.example` + `.env.testing` — drop all `MEDICATION_TRACKING_*` keys.
- `routes/api/api.php` — add `PUT /` inside the existing `can:manage,schedule` group.

### API

- `PUT /api/v1/patient/medication-tracking/schedules/{schedule}` — new, `200` on success.
- `POST /api/v1/patient/medication-tracking/schedules` — wire-compatible, but the response returns after the sync window is materialized; the async tail is queued.

### Operational

- **No new env vars.** Old `MEDICATION_TRACKING_*` keys are removed from stage/prod's env files — done on deploy.
- New queue load: one `GenerateScheduledDosesJob` per non-as_needed create or edit. Modest — typical schedule materializes a few thousand rows per job; chunked inserts at 1 000/chunk.
- **Existing schedules already have ~730 days of doses materialized**. The new generation cap doesn't shrink them — it just means new edits regenerate up to 365 days forward, not 730. Over time, edited schedules redraw to the 365-day horizon; untouched schedules keep their 730-day tail until those rows naturally age out. No data migration.

## Out of Scope (deferred to separate proposals)

- **Monthly backfill cron** — the natural next slice: a scheduled command that tops up schedules whose horizon is closer than `now + async_horizon_days - buffer`. Without it, an old untouched schedule's horizon shortens as time passes. Deferred to a separate change, partly because it needs decisions on the buffer size + cadence that don't belong in this slice.
- **Untake** (undo a `taken` intake via Log History) — separate change.
- **Retroactive intake edits** (changing a past dose's planned quantity) — explicit non-goal; the forward-only invariant stays.
