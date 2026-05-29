## Why

`POST /api/v1/patient/medication-tracking/schedules` currently accepts any well-formed `starts_on` date, including dates in the past. A past start date produces a schedule whose sync window is already behind "now", so `ScheduledDoseGenerator` materializes dose rows for days the patient cannot meaningfully act on, and the daily-doses view immediately shows "missed" doses the patient never had a chance to take. The edit flow already enforces "no past `ends_on`" via `UpdateScheduleValidator::assertEndsOnNotInPast(...)` throwing `EndsOnInPastException` — the symmetric rule for `starts_on` at create time is missing. This change mirrors that pattern exactly.

## What Changes

- Add a new domain exception `App\Services\MedicationTracking\Schedule\Exception\StartsOnInPastException` extending `\DomainException`, mirroring `EndsOnInPastException`.
- Inject `Carbon\FactoryImmutable` into `CreateScheduleValidator` and add a private `assertStartsOnNotInPast(CreateScheduleDto $dto): void` method that compares `$dto->localStartsOn` against `now($dto->timezone)->startOfDay()`. Throw `StartsOnInPastException` when `localStartsOn` is earlier.
- Call `assertStartsOnNotInPast(...)` as the **first** step in `CreateScheduleValidator::validate(...)` — before the product / limit / duplicate-order checks — so we don't run any DB query for a request that is going to fail on a pure-DTO predicate, mirroring how the update path runs `assertEndsOnNotInPast` first.
- Catch `StartsOnInPastException` in `ScheduleController::create(...)` and map it to `422 {"message": …}`, joining the existing `MedicationTrackingScheduleLimitReachedException | ProductConstraintException` catch tuple.
- `as_needed` schedules that omit `starts_on` still default to "today in the provided timezone" via the existing `CreateRequest::getStartsOnLocal()` fallback — no behaviour change for the nullable path. The new check only fires when `localStartsOn` resolves to a past date.
- `CreateRequest` itself is NOT modified: semantic "past date" validation lives in the service layer, matching the project's domain-exceptions convention (CLAUDE.md §5) and the update flow 1:1.

## Capabilities

### New Capabilities

<!-- none -->

### Modified Capabilities
- `medication-tracking-scheduler`: the "Create a custom medication schedule" requirement gains a scenario rejecting `starts_on` earlier than today in the request timezone via a `StartsOnInPastException` mapped to `422 {message}`.

## Impact

- `app/Services/MedicationTracking/Schedule/Exception/StartsOnInPastException.php` — new file, ~10 lines, mirror of `EndsOnInPastException`.
- `app/Services/MedicationTracking/Schedule/Create/CreateScheduleValidator.php` — adds `FactoryImmutable` to the constructor and the `assertStartsOnNotInPast(...)` method; one new line at the top of `validate(...)`.
- `app/Http/Controllers/Api/Patient/MedicationTracking/ScheduleController.php` — `use StartsOnInPastException;` and add the class to the existing 422 catch tuple in `create(...)`.
- `tests/Feature/...` covering the schedule-create endpoint — new tests asserting `422 {message: "The starts_on date cannot be before today in the schedule timezone."}` for past `starts_on` (custom + as_needed paths) and `201` for today.
- No DB / route / DTO / FormRequest / OpenAPI-schema changes.
- Response shape note: the wire shape on past `starts_on` becomes `{"message": "..."}` (matching the past-`ends_on` response on update), NOT Laravel's `{"errors": {"starts_on": [...]}}`. Frontend that already handles the update-flow shape is unaffected.
