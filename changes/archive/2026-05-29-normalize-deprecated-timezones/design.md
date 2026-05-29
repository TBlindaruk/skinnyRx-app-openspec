## Context

Laravel's `timezone:all_with_bc` rule validates against `timezone_identifiers_list(DateTimeZone::ALL_WITH_BC)`. Whether a deprecated backward-compat alias (e.g. `Europe/Kiev`) appears in that list depends on the PHP/tzdata build of the host. On stage the alias is absent, so `Europe/Kiev` fails validation even though PHP can still construct `new DateTimeZone('Europe/Kiev')`.

The medication tracker exists in two generations, and five of its FormRequests accept a `timezone` field (all using `timezone:all_with_bc`):

- Old tracker (`App\Models\MedicationSchedule`, `MedicationScheduleController`): `MedicationSchedule/CreateRequest`, `MedicationSchedule/UpdateRequest`.
- New tracker (`App\Models\MedicationTracking\*`): `Schedule/CreateRequest`, `ScheduledDose/IndexRequest`, `ScheduledDose/NextRequest` (the last two are GET requests with `timezone` as a query parameter).

This change is scoped to those five requests. Appointment requests also accept a `timezone` field, but they take US timezones and are not affected by this bug, so they are explicitly out of scope. None of the five target requests currently declare `prepareForValidation()` for the timezone field, so a shared normalization step can be introduced cleanly. `prepareForValidation()` runs for both body and query input, so it works uniformly for the POST and GET endpoints.

Per CLAUDE.md: domain helpers live in `app/Helpers/` (static methods like `PhoneNumber::normalize`), validation/normalization belongs at the request boundary, and no facades/global helpers in app code.

## Goals / Non-Goals

**Goals:**
- Accept deprecated timezone aliases (primarily `Europe/Kiev`) on every environment and silently upgrade them to the canonical identifier (`Europe/Kyiv`).
- Guarantee downstream code (persistence, resources, scheduling) only ever sees canonical identifiers.
- One shared seam, applied consistently to the five medication-tracker `timezone`-accepting requests (old + new tracker).

**Non-Goals:**
- Backfilling existing DB rows that already store `Europe/Kiev` (PHP still constructs that zone, so reads keep working). Can be a follow-up command if needed.
- Building the full IANA backward-link table. We map the aliases relevant to our user base; the map is trivially extensible.
- Touching the appointment requests / `allowed_timezones` config (US zones, not affected by this bug). The trait is reusable there later if needed.

## Decisions

### 1. Normalize at the request boundary in `prepareForValidation()`, not in a validation rule

Normalizing before validation means the merged value flows into validation **and** into every getter / DTO / model write. A custom validation rule cannot mutate the input, so it could accept `Europe/Kiev` but would leave the deprecated string to be persisted — defeating "use Kyiv everywhere". `prepareForValidation()` is the standard mutation seam and keeps the existing `timezone` / `timezone:all_with_bc` rules intact as the final guard against genuinely invalid values.

*Alternative considered:* a custom `Rule` object accepting both forms — rejected because it doesn't normalize the stored value and would have to be duplicated everywhere `Rule::in` is used.

### 2. Explicit alias map in a `TimezoneNormalizer` helper

`App\Helpers\TimezoneNormalizer::normalize(string $tz): string` holds a `const` array of `deprecated => canonical` and returns the mapped value, or the input unchanged when not present. Matches the existing static-helper convention (`PhoneNumber`, `Bmi`). PHP exposes no direct "canonical name" API for backward links, so an explicit map is the robust choice and keeps behavior deterministic across builds. Seeded with `Europe/Kiev => Europe/Kyiv`; add more entries as they surface.

*Alternative considered:* deriving canonical names from `DateTimeZone::getLocation()` / abbreviation lookups — rejected as unreliable and indirect.

### 3. Shared trait `NormalizesTimezoneInput` with a key-parameterized method

`App\Http\Requests\Api\Common\NormalizesTimezoneInput` exposes `normalizeTimezone(string $key): void`, which, when `$key` is filled, merges `TimezoneNormalizer::normalize(...)` back under the same key. It lives beside the analogous `App\Http\Requests\Api\Common\NormalizesEmail` trait and mirrors its shape (`filled()` + `merge()`, no static-analysis annotations needed). Each of the five medication-tracker FormRequests `use`s the trait and calls `$this->normalizeTimezone('timezone')` from its own `prepareForValidation()`. Passing the key keeps the trait reusable for any future field name.

*Naming:* the obvious `NormalizesTimezone` is already taken by an unrelated concern (it normalizes datetime *values* with TZ offsets into `CarbonImmutable`, backed by `App\Services\Dates\TimezoneNormalizer`). The new trait is named `NormalizesTimezoneInput` to avoid that collision.

## Risks / Trade-offs

- [Map is incomplete — another deprecated alias surfaces] → The helper passes unmapped values through unchanged, so behavior is no worse than today; adding an entry is a one-line change. Document the map as the single place to extend.
- [A future request uses a different field name for timezone] → The trait only handles `timezone`; such a request would simply not be normalized. Mitigation: keep the field name `timezone` as the convention (all current requests already do).
- [Existing rows store `Europe/Kiev`] → Out of scope; PHP still constructs that zone for reads. A backfill command can be added later if canonical consistency in storage is required.
- [Trait overriding a request's own `prepareForValidation()`] → Verified none of the five targets define one. New code that needs both must call the helper explicitly.

## Migration Plan

1. Add `App\Helpers\TimezoneNormalizer` with the alias map + `normalize()`.
2. Add `App\Http\Requests\Api\Common\NormalizesTimezoneInput` trait (`normalizeTimezone(string $key)`), beside `NormalizesEmail`.
3. Apply the trait to the five medication-tracker timezone-accepting FormRequests (old + new tracker), calling `$this->normalizeTimezone('timezone')` from `prepareForValidation()`.
4. Add integration (feature) tests proving `Europe/Kiev` is accepted and normalized to `Europe/Kyiv` on both the old and new trackers, and that a genuinely invalid timezone is still rejected with 422.
5. Deploy. No DB change, no config change. Rollback = revert the trait usage (behavior reverts to rejecting deprecated aliases).
