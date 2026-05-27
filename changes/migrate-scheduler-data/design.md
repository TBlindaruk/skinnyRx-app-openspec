## Context

Two medication schedulers coexist:

- **Legacy** — `MedicationSchedule` (`medication_schedules`, unique on `order_id`): `apply_times` (JSON array of `H:i`), `starts_at` (date), `timezone`, `product_id`. Reminders are driven by `SendPatientDoseReminderNotification` (every minute): Phase 1 (`initializePendingRows`) inserts a pending `medication_dose_reminders` row per active schedule; Phase 2/3 sends due reminders and queues the next one.
- **New** — `MedicationTrackingSchedule` (`medication_tracking_schedules`, soft-deletes, unique on `(patient_id, order_id)` where `order_id` not null) with `MedicationTrackingScheduleIntake` rows and pre-materialized `MedicationTrackingScheduledDose` rows. Created through `ScheduleCreator::create(CreateScheduleDto)`, which validates (`CreateScheduleValidator`), persists, creates intakes (`CreateScheduleIntakesAction`), and materializes doses (`ScheduledDoseGenerator`) in one transaction.

We must carry existing legacy schedules into the new scheduler and prevent double-notification while both run during the cutover. The repo forbids facades/static model calls and requires `final` classes with constructor/`handle()` DI (see CLAUDE.md). An existing template for this kind of work is `MedicationSchedule/BackfillTimezoneFromShippingAddress` (chunked, `--dry-run`, `ModelRepository`, `LoggerInterface`, progress bar).

## Goals / Non-Goals

**Goals:**
- One artisan command that backfills every legacy schedule into the new scheduler, reusing `ScheduleCreator` (no hand-rolled inserts) so intakes and doses match normal creation.
- Idempotent and re-runnable: never duplicate a `(patient_id, order_id)` schedule.
- Stop legacy dose-reminder notifications for any order that now lives in the new scheduler — both future (guard in the job) and in-flight (delete pending unsent reminders on tracking-schedule creation, covering migration and the patient API alike).
- `--dry-run` + progress/summary output matching the existing backfill command.

**Non-Goals:**
- No schema migration; legacy `medication_schedules` rows are kept intact (reversible cutover).
- Not removing the legacy job or legacy tables — that is a later cleanup change.
- Not migrating any intake quantity/unit history (legacy has none) beyond a sane default.
- Not touching the weekly `SendPatientDoseReminderWithoutScheduleNotification` job — it keys off the presence of a legacy `MedicationSchedule`, which the migration leaves in place, so migrated patients still count as "has legacy schedule" and are unaffected.

## Decisions

### Reuse `ScheduleCreator` rather than raw inserts
Calling `ScheduleCreator::create()` per legacy schedule gives us validation, intakes, and dose materialization for free and guarantees migrated schedules are indistinguishable from API-created ones. Alternative (bulk SQL insert into the three tables) was rejected: it would duplicate the dose-generation logic and drift from the canonical creator. Trade-off: per-row transactions are slower, acceptable for a one-shot backfill on a chunked query.

### Field mapping (product-type-aware — order-linked constraints)
Every legacy schedule has an `order_id`, so each migrated schedule is **order-linked** and must satisfy `ProductConstraintValidator`: exactly one intake, `name == product->getName()`, `unit == DoseUnit::forProductType(type)`, and a per-type schedule rule. The legacy `MedicationSchedule` API also validates `apply_times` as `size:1`, so there is always exactly one time. The legacy→new cadence mapping is the same branch the legacy `NextDoseReminderCalculator` already uses:

- `patient_id` ← `order->getPatientId()` (legacy schedule has no patient column; reach it through the eager-loaded `order`).
- `order` / `product` ← the eager-loaded `Order` / `Product` models passed straight into `CreateScheduleDto`. **Product is mandatory** (order-linked); a legacy row whose product is missing is skipped.
- `name` ← `product->getName()` (must match exactly — no fallback constant).
- `intakes` ← exactly one `ScheduledIntakeDto` from `apply_times[0]`, `quantity = 1.0` (whole number, satisfies the pills rule), `unit = DoseUnit::forProductType($product->getType())`. If a legacy row unexpectedly has >1 apply time, take the first and log a warning (mirrors `NextDoseReminderCalculator`, which uses `apply_times[0]`).
- `scheduleType` / `scheduleRule` ← branch on `product->getType()`:
  - **INJECTION** → `ScheduleType::WEEKLY`, `WeeklyRuleDto(daysOfWeek: [starts_at->dayOfWeekIso])` (ISO 1=Mon … 7=Sun, exactly one day — anchored on the legacy `starts_at` weekday, same as the legacy weekly cadence).
  - **PILLS** / **DROPS** → `ScheduleType::EVERY_DAY`, `EveryDayRuleDto` (empty rule).
- `startsOn` ← `starts_at`; `endsOn` ← `null`; `timezone` ← legacy `timezone`.

### Idempotency via the creator's own guard, plus a pre-check
`CreateScheduleValidator` already throws `MedicationTrackingScheduleAlreadyExistsException` for a duplicate `(patient_id, order_id)`. The command ALSO pre-filters with `whereNotExists` against `medication_tracking_schedules` so re-runs skip cleanly and cheaply; the exception is caught as a belt-and-suspenders skip. Limit/product-constraint exceptions are caught, logged, counted as failures, and do not abort the run.

### Suppress legacy notifications two ways
1. **Future:** add a `whereNotExists` sub-query against `medication_tracking_schedules` (matching on `order_id`, `whereNull(deleted_at)`) to `SendPatientDoseReminderNotification::initializePendingRows`, so no new pending reminder is created for an order that has a tracking schedule.
2. **In-flight:** clean up already-queued reminders **on tracking-schedule creation**, via a `MedicationTrackingScheduleObserver::created` hook (registered with `#[ObservedBy]`, per §9). If the created schedule has an `order_id`, it deletes pending unsent (`sent_at IS NULL`) `medication_dose_reminders` for that order's legacy schedule (`MedicationDoseReminderRepository::deleteUnsentByOrderId`). Already-sent rows are kept as history.

The observer — not the migration command — is the right home for (2): the `created` event fires for **both** the migration (which calls `ScheduleCreator::create`) and the normal patient API (`ScheduleController::create`). A patient who creates an order-linked tracking schedule through the app would otherwise keep receiving the legacy reminder, since the `whereNotExists` guard only blocks *re-initialization*, not an already-pending row. The hook fires inside the `ScheduleCreator` transaction, so the cleanup is atomic with schedule creation and rolls back if dose generation fails. The migration command therefore no longer touches reminders directly.

This pair fully stops legacy reminders for an order once it has a tracking schedule, without depending on ordering relative to the per-minute job.

### Command shape
`final class MigrateLegacySchedulesCommand extends Command`, signature `medication-schedules:migrate-to-tracking {--dry-run}`, DI via `handle(ModelRepository, ScheduleCreator, LoggerInterface)`. Chunk legacy schedules with `chunkById` (size 500), eager-load `order.patient` and `product`. Per row inside a try/catch: build DTO → (skip if dry-run) `create()`. Reminder cleanup is handled by the observer on `created`, not the command. Track `migrated/skipped/failed`, render a summary, return `FAILURE` if any failed else `SUCCESS`. No business logic in the command beyond orchestration.

## Risks / Trade-offs

- **Per-patient schedule limit blocks some migrations** → catch `MedicationTrackingScheduleLimitReachedException`, log with legacy id + order id, count as failed; operator reviews. Migration does not bypass the limit (keeps invariants intact).
- **Product type missing/unmapped for `DoseUnit::forProductType`** → guard the product; if null, skip the row and log (the new schedule needs a product/unit). Counted as skipped.
- **Both schedulers fire between migration of an order and the next job tick** → mitigated by deleting pending unsent reminders at migration time AND the `whereNotExists` guard; worst case a single already-sent reminder that predates migration, which is acceptable.
- **Large volume / long run** → chunked query + per-row transaction; safe to interrupt and re-run thanks to idempotency.
- **Dose materialization volume** (every_day × intakes up to horizon) is significant per schedule → same cost as normal creation; acceptable for a one-time job, run off-peak.

## Migration Plan

1. Ship the `whereNotExists` guard in `SendPatientDoseReminderNotification` and the new command together.
2. Run `--dry-run` on production data, review the migrated/skipped/failed summary and logs.
3. Run for real (off-peak). The command is idempotent, so it can be re-run to pick up failures after fixing root causes.
4. Monitor: migrated orders should stop receiving legacy reminders and start producing new-scheduler doses.
5. Rollback: legacy `medication_schedules` and sent `medication_dose_reminders` are untouched; deleting the created `medication_tracking_*` rows for migrated orders restores the prior state.

## Open Questions

- Default intake `quantity` when the legacy schedule had no notion of dose — assumed `1.0` (whole number so the pills constraint passes; `unit` comes from `DoseUnit::forProductType`). Confirm with product.
- Should the command also soft-delete the legacy `MedicationSchedule` after a successful migration, or strictly leave it (current assumption: leave it, for reversibility and to keep the weekly without-schedule job behavior)?
- INJECTION day-of-week is anchored on the legacy `starts_at` weekday (matching `NextDoseReminderCalculator`). Confirm that is the intended cadence rather than a stored prescription cadence.
