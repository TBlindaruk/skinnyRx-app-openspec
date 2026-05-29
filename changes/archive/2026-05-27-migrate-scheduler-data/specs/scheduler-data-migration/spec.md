## ADDED Requirements

### Requirement: Backfill legacy schedules into the new tracking scheduler

The system SHALL provide an artisan command that reads every legacy `medication_schedules` row and creates the equivalent record in the new medication-tracking scheduler (`medication_tracking_schedules` with its intakes and pre-materialized scheduled doses), reusing the existing `ScheduleCreator` so that intakes and scheduled-dose generation follow the same rules as a normally created schedule.

Every legacy schedule is order-linked, so the created schedule SHALL satisfy the order-linked product constraints: exactly one intake, `name` equal to the product name, and `unit = DoseUnit::forProductType(productType)`. The system SHALL map legacy fields as follows: `order_id` → schedule `order`, `product_id` → schedule `product`, the order's patient → `patient_id`, `starts_at` (date) → `starts_on`, `timezone` → `timezone`, and the single `apply_times` entry → one intake with `quantity = 1`. The cadence SHALL branch on product type, matching the legacy `NextDoseReminderCalculator`: an INJECTION product SHALL map to `schedule_type = weekly` with exactly one day of week anchored on the legacy `starts_at` weekday (ISO 1=Monday … 7=Sunday); a PILLS or DROPS product SHALL map to `schedule_type = every_day` with an empty rule.

#### Scenario: Migrate a pills/drops legacy schedule (daily cadence)

- **WHEN** the command runs against a legacy `MedicationSchedule` for an order whose patient has no matching new schedule, with `apply_times = ["08:00"]`, a non-null `timezone`, a PILLS or DROPS product, and a `starts_at`
- **THEN** the system SHALL create one `medication_tracking_schedule` with `schedule_type = every_day`, `name` equal to the product name, and `order_id`/`product_id`/`patient_id`/`starts_on`/`timezone` carried over from the legacy row
- **AND** SHALL create one `medication_tracking_schedule_intake` (time `08:00`, `quantity = 1`, `unit` from the product type) and materialize the corresponding `medication_tracking_scheduled_doses` up to the configured horizon

#### Scenario: Migrate an injection legacy schedule (weekly cadence)

- **WHEN** the command runs against a legacy `MedicationSchedule` whose order product is an INJECTION, with a `starts_at` falling on a Wednesday
- **THEN** the system SHALL create one `medication_tracking_schedule` with `schedule_type = weekly` and a `schedule_rule` of `{ "days_of_week": [3] }` (the ISO weekday of `starts_at`)
- **AND** the materialized `medication_tracking_scheduled_doses` SHALL fall only on that weekday up to the configured horizon

#### Scenario: Legacy schedule is missing a timezone but it can be resolved from the shipping address

- **WHEN** the command encounters a legacy `MedicationSchedule` whose `timezone` is null (the `timezone` column is nullable, unlike the non-null `order_id`/`product_id` foreign keys)
- **THEN** the system SHALL resolve the timezone from the order patient's shipping address using `AddressTimezoneResolver` (the same resolver the `medication-schedules:backfill-timezone` command uses)
- **AND** SHALL use the resolved timezone to create the new-scheduler record

#### Scenario: Legacy schedule has no timezone and none can be resolved

- **WHEN** a legacy `MedicationSchedule` has a null `timezone` and the timezone cannot be resolved from the shipping address (the patient has no shipping address, or its state is unsupported)
- **THEN** the system SHALL skip that schedule without creating a new-scheduler record and SHALL count it as skipped, because the new scheduler requires a timezone to materialize doses

### Requirement: Migration is idempotent and safe to re-run

The command SHALL be safe to run multiple times. A legacy schedule whose `(patient_id, order_id)` already has a non-deleted `medication_tracking_schedule` SHALL be skipped rather than duplicated, and the per-patient schedule limit and product constraints enforced by `ScheduleCreator` SHALL be respected — a schedule that the creator rejects SHALL be logged and counted as failed/skipped without aborting the whole run.

#### Scenario: Order already exists in the new scheduler

- **WHEN** the command processes a legacy schedule whose order already has a `medication_tracking_schedule` for the same patient
- **THEN** the system SHALL NOT create a second new-scheduler record for that order
- **AND** SHALL count the row as skipped and continue with the remaining legacy schedules

#### Scenario: Creator rejects a schedule

- **WHEN** creating the new schedule throws a domain exception (e.g. per-patient limit reached or a product constraint)
- **THEN** the system SHALL log the failure with the legacy schedule id and order id and continue processing the remaining rows, exiting with a non-zero status if any failures occurred

#### Scenario: Dry run makes no changes

- **WHEN** the command is invoked with `--dry-run`
- **THEN** the system SHALL report what would be migrated without writing any `medication_tracking_*` rows and without deleting any `medication_dose_reminders`

### Requirement: Stop legacy dose-reminder notifications once an order has a tracking schedule

Once an order exists in the new scheduler, the legacy dose-reminder pipeline SHALL NOT notify the patient about that order, so a patient is never reminded twice for the same order. This SHALL hold regardless of how the tracking schedule was created (the migration command OR the normal patient-facing API), so the behavior is a property of tracking-schedule creation, not of the migration command alone.

Two mechanisms enforce this:

1. The legacy `SendPatientDoseReminderNotification` job SHALL NOT initialize new `medication_dose_reminders` rows for a legacy schedule whose order already has a non-deleted `medication_tracking_schedule`.
2. On creation of any order-linked `medication_tracking_schedule`, the system SHALL delete any pending (unsent, `sent_at IS NULL`) `medication_dose_reminders` belonging to that order's legacy schedule, so already-queued legacy reminders do not fire. Already-sent reminders (`sent_at` not null) SHALL be left untouched as history. Creating a custom (non-order) tracking schedule SHALL NOT delete any legacy reminders.

#### Scenario: Legacy reminder initialization skips an order with a tracking schedule

- **WHEN** the legacy dose-reminder job runs and a legacy schedule's order already has a non-deleted `medication_tracking_schedule`
- **THEN** the job SHALL NOT create a new pending `medication_dose_reminder` for that schedule
- **AND** schedules whose order is not in the new scheduler SHALL still be initialized as before

#### Scenario: Creating an order-linked tracking schedule clears in-flight legacy reminders

- **WHEN** an order-linked `medication_tracking_schedule` is created for an order that has pending unsent `medication_dose_reminders` (whether by the migration command or the patient API)
- **THEN** the system SHALL delete those pending unsent reminders for that order's legacy schedule
- **AND** SHALL leave already-sent reminders (`sent_at` not null) untouched as history

#### Scenario: Creating a custom tracking schedule leaves legacy reminders intact

- **WHEN** a custom `medication_tracking_schedule` with no `order_id` is created
- **THEN** the system SHALL NOT delete any `medication_dose_reminders`

### Requirement: Stop the weekly no-schedule nudge for patients who already track

The weekly `SendPatientDoseReminderWithoutScheduleNotification` job (the Saturday cron) nudges patients with an active order but no medication schedule to set one up. It SHALL NOT nudge a patient who already tracks medication in the new scheduler. The exclusion SHALL be patient-level: a patient who has any non-deleted `medication_tracking_schedule` (`deleted_at IS NULL`) SHALL be excluded, regardless of whether that schedule is linked to an order. The pre-existing legacy check SHALL remain order-level: a patient SHALL still be selected only if they have at least one active order with no legacy `medication_schedules` row. A patient SHALL therefore receive the weekly nudge only when they have an active order without a legacy schedule AND have no non-deleted tracking schedule at all.

#### Scenario: Patient with a new tracking schedule is not nudged

- **WHEN** the weekly job runs and a patient has an active order with no legacy `medication_schedules` row but has a non-deleted `medication_tracking_schedule`
- **THEN** the system SHALL NOT send that patient the no-schedule dose reminder

#### Scenario: Patient with only a custom (no-order) tracking schedule is not nudged

- **WHEN** the weekly job runs and a patient has an active order with no legacy schedule and their only tracking schedule has a null `order_id`
- **THEN** the system SHALL NOT send that patient the no-schedule dose reminder, because the exclusion matches on `patient_id`

#### Scenario: Patient with no schedule of any kind is still nudged

- **WHEN** the weekly job runs and a patient has an active order with no legacy `medication_schedules` row and no non-deleted `medication_tracking_schedule`
- **THEN** the system SHALL send that patient the no-schedule dose reminder as before

#### Scenario: Soft-deleted tracking schedule does not suppress the nudge

- **WHEN** the weekly job runs and a patient's only `medication_tracking_schedule` is soft-deleted (`deleted_at` is not null)
- **THEN** that schedule SHALL NOT count as tracking, and the patient SHALL be nudged if they otherwise qualify
