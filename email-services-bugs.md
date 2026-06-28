# Email Services — Bug Report

> Bugs found (and several fixed) during the initial build of the email feature, 2026-06-26.
> Backend files: `controllers/emails.js`, `controllers/handlers/emails.js`, `routes/emails.js`,
> `services/{sesMailer,emailDispatcher,emailAttachments,emailTemplate}.js`, `jobs/emailScheduler.js`,
> `models/email.js`. Frontend: `emails/*`, `services/emails.service.ts`, `state/emails/*`.
> Full-stack reference: `Frontistirio-email-services-brain-repo/README.md`.

> **Note on the two systemic defects:** this feature was written to AVOID both of them.
> The reads (`getStoreEmails`/`getEmail`) filter `isDeleted:false`, and `watchEmails` →
> `emitToStoreUsers` uses the correct `Promise.all(...).then(filter)` pattern (awaits
> `isStoreUser` before filtering), so it does **not** carry the RT-01/RT-02 cross-tenant
> leak that the other ~10 watchers do. Keep it that way if you copy this handler.

---

## ✅ Fixed during the build

### BUG-EML-001 · Change-stream pre-image requirement crashed the whole process · **FIXED**

**Severity:** Critical (process crash)
**File:** `controllers/handlers/emails.js` — `watchEmails`

**Problem:**
The watcher was opened with `fullDocumentBeforeChange: "required"` (copied from
`watchEducationalMaterial`). The other collections have `changeStreamPreAndPostImages`
enabled in Atlas, but the **new `emails` collection does not**. On the first `update`
event MongoDB errored (`code 47 NoMatchingDocument — pre-image was not found`), and because
the change stream's `error` event was **unhandled**, Node crashed the entire API
(`nodemon app crashed`).

**Fix (applied):**
- Switched to `fullDocumentBeforeChange: "whenAvailable"` — we soft-delete, so we never
  need the before-image (the `delete` branch tolerates `before == null`).
- Added a `changeStream.on("error", …)` handler that logs and re-establishes the watcher
  after 5s, so a future transient stream error can never take the process down.

```js
const changeStream = Email.watch([], {
  fullDocument: "updateLookup",
  fullDocumentBeforeChange: "whenAvailable", // was "required" → crash
});
changeStream.on("error", (err) => {
  console.error("⚠️ Email change stream error — restarting in 5s:", err.message);
  try { changeStream.close(); } catch (_) {}
  setTimeout(() => exports.watchEmails(io, userSockets), 5000);
});
```

> The other ~10 watchers still use `"required"` and have **no** `error` handler — they only
> survive because pre-images happen to be enabled on those collections. A single stream
> error there would crash the API too. See `real-time-sync-bugs.md`.

---

### BUG-EML-002 · Message rollup stuck on `sending` forever · **FIXED**

**Severity:** High (UX / data correctness)
**File:** `models/email.js` — `recomputeRollup()`

**Problem:**
`recomputeRollup` counted a recipient whose status was `sent` (SES accepted it, awaiting the
delivery webhook) in the **same bucket as `pending`**, then set the message to `sending`
whenever `pending > 0`. With no SNS webhook configured the `delivered` event never arrives,
so a fully-sent email stayed on `sending` **permanently** — the user reported "stays in
sending, no notification".

**Fix (applied):**
Separate `notDispatched` (truly `pending`) from `sent`. The message reaches `sent` as soon
as everything is handed off; individual recipients upgrade `sent → delivered` later if/when
the webhook is wired.

```js
if (notDispatched > 0)          this.status = "sending";
else if (failed && !handedOff)  this.status = "failed";
else if (failed)                this.status = "partially_failed";
else                            this.status = "sent";   // handedOff = sent + delivered
```

Existing stuck rows are repaired with `scripts/recomputeEmails.js` (idempotent).

---

## 🟠 Open findings (not yet fixed)

### BUG-EML-003 · Sending From a `@gmail.com` address lands in spam

**Severity:** High (deliverability — feature unusable for real recipients)
**Where:** `.env` `SES_FROM=kiziridis.k2000@gmail.com`; used in `controllers/emails.js` (`fromAddress`).

**Problem:**
Sending "From `@gmail.com`" through SES fails SPF (Amazon IP not in gmail.com's record),
DKIM (can't sign for a domain you don't own) and therefore **DMARC alignment for gmail.com**.
Gmail aggressively spam-folders mail that spoofs its own domain. Confirmed in testing — the
message arrived in **Spam**.

**Fix:** verify an **owned domain** in SES, enable **Easy DKIM** (3 CNAMEs), publish SPF
(`include:amazonses.com`) + a DMARC record, and set `SES_FROM=no-reply@<domain>`. No code
change — only `.env` + DNS.

---

### BUG-EML-004 · `/emails/ses-webhook` does not verify the SNS signature

**Severity:** Medium (security — status spoofing)
**File:** `controllers/emails.js` — `handleSesWebhook`

**Problem:**
The webhook is public (correctly — AWS calls it) but trusts **any** POST that looks like an
SNS notification. An attacker who learns the URL can POST forged `Delivery`/`Bounce` events
to flip recipients' statuses (or POST a `SubscriptionConfirmation` with an arbitrary
`SubscribeURL`, which we blindly `GET`). No data/PII is exposed, but the delivery dashboard
can be falsified.

**Fix:** verify the SNS message signature (download `SigningCertURL` — validate it's an
`*.amazonaws.com` cert — and check the signature over the canonical fields) before acting;
restrict `SubscribeURL` confirmation to `sns.*.amazonaws.com` hosts. The `sns-validator`
npm package does this.

---

### BUG-EML-005 · A crash mid-dispatch strands an email in `sending`

**Severity:** Medium (reliability)
**File:** `services/emailDispatcher.js` (called by `jobs/emailScheduler.js`)

**Problem:**
`claimDueScheduledEmail` atomically flips a scheduled email `scheduled → sending`, then
`dispatchEmail` sends. If the process dies between the claim and the final `save()`, the
email is left `status:"sending"` with `pending` recipients — and the scheduler won't re-claim
it (it only looks for `status:"scheduled"`). It silently never sends.

**Fix:** add a startup/periodic recovery sweep that re-dispatches `status:"sending"` emails
that still have `pending` recipients (idempotent — already-`sent` recipients are skipped).

---

### BUG-EML-006 · Immediate send loops recipients synchronously in the request

**Severity:** Medium (scalability)
**File:** `controllers/emails.js` → `services/emailDispatcher.js`

**Problem:**
For an immediate send, `await dispatchEmail()` runs **inside the HTTP request**, making one
SES call per recipient sequentially. Fine for school-size lists; a large blast blocks the
request and risks a client/gateway timeout.

**Fix:** enqueue the send (BullMQ) and return immediately; the worker drains it. The
per-recipient status UI already updates live over the socket, so the UX is unaffected.

---

### BUG-EML-007 · In-process scheduler — no distributed retry/backoff

**Severity:** Low
**File:** `jobs/emailScheduler.js`

**Problem:**
`startEmailScheduler()` is a per-process `setInterval`. The atomic `claimDueScheduledEmail`
**does** prevent double-sends even across multiple instances, so this is not a correctness
bug — but there's no retry/backoff, the timer dies with the process, and N instances each
spin a redundant timer.

**Fix:** move scheduling to BullMQ delayed jobs (same handler API) when the app scales past
one instance.

---

### BUG-EML-008 · Orphaned ad-hoc attachments accumulate in S3

**Severity:** Low (cost / housekeeping)
**File:** `services/emailAttachments.js` — `uploadAdHocAttachments`

**Problem:**
Files uploaded at compose time are stored under `email-attachments/{group}/{store}/...` and
**never deleted** — not even if the send fails or the email is later soft-deleted.

**Fix:** an S3 lifecycle policy on the `email-attachments/` prefix, or a cleanup job tied to
email soft-delete.

---

### BUG-EML-009 · `applyDeliveryEvent` email-fallback can match the wrong email

**Severity:** Low
**File:** `controllers/handlers/emails.js` — `applyDeliveryEvent`

**Problem:**
The primary match is by the unique `providerMessageId` (correct). The fallback path
(`{ "recipients.email": recipientEmail }`) is **not scoped** — if the same address appears in
multiple emails and a `providerMessageId` is somehow missing, `findOne` could update an
unrelated email's recipient.

**Fix:** the fallback is rarely hit (every send stores a `providerMessageId`); if kept, scope
it (e.g. most-recent email for that address) or drop it entirely.

---

### BUG-EML-010 · Total attachment size only checked after building the MIME

**Severity:** Low
**File:** `services/sesMailer.js`

**Problem:**
multer caps each file at 10 MB, but several files + materials combined can exceed the 10 MB
SES message limit. We only detect it **after** building the full MIME (per recipient), then
throw `attachment_too_large` → that recipient is marked `failed`. Wasteful and surfaces late.

**Fix:** sum attachment sizes up front in the controller and reject the send with a clear
message before creating the email / building any MIME.
