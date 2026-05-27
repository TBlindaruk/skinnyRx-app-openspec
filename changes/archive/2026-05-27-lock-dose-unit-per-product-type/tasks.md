## 1. Fix the mapping

- [x] 1.1 In `app/Models/MedicationTracking/Enum/DoseUnit.php`, change `forProductType()` to map `ProductType::INJECTION => self::UNIT` and `ProductType::PILLS => self::MG` (keep `DROPS => self::ML`)

## 2. Update injection tests

- [x] 2.1 In `tests/Feature/.../ScheduleController/Product/Injection/CreateTest.php`, change the successful-create intake from `DoseUnit::INJECTION->value` to `DoseUnit::UNIT->value`
- [x] 2.2 In the same file, update the constraint-violation test so it uses a now-wrong unit for injection (e.g. `DoseUnit::INJECTION->value`) and still asserts the `422` `ProductConstraintException`

## 3. Update pills tests

- [x] 3.1 In `tests/Feature/.../ScheduleController/Product/Pills/CreateTest.php`, change the successful-create intake from `DoseUnit::UNIT->value` to `DoseUnit::MG->value`
- [x] 3.2 In the same file, update the constraint-violation test so it uses a now-wrong unit for pills (e.g. `DoseUnit::UNIT->value`) and still asserts the `422` `ProductConstraintException`
- [x] 3.3 Confirm the existing whole-number-quantity test for pills still passes (rule is unchanged)

## 4. Verify

- [x] 4.1 Run the medication-tracking schedule feature tests (injection + pills) and confirm they pass
