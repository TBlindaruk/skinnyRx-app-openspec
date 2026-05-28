## ADDED Requirements

### Requirement: Per-call HTTP request isolation in NotificationClient

`App\Services\Clients\Internal\Notification\Client\NotificationClient` SHALL build a fresh `Illuminate\Http\Client\PendingRequest` for every outbound call to the notification microservice. No HTTP option, no header, and no Guzzle-mergeable state set during one call SHALL be observable on a subsequent call made via the same `NotificationClient` instance.

This requirement exists because `Illuminate\Http\Client\PendingRequest::withHeaders()` mutates the same instance via `array_merge_recursive`, so any reuse of a `PendingRequest` across calls would accumulate header values (and other mergeable Guzzle options) silently across the lifetime of the worker process.

#### Scenario: Two consecutive sends emit distinct X-Request-ID values

- **WHEN** `NotificationClient::send(...)` is called twice in succession on the same client instance, with the HTTP factory faked
- **THEN** both recorded outbound requests carry exactly one `X-Request-ID` header
- **AND** the two `X-Request-ID` header values are different from each other
- **AND** neither `X-Request-ID` header value contains a `,` character (i.e., neither is a comma-joined accumulation of multiple IDs)

#### Scenario: Mixed endpoints on the same client instance do not leak headers

- **WHEN** `NotificationClient::upsertCustomer(...)` is called and then `NotificationClient::send(...)` is called on the same client instance
- **THEN** both recorded requests carry a single, distinct `X-Request-ID` header value
- **AND** neither value contains a `,` character

#### Scenario: Static headers are present on every call

- **WHEN** any of `NotificationClient::send`, `NotificationClient::upsertCustomer`, or `NotificationClient::trackEvent` is invoked
- **THEN** the outbound request carries the `Accept: application/json`, `Content-Type: application/json`, and `User-Agent: InternalAppClient/1.0.0` headers
- **AND** carries an `Authorization: Bearer <token>` header sourced from `App\Services\InternalService\TokenProvider`
- **AND** carries an `X-Request-ID` header whose value matches the `notif_<uniqid>_<8 hex chars>` shape produced by `NotificationClient`

### Requirement: NotificationClient depends on the HTTP factory, not a pre-built PendingRequest

`App\Services\Clients\Internal\Notification\Client\NotificationClient` SHALL receive `Illuminate\Http\Client\Factory`, `App\Services\Clients\Internal\Notification\Config\NotificationConfig`, and `App\Services\InternalService\TokenProvider` via constructor injection. It MUST NOT accept a `PendingRequest` as a constructor parameter. `App\Providers\NotificationServiceProvider` SHALL wire the client with those three collaborators (plus the existing `LoggerInterface` and `PayloadBuilderResolver`) and SHALL NOT construct or cache a `PendingRequest` at provider boot.

#### Scenario: Constructor signature

- **WHEN** PHP reflection inspects `NotificationClient::__construct`
- **THEN** the parameter types are exactly `Illuminate\Http\Client\Factory`, `Psr\Log\LoggerInterface`, `App\Services\Clients\Internal\Notification\Config\NotificationConfig`, `App\Services\InternalService\TokenProvider`, and `App\Services\Clients\Internal\Notification\Client\PayloadBuilder\PayloadBuilderResolver`
- **AND** no parameter has type `Illuminate\Http\Client\PendingRequest`

#### Scenario: Provider wires the new dependencies

- **WHEN** the container resolves `NotificationClient`
- **THEN** `NotificationServiceProvider::registerNotificationClient()` passes a singleton `Illuminate\Http\Client\Factory`, the bound `NotificationConfig`, and the bound `TokenProvider` into the constructor
- **AND** no call to `$httpFactory->baseUrl(...)`, `->timeout(...)`, `->withToken(...)`, or `->withHeaders(...)` occurs inside the provider closure
