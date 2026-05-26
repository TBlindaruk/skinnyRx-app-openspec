## Context

The medication-tracking scheduler currently materializes `scheduled`-type dose rows only at **write** time:

- `ScheduleCreator` inserts the first `sync_horizon_days` (14) of doses inside the create transaction, then dispatches `GenerateScheduledDosesJob` to fill the **async tail** to `min(ends_on, starts_on + async_horizon_days)` (365 days).
- `ScheduleEditor` does the same on edit, starting from `now`.
- The job's body is just `ScheduledDoseGenerator::generate($schedule, $from->addDays(SYNC_HORIZON_DAYS), $maxDays = ASYNC_HORIZON_DAYS - SYNC_HORIZON_DAYS)` — generation always clips to `ends_on` internally via `ScheduleLocalWindowResolver`.

This means the horizon is **frozen at the last write**. For a schedule with no `ends_on` (open-ended) created on 2026-01-15, doses exist up to ~2027-01-15; on 2026-09-01 the patient still only sees doses through 2027-01-15 instead of through 2027-09-01. The same applies, in a milder form, to schedules with a far-future `ends_on`. The goal of this change is to introduce a **rolling, write-independent advance** of the horizon so every live schedule is always materialized to `min(ends_on, today + async_horizon_days)`.

Key facts the design leans on:

- `medication_tracking_scheduled_doses` has a partial unique index `(intake_id, scheduled_at) WHERE type = 'scheduled'`, and `ScheduledDoseGenerator` uses `insertOrIgnore` in 1 000-row chunks. So a re-run that overlaps the existing window is a **safe no-op** at the row level.
- `ScheduleLocalWindowResolver::clampEnd(...)` already handles the "clip to `ends_on` or default to year-ahead" rule we need.
- `MedicationTrackingConfig::ASYNC_HORIZON_DAYS = 365` is the existing rolling-window length — we reuse it; introducing a new constant would risk drift between the on-write tail and the cron top-up.

## Goals / Non-Goals

**Goals:**

- Once a month, on every live, non-`as_needed` schedule, top up the scheduled-dose materialization to `min(ends_on, today_in_tz + ASYNC_HORIZON_DAYS)`.
- Be idempotent: running twice in a row produces zero additional rows.
- Be safe under deletes / edits racing with the cron: never resurrect doses on a soft-deleted schedule; never replay past doses; never overwrite logged doses.
- Fan out work to the queue, one job per schedule, so a single bad schedule doesn't stall the whole batch and so retries are scoped.
- Match existing patterns (`ProcessScheduledOrderCancellations`, `GenerateScheduledDosesJob`) so future readers feel at home.

**Non-Goals:**

- **Not** rebuilding past doses or filling gaps from before "today" — the cron only extends forward. Past gaps are the domain of edit / explicit operator intervention.
- **Not** modifying any existing dose row (`planned_quantity`, `planned_unit`, `timezone`, `scheduled_at`). The cron only **inserts** new rows.
- **Not** generating doses on `as_needed` schedules (those by definition have zero `scheduled` rows).
- **Not** introducing a new configuration knob for the cron horizon. We reuse `medication_tracking.async_horizon_days`.
- **Not** changing `GenerateScheduledDosesJob`. The cron path uses a separate, dedicated job class to keep responsibilities and observability clean.

## Decisions

### Decision 1 — One artisan command + per-schedule queued jobs

The command (`medication-tracking:regenerate-doses`) is a thin **fanner-outer**: it streams every eligible schedule via `ModelRepository::getQuery(...)->cursor()` and dispatches one queued job per row. The actual generation runs in the queue.

**Why:**

- Generating doses for ~N thousand schedules synchronously in the scheduler tick would hold the cron runner and one DB connection for the entire batch.
- Per-schedule jobs give Horizon clean retry boundaries (one schedule failing doesn't poison the rest) and clean log lines (one log/Sentry per failure, tagged with `schedule_id`).
- This mirrors the existing `ProcessScheduledOrderCancellations` pattern (iterate + dispatch) and the existing on-write `GenerateScheduledDosesJob` shape.

**Alternative considered — single job processing all schedules:** rejected. Failure handling would either need a manual try/catch per schedule (re-implementing what the queue already gives us) or risk losing the rest of the batch on one bad row. Visibility would also be much worse.

### Decision 2 — Reuse the existing `GenerateScheduledDosesJob`, no new job class

The command computes `localToday = clock->now()->setTimezone(schedule.timezone)->startOfDay()` per schedule and dispatches the **existing** `App\Jobs\MedicationTracking\GenerateScheduledDosesJob` with `(scheduleId, localWindowStartIso = localToday->toIso8601String(), maxDays = ASYNC_HORIZON_DAYS)`. We do not introduce a second job class.

**Why:**

- `GenerateScheduledDosesJob`'s existing constructor `(int $scheduleId, string $localWindowStartIso, int $maxDays)` already models exactly what we need — the cron call is functionally indistinguishable from a post-edit call with `from = today`. The job's name describes its action ("generate doses for this schedule from this point forward"), not its caller, so reuse is honest.
- The defensive race-case handling the cron needs is already there: `whereKey()->first()` returns `null` for soft-deleted schedules → job returns; the generator returns immediately on `AS_NEEDED`; `ScheduleLocalWindowResolver` produces an empty window for `ends_on < localWindowStart` → generator returns. A separate cron-specific job class would re-implement these guards for no behavioral gain.
- One less class, one less test file, one less hop between command and generator.

**Alternative considered — dedicated `RegenerateScheduledDosesJob`:** rejected. It would duplicate the defensive checks already in `GenerateScheduledDosesJob` + the generator, and add an arbitrary boundary ("this one is for cron, that one is for write paths") that future readers would have to internalize. The only argument for it was queue/log-channel separation, but we ship with neither job pinning a queue, so the separation buys nothing today.

### Decision 3 — Resume from the last generated dose; keep the horizon endpoint fixed at today + 365

For each schedule the command computes:

```php
$localToday    = $now->setTimezone($tz)->startOfDay();
$endOfHorizon  = $localToday->addDays(ASYNC_HORIZON_DAYS);
$startsOnLocal = $clock->parse($schedule->getStartsOn()->toDateString(), $tz)->startOfDay();
$dayAfterLastDose = $lastScheduledAt
    ? $clock->parse($lastScheduledAt)->setTimezone($tz)->startOfDay()->addDay()
    : null;
$localWindowStart = max($startsOnLocal, $dayAfterLastDose);  // pseudo
if ($localWindowStart >= $endOfHorizon) { continue; }        // already covered
$maxDays = (int) $localWindowStart->diffInDays($endOfHorizon);
```

The cursor must therefore hydrate `id`, `timezone`, `starts_on`, and the `last_scheduled_at` correlated-subquery alias.

**Why:**

- **Resume from the day AFTER the last generated dose.** The naïve choice (always start at `localToday` with `maxDays = 365`) is functionally correct because of the partial unique index and `insertOrIgnore` — but for a year-old schedule it forces the generator to *build* (in PHP) 335 dose rows that already exist, just to have the DB ignore them. At "millions of schedules" scale that wasted CPU adds up. Resuming from `last_dose + 1 day` keeps each cron run's per-schedule work proportional to the actual gap (typically ~30 days, since the previous cron ran a month ago) and avoids one extra wasted day re-evaluating the last existing dose's own day.
- **Fixed endpoint at today + 365.** The product requirement is "always have ~1 year visible from today". The endpoint MUST be today + 365 regardless of where the start lands, so we hit the same horizon every month. Using `maxDays = today + 365 - windowStart` (instead of always 365) makes that explicit.
- **Never earlier than `starts_on`.** The generator's contract is `windowStart >= starts_on`. For future-dated schedules with no doses yet, taking `localWindowStart = startsOnLocal` keeps that invariant (`startsOnLocal > localToday`).
- **Skip when already past the horizon.** If `localWindowStart >= endOfHorizon` the schedule is already materialized to or past the target endpoint (e.g. someone manually pre-generated further). Skipping avoids dispatching a job that would do nothing.
- **Per-schedule TZ for `localToday`.** A single UTC instant would silently drift the start-of-day for schedules in non-UTC zones. We do this work in the command (rather than the job) because the cron is the only caller that derives the window from "now"; existing on-write callers (`ScheduleCreator`, `ScheduleEditor`) already pass the right ISO string from their own context. Keeping `GenerateScheduledDosesJob`'s interface unchanged preserves that contract.

**Alternative considered — always `start = today`, `maxDays = 365`:** rejected for the wasted-PHP-work reason above. Functionally identical because of insertOrIgnore, but ~10× more generator work per schedule per month at steady state.

**Alternative considered — always `start = starts_on`, `maxDays = today + 365 - starts_on`:** rejected for old schedules. A schedule with `starts_on = 2020-01-01` would force the generator to iterate ~7 years of (already-existing) doses on every cron run. Resume-from-last is strictly better.

### Decision 4 — Eligibility + last-generated-dose lookup as one SQL statement

The command issues exactly one statement (rendered Postgres):

```sql
SELECT
    mts.id,
    mts.timezone,
    mts.starts_on,
    (
        SELECT MAX(d.scheduled_at)
        FROM medication_tracking_scheduled_doses d
        WHERE d.medication_tracking_schedule_id = mts.id
          AND d.type = 'scheduled'
    ) AS last_scheduled_at
FROM medication_tracking_schedules mts
WHERE mts.deleted_at IS NULL
  AND mts.schedule_type != 'as_needed'
  AND (mts.ends_on IS NULL OR mts.ends_on >= '<today-utc>')
  AND EXISTS (
      SELECT 1
      FROM orders o
      WHERE o.patient_id = mts.patient_id
        AND o.status IN ('paid', 'sent_to_pharmacy', 'intake_completed',
                         'nurse_approved', 'charged', 'shipped',
                         'ready_for_refill', 'delivered')
  )
```

The Eloquent build uses `whereExists($modelRepository->getQuery(Order::class)->whereColumn(...)->whereIn(...))` for the patient-active gate, plain `where('ends_on', '>=', $today)` for the live-window filter (`ends_on` is already a `date` column, so `whereDate` would only add a redundant `::date` cast), and `selectSub(...)` over `ModelRepository::getQuery(MedicationTrackingScheduledDose::class)` with `selectRaw('MAX(scheduled_at)')` + `whereColumn(...)` for the correlated `last_scheduled_at`.

**Why:**

- `as_needed` is excluded at SQL — the strategy already yields zero rows, but skipping the dispatch saves ~one job per `as_needed` schedule.
- `ends_on IS NULL OR ends_on >= today` is the live-window filter — a schedule whose `ends_on` is in the past has nothing to top up.
- `SoftDeletes` default scope removes trashed schedules.
- The patient-active gate is enforced by `WHERE EXISTS` — schedules whose patient has zero active orders are dropped entirely; custom (`order_id IS NULL`) and order-linked schedules go through the same gate (it's per-patient, not per-schedule-order). `EXISTS` short-circuits on the first matching active order per patient, which is strictly cheaper than `INNER JOIN + DISTINCT` (which would multiply rows by N active orders and then dedup).
- The "today" comparison here is intentionally **server-time** (UTC), not per-schedule. Using server-time as a coarse filter is fine because the eligibility error bound is at most one day on either side; the per-schedule dispatch then carries the correct timezone-aware `localWindowStart`.
- The correlated `last_scheduled_at` subquery uses the existing `medication_tracking_scheduled_doses_schedule_id_index`; per-row evaluation is an index seek + filter on `type = 'scheduled'`. If we observe planner regressions at scale a composite index `(medication_tracking_schedule_id, type, scheduled_at DESC)` would turn it into an index-only scan; we are not adding that index up-front because the workload (monthly batch, off-hours) is far from latency-sensitive.
- `cursor()` keeps the working set bounded — at expected scale (millions of orders, tens of thousands of schedules) the join's filtered output is small enough that the cursor never materializes a large result set in PHP, and the SQL plan is a single hash/merge join over indexed columns.

**Alternative considered — `INNER JOIN orders ON patient_id` + `DISTINCT`:** functionally equivalent (Postgres can plan either as a semi-join) but forces the planner to materialize and dedup N-per-patient duplicate rows when `DISTINCT` isn't explicitly recognized as redundant. `WHERE EXISTS` is the canonical way to write a semi-join — clearer intent, no dedup phase. Rejected.

**Alternative considered — `whereIn(patient_id, SELECT patient_id FROM orders WHERE …)` subquery:** also functionally equivalent. `EXISTS` is marginally clearer in this domain because it reads as "the schedule's patient has at least one active order" rather than "the schedule's patient_id is in a set of patient_ids".

### Decision 5 — Cadence: monthly on the 1st, 02:00 UTC

`$schedule->command(...)->monthlyOn(1, '02:00')`. 02:00 UTC is well outside US business hours for every TZ we serve and matches the off-peak slot existing maintenance commands run in.

**Why not weekly?** The horizon is 365 days; a one-month gap leaves at most 30 missing days of pre-materialized doses at the trailing edge, which still leaves the patient with **335+ days** of materialized doses visible at all times. Monthly is the lowest cost cadence that meets the product requirement ("kept up to roughly one year ahead").

**Why not daily?** Cost: even at a 1 ms-per-noop pace, daily × thousands of schedules × 30 days = ~order-of-magnitude more queue work than monthly, all of it ignored by `insertOrIgnore`.

### Decision 6 — Lean on `GenerateScheduledDosesJob`'s existing defensive checks

We rely on the existing job + generator pipeline to handle race cases between dispatch and pickup:

- `whereKey()->first()` returns `null` for both missing and soft-deleted schedules (default `SoftDeletes` scope) — the job returns cleanly on `null`.
- `ScheduledDoseGenerator` returns immediately if `schedule_type === AS_NEEDED` (covers the defensive case where a schedule's type changed between SQL filter and worker pickup).
- `ScheduleLocalWindowResolver::clampEnd` produces a window where `end <= start` when `ends_on < localWindowStart`; `ScheduledDoseGenerator` detects the empty window via `ScheduleLocalWindow::isEmpty()` and returns.

A cleanly-exiting job MUST NOT be marked failed and MUST NOT emit a Sentry error — `GenerateScheduledDosesJob` already meets this bar.

## Risks / Trade-offs

- **[Risk] The monthly cron fires while a long-running edit transaction is still open on the same schedule.** → The edit transaction holds the schedule row's lock; the cron's `insertOrIgnore`s touch `medication_tracking_scheduled_doses`, not the schedule row, so they won't deadlock. Worst case the cron inserts rows the edit was about to delete (`scheduled_at >= now AND tracking_log_id IS NULL`) — and then the edit's deletion runs and removes them. End state matches the edit's intent. Acceptable.
- **[Risk] Queue backlog if there are many thousands of schedules.** → Per-schedule jobs are tiny (a single `cursor()` walk over 365 days × a couple of intakes is microseconds of CPU + one batched insert). At expected scale this is comfortably under one minute of queue time per 10 k jobs. If it ever becomes a problem we can add Horizon throttling on the medication-tracking queue without touching this code.
- **[Risk] Drift between on-write `ASYNC_HORIZON_DAYS` and the cron horizon if someone changes one.** → We reference the same constant, no duplication. Mitigation is structural.
- **[Risk] A schedule with a buggy `schedule_rule` (e.g. malformed `days_of_week` payload from a prior edit) makes the generator throw.** → That job fails, gets retried with the queue's normal backoff, then dead-letters; the other schedules in the batch are unaffected (Decision 1). The dead-letter is the right signal — a corrupt `schedule_rule` is a real bug we want surfaced.
- **[Trade-off] We chose monthly cadence and a fixed 1-year rolling horizon.** Product wants "always have ~1 year visible". Picking weekly would burn ~4× the queue work for the same UX. If product later wants e.g. "always have at least 6 months visible", we'd raise cadence (or lower horizon) — both are one-line changes.

## Migration Plan

- This is purely additive: new command, new job, one new `Kernel::schedule()` line.
- On first deploy, the next 1st-of-month tick will run the top-up. Schedules created before this change that already have 365 days materialized will be untouched (idempotent); schedules whose horizon has fallen behind will catch up in a single run.
- **Manual catch-up:** running `php artisan medication-tracking:regenerate-doses` ad-hoc is supported and the recommended way to seed the horizon on day one if we want to avoid waiting for the first cron tick.
- **Rollback:** remove the `Kernel::schedule()` line and re-deploy. The job class and command can stay (they are dormant without a scheduler entry). No DB state needs to be reverted — the materialized rows the job added are correct in their own right and would have been added by the next edit anyway.

## Open Questions

- Should the per-schedule job pin a dedicated queue so we can throttle it independently of on-write traffic? **Default position:** no — match `GenerateScheduledDosesJob`, which inherits the default queue. If we observe contention later, introduce `QueueName::MEDICATION_TRACKING` (or similar) and pin **both** jobs to it in the same change.
