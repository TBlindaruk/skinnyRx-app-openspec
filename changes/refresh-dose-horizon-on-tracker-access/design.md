## Context

`MedicationTrackingScheduledDose` rows are a pre-materialized, year-long horizon read **only** by the patient's own tracker endpoints (`ScheduledDoseController`, `TrackingLog*Controller`). No reminder/notification consumes them — `NextDoseReminderCalculator` works on the legacy `MedicationSchedule` model and computes cadence algorithmically from `apply_times`, never touching the materialized rows. Today the horizon is maintained by `RegenerateScheduledDosesCommand`, a monthly cron that sweeps every schedule and (in the prior in-progress design) tried to gate by patient engagement.

Because the only reader is the patient themselves, the horizon only needs to be current *when the patient looks*. That makes a lazy, on-access model strictly better than a cron + engagement-heuristic: generate exactly for who is using the tracker, exactly when.

The existing `TrackActivity` middleware is the template: it reads `last_active_at` off the guard-hydrated patient (zero extra query), and dispatches a queued write only when the value is stale past a throttle window. We mirror that shape.

## Goals / Non-Goals

**Goals:**
- Keep a patient's dose horizon current, triggered by their interaction with the medication tracker.
- Zero extra query in the hot request path; never block the response.
- Throttle the top-up attempt to once per 7 days per patient.
- Reuse the existing per-schedule window math + `ScheduledDoseGenerator`; keep idempotency.
- Delete the cron and its engagement heuristic.

**Non-Goals:**
- No change to create-time / edit-time dose generation.
- No proactive generation for patients who never open the tracker (intentional — nothing reads their horizon).
- No per-schedule throttle marker (patient-level is sufficient and free to read).
- No synchronous, in-request generation.

## Decisions

### Decision 1: Middleware on the `medication-tracking` route group, mirroring `TrackActivity`

Register `EnsureDoseHorizon` on the `/api/v1/patient/medication-tracking` group (api.php:872), which already sits under `auth.patients` where `TrackActivity` runs. The middleware:

```
patient = guard->user()                       // already hydrated
refreshedAt = patient.getDoseHorizonRefreshedAt()      // 0 extra queries
if refreshedAt === null || refreshedAt < now - HORIZON_REFRESH_THROTTLE_DAYS:
    dispatch RefreshPatientDoseHorizon(patientId, now)
return next(request)                           // never blocks
```

**Why the whole group, not just the GET scheduled-doses route:** the marker is patient-level and the job tops up all of the patient's schedules, so the entry route is irrelevant; the throttle makes extra triggers free. Attaching to the group is the simplest correct seam. (The group also contains legacy `medication-schedules` endpoints over the old `MedicationSchedule` model — harmless, since the job operates on `MedicationTrackingSchedule` regardless of entry point.)

### Decision 2: Throttle marker is a `patients.dose_horizon_refreshed_at` column

A nullable timestamp on `patients`, cast `immutable_datetime`, exposed via `getDoseHorizonRefreshedAt()`. Read for free off the guard-hydrated patient — exactly like `last_active_at`.

**Why a column over Redis cache:** persistent and auditable; a cache flush would otherwise trigger a thundering herd of top-up jobs on the next request wave. **Why patient-level over per-schedule:** per-schedule would require loading schedules just to decide whether to act, defeating the free read; once-a-week per patient is plenty given a 365-day horizon.

No backfill: `NULL` simply means "first tracker access tops up." Existing materialized doses stay valid.

### Decision 3: Per-patient job reuses the existing window math + generator

`RefreshPatientDoseHorizon(patientId, refreshedAtTimestamp)` iterates the patient's eligible schedules (not-trashed, `!= as_needed`, `ends_on` open), and for each computes the same window the cron did:

- `localToday`, `endOfHorizon = localToday + ASYNC_HORIZON_DAYS`, `startsOnLocal`, `dayAfterLastDose` (from `MAX(scheduled_at)`), `localWindowStart = max(...)`;
- if `localWindowStart < endOfHorizon`, generate via `ScheduledDoseGenerator` (same idempotent `insertOrIgnore` path).

A patient has at most `SCHEDULE_LIMIT_PER_PATIENT = 50` schedules, so a single job handling them inline is fine — no second fan-out layer of per-schedule jobs is needed. After processing all schedules the job sets `patient.dose_horizon_refreshed_at = refreshedAtTimestamp`.

**Why pass the timestamp in (not `now()` in the job):** matches `TrackActivityJob`; the marker reflects the moment that triggered the attempt, and stays deterministic/testable.

### Decision 4: Stamp the marker after generation, accept the benign race

Two requests within the throttle window could both find a stale marker and both dispatch before either stamps it. Both jobs are idempotent (`insertOrIgnore` + partial unique index), so the worst case is one redundant pass — identical to the `TrackActivity` double-write race. Stamping *after* success (rather than optimistically in the middleware) ensures a failed job doesn't advance the marker and silently skip top-up for a week.

## Risks / Trade-offs

- **[A patient with an active order who never opens the tracker never gets topped up]** → Intended: nothing but their own tracker reads the horizon, so there is nothing to serve until they open it — at which point their request tops it up. The horizon is a year long, so the first read after a gap still shows ample materialized doses while the async job extends the tail.
- **[First post-gap request returns a not-yet-extended horizon (job runs after response)]** → Eventual-consistency, same as `TrackActivity`. Harmless because the materialized horizon is far longer than any realistic gap; the tail extends moments later.
- **[Losing the cron removes a guaranteed sweep]** → By design. If a guaranteed floor is ever needed, a low-frequency safety cron can be reintroduced as a separate change; not included here to keep the model clean.
- **[Job cost concentrated on active users]** → Bounded: ≤50 schedules/patient, once/week/patient, spread across real traffic rather than a single monthly spike.

## Migration Plan

1. Ship migration adding `patients.dose_horizon_refreshed_at` (nullable, no backfill).
2. Add middleware + job + `Patient` getter + config knob.
3. Attach middleware to the `medication-tracking` route group.
4. Remove `RegenerateScheduledDosesCommand` + its `Kernel::schedule()` registration + its test.

Rollback: revert the code; drop the column. Existing doses are untouched throughout (generation only ever `insertOrIgnore`s and never deletes).

## Open Questions

_None outstanding — flag if a low-frequency safety-net cron is desired after all._
