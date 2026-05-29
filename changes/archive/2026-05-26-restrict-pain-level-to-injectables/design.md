## Context

Two endpoints currently treat `pain_level` (integer 0–10) as unconditionally required:

- `POST /api/v1/patient/medication-tracking/tracking-logs/by-dose` — scheduled log against a `medication_dose_id`.
- `POST /api/v1/patient/medication-tracking/tracking-logs/by-schedule` — off-schedule log against a `medication_tracking_schedule_id`.

Both flows resolve the owning `MedicationTrackingSchedule` early (the controllers already eager-load `RELATIONSHIP_PRODUCT`) and then call an action that persists a `tracking_logs` row. `tracking_logs.pain_level` is already a nullable integer in the DB.

Schedules in this system come in two flavours (see `medication-tracking-scheduler` spec):

| Schedule kind | `product_id` | `product.type` |
|---|---|---|
| Custom (patient-created)   | `NULL`       | n/a |
| Order-linked, oral         | not null     | `PILLS` / `DROPS` |
| Order-linked, injectable   | not null     | `INJECTION` |

Pain level is only meaningful for the third row. The product is reachable via the schedule the request already pins down — for by-dose: `dose.medicationTrackingSchedule.product`; for by-schedule: `schedule.product`. Both controllers already eager-load the product, so no extra query is needed.

## Goals / Non-Goals

**Goals:**
- `pain_level` is accepted (and required) only when the resolved schedule's product is `ProductType::INJECTION`.
- `pain_level` is rejected with a `422` validation error when the schedule is custom (no product) or its product is `PILLS` / `DROPS`.
- Persisted `tracking_logs.pain_level` is `NULL` for every non-injection log.
- Same rule applies symmetrically to scheduled (by-dose) and off-schedule (by-schedule) logs.
- OpenAPI schemas accurately describe the conditional rule.

**Non-Goals:**
- Backfilling or scrubbing existing `tracking_logs.pain_level` values for historical non-injection logs. The column stays as-is for past rows.
- Touching the legacy generic `POST /api/v1/patient/tracking-logs` (`CreateRequest.php`) or `UPDATE` endpoint — those already use `RequiredIf($type === MEDICATION)` and are out of this change's scope.
- Changing the read-side: `TrackingLogResource` continues to expose `pain_level` as nullable. No filtering by product type on read.
- Adding new product types or changing `ProductType` enum semantics.

## Decisions

### Decide whether `pain_level` is allowed at the FormRequest layer, not in the action

The conditional belongs in the validation layer because it determines whether the input is valid. The cleanest place is `prepareForValidation()` + `rules()`, both of which run after the route is matched and the request body is parsed.

**How**: Resolve the schedule/dose during the FormRequest's `rules()` execution by injecting `ModelRepository` and `PatientGuard` via Laravel's method DI (FormRequest supports parameter DI in `rules()` and other methods through `app(...)` resolution, but to stay within CLAUDE.md §3 we will instead resolve in the controller and pass an "injectable" flag into the validator via `merge(...)` from `prepareForValidation`).

After re-checking the existing pattern in this codebase: `CreateByDoseRequest::rules()` already has access to `$this`; the schedule/dose lookup is non-trivial and would duplicate work the controller does. The chosen approach:

1. **Controller resolves** the dose/schedule (it already does) and computes a boolean `painLevelExpected`.
2. **Controller calls a thin validator helper** `assertPainLevelMatchesProduct($request, $painLevelExpected)` *before* dispatching to the action. The helper throws `ValidationException` with the same field-keyed error shape Laravel's validator produces.
3. **FormRequest** drops `pain_level` from `required`, declares it as `nullable|integer|between:0,10`. The cross-field check stays in the controller, which is where the schedule identity already lives.

This keeps the FormRequest free of DI on a model lookup, keeps the controller small (one extra line — already shaped like an authorization check), and produces the same `422` shape callers expect.

**Alternative considered**: Push the lookup into `prepareForValidation()` via `app(ModelRepository::class)`. Rejected because it hides a DB hit inside a FormRequest and violates the project's "constructor DI only" rule.

**Alternative considered**: Validate in the action and let it throw a domain exception the controller maps to 422. Rejected because validation failures should be `ValidationException` (field-keyed JSON), not domain exceptions (message-only), to match every other endpoint in this capability.

### DTO field type becomes nullable

`CreateDoseTrackingLogDto::$painLevel` and `CreateOffScheduleTrackingLogDto::$painLevel` change from `int` to `?int`. The action writes the value verbatim into `tracking_logs.pain_level`; a `null` DTO field maps to a `NULL` row.

**Alternative considered**: Keep DTO field non-null and have the action substitute `null` for non-injection schedules. Rejected because it spreads the "is injectable?" logic across the controller and the action; the DTO is the boundary and should already reflect the rule applied above it.

### Source of truth for "is injectable"

A single private helper on each controller (or one shared trait) maps the schedule's product to a boolean:

```
private function shouldExpectPainLevel(MedicationTrackingSchedule $schedule): bool
{
    $product = $schedule->getProduct();
    return $product !== null && $product->getType() === ProductType::INJECTION;
}
```

Custom (`product === null`) and oral (`PILLS`/`DROPS`) both return `false`. Three line helper; no abstraction class created.

**Alternative considered**: Add `isInjectable(): bool` to `MedicationTrackingSchedule`. Rejected — the model is a thin data shape (§6 of CLAUDE.md); the rule lives at the HTTP boundary, not in the model.

### OpenAPI documentation

Drop `pain_level` from the `required` array, mark it `nullable`, and describe the rule in the property `description`. Same edit applied to both `CreateByDoseRequest` and `CreateByScheduleRequest` OA schemas.

### Existing tests

The current feature tests pass a `pain_level` for every test setup. They will be split:

- For tests that drive injection schedules → keep `pain_level` required, assertion stays.
- For tests that drive PILLS/DROPS/custom schedules → drop `pain_level` from the payload, assert the persisted log's `pain_level` is `NULL`.
- Add two new tests per endpoint: (a) sending `pain_level` against a custom schedule → `422` with `pain_level` field error; (b) injection schedule with no `pain_level` → `422` with `pain_level` required.

## Risks / Trade-offs

- **[Risk] Mobile/web clients still send `pain_level` for non-injection schedules** → Mitigation: coordinated release with the frontend team (same SRD-1448 epic). The breaking semantics is intentional — silently dropping the value would hide client bugs.
- **[Risk] Existing in-flight requests in production at deploy time** → Mitigation: a `422` is a recoverable client error; no data corruption.
- **[Risk] Historical `pain_level` rows for non-injection logs remain in the DB** → Trade-off: explicitly out of scope. Reporting/analytics consumers should filter by product type if they only care about injection pain.
- **[Risk] The controller now performs cross-field validation outside the FormRequest** → Mitigation: keep the helper local and minimal; one explicit call site per controller. The pattern is already used elsewhere when validation depends on a DB-resolved entity (e.g. patient-ownership checks in the same two controllers).
- **[Trade-off] No shared trait for the "shouldExpectPainLevel" helper** → With only two call sites, a trait would be ceremony. If a third endpoint joins, extract then.

## Migration Plan

1. Merge the API change to `stage`; frontend rolls out the client form change against staging.
2. Verify staging logs: the new feature tests cover the matrix; manual smoke on staging confirms a custom-schedule POST without `pain_level` returns `201`.
3. Promote to production in the same release window as the mobile/web client updates.
4. **Rollback**: revert the PR. No DB migration; schema is unchanged. Any logs created during the rollout retain whatever pain_level was sent (null for the new flow, integer for the old).

## Open Questions

- None. Behavior for the read-side (`TrackingLogResource`) is unchanged; behavior for the unrelated generic `POST /api/v1/patient/tracking-logs` is unchanged.
