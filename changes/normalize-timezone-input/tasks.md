## 1. Relax the validator on scheduler FormRequests

- [x] 1.1 In `app/Http/Requests/Api/Patient/MedicationTracking/Schedule/CreateRequest.php` change the `timezone` rule from `'timezone:all'` to `'timezone:all_with_bc'`.
- [x] 1.2 Same change in `app/Http/Requests/Api/Patient/MedicationTracking/ScheduledDose/IndexRequest.php`.
- [x] 1.3 Same change in `app/Http/Requests/Api/Patient/MedicationTracking/ScheduledDose/NextRequest.php`.
- [x] 1.4 Same change in `app/Http/Requests/Api/Patient/MedicationTracking/MedicationSchedule/CreateRequest.php` (keep `'nullable'`).
- [x] 1.5 Same change in `app/Http/Requests/Api/Patient/MedicationTracking/MedicationSchedule/UpdateRequest.php` (keep `'nullable'`).

## 2. Feature tests — new scheduler

- [x] 2.1 In `tests/Feature/.../ScheduleController/CreateTest.php` add a scenario that POSTing with `timezone=Europe/Kiev` returns `201` and the schedule is persisted; do not assert any canonical form of the stored string.
- [x] 2.2 In the same file add a scenario that POSTing with `timezone=Not/A_Zone` returns `422` with a `timezone` validation error (rejection still fires through the relaxed rule).
- [x] 2.3 In `tests/Feature/.../ScheduledDoseController/IndexTest.php` add a scenario that a schedule stored under `Europe/Kyiv` returns the same dose count when queried with `timezone=Europe/Kiev`.
- [x] 2.4 In `tests/Feature/.../ScheduledDoseController/NextTest.php` add the equivalent: a Kiev-aliased query still returns the next dose.
- [x] 2.5 Confirm the pre-existing `index_invalidTimezone_returns422` and `next_invalidTimezone_returns422` tests still pass against the relaxed rule (they should — both submit `Atlantis/Lost_City`).

## 3. Feature tests — legacy scheduler

- [x] 3.1 In `tests/Feature/.../MedicationScheduleControllerTest.php` add: POSTing to legacy create with `timezone=Europe/Kiev` returns `201` and the row is persisted.
- [x] 3.2 In the same file add: PATCHing legacy update with `timezone=Europe/Kiev` returns `200`.
- [x] 3.3 In the same file add an "unknown timezone" `422` scenario on update for parity with the existing `create_withInvalidTimezone_shouldReturn422`.

## 4. Verification

- [x] 4.1 Run the full `tests/Feature/Http/Controllers/Api/Patient/MedicationTracking` suite; everything must stay green.
- [ ] 4.2 Run `make run-pint`, `make run-phpstan`, `make run-psalm` on demand (do not auto-run) and address any new findings without growing the baselines.
