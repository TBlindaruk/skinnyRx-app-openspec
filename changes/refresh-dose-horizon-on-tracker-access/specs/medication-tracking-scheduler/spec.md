## ADDED Requirements

### Requirement: On-access top-up of the scheduled-dose horizon

The system SHALL keep each patient's `scheduled`-type dose horizon current **lazily**, triggered by the patient interacting with the medication tracker, rather than by a periodic cron. Because materialized `scheduled`-type doses are read only by the patient's own tracker endpoints, the horizon only needs to be current when the patient accesses it.

The system SHALL define a `EnsureDoseHorizon` middleware registered on the `/api/v1/patient/medication-tracking` route group (the group that already runs under `auth.patients` alongside `TrackActivity`). On every request to that group the middleware SHALL:

- read the authenticated patient's `dose_horizon_refreshed_at` from the already-hydrated patient instance, performing NO additional query for it;
- if `dose_horizon_refreshed_at` is `NULL` OR strictly older than `now - MedicationTrackingConfig::HORIZON_REFRESH_THROTTLE_DAYS` (7 days), dispatch exactly one `RefreshPatientDoseHorizon(patientId, now)`;
- otherwise dispatch nothing;
- in all cases pass the request to the next handler WITHOUT waiting for any generation work (the middleware MUST NOT block the response).

The system SHALL define a `RefreshPatientDoseHorizon` that, for the given patient, iterates every schedule matching ALL of the following:

- not soft-deleted (the default `SoftDeletes` global scope MUST apply);
- `schedule_type != as_needed`;
- `ends_on IS NULL` OR `ends_on >= today`.

For each such schedule the job SHALL compute the per-schedule window identically to the previous cron logic:

- `localToday = clock->now()->setTimezone(schedule.timezone)->startOfDay()`;
- `endOfHorizon = localToday + MedicationTrackingConfig::ASYNC_HORIZON_DAYS`;
- `startsOnLocal = startOfDay(schedule.starts_on) in schedule.timezone`;
- `dayAfterLastDose` derived from `MAX(medication_tracking_scheduled_doses.scheduled_at) WHERE medication_tracking_schedule_id = schedule.id AND type = 'scheduled'`, parsed in the app's default timezone, converted to `schedule.timezone`, `startOfDay()`, then `addDay()`; or null when no `scheduled`-type dose row exists yet;
- `localWindowStart = max(startsOnLocal, dayAfterLastDose)`;
- if `localWindowStart >= endOfHorizon`, the job SHALL skip that schedule (no generation);
- otherwise the job SHALL extend the horizon for that schedule via the existing `ScheduledDoseGenerator` over `[localWindowStart, endOfHorizon]` (`maxDays = (int) localWindowStart->diffInDays(endOfHorizon)`).

After processing all of the patient's schedules the job SHALL set `patient.dose_horizon_refreshed_at = refreshedAt` (the timestamp passed in at dispatch).

The system SHALL NOT gate top-up by the patient's order status or `last_active_at`: accessing the tracker is itself the eligibility signal.

Generation SHALL remain idempotent: the partial unique index `(intake_id, scheduled_at) WHERE type = 'scheduled'` together with `insertOrIgnore` makes overlapping or repeated windows produce zero net new rows. The job MUST NOT modify or delete any existing dose row.

#### Scenario: First tracker access tops up the horizon

- **WHEN** a patient whose `dose_horizon_refreshed_at` is `NULL` sends any request to the `/api/v1/patient/medication-tracking` route group
- **THEN** the `EnsureDoseHorizon` middleware SHALL dispatch one `RefreshPatientDoseHorizon` for that patient
- **AND** the response SHALL NOT wait for the job
- **AND** when the job runs it SHALL extend each of the patient's eligible schedules up to `today + ASYNC_HORIZON_DAYS` and set `dose_horizon_refreshed_at` to the dispatch timestamp

#### Scenario: Repeat access within the throttle window does nothing

- **WHEN** a patient whose `dose_horizon_refreshed_at` is 2 days ago sends a request to the medication-tracking group
- **THEN** the middleware SHALL NOT dispatch a job (2 days < 7-day throttle)
- **AND** no new dose rows SHALL be generated as a result of this request

#### Scenario: Access after the throttle window re-tops-up

- **WHEN** a patient whose `dose_horizon_refreshed_at` is 8 days ago sends a request to the medication-tracking group
- **THEN** the middleware SHALL dispatch one `RefreshPatientDoseHorizon`
- **AND** the job SHALL extend any schedule whose materialized horizon is short and SHALL update `dose_horizon_refreshed_at` to the new dispatch timestamp

#### Scenario: Horizon already current — no net new rows

- **WHEN** the job runs for a patient whose schedules are all materialized to or past `today + ASYNC_HORIZON_DAYS`
- **THEN** every schedule's `localWindowStart` SHALL be `>= endOfHorizon` and be skipped, OR generation SHALL `insertOrIgnore` zero net new rows
- **AND** `dose_horizon_refreshed_at` SHALL still be updated to the dispatch timestamp (the throttle resets regardless of whether rows were added)

#### Scenario: Job skips `as_needed` and expired schedules

- **WHEN** the job iterates a patient owning an `as_needed` schedule and a schedule with `ends_on = yesterday`
- **THEN** neither SHALL receive generated doses (both excluded by the eligibility filter)

#### Scenario: Patient who never accesses the tracker is never topped up

- **WHEN** a patient owns eligible schedules but sends no request to the medication-tracking group
- **THEN** no `RefreshPatientDoseHorizon` SHALL be dispatched for them and their horizon SHALL NOT be extended
- **AND** this holds regardless of the patient's order status — nothing other than the patient's own tracker endpoints reads the materialized horizon

#### Scenario: Concurrent stale requests are safe

- **WHEN** two requests from the same patient with a stale `dose_horizon_refreshed_at` arrive close enough that both dispatch a job before either stamps the marker
- **THEN** both jobs SHALL run idempotently — the second produces zero net new rows via `insertOrIgnore` and the partial unique index
- **AND** the final `dose_horizon_refreshed_at` SHALL reflect a dispatch timestamp within the throttle window

## REMOVED Requirements

### Requirement: Periodic top-up of the scheduled-dose horizon

**Reason**: Replaced by the "On-access top-up of the scheduled-dose horizon" requirement. The monthly `medication-tracking:regenerate-doses` cron materialized doses for every schedule regardless of whether the patient ever opens the tracker, then needed an engagement heuristic to avoid churned patients. Since nothing other than the patient's own tracker endpoints reads materialized `scheduled`-type doses, generation is now demand-driven and the periodic full-table sweep — and its eligibility heuristic — are unnecessary.

**Migration**: Remove the `medication-tracking:regenerate-doses` artisan command and its `App\Console\Kernel::schedule()` registration. Horizon maintenance is fully covered by on-access top-up. No data migration is required: existing materialized dose rows remain valid, and the first tracker access per patient (with a `NULL` `dose_horizon_refreshed_at`) extends the horizon as needed.
