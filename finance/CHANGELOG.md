# Changelog

All notable changes to qTap Finance are documented in this file.

## [3.11.2] - 2026-03-26

### Added
- **Fees tab in Reports** — new last tab showing fee matrix rate card per slab with columns: Grade, Per Month, Per Term, Per Cycle, Per Tenure, Annual Total; one DataTable per slab with collection mode badge and CSV export

## [3.11.1] - 2026-03-26

### Added
- **Per-slab Save button** — each fee slab accordion now has a dedicated Save button that saves all fields for that slab in one AJAX request, bypassing `max_input_vars` limits

### Fixed
- **Fee matrix autosave not working** — fixed missing script dependency (`kdc-qtap-finance-admin`) causing `kdcQtapFinance` to be undefined when autosave fired
- **Autosave UX** — made autosave silent on success (no spinner/checkmark); only shows error indicator on failure

## [3.11.0] - 2026-03-26

### Added
- **Cross-Year Overview report** — new "Overview" tab in Reports aggregates totals across all academic years (standard + multi-year) in one summary table with Group, Year, Students, Total Due, Received, Balance columns; lazy-loaded on first click with CSV export
- **Groups CSV import/export** — dedicated export checkbox, CSV sheet (`qtap_finance_groups`), import target with "Replace all" / "Update existing" options, and sample template
- **Fee matrix autosave on blur** — individual fields save via dedicated AJAX handler (`autosave_fee_field`), bypassing PHP `max_input_vars` limits; ✓/✗ indicators per field
- **Full Cycle collection mode** — new `full_cycle` option for multi-year programs (one billing period per 12-month cycle)
- **max_input_vars detection** — full-form save detects PHP truncation and shows clear error message

### Changed
- **Fee matrix grid transposed** — grades as rows, fee_types as columns (previously fee_types as rows)
- **Fee matrix CSV rewritten** — 12-column format exports all 4 fee_types with `slab_type`, `collection_mode`, `fee_type`, `enabled`; import handles standard/custom slabs and `_show_range` rows
- **Fee matrix JS extracted** — 800 lines moved from inline PHP to `assets/js/kdc-qtap-finance-fee-matrix.js` with `wp_localize_script`; trait reduced from 1889 to 1093 lines
- **Per Tenure disabled by default** — new slabs/years default all fee_types to disabled
- **Beginner's guide v2.1** — added Part 6B: Standard + Multi-Year coexistence (fee matrix, enrollments, reports)

### Fixed
- **Division by zero** — `validate_payment_cycle_for_term()` crashed in PHP 8 for `full_tenure` (period_size=0)
- **Broken custom slab template** — unclosed `</div>` in JS template caused malformed DOM
- **Singular/plural grade count** — "1 grades configured" → "1 grade configured"
- **Dead `curCode` variable** — removed orphaned `kdcFeeMatrixCurrencyCode` reference

## [3.10.12] - 2026-03-26

### Added
- **Full Cycle collection mode** — new `full_cycle` option in Collection Mode dropdown for multi-year programs
- **Fee matrix autosave** — individual fields save on blur via dedicated AJAX handler, bypassing PHP `max_input_vars` limits
- **max_input_vars detection** — full-form save now detects PHP truncation and shows a clear error message

### Changed
- **Fee matrix grid transposed** — grades as rows, fee_types as columns (was: fee_types as rows, grades as columns)
- **Inline JS extracted** — 800 lines moved to `assets/js/kdc-qtap-finance-fee-matrix.js` with `wp_localize_script`; PHP trait reduced from 1889 to 1093 lines
- **Per Tenure no longer enabled by default** — new slabs/years default all fee_types to disabled

### Fixed
- **Division by zero** — `validate_payment_cycle_for_term()` crashed in PHP 8 when `full_tenure` (period_size=0) was passed
- **Broken custom slab template** — unclosed `</div>` in JS template caused malformed DOM when cloning new custom slabs
- **Singular/plural grade count** — "1 grades configured" now correctly shows "1 grade configured"
- **Dead `curCode` variable** — removed orphaned `kdcFeeMatrixCurrencyCode` reference from JS

## [3.10.11] - 2026-03-26

### Changed
- **Fee Matrix CSV export rewritten** — exports all 4 fee_types (per_month, per_term, per_cycle, per_tenure) per grade instead of picking one; new columns: `slab_type`, `collection_mode`, `fee_type`, `enabled`; removed dead `is_na` column
- **Fee Matrix CSV import rewritten** — imports full fee_types structure for standard slabs, flat grades for custom slabs; handles `_show_range` setting rows; proper standard/custom slab distinction via `slab_type` column
- **Fee Matrix sample CSV updated** — new 12-column format with realistic examples showing multiple fee_types, collection modes, custom slabs with due_date, and show_range rows

## [3.10.10] - 2026-03-26

### Added
- **Groups CSV export** — new "Groups" checkbox in export UI with dedicated `qtap_finance_groups` CSV sheet (one row per group item with group metadata repeated)
- **Groups CSV import** — new "Groups" import target with field mapping, "Replace all" and "Update existing" options, two-pass aggregation by group title
- **Groups sample CSV template** — instruction row + sample groups with grade items and group references
- **Groups JSON import** — dedicated `groups` section handler for JSON import (previously only via General Settings)

## [3.9.6]

### Fixed
- **Waterfall allocation uses settings slab order** — partial payment allocation within the same billing period and fee type now follows the fee matrix settings order (drag-and-drop sorted) instead of alphabetical slab slug; also adds `sort_order` as a tiebreaker in the allocator for deterministic ordering

## [3.8.6]

### Fixed
- **Enrollment CSV export missing** — `get_enrollments_csv()` removed redundant `get_user_by()` calls per user (names already in export data), deduplicated mobile number lookups via pre-fetch cache; fixed `selected_slabs` field name to `fee_slabs` (matching actual enrollment meta key); added `payment_cycle` column
- **CSV export error isolation** — each export section (fee matrix, enrollments, payments) now wrapped in try/catch so one section's failure doesn't prevent others from being exported

## [3.8.5]

### Fixed
- **REST API permission fallback too permissive** — `check_mobile_api_permission()` allowed any logged-in user (including subscribers) when parent plugin unavailable; now requires `manage_options` capability
- **Cross-plugin fatal error risk** — added `method_exists()` guard before calling `find_users_by_mobile()` on mobile plugin instance; prevents fatal if older mobile plugin version lacks the method
- **Fee matrix sample CSV format mismatch** — sample template used one-row-per-slab format with `grade_<slug>` columns, but importer expects one-row-per-grade format with `grade` and `amount` fields; rewrote sample to match import field mapping

## [3.8.0]

### Refactored
- **User Meta class** — split monolithic 2,932-line `class-kdc-qtap-finance-user-meta.php` into 5 focused traits: rendering, enrollments, payments, user-fees, notifications
- **User Profile JS** — split monolithic 847-line `kdc-qtap-finance-user-profile.js` into 6 modules: common (shared config/utilities), enrollments, payments, user-fees, refunds, notifications

## [3.7.3]

### Fixed
- **Notification Preview: AJAX variables** — used `ajaxurl`, `nonce`, `userId` (matching all other AJAX calls in the file) instead of non-existent `config.ajaxUrl`/`config.nonce`/`config.userId`; added null guard on error response data

## [3.7.2]

### Fixed
- **Notification Preview: ReferenceError** — `cfg` variable not defined; corrected to `config` in the notification preview AJAX call on user profile page

## [3.6.8]

### Changed
- **Report block: login form** — switched to WordPress native `wp_login_form()` with styled container for reliable rendering regardless of parent plugin state or caching

## [3.6.7]

### Fixed
- **Report block: login form not visible** — parent frontend components CSS/JS not enqueued before login check; now loads `kdc_qtap_enqueue_frontend_components()` first

## [3.6.6]

### Added
- **Report block: login form** — logged-out users see a login form instead of a blank page; uses parent's `kdc_qtap_render_login_required()` with fallback

## [3.6.5]

### Fixed
- **Report XLSX filename** — entire filename is now lowercase

## [3.6.4]

### Fixed
- **Report block: Full Width layout** — `alignfull` now uses `100vw` with proper padding; year selector and controls center-aligned (`max-width: 1200px`); tabs and DataTable stretch full width

## [3.6.3]

### Added
- **Report: Show User ID toggle** — new checkbox to show/hide the User ID column in report tabs and XLSX export (hidden by default)

### Fixed
- **Report block: Full Width** — `alignfull` class now sets `width: 100%` so the report stretches to full viewport width

## [3.6.2]

### Added
- **Report block: Wide & Full Width alignment** — block now supports `wide` and `full` alignment options in the Gutenberg toolbar

## [3.6.1]

### Added
- **Report download notification** — when Excel is downloaded, the XLSX is sent as an email attachment to the downloading user via the parent notification system, with full audit logging
- **Email attachment support** — parent email channel (`kdc-qtap`) now passes `email_attachments` array to `wp_mail()` 5th parameter

## [3.6.0]

### Added
- **Frontend Report Block** (`kdc-qtap/finance-report`) — new Gutenberg block that renders the full finance report on the frontend for authorized users. Includes year selector, term/month breakup, group tabs with DataTables, Only Dues / Show Custom / Show Division toggles, CSV export, and Excel download.
- **REST API permission for reports** — report AJAX endpoint now uses `kdc_qtap_can_access_rest_api()` instead of `manage_options`, allowing any role with REST API access to view reports on the frontend.

## [3.5.39]

### Fixed
- **Frontend: user fees rendered twice** — v3.5.38 accidentally added a duplicate user fees section in `renderTermsFees()` where one already existed; removed the duplicate

## [3.5.38]

### Fixed
- **Frontend: User fees missing in term view** — `renderTermsFees()` (the default render path for term-grouped data) was missing the user fees section entirely; only the legacy `renderFees()` path had it. User fees now render in both paths.

## [3.5.37]

### Fixed
- **Frontend: inactive user fees still showing** — second payment loop in `ajax_get_fees()` was missing the inactive status check, allowing deactivated fees to appear in the User Fees section

## [3.5.36]

### Fixed
- **User fee label** — `get_slab_label()` now resolves `_user_fee_` slugs to their `installment_label` instead of showing the raw slug; fixes display in payment history, notifications, WC orders, and all other call sites

## [3.5.35]

### Added
- **User Fee modal: two sections** — modal now shows "Add New Fee" form at top and "Existing Fees" list at bottom with edit/delete actions
- **User Fee CRUD** — new AJAX handlers for listing (`get_user_fees`), updating (`update_user_fee`), and soft-deleting (`delete_user_fee`) user fees
- **Inactive payment status** — new `inactive` status for soft-deleted fees; record stays in DB for audit trail

### Changed
- **User Fee delete → soft-delete** — deactivating a user fee sets status to `inactive` instead of removing the DB row; adds deactivation note with admin name and date
- **Inactive exclusions** — inactive payments excluded from: frontend fees block, admin reports, payment history table, year summary totals, and WC payability
- **User Fee modal list** — inactive fees shown greyed out with no edit/delete actions; active unpaid fees show edit (pencil) and delete (trash) icons

## [3.5.34]

### Fixed
- **User fee modal icon** — changed from dollar sign (`dashicons-money-alt`) to user icon (`dashicons-admin-users`)
- **User fee button color** — uses WP admin theme color CSS variable
- **Enrollment card button order** — reordered to User Fee > Edit > Delete

## [3.5.33]

### Added
- **Frontend: User Fees section** — third dedicated section in the fees block showing `_user_fee_*` payments with proper fee name labels (not slugs); section titled with dynamic student label
- **Report: User Fees columns** — separate `{Student} Fees Collection` / `{Student} Fees Received` columns in group tabs when Show Custom is ON; excluded from SUMMARY tab entirely
- **Frontend: Grade-Specific Fees label** — custom fees section now uses dynamic grade label

### Changed
- **User fee slug format** — now `_user_fee_{userId}_{unixtime}` for better traceability
- **User fee icon** — changed from coin to user icon (`dashicons-admin-users`)
- **User fee display name** — uses `installment_label` (admin-entered name) instead of raw slab slug
- **Report columns** — `getFilteredColumns()` and `adjustTotals()` now accept `isSummary` parameter for proper column/total scoping

## [3.5.32]

### Fixed
- **Pending verification notice relocated** — admin notice for pending payment verifications was missing `inline` class, causing WordPress core JS to move it to the admin notice area on the Import/Export page

## [3.5.31]

### Added
- **User-specific fee creation** — admin can create one-off fees for individual users from the profile page (Add Fee button next to enrollment actions); stored with `_user_fee_` slab prefix; appears in frontend fees block under "Other"
- **Offline refund recording** — admin can record refunds via Refund button on any payment with amount_paid > 0; creates negative transaction, reduces amount_paid, recalculates status and item allocations; refund reason required for audit trail

### Fixed
- **Duplicate payment prevention** — enrollment cleanup now deletes ALL unpaid payments before regeneration (not just those with `term_key`), preventing doubled amounts when re-importing or re-saving enrollments
- **User-specific fees protected** — payments with `_user_fee_` prefix are never deleted during enrollment regeneration

## [3.5.30]

### Changed
- **Report: XLSX filename reordered** — format is now `{year}[_DUES]_{institute}_finance-report_{date}_{timezone}.xlsx` (e.g., `2025-2026_DUES_School-Name_finance-report_2026-03-17_08-45_Asia-Kolkata.xlsx`)

## [3.5.29]

### Added
- **Report: XLSX filename includes `_DUES` tag** — when "Only Dues" is checked, the downloaded filename includes `_DUES` before the institute name (e.g., `finance-report_2025-2026_DUES_School-Name_...xlsx`)
- **Report: Summary-only groups show bracketed names** — groups with Report Summary ON but Report Tab OFF display as `[Group Title]` in the SUMMARY tab to distinguish them from tab groups

## [3.5.28]

### Added
- **Fee Matrix: slug badge on custom slab title bar** — custom slabs now show `_custom_N` slug in the accordion header, matching regular slabs
- **Fee Matrix: click-to-copy slug** — clicking any slug badge copies it to clipboard with "Copied!" feedback

### Changed
- **Fee Matrix: slug badge placement** — moved custom slab slug from Fee Name input area to title bar only (consistent with regular slabs)

## [3.5.27]

### Added
- **Fee Matrix: slug reference** — subtle `<code>` badge showing the fee slug next to each slab name in the accordion header (regular slabs) and next to the Fee Name input (custom slabs); helps when preparing CSV imports
- **Grouping: Report Summary checkbox** — independent control for summary row inclusion; groups can appear in summary without a tab, or vice versa
- **Fee Matrix: Show Range per-grade toggle** — controls whether `{range}` appears in payment titles for each grade

### Changed
- **Grouping: "Include in Report" renamed to "Report Tab"** — now only controls tab creation
- **Report AJAX** — sends `summary_groups` separately from tab `groups` for independent summary/tab rendering
- **Fee slab names: commas stripped** — commas are silently removed from fee category and custom slab names on save to prevent CSV import ambiguity

## [3.5.26]

### Added
- **Fee Matrix: Show Range per-grade toggle** — new checkbox row below Annual Total in each slab; controls whether `{range}` appears in payment titles for that grade; global per-grade setting synced across all slab cards; default enabled
- **Grouping: Report Summary checkbox** — independent control for summary row inclusion; groups can now appear in summary without having a tab, or vice versa
- **Helper function** `kdc_qtap_finance_get_show_range( $grade )` — returns whether range should be displayed for a grade

### Changed
- **Grouping: "Include in Report" renamed to "Report Tab"** — now only controls tab creation; summary row controlled by new "Report Summary" checkbox
- **Report AJAX handler** — sends `summary_groups` (with rows) separately from `groups` (tab groups); JS re-aggregates summary from summary_groups with toggle support
- **Report XLSX export** — Summary sheet built from summary_groups independently of tab groups
- **Release shortcut** — `release` keyword now triggers the full release sequence

## [3.5.25]

### Fixed
- **Report: Custom fees missing in both breakup modes** — month-wise breakup had no `_custom_c`/`_custom_r` columns and custom payments (which have no payment items) were silently dropped; term-wise was correct but month-wise now also buckets custom fee amounts at the payment level and includes Custom Fees Collection/Received columns

## [3.5.24]

### Fixed
- **Report tab fatal error** — `report_aggregate_rows()` tried to sum the `division` string field, causing `TypeError: Unsupported operand types: string + float` on PHP 8+; added `division` to the skip list

### Changed
- **Reorganized AJAX trait** — moved report AJAX handler and 4 private helpers (`report_build_columns`, `report_resolve_group_items`, `report_build_user_row`, `report_aggregate_rows`) from `trait-kdc-qtap-finance-admin-ajax.php` into `trait-kdc-qtap-finance-admin-tab-report.php` so each tab's rendering and data logic are co-located

## [3.5.23]

### Added
- **Shared currency formatter** — new `kdc-qtap-finance-currency.js` utility exposes `kdcQtapFinanceFmtAmount()` used by all JS scripts; replaces 4 duplicate implementations (`fmtCurrency`, `formatCurrency`, `formatAmount`, inline `toLocaleString`)
- **Report: Show Division toggle** — adds a division column (from enrollment data) next to the student name in group tabs; uses dynamic label from Labels settings; included in CSV and XLSX exports
- **Report: Custom fees total recalculation** — "Show Custom Fees" toggle now recalculates Total Collection, Total Received, Balance, and Excess to include/exclude custom fee amounts
- **Report: XLSX filename format** — `finance-report_{year}_{institute}_{date}_{timezone}.xlsx` using WP General Settings timezone

### Changed
- **JS currency consolidation** — `formatAmount()` (user-profile), `fmtCurrency()` (fee-matrix), `formatCurrency()` (frontend block) all delegate to shared `kdcQtapFinanceFmtAmount()`; removed `kdcFeeMatrixCurrencyCode` inline variable
- **Block AJAX currency response** — added missing `decimals` field to currency array

## [3.5.22]

### Changed
- **Notification amount formatting** — `resolve_amount_due`, `resolve_amount_paid`, `resolve_balance`, `resolve_transaction_amount` now use `kdc_qtap_finance_format_amount()` instead of hardcoded currency symbol and `number_format()`; respects currency settings (symbol, decimals, Indian grouping)
- **REST API & User Meta** — minor cleanup to use consistent helper functions
- **Admin tabs** — enrollment and verify-payments tab refinements

## [3.5.21]

### Fixed
- **Report DataTable not rendering** — removed DataTables Responsive extension which caused `Uncaught DataTables Responsive requires DataTables 2 or newer` console error; replaced with `scrollX: true` for horizontal scrolling on wide tables; removed unused responsive JS/CSS files from bundle

## [3.5.20]

### Fixed
- **CSV import row-level tracking** — all import handlers (fee matrix, enrollments, payments, transactions, enrollment+payments) now return `row_imported`, `row_updated`, `row_skipped`, `row_errors` arrays alongside summary counts; required by parent plugin's job processor to correctly display per-row status in the import report and render the Updated counter in the job detail UI

## [3.5.19]

### Fixed
- **Report tab switching** — `responsive.recalc()` threw an error on hidden panes killing the click handler; now wrapped in try/catch with separate `columns.adjust().draw(false)` call

## [3.5.18]

### Changed
- **Report: "Show Custom Fees" toggle** — renamed from "Show Other"; controls visibility of Custom Fees Collection/Received columns (slabs with `_custom_` prefix)
- **Report: Totals exclude custom fees** — Total Collection, Total Received, Balance, and Excess now reflect only regular term/month fees; custom fee amounts are separate and only visible when the toggle is on
- **Report: Responsive DataTables** — bundled DataTables Responsive 3.0.3 extension; tables now collapse columns on smaller viewports with expand/collapse child rows
- **Report: Reporting Breakup dropdown** — live control on Report tab triggers AJAX reload to switch between term-wise and month-wise column structure

### Added
- **CSV import: `payment_cycle` field** — enrollments import now accepts `payment_cycle` column (monthly, quarterly, half_yearly, full_year) with aliases; reflected in sample CSV template
- **CSV import: Enrollment + Payments combined group** — new import target that creates enrollments and records initial payments in one step; applies `amount_paid` via waterfall allocation across generated payments
- **CSV sample: Dynamic data** — enrollment sample template now uses actual configured academic years, grades, divisions, and slabs instead of hardcoded examples

## [3.5.17]

### Fixed
- **Report tab literal `\t` crash** — removed literal backslash-t characters in AJAX trait (line 500) that caused `Fatal error: Undefined constant "t\t"` on PHP 8+, breaking Preview Installments and frontend fees display
- **Report term column mapping** — payments stored billing period keys (`bp_0`) as `term_key` but report columns used term slugs (`term_abc1`); now resolves via `period_start` month → term slug lookup so term Collection/Received populate correctly

### Changed
- **Report student name format** — changed from "First Last" to "Last First" for consistency with accountant workflows
- **Report tab year selector** — replaced plain dropdown with the standard `.kdc-qtap-finance-year-selector` card (matching Enrollments/Fee Matrix tabs)
- **Reporting Breakup moved to Report tab** — removed from Institute tab; now a live dropdown on the Report tab controls row that triggers AJAX reload
- **SUMMARY tab** — first column header uses Grade label (not Student); sorting disabled to preserve group order

### Added
- **Total Collection / Total Received columns** — aggregate columns added before Balance/Excess in the report
- **Only Dues toggle** — filters all report tabs to show only rows with balance > 0 (client-side, no AJAX)
- **Show Other toggle** — controls visibility of "Other Collection/Received" columns for unassigned payments
- **Download Excel button** — exports all tabs as sheets in a single XLSX file using bundled SheetJS (xlsx-0.20.3)

## [3.5.16]

### Added
- **Finance Report tab** — live, sortable, searchable DataTable report for accountants; tabs per active Group (from Grouping tab with "Include in Report" enabled), SUMMARY tab showing aggregated totals, term-wise or month-wise column breakup (controlled by new "Reporting Breakup" setting in Institute tab), Collection / Received / Balance / Excess columns, CSV download, colour-coded Balance (red) and Excess (amber) cells
- **Reporting Breakup setting** — new select field in Institute tab (`term` or `month`) to control Report column structure; persisted in `kdc_qtap_finance_settings`
- **Bundled DataTables** — DataTables 2.1.8 and Buttons 3.1.2 JS/CSS served locally from `assets/lib/datatables/` instead of CDN for reliability and offline use

## [3.5.15]

### Fixed
- **HPOS compatibility for order trash/restore** — registering `woocommerce_trash_order` and `woocommerce_untrash_order` hooks alongside `wp_trash_post`/`untrashed_post` so trash/restore actions fire on HPOS sites; removed `get_post_type()` check (always false for HPOS order IDs) in favour of `wc_get_order()` validation; HPOS fallback reads `_wp_trash_meta_status` from order meta when post meta is unavailable

## [3.5.14]

### Fixed
- **WC admin Fee Details column** — fixed broken student name display (was reading `user_display_name` meta key which was never written; now reads `student_name`)
- **Consistent student name across all WC paths** — all 6 order/cart creation code paths now use `first_name + last_name` with `display_name` fallback, and `kdc_qtap_finance_label('student')` as the visible meta key
- **Visible order meta on all paths** — Student, Academic Year, Grade now stored as visible (label-keyed) order meta on direct orders, cart-based orders, and block checkout orders
- **Missing order-level meta** — `create_fee_order()` and `create_multi_fee_order()` now set `student_name`, `academic_year`, `grade`, `division` at order level (was only `payment_id` and `is_fee_payment`)
- **Item title showing `bp_0`** — billing period keys now resolve to `installment_label` in both `create_term_order()` and `build_term_cart()`
- Removed dead (unhooked) methods `display_order_item_meta()` and `display_order_item_meta_email()`

## [3.5.13]

### Fixed
- **PDF invoice enrollment details now visible** — reads fee-order meta keys (`student_name`, `academic_year`, `grade`, `division`) with enrollment meta as fallback
- **Visible order meta added** — Student, Academic Year, Grade now stored as visible (non-underscore) order meta using label keys, displayed in WC order admin, emails, and PDF invoices
- Removed payee name hook (`wpo_wcpdf_before_billing_address`)

## [3.5.12]

### Fixed
- **WooCommerce order student name** — uses `first_name + last_name` instead of `display_name` for order meta, product line item meta, and all WC payment flows; falls back to `display_name` if name fields are empty

## [3.5.11]

### Fixed
- **Per-term range month order** — term months now sorted in academic-year order (handles Dec→Jan wrap); fixes reversed labels like "Jan 2026 to Dec 2025" → correct "Dec 2025 to May 2026"

## [3.5.10]

### Changed
- **Fee slabs pre-checked on enrollment** — all available slabs are checked by default when adding or changing grade; admin only needs to untick unwanted slabs

## [3.5.9]

### Fixed
- **Cross year separator spacing** — spaces are now auto-padded around the cross-year separator at render time (`make_range()` wraps with ` {sep} `); stored value is bare text (e.g., `to`, `-`), sanitized with `sanitize_text_field()`

## [3.5.8]

### Changed
- **Date Range Format** — split into 3 fields: Date Format (PHP date format with validation), Same Year Separator (default `-`), Cross Year Separator (default ` - `); unified real-time preview row showing Monthly, Same year, and Cross year examples
- `make_range()` uses all three settings (format + both separators)

## [3.5.7]

### Added
- **Date Range Format setting** — Institute tab field to customize date format using WordPress format characters (e.g., `M Y`, `F Y`, `m/Y`), with inline live preview
- **Payment Title Format setting** — Institute tab field with `{year}`, `{term}`, `{range}` placeholders for customizable payment record naming (default: `[{year}] {term} {range}`), with inline live preview
- `make_range()` now uses the configured date format setting
- Enrollment payment creation uses the configured payment title format
- Generator output includes separate `term_label` and `range` fields for template rendering

## [3.5.6]

### Changed
- **Unified `make_range()` helper** — all date range formatting (billing period labels, per-term item labels, per-year item labels, per-month item labels) now uses a single shared method for consistent output

## [3.5.5]

### Fixed
- **Calendar year mapping for line item labels** — `get_calendar_year_for_month()` now uses the actual year start month from settings instead of hardcoded July; fixes "Jun 2026" showing instead of "Jun 2025" when academic year starts in June

## [3.5.4]

### Fixed
- **Edit enrollment regenerates payment blocks** — changing payment cycle on an existing enrollment now correctly deletes unpaid payments, skips items already covered by paid payments, creates new billing periods for remaining balance only, and redistributes any overpayment credit via waterfall

## [3.5.3]

### Changed
- **Payment label format** — `[{Year}] {Term Name} {Range}`: e.g. `[2025-2026] 1st Term Jun-2025` (monthly), `[2025-2026] 1st Term Jun-Aug 2025` (quarterly), `[2025-2026] 2nd Term Dec 2025 - May 2026` (half yearly cross-year), `[2025-2026] Jun 2025 - May 2026` (yearly)

## [3.5.2]

### Fixed
- **Billing period label format** — clean date-only labels without term name prefix: Monthly = "Jun-2025", Quarterly = "Jun-Aug 2025" or "Dec 2025 - Feb 2026" (cross-year), Half Yearly = "Jun-Nov 2025" or "Dec 2025 - May 2026", Yearly = "Jun 2025 - May 2026"
- Consistent hyphen-separated format across billing period labels, per-month item labels, and per-term/per-year range labels

## [3.5.1]

### Fixed
- **Payment records and Pay button match billing cycle** — each billing period now creates a uniquely-keyed payment record (e.g., "1st Term: Jun 2025" for monthly) instead of all sharing the term name
- Frontend groups payments by billing period key, not academic term — monthly cycle shows 12 cards with individual Pay buttons matching the cycle amount
- Card headers use payment's `installment_label` when billing period doesn't match a term config entry
- Frontend group ordering by due_date ASC instead of term config order (supports non-term-aligned billing periods)

## [3.5.0]

### Changed
- **Granular line-item installments** — installment generator now produces individual per-month, per-term, and per-year items regardless of payment cycle; payment cycle only determines how items are grouped into payment records
- **Waterfall allocation uses fee_type** — partial payment allocation now prioritizes by `fee_type` (per_year → per_term → per_month) instead of slab `collection_mode` lookup
- DB v2.1.0: added `fee_type` column (`per_month`/`per_term`/`per_year`) to `payment_items` table
- `fee_type` included in frontend fees and admin preview AJAX responses

### Fixed
- Academic-year month ordering in sort (handles Dec→Jan calendar year wrap correctly via `order_index`)
- Multiple billing periods per term now create correctly (removed duplicate `term_key` check that blocked quarterly billing with half-yearly terms)

## [3.4.6]

### Fixed
- **Preview Installments not loading slabs or generating** — two bugs: nonce action mismatch (`kdc_qtap_nonce` vs `kdc_qtap_finance_nonce`) caused "Request failed", and slab AJAX response format mismatch (returned `{html}` but JS expected array)
- Added `slabs` array to `ajax_get_available_slabs` response for programmatic consumers

## [3.4.5]

### Fixed
- **Preview Installments button not working** — JS executed before `kdcQtapFinance` was defined (inline IIFE ran before footer scripts); wrapped in `jQuery(document).ready()` to defer execution
- Fixed preview year sourced from non-existent property; now reads from hidden input

## [3.4.4]

### Added
- **Preview Installments** — "Preview Installments" button on Fee Matrix tab opens a modal to simulate enrollment and preview payment installments without creating records
- Modal form: Grade, Payment Cycle, Fee Slabs (auto-loaded per grade), Exempt checkbox
- Output: term-grouped cards with due dates, line items, term totals, and grand total bar
- Custom slabs shown as separate one-time entries; exempt mode zeroes all amounts

## [3.4.3]

### Added
- **Partial payment waterfall allocation** — when a partial lump-sum payment is received, it allocates across fee components in priority order: yearly fees first, then half-yearly, quarterly, and monthly (chronologically)
- **Per-item `amount_paid` tracking** — payment items now track individual paid amounts (DB v2.0.0: `amount_paid` column on `payment_items` table)
- Item-level paid/balance display on frontend fees block (shows "₹X paid" for partially-allocated items)
- Allocation triggers automatically from: admin manual recording, WooCommerce order completion, offline payment verification, transaction deletion/refund

## [3.4.2]

### Added
- **Payment Cycle on user profile** — Add New Enrollment form, Edit Enrollment modal, and enrollment card all show Payment Cycle selector (Default/Monthly/Quarterly/Half Yearly/Yearly)
- `data-payment-cycle` attribute on edit enrollment button for pre-population

## [3.4.1]

### Changed
- **Default admin tab** — Enrollments is now the first/default tab; Fee Matrix moved to second position
- Tab order reorganized for workflow priority

## [3.4.0]

### Added
- **Payment Cycle** — term-level default and enrollment-level override for billing frequency (Monthly/Quarterly/Half Yearly/Yearly)
- **Payment Cycle dropdown on Terms** — General tab term rows now include a Payment Cycle selector with division warning when months don't divide evenly
- **Enrollment Payment Cycle column** — Enrollments tab shows each user's payment cycle (Default or override)
- **Bulk Payment Cycle change** — select multiple enrollments and apply a new payment cycle; confirmation dialog warns about unpaid payment regeneration
- New `generate_billing_period_payments()` in installment generator — unified billing periods across all slabs based on payment cycle
- Helper methods: `get_term_default_payment_cycle()`, `get_effective_payment_cycle()`, `validate_payment_cycle_for_term()`

## [3.3.14]

### Changed
- **Grouping cards CSS** — all colors now use CSS custom properties derived from `--wp-admin-theme-color` via `color-mix()`; adapts to any WordPress admin color scheme
- Grade-division item rows now have themed background with border for visual distinction

## [3.3.13]

### Changed
- **Year config consolidated** — removed duplicated First Month / Start Date from Institute tab; now lives only in General tab → Academic Terms section (saved via AJAX)
- Start Date field added to Academic Terms section alongside Year Starts In dropdown

### Added
- **Academic year validation** — input now validates format: accepts `YYYY` (single year) or `YYYY-YYYY` (consecutive range); auto-expands `YYYY-YY` shorthand; rejects invalid formats (non-numeric, non-consecutive, 5+ digits)
- JS-side validation with inline error messages on add, enter, and paste

## [3.3.12]

### Added
- **Include in Report checkbox** — each group card has a checkbox and header badge icon (green check when included, grey marker when excluded)
- **Multi-select group include** — "Include Group" is now a multi-select; add multiple groups at once with duplicate/self-reference prevention

## [3.3.11]

### Changed
- **Narrow group cards** — Grouping tab cards now display in a responsive CSS grid (`auto-fill, minmax(280px, 1fr)`) instead of full-width rows
- Compact card body with stacked inputs, smaller controls, inline item rows

### Fixed
- **Grade labels on mobile fee totals** — `data-grade` attributes added to all JS-generated grade totals `<td>` elements; CSS `::before` pseudo-elements now show grade names on mobile

## [3.3.10]

### Added
- **Expand/Collapse All** buttons on Grouping tab
- **Sortable group cards** — drag to reorder, form indices re-indexed on drop to persist order

## [3.3.9]

### Added
- **Group nesting** — groups can include other existing groups as members alongside grade-division pairs; self-reference and duplicate prevention built in
- **Release shortcut** — documented in CLAUDE.md for "version bump, git commit, push origin, zip update" workflow

## [3.3.8]

### Added
- **Grouping tab** — create named groups of grade-division pairs for reporting and data selection; accordion cards with add/remove UI
- **Drag-and-drop sorting** — all multi-select fields (Grades, Divisions, Fee Categories, Academic Years) are now sortable via drag; order persists across fee matrix, enrollments, and dropdowns

## [3.3.7]

### Changed
- **Responsive fee matrix** — on mobile, tables flip to grade-first card layout with 2-column grid instead of horizontal scroll; grade totals bar wraps into tag layout

## [3.3.6]

### Added
- **First Month & Start Date settings** — per-year configuration on Institute tab for academic year start month (Jan-Dec, default April) and official start date
- **Lock/unlock editing** — once set, First Month and Start Date are locked; unlock requires confirmation warning about impact on current enrollments

## [3.3.5]

### Added
- **Admin enrollment delete** — single and bulk delete on Enrollments tab with confirmation alert; cascades to associated payments and transactions

## [3.3.4]

### Changed
- **Order meta labels use finance settings** — Grade meta key is always the grade label only; value shows `grade division` (space-separated, no hyphens)
- **Student name** uses `first_name last_name` consistently
- **Enrollment meta stored as internal** — hidden from WC order display, only rendered via PDF invoice hooks
- **Order trash deletes transactions** — trashing a WC order now deletes transaction records and resets payment to pending (instead of creating refund records)

### Added
- **User profile label prefix** — First Name / Last Name labels prefixed with student label (e.g. "Member's First Name") on admin profile pages
- **Payee name on PDF invoice** — billing first name shown bold at 1.16em before billing address (no label prefix)

## [3.3.3]

### Changed
- **WooCommerce class refactored into 8 traits** — split 2,678-line class into focused traits for smaller footprint and maintainability

### Added
- **Enrollment metadata on all WC orders** — Student Name, Academic Year, Grade added to any order for enrolled users
- **PDF Invoice integration** — enrollment details after billing address (WP Overnight plugin)

## [3.3.2]

### Fixed
- **WooCommerce fee product not purchasable** — other plugins filtering `woocommerce_is_purchasable` to false now overridden for the fee product
- **Block checkout payment tracking** — added `woocommerce_store_api_checkout_order_processed` hook for WC block checkout support
- **Fallback order detection** — `process_completed_order` now detects fee orders from line item meta when order-level meta is missing

## [3.3.1]

### Fixed
- **INR currency formatting** — `formatCurrency()` now uses Indian lakh/crore grouping (`##,##,###`) when currency code is INR
- **safeReload recursion** — prevented infinite recursion in frontend reload helper
- **WooCommerce order restore** — fixed order status restoration on payment reversal
- **Pay button fatal error** — fixed fatal error when Pay button rendered without WooCommerce active

## [3.3.0]

### Added
- **WooCommerce cart checkout** — fee payments now go through standard WC cart/checkout flow
- **Payment items** — new `KDC_qTap_Finance_Payment_Item` class for WC line item management
- **Custom fees in WC orders** — each fee slab becomes a separate WC line item with metadata
- **Order-payment sync** — automatic payment status updates when WC order status changes

### Fixed
- Frontend fees block: pay buttons, overdue status detection, enrollment slab matching
- WooCommerce term payment: one line item per term, multi-payment completion handling
- Offline pay button now shows alongside WooCommerce pay button

## [3.2.0]

### Added
- **Term-grouped payments** — payments grouped by term in frontend fees block and AJAX responses
- **WooCommerce term orders** — `create_term_order()` creates one WC order per term with all payment IDs
- **Term key on payments** — new `term_key` column (DB v1.8.0) linking payments to terms
- **Term-grouped frontend** — fees block renders payments grouped by term with term headers and due dates
- **Fee matrix UX** — improved fee matrix admin interface

## [3.1.0]

### Added
- **Terms system** — define academic terms with months, due dates, and ordering per year
- **Installment generator** — pure calculation class (`KDC_qTap_Finance_Installment_Generator`) generates installments from terms + fee matrix
- **Fee types per slab** — slabs now support `per_month`, `per_term`, and `per_year` amounts per grade
- **Collection modes** — monthly, quarterly, half_yearly, full_year billing periods
- **Onboarding wizard** (v3.0.8) — guided setup with business type selection, institute details, currency, fee matrix
- DB v1.7.0: added `installment_label`, `period_start`, `period_end` columns to payments

### Changed
- Fee matrix structure redesigned with `fee_types` and `collection_mode` (backward compatible with legacy `grades` structure)
- Notification types renamed from `education_*` to `finance_*`
- Import enrollment improvements and fee slabs loading fix

## [3.0.0]

### Changed
- **Major Rebrand: qTap Education → qTap Finance** — plugin renamed for broader business type support
- All file names, class names, constants, options, and hooks renamed from `education` to `finance`
- Automatic migration from `kdc-qtap-education` data on activation

### Added
- **Label Editor** (v2.13.0) — customize terminology per business type (Education, Housing Society, Club, Subscription, Custom)
- Import options UI with create users, skip duplicates, update existing, send notifications

## [2.12.17]

- Maintenance release

## [2.8.6]
- UI/UX alignment with kdc-qtap-mobile plugin for family consistency
- User list items now use display/actions container pattern
- Name displayed as bold block element, email on new line
- Message styles updated to use left-border accent pattern
- Loading state spinner alignment fixed
- Added responsive breakpoint at 600px for user list
- Inline form actions stack on mobile

## [2.8.5]
- Added hidden wp:search block to trigger block-theme input/button CSS loading
- Hidden element uses aria-hidden, data-nosnippet, and visual hiding for accessibility/SEO

## [2.8.4]
- Changed block icon from money-alt ($) to database dashicon
- Replaced offline payment modal with inline form within fee card
- Inline form slides in below fee card for smoother UX
- Success message shows in-place before auto-reloading fees
- Removed modal overlay CSS (simpler, more theme-integrated)
- Better mobile experience without overlay management

## [2.8.3]
- Refactored frontend CSS to use theme styles instead of custom styling
- Reduced CSS from 822 lines to ~280 lines (66% reduction)
- All buttons now use WordPress `wp-element-button` class for theme consistency
- Form inputs, selects inherit theme styling
- Only essential structural CSS and semantic colors retained (status badges, balance highlight)
- Modal overlay kept minimal with essential positioning only
- Improved theme integration and consistency

## [2.8.2]
- Added user selector for admin/staff on frontend fees block
- Admin/staff can search users by ID, name, email, or mobile number
- Admin/staff can browse users by grade and division
- Single user result auto-selects; multiple results show selection list
- Selected user's fees displayed with clear option to reset
- New helper function: kdc_qtap_education_can_view_other_users_fees()
- New AJAX handlers: search_users, get_users_by_grade
- New Enrollment method: get_all_by_criteria() for grade/division filtering

## [2.8.1]
- Added offline payment submission for users (always available)
- Users can submit: Reference/UTR Number, Payment Date, Receipt Screenshot, Notes
- User-submitted payments marked as "pending" verification status
- WooCommerce suggestion moved to Institute settings (Online Payments section)
- Admin notice shows count of payments pending verification
- New database fields: verification_status, verified_by, verified_at
- Transaction verification methods: verify(), reject(), get_pending_verification()
- Modal form for offline payment submission on frontend
- Database version updated to 1.4.0

## [2.8.0]
- Added WooCommerce integration for fee payments (conditional support)
- Virtual "Fees" product created on-the-fly per order (no persistent WC product)
- Order item metadata includes: Academic Year, Grade, Division, Fee Slab, Amount, User Details
- Automatic transaction recording on order completion
- Automatic payment status updates (pending → partial → paid)
- Refund/cancellation handling with payment reversal
- Support for multiple fee payments in single order ("Pay All Dues")
- Admin order display shows fee details with link to payment record
- Frontend "Pay Now" button on each fee with balance due
- Frontend "Pay All Dues" button in summary section
- WooCommerce My Account dashboard shows fees section (uses existing fees block)
- New setting: "Enable WooCommerce Payments" in Institute settings
- New classes: KDC_qTap_Education_WooCommerce, KDC_qTap_Education_WooCommerce_Frontend

## [2.7.19]
- Refactored export filters to use parent's `kdc_qtap_render_export_filter_panel()` helper
- Child plugin now only defines filter configuration (type, name, label, options)
- Parent plugin handles all HTML/CSS/JS rendering
- Removed custom CSS and JavaScript (now handled by parent)
- Uses declarative field configuration arrays
- Requires parent plugin v1.9.13+

## [2.7.18]
- Refactored export filters to use parent plugin's CSS classes
- Uses `kdc-export-sub-options`, `kdc-export-filter-header`, `kdc-export-filter-inputs`, `kdc-export-filter-field`
- Removed custom CSS (now uses parent's styles)
- Parent plugin handles visibility toggling automatically
- Requires parent plugin v1.9.12+

## [2.7.17]
- Added "toggle all" link for all grade checkbox groups
- Added Payments & Transactions export filters:
  - Date range (start/end with validation)
  - Amount range (min/max)
  - Payment status (radio: All/Completed/Pending/Failed/Refunded)
  - Filter by grade (checkboxes with toggle all)
- Date inputs prevent future dates and ensure end >= start
- Updated CSS to use WordPress core styles and admin theme colors

## [2.7.16]
- Changed Fee Type filter from dropdown to radio button group
- Changed Grade filter from multi-select to checkbox group
- Updated CSS for radio and checkbox group layouts (flexbox)

## [2.7.15]
- Refactored to use parent's `kdc_qtap_export_option_after` hook for filter panels
- Filter panels now properly nested under their respective checkboxes
- Separated filter panel rendering into dedicated methods
- Requires parent plugin with `kdc_qtap_export_option_after` hook support

## [2.7.14]
- Removed JavaScript DOM positioning that broke parent layout
- Filter panels now render inline below Education export group
- Requires parent plugin update with `kdc_qtap_export_option_after` hook for proper nesting

## [2.7.13]
- Cleaned up changelog entries

## [2.7.12]
- Fixed filter panel nesting - now inserts after label, inside same container
- Prevents filter panels from breaking out of Education group

## [2.7.11]
- Fixed JavaScript syntax error

## [2.7.10]
- Fixed export filter panels not appearing
- Added DOMContentLoaded to ensure DOM is ready

## [2.7.9]
- Fixed export filter panel placement/nesting issue
- Filter panels now properly insert after checkbox list items
- Added wrapper div with `clear: both` for proper layout flow
- Works with parent plugin's checkbox list structure (li or div)

## [2.7.8]
- Use parent plugin's `json_only` flag for export options
- Added `'json_only' => true` to General Settings and Institution Info
- Removed JavaScript workaround for hiding JSON-only checkboxes
- Requires parent plugin update with `json_only` support

## [2.7.7]
- Export filter UI improvements:
  - Uses `var(--wp-admin-theme-color)` for border color
  - Filter panels aligned under checkboxes (22px margin)
  - Cleaner CSS with dedicated class `.kdc-education-export-filters`
- Renamed "Fee Matrix (All Years)" to "Fee Matrix" (now filterable)
- Hide "General Settings" and "Institution Info" when CSV format selected
  - These are JSON-only exports (not applicable to CSV)
  - Checkboxes auto-uncheck when hidden

## [2.7.6]
- Added explicit `skip_duplicates` option support in all import functions
- Import options now properly handle:
  - `skip_duplicates` - Skip rows where record already exists
  - `update_existing` - Update existing records instead of skipping
  - `create_new_user` - Create new WP user when not found
  - `send_welcome_email` - Send WP welcome email to new users
- Consistent decision logic across fee_matrix, enrollments, and payments imports

## [2.7.5]
- Added Fee Matrix export filters:
  - Fee Type: All / Standard Only / Custom Only
  - Grade multi-select filter
- Standard fees have `is_na` field (mark fees N/A for specific grades)
- Custom fees are grade-specific (only exported for enabled grades)
- Filter UI appears when "Fee Matrix" checkbox is selected
- Both JSON and CSV exports respect the filters

## [2.7.4]
- Added enrollment export filter for grades (multi-select)
- Filter UI appears when "Enrollments" export checkbox is selected
- Exports only enrollments matching selected grades (leave empty for all)
- Note: `is_na` field is in Fee Matrix export, not enrollments
  - Used to mark fees as "Not Applicable" for specific grades

## [2.7.3]
- Enhanced CSV import reporting with detailed row tracking:
  - `row_imported` - Successfully created records with details
  - `row_updated` - Successfully updated records with details
  - `row_skipped` - Skipped records with reason
  - `row_errors` - Failed records with error message
- User creation now uses email as username (instead of sanitized email)
- Uses stronger password generation (24 chars with special chars)
- Updated all import functions: fee_matrix, enrollments, payments, transactions
- Removed double counting of errors as skipped

## [2.7.2]
- Updated import option names to match parent plugin:
  - `create_users` → `create_new_user`
  - `depends_on` now references `create_new_user`
- Changed from `wp_new_user_notification()` to `wp_send_new_user_notifications()`

## [2.7.1]
- Added "Send WordPress welcome email" option for enrollment CSV import
- Option only appears when "Create new users" is enabled
- Uses `wp_new_user_notification()` to send standard WordPress welcome email
- Disabled by default for bulk imports

## [2.7.0]
- Added `top_row` column support for CSV import
- Enables importing multiple enrollments per user in single CSV
- `top_row=true` marks rows with user creation data (first_name, last_name)
- Subsequent rows for same user leave top_row empty
- Added `is_top_row()` helper method for boolean value parsing
- Tracks created users in batch to avoid duplicate creation attempts
- Helpful error message when user not found and top_row not set
- CSV export marks first row per user as top_row=true
- Reordered CSV columns: data fields first, user creation fields last

## [2.6.13]
- Removed `user_login` fallback from enrollment import (uses email/user_id only)
- Removed `user_login` fallback from payment import (uses email/user_id only)
- CSV import/export now uses `user_email` as primary identifier
- Settings only exported in JSON format (not CSV) - already working

## [2.6.12]
- Removed `user_login` from CSV export data (payments and enrollments)
- Removed `user_login` from SQL query for payments export
- Import still supports `user_login` fallback for backwards compatibility

## [2.6.11]
- CSV export keys renamed to use `qtap_education_` prefix:
  - `qtap_education_fee_matrix`
  - `qtap_education_enrollments`
  - `qtap_education_payments`
  - `qtap_education_transactions`
- Settings no longer exported to CSV (JSON export only)
- Sample CSV data updated with same prefix convention

## [2.6.10]
- Transaction CSV export now includes `payment_id` and `wc_order_id` columns
- Updated sample CSV data with payment_id and wc_order_id examples
- Enables tracking transactions back to their parent payment and WooCommerce orders

## [2.6.9]
- Payment History "Edit Payment" action now requires `?debug=1` query parameter
- Normal view shows only: Add Transaction (+) and View Transactions (▼)
- Debug mode enables direct payment editing for troubleshooting

## [2.6.8]
- Fixed modal overflow on mobile - was wider than viewport
- Override inline `min-width: 450px` with `min-width: auto !important`
- Modal now uses `position: relative` instead of absolute centering on mobile
- Added `box-sizing: border-box` to prevent overflow
- Modal fills available width with proper padding
- Specific selectors for all education modals to ensure override

## [2.6.7]
- Renamed `.kdc-fee-slab-item` to `.kdc-qtap-education-fees-item`
- Mobile: RTE badge now inline with fee slab name (saves vertical space)
- Mobile: Amount and due date hidden to reduce clutter
- Modal dropdowns (Grade, Division) now full-width on mobile
- Fixed select elements overflowing modal container on mobile

## [2.6.6]
- **Mobile-responsive Fee Slabs checkboxes**
- New `.kdc-fee-slab-item` component with structured layout
- Fee slab name on own line with amount, badges, due date below
- On mobile (< 600px): meta info stacks vertically
- **Mobile-responsive modals**
- All modals adapt to mobile viewport (full width minus padding)
- Grid layouts collapse to single column
- Form fields use 16px font to prevent iOS zoom
- Buttons have 44px minimum touch targets
- New enrollment form table stacks on mobile
- Fee slabs container has max-height with scroll on mobile

## [2.6.5]
- **Mobile-responsive Payment History table**
- Table transforms to card layout on screens < 600px
- Fee slab name as card header
- Amount and Paid in 2-column grid with labels
- Due date and status badge inline
- Action buttons with 44px touch targets
- Transaction records row adapts to card layout
- Summary section wraps on mobile
- Status badges with unified color scheme
- Added aria-labels to action buttons

## [2.6.4]
- **Mobile-responsive enrollments table**
- Table transforms to card layout on screens < 782px
- Cards show student name as header with status badge
- Meta info (Grade, Division, RTE) displayed as inline tags
- Financial data (Total, Paid, Balance) in 3-column grid
- Full-width "View" button for easy touch interaction
- Unified status badge colors matching style guide
- Hidden ID column on mobile for cleaner layout

## [2.6.3]
- **Added WCAG AAA accessibility support for enrollment cards**
- Added `.kdc-qtap-accessible` wrapper class when accessibility mode enabled
- Updated accessible CSS with enrollment card specific styles:
  - High contrast badges (7:1 ratio)
  - 44x44px minimum touch targets for action icons
  - Focus indicators with 3px outline
  - Reduced motion support
  - High contrast mode (Windows) support
- Added aria-labels to icon-only buttons
- Added screen-reader-text for visual icons
- Added role="button" and tabindex to toggle elements
- Hooks into `kdc_qtap_admin_enqueue_scripts` for conditional CSS loading

## [2.6.2]
- **Receipt endpoint now requires user to be logged in**
- Non-logged-in users are redirected to login page
- After login, user is redirected back to the receipt URL

## [2.6.1]
- **Receipt URLs now serve files directly without exposing actual file path**
- URL stays as `{siteurl}/fees/receipt/{id}` - no redirect
- Supports images (JPEG, PNG, GIF, WebP) and PDFs
- Proper MIME type headers for inline display
- File is served with 1-day cache headers

## [2.6.0]
- **UI: Removed underlines from action icons**
- All action icon links now have `text-decoration: none`
- **UI: Unified color scheme with style guide**
- Status badges now match Fee Matrix colors:
  - Paid/RTE: green (#edfaef bg, #1e5a1e text)
  - Pending: amber (#fef8ee bg, #8b5c00 text)
  - Overdue: red (#fcf0f1 bg, #8a2424 text)
- Action icons unified:
  - Edit: uses --wp-admin-theme-color
  - Delete: WordPress red (#d63638)
  - Add payment: WordPress green (#00a32a)
  - Receipt: uses --wp-admin-theme-color
- **Feature: Shareable receipt URLs**
- New endpoint: `{siteurl}/fees/receipt/{uniqueid}`
- Receipt links are now shareable pseudo-URLs that redirect to actual file
- Unique ID is obfuscated (not sequential transaction ID)

## [2.5.22]
- **Fixed transaction receipt icon not showing**
- Added receipt_url computation when fetching transactions (was missing)
- **Updated colors to use WordPress admin theme color**
- Current enrollment border uses `var(--wp-admin-theme-color)`
- Current badge background uses admin theme color
- Icon links (edit, view transactions) use admin theme color
- Preserved specific colors: delete (red), add payment (green), receipt (purple)

## [2.5.21]
- **Moved receipt icon to Transaction Records actions**
- Receipt icon now shows per-transaction (if that transaction has a receipt)
- Removed old receipt icon from Payment History row (was checking all transactions)
- Icon appears after Edit and Delete icons in transaction row

## [2.5.20]
- **Improved enrollment status badge logic**
- Badge now always shows (even with no payments = Pending)
- "Paid" badge: All payments are paid or exempt
- "Overdue" badge: Any payment with balance > 0 and due date in the past
- "Pending" badge: Payments with balance > 0 and due date in future (or no due date)
- Exempt payments are treated as settled (equivalent to paid)
- Removed "Partial" status - simplified to Paid/Pending/Overdue

## [2.5.19]
- **Fixed modal inputs triggering WordPress form dirty check (proper fix)**
- Moved all modals to admin_footer hook - now rendered outside the user profile form
- New render_profile_modals() function outputs modals after the form closes
- This prevents modal inputs from being detected as form changes

## [2.5.18]
- **Fixed modal inputs triggering WordPress form dirty check**
- Stopped change/input events from bubbling up to parent form
- Applies to Edit Enrollment, Payment, Edit Payment, and Edit Transaction modals
- Targeted fix without affecting rest of WordPress

## [2.5.17]
- **Removed unnecessary form dirty check workarounds**
- Issue was caused by kdc-qtap-mobile plugin, not this plugin
- Cleaned up excess code from v2.5.11-v2.5.16

## [2.5.16]
- **More aggressive fix for "Changes may not be saved" warning**
- Now disables WordPress form dirty detection multiple times (immediately, 100ms, 500ms, 1000ms)
- Added periodic check every 2 seconds to keep it disabled
- This handles cases where WordPress sets up the handler after our script runs

## [2.5.15]
- **Auto-update payment status when RTE status changes**
- When enrollment is edited and RTE is enabled, pending payments for RTE-exempt fee slabs are automatically changed to "Exempt" status
- Amount due is set to 0 and note is added indicating RTE exemption
- Payments that are already paid or have partial payments are not affected
- If RTE is disabled, exempt payments are changed back to pending with original amount restored

## [2.5.14]
- **Fixed "Changes may not be saved" warning on User Profile page (root cause)**
- Fee Matrix form change detection now only runs when on the Fee Matrix page
- Added early return if Fee Matrix form element doesn't exist
- This prevents the beforeunload handler from being registered on other pages

## [2.5.13]
- **Fixed false "Changes may not be saved" warning on User Profile page**
- Disabled WordPress form dirty detection immediately on page load
- Our modal elements no longer trigger the unsaved changes warning
- Normal user profile form changes still work correctly

## [2.5.12]
- **Payment History UI Improvements**
- Right-aligned Amount, Paid, and Due Date columns in Payment History table
- Added receipt icon link in Actions column when a transaction has an attached receipt
- Receipt icon opens the media file in a new tab
- Also updated AJAX-loaded payment table with same alignment

## [2.5.11]
- **Fixed browser "Changes may not be saved" warning on User Profile page**
- Disabled WordPress form dirty detection before AJAX calls that reload the page
- Fixed warning appearing when updating enrollment, recording payments, etc.
- Also improved Fee Matrix page to properly unbind beforeunload on form submit

## [2.5.10]
- **Fixed "Error updating enrollment" when saving unchanged enrollment data**
- Enrollment save now always succeeds and fires the sync action
- Fixed issue where `update_user_meta` returning false for unchanged data was treated as an error
- Preserved original enrolled_date and enrolled_by when updating existing enrollments

## [2.5.9]
- Fixed empty Grade-Specific Fee card appearing at the end of the list
- Empty/invalid custom slabs are now filtered out before display
- Custom slab indices are now re-indexed after filtering and sorting for consistent form field names

## [2.5.8]
- **Auto-Sync Payments When Fee Matrix is Updated**
- When a new Standard Fee is created or grade amounts assigned, payment entries are automatically created for all enrolled users
- When a new Grade-Specific Fee is added, payment entries are automatically created for enrolled users in applicable grades
- Existing payments are not affected - only new fee slabs get payment entries
- RTE exemption rules are applied automatically when creating new payment entries

## [2.5.7]
- **Fixed Payment History Not Updating When Fee Slabs Changed**
- Payment records now sync when enrollment fee slabs are updated
- New fee slabs create new payment records automatically
- Deselected fee slabs remove payment records (only if no payments made)
- Payments with existing transactions are preserved when slabs are deselected

## [2.5.6]
- Removed date format description from date input labels

## [2.5.5]
- Renamed "Grade-Specific Fee Categories" to "Grade-Specific Fees"
- Added (yyyy-mm-dd) format hint to all date input labels

## [2.5.4]
- **UI Improvements**
- Fixed icon alignment in "Add Grade-Specific Fee" button
- "Save Fee Matrix" button now disabled until changes are detected
- Added "You have unsaved changes" notice when form is modified
- Added browser warning when leaving page with unsaved changes
- Save button re-enables if save fails, stays disabled after successful save

## [2.5.3]
- **Grade-Specific Fees: Collapsible Accordions with Summary**
- Grade-Specific Fees now use collapsible accordion UI (same as Standard Fees)
- Added slab summary showing grades count and total amount in header
- Added due date badge with color coding (overdue/today/upcoming)
- Added RTE Exempt badge in header
- Added "Filter by Grade" dropdown to filter Grade-Specific Fees
- Grade-Specific Fees now sorted by first applicable grade
- Dynamic summary updates when changing grade selections or amounts
- Unified CSS between Standard Fees and Grade-Specific Fees

## [2.5.2]
- **Fixed Fatal Error on Import**
- Fixed "Non-static method cannot be called statically" error during data import
- Made `KDC_qTap_Education_Database::create_tables()` method static

## [2.5.1]
- **WordPress 6.7+ Compatibility Fix**
- Fixed translation loading timing issue that could trigger notices in other plugins
- Plugin initialization now deferred to `plugins_loaded` hook
- Prevents `_load_textdomain_just_in_time` notices in WordPress 6.7+

## [2.5.0]
- **Major Rebrand: qTap School → qTap Education**
- Renamed plugin from "qTap School" to "qTap Education" for broader appeal
- Updated all file names, function names, class names, and constants
- All references to "school" changed to "education" throughout codebase
- Plugin slug changed from `kdc-qtap-school` to `kdc-qtap-education`
- Text domain changed from `kdc-qtap-school` to `kdc-qtap-education`
- Database option names updated (backward compatible)
- No functional changes - this is a naming update only

## [2.4.21]
- **Fixed Grade-Specific Fee Error & CSS Consistency**
- Fixed "Undefined array key 'name'" error when creating new Grade-Specific Fees
- Grade-Specific Fees now use same CSS structure as standard fees
- Removed inline styles from Grade-Specific Fee markup
- Both existing and new Grade-Specific Fees use `.kdc-qtap-education-fee-slab-content` wrapper
- Consistent meta field layout between standard and Grade-Specific fees
- Added proper `.required` class for required field indicators

## [2.4.20]
- **Fee Matrix: Mobile UI Polish**
- Fee slab header now displays as vertical list on mobile (not inline)
- Accordion icon positioned to top-right corner on mobile
- Description field properly constrained to container width
- Date input field sized appropriately (auto width, max 160px on mobile)
- Added overflow hidden to content area to prevent field overflow
- RTE checkbox field no longer stretches unnecessarily
- Improved spacing and alignment for all meta fields

## [2.4.19]
- **Fee Matrix: Mobile UI Improvements**
- Added comprehensive mobile-responsive CSS for Fee Matrix
- Header elements now wrap properly on small screens
- Grade cards use CSS Grid for responsive 2-column/1-column layout
- Badges no longer overflow on mobile
- Select Toggle button becomes full-width on mobile
- Meta fields (Description, Due Date, RTE) stack vertically on mobile
- Replaced inline styles with proper CSS classes for better maintainability
- Added support for extra small screens (< 480px)

## [2.4.18]
- **Fee Matrix: Select Toggle & Subtle Badge Colors**
- Added "Select Toggle" button for Applicable Grades & Amounts sections
- Toggle quickly selects/deselects all grades in a fee category
- Works for both standard fees and Grade-Specific Fees
- Updated due date badge colors to be more subtle and eye-friendly

## [2.4.17]
- **Fee Matrix: Due Date Badge in Header**
- Due date now displayed as a colored badge in the fee slab accordion header
- Red badge for overdue dates (past)
- Yellow/orange badge for due today
- Green badge for upcoming dates (future)
- Badge updates dynamically when due date is changed
- Calendar icon included in the badge for visual clarity

## [2.4.16]
- **Fixed Fee Matrix Save Issue**
- Fixed parse_str stripping nested array data during AJAX save
- Fixed unchecked grade checkboxes not being saved correctly
- Sanitize method now ensures all configured grades are included in saved data
- Proper handling of enabled/disabled state for each grade

## [2.4.15]
- **General Settings: Removal Confirmation**
- Added confirmation dialog when removing Academic Years, Grades, or Fee Categories
- Warning message prevents accidental deletion of items with associated data
- Division removal does not require confirmation (less critical)

## [2.4.14]
- **Fee Matrix: Unified Grade Checkbox UI**
- Replaced table-based N/A checkbox layout with card-based checkbox + amount layout
- Standard fees now use the same UI flow as Grade-Specific Fees
- Each grade displayed as a card with checkbox to enable/disable and amount input
- Changed data structure from `na: true/false` to `enabled: true/false`
- Full backward compatibility with legacy `na` field data
- Consistent styling across all fee types

## [2.4.13]
- **Export Options: Parent Helper Integration**
- Replaced custom export markup with `kdc_qtap_render_export_group()` parent helper function
- Uses `var(--wp-admin-theme-color)` for consistent styling
- Auto-detects icon from `kdc_qtap_register_plugin()` registration
- Updated checkbox names to match parent helper format

## [2.4.12]
- **Plugin Check Fixes**
- Removed `%i` placeholder usage (only supported in WP 6.2+)
- Replaced `move_uploaded_file()` with WordPress Filesystem API
- Replaced `finfo_open()` MIME detection with `wp_check_filetype()`
- Added proper `$_POST['data']` validation before use

## [2.4.11]
- Changed plugin registration icon from emoji to dashicon (`dashicons-welcome-learn-more`)

## [2.4.10]
- **Institute Tab: Consistent Year Selector UI**
- Current Academic Year field now uses highlighted card-style matching Enrollments/Fee Matrix tabs
- Updated description for clarity
- Institute Address textarea now uses `regular-text` class

## [2.4.9]
- **Reverted Academic Year Selector UI**
- Restored highlighted card-style year selector for Enrollments and Fee Matrix tabs
- Calendar icon with blue accent border for visual prominence
- Added descriptive help text explaining selector purpose

## [2.4.8]
- **Academic Year Selector: Enhanced UI**
- Enrollments & Fee Matrix tabs now use standard form-table layout for year selector
- Added descriptive help text explaining the selector's purpose
- Consistent styling with other WordPress settings fields

## [2.4.7]
- **Settings UI: WordPress Native Styling**
- Refactored all settings to use standard `<table class="form-table">` layout
- Moved field descriptions after inputs (WordPress standard pattern)
- Simplified Enrollments and Fee Matrix year filter selectors
- Removed unnecessary inline styles throughout settings pages

## [2.4.6]
- **General Settings: Updated Fee Categories field**
- Renamed "Fees Slabs (Legacy)" to "Fee Categories"
- Updated description for clarity
- Consistent naming throughout UI and JS localization

## [2.4.5]
- Removed `max-width: 800px` constraint from settings pages
- Now uses default WP-Admin dashboard width for better space utilization

## [2.4.4]
- **CSV Import: Extensibility Hook**
- Added `kdc_qtap_education_import_group` filter for custom import groups
- Extensions can now add new import types without modifying core plugin
- Compatible with parent plugin v1.7.2+ batch processing and progress bar

## [2.4.3]
- **CSV Import: Enhanced Error Reporting**
- Import results now include `row_errors` array for detailed per-row error tracking
- Compatible with parent plugin's import report generation
- All four import types updated: Fee Matrix, Enrollments, Payments, Transactions

## [2.4.2]
- All custom blue colors now use WordPress Admin Theme color
- Uses `var(--wp-admin-theme-color, #2271b1)` CSS variable with fallback
- Respects user's chosen admin color scheme

## [2.4.1]
- **Settings > Institute tab UI improvements**
- Moved Current Academic Year to top with prominent highlighted card design
- Added calendar icon and description for Academic Year selector
- Improved overall page structure and visual hierarchy

## [2.4.0]
- Removed `payment_mode` field from Enrollments CSV export/import
- Field was no longer relevant for enrollment data
- Simplified enrollment structure

## [2.3.9]
- Enhanced Academic Year selector UI in Enrollments tab
- Added calendar icon and prominent card-style design
- Added description text explaining the selector's purpose

## [2.3.8]
- Removed custom CSS from Academic Year selector in Enrollments tab
- Now uses WP-Admin default styling for consistency

## [2.3.7]
- **NEW: CSV Import System with Header Mapping**
- Integrates with parent plugin (kdc-qtap v1.7.0+) CSV import system
- **Importable Data Types:** Fee Matrix, Enrollments, Payments, Transactions
- **Smart Column Mapping:** Auto-maps columns using field aliases
- **Field Aliases:** Supports multiple naming conventions
- **Import Options:** Skip or update existing records, create new users
- **Validation:** Skips instruction rows, reports row-level errors

## [2.3.6]
- **IMPROVED: CSV Export/Import with Detailed Sample Templates**
- Added `description` and `is_na` fields to Fee Matrix CSV
- Added `first_name`, `last_name`, `roll_number` fields to Enrollments
- Added `reference` and `receipt_url` fields to Transactions
- **NEW: Media Library Receipts** - Transactions support `attachment_id`
- Database updated to v1.3.0 with `attachment_id` column

## [2.3.5]
- **IMPROVED: Fee Matrix Accordion UI**
- Fee slabs now collapsible with smooth accordion animation
- Visual summary shows grades count and total amount
- RTE Exempt badge visible in collapsed state
- Expand All / Collapse All buttons for quick navigation
- Keyboard accessible (Tab, Enter, Space)

## [2.3.4]
- **NEW: CSV Export Support**
- Creates Google Sheets compatible CSV files
- Exports create ZIP with multiple CSV sheets
- **NEW: Sample CSV Templates** for bulk import

## [2.3.3]
- **NEW: Fees Currency Fieldset**
- Currency settings grouped in dedicated fieldset
- **NEW: Currency Decimal Places Setting** (0-3)
- Helper function: `kdc_qtap_education_get_currency_decimals()`

## [2.3.2]
- **IMPROVED: Export Data Structure**
- Transactions now nested as child array within each payment entry
- **FIXED: Payment Method Display** in transaction records
- **NEW: Edit Transaction** functionality

## [2.3.1]
- **IMPROVED: Automatic Overdue Detection**
- Payment History shows "Overdue" status automatically when due date has passed
- **NEW: Enrollments Dashboard** with summary cards
- **NEW: All Overdue Payments View** across all academic years
- **CHANGED: Delete Transaction Instead of Payment**

## [2.3.0]
- **NEW: Granular Data Export/Import**
- Master toggle with sub-checkboxes for individual sections
- Full import support with user matching by email/login
- **NEW Export class:** `KDC_qTap_Education_Export`
- Requires qTap App v1.4.0+ for parent export hooks

## [2.2.2]
- **NEW: Payment History Edit/Delete**
- **NEW: Grade-Level Custom Fee Slabs**
- New AJAX endpoints for payment management

## [2.2.1]
- **Improved Add New Enrollment UX**
- AJAX-based saving without page reload
- Improved Edit/Delete button styling with dashicons

## [2.2.0]
- **BREAKING: Removed Allowed Roles setting**
- Now uses parent plugin's (qTap App v1.5.0+) REST API roles setting
- **NEW: Edit/Delete Enrollment buttons** on user profile cards
- New AJAX endpoints for enrollment management

## [2.1.9]
- **Enhanced Record Payment Modal** with additional fields
- Database schema updated (v1.2.0) with new transaction columns
- Receipt file upload support

## [2.1.8]
- Fee Matrix save/copy now uses Allowed Roles permission

## [2.1.7]
- **NEW: Role-Based Access Control**
- Added "Allowed Roles" setting in Institute tab
- New capability helper functions

## [2.1.6]
- **Redesigned User Profile** - Card-based layout for school enrollments
- Collapsible cards for easy navigation through enrollment history

## [2.1.5]
- **NEW: Enrollments Tab** - Admin interface to view all enrollments year-wise
- Filter by Academic Year, Grade, Payment Status

## [2.1.4]
- **WooCommerce Integration Preparation**
- New Transaction and Payment methods for WooCommerce

## [2.1.3]
- **Fixed: Academic Year Change** - Now dynamically loads enrollment data via AJAX

## [2.1.2]
- **NEW: Currency Settings** - Configure fees currency in Institute tab
- Helper functions for currency formatting

## [2.1.1]
- **NEW: Institute Tab** - New settings tab for institute information
- Moved Current Academic Year from General to Institute tab

## [2.1.0]
- **Architecture Restructure** - Complete overhaul of fee management system
- **NEW: Dynamic Fee Slabs** - Define fee components in General Settings
- **NEW: Card-Based Fee Matrix** - Redesigned UI with per-slab configuration
- **NEW: Per-Slab RTE Exempt** - Mark individual fee slabs as exempt
- **NEW: Fees Block** - Gutenberg block "qTap Education Fees"
- **NEW: Shortcode** - `[kdc_qtap_education_fees]`
- **NEW: Year Selector** - Users can view historic fee data

## [2.0.0]
- **NEW: Fee Matrix** - Grid-based fee pricing (Year × Grade × Slab)
- **NEW: Enrollment System** - Per-year enrollment with payment modes
- **NEW: Payment Tracking** - Custom tables for payments and transactions
- **NEW: Partial Payments** - Support for multiple partial payments
- **NEW: RTE Support** - Right to Education with admin approval audit trail
- **NEW: Fees Endpoint** - Dedicated /fees page for mobile access
- Added Database, Fee Matrix, Enrollment, Payment, Transaction classes
- Daily cron for overdue status updates

## [1.7.2]
- Fixed parent export integration

## [1.7.1]
- Fixed WordPress Plugin Checker slow query warning

## [1.7.0]
- **BREAKING: Requires qTap App v1.4.0+**
- Removed Import/Export tabs (now in parent plugin)
- Added parent plugin integration hooks

## [1.6.5]
- Fixed translation loading timing for WordPress 6.7+
