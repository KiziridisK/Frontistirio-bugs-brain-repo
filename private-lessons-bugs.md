# Private Lessons — Bug Report

> Analysed files: `models/private-lessons.js`, `controllers/private-lessons.js`,
> `controllers/handlers/private-lessons.js`, `controllers/hourly_rates.js`,
> `controllers/handlers/hourly_rates.js`, `controllers/student_hourly_rates.js`,
> `controllers/handlers/student_hourly_rates.js`, `models/hourly_rates.js`,
> `models/student_hourly_rates.js`, `routes/private-lessons.js`,
> `routes/hourly_rates.js`, `routes/student_hourly_rates.js`

---

> ⚠️ **Staleness note (re-verified against current code, June 2026).** The codebase has
> moved on since this report. Confirmed **fixed**:
> - **BUG-01** — `models/private-lessons.js` now has a single `student_id` (ref `Student`)
>   plus a separate `teacher_id` field. The duplicate-key override is gone.
> - **BUG-02** — `deleteStudentPrivateLessons` *controller* is now implemented and sends a
>   response (lines 149–192). (Verify the matching handler is no longer an empty stub.)
> - **BUG-03** — `exportStudentPrivateLessons` now has an `else` branch that returns
>   `no_lessons_found_for_period` (line ~380), so it no longer hangs on empty lessons.
>
> Still present / re-confirmed: **BUG-08** (wrong bucket name in the log URL, line 357) and
> **BUG-09** (pointless transaction around the read-only export). A **new** related defect —
> the billing PDF printing the wrong dates — is filed in `pdf-generation-bugs.md` (PDF-01).
> The remaining items below were not re-verified line-by-line in this pass.

---

## BUG-01 — CRITICAL · Duplicate `student_id` field in model overwrites with Teacher ref

**File:** `models/private-lessons.js` lines 28–38

The schema defines `student_id` **twice**. In JavaScript object literals the second key wins, so the first definition (correctly referencing `Student`) is silently discarded and the field ends up referencing `Teacher` with `required: false, default: null`.

```js
// first definition — OVERRIDDEN, never used
student_id: {
  type: mongoose.Schema.Types.ObjectId,
  ref: "Student",
  required: true,
},

// second definition — this one wins
student_id: {
  type: mongoose.Schema.Types.ObjectId,
  ref: "Teacher",   // ← wrong ref
  required: false,
  default: null,
},
```

**Impact:** Every saved `PrivateLesson` document has `student_id` stored without a `required` constraint and with the wrong population target. Queries that filter by `student_id` still work (the value is stored), but any `populate("student_id")` call returns Teacher documents instead of Student documents. Validation (`required: true`) is also silently lost.

**Fix:** Remove the second `student_id` block. If a `teacher_id` field is needed, add it under a separate name.

---

## BUG-02 — HIGH · `deleteStudentPrivateLessons` controller never sends a response — request hangs

**File:** `controllers/private-lessons.js` lines 149–163

The controller body is an empty `try` block. No response is ever sent, so every DELETE request to `/private-lessons/delete-student-private-lesson` hangs until the client times out.

```js
exports.deleteStudentPrivateLessons = async (req, res) => {
  const session = await mongoose.startSession();
  session.startTransaction();
  try {
    // ← EMPTY — no logic, no res.json()
  } catch (err) { ... }
  finally { session.endSession(); }
};
```

The handler in `controllers/handlers/private-lessons.js` (lines 61–64) is equally empty:
```js
exports.deleteStudentPrivateLessons = async () => {
  try {
  } catch (err) {}
};
```

**Impact:** Delete functionality is completely non-functional.

---

## BUG-03 — HIGH · `exportStudentPrivateLessons` sends no response when the lessons array is empty — request hangs

**File:** `controllers/private-lessons.js` lines 212–351

All logic — including `res.json(...)` — is inside `if (lessons?.length > 0)`. When there are no lessons in the requested date range the `if` is skipped and the function reaches the `finally` block without ever calling `res.json()` or `res.status()`, leaving the HTTP connection open indefinitely.

```js
if (lessons?.length > 0) {
  // ... all processing ...
  res.json({ success: true, ... });
}
// NO ELSE — empty lessons → no response sent
```

**Fix:** Add an `else` branch that returns an appropriate response (e.g., `{ success: false, message: "no_lessons_found" }`).

---

## BUG-04 — HIGH · Edit private lesson endpoint does not exist in the backend routes

**File:** `routes/private-lessons.js`

The frontend service calls `POST /private-lessons/edit-store-class` (noted with a ⚠️ in the brain repo). This route is **never registered** in `routes/private-lessons.js`. Any attempt to edit a private lesson from the frontend results in a 404.

**Fix:** Add a `PUT`/`POST` route (e.g., `/edit-student-private-lesson`) and implement the corresponding controller and handler.

---

## BUG-05 — MEDIUM · `getStudentPrivateLessons` query has no `store_id` filter — cross-store data leak

**File:** `controllers/handlers/private-lessons.js` lines 5–22

The Mongoose query fetches lessons for a given student but does **not** filter by `store_id`. If the same student is enrolled in multiple stores, all stores' lessons are returned to any store user querying that student.

```js
const lessons = await PrivateLesson.find({
  student_id: student_id,
  period_id: periodId,
  isDeleted: false,
  start_time: { $gte: new Date(start), $lte: new Date(end) },
  // ← missing: store_id
});
```

This same handler is called by both `getStudentPrivateLessons` and `exportStudentPrivateLessons`, so the PDF export also leaks cross-store lessons.

**Fix:** Pass `store_id` into the handler and add it to the query filter.

---

## BUG-06 — MEDIUM · `editStoreHourlyRates` controller is a stub — no response sent

**File:** `controllers/hourly_rates.js` lines 98–120

The `editStoreHourlyRates` controller commits an empty transaction and never sends a response. The route `POST /edit-store-hourly-rates` is wired up and accessible but non-functional.

```js
exports.editStoreHourlyRates = async (req, res) => {
  const session = await mongoose.startSession();
  session.startTransaction();
  try {
    // ← EMPTY
    await session.commitTransaction();
    // ← no res.json()
  } catch (error) { ... }
};
```

---

## BUG-07 — MEDIUM · Query param name mismatch for `getStoreStudentHourlyRates`

**File:** `controllers/student_hourly_rates.js` line 29

The controller reads `req.query.student_id`:
```js
const student_id = req.query.student_id;
```

But the brain repo documents the frontend sending `?studentId=xxx` (camelCase). If the frontend actually sends `studentId`, the backend receives `undefined`, triggering the `invalid_params` error and returning a 400 on every call.

**Fix:** Align the query param name — either update the frontend to send `student_id` or update the backend to read `req.query.studentId`.

---

## BUG-08 — LOW · Wrong S3 bucket name used when building the log URL

**File:** `controllers/private-lessons.js` line 328

After uploading to `logeion-private-lesson-costs`, the log URL is constructed using the **educational-material** bucket:

```js
const s3Url = `https://logeion-educational-material.s3.${process.env.AWS_REGION}.amazonaws.com/${s3Key}`;
```

The variable is only used for `console.log` so it does not affect functionality, but it produces a misleading URL in logs and makes debugging harder.

**Fix:** Change the bucket name in the string to `logeion-private-lesson-costs`.

---

## BUG-09 — LOW · MongoDB transaction in `exportStudentPrivateLessons` is unnecessary

**File:** `controllers/private-lessons.js` lines 166–361

The export controller wraps its entire logic in a MongoDB transaction (`startSession → startTransaction → commitTransaction`), but the operation contains **no database writes** — it only reads lessons, reads rates, generates a PDF, and uploads to S3. Transactions add latency and hold server-side resources for no benefit here.

**Fix:** Remove the session/transaction and simply `await` the async calls directly.

---

## BUG-10 — LOW · No DELETE route for store-level hourly rates

**File:** `routes/hourly_rates.js`

The brain repo documents `DELETE /pricing-settings/delete-store-pricing-setting`, but no DELETE route exists in `routes/hourly_rates.js`. Store-level pricing settings can be created but never deleted via the API.

---

## Summary Table

| ID | Severity | Description |
|---|---|---|
| BUG-01 | CRITICAL | Duplicate `student_id` in model silently overrides with Teacher ref |
| BUG-02 | HIGH | `deleteStudentPrivateLessons` is an empty stub — request hangs |
| BUG-03 | HIGH | `exportStudentPrivateLessons` sends no response for empty lessons — request hangs |
| BUG-04 | HIGH | Edit lesson endpoint (`/edit-store-class`) not registered in routes — 404 always |
| BUG-05 | MEDIUM | Missing `store_id` filter in lesson query — cross-store data leak |
| BUG-06 | MEDIUM | `editStoreHourlyRates` is an empty stub — no response sent |
| BUG-07 | MEDIUM | Query param mismatch (`studentId` vs `student_id`) breaks student rate fetching |
| BUG-08 | LOW | Wrong S3 bucket name in log URL construction |
| BUG-09 | LOW | Unnecessary MongoDB transaction in export controller |
| BUG-10 | LOW | No DELETE route for store hourly rates |
