## Context

The medication-tracking scheduler is already in production. This document captures the architectural decisions baked into the code under `App\…\MedicationTracking\…`, so the spec has a "how" companion that future changes can argue against.

Source code layout:
- HTTP: `app/Http/Controllers/Api/Patient/MedicationTracking/*` (5 controllers + 2 invokable in `TrackingLog/`)
- Services: `app/Services/MedicationTracking/Schedule/*`, `…/ScheduledDose/*`, `…/TrackingLog/*`
- Models: `app/Models/MedicationTracking/*`
- DTOs: `app/DTO/MedicationTracking/*`
- Resources: `app/Http/Resources/Api/MedicationTracking/*`
- Provider: `app/Providers/MedicationTrackingServiceProvider.php`
- Migrations: tables `medication_tracking_schedules`, `medication_tracking_schedule_intakes`, `medication_tracking_scheduled_doses`, plus widened columns on `tracking_logs`.

## Goals / Non-Goals

**Goals**
- Single source of truth for the patient-facing scheduling and dose-logging API as it exists today.
- Capture invariants that aren't visible from a casual reading of one controller — order-product locking, the 409 vs 422 split, transactional boundaries, forward-only quantity edits, off-schedule synthetic doses.
- Make future tickets (`PUT/DELETE /schedules`, async dose generation, retroactive edits) easy to slice against an explicit baseline.

**Non-Goals**
- No behavior changes.
- Not documenting the legacy `/medication-tracking/medication-schedules` routes (older `MedicationSchedule` model with `apply_times`).
- Not documenting `MedicationDoseReminder` and the dose-reminder notification pipeline.
- Not documenting the plain `/medication-tracking/tracking-logs` `index` / `create` / `update` / `delete` endpoints (these predate the scheduler and serve a broader "tracking log" capability).

## Decisions

### Decision 1: Intakes are a real `HasMany`, not embedded JSON

The previous design embedded intakes inside `schedule_rule` JSON. Two needs forced a relational table:

- The daily-view query joins doses to their owning intake so the FE can show the planned slot per item.
- `PATCH .../intakes/{intake}/planned-quantity` needs an addressable, immutable id.

`schedule_rule` therefore stores **only rule metadata** (`days_of_week`, `interval_days`, …) — never intakes. `every_day` and `as_needed` schedules persist `schedule_rule = {}`. The serialization shape is owned by `ScheduleRuleDto::toStorageArray()` and used by both DB writes and the response resource.

A partial unique index `(medication_tracking_schedule_id, time) WHERE time IS NOT NULL` guards against two timed intakes at the same time on one schedule. `as_needed` intakes are exempt by design — they have no `time`.

### Decision 2: Scheduled doses are pre-materialized synchronously, inside one transaction

`ScheduleCreator` runs `CreateScheduleValidator` **before** opening the transaction (so 422/409 never starts one), then inside `DatabaseManager::transaction(...)`:

1. inserts the schedule row,
2. bulk-inserts intake rows,
3. runs `ScheduledDoseGenerator::generate(...)`, which delegates to one of four strategies (`EveryDay` / `Weekly` / `EveryFewDays` / `AsNeeded`) and bulk-inserts dose rows in chunks of 1 000 via `insertOrIgnore`.

The dose horizon is `medication_tracking.dose_generation_horizon_days` (default 730). For an `every_day` schedule with 16 intakes that's ~11 680 rows synchronously per POST. This is a **known temporary trade-off** documented in `docs/features/SRD-1233-medication-tracking-api.md`; a follow-up change will move generation into a queued background job. The current spec describes the synchronous behavior so the future change can delta against it.

`AsNeededScheduledDoseStrategy` intentionally yields zero rows — as-needed schedules only get dose rows via the by-schedule off-schedule logging endpoint.

A partial unique index `(intake_id, scheduled_at) WHERE type = 'scheduled'` is what makes `insertOrIgnore` idempotent. `off_schedule` rows are not covered by the index, which is intentional — duplicates at the same `logged_at` are real events.

### Decision 3: Both `scheduled` and `off_schedule` events live in one table

`medication_tracking_scheduled_doses` carries a `type` enum (`scheduled` | `off_schedule`):

- `scheduled` rows are pre-materialized at create time as above.
- `off_schedule` rows are inserted on-the-fly by the by-schedule logging endpoint as a synthetic companion to each off-schedule `TrackingLog`. Their `intake_id` is `NULL`, `scheduled_at` equals the log's `logged_at`, `tracking_log_id` points at the new log.

This is **why** the daily-view query reads from one table for both: a single `LEFT JOIN` to `tracking_logs` plus an OR predicate (`logged_at` in day OR `scheduled` + unlogged + `scheduled_at` in day) returns everything in chronological order. No polymorphism, no UNION.

### Decision 4: Forward-only intake quantity edits

`PATCH .../planned-quantity` updates the intake's `quantity` AND every `scheduled`-type dose for the intake at or after the selected dose's `scheduled_at` **where `tracking_log_id IS NULL`**. Two `UPDATE`s in one transaction.

Past or logged doses are immutable by construction — there is no retroactive edit path today. This deliberately sidesteps re-running the dose horizon generator on every edit. A future retroactive-edit endpoint will be a separate capability change.

### Decision 5: Order-product locking flows through a strategy, not a switch

`ProductConstraintValidator` runs cross-cutting checks (name match, exactly one intake, locked `DoseUnit`) and then dispatches to a per-product strategy:

- `PillsProductConstraintStrategy` → locks to `every_day`, applies `PlannedQuantityForProductType`.
- `InjectionProductConstraintStrategy` → locks to `weekly`, requires `days_of_week.length == 1`.
- `DropsProductConstraintStrategy` → locks to `every_day`.

The same `PlannedQuantityForProductType` rule is **reused** by `UpdatePlannedQuantityRequest`, so per-product-type quantity bounds cannot drift between create and edit paths.

The `Order` and `Product` entities are looked up by the controller (`ScheduleController::resolveOrder(...)`), scoped to the patient + `OrderStatus::active()`, eager-loaded with `product`, and carried through `CreateScheduleDto`. The validator and persistence layer never re-query.

### Decision 6: Existence/ownership checks live in the controller, not in custom rules

Three custom rules were removed in favor of controller-side `first()` + `ValidationException::withMessages([...])`:

- `ValidScheduleOrder` → `ScheduleController::resolveOrder(...)`
- `ValidMedicationDose` → `CreateByDoseController` resolves the dose
- `ValidMedicationTrackingSchedule` → `CreateByScheduleController` resolves the schedule

The FormRequest layer is now focused on shape validation; existence and ownership are checked where the resolved entity is needed anyway. FormRequests expose plain `getOrderId(): ?int`, `getMedicationDoseId(): int`, `getMedicationTrackingScheduleId(): int`.

### Decision 7: `logged_at` is strict ISO 8601 with offset

Both by-dose and by-schedule require `logged_at` matching `Y-m-d\TH:i:sP` (e.g. `2026-05-15T08:05:00-04:00`). The DTO carries a typed `?CarbonImmutable` parsed by Laravel's `$this->date('logged_at')?->toImmutable()`. The previous `loggedAtAsString: ?string` and `TimezoneNormalizer` plumbing have been removed from the by-dose / by-schedule flow because the offset is implied by the input string.

(The legacy `TrackingLogController` `create`/`update` endpoints still use `TimezoneNormalizer` and a `?string` field — they are out of scope for this capability.)

### Decision 8: Configuration is read in the provider, not in services

`MedicationTrackingServiceProvider` resolves both knobs at boot:

- `medication_tracking.dose_generation_horizon_days` (default 730) → `ScheduledDoseGenerator(horizonDays: ...)`
- `medication_tracking.schedule_limit_per_patient` (default 50) → `CreateScheduleValidator(scheduleLimitPerPatient: ...)`

Both are asserted as positive integers via `Webmozart\Assert\Assert::positiveInteger(...)` before binding. Services receive primitive typed values via named-argument constructor calls and never call `config(...)` themselves — per `CLAUDE.md` §16. The provider implements `DeferrableProvider` and lists both bindings in `provides()`.

### Decision 9: Known anti-pattern flagged but not fixed in this baseline

`ScheduleResource::getImagePath()` resolves the image URL via `app()->get(Generator::class)` — service location. The feature doc flags it as "a known anti-pattern per `CLAUDE.md` — flag for cleanup when the dose-generation refactor lands." The spec documents the image-path behavior; the cleanup is intentionally out of scope for this baseline change.

## Risks / Trade-offs

- **Synchronous dose materialization**: every `every_day` Rx-linked schedule costs ~11 680 INSERTs at the default horizon. Acceptable now (the feature ships); a queued background job is on the roadmap. The spec describes the synchronous semantics so the future change is a clear delta.
- **Off-schedule duplicates**: two by-schedule logs at the same `logged_at` produce two dose rows. Intentional — "I forgot, I dosed again" is two events. A future cleanup change might want to add a soft-dedup, but the spec preserves the current behavior.
- **No edit/delete on schedules yet**: covered by an explicit out-of-scope note in the proposal and feature doc. The next ticket will need to define the side-effects on already-materialized doses (regen vs. soft-delete vs. leave-as-is) — that's a separate proposal.
