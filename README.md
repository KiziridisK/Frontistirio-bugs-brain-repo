# Frontistirio — Bugs Brain

> Cross-feature bug & weakness register for **Frontistirio** (Angular/Ionic frontend) and
> **Frontistirio-API** (Node/Express/MongoDB backend). Each file is a per-feature deep-dive
> with file/line references and suggested fixes. Severities: CRITICAL → HIGH → MEDIUM → LOW.

---

## Reports

| File | Scope | Highest severity |
|---|---|---|
| [authentication-security-bugs.md](authentication-security-bugs.md) | login, users, socket auth | **CRITICAL** |
| [real-time-sync-bugs.md](real-time-sync-bugs.md) | Change Streams → Socket.IO → NgRx | **CRITICAL** |
| [bootstrap-bugs.md](bootstrap-bugs.md) | `/bootstrap/getSingle` | HIGH |
| [store-management-bugs.md](store-management-bugs.md) | stores, teaching periods (scoping key) | HIGH |
| [test-cycles-bugs.md](test-cycles-bugs.md) | test cycles & tests | HIGH |
| [pdf-generation-bugs.md](pdf-generation-bugs.md) | pdfkit + S3 export | HIGH |
| [grading-system-bugs.md](grading-system-bugs.md) | grades / student-course-grade | HIGH (doc/contract) |
| [student-management-bugs.md](student-management-bugs.md) | student CRUD & relations | MEDIUM |
| [course-class-management-bugs.md](course-class-management-bugs.md) | courses & classes | MEDIUM |
| [educational-material-bugs.md](educational-material-bugs.md) | educational material | HIGH (earlier pass; BUG-011 grade-enforcement fixed 2026-06-25 w/ course-level perms) |
| [private-lessons-bugs.md](private-lessons-bugs.md) | private lessons & rates | CRITICAL (partly fixed — see staleness note) |
| [email-services-bugs.md](email-services-bugs.md) | email send/schedule + SES delivery tracking | HIGH (2 fixed 2026-06-26; deliverability + webhook-auth open) |

---

## The two systemic, cross-cutting defects (fix these first)

1. **Real-time store isolation is broken (RT-01/RT-02).** Every watcher's `update` branch uses
   `Array.prototype.filter(async …)`, which lets **all** connected users pass → entity updates
   are broadcast to **every store** (cross-tenant leak). And `isStoreUser` is an async DB lookup
   used in a synchronous boolean, so the role gate is a no-op on insert/delete. Same two-line
   mistake is copy-pasted into ~10 handlers.

2. **Soft-delete is not honored on read paths.** Students, courses, classes and grades are fetched
   without `isDeleted: false` (the period aggregations only `$match { store_id }`). The brain docs
   claim "all queries include `isDeleted:false`" — they don't. Soft-deleted records (incl. PII) ship
   in the bootstrap payload; the frontend only masks some of them in list views.

Also high-impact and quick: open `/users/register-superadmin` (no auth) and the
`change-password` IDOR (`authentication-security-bugs.md` SEC-01/SEC-02).

---

## Recurring anti-patterns observed

- **`session.abortTransaction()` not awaited** before `finally { session.endSession() }`.
- **Handlers/controllers that don't always send a response** (empty `catch {}`, success-only `if`
  with no `else`) → request hangs until client timeout.
- **`const periodId = default_period._id`** with no null check — throws whenever a store has no
  default period (which itself is too easy to reach — see store-management SM-01/SM-02/SM-03).
- **`findByIdAndUpdate(_id, { $set: req.body })`** — no store scoping (IDOR) + mass assignment.
- **Heavy `console.log` of secrets / passwords / tokens / full documents** across controllers.
- **Brain-doc drift** — several docs describe an earlier version of the code (field names, S3
  buckets, helper sync/async, soft-delete guarantees). Mismatches are flagged inline per report.

---

## Method / provenance

Findings come from reading the actual `Frontistirio-API` source (controllers, handlers, models,
routes, middleware, helpers, `index.js`) and cross-referencing each feature's brain repo. Line
numbers reference the code as of this pass (June 2026). Items marked "verify" still need a quick
confirmation against the Angular side (reducers/services).
