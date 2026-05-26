## Context

The medication-tracking scheduler ships with create + read but no mutation on the schedule itself beyond per-intake quantity. SRD-1169 wants delete. We're slicing it off as its own change so the diff stays small, reviewable, and shippable independently.

Touch points are tight: one model change (SoftDeletes), one migration (column + index recreation), one new service (`ScheduleDeleter`), one new controller method, one route line, plus tests.

## Goals / Non-Goals

**Goals**
- Soft-delete a schedule with strict history preservation: past doses, logged doses, off-schedule doses, and tracking logs survive.
- Hard-delete future, unlogged `scheduled`-type doses so reads stop surfacing the deleted schedule.
- Allow re-creating an Rx-linked schedule for the same `order_id` after delete.

**Non-Goals**
- No edit endpoint.
- No async dose generation refactor (kept the existing synchronous 730-day horizon).
- No monthly backfill cron.
- No Untake.
- No changes to `tracking_logs` shape or routes.

## Decisions

### Decision 1: Soft-delete the schedule, hard-delete future-unlogged doses

`MedicationTrackingSchedule` gets the `SoftDeletes` trait. The schedule row survives because off-schedule tracking logs still reference it (via `tracking_logs.medication_dose_id → medication_tracking_scheduled_doses.medication_tracking_schedule_id`). Soft-deleting also keeps the row queryable via `withTrashed()` if Log History needs to display the schedule's name next to a historical log.

Future, unlogged `scheduled`-type doses are **hard-deleted**: they have nothing referencing them (`tracking_log_id IS NULL`), they cost nothing to recreate if needed, and they would otherwise force every dose-read query to add `WHERE schedule.deleted_at IS NULL`. Eloquent's `SoftDeletes` global scope on the schedule handles the read filtering via the existing joins — see Decision 4.

### Decision 2: Migrate the partial unique index instead of nulling `order_id`

Today the partial index is `(patient_id, order_id) WHERE order_id IS NOT NULL`. After soft-delete, the trashed row still has `order_id` set, so a new schedule for the same order would collide on the index.

Two ways to fix it:

1. **Null out `order_id` on the trashed row** as part of the delete transaction.
2. **Migrate the index** to `(patient_id, order_id) WHERE order_id IS NOT NULL AND deleted_at IS NULL`.

I'm going with (2):
- Keeps the original `order_id` link on the deleted row, which is useful for reports that want to attribute historical logs to the order they came from.
- The intent is explicit in the index predicate — no hidden "we null this field at delete time" footgun for someone reading the schema cold.
- One-shot migration: drop the old partial index, recreate with the extended predicate. The window where neither index exists is the only risk; on a low-traffic table like `medication_tracking_schedules` (one row per patient-medication, not high-velocity), it's negligible. If we ever decide it's not, we can run `CREATE INDEX CONCURRENTLY` manually before the migration.

`CreateScheduleValidator`'s existence check already uses `ModelRepository::getQuery(MedicationTrackingSchedule::class)`, which applies the default `SoftDeletes` scope, so non-trashed-only is automatic at the application layer too.

### Decision 3: `ScheduleDeleter` is a one-method service, not bolted into the controller

The actual logic — transaction, future-dose cleanup, soft-delete call — lives in `App\Services\MedicationTracking\Schedule\ScheduleDeleter::delete(MedicationTrackingSchedule $schedule): void`. The controller's `delete` method just calls the service and returns 204. Mirrors the shape of `ScheduleCreator` / `ScheduleIntakeController + UpdateScheduledDosePlannedQuantityAction` and keeps the controller thin per CLAUDE.md §8.

The service depends on `DatabaseManager`, `ModelRepository`, and `FactoryImmutable` (for `now`). All constructor-promoted `private readonly` per CLAUDE.md §3.

### Decision 4: Read paths must surface history; future unlogged rows are dropped at delete time, not at read time

The list endpoint (`GET /schedules`) and `GET /scheduled-doses/next` *do* hide soft-deleted schedules — they're "what's active right now" reads, and a trashed schedule isn't active.

The daily-view (`GET /scheduled-doses`) and tracking-logs (`GET /tracking-logs`) reads are different: they reflect **what actually happened**. A patient who took a dose yesterday and then deletes the schedule today still expects to see that intake in their day-view and in Log History. The delete already removed everything that wouldn't have happened (future unlogged doses are hard-deleted in the same transaction), so anything still in the dose / tracking-log tables for a trashed schedule is genuine history and MUST surface.

Two surgical fixes to bypass the `SoftDeletes` global scope on these read paths only:

- `ScheduledDoseController::index`: keep `joinRelationship(MEDICATION_TRACKING_SCHEDULE, fn ($join) => $join->withTrashed())`. The power-joins package's `PowerJoinClause::withTrashed()` strips the auto-added `deleted_at IS NULL` predicate from the join. Eager load: `with([RELATIONSHIP_MEDICATION_TRACKING_SCHEDULE => fn ($q) => $q->withTrashed()->with(RELATIONSHIP_PRODUCT)])` so the relation also returns trashed parents. Patient ownership is still scoped via `medication_tracking_schedules.patient_id`.
- `TrackingLogController::index`: in the `orWhereHas(medicationDose.medicationTrackingSchedule, ...)` callback, call `->withoutGlobalScope(SoftDeletingScope::class)` before the `patient_id` filter. This is critical for **custom** schedules (no `order_id`) — the alternative `whereHas(order)` branch can't match them, so without the scope-bypass their logs disappear from Log History entirely once the schedule is deleted.

`ScheduledDoseController::next` stays as-is. It only returns `type = 'scheduled'` rows whose `scheduled_at >= now`, and the delete transaction has already hard-removed all such rows for a trashed schedule, so the result is correct whether or not we bypass the scope.

The list endpoint `GET /schedules` does NOT bypass the scope — that endpoint is about "what's active in my tracker right now", and a trashed schedule isn't.

### Decision 5: Implicit binding does not include trashed — 404 on already-deleted

We deliberately do not add `withTrashed()` to the route binding. A DELETE on an already-trashed schedule returns 404, which is the correct REST semantic for an idempotent delete that you can no longer act on. If a caller needs to re-fetch state, they can hit `GET /schedules` and confirm absence.

## Risks / Trade-offs

- **Migration drops + recreates the partial index.** Two-step DDL on an existing table. On `medication_tracking_schedules` this table is small and write-quiet, so the brief window without the unique guarantee is acceptable. If/when this table grows, a manual `CREATE INDEX CONCURRENTLY` pre-step before deploy is the standard hedge.
- **No optimistic locking.** Two concurrent DELETEs on the same schedule will both succeed-ish: the first hard-deletes the future doses and sets `deleted_at`; the second finds no future doses (already gone) and sets `deleted_at` again (idempotent). No corruption, but no signal to the caller either. Same exposure as the rest of the repo's mutation endpoints. Accepted.
- **`tracking_logs` carry their own `order_id` and `product_id`**, set at log create time and never touched on delete. So Log History attribution survives the schedule's soft-delete without depending on the schedule row at all. Good — but worth noting that any future report that joins through `schedule.order_id` will need `withTrashed()` to read deleted-schedule history.
