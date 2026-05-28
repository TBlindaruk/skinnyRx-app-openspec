## Context

`ResolvesScheduleRequest` (the trait shared by `CreateRequest` and `UpdateRequest` for `/api/v1/patient/medication-tracking/schedules`) declares per-intake rules under `schedule_rule.intakes.*` — `time` (`H:i`), `quantity > 0`, `unit` (enum). It never asserts that `time` is unique inside the array.

The database does — via the partial unique index `medication_tracking_schedule_intakes_schedule_id_time_unique` on `(medication_tracking_schedule_id, time)`. When the FormRequest lets a payload like `[ { time: "09:00", … }, { time: "09:00", … } ]` through, the request reaches `CreateScheduleIntakesAction::execute(...)`, which bulk-inserts and trips `SQLSTATE[23505]`. The exception is unhandled, so the patient sees a generic 500 with raw SQL embedded — both an ergonomics bug and a small information-disclosure smell.

The fix belongs at the FormRequest layer: that is the only seam in this codebase where input contracts are pinned, and Laravel's built-in `distinct` rule handles per-array-element uniqueness for free.

## Goals / Non-Goals

**Goals:**
- Convert "two intakes with the same `time`" into a 422 with a field-keyed error path, for both create and edit.
- Keep the DB unique index as the last-resort race guard (concurrency, future writes from other code paths).
- Zero changes to DTOs, services, actions, migrations, or the response shape.

**Non-Goals:**
- Pretty-printing arbitrary `QueryException` from the action layer. The action layer remains uninstrumented for this specific constraint; it should never fire from a normal API request.
- Catching the `QueryException` and rethrowing as a domain exception. That is strictly weaker than blocking the request at validation: it would still cost a transaction open + insert before failing, would not produce a field-keyed error path, and would muddle the boundary between "validator says no" and "the database happened to race".
- Deduping intakes silently in the service. Two intakes at `09:00` is a user mistake, not data the server should massage.

## Decisions

### Decision: Use Laravel's built-in `distinct` rule on `schedule_rule.intakes.*.time`

Add `'distinct'` to the rule list for `EVERY_DAY`, `WEEKLY`, and `EVERY_FEW_DAYS` branches of `ResolvesScheduleRequest::scheduleRuleRules(...)`. All `time` values are strings in `H:i` format (`date_format:H:i` validates earlier in the same rule list), so the default loose-equality comparison is correct without `distinct:strict`.

**Alternatives considered:**
- **Custom `App\Rules\DistinctBy` rule.** Overkill — there is no second use case in the repo today (the explore pass found none), and the framework rule is well-known.
- **Catch the `QueryException` in `CreateScheduleIntakesAction` / `UpdateScheduleIntakesAction` and rethrow as a domain exception.** Rejected per Non-Goals: validation must fail before the action runs, both to give a field-keyed error and to avoid opening a transaction for an obviously-bad payload.

### Decision: `AS_NEEDED` branch unchanged

`AS_NEEDED` already pins `schedule_rule.intakes` to `size:1` and `schedule_rule.intakes.*.time` to `prohibited`. A duplicate-time payload cannot reach the database on this path, and adding `distinct` would be dead validation.

### Decision: Apply on the trait, not per-FormRequest

Both `CreateRequest` and `UpdateRequest` derive their `schedule_rule.*` validation from `ResolvesScheduleRequest::scheduleRuleRules(...)`. Touching the trait gives both endpoints the rule in one edit and avoids drift.

## Risks / Trade-offs

- [Race condition between two concurrent edits replacing intakes on the same schedule] → The DB partial unique index already covers it; `UpdateScheduleIntakesAction` deletes the old intake set and re-inserts inside the schedule's transaction, so two concurrent edits would have to interleave inside a single schedule's transaction window. Mitigation: leave the DB index in place; do not weaken it.
- [`distinct` is case-sensitive and value-equality-based] → All `time` values are produced under `date_format:H:i`, so the canonical form is `HH:mm`. Loose equality is sufficient; `distinct:strict` is not needed and would diverge from the DB comparison (which normalizes to `09:00:00`).
- [A future change introduces a per-intake `time` with a different format / extra precision] → If that ever happens, this validator and the unique index BOTH need re-evaluation in tandem. Document the linkage in `ResolvesScheduleRequest`'s docblock so the next reader sees the constraint pair.

## Migration Plan

- Code change only. No data migration; no DB migration; no feature flag.
- Rollback: revert the trait edit. The DB constraint will resume catching duplicates as before.
