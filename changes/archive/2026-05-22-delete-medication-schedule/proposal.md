## Why

[SRD-1169](https://skinnyrx.atlassian.net/browse/SRD-1169) — slice 1 of "Manage and delete medications". This change adds **delete only**; edit, async dose generation, and the monthly backfill cron are deliberately deferred to separate changes so each lands as a focused, reviewable slice.

Today there is no way for a patient to remove a schedule from their tracker. The current baseline (`openspec/specs/medication-tracking-scheduler/spec.md`) explicitly lists `DELETE /schedules/{schedule}` as out of scope.

History is sacred: deletion MUST NOT remove past doses or any dose that already has a `tracking_log`. The Log History feature relies on those rows surviving.

## What Changes

- Add `DELETE /api/v1/patient/medication-tracking/schedules/{schedule}` (auth: `can:manage,schedule`).
- Use Eloquent `SoftDeletes` on `MedicationTrackingSchedule` — the row survives so `tracking_logs → medication_tracking_scheduled_doses → schedule` joins for Log History keep working.
- Inside one DB transaction: delete **future, unlogged** `scheduled`-type doses (`scheduled_at >= now AND type = 'scheduled' AND tracking_log_id IS NULL`); leave logged and off-schedule rows untouched; set `deleted_at`.
- Migrate the existing partial unique index `(patient_id, order_id) WHERE order_id IS NOT NULL` to also require `deleted_at IS NULL`, so a patient who deletes an Rx schedule can re-create one for the same order. Keeps the original `order_id` link on the deleted row (useful for reports).
- `GET /schedules`, `GET /scheduled-doses`, `GET /scheduled-doses/next` automatically stop surfacing soft-deleted schedules thanks to the global `SoftDeletes` scope and the existing joins through `medication_tracking_schedules`.

## Capabilities

### New Capabilities

_None — this change extends the existing `medication-tracking-scheduler` capability._

### Modified Capabilities

- `medication-tracking-scheduler`:
  - **ADDED**: "Soft-delete a medication schedule" requirement with full preservation/exclusion scenarios.
  - **MODIFIED**: "Reject duplicate schedules for the same order" — clarify that soft-deleted schedules are excluded from the uniqueness check (via the migrated partial index predicate).

## Impact

### Code

- `app/Http/Controllers/Api/Patient/MedicationTracking/ScheduleController.php` — add `delete(...)` method.
- `app/Services/MedicationTracking/Schedule/ScheduleDeleter.php` (new) — orchestrates the delete transaction.
- `app/Models/MedicationTracking/MedicationTrackingSchedule.php` — add `use SoftDeletes;` and `FIELD_DELETED_AT` constant.
- `routes/api/api.php` — add the DELETE route inside the existing `can:manage,schedule` group.
- Migration: add `deleted_at` column + index on `medication_tracking_schedules`, and recreate the partial unique index with `... AND deleted_at IS NULL`.

### API

- `DELETE /api/v1/patient/medication-tracking/schedules/{schedule}` — new, `204` on success, `403` for non-owner, `404` for already-deleted, `401` unauthenticated.
- `GET /schedules` — wire-compatible; soft-deleted rows are no longer returned.
- `GET /scheduled-doses` / `next` — wire-compatible; future unlogged doses of deleted schedules are gone, historical logged rows remain visible on the day they were logged.

### Operational

- No new env vars. No new queues. No new cron jobs.
- One migration on an existing table — adds nullable column and recreates one partial index. Safe to run online (PostgreSQL `CREATE INDEX CONCURRENTLY` is preferred but the migration helper doesn't expose it; if traffic on this table is heavy, run it manually as a follow-up).

## Out of Scope (deferred to separate proposals)

- **Edit a medication schedule** (`PUT /schedules/{schedule}`) — separate change.
- **Async dose generation refactor** (14-day sync + 1-year async + monthly backfill cron) — separate change. We keep today's synchronous 730-day generator behavior.
- **Untake** (undo a `taken` intake) — separate change, lives under the Log History work.
- **My Medications screen** layout — FE-only.
- **SkinnyRx subscription banner** re-appearing after delete of an Rx-linked schedule — handled by the existing subscription service; this change just makes sure `GET /schedules` omits the deleted row so the banner-decision query returns the right state.
