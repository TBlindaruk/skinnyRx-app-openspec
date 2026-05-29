## Context

`POST /api/v1/patient/medication-tracking/schedules` already runs two layers of validation:

1. **`CreateRequest`** — shape only (`date_format:Y-m-d`, `required-unless-as-needed`, IANA timezone, intake bounds, schedule_rule rules).
2. **`CreateScheduleValidator`** — semantic checks against DB + DTO state (product constraints for order-linked schedules, per-patient schedule cap, duplicate `(patient_id, order_id)` short-circuit before the DB partial unique index).

The edit path uses the same split: `UpdateRequest` for shape, `UpdateScheduleValidator` for semantics. The "no past `ends_on`" rule lives in the second layer:

```php
// UpdateScheduleValidator
private function assertEndsOnNotInPast(MedicationTrackingSchedule $schedule, UpdateScheduleDto $dto): void
{
    if ($dto->localEndsOn === null) {
        return;
    }
    $todayInScheduleTimezone = $this->factoryImmutable->now($schedule->getTimezone())->startOfDay();
    if ($dto->localEndsOn->lessThan($todayInScheduleTimezone)) {
        throw new EndsOnInPastException('The ends_on date cannot be before today in the schedule timezone.');
    }
}
```

`EndsOnInPastException extends \DomainException` and the controller maps it to `422 {"message": ...}`:

```php
} catch (ProductConstraintException | EndsOnInPastException $exception) {
    return $this->responseFactory->json(['message' => $exception->getMessage()], 422);
}
```

We mirror that pattern for create's `starts_on`. There is no Laravel-rules step involved.

## Goals / Non-Goals

**Goals:**
- Reject create requests whose `starts_on` is earlier than today in the request's `timezone`.
- Mirror the update flow's approach 1:1 — same exception shape, same controller mapping, same "validator first" ordering.
- No DB query for a request that is going to fail on a pure-DTO predicate.

**Non-Goals:**
- Do not add a Laravel validation rule to `CreateRequest`. The semantic check lives in the service layer, like its `ends_on` twin.
- Do not change the update / delete / list / dose-logging endpoints.
- Do not change DTOs, migrations, routes, or OpenAPI schemas.
- Do not introduce a generic `NotInPast` rule class — only the `starts_on` site needs it today; if the pattern repeats, lift later.

## Decisions

### Place the check in `CreateScheduleValidator`, not the FormRequest

The semantic rule "X must be ≥ today in timezone Y" is logically identical to the existing `assertEndsOnNotInPast(...)` on update. Putting it in the FormRequest would create asymmetry with the update flow (which can't validate in the FormRequest because the timezone lives on the bound model, not in the payload), and would split the schedule's "valid date window" logic across two layers for no reason.

CLAUDE.md §5 ("Throw domain exceptions, don't return false") and §19's "Untyped `array` returns... → A typed DTO or domain exception" both point at the same convention.

**Alternative considered:** Laravel `after_or_equal:YYYY-MM-DD` built dynamically from the request's timezone. Rejected — it works, but it asymmetric to update and would leave the spec's `starts_on`/`ends_on` rules described in two different sublanguages.

### Throw a new `StartsOnInPastException`, don't reuse `EndsOnInPastException`

Same shape, but the message and semantic meaning are distinct. Reusing would cost no code but make catch tuples and tests ambiguous. The exception class is ~10 lines.

```php
final class StartsOnInPastException extends \DomainException
{
}
```

### Comparison source = `now($dto->timezone)->startOfDay()` via injected `FactoryImmutable`

The DTO carries `localStartsOn` (already a `CarbonImmutable` in the request's TZ) and `timezone` (the IANA string from the request body). We compute "today" from the SAME timezone. Injecting `Carbon\FactoryImmutable` (the project's standard wrapped clock — used by `UpdateScheduleValidator` already) keeps the comparison test-mockable via `Carbon::setTestNow(...)` or factory replacement.

```php
private function assertStartsOnNotInPast(CreateScheduleDto $dto): void
{
    $todayInRequestTimezone = $this->factoryImmutable->now($dto->timezone)->startOfDay();

    if ($dto->localStartsOn->lessThan($todayInRequestTimezone)) {
        throw new StartsOnInPastException('The starts_on date cannot be before today in the schedule timezone.');
    }
}
```

`$dto->localStartsOn` is never null at this point — `CreateRequest::getStartsOnLocal()` returns "today in TZ" for the `as_needed` omit path, so the value is always a `CarbonImmutable`. No null guard needed (cf. update's `EndsOn`, which IS nullable).

### Run `assertStartsOnNotInPast` FIRST in `validate(...)`

Order matters:

```
validate(CreateScheduleDto $dto):
  1. assertStartsOnNotInPast($dto);          ←── new, pure-DTO, cheap
  2. if ($dto->product !== null) productConstraintValidator->validate(...);
  3. count(*) check for schedule cap;        ←── DB query
  4. order-schedule exists check;            ←── DB query
```

A past date never reaches the DB. Mirrors update where `assertEndsOnNotInPast` is also line 1.

### Controller catch — extend the existing 422 tuple

```php
} catch (
    MedicationTrackingScheduleLimitReachedException
    | ProductConstraintException
    | StartsOnInPastException $exception  // new
) {
    return $this->responseFactory->json(['message' => $exception->getMessage()], 422);
}
```

`MedicationTrackingScheduleAlreadyExistsException` stays in its own 409 branch as today.

### Error message wording

`"The starts_on date cannot be before today in the schedule timezone."` — verbatim shape of the `ends_on` twin. "Schedule timezone" is technically the request-supplied timezone at create time (there's no row yet), but using the same noun phrase keeps the API surface coherent with the update message. The frontend already handles the update string for past-`ends_on` and renders a localised user message; same flow applies here.

## Risks / Trade-offs

- **Wire shape changes vs. a Laravel-rules implementation.** Past `starts_on` produces `{"message": "..."}`, not `{"errors": {"starts_on": [...]}}`. → Mitigation: this matches the existing past-`ends_on` shape on update, so frontend treatment is already uniform.
- **Clock skew across regions.** "Today" is server-evaluated using the request's timezone. A client whose device is hours ahead and submits its local-tomorrow can still get rejected if the server's wall clock hasn't crossed midnight in that TZ yet. → Mitigation: same skew window applies to every "today"-anchored rule in this codebase (sync horizon, async tail, etc.). Acceptable. Covered in the scenario.
- **DST transition days.** `FactoryImmutable::now($tz)->startOfDay()` handles DST cleanly via Carbon. No mitigation needed.
- **`as_needed` with an explicitly past `starts_on`.** Today's behaviour: accepted (because the field is nullable and the FormRequest doesn't compare to today). After this change: rejected. → Mitigation: covered by a dedicated spec scenario; surfaced in "What Changes".
- **Order of exceptions in the controller's catch tuple.** PHP catches the FIRST matching `catch` block; all three exceptions are siblings of `\DomainException` with no inheritance between them, so order inside the tuple is purely cosmetic. No risk.

## Migration Plan

Single-PR change. Deploy normally. No feature flag — purely additive validation; clients submitting today-or-later (the documented happy path) are unaffected.

Rollback: revert the PR. No data is migrated.
