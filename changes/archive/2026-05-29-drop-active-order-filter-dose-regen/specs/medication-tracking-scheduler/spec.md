## MODIFIED Requirements

### Requirement: Periodic top-up of the scheduled-dose horizon

The system SHALL run a monthly background job that re-materializes the `scheduled`-type dose horizon for every eligible `medication_tracking_schedule`, so the rolling year of pre-materialized doses stays current without depending on schedule writes.

The system SHALL define an artisan command `medication-tracking:regenerate-doses` that, when invoked, iterates every schedule matching ALL of the following:

- not soft-deleted (the default `SoftDeletes` global scope MUST apply);
- `schedule_type != as_needed`;
- `ends_on IS NULL` OR `ends_on >= today` (server-time UTC date is acceptable for this coarse filter — the per-schedule job re-evaluates in the schedule's timezone).

The command SHALL NOT filter schedules by the patient's order status. Eligibility is a property of the schedule alone (type / `ends_on` / not-trashed); a schedule is topped up regardless of whether its patient has any order in `OrderStatus::active()`. (A patient-state eligibility gate may be reintroduced later under a better-defined rule; it is intentionally absent for now.)

The command SHALL stream the rows via `ModelRepository::getQuery(...)->cursor()`. The eligibility query SHALL be a single SQL statement applying only the type / `ends_on` / soft-delete predicates above, with no semi-join against `orders`.

The same SELECT SHALL include a correlated subquery aliased `last_scheduled_at` returning `MAX(medication_tracking_scheduled_doses.scheduled_at) WHERE medication_tracking_schedule_id = medication_tracking_schedules.id AND type = 'scheduled'`, so the cron does not need a second round-trip per schedule to discover where the previous materialization left off.

For each row the cursor yields, the command SHALL compute the per-schedule window:

- `localToday = clock->now()->setTimezone(schedule.timezone)->startOfDay()`;
- `endOfHorizon = localToday + MedicationTrackingConfig::ASYNC_HORIZON_DAYS` (the fixed "365 days from today" target);
- `startsOnLocal = startOfDay(schedule.starts_on) in schedule.timezone`;
- `dayAfterLastDose = startOfDay(last_scheduled_at parsed in the app's default timezone — which `DoseRowFactory` writes in via `setTimezone(factory->now()->getTimezone())->toDateTimeString()`) converted to `schedule.timezone`, then `addDay()`; or null if no scheduled-type dose row exists for this schedule yet. We advance one full day so the generator does not waste work re-evaluating the day of the last existing dose (which `insertOrIgnore` would silently skip anyway);
- `localWindowStart = max(startsOnLocal, dayAfterLastDose)` (resume from the day AFTER the last materialized dose, but never earlier than `starts_on` — for future-dated schedules this naturally pushes the start past today);
- if `localWindowStart >= endOfHorizon` (the materialized horizon already extends to or past today + 365), the cron SHALL skip the schedule with no dispatch;
- otherwise dispatch exactly one `App\Jobs\MedicationTracking\GenerateScheduledDosesJob` with `(scheduleId, localWindowStartIso = localWindowStart->toIso8601String(), maxDays = (int) localWindowStart->diffInDays(endOfHorizon))`.

To support these per-schedule computations the cursor SHALL hydrate at minimum the `id`, `timezone`, and `starts_on` columns plus the `last_scheduled_at` alias. The `ends_on >= today` predicate SHALL be plain `where(...)` (not `whereDate(...)`) since `ends_on` is already a `date` column and the implicit `::date` cast that `whereDate` adds would needlessly hide the column from any future index.

The system SHALL register the command in `App\Console\Kernel::schedule()` with cadence `monthlyOn(1, '02:00')` (UTC, 1st day of month), `onOneServer()`, and `withoutOverlapping()`.

The cron deliberately reuses the existing `GenerateScheduledDosesJob` defined by the "Asynchronous dose tail via queued job" requirement above — no second job class is introduced. The defensive race-case behavior the cron relies on is already covered by that requirement:

- the job re-fetches the schedule and exits cleanly when the row is missing (covers "schedule deleted between dispatch and run", because `whereKey()->first()` ignores trashed rows by default);
- `ScheduledDoseGenerator` returns immediately when `schedule_type === AS_NEEDED`;
- `ScheduleLocalWindowResolver` produces an empty window when `ends_on < localWindowStart`, and the generator returns on `$window->isEmpty()` (covers "schedule expired between dispatch and run").

The job is therefore idempotent under cron re-runs: the partial unique index `(intake_id, scheduled_at) WHERE type = 'scheduled'` together with `insertOrIgnore` makes overlapping windows produce zero net new rows. The job MUST NOT modify any existing dose row and MUST NOT delete rows — properties already guaranteed by the existing job spec.

The artisan command takes no arguments or options — it always operates over every eligible schedule. Operator-driven runs use the same `php artisan medication-tracking:regenerate-doses` invocation as the cron.

#### Scenario: Monthly cron tops up a long-running open-ended schedule by resuming from the last generated dose

- **WHEN** a non-trashed, non-`as_needed` schedule has `ends_on=null`, `starts_on=2026-01-15`, and doses materialized up to `2027-01-15` (last generated dose local date)
- **AND** the monthly cron fires on `2026-09-01 02:00 UTC`
- **THEN** the command SHALL compute `localWindowStart = 2027-01-16 00:00 in schedule.timezone` (the day AFTER the last generated dose's local date — strictly greater than `starts_on`)
- **AND** `endOfHorizon = 2026-09-01 + 365 days` in the schedule's timezone
- **AND** dispatch one `GenerateScheduledDosesJob` for the schedule with that `localWindowStartIso` and `maxDays = diffInDays(localWindowStart, endOfHorizon)` (≈ 242 days, not 365)
- **AND** when the worker handles the job, dose rows SHALL be inserted for every previously-unmaterialized date in `[2027-01-16, 2027-09-01]` according to the schedule's strategy
- **AND** the generator SHALL NOT re-evaluate the day of the last existing dose — it is excluded from the window entirely

#### Scenario: First cron run for a schedule with no doses yet starts from `starts_on`

- **WHEN** an eligible schedule has `starts_on = 2026-05-10`, no `scheduled`-type dose rows yet (the create-time `GenerateScheduledDosesJob` failed or hasn't run), and today is `2026-05-15`
- **THEN** the `last_scheduled_at` subquery SHALL return `NULL`
- **AND** the command SHALL compute `localWindowStart = startsOnLocal = 2026-05-10 00:00 in schedule.timezone` and `maxDays = (2026-05-15 + 365) - 2026-05-10 = 370`
- **AND** dispatch the job with those parameters — the generator will materialize the full schedule from `starts_on` to `today + 365`

#### Scenario: Future-dated schedule starts at `starts_on`, not today

- **WHEN** an eligible schedule has `starts_on = today + 5 days` and no doses yet
- **THEN** `localWindowStart` SHALL equal `startsOnLocal` (which is strictly greater than `localToday`)
- **AND** `maxDays = endOfHorizon - starts_on = 365 - 5 = 360`
- **AND** the dispatched job SHALL generate doses for `[starts_on, today + 365]` — never for dates earlier than `starts_on`

#### Scenario: Materialized horizon already extends past today + 365 — no dispatch

- **WHEN** an eligible schedule has its last `scheduled`-type dose at `today + 700` (e.g. because a prior cron ran with a longer horizon or an edit pre-materialized further)
- **THEN** `localWindowStart` SHALL be `today + 701` (day after last dose) which is `>= endOfHorizon = today + 365`
- **AND** the command SHALL skip the schedule entirely — no `GenerateScheduledDosesJob` SHALL be dispatched for it

#### Scenario: Monthly cron clips at `ends_on`

- **WHEN** a schedule has `ends_on = today + 60 days` and last `scheduled`-type dose at `today + 60 days`
- **AND** the cron dispatches and the job runs
- **THEN** `localWindowStart = today + 61` (day after last dose), `endOfHorizon = today + 365`, `maxDays = 304`
- **AND** the generator passes those into `ScheduleLocalWindowResolver`, which clamps the window's `end` to `ends_on + 1 day = today + 61` because `ends_on < windowStart + maxDays`
- **AND** the resulting window is `[today + 61, today + 61)` — empty — so `ScheduleLocalWindow::isEmpty()` is true and the generator returns immediately
- **AND** zero new dose rows SHALL be created — no dose beyond `ends_on` exists

#### Scenario: Cron skips `as_needed` schedules

- **WHEN** the command iterates schedules and encounters an `as_needed` schedule
- **THEN** the command SHALL NOT dispatch a job for it (the SQL filter excludes it)
- **AND** zero `scheduled`-type dose rows SHALL be created for `as_needed` schedules regardless of how often the cron fires

#### Scenario: Cron skips schedules whose `ends_on` is in the past

- **WHEN** a schedule has `ends_on = yesterday`
- **THEN** the command SHALL NOT dispatch a job for it
- **AND** no dose row SHALL be added for that schedule

#### Scenario: Cron dispatches for an eligible schedule even when the patient has no active order

- **WHEN** a patient owns a non-trashed, non-`as_needed` schedule with `ends_on` null or `>= today`, and zero orders whose `status` is in `OrderStatus::active()` (every order is `canceled` / `on_hold` / etc., or the patient has no orders at all)
- **THEN** the command SHALL still dispatch a `GenerateScheduledDosesJob` for that schedule — the patient's order status is no longer part of the eligibility filter
- **AND** the same holds for both custom (`order_id IS NULL`) and order-linked schedules

#### Scenario: Schedule is soft-deleted between dispatch and run

- **WHEN** the command dispatches a job for schedule `X`
- **AND** before the worker handles the job, the patient deletes the schedule (its `deleted_at` is set)
- **THEN** `GenerateScheduledDosesJob::handle()` SHALL resolve the schedule via `whereKey()->first()` (which respects the `SoftDeletes` global scope), see `null`, and exit cleanly
- **AND** the job SHALL NOT insert any dose rows
- **AND** the job SHALL NOT be marked failed and SHALL NOT emit a Sentry error

#### Scenario: Schedule expires between dispatch and run

- **WHEN** the command dispatches a job, and between dispatch and worker pickup the schedule's `ends_on` is updated by an edit to a date strictly before `localToday`
- **THEN** `ScheduleLocalWindowResolver::clampEnd` SHALL produce a window where `end <= start`, `ScheduledDoseGenerator` SHALL detect the empty window and return immediately, and the job SHALL exit cleanly without inserting rows
- **AND** the job SHALL NOT be marked failed

#### Scenario: Job is idempotent under re-run

- **WHEN** the cron dispatches `GenerateScheduledDosesJob` twice in a row for the same schedule with no intervening changes
- **THEN** the first run inserts the missing tail (possibly zero rows if already current)
- **AND** the second run inserts zero net new rows because `insertOrIgnore` plus the partial unique index `(intake_id, scheduled_at) WHERE type = 'scheduled'` collapse the overlap
