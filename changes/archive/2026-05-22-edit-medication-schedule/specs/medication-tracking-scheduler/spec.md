## MODIFIED Requirements

### Requirement: Materialize scheduled doses on create with a split sync/async horizon

`ScheduleCreator` SHALL persist the schedule, intakes, and the **sync window** of `scheduled`-type doses inside one `DatabaseManager::transaction`. The sync window is `[starts_on, starts_on + medication_tracking.sync_horizon_days)` clipped to `ends_on` if set (default 14 days).

After the transaction commits, `ScheduleCreator` SHALL dispatch `GenerateScheduledDosesJob` to materialize the **async tail** out to `min(ends_on, starts_on + medication_tracking.async_horizon_days)` (default 365 days). The dispatch MUST use Laravel's `afterCommit()` so a transaction rollback never leaves a phantom job in the queue.

`AsNeededScheduledDoseStrategy` SHALL still yield zero rows. For an `as_needed` schedule, the create flow SHALL skip both the sync `generate()` call and the async job dispatch.

#### Scenario: Patient creates a daily schedule
- **WHEN** the patient POSTs a valid `every_day` schedule with 2 intakes and the default horizon config (sync=14, async=365)
- **THEN** the system SHALL respond `201` and exactly `2 × 14 = 28` `scheduled`-type dose rows SHALL be persisted in `medication_tracking_scheduled_doses` by the time the response returns
- **AND** exactly one `GenerateScheduledDosesJob` for the new schedule SHALL be on the queue

#### Scenario: Async tail completes
- **WHEN** the queue worker processes the job for the schedule above
- **THEN** doses SHALL be inserted from `starts_on + sync_horizon_days` up to `min(ends_on, starts_on + async_horizon_days)`, in chunks of 1 000 via `insertOrIgnore`
- **AND** running the job a second time against the same schedule SHALL be a no-op (the partial unique index `(intake_id, scheduled_at) WHERE type = 'scheduled'` silently drops colliding rows)

#### Scenario: Schedule is soft-deleted before the job runs
- **WHEN** the queue worker picks up the job after the schedule's `deleted_at` is set
- **THEN** the job SHALL exit cleanly without inserting any rows; the job SHALL NOT be marked as failed

#### Scenario: Dose generation fails inside the sync window
- **WHEN** the synchronous portion throws inside the create transaction
- **THEN** the transaction SHALL roll back; no schedule, intake, or dose row SHALL be visible; **no** `GenerateScheduledDosesJob` SHALL be queued

#### Scenario: `as_needed` schedule
- **WHEN** the patient POSTs an `as_needed` schedule
- **THEN** zero `scheduled`-type dose rows SHALL be inserted
- **AND** no `GenerateScheduledDosesJob` SHALL be dispatched

## ADDED Requirements

### Requirement: Edit a medication schedule

The system SHALL expose `PUT /api/v1/patient/medication-tracking/schedules/{schedule}` so the owning patient can update an existing schedule. Authorization MUST go through the `can:manage,schedule` policy. The implicit route binding MUST NOT resolve trashed rows.

**Editable fields:**
- `ends_on` (nullable; if set, MUST be `>= today` in the schedule's timezone),
- `schedule_type` (custom schedules only — order-linked schedules keep the product-mandated type),
- `schedule_rule` (the rule shape MUST match the resulting `schedule_type`),
- intakes (full replacement — the new intake list replaces the old row-set).

**Immutable fields** — `name`, `starts_on`, `timezone`, `product_id`, `order_id`. The request body MAY include them but the server SHALL ignore them: the FormRequest doesn't validate them, the DTO doesn't carry them, and the persistence layer never reads them. End state: their values on the row are unchanged after the edit.

For order-linked schedules the existing `ProductConstraintValidator` rules still apply: intake count, intake `unit`, `PlannedQuantityForProductType` bounds, INJECTION `days_of_week.length == 1`, and the locked `schedule_type`.

The edit MUST run inside one `DatabaseManager::transaction`, in this order:
1. Compute `now`.
2. Hard-delete future, unlogged `scheduled`-type doses (`scheduled_at >= now AND type = 'scheduled' AND tracking_log_id IS NULL`).
3. Update the schedule row (`ends_on`, `schedule_type`, `schedule_rule`).
4. Replace the intake rows.
5. Materialize the new sync window via `ScheduledDoseGenerator::generate($schedule, $now, $now->addDays(SYNC_HORIZON_DAYS))`.

After commit, the edit SHALL dispatch `GenerateScheduledDosesJob` for the async tail (the job is constructed with `from = now` and will generate `[from + SYNC, from + ASYNC)`).

History preservation, by design (mirrors delete):
- Past `scheduled`-type doses (`scheduled_at < now`) are unchanged.
- `scheduled`-type doses with `tracking_log_id IS NOT NULL` are unchanged, regardless of `scheduled_at` (logged history is immutable, even if scheduled in the future).
- `off_schedule` doses are unchanged.
- `tracking_logs` referencing the schedule are unchanged.

The response SHALL be `200` with the updated `ScheduleResource` (intakes + product eager-loaded).

#### Scenario: Patient changes the dose on a custom every-day schedule
- **WHEN** the owner PUTs a custom every-day schedule with the same `schedule_type` and a new `quantity` on its single intake
- **THEN** the system SHALL respond `200`
- **AND** every past `scheduled`-type dose SHALL keep its old `planned_quantity`
- **AND** every `scheduled`-type dose with `scheduled_at >= now AND tracking_log_id IS NULL` SHALL be removed and replaced with new rows at the new quantity inside the sync window
- **AND** any `scheduled`-type dose with `tracking_log_id IS NOT NULL` SHALL be untouched (logged history is immutable)

#### Scenario: Patient changes `schedule_type` on a custom schedule
- **WHEN** the owner PUTs a custom schedule with a new `schedule_type` and a compatible `schedule_rule`/intakes (e.g. `every_day` → `weekly` with `days_of_week`)
- **THEN** the system SHALL respond `200`, the new `schedule_type` SHALL be persisted, and the new strategy SHALL drive the sync window + async tail generation
- **AND** the previous strategy's future rows SHALL be gone (step 2 of the transaction)

#### Scenario: Patient tries to change `schedule_type` on an order-linked schedule
- **WHEN** the owner PUTs an order-linked schedule with a `schedule_type` other than the product's mandated type
- **THEN** the system SHALL respond `422` with a `ProductConstraintException` message; the schedule, intakes, and existing doses SHALL be unchanged

#### Scenario: Patient sets `ends_on` to a past date
- **WHEN** the request includes `ends_on` < today in the schedule's timezone
- **THEN** the system SHALL respond `422` with a validation error on `ends_on`

#### Scenario: Edit narrows the window past existing future doses
- **WHEN** the schedule had doses already materialized out to `+360 days` and the patient PUTs `ends_on = now + 30 days`
- **THEN** step 2 of the transaction SHALL hard-delete every `scheduled`-type future-unlogged dose at `scheduled_at >= now` (which includes everything past `ends_on` too)
- **AND** the async tail job SHALL only generate up to `ends_on`, not the year horizon

#### Scenario: Patient PUTs an immutable field
- **WHEN** the request body includes any of `name`, `starts_on`, `timezone`, `product_id`, `order_id`
- **THEN** the system SHALL respond `200` with the rest of the edit applied
- **AND** the values of those fields on the schedule row SHALL be unchanged — they are silently ignored, never read

#### Scenario: Different patient tries to edit
- **WHEN** an authenticated patient PUTs a schedule they do not own
- **THEN** `can:manage,schedule` SHALL return `403`

#### Scenario: Edit on a trashed schedule
- **WHEN** the route's `{schedule}` resolves to a soft-deleted row
- **THEN** the implicit binding SHALL NOT resolve it and the response SHALL be `404`

#### Scenario: Edit transaction fails mid-flight
- **WHEN** any step inside the edit transaction throws
- **THEN** the transaction SHALL roll back; the schedule, intakes, and existing doses SHALL be unchanged
- **AND** **no** `GenerateScheduledDosesJob` SHALL be queued (dispatch happens after commit)

#### Scenario: Async tail completes after edit
- **WHEN** the queue worker processes the post-edit job
- **THEN** dose rows SHALL be materialized from `now + sync_horizon_days` to `min(ends_on, now + async_horizon_days)`, idempotently against the partial unique index

---

### Requirement: Asynchronous dose tail via queued job

The system SHALL define `App\Jobs\MedicationTracking\GenerateScheduledDosesJob`, a queued job dispatched by both `ScheduleCreator` (after the create transaction commits) and `ScheduleEditor` (after the edit transaction commits). The job MUST:
- accept the schedule id (primitive) and a `from` date (ISO-8601 string);
- re-fetch the schedule at handle time with `withTrashed()`;
- exit cleanly if the schedule is `null`, `trashed()`, or `schedule_type === AS_NEEDED`;
- otherwise call `ScheduledDoseGenerator::generate($schedule, $from->addDays(SYNC_HORIZON_DAYS), $from->addDays(ASYNC_HORIZON_DAYS))` — the generator clips at `ends_on` internally.

The job MUST be idempotent and MUST NOT hold a database transaction (each chunked `insertOrIgnore` is independent and the partial unique index makes re-runs harmless).

#### Scenario: Job runs for a live schedule
- **WHEN** the queue worker handles the job for a non-deleted, non-`as_needed` schedule
- **THEN** dose rows SHALL be inserted from `from + SYNC_HORIZON_DAYS` up to `min(ends_on, from + ASYNC_HORIZON_DAYS)`, in 1 000-row chunks via `insertOrIgnore`

#### Scenario: Job runs for a soft-deleted schedule
- **WHEN** the queue worker handles the job for a schedule whose `deleted_at` is set
- **THEN** the job SHALL exit cleanly without inserting any rows; the failure handler SHALL NOT mark it as failed

#### Scenario: Job runs for an `as_needed` schedule
- **WHEN** the job is dispatched against an `as_needed` schedule (defensively — `ScheduleCreator` / `ScheduleEditor` already skip the dispatch)
- **THEN** the job SHALL exit cleanly without inserting any rows

#### Scenario: Two jobs for the same schedule race
- **WHEN** create and a follow-on backfill (future change) both dispatch a job for the same schedule and both run before the other completes
- **THEN** the partial unique index SHALL silently drop the collisions on the second run; no duplicate dose rows SHALL exist
