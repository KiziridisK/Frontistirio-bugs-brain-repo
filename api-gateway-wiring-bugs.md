# API Gateway wiring — Bug Report

> Scope: the **REST API Gateway** (`2ba6w3j324`, stage `Prod`) that fronts the Express backend on
> EC2 — not the backend code. These are cases where the gateway's method/resource map disagreed with
> what `Frontistirio-API/routes/*` actually serves, so a request never reached Express at all.
> Found 2026-07-23 by **diffing every Express endpoint against the live gateway** (see the AWS
> runbook `Frontistirio-aws-infrastructure-brain-repo` §0, deployment `cxdpxp`) — the first time the
> two sides were compared as a whole rather than one feature at a time.
>
> **The probe that classifies every one of these** (runbook §0):
> `403 Missing Authentication Token` = the *gateway* has no matching method (wiring bug, the subject
> of this file) · `404 Cannot … (HTML)` = gateway reached EC2 but the *backend* lacks the route (not
> deployed) · `401 No token provided (JSON)` = fully wired, auth middleware rejected. **Cache-bust the
> probe** (`?cb=$RANDOM`) — this is an edge-optimized API and stale 403s replay from CloudFront
> (runbook §5.18).

---

## GW-01 · HIGH (FIXED 2026-07-23) — six endpoints wired with the wrong HTTP verb → 403 in production

The gateway carried a **different method** than the frontend sends, so every call returned
`403 Missing Authentication Token` and the feature was dead in the deployed app. The backend had the
correct route the whole time — only the gateway method was wrong — so each one was **user-visible
breakage in prod with no backend error to show for it** (the request never arrived).

| Path | Gateway had | App sends (`routes/*.js` + service) | User-facing consequence |
|---|---|---|---|
| `/teaching-periods/get-store-teaching-period` | GET | POST | period detail / import broken |
| `/teaching-periods/get-store-teaching-period-classes` | GET | POST | ” |
| `/teaching-periods/get-store-teaching-period-students` | GET | POST | ” |
| `/student-course-grades/approve-course-grade` | GET | POST | admin **grade approval** broken |
| `/test-cycles/delete-store-test-cycle` | DELETE | POST | cannot delete a test cycle |
| `/test-cycles/delete-test-cycle-test` | DELETE | POST | cannot remove a test from a cycle |

Express side is POST for all six, e.g. `routes/teaching_periods.js`
(`router.post("/get-store-teaching-period", …)`), `routes/student-course-grade.js`
(`router.post("/approve-course-grade", …)`), `routes/tests.js`
(`router.post("/delete-test-cycle-test", …)`); frontend services POST too
(`teaching-periods.service.ts`, `course-grade-service.service.ts`, `test-cycle.service.ts`).

**Fix applied:** added the correct POST method (HTTP_PROXY → EC2, CORS OPTIONS) alongside the
pre-existing wrong-verb method. The stale GET/DELETE methods were **left in place** — adding the right
verb is purely additive and fixes the feature instantly; removing the wrong one is a separate,
riskier change (nothing calls it, but deleting a live method is worth its own deliberate pass). These
six went **live immediately** because EC2 already served the routes. Verified 401 post-deploy.

**Follow-up (open):** remove the six orphaned GET/DELETE methods so the gateway map matches Express
1:1. Low risk (no caller), but do it as its own change with a fresh diff.

---

## GW-02 · HIGH (FIXED 2026-07-23) — `GET /announcements/get-announcement/{announcementId}` was method-less

The `{announcementId}` child resource existed but carried **no method**; the `GET` sat on the
**parent** `/announcements/get-announcement` with an unmapped `{announcementId}` baked into its
integration URI. So the announcement-detail read returned `403 Missing Authentication Token` in prod —
the announcement detail page was broken. First recorded in the runbook's 2026-07-16 entry and left
unfixed until now.

**Fix applied:** a properly mapped `GET` on the `{param}` child —
`method.request.path.announcementId: true` + `integration.request.path.announcementId` mapped onto it,
HTTP_PROXY to `…/announcements/get-announcement/{announcementId}` — plus a MOCK OPTIONS. This is the
**path-param rule** the runbook §0 documents: a `{param}` method belongs on the child, never the
parent. Verified 401 post-deploy.

---

## GW-03 · HIGH (FIXED 2026-09-25) — five resources with the right path but an integration URI pointing at a *different* Express route

A whole defect class GW-01/GW-02 could not see. The resource path and method were **correct**, so
every diff and every probe said "healthy" — but the `HTTP_PROXY` **integration URI** forwarded to
another path. Found by comparing, for all 262 integrations, the resource path against the path in
its own URI (`scripts/gateway-audit.js` check B in the AWS runbook repo).

| Path (resource — correct) | Integration URI forwarded to | Symptom in prod |
|---|---|---|
| `POST /grade-scales/create-grade-scale` | `…/create-grade-scale**s**` | `404` — cannot create a grading scale |
| `POST /grade-scales/edit-grade-scale` | `…/edit-grade-scale**s**` | `404` — cannot edit |
| `DELETE /grade-scales/delete-grade-scale` | `…/delete-grade-scale**s**` | `404` — cannot delete |
| `GET /test-cycles/get-teacher-test-cycle-tests` | `…/get-store-test-cycles` | **silently wrong data** — a teacher got the store-wide test-cycle list instead of their own tests |
| `POST /students/upsert-student-details` | `…/create-store-student` | **silently wrong write** — saving the student «Εγγραφές» tab hit the *create* handler with `{changes, studentId}` |

All five are called by the frontend (`grade-scales.service.ts` ×3, `test-cycle.service.ts`,
`student-service.service.ts`) and all five exist in Express at the correct path, guarded by
`authMiddleware`.

**Two severities in one table.** The three `grade-scales` typos were *loud* — a plain `404`. The other
two were **worse**: the wrong handler is also auth-guarded, so a token-less probe answers `401` exactly
like a healthy endpoint, and an authenticated call returns `200` with the wrong data. `get-teacher-test-cycle-tests`
handed a teacher a store-scoped list (a scope leak as well as a bug); `upsert-student-details` routed an
update into `createStoreStudent`.

**Fix applied:** `update-integration --patch-operations op=replace,path=/uri` on each, setting the URI
path equal to the resource path — the invariant the other 257 integrations already satisfy. Old values
backed up first. Verified with `test-invoke-method` **before** deploying (it bypasses the edge cache and
needs no deployment): all five `401 NO_TOKEN` with the correct `Endpoint request URI`. Deployed
**Prod `tg9ssq` + dev `825ay1`**, then re-probed live: all five `401` on both stages (the three
`grade-scales` flipped `404 → 401`; dev served an edge-cached `404` on `delete-grade-scale` for about a
minute first — runbook §5.18).

**Why now and not in 2026-07-23:** GW-01's diff compared `"METHOD path"` on both sides. That is exactly
blind to this — the path *is* right. Nothing short of reading each integration's URI finds it.

---

## GW-04 · MEDIUM (OPEN) — two integrations are plain `HTTP`, not `HTTP_PROXY`, so backend status codes collapse to `200`

| Path | Effect |
|---|---|
| `POST /test-cycles/export-store-test-seating` | Express returns `401 {"code":"NO_TOKEN"}`; the **client receives `200`** with that body |
| `POST /users/register-superadmin` | same collapse |

Both carry a single `integrationResponses` entry `{"200": {statusCode: "200"}}` and no selection
pattern, so *every* backend status — 401, 404, 500 — is rewritten to `200`. The body passes through, so
the caller sees a success status wrapping an error payload and any `catch`/`if (!res.ok)` branch in the
frontend never fires. A non-proxy integration also does not pass binary through without explicit binary
media types, which matters for a PDF-export endpoint.

**Not fixed here** — unlike GW-03 this is not a typo with one obviously-correct value: converting
`HTTP → HTTP_PROXY` drops the integration/method responses and changes response handling on an endpoint
that works today for authenticated callers. Worth its own deliberate pass with the frontend callers in
hand. Detected by `gateway-audit.js` check C, which fails the run while they remain.

**Related, not a gateway bug:** `register-superadmin` is also tracked as **SEC-01** in
`authentication-security-bugs.md` (still open, re-confirmed 2026-09-25). The `200` collapse only makes
it quieter. ⚠ This repo is **public** — SEC-01 is a live production hole; see the note in `README.md`
about making the repo private before adding any more detail to it.

---

## Root cause & prevention

None of these were introduced by the feature work of 2026-07-23; they were **latent gateway drift**
that only surfaced because this was the first Express↔gateway *diff*. The earlier per-feature scripts
(`add-calendar-endpoints.sh`, `add-course-syllabus-endpoints.sh`) only ever looked at the endpoints
they were adding, so a wrong verb or orphaned param on a path nobody was touching stayed invisible.

**2026-09-25 update — the prevention below was necessary but NOT sufficient.** GW-03 proved a
`"METHOD path"` diff is blind to a wrong integration URI, and a probe sweep is blind to it too (the
wrong handler answers `401` just like the right one). The gateway has **two** sides per endpoint and
only one of them was ever being checked. Use **`scripts/gateway-audit.js`** in the AWS runbook repo
instead of `route-diff.js` alone: it runs routability (A), **URI-vs-path (B)**, integration type (C),
orphans (D) and public routes (E), and exits non-zero on A/B/C. Since the root `/{proxy+}` catch-all
(2026-09-21) a *missing* resource is no longer a bug at all — which makes B and C the only checks that
still find real breakage.

**Prevention:** run the full diff each round, not a hand-written feature list. `sync-missing-endpoints.sh`
in the AWS runbook is table-driven off exactly this diff — re-derive it (enumerate Express by walking
each router's `router.stack`; enumerate the gateway via `get-resources`; diff on `"METHOD path"` with
`:param`→`{param}`) and a clean run should surface **only** `POST /adress/create` (a dead route no
frontend calls). Anything else is a new drift to fix.

**Related:** the class of "the endpoint doesn't work" reports is settled in one cache-busted curl via
the 403/404/401 triad above — reach for it before assuming the backend is at fault.
