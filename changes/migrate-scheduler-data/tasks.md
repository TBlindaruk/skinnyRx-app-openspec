## 1. Suppress legacy notifications once an order has a tracking schedule

- [x] 1.1 In `SendPatientDoseReminderNotification::initializePendingRows`, add a `whereNotExists` sub-query against `medication_tracking_schedules` matching `order_id` and `deleted_at IS NULL`, so orders with a tracking schedule are never re-initialized.
- [x] 1.2 Add `MedicationDoseReminderRepository::deleteUnsentByOrderId(int $orderId)` (deletes unsent reminders whose legacy schedule belongs to that order).
- [x] 1.3 Add `MedicationTrackingScheduleObserver::created` (registered via `#[ObservedBy]`) that, when the created schedule has an `order_id`, calls `deleteUnsentByOrderId`. This covers both the migration and the patient API; the migration command no longer deletes reminders directly.

## 2. Migration command

- [x] 2.1 Create `final class MigrateLegacySchedulesCommand extends Command` in `app/Console/Commands/MedicationSchedule/` with signature `medication-schedules:migrate-to-tracking {--dry-run}` and a clear `$description`.
- [x] 2.2 Inject dependencies via `handle(ModelRepository, ScheduleCreator, LoggerInterface): int`.
- [x] 2.3 Build the legacy query: `getQuery(MedicationSchedule::class)`, eager-load `order.patient` and `product`, and `whereNotExists` against an existing non-deleted `medication_tracking_schedule` for the same `(patient_id, order_id)`; `chunkById` at size 500 with progress bar (mirror `BackfillTimezoneFromShippingAddress`).
- [x] 2.4 Map each legacy row to `CreateScheduleDto`: `patientId` from `order->getPatientId()`, `order`/`product` models, `name = product->getName()`, exactly one `ScheduledIntakeDto` from `apply_times[0]` (`quantity = 1.0`, `unit = DoseUnit::forProductType($product->getType())`), `startsOn = starts_at`, `endsOn = null`, `timezone`. Branch cadence on product type (matching `NextDoseReminderCalculator`): INJECTION → `scheduleType = WEEKLY` + `WeeklyRuleDto([starts_at->dayOfWeekIso])`; PILLS/DROPS → `scheduleType = EVERY_DAY` + `EveryDayRuleDto`.
- [x] 2.5 Skip rows with a null `timezone` (nullable column) or empty `apply_times`; log and count as skipped. `order`/`product` are non-null FKs, so they are read via typed getters without a guard.
- [x] 2.6 On a non-dry-run, call `ScheduleCreator::create($dto)` (the observer handles reminder cleanup on `created`).
- [x] 2.7 Catch `MedicationTrackingScheduleAlreadyExistsException` (count as skipped) and `MedicationTrackingScheduleLimitReachedException` / `ProductConstraintException` / other `Throwable` (log with legacy id + order id, count as failed); never abort the whole run.
- [x] 2.8 Track `migrated/skipped/failed`, print a summary, return `Command::FAILURE` if any failed else `Command::SUCCESS`; on `--dry-run` make zero writes.

## 3. Tests

- [x] 3.1 Feature test: migrating a pills/drops legacy schedule creates one `every_day` tracking schedule with one intake and materialized doses (carried-over fields); migrating an injection legacy schedule creates a `weekly` schedule with `days_of_week = [starts_at ISO weekday]`.
- [x] 3.2 Feature test: re-running the command (order already in new scheduler) creates no duplicate and counts the row as skipped.
- [x] 3.3 Feature test: legacy schedule with null timezone is skipped without writes.
- [x] 3.4 Feature test: migration deletes pending unsent `medication_dose_reminders` for the order but leaves sent ones intact (via the observer on the creation path).
- [x] 3.7 Observer test: creating an order-linked tracking schedule deletes that order's pending unsent legacy reminders (keeps sent); creating a custom (no-order) schedule deletes none.
- [x] 3.5 Feature test: `--dry-run` writes no `medication_tracking_*` rows and deletes no reminders.
- [x] 3.6 Feature test: `SendPatientDoseReminderNotification` does not initialize a pending reminder for an order that has a non-deleted `medication_tracking_schedule`, but still initializes for orders that don't.

## 4. Verification

- [ ] 4.1 Run `make run-pint`, `make run-phpstan`, `make run-psalm` and the new tests; ensure baselines are not grown.
