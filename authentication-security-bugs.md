# Authentication & Users — Bug Report

> Deep-dive of the auth/users layer found several **critical** security issues.
> Files: `controllers/users.js`, `controllers/handlers/users.js`, `routes/users.js`,
> `middleware/auth.js`, `middleware/roles.js`, `index.js` (socket layer).
> JWT payload (built in `login`): `{ id, role, store: <storeId|null>, group: <groupId|null>, student }`.
> Note: `req.user.store` is the **store _id string** (not an object), so `req.user.store._id ?? req.user.store` always resolves to the id.

---

## SEC-01 · ~~CRITICAL~~ — Open superadmin registration (no auth) — ✅ FIXED 2026-09-26

**File:** `routes/users.js` line 9

```js
router.post("/register-superadmin", usersController.createSuperAdmin);
```

There was **no `authMiddleware` and no role guard**. Any anonymous client could `POST /users/register-superadmin` with `{ email, password, username }` and create a fully privileged superadmin account, then log in and control every group/store in the system.

**It got worse before it got fixed.** The June pass assumed the blast radius was limited because the
path was not wired in API Gateway (only a direct call to EC2:3000 reached it, which the world-open
SG allows). Since the **root `/{proxy+}` catch-all added 2026-09-21** (300-resource quota, see
`api-gateway-wiring-bugs.md` / the aws-infrastructure brain repo) *every* path is forwarded to
Express — so this was reachable from the public gateway URL too. Lesson: a "not wired in the gateway"
mitigation evaporates the moment a catch-all is added.

**Fixed 2026-09-26:**
- The route is now `authMiddleware + isSuperAdmin` (same pair as `register-store-admin`), so only an
  existing superadmin can mint another.
- First-superadmin bootstrap moved out of the HTTP surface into **`scripts/create-superadmin.js`**
  (requires shell access + the Mongo URI; refuses to run when a superadmin already exists unless
  `--force`, refuses an email already in use, never logs the password).
- `createSuperAdmin` hardened: validates `email`/`password` before hashing (a missing password used
  to make `bcrypt.hash(undefined)` throw → 500 instead of 400), aborts the transaction on the
  early-return paths, stops `console.log`-ing the password hash and the created user, and records
  `createdBy: req.user.id`.
- Dead frontend caller removed: `home.page.ts register()` (never bound in the template) posted a
  hardcoded superadmin payload.

**Verified** against the local API on the dev DB: anonymous → `401 NO_TOKEN`, forged/malformed token
→ `403 INVALID_TOKEN`, valid **store-user** token → `403 Forbidden: insufficient role`, and no user
rows were created by any of the three attempts (superadmin count in dev unchanged at 1).

---

## SEC-02 · ~~CRITICAL~~ — `change-password` is unauthenticated-by-design (IDOR) and broken — ✅ FIXED (verified 2026-09-26)

> Current code: `router.post("/change-password", authMiddleware, usersController.changePassWord)`
> and the controller takes the target from `req.user.id` ("Identify the user from the authenticated
> token, never from the body"), uses `usersHandler.updatePassword` (no missing-import throw), and
> revokes every refresh token for that user. All three problems below are gone; the description is
> kept for the record.


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

## SEC-07 · CRITICAL — `passwordHash` (bcrypt) leaked to clients via the bootstrap & user reads  ✅ FIXED 2026-07-02

**Files:** `controllers/handlers/students.js`, `controllers/handlers/users.js`, `controllers/bootstrap.js` (consumer), `controllers/students.js` (`getStoreStudentUsers`).

The bootstrap payload and several user-read endpoints shipped the full `User` document — **including the bcrypt `passwordHash`** — to the browser, where it lands in the NgRx store. It was observed being echoed back to the server: an announcements `POST /announcements/send` body contained a recipient whose `userId` was the whole `User` object with `"passwordHash": "$2b$10$…"`.

Leak vectors (all returned to a client):

| Source | Ships to |
|---|---|
| `fetchStorePeriodStudents` → `Student.populate(…, { path: "user_id" })` (no projection) | bootstrap `students[*].user_id` (store-user login) |
| `watchStudents` → `.populate("user_id")` | Socket.IO `studentAdded` event |
| `findUserById` | bootstrap `user` (also used by `isStoreUser`/`isSuperAdmin` — role/store only) |
| `findUserById2` | `getStoreStudentUsers` response (`users[*].user`) |
| `getStoreUsers` (`User.find({ store })`) | `GET /users/getStoreUsers` |
| `activateDeActivateUser` (`User.findOne`) | `activate`/`deActivateUser` responses |
| `registerUser` (returns freshly-saved doc) | `createSuperAdmin`, `createStoreAdmin`, `createStoreStudentUser` responses |

Root cause: `.populate("user_id")` / `User.find*` with **no field projection**, and returning the raw saved doc from `registerUser`.

**Fix (minimal, allow-secret-exclusion):** exclude the secret on every client-facing path — `.populate("user_id", "-passwordHash")` / `{ path: "user_id", select: "-passwordHash" }`, `.select("-passwordHash")` on the raw `User.find*` reads, and strip it from `registerUser`'s return via `toObject()` destructure. The **login** path (`findUserByEmail`, used by `bcrypt.compare`) intentionally keeps the hash; its response object is already curated. Note the User schema's only secret is `passwordHash` (no salt/reset-token fields), so `-passwordHash` fully closes it. Defense-in-depth option not taken (per "keep it minimal"): `select:false` on the schema field + `.select("+passwordHash")` in login — more robust but unreliable here because most reads use `.lean()`/aggregate, which bypass schema `toJSON` transforms.

**Residual/notes:** dead `findUser` in `controllers/contact_info.js` & `controllers/group.js` reference an **unimported** `User` (would `ReferenceError`) and aren't routed — left as-is. Verified with `node --check`; no test suite in the backend.

---

## Summary

| ID | Severity | Description |
|---|---|---|
| SEC-01 | ~~CRITICAL~~ ✅ FIXED | `/users/register-superadmin` had no auth — anyone could create a superadmin → now `authMiddleware + isSuperAdmin`, first one seeded via `scripts/create-superadmin.js` (2026-09-26) |
| SEC-02 | ~~CRITICAL~~ ✅ FIXED | `change-password` (GET) had no authz → now POST, identity from the token, sessions revoked (verified 2026-09-26) |
| SEC-03 | HIGH | `createStoreAdmin` mis-aligned `registerUser` args → store-user saved with `store:""`, outside transaction |
| SEC-04 | HIGH | Socket handshake unauthenticated; client self-declares `storeId/role` |
| SEC-05 | MEDIUM | JWT secret + plaintext password + tokens logged to console |
| SEC-06 | LOW | `activate/deActivateUser` use `if (res)` → always-success, dead else branch |
| SEC-07 | CRITICAL | bcrypt `passwordHash` shipped to clients via bootstrap/user reads/`registerUser` — **FIXED 2026-07-02** |
