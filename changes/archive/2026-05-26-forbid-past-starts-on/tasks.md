## 1. Domain exception

- [x] 1.1 Create `app/Services/MedicationTracking/Schedule/Exception/StartsOnInPastException.php` as `final class StartsOnInPastException extends \DomainException {}`, mirroring `EndsOnInPastException.php`. `declare(strict_types=1);`, same namespace prefix, no extra logic.

## 2. Validator

- [x] 2.1 In `app/Services/MedicationTracking/Schedule/Create/CreateScheduleValidator.php`, inject `Carbon\FactoryImmutable $factoryImmutable` as a third constructor-promoted `private readonly` dependency (the existing two stay).
- [x] 2.2 Add `private function assertStartsOnNotInPast(CreateScheduleDto $dto): void` that compares `$dto->localStartsOn` with `$this->factoryImmutable->now($dto->timezone)->startOfDay()` and throws `StartsOnInPastException('The starts_on date cannot be before today in the schedule timezone.')` when `localStartsOn` is `lessThan(...)` today. Do not null-check `localStartsOn` — `CreateRequest::getStartsOnLocal()` always returns a `CarbonImmutable`.
- [x] 2.3 Call `$this->assertStartsOnNotInPast($dto);` as the FIRST line of `validate(...)`, before the product / cap / duplicate-order checks.
- [x] 2.4 Update the `@throws` PHPDoc on `validate(...)` to include `StartsOnInPastException`.

## 3. Controller

- [x] 3.1 In `app/Http/Controllers/Api/Patient/MedicationTracking/ScheduleController.php`, `use App\Services\MedicationTracking\Schedule\Exception\StartsOnInPastException;`.
- [x] 3.2 In `create(CreateRequest $request)`, add `StartsOnInPastException` to the existing `MedicationTrackingScheduleLimitReachedException | ProductConstraintException` catch tuple — same `responseFactory->json(['message' => $exception->getMessage()], 422)` branch. Do NOT widen the 409 catch (that one stays exclusive to `MedicationTrackingScheduleAlreadyExistsException`).
- [x] 3.3 If the OpenAPI 422 response description on `create()` enumerates concrete causes, append "or `starts_on` earlier than today" to the description string.

## 4. Tests

- [x] 4.1 In the feature test class covering `POST /api/v1/patient/medication-tracking/schedules`, add a test that POSTs a valid every-day payload with `starts_on` set to yesterday in `America/New_York`. Use `Carbon::setTestNow(...)` (or the project's `FactoryImmutable` test seam) to pin "now" deterministically. Assert: HTTP 422, JSON `{"message": "The starts_on date cannot be before today in the schedule timezone."}`, and `assertDatabaseCount('medication_tracking_schedules', 0)` + `assertDatabaseCount('medication_tracking_scheduled_doses', 0)`. — Added `create_startsOnInPast_returns422AndPersistsNothing` in `ScheduleController/CreateTest.php`.
- [x] 4.2 Add a test that POSTs the same payload with `starts_on` equal to today in the request's `timezone` and asserts `201` — locks in the `lessThan(...)` (today-inclusive) semantics. — Already covered by pre-existing `EveryDay/CreateTest::create_customEveryDaySchedule_materializesOneRowPerDayPerIntake` (uses `starts_on=2026-05-15` = pinned today and asserts `assertCreated()`).
- [x] 4.3 Add an `as_needed` test with an explicitly past `starts_on` — asserts `422` and the same message body. Documents that the rule fires even on the nullable branch when a value is supplied. — Added `create_asNeededWithExplicitPastStartsOn_returns422` in `AsNeeded/CreateTest.php`.
- [x] 4.4 Add an `as_needed` test that OMITS `starts_on` and asserts `201` — locks in the "omit defaults to today" path still working unchanged. — Already covered by pre-existing `AsNeeded/CreateTest::create_asNeededScheduleWithoutStartedOn_defaultsToTodayInRequestedTimezone`.
- [x] 4.5 Skipped: `DB::enableQueryLog()` would require the forbidden `DB::` facade (CLAUDE.md §3). `assertDatabaseCount` already covers the "no rows persisted" half of the claim; the ordering inside `validate(...)` is structurally guaranteed by the source.
- [x] 4.6 Existing schedule-create feature tests use `starts_on=2026-05-15` (= pinned `setTestNow`) or no `starts_on` at all, so they pass untouched. Verified by reading them; full suite confirms in 5.3.

## 5. Verification

- [x] 5.1 Run `make run-pint` and `make run-phpcs`. Both clean.
- [x] 5.2 Run `make run-phpstan` and `make run-psalm`; confirm no new baseline entries are needed. PHPStan: `[OK] No errors`. Psalm: `No errors found!`. Baselines untouched.
- [x] 5.3 Run `make code-test` and confirm green. Full suite: 3298 tests, 11033 assertions, all passing. Two pre-existing tests had to be updated because they relied on the now-forbidden past-`starts_on` behaviour: (a) `EveryDay/CreateTest::create_backdatedStart_materializesPastSlots` — removed (unreachable via API; generator's range-handling still covered by other create tests); (b) `MedicationTrackingScheduleCreatorTest` — added `setTestNow('2026-05-15 12:00:00')` in `setUp()` so the DTO's `localStartsOn=2026-05-15` aligns with "today" in the test clock.
