## 1. Remove the active-order filter from the command

- [x] 1.1 In `app/Console/Commands/MedicationTracking/RegenerateScheduledDosesCommand.php::eligibleSchedulesQuery()`, delete the `$activeStatusValues` and `$patientHasActiveOrderSubquery` locals and the `->whereExists($patientHasActiveOrderSubquery)` call on the main query.
- [x] 1.2 Remove the now-unused `$ordersTable` local and the `use App\Models\Order\Order;` / `use App\Models\Order\OrderStatus;` imports (verify they are not referenced elsewhere in the file first).
- [x] 1.3 Confirm the remaining query keeps the `select`, `selectSub(last_scheduled_at)`, `schedule_type != as_needed`, and `ends_on` null-or-future predicates unchanged.

## 2. Update tests

- [x] 2.1 Add/adjust a test asserting the command dispatches for an eligible schedule whose patient has **no** active order (covers the new spec scenario), for both a custom (`order_id IS NULL`) and an order-linked schedule.
- [x] 2.2 In `tests/Feature/Console/Commands/MedicationTracking/RegenerateScheduledDosesCommandTest.php`, remove or invert the now-obsolete active-order tests: `handle_skipsSchedulesOfPatientsWithNoOrders`, `handle_skipsSchedulesOfPatientsWhoseOrdersAreAllInactive`, `handle_dispatchesForCustomScheduleWhenPatientHasAnyActiveOrder`, `handle_dispatchesForOrderLinkedScheduleWhenPatientHasAnyActiveOrder`, `handle_patientWithManyActiveOrders_dispatchesScheduleOnlyOnce`.
- [x] 2.3 Drop the now-irrelevant active-order setup from `handle_dispatchesJobsOnlyForEligibleSchedulesOfActivePatients` (rename to reflect that eligibility no longer depends on patient order status).
- [ ] 2.4 Verify the retained window/horizon tests (`whenNoDosesYet…`, `whenDosesAlreadyMaterialized…`, `whenMaterializedHorizonAlreadyCoversTodayPlus365…`, `whenStartsOnIsInTheFuture…`) still pass without their active-order preconditions.

## 3. Verify

- [ ] 3.1 Run the command's feature test suite (inside the container) and confirm green.
- [ ] 3.2 Run PHPStan + Psalm + Pint on the changed files; confirm no unused-import or analyser regressions and baselines are not grown.
