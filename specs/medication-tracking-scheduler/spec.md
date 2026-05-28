# medication-tracking-scheduler Specification

## Purpose
Patient-facing API for creating and viewing medication schedules (Rx-linked or custom), pre-materializing scheduled-dose rows across a configurable horizon, exposing a daily and "next" dose view, updating planned quantity per intake forward in time, and logging intake events (against a planned dose, or off-schedule against a schedule). Covers the seven endpoints under `/api/v1/patient/medication-tracking/{schedules,scheduled-doses,tracking-logs/by-*}`.
## Requirements
### Requirement: Create a custom medication schedule

The system SHALL allow an authenticated patient to create a medication schedule that is not tied to any order. The schedule MUST persist its intakes as a real `medication_tracking_schedule_intakes` relation and MUST pre-materialize scheduled-dose rows up to the configured horizon, all inside a single database transaction.

The supported `schedule_type` values are `every_day`, `weekly`, `every_few_days`, and `as_needed`. The `schedule_rule` payload SHALL carry only rule-specific metadata (e.g. `days_of_week` for `weekly`, `interval_days` for `every_few_days`); intakes SHALL be a top-level array on the request and on the response, never embedded inside `schedule_rule`.

For `every_day`, `weekly`, and `every_few_days` schedules, the `schedule_rule.intakes[*].time` values SHALL be distinct within a single request. Enforcement happens at the FormRequest layer via Laravel's built-in `distinct` rule on `schedule_rule.intakes.*.time`; the database partial unique index `(medication_tracking_schedule_id, time)` remains the authoritative race guard against concurrent writes but SHALL NOT be the patient-facing error path. `as_needed` is unaffected — that schedule type permits exactly one intake and prohibits `time`.

`localStartsOn` (the request's `starts_on` interpreted in the request's `timezone`) SHALL NOT be earlier than today in that timezone. Enforcement happens in `CreateScheduleValidator::assertStartsOnNotInPast(...)` as the FIRST step of `validate(...)`, mirroring `UpdateScheduleValidator::assertEndsOnNotInPast(...)` on the edit path. A violation SHALL throw `StartsOnInPastException` (extending `\DomainException`); the controller SHALL map it to `422 {"message": "The starts_on date cannot be before today in the schedule timezone."}`. The check MUST run before any DB query, so a past-`starts_on` request SHALL NOT incur the schedule-cap `count(*)` or the duplicate-order existence lookup. When `starts_on` is omitted on the `as_needed` path, `CreateRequest::getStartsOnLocal()` SHALL continue to default it to "today in the provided timezone" — the check is satisfied trivially.

The `timezone` field SHALL accept any IANA name in PHP's `DateTimeZone::ALL_WITH_BC` set — i.e. canonical zones (e.g. `Europe/Kyiv`, `America/New_York`) **and** the backward-compatibility aliases PHP recognizes (`Europe/Kiev`, `US/Eastern`, `Asia/Calcutta`, …). The stored and returned value MUST equal the value the patient submitted; the system makes no claim about a canonical form. Unknown identifiers (case-mismatched or not in `ALL_WITH_BC`) MUST be rejected with `422`.

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
- **WHEN** the patient POSTs with a `timezone` value not in `DateTimeZone::ALL_WITH_BC` (e.g. `Not/A_Zone`, or `europe/kiev` with the wrong case)
- **THEN** the system SHALL respond `422` with a validation error on `timezone`

#### Scenario: Patient sends a deprecated IANA alias
- **WHEN** the patient POSTs with `timezone=Europe/Kiev` (or any other backward-compatibility alias such as `US/Eastern`, `Asia/Calcutta`)
- **THEN** the system SHALL respond `201` with the schedule persisted using the same string the patient sent

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

---

### Requirement: Create an order-linked medication schedule

When the patient supplies an `order_id`, the system SHALL derive the product from the order (the API MUST NOT accept a `product_id` field), and SHALL reject any field that drifts from the prescribed product. The order MUST be active and owned by the patient.

For order-linked schedules the system SHALL enforce, in addition to the custom-schedule rules:
- `name` matches the product name,
- `schedule_type` is `every_day` for `PILLS`/`DROPS` and `weekly` for `INJECTION`,
- exactly one intake,
- the intake `unit` matches `DoseUnit::forProductType(...)` — which is `unit` for `INJECTION`, `mg` for `PILLS`, and `ml` for `DROPS`,
- `quantity` is within the product type's planned-quantity bounds (`PlannedQuantityForProductType`); for `PILLS` the quantity MUST be a whole number (entered as whole `mg`),
- for `INJECTION`, `schedule_rule.days_of_week` contains exactly one day.

#### Scenario: Patient creates a weekly injection schedule from an order
- **WHEN** the patient POSTs with `order_id` referencing an active injection order, `schedule_type=weekly`, one intake whose `unit=unit`, and `days_of_week=[3]`
- **THEN** the system SHALL respond `201` with `product_id` and `order_id` populated from the resolved order, and `image_path` resolved from the product
- **AND** the schedule row in `medication_tracking_schedules` SHALL carry both `product_id` (denormalized from the order) and `order_id`

#### Scenario: Patient creates an every-day pills schedule from an order
- **WHEN** the patient POSTs with `order_id` referencing an active pills order, `schedule_type=every_day`, and one intake whose `unit=mg` and a whole-number `quantity`
- **THEN** the system SHALL respond `201` with `product_id` and `order_id` populated from the resolved order
- **AND** the schedule row SHALL carry both `product_id` and `order_id`

#### Scenario: Patient sends an `order_id` that does not resolve to an active order they own
- **WHEN** the patient POSTs with an `order_id` that is not theirs, is cancelled, or does not exist
- **THEN** the system SHALL respond `422` with a validation error keyed on `order_id` (`"The selected order is invalid."`), raised by `ScheduleController::resolveOrder(...)`
- **AND** the system SHALL NOT open a database transaction

#### Scenario: Patient sends an order-locked field that drifts from the prescribed product
- **WHEN** the patient POSTs with an `order_id` and any of: a `name` that does not match the product name; a `schedule_type` that is not the product's mandated type; more than one intake; an intake `unit` that does not match `DoseUnit::forProductType(...)` (e.g. `unit=injection` for an injection order, or `unit=unit` for a pills order); an injection schedule with `days_of_week.length != 1`; or a `quantity` outside the product type's planned-quantity bounds (e.g. a fractional `quantity` for pills)
- **THEN** the system SHALL respond `422` with a `ProductConstraintException` message
- **AND** no schedule, intake, or dose row SHALL be persisted

#### Scenario: Patient sends `product_id` directly in the request body
- **WHEN** the patient POSTs a request body containing a `product_id` field
- **THEN** the system SHALL ignore the field — the product is always derived from the resolved order, never from the request

---

### Requirement: Reject duplicate schedules for the same order

The system SHALL allow at most one **active** schedule per `(patient_id, order_id)` pair, where "active" means the schedule row is not soft-deleted. Two different orders for the same product are allowed. Enforcement happens both in `CreateScheduleValidator` (pre-transaction lookup, scoped to non-trashed rows via Eloquent's default `SoftDeletes` scope) and at the database level via the partial unique index `(patient_id, order_id) WHERE order_id IS NOT NULL AND deleted_at IS NULL`. The DB index is the authoritative race guard.

#### Scenario: Patient already has an active schedule for the given order
- **WHEN** the patient POSTs a schedule with an `order_id` for which they already own a non-deleted schedule
- **THEN** the system SHALL respond `409 Conflict` with a `MedicationTrackingScheduleAlreadyExistsException` message

#### Scenario: Two concurrent creates race for the same order
- **WHEN** two requests with the same `(patient_id, order_id)` arrive concurrently and both pass the validator
- **THEN** the database partial unique index SHALL make one of them fail and the second response SHALL be `409`

#### Scenario: Patient creates schedules for the same product across two different orders
- **WHEN** the patient has two distinct active orders for the same product and POSTs a schedule for each
- **THEN** the system SHALL respond `201` for both — the `(patient_id, order_id)` uniqueness is per-order, not per-product

#### Scenario: Patient re-creates a schedule for an order whose previous schedule was deleted
- **WHEN** the patient previously had a schedule for `order_id = X`, soft-deleted it, and now POSTs a new schedule for the same `order_id = X`
- **THEN** the system SHALL respond `201` — the partial unique index excludes the trashed row via `deleted_at IS NULL`, and the validator's existence check uses the default `SoftDeletes` scope which also excludes it
- **AND** both schedule rows (the trashed one and the new one) SHALL coexist; the trashed one keeps its `order_id` for historical reporting

---

### Requirement: Enforce the per-patient schedule cap

The system SHALL reject schedule creation when the patient already owns `medication_tracking.schedule_limit_per_patient` schedules (default 50). The cap MUST be enforced in `CreateScheduleValidator` before any transaction opens.

#### Scenario: Patient is at the cap
- **WHEN** the patient owns exactly `schedule_limit_per_patient` schedules and POSTs another
- **THEN** the system SHALL respond `422` with a `MedicationTrackingScheduleLimitReachedException` message
- **AND** no transaction SHALL be opened and no rows SHALL be inserted

---

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

---

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

---

### Requirement: List the patient's medication schedules

The system SHALL expose `GET /api/v1/patient/medication-tracking/schedules` returning a paginated `CollectionResource` of the patient's schedules. Order-linked schedules (`product_id IS NOT NULL`) MUST come first, then custom schedules, both ordered by `id DESC`. The `intakes` and `product` relations MUST be eager-loaded. An optional `id[]` query parameter SHALL restrict the result to the given ids.

#### Scenario: Patient lists schedules with no filter
- **WHEN** the patient GETs the endpoint with `page` and `per_page` query params
- **THEN** the response SHALL contain order-linked schedules first, then custom schedules, both by `id DESC`
- **AND** every item SHALL include its `intakes` array and `image_path` resolved from the linked product (or `null` for custom)

#### Scenario: Patient passes `id[]` filter
- **WHEN** the patient GETs with `?id[]=88&id[]=89`
- **THEN** the response SHALL only include schedules whose id is in the list AND whose `patient_id` matches the authenticated patient — `id[]` MUST NOT bypass the patient scope

#### Scenario: Patient passes empty `id[]`
- **WHEN** the `id[]` parameter is absent or empty
- **THEN** the filter SHALL be a no-op and the full patient-scoped list SHALL be returned

---

### Requirement: Show scheduled doses for a single local day

The system SHALL expose `GET /api/v1/patient/medication-tracking/scheduled-doses` returning a non-paginated array of dose rows for the requested local day. The query MUST return both (a) scheduled-but-not-logged doses whose `scheduled_at` falls in the day window and (b) every dose (scheduled or off-schedule) whose attached `tracking_log.logged_at` falls in the day window. The result MUST be ordered by `COALESCE(logged_at, scheduled_at)`.

The day window MUST be computed by converting `date` + `timezone` into the storage TZ and using `[dayStart, dayStart + 1 day)`. Each dose item MUST expose `planned_timezone` (the schedule's TZ) and `requested_timezone` (the TZ from the request) so the FE can detect mismatches.

The `timezone` query parameter SHALL accept any IANA name in `DateTimeZone::ALL_WITH_BC` (canonical or backward-compatibility alias). Day-window math MUST produce identical results regardless of which form the patient sent (PHP's `DateTimeZone` treats both as the same zone). Unknown identifiers MUST be rejected with `422`.

#### Scenario: Patient asks for today's doses
- **WHEN** the patient GETs `?date=2026-05-15&timezone=America/New_York`
- **THEN** the response SHALL be a JSON array (no envelope) ordered chronologically by `COALESCE(logged_at, scheduled_at)`
- **AND** each item SHALL include `type` (`scheduled` | `off_schedule`), `medication_tracking_schedule_id`, `intake_id` (nullable for off-schedule), `planned_time`/`planned_timezone`, `requested_time`/`requested_timezone`, `planned_quantity`, `planned_unit`, and `tracking_log` (nullable)

#### Scenario: Day window spans two UTC dates
- **WHEN** the patient requests a day in a TZ whose midnight crosses a UTC boundary (e.g. `America/New_York`)
- **THEN** the query SHALL still return only doses inside that local day, not the UTC day

#### Scenario: Day contains an off-schedule log
- **WHEN** the patient logged an off-schedule dose during the requested day
- **THEN** the response SHALL include the corresponding `off_schedule` dose row with its nested `tracking_log` populated, regardless of its `scheduled_at`

#### Scenario: Patient queries with a deprecated IANA alias
- **WHEN** the patient GETs `?date=2026-05-15&timezone=Europe/Kiev`
- **THEN** the response SHALL be identical to the same query with `timezone=Europe/Kyiv` (same items, same order)

#### Scenario: Patient queries with an unknown timezone
- **WHEN** the patient GETs with a `timezone` value not in `DateTimeZone::ALL_WITH_BC`
- **THEN** the system SHALL respond `422` with a validation error on `timezone`

---

### Requirement: Return the next upcoming scheduled dose

The system SHALL expose `GET /api/v1/patient/medication-tracking/scheduled-doses/next` returning the next `type = 'scheduled'` dose whose `scheduled_at` is at or after the start of the requested local day. If none exists, the endpoint SHALL return `204 No Content` with an empty body.

The `timezone` query parameter SHALL accept any IANA name in `DateTimeZone::ALL_WITH_BC`. Unknown identifiers MUST be rejected with `422`.

#### Scenario: Patient has an upcoming dose
- **WHEN** the patient GETs `?date=2026-05-15&timezone=America/New_York` and at least one scheduled, future dose exists
- **THEN** the response SHALL be `200` with a single `ScheduledDoseResource` (same shape as items in the daily-doses endpoint)

#### Scenario: Patient has no upcoming doses
- **WHEN** the patient has no `scheduled`-type doses at or after the requested day start
- **THEN** the response SHALL be `204 No Content` with an empty body

#### Scenario: `next` skips off-schedule rows
- **WHEN** the patient's only future dose is an `off_schedule` row
- **THEN** the endpoint SHALL ignore it and respond `204` — `next` SHALL only consider `type = 'scheduled'`

#### Scenario: Patient asks for the next dose with a deprecated IANA alias
- **WHEN** the patient GETs `?date=2026-05-15&timezone=Europe/Kiev`
- **THEN** the response SHALL be identical to the same query with `timezone=Europe/Kyiv`

#### Scenario: Patient asks for the next dose with an unknown timezone
- **WHEN** the patient GETs with a `timezone` value not in `DateTimeZone::ALL_WITH_BC`
- **THEN** the system SHALL respond `422` with a validation error on `timezone`

---

### Requirement: Update planned quantity on an intake forward-only

The system SHALL expose `PATCH /api/v1/patient/medication-tracking/schedules/{schedule}/intakes/{intake}/planned-quantity`. The change MUST propagate forward only: the intake row's `quantity` is overwritten, and every `type = 'scheduled'` dose for the intake at or after the selected dose's `scheduled_at` with `tracking_log_id IS NULL` is updated. Logged or past doses SHALL NOT be modified. The route MUST be protected by the `manage` policy and `scopeBindings()` so the `{intake}` must belong to the path `{schedule}`. The `PlannedQuantityForProductType` rule from the create flow MUST be reused on this endpoint so per-product-type quantity bounds cannot drift between create and edit.

#### Scenario: Patient bumps quantity for future doses
- **WHEN** the patient PATCHes with `{ scheduled_dose_id: 7, planned_quantity: 2.5 }` where dose 7 is unlogged and in the future
- **THEN** the system SHALL update the intake's `quantity` to `2.5`, and every later `scheduled`-type dose of the same intake with `tracking_log_id IS NULL` SHALL also be updated
- **AND** any earlier doses, and any doses that already have a `tracking_log_id`, SHALL remain at their previous `planned_quantity`

#### Scenario: Patient sends a `scheduled_dose_id` that does not belong to the path `{intake}`
- **WHEN** the patient PATCHes with a `scheduled_dose_id` whose `intake_id` differs from the route's `{intake}`
- **THEN** the system SHALL respond `422` with a validation error — the dose-to-intake relationship is checked before the action runs

#### Scenario: Path `{intake}` does not belong to path `{schedule}`
- **WHEN** the patient sends an `{intake}` from a different schedule
- **THEN** Laravel's `scopeBindings()` SHALL return `404` before the request reaches the controller or validator

#### Scenario: Different patient tries to edit
- **WHEN** an authenticated patient PATCHes a schedule they do not own
- **THEN** the `can:manage,schedule` middleware SHALL return `403`

#### Scenario: Quantity violates product-type bounds for an order-linked schedule
- **WHEN** the patient PATCHes an order-linked schedule with a `planned_quantity` outside the product type's bounds
- **THEN** the system SHALL respond `422` — the same `PlannedQuantityForProductType` rule used at create time fires

---

### Requirement: Log a tracking event against a scheduled dose

The system SHALL expose `POST /api/v1/patient/medication-tracking/tracking-logs/by-dose`. Given a `medication_dose_id` that resolves to a `scheduled`-type dose on a schedule owned by the patient, the endpoint SHALL atomically insert a `tracking_logs` row AND set the dose's `tracking_log_id` to point at the new log. The dose row SHALL NOT be duplicated and SHALL NOT be moved.

`logged_at` is optional; when provided it MUST be a strict ISO 8601 string with offset (`Y-m-d\TH:i:sP`) and SHALL default to "now" otherwise. The action SHALL throw `ScheduledDoseTooFarInFutureException` when `dose.scheduled_at > now + 1 day`, which the controller MUST map to `422`.

`pain_level` (integer, `between:0,10`) SHALL be required if and only if the resolved schedule's product is of type `INJECTION`. For schedules whose product is `PILLS` or `DROPS`, or for custom schedules whose `product_id IS NULL`, `pain_level` MUST be absent (or explicitly `null`); supplying a value SHALL cause the action to throw `PainLevelDoesNotMatchProductException` ("The pain level field must not be present for this medication."), which the controller MUST map to `422` with a `{"message": ...}` body. Omitting `pain_level` for an injection schedule SHALL throw the same exception with the message "The pain level field is required for this medication." When the rule allows omission, the persisted `tracking_logs.pain_level` SHALL be `NULL`.

#### Scenario: Patient logs "taken" against a planned slot for an injection
- **WHEN** the patient POSTs with a valid `medication_dose_id` resolving to an injection schedule, `medication_action=taken`, `dose_value`, `pain_level`, optional `note`
- **THEN** the system SHALL respond `201` with a `TrackingLogResource` whose `medication_dose_id` echoes the dose
- **AND** the dose row's `tracking_log_id` SHALL now point at the new log; the dose's `scheduled_at` is unchanged
- **AND** the persisted log's `pain_level` SHALL equal the submitted value

#### Scenario: Patient logs against a PILLS or DROPS scheduled dose without pain_level
- **WHEN** the patient POSTs with a valid `medication_dose_id` resolving to a `PILLS` or `DROPS` schedule and omits `pain_level` (or sends it as `null`)
- **THEN** the system SHALL respond `201` with a `TrackingLogResource`
- **AND** the persisted log's `pain_level` SHALL be `NULL`

#### Scenario: Patient logs against a custom (non-Rx) scheduled dose without pain_level
- **WHEN** the patient POSTs with a valid `medication_dose_id` resolving to a schedule whose `product_id IS NULL` and omits `pain_level`
- **THEN** the system SHALL respond `201` with a `TrackingLogResource`
- **AND** the persisted log's `pain_level` SHALL be `NULL`

#### Scenario: Patient sends pain_level for a non-injection schedule
- **WHEN** the patient POSTs with `pain_level` present on a `PILLS`, `DROPS`, or custom (`product_id IS NULL`) schedule
- **THEN** the system SHALL respond `422` with `{"message": "The pain level field must not be present for this medication."}`
- **AND** no `tracking_logs` row SHALL be persisted; the dose's `tracking_log_id` SHALL remain unchanged

#### Scenario: Patient omits pain_level for an injection schedule
- **WHEN** the patient POSTs against an injection schedule without `pain_level`
- **THEN** the system SHALL respond `422` with `{"message": "The pain level field is required for this medication."}`

#### Scenario: Patient omits `dose_unit`
- **WHEN** the patient POSTs without `dose_unit`
- **THEN** the system SHALL default to the dose's `planned_unit`

#### Scenario: Patient sends `medication_dose_id` for someone else's schedule
- **WHEN** the dose id resolves to a schedule whose `patient_id` is not the caller's
- **THEN** the system SHALL respond `422` with `{ "medication_dose_id": "The selected medication dose is invalid." }`

#### Scenario: Patient tries to log against a far-future dose
- **WHEN** the dose's `scheduled_at` is more than 1 day after `now`
- **THEN** the system SHALL respond `422` with the `ScheduledDoseTooFarInFutureException` message

#### Scenario: `logged_at` does not carry an offset
- **WHEN** the patient sends `logged_at` without a timezone offset (e.g. `2026-05-15T08:05:00`)
- **THEN** the system SHALL respond `422` with a `date_format` validation error — only strict `Y-m-d\TH:i:sP` is accepted

---

### Requirement: Log an off-schedule intake against a schedule

The system SHALL expose `POST /api/v1/patient/medication-tracking/tracking-logs/by-schedule` for events the patient takes outside the planned slots. Given a `medication_tracking_schedule_id` they own, the endpoint SHALL atomically insert a `tracking_logs` row AND a fresh `medication_tracking_scheduled_doses` row of `type = 'off_schedule'` whose `tracking_log_id` points at the new log. `dose_unit` MUST be required (the schedule's planned unit is not assumed). Two off-schedule logs at the same `logged_at` SHALL produce two separate dose rows — duplicates are intentional.

`logged_at` MUST be strict ISO 8601 with offset and SHALL default to "now". The action SHALL throw `OffScheduleLogWindowException` when `logged_at > now + 1 day` ("Off-schedule logs cannot be created in the future.") or when `logged_at < now - 90 days` ("Off-schedule logs cannot be backfilled more than 90 days in the past."), which the controller MUST map to `422`.

`pain_level` (integer, `between:0,10`) SHALL be required if and only if the resolved schedule's product is of type `INJECTION`. For schedules whose product is `PILLS` or `DROPS`, or for custom schedules whose `product_id IS NULL`, `pain_level` MUST be absent (or explicitly `null`); supplying a value SHALL cause the action to throw `PainLevelDoesNotMatchProductException` ("The pain level field must not be present for this medication."), which the controller MUST map to `422` with a `{"message": ...}` body. Omitting `pain_level` for an injection schedule SHALL throw the same exception with the message "The pain level field is required for this medication." When the rule allows omission, the persisted `tracking_logs.pain_level` SHALL be `NULL`.

#### Scenario: Patient logs an off-schedule dose for an injection
- **WHEN** the patient POSTs against an injection schedule with valid `medication_tracking_schedule_id`, `medication_action`, `dose_value`, `dose_unit`, `pain_level`, optional `note`, and an in-window `logged_at`
- **THEN** the system SHALL respond `201` with a `TrackingLogResource`
- **AND** there SHALL be a new `medication_tracking_scheduled_doses` row with `type=off_schedule`, `intake_id=NULL`, `scheduled_at=log.logged_at`, `planned_quantity=log.dose_value`, `planned_unit=request.dose_unit`, `tracking_log_id=log.id`, and `timezone=schedule.timezone`
- **AND** the persisted log's `pain_level` SHALL equal the submitted value

#### Scenario: Patient logs an off-schedule dose for a PILLS or DROPS schedule without pain_level
- **WHEN** the patient POSTs against a `PILLS` or `DROPS` schedule with the required fields and omits `pain_level`
- **THEN** the system SHALL respond `201`
- **AND** the persisted log's `pain_level` SHALL be `NULL`

#### Scenario: Patient logs an off-schedule dose for a custom schedule without pain_level
- **WHEN** the patient POSTs against a schedule whose `product_id IS NULL` and omits `pain_level`
- **THEN** the system SHALL respond `201`
- **AND** the persisted log's `pain_level` SHALL be `NULL`

#### Scenario: Patient sends pain_level off-schedule against a non-injection schedule
- **WHEN** the patient POSTs `pain_level` with a request that targets a `PILLS`, `DROPS`, or custom schedule
- **THEN** the system SHALL respond `422` with `{"message": "The pain level field must not be present for this medication."}`
- **AND** no `tracking_logs` row and no `medication_tracking_scheduled_doses` row SHALL be persisted

#### Scenario: Patient omits pain_level off-schedule for an injection schedule
- **WHEN** the patient POSTs against an injection schedule without `pain_level`
- **THEN** the system SHALL respond `422` with `{"message": "The pain level field is required for this medication."}`

#### Scenario: Schedule does not belong to the patient
- **WHEN** the resolved schedule has a different `patient_id`
- **THEN** the system SHALL respond `422` with `{ "medication_tracking_schedule_id": "The selected medication schedule is invalid." }`

#### Scenario: Patient omits `dose_unit`
- **WHEN** the request body does not include `dose_unit`
- **THEN** the system SHALL respond `422` — `dose_unit` is required for off-schedule logs

#### Scenario: `logged_at` is in the future
- **WHEN** the request's `logged_at` is more than 1 day after `now`
- **THEN** the system SHALL respond `422` with the future-window message

#### Scenario: `logged_at` is older than the 90-day backfill window
- **WHEN** the request's `logged_at` is more than 90 days before `now`
- **THEN** the system SHALL respond `422` with the backfill-window message

#### Scenario: Two off-schedule logs at the same instant
- **WHEN** the patient submits two by-schedule requests with the same `logged_at`
- **THEN** the system SHALL accept both — two `off_schedule` dose rows SHALL exist (the partial unique index only covers `type = 'scheduled'`)

---

### Requirement: All endpoints require an authenticated patient

Every endpoint in this capability MUST be authenticated via the patient guard (`sanctum`/JWT). Unauthenticated requests SHALL return `401`. The patient identity used to scope every query MUST come from `PatientGuard::getPatient()` — controllers MUST NOT trust patient identifiers from the request body.

#### Scenario: Unauthenticated request
- **WHEN** any of the seven endpoints is called without a valid session
- **THEN** the system SHALL respond `401` with the standard `UnauthorizedError` payload

#### Scenario: Cross-patient access attempt
- **WHEN** an authenticated patient sends an id (`order_id`, `medication_dose_id`, `medication_tracking_schedule_id`, schedule id, intake id) that belongs to a different patient
- **THEN** the system SHALL respond either `422` with a validation error or `403`/`404` from the policy/route binding — it SHALL NEVER expose another patient's data

---

### Requirement: Soft-delete a medication schedule

The system SHALL expose `DELETE /api/v1/patient/medication-tracking/schedules/{schedule}` so the owning patient can remove a schedule from their tracker. Authorization MUST go through the `can:manage,schedule` policy. The implicit route binding MUST NOT resolve trashed rows, so deleting an already-deleted schedule returns `404`.

The operation MUST run inside one `DatabaseManager::transaction`:
1. Compute `now` once.
2. Delete future, unlogged `scheduled`-type doses: `scheduled_at >= now AND type = 'scheduled' AND tracking_log_id IS NULL` AND `medication_tracking_schedule_id = schedule.id`.
3. Soft-delete the schedule via Eloquent's `SoftDeletes` trait (sets `deleted_at`).

Preserved by design (untouched by the delete):
- Past `scheduled`-type doses (`scheduled_at < now`).
- Any `scheduled`-type dose with `tracking_log_id IS NOT NULL` (already logged), regardless of `scheduled_at`.
- All `off_schedule` doses (they are synthetic companions to tracking logs).
- All `tracking_logs` referencing the schedule.

The response SHALL be `204 No Content` with an empty body.

#### Scenario: Patient soft-deletes their schedule
- **WHEN** the owner DELETEs a non-deleted schedule
- **THEN** the system SHALL respond `204`
- **AND** the schedule's `deleted_at` SHALL be set
- **AND** every `scheduled`-type dose with `scheduled_at >= now AND tracking_log_id IS NULL` SHALL be removed from `medication_tracking_scheduled_doses`
- **AND** every dose with `tracking_log_id IS NOT NULL` SHALL remain, regardless of `scheduled_at`
- **AND** every `off_schedule` dose SHALL remain
- **AND** every `tracking_log` referencing the schedule SHALL remain

#### Scenario: List omits soft-deleted schedules
- **WHEN** the patient GETs `/schedules` after deleting a schedule
- **THEN** the deleted schedule SHALL NOT appear in the result
- **AND** the same id passed via `?id[]=...` SHALL also NOT appear

#### Scenario: Daily scheduled-doses view surfaces past logged doses of the deleted schedule
- **WHEN** the patient GETs `/scheduled-doses?date=...&timezone=...` for a day on which the deleted schedule had logged doses
- **THEN** the response SHALL include those logged dose rows with their nested `tracking_log` (so Log History reflects what actually happened that day)
- **AND** the response SHALL NOT include any `scheduled`-type unlogged rows for the deleted schedule (they were hard-deleted on delete)
- **AND** to make this possible the daily-view query's join against `medication_tracking_schedules` MUST bypass the model's `SoftDeletes` global scope (e.g. `joinRelationship('relation', fn ($join) => $join->withTrashed())`), and the eager-loaded schedule relation MUST also use `withTrashed()`

#### Scenario: Tracking-logs index surfaces logs whose schedule was deleted
- **WHEN** the patient GETs `/tracking-logs` after deleting a schedule (custom or order-linked) that had logs
- **THEN** the response SHALL include those tracking logs
- **AND** the `orWhereHas` branch through `medicationDose.medicationTrackingSchedule.patient_id` MUST disable the `SoftDeletes` global scope (`->withoutGlobalScope(SoftDeletingScope::class)`) so logs of trashed schedules — especially custom ones with no `order_id` to match via the order branch — keep surfacing in Log History

#### Scenario: Next-dose view skips deleted schedules
- **WHEN** the patient GETs `/scheduled-doses/next` after deleting their only upcoming schedule
- **THEN** the response SHALL be `204 No Content`

#### Scenario: Deleting an already-deleted schedule
- **WHEN** the patient DELETEs a schedule that is already soft-deleted
- **THEN** the route's implicit binding SHALL NOT resolve the trashed row and the system SHALL respond `404`

#### Scenario: Different patient tries to delete
- **WHEN** an authenticated patient DELETEs a schedule they do not own
- **THEN** `can:manage,schedule` SHALL return `403`
- **AND** the target schedule and its doses SHALL be unchanged

#### Scenario: Unauthenticated request
- **WHEN** the request has no valid patient session
- **THEN** the system SHALL respond `401`

#### Scenario: Delete transaction fails mid-flight
- **WHEN** any step inside the delete transaction throws
- **THEN** the transaction SHALL roll back; the schedule SHALL NOT be trashed and the future doses SHALL NOT be removed

