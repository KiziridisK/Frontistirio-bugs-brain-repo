# Bootstrap — Bug Report

> Deep-dive of the single-call bootstrap endpoint.
> File: `controllers/bootstrap.js` (`getSingle`), with helper `teaching_periods.getStoreDefaultPeriod`.

---

## BS-01 · HIGH — Student bootstrap crashes when the student has no enrollment for the default period

**File:** `controllers/bootstrap.js` lines 141–155 (student branch)

```js
const period_courses = _.find(student.period_courses, pc => pc.period.toString() === default_period._id.toString());
const period_classes = _.find(student.period_class,   pc => pc.period.toString() === default_period._id.toString());
// ...
promises.push(classesHandler.getClassesByIds([period_classes.classes])); // ← period_classes can be undefined
promises.push(coursesHandler.getCoursesById(period_courses.courses, storeId)); // ← period_courses can be undefined
```

If a student exists but has **no `period_class` / `period_courses` entry for the current default period** (very common: account created, not yet enrolled, or period switched), `_.find` returns `undefined` and the next line throws `Cannot read properties of undefined (reading 'classes' / 'courses')`. The whole bootstrap fails → student gets a 400 and an empty app on login.

**Fix:** guard before dereferencing:
```js
const classId = period_classes?.classes ? [period_classes.classes] : [];
const courseIds = period_courses?.courses ?? [];
```

---

## BS-02 · MEDIUM — Student bootstrap returns `students: [null]` when no Student doc matches the user

**File:** `controllers/bootstrap.js` lines 136–171

`getStudentByUserId(userId, default_period._id)` can return `null` (e.g. student soft-deleted, or wrong period). The `if (student)` guard skips the promise pushes, but the response is still built with `const students = [student];` → `students: [null]`, and `courses/classes/...` come from an **empty** `promiseRes` so they are `undefined`. The frontend dispatches a `[null]` student into NgRx.

**Fix:** if `!student`, return `{ success: false, message: "student_not_found" }` (or empty arrays) instead of `[null]`.

---

## BS-03 · HIGH — `parent` and `teacher` roles never send a response → request hangs

**File:** `controllers/bootstrap.js` lines 203–207

```js
} else if (userRole === "parent") {
  console.log("fetching bootstrap data of parent");   // no res.*()
} else if (userRole === "teacher") {
  console.log("fetching bootstrap data of teacher");  // no res.*()
}
```

These branches only log. No `res.json` / `res.status` is ever called and there is no `else`/timeout, so `GET /bootstrap/getSingle` **hangs until the client times out** for any parent or teacher account. (Both roles can already authenticate.)

**Fix:** return an explicit response — even `res.status(501).json({ success:false, message:"role_bootstrap_not_implemented" })` — so the client fails fast.

---

## BS-04 · MEDIUM — No default period ⇒ opaque crash for student, silent-empty for store-user

**File:** `controllers/bootstrap.js` lines 34–41, 132–139

For non-superadmins `default_period = await getStoreDefaultPeriod(req.user?.store)`.

- **student branch** immediately does `getStudentByUserId(userId, default_period._id)` (line 136) → if `default_period` is `null`, `default_period._id` throws → caught → generic 400.
- **store-user branch** passes `periodId = undefined` into all 15 queries → period-scoped aggregations match nothing → app loads "empty" with no error surfaced.

Combine with `store-management-bugs.md` SM-01/SM-02 (a store can end up with zero or several `default:true` periods).

**Fix:** detect `!default_period` early and return a clear `no_default_period` response for all non-superadmin roles.

---

## Summary

| ID | Severity | Description |
|---|---|---|
| BS-01 | HIGH | Student bootstrap dereferences `period_classes`/`period_courses` that may be `undefined` → crash |
| BS-02 | MEDIUM | Returns `students:[null]` + undefined arrays when no Student doc matches |
| BS-03 | HIGH | `parent`/`teacher` roles send no response → request hangs |
| BS-04 | MEDIUM | Missing default period → student crash / store-user silent-empty, no clear error |
