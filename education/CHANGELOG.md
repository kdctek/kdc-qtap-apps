# qTap Education — Changelog

All notable changes to this plugin will be documented here.

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
