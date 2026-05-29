## Context

Two coupled concerns land together:

1. **Edit endpoint** — second slice of SRD-1169. The history-preservation contract mirrors delete: past + logged doses are immutable; future-unlogged doses are regenerated from "now" forward.
2. **Async dose generation** — the current generator materializes up to 730 days synchronously inside the create transaction. With edits regenerating the future, the same synchronous pressure would apply to every PUT. Split into 14-day sync + 365-day async via a queued job; both create and edit funnel through the same generator/job API.

The monthly backfill cron (the natural follow-on — "extend any schedule whose horizon shortened as time passed") is deliberately deferred to its own slice.

## Goals / Non-Goals

**Goals**
- Add `PUT /schedules/{schedule}` with strict history-preservation invariants.
- One generator API used by create, edit, and future backfill (`generate(schedule, from, mode)`).
- 14-day sync + 365-day async tail via a single queued job.
- Order-product locking still applies on edit for Rx-linked schedules.

**Non-Goals**
- Monthly backfill cron (separate slice).
- Untake (separate slice, Log History work).
- Retroactive intake edits (past doses are immutable).
- Editing the legacy `/medication-tracking/medication-schedules` (the older `MedicationSchedule` model with `apply_times`).
- Shortening already-materialized 730-day tails on existing schedules.

## Decisions

### Decision 1: Generator is a primitive — callers compute their window

`ScheduledDoseGenerator::generate(...)` takes two `CarbonImmutable` bounds and fills the window:

```php
public function generate(
    MedicationTrackingSchedule $schedule,
    CarbonImmutable $windowStart,
    CarbonImmutable $windowEnd,  // exclusive
): void
```

It internally clips `windowEnd` at `ends_on` when that's set, then yields rows via the per-`ScheduleType` strategy. The generator has **no knowledge** of sync/async horizons — that's a caller concern. Strategies iterate using the inclusive `windowStart` / exclusive `windowEnd` contract (no double-counting on edge dates). `EveryFewDays` anchors on `schedule.starts_on` so the rule's cadence stays stable regardless of where the window starts — the strategy iterates from `starts_on` and filters to the window.

Callers compute their own windows from `MedicationTrackingConfig` constants:

| Caller | Window |
|---|---|
| `ScheduleCreator` (sync portion) | `[starts_on, starts_on + SYNC_HORIZON_DAYS)` |
| `ScheduleEditor` (sync portion) | `[now, now + SYNC_HORIZON_DAYS)` |
| `GenerateScheduledDosesJob` (async tail) | `[from + SYNC_HORIZON_DAYS, from + ASYNC_HORIZON_DAYS)`, where `from` is passed at dispatch time |

No `Mode` enum. The earlier draft had one (`SYNC_WINDOW` / `ASYNC_TAIL` / `FULL`), but it was abstracting something callers already know — their context tells them which window to ask for. A future backfill cron can compute its own window the same way (`[max(now, lastDose+interval), now + ASYNC_HORIZON_DAYS)`).

Both `ScheduleCreator` and `ScheduleEditor` dispatch via `dispatch(...)->afterCommit()` (or `DatabaseManager::transaction(...)`'s after-commit hook) so a rollback never leaves a phantom job in the queue. The job re-fetches the schedule by id and short-circuits on trashed or `as_needed` — see Decision 6.

Picking 14 + 365 (vs. 7 + 365, 30 + 730, etc):
- 14 days is enough that the FE's daily-view and "next dose" use cases never need to wait for the queue to drain — the patient can navigate two weeks forward immediately.
- 365 days is the longest horizon meaningful for the tracker — beyond that, prescription renewals/re-onboarding reset state anyway.
- Both are config-tunable (env keys) so we can dial sync down to 7 under latency pressure or up to 30 for a specific deploy.

### Decision 2: Edit transaction order mirrors delete

In `ScheduleEditor::edit(...)`:

1. **Compute `now` once** (in the schedule's timezone). Every subsequent step uses this snapshot.
2. **Hard-delete future-unlogged scheduled doses.** Predicate identical to delete: `scheduled_at >= now AND type = 'scheduled' AND tracking_log_id IS NULL` scoped to this schedule.
3. **Update the schedule row.** `ends_on`, `schedule_type`, `schedule_rule`. Immutable fields are never touched even if the request body somehow leaks past validation.
4. **Replace the intake set.** Insert the new intake rows, delete the rows that no longer match. See Decision 3 for what happens to past doses that referenced removed intakes.
5. **Materialize the new sync window** with `generate($schedule, $now, Mode::SYNC_WINDOW)`.

After `commit`, dispatch the async job.

The first three steps happen on the **pre-update** schedule shape; step 5 reads the updated schedule + new intake set. The order matters: step 2 needs the OLD strategy to identify which future rows to keep (it doesn't — predicate is purely on dose properties, no strategy involvement, so this is safe regardless of order). Step 5 must run on the updated state.

### Decision 3: Past doses keep their `intake_id` even when the intake is replaced

When intakes are replaced, the old `medication_tracking_schedule_intakes` rows are deleted. Past `scheduled`-type doses (those with `scheduled_at < now` or `tracking_log_id IS NOT NULL`) may still reference them via `intake_id`. We **rely on the application invariant that past doses are read-only** and accept a possibly-dangling `intake_id`.

Alternatives considered:
- Soft-delete the intake rows — adds a `deleted_at` column to a table that has none and complicates every join. Rejected.
- Re-point past doses at a synthetic "deleted" intake — workable but obscure. Rejected.
- Null out `intake_id` on past doses — loses the slot identity. Rejected; future Log History queries may want to know "this used to be the 8am dose" even if the intake row is gone.

Each dose already denormalizes `planned_quantity` and `planned_unit`, so the historical row is self-describing without the intake. The dangling foreign key is documented as an invariant: **a dose's `intake_id` may point at a deleted intake row when `tracking_log_id IS NOT NULL` or `scheduled_at < schedule.updated_at`**.

### Decision 4: `UpdateRequest` silently ignores immutable fields

`PUT` semantics on the wire are "the body is the new state", but five fields stay immutable: `name`, `starts_on`, `timezone`, `product_id`, `order_id`. The FormRequest doesn't validate them, the `UpdateScheduleDto` doesn't carry them, and the editor / persistence layer never reads them. End state: their values on the row are unchanged after the edit.

An earlier draft used Laravel's `prohibited` rule to reject these fields with `422`. We dropped that — the API contract is "we update what we update; everything else is at-rest". Clients that send extra fields get a successful response and no surprise side-effects.

### Decision 5: No partial PATCH semantics

`PUT` is a full-state replace for the editable fields. We don't support partial updates ("change just the dose, leave everything else"). Reasons:

- Keeps the contract simple: the FE assembles the new state and sends it once.
- Avoids ambiguity around `null` ("set to null" vs. "leave alone") for `ends_on` and `schedule_rule.days_of_week`.
- Mirrors how create works: the patient already knows how to build a full schedule payload.

If a partial-update use case emerges later (e.g. "I just want to bump dose without re-sending the intake list"), it can be a separate `PATCH /schedules/{schedule}` endpoint with its own ticket. Out of scope here.

### Decision 6: Job idempotency relies on the existing unique index + a fresh fetch

`GenerateScheduledDosesJob::handle(...)`:

```php
$schedule = $modelRepository->getQuery(MedicationTrackingSchedule::class)
    ->withTrashed()
    ->whereKey($this->scheduleId)
    ->first();

if ($schedule === null || $schedule->trashed()) {
    return;
}
if ($schedule->getScheduleType() === ScheduleType::AS_NEEDED) {
    return;
}

$from = CarbonImmutable::parse($this->fromIso);
$generator->generate(
    $schedule,
    $from->addDays(MedicationTrackingConfig::SYNC_HORIZON_DAYS),
    $from->addDays(MedicationTrackingConfig::ASYNC_HORIZON_DAYS),
);
```

`insertOrIgnore` against `(intake_id, scheduled_at) WHERE type = 'scheduled'` makes re-runs no-ops. We don't use `ShouldBeUnique`: when a follow-on change (the monthly backfill cron) also dispatches for the same schedule, both should be allowed to run — the second is a free idempotent confirm.

The job takes the schedule's `ends_on` and the current `async_horizon_days` setting at handle time, so a config change or `ends_on` edit between dispatch and handle Just Works.

### Decision 7: Replace the config layer with one constants class

The current `env → config/medication_tracking.php → provider reads → primitive into ctor` is four layers of indirection for "what's 14". In practice nobody tuned these per environment — `.env.example` and `.env.testing` always carried the same values. It's cargo-cult config.

Replace with a single class holding all medication-tracking knobs:

```php
namespace App\Services\MedicationTracking;

final class MedicationTrackingConfig
{
    public const SYNC_HORIZON_DAYS         = 14;
    public const ASYNC_HORIZON_DAYS        = 365;
    public const SCHEDULE_LIMIT_PER_PATIENT = 50;
}
```

`MedicationTrackingServiceProvider` reads from the class, asserts the one cross-value invariant, and wires primitives into service constructors as before:

```php
$this->app->singleton(ScheduledDoseGenerator::class, fn (Application $app) => new ScheduledDoseGenerator(
    // ... services ...
    syncHorizonDays:  MedicationTrackingConfig::SYNC_HORIZON_DAYS,
    asyncHorizonDays: MedicationTrackingConfig::ASYNC_HORIZON_DAYS,
));

$this->app->bind(CreateScheduleValidator::class, fn (Application $app) => new CreateScheduleValidator(
    modelRepository: $app->make(ModelRepository::class),
    productConstraintValidator: $app->make(ProductConstraintValidator::class),
    scheduleLimitPerPatient: MedicationTrackingConfig::SCHEDULE_LIMIT_PER_PATIENT,
));

Assert::greaterThan(
    MedicationTrackingConfig::ASYNC_HORIZON_DAYS,
    MedicationTrackingConfig::SYNC_HORIZON_DAYS,
);
```

Service constructor signatures don't change — they still take `int` primitives via named arguments. Tests that need to vary a knob (the `CreateScheduleValidator` cap test currently uses `config()->set(...)`) re-bind the service in the container with a different primitive instead. One-time refactor of a single existing test.

Files deleted: `config/medication_tracking.php`. Keys removed from `.env.example` and `.env.testing`: `MEDICATION_TRACKING_DOSE_GENERATION_HORIZON_DAYS`, `MEDICATION_TRACKING_SYNC_HORIZON_DAYS` (never landed), `MEDICATION_TRACKING_ASYNC_HORIZON_DAYS` (never landed), `MEDICATION_TRACKING_SCHEDULE_LIMIT_PER_PATIENT`.

The `Assert::greaterThan` stays — copy-paste errors are still possible inside the constants class, and the assertion is free at boot.

**Why one class and not constants per service?** Centralizing avoids the "where do I look for the dose horizon" hunt. The class is small, focused, and explicitly named — a clear single source of truth for medication-tracking domain knobs.

### Decision 8: Authorization unchanged — `can:manage,schedule` covers edit and delete

The policy already grants `manage` to the owning patient. Both PUT and DELETE on `/schedules/{schedule}` go through the same `'middleware' => 'can:manage,schedule'` group (already in place for the per-intake `PATCH /planned-quantity`).

## Risks / Trade-offs

- **In-flight 730-day tails.** Schedules created before this change still have ~730 days of doses materialized. The new generation cap doesn't shrink them — and our edit only deletes future-unlogged rows, then materializes a fresh 365-day tail. If a patient edits an old schedule, the tail past the new 365-day horizon is silently dropped (step 2). For unedited old schedules, the 730-day tail keeps draining naturally as time passes. No data migration; accepted.
- **Edit + concurrent intake replacement.** Two devices editing the same schedule in parallel can clobber each other's intake set. Same exposure as any other PUT-replace API in the repo. Out of scope.
- **Replaced intakes leave dangling FKs.** Documented as invariant — past doses are read-only and self-describing via denormalized `planned_quantity` / `planned_unit`. If we ever need a strict-FK report, it can read with `withTrashed`-style joins or accept nullable joins.
- **Async tail latency.** The patient sees their next 14 days immediately; the rest fills as the queue worker catches up. With Horizon (already running) and one job per non-as_needed create/edit, queue load is modest. If we ever see backlog, we tune `sync_horizon_days` upward per deploy.
- **No retry hardening on the job.** If a job throws, Laravel's default retry kicks in. The work is idempotent — retries are safe. We're not adding `tries`, `backoff`, or a custom `failed()` handler in this slice. If failures correlate with specific schedules, we can add a Sentry tag in a follow-up.
