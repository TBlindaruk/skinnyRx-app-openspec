## Context

`App\Models\MedicationSchedule` is the legacy order-linked schedule model (`medication_schedules` table) that pairs a patient `Order` with its prescribed `Product`. The model declares:

```php
public function product(): BelongsTo
{
    return $this->belongsTo(Product::class);
}

public function getProduct(): Product
{
    /** @var Product */
    return $this->product;
}
```

`App\Models\Product\Product` uses Eloquent `SoftDeletes`, so any `belongsTo(Product::class)` automatically applies `whereNull(products.deleted_at)`. When a product is archived in the catalog (a normal admin action), every `MedicationSchedule` that still references that product has `$this->product === null` and `getProduct()` throws a `TypeError` against its non-null return type.

The schedule's `product_id` itself is never set to null — the DB row still points at the (now-trashed) catalog row. The lifecycle is: schedule outlives product.

Existing project convention (CLAUDE.md §6) explicitly addresses this: *"Soft-deleted relations included with `->withTrashed()` where the domain expects it."* That convention applies here.

## Goals / Non-Goals

**Goals:**
- `MedicationSchedule::getProduct()` returns a non-null `Product` for every schedule that has a `product_id`, regardless of whether the product is soft-deleted.
- Existing consumers continue to work unchanged — no signature ripple.
- Regression coverage prevents this from regressing.

**Non-Goals:**
- Not changing the `Product` model, its soft-delete behavior, or any other `belongsTo(Product::class)` relation elsewhere in the codebase. Each relation can decide independently whether it wants trashed rows.
- Not introducing nullable returns on `getProduct()`. The schedule cannot exist without a product; null would just shift the burden onto every caller.
- Not making `product_id` nullable or back-filling schedules whose product was hard-deleted. Hard deletes shouldn't happen for `Product` (the model is soft-delete-only); if they did, that's a data-integrity bug, not a relation-scoping bug.
- Not touching `MedicationTrackingSchedule` (the new tracker covered by the `medication-tracking-scheduler` spec) — it already exposes `getProduct(): ?Product` (nullable) and handles missing products via the `?->` chain in its resource.

## Decisions

### Decision 1: Use `->withTrashed()` on the relation

```php
public function product(): BelongsTo
{
    return $this->belongsTo(Product::class)->withTrashed();
}
```

**Why this over alternatives:**

- **vs. making `getProduct()` nullable** — would force every caller (`NextDoseReminderCalculator`, `CreateDoseTrackingLogAction`, `CreateOffScheduleTrackingLogAction`) to add null-guards for a case that domain-wise should never happen: a schedule referencing a *gone* product. The domain truth is that the product *exists, it's just retired*.
- **vs. hard-deleting `MedicationSchedule` rows when their product is soft-deleted** — destroys patient history (active dose plans, intake logs that reference the schedule). Patients still need to see/finish their plan.
- **vs. a cascading scope on the model** — overkill for a single relation; no other relation has this problem.

This matches the convention already used elsewhere in the repo for soft-delete-bearing belongsTo relations the domain expects to survive deletes.

### Decision 2: Keep `getProduct(): Product` non-nullable

The return type stays non-null because, after the fix, the only way the relation resolves to null is if `product_id` itself is null (schema-disallowed) or points at a hard-deleted row (not a supported state for `Product`). Callers can rely on the contract.

If a future caller needs to distinguish trashed vs. active, it can call `$schedule->getProduct()->trashed()` or check `getDeletedAt()` — both already exist on `Product`.

### Decision 3: Add one regression test, not a sweep

The fix is a one-line scope change. Coverage needs:
- A schedule whose product is soft-deleted after creation.
- `getProduct()` returns a non-null `Product`.
- The returned product reports `trashed() === true` (sanity check that the fix works through `withTrashed`, not by accident).

No need to test every downstream caller — they all consume read-only fields (`type`, `id`, `image_path`) that survive a soft delete.

## Risks / Trade-offs

- **Risk**: Code elsewhere assumes that any `Product` instance it receives is active.
  → **Mitigation**: surveyed the four `$schedule->getProduct()` consumers (proposal lists them). They read `getType()`, identity, and pass the product to a validator that doesn't care about delete state. No callers branch on "is this product active?" today.

- **Risk**: Eager-loaded queries (`with('product')`) start returning soft-deleted rows where previously they would have been silently dropped — a behavior change.
  → **Mitigation**: that *is* the fix. Today those rows aren't "silently dropped," they trigger a TypeError. The new behavior is strictly better. No production code uses `with(MedicationSchedule::RELATIONSHIP_PRODUCT)` in a way that filters on product activeness.

- **Trade-off**: We diverge slightly from "default Eloquent scoping" — a future reader might be surprised to get a trashed product. The PHPStan `@return BelongsTo<Product, $this>` annotation and the existing CLAUDE.md §6 convention (referenced in the relation comment) document the intent.

## Migration Plan

1. Land the one-line model change + test on `stage` first (matches the trace location).
2. No data migration required; no API contract change.
3. **Rollback**: revert the single commit. The fix is purely a runtime relation scope; no schema or persisted state changes.
