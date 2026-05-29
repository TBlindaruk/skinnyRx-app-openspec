# medication-schedule-product-resolution Specification

## Purpose

Defines the contract for resolving the `Product` related to a (legacy) `App\Models\MedicationSchedule` row. The medication schedule is a long-lived patient artifact — it pairs a patient's `Order` with the prescribed `Product` and drives downstream behavior (dose reminder push notifications via `SendPatientDoseReminderNotification`, tracking-log creation via `CreateDoseTrackingLogAction` and `CreateOffScheduleTrackingLogAction`, schedule-image rendering in resources). Because `Product` uses `SoftDeletes`, the catalog row can disappear from a default `belongsTo` query while the schedule still references it. This capability codifies that the schedule's product MUST remain resolvable for the lifetime of the schedule, regardless of catalog soft-delete state.

## Requirements

### Requirement: Resolve product for the lifetime of the medication schedule

`App\Models\MedicationSchedule::getProduct()` SHALL return a non-null `App\Models\Product\Product` for every persisted schedule, including schedules whose underlying product has been soft-deleted in the `products` table after the schedule was created. The schedule outlives the product's catalog availability; consumers of `getProduct()` MUST be able to rely on a `Product` instance regardless of the product's `deleted_at` state.

#### Scenario: Schedule references an active product

- **WHEN** a `MedicationSchedule` row has `product_id` pointing at a `Product` whose `deleted_at` is null
- **THEN** `getProduct()` returns that `Product` instance
- **AND** `getProduct()->trashed()` returns false

#### Scenario: Schedule references a soft-deleted product

- **WHEN** a `MedicationSchedule` row has `product_id` pointing at a `Product` whose `deleted_at` is set (soft-deleted after the schedule was created)
- **THEN** `getProduct()` returns the same `Product` instance (with its catalog attributes intact) instead of throwing a `TypeError`
- **AND** `getProduct()->trashed()` returns true
- **AND** downstream consumers that read `getType()`, identity, or image path on the returned product continue to function

#### Scenario: Eager loading via the product relation includes soft-deleted rows

- **WHEN** schedules are loaded with `with(MedicationSchedule::RELATIONSHIP_PRODUCT)` or via `whenLoaded(MedicationSchedule::RELATIONSHIP_PRODUCT)` in a resource
- **THEN** the loaded `product` relation includes rows where `products.deleted_at IS NOT NULL`
- **AND** `getProduct()` on those schedules returns a non-null `Product`
