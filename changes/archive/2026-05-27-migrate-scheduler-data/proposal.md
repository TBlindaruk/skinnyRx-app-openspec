## Why

The legacy medication scheduler (`medication_schedules` + `medication_dose_reminders`) is being replaced by the new medication-tracking scheduler (`medication_tracking_schedules` + intakes + pre-materialized scheduled doses). Existing patients still have their schedules only in the old tables, so they would lose continuity unless their data is carried over. During the transition both systems can run at once, which would double-notify a patient about the same order — we need a one-shot backfill plus a guard that silences the legacy dose-reminder once an order lives in the new scheduler.

## What Changes

- Add an idempotent artisan command that reads every legacy `MedicationSchedule` and creates the equivalent `MedicationTrackingSchedule` (with intakes and materialized scheduled doses) via the existing `ScheduleCreator`, mapping `apply_times` → daily intakes, `starts_at` → `starts_on`, and carrying over `order_id`, `product_id`, `timezone`.
- Make the command safe to re-run: a legacy schedule whose `(patient_id, order_id)` already exists in the new scheduler is skipped, not duplicated.
- When a legacy order has been migrated, stop the legacy dose-reminder notification for that order: the `SendPatientDoseReminderNotification` job MUST NOT initialize new reminder rows for an order that already has a (non-deleted) `medication_tracking_schedule`, and the migration MUST delete that order's pending (unsent) `medication_dose_reminders` so no in-flight legacy reminder fires.
- Stop the weekly "you have no schedule" nudge for patients who already track in the new scheduler: the `SendPatientDoseReminderWithoutScheduleNotification` job (the Saturday cron) currently only checks the legacy `medication_schedules`, so a patient who migrated to (or self-created) a new tracking schedule still gets nagged to "set up a schedule". The job MUST also exclude any patient who has a non-deleted `medication_tracking_schedule`. This exclusion is patient-level (matched on `patient_id`), not order-level, so a patient with a custom tracker that has no `order_id` is also silenced; the existing legacy check stays order-level on `order_id`.
- Add `--dry-run` and progress reporting consistent with the existing backfill command.

## Capabilities

### New Capabilities
- `scheduler-data-migration`: the artisan command that backfills legacy medication schedules into the new medication-tracking scheduler, its idempotency/skip rules, its field mapping, and the suppression of legacy dose-reminder notifications (both the per-order `SendPatientDoseReminderNotification` and the patient-level weekly `SendPatientDoseReminderWithoutScheduleNotification`) once a patient/order exists in the new scheduler.

### Modified Capabilities
<!-- None. The new medication-tracking-scheduler behavior is unchanged; this change only feeds it data and reads from it. -->

## Impact

- **New:** `app/Console/Commands/MedicationSchedule/MigrateLegacySchedulesCommand.php` (name TBD in design), scheduled or run manually.
- **New:** `app/Services/MedicationTracking/Schedule/LegacyScheduleConverter.php` — maps a legacy `MedicationSchedule` to a `CreateScheduleDto` (timezone resolution + cadence/intake), keeping the command a thin orchestrator.
- **Modified:** `app/Jobs/Patient/Notification/SendPatientDoseReminderNotification.php` — add a `whereNotExists` guard against `medication_tracking_schedules` in `initializePendingRows`.
- **Modified:** `app/Jobs/Patient/Notification/SendPatientDoseReminderWithoutScheduleNotification.php` — add a patient-level `whereNotExists` guard against non-deleted `medication_tracking_schedules` (matched on `patient_id`) alongside the existing order-level legacy `medication_schedules` check.
- **Reuses:** `App\Services\MedicationTracking\Schedule\ScheduleCreator`, `CreateScheduleDto`, `ScheduleRuleDto`, `ScheduledIntakeDto`, `DoseUnit::forProductType`, `App\Repositories\ModelRepository`.
- **Data:** writes `medication_tracking_schedules`, `medication_tracking_schedule_intakes`, `medication_tracking_scheduled_doses`; deletes unsent rows from `medication_dose_reminders`. No schema migration; legacy `medication_schedules` rows are left intact (reversible).
- **Tests:** new `tests/Feature` coverage for the command and the notification guard.
