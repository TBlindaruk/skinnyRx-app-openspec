## ADDED Requirements

### Requirement: Show a single medication schedule

The system SHALL expose `GET /api/v1/patient/medication-tracking/schedules/{schedule}` returning a single `ScheduleResource` for the schedule identified by the path. The shape of the response item MUST be identical to one item from `GET /schedules` — i.e. `id`, `name`, `product_id`, `order_id`, `schedule_type`, `schedule_rule`, `intakes` (eager-loaded array of `ScheduleIntakeResource`), `starts_on`, `ends_on`, `timezone`, `image_path` (resolved from the linked product, or `null` for custom schedules), and `created_at`.

Authorization MUST go through the existing `can:manage,schedule` policy — only the owning patient SHALL receive `200`; any other authenticated patient SHALL receive `403`. The route binding MUST NOT resolve soft-deleted schedules, so a deleted schedule (or a non-existent id) SHALL return `404`. Unauthenticated requests SHALL return `401`.

The controller MUST eager-load the `intakes` and `product` relations before passing the model to `ScheduleResource` so that `intakes` and `image_path` are populated; lazy loading is not acceptable because `whenLoaded` would silently return an empty intakes array.

#### Scenario: Owning patient fetches a custom schedule
- **WHEN** the owning patient GETs `/api/v1/patient/medication-tracking/schedules/{schedule}` for a custom (no order) schedule they own
- **THEN** the system SHALL respond `200` with a single `ScheduleResource`
- **AND** the response SHALL contain `product_id = null`, `order_id = null`, `image_path = null`, `intakes` populated from the persisted intakes, and the same `created_at`/`starts_on`/`timezone` echo as the list endpoint would return

#### Scenario: Owning patient fetches an order-linked schedule
- **WHEN** the owning patient GETs the endpoint for an order-linked schedule
- **THEN** the system SHALL respond `200` with `product_id` and `order_id` populated and `image_path` resolved from the linked product (same value the list endpoint produces for the same schedule)

#### Scenario: Different patient tries to fetch
- **WHEN** an authenticated patient GETs a schedule whose `patient_id` is not theirs
- **THEN** `can:manage,schedule` SHALL return `403`
- **AND** no schedule data SHALL be returned in the response body

#### Scenario: Schedule does not exist
- **WHEN** the patient GETs a `{schedule}` id that does not exist
- **THEN** Laravel's implicit route binding SHALL return `404` before the controller runs

#### Scenario: Schedule is soft-deleted
- **WHEN** the patient GETs a `{schedule}` id whose row has `deleted_at` set
- **THEN** the implicit route binding SHALL NOT resolve the trashed row and the system SHALL respond `404`
- **AND** the response SHALL NOT leak any field of the deleted schedule

#### Scenario: Unauthenticated request
- **WHEN** the endpoint is called without a valid patient session
- **THEN** the system SHALL respond `401` with the standard `UnauthorizedError` payload
