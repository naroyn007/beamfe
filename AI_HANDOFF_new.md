## SSO HANDSHAKE — PLATFORM HUB ↔ BEAM//FE

### Current Status

The Platform Hub is the central authentication/launcher for the engineering tools.

Platform Hub and BEAM//FE use the **same Supabase projects**:

* Development: `https://xhjqnkpsuyzodgnhhuwb.supabase.co`
* Production: `https://swhilchfrytknrbjhrlj.supabase.co`

### Platform Hub SSO Bridge

`auth-confirm.html` already implements the cross-subdomain session cookie bridge.

On successful authentication or token refresh:

```js
function writeSessionCookies(session) {
  if (!canBridge || !session) return;
  const maxAge = 60 * 60 * 24 * 30;
  document.cookie = `sb-access-token=${session.access_token}; Domain=${COOKIE_DOMAIN}; Path=/; Max-Age=${maxAge}; SameSite=Lax; Secure`;
  document.cookie = `sb-refresh-token=${session.refresh_token}; Domain=${COOKIE_DOMAIN}; Path=/; Max-Age=${maxAge}; SameSite=Lax; Secure`;
}
```

The bridge listens for:

```js
supabaseClient.auth.onAuthStateChange((event, session) => {
  if (event === 'SIGNED_IN' || event === 'TOKEN_REFRESHED') {
    writeSessionCookies(session);
  } else if (event === 'SIGNED_OUT') {
    clearSessionCookies();
  }
});
```

The cookie domain is `.roi-engg.com`.

### BEAM//FE SSO TODO

BEAM//FE currently has its own Supabase Auth session and direct email/password login.

It must be updated to consume the Platform Hub cookies on production:

```text
sb-access-token
sb-refresh-token
```

Then establish the Supabase session using:

```js
await sb.auth.setSession({
  access_token,
  refresh_token
});
```

After `setSession()`, the existing BEAM//FE license flow should remain unchanged:

```text
SSO session
    ↓
checkLicenseAndRoute(userEmail)
    ↓
valid Beam Analysis license?
    ├─ YES → open calculator
    └─ NO  → existing license-needed screen
```

### Important Requirements

1. **Do not remove direct email/password login.**
   It remains the fallback, especially for localhost development.

2. **Do not modify Platform Hub's existing cookie bridge unless necessary.**
   The bridge is already working conceptually.

3. **Do not expose or log access/refresh tokens.**

4. **Do not use `.roi-engg.com` cookies for localhost.**
   Localhost should continue using the normal direct-login flow.

5. Production SSO should work across:

   * Platform Hub
   * BEAM//FE
   * Aluma Frame
   * Table Leg Calculator
   * Cuplok

6. The receiving calculator must still perform its own product-specific license check. Authentication ≠ product license.

### Current BEAM//FE Auth Code

BEAM//FE production Supabase:

```js
const SUPABASE_URL =
  'https://swhilchfrytknrbjhrlj.supabase.co';
```

BEAM//FE development Supabase:

```js
const SUPABASE_URL =
  'https://xhjqnkpsuyzodgnhhuwb.supabase.co';
```

Existing license function:

```js
async function checkLicenseAndRoute(userEmail) {
  const result = await callVerifyLicense('check_license');

  if (result.valid) {
    currentLicense = {
      license_type: result.license_type,
      expires_at: result.expires_at
    };
    currentUserEmail = userEmail;
    hideLicenseGate();
    renderLicenseFooter();
    return true;
  }

  // existing license-needed handling remains unchanged
}
```

### Next Development Step

Modify only BEAM//FE `index.html`.

Add an SSO bootstrap before/around the existing:

```js
const { data } = await sb.auth.getSession();
```

The bootstrap should:

1. Detect production `roi-engg.com`.
2. Read the two SSO cookies.
3. If both exist, call `sb.auth.setSession()`.
4. Allow Supabase to refresh the session normally.
5. Run the existing `checkLicenseAndRoute()`.
6. Fall back to the existing login screen if no SSO cookies/session exist.
7. Never print token values to console.

Do **not** change the license database or license Edge Function as part of this SSO handshake unless testing proves it is required.

### Git Safety

Current BEAM//FE branch:

```text
development
```

Remote:

```text
origin/development
```

The previous uppercase/lowercase branch issue has been resolved. Do not force-push or recreate the branch.

Untracked `.vscode/` exists locally and should not be changed unless specifically needed.


---

## Calculator Feature Updates — Session Log

Everything below is scoped to `index.html` (the calculator itself), separate from the SSO work above. Current live state as of this session; check git log / diff against production before assuming any of this is already deployed.

### 1. Sag / Hog moment capacity

Section properties now carry two moment capacities instead of one:

- `allowFbSag` (Sag Moment) and `allowFbHog` (Hog Moment) — new input IDs, replacing the old single `allowFb`.
- Raw solver sign convention: **positive M = sagging, negative M = hogging** (unchanged; this is the raw value *before* the display-only invert that makes the moment diagram draw downward for sagging).
- Demand: `Msag = max(0, Mmax)`, `Mhog = max(0, -Mmin)`, each checked against its own capacity (`bendingSagPass`, `bendingHogPass`); combined `bendingPass` is true only if neither fails.
- **Hog auto-mirrors Sag** in the UI until the user types directly into the Hog field (tracked via `hogEl.dataset.userEdited`), covering the common symmetric-section case with zero extra input.
- **Backward compatible**: old saved models / Excel sheets with a single `allowFb` value load into both Sag and Hog.
- **Excel import headers**: `Mb Sag`, `Mb Hog` (also accepts `Fb`, `Mb`, `Fb Sag`, `M Sag`, `Sag Moment`, etc. — see `EXCEL_FIELD_ALIASES` in the code for the full alias list). If a sheet only has a Sag column, Hog defaults to it on import, same as the UI behavior.
- ⚠️ Already fixed once: the alias list originally didn't include the literal normalized form of the header text shown in the import preview (`Mb Sag` → `mbsag`), so a column named exactly what the UI told the user to name it was silently ignored. Aliases now include `mbsag`/`mbhog` explicitly — if adding more alias variants later, always sanity-check the normalized form (`normalizeExcelHeader()`) against the list.

### 2. Diagram capacity limit lines

Shear, Moment, and Deflection diagrams all draw dashed capacity-limit line(s) via `computeSidedLimitLines()`:

- Only draws a line for whichever side(s) the curve's raw values actually reach (with a small epsilon so a curve that merely touches zero at a support doesn't spuriously draw both sides).
- Moment: sag line at `+FbSag`, hog line at `-FbHog` (raw sign, flipped the same way as the curve itself via the existing `invertY` display flip — no separate sign handling needed).
- Shear: symmetric `±Fv`.
- Deflection: symmetric `±deflAllowIn` (the `L/denom` limit).
- Same lines are drawn on the PDF report's diagrams (`buildPrintReport` → `drawSeries` calls), not just on-screen.

### 3. Instability / uplift warning

A `Free` support carrying a net-uplift reaction (and not fully lifted off, per `solveWithLiftoff`) is now surfaced as an unresolved state, not just a table row:

- `unstableSupports` computed in `runAnalysis`, passed into `R`.
- Persistent red banner above the schematic (`#unstableBanner` / `renderUnstableBanner()`) listing each affected support and the exact kN of tie-down capacity it needs.
- Schematic symbol itself changes for that support (red, dashed, broken triangle + upward warning arrow — distinct from both a normal stable Free support and a fully-lifted-off one).
- Same warning box appears on page 1 of the PDF report (`.pr-unstable`), so a report can't be generated looking "clean" when it isn't.

### 4. Schematic interactivity (major rework this session)

The schematic canvas (`drawSchematic` / `onSchematic*` handlers) now supports hover highlighting and several distinct drag behaviors, layered by hit-region priority in `hitTest()`:

```
order = ['pointload-mag', 'pointload', 'support', 'udl-mag', 'udl-scale', 'udl', 'beam']
```

**Hover highlight**: hovering any item recolors the *actual object* (not an outline) to a bright glow-amber (`HOVER_GLOW = '#FFC93B'`, distinct from the structural amber already used for tie-downs) plus a canvas shadow blur for a genuine glow. Tracked via `hoveredItem = {type, index, sub}` and `isHovered(type, i, sub)`; only redraws on actual target change (not every mousemove).

**Point load** — two independent hit regions, two independent drags:
- `pointload` (line + arrowhead): horizontal drag = change position.
- `pointload-mag` (the value label only): vertical drag = change magnitude, 2-decimal rounding, scale reads off `schematicGeom.maxPointLoadMag`.

**UDL** — three independent hit regions:
- `udl-mag` (sub: `start`/`end`) — one label at **each** end, always shown even when the load is equal (not just when trapezoidal — this was a deliberate change from an earlier version that only showed one shared label for equal loads). Vertical drag on one changes only that end, 2-decimal rounding.
- `udl-scale` — the top connecting line itself (flat when equal, sloped when trapezoidal). Vertical drag scales **both ends by the same multiplicative factor** (not the same absolute delta), so a trapezoid keeps its proportions while resizing and an equal load stays equal throughout the drag. This is the "equal drag" restore — it's what a user drags to bump a uniform load up/down now that the two end-labels are independent. Hit-tested via actual distance-to-line-segment (`pointNearSegment()`), not a bounding box, since the line can be sloped.
- **Double-click the top line** on a trapezoidal UDL snaps it flat to the higher of its two current values (`magStart = magEnd = max(...)`).
- `udl` (broad body region) — horizontal drag = move the whole span; drag near an edge = resize that edge (existing `udlModeAtX()` edge-detection, unchanged).

All magnitude-type drags (`udl-mag`, `udl-scale`, `pointload-mag`) share the same scale convention: a `pxPerUnit` derived from `schematicGeom.arrowSpace / <that item's max magnitude>`, frozen at drag-start so it doesn't shift mid-drag as the value changes. Live drag readout tooltip (top-left of canvas during any drag) shows the value(s) being dragged to in real time.

⚠️ Cleaned up this session: an earlier, half-finished attempt at handling vertical drag directly on the general `udl` body (via a `draggingItem.axis` flag) was found dead in the code — it set an `ns-resize` cursor but the actual value-mutation logic never checked the axis, so dragging vertically on the body did nothing useful. Removed entirely in favor of the dedicated `udl-scale` line handle above. If similar "half-wired" behavior turns up elsewhere, assume it's leftover from an earlier iteration, not intentional.

**Known gap as of this session**: dragging the UDL's `udl-scale` line does not, and should not, affect point loads — they're entirely separate arrays/hit-regions with no shared state. If a user reports the two interacting, it's a hit-region overlap/priority bug, not a data-model issue — check `hitTest()` ordering and the actual pixel geometry pushed for each region first.

### 5. Other UI fixes this session

- **Point load moved & recolored**: line/label pushed 15px above the UDL's max arrow height (`PL_TOP = top - 15`) so its label never sits inside a tall UDL's fill; color changed from amber to violet (`#C792EA` screen / `#7c3aed` print) since amber is now used for tie-down supports *and* the hover glow — legend split into separate "Point Load" / "Tie-down" swatches to match.
- **Number inputs auto-select on focus** app-wide (`document.addEventListener('focus', ..., true)` + deferred `.select()` to survive Chrome's click-collapses-selection quirk) — typing immediately overwrites instead of requiring backspace first.
- **Login screen bug fixed**: `input[type=password]` and `input[type=date]` were never included in either CSS input rule (`input[type=number], input[type=text], select{...}` and the `.license-box`-specific one) — password field rendered as an unstyled browser default, visually mismatched from the email field above it. Both now included in both rules.

### Current Project Milestone — Project-Scoped Multi-Analysis COMPLETE (2026-09-19)

**Status: Development milestone complete — ready for production promotion**

Completed in `development`:
- Project-level shared details: Project Name, Designed By, Checked By, and Date.
- Analysis dropdown with **+ New Analysis**.
- Each analysis independently stores its Analysis Name, Beam / Member ID, beam data, section properties, allowables, supports, point loads, UDLs, and UDL mode.
- Switching analyses preserves shared project information and each analysis's own model data.
- Save/Load/autosave upgraded to version 2 with backward-compatible migration.
- Restore/autosave now captures and restores the shared project details reliably.
- PDF generation includes all analyses in one consolidated report with per-analysis data and page numbering.
- Each analysis has its own two-sheet report structure.
- PDF Section 6 pagination logic and sheet-container handling were corrected during multi-analysis development.
- PDF table headers, title accent, and PDF-only accent color picker are retained.
- PDF header simplified to show the analysis name directly (for example, **Analysis 1**) without the redundant "Analysis:" prefix.
- Full JavaScript syntax check passes after the final development changes.
- `main` has not been modified.

**Final development test notes:**
- The browser/printer may still determine final physical pagination depending on print settings; the working printer-setting adjustment is accepted for this milestone.
- Multi-analysis behavior, Restore/autosave, and PDF generation are ready for promotion from `development`.

### Next Horizon — Production Promotion

Before production promotion, the new **per-opening Disclaimer Acceptance Gate** is now implemented in `development`:
- Authenticated users see the BEAM//FE Important Notice & Disclaimer before entering the app on each page opening.
- The checkbox is reset unchecked each time the gate is shown; Continue remains disabled until actively checked.
- Acceptance is recorded only when **Continue** is clicked.
- Supabase development project contains `public.beamfe_legal_acceptances` with RLS; authenticated users can insert only rows whose `user_id` matches `auth.uid()`.
- Each acceptance records `product_id`, document type/version, SHA-256 hash of the displayed disclaimer text, server timestamp, user-agent, and path.
- License upgrade within the same open session does not unnecessarily show the disclaimer again.
- The previous PDF footer disclaimer remains removed; the PDF remains focused on the calculation report.
- Development DB security verified: `beamfe_legal_acceptances` is RLS-protected and `authenticated` has INSERT-only table privilege; no `anon` privilege is granted.
- **Production deployment note:** the `beamfe_legal_acceptances` table/policy has now been replicated to the production Supabase project. A comparison of BEAM//FE-named tables confirmed the existing license/trial tables already match between development and production; the legal-acceptance table was the missing BEAM//FE table.


Promote the tested `development` branch to production/`main` when ready. Do not make additional changes to `main` as part of this milestone.

---

## PDF Company Branding — COMPLETE

**Status: Implemented in `development`.** This replaces the earlier "planned" roadmap entry of the same name — everything below is built and live, not proposed.

### What's built

- **Company Branding panel** in Model Input, between Project Details and Beam Items: Company Name, Company Address (**3-line textarea**, not a single-line input — prints with line breaks preserved via `white-space:pre-line` on `.pr-brand-address`), Company Description (optional), and a logo picker with live preview + Remove button. Collapsible via a "Minimize"/"Expand" toggle (`#coBrandingToggle` / `#coBrandingBody`); collapsed state remembered in `localStorage` (`beamfe_co_branding_minimized`) as a pure display preference, not synced to Supabase.
- **PDF Accent Color picker** was moved into this same panel (previously lived near the Run Analysis button) and is now persisted alongside branding, not just a local, unsaved `<input type="color">`.
- **Backend: Supabase-persisted, per-user.** No new tables — reuses the existing `user_settings` table (`user_id` PK, `settings` jsonb, RLS already scoped to `auth.uid() = user_id`). Branding lives under a **namespaced key**, `settings.companyBranding = {name, address, description, logoUrl, accentColor}`, so it can't clobber other tools' flat keys already living in the same jsonb blob (e.g. Cuplok's tool settings use unnamespaced top-level keys in the same table). Every write does **read → merge → write**, never a blind overwrite of the whole `settings` object.
- **Logo storage**: existing `company-assets` Storage bucket (public read), new path convention `user-logos/<uid>/logo.<ext>` (separate from the pre-existing `logos/<org-id>.png` convention used by the unrelated organizations/Yard Ops system). Upload → `sb.storage.from('company-assets').upload(...)` → public URL saved to `logoUrl` → fetched once per session and converted to a data URI (`hydrateLogoDataUrl()`) for the live preview and for PDF embedding, same reasoning as the existing chart-canvas-to-dataURL conversion (avoids live-fetch/CORS issues at print time).
- **Load/save lifecycle**: `loadCompanyProfileFromSupabase()` fires once a user is confirmed authenticated+licensed (hooked into `checkLicenseAndRoute`'s success path), not at cold page load. `doSignOut()` clears `companyProfile` back to blank and re-renders the form, so branding doesn't leak to the next person signing in on a shared device.
- **Save status feedback**: `#coSaveStatus` shows "Saving…" / "Saved ✓" / "Save failed: …" / "Not saved — please sign in first." — added because the original auto-save-on-blur behavior gave no visual confirmation either way.

### PDF header/footer layout (multiple iterations, final state below)

- **Single row, not two**: company details + logo are inline with `.pr-titleblock` (same row as "Beam analysis report" + analysis name), not a separate row above it.
- **Order**: company details first (right-aligned text), logo to its right, flush against the page's right margin (`.pr-brand-inline{margin-left:auto}` on the whole group).
- **Logo sizing**: natural aspect ratio — single `max-height` constraint (not a paired max-width+max-height "box" that would letterbox non-matching aspect ratios). `object-fit:contain` kept only as a safety net for the rare case both dimensions get constrained.
  - ⚠️ Real bug hit and fixed here: an earlier version used `max-height:100%` on the `<img>` relying on `align-items:stretch` on the flex row to give it a definite parent height — this caused the whole row to not reach the true right edge (~80% across instead of flush). Root cause: percentage-height on a replaced element inside a stretched flex item is unreliable across rendering engines. Fixed by switching to a **fixed** `max-height` (not a percentage) plus `width:max-content;height:max-content;align-self:center` on the container. If logo sizing bugs resurface, check for percentage-height-of-stretched-flex-parent patterns first.
- **Date moved from footer into the header**, directly under the analysis name: `Beam analysis report / Analysis Name / Date: 19 Sep 2026`. New `.pr-sub-date` CSS rule.
- **Title block top-aligned, not center-aligned**, with the company-info block (`align-items:flex-start` on `.pr-titleblock` and `.pr-brand-inline`, was `center`). Center-alignment made the title look "too low" once company info + the added Date line changed the relative block heights.
- **Footer**: disclaimer text now shows **only on page 2**, not page 1 (`pageFooterHtml(sheetLabel, showDisclaimer)` takes a boolean; page 1 passes `false`, page 2 passes `true`). Page number is the only thing left in `.pr-footer-meta`, centered via `text-align:center` (no longer needs the old 3-column grid trick since Date isn't sharing the row anymore).
  - ⚠️ Real bug found and fixed here: **page 1 never had a footer at all**. The original single-footer markup sat once at the very end of the combined page1+page2 template, and the page-splitting JS (`internalPageBreak.nextSibling` relocation) moved everything after the break — including that one footer — wholesale onto page 2. Fixed by wrapping each page's content in its own `.pr-body` div and giving each page (`pageFooterHtml(page1Label, false)` and `pageFooterHtml(page2Label, true)`) its own footer as a sibling of that page's `.pr-body`, not shared.
- **Page number pinned to the true bottom of the physical page**, not floating after whatever content precedes it. Technique: `.pr-page{height:100vh; display:flex; flex-direction:column}` inside `@media print` (100vh maps to one physical page's content box in a paginated print context), with `.pr-body{flex:1 1 auto}` so it fills available space and pushes the footer down. This applies independently to both the original `.pr-page` div and the JS-created `.pr-page2` div (same class, same rule).
- **Section 5/6 header spacing**: when Code Check (5) and Diagrams (6) both land on page 2, their section-title headers were crowding (`.pr-page.pr-page2 .pr-section-title{margin-top:6px}`). Bumped to `18px` — a real fix, not vestigial.

### Known-unresolved item — do not assume fixed

The `company-assets` Storage bucket's write/update RLS policies are **bucket-wide with no path restriction** (`bucket_id = 'company-assets'`, no owner/folder check) — this predates this session's work and was **found, not created**, while building the per-user logo upload. Practically: any authenticated user across the whole product suite can currently overwrite or delete any file in that bucket, including the real ROI Engineering org logo. Adding a narrower policy scoped to `user-logos/<uid>/` does **not** fix this — Postgres RLS policies are OR'd together, so a stricter policy alongside the existing loose one grants nothing extra; the loose one still wins. Properly closing this requires *replacing* the two existing loose policies, which was deliberately **not done** without first checking what the Yard Ops org-logo upload flow depends on. Flag this before anyone treats the bucket as locked down.

---

## PDF Report Header — Final Pixel Adjustments (2026-09-19)

**Status: Applied in `development`.** These are the current verified pixel/layout values in `index.html` and should be preserved unless the report header is intentionally redesigned.

- **Title block vertical position:** `.pr-titleblock > div:first-child` has **`padding-top: 10px`**. This moves the report title / analysis name / date block down within the header without changing the company branding block.
- **Title-to-divider spacing:** `.pr-titleblock` uses **`padding-bottom: 6px`** before the 2px accent divider. This reduced the previous gap and keeps the logo/header visually closer to the divider.
- **Company details ↔ logo alignment:** `.pr-brand-inline` uses **`align-items: center`**, so the logo is vertically centered against the full company-details block.
- **Logo sizing:** the runtime logo sizing rule uses the rendered company-details height plus **28px**, capped at **58px**:

  `Math.min(info.getBoundingClientRect().height + 28, 58)`

  The logo keeps its natural aspect ratio; do not reintroduce percentage-based `max-height` sizing or a paired fixed width/height box, as those caused earlier alignment/rendering issues.
- **Current header structure:** report title/date on the left; company details followed by logo on the right, all on one header row.

**Verified directly from `development/index.html` on 2026-09-19.** `main` was not modified.

---

## Model Summary Section Removed / Loadings Table Restructured

**Status: Complete in `development`.**

- **Old PDF Section 2 ("Model Summary") removed entirely** — its four columns (Span length, Supports, Loads combined-case count, Tributary width) were mostly redundant: span length is already labeled on the schematic's own dimension line, Supports duplicates the more complete Section 5 (now 4) Reactions table, and the Loads count column was a vague summary of what Section 4 (now 3) Loadings already lists in full detail per-row.
- **Sections renumbered 1–6** with no gaps: 1 Beam schematic, 2 Section properties, 3 Loadings, 4 Reactions, 5 Code check summary, 6 Shear/moment/deflection diagrams.
- **Distributed Load table columns changed**: dropped **Type** (Area/Line), added **Trib. Width (m)** as the last column instead — showing the actual global tributary width for Area-mode rows, or `1.00` for Line-mode rows (dimensionally correct: a Line load in kN/m is the same as an Area load in kN/m² with tributary width exactly 1.0). This makes tributary width visible per-row directly in the table that needs it, replacing what the deleted Model Summary section used to show as one global value.
- **Profile column kept** (Uniform/Triangular/Trapezoidal) — reviewed and judged genuinely useful (at-a-glance shape classification) even though it's derivable from the two magnitude columns, unlike the deleted Model Summary columns which duplicated *more detailed* data shown elsewhere.
- Dead code cleaned up: `supSummary`/`loadSummary` JS variables and the `.pr-model-summary` CSS block, which only existed to feed the removed section.
- **PDF Loadings tables now hide entirely when empty** instead of showing a "None" placeholder row — Point Load table only renders if `R.pointLoads.length`, Distributed Load table only if `R.udls.length`. If a model somehow has neither, a single "No loads defined for this analysis." note shows instead of two empty-looking tables.

---

## Concrete Pour Pressure Calculator — NEW

**Status: Complete in `development`.** New quick-add tool on the schematic toolbar, "+ Concrete Pressure" button right after "+ Distributed Load" (`openConcretePressureCalc()`).

### What it does

Builds a formwork concrete-pressure UDL (or pair of UDLs) from four inputs: **Unit Wt (kN/m³, default 25)**, **Pour Height (m)**, **Pmax (kN/m²)**, **Start (m)**.

- **Pmax is a direct user input, deliberately not computed by the tool.** The full ACI 347 rate-of-placement/temperature formula (which is what actually determines a real Pmax) was researched mid-session but the SI/metric version came back with conflicting coefficients across sources (a 7.2 vs 2.7 discrepancy on what should be the same constant) — rather than guess at a safety-critical formula, the decision was to require Pmax as an input the user has already calculated by their own trusted method, and have the tool handle only the resulting load-shape geometry.
- **Load shape**: flat plateau at Pmax starting from `Start`, tapering linearly down to 0 over the final `h1 = Pmax / Unit Wt` metres, ending at `End = Start + Pour Height`. (Direction was inverted once during development — originally 0-at-Start ramping up, now Pmax-at-Start ramping down, per explicit correction.) Built as **two Area-mode UDLs** (plateau + ramp) when a plateau exists, or **one triangular UDL** when Pmax exactly equals the full uncapped hydrostatic value (`Unit Wt × Pour Height`) — no plateau in that case, handled by the same formula degenerating cleanly (`k = End − h1` collapses to `Start`).
- **Validation**: Pmax cannot exceed the full hydrostatic pressure (`Unit Wt × Pour Height`) — rejected as physically invalid, not silently clamped, since pressure is never greater than that per the source material found.
- **Beam-length handling**: if the pour zone would extend past the beam's end, the load is **clipped and tapered at the beam's end** (not blocked) — the magnitude at the clip point is the correctly interpolated value from the original ramp, not just chopped flat. Only genuinely blocks Add when `Start` itself is at or past the beam's end (nothing at all would fit). Three clip cases handled in `computeLoadRows()`: clip lands in the ramp (plateau stays full, ramp shortens + tapers), clip lands in the plateau (ramp doesn't fit at all, single flat row), or no clipping needed.
- **Follows the app's existing Line/Area UDL toggle** (`mode: udlModeGlobal`), not hardcoded to Area — matches how every other UDL-creation path in the app already behaves. The popup shows a "Created as: Line (kN/m) / Area (kN/m² × trib.)" note (same pattern as the manual UDL editor) so it's clear which mode is active. Note: since Unit Wt × Pour Height is physically an area-based pressure, creating one of these in Line mode applies that number directly as kN/m with **no** tributary-width multiplication — that's on the user to account for, the tool doesn't second-guess the toggle.

---

## Model Input Panel Reorganized

**Status: Complete in `development`.**

- **"Run Analysis" button removed entirely**, along with its listener and the now-unused `.run-btn` CSS. It was redundant — the app already auto-runs analysis on essentially every input change (beam length, supports, loads, section properties, etc. all have their own `change`-triggered `runAnalysis()` calls).
- **Generate Report (PDF), Save Model, Load Model** moved to the very top of the Model Input panel, above Project Details, in a single 3-column row (`.row3`, already existed in CSS). Previously these lived at the bottom of the panel, with Save/Load as a pair and Report as its own full-width button above them.

---

## Primary Beam — Equipment Leg Capacity Check (2026-09-19)

**Status: Implemented in `development`.**

### What was added

- New per-analysis **Beam Classification** control:
  - **Primary Beam** toggle.
  - When enabled, the user can enter **Proprietary Equipment Leg Capacity (kN)**.
- The setting is stored with each analysis and survives:
  - analysis switching,
  - Save/Load,
  - autosave,
  - legacy model migration (older models default to Secondary Beam / off).
- When Primary Beam is enabled, the Reactions panel adds:
  - **Leg Capacity**
  - **Utilization**
  - per-support **OK — LEG CAPACITY** / **FAIL — LEG CAPACITY** status.
- The comparison is direct:
  - bearing reaction = compression load transferred into the equipment leg;
  - utilization = bearing reaction / proprietary leg capacity;
  - **≤ 100% = OK**, **> 100% = FAIL**.
- If Primary Beam is enabled but no leg capacity is entered, the reaction status reports **LEG CAPACITY NOT ENTERED** rather than assuming a capacity.
- Uplift reactions are **not** compared against the compression leg capacity. Existing Free/Tie-down uplift handling remains unchanged.
- The PDF **Section 4 — Reactions** uses the same Primary Beam / equipment-leg comparison and records the entered proprietary leg capacity.

### Engineering intent

The Primary Beam classification explicitly identifies the load path assumed by this check: the beam support reaction is treated as the load transferred directly into the proprietary equipment leg.

The leg capacity is a **user/manufacturer/proprietary input**. BEAM//FE does not calculate or derive that equipment capacity.

This check is intentionally a direct reaction-versus-capacity comparison and should not be interpreted as a new FE support condition or as a calculation of the proprietary equipment's internal leg resistance.

**Development only — `main` was not modified.**


---

## Still Open

- **Company Branding backend scope decision** (see the original roadmap's point 5) has been effectively answered by implementation: it's **per-user** (`user_settings`, keyed to `auth.uid()`), not global/fixed. Confirmed and built, not still pending.
- **`company-assets` bucket RLS gap** — see "Known-unresolved item" above. Not fixed.
- **BEAM//FE SSO bootstrap** (top of this document) — still not implemented, unrelated to any of this session's work.
- Multi-analysis milestone and Disclaimer Acceptance Gate remain ready for production promotion as previously noted; nothing in this session's work has been promoted to `main`.
