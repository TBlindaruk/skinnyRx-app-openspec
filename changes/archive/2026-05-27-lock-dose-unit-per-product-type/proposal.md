## Why

`DoseUnit::forProductType()` locks the intake unit for every order-linked schedule, and two of the three mappings are wrong for the product team's intent:
- `INJECTION` maps to `injection`, but should be `unit`.
- `PILLS` maps to `unit`, but should be `mg`.

Because the create flow strictly validates the intake unit against `forProductType(...)`, the FE is forced to send the wrong unit and cannot send the correct one.

## What Changes

- Map `ProductType::INJECTION` to `DoseUnit::UNIT` (`'unit'`) instead of `DoseUnit::INJECTION` (`'injection'`).
- Map `ProductType::PILLS` to `DoseUnit::MG` (`'mg'`) instead of `DoseUnit::UNIT` (`'unit'`).
- `ProductType::DROPS` stays `DoseUnit::ML` (`'ml'`) — unchanged.
- Tablets (`PILLS`) keep the whole-number quantity rule — quantity is entered as whole `mg` (e.g. `25`). `PlannedQuantityForProductType` is unchanged (it keys off `ProductType::PILLS`, not the unit).
- **BREAKING**: order-linked injection schedules now require `unit=unit` (was `injection`), and order-linked pills schedules now require `unit=mg` (was `unit`). Clients sending the old values receive `422`. FE must deploy in lockstep.
- Update the injection and pills feature tests.

## Capabilities

### New Capabilities
<!-- none -->

### Modified Capabilities
- `medication-tracking-scheduler`: the expected intake unit for order-linked schedules changes — `INJECTION` from `injection` to `unit`, and `PILLS` from `unit` to `mg`.

## Impact

- `app/Models/MedicationTracking/Enum/DoseUnit.php` — `forProductType()` mapping.
- `app/Services/MedicationTracking/Schedule/Create/Product/ProductConstraintValidator.php` — unchanged logic, now enforces the new units.
- `app/Rules/MedicationTracking/PlannedQuantityForProductType.php` — unchanged (pills stay whole-number).
- `tests/Feature/.../ScheduleController/Product/Injection/CreateTest.php` and `.../Product/Pills/CreateTest.php` — assertions updated.
- API consumers (FE): injection POSTs send `unit=unit`; pills POSTs send `unit=mg`.
