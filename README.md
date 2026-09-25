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
| [ota-live-updates-bugs.md](ota-live-updates-bugs.md) | OTA live updates (Capgo + S3/CloudFront) | **CRITICAL** (all 4 fixed 2026-07-16; no-rollback + no-staged-rollout open) |
| [ui-layout-bugs.md](ui-layout-bugs.md) | `global.scss` breadcrumb-bar layout (frontend, cosmetic) | MEDIUM (2026-07-23; fixed on 2 pages, global fix open) |
| [api-gateway-wiring-bugs.md](api-gateway-wiring-bugs.md) | REST API Gateway method/param map **and integration URIs** vs Express routes | HIGH (2026-09-25; 12 wiring bugs fixed — GW-03 added 5 wrong-URI ones incl. 2 that returned wrong data silently; GW-04 + orphaned-verb cleanup open) |
| [panellinies-bugs.md](panellinies-bugs.md) | Πανελλήνιες / μηχανογραφικό (IDEA-09) | HIGH (2026-09-20; 5 fixed προ-deploy, 4 ανοιχτά — το PAN-10 αφορά **16 άλλα αρχεία** του app) |
| [assignments-bugs.md](assignments-bugs.md) | εργασίες σπιτιού (IDEA-06): παράδοση, διόρθωση, αρχεία S3 | MEDIUM (2026-09-21, από ανάγνωση κώδικα· ASG-01 η διόρθωση χάνεται σε νέα παράδοση) |

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
   **Partially fixed 2026-07-14:** the 4 hot period aggregations
   (`fetchStorePeriodStudents/Courses/Classes/Teachers`) now `$match { …, isDeleted:{$ne:true} }`
   (verified: 5 soft-deleted teachers stopped leaking into bootstrap). The rest of the read paths are
   still unfixed. See the **Frontistirio-database-indexing-brain-repo** for the full pass + index catalog.

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
- **Same-specificity CSS clashes in `global.scss`** decided by source order — a later utility class
  silently disables an earlier layout rule on elements that carry both (see `ui-layout-bugs.md`).

---

## Method / provenance

Findings come from reading the actual `Frontistirio-API` source (controllers, handlers, models,
routes, middleware, helpers, `index.js`) and cross-referencing each feature's brain repo. Line
numbers reference the code as of this pass (June 2026). Items marked "verify" still need a quick
confirmation against the Angular side (reducers/services).
