## 1. Database

- [x] 1.1 Migration: add `deleted_at TIMESTAMP NULL` to `medication_tracking_schedules` (use the standard `$table->softDeletes()` helper); recreate the partial unique index as `(patient_id, order_id) WHERE order_id IS NOT NULL AND deleted_at IS NULL`. Migration is the anonymous-class form. The `up()` does `DROP INDEX medication_tracking_schedules_unique_patient_order; CREATE UNIQUE INDEX ... WHERE ... AND deleted_at IS NULL`. The `down()` is the reverse (drop the new index, recreate the old one, drop `deleted_at`).

## 2. Model

- [x] 2.1 Update `app/Models/MedicationTracking/MedicationTrackingSchedule.php`: `use Illuminate\Database\Eloquent\SoftDeletes;` trait; add `public const FIELD_DELETED_AT = 'deleted_at';`; append `self::FIELD_DELETED_AT => 'immutable_datetime'` to `$casts`.

## 3. Deleter service

- [x] 3.1 Created `app/Services/MedicationTracking/Schedule/ScheduleDeleter.php` — `final readonly`, DI of `DatabaseManager`, `ModelRepository`, `FactoryImmutable`. Inside one transaction: compute `now`, hard-delete future-unlogged `scheduled`-type doses, soft-delete the schedule via `ModelRepository::delete($schedule)`.

## 4. Controller and route

- [x] 4.1 Added `delete(MedicationTrackingSchedule $schedule): Response` to `ScheduleController` with `OA\Delete` attribute. Constructor inject `ScheduleDeleter`. Returns `204` via `responseFactory->noContent()`.
- [x] 4.2 Route registered at `routes/api/api.php:881` inside the `can:manage,schedule` group.

## 4b. History read-path adjustments (added during apply)

The daily-view and Log History reads must keep surfacing the deleted schedule's past events:

- [x] 4b.1 `ScheduledDoseController::index` — replaced `joinRelationship(MEDICATION_TRACKING_SCHEDULE)` with a plain `leftJoin` against `medication_tracking_schedules` so the SoftDeletes global scope no longer filters the join. Added `withTrashed()` on the eager-loaded schedule relation. Patient scoping via `medication_tracking_schedules.patient_id` unchanged.
- [x] 4b.2 `TrackingLogController::index` — added `->withoutGlobalScope(SoftDeletingScope::class)` inside the `orWhereHas('medicationDose.medicationTrackingSchedule', ...)` callback. Critical for custom schedules whose logs carry no `order_id` and would otherwise vanish from Log History after the schedule is deleted.

## 5. Tests

- [x] 5.1–5.3 `tests/Feature/Http/Controllers/Api/Patient/MedicationTracking/ScheduleController/Delete/DeleteScheduleTest.php` — 10 tests, all passing:
  - Happy path (history preserved, future-unlogged removed).
  - 404 / 403 / 401.
  - `GET /schedules` and `id[]` filter omit trashed.
  - `GET /scheduled-doses` returns nothing for the deleted schedule (design.md Decision 4 updated — historical logs live in Log History, not the daily view).
  - `GET /scheduled-doses/next` returns 204.
  - Re-create succeeds for the same `order_id` after delete (partial index migration).
  - Live duplicate still returns 409 (regression on the new index predicate).

## 6. Verify

- [x] 6.1 Pint, PHPStan (app/ only — `tests/` not in path), Psalm — all clean on touched `app/` files.
- [x] 6.2 `php artisan test tests/Feature/.../MedicationTracking/ tests/Feature/Services/MedicationTracking/ tests/Unit/Services/MedicationTracking/` — 168 passed.
- [x] 6.3 `openspec validate delete-medication-schedule` passes.
