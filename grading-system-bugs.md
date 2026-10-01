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

## GR-07 · MEDIUM — τάξεις (`Grade`) shipped soft-deleted in every bootstrap — ✅ FIXED 2026-09-26

**File:** `controllers/handlers/grades.js` `fetchStoreGrades`, called 4× in `controllers/bootstrap.js`
(store-user, student, parent and teacher payloads) plus `GET /grades/get-store-grades`.

`Grade.find({ store_id })` carried **no** `isDeleted` filter, so soft-deleted τάξεις went out to every
client on every login — the last unfixed read of the systemic soft-delete defect that was still on a
hot path.

The catch: the τάξεις **recycle bin is client-side** (`grades.component.ts` filters
`!!showRecycleBin !== !!grade.isDeleted` over the ngrx store), and its only action is *permanent*
delete — there is no restore for τάξεις. So simply filtering the read would have emptied the bin and
left soft-deleted rows unreachable and unpurgeable forever (exactly what already happened to students
on 2026-07-14).

**Fix — split the two reads instead of filtering one payload:**
- `fetchStoreGrades(storeId, { includeDeleted = false })` — excludes soft-deleted by default, so
  bootstrap and `get-store-grades` are clean.
- New `fetchStoreDeletedGrades(storeId)` + controller `getStoreDeletedGrades` + **`// NEW ROUTE`**
  `GET /grades/get-store-deleted-grades` (`authMiddleware + isStoreUser`) — the bin's own read. No
  API Gateway work needed (root `/{proxy+}` catch-all).
- Frontend: `GradesService.getStoreDeletedGrades()`; `grades.component.ts` keeps `activeGrades` (store)
  and `deletedGrades` (fetched on init for the bin badge count, re-fetched on each bin open) and
  composes both into the list the filters run on; `grade-item` got an `@Output() purged` so a
  permanent delete drops the row (no realtime event can — the row was never in the store).

**Verified** on the dev DB: a temporary soft-deleted τάξη was invisible to `fetchStoreGrades` /
`GET /grades/get-store-grades` (9 rows, none deleted) and visible to
`GET /grades/get-store-deleted-grades` (`{success:true,grades:[…]}`), then removed. Frontend: `tsc`
clean + AOT `ng build` clean. No UI click-through yet.

---

## GR-08 · MEDIUM (new, 2026-09-26) — `deleteStoreGrade` is not store-scoped (IDOR) and `/grades/get-all` is unauthenticated

Two things spotted while fixing GR-07, **both still open**:

1. `controllers/grades.js deleteStoreGrade` → `gradeHandler.deleteStoreGrade(gradeId, permanently)`
   takes the id straight from the body and never checks `grade.store_id === req.user.store`. Any
   store-user can soft- **or hard**-delete another store's τάξη by id. (Same shape as ST-01; contrast
   the student delete, which does guard the store.) It also has no referential guard, so a τάξη can be
   deleted while students/τμήματα still reference it — their τάξη name then resolves to nothing in the
   UI, since every page filters `!g.isDeleted` client-side.
2. `routes/grades.js` last line: `router.get("/get-all", gradeController.getAllGrades)` — **no
   `authMiddleware`** — and `fetchAllGrades()` is `Grade.find()` across *every* store. Same family as
   SEC-01: an anonymous cross-tenant read, reachable through the gateway catch-all. `classes.js`,
   `course.js` and others have the same `/get-all` pattern — worth one sweep over all of them.

---

## Summary

| ID | Severity | Description |
|---|---|---|
| GR-01 | HIGH | Brain doc says field `grade`; real model/controller use `score`/`comment`/`period_timeline`(`period_key`) |
| GR-02 | ~~MEDIUM~~ ✅ FIXED | Read query omitted `isDeleted:false` → now filtered (2026-07-06) |
| GR-03 | MEDIUM | Read-then-write upsert without unique index → duplicate grade records (now on upsert **and** submit) |
| GR-04 | LOW | `visible` flag unused (no student grade endpoint); approval uses a separate `status` field |
| GR-05 | LOW | `student_ids` validation passes when undefined / breaks on non-array |
| GR-06 | ~~HIGH~~ ✅ FIXED | Teacher grade submit lacked course-ownership check (IDOR) → now gated server-side (2026-07-06); per-student check **still open** (re-verified 2026-09-26: `student-course-grade.js` checks `courseIds` only) |
| GR-07 | ~~MEDIUM~~ ✅ FIXED | τάξεις shipped soft-deleted in every bootstrap → read split + dedicated recycle-bin endpoint (2026-09-26) |
| GR-08 | MEDIUM | **Open (new):** `deleteStoreGrade` not store-scoped (cross-store delete) + unauthenticated `GET /grades/get-all` returns every store's τάξεις |
