## Why

Patients on the legacy and new medication schedulers cannot submit a schedule (or query scheduled doses) when their device reports a deprecated IANA timezone name such as `Europe/Kiev`. Laravel's `timezone:all` validator is backed by `DateTimeZone::ALL`, which excludes backward-compatibility (BC) aliases, so the request fails with `422` even though PHP's `DateTimeZone` itself parses the alias just fine. Downstream code (Carbon, dose-window math, `medication_tracking_scheduled_doses.timezone` reads) all work transparently with BC names, so the only thing blocking these patients is the validator.

## What Changes

- Switch the `timezone` validator on the five patient-facing scheduler FormRequests from `timezone:all` to `timezone:all_with_bc`, so any name in `DateTimeZone::ALL_WITH_BC` is accepted — that is, every canonical IANA zone **plus** the deprecated aliases PHP's tzdata recognizes (`Europe/Kiev`, `US/Eastern`, `Asia/Calcutta`, etc.). Unknown identifiers still fail with `422`.
- No canonicalization, no remapping. Whatever the client sends is what gets stored. Two patients in the same city may end up with two different stored strings (`Europe/Kiev` and `Europe/Kyiv`); we accept this trade-off because PHP/Carbon treat both as the same zone for all time math, day-window queries, and DST transitions.
- Affected files:
  - **New scheduler:** `MedicationTracking/Schedule/CreateRequest`, `MedicationTracking/ScheduledDose/IndexRequest`, `MedicationTracking/ScheduledDose/NextRequest`.
  - **Legacy scheduler:** `MedicationTracking/MedicationSchedule/CreateRequest`, `MedicationTracking/MedicationSchedule/UpdateRequest`.

## Capabilities

### New Capabilities
<!-- None. -->

### Modified Capabilities
- `medication-tracking-scheduler`: the "valid IANA `timezone`" requirements relax to accept BC aliases (e.g. `Europe/Kiev`) for both schedule creation and dose-list queries. Unknown identifiers continue to return `422`. The system does not promise any canonical form for the stored or returned value — it echoes back whatever the client sent.

## Impact

- **Modified:** five FormRequest files under `app/Http/Requests/Api/Patient/MedicationTracking/` — one-line change in each (`'timezone:all'` → `'timezone:all_with_bc'`).
- **Out of scope (no change here):** appointment FormRequests (`Api/Appointment/*Request.php`) currently use `'timezone'` (which is equivalent to `DateTimeZone::ALL` — also rejects `Europe/Kiev`) and the manager availability-slot endpoint uses an explicit `Rule::in($allowedTimezones)` list. They are not part of the patient scheduler flow that triggered this fix; the same one-line relaxation can be applied there in a follow-up if the same complaint surfaces.
- **Data:** no migration, no schema change. Storage starts accepting BC strings exactly as sent — PHP's `DateTimeZone` constructor reads both forms identically, so dose generation, day-window math, and resource rendering behave the same regardless of which form is stored.
- **Tests:** new feature-test scenarios on each of the five endpoints confirming `Europe/Kiev` (and a wrong-case / unknown control) behave correctly. No unit-level helper or custom rule to test — the change is the framework's built-in rule.
