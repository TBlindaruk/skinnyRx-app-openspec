## Context

`RegenerateScheduledDosesCommand` builds a single eligibility query over `medication_tracking_schedules`, streamed via `ModelRepository::getQuery(...)->cursor()`. Today that query carries four predicates: not-trashed, `schedule_type != as_needed`, `ends_on` open-or-future, and a `WHERE EXISTS` semi-join asserting the patient has at least one order in `OrderStatus::active()`. We want to drop the last predicate now and reconsider patient-state eligibility later.

## Goals / Non-Goals

**Goals:**
- Remove the active-order semi-join from the eligibility query so every schedule passing the type / `ends_on` / soft-delete filters is topped up.
- Keep the per-schedule window math, dispatch, idempotency, and cron cadence byte-for-byte identical.

**Non-Goals:**
- Designing the eventual replacement eligibility rule (deferred — tracked only as a note in the spec/proposal).
- Touching `GenerateScheduledDosesJob`, the generator, or the schedule.

## Decisions

- **Remove the predicate at the query layer, not the cursor loop.** Deleting `whereExists($patientHasActiveOrderSubquery)` (and the `$activeStatusValues` / `$patientHasActiveOrderSubquery` locals that feed it) keeps the "filter in SQL, compute per-row in PHP" shape intact. Alternative — keep the subquery but make it a no-op via config flag — rejected: it leaves dead code and an unused `orders` join the planner still has to consider; the user explicitly wants the filter gone for now, cleanly.
- **Drop the now-unused `Order` / `OrderStatus` imports.** They are referenced only by the removed subquery; leaving them would fail the unused-import checks.

## Risks / Trade-offs

- More schedules dispatched per run (churned/never-onboarded patients are no longer skipped) → acceptable and reversible: the job is idempotent and inserts nothing once a horizon is current, so the extra cost is bounded query/dispatch work, not duplicate rows.
- Loss of the active-order gate's intent (don't materialize for churned patients) → explicitly accepted; the spec records that a better-defined gate may return later.

## Migration Plan

Pure code change — no schema, queue-contract, or data migration. Rollback = revert the commit; the dropped predicate is self-contained.
