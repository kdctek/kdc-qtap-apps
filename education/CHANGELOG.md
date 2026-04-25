# qTap Education — Changelog

All notable changes to this plugin will be documented here.

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
