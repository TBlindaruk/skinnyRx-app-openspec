## 1. DTO updates

- [x] 1.1 Change `CreateDoseTrackingLogDto::$painLevel` from `int` to `?int` in `app/DTO/MedicationTracking/TrackingLog/CreateDoseTrackingLogDto.php`.
- [x] 1.2 Change `CreateOffScheduleTrackingLogDto::$painLevel` from `int` to `?int` in `app/DTO/MedicationTracking/TrackingLog/CreateOffScheduleTrackingLogDto.php`.

## 2. FormRequest updates (validation rules + OpenAPI)

- [x] 2.1 In `CreateByDoseRequest.php`: drop `pain_level` from the OA `required` list, mark the property `nullable: true`, change the rule to `['nullable', 'integer', 'between:0,10']`, and change `getPainLevel()` to return `?int` (use the `has('pain_level')` pattern from `CreateRequest::getPainLevel`).
- [x] 2.2 In `CreateByScheduleRequest.php`: same edits — drop from OA `required`, `nullable: true`, rule `['nullable', 'integer', 'between:0,10']`, `getPainLevel(): ?int`.
- [ ] 2.3 Verify OpenAPI generation still succeeds (`php artisan l5-swagger:generate` or whichever generator the project uses) — schemas reflect the new conditional contract. _(deferred — manual)_

## 3. Controller cross-field validation

- [x] 3.1 In `CreateByDoseController::__invoke`, after resolving `$dose` (which already eager-loads `medicationTrackingSchedule.product`), compute `$injectable = $dose->getMedicationTrackingSchedule()->getProduct()?->getType() === ProductType::INJECTION` and call a new private helper `assertPainLevelMatchesProduct(?int $painLevel, bool $injectable): void` that throws `ValidationException::withMessages([...])`:
  - if `$injectable && $painLevel === null` → `pain_level` "The pain level field is required for this medication."
  - if `!$injectable && $painLevel !== null` → `pain_level` "The pain level field must not be present for this medication."
- [x] 3.2 In `CreateByScheduleController::__invoke`, after resolving `$schedule` (which already eager-loads `product`), apply the same helper using `$schedule->getProduct()?->getType()`.
- [x] 3.3 Pass `painLevel: $request->getPainLevel()` (now `?int`) into both DTOs — the DTOs already accept nullable in step 1.
- [x] 3.4 Import `ProductType` and `ValidationException` in the two controllers; do NOT add a facade/static call beyond what's already there.

## 4. Spec sync

- [x] 4.1 Confirm the two MODIFIED requirements in `openspec/changes/restrict-pain-level-to-injectables/specs/medication-tracking-scheduler/spec.md` match the implementation; reconcile any wording drift.

## 5. Feature tests

- [x] 5.1 Update existing tests in `tests/Feature/Http/Controllers/Api/Patient/MedicationTracking/TrackingLogController/` (CreateByDoseTest, CreateByScheduleTest): pin product types where needed, drop `pain_level` for non-injection schedules, assert persisted value/`NULL`.
- [x] 5.2 Add a `by-dose` test: POSTing `pain_level` against a custom schedule returns `422` with a `pain_level` field error and no `tracking_logs` row is persisted.
- [x] 5.3 Add a `by-dose` test: POSTing `pain_level` against a `PILLS`/`DROPS` schedule returns `422` with a `pain_level` field error.
- [x] 5.4 Add a `by-dose` test: POSTing without `pain_level` against an INJECTION schedule returns `422` with a `pain_level` "required" field error.
- [x] 5.5 Add `by-schedule` counterparts of 5.2, 5.3, 5.4 — each verifying no `medication_tracking_scheduled_doses` row is created for the failure cases.
- [x] 5.6 Add a `by-dose` test: PILLS/DROPS schedule, no `pain_level` → `201` with persisted `pain_level = NULL` (existing tests already cover this matrix after update; plus explicit `painLevelExplicitNullOnNonInjectionSchedule_persistsNullPainLevel`).
- [x] 5.7 Add a `by-dose` test: custom schedule (`product_id IS NULL`), no `pain_level` → `201` with persisted `pain_level = NULL` (covered by existing `createByDose_onProductlessSchedule_fallsBackToPlannedUnit` after update).
- [x] 5.8 Add `by-schedule` counterparts of 5.6 and 5.7 (`onPillsSchedule_persistsNullPainLevel`, `onDropsSchedule_persistsNullPainLevel`, plus the custom-schedule happy path in `createBySchedule_createsTrackingLogAndOffScheduleDose`).

## 6. Static analysis & style

- [x] 6.1 Run `make run-pint` / `make fix-pint` — no diff after Pint.
- [x] 6.2 Run `make run-phpstan` — passes at level 8 with no new baseline entries.
- [x] 6.3 Run `make run-psalm` — passes at level 8 with no new baseline entries.
- [x] 6.4 Run `make code-test` (paratest) — full suite green.

## 7. Verification

- [x] 7.1 Hit both endpoints manually (or via tinker / a scratch test) using each schedule kind (injection, pills, drops, custom) to confirm the `201` / `422` matrix. _(Covered by the feature tests in §5; manual smoke deferred to staging.)_
- [x] 7.2 Inspect a sample `tracking_logs` row for each non-injection case and confirm `pain_level IS NULL`. _(Asserted in feature tests: `onPillsSchedule_persistsNullPainLevel`, `onDropsSchedule_persistsNullPainLevel`, `createBySchedule_createsTrackingLogAndOffScheduleDose`, `painLevelExplicitNullOnNonInjectionSchedule_persistsNullPainLevel`, etc.)_
- [ ] 7.3 Confirm regenerated OpenAPI doc shows `pain_level` outside `required` and `nullable: true` for both `CreateByDoseRequest` and `CreateByScheduleRequest`. _(deferred — manual)_
