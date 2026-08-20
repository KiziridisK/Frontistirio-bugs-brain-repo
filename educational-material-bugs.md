# Educational Material — Bug Report

> Bugs identified during code review of the educational material feature.
> Files: `controllers/educational-material.js`, `controllers/handlers/educational-material.js`, `routes/educational-material.js`

---

## BUG-001 · Double `res.json()` on every successful download request

**Severity:** High  
**File:** `controllers/educational-material.js` — `getStoreEducationalMaterial`  
**Lines:** ~54–63

**Problem:**  
After the `if (material)` block sends `res.json({ success: true, downloadUrl, ... })`, execution falls through to a second `res.json({ success: false })` outside the block. This fires on every successful request, causing a "Cannot set headers after they are sent" error.

**Fix:**  
Add a `return` before `res.json(...)` inside the `if (material)` block, or restructure with `else`.

```js
// Before (broken)
if (material) {
  // ...
  res.json({ success: true, downloadUrl, filename, filetype });
}
res.json({ success: false }); // ← always runs after the if block

// After (fixed)
if (material) {
  // ...
  return res.json({ success: true, downloadUrl, filename, filetype });
}
return res.json({ success: false });
```

---

## BUG-002 · Permanent delete sends double response and aborts a committed transaction

**Severity:** High  
**File:** `controllers/educational-material.js` — `deleteStoreEducationalMaterial`  
**Lines:** ~368–404

**Problem:**  
When `permanently = true`, the S3 delete succeeds and then `commitTransaction()` + `res.json({ success: true })` are called inside the inner `try` block. Execution then continues to the outer `if (result && !permanently)` → false → `else` which calls `session.abortTransaction()` and sends a second `res.json({ success: false })`. The session has already been committed, so the abort throws or silently fails, and two responses are sent.

**Fix:**  
Add a `return` after the permanent-delete `res.json()` call so execution stops there, preventing the `else` branch from running.

```js
if (material && permanently) {
  // ...S3 delete...
  await session.commitTransaction();
  return res.json({ success: true, message: "delete_material_success" }); // ← add return
}

if (result && !permanently) {
  await session.commitTransaction();
  return res.json({ success: true, message: "delete_material_success" });
}

// only reach here on a genuine failure
await session.abortTransaction();
return res.json({ success: false, message: "delete_material_fail" });
```

---

## BUG-003 · `this.getPeriodEducationalMaterial` crashes at runtime in student material handler

**Severity:** High  
**File:** `controllers/handlers/educational-material.js` — `getStudentEducationalMaterial`  
**Line:** ~205

**Problem:**  
`this` does not refer to `exports` in a CommonJS module. Calling `this.getPeriodEducationalMaterial(...)` will throw `TypeError: this.getPeriodEducationalMaterial is not a function` at runtime whenever student materials are fetched.

**Fix:**  
Replace `this.` with `exports.`:

```js
// Before (broken)
const period_material = await this.getPeriodEducationalMaterial(store_id, period_id);

// After (fixed)
const period_material = await exports.getPeriodEducationalMaterial(store_id, period_id);
```

---

## BUG-004 · `async` callback inside `.filter()` — all socket users pass the check

**Severity:** High  
**File:** `controllers/handlers/educational-material.js` — `watchEducationalMaterial`  
**Lines:** ~336–341 (update change stream handler)

**Problem:**  
`Array.prototype.filter()` does not await async callbacks — it receives a `Promise` object which is always truthy. Every connected user passes the filter, so update events are broadcast to **all** users regardless of store.

```js
// Broken — filter cannot await async callbacks
const storeUsers = Object.keys(userSockets).filter(async (userId) => {
  return (
    isStoreUser(userId) &&
    (await getUserStoreId(userId, userSockets)) === store_id.toString() // ← never awaited
  );
});
```

**Fix:**  
Collect results with `Promise.all` + `map`, then filter on the resolved values:

```js
const userIds = Object.keys(userSockets);
const checks = await Promise.all(
  userIds.map(async (userId) => ({
    userId,
    match:
      isStoreUser(userId) &&
      (await getUserStoreId(userId, userSockets)) === store_id.toString(),
  }))
);
const storeUsers = checks.filter((c) => c.match).map((c) => c.userId);
```

Apply the same fix to the **insert** change handler if `getUserStoreId` is async there too.

---

## BUG-005 · Soft-deleted materials appear in store listings

**Severity:** Medium  
**File:** `controllers/handlers/educational-material.js` — `getStoreEducationalMaterials`  
**Lines:** ~159–163

**Problem:**  
`EducationalMaterial.find({ store_id })` has no `isDeleted: false` filter. Soft-deleted materials are included in the list returned to store admins.

**Fix:**

```js
exports.getStoreEducationalMaterials = async (store_id) => {
  return await EducationalMaterial.find({ store_id, isDeleted: false });
};
```

---

## BUG-006 · Soft-deleted materials can still generate download URLs

**Severity:** Medium  
**File:** `controllers/handlers/educational-material.js` — `getEducationalMaterial`  
**Lines:** ~155–157

**Problem:**  
`EducationalMaterial.findById(id)` does not check `isDeleted`. A permanently soft-deleted material can still be fetched by ID and a valid pre-signed S3 URL will be returned.

**Fix:**

```js
exports.getEducationalMaterial = async (id) => {
  return EducationalMaterial.findOne({ _id: id, isDeleted: false });
};
```

---

## BUG-007 · S3 bucket name is hardcoded instead of using env var

**Severity:** Medium  
**File:** `controllers/educational-material.js`  
**Lines:** ~135 (create) and ~278 (edit/replace file)

**Problem:**  
Both upload paths hardcode `Bucket: "logeion-educational-material"`. The brain doc specifies `process.env.AWS_BUCKET_NAME`. If the bucket is renamed or the app runs against a different environment, this will silently fail.

**Fix:**

```js
Bucket: process.env.AWS_BUCKET_NAME,
```

---

## BUG-008 · No file size limit on multer — DoS risk

**Severity:** Medium  
**File:** `routes/educational-material.js`  
**Lines:** ~10–20

**Problem:**  
The multer config has no `limits` option. Any authenticated store-user can upload arbitrarily large files, potentially exhausting disk space in the `uploads/` temp folder and incurring unbounded S3 costs.

**Fix:**

```js
const upload = multer({
  storage,
  limits: { fileSize: 50 * 1024 * 1024 }, // e.g. 50 MB cap
});
```

---

## BUG-009 · Missing routes: list-all and student-materials endpoints

**Severity:** Medium  
**File:** `routes/educational-material.js`

**Problem:**  
Two handlers exist in `controllers/handlers/educational-material.js` but are never wired up as routes:
- `getStoreEducationalMaterials` — no `GET /` route for store admin to list all materials for the active period
- `getStudentEducationalMaterial` — no route for students to fetch their accessible materials

Both are documented in the brain repo as required endpoints.

**Fix:**  
Add routes to `routes/educational-material.js`:

```js
router.get(
  "/",
  authMiddleware,
  isStoreUser,
  educationalMaterialController.listStoreEducationalMaterials,
);

router.get(
  "/student/:studentId",
  authMiddleware,
  isStoreOrStudentUser,
  educationalMaterialController.getStudentMaterials,
);
```

And implement the corresponding controller functions that call the handlers.

---

## BUG-010 · Parent role cannot access materials despite `visibleToParents` flag existing

**Severity:** Low  
**File:** `routes/educational-material.js`

**Problem:**  
The model stores `visibleToParents` per permission entry, but the `parent` role is not included in any route's middleware. Parents have no way to retrieve materials at all, making the flag unused end-to-end.

**Fix:**  
Either add a dedicated `GET /parent/:studentId` route with `isParent` middleware, or extend `isStoreOrStudentUser` to also allow `parent` where appropriate, and implement the filtering logic in the controller (similar to the student flow, but checking `visibleToParents: true`).

---

## BUG-011 · Student grade level was never actually enforced (FIXED 2026-06-25)

**Severity:** High (access control)  
**Status:** ✅ Fixed 2026-06-25 (same pass that added course-level permissions)  
**File:** `controllers/handlers/educational-material.js` — `getStudentEducationalMaterial`

**Problem:**  
The filter computed a grade match into a local `exists` flag but **never read it** in the
return decision. Whether a student saw a material depended only on the class check (or, when
no classes were set, it returned `true` for any student whose grade list was merely non-empty).
So a student could see a material targeted at a **different grade**, as long as the class
matched or no class was specified.

```js
// Before (broken) — `exists` is set but never used
let exists = false;
_.each(perm.grades, (pdg) => {
  if (pdg.toString() === period_grade.grade.toString()) exists = true;
});
if (perm.classes?.length > 0) {
  let class_exists = false;
  _.each(perm.classes, (pdc) => { if (pdc.toString() === period_classes.classes.toString()) class_exists = true; });
  if (class_exists) return true;   // ← grade `exists` ignored
} else {
  return true;                     // ← grade `exists` ignored
}
return false;
```

**Fix (applied):**  
Rewrote the filter to enforce all three levels as AND-constraints — **Grade AND Class AND
Course** — with safe optional chaining for missing student period entries:

```js
const grades = (perm.grades ?? []).map((g) => g.toString());
if (grades.length === 0) return false;
if (!student_grade_id || !grades.includes(student_grade_id)) return false;   // grade now enforced

const classes = (perm.classes ?? []).map((c) => c.toString());
if (classes.length > 0 && (!student_class_id || !classes.includes(student_class_id))) return false;

const courses = (perm.courses ?? []).map((c) => c.toString());          // NEW level
if (courses.length > 0 && !courses.some((cid) => student_course_ids.includes(cid))) return false;

return true;
```

**Related:** the `courses[]` level itself was added in the same pass (model + create/upsert
handlers + frontend cascade UI). See the educational-material brain repo change log (2026-06-25).

---

## Minor

**Typo — `sessiion` (double `i`) in delete handler parameter**  
`controllers/handlers/educational-material.js` line ~133  
Parameter is named `sessiion` instead of `session`. Doesn't break anything since it's consistent within the function, but worth cleaning up.

---

## BUG-012 · Any authenticated user could download any material (IDOR) — FIXED 2026-07-17

**Severity:** Critical
**File:** `controllers/educational-material.js` — `getStoreEducationalMaterial`
**Lines:** ~20–65 (pre-fix)

**Problem:**
The download route `GET /educational-material/get-educational-material/:materialId` looked the
material up by raw id and signed an S3 URL for it, with **no authorization check at all**. The
route guard (`authorizeRoles("store-user","student","parent","teacher")`) only proved the caller
was logged in *as some role* — never that the material was theirs to read.

Consequences: any authenticated student/parent/teacher could download **any file of any store**
by guessing or replaying a 24-hex ObjectId — including other stores' material, and material
whose `period_permissions` explicitly excluded them. Soft-deleted material was downloadable too.

**Fix:**
Added `canDownloadMaterial(user, material)`, called before signing the URL (403 otherwise):

- store isolation first — `material.store_id` must equal the caller's store, for every role;
- soft-deleted material is refused for everyone except store-users;
- **store-user** → anything in their own store;
- **teacher** → own uploads (`createdBy`), material whose `period_permissions.courses` intersect
  their period courses, or material attached to the syllabus of one of their courses;
- **student** → whatever `getStudentEducationalMaterial` grants (the Grade→Class→Course ladder),
  or material attached to the syllabus of a course they attend;
- **parent** → the union of the above over their children.

The syllabus arm exists because attaching material to a chapter is itself a grant — see
`handlers/course-syllabus.js` `getSyllabusMaterialIdsForCourses`. Found while building the
course-syllabus feature (the feature widens who reads material, so the missing check had to go
first).

---

## BUG-013 · Course-level permissions are invisible to students who attend via a class

**Severity:** Medium — OPEN
**File:** `controllers/handlers/educational-material.js` — `getStudentEducationalMaterial`
**Lines:** ~355–375 (Level 3 — Course)

**Problem:**
Level 3 tests the student's course ids against `student.period_courses` only. But
`period_courses` on the STUDENT doc is written solely for directly-assigned students
(`updateStoreStudentCourses`, called from `upsertCourseDetails` for not-in-class students).
Assigning a course to a **class** writes to the CLASS doc (`updateStoreClassCourses`) and never
copies it down to the class's students.

So a material scoped to a course is hidden from every student who attends that course through
their τμήμα — `student_course_ids` is empty for them, and `courses.length > 0` then fails the
check. It only appears to work because most such material also carries a `classes` filter, which
passes at Level 2.

**Fix:** resolve the student's courses as the union of both paths, as
`handlers/course-syllabus.js` `getStudentPeriodCourseIds(student, period_id)` already does:

```js
const student_course_ids = await syllabusHandler.getStudentPeriodCourseIds(student, period_id);
```

**Related:** the same root cause hit bootstrap — see BS-05 in `bootstrap-bugs.md`, fixed
2026-07-17. This one is left OPEN: it changes who sees existing material, so it wants a
deliberate pass rather than a drive-by.
