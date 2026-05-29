## 1. Throttle marker on patients

- [x] 1.1 Add a migration adding nullable `dose_horizon_refreshed_at` timestamp to `patients` (anonymous-class form, table/column constants, no backfill).
- [x] 1.2 Add `dose_horizon_refreshed_at` to `Patient::$casts` as `immutable_datetime` and expose `getDoseHorizonRefreshedAt(): ?CarbonImmutable` (mirror `HasActivity::getLastActiveAt()`; add a `FIELD_DOSE_HORIZON_REFRESHED_AT` constant).

## 2. Config knob

- [x] 2.1 Add `HORIZON_REFRESH_THROTTLE_DAYS = 7` to `MedicationTrackingConfig`.

## 3. Per-patient top-up job

- [x] 3.1 Add `RefreshPatientDoseHorizon` (extends `App\Jobs\Job`) taking `(int $patientId, CarbonImmutable $refreshedAt)`; queue it appropriately.
- [x] 3.2 In `handle()`, load the patient's eligible schedules (not-trashed, `!= as_needed`, `ends_on` null or `>= today`) via `ModelRepository::getQuery(...)`, with the `last_scheduled_at` MAX subquery.
- [x] 3.3 For each schedule compute the per-schedule window (localToday / endOfHorizon / startsOnLocal / dayAfterLastDose / localWindowStart) and, when `localWindowStart < endOfHorizon`, extend via `ScheduledDoseGenerator`. Reuse the existing window logic rather than duplicating it.
- [x] 3.4 After processing, set `patient.dose_horizon_refreshed_at = $this->refreshedAt` through `ModelRepository::update(...)`.

## 4. Middleware trigger

- [x] 4.1 Add `EnsureDoseHorizon` middleware (`final readonly`, constructor DI: `Guard`, `Dispatcher`, `FactoryImmutable`) mirroring `TrackActivity`.
- [x] 4.2 In `handle()`, resolve the patient from the guard, read `getDoseHorizonRefreshedAt()` (no extra query), and dispatch `RefreshPatientDoseHorizon` only when null or older than `now - HORIZON_REFRESH_THROTTLE_DAYS`; always call `$next` without blocking.
- [x] 4.3 Register the middleware (alias in `Http\Kernel` if needed) and attach it to the `/medication-tracking` route group in `routes/api/api.php`.

## 5. Remove the cron

- [x] 5.1 Remove the `RegenerateScheduledDosesCommand::class` `$schedule->command(...)` registration from `App\Console\Kernel`.
- [x] 5.2 Delete `app/Console/Commands/MedicationTracking/RegenerateScheduledDosesCommand.php`.
- [x] 5.3 Delete `RegenerateScheduledDosesCommandTest` (and any helpers it alone uses).

## 6. Tests

- [x] 6.1 Middleware test: stale/null marker dispatches exactly one job; fresh marker dispatches none; response is not blocked. Use `Bus`/dispatcher fake and the test clock.
- [x] 6.2 Middleware boundary test around the 7-day throttle (just-inside no dispatch, just-outside dispatch).
- [x] 6.3 Job test: extends short horizons up to `today + ASYNC_HORIZON_DAYS`, skips `as_needed` / expired schedules, is idempotent on re-run (zero net new rows), and stamps `dose_horizon_refreshed_at`.
- [x] 6.4 Job test: patient with all-current horizon gets zero new rows but marker still updated.
- [x] 6.5 Confirm a patient who never hits the tracker group gets no job dispatched (covered via middleware test scope).

## 7. Verification

- [x] 7.1 Run the medication-tracking suite in the container (`make code-test` filtered) — green.
- [x] 7.2 `make run-phpstan` and `make run-psalm` — no new findings, baselines not grown.
- [x] 7.3 `make fix-pint` — no style drift.
- [x] 7.4 `openspec validate refresh-dose-horizon-on-tracker-access` passes.
