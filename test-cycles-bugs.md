# Test Cycles — Bug Report

> File: `controllers/tests.js` (+ `helpers/createTestCyclePdf.js`).

---

## TC-01 · HIGH — `deleteStoreTestCycle`: empty `catch{}`, mis-placed auth check, and no-response paths → request hangs

**File:** `controllers/tests.js` lines 269–330

Multiple defects in one handler:

1. **`catch (err) {}` is completely empty** (line 329) — any thrown error (e.g. `default_period._id` when no default period, a DB error) is swallowed **and no response is sent** → the request hangs until client timeout.
2. **Authorization is checked *after* the work is done.** The `if (!user_id || !store_id || !periodId || !test_cycle_id || !test_cycle) throw ...` guard sits at lines 326–328 — *below* the soft/hard delete that already ran (lines 286–324). It can never protect the operation, and when it does throw it lands in the empty catch (hang).
3. **Hard-delete success path is conditional with no else:** `if (deleted_test_cycle.deletedTestCycle) { commit; res.json(...) }` — if that field is falsy, nothing is committed and **no response is sent** (hang); if `deleted_test_cycle` is null, `.deletedTestCycle` throws into the empty catch (hang).

**Fix:** move the auth/param guard to the top; populate the catch with `abortTransaction` + an error response; ensure every branch sends a response.

---

## TC-02 · MEDIUM — Success messages for create vs update are swapped

**File:** `controllers/tests.js` lines 154–158

```js
message: existing_cycle ? "create_test_cycle_success" : "update_test_cycle_success"
```

When `existing_cycle` is truthy the cycle is **updated**, but the message says `create_…`; when creating new it says `update_…`. Backwards. Low functional impact, but misleading toasts / i18n.

**Fix:** swap the ternary branches.

---

## TC-03 · MEDIUM — `session.abortTransaction()` not awaited in catch blocks

**File:** `controllers/tests.js` lines 161 (`upsertStoreTestCycles`) and 258 (`upsertStoreTestCycleTests`)

```js
} catch (err) {
  session.abortTransaction();   // not awaited
  ...
  res.json({ success: false, ... });
} finally { session.endSession(); }
```

The abort is fire-and-forget; `session.endSession()` in `finally` may run before the abort settles, risking "Cannot use a session that has ended" / lingering transactions. (`deleteStoreTestCycleTest` and `exportTestCycleReport` correctly `await` it.)

**Fix:** `await session.abortTransaction();`.

---

## TC-04 · MEDIUM — Test-cycle report uploads to a different S3 bucket than documented

**File:** `controllers/tests.js` lines 470, 485

```js
Bucket: "logeion-test-cycle-reports",
```

The brain doc (`pdf-generation` + `test-cycles`) states test-cycle reports go to **`logeion-educational-material`** under the `test-cycle-reports/` prefix. The code uses a **separate, hardcoded** bucket `logeion-test-cycle-reports`. If that bucket doesn't exist / isn't configured, `PutObjectCommand` fails and the export errors out. Either the doc or the bucket name is wrong — reconcile, and move the bucket name to an env var (same hardcoding smell as educational-material BUG-007).

---

## TC-05 · LOW — Report rows are sorted in reverse alphabetical order

**File:** `controllers/tests.js` `buildTestCycleStructure` lines 515–580

Every `localeCompare` is called as `b.localeCompare(a)` (students, courses, grades), i.e. **descending**. The PDF lists students/courses/grades Z→A, almost certainly unintended.

**Fix:** swap to `a.localeCompare(b)` for ascending order.

---

## TC-06 · LOW — `upsertStoreTestCycles` can hang if `test_cycle` is falsy without throwing

**File:** `controllers/tests.js` lines 150–159

The success response is inside `if (test_cycle) { … }` with no `else`. Create/update normally throw on failure, but if either ever resolves falsy without throwing, no response is sent and the transaction neither commits nor aborts → hang. Add an `else` that aborts + responds.

---

## Summary

| ID | Severity | Description |
|---|---|---|
| TC-01 | HIGH | `deleteStoreTestCycle`: empty catch + auth-check-after-work + no-response branches → hangs/swallowed errors |
| TC-02 | MEDIUM | create/update success messages swapped |
| TC-03 | MEDIUM | `abortTransaction()` not awaited before `endSession()` |
| TC-04 | MEDIUM | Uploads to `logeion-test-cycle-reports` (hardcoded) vs documented `logeion-educational-material` |
| TC-05 | LOW | Report sorts students/courses/grades in reverse alphabetical order |
| TC-06 | LOW | `upsertStoreTestCycles` has a no-response path if `test_cycle` is falsy |
