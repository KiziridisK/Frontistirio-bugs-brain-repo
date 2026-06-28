# Real-Time Sync — Bug Report

> Deep-dive of the MongoDB Change Stream → Socket.IO → NgRx pipeline.
> The store-scoping of real-time events is **broken across the board**. The same two
> mistakes are copy-pasted into every watcher.
> Files: every `controllers/handlers/*.js` `watchXxx`, `helpers/isStoreUser.js`,
> `helpers/getUserStoreId.js`, `helpers/emitToSpecificUser.js`, `index.js`.

---

## RT-01 · CRITICAL — `Array.prototype.filter(async …)` in UPDATE handlers leaks updates to ALL stores

**Severity:** Critical (cross-tenant data leak)
**Files / lines (the `update` branch of each watcher):**

| File | Line |
|---|---|
| `controllers/handlers/course.js` | 867 |
| `controllers/handlers/class.js` | 490 |
| `controllers/handlers/classroom.js` | 170 |
| `controllers/handlers/grades.js` | 148 |
| `controllers/handlers/store-settings.js` | 126 |
| `controllers/handlers/students.js` | 1230 |
| `controllers/handlers/teacher.js` | 962 |
| `controllers/handlers/teaching_periods.js` | 282 |
| `controllers/handlers/tests.js` | 487 |
| `controllers/handlers/educational-material.js` | ~336 (already filed as BUG-004) |

```js
const storeUsers = Object.keys(userSockets).filter(async (userId) => {
  return isStoreUser(userId) &&
         (await getUserStoreId(userId, userSockets)) === store_id.toString();
});
```

`filter` ignores the returned `Promise` and only sees that it is **truthy**, so **every** connected user passes. Result: when any document of these collections is *updated*, the `xxxUpdated` event is emitted to **every connected user in every store** — students see other stores' grade/student/teacher updates, etc. This is the most serious data-isolation defect in the system.

**Fix:** resolve first, then filter on plain values:
```js
const ids = Object.keys(userSockets);
const checks = await Promise.all(ids.map(async (userId) => ({
  userId,
  ok: (await isStoreUser(userId)) && getUserStoreId(userId, userSockets) === store_id.toString(),
})));
const storeUsers = checks.filter(c => c.ok).map(c => c.userId);
```

---

## RT-02 · HIGH — `isStoreUser` is async but used in a synchronous boolean → role gate is a no-op

**File:** `helpers/isStoreUser.js`

```js
async function isStoreUser(userId) {
  const user = await userHandler.findUserById(userId); // DB lookup!
  return user?.role === 'store-user';
}
```

The **insert** and **delete** branches of every watcher use a *synchronous* `filter` but still call `isStoreUser(userId)` (no `await`):
```js
return isStoreUser(userId) && getUserStoreId(...) === store_id;
//     ^^^^^^^^^^^^^^^^^^^ returns a Promise → always truthy
```
So the role check is silently bypassed: insert/delete events are delivered to **all roles** connected for the target store (students/parents receive store-admin entity events for their store), not just `store-user`s. The store filter still works for insert/delete (the right-hand side is real), so this is "only" intra-store role leakage — but combined with RT-01 the update path leaks across stores entirely.

**Note — brain-doc inaccuracy:** `real-time-sync` README documents `isStoreUser` as a synchronous in-memory lookup (`userSockets[userId]?.role === 'store-user'`). The real implementation does an **async DB query per connected user per event** — both a correctness bug (above) and a performance problem (N queries per change, results discarded).

**Fix:** either make a synchronous in-memory variant that reads `userSockets[userId].role` (matches the doc, removes the DB hit), or `await` it inside a `Promise.all` map as in RT-01.

---

## RT-03 · MEDIUM — `emitToSpecificUser` crashes if the user disconnected

**File:** `helpers/emitToSpecificUser.js`

```js
const socketId = userSockets[userId].socketId;  // throws if userSockets[userId] is undefined
```

No optional chaining. If a user disconnects between building `storeUsers` and the `forEach` emit (or `userSockets` was mutated), this throws `Cannot read properties of undefined (reading 'socketId')` inside the change-stream `change` callback → unhandled rejection. The brain-doc version shows `userSockets[userId]?.socketId` — the real code dropped the `?.`.

**Fix:** `const socketId = userSockets[userId]?.socketId; if (socketId) io.to(socketId).emit(...)`.

---

## RT-04 · MEDIUM — UPDATE events emit dotted-path deltas that the NgRx reducers cannot merge

**File:** all watcher `update` branches + frontend `updateXxxFields` reducers

Watchers emit the Mongo delta verbatim:
```js
emitToSpecificUser(io, userId, "courseUpdated",
  { id: updatedCourseId, changes: change.updateDescription.updatedFields }, userSockets);
```

For updates to **nested period arrays** (the core bidirectional pattern — assigning a class/teacher/student to a course, etc.), `updatedFields` keys are **dotted/positional paths**, e.g. `{ "period_classes.0.classes": [...] }` or `{ "period_courses.1.courses": [...] }`. The reducers merge with a shallow spread:
```js
state.courses.map(c => c._id === id ? { ...c, ...changes } : c)
```
This adds a literal property named `"period_classes.0.classes"` to the object instead of mutating the nested array. So **real-time UI does not reflect course/class/teacher/student relationship changes** — the period arrays look unchanged until the next full REST refetch (navigation/bootstrap). Scalar field updates (e.g. `name`, `default`) merge fine; only nested-array updates are affected.

**Fix:** for these collections, emit the freshly-looked-up `fullDocument` (the watchers already request `fullDocument: 'updateLookup'`) and have the reducer replace the whole record, or translate dotted paths into a proper nested merge on the client. Worth verifying against the actual reducers in `state/*/`.

---

## RT-05 · LOW — Known gaps (confirmed in code)

- **In-memory `userSockets`** — a server restart clears all registrations (`index.js`); no Redis adapter, so horizontal scaling is unsupported (as documented).
- **Reconnect re-register** sends only `userId` (frontend `web-socket.service.ts`) rather than the full `{ userId, role, storeId }`, so the post-reconnect `userSockets` entry can be incomplete.
- **No missed-event queue** — events during a disconnect are lost until the next REST fetch.

---

## Summary

| ID | Severity | Description |
|---|---|---|
| RT-01 | CRITICAL | `filter(async …)` in 9+ UPDATE watchers → updates broadcast to every store |
| RT-02 | HIGH | `isStoreUser` is async/DB-backed but used as a sync boolean → role gate is a no-op (+ doc inaccuracy, perf) |
| RT-03 | MEDIUM | `emitToSpecificUser` lacks `?.` → can throw if user disconnected mid-emit |
| RT-04 | MEDIUM | Dotted-path deltas can't be merged by shallow-spread reducers → nested relationship changes don't sync live |
| RT-05 | LOW | In-memory registry, partial reconnect re-register, no missed-event queue |
