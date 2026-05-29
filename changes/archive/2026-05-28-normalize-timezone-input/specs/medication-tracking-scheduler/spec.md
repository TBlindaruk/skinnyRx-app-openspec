## MODIFIED Requirements

### Requirement: Create a custom medication schedule

The system SHALL allow an authenticated patient to create a medication schedule that is not tied to any order. The schedule MUST persist its intakes as a real `medication_tracking_schedule_intakes` relation and MUST pre-materialize scheduled-dose rows up to the configured horizon, all inside a single database transaction.

The supported `schedule_type` values are `every_day`, `weekly`, `every_few_days`, and `as_needed`. The `schedule_rule` payload SHALL carry only rule-specific metadata (e.g. `days_of_week` for `weekly`, `interval_days` for `every_few_days`); intakes SHALL be a top-level array on the request and on the response, never embedded inside `schedule_rule`.

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

#### Scenario: Patient sends an invalid timezone
- **WHEN** the patient POSTs with a `timezone` value not in `DateTimeZone::ALL_WITH_BC` (e.g. `Not/A_Zone`, or `europe/kiev` with the wrong case)
- **THEN** the system SHALL respond `422` with a validation error on `timezone`

#### Scenario: Patient sends a deprecated IANA alias
- **WHEN** the patient POSTs with `timezone=Europe/Kiev` (or any other backward-compatibility alias such as `US/Eastern`, `Asia/Calcutta`)
- **THEN** the system SHALL respond `201` with the schedule persisted using the same string the patient sent

#### Scenario: Patient sends `ends_on` earlier than `starts_on`
- **WHEN** the patient POSTs with `ends_on < starts_on`
- **THEN** the system SHALL respond `422` with a validation error on `ends_on`

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
- **THEN** the system SHALL respond `200` with the same dose set it would return for `timezone=Europe/Kyiv` (window math depends on the underlying zone, which is identical)

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
- **THEN** the system SHALL respond `200` with the same `ScheduledDoseResource` it would return for `timezone=Europe/Kyiv`

#### Scenario: Patient asks for the next dose with an unknown timezone
- **WHEN** the patient GETs with a `timezone` value not in `DateTimeZone::ALL_WITH_BC`
- **THEN** the system SHALL respond `422` with a validation error on `timezone`
