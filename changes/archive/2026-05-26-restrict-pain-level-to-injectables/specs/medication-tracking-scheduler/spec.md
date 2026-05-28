## MODIFIED Requirements

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
