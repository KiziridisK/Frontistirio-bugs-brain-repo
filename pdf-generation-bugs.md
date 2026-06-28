# PDF Generation — Bug Report

> Files: `helpers/createStudentLessonsPdf.js`, `helpers/createTestCyclePdf.js`,
> and the controllers that call them (`controllers/private-lessons.js`,
> `controllers/tests.js`).

---

## PDF-01 · HIGH — Private-lesson billing PDF prints the wrong dates (`date_from`/`date_to` vs `start_time`/`end_time`)

**File:** `helpers/createStudentLessonsPdf.js` lines 46–47 (`drawTableRow`)

```js
doc.text(formatDateTime(lesson.date_from), COL.FROM, y, { width: 120 });
doc.text(formatDateTime(lesson.date_to),   COL.TO,   y, { width: 120 });
```

The `PrivateLesson` model stores **`start_time`** and **`end_time`** (Date) — there is no `date_from`/`date_to` on a lesson. `exportStudentPrivateLessons` passes the lessons straight from `getStudentPrivateLessons` into the PDF helper without remapping, so `lesson.date_from`/`lesson.date_to` are `undefined`.

`formatDateTime(undefined)` → `dayjs(undefined)` → **the current date/time**. Every row's "Από"/"Έως" columns therefore show *now* instead of the lesson's real start/end. Hours and cost are correct (they use `lesson.duration`), but the dates on the customer's billing report are wrong.

**Fix:** read `lesson.start_time` / `lesson.end_time` (or map them to `date_from`/`date_to` before calling the helper).

---

## PDF-02 · MEDIUM — Hardcoded bucket names; test-cycle bucket disagrees with the brain doc

**Files:** `controllers/private-lessons.js` line 342 (`logeion-private-lesson-costs`), `controllers/tests.js` lines 470/485 (`logeion-test-cycle-reports`)

Bucket names are string literals rather than env vars, and the test-cycle bucket (`logeion-test-cycle-reports`) does not match the documented bucket (`logeion-educational-material` + `test-cycle-reports/` prefix). See `test-cycles-bugs.md` TC-04. Move all bucket names to `process.env` and reconcile the doc.

---

## PDF-03 · LOW — Misleading S3 URL log uses the wrong bucket

**File:** `controllers/private-lessons.js` line 357

```js
const s3Url = `https://logeion-educational-material.s3.${process.env.AWS_REGION}.amazonaws.com/${s3Key}`;
```

The object was uploaded to `logeion-private-lesson-costs`, but the logged URL references the educational-material bucket. Log-only (the returned link is a proper presigned URL), but misleads debugging. (Matches the older `private-lessons-bugs.md` BUG-08.)

---

## PDF-04 · LOW — Unnecessary MongoDB transaction around read-only export

**Files:** `controllers/private-lessons.js` `exportStudentPrivateLessons`, `controllers/tests.js` `exportTestCycleReport`

Both export controllers open `startSession()/startTransaction()` but perform **no DB writes** (read + PDF + S3). The transaction only adds latency and holds resources. (The brain doc calls this a "defensive pattern"; it provides no benefit here.) Drop the session and `await` the reads directly.

---

## PDF-05 · LOW — Column offsets differ from the brain doc

**File:** `helpers/createStudentLessonsPdf.js` line 89

Actual: `{ COURSE:50, FROM:170, TO:300, HOURS:420, COST:480 }`. Brain doc lists `{ ... TO:290, HOURS:410, COST:460 }`. Doc inaccuracy only — update the doc.

---

## Operational notes (confirmed, by design)

- Presigned URLs expire in **60s**; the frontend must `window.open` immediately.
- No S3 lifecycle/cleanup — generated PDFs accumulate indefinitely.

---

## Summary

| ID | Severity | Description |
|---|---|---|
| PDF-01 | HIGH | Lessons PDF reads `date_from`/`date_to` (don't exist) → all row dates show "now" |
| PDF-02 | MEDIUM | Hardcoded buckets; test-cycle bucket ≠ documented bucket |
| PDF-03 | LOW | Logged S3 URL uses wrong bucket name |
| PDF-04 | LOW | Read-only exports wrapped in a pointless transaction |
| PDF-05 | LOW | PDF column offsets differ from the brain doc |
