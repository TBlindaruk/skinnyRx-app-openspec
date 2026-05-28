## 1. Swap NotificationClient's HTTP dependency

- [x] 1.1 In `app/Services/Clients/Internal/Notification/Client/NotificationClient.php`, change the constructor to inject `Illuminate\Http\Client\Factory $httpFactory`, `Psr\Log\LoggerInterface $logger`, `App\Services\Clients\Internal\Notification\Config\NotificationConfig $config`, `App\Services\InternalService\TokenProvider $tokenProvider`, and `App\Services\Clients\Internal\Notification\Client\PayloadBuilder\PayloadBuilderResolver $payloadBuilderResolver` (all `private readonly`). Remove the `Illuminate\Http\Client\PendingRequest $http` parameter and the `use Illuminate\Http\Client\PendingRequest;` import.
- [x] 1.2 In `NotificationClient::request()`, replace `$this->http->withHeaders(['X-Request-ID' => $requestId])->retry(...)` with a fresh per-call builder: `$this->httpFactory->baseUrl($this->config->url)->timeout($this->config->timeout)->connectTimeout($this->config->connectTimeout)->withToken($this->tokenProvider->provide())->withHeaders(['Accept' => 'application/json', 'Content-Type' => 'application/json', 'User-Agent' => 'InternalAppClient/1.0.0', 'X-Request-ID' => $requestId])->retry(...)->post($endpoint, $payload)`. Keep the existing retry callbacks, `throw: false`, the success/failure branches, the logger calls, the `extractErrorDetails` flow, and the `NotificationApiException` wrapping byte-for-byte unchanged.
- [x] 1.3 Confirm `final readonly class NotificationClient` stays `final readonly` and that all class-level imports are tidy (drop `PendingRequest`, add `Illuminate\Http\Client\Factory as HttpFactory`, `App\Services\Clients\Internal\Notification\Config\NotificationConfig` already imported, add `App\Services\InternalService\TokenProvider`).

## 2. Rewire NotificationServiceProvider

- [x] 2.1 In `app/Providers/NotificationServiceProvider.php::registerNotificationClient()`, delete the `$http = $httpFactory->baseUrl(...)->timeout(...)->connectTimeout(...)->withToken(...)->withHeaders([...])` chain.
- [x] 2.2 Update the closure to resolve `HttpFactory`, the existing `NotificationConfig`, the existing `TokenProvider`, `LoggerInterface`, and `PayloadBuilderResolver`, and pass them to the new `NotificationClient` constructor via named arguments (one per line, matching CLAUDE.md §3 examples).
- [x] 2.3 Leave the `singleton(NotificationClient::class, …)` binding in place. Leave `provides()` unchanged. Leave `registerNotificationConfig()` and `registerTemplateNotificationFactory()` untouched.

## 3. Regression test: per-call X-Request-ID isolation

- [x] 3.1 Create `tests/Unit/Services/Clients/Internal/Notification/Client/NotificationClientTest.php` extending `Tests\TestCase`. Use `Illuminate\Http\Client\Factory::fake()` (or a hand-built `Factory` with `fake()`) so outbound HTTP is recorded, not sent. Resolve `NotificationClient` through the container so it picks up the real provider wiring.
- [x] 3.2 Add `send_whenCalledTwiceOnSameInstance_shouldSendDistinctXRequestIdHeaders`: build a minimal `NotificationRequestDTO` (any channel/recipient combination accepted by `PayloadBuilderResolver`), call `$client->send($dto)` twice, then assert via the factory's `recorded()` (or `assertSent` introspection) that both recorded requests have an `X-Request-ID` header, the two values differ, and neither value contains `,`.
- [x] 3.3 Add `send_shouldIncludeStaticHeadersAndBearerToken`: stub `TokenProvider::provide()` to return a deterministic value, call `$client->send($dto)` once, and assert the recorded request carries `Accept: application/json`, `Content-Type: application/json`, `User-Agent: InternalAppClient/1.0.0`, `Authorization: Bearer <stubbed token>`, and an `X-Request-ID` whose value matches `/^notif_[0-9a-f]+_[0-9a-f]{8}$/`.
- [x] 3.4 Match the file's neighbouring test style (PHPUnit 10, `#[Test]` attributes, snake_case-with-camelCase method naming as used in `tests/Unit/Services/Clients/Internal/Notification/Client/PayloadBuilder/PayloadBuilderResolverTest.php`). Do **not** introduce Pest.

## 4. Verify

- [x] 4.1 Run `php artisan test --filter=NotificationClientTest` and confirm both new test cases pass. Verified — `Tests: 2 passed (8 assertions)` via `docker exec skinny-app …` in session.
- [ ] 4.2 Run `php artisan test --filter='Notification|Send.*Notification'` to confirm the existing notification job/listener feature tests still pass with the new constructor signature.
- [ ] 4.3 Run `make run-phpstan` and `make run-psalm` and confirm no new entries land in `phpstan-baseline.neon` / `psalm-baseline.xml`.
- [x] 4.4 Grep the repo for `new NotificationClient(` outside `NotificationServiceProvider.php` and confirm there are no other direct constructions (the only legitimate call site is the provider closure). Verified — only call site is `NotificationServiceProvider.php:58`.
- [ ] 4.5 After deploy to stage, confirm the notification service stops emitting `Idempotency: concurrent request rejected` with comma-joined `request_id` values for `/api/v1/notifications/send-template`.
