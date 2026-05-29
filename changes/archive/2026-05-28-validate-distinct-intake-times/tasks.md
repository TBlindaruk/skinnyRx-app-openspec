## 1. Validation rule

- [x] 1.1 Add `'distinct'` to the rule list for `schedule_rule.intakes.*.time` in the `EVERY_DAY`, `WEEKLY`, and `EVERY_FEW_DAYS` branches of `ResolvesScheduleRequest::scheduleRuleRules(...)` at `app/Http/Requests/Api/Patient/MedicationTracking/Schedule/Concerns/ResolvesScheduleRequest.php`. Leave `AS_NEEDED` untouched (size:1 + prohibited `time`).

## 2. Tests

- [x] 2.1 Add a feature test under `tests/Feature/Api/Patient/MedicationTracking/Schedule/` for the create endpoint that POSTs an `every_day` schedule with two intakes at the same `time`, asserts `422`, and asserts the error path includes the duplicate `schedule_rule.intakes.<index>.time` key. Confirm no `medication_tracking_schedules`, no intake rows, and no `scheduled_doses` rows were persisted.
- [x] 2.2 Add the analogous feature test for the update endpoint (PUT with a duplicate-time intake list) — assert `422`, error path keyed on the index, schedule row unchanged, intakes unchanged, future-unlogged doses unchanged.
- [x] 2.3 Add a passing-path test variant ensuring two intakes at distinct times (e.g. `09:00` and `21:00`) still create/edit successfully — guards against accidentally over-broad `distinct` application.

## 3. Verify

- [ ] 3.1 Run `make code-test` (or `php artisan test --parallel --filter=MedicationTracking/Schedule`) and confirm green.
- [ ] 3.2 Run `make run-phpstan` and `make run-psalm` — no new baseline entries.
- [ ] 3.3 Run `make run-pint` — no diffs against the changed file's neighbours.
