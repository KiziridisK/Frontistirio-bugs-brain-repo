# Student Management — Bug Report

> Files: `controllers/students.js`, `controllers/handlers/students.js`.

---

## ST-01 · MEDIUM (security/IDOR) — Student detail updates are not scoped to the caller's store

**File:** `controllers/students.js` `upsertStudentDetails` (lines 367–456) → handler `updateStudentRelations` (lines 910+), and `courseHandler.updateCourseStudentRelation`

`upsertStudentDetails` takes `body.studentId` and immediately calls `updateStudentRelations(student_id, periodId, changes, …)`. That handler operates purely on `studentObjectId` — it **never filters by `store_id`**. Likewise `updateCourseStudentRelation` operates on a raw `courseId`. So an authenticated store-user can pass **another store's** `studentId`/`courseId` and rewrite their grade/class/course assignments (cross-store write / IDOR).

Contrast `upsertStudentUnavailability` (same file), which correctly loads `Student.findOne({ _id, store_id, isDeleted:false })` first.

**Fix:** load the student scoped to `req.user.store` (and verify each course belongs to the store) before mutating; reject if not found.

---

## ST-02 · MEDIUM — `getStoreStudentTestCycleTests` references an undefined `session`

**File:** `controllers/students.js` lines 638–644

```js
const default_period = await teachinPeriodHandler.getStoreDefaultPeriod(req.user?.store);
if (!default_period?._id) {
  await session.abortTransaction();   // ← `session` is never defined in this function
  return res.json({ success: false, message: "default_period_not_found" });
}
```

This handler has no transaction/`session`. When there is no default period, `session.abortTransaction()` throws `ReferenceError: session is not defined`, which is swallowed by the outer `catch` and returns a generic `{ success:false }` instead of `default_period_not_found`. (This is the student-facing home-page tests call.)

**Fix:** delete the stray `await session.abortTransaction();` line.

---

## ST-03 · MEDIUM — `getStoreStudentUsers` crashes when a student has no parents

**File:** `controllers/students.js` lines 285–292

```js
const user_ids = {
  student: student.user_id ? student.user_id : null,
  father: student.parents.father.user_id ?? null,  // throws if parents/father is null
  mother: student.parents.mother.user_id ?? null,
};
```

`getStudentById` populates `parents.father`/`parents.mother`, but if a student was created without parents (or a ref is null), `student.parents.father` is `null` and `.user_id` throws. Caught by the catch block, which returns the misleading message `create_student_fail` with HTTP 422.

**Fix:** `student.parents?.father?.user_id ?? null` (and same for mother), and return a meaningful error.

---

## ST-04 · MEDIUM — `upsertStudentDetails` assumes course-change arrays always exist

**File:** `controllers/students.js` lines 396, 416

```js
if (changes.newlyAssignedStudentCourses.length > 0) { ... }
if (changes.unAssignedStudentCourses.length > 0) { ... }
```

The controller reads `.length` directly. If the client sends a `changes` object that omits these keys (e.g. only a grade or class change), this throws `Cannot read properties of undefined (reading 'length')`. The downstream handler `updateStudentRelations` already defaults them to `[]`, so the crash is purely in the controller.

**Fix:** `(changes.newlyAssignedStudentCourses ?? []).length` (and same for unassigned).

---

## ST-05 · LOW — Dead `obj` template + opaque error on missing default period

**File:** `controllers/students.js` lines 24–84

`createStoreStudent` builds a large `obj` literal (firstName/parents/…) that is **never used** — dead code. Also `const periodId = default_period._id;` (line 84) throws if `getStoreDefaultPeriod` returns null; the catch returns a generic `create_student_fail`. Validate the period explicitly and remove the dead object.

---

## Note on soft-deleted students leaking (see course-class report CC-02) — ✅ FIXED (2026-07-14 + 2026-09-26)

`fetchStorePeriodStudents` (handler line 125) matched only `{ store_id }` — **no `isDeleted: false`** — and `fetchStoreStudents` (line 27) was `Student.find({ store_id })`. The brain doc's Soft-Delete section claims "All DB queries include `{ isDeleted: false }`" — that was **not** true here. Soft-deleted students (with PII and parent records) were sent in the bootstrap payload; only the frontend list view filtered them out.

**Fixed:** the period aggregation on 2026-07-14 (indexing pass), and `fetchStoreStudents` on
2026-09-26 — measured on the dev DB: `61 → 57` students returned, i.e. **4 soft-deleted students
(with their parent records) stopped shipping** through `GET /students/get-store-students`, which the
add-student page uses for duplicate/sibling suggestions.

⚠️ Consequence to keep in mind (already true since 2026-07-14, see the student-delete brain note):
the students **recycle bin is client-side**, so with the reads filtered it is empty after a reload
and restore is unreachable from the UI. The τάξεις bin hit the same wall on 2026-09-26 and got the
proper fix — a dedicated `GET /grades/get-store-deleted-grades` read (see `grading-system-bugs.md`
GR-07). Students/τμήματα/μαθήματα still need the same treatment.

---

## Summary

| ID | Severity | Description |
|---|---|---|
| ST-01 | MEDIUM | `upsertStudentDetails`/`updateStudentRelations` not store-scoped → cross-store write (IDOR) |
| ST-02 | MEDIUM | `getStoreStudentTestCycleTests` calls `session.abortTransaction()` with no `session` defined |
| ST-03 | MEDIUM | `getStoreStudentUsers` crashes when `parents.father/mother` is null |
| ST-04 | MEDIUM | `upsertStudentDetails` reads `.length` of possibly-undefined course arrays |
| ST-05 | LOW | Dead `obj` literal; opaque error when no default period |
| (note) | ~~MEDIUM~~ ✅ FIXED | Student fetch queries omitted `isDeleted:false` → soft-deleted students leaked (aggregation 2026-07-14, flat find 2026-09-26) |
