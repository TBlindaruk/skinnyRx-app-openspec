## Why

Stage is emitting a `TypeError` from `App\Models\MedicationSchedule::getProduct()` because the method's return type is non-null `Product`, but the underlying `product()` `belongsTo` relation excludes soft-deleted rows by default. `Product` uses `SoftDeletes`, so any `MedicationSchedule` whose product was archived after the schedule was created — a normal lifecycle event — currently crashes any code path that touches `getProduct()`.

The schedule row itself still has a valid `product_id`, and the domain expects the schedule to outlive the product (the user still needs to see and act on their dose plan after the catalog SKU is retired). The relation just needs to include trashed rows.

Trace seen on stage:

```
stage.ERROR: App\Models\MedicationSchedule::getProduct(): Return value must be of type App\Models\Product\Product, null returned
{"trace_id":"8a9ab5ad-191f-41cb-9897-d7fa48fb26a4",
 "exception":"[object] (TypeError: ... at /app/app/Models/MedicationSchedule.php:80)"}
```

## What Changes

- `MedicationSchedule::product()` resolves the related `Product` even when it has been soft-deleted (`->withTrashed()`), matching the existing CLAUDE.md §6 convention ("Soft-deleted relations included with `->withTrashed()` where the domain expects it").
- `MedicationSchedule::getProduct()` keeps its non-null `Product` return type and stops throwing on soft-deleted products.
- Add a regression test covering: a schedule whose product was soft-deleted after the schedule was created still returns a `Product` instance from `getProduct()`.
- Audit the four current consumers of `$schedule->getProduct()` to confirm none of them break when the returned product is trashed:
  - `App\Services\MedicationTracking\NextDoseReminderCalculator::calculate` — reads `getType()`, safe on trashed rows.
  - `App\Services\MedicationTracking\TrackingLog\CreateDoseTrackingLogAction::execute` — passes the product to `PainLevelMatchesProductValidator`, safe.
  - `App\Services\MedicationTracking\TrackingLog\CreateOffScheduleTrackingLogAction::execute` — same validator, safe.

No behavior change for the un-deleted case.

## Capabilities

### New Capabilities

- `medication-schedule-product-resolution`: rule that `MedicationSchedule::getProduct()` must resolve the originating `Product` for the lifetime of the schedule, including after the product has been soft-deleted from the catalog.

### Modified Capabilities

<!-- none -->

## Impact

- **Code touched**: `app/Models/MedicationSchedule.php` (one-line relation change).
- **Tests**: one new feature test under `tests/Feature/` exercising the soft-deleted-product path.
- **Callers**: no signature changes. All four current call sites (`NextDoseReminderCalculator`, two `TrackingLog` create actions, and the `getImagePath` accessor in legacy ScheduleResource paths that proxy through `MedicationTrackingSchedule`) continue to work; they read fields that survive a soft delete (`type`, `image_path`, identity).
- **External systems**: none. No migration, no API contract change.
- **Risk**: low — `withTrashed()` on a `belongsTo` is a local change, additive to the query scope.
