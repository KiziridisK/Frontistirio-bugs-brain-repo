# Grading System — Bug Report

> Files: `models/student-course-grade.js`, `controllers/student-course-grade.js`,
> `controllers/handlers/student-course-grade.js`.

---

## GR-01 · HIGH (doc/contract mismatch) — Model uses `score`/`comment`/`period_timeline`, not `grade`

**File:** `models/student-course-grade.js`

Actual schema fields:
```js
{ store_id, course_id, student_id,
  comment: Mixed, score: Mixed, visible: Boolean,
  period_timeline: Mixed (required), period_id, createdBy, updatedBy, isDeleted, deletedAt }
```

The brain doc (`grading-system`) documents the field as `grade: Mixed` and the upsert payload as `body: { student_id, course_id, grade, period_id }`. That is **wrong**. The real upsert controller (`createStoreStudentCourseGrades`) reads `{ student_id, course_id, period_key, score, comment }` and requires `period_key` (stored as `period_timeline`). A frontend built to the documented contract would send `grade` and omit `period_key` → `invalid_params` (missing `period_key`) and no score saved.

**Action:** correct the brain doc; confirm the Angular `CourseGradeService.upsertCourseGrade` actually sends `score`, `comment`, `period_key` (not `grade`).

---

## GR-02 · MEDIUM — `getStoreStudentCourseGrades` omits `isDeleted: false` — ✅ FIXED 2026-07-06

**File:** `controllers/handlers/student-course-grade.js`

```js
const query = { store_id, period_id, course_id, student_id: { $in: studentIds.map(...) } };
return StudentCourseGrades.find(query);   // no isDeleted filter
```

`updateStudentCourseGrade` filters `isDeleted: false`, so the model supports soft delete — but the read path returned soft-deleted grade records.

**FIXED** while building the grade-approval workflow: the query now sets `query.isDeleted = false`. This
became load-bearing because reject soft-deletes a pending grade — without the filter the rejected value
would still show in the gradebook. See the `grading-system` brain "Grade & Comment Approval Workflow".

---

## GR-03 · MEDIUM — Upsert is read-then-write with no unique index → duplicate grades under concurrency

**File:** `controllers/student-course-grade.js` lines 95–131

The "upsert" first calls `getStoreStudentCourseGrades(...[period_key]...)`; if none found it creates, else updates. There is **no unique index** on `(student_id, course_id, period_id, period_timeline)`. Two concurrent saves (e.g. double-click, or two graders) both read "none found" and both insert → duplicate grade records for the same cell. The grouped read (`_.groupBy(..., 'student_id')`) then returns an array with two entries and the UI shows whichever it picks.

**Fix:** add a unique compound index and use a real `findOneAndUpdate(..., { upsert: true })` keyed on that tuple.

**Still open + now wider (2026-07-06):** the new `submitStudentCourseGrade` (teacher approval path) reuses the
same read-then-write logic, so the duplicate-under-concurrency window now exists on two endpoints. A unique
compound index would fix both at once.

---

## GR-04 · LOW — `visible` flag is unused end-to-end

**File:** model `visible` + controllers

The schema has `visible: Boolean` (intended to control whether a student can see the grade), but there is **no student-facing grade endpoint** and the store-user read query never references `visible`. The flag is dead until a student grade view is added.

---

## GR-05 · LOW — Weak param validation in `getStoreStudentCourseGrades`

**File:** `controllers/student-course-grade.js` line 34

```js
if (!course_id || !periodId || student_ids?.length == 0) throw new Error("invalid_params");
```

If `student_ids` is `undefined`, `student_ids?.length == 0` is `undefined == 0` → `false`, so the guard passes and `undefined` flows into the handler, where `studentIds.map(...)` throws `Cannot read properties of undefined`. Also a single `?student_ids=x` (no `[]`) arrives as a string, and `"x".map` throws. Normalize to an array and validate non-empty.

---

## GR-06 · HIGH (broken access control) — teacher grade submit had no ownership check — ✅ FIXED 2026-07-06

**File:** `controllers/student-course-grade.js` — `submitStudentCourseGrade`

As originally built (grade-approval session), the endpoint was gated by `isTeacher` only. It read
`{ student_id, course_id, period_key, score, comment }` from the body and wrote the grade **without
verifying the teacher actually teaches that course/student** — it scoped only by `req.user.store`. So
any authenticated teacher could submit (and, with `require_grade_approval` OFF, directly apply) a grade
for **any course of any student in their store**, just by POSTing an arbitrary `course_id`/`student_id`.

**Fix (2026-07-06, teacher-view session):** added a server-side ownership gate — resolve the teacher via
`teacherHandler.getTeacherByUserId(user_id)`, assert `course_id ∈ getTeacherPeriodAssignmentIds(teacher,
periodId).courseIds` (throws `course_not_assigned_to_teacher`), and the same for the new teacher read
endpoint `getTeacherStudentCourseGrades` (shared helper `resolveTeacherForCourse(req, course_id)`).
The teacher **material upload** endpoint got an analogous grade/class/course scope gate. **Still open:**
per-**student** validation is not enforced on submit (only per-course) — a teacher could target a
`student_id` not actually in that course (low risk; the course check + admin approval are the guards).

---

## Summary

| ID | Severity | Description |
|---|---|---|
| GR-01 | HIGH | Brain doc says field `grade`; real model/controller use `score`/`comment`/`period_timeline`(`period_key`) |
| GR-02 | ~~MEDIUM~~ ✅ FIXED | Read query omitted `isDeleted:false` → now filtered (2026-07-06) |
| GR-03 | MEDIUM | Read-then-write upsert without unique index → duplicate grade records (now on upsert **and** submit) |
| GR-04 | LOW | `visible` flag unused (no student grade endpoint); approval uses a separate `status` field |
| GR-05 | LOW | `student_ids` validation passes when undefined / breaks on non-array |
| GR-06 | ~~HIGH~~ ✅ FIXED | Teacher grade submit lacked course-ownership check (IDOR) → now gated server-side (2026-07-06); per-student check still open |
