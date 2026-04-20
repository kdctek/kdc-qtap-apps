# Changelog

All notable changes to qTap Finance are documented in this file.

## [3.16.5] - 2026-04-20

### Added
- **Credits audit tab at `Finance > Credits`.** Lists every user with a positive `kdc_qtap_finance_credit` balance via a single indexed usermeta scan, sorted by credit desc. Columns: user id, student name + login, email, credit (right-aligned mono), actions. Summary card at the top shows total-users and total-outstanding. Each row has a reason-required "Zero Out" button and a "View" deep-link to the user profile in a new tab.
- **Audit log on zero-out.** Every "Zero Out" action appends an entry to `kdc_qtap_finance_credit_adjustments` user meta (JSON array) capturing: prior balance, new balance (0), delta, reason, admin id + login, and MySQL timestamp. Action also emits a `kdc_qtap_debug_log()` line for correlation with import logs.
- **`ajax_zero_user_credit` handler** (action `kdc_qtap_finance_zero_user_credit`, nonce `kdc_qtap_finance_nonce`) — validates capability, user existence, and non-empty reason before mutating meta.

### Files changed
- New [trait-kdc-qtap-finance-admin-tab-credits.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-admin-tab-credits.php).
- [class-kdc-qtap-finance-admin.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-admin.php) — require + `use` trait, register tab in `get_tabs()` between Verify Payments and Reminder Queue, wire `case 'credits'` in render switch, register AJAX handler.

## [3.16.4] - 2026-04-20

### Fixed
- **Transaction delete now reverses parked credit and cascades trickle children.** Previously when `record_transaction()` overflowed a ₹1,31,300 payment onto a term with only ₹1,30,800 due, the extra ₹500 got parked in `kdc_qtap_finance_credit` user meta — but deleting the source transaction via the Payment History inline button only reversed `amount_paid` and left the ₹500 credit stranded. Similarly, if a source transaction spawned a "Carry Forward" trickle child on the next term, that child became orphaned on delete of the parent. Both now unwind correctly in a single delete.

### Added
- **Database schema v2.2.0** — two new columns on the transactions table (migration auto-runs on plugin load):
  - `parent_transaction_id BIGINT UNSIGNED NULL` + index — points from a trickle child (or auto-credit child) back to the source transaction that spawned it.
  - `credit_parked DECIMAL(14,2) NOT NULL DEFAULT 0` — records how much of *this* transaction's amount ultimately landed in the user's credit meta (via `kdc_qtap_finance_add_user_credit()`). Enables exact reversal on delete.
- **`KDC_qTap_Finance_Payment_Transaction::create()` / `::update()`** now accept the two new fields.
- **`KDC_qTap_Finance_Payment::trickle_excess_forward()`** accepts an optional `$source_transaction_id` — stamps `credit_parked` on the source when the chain ends in user credit, and sets `parent_transaction_id` on downstream trickle rows so they cascade on delete.
- **`KDC_qTap_Finance_Payment::trickle_excess_from_payment()`** (public wrapper used by `Payment_Transaction::verify()` etc.) gets a matching optional `$source_transaction_id` parameter for callers that want the same lineage tracking on post-hoc trickle.
- **`KDC_qTap_Finance_Payment::record_transaction()`** now also stamps `parent_transaction_id` on the auto-applied-credit child row so a delete of the source cascades the child too. (Auto-credit *consumption* reversal — adding the consumed credit back to user meta — is a documented follow-up.)

### Changed
- **`KDC_qTap_Finance_Payment_Transaction::delete()`** reverses in a defined order: (1) recursively delete children (trickle + auto-credit rows found via `parent_transaction_id`), each rolling back its own `amount_paid`; (2) refund any `credit_parked` back to the user's credit meta via `consume_user_credit()` (clamped at zero if the user has since spent some); (3) delete this row + reverse `amount_paid` on its payment.

### Files changed
- [class-kdc-qtap-finance-database.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-database.php) — `DB_VERSION` bumped to 2.2.0; migration 2.2.0 block adds both columns + index on upgrade; CREATE TABLE SQL includes them for fresh installs.
- [class-kdc-qtap-finance-payment-transaction.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-payment-transaction.php) — `create()` / `update()` accept `parent_transaction_id` + `credit_parked`; `delete()` rewritten with three-step cascade.
- [class-kdc-qtap-finance-payment.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-payment.php) — `record_transaction()` threads `parent_transaction_id` to the auto-credit child row and passes the source transaction id into trickle; `trickle_excess_forward()` + `trickle_excess_from_payment()` accept the source id and stamp lineage / parked credit.

## [3.16.3] - 2026-04-20

### Changed
- **Transactions CSV import: auto-append `WC Order #<number>` to transaction notes when a WC order is generated.** Mirrors the existing behaviour of `ajax_record_payment()` — the order number (honours any prefix/sequential plugin, not just the raw post ID) is joined onto the user-provided `notes` with ` | ` so audit trails look identical whether a payment was recorded via the Record Payment modal or a bulk CSV import. Transactions imported with `wc_order=no` (or `false` / `0`) are unaffected. Historical rows imported before this version are *not* backfilled.

### Files changed
- [trait-kdc-qtap-finance-import-csv-processors.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-import-csv-processors.php) — `import_transactions_csv()` now appends the WC Order note bit between order creation and the `record_transaction()` call.

## [3.16.2] - 2026-04-19

### Added
- **Transactions CSV import: optional `payee_name` column.** Name of the person who made the payment. When supplied, the importer updates the user's `billing_first_name` user meta before the WC order is created and also sets `billing_first_name` on the order itself — mirroring the behaviour of the admin Record Payment modal so WCPDF receipts and the orders table consistently reflect the payer.
- **Transactions CSV import: optional `receipt_url` column.** External URL pointing to a receipt image / PDF. The importer downloads the file via `wp_safe_remote_get()` (HTTPS, 20 s timeout, 5 redirects), enforces the same validation as the Record Payment upload path (JPG / PNG / GIF / WebP / PDF, ≤ 2 MB), saves it into the plugin receipts folder using the canonical `receipt-<payment_id>-<user_id>-<hash>.<ext>` filename, and patches the resulting filename onto the freshly-created transaction row via `receipt_file`. From there the existing `/receipt/<id>` rewrite + fees endpoint serves it exactly like receipts uploaded through the UI.
- **Alias-based auto-mapping** for the two new columns — `payee_name` matches `payee / paid_by / payer / payer_name`; `receipt_url` matches `receipt / receipt_image / source_url / image_url`.
- **Non-fatal receipt failures.** A URL that returns non-2xx, empty body, oversize content, or an unsupported MIME type logs a row-level warning and the transaction is still recorded. This avoids losing legitimate payment data when a staff member supplies a broken receipt link.

### Files changed
- [trait-kdc-qtap-finance-import-csv-targets.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-import-csv-targets.php) — two new optional fields in `get_transactions_import_fields()`.
- [trait-kdc-qtap-finance-import-csv-processors.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-import-csv-processors.php) — new private `download_receipt_from_url()` helper; `import_transactions_csv()` parses payee_name / receipt_url, updates user meta, applies billing name to WC order, and patches `receipt_file` alongside `reference` on the transaction row.

## [3.16.1] - 2026-04-18

### Fixed
- **Transactions CSV import now reports "Updated" in job stats.** The v3.16.0 importer increments `success` (imported) and `skipped` (duplicates) but never `updated`, so the parent plugin's Jobs > Import stats summary showed `0 Updated` even when every row successfully bumped a Payment row's `amount_paid` via `record_transaction()`. The importer now emits `updated++` alongside `success++` on each successful row — so a 100-row CSV that processes cleanly shows `100 imported, 100 updated` (reflecting 100 new transaction records + 100 Payment rows affected).

### Files changed
- [trait-kdc-qtap-finance-import-csv-processors.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-import-csv-processors.php) — `import_transactions_csv()` initialises `'updated' => 0` and increments after each successful `record_transaction()`.

## [3.16.0] - 2026-04-19

### Changed (breaking for legacy CSV format)
- **Fee Transactions CSV import rewritten around the trickle engine.** The old importer expected one CSV row per term/slab split and used raw `$wpdb->insert` + `UPDATE payments SET amount_paid = amount_paid + X` statements — bypassing `Payment::record_transaction()` entirely (so: no cap at `amount_due`, no overflow trickle, no credit consumption, no payment-item waterfall allocation, no audit styling). New format: **one CSV row per real payment event**. Routes every row through `Payment::record_transaction()` so a single ₹2,00,000 RTGS entry automatically pays term 1, carries the excess into term 2 as a `Carry Forward` row, and honours all the downstream audit/display rules.

### Added
- **`kdc_qtap_finance_parse_flexible_date()`** helper — accepts `YYYY-MM-DD`, `DD-MM-YYYY`, `MM-DD-YYYY` with `-`, `/`, `.`, or space separators; disambiguates via site-wide option `kdc_qtap_finance_date_order_preference` (`dmy` default / `mdy`); strtotime fallback handles named-month variants (e.g. `21-Jul-2025`). Returns canonical `Y-m-d` or `null`.
- **Flexible user resolution** in the Transactions importer — any one of `user_id`, `user_login`, or `user_email` (first non-empty wins). Rows missing all three are rejected with "No valid user identifier".
- **Auto-anchor** — when `slab` is empty, the importer picks the earliest pending regular payment for `user + year` (excludes `_custom_*`, `[…]`, `_user_fee_*` slabs; orders by `due_date ASC, id ASC`). Optional `start_slab` / `slab` columns override when staff want to target a specific term or custom bucket.
- **`wc_order` column** (default `true`) — accepts truthy (`1`, `true`, `yes`, `y`, `t`) / falsy (`0`, `false`, `no`, `n`, `f`). When `true`, the importer creates a completed WC order before `record_transaction()` runs (same ordering as admin Record Payment). When `false`, only the transaction + trickle runs — for historical back-fills.
- **Alias-based reference column** — row honours `reference`, `utr`, or `transaction_id` (first non-empty); stored on the freshly-created transaction row via a follow-up `KDC_qTap_Finance_Payment_Transaction::update()`.
- **Dedupe on re-import** — when the `skip_duplicates` option is on, a transaction matching `(user_id + payment_date + amount + reference)` across all payments is skipped with *"Duplicate already imported"*. Lets staff safely re-run the same CSV.
- **Stricter validation** — invalid `payment_method` (not in `get_all_payment_methods()`), unparseable `payment_date`, amount ≤ 0, empty year all reject with per-row error logs that pinpoint row number + cause.

### Files changed
- [kdc-qtap-finance-helper-functions.php](kdc-qtap-finance/includes/kdc-qtap-finance-helper-functions.php) — new `kdc_qtap_finance_parse_flexible_date()`.
- [trait-kdc-qtap-finance-import-csv-processors.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-import-csv-processors.php) — `import_transactions_csv()` fully rewritten; raw DB inserts + manual `amount_paid` bumping removed; routes through `Payment::record_transaction()`.
- [trait-kdc-qtap-finance-import-csv-targets.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-import-csv-targets.php) — `get_transactions_import_fields()` updated with the new 11-field schema (user_id/login/email, year, amount, date, method, method_title, reference, slab, wc_order, notes); description clarified that allocation is automatic via trickle.

### Migration notes
No database migration needed — the schema didn't change. Existing sample CSVs with the old per-term-slab format will no longer import cleanly; regenerate them using the per-event format. Staff can re-import historical CSVs with the new format and the trickle flow will produce correct term-by-term allocation plus `Carry Forward` audit rows.

## [3.15.53] - 2026-04-19

### Changed
- **Record Payment success alert gated on carry-forward.** Previously any successful save triggered the native `alert()` with the server's breakdown — noisy for regular/partial payments where the message was just *"Payment recorded — ₹X applied to this term."* Now `ajax_record_payment()` returns a `has_carry_forward` boolean in the success payload; JS only alerts when that flag is `true`. Plain saves stay silent (the page reload is signal enough).

## [3.15.52] - 2026-04-19

### Fixed
- **Live overflow notice never appeared** in the Record Payment modal — the JS in v3.15.51 targeted `#kdc-qtap-finance-payment-overflow-notice` but that DIV was never added to the modal template. Added it right below the Amount input with a yellow-outlined warning style so the admin sees: *"₹X will settle this term. ₹Y will be carried forward to the next pending term (or parked as credit if none)."* as soon as they type an amount above the term's balance.

## [3.15.51] - 2026-04-19

### Changed
- **Trickle transactions now use method `credit`, not the source method.** Previously the downstream (carry-forward) row inherited the source payment's method (e.g. RTGS) — that was accounting-incorrect because term 2 wasn't actually paid by RTGS, it was settled by overflow. `trickle_excess_forward()` now calls `record_transaction()` with `$payment_method = 'credit'` and a synthesised `payment_method_title` *"Carry Forward (from RTGS)"* so the display remains readable while the underlying method key is correct.
- **Carry-forward rows visually flagged** in the user-profile Payment History: amount coloured grey + italic, prefixed with `↪` arrow, with tooltip *"Internal allocation from a prior payment — not a new receipt"*. Signals clearly that summing the column would double-count: the actual money came in on the source row.

### Added
- **Pre-submit overflow preview** — typing above the term's balance in the Record Payment modal shows a live note below the field: *"₹X will settle this term. ₹Y will be carried forward to the next pending term (or parked as credit if none)."* Uses the `data-term-balance` attribute stashed when the Record Payment button is clicked.
- **Post-submit split alert** — on success the admin sees a native alert with the server's breakdown (e.g. *"Payment recorded — ₹3,36,050 applied to this term, ₹38,950 carried forward to the next term."*) before `safeReload()` refreshes the profile.

## [3.15.50] - 2026-04-19

### Changed
- **Trickle + auto-credit transactions now inherit the source payment's date.** Previously the recursive trickle transactions and the credit-auto-apply sub-transaction both used `current_time( 'mysql' )`, which caused the "next term" row to appear dated on admin action day (e.g. 2026-04-19) instead of the actual payment date (e.g. 2025-07-21). `Payment::record_transaction()` now threads `payment_date` / `transaction_date` into the credit sub-transaction via `$extras`, and `trickle_excess_forward()` already propagates `$extras` recursively so every downstream trickle row reflects the real date the user paid.
- **Overpayment migration (`migrate_overpayment_trickle_3_15_49`)** now looks up each overpaid source payment's most recent transaction and passes its `payment_date` as `$extras` to `trickle_excess_from_payment()`. Re-runnable via a new `kdc_qtap_finance_overpayment_trickle_dated_done` gate so existing reconciled rows get their dates corrected on upgrade.

### Added
- **`Credit` payment method in Record Payment dropdown.** New built-in method added to `KDC_qTap_Finance_Database::get_all_payment_methods()`. When admin selects `Credit`, `ajax_record_payment()` validates the user has enough parked credit (`kdc_qtap_finance_get_user_credit`), consumes it via `kdc_qtap_finance_consume_user_credit()` **before** calling `record_transaction()`. Because the payment method is `'credit'`, the `record_transaction()` auto-credit branch is skipped — preventing double-consumption. The split message on success reports exactly what happened.

## [3.15.49] - 2026-04-19

### Fixed
- **Admin Record Payment bypassed trickle + credit** — `ajax_record_payment()` in `trait-kdc-qtap-finance-user-meta-payments.php` was manually incrementing `amount_paid` and writing its own transaction row. An admin entering ₹2,00,000 on a term with ₹1,61,050 remaining balance left the row at `amount_paid = 3,75,000 > amount_due = 3,36,050` and never propagated the ₹38,950 surplus. Now routes through `Payment::record_transaction()` — caps the row, auto-consumes available credit, and trickles any overflow to the next pending regular term (falls back to user credit only if no pending rows remain). The immediate transaction row is still patched with `receipt_file`, `reference`, `transaction_type`, `recorded_by` (fields `record_transaction()` doesn't accept directly).
- **Success message now explains the split** — the admin sees e.g. *"Payment recorded — ₹3,36,050 applied to this term, ₹38,950 carried forward to the next term."* Computed by comparing pre- and post-call user credit balance against the amount that exceeded the term's remaining due.

### Added
- **`migrate_overpayment_trickle_3_15_49()`** — one-time upgrade task that scans every regular payment with `amount_paid > amount_due`, caps it at the due amount (flips status to `paid`), and calls `Payment::trickle_excess_from_payment()` to push the surplus into the next pending term (or park as credit if nothing pending). Gated once by option `kdc_qtap_finance_overpayment_trickle_done`. Handles rows created by pre-3.15.49 admin Record Payment where the gate in `migrate_reclaim_legacy_overpayments_3_15_43` had already been set before those rows became overpaid.

## [3.15.48] - 2026-04-19

### Fixed
- **Legacy-overpayment credit never trickled forward** — the v3.15.43 backfill capped overpaid term rows and parked the surplus as user credit, but never moved the credit into the next pending term. That left a term-1 row showing `amount_paid > amount_due` on-screen and a term-2 row underpaid, and any receipt generated from the original transaction still attributed the full amount to term 1.
- New one-time migration `migrate_trickle_user_credits_3_15_48()` walks every user with `kdc_qtap_finance_credit > 0`, finds their pending regular payments in due-date order, consumes credit against each until exhausted, updates `amount_paid`/`status`, and writes a `credit` transaction row on each recipient term with the note *"Legacy credit trickled forward from prior overpayment"*. Gated once by option `kdc_qtap_finance_credit_trickle_done`.

## [3.15.47] - 2026-04-19

### Removed
- **Excess column** from Report — since v3.15.28, `record_transaction()` guarantees `amount_paid ≤ amount_due` via trickle + credit, so Excess is always 0. Credit column now replaces it end-to-end. Dropped:
  - `excess` column entry in `report_build_columns()`
  - `$row['excess']` in `report_build_user_row()` and the Summary `aggregateRows`
  - `row.excess` computation in `adjustTotals` and the Summary aggregator
  - `kdc-col-excess` / `kdc-amount-excess` CSS rules + `createdRow` red-highlight branch
  - `'excess'` from `numericTypes` footer-sum set and the render-type switch
  - `excess` i18n string in `class-kdc-qtap-finance-admin.php`

### Fixed
- **Credit column alignment** — added `td.kdc-col-credit` to the right-align + monospace rule set so values line up with other amount columns. Applied green colour (`#00632a`) to distinguish from red Balance.

## [3.15.46] - 2026-04-19

### Fixed
- **Report Credit column** now renders through `amountRender` so values appear with the configured currency symbol (₹) and Indian-lakh formatting, matching every other numeric column. Previously showed raw numbers.

### Removed
- **Excess Payments tab** — dropped the dedicated tab, the `#kdc-report-pane-excess` + `#kdc-report-dt-excess` markup, the `renderExcessPayments()` renderer, its call from `render()`, and the `excessPayments` / `excessDesc` / `noExcess` i18n strings in `class-kdc-qtap-finance-admin.php`. Excess values are still present per-row in the main Report (last column before Credit); staff who need an excess-only view can filter/sort that column.

## [3.15.45] - 2026-04-18

### Changed
- Maintenance patch — re-publishes v3.15.44 Credit visibility changes (Fees-block notice + Report Credit column) as a clean stable tag.

## [3.15.44] - 2026-04-18

### Added
- **Annual vs Full Program view toggle on the Report tab** — next to the year dropdown. Annual view (default for single-year selections) slices multi-year enrollments (e.g. a `2025-2027` program) into just the selected year's portion, so a student in a multi-year program appears in every year of their span with only that year's installments. Full Program view preserves legacy exact-match behavior. Toggle auto-hides when a multi-year key is selected.
- **Year-range helpers** in `kdc-qtap-finance-helper-functions.php` — `parse_year_range()`, `year_range_covers()`, `is_start_slice()`, `cycle_for_slice()`, `year_date_range()` (academic-year-aligned date window using the per-year configured start month).
- **`KDC_qTap_Finance_Payment::get_by_user_year_annual()`** — filters installments by `period_start` date window instead of `academic_year` string equality, naturally slicing multi-year payments into the correct single-year bucket. Custom fees (NULL `period_start`) merged via exact year match.
- **Excess Payments tab** in the Report page — lists students whose total received exceeds total collection for the selected year. Dedupes by user across groups (keeps the largest excess), sorts by excess descending, supports CSV export with footer totals.
- **Clickable User ID** — the ID column in the Report tables now links to the WP user-edit screen in a new tab.
- **Available-credit notice** on the Fees block — shows any parked credit the signed-in user has with a "will be applied automatically on your next payment" message. Hidden when staff view another user's fees via the switcher.
- **Report — Credit column** — new `Credit` column appended to every row (per-user credit from `kdc_qtap_finance_credit` user meta). Aggregated per group in the Summary tab via `aggregateRows()`. Included in footer totals (`numericTypes` updated in `kdc-qtap-finance-report.js`).

### Changed
- **`KDC_qTap_Finance_Enrollment::get_all_by_year()` and `::get_all_by_criteria()`** accept a new `view_mode` arg (`'full_program'` default, `'annual'` union). In annual mode, they include multi-year enrollments whose parsed range covers the target year. Exact-match wins over span-match when a user has both. Return rows are decorated with `source_year` so callers can distinguish the two. All existing callers unchanged due to backward-compatible defaults.
- **Report fee matrix summary** — `per_cycle` multiplier collapses to 1 in Annual view; full program view keeps `× duration` behavior unchanged.

## [3.15.43] - 2026-04-16

### Added
- **`kdc_qtap_finance_net_amount_for_payment( $payment, &$credit_budget )`** helper in `kdc-qtap-finance-helper-functions.php` — returns `{ gross, credit_applied, net }` for a payment row. Running `$credit_budget` (passed by reference) lets multi-term carts share the same credit pool without double-application. No DB writes — pure projection.
- **`KDC_qTap_Finance_Payment::trickle_excess_from_payment()`** — public wrapper around `trickle_excess_forward()` so callers that already created their own transaction row (offline verify) can still trigger the downstream trickle + credit path.
- **`kdc_qtap_finance.php` — legacy overpayment migration** (`migrate_reclaim_legacy_overpayments_3_15_43()`) — scans all regular payments on upgrade, caps any `amount_paid > amount_due` at `amount_due`, moves the surplus into user credit, writes a `credit_backfill` transaction row per affected payment. Gated once by `kdc_qtap_finance_legacy_overpayment_backfill_done` option.

### Changed
- **Cart builders apply projected credit** — `ajax_create_fee_order()`, `ajax_create_multi_fee_order()`, and `build_term_cart()` in `trait-kdc-qtap-finance-wc-frontend-ajax.php` now call `kdc_qtap_finance_net_amount_for_payment()` with a shared running budget. Line-item amount is net of credit. Label gets "— ₹X applied from credit" suffix on lines where credit was applied. Fully-covered lines are skipped from the WC cart (finalised via `record_transaction()` at order completion).
- **`process_completed_order()`** in `trait-kdc-qtap-finance-wc-status.php` — replaces manual `Payment_Transaction::create()` + `Payment::update()` with a single `Payment::record_transaction()` call. This engages auto-credit consumption, overflow trickle to the next pending regular term, and final-excess parking as user credit for online payments. Uses the line-item subtotal (which already reflects any projected credit) as the transaction amount.
- **`Payment_Transaction::verify()`** now also engages trickle + credit: before committing the amount it consumes available credit for the remaining balance, caps `amount_paid` at `amount_due`, and calls `trickle_excess_from_payment()` with any overflow. Matches the behaviour of online checkout and admin Record Payment.
- **`Payment::record_transaction()` signature** extended with an optional `$extras` array — `payment_method_title`, `payment_date`, `transaction_date` — so callers can preserve the display title and the actual payment date on the transaction row without losing the trickle/credit behaviour.

### Fixed
- **WC order completion silently bypassed trickle + credit** — online payments never applied parked credit nor trickled overpayment to the next term. Now routed through the canonical `record_transaction()` path.
- **Cart amount didn't reflect user credit** — students with parked credit still saw the full term amount in the WooCommerce cart even though credit was available. Now deducted upfront.

## [3.15.42] - 2026-04-16

### Changed
- **Fees block restricts year dropdown to enrolled years** — `render_block()` now reads `KDC_qTap_Finance_Enrollment::get_all( $user_id )` and intersects the institute's `academic_years` with the user's enrolled keys. Only the intersection is passed to the dropdown (and to `academicYears` in the JS localisation).
- If the current default year isn't in the user's enrolled set, the block picks the user's enrollment with the latest end-year (same heuristic as `kdc_qtap_finance_get_latest_enrollment`).
- Staff with `kdc_qtap_finance_can_view_other_users_fees()` retain the **full** year list — they drive the user selector and need to view any user's historical data.

### Added
- **"No active enrollments" notice** — when `get_all()` is empty, `render_block()` short-circuits and returns a friendly card with the admin-email mailto link labelled "Contact Admin". Avoids showing an empty year dropdown.

## [3.15.41] - 2026-04-16

### Changed
- **Receipts badge + filter icons/colors aligned** — FEE badge now uses `coins` icon + green (`#00632a`/`#e7f5ea`); POS badge uses `shopping-cart` icon + purple (`#7b2cbf`/`#f3e8ff`). Both match the Source filter chips exactly.

## [3.15.40] - 2026-04-16

### Changed
- Removed the **Re-run backfill** button + JS handler from the Receipts tab header. The AJAX endpoint `kdc_qtap_finance_backfill_order_enrollment` remains for programmatic calls.

## [3.15.39] - 2026-04-16

### Changed
- **FEE source filter icon** — replaced `receipt` (which renders a `$` line through the icon) with `coins` (currency-neutral). Per icon policy: never any icon that renders a currency symbol.
- **Receipts filter panel redesigned** — replaced the always-expanded two-row checkbox panel with a native `<details>/<summary>` collapsible. Collapsed state shows a compact single-line summary: *"Status: Completed, Processing · Source: FEE, POS, Other"*. Click the chevron to expand and edit checkboxes. Zero JS for the toggle (native HTML). New CSS in `.kdc-qtap-finance-receipts-filters` with triangle rotation, separator, and consistent spacing.

## [3.15.38] - 2026-04-16

### Changed
- **`kdc_qtap_finance_lucide()` now delegates to parent** — calls `kdc_qtap_lucide()` (kdc-qtap v2.7.3+) instead of maintaining its own 26-icon SVG map. The duplicate icon paths array has been removed from the Finance helper file. All existing call sites (`kdc_qtap_finance_lucide('receipt', ...)` etc.) continue to work unchanged via the thin wrapper. Falls back to empty string if parent is too old (pre-2.7.3).

## [3.15.37] - 2026-04-16

### Added
- **Receipts tab — Source filter** (FEE / POS / Other) — second filter row below Status in a unified panel with `border-top` separator. Each source has a Lucide icon (receipt/green, shopping-cart/purple, package/grey). All three pre-selected on first load; unchecking auto-fires AJAX reload (opacity overlay while loading). Post-query filtering applies OR within sources, AND with the Status filter.
- **`KDC_qTap_Finance_Block_Editor::detect_pos_order( $order )`** static helper — extracted the inline `stripos` + `_woocommerce_pos_uuid` + meta-scan POS detection into a reusable method. Used by both the source filter and the per-row badge render.

### Changed
- Status + Source checkbox AJAX handlers now carry **both** filter sets in the URL params so toggling one doesn't drop the other.
- Reset button clears `source[]` in addition to `status[]` and `search`.

## [3.15.36] - 2026-04-16

### Changed
- **WC admin Status column CSS** — matched parent plugin's `kdc_qtap_order_by` column spec: `width: 2.2em !important; max-width: 2.2em !important; min-width: 2.2em !important; padding: 4px !important; text-align: center !important;`. Both icon columns now render at identical narrow width, consistent with the checkbox column.

## [3.15.35] - 2026-04-16

### Added
- **Default to Completed status** — WC Orders admin page now loads with `?status=wc-completed` when no status filter is set, so staff see completed orders first
- **Order ID link in Student column** — clickable `#order_number` prefix before payee name links to the order edit screen

## [3.15.34] - 2026-04-16

### Added
- **WC admin Orders table — Student column** — new column between Status and Order showing payee name in bold with student name, academic year, and grade/division as gray sub-text. Skips duplicating student name if same as payee.

### Changed
- **Status icon column width** — changed from 48px to 2.5em to match checkbox column width
- **Refactored column render callbacks** — consolidated HPOS and legacy column rendering into a single dispatch method

## [3.15.33] - 2026-04-16

### Fixed
- **WC admin Orders Receipt # column regenerated the PDF on each click** — replaced the `admin-ajax.php?action=generate_wpo_wcpdf` URL with the WC PDF public endpoint via `WPO_WCPDF()->endpoint->get_document_link( $order, 'receipt' )` (falls back to `'invoice'`). Clicking now opens the *existing* generated PDF rather than triggering a regeneration. When no document link is available (plugin inactive, no document generated), the receipt number is rendered as plain text — same graceful fallback as before.

## [3.15.32] - 2026-04-16

### Changed
- **Record Payment — Payment Mode defaults to NEFT** — `<option>` loop now calls `selected( 'neft', $method_key )` so the NEFT option is pre-checked on modal open. Falls back to `— Select Mode —` only if NEFT is not in the configured payment methods list.
- **Record Payment — field label** `Reference / Receipt #` → `UTR / Transaction ID`.

### Fixed
- **Edit Transaction modal didn't pre-select the saved payment method** — the edit link renders `data-payment-method="…"` but the JS was reading `$btn.data('payment-mode')` (legacy key). Updated both [kdc-qtap-finance-user-profile-payments.js](kdc-qtap-finance/assets/js/kdc-qtap-finance-user-profile-payments.js) and [kdc-qtap-finance-user-profile.js](kdc-qtap-finance/assets/js/kdc-qtap-finance-user-profile.js) to read `data-payment-method` with `data-payment-mode` as a safety fallback.

## [3.15.31] - 2026-04-16

### Fixed
- **Report group-tab footer totals empty** — `footerCallback` was only updating the original `<tfoot>` inside the source table. With `scrollX: true`, DataTables clones the tfoot into `.dataTables_scrollFootInner` and shows that clone while hiding the original, so the computed totals never reached the screen. Now writes to **both** the original (`api.table().footer()`) and the cloned scrollFoot — totals now render in every group tab. Summary tab was unaffected (no scrollX cloning of tfoot there).

## [3.15.30] - 2026-04-16

### Added
- **WC admin Orders table — Status icon column** — replaces the default text status column with a circular icon badge using the same icon mapping as the staff console. Positioned between `By` (parent plugin) and `Order` columns. Default fallback for unknown statuses.
- **WC admin Orders table — Receipt # column** — clickable receipt number that opens the WC PDF Invoices receipt/invoice in a new tab. Positioned after `Actions` and before `Transaction ID`. Falls back to plain text if WC PDF Invoices plugin is not active.
- **`kdc_qtap_finance_render_order_status()` helper** — single source of truth for rendering WC order status badges across the plugin (admin columns, staff console, user profile, receipts, emails). Accepts size/wrapper_size/show_label/tooltip/class args; handles `wc-` prefix and unknown status fallback.

### Changed
- **Staff console Receipts tab** — refactored inline status badge HTML to use the new `kdc_qtap_finance_render_order_status()` helper.
- **User profile orders list** — now displays status icons (was text-only) via the new helper.

## [3.15.29] - 2026-04-16

### Changed
- **Report group tabs default-sort by Student name** — `initDataTable()` resolves the `name` column's index dynamically (optional ID/Division columns shift positions) and passes that index to DataTables' `order` option. Previously the hard-coded `[1, 'asc']` could land on a fees column when ID was hidden.
- **Term Split toggle pre-checked on load** — the checkbox HTML now carries `checked`; staff opt-out by unchecking (matches the new default-enabled behaviour).

## [3.15.28] - 2026-04-16

### Added
- **Excess trickles forward** in `KDC_qTap_Finance_Payment::record_transaction()` — when an `amount_paid` would exceed `amount_due`, the overflow is no longer silently capped & lost. New `trickle_excess_forward()` finds the next pending regular payment for the same user/year (custom and user_fee buckets are skipped) via `find_next_pending_regular()` and recursively records the overflow against it. Each downstream payment writes its own `transaction` row with a "Excess X carried forward from payment #N" note.
- **User credit (parked overflow)** — when no further pending regular payments remain, the leftover is stored in `kdc_qtap_finance_credit` user meta. New helpers in [kdc-qtap-finance-helper-functions.php](kdc-qtap-finance/includes/kdc-qtap-finance-helper-functions.php):
  - `kdc_qtap_finance_get_user_credit( $user_id )` — read balance.
  - `kdc_qtap_finance_add_user_credit( $user_id, $amount )` — append.
  - `kdc_qtap_finance_consume_user_credit( $user_id, $amount )` — debit, returns amount used.
- **Credit auto-applied to new transactions** — at the start of `record_transaction()` (regular bucket only, when method is not already `credit`), any remaining balance after `$amount` is consumed from the user's credit. A separate `credit` transaction record is written so the audit trail shows: paid via UPI ₹X + ₹Y from credit. The combined amount feeds into the same overflow / trickle path so cascading still works.
- **User profile fee summary** now shows a green **Credit** pill (when balance > 0) right after Balance, with tooltip explaining auto-application.

### Changed
- Surfaced credit in [trait-kdc-qtap-finance-user-meta-rendering.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-rendering.php) inside `.kdc-enrollment-card__summary`.

## [3.15.27] - 2026-04-16

### Fixed
- **XLSX export ignored Term Split** — group sheets and the Summary sheet now call `redistributeTermBalance()` on their rows (when `#kdc-report-term-splt` is checked) before serialising with SheetJS, so exported values match the on-screen split (Collection + Received + per-term Balance).
- **Footer totals hidden in Staff Console report** — DataTables' `.dataTables_scrollFoot` wrapper was being hidden/zero-sized by the staff-console overlay (which only set width on scroll/head/body). Added `.dataTables_scrollFoot` + inner to the width-reset list and force `display: block/table-footer-group` via explicit `!important` overrides in both the staff-console CSS and the standalone report CSS.

### Reverted
- **"Via" column in WC admin Orders table** added in 3.15.26 — removed from kdc-qtap-finance. Ownership moved to kdc-qtap-mobile (which will handle WhatsApp `created_via` detection in addition to POS/checkout/admin). Avoids duplicate columns when both plugins ran the hook.

## [3.15.26] - 2026-04-16

### Added
- **"Via" column in WC admin Orders table** (HPOS + legacy) — narrow column (checkbox-width, 2.5em) inserted right after the checkbox column in both the HPOS orders list and the legacy shop_order posts list. Renders a dashicon keyed off `$order->get_created_via()` with the full value surfaced via tooltip. POS detection uses the same broad rules as the Receipts badge (`created_via` stripos + `_woocommerce_pos_uuid` meta + regex scan of all meta keys). Useful for diagnosing which `created_via` token a given POS plugin actually writes so staff can verify badge parity. Registered via:
  - `manage_woocommerce_page_wc-orders_columns` + `manage_woocommerce_page_wc-orders_custom_column` (HPOS)
  - `manage_edit-shop_order_columns` + `manage_shop_order_posts_custom_column` (legacy CPT)
  - Admin CSS scoped to the orders screen only via `get_current_screen()` gate.

## [3.15.25] - 2026-04-16

### Fixed
- **POS badge: `_woocommerce_pos_uuid` now authoritative** — explicit direct check for this meta key. Fallback regex relaxed from `/(^_?(wc)?pos(_|$))|cashier/i` to `/(?:^|_)(?:wc)?pos(?:_|$)/` so it also catches long keys like `_woocommerce_pos_uuid` where `pos` sits between underscores mid-key.

## [3.15.24] - 2026-04-16

### Changed
- **Term Split now also redistributes Collection and Balance** — previously only Received columns were split. Now `redistributeTermBalance()` additionally splits each row's `_c` across terms using the same canonical mode-based cap, and recomputes per-term `_b = max(0, newC - newR)`. For annual-cycle students, Term Collection no longer dumps the full year into term 1; it mirrors the matrix plan's per-term splits (e.g., 1,31,300 / 1,30,800), matching what staff expect when the toggle is on.

## [3.15.23] - 2026-04-16

### Fixed
- **POS badge still missed wcpos orders in production** — `created_via` check alone wasn't enough since some POS plugins don't set it. Detection now scans **all order meta keys** via a single regex `/(^_?(wc)?pos(_|$))|cashier/i` and flags the order when any matched key has a truthy scalar value. Covers wcpos `_pos`, woocommerce-pos `_wcpos_*`, Actuality POS `_pos_order` / `_pos_cashier`, and similar variants.

## [3.15.22] - 2026-04-16

### Fixed
- **Term Split for annual-cycle students** — `redistributeTermBalance()` no longer relies on the row's own `_c` as a cap (which would equal the full annual amount for annual-cycle students, trapping their received in term 1). Now derives a canonical per-term cap via the **mode of non-zero `_c` values across all rows in the group** — the most common value is the matrix-plan amount. Annual-cycle payers' received amount spreads across terms matching the matrix plan, with the overflow into later terms rendered as a grey advance (`+amount`).

### Changed
- **POS badge detection broadened** — `$is_pos_order` now matches any `created_via` string containing `pos` (case-insensitive via `stripos`) and also checks `_pos` / `_pos_order` order meta, covering WooCommerce POS plugin variants that set different `created_via` tokens.

## [3.15.21] - 2026-04-15

### Added
- **POS badge in Receipts tab** — amber pill with a coins icon next to the order number when `$order->get_created_via()` returns `woocommerce-pos` or `wcpos`; visually distinguishes in-store POS sales from online checkout orders. Tooltip: *"Created via WooCommerce POS"*.

## [3.15.20] - 2026-04-15

### Changed
- **Checkbox ID** `#kdc-report-term-balance` → `#kdc-report-term-splt` (matches the renamed "Term Split" label). JS selectors updated in `kdc-qtap-finance-report.js`.

## [3.15.19] - 2026-04-15

### Changed
- **Toggle renamed** `Term Balance` → `Term Split` with new tooltip: *"Split full-term payments across terms as an advance (shown in grey). Balance columns always show irrespective of this toggle."* Avoids the confusion where users read "Term Balance" as a column-visibility toggle rather than a reporting-split toggle.
- **Balance columns always render** — reverted the `group_balance` filter added in 3.15.18; Term Balance / Month Balance / per-group balance columns now always show in the Report tables regardless of the Term Split toggle state.

## [3.15.18] - 2026-04-15

### Fixed
- **Term Balance columns show regardless of toggle** — `getFilteredColumns()` now excludes `group_balance`-type columns (1st Term Balance, 2nd Term Balance, monthly balances, etc.) when `#kdc-report-term-balance` is unchecked. The toggle now controls both the advance-redistribution overlay and the visibility of the balance columns, consistent with how the checkbox is labeled.

## [3.15.17] - 2026-04-15

### Changed
- **Report DOM restructure** — `#kdc-report-container` is now rendered as a **sibling** of `.kdc-qtap-finance-report-block` instead of nested inside it. Year selector + controls stay at standard page width via the block wrapper; the container gets the `alignfull` class when the block is aligned full and breaks out to `100vw` independently. Clean separation of concerns — no more fighting a parent's `overflow-x:auto` or width caps.
- **Staff Console overlay CSS** simplified — applies the 100vw breakout directly to `#kdc-report-container` (now a sibling); removed the old `.kdc-report-tabs-body`-targeted rules that were a workaround for the nested structure.
- **Frontend report CSS** — removed the old `.kdc-qtap-finance-report-block.alignfull #kdc-report-container { width: 100% }` rule (no longer matches); replaced with `#kdc-report-container.alignfull` full-viewport breakout.

## [3.15.16] - 2026-04-15

### Added
- **`kdc_qtap_finance_get_latest_enrollment( $user_id )`** helper — returns the user's enrollment with the highest end-year (independent of institute's "current" year)
- **`KDC_qTap_Finance_WooCommerce::apply_enrollment_meta( $order, $enforce_floor = false )`** static helper — single source of truth for writing student name, academic year, grade, division onto a WC order. Shared by the live hook and the on-demand backfill button
- **`woocommerce_new_order` hook** — enrollment tagging now catches admin-created, POS, REST, and store-API orders (previously only front-end checkout via `woocommerce_checkout_order_created`)
- **Re-run backfill** button in Receipts tab header — re-scans all orders with `date_paid >= 2026-04-01` via new AJAX action `kdc_qtap_finance_backfill_order_enrollment` and applies enrollment meta to any without it; shows count on completion

### Changed
- `add_enrollment_meta_to_order()` now writes to the **fee-order meta keys** (`_kdc_qtap_finance_academic_year`, `_grade`, `_division`, `_student_name`) instead of only the parallel `_enrollment_*` keys, so Receipts tab shows Year · Grade for non-fee orders (Cash, shop, POS). Writes are idempotent — only fills a key when not already set, so fee orders' own values are never overwritten. Parallel `_enrollment_*` keys still written for PDF fallback back-compat.

## [3.15.15] - 2026-04-15

### Fixed
- **Order items modal — "Request failed" for non-fee orders** — removed the `_kdc_qtap_finance_is_fee_payment=yes` gate in `ajax_staff_order_items()` so the modal now loads line items for any WooCommerce order in the Receipts list (matches the relaxed query introduced in 3.15.14). Empty grade/year meta is trimmed so the header no longer shows stray separators.

## [3.15.14] - 2026-04-15

### Changed
- **Receipts tab** now loads **all** WooCommerce orders (no longer restricted to `_kdc_qtap_finance_is_fee_payment=yes`); finance-linked orders get a small green **FEE** pill next to the order number so staff can still spot them at a glance
- **Report layout inside Staff Console** — header, controls, AND the grade tabs nav now all stay within the normal tab content width; only `.kdc-report-tabs-body` (the actual data panes) breaks out to `100vw` via `calc(50% - 50vw)`

## [3.15.13] - 2026-04-14

### Added
- **Term Balance** toggle on the Report controls row — redistributes received overflow from earlier terms into later terms' received columns; advance-attributed amount rendered in dark grey italic (e.g., `1,30,800 +1,31,300`) so staff can see both the term's actual payment and the reporting attribution. Only applies to term/month received columns (skips Custom / User Fees)
- **Column footer totals** on all Report tables — `<tfoot>` row sums each numeric column from currently-visible rows; styled with monospace + tabular-nums; included in CSV export (`footer: true`) and in every XLSX sheet (group tabs + Summary) via a new `buildTotalsRow()` helper
- `total` i18n string for the footer label

### Changed
- **Report layout inside Staff Console** — header area (Academic Year, Download Excel, Reporting Breakup, toggles) now stays within the normal tab content width; only `#kdc-report-container` breaks out to `100vw` so the wide data tables expand edge-to-edge

## [3.15.12] - 2026-04-14

### Fixed
- **Report Balance columns formatting** — the new per-group `group_balance` columns (Term Balance, Custom Fees Balance, User Fees Balance, per-Month Balance) now match the rest: monospace currency font, tabular-nums, right-aligned

## [3.15.11] - 2026-04-14

### Fixed
- **Duplicate academic year** in WC order line item titles — `build_item_name()` now strips any `[YYYY-YYYY]` bracket from the fee category before the template appends the year at the end; affects all new orders created via checkout, offline form, admin record payment, staff console, and direct payment link (underlying Payment records unchanged)

### Changed
- **User profile WC Orders section** now loads **all** orders for the user (not only fee-payment orders); fee-payment orders get a small blue **FEE** badge next to the order # so staff can still distinguish at a glance

## [3.15.10] - 2026-04-14

### Added
- **Term / Month / Custom / User-fee balance columns** in the Report — each `Collection` + `Received` pair is followed by a `Balance` column (= max(0, collection − received)); aggregated rows recompute balance from summed collection/received to stay internally consistent
- HTML-entity decode on Staff Console order-items modal response (`₹` and other entities render natively instead of as `&#8377;`)

### Changed
- **Order-items modal redesigned** — wider (720px), CSS-grid header separates name/qty from total, meta rows now in a two-column `<dl>` with light background, slab breakdown moved to a bordered card with per-row period/amount split, tabular-numerics for currency alignment, thicker bold grand-total footer
- **Line item name for single-payment tenures/cycles** — `build_item_name()` now inspects the payment's items and uses `Full Tenure (range)` / `Full Cycle (range)` when the payment is homogeneous `per_tenure` / `per_cycle`, instead of the misleading `1st Term …` label; falls back to the existing slab-label logic for mixed / term / monthly payments

### Fixed
- **Report whitespace inside Staff Console** — removed the `margin-left: 0 !important` override that was cancelling the report block's `margin-left: calc(50% - 50vw)` breakout; the Report tab now truly fills the viewport when the block is `alignfull`

## [3.15.9] - 2026-04-14

### Added
- **Status icons** on Receipts — filter checkboxes and table cells now show Lucide status icons (completed → `check-circle-2`, processing → `loader`, on-hold → `pause-circle`, pending → `clock`, cancelled → `x-circle`, refunded → `undo-2`, failed → `alert-circle`, draft → `file-text`, POS statuses → `shopping-cart`/`coins`); table cells render icon-only with status label as tooltip (title attr)
- **Order items modal** — `list` icon button on each receipts row opens a modal that lazy-loads the order's line items via AJAX (`kdc_qtap_finance_staff_order_items`); shows item name, qty, total, per-slab breakdown, and order total; closes on overlay click, X button, or Esc
- **`kdc_qtap_finance_status_icon_info()`** helper — maps WC status slug to Lucide icon + color

### Changed
- **Receipts column order** — Status moved to 2nd-last position (between Receipt # and Actions) per layout request
- **Report full-width** strengthened further — added selectors for `.kdc-report-pane`, `.kdc-report-tabs-body`, `.kdc-report-tabs-nav` and forced inner DataTables to 100% width; grade tabs (Fees, Summary, etc.) now stretch edge-to-edge inside the Staff Console

## [3.15.8] - 2026-04-14

### Fixed
- **Report tab whitespace inside Staff Console** — strengthened CSS override to nullify the 1200px caps on `#kdc-report-container`, `.kdc-report-controls`, `.kdc-report-group`, DataTables wrappers, and inner report tables so all grade tabs (Fees, Overview, per-grade) stretch edge-to-edge

### Added
- Lucide icons: `loader`, `pause-circle`, `x-circle`, `undo-2`, `alert-circle`, `list`, `x`, `package`

## [3.15.7] - 2026-04-14

### Added
- **AJAX filter + pagination** on the Receipts tab — new `kdc-qtap-finance-staff-receipts-ajax.js` intercepts status checkbox toggles, filter form submit, and pagination clicks; fetches the current URL, swaps in the receipts section, and syncs `pushState` (Back/Forward works via `popstate`); fades the section during load and scrolls the updated table into view
- **Recent Fee Orders — Receipt # clickable to PDF** (parity with Receipts tab) when WCPDF is available; external-link Lucide indicator

### Changed
- **Recent Fee Orders eye icon** now opens the wp-admin order edit screen in a new tab (via `$order->get_edit_order_url()`, HPOS-aware) instead of the customer view-order page; visible only to users with `edit_shop_orders` / `manage_woocommerce`
- **Receipts eye icon** same change — wp-admin edit in new tab, capability-gated

## [3.15.6] - 2026-04-14

### Changed
- **Receipts pagination active** — `per_page` reduced from 50 to 20 so the existing pagination row appears when there are more than 20 records
- **Status filter auto-applies** on checkbox toggle (`onchange="this.form.submit()"`); Apply button now `<noscript>` fallback only
- **Report tab embedded full-width** in the Staff Console — passes `align => full` to the report block render and overrides its inner 1200px caps via scoped CSS so the year selector, controls, and report table expand to the full block width

## [3.15.5] - 2026-04-14

### Added
- **`kdc_qtap_finance_lucide()`** — inline Lucide SVG helper for frontend UI (16 icons: `graduation-cap`, `coins`, `clock`, `alert-triangle`, `users`, `receipt`, `shopping-cart`, `bar-chart-3`, `layout-dashboard`, `search`, `plus`, `plus-circle`, `eye`, `check-circle-2`, `arrow-left`, `external-link`, `file-text`); honors the icon-policy memory (Lucide on frontend)

### Changed
- **Staff Dashboard stat cards** now render Lucide SVGs instead of emojis (🎓 🪙 ⏳ ⚠️ → `graduation-cap` / `coins` / `clock` / `alert-triangle`)
- **Receipts tab status filter** now uses `wc_get_order_statuses()` so custom order statuses from other plugins are included; defaults to `completed + processing` on first load
- **Receipts table** improvements:
  - **Payment** column shows method title + UTR/`transaction_id` underneath (from `pay_utr` / `transaction_id` order meta)
  - **Customer** column appends `Year · Grade Division` line (from order meta, no labels)
  - **Receipt #** is clickable when WCPDF is available — opens receipt PDF in a new tab via `WPO_WCPDF()->endpoint->get_document_link()` (falls back to invoice)
  - Action buttons use Lucide (`eye` for View)
- **Menu:** POS opens in new tab; Report is now an internal tab that renders the existing report block inline (no separate page needed)

## [3.15.4] - 2026-04-14

### Added
- **Payee Name field** on Record Payment modal — optional input; when entered, updates `billing_first_name` on both the user and the new WC order; when empty, falls back to existing user `billing_first_name` meta, then to `first_name + last_name`
- **Internal menu** on the Staff Console — Overview, Receipts, POS (→ `/pos/`), Report (→ `/report/`); Receipts is a new full-width tab with paginated fee-order list + search
- **User Fee Product selector** on the Institute settings tab — exposes the v3.15.1 user-level WC product (`kdc_qtap_finance_wc_user_product_id`, SKU `QTAP-FEE-USER`) alongside the existing Grade product selector; both dropdowns now show SKU in brackets

### Changed
- **Staff Console fully fluid** — removed card wrapper (background, border, shadow, padding); block now matches its alignment width without an inner container box
- **WC product settings save handler** iterates both product ID fields with the same validate/delete logic

## [3.15.3] - 2026-04-14

### Changed
- **Staff Console UI made fluid** — removed the 1200px max-width cap; added `box-sizing: border-box` and `<h*>` font normalization so the console no longer inherits theme-specific heading styles (purple/serif); tables wrap in a horizontal-scroll container on narrow viewports
- **Year switcher** in dashboard header — dropdown populated from configured academic years; switches the year context for Enrolled / Overdue / Outstanding stats via `?year=` query param; preserves other query args; `<noscript>` fallback button
- **No `$` glyphs** — stat cards now use emoji (🎓 🪙 ⏳ ⚠️) instead of `dashicons-money-alt` which renders as a dollar sign in some themes
- **Recent Fee Orders table** tightened — date moved under order # as subtext, View turned into icon-only button, amount nowrap + right-aligned, header cells nowrap, smaller padding

## [3.15.2] - 2026-04-14

### Added
- **Staff Console dashboard** — landing view replaced with a proper dashboard: 4 stat cards (Enrolled Students, Today's Collections, Pending Verifications, Overdue Fees with outstanding amount), compact name/mobile/email search bar with Create New, two-column body showing Pending Verifications (actionable cards with Review buttons) and Recent Fee Orders (read-only with receipt numbers and customer drilldown links)
- **`GET /wp-json/kdc/v1/qtap/staff-search?q=…`** — searches users by `user_login`, `user_email`, `display_name`, `first_name`/`last_name` user meta, and mobile number; returns matched users for the Staff Console picker
- **Multi-match picker** — when search returns >1 user, the find panel now shows a clickable list instead of always taking the first match; smart prefill of name/email/mobile field on no-match based on input shape

### Fixed
- **Find user by name was failing** — old endpoint `/qtap/mobile/{identity}` only matched mobile/email; replaced with the new `/qtap/staff-search` endpoint that also covers first/last name meta

## [3.15.1] - 2026-04-14

### Added
- **Split fee products** — grade-level and user-level fees now use distinct hidden WC products with their own SKUs (`QTAP-FEE-GRADE`, `QTAP-FEE-USER`); ensures separation in receipts, invoices, and product reports
- **Migration to backfill flat enrollment meta** — writes per-year `kdc_qtap_finance_grade_{year}` and `kdc_qtap_finance_division_{year}` user meta from the serialized enrollments meta; enables indexed `meta_query` filtering on the WP admin users list

### Fixed
- **Staff Console login prompt** — now renders `wp_login_form()` inline instead of just a redirect link
- **`set_profile_user()` setter** added to `KDC_qTap_Finance_User_Meta` so frontend contexts (Staff Console block) can initialize modal context without poking a private property

## [3.15.0] - 2026-04-14

### Added
- **Staff Console frontend block** (`kdc-qtap/staff-console`) — accredited staff (roles granted API access via qTap settings) can now find/create users, assign enrollments, record offline payments, and view fee orders **without visiting wp-admin**; reuses the existing admin user profile UI, modals, and AJAX handlers as-is — no new forms, endpoints, or capabilities
- **Read-only WC Orders section** on user profile — shows fee order list with order #, date, amount, status, and WCPDF receipt number; "View" link opens customer `view-order` endpoint (never wp-admin order editor); rendered on both admin user profile and the new Staff Console block; skipped when WooCommerce is inactive
- **`enqueue_profile_assets_for_user()`** public method on `KDC_qTap_Finance_User_Meta` — allows frontend blocks to enqueue the full admin profile asset stack (CSS + all JS modules + localized strings) for a given user

### Changed
- **Staff access model** — now uses the parent plugin's existing `kdc_qtap_can_access_rest_api()` permission for the new Staff Console block; admin configures which roles have access via qTap → Common Settings → API Settings → `rest_api_roles`

## [3.14.10] - 2026-04-14

### Added
- **WC order number appended to admin-recorded transaction notes** — when an admin records a payment with "Generate WooCommerce Order" checked, the resulting order's number (e.g., `WC Order #12345`) is now appended to both the payment notes and the transaction notes for easy cross-reference from the user profile view

## [3.14.9] - 2026-04-14

### Added
- **Offline/admin WC orders now populate standard gateway meta keys** — `paywith_method` = payment method title (from `kdc-qtap-finance-payment-method` form field), `pay_utr` and `transaction_id` = reference/UTR (from `kdc-qtap-finance-payment-reference` form field), falling back to `TXN-{id}` when empty; enables WCPDF receipts and downstream consumers to read offline payment info via the same keys online gateways use

### Fixed
- **Admin-recorded transactions missing `payment_method_title`** — `ajax_record_payment()` now passes the resolved method label to `Transaction::create()`, so transaction rows store the human-readable title (e.g., "Bank Transfer") not just the slug (`bank_transfer`)

### Changed
- **WCPDF `pdf_receipt_payment_details()` simplified** — now prefers order meta (`paywith_method`, `pay_utr`, `transaction_id`) over transaction data since all flows populate them consistently

## [3.14.8] - 2026-04-14

### Added
- **Fee breakup on offline/admin WC orders** — `create_fee_order()` now attaches the full payment_item breakdown (slab + period @ amount) to the line item, matching what `create_term_order()` already did for online checkout; offline payment submissions and admin-recorded transactions now show term/cycle/tenure/monthly fee breakdowns on the linked WC order and PDF receipts/invoices

## [3.14.7] - 2026-04-14

### Fixed
- **Admin record payment wasn't creating WC order** — `create_fee_order()` was being called AFTER the payment's `amount_paid` was already bumped, so the helper's balance-due check rejected the amount (`amount_exceeds_balance`) and the WP_Error was silently dropped; now the WC order is created first (before the payment update) so the balance check passes

## [3.14.6] - 2026-04-14

### Added
- **Flat enrollment meta backfill migration** — backfills `kdc_qtap_finance_grade_{year}` and `kdc_qtap_finance_division_{year}` user meta from the serialized `kdc_qtap_finance_enrollments` meta; runs once on upgrade to enable native indexed `meta_query` filtering on the WP admin users list

## [3.14.5] - 2026-04-13

### Added
- **"Generate WooCommerce Order" checkbox on offline payment approval/rejection** — admins can now approve/reject historic offline payments without keeping a WC order; when unchecked, the linked WC order is trashed and unlinked from the transaction; only shown when the transaction has a linked WC order

## [3.14.4] - 2026-04-13

### Fixed
- **Prior-year dues block not sticking** — `updateStrictPayButtons()` was re-enabling the first term's pay button after our disable code ran, bypassing the cross-year block; now early-returns when `prior_year_dues` is true (v3.14.3 regression fix)

## [3.14.3] - 2026-04-13

### Changed
- **Prior-year dues always block payment** — when a user has unpaid fees from a past academic year, Pay buttons are now disabled regardless of the `payment_order_regular` rule (previously, `warn` mode showed a banner but still allowed payment); within-year order still respects the rule
- **Server-side cross-year enforcement** — `validate_sequential_payment()` now rejects submissions with prior-year dues for both `strict` and `warn` rules (only `none` bypasses)
- **Individual fee pay button** now also disabled on prior-year dues (previously only term-level pay button was disabled)

## [3.14.2] - 2026-04-13

### Added
- **"Generate WooCommerce Order" checkbox** — admin Record Payment modal now has an opt-out checkbox (checked by default) to skip WC order creation for transaction-only recording (e.g., reconciling historical payments); only shown when WooCommerce is active

## [3.14.1] - 2026-04-13

### Added
- **WCPDF receipt/invoice payment details** — shows "Paid With" method and "UTR / Ref" below the payment method on PDF receipts/invoices via `wpo_wcpdf_after_order_data` hook; online payments read `paywith_method` and `pay_utr` from order meta, offline/admin payments use transaction data, falls back to `TXN-{id}` if no UTR
- **WCPDF receipt number integration** — `get_wcpdf_receipt_number()` helper reads `_wcpdf_receipt_number` / `_wcpdf_invoice_number` from WC order meta on-the-fly (no extra DB column)
- **Receipt number in frontend** — shown next to status badge on fee cards
- **Receipt number in admin** — shown below payment method in user profile transaction rows
- **Receipt number in exports** — new "Receipt Number" column in transaction CSV/Excel exports
- **`{{receipt_number}}` template variable** — available in WhatsApp/email notification templates

## [3.14.0] - 2026-04-13

### Added
- **WooCommerce order for offline payments** — when WooCommerce is active, submitting an offline payment form now creates a WC order with `on-hold` status; admin approval sets it to `completed`, rejection sets it to `cancelled`
- **WooCommerce order for admin-recorded transactions** — recording a payment from wp-admin user profile now creates a WC order with `completed` status immediately
- **Order notes** — offline/admin orders include payment method, reference/UTR, and verification action notes
- **Order metadata** — offline orders tagged with `_kdc_qtap_finance_offline_payment`, admin orders with `_kdc_qtap_finance_admin_recorded` for identification

### Changed
- **`create_fee_order()`** now accepts optional `$initial_status` parameter (default `'pending'`, backwards-compatible)
- **`link_order_to_payment()`** visibility changed from `private` to `public` to support cross-trait order linking
- **`Transaction::update()`** now supports `wc_order_id` field for backfilling order links
- **Refund/cancel guard** — `process_refunded_order()` skips reversal for offline/admin-recorded orders to prevent double-debit

## [3.13.9] - 2026-04-03

### Added
- **Force Reset & Sync** button — allows admins to cancel a stuck/stale sync and restart from scratch; cancels pending Action Scheduler batches
- **Auto-stale detection** — syncs running longer than 5 minutes are automatically flagged as stale with a clear warning message
- **Sync logs panel** — real-time monospace log display showing started time, deleted count, enrolled total, processed, synced, errors, and elapsed time
- **Progress bar** — visual percentage bar during sync processing
- **Auto-check on page load** — Fee Matrix page automatically detects running or stale syncs and shows status/logs

### Changed
- **Sync error handling** — "already in progress" error now shows Force Reset button; AJAX failures also show Force Reset for recovery

## [3.13.8] - 2026-04-03

### Changed
- **Sync Payments — bulk SQL approach** — replaces ~15,000 individual cascade DELETEs with 2 bulk SQL queries (unpaid items + payments); regeneration batches reduced to 10 users/batch with only INSERTs needed
- **Concurrent sync prevention** — blocks duplicate sync jobs; returns error if sync already in progress
- **Progress percentage** — polling UI shows "Processing X of Y users... (Z%)"
- **Due dates sync** moved from background batch to AJAX handler (lightweight bulk UPDATEs)

## [3.13.7] - 2026-04-03

### Changed
- **Sync Payments via background processing** — uses Action Scheduler (WooCommerce) or WP Cron fallback; processes users in batches of 50 to avoid timeouts and memory exhaustion
- **Sync Payments UI** — button now shows real-time progress via polling (e.g., "Processing 150 of 800 users..."), with final success/error count

### Fixed
- **Sync Payments JS** — wrong year selector ID (`matrix-year` → `year-select`) and wrong localized variable (`kdcQtapFinanceAdmin` → `kdcQtapFinance`)

## [3.13.6] - 2026-04-03

### Added
- **Sync Payments button** — dedicated button on Fee Matrix page to regenerate unpaid payments from current matrix amounts; paid payments are never affected
- **AJAX sync handler** — `kdc_qtap_finance_sync_payments` action with confirmation dialog, spinner, and success/error feedback

### Changed
- **Fee matrix save no longer auto-syncs payment amounts** — only due dates are synced automatically on matrix save (lightweight); amount sync is now manual via the Sync Payments button

## [3.13.5] - 2026-04-02

### Changed
- **Users table replan** — ID column is now the primary column (clickable to edit, with row actions); separate Last Name and First Name columns replace the Display Name column
- **Meta-based sorting** — Last Name and First Name columns now sort alphabetically via `pre_get_users` hook with `meta_key` + `meta_value` ordering

## [3.13.4] - 2026-04-02

### Added
- **Bulk "Associate Users"** — select multiple users on wp-admin Users page, apply bulk action to link them all as a family group; merges existing associations
- **"Enrollment" column** — shows Grade Division [Year] per enrollment on Users list table, sortable
- **"Associated" column** — shows clickable linked user names on Users list table, sortable
- **Custom row actions** — Name column shows Edit, View, Delete, Send password reset on hover

### Changed
- **Hidden columns** — removed Username and Posts columns from Users list table for cleaner view

## [3.13.3] - 2026-04-02

### Changed
- **Switch endpoint slug** — changed from `switch-student` to `switch` so label changes don't break the URL
- **Removed switch-back link** — removed from associated users bar (both WC dashboard and standalone fees page)

## [3.13.2] - 2026-04-02

### Removed
- **FAB moved to kdc-qtap-mobile** — FAB trait, JS, CSS, and admin setting removed from finance plugin; single source of truth is now `kdc_qtap_mobile_enable_fab_menu` option in the mobile plugin

## [3.13.1] - 2026-04-02

### Added
- **qTap FAB Menu** — draggable floating action button on all pages for logged-in users; position persists via localStorage; viewport-aware menu opens up/down based on screen position; dynamic items: Fees, Mobile, Dashboard, Switch Student (sub-menu), Logout
- **FAB admin setting** — "qTap Menu (FAB)" checkbox in Finance admin Institute tab to enable/disable the FAB
- **WC My Account "Switch Student" endpoint** — new sidebar menu item + `/my-account/switch-student/` page showing associated users bar; only visible when user has associations

### Fixed
- **Associated users bar not showing on WC pages** — WooCommerce action hooks pass endpoint value as first arg, which overwrote the `$echo` parameter making it falsy; method now always echoes directly

## [3.13.0] - 2026-04-02

### Added
- **Associated Users** — bidirectional family group linking between WordPress users; admin manages associations on user profile page with searchable multi-select chip UI
- **Bidirectional sync** — adding user A to user B's associations automatically adds B to A's list (and all group members see each other); removal also syncs
- **Frontend user switching** — associated users bar on WooCommerce My Account dashboard, fees endpoint, and standalone fees page; instant login switch between family members
- **Switch back** — transient-based (1hr TTL) switch-back link appears after switching, redirects to My Account
- **Thank You page integration** — associated users bar on WooCommerce order confirmation when the order contains the Finance Fee product
- **Non-WooCommerce fallback** — associated users bar renders at the top of the fees block when WooCommerce is not active
- **Configurable label** — "Associated Users" field uses the plugin's customizable student label (e.g., "Associated Students")

### Changed
- **Enrollment card layout** — year (bold) + grade inline on one row; due date and terms as separate lines; amount + status badge stacked on right
- **Enrollment card details** — shows term count, outstanding terms pending, next due date with calendar icon
- **Enrollment card badge** — split into amount (bold) + status label (badge)
- **Pay button trickle logic** — individual Pay button stays disabled until all previous terms are actually paid
- **Duplicate "Pay Selected" bar** — fixed sticky payment bar duplicating on year change
- **Dashboard cards link** — use standalone `/fees/?year=` page instead of WooCommerce endpoint URL

## [3.12.3] - 2026-04-01

### Changed
- **Enrollment card layout** — year (bold) + grade inline on one row; due date and terms as separate lines below; amount + status badge stacked on the right column
- **Enrollment card term details** — shows term count, outstanding terms pending, and next due date with calendar icon
- **Enrollment card badge** — split into amount (bold) + status label (badge); larger badge size

## [3.12.2] - 2026-04-01

### Added
- **My Account enrollment summary cards** — WooCommerce My Account dashboard now shows enrollment cards per academic year with grade, division, and payment status badge (Paid/due/overdue) instead of the full fees block
- **`get_user_year_summaries()`** — new Payment class method for efficient per-year payment totals in a single DB query

### Changed
- Dashboard enrollment cards link to standalone `/fees/?year=` page instead of WooCommerce endpoint URL

## [3.12.1] - 2026-03-31

### Fixed
- **Pay button trickle logic** — individual Pay button now stays disabled until all previous terms are actually paid (not just when checkbox is enabled); trickle still enables checkboxes for batch selection via sticky footer
- **Duplicate "Pay Selected" bar** — fixed sticky payment bar duplicating on year change by removing stale bar before inserting new one

## [3.12.0] - 2026-03-31

### Added
- **WooCommerce My Account Fees tab** — dedicated `fees` endpoint in My Account sidebar using `EP_PAGES` (matching mobile plugin pattern)
- **URL `?year=` parameter** — fees block reads `?year=2026-2027` or `?year=2026` (smart resolution via June cutoff) to deep-link academic years
- **Year persistence** — selected academic year saved to `sessionStorage`; persists across page navigation
- **Collapsible fee cards** — term, custom, and user fee cards collapsed by default; click header to expand; chevron indicator with `aria-expanded`
- **Card header redesign** — two-row layout: term name + range (inline) + status badge + chevron on row 1; due date with calendar icon on row 2; JS parses `term_label` into structured parts
- **Aggregate term status** — PHP computes `term_status` (paid/partial/overdue/pending) from line items; displayed as badge in card header
- **Pay Together hides individual pay** — checking "Pay Together" hides the card's pay-actions; unchecking or disabled checkbox shows them again

### Changed
- **Typography: CSS custom properties** — override `--kdc-qtap-font-size-xs/sm/base/lg` on fees block container with absolute px values (12/14/16/18px); all parent components resolve correctly regardless of theme root
- **Card layout: flex column** — `.kdc-qtap-card` is now `display:flex; flex-direction:column; min-width:0` preventing child overflow; inline form uses negative margin for full-bleed within card padding
- **Mobile responsive** — line items table converts to 2x2 grid (label/sublabel left, amount/status right); footer checkbox full-width with offline icon + pay button sharing second row; `.kdc-qtap-finance-pay-actions` wrapper keeps icon and button together
- **No inline styles** — all remaining inline `style=` replaced with CSS classes; only dynamic values (progress bar width, form display:none) remain inline

### Fixed
- **`/fees/` page 404** — removed redundant `query_vars` filter; endpoint uses `EP_PAGES` only (not `EP_ROOT`)
- **Trickle disables Pay button** — term Pay button disabled alongside its checkbox; custom/user fee buttons and offline icon independent
- **Card header border** — removed `border-bottom`, `margin-bottom`, `padding-bottom` from card headers in fees block

## [3.11.19] - 2026-03-31

### Changed
- **Card header label parsing** — JS now parses `term_label` into structured parts (term name, academic year, range) when PHP fields are unavailable; academic year shown in normal weight via `.kdc-qtap-finance-card-header__year` CSS class
- **Inline style cleanup** — replaced last inline `style="margin-bottom:20px"` on redirect message with `.kdc-qtap-finance-redirect-message` CSS class

## [3.11.18] - 2026-03-31

### Changed
- **Card header redesign** — two-row layout: row 1 has term name (bold) + aggregate status badge (pending/paid/partial/overdue) + collapse chevron; row 2 has month range + due date with calendar icon; due date moved from body to header
- **PHP term response** — added `term_name`, `term_range`, and `term_status` fields for structured header rendering
- **Mobile footer** — at ≤480px, "Pay Together" checkbox takes full width; offline icon stays inline with Pay button

## [3.11.17] - 2026-03-31

### Added
- **Year selection persistence** — selected academic year saved to `sessionStorage`; persists across page navigation within the session; priority: URL `?year=` → sessionStorage → default

### Fixed
- **Trickle disables Pay button** — when a regular term's "Pay Together" checkbox is disabled by trickle logic, the corresponding "Pay" button is now also disabled (opacity 0.5); re-enables when the previous term is selected
- **Custom/user fee buttons independent** — custom and user fee pay buttons are never affected by the trickle logic (only regular term buttons)
- **Offline icon always enabled** — offline payment icon stays active regardless of trickle state

## [3.11.16] - 2026-03-31

### Fixed
- **Font-size root cause fix** — override `--kdc-qtap-font-size-xs/sm/base/lg` CSS custom properties with absolute `px` values on the fees block container; parent components (buttons, badges, inputs, card titles) use these vars with `rem` which bypasses any container font-size — block-scoped vars ensure 12/14/16/18px regardless of theme root

## [3.11.15] - 2026-03-31

### Fixed
- **Typography simplification** — removed all fractional em/rem font-size values that caused compounding; children now inherit container baseline naturally; form fields (input/select/textarea/label) use `font-size: inherit` to override theme/WooCommerce smaller sizes; only hints and summary totals use explicit px

## [3.11.14] - 2026-03-31

### Changed
- **Fee card header cleanup** — removed `border-bottom`, `margin-bottom`, and `padding-bottom` from card headers inside fees block; removed amount display from card headers for cleaner collapsed view

## [3.11.13] - 2026-03-31

### Fixed
- **`/fees/` page conflict** — removed redundant `query_vars` filter that was causing WordPress to treat `fees` as a global query variable; `add_rewrite_endpoint` with `EP_PAGES` already handles the query var internally (matching the mobile plugin pattern)

## [3.11.12] - 2026-03-31

### Added
- **Collapsible fee cards** — term, custom, and user fee cards are collapsed by default showing header + footer; click header to expand and reveal line items; chevron indicator rotates on toggle with `aria-expanded` accessibility

### Fixed
- **`/fees/` page 404** — WooCommerce My Account endpoint renamed from `fees` to `qtap-fees` to avoid rewrite collision with the existing `/fees/` WordPress page; My Account tab now at `my-account/qtap-fees/`

## [3.11.11] - 2026-03-31

### Added
- **URL `?year=` parameter** — fees block reads `?year=2026-2027` from URL to pre-select academic year; smart resolution for short form (`?year=2026` resolves based on current month using June cutoff); validates against configured years

### Changed
- **Typography cascade** — font sizes now use `em` (not `rem`) relative to container baseline; cascade: qTap custom variable → theme default → 14px minimum floor via `max(var(--kdc-qtap-font-size-base, 1em), 14px)`

## [3.11.10] - 2026-03-31

### Added
- **WooCommerce My Account Fees tab** — added a dedicated "Fees" endpoint and menu item in the WooCommerce My Account sidebar; renders the full fees block at `my-account/fees/` with dynamic label from custom settings

## [3.11.9] - 2026-03-31

### Fixed
- **Font size legibility** — increased all font sizes across the fees block; minimum is now 12px, main content text is 14px+, ensuring readability on mobile and small screens

## [3.11.8] - 2026-03-31

### Added
- **Payment Order Rules** — per-type configurable enforcement (strict/warn/none) for regular, custom, and user fees with admin settings
- **Cross-year sequential enforcement** — unpaid fees from prior academic years block or warn current year payments based on rule type
- **Multi-payment selection** — "Pay Together" checkboxes with trickle logic (strict fees sequential, warn/none always enabled) and sticky "Pay Selected" bar
- **Offline bank icon** — inline bank dashicon link that opens the offline payment form on click; available on term, custom, and user fee cards

### Changed
- **Fees block typography rework** — established consistent rem-based type scale; replaced all inline `font-size`/`font-weight` with CSS classes (`.kdc-qtap-finance-term-line-items`, `.kdc-qtap-finance-detail-row`, `.kdc-qtap-finance-section-title`, `.kdc-qtap-finance-banner`, `.kdc-qtap-finance-form-header`)
- **CSS refactor** — removed all inline styles from JS; moved to dedicated CSS classes with BEM-like modifiers (`--muted`, `--bold`, `--warning`, `--error`)
- **Dynamic fee labels** — section titles and button text now use configurable `custom_labels` from settings instead of hardcoded "Fees"

### Fixed
- **Custom fee sync** — `sync_payments_on_fee_matrix_save()` now includes `_custom_slabs` so new custom fees are created for existing enrolled students
- **Term due date sync** — updating academic term due dates now propagates to all existing unpaid payments
- **Trickle checkbox scope** — only `strict`-type checkboxes follow sequential trickle; `warn`/`none` types are always enabled
- **Dashicons on frontend** — enqueued `dashicons` CSS for offline bank icon rendering
- **Duplicate class attributes** — merged split `class=` attributes on offline icon `<a>` elements
- **Warning alignment** — warnings placed after card footer (not inline); fixed left padding misalignment

## [3.11.7] - 2026-03-26

### Improved
- **Fee summary card UI** — added proper CSS layout for summary rows (flex spacing, visual hierarchy, border separators) and a progress bar showing paid vs total percentage
- **Import refactor** — streamlined import class by extracting grid import logic into a dedicated trait

### Fixed
- **Overview tab academic year** — dropdown selection now correctly filters the overview data instead of always showing current year
- **Overview summary groups** — all configured groups now display even when they have zero enrollments
- **Overview user indexing** — fixed array index vs user_id mismatch in enrollment lookups
- **Overview group refs** — resolved group references by title instead of non-existent ID field

## [3.11.6] - 2026-03-26

### Fixed
- **Fees tab bracket labels** — groups without "Include in Report" now display with `[brackets]` in the Fees tab, matching the Summary tab convention

## [3.11.5] - 2026-03-26

### Changed
- **Monospace numbers in reports** — numeric columns use monospace font (`ui-monospace, SF Mono, Menlo, Consolas`) with `tabular-nums` for consistent digit-width alignment across rows

## [3.11.4] - 2026-03-26

### Added
- **Fee Matrix [Grid] import** — new CSV import type that accepts a school-friendly grid format (grades as rows, fee slabs as columns) with two-row header encoding slab slugs and fee types; includes fuzzy matching for slabs and grades, full verification preview with slab mapping dropdowns, and zero-amount auto-disable

### Changed
- **Fees tab rewritten** — now shows single summary table with groups as rows and fee type columns (Per Month, Per Term, Per Cycle, Per Tenure, Annual Total) totalled across all slabs, replacing the per-slab per-grade breakdowns

## [3.11.3] - 2026-03-26

### Fixed
- **Report 500 error** — fixed fatal error from non-existent `get_months_count()` method; replaced with terms-based month calculation
- **Fees tab annual total** — now correctly sums all 4 fee types (per_month × months + per_term × terms + per_cycle × cycles + per_tenure)

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
