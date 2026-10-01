# UI / Layout — Bug Report

> File: `src/global.scss`. Frontend-only, cosmetic — but it affects **every page that uses
> `.breadcrumb-bar`**, which is nearly all of them. Found 2026-07-23 while fixing the oversized
> buttons on the test-cycle-details and classroom-diagram pages.

---

## UI-01 · MEDIUM — `.breadcrumb-actions ion-button { flex: 1 1 auto }` inflates every header button

**File:** `src/global.scss` line ~156

```scss
.breadcrumb-actions ion-button {
  flex: 1 1 auto; /* let buttons grow equally if needed */
  min-width: 0;
  max-width: 100%;
}
```

`flex-grow: 1` is unconditional, so header action buttons stretch to fill whatever width the actions
container gets, regardless of their label. Measured on `test-cycle-details` at a 1058px bar:
`Add test day` **503px**, `Export report` **507px** — for two-word labels. The comment says "if
needed", but nothing makes it conditional.

**Fix:** `flex: 0 0 auto` in the base rule, and keep `flex: 1 1 auto` only inside the existing
`@media (max-width: 768px)` block, where full-width tap targets are actually wanted.

---

## UI-02 · MEDIUM — `.page-background` kills `.breadcrumb-bar`'s flex layout (source-order clash)

**File:** `src/global.scss` — `.breadcrumb-bar` line ~125 vs `.page-background` line ~389

Markup across the app is `<div class="breadcrumb-bar page-background">`. Both rules set `display`
at the same specificity, and `.page-background` is declared **later**, so it wins:

```scss
.breadcrumb-bar  { display: flex; flex-flow: row; justify-content: space-between; }  /* line 125 */
.page-background { display: flow-root; }                                            /* line 389 */
```

Computed `display` on such an element is **`flow-root`**, not `flex`. Consequences: the
`justify-content: space-between`, the `ion-breadcrumbs { flex: 1 }` and the actions' `flex-shrink: 0`
are all inert, and the action buttons render as a **block on their own row** under the breadcrumbs
instead of sitting at the right end of the bar. Combined with UI-01 this is what produced the
full-width buttons (the κάτοψη `Save` measured **1054px**).

`flow-root` is deliberate — it establishes a BFC so a last child's bottom margin does not escape and
leave an unpainted strip (see the comment at line 393). That reasoning does **not** conflict with
flex: a flex container also establishes a formatting context and child margins do not collapse
through it.

**Fix:** raise the specificity of the bar rule, e.g. `.breadcrumb-bar.page-background { display: flex; }`
in `global.scss` (or reorder the two blocks).

---

## Status

**FIXED globally — 2026-09-26.** Both rules now live in `src/global.scss`:

- **UI-02** — a more specific `.breadcrumb-bar.page-background { display: flex }` (line ~152) is
  declared after `.page-background`, so the bar flexes again and the actions stay on the
  breadcrumbs' row.
- **UI-01** — the base `.breadcrumb-actions ion-button` rule no longer sets `flex` at all;
  `@media (min-width: 769px)` gives it `flex: 0 0 auto` (height 34px) and the
  `@media (max-width: 768px)` block `flex: 0 0 auto` (height 36px). Nothing grows unconditionally.

The two page-scoped workarounds added on 2026-07-23 were **removed** the same day
(`test-cycle-details.component.scss`, `classroom-diagram.component.scss`); each now keeps only its
own button cosmetics (weight, radius, padding, the κάτοψη Save's green shadow) and its deliberate
`@media (max-width: 768px)` rule that stretches those two pages' actions to full-width tap targets.

Measured on the harness at the bug report's own 1058px bar: the two test-cycle buttons come out
**161px / 169px** with `flex-grow: 0` (they were 503px / 507px), computed `display` on the bar is
`flex`, and the actions sit at its right end on the same row as the crumbs. At 390px the bar is two
rows, the actions right-aligned, no horizontal page scroll.
