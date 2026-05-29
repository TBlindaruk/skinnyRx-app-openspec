# timezone-normalization Specification

## Purpose
Accept inbound timezone identifiers on the medication-tracker FormRequests, normalize deprecated IANA aliases (e.g. `Europe/Kiev`) to their canonical form (e.g. `Europe/Kyiv`) at the request boundary before validation runs, and guarantee that validation, persistence, resources, and scheduling only ever see canonical identifiers. This prevents otherwise-valid requests from failing `timezone:all_with_bc` validation on PHP/tzdata builds that drop backward-compatibility links.

## Requirements
### Requirement: Deprecated timezone aliases are normalized to canonical identifiers

The system SHALL normalize a deprecated IANA timezone alias supplied in an inbound `timezone` field to its canonical identifier before that value is validated, persisted, or used downstream. At minimum `Europe/Kiev` MUST be normalized to `Europe/Kyiv`.

#### Scenario: Deprecated alias is upgraded to canonical

- **WHEN** a request supplies `timezone` as `Europe/Kiev`
- **THEN** the value seen by validation and all downstream code is `Europe/Kyiv`

#### Scenario: Canonical identifier is left unchanged

- **WHEN** a request supplies `timezone` as `Europe/Kyiv` or `America/New_York`
- **THEN** the value is passed through unchanged

#### Scenario: Unknown or unmapped value is left unchanged

- **WHEN** a request supplies a `timezone` value that is not in the alias map
- **THEN** the value is passed through unchanged so existing validation still decides its validity

### Requirement: Normalized deprecated aliases pass timezone validation

The system SHALL accept a request whose `timezone` field is a deprecated alias that maps to a valid canonical identifier, on any supported PHP/tzdata build, without returning a timezone validation error.

#### Scenario: Previously-rejected alias now validates

- **WHEN** a request is sent with `timezone` `Europe/Kiev` to an endpoint that validates timezone with `timezone:all_with_bc`
- **THEN** the request does not fail with `The timezone field must be a valid timezone`
- **AND** the request proceeds with `Europe/Kyiv`

#### Scenario: Genuinely invalid timezone is still rejected

- **WHEN** a request is sent with `timezone` `Not/AZone`
- **THEN** the request still fails timezone validation

### Requirement: Normalization applies across both medication trackers

The system SHALL apply the same timezone normalization to every medication-tracker FormRequest that accepts a `timezone` field — both the old `MedicationSchedule` tracker and the new `MedicationTracking` tracker — using one shared seam. This includes GET endpoints that take `timezone` as a query parameter.

#### Scenario: Old tracker normalizes

- **WHEN** the old-tracker schedule endpoint receives `Europe/Kiev`
- **THEN** it normalizes the value to `Europe/Kyiv`

#### Scenario: New tracker normalizes

- **WHEN** a new-tracker schedule or scheduled-dose endpoint receives `Europe/Kiev` (in body or query)
- **THEN** it normalizes the value to `Europe/Kyiv` using the same shared seam
