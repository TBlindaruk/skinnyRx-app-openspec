## 1. Constants class + provider rewiring

- [x] 1.1 Create `app/Services/MedicationTracking/MedicationTrackingConfig.php` — `final class` (not readonly — only `public const` members). Constants: `SYNC_HORIZON_DAYS = 14`, `ASYNC_HORIZON_DAYS = 365`, `SCHEDULE_LIMIT_PER_PATIENT = 50`. No constructor, no methods.
- [x] 1.2 Delete `config/medication_tracking.php`.
- [x] 1.3 Remove all `MEDICATION_TRACKING_*` keys from `.env.example` and `.env.testing`.
- [x] 1.4 `MedicationTrackingServiceProvider`:
  - drop the `Repository $config` resolution,
  - `Assert::greaterThan(MedicationTrackingConfig::ASYNC_HORIZON_DAYS, MedicationTrackingConfig::SYNC_HORIZON_DAYS)` at the top of `register()`,
  - bindings pass `MedicationTrackingConfig::*` constants as named arguments into `ScheduledDoseGenerator` (with `syncHorizonDays` + `asyncHorizonDays`) and `CreateScheduleValidator` (with `scheduleLimitPerPatient`),
  - `provides()` unchanged.

## 2. Generator refactor

- [x] 2.1 Refactor `ScheduledDoseGenerator`:
  - drop the `int $horizonDays` constructor param (it had no replacement — the generator stops knowing about horizons),
  - `generate(MedicationTrackingSchedule $schedule, CarbonImmutable $windowStart, CarbonImmutable $windowEnd): void`,
  - body clips `windowEnd` at `ends_on` when set; early-returns when the window is empty,
  - chunked `insertOrIgnore` at 1 000 rows unchanged.
- [x] 2.2 Update `ScheduledDoseStrategy` interface — rename `startDate`/`endDate` → `windowStart`/`windowEnd`, document that `windowEnd` is **exclusive**.
- [x] 2.3 Update each strategy implementation:
  - `EveryDay`: iterate `for ($day = $windowStart; $day->lessThan($windowEnd); $day = $day->addDay())`.
  - `Weekly`: same iteration; filter by `daysOfWeek`.
  - `EveryFewDays`: anchor on `schedule->getStartsOn()` (parse in schedule TZ), iterate by `intervalDays`, yield only within `[windowStart, windowEnd)`. This keeps cadence stable regardless of where the window starts.
  - `AsNeeded`: still returns `[]`.

## 3. `GenerateScheduledDosesJob`

- [x] 3.1 Create `app/Jobs/MedicationTracking/GenerateScheduledDosesJob.php`. Extends `App\Jobs\Job`. Constructor: `int $scheduleId, string $fromIso`.
- [x] 3.2 `handle(ModelRepository $repo, ScheduledDoseGenerator $generator): void`:
  - re-fetch the schedule via `getQuery(MedicationTrackingSchedule::class)->withTrashed()->whereKey($this->scheduleId)->first()`,
  - return early if `null` or `trashed()`,
  - return early if `schedule_type === AS_NEEDED`,
  - parse `$from = CarbonImmutable::parse($this->fromIso)`,
  - call `$generator->generate($schedule, $from->addDays(MedicationTrackingConfig::SYNC_HORIZON_DAYS), $from->addDays(MedicationTrackingConfig::ASYNC_HORIZON_DAYS))`.

## 4. Refactor `ScheduleCreator`

- [x] 4.1 Inside the existing `db->transaction(...)`, after persisting schedule + intakes, call `$generator->generate($schedule, $startsOnInTimezone, $startsOnInTimezone->addDays(MedicationTrackingConfig::SYNC_HORIZON_DAYS))`. Skip the call entirely for `as_needed`.
- [x] 4.2 After the transaction commits, dispatch `GenerateScheduledDosesJob::dispatch($schedule->getId(), $startsOn->toIso8601String())->afterCommit()` for non-`as_needed` schedules.

## 5. Edit endpoint

- [x] 5.1 `app/Http/Requests/Api/Patient/MedicationTracking/Schedule/UpdateRequest.php`:
  - allowed fields: `ends_on` (nullable, `date_format:Y-m-d`, `after_or_equal:today` evaluated in the schedule's TZ — apply in `prepareForValidation` or via a custom rule), `schedule_type`, `schedule_rule`, `intakes[]`,
  - prohibited: `name`, `starts_on`, `timezone`, `product_id`, `order_id` via the `prohibited` rule with a clear message,
  - typed getters: `getEndsOn(): ?CarbonImmutable`, `getScheduleType(): ScheduleType`, `getScheduleRule(): ScheduleRuleDto`, `getIntakes(): array`.
- [x] 5.2 `app/DTO/MedicationTracking/Schedule/UpdateScheduleDto.php` — mirrors `CreateScheduleDto` minus immutable fields. Constructor-promoted public properties.
- [x] 5.3 `app/Services/MedicationTracking/Schedule/Update/UpdateScheduleValidator.php`:
  - constructor injects `ProductConstraintValidator`,
  - validate: order-linked schedules call into `ProductConstraintValidator` with the request's would-be state; reject `schedule_type` drift; assert `schedule_rule` matches the resulting `schedule_type`.
- [x] 5.4 `app/Services/MedicationTracking/Schedule/ScheduleEditor.php`:
  - DI: `DatabaseManager, ScheduledDoseGenerator, UpdateScheduleValidator, FactoryImmutable, ModelRepository`,
  - `edit(MedicationTrackingSchedule $schedule, UpdateScheduleDto $dto): MedicationTrackingSchedule`,
  - flow per design.md Decision 2: validator → tx { compute `now` → delete future-unlogged → update schedule row → replace intakes → `generator->generate($schedule, $now, $now->addDays(MedicationTrackingConfig::SYNC_HORIZON_DAYS))` (skip for `as_needed`) } → after-commit dispatch `GenerateScheduledDosesJob` (skip for `as_needed`).
- [x] 5.5 `ScheduleController::update(MedicationTrackingSchedule $schedule, UpdateRequest $request): JsonResponse`:
  - constructor-inject `ScheduleEditor`,
  - build the DTO, call the editor, return `200` with `ScheduleResource` (intakes + product eager-loaded),
  - catch `ProductConstraintException` → 422 with the exception message,
  - `OA\Put` attribute mirroring `create`.
- [x] 5.6 `routes/api/api.php` — add `Route::put('/', [ScheduleController::class, 'update'])->name('update');` inside the existing `can:manage,schedule` group (next to the existing `delete` route).

## 6. Tests

- [x] 6.1 `tests/Unit/Services/MedicationTracking/ScheduledDose/ScheduledDoseGeneratorTest.php`:
  - given `[windowStart, windowEnd)` for an `every_day` schedule with 2 intakes → exactly `days × 2` rows.
  - `windowEnd` past `ends_on` is clipped — rows stop at `ends_on`.
  - `as_needed` → zero rows regardless of window.
  - Idempotent re-run (re-call against same window produces no new rows).
  - `EveryFewDays` cadence anchored on `starts_on` even when the window starts mid-cadence (e.g. interval=3, starts_on=day 0, window=[day 5, day 20) → rows at days 6, 9, 12, 15, 18; NOT 5, 8, 11, ...).
- [x] 6.2 `tests/Feature/Jobs/MedicationTracking/GenerateScheduledDosesJobTest.php`:
  - happy path: dispatches, processes, doses materialized.
  - schedule trashed before handle → no rows, no failure.
  - `as_needed` schedule → no rows.
- [x] 6.3 `tests/Feature/Http/Controllers/Api/Patient/MedicationTracking/ScheduleController/Update/UpdateScheduleTest.php`:
  - happy path on a custom every-day schedule: past dose untouched, future-unlogged replaced at new quantity, logged future dose untouched.
  - `schedule_type` change on custom: `every_day` → `weekly`, async tail generated correctly.
  - `schedule_type` change attempt on order-linked: 422 (`ProductConstraintException`).
  - `ends_on` in the past → 422.
  - `ends_on = null` when previously set → 200, async tail extends to year horizon.
  - Each prohibited field (`name`, `starts_on`, `timezone`, `product_id`, `order_id`) → 422.
  - Different patient → 403.
  - Unauthenticated → 401.
  - Trashed schedule → 404.
  - `Queue::fake()` + `Queue::assertPushed(GenerateScheduledDosesJob::class, 1)` to confirm the after-commit dispatch.
- [x] 6.4 Update `tests/Feature/Http/Controllers/Api/Patient/MedicationTracking/ScheduleController/CreateTest.php`:
  - after `201`, only `sync_horizon_days × intakes` rows exist; not the full year.
  - exactly one `GenerateScheduledDosesJob` is queued for non-`as_needed` creates.
  - `as_needed` creates: zero rows + zero jobs queued.
- [x] 6.5 Update `tests/Feature/Http/Controllers/Api/Patient/MedicationTracking/ScheduledDoseController/IndexTest.php` and `NextTest.php` if any fixture relied on a full 730-day tail being materialized synchronously. Where needed, dispatch the job inline (`Queue::fake(); ...; $job->handle($repo, $generator);`).
- [x] 6.6 Update existing `tests/Feature/Services/MedicationTracking/Schedule/MedicationTrackingScheduleCreatorTest.php` (or equivalent) to assert the new dispatch.
- [x] 6.7 Update `CreateTest::create_whenPatientHasReachedScheduleLimit_returns422` (the only existing test that uses `config()->set('medication_tracking.schedule_limit_per_patient', 3)`): re-bind `CreateScheduleValidator` in the container with `scheduleLimitPerPatient: 3` instead of touching config. Same test intent, swapped mechanism.

## 7. Verify

- [x] 7.1 `make check-code` is green; no new baseline entries.
- [x] 7.2 `php artisan test --parallel --filter MedicationTracking` passes.
- [x] 7.3 `php artisan l5-swagger:generate` succeeds with no warnings.
- [x] 7.4 `openspec validate edit-medication-schedule` passes.
