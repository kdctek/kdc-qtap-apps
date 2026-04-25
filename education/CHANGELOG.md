# qTap Education — Changelog

All notable changes to this plugin will be documented here.

## [1.0.29] — 2026-04-25

### Added
- **OU path validation at Test Connection.** The Test Connection button now also probes `GET /admin/directory/v1/customer/my_customer/orgunits/<path>` to confirm the configured OU actually exists before save. If the OU is missing, the panel shows `⚠ OU check failed: OU path "..." does not exist...` with a hint about including parent OUs (e.g. `/Indian Education Revival Trust (IERT)/Tridha Parents Group`). When the OU check fails, `verified` stays `false` so the Create form's checkbox stays gated. Catches the most common Workspace setup mistake before the first student is created.
- **Public `verify_ou_exists()`** method on the GWS adapter so REST handlers, job handlers, or future user-edit flows can validate an OU path without re-running the full Test Connection.
- **`SCOPE_ORGUNIT_RO` constant** for future read-only OU listing (the dropdown picker is the planned follow-up).

### Changed
- **Auto-prefix `/` on OU path save.** If an admin enters `Tridha Parents Group` (without leading slash, as the form lets them), the save handler now silently normalizes it to `/Tridha Parents Group`. Empty stays `/`. Trailing slashes are trimmed. Same normalization is applied in the REST test-connection handler so the live test runs against the canonical form.
- **Path-segment URL encoding** in the OU verify call: each segment is `rawurlencode`'d individually so spaces and parens (e.g. `Indian Education Revival Trust (IERT)`) survive, but slashes between segments stay unencoded (Google's API uses them as URL path separators).

### Note
- The auto-prefix only fixes the **leading slash** mistake. The other common mistake — missing **parent OU** in the path (e.g. entering `/Tridha Parents Group` when the OU is actually nested under `/Indian Education Revival Trust (IERT)`) — is now caught by the new OU-exists check at Test Connection time, with a specific error message pointing the admin to the full path. The OU dropdown picker (deferred to a future release) will eliminate both classes of error entirely.

## [1.0.28] — 2026-04-25

### Added
- **Google Workspace lifecycle controls on the WP user-edit screen.** New "Google Workspace" section appears on user-edit / profile screens for any user with a `kdc_qtap_education_workspace_status` value. State-aware UI:
  - **`created`**: shows status header (Workspace user ID, created-at) plus a dropdown — *No change* (default) / *Activate* / *Suspend* / *Delete*. Save runs the corresponding Admin SDK call inline; admin gets a one-shot `admin_notices` flash with the result. Delete uses a JS `confirm()` because Google has no soft-delete.
  - **`pending` / `processing`**: shows the status; no actions (job is in flight or about to be).
  - **`failed:<code>`**: shows status + the verbatim error message. If the seed password is still on file, also shows a "Retry Workspace creation on save" checkbox that re-schedules the creation job.
  - **No status meta at all**: section is omitted (keeps user-edit tidy for non-Education users).
- Three new methods on `KDC_qTap_Education_Google_Workspace`: `activate_user($id)`, `suspend_user($id)`, `delete_user($id)`. Activate/Suspend `PATCH /admin/directory/v1/users/<id>` with `{ suspended: false|true }`; Delete `DELETE /users/<id>`.

### Changed
- On Activate/Suspend, a new `kdc_qtap_education_workspace_suspended` user_meta is set (1 = suspended, 0 = active) so reports/filters can quickly find suspended accounts without round-tripping to Google.
- On Delete, `workspace_status` is set to `deleted` and the Workspace user ID / created-at / error metas are wiped. Status survives so admins can see the account was once provisioned.

### Notes
- Lifecycle actions run inline (not via the cron-scheduled job) because the admin is staring at the screen and expects immediate feedback. Failures surface in the admin notice.
- Delete is **irreversible** — Google's API doesn't expose a soft-delete. The JS confirm() prompt is the only guardrail; consider a stronger confirmation pattern (type DELETE) in a later release if accidental deletions become an issue.

## [1.0.27] — 2026-04-25

### Added
- **Google Workspace user provisioning.** When a school configures the integration in *qTap Education > Settings > Google Workspace*, ticking "Create Google Account" on the Create Student form now actually creates a user in the school's Workspace via the Admin SDK. The flow:
  1. Staff submits the form. WP user is created (fast); a background job is scheduled and the response returns immediately.
  2. ~15 seconds later (or on the next admin page load if WP-Cron is sleepy), the job runs. Hand-rolled service-account JWT auth (RS256 via `openssl_sign`) exchanges for an OAuth access token, then `POST /admin/directory/v1/users` creates the Workspace user in the configured OU with `changePasswordAtNextLogin: true`.
  3. On success: user_meta `kdc_qtap_education_workspace_status=created`, plus the welcome notification fires (always) and credentials notification fires (only if "Send login to Contacts" was ticked).
  4. On failure: status is set to `failed:<code>` with the verbatim error message; staff can retry from the user-edit screen (next release).
- **Settings page with verbose setup guide.** New `qTap Education > Settings` page has tabs (General + Google Workspace). The Workspace tab includes a master toggle + reveal pattern, fields for domain / super-admin email / service-account JSON upload / OU path / force-password-change toggle, and a step-by-step setup guide with direct links to Google Cloud Console (Admin SDK enablement, Service Account creation) and Workspace Admin (Domain-wide Delegation page). Test Connection button probes the credentials inline before save.
- **Notification types registered with parent.** Two new types via `kdc_qtap_register_notification_type()`:
  - `qtap_education_welcome_new_user` — sent to all contacts after Workspace creation succeeds. Default channels: email + WhatsApp. Variables: `{{site_name}}`, `{{user_name}}`, `{{contact_name}}`, `{{login_url}}`, `{{app_url}}`.
  - `qtap_education_login_credentials` — sent to all contacts when "Send login to Contacts" is ticked, **only after Workspace creation succeeds**. Variables include `{{login}}`, `{{email}}`, `{{password}}`. Templates editable in *qTap > Notifications*.
- **AES-256-GCM encryption helper** (`KDC_qTap_Education_Secret`) for the service-account JSON at rest, keyed off WP `AUTH_KEY`. The encrypted blob is stored in the `kdc_qtap_education_settings` option; decrypted only at API-call time.
- **`kdc-qtap-frontend-helpers` viewScript dep** (carried over from v1.0.26) — already explicit in the block registration.

### Changed
- **"Create Google Account" toggle is gated.** When the Workspace integration is not enabled+verified in settings, the toggle is hidden and replaced with a one-line note pointing the staff (admin: with link, non-admin: with message) to *Settings > Google Workspace*. Prevents collecting opt-in for an integration that won't fire.
- **`send_login_to_contacts` timing.** When Workspace integration is active, the credentials notification now waits until Workspace creation succeeds (so the credentials emailed actually work). When not active, falls back to inline dispatch as before.
- **Welcome + credentials notifications now use parent's framework.** Templates are admin-editable in *qTap > Notifications* and respect channel-enabled flags.

### Fixed
- **`$login_result` undefined warning** in the create-student response when `username_override` was supplied (introduced in v1.0.22). Now uses `isset()` guard. The bug was harmless in practice (the override path returns null for `username_taken_suffix` anyway) but PHP would emit a notice.

### Architecture notes
- Used `wp_schedule_single_event` for the Workspace creation job rather than parent's `kdc_qtap_jobs` table because (a) the job is per-user, not per-batch — natural status home is user_meta; (b) parent's job system is shaped for import/export with a results array, awkward fit for an API call.
- Hand-rolled JWT signing avoids a Composer dep on `google/apiclient`. ~30 lines using `openssl_sign(OPENSSL_ALGO_SHA256)`. Token cached in WP transients keyed by (client_email, sub, scope) until expiry minus 60s.
- Service account requires domain-wide delegation in Workspace Admin → API Controls → Domain-wide delegation, with scope `https://www.googleapis.com/auth/admin.directory.user`. Setup guide in the settings page walks through this.
- The `KDC_qTap_Education_Secret` helper is a candidate for promotion to parent if a second qTap plugin needs encrypted option storage.

## [1.0.26] — 2026-04-25

### Added
- **Full-page transaction loader on Create Student submit.** Wired the parent qTap App's `KdcQtapUI.showPageLoader()` (introduced in `kdc-qtap` v2.7.6+, see `kdc-qtap/docs/CHILD-PLUGIN_PAGE-LOADER.md`) around the only server-mutating transaction in this plugin: the Create Student form POST. While the request is in flight the user sees a centered overlay with "Creating student…", switching to "Finalizing…" once the server returns and the success card is rendering. Hidden in `.finally()` so the overlay can never get stuck. Routine fetches (search, slabs refetch, username availability check, associated-user autocomplete) intentionally do NOT use the page loader — they have inline status indicators per the parent's "When NOT to use it" guidance.
- **Explicit `kdc-qtap-frontend-helpers` dependency on the viewScript.** Pre-registered the dashboard block's viewScript handle with the helpers script as a hard dependency so `window.KdcQtapUI` is guaranteed to be defined before `view.js` runs, regardless of theme enqueue order.

## [1.0.25] — 2026-04-25

### Changed
- **Use parent's `--kdc-qtap-radius-md` directly instead of a custom variable.** v1.0.23 introduced `--qtap-edu-field-radius` to unify field/button radii — but the parent already exposes `--kdc-qtap-radius-md` (the same token its own `kdc-qtap-btn` uses). Dropped the custom var, dropped the `.kdc-qtap-btn` override (parent already sets it correctly), and pointed all 12 field-shaped elements at `var(--kdc-qtap-radius-md)`. One token, parent-controlled, consistent across all qTap dashboards.

## [1.0.24] — 2026-04-25

### Changed
- **Use parent qTap color tokens.** Swept 49 hardcoded color literals (`#2271b1`, `#d63638`, `#1f7a1f`, `#646970`, `#1d2327`, `#d4a017`, `#5d4500`, `#5d2226`) and replaced them with the parent's CSS variables (`--kdc-qtap-color-primary`, `--kdc-qtap-color-error`, `--kdc-qtap-color-success-dark`, `--kdc-qtap-color-text-muted`, `--kdc-qtap-color-input-text`, `--kdc-qtap-color-warning`, `--kdc-qtap-color-warning-dark`, `--kdc-qtap-color-error-dark`). Any future qTap brand-color change in `kdc-qtap` now flows through to this dashboard automatically. Same pattern should be adopted by all qTap-series dashboards (Finance, Mobile, etc.) for visual consistency across the suite.

## [1.0.23] — 2026-04-25

### Changed
- **Unified field + button border-radius.** Inputs/selects were `0.25em`, adjustment-row inputs `0.35em`, contact-add `0.3em`, adjustment-add `0.5em`, parent `kdc-qtap-btn` was a fixed 4px — five different radii on visually-similar elements made the form read as patchwork. Introduced a single CSS variable `--qtap-edu-field-radius: 0.4em` on the dashboard wrapper and pointed every field-shaped element (form inputs, search row, contact row, adjustment row, conflict input, control buttons, contact-add, adjustment-add, contact-remove, adjustment-remove, all `.kdc-qtap-btn` instances) at it. Card-shaped elements (slab cards, pill toggles, notice cards, success card, conflict alert) keep their larger radii intentionally — that's the field/card distinction.

## [1.0.22] — 2026-04-25

### Added
- **Live username availability check.** New `GET /students/check-username?username=...` REST endpoint. The Create form fires it 350ms after the staff stops typing first/last; the preview shows a `✓ Available` / `⚠ Taken` badge next to the username so conflicts are caught before submit, not after.
- **Editable conflict-resolution field.** When the live check finds a conflict, an inline editable input appears inside the preview card with the suggested suffixed alternative (`zuvi.kudmule1`) pre-filled. Staff can pick a different username and click "Check" to re-verify. On submit, `username_override` is sent and the server uses it verbatim (no auto-suffix). When there's no conflict the alert stays hidden and submit goes through unchanged.
- **Server respects `username_override`.** `POST /students` validates the override against the Workspace-safe regex + a fresh `username_exists()` check; returns 409 if the override was claimed between the live check and submit.

### Changed
- **Search row matches Create form field size for real this time.** v1.0.21's override only set `min-height` while Finance's staff-find ships fixed `height: 34px` and `font-size: 13px`. Now the dashboard scope explicitly resets `height: auto` plus matches the create-form padding (`0.45em 0.6em`) and border so the search input/select line up exactly with First Name / Last Name / Gender.
- **Preview labels.** `Username (Google Workspace)` → `Username` (the qualifier was redundant); `Default email` → `Default email (Google Workspace)` (the email IS the Workspace login, so the qualifier moves there).

## [1.0.21] — 2026-04-25

### Changed
- **Search row inputs match the Create form's field size.** Finance's `staff-find__field` + `__input` ship their own (smaller) padding/min-height that didn't line up with our larger Create form on this theme. Added a dashboard-scoped override (1em font / 0.7em 0.9em padding / 2.75em min-height) so the search row's "All fields" select and "Type a name…" input match First Name / Last Name / Gender. Override is scoped to `.qtap-education-dashboard` so Finance's own pages aren't affected.
- **Dashboard's `.kdc-qtap-btn` overrides em-based.** The parent ships fixed-pixel font sizes via CSS variables, which on this theme resolved smaller than the surrounding 1em form labels. Now every `.kdc-qtap-btn` instance inside the dashboard reads at 1em / 2.5em min-height (`--lg`: 2.75em / 0.7-1.25em padding) so all action buttons line up with the form text.

## [1.0.20] — 2026-04-25

### Changed
- **Unified button scale across the dashboard.** The form's Create + Cancel pair was small relative to inputs, the top-bar Find + Create-new-{Student} pair didn't match it, and the dashed "+ Add" buttons (contacts, adjustments, expand/collapse) sat at 0.85em while everything around them was 1em. Now:
  - All `kdc-qtap-btn` instances on the dashboard use the `--lg` modifier (44px min-height, 12/24px padding, larger font) for a single primary scale.
  - Icons in those buttons bumped to 18px to keep the proportion.
  - Dashed action buttons (`__contact-add`, `__adj-add`, `__control-btn`) bumped to 1em / 500-weight and 2.75em min-height so they align with `--lg`.
- Net effect: every action button on the create flow reads at the same visual weight, no more "small in some places, big in others."

## [1.0.19] — 2026-04-25

### Fixed
- **Resting × button contrast.** v1.0.17's hover fix made the hover state look great but the resting state's `opacity: 0.6` + neutral border made the × icon look nearly invisible by comparison. Switched the resting state to a visible red glyph (`color: #d63638`) with a 35%-alpha red border, so the affordance reads as destructive at rest too. Hover behavior unchanged (solid red bg, white glyph). Same fix applied to the adjustment-row × button.

## [1.0.18] — 2026-04-25

### Changed
- **Find + Create new {Student} now match.** They're peer call-to-actions in the same control bar, so both use `kdc-qtap-btn--primary`. Previously Create was `--secondary` (transparent/outline) which read as a different visual weight on the host theme. The form's submit/cancel pair keeps its primary/secondary hierarchy since that's the right pattern *inside* a form.

## [1.0.17] — 2026-04-25

### Changed
- **Contacts repeater is capped at qTap Mobile's per-user max.** Reads the parent option `kdc_qtap_block_default_max_numbers` (1-20, default 5) and disables the "Add another contact" button at the limit. Add button now also shows a `(n / max)` counter so the staff sees where they are. REST endpoint `POST /students` defensively trims any oversized payload before save.
- **Hover contrast bumped on destructive buttons.** The contact-row × and adjustment-row × hover states were a faint pink tint that read as nearly disabled; now solid `#d63638` with white glyph + matching focus state. The "Add another contact" hover also flips to solid blue with white text instead of a tinted background.

## [1.0.16] — 2026-04-25

### Added
- **Live fee-slab amounts + due dates in the Create form.** New `GET /kdc/v1/qtap/education/slabs?year&grade` REST endpoint wraps `KDC_qTap_Finance_Enrollment::get_available_slabs_for_enrollment()` and returns each applicable slab with its amount, due date, and exempt flag. The Create form's slab cards refetch when the year/grade selectors change and now show amount + `Due: <date>` per card. Slabs that don't apply to the chosen grade are hidden and unticked so they aren't submitted.
- **Server-side currency formatting.** Slab amounts are formatted via `kdc_qtap_finance_format_amount()` so currency rendering matches Finance throughout the site (symbol, decimals, locale grouping). The endpoint returns both `amount` (raw) and `amount_formatted` (display string).

### Changed
- **Contact mobiles are normalized to E.164 on save.** If the input already starts with `+` it's stored as full E.164. Otherwise the dial code from the parent's `kdc_qtap_settings[phone][default_country]` is prepended (e.g. `9876543210` → `+919876543210` for `IN`). New private helpers `to_e164()` + `get_default_dial_code()` in the REST controller.
- **Tightened the unified font scale + bumped base size.** The form still felt small on the host theme even after v1.0.15's collapse to 5 sizes; bumped the wrapper to `font-size: 1.125em` so all em-based children scale up uniformly. Also collapsed remaining outliers (1.2 / 1.1 / 1.05 / 0.95 / 0.9em → canonical 1 or 0.85em). The 5-step scale is now the only set in use: `2 / 1.25 / 1 / 0.85 / 0.75em`.

## [1.0.15] — 2026-04-25

### Changed
- **Unified the form's font scale.** The dashboard had grown 8 distinct micro-sizes (0.7 / 0.75 / 0.8 / 0.85 / 0.9 / 0.95 / 1 / 1.5 / 2.5em) which read as visual noise. Collapsed to a 5-step scale used consistently:
  - `2.5em` — hero count number (kept)
  - `1.5em` — dashboard title (kept)
  - `1em` — body: inputs, buttons, primary text, names, descriptions, contact-row inputs, adjustment-row inputs/notes, search empty/status, form status
  - `0.85em` — small: field labels, fieldset legends, helper text, slab descriptions, contact-add / adjustment-add buttons, pill-toggle `--sm`
  - `0.75em` — micro: badge, chevron
- Affected ~12 selectors. Result: form chrome reads as one consistent surface instead of N micro-zoomed regions.

## [1.0.14] — 2026-04-25

### Fixed
- **Slab card text was washed out.** The `.__field span { opacity: 0.7 }` rule cascaded to nested spans inside the slab cards (`.__slab-card__name` etc.), muting the text. Tightened the selector to `.__field > span` (direct child only) so descendant spans keep their own colors. Replaced `opacity` with explicit `color` so semibold-on-light reads cleanly. Slab description colour also bumped slightly darker.

### Changed
- **Google + Send-login + Exempt toggles now use the slab-card visual.** Previous render was a small rounded pill; new render is a rectangular card with a 1.4em check + label, matching `Fee Categories`. Single class change (the base `.__pill-toggle` rules now produce slab-card geometry); the existing `--sm` modifier still produces the small pill used for the Adjustments multi-select chips.

## [1.0.13] — 2026-04-25

### Fixed
- **Search result year was always blank.** `KDC_qTap_Finance_Enrollment::get_current()` returns the enrollment payload but the academic year is the *outer* key in Finance's `kdc_qtap_finance_enrollments` user_meta, not a field inside it. The search endpoint now resolves the year separately via `kdc_qtap_finance_get_current_academic_year()` so the meta line shows `Grade · Division · [Year]` instead of `Grade · Division · []`.

### Changed
- **Adjustments section starts empty.** The default empty row was confusing — looked like data the staff needed to fill. Now the first row appears only when `+ Add Discount / Surcharge` is clicked. Implemented via `<template data-role="adj-template">` so the template content isn't even in the rendered DOM until JS clones it.
- **Required fields** (HTML + REST validation): `Gender`, `Academic Year`, `Grade`. Form uses `required` attributes for native browser feedback; client-side `submitForm()` guards before fetch; REST endpoint returns `400 qtap_edu_missing_gender` / `qtap_edu_missing_enrollment` if either is empty.
- **Mobile + Email inputs upgraded** with `inputmode` + `autocomplete` so mobile keyboards open in the right layout (numeric keypad for tel, email keyboard for email). Mobile also has a `pattern="[0-9+\-\s()]*"` so non-numeric chars get a soft warning on submit.
- **Exempt from fees + Adjustments' "Apply to" slab checkboxes** restyled as pill-toggles to match the Google Workspace toggles + slab cards. No more bare browser checkboxes anywhere in the form. Adjustments uses a `--sm` variant (tighter padding, smaller font) so the chips read as multi-select tags rather than full toggles.

## [1.0.12] — 2026-04-25

### Added
- **Custom search REST endpoint** `GET /kdc/v1/qtap/education/students/search?q=…&field=…&roles=…` that joins the user record with the current-year Finance enrollment in one round-trip. Returns `{users: [{id, name, first_name, last_name, email, login, year, grade, division, edit_url}]}`. Replaces the core `/wp/v2/users` call so each search result card can show enrollment context without an N+1 fetch from the client.
- **Search result card meta line** — `Grade · Division · [Year]` (only the bits that are populated). Mirrors Finance staff-console's `__meta` line shape.
- **Live debounced search restored** (350 ms after the last keystroke) on top of the explicit Find button. Switching the field selector with a query already in the box re-fires the search.
- **Adjustments section** in the Enrollment fieldset — repeatable rows for Discounts / Surcharges. Each row: type (Discount / Surcharge), label, mode (Fixed / %), amount, optional notes, and optional per-slab targeting checkboxes (leave all unchecked → applies to every slab). The whole `adjustments[]` array passes through to `KDC_qTap_Finance_Enrollment::save()`, which is Finance's canonical handler for these values, so no parallel logic.
- **`Exempt` badge on Fee Slab cards** when the slab is marked exempt in `kdc_qtap_finance_settings.fees_slabs.{slug}.exempt`.

### Changed
- **Fee slabs are checked by default** — most enrollments use the full slab set, so default-on saves clicks. Staff still uncheck any that don't apply.
- **Send login to Contacts auto-disables when Create Google Account is off.** No Workspace account = nothing to send. Toggle uncheckes itself + becomes greyed-out + non-clickable until the staff re-enables Create Google Account.
- **Payment Cycle placeholder**: `— auto —` → `-- Default --` to match the user's preferred phrasing.

## [1.0.11] — 2026-04-25

### Added
- **Search results + Associated User picker** now render with Finance's `.kdc-qtap-finance-student-card` markup (responsive grid, name + email, hover lift) so they look identical to the Finance staff-console search.
- **Payment Cycle** field in the Enrollment fieldset (Monthly / Quarterly / Half-Yearly / Full Year / Full Cycle / Full Tenure — sourced from `KDC_qTap_Finance_Fee_Matrix::COLLECTION_MODES`). Empty value falls through to Finance's auto-resolution.
- **Two Google Workspace toggles** under the username/email preview:
  - **Create Google Account** *(default ON)* — controls whether the REST response surfaces the seed password. Off = local-only WP user, no password returned.
  - **Send login to Contacts** *(default OFF)* — when on, after creating the user the REST endpoint dispatches the credentials to each contact's email/WhatsApp via the parent's `kdc_qtap_send_notification()` framework.
- **Slab cards (Fee Categories)** — replaced the row of bare browser checkboxes with a responsive grid of pill-cards. Hidden checkbox + state-aware `__check` indicator; descriptions surface inline if Finance settings provide one.
- **Server-side validation: at least one contact required** (with name, mobile, or email). Mirrored client-side in `submitForm()` so the staff sees the error before hitting the network.

### Changed
- **Custom checkboxes throughout the form.** Browser-default checkboxes looked unfit beside Finance's UI; replaced with a hidden native input + painted `__check` indicator (Lucide check on tick). Affects the Exempt-from-fees toggle, the two Google Workspace toggles, and the Fee Categories cards.
- **Bigger border-radii across the form** for a softer, more current look — buttons, inputs, selects, fieldsets, success cards, slab cards all bumped to `0.5em` / `0.6em` / `0.7em` from `0.25em` / `0.3em`.
- **Button classes use parent's `.kdc-qtap-btn` directly** instead of `kdc_qtap_get_button_class()`. The theme-detecting helper was returning a bare `.button` class on Tridha that inherited zero color from the active Beaver Builder theme. Parent's qTap button family is unconditional and CSS-loaded via `kdc_qtap_enqueue_frontend_components()`, so styling is now guaranteed regardless of theme.
- **Associated User label**: "(parent / guardian)" → "(Sibling)" — matches the more common case for new student creation (parent connection is via the `_parent` email suffix; the explicit picker is for sibling / family linkage).
- **Live preview area** now also houses the two Google Workspace toggles, framed with a dashed separator so the relationship between the generated username/email and the Workspace intent is visible at a glance.

### Fixed
- v1.0.10 follow-up: search results container now reliably visible because `setStatus()` toggles `.is-info` / `.is-success` / `.is-error` (Finance's CSS hides the bare container).

## [1.0.10] — 2026-04-25

### Fixed
- **Search results were rendering into a hidden container.** Finance's `.kdc-qtap-finance-staff-find-result` is `display: none` by default and only becomes visible when one of `.is-success` / `.is-info` / `.is-error` is also set on it. v1.0.9 set `innerHTML` but not the state class, so clicking **Find** appeared to do nothing — the markup was correct, the result was just hidden under the cushion. Now `setStatus( html, state )` toggles the state class together with content and clears it on empty queries. Also added a small CSS override to suppress Finance's coloured tint when the container holds our search list (the green/blue background would clash with the rendered list of users).

## [1.0.9] — 2026-04-25

### Changed
- **Adopted the qTap-series UI conventions across the dashboard** so the Education block looks like the Finance staff console rather than a one-off. The render callback now enqueues `kdc_qtap_enqueue_frontend_components()` (parent's button/input framework) plus Finance's `kdc-qtap-finance-staff-console` stylesheet, and the markup uses the same primitives:
  - Top-level **Count** / **{Student}** tabs render as `.kdc-qtap-finance-staff-menu` with Lucide icons (`bar-chart-3`, `user`) via the parent's `kdc_qtap_lucide()` helper. Active state is mapped from our hidden radio's `:checked` to the same visual treatment Finance uses for `.is-active`.
  - Header uses Finance's `.kdc-qtap-finance-staff-dashboard__header` pattern (title + tagline left).
  - **{Student} → Find** is now the Finance `.kdc-qtap-finance-staff-find` pattern: field selector (All fields / Name / Email) + input + explicit **Find** button. Search is **click-to-find** (matches Finance's Receipts UX), no longer live-debounced.
  - Form Submit / Cancel / Find / "+ Create new {Student}" buttons all resolve through `kdc_qtap_get_button_class('primary' | 'secondary')` so they pick up the active theme's button style (WooCommerce, Block theme, or qTap custom UI mode).

### Added
- **Username = `first.last`** (lowercased, Workspace-sanitized — diacritics stripped, special chars removed; first char forced to a letter; numeric suffix only on collision, with a server-side warning).
- **Email = `<username>_parent@<site_host>`** (e.g. `zuvi.kudmule_parent@tridha.edu.in`). The `_parent` suffix flags the mailbox as the parent/guardian contact since students don't have their own email yet.
- **Random 20-char password** generated and **returned in the REST response** so staff can copy it once into Google Workspace setup. Site auth itself uses Google Login or WhatsApp OTP — the password is purely a Workspace seed and is never re-surfaced.
- **Live username + email + warnings preview** under the name fields. As the staff types First / Last name, the UI shows the exact `first.last` and `first.last_parent@…` strings that the server would generate, plus inline warnings if the input contains spaces or characters Google Workspace will reject. Mirrors the server-side sanitization rules (NFD-strip diacritics + lowercase + `[^a-z0-9.\-]` strip).
- **Success card** after create: shows the new user's name + edit-link, the assigned login + email, the generated password with a one-click **Copy** button, and any server-side warnings (e.g. "first.last was already taken; first.last2 was assigned instead").

## [1.0.8] — 2026-04-25

### Added
- **Two-level tabs.** Top-level **Count** | **{Student}** sections. The whole previous block (Grade / Division / Exempt / Unassigned) now lives inside Count. Pure-CSS toggling (radio + `:checked`) at both levels — distinct class prefixes (`__section-*` outer, `__tab-*` inner) so the two layers don't fight each other.
- **{Student} tab — Search.** Live debounced search field (300 ms) calling `/wp/v2/users?search=…&roles=kdc_qtap_student,kdc_qtap_parent,subscriber&context=edit`. Each result links to `wp-admin/user-edit.php?user_id=…` in a new tab. Min query length 2 chars; results show last/first name, email, roles, and the user ID. Uses the `wp_rest` nonce so any user with a REST-API role configured in qTap App Settings can search.
- **{Student} tab — Inline create form.** Toggleable (no page reload). Captures First Name, Last Name, Gender (from Finance `gender_options`), Date of Birth, an Associated User (parent/guardian, picked via the same kind of debounced search), a **repeatable Contact list** (Name / Mobile / Email — kdc-qtap-mobile's `kdc_qtap_mobile_numbers` shape), and an **Enrollment** block (Academic Year, Grade, Division, Fee Slabs multi-select, Exempt flag) — all sourced from Finance settings. Submit goes to a new REST endpoint and reports success/failure inline; the "+ Create new" button toggles it open/closed without nuking page state.
- **REST endpoint** `POST /wp-json/kdc/v1/qtap/education/students`, gated by `kdc_qtap_rest_permission_check()` (parent's REST role config). Creates a WP user with role `kdc_qtap_student`, sets `first_name`, `last_name`, `display_name`, `kdc_qtap_finance_gender`, `kdc_qtap_finance_dob`, `kdc_qtap_mobile_numbers`, runs Finance's `sync_associated_users()` for bidirectional parent/student linking when an associated user is provided, and calls `KDC_qTap_Finance_Enrollment::save()` so the enrollment row is written via Finance's canonical helper (which auto-assigns applicable fee slabs when the form leaves the slab boxes unchecked). Returns `{user_id, login, name, edit_url, enrolled}`.
- New class `KDC_qTap_Education_REST_API` wired through `load_dependencies()` + `init_components()`. Single endpoint today, room to grow into a `qtap-education` Abilities API surface later — same auth path the planned parent mobile app will use.

## [1.0.7] — 2026-04-25

### Changed
- **Replaced the Unicode chevron `▶` with an inline Lucide `chevron-right` SVG** to comply with the project's frontend icon policy (Lucide on frontend, Dashicons in WP admin, no emojis, no `$` / currency glyphs). The SVG inherits `currentColor` and uses the existing `transform: rotate(90deg)` for the expand state, so the visual behaviour is unchanged.
- **Removed the middle-dot (`·`) marker on student rows** in the Unassigned tab; it was decorative only and not a Lucide icon. The existing column indent already conveys the hierarchy. Empty leading cell preserves alignment.

## [1.0.6] — 2026-04-25

### Added
- **4th tab: "Unassigned"** — per-grade list of students whose enrollment has no division. Each row shows last name, first name, and the user_id rendered as a link to `wp-admin/user-edit.php?user_id=X` opened in a new tab. No gender breakdown — this is an operational list, not an aggregate. Uses the same expand/collapse + Expand-all/Collapse-all controls as the other tabs.
- **Finance custom labels respected throughout the dashboard.** Tab labels, column headers, summary band, and empty-state copy now resolve through `KDC_qTap_Education_Dashboard::label()` → `kdc_qtap_finance_label()` so an institution that calls grades "Levels" and divisions "Sections" sees those words automatically. Falls back to English defaults if Finance's helper isn't loaded.
- `KDC_qTap_Education_Dashboard::build_unassigned_view()` data builder + `fetch_user_names()` batched query (one SQL on `wp_users` for `display_name`, one on `wp_usermeta` for `first_name` / `last_name`, no N+1).
- `KDC_qTap_Education_Dashboard::label( $key, $plural )` public helper for any future code that needs Finance-aware labels.

### Changed
- **Renamed the "RTE" tab → "Exempt"** because Finance's `exempt` flag is generic — it could mark RTE seats, scholarship students, staff-family discounts, or anything else exempted from fees. "RTE" was inaccurately specific. Internal view key also renamed from `'rte'` → `'exempt'`. CSS selectors updated.
- View-summary shape: each view now carries a `'type'` field (`'aggregate'` for Grade/Division/Exempt, `'list'` for Unassigned). render.php dispatches to the right panel renderer based on it.

## [1.0.5] — 2026-04-25

### Added
- **Expand-all / Collapse-all controls + per-row toggle** for each tab's table. Outer rows (grades in Grade/RTE tabs, divisions in Division tab) are clickable and now show a small chevron that rotates 90° when expanded. Sub-rows are server-rendered with `is-hidden` so the **default state on first paint is collapsed** — no flash of expanded content before the JS attaches. Keyboard-accessible (Enter / Space toggle, Tab focuses outer rows, role=button + aria-expanded).
- New `blocks/dashboard/view.js` (~80 lines, vanilla, no dependencies) registered as the block's `viewScript` so WordPress only loads it on pages that contain the block.

### Changed
- Outer rows now have hover styling and `cursor: pointer` to invite the click. `user-select: none` so dragging-to-select doesn't fight the toggle.

### Added
- **Three tabs on the dashboard**: **Grade** (default — same view as before), **Division** (inverse grouping — outer = division, sub-rows = grades), and **RTE** (Grade > Division but only counting enrollments where Finance's `exempt` flag is `true`). Each tab gets its own total + per-gender pills + breakdown table; the RTE tab's total reflects only exempt students. Tab switching is pure CSS (radio + `:checked ~ .panel--X`) — no JavaScript runtime, no AJAX, accessible via keyboard, multiple instances of the block on the same page get unique radio names so they don't interfere.
- `KDC_qTap_Education_Dashboard::build_view()` private helper that builds any outer-by-inner view from one enrollment iteration. Accepts an optional filter callable so the same code path serves the Grade, Division, and RTE views (the RTE view passes `fn($e) => !empty($e['exempt'])`).

### Changed
- Output shape of `KDC_qTap_Education_Dashboard::build_summary()` changed: dropped the top-level `total`, `by_gender`, `grades` keys in favour of a single `views` map (`grade`, `division`, `rte`), each with its own `total`, `by_gender`, and `rows` (outer → sub). The `gender_keys` and `year` keys at the top level are unchanged. **Breaking** for any external code that read the old shape — but the dashboard block was the only consumer.

## [1.0.3] — 2026-04-25

### Changed
- **Hide gender columns whose total is 0** across all enrollments. Previously the table always rendered every configured `gender_options` value plus an `Unspecified` bucket, leaving entire columns of zeros (e.g. `Other`, `Unspecified` on Tridha). Now those columns drop out of both the summary pills and the table — the column header is computed once from `summary['by_gender']` totals and pruned before rendering.
- **All font sizes (and proportional spacing) switched from `rem` to `em`.** The dashboard root no longer sets `font-size`, so it inherits whatever size the theme/wrapper provides; every child uses `em` so the whole component scales together. To make the dashboard larger or smaller you can now set `font-size` on a wrapping element — no plugin update needed. Reasonable defaults: title `1.5em`, total value `2.5em`, total label / table headers `0.75em`, pills / table body `1em`. Padding, gaps, border-radii also converted so they remain proportional at any size.
- **Pill colour helpers now also match `--boy` / `--girl`** in addition to `--male` / `--female`, since Tridha (and likely other Indian schools) configures their `gender_options` as Boy/Girl. The colour cue now carries through regardless of label phrasing.

## [1.0.2] — 2026-04-25

### Changed
- **Doubled font sizes** across the dashboard for projection/screen-share readability — title 1.75→3.5rem, meta 1→2rem, total value 2.25→4.5rem, total label 0.85→1.7rem, summary pills 1→2rem (with proportional padding), table 1.05→2.1rem (with proportional cell padding), header row 0.875→1.75rem.

## [1.0.1] — 2026-04-25

### Changed
- **Dashboard rows now follow Finance grade order** instead of alphabetical. Previously `I, II, III, IV, IX, Jr. K.g.` (alphabetical via `ksort`); now `Playgroup, Nursery, Jr. K.g., Sr. K.g., I, II, …, X, AS & A` matching the **Finance > Settings > Grades/Classes** configured order. Same fix applied to divisions (uses `Finance > Settings > Divisions` order); the `Unassigned` bucket always sinks to the end of its grade. Grades or divisions present in enrollment data but missing from Finance settings (e.g. legacy data) fall through alphabetically after the configured ones, so they remain visible.
- **Bumped font sizes** across the dashboard for better readability — title 1.5→1.75rem, total 1.85→2.25rem, summary pills 0.9→1rem, table 0.95→1.05rem with larger cell padding, header row 0.8→0.875rem. Per-cell gender count is now slightly less muted (rgba 0.7→0.85) so zero values still read clearly.

## [1.0.0] — 2026-04-25

### Added
- Initial plugin scaffold.
- Dependency checks for `kdc-qtap` (parent) and `kdc-qtap-finance`.
- Registration with the qTap App dashboard at priority 30.
- Admin submenu placeholder under qTap App ("Education").
- Helper functions: `kdc_qtap_education_is_active()`, `kdc_qtap_education_version()`, `kdc_qtap_education_get_settings()`.
- **Frontend dashboard block** `qtap/education-dashboard`. Server-rendered, gated by `is_user_logged_in()` + `kdc_qtap_can_access_rest_api()` (REST role list managed in qTap App > Settings). Shows total enrollments for the current academic year and a `Year > Grade > Division` breakdown table with per-row gender counts. Gender values come from Finance's `kdc_qtap_finance_gender` user_meta and `gender_options` setting; users with no recorded gender fall into an "Unspecified" bucket. Single batched usermeta query — no AJAX, no caching.
- `KDC_qTap_Education_Dashboard::build_summary( $year = '' )` data builder (defaults to current academic year). Returns total + per-gender + per-grade + per-division counts as a flat array consumable by the block template or future REST/Abilities endpoints.

### Reserved (planned)
- `includes/abilities/` — WP Abilities API surface for the parent mobile app and other connectors. Will live inside this plugin (not a sibling) so the plugin remains self-contained. Ability category: `qtap-education`.
