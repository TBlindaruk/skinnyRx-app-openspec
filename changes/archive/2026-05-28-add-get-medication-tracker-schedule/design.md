## Context

`GET /api/v1/patient/medication-tracking/schedules` (list) already returns the full `ScheduleResource` with `intakes` and `image_path` resolved. The FE wants to deep-link to a single schedule (refresh, detail view, share-style URLs) without filtering the list. The same `{schedule}` path already exists for `DELETE` and intake operations, and it sits behind `can:manage,schedule` + Laravel's implicit binding — we just need a `Route::get(...)` on the same group.

The existing `MedicationTrackingSchedule` model is soft-deletable and the route binding does not resolve trashed rows, which is exactly the behavior described in the spec for `DELETE` on an already-deleted schedule (`404`). The same will apply naturally to `GET`.

## Goals / Non-Goals

**Goals:**
- One new `GET /schedules/{schedule}` endpoint returning the same `ScheduleResource` shape used by the list.
- Reuse the existing `can:manage,schedule` policy for authorization (`403` for non-owners).
- Reuse the existing implicit binding (`404` for missing or soft-deleted schedules).
- Match the list endpoint's eager-loading (`intakes` + `product`) so `image_path` and `intakes` are populated.

**Non-Goals:**
- No new fields, no new DTO, no new resource class — strictly the existing `ScheduleResource`.
- No filter/include query params — this is a single-item lookup; the list endpoint already handles selective fetch via `?id[]=`.
- No surfacing of soft-deleted schedules. If the FE ever needs deleted history, that's a separate change.

## Decisions

### Add the route inside the existing `{schedule}` group (behind `can:manage,schedule`)

The route group at `routes/api/api.php:879-891` is already gated by `'middleware' => 'can:manage,schedule'`. Placing the new `GET` there is the lowest-noise way to get authorization for free. Alternative: a top-level `Route::get('/{schedule}', ...)` outside the group with a manual policy call inside the controller — rejected, because it duplicates what middleware already does and diverges from how `DELETE` is wired today.

### Reuse `ScheduleResource` verbatim (no separate "show" resource)

The user explicitly asked for "the same response as the list". The list passes each item through `ScheduleResource::make(...)`; the show endpoint will do exactly the same. Alternative: a richer resource with extra fields (e.g. expanded order or product details) — out of scope; the FE has not asked for it.

### Eager-load `intakes` + `product` before resourcing

`ScheduleResource` reads `whenLoaded(RELATIONSHIP_INTAKES)` and reads `getProduct()?->getImagePath()` directly. The list endpoint loads both relations to avoid N+1 and to make `image_path` resolvable. We do the same here with `$schedule->load([...])` after the route binding resolves the model. Alternative: rely on lazy loading — rejected, because `whenLoaded` would return an empty array if `intakes` is not eager-loaded, silently dropping data.

### `404` for soft-deleted schedules (no `withTrashed()`)

The route binding does not opt into `withTrashed()`, so a deleted schedule returns `404` automatically. This matches the existing `DELETE` behavior described in the spec ("Deleting an already-deleted schedule" returns `404`) and keeps the contract simple: "deleted means gone for the FE". Alternative: surface trashed rows with a `deleted_at` field — rejected, no consumer asked for it.

## Risks / Trade-offs

- [Risk] Implicit route binding on a soft-deletable model that isn't covered by tests today would silently change behavior if someone later adds `withTrashed()` to the binding. → Mitigation: feature test explicitly asserts `404` on a soft-deleted schedule, locking the contract.
- [Trade-off] No "include" query params — the response is fixed-shape. If the FE later needs (e.g.) the linked order's status, we'd add it explicitly to `ScheduleResource` rather than introducing dynamic includes.

## Migration Plan

- Purely additive — no DB, no config, no contract changes for existing endpoints.
- Deploy with the regular release; no feature flag.
- Rollback: revert the route + controller method + test file. Nothing else touches state.
