## 1. Fix the relation scope

- [x] 1.1 In `app/Models/MedicationSchedule.php`, change `product()` to `return $this->belongsTo(Product::class)->withTrashed();` so the relation resolves soft-deleted catalog rows.
- [x] 1.2 ~~Add a `@return BelongsTo<Product, $this>` PHPDoc on `product()`~~ — skipped. The existing codebase pattern for `belongsTo(...)->withTrashed()` (e.g. `Order::product`, `Prescription::drug`, `Order::patient`) does not carry that generic annotation, and adding it diverges from the convention without changing static-analysis behavior.
- [x] 1.3 Left `getProduct(): Product` non-nullable. None of the four current consumers (`NextDoseReminderCalculator`, `CreateDoseTrackingLogAction`, `CreateOffScheduleTrackingLogAction`, indirect resource usages) were touched — their reads are safe on trashed products.

## 2. Regression test (integration, not unit-on-relation)

- [x] 2.1 Decision change: instead of a thin unit test on `MedicationSchedule::product()` itself, add an integration test on the real crashing path — `SendPatientDoseReminderNotification` eager-loads the product relation and feeds it through `NextDoseReminderCalculator::calculate`, which is exactly where the stage TypeError originated.
- [x] 2.2 Added `handle_ShouldInitPendingRow_WhenScheduleProductIsSoftDeleted` to `tests/Feature/Jobs/Patient/Notification/SendPatientDoseReminderNotificationTest.php`. The test creates a patient+order+product, soft-deletes the product, builds a `MedicationSchedule` for that order/product, runs the job, and asserts a pending `MedicationDoseReminder` row was inserted (proving the job did not crash on `$schedule->getProduct()->getType()`).
- [x] 2.3 Naming + style match the surrounding `handle_ShouldOutcome_WhenCondition` convention used throughout `SendPatientDoseReminderNotificationTest`.

## 3. Verify

- [ ] 3.1 Run `make code-test` (or `php artisan test --parallel --filter=MedicationScheduleProductRelationTest`) and confirm the two new tests pass.
- [ ] 3.2 Run `make run-phpstan` and `make run-psalm` and confirm no new entries land in `phpstan-baseline.neon` / `psalm-baseline.xml`.
- [ ] 3.3 Eyeball Sentry on stage after deploy: the `MedicationSchedule::getProduct()` TypeError (trace id `8a9ab5ad-191f-41cb-9897-d7fa48fb26a4`) should stop firing.
