## 1. Normalization seam

- [x] 1.1 Add `App\Helpers\TimezoneNormalizer` with a `const` `deprecated => canonical` alias map (seeded with `Europe/Kiev => Europe/Kyiv`) and a static `normalize(string $timezone): string` that returns the mapped canonical value or the input unchanged.
- [x] 1.2 Add `App\Http\Requests\Api\Common\NormalizesTimezoneInput` trait (beside `NormalizesEmail`) with `normalizeTimezone(string $key): void` that merges `TimezoneNormalizer::normalize(...)` back under `$key` when filled (named `…Input` to avoid the existing `Concerns\NormalizesTimezone` datetime concern).

## 2. Apply to medication-tracker FormRequests

Each request `use`s the trait and calls `$this->normalizeTimezone('timezone')` from `prepareForValidation()`.

- [x] 2.1 Old tracker — `MedicationSchedule/CreateRequest` and `UpdateRequest`.
- [x] 2.2 New tracker — `Schedule/CreateRequest`.
- [x] 2.3 New tracker — `ScheduledDose/IndexRequest` and `NextRequest`.

## 3. Integration (feature) tests

Note: SRD-1336 already added "Kiev is accepted" tests across both trackers (with the rule on `timezone:all_with_bc`, which is exactly what still fails on stage). These were strengthened to assert the value is normalized to `Europe/Kyiv` rather than merely accepted.

- [x] 3.1 Old tracker: `POST /medication-schedules` with `timezone: Europe/Kiev` succeeds (201) and the stored/returned timezone is `Europe/Kyiv` (`create_withDeprecatedKievAlias_isNormalizedToKyiv`).
- [x] 3.2 New tracker: `Schedule/Create` with `timezone: Europe/Kiev` succeeds and persists `Europe/Kyiv` (`create_withDeprecatedKievAlias_isNormalizedToKyiv`).
- [x] 3.3 New tracker: `ScheduledDose` Index/Next with `?timezone=Europe/Kiev` accepted and behaves as `Europe/Kyiv` (`index_withDeprecatedKievAlias_isAccepted`, `next_withDeprecatedKievAlias_isAccepted`).
- [x] 3.4 A genuinely invalid timezone still returns a 422 timezone validation error on both trackers (`create_withInvalidTimezone_shouldReturn422`, `update_withInvalidTimezone_shouldReturn422`, `create_withUnknownTimezone_returns422`).

## 4. Quality gates

- [ ] 4.1 Run Pint, PHPStan level 8, Psalm level 8 on the new/changed files; no baseline growth.
- [ ] 4.2 Run the affected feature tests.
