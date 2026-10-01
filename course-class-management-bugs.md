# Course & Class Management — Bug Report

> Files: `controllers/handlers/course.js`, `controllers/handlers/class.js`
> (controllers `controllers/course.js`, `controllers/class.js`).

---

## CC-01 · MEDIUM (security) — `editStoreCourse` / `editStoreClass`: mass assignment + IDOR

**Files:** `controllers/handlers/course.js` lines 54–70; `controllers/handlers/class.js` lines 49–64

```js
exports.editStoreCourse = async (courseData) => {
  const { _id, ...updates } = courseData;
  return Course.findByIdAndUpdate(_id, { $set: updates }, { new: true });
};
// editStoreClass is identical for ClassModel
```

Two issues:
1. **No store scoping** — the update keys only on `_id`. A store-user can edit **any** course/class in the database (including other stores') if they know/iterate the `_id`.
2. **Mass assignment** — `updates` is the raw client object minus `_id`, spread straight into `$set`. A client can overwrite `store_id`, `period_classes`/`period_teachers`/`period_students`, `isDeleted`, `createdBy`, etc. — e.g. silently re-home a course to another store or wipe its period relationships.

**Fix:** scope the query (`findOneAndUpdate({ _id, store_id: req.user.store }, …)`) and whitelist editable fields (`name`, `description`, `grade_id`, `grade_scale`, `grade_scenario`).

---

## CC-02 · ~~MEDIUM~~ — Soft-deleted courses/classes are returned (no `isDeleted` filter) — ✅ FIXED (2026-07-14 + 2026-09-26)

**Files:**
- `course.js` `fetchStorePeriodCourses` (line 112 `$match: { store_id }`) and `fetchStoreCourses` (line 74 `Course.find({ store_id })`)
- `class.js` `fetchStorePeriodClasses` (line 141–142 `$match: { store_id }`)

None of the list/aggregation queries filter `isDeleted`. Soft-deleted courses and classes are loaded into the bootstrap payload and NgRx store. The brain-doc soft-delete contract ("all queries include `isDeleted:false`") is not honored on the read path. (Same family as student `isDeleted` leak — see `student-management-bugs.md`.)

**Fix:** add `isDeleted: { $ne: true }` to the `$match` / `find`.

**Fixed in two rounds:**
- **2026-07-14** (indexing pass): `fetchStorePeriodCourses` / `fetchStorePeriodClasses` — the
  aggregations that actually feed bootstrap — now `$match { store_id, isDeleted: { $ne: true } }`.
- **2026-09-26:** the flat `fetchStoreCourses` / `fetchStoreClasses` finds too. Measured on the dev
  DB before/after: courses `102 → 99` returned (3 soft-deleted stopped shipping), classes unchanged
  (none deleted there), and every returned row has `isDeleted` falsy.

---

## CC-03 · MEDIUM — `getClassesByIds([period_classes.classes])` wraps a single id in an array (bootstrap student path)

**File:** `controllers/bootstrap.js` line 153, consuming `class.js` `getClassesByIds`

The student bootstrap passes `[period_classes.classes]` — `period_class.classes` is a single ObjectId (per the model), so this builds a one-element array. When the field is missing/undefined it becomes `[undefined]`, which `getClassesByIds` converts to `new ObjectId(undefined)` → throws/CastError. (Root cause shared with `bootstrap-bugs.md` BS-01.) Guard the value before wrapping.

---

## CC-04 · LOW — `importCourses` duplicate-check matches on name only, ignores soft-delete

**File:** `controllers/handlers/course.js` lines 246–261

```js
const existing = await Course.findOne({
  store_id, name: oldCourse.name, grade_id: oldCourse.grade_id,
  "period_classes.period": periodId,
});
if (existing) continue;
```

Dedup is by `(store_id, name, grade_id, period present)`. It does not consider `isDeleted`, so a soft-deleted course with the same name blocks re-import; and two courses with the same name but different scale/scenario collapse. Also `period_teachers`/`period_students` of the source are intentionally **not** copied (empty arrays) — correct per design, noted for clarity.

---

## CC-05 · LOW — `removeCourseClasses` empty-entry cleanup can delete a freshly-created period entry

**File:** `controllers/handlers/course.js` lines 447–460

After `$pull`-ing the class, it `$pull`s any `period_classes` subdoc whose `classes` array is now empty for that period. If a period entry legitimately exists with zero classes (e.g. just created via `updateCourseClasses` step 1, or after removing the last class), the cleanup removes the period bucket entirely. The next assignment re-creates it, so it's self-healing, but it makes `period_classes` entries flthat flap in/out and complicates the `period present` checks used elsewhere (e.g. CC-04's dedup, `fetchStorePeriodCourses.hasPeriod`).

---

## Note — `watchCourses` / `watchClasses` update leak

The `update` branch of both watchers has the `filter(async …)` cross-store leak and the dotted-path delta-merge problem. Filed centrally in `real-time-sync-bugs.md` (RT-01, RT-04). The bidirectional period updates are exactly the nested-array updates that RT-04 fails to sync live.

---

## Summary

| ID | Severity | Description |
|---|---|---|
| CC-01 | MEDIUM | `editStoreCourse`/`editStoreClass` — no store scoping + mass assignment (IDOR) |
| CC-02 | MEDIUM | Course/class list & period aggregations omit `isDeleted` → soft-deleted records leak |
| CC-03 | MEDIUM | Bootstrap `getClassesByIds([period_classes.classes])` breaks when class missing/undefined |
| CC-04 | LOW | `importCourses` dedup ignores soft-delete and scale/scenario |
| CC-05 | LOW | `removeCourseClasses` cleanup removes empty period buckets (flapping) |
