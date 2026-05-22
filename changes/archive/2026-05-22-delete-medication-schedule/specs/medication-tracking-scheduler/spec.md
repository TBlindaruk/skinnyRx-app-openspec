## ADDED Requirements

### Requirement: Soft-delete a medication schedule

The system SHALL expose `DELETE /api/v1/patient/medication-tracking/schedules/{schedule}` so the owning patient can remove a schedule from their tracker. Authorization MUST go through the `can:manage,schedule` policy. The implicit route binding MUST NOT resolve trashed rows, so deleting an already-deleted schedule returns `404`.

The operation MUST run inside one `DatabaseManager::transaction`:
1. Compute `now` once.
2. Delete future, unlogged `scheduled`-type doses: `scheduled_at >= now AND type = 'scheduled' AND tracking_log_id IS NULL` AND `medication_tracking_schedule_id = schedule.id`.
3. Soft-delete the schedule via Eloquent's `SoftDeletes` trait (sets `deleted_at`).

Preserved by design (untouched by the delete):
- Past `scheduled`-type doses (`scheduled_at < now`).
- Any `scheduled`-type dose with `tracking_log_id IS NOT NULL` (already logged), regardless of `scheduled_at`.
- All `off_schedule` doses (they are synthetic companions to tracking logs).
- All `tracking_logs` referencing the schedule.

The response SHALL be `204 No Content` with an empty body.

#### Scenario: Patient soft-deletes their schedule
- **WHEN** the owner DELETEs a non-deleted schedule
- **THEN** the system SHALL respond `204`
- **AND** the schedule's `deleted_at` SHALL be set
- **AND** every `scheduled`-type dose with `scheduled_at >= now AND tracking_log_id IS NULL` SHALL be removed from `medication_tracking_scheduled_doses`
- **AND** every dose with `tracking_log_id IS NOT NULL` SHALL remain, regardless of `scheduled_at`
- **AND** every `off_schedule` dose SHALL remain
- **AND** every `tracking_log` referencing the schedule SHALL remain

#### Scenario: List omits soft-deleted schedules
- **WHEN** the patient GETs `/schedules` after deleting a schedule
- **THEN** the deleted schedule SHALL NOT appear in the result
- **AND** the same id passed via `?id[]=...` SHALL also NOT appear

#### Scenario: Daily scheduled-doses view surfaces past logged doses of the deleted schedule
- **WHEN** the patient GETs `/scheduled-doses?date=...&timezone=...` for a day on which the deleted schedule had logged doses
- **THEN** the response SHALL include those logged dose rows with their nested `tracking_log` (so Log History reflects what actually happened that day)
- **AND** the response SHALL NOT include any `scheduled`-type unlogged rows for the deleted schedule (they were hard-deleted on delete)
- **AND** to make this possible the daily-view query's join against `medication_tracking_schedules` MUST bypass the model's `SoftDeletes` global scope (e.g. `joinRelationship('relation', fn ($join) => $join->withTrashed())`), and the eager-loaded schedule relation MUST also use `withTrashed()`

#### Scenario: Tracking-logs index surfaces logs whose schedule was deleted
- **WHEN** the patient GETs `/tracking-logs` after deleting a schedule (custom or order-linked) that had logs
- **THEN** the response SHALL include those tracking logs
- **AND** the `orWhereHas` branch through `medicationDose.medicationTrackingSchedule.patient_id` MUST disable the `SoftDeletes` global scope (`->withoutGlobalScope(SoftDeletingScope::class)`) so logs of trashed schedules — especially custom ones with no `order_id` to match via the order branch — keep surfacing in Log History

#### Scenario: Next-dose view skips deleted schedules
- **WHEN** the patient GETs `/scheduled-doses/next` after deleting their only upcoming schedule
- **THEN** the response SHALL be `204 No Content`

#### Scenario: Deleting an already-deleted schedule
- **WHEN** the patient DELETEs a schedule that is already soft-deleted
- **THEN** the route's implicit binding SHALL NOT resolve the trashed row and the system SHALL respond `404`

#### Scenario: Different patient tries to delete
- **WHEN** an authenticated patient DELETEs a schedule they do not own
- **THEN** `can:manage,schedule` SHALL return `403`
- **AND** the target schedule and its doses SHALL be unchanged

#### Scenario: Unauthenticated request
- **WHEN** the request has no valid patient session
- **THEN** the system SHALL respond `401`

#### Scenario: Delete transaction fails mid-flight
- **WHEN** any step inside the delete transaction throws
- **THEN** the transaction SHALL roll back; the schedule SHALL NOT be trashed and the future doses SHALL NOT be removed

## MODIFIED Requirements

### Requirement: Reject duplicate schedules for the same order

The system SHALL allow at most one **active** schedule per `(patient_id, order_id)` pair, where "active" means the schedule row is not soft-deleted. Two different orders for the same product are allowed. Enforcement happens both in `CreateScheduleValidator` (pre-transaction lookup, scoped to non-trashed rows via Eloquent's default `SoftDeletes` scope) and at the database level via the partial unique index `(patient_id, order_id) WHERE order_id IS NOT NULL AND deleted_at IS NULL`. The DB index is the authoritative race guard.

#### Scenario: Patient already has an active schedule for the given order
- **WHEN** the patient POSTs a schedule with an `order_id` for which they already own a non-deleted schedule
- **THEN** the system SHALL respond `409 Conflict` with a `MedicationTrackingScheduleAlreadyExistsException` message

#### Scenario: Two concurrent creates race for the same order
- **WHEN** two requests with the same `(patient_id, order_id)` arrive concurrently and both pass the validator
- **THEN** the database partial unique index SHALL make one of them fail and the second response SHALL be `409`

#### Scenario: Patient creates schedules for the same product across two different orders
- **WHEN** the patient has two distinct active orders for the same product and POSTs a schedule for each
- **THEN** the system SHALL respond `201` for both — the `(patient_id, order_id)` uniqueness is per-order, not per-product

#### Scenario: Patient re-creates a schedule for an order whose previous schedule was deleted
- **WHEN** the patient previously had a schedule for `order_id = X`, soft-deleted it, and now POSTs a new schedule for the same `order_id = X`
- **THEN** the system SHALL respond `201` — the partial unique index excludes the trashed row via `deleted_at IS NULL`, and the validator's existence check uses the default `SoftDeletes` scope which also excludes it
- **AND** both schedule rows (the trashed one and the new one) SHALL coexist; the trashed one keeps its `order_id` for historical reporting
