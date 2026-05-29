## Why

When a patient submits two intakes with the same `time` (e.g. both at `09:00`) while creating or editing a medication tracking schedule, the request currently bypasses FormRequest validation, reaches `CreateScheduleIntakesAction::execute(...)`, and dies on the Postgres unique constraint `medication_tracking_schedule_intakes_schedule_id_time_unique` with a `SQLSTATE[23505]` — surfacing to the patient as a generic 500. The DB constraint is the right safety net, but it MUST NOT be the patient-facing error path.

## What Changes

- Reject duplicate `time` values in `schedule_rule.intakes[*]` at the FormRequest layer for `EVERY_DAY`, `WEEKLY`, and `EVERY_FEW_DAYS` schedules, so a duplicate produces a typed 422 validation error keyed on `schedule_rule.intakes.<index>.time`.
- Applies to both the create and edit endpoints — the rule is added in the shared `ResolvesScheduleRequest` trait so both inherit it.
- `AS_NEEDED` is unaffected: that schedule type allows exactly one intake and prohibits `time`, so duplicates cannot arise.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `medication-tracking-scheduler`: adds two scenarios — one under "Create a custom medication schedule" and one under "Edit a medication schedule" — that pin the new "duplicate intake `time` → 422" behavior; the DB unique index remains the authoritative race guard.

## Impact

- Code: `app/Http/Requests/Api/Patient/MedicationTracking/Schedule/Concerns/ResolvesScheduleRequest.php` (rule additions only).
- Tests: feature tests for the create and update endpoints under `tests/Feature/Api/Patient/MedicationTracking/Schedule/` get coverage for the duplicate-time path.
- No DB migration, no DTO change, no service change. The action layer continues to rely on the unique index as the final guard for concurrent races.
- No effect on order-linked schedules in practice — they are constrained to a single intake by `ProductConstraintValidator`, so `distinct` is a no-op there.
