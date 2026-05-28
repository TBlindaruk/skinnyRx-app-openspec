## Why

Stage is logging `Idempotency: concurrent request rejected` from the internal notification microservice, with a `request_id` field that contains **seven different IDs** comma-joined:

```
stage.INFO: Idempotency: concurrent request rejected
{
  "request_id": "notif_6a182e5c9d48c_e4b82cbf, notif_6a182e60c3eb5_91cfd7fd, notif_6a182e61dcbb4_13a59726, ...",
  "endpoint": "/api/v1/notifications/send-template"
}
```

That field is the value of the `X-Request-ID` header **we** send, not something the notification service composes. So the notification service is correctly logging exactly one header value per request — the bug is on our side: we are sending a comma-joined header value.

Root cause (`app/Services/Clients/Internal/Notification/Client/NotificationClient.php`, `app/Providers/NotificationServiceProvider.php`):

- `NotificationClient` is bound as a `singleton` and receives a single `Illuminate\Http\Client\PendingRequest` instance in its constructor.
- Every call to `NotificationClient::send|upsertCustomer|trackEvent` ultimately calls `$this->http->withHeaders(['X-Request-ID' => $requestId])` on the **same** `PendingRequest`.
- Laravel's `PendingRequest::withHeaders()` merges options with `array_merge_recursive`, so repeated calls with the same header key produce an array of values, which Guzzle then transmits as a comma-separated header.
- In Horizon workers the singleton lives for the entire worker lifetime, so each subsequent job accumulates another `X-Request-ID`. After N jobs the notification service receives N comma-joined IDs — and rejects the request because the de-duplicated idempotency key matches an in-flight one.

This means every notification after the first in a worker process is at risk of being silently rejected for "concurrent request" reasons, while emitting noisy alerts on the notification side.

## What Changes

- Stop sharing one `PendingRequest` across notification calls. `NotificationClient` is changed to depend on `Illuminate\Http\Client\Factory` (plus the typed config and token provider it already needs), and constructs a fresh `PendingRequest` inside `request()` for every outbound call.
- `NotificationServiceProvider::registerNotificationClient()` is updated to inject the factory + `NotificationConfig` + `TokenProvider` instead of a pre-built `PendingRequest`. The base URL, timeouts, bearer token, and the static `Accept`/`Content-Type`/`User-Agent` headers move into the per-call builder.
- Each call still sends exactly one `X-Request-ID` header — its value is a single freshly generated ID, no comma-joining.
- Add a unit test for `NotificationClient` that pins this behaviour: when two consecutive `send()` calls share a client instance, the recorded outbound requests carry **different** `X-Request-ID` values and neither header contains a comma.
- No public surface change. `NotificationClient::send|upsertCustomer|trackEvent` keep the same signatures and the same DTOs. No consumer (`SendNotificationApi`, listeners, jobs) is touched.

No behaviour change for any caller. No config changes, no env changes, no API contract change with the notification service.

## Capabilities

### New Capabilities

- `notification-client-http-isolation`: rule that every outbound HTTP request from `NotificationClient` is built from a fresh `PendingRequest` and carries a single, unique `X-Request-ID` header value.

### Modified Capabilities

<!-- none -->

## Impact

- **Code touched**:
  - `app/Services/Clients/Internal/Notification/Client/NotificationClient.php` — swap injected `PendingRequest` for `Illuminate\Http\Client\Factory`, `NotificationConfig`, `TokenProvider`; build the per-call request inside `request()`.
  - `app/Providers/NotificationServiceProvider.php` — drop the pre-built `PendingRequest` chain inside `registerNotificationClient`; pass the factory + config + token provider directly.
- **Tests**: one new unit test under `tests/Unit/Services/Clients/Internal/Notification/Client/NotificationClientTest.php` covering header isolation between calls. The existing `SendNotificationApiTest` keeps mocking `NotificationClient` and is unaffected.
- **Callers**: none. `NotificationClient` is the only consumer of the singleton `PendingRequest`; its own public methods are unchanged.
- **External systems**: stops emitting the spurious `Idempotency: concurrent request rejected` log lines on the notification service and stops their false-positive idempotency rejections; no contract change.
- **Risk**: low — change is contained to one client class and its provider; behaviour is verified by a targeted unit test using `Http::fake()` recording.
