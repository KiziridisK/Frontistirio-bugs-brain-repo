# Store Management / Teaching Periods — Bug Report

> The teaching period is the scoping key for the entire platform, so defects here
> ripple everywhere (bootstrap, every period-scoped query).
> File: `controllers/handlers/teaching_periods.js` (+ callers).

---

## SM-01 · HIGH — `getStoreDefaultPeriod` does not filter `isDeleted` → can return a deleted period

**File:** `controllers/handlers/teaching_periods.js` lines 215–229

```js
exports.getStoreDefaultPeriod = async (storeId, session = null) => {
  const query = TeachingPeriod.findOne({ store_id: storeId, default: true });
  // ← no isDeleted: false
};
```

The brain doc (`store-management` and `bootstrap`) states this query includes `isDeleted: false`. It **does not**. Since soft-delete (`deleteStoreTeachingPeriod`) sets `isDeleted: true` but **leaves `default: true` untouched** (see SM-03), this function can return a **soft-deleted** period. Everything in the store then scopes to a deleted period.

**Fix:** `findOne({ store_id: storeId, default: true, isDeleted: { $ne: true } })`.

---

## SM-02 · HIGH — "Exactly one default period" invariant is not enforced

**File:** `controllers/handlers/teaching_periods.js` — `createStoreTeachingPeriods` (line 40) and `editStoreTeachingPeriod` (lines 98–114)

`setDefaultTeachingPeriod` correctly unsets every other period before setting one default. But two other write paths set the `default` flag **directly, without unsetting the others**:

- **Create:** `default: isDefault ?? false` — a new period can be created with `default: true`.
- **Edit:** `...(typeof isDefault !== "undefined" && { default: isDefault })` — any period can be flipped to `default: true` via edit.

So a store can easily end up with **two or more periods marked `default: true`**. `getStoreDefaultPeriod` uses `findOne`, which then returns an arbitrary one — meaning the active period (and therefore all bootstrap data) becomes non-deterministic.

**Fix:** route all default changes through `setDefaultTeachingPeriod`; in create/edit, ignore the incoming `default` flag (or, if `true`, run the unset-others step in the same transaction). A partial unique index `{ store_id: 1 }` on `{ default: true }` would enforce it at the DB level.

---

## SM-03 · MEDIUM — Deleting the default period leaves the store with a dangling/absent default

**File:** `controllers/handlers/teaching_periods.js` `deleteStoreTeachingPeriod` lines 127–162

Soft delete sets `isDeleted/deletedAt` but does **not** clear `default` and does **not** reassign a new default. There is also no guard preventing deletion of the currently-active period. After deleting the default period:
- with SM-01 unfixed → `getStoreDefaultPeriod` still returns the deleted period;
- with SM-01 fixed → `getStoreDefaultPeriod` returns `null` → store-user bootstrap silently empty, student bootstrap crashes (see `bootstrap-bugs.md` BS-04).

**Fix:** refuse to delete the active default (or, on delete, unset `default` and promote another non-deleted period to default within the transaction).

---

## SM-04 · LOW — `session = {}` default parameter is passed to Mongoose as a session

**File:** `controllers/handlers/teaching_periods.js` — `createStoreTeachingPeriods` (line 13), `deleteStoreTeachingPeriod` (line 132), `setDefaultTeachingPeriod` (line 176)

Defaults are `session = {}` (an empty object) rather than `null`. Internally these use `session || undefined` / `{ session }`, and `{}` is truthy, so an empty object can be handed to `.save({ session: {} })` / `.session({})`. In practice the controllers always pass a real session, so this is latent — but if any handler is ever called without a session it will pass `{}` into Mongoose. Use `session = null` for consistency with the rest of the codebase.

---

## Summary

| ID | Severity | Description |
|---|---|---|
| SM-01 | HIGH | `getStoreDefaultPeriod` missing `isDeleted` filter → can return a deleted period (doc says otherwise) |
| SM-02 | HIGH | create/edit set `default` directly without unsetting others → multiple defaults possible |
| SM-03 | MEDIUM | Soft-deleting the default period doesn't clear/reassign default → no usable active period |
| SM-04 | LOW | `session = {}` default param can be passed into Mongoose as a session |
