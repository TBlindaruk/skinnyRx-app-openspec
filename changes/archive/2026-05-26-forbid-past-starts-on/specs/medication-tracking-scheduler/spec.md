## MODIFIED Requirements

### Requirement: Create a custom medication schedule

The system SHALL allow an authenticated patient to create a medication schedule that is not tied to any order. The schedule MUST persist its intakes as a real `medication_tracking_schedule_intakes` relation and MUST pre-materialize scheduled-dose rows up to the configured horizon, all inside a single database transaction.

The supported `schedule_type` values are `every_day`, `weekly`, `every_few_days`, and `as_needed`. The `schedule_rule` payload SHALL carry only rule-specific metadata (e.g. `days_of_week` for `weekly`, `interval_days` for `every_few_days`); intakes SHALL be a top-level array on the request and on the response, never embedded inside `schedule_rule`.

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
