# Authentication & Users — Bug Report

> Deep-dive of the auth/users layer found several **critical** security issues.
> Files: `controllers/users.js`, `controllers/handlers/users.js`, `routes/users.js`,
> `middleware/auth.js`, `middleware/roles.js`, `index.js` (socket layer).
> JWT payload (built in `login`): `{ id, role, store: <storeId|null>, group: <groupId|null>, student }`.
> Note: `req.user.store` is the **store _id string** (not an object), so `req.user.store._id ?? req.user.store` always resolves to the id.

---

## SEC-01 · CRITICAL — Open superadmin registration (no auth)

**File:** `routes/users.js` line 9

```js
router.post("/register-superadmin", usersController.createSuperAdmin);
```

There is **no `authMiddleware` and no role guard**. Any anonymous client can `POST /users/register-superadmin` with `{ email, password, username }` and create a fully privileged superadmin account, then log in and control every group/store in the system.

**Fix:** Remove the route entirely after the first admin is seeded, or guard it (e.g. `authMiddleware, isSuperAdmin`, or a one-time bootstrap token / env check). At minimum, disable it in production.

---

## SEC-02 · CRITICAL — `change-password` is unauthenticated-by-design (IDOR) and broken

**File:** `routes/users.js` line 48 + `controllers/users.js` `changePassWord` (~line 380)

```js
router.get("/change-password", authMiddleware, usersController.changePassWord);
```

Two problems:

1. **No ownership/authorization check.** The controller takes `userId` straight from the request body and updates *that* user's password. Any authenticated user (a student, a parent) can change **any** other user's password — including a superadmin — by supplying their `_id`. Full account takeover.
2. **It is a `GET` route** performing a state change while reading `req.body` — wrong verb, and bodies on GET are unreliable across clients/proxies.
3. **It currently throws at runtime anyway:** `changePassWord` calls `User.findByIdAndUpdate(...)` but the `User` model is **never imported** in `controllers/users.js` (only `usersHandler` is). → `ReferenceError: User is not defined` → 500 on every call. So the feature is both insecure-by-design and non-functional.

**Fix:**
- Change to `POST` (or `PUT`).
- Derive the target id from `req.user.id` for self-service, or require `isSuperAdmin`/store-admin authorization to change another user's password.
- Import the model: `const User = require("../models/users");` (or call a handler that already imports it).

---

## SEC-03 · HIGH — `createStoreAdmin` passes mis-aligned args to `registerUser` → store-user created with empty `store`

**File:** `controllers/users.js` `createStoreAdmin` lines 80–89

`registerUser` signature (`handlers/users.js`):
```js
registerUser(email, passwordHash, username, role, periodId, user_id="", store="", student_id="", session=null)
```

The call **omits the `periodId` argument**, shifting everything after `role` by one:
```js
registerUser(
  email, passwordHash, username, role,
  user_id,   // → bound to periodId
  store,     // → bound to user_id (createdBy)
  "",        // → bound to store     ← STORE ENDS UP EMPTY
  session    // → bound to student_id
  // session arg ends up undefined
);
```

Result for every store-admin created through `POST /users/register-store-admin`:
- `store: ""` → the user has **no store binding**. At login the JWT gets `store: null`; bootstrap then can't resolve a default period and returns a 400/empty store. The account is effectively unusable.
- `createdBy` = the store id (wrong).
- `period_subscriptions` = `[{ period: <creating admin's user id>, active: true }]` (garbage).
- `session` is `undefined` → the user save runs **outside** the surrounding transaction, so a later abort won't roll it back.

`createStoreStudentUser` (same file) calls `registerUser` with the full, correctly-aligned 9 args — so the bug is isolated to `createStoreAdmin`.

**Fix:** Pass `periodId` (or `null`) explicitly in the 5th position and `session` in the 9th:
```js
registerUser(email, passwordHash, username, role, /*periodId*/ null, user_id, store, "", session);
```

---

## SEC-04 · HIGH — Socket.IO connections are unauthenticated; client self-declares store & role

**File:** `index.js` lines 75–96

```js
socket.on("register", ({ userId, role, storeId }) => {
  if (!userId) return;
  userSockets[userId] = { socketId: socket.id, role, storeId };
});
```

The websocket handshake performs **no JWT verification**. The client sends its own `userId`, `role`, and `storeId`, and the server trusts them verbatim. Combined with the change-stream watchers (which target `userSockets[*].storeId`), a malicious client can `register` with **any** `storeId` and receive that store's real-time `*.Added/Updated/Deleted` events — a cross-tenant data feed. See also `real-time-sync-bugs.md` (RT-01/RT-02), which makes the leak even broader.

**Fix:** Authenticate the socket handshake (`io.use((socket, next) => verify JWT)`), and derive `userId/role/storeId` from the verified token, never from the client payload.

---

## SEC-05 · MEDIUM — Secrets and plaintext credentials written to logs

**Files:** `middleware/auth.js` (lines ~9, 11, 13), `controllers/users.js` `login` (lines 302–303, 356, 361)

`auth.js` logs the raw bearer token and `process.env.JWT_SECRET` on every request. `login` logs the **plaintext password**, the JWT secret, and the issued token. Anyone with log access can mint tokens for any user and harvest credentials.

**Fix:** Remove all `console.log` of tokens, secrets, and passwords (the whole file base is heavy with debug logging — see also the per-feature reports).

---

## SEC-06 · LOW — `activateUser` / `deActivateUser` always report success (`if (res)`)

**File:** `controllers/users.js` lines ~220 and ~273

```js
if (res) {                  // res is the Express response object → ALWAYS truthy
  await session.commitTransaction();
  res.json({ success: true, user: updated_user, ... });
} else { /* dead code */ }
```

The intended guard was almost certainly `if (updated_user)`. As written, the commit + success response fire even when the handler returned nothing, and the `else` branch is unreachable.

**Fix:** `if (updated_user) { ... } else { abort + 422 }`.

---

## Summary

| ID | Severity | Description |
|---|---|---|
| SEC-01 | CRITICAL | `/users/register-superadmin` has no auth — anyone can create a superadmin |
| SEC-02 | CRITICAL | `change-password` (GET) has no authz → any user resets any password; also throws (`User` not imported) |
| SEC-03 | HIGH | `createStoreAdmin` mis-aligned `registerUser` args → store-user saved with `store:""`, outside transaction |
| SEC-04 | HIGH | Socket handshake unauthenticated; client self-declares `storeId/role` |
| SEC-05 | MEDIUM | JWT secret + plaintext password + tokens logged to console |
| SEC-06 | LOW | `activate/deActivateUser` use `if (res)` → always-success, dead else branch |
