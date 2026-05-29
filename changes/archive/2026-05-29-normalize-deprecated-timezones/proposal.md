## Why

Clients (browsers, older libraries) sometimes send deprecated IANA timezone aliases such as `Europe/Kiev`. On environments whose PHP/tzdata build drops backward-compatibility links, Laravel's `timezone:all_with_bc` rule rejects these aliases, so otherwise-valid requests fail with `422 The timezone field must be a valid timezone.` This is a real production-shaped bug: the patient app sends `Europe/Kiev` and medication-schedule creation breaks on stage.

## What Changes

- Add a single normalization seam that maps deprecated IANA timezone aliases to their canonical identifiers (primary case: `Europe/Kiev` → `Europe/Kyiv`).
- Normalize the inbound `timezone` field at the request boundary (in `prepareForValidation`) so that both validation and everything downstream (persistence, resources, scheduling) work with the canonical identifier.
- Apply this consistently across the medication-tracker FormRequests (both the old `MedicationSchedule` tracker and the new `MedicationTracking` tracker) that accept a `timezone` field, so the deprecated alias is accepted and silently upgraded.

## Capabilities

### New Capabilities
- `timezone-normalization`: Accepting inbound timezone identifiers, normalizing deprecated IANA aliases to their canonical form before validation, and guaranteeing downstream code only ever sees canonical identifiers.

### Modified Capabilities
<!-- No existing spec's requirements change. -->

## Impact

- New helper: `App\Helpers\TimezoneNormalizer` (alias→canonical map + `normalize()`).
- New FormRequest trait `App\Http\Requests\Api\Common\NormalizesTimezoneInput` (`normalizeTimezone(string $key)`), placed beside the analogous `NormalizesEmail` trait and called from each request's `prepareForValidation()`.
- Affected FormRequests — only the medication-tracker `timezone`-accepting requests:
  - Old tracker (`MedicationScheduleController` / `App\Models\MedicationSchedule`):
    - `Api/Patient/MedicationTracking/MedicationSchedule/CreateRequest`, `UpdateRequest`
  - New tracker (`MedicationTracking` controllers / models):
    - `Api/Patient/MedicationTracking/Schedule/CreateRequest`
    - `Api/Patient/MedicationTracking/ScheduledDose/IndexRequest`, `NextRequest` (GET query param)
- Out of scope: the appointment requests (`Api/Appointment/*`, `Api/Manager/Appointment/AvailabilitySlot/SetRequest`) — they accept US timezones and are not affected by this bug.
- No DB migration; no breaking API change (deprecated aliases were previously rejected, now accepted).
- Pre-existing rows already storing `Europe/Kiev` are out of scope (PHP can still construct that zone, so reads keep working); a backfill is a non-goal here.
