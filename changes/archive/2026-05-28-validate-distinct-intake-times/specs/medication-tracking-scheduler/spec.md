## MODIFIED Requirements

### Requirement: Create a custom medication schedule

The system SHALL allow an authenticated patient to create a medication schedule that is not tied to any order. The schedule MUST persist its intakes as a real `medication_tracking_schedule_intakes` relation and MUST pre-materialize scheduled-dose rows up to the configured horizon, all inside a single database transaction.

The supported `schedule_type` values are `every_day`, `weekly`, `every_few_days`, and `as_needed`. The `schedule_rule` payload SHALL carry only rule-specific metadata (e.g. `days_of_week` for `weekly`, `interval_days` for `every_few_days`); intakes SHALL be a top-level array on the request and on the response, never embedded inside `schedule_rule`.

For `every_day`, `weekly`, and `every_few_days` schedules, the `schedule_rule.intakes[*].time` values SHALL be distinct within a single request. Enforcement happens at the FormRequest layer via Laravel's built-in `distinct` rule on `schedule_rule.intakes.*.time`; the database partial unique index `(medication_tracking_schedule_id, time)` remains the authoritative race guard against concurrent writes but SHALL NOT be the patient-facing error path. `as_needed` is unaffected — that schedule type permits exactly one intake and prohibits `time`.

`localStartsOn` (the request's `starts_on` interpreted in the request's `timezone`) SHALL NOT be earlier than today in that timezone. Enforcement happens in `CreateScheduleValidator::assertStartsOnNotInPast(...)` as the FIRST step of `validate(...)`, mirroring `UpdateScheduleValidator::assertEndsOnNotInPast(...)` on the edit path. A violation SHALL throw `StartsOnInPastException` (extending `\DomainException`); the controller SHALL map it to `422 {"message": "The starts_on date cannot be before today in the schedule timezone."}`. The check MUST run before any DB query, so a past-`starts_on` request SHALL NOT incur the schedule-cap `count(*)` or the duplicate-order existence lookup. When `starts_on` is omitted on the `as_needed` path, `CreateRequest::getStartsOnLocal()` SHALL continue to default it to "today in the provided timezone" — the check is satisfied trivially.

#### Scenario: Patient creates an every-day custom schedule
- **WHEN** the patient POSTs to `/api/v1/patient/medication-tracking/schedules` with `schedule_type=every_day`, `starts_on`, a valid IANA `timezone`, and 1–16 intakes each with `time` (`H:i`), `quantity > 0`, and a `unit` from `DoseUnit`
- **THEN** the system SHALL respond `201 Created` with a `ScheduleResource` whose `product_id` and `order_id` are `null`, `image_path` is `null`, `schedule_rule` is an empty object `{}`, and `intakes` is the persisted list with assigned ids
- **AND** the system SHALL have materialized `every_day × intakes` scheduled-dose rows up to the configured horizon (`medication_tracking.dose_generation_horizon_days`, default 730)

#### Scenario: Patient creates a weekly custom schedule
- **WHEN** the patient POSTs with `schedule_type=weekly` and `schedule_rule.days_of_week` containing 1–7 distinct integers (1=Monday … 7=Sunday)
- **THEN** the system SHALL respond `201` with `schedule_rule = { "days_of_week": [...] }` echoed back
- **AND** the dose generator SHALL only materialize doses on the listed weekdays, up to the horizon

#### Scenario: Patient creates an every-few-days custom schedule
- **WHEN** the patient POSTs with `schedule_type=every_few_days` and `schedule_rule.interval_days >= 2`
- **THEN** the system SHALL respond `201` with `schedule_rule = { "interval_days": N }`
- **AND** the dose generator SHALL materialize doses every `interval_days` starting at `starts_on`, up to the horizon

#### Scenario: Patient creates an as-needed schedule
- **WHEN** the patient POSTs with `schedule_type=as_needed`, exactly one intake whose `time` is omitted, and (optionally) `starts_on`
- **THEN** the system SHALL respond `201` with an intake whose `time` is `null`
- **AND** the system SHALL materialize zero scheduled-dose rows (as-needed schedules only produce doses when the patient logs off-schedule)

#### Scenario: Patient sends an intake count outside the 1–16 range
- **WHEN** the patient POSTs with zero or more than 16 intakes for a non-`as_needed` schedule
- **THEN** the system SHALL respond `422 Unprocessable Entity` with a validation error
- **AND** no schedule, intake, or dose row SHALL be persisted

#### Scenario: Patient sends two intakes with the same `time` on a non-`as_needed` schedule
- **WHEN** the patient POSTs with `schedule_type` ∈ {`every_day`, `weekly`, `every_few_days`} and an `schedule_rule.intakes` array containing two entries whose `time` strings are equal (e.g. both `09:00`)
- **THEN** the system SHALL respond `422` with a validation error keyed on the duplicate index — `{ "schedule_rule.intakes.<index>.time": [...] }` — produced by the FormRequest's `distinct` rule
- **AND** no database transaction SHALL be opened and no schedule, intake, or dose row SHALL be persisted
- **AND** the response SHALL NOT include the raw `SQLSTATE[23505]` text or any database constraint name

#### Scenario: Patient sends an invalid timezone
- **WHEN** the patient POSTs with a `timezone` that is not a valid IANA zone
- **THEN** the system SHALL respond `422` with a validation error on `timezone`

#### Scenario: Patient sends `ends_on` earlier than `starts_on`
- **WHEN** the patient POSTs with `ends_on < starts_on`
- **THEN** the system SHALL respond `422` with a validation error on `ends_on`

#### Scenario: Patient sends `starts_on` earlier than today in the request timezone
- **WHEN** the patient POSTs with `starts_on` set to a calendar date earlier than `FactoryImmutable::now($timezone)->startOfDay()` in the request's `timezone`
- **THEN** `CreateScheduleValidator::assertStartsOnNotInPast(...)` SHALL throw `StartsOnInPastException` and the controller SHALL respond `422` with `{"message": "The starts_on date cannot be before today in the schedule timezone."}`
- **AND** the check SHALL fire before the schedule-cap `count(*)` query and before the duplicate-order existence query — no DB read SHALL be issued by the validator on this path
- **AND** no transaction SHALL be opened and no schedule, intake, or dose row SHALL be persisted

#### Scenario: `as_needed` request with an explicitly past `starts_on`
- **WHEN** the patient POSTs `schedule_type=as_needed` with `starts_on` set to a past calendar date in the request's `timezone`
- **THEN** the system SHALL respond `422` with the same `{"message": ...}` payload — the rule applies whenever `starts_on` is supplied, not only on non-`as_needed` schedules
- **AND** no schedule SHALL be persisted

#### Scenario: `as_needed` request that omits `starts_on`
- **WHEN** the patient POSTs `schedule_type=as_needed` without a `starts_on` field
- **THEN** `CreateRequest::getStartsOnLocal()` SHALL default `localStartsOn` to today's start in the request's `timezone`, the validator's past-check SHALL pass trivially, and the system SHALL respond `201`

#### Scenario: Patient sends `starts_on` equal to today in the request timezone
- **WHEN** the patient POSTs with `starts_on` equal to today's date in the request's `timezone`
- **THEN** the system SHALL respond `201` — the validator compares with `lessThan(...)`, so today is allowed (inclusive lower bound)

### Requirement: Edit a medication schedule

The system SHALL expose `PUT /api/v1/patient/medication-tracking/schedules/{schedule}` so the owning patient can update an existing schedule. Authorization MUST go through the `can:manage,schedule` policy. The implicit route binding MUST NOT resolve trashed rows.

**Editable fields:**
- `ends_on` (nullable; if set, MUST be `>= today` in the schedule's timezone),
- `schedule_type` (custom schedules only — order-linked schedules keep the product-mandated type),
- `schedule_rule` (the rule shape MUST match the resulting `schedule_type`),
- intakes (full replacement — the new intake list replaces the old row-set).

For `every_day`, `weekly`, and `every_few_days` schedule types, the replacement intake list MUST contain distinct `time` values. Enforcement happens at the FormRequest layer via the same `distinct` rule used on the create path; the database partial unique index `(medication_tracking_schedule_id, time)` remains the authoritative race guard but SHALL NOT be the patient-facing error path. `as_needed` is unaffected (single-intake, `time` prohibited).

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

#### Scenario: Patient PUTs two intakes with the same `time` on a non-`as_needed` schedule
- **WHEN** the owner PUTs a custom schedule with `schedule_type` ∈ {`every_day`, `weekly`, `every_few_days`} and a replacement `schedule_rule.intakes` array containing two entries whose `time` strings are equal (e.g. both `09:00`)
- **THEN** the system SHALL respond `422` with a validation error keyed on the duplicate index — `{ "schedule_rule.intakes.<index>.time": [...] }` — produced by the FormRequest's `distinct` rule
- **AND** no database transaction SHALL be opened; the existing schedule row, its intakes, and its future-unlogged doses SHALL be unchanged
- **AND** the response SHALL NOT include the raw `SQLSTATE[23505]` text or any database constraint name

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
