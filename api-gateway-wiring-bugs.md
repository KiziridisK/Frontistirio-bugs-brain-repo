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

## Root cause & prevention

None of these were introduced by the feature work of 2026-07-23; they were **latent gateway drift**
that only surfaced because this was the first Express↔gateway *diff*. The earlier per-feature scripts
(`add-calendar-endpoints.sh`, `add-course-syllabus-endpoints.sh`) only ever looked at the endpoints
they were adding, so a wrong verb or orphaned param on a path nobody was touching stayed invisible.

**Prevention:** run the full diff each round, not a hand-written feature list. `sync-missing-endpoints.sh`
in the AWS runbook is table-driven off exactly this diff — re-derive it (enumerate Express by walking
each router's `router.stack`; enumerate the gateway via `get-resources`; diff on `"METHOD path"` with
`:param`→`{param}`) and a clean run should surface **only** `POST /adress/create` (a dead route no
frontend calls). Anything else is a new drift to fix.

**Related:** the class of "the endpoint doesn't work" reports is settled in one cache-busted curl via
the 403/404/401 triad above — reach for it before assuming the backend is at fault.
