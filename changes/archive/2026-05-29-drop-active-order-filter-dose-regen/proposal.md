## Why

The `medication-tracking:regenerate-doses` cron currently skips any schedule whose patient has no order in `OrderStatus::active()`. This "has an active order" heuristic is too coarse — it conflates "patient churned" with legitimate states (e.g. a patient between subscriptions, or on hold) and risks letting a live schedule's horizon silently stop being topped up. We are not confident this is the right gate, so we want to drop it for now and revisit with a better-defined eligibility rule later.

## What Changes

- Remove the per-patient active-order eligibility gate from the `medication-tracking:regenerate-doses` command — the `WHERE EXISTS (SELECT 1 FROM orders ... WHERE status IN (OrderStatus::active()))` semi-join predicate is dropped.
- The command now tops up the scheduled-dose horizon for **every** schedule passing the remaining filters (not soft-deleted, `schedule_type != as_needed`, `ends_on IS NULL OR ends_on >= today`), regardless of the patient's order status.
- The per-schedule window computation, dispatch logic, idempotency, and cron cadence are unchanged.

## Capabilities

### New Capabilities

_None._

### Modified Capabilities

- `medication-tracking-scheduler`: the "Periodic top-up of the scheduled-dose horizon" requirement drops the active-order eligibility gate from the command's filter set and its `WHERE EXISTS` query semantics.

## Impact

- `app/Console/Commands/MedicationTracking/RegenerateScheduledDosesCommand.php` — remove the active-order subquery and `whereExists(...)`; drop the now-unused `Order` / `OrderStatus` imports and the `$activeStatusValues` / `$patientHasActiveOrderSubquery` locals.
- Feature tests asserting the active-order filtering behavior need to be updated/removed.
- No DB schema, API, or queue-contract changes. More schedules may now be dispatched per run (a deliberate, reversible widening).
