## 1. Verify schedule creation

- [x] 1.1 Read `ScheduleController::create` — `resolveOrder()` runs before any service call; the catch list maps `MedicationTrackingScheduleAlreadyExistsException` → 409, `MedicationTrackingScheduleLimitReachedException` / `ProductConstraintException` → 422; no `product_id` field is read from the request.
- [x] 1.2 Read `ScheduleCreator::create` — `validator->validate($dto)` runs **before** `db->transaction(...)`; inside the tx the order is `persistSchedule → createIntakes->execute → generator->generate`.
- [x] 1.3 Read `CreateScheduleValidator` — cap check uses the injected `$scheduleLimitPerPatient` (resolved from `medication_tracking.schedule_limit_per_patient` in the provider); duplicate-order check is `where(patient_id)->where(order_id)->exists()`.
- [x] 1.4 Read `ProductConstraintValidator` and the three strategies — name/intake-count/unit are checked cross-cuttingly; `PillsProductConstraintStrategy` locks to `every_day` + planned-quantity bounds, `InjectionProductConstraintStrategy` locks to `weekly` + `days_of_week.length == 1`, `DropsProductConstraintStrategy` locks to `every_day`. Matches the spec.
- [x] 1.5 Migration confirms the partial unique index `(patient_id, order_id) WHERE order_id IS NOT NULL` on `medication_tracking_schedules`.

## 2. Verify dose materialization

- [x] 2.1 `ScheduledDoseGenerator::INSERT_CHUNK_SIZE = 1000`, uses `$table->insertOrIgnore($buffer)` per chunk.
- [x] 2.2 `AsNeededScheduledDoseStrategy` returns no rows; the other three iterate up to `horizonDays`.
- [x] 2.3 `MedicationTrackingServiceProvider` asserts `Assert::positiveInteger(...)` on both `dose_generation_horizon_days` and `schedule_limit_per_patient`, passes them via named arguments. Provider is `DeferrableProvider` and lists both bindings in `provides()`.
- [x] 2.4 Migration confirms the partial unique index `(intake_id, scheduled_at) WHERE type = 'scheduled'`.

## 3. Verify daily and next dose views

- [x] 3.1 `ScheduledDoseController::index` — OR between `(logged_at >= dayStart AND logged_at < dayEnd AND tracking_log_id NOT NULL)` and `(type = 'scheduled' AND tracking_log_id IS NULL AND scheduled_at in window)`; `orderByRaw('COALESCE(logged_at, scheduled_at)')`; patient scope via `joinRelationship → where(MedicationTrackingSchedule.patient_id)`.
- [x] 3.2 `ScheduledDoseController::next` — filters `type = 'scheduled'`, orders by `scheduled_at` ASC, returns `responseFactory->noContent()` (204) when no row.

## 4. Verify forward-only quantity update

- [x] 4.1 Route `routes/api/api.php:881-885` uses `scopeBindings()` and the parent group has `'middleware' => 'can:manage,schedule'`.
- [x] 4.2 `UpdateScheduledDosePlannedQuantityAction::execute` — inside one `db->transaction`: updates intake's `quantity`, then `where(type=scheduled) AND where(intake_id) AND where(scheduled_at >= dose.scheduled_at) AND whereNull(tracking_log_id)` update of `planned_quantity`.
- [x] 4.3 `UpdatePlannedQuantityRequest` — `PlannedQuantityForProductType` is on the `planned_quantity` rule list (same rule reused from `PillsProductConstraintStrategy`).

## 5. Verify tracking-log endpoints

- [x] 5.1 `CreateByDoseController::__invoke` — dose lookup is `whereKey($id)->whereHas(schedule, patient_id check)`; on miss throws `ValidationException::withMessages(['medication_dose_id' => 'The selected medication dose is invalid.'])`. `CreateDoseTrackingLogAction` updates the dose's `tracking_log_id` and inserts the log in one tx; future-gate throws `ScheduledDoseTooFarInFutureException` which the controller maps to 422 with the exception message.
- [x] 5.2 `CreateByScheduleController::__invoke` — schedule lookup is `whereKey($id)->where(patient_id)->first()`; on miss throws `ValidationException::withMessages(['medication_tracking_schedule_id' => '...'])`. `CreateOffScheduleTrackingLogAction` defines `private const OFF_SCHEDULE_BACKFILL_DAYS = 90`; inside the tx inserts both the `TrackingLog` and a synthetic `MedicationTrackingScheduledDose` (`type=off_schedule`, `intake_id=null`, `scheduled_at=log.loggedAt`, `planned_quantity=log.doseValue`, `planned_unit=dto.doseUnit`, `tracking_log_id=log.id`, `timezone=schedule.timezone`). Window guard throws `OffScheduleLogWindowException` on both future and >90d-past cases, mapped to 422.
- [x] 5.3 Both `CreateByDoseRequest` and `CreateByScheduleRequest` declare `logged_at => 'nullable|date_format:Y-m-d\TH:i:sP'`.

## 6. Final cross-check

- [x] 6.1 The spec matches the feature doc end-to-end. **No drift found.** The only known anti-pattern (`ScheduleResource::getImagePath()` using `app()->get(Generator::class)`) is acknowledged in `design.md` Decision 9 and explicitly out of scope.
- [x] 6.2 `openspec validate document-medication-tracking-scheduler` — passes.
- [x] 6.3 `openspec status --change ...` — all 4 artifacts complete.
