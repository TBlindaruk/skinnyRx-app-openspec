## Context

The patient scheduler endpoints (both legacy `MedicationSchedule/*` and the new `MedicationTracking/*`) validate the incoming `timezone` field with Laravel's built-in `timezone:all` rule. That rule resolves to `DateTimeZone::ALL` and therefore rejects every entry in PHP's BC alias set — including `Europe/Kiev`, which is still what some browsers, mobile OSes, and older clients send. The result is a `422` for a value that PHP itself parses without complaint.

Downstream code uses the stored string only through `new DateTimeZone($schedule->timezone)` / Carbon's wrapper, both of which accept BC names transparently and produce identical dose-window math. There is no existing query that compares the column against a literal canonical name. The bug is purely at the validator.

An earlier iteration of this change introduced a `TimezoneCanonicalizer` helper, a custom `Timezone` rule, and `prepareForValidation()` on each FormRequest to map BC → canonical before persistence. That approach was rejected as over-engineered for the problem: it added ~150 lines of vendored alias data plus per-request normalization code, in exchange for "clean" storage that no part of the system actually depends on. The simpler swap to `timezone:all_with_bc` does exactly what the user needs.

## Goals / Non-Goals

**Goals:**
- Accept deprecated IANA names (`Europe/Kiev` etc.) on every patient scheduler FormRequest that currently uses `timezone:all`.
- Keep the existing `422` behavior for truly unknown strings.
- Minimize code change so the diff is reviewable in seconds.

**Non-Goals:**
- No canonicalization. The stored / echoed value equals whatever the client sent; the system does not promise a canonical form anywhere.
- No backfill of existing rows. (They were never broken — PHP handles both forms.)
- No change to appointment FormRequests or the manager availability-slot endpoint. Same one-line fix could ship there, but the user's report was scoped to schedulers and broadening the diff risks surprising those flows.
- No new helper/rule classes, no DI wiring, no `prepareForValidation()` hooks. The framework's built-in rule already does the right thing under `:all_with_bc`.

## Decisions

### Swap `timezone:all` → `timezone:all_with_bc`, do nothing else
Laravel's `validateTimezone` reads the parameter as the `DateTimeZone::*` constant name. Passing `all_with_bc` makes the rule call `timezone_identifiers_list(DateTimeZone::ALL_WITH_BC)`, which includes canonical zones, BC aliases (`Europe/Kiev`, `US/Eastern`, …), POSIX-style entries (`CET`, `EST`, `Etc/GMT+5`), and synonym names (`UTC`, `GMT`, `Zulu`, …). All of them parse cleanly through `new DateTimeZone(...)` and Carbon, so accepting the broader set is safe.

We deliberately do not split "BC aliases we like" from "POSIX entries we don't like" — the only honest line is "what PHP recognizes." A client sending `CET` gets the same DST behavior PHP gives Carbon for that zone; if business cares about a narrower set later, that's a separate change.

### Storage stays as-sent; no canonicalization
The `timezone` column on `medication_tracking_schedules` and `medication_schedules` is read only by code that constructs a `DateTimeZone` from it. PHP's constructor is BC-tolerant and produces identical transitions for `Europe/Kiev` and `Europe/Kyiv`, so dose-list queries, "next dose" calculations, day-window floors, and dose-resource rendering all work identically regardless of which form is stored. The hypothetical downside — two patients in Kyiv stored under different strings — has no functional consequence and no current consumer (no `GROUP BY timezone` analytics, no string-equality joins). If a consumer ever needs canonical storage, a one-shot backfill is straightforward.

### Apply only to scheduler FormRequests
The user's report named the schedulers. Appointment FormRequests use `'timezone'` (equivalent to `DateTimeZone::ALL` — same rejection of `Europe/Kiev`) but no incident is open against them; the manager availability-slot endpoint uses `Rule::in($allowedTimezones)` with a hand-picked set, which is a different policy. Touching them in the same change would risk regressions in flows that nobody reported broken. Same one-line fix is available if/when needed.

## Risks / Trade-offs

- **Mixed-string storage** → no consumer depends on canonical form; if one appears later, run a one-shot `UPDATE` that maps BC→canonical for the affected rows. Cost is bounded by `SELECT DISTINCT timezone` cardinality (tiny).
- **POSIX-style names (`CET`, `EST`, `Etc/GMT+5`) now also pass** → these are still real PHP zones, parse fine, and produce sensible behavior; the previous "they are blocked" behavior wasn't load-bearing. If business later wants to constrain to "real IANA names only," that's a different rule (`timezone:all` would re-establish it).
- **Client display** → response echoes the request value (`Europe/Kiev` in, `Europe/Kiev` out). Frontend that displays the timezone verbatim will show the alias to the user. Considered acceptable — the client sent that string, so they presumably can render it.
- **Edge: case differences (`europe/kiev`)** → IANA names are case-sensitive in PHP; `timezone:all_with_bc` still rejects mis-cased inputs. Test covers this.

## Migration Plan

1. Ship the five-line change.
2. Deploy. New schedules and queries with `Europe/Kiev` (and any other BC name) immediately succeed.
3. No data migration. No rollback step beyond reverting the PR.
