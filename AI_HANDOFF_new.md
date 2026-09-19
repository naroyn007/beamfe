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

### Current Project Milestone — PDF Report + Corporate Styling (2026-09-19)

### Next Horizon — Project-Scoped Multi-Analysis (2026-09-19)

**Status: Development testing**

Implemented in `development`:
- Project Details are now shared across the project: Project Name, Designed By, Checked By, and Date.
- Added an **Analysis** dropdown plus **+ New Analysis** inside the Project Details area.
- Each analysis has its own Analysis Name and Beam / Member ID.
- Each analysis independently retains its beam length, tributary width, section properties, allowables, supports, point loads, UDLs, and UDL mode.
- Switching analyses saves the current model and loads the selected analysis without re-entering shared project details.
- New analyses start as a copy of the current analysis, then receive a new Analysis N / B-N identity for editing.
- Save/Load/autosave format upgraded to version 2 with an `analyses[]` structure and backward-compatible migration from the previous single-analysis JSON format.
- PDF generation now gathers and validates every analysis in the project and compiles them into one consolidated report, with shared project information synchronized across report pages.
- Each analysis retains the existing two-sheet report structure; page numbering is calculated across the complete consolidated report.
- PDF section-property values are bound to the correct analysis rather than the currently visible UI analysis.
- Invalid analyses are reported and excluded from the consolidated PDF instead of inheriting a previous analysis result.
- **PDF pagination regression fixed:** the standalone Analysis Name row was adding vertical height and could cause Section 6 to jump unexpectedly. Analysis Name is now included inside the existing report header, preserving the intended two-sheet layout.
- Full JavaScript syntax check passes after the fix.
- **PDF sheet-structure fix:** each analysis is now split into two real `.pr-page` containers (page 1 and page 2), with explicit page breaks between sheets and between analyses. This prevents the browser from treating an entire analysis as one flowing container and creating an unintended third sheet.
- Follow-up pagination fix: after splitting the sheets into real page containers, the old compact page-2 CSS selectors no longer matched. Restored the compact spacing directly on `.pr-page2` (header, info grid, section spacing, diagrams, captions, and footer) to prevent page-2 overflow.
- **Restore/autosave project details fix:** Project Name, Designed By, Checked By, and Date now trigger autosave directly when edited, so Restore captures the latest shared project details even when no analysis action was run afterward. Restore also explicitly writes/clears those fields from the saved project record.

**Testing focus:**
- Create Analysis 1 and Analysis 2 under the same project; change supports/loads independently and switch back and forth.
- Confirm Project Name / Designed By / Checked By / Date remain unchanged across every analysis.
- Save, reload, and confirm all analyses return correctly.
- Generate a PDF with 2–3 analyses and confirm each analysis has its own Beam / Member ID, model data, results, and diagrams.
- Confirm no third-page behavior is introduced inside an individual analysis's two-sheet layout.

**Status: Development testing**

Completed in `development`:
- UDL top-line drag corrected so it changes only UDL magnitude and does not modify point-load magnitude.
- PDF report refined to target a strict **2-page maximum**.
- Section 6 (Code Check Summary) now moves to page 2 with Section 7 when the model becomes too dense, instead of allowing a third page.
- Page 2 diagrams restored to a larger, more useful size after earlier over-compression.
- PDF table column headers use full-cell maroon fills with white text.
- Model Summary headers are forced to one line, including larger support counts such as 7 supports.
- PDF report title rule uses the same maroon accent.
- Added a **PDF Accent Color** picker on the development UI. It affects PDF export only and automatically selects black/white header text for contrast.
- Normal engineering diagram colors remain separate from the PDF branding accent.

Current behavior / test focus:
- `main` is not being changed as part of this PDF/UI development work.
- Test PDF export with 6–10 supports and multiple point/UDL loadings.
- Confirm Section 6 stays on page 1 for smaller models and moves cleanly to page 2 for dense models, while the complete report remains 2 pages.
- Confirm the selected PDF Accent Color appears as the full table-header fill and title rule in the printed/PDF output.
- Confirm Model Summary column headers remain one line.

Next milestone after this test:
- Finalize the PDF corporate color behavior and promote the tested `development` changes to `main` only after visual verification.

**Repository write status:** GitHub connector writes to `development` are currently working (latest tested commit flow succeeds).
