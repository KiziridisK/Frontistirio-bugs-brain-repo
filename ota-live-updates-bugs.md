# OTA Live Updates — Bug Report

> Bugs found (and all four fixed) while building the OTA live-update feature, 2026-07-16.
> Frontend: `src/app/services/live-update.service.ts`, `capacitor.config.ts`, `src/environments/*`.
> Tooling: `scripts/deploy-ota.js`, `package.json` (`ota.*`, deploy scripts).
> Infra: `s3://frontistirio-bucket-dev/ota/` behind CloudFront `EGTO4YGAVYP3O`.
> Full reference: `Frontistirio-ota-live-updates-brain-repo/README.md`.

> **Note on the two systemic defects:** neither applies here. This feature touches **no** MongoDB
> collection, **no** change-stream watcher and **no** API route — it is a static file on CloudFront
> plus a client-side service. That was a deliberate design goal (see the brain repo §2), not luck.

> **Unusual for this register:** these are not defects discovered in old code. Three of the four were
> introduced *during this build* and caught before or shortly after they could hurt anyone — but
> BUG-OTA-001 and BUG-OTA-002 were both live-fire, and 001 endangered code that predates the feature.

---

## ✅ Fixed during the build

### BUG-OTA-001 · `npm run deploy` silently deletes every OTA bundle · **FIXED**

**Severity:** CRITICAL (breaks every installed app; data loss on the CDN)
**File:** `package.json` — `deploy:s3`

**Problem:**
The pre-existing web deploy is `aws s3 sync ./www s3://frontistirio-bucket-dev --delete`. `--delete`
removes anything in the **bucket** that is absent from `./www`. Putting OTA artifacts in the same
bucket means the first `npm run deploy` after an OTA push wipes `/ota/bundles/*` **and**
`/ota/updates.json`. Installed apps then request a manifest that 404s.

Confirmed, not theorised — dry-run against a probe object:
```
$ aws s3 sync ./www s3://frontistirio-bucket-dev --delete --dryrun
(dryrun) delete: s3://frontistirio-bucket-dev/ota/_probe.txt
```

**Fix (applied):**
```
"deploy:s3": "aws s3 sync ./www s3://frontistirio-bucket-dev --region eu-central-1 --delete --exclude \"ota/*\""
```
Re-verified by dry-run: with the exclude, `/ota/` is untouched.

**Do not regress:** anyone editing `deploy:s3` (or adding a second bucket sync) must keep the exclude.
A separate bucket would be structurally safer but costs another origin + distribution.

---

### BUG-OTA-002 · CORS blocks the manifest fetch — the phone silently never updates · **FIXED**

**Severity:** CRITICAL (feature 100% non-functional; fails silently)
**File:** `src/app/services/live-update.service.ts` — `fetchManifest()`

**Problem:**
The service fetched the manifest with `window.fetch(environment.otaUrl)`. The WebView serves the app
from `https://localhost`, so that request is **cross-origin**, and S3/CloudFront send no
`Access-Control-Allow-Origin`:
```
$ curl -H "Origin: https://localhost" .../ota/updates.json
HTTP/1.1 200 OK
(no access-control-allow-origin)
```
The fetch was rejected by the WebView before reaching the network, landed in `check()`'s own
`catch`, and produced one console line. **Every other signal looked perfect** — manifest live and
correct, zip downloadable, checksum matching, gates logically passing, plugin compiled into the APK —
which is exactly why it burned a full debug cycle and an extra APK.

**Root cause is architectural:** manual mode moves the manifest fetch into JS, and therefore under the
browser's CORS policy. Auto mode's native POST would never have hit this. It is the price of dropping
the backend (brain repo §2).

**Fix (applied):** `CapacitorHttp.get({ url, headers: { 'Cache-Control': 'no-cache' } })` — issued from
native code, where CORS does not apply. **No S3/CloudFront CORS config needed.** Verified it needs no
config flag: `Bridge.registerAllPlugins()` registers `CapacitorHttp` unconditionally, and its
`enabled` flag only gates *patching* `window.fetch`/XHR, not `request()`.
Also added `safeParse` — `CapacitorHttp` only auto-parses `application/json`, so the HTML from the
CloudFront 404→200 rewrite (BUG-OTA-004) arrives as a **string**.

**Cost:** required a new APK. The broken fetch lived inside the update mechanism itself, so it could
not self-heal — the bootstrap trap (brain repo §7).

---

### BUG-OTA-003 · A forgotten `minNative` bump white-screens every device · **FIXED (mechanically)**

**Severity:** CRITICAL (every phone, unrecoverable OTA — needs a store update to fix)
**File:** `scripts/deploy-ota.js`, `package.json` — `ota.native`

**Problem:**
`ota.minNative` gates which APKs may take a bundle. Add a Capacitor plugin, forget to bump it, and
old APKs pass the gate, take a bundle whose JS calls into native code they don't have, and white-screen.
`appReadyTimeout` rolls back only if the bundle fails to boot — which it does here, so devices
recover, but every user sees a broken launch first. Making `release` a one-command flow (correctly)
made forgetting the bump trivial. **Caught by the user during review, not by testing.**

**Fix (applied):** the script records the native surface (sorted `name@version` of every Capacitor
plugin + a hash of `capacitor.config.ts`) in `package.json` `ota.native` and **refuses to publish**
when it changed, unless `minNative === version` or `--accept-native-change`. Re-blessed only after a
real publish; a dry run never writes.

All three paths verified:
```
plugin added, minNative forgotten   → ✖ refuses, names the plugin, gives the 3-step fix
plugin added, minNative == version  → ✓ publishes, prints what changed
ordinary JS-only release            → ✓ publishes silently (no false positive)
```

**Detection method:** a package is a Capacitor plugin exactly when its `package.json` has a
`capacitor` key — verified to reproduce `cap sync`'s list exactly, no false positives.

**Residual gap:** the surface covers plugins + `capacitor.config.ts` only. Hand-editing
`AndroidManifest.xml` or `variables.gradle` does **not** trip it (see Open, below).

---

### BUG-OTA-004 · Unquoted `--cache-control` breaks the manifest upload · **FIXED**

**Severity:** HIGH (every OTA publish fails at the last step — but loudly)
**File:** `scripts/deploy-ota.js` — `aws()`

**Problem:**
`aws` is a `.cmd` on Windows, so it must run through a shell; Node then joins argv **raw**, without
quoting. `--cache-control` `no-cache, max-age=0` split on its space into two arguments. Reproduced:
```
WITHOUT quoting: ["--cache-control","no-cache,","max-age=0", …]
WITH quoting:    ["--cache-control","no-cache, max-age=0", …]
```

**Fix (applied):** `execSync` with a hand-quoted command string (`quote()` wraps any arg containing
`[\s,"&|<>^]`). Caught by reading the dry-run output before the first real publish.

---

## ⚠️ Open weaknesses (accepted, not bugs)

| # | Severity | Issue |
|---|---|---|
| OTA-W1 | HIGH | **No rollback command.** Recovering a bundle that boots but misbehaves means manually rebuilding an older `www/` and re-publishing. `appReadyTimeout` only catches bundles that fail to *boot*. A `deploy:ota:rollback <version>` (re-point `updates.json` at an older, already-uploaded zip) would be ~20 lines and is the highest-value follow-up. |
| OTA-W2 | MEDIUM | **No staged rollout, no stats, no channels.** Every device takes every bundle at once; a bad push reaches 100% immediately. Structural cost of dropping the server (brain repo §2). |
| OTA-W3 | MEDIUM | **`ota.native` doesn't cover `android/`.** `AndroidManifest.xml`, `variables.gradle`, icons and splash are all native and all invisible to the guard. `android/` is gitignored, so there is nothing stable to hash — the guard is a floor, not a ceiling. |
| OTA-W4 | LOW | **CloudFront's 403/404 → `/index.html` 200 rewrite makes every misconfiguration fail soft**, as HTML with a success status. The shape check contains it, but a broken path will always *look* like "no update available" rather than an error. A dedicated `/ota/*` behavior without the rewrite would fix it properly. |
| OTA-W5 | LOW | **Bundles accumulate in S3 forever.** `autoDeletePrevious` prunes the *device*, nothing prunes the bucket. ~1.6MB per release. A lifecycle rule on `ota/bundles/` would cap it. |
| OTA-W6 | LOW | **`minSdkVersion` was raised 22 → 23** (forced by capgo's `androidx.work` chain). Drops Android 5.1 — negligible in 2026, but it is a device-reach reduction that was made for a tooling reason, not a product one. |
