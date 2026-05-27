## Context

`DoseUnit::forProductType()` (in `app/Models/MedicationTracking/Enum/DoseUnit.php`) is the single source of truth for which dose unit an order-linked schedule must use. It currently returns:

```php
match ($type) {
    ProductType::INJECTION => self::INJECTION, // 'injection'
    ProductType::PILLS     => self::UNIT,      // 'unit'
    ProductType::DROPS     => self::ML,        // 'ml'
};
```

`ProductConstraintValidator` reads this value and rejects any intake whose `unit` differs. Two of the three mappings are wrong for the product team's intent: injections should lock to `unit`, and tablets should lock to `mg`. Drops (`ml`) are already correct.

## Goals / Non-Goals

**Goals:**
- Order-linked injection schedules accept and require intake `unit=unit`.
- Order-linked pills schedules accept and require intake `unit=mg`.
- The constraint validator and the create flow stay otherwise unchanged.

**Non-Goals:**
- Removing the `DoseUnit::INJECTION` enum case. It remains a valid value for custom (non-order) schedules where the patient picks any unit.
- Changing the pills quantity rule. Tablets keep whole-number quantities even though the unit label is now `mg` (a patient enters whole `mg`, e.g. `25`).
- Any data migration of existing schedules/doses. This only affects validation of new order-linked schedules.

## Decisions

**Change only the `forProductType()` mapping.** Map `ProductType::INJECTION => self::UNIT` and `ProductType::PILLS => self::MG`; leave `DROPS => self::ML`. The validator already derives the expected unit from this method, so this flips both the accepted values and the rejection messages (`'The intake unit for this medication must be "unit"/"mg".'`).

- Alternative considered: relax the validator to accept multiple units. Rejected — it weakens the product constraint; product wants exactly one unit per type.

**Keep `PlannedQuantityForProductType` untouched.** It enforces whole-number quantity for `ProductType::PILLS`, keyed off the product type, not the unit string. Moving the pills *unit* to `mg` does not change the type, so the whole-number rule still applies — which matches the requirement (tablets entered as whole mg).

**Keep the `INJECTION` enum case.** Still a valid `DoseUnit` for custom schedules; removing it would be a broader breaking change with no benefit here.

## Risks / Trade-offs

- [Breaking change across two product types at once] → Order-linked injection (`injection`→`unit`) and pills (`unit`→`mg`) POSTs with the old units now return `422`. Mitigation: FE deploys in lockstep with this change; called out in the proposal Impact section.
- [Existing rows persisted with the old units] → No migration; only the validation gate changes. Already-stored intakes keep their values. A historical backfill, if wanted, is a separate change and out of scope here.
