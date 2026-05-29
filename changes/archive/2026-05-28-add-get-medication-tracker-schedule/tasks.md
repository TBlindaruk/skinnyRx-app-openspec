## 1. Route

- [x] 1.1 Add `Route::get('/', [ScheduleController::class, 'get'])->name('get')` inside the existing `{schedule}` group (`'middleware' => 'can:manage,schedule'`) in `routes/api/api.php`, sitting next to the existing `DELETE`.

## 2. Controller

- [x] 2.1 Add `public function get(MedicationTrackingSchedule $schedule): JsonResponse` to `app/Http/Controllers/Api/Patient/MedicationTracking/ScheduleController.php`.
- [x] 2.2 Inside the method, call `$schedule->load([MedicationTrackingSchedule::RELATIONSHIP_INTAKES, MedicationTrackingSchedule::RELATIONSHIP_PRODUCT])` and return `$this->responseFactory->json(ScheduleResource::make($schedule))`.
- [x] 2.3 Add an `#[OA\Get(...)]` attribute mirroring the list endpoint's style: path `/api/v1/patient/medication-tracking/schedules/{schedule}`, `sanctum` security, tag `Patient/MedicationTracking/Schedule`, `schedule` path parameter (integer), and responses for `200` (single `ScheduleResource`), `401` (`UnauthorizedError`), `403` (forbidden — not the owner), `404` (`NotFoundError`).

## 3. Tests

- [x] 3.1 Create `tests/Feature/Api/Patient/MedicationTracking/Schedule/GetScheduleTest.php` extending the project's `TestCase` and using `LazilyRefreshDatabase`. (Placed at `tests/Feature/Http/Controllers/Api/Patient/MedicationTracking/ScheduleController/GetTest.php` to match the existing convention used by sibling `IndexTest.php` / `CreateTest.php`. `LazilyRefreshDatabase` is on the project `TestCase`.)
- [x] 3.2 Cover: owning patient fetches a custom schedule → `200`, response shape matches one item from the list endpoint (compare structural keys + `id`, `product_id=null`, `order_id=null`, `image_path=null`, populated `intakes`).
- [x] 3.3 Cover: owning patient fetches an order-linked schedule → `200`, `product_id`/`order_id` populated, `image_path` resolved.
- [x] 3.4 Cover: another patient → `403`.
- [x] 3.5 Cover: non-existent id → `404`.
- [x] 3.6 Cover: soft-deleted schedule → `404` (use `MedicationTrackingScheduleFactory` then `$schedule->delete()`).
- [x] 3.7 Cover: unauthenticated request → `401`.

## 4. Verification

- [x] 4.1 Manually verify response equality against the list endpoint: pick one schedule id, hit both `GET /schedules?id[]=N` and `GET /schedules/N`, confirm the show response equals the single item in the list's `data[0]`. (Covered programmatically in test 3.2 — `assertSame($listItem, $showItem)` asserts byte-for-byte equality against `data.0` of the list filtered by `?id[]=N`.)
