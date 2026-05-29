## MODIFIED Requirements

### Requirement: Create an order-linked medication schedule

When the patient supplies an `order_id`, the system SHALL derive the product from the order (the API MUST NOT accept a `product_id` field), and SHALL reject any field that drifts from the prescribed product. The order MUST be active and owned by the patient.

For order-linked schedules the system SHALL enforce, in addition to the custom-schedule rules:
- `name` matches the product name,
- `schedule_type` is `every_day` for `PILLS`/`DROPS` and `weekly` for `INJECTION`,
- exactly one intake,
- the intake `unit` matches `DoseUnit::forProductType(...)` — which is `unit` for `INJECTION`, `mg` for `PILLS`, and `ml` for `DROPS`,
- `quantity` is within the product type's planned-quantity bounds (`PlannedQuantityForProductType`); for `PILLS` the quantity MUST be a whole number (entered as whole `mg`),
- for `INJECTION`, `schedule_rule.days_of_week` contains exactly one day.

#### Scenario: Patient creates a weekly injection schedule from an order
- **WHEN** the patient POSTs with `order_id` referencing an active injection order, `schedule_type=weekly`, one intake whose `unit=unit`, and `days_of_week=[3]`
- **THEN** the system SHALL respond `201` with `product_id` and `order_id` populated from the resolved order, and `image_path` resolved from the product
- **AND** the schedule row in `medication_tracking_schedules` SHALL carry both `product_id` (denormalized from the order) and `order_id`

#### Scenario: Patient creates an every-day pills schedule from an order
- **WHEN** the patient POSTs with `order_id` referencing an active pills order, `schedule_type=every_day`, and one intake whose `unit=mg` and a whole-number `quantity`
- **THEN** the system SHALL respond `201` with `product_id` and `order_id` populated from the resolved order
- **AND** the schedule row SHALL carry both `product_id` and `order_id`

#### Scenario: Patient sends an `order_id` that does not resolve to an active order they own
- **WHEN** the patient POSTs with an `order_id` that is not theirs, is cancelled, or does not exist
- **THEN** the system SHALL respond `422` with a validation error keyed on `order_id` (`"The selected order is invalid."`), raised by `ScheduleController::resolveOrder(...)`
- **AND** the system SHALL NOT open a database transaction

#### Scenario: Patient sends an order-locked field that drifts from the prescribed product
- **WHEN** the patient POSTs with an `order_id` and any of: a `name` that does not match the product name; a `schedule_type` that is not the product's mandated type; more than one intake; an intake `unit` that does not match `DoseUnit::forProductType(...)` (e.g. `unit=injection` for an injection order, or `unit=unit` for a pills order); an injection schedule with `days_of_week.length != 1`; or a `quantity` outside the product type's planned-quantity bounds (e.g. a fractional `quantity` for pills)
- **THEN** the system SHALL respond `422` with a `ProductConstraintException` message
- **AND** no schedule, intake, or dose row SHALL be persisted

#### Scenario: Patient sends `product_id` directly in the request body
- **WHEN** the patient POSTs a request body containing a `product_id` field
- **THEN** the system SHALL ignore the field — the product is always derived from the resolved order, never from the request
