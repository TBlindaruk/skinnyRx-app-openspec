## 1. Per-schedule queued job

- [x] 1.1 **No new job class — reuse `App\Jobs\MedicationTracking\GenerateScheduledDosesJob`.** Its existing `(scheduleId, localWindowStartIso, maxDays)` constructor is exactly the cron's needed shape; its existing null-schedule / `AS_NEEDED` / empty-window guards cover all cron race cases.

## 2. Artisan command

- [x] 2.1 Create `app/Console/Commands/MedicationTracking/RegenerateScheduledDosesCommand.php` as `final class … extends Illuminate\Console\Command`.
- [x] 2.2 Define `$signature = 'medication-tracking:regenerate-doses'` (no arguments / options — the cron and operator runs are identical) and a short `$description`.
- [x] 2.3 `handle(...)` takes via parameter DI: `ModelRepository`, `Illuminate\Contracts\Bus\Dispatcher`, `FactoryImmutable`, `LoggerInterface`.
- [x] 2.4 Build the eligibility query as a single statement: `whereExists` on `orders` correlated via `patient_id` with `whereIn(orders.status, OrderStatus::active())` (semi-join, no `DISTINCT` needed), plus `where(schedule_type != as_needed)` and `where(ends_on IS NULL OR ends_on >= today)` (plain `where`, not `whereDate`, because `ends_on` is already a `date` column). Add a correlated `selectSub` aliased `last_scheduled_at` returning `MAX(scheduled_at) WHERE schedule_id = mts.id AND type = 'scheduled'`.
- [x] 2.5 Stream via `->cursor()` and, for each row, compute the per-schedule window: `localToday = clock->now()->setTimezone(tz)->startOfDay()`, `endOfHorizon = localToday + ASYNC_HORIZON_DAYS`, `startsOnLocal = $schedule->getLocalStartsOn()`, `dayAfterLastDose = startOfDay(last_scheduled_at parsed in app default TZ, then setTimezone(tz))->addDay()`. Take `localWindowStart = max(startsOnLocal, dayAfterLastDose)`. Skip the schedule if `localWindowStart >= endOfHorizon` (returned as `maxDays = 0` from the helper). Otherwise dispatch `new GenerateScheduledDosesJob($id, $localWindowStart->toIso8601String(), $localWindowStart->diffInDays($endOfHorizon))`. Track dispatched count and emit one summary log line at the end (`dispatched_count`).
- [x] 2.6 Return `Command::SUCCESS`.

## 3. Scheduler wiring

- [x] 3.1 In `app/Console/Kernel.php::schedule(...)`, add `$schedule->command(RegenerateScheduledDosesCommand::class)->monthlyOn(1, '02:00')->onOneServer()->withoutOverlapping()`. Place it grouped with the other medication-tracking commands (if any) or near the existing `monthly`/maintenance entries.

## 4. Tests — Job

- [x] 4.1 **No new job-class tests needed** — `GenerateScheduledDosesJob`'s existing behavior is already covered by the medication-tracking-scheduler spec's "Asynchronous dose tail via queued job" requirement and its tests. The cron's reliance on the generator's empty-window / `AS_NEEDED` short-circuits is asserted indirectly via the command tests.

## 5. Tests — Command

- [x] 5.1 `tests/Feature/Console/Commands/MedicationTracking/RegenerateScheduledDosesCommandTest.php`: fake the bus, run the command, and assert the exact set of `GenerateScheduledDosesJob` dispatches for: (a) eligible live schedules of patients with at least one active order, (b) `as_needed` schedules (skipped), (c) trashed schedules (skipped), (d) schedules with past `ends_on` (skipped), (e) schedules of patients with no orders (skipped), (f) schedules of patients with only inactive orders (skipped), (g) custom schedule of a patient with any active order (dispatched), (h) order-linked schedule whose own order is inactive but whose patient has any other active order (dispatched), (i) patient with many active orders → schedule dispatched exactly once (DISTINCT collapses the join). Plus window-math scenarios: (j) no doses yet → start from `starts_on`, fill to today+365; (k) resume from last generated dose's local date when one exists; (l) materialized horizon already past today+365 → no dispatch; (m) future-dated `starts_on` → start at `starts_on` (not today).

## 6. Tests — Scheduling cadence

- [x] 6.1 Add a small test under `tests/Feature/Console/Schedule/RegenerateScheduledDosesScheduleTest.php` that loads the kernel schedule and asserts a `RegenerateScheduledDosesCommand` entry exists with `monthlyOn(1, '02:00')` expression (`0 2 1 * *`).

## 7. Static analysis & docs

- [x] 7.1 Run `make run-pint && make run-phpcs && make run-phpstan && make run-psalm`. Fix any new issues directly — DO NOT append to either baseline.
- [x] 7.2 Run `make code-test` (or at minimum the medication-tracking suite) to confirm both new tests pass and existing tests are unaffected.
- [x] 7.3 Update `openspec/specs/medication-tracking-scheduler/README.md` only if it lists capabilities / cron jobs; otherwise leave it alone (archive step will fold the new requirement into `spec.md`).

## 8. Validation pass before PR

- [x] 8.1 `openspec validate monthly-dose-regeneration` returns success.
- [x] 8.2 Manual smoke: in a local environment with a seeded live schedule, run `php artisan medication-tracking:regenerate-doses`, observe the queue picks up the job, verify new `medication_tracking_scheduled_doses` rows exist out to `today + 365 days` (or `ends_on`), and verify a second run produces zero new rows.
