# Changelog

All notable changes to qTap Finance are documented in this file.

## [3.16.98] - 2026-04-26

### Fixed — Fee Stats click-through now lands on the same orders the chart counted

The Receipts tab and the Fee Stats chart were running on **two different taxonomies**:

- **Fee Stats inner ring** classifies orders via `detect_order_via_channel()` — 6 buckets keyed off `created_via`: *wcpos / whatsapp / admin / checkout / rest-api / other*. Same classifier the parent's WC Order admin "By" column uses.
- **Receipts tab Source filter** classifies via `_kdc_qtap_finance_is_fee_payment` order-meta + `detect_pos_order()` — 3 buckets: *fee / pos / other*.

When you clicked an inner-ring slice, the JS did a lossy projection — `admin → fee`, `wcpos → pos`, everything else → `other`. That works only when the two taxonomies happen to agree on every order. They don't:

- Order with `created_via=admin` but no `_kdc_qtap_finance_is_fee_payment=yes` meta → chart counts it under **Admin**, but the click-through `?source[]=fee` filter on Receipts excludes it. The order then *appears* under the **Checkout** drill-through (which expands to `source=other` = `! is_fee && ! is_pos`).
- This produced the "why is this order showing as Checkout in Fee Stats?" report.

**Fix:** Receipts now speaks the same 6-channel `via_channel[]` taxonomy as Fee Stats. New URL contract:

```
?tab=receipts&via_channel[]=admin
?tab=receipts&via_channel[]=checkout&via_channel[]=admin
?tab=receipts&via_channel[]=wcpos
…
```

The Fee Stats JS click-through now passes `via_channel[]` directly — outer-ring click sends `payment_method[]=<bucket>` plus `via_channel[]` for each currently-active source chip; inner-ring click sends `via_channel[]=<that channel>`. The lossy `viaToSource()` projection is removed.

### Added — Receipts: "By" pill row + active-filter chip

New filter row (after Payment) with one pill per channel: *POS / WhatsApp / Admin / Checkout / REST API / Other*. Each pill carries an earth-tone Lucide icon matching the Fee Stats inner-ring palette so a click-through from the chart lands on the same hue. Pills include the same subtle count badge the v3.16.93 row got — number of orders in the date range matching that channel, computed via SQL (no per-order WC_Order hydration).

The aggregator's existing meta scan was extended to also pull `_created_via` (legacy) / `wp_wc_orders.created_via` column (HPOS) so the badge counts come for free out of the same query batch — no extra SQL per channel.

Active selections surface in the existing "Active filters" chip row as `By: Admin, Checkout`, with a × link to clear just that filter (same UX as the existing Source/Payment chips).

## [3.16.97] - 2026-04-26

### Changed — Fee Stats: scoped to the current active academic year only

The Fee Stats chart was counting **every** fee-tagged order in the date range, regardless of which academic year the receipt belonged to. On a school site running multi-year cohorts in parallel (e.g. catching up on prior-year arrears in April), this mixed last year's fee receipts into "this month" — making the donut misleading.

v3.16.97 filters the AJAX aggregator to only orders whose `_kdc_qtap_finance_academic_year` order-meta matches `kdc_qtap_finance_settings.current_academic_year`. Orders without the meta or with a different year are excluded from both the count and the amount totals on every ring (method + origin) and every legend / table row. The AJAX response now also returns an `academic_year` field for the JS layer, so future variants can re-display the scope alongside the totals.

**Defensive fallback:** when the active-year setting is empty (dev / staging where `current_academic_year` hasn't been configured), the filter is skipped and the chart counts all orders — so a misconfigured setting doesn't render an empty chart with no clue why.

### Added — Fee Stats: active-academic-year chip in the page header

Right next to the **Fee Stats** H2 — a small pill: *calendar-icon · AY · 2025-2026* — surfacing exactly which academic year the chart is scoped to. Title attribute on hover points to the setting that controls it (*qTap Finance > Settings > General*) so a confused staff member knows where to switch the active year.

The chip only renders when the active-year setting is configured (matching the AJAX filter behaviour).

### Added — Fee Stats: percent on each slice

v3.16.96 painted the bucket label onto each slice (UPI / Cash / NEFT / Admin / Checkout). v3.16.97 stacks the percent below it as a second line ("UPI" / "11.8%"). The width-fitting heuristic adapts:

- **Tier 1** — both lines fit → render label + percent stacked.
- **Tier 2** — only the label fits → render label only (skip the percent).
- **Tier 3** — neither fits → skip both.

Percent denominator is the dataset's own data sum, so it automatically tracks the active measure (count vs amount) — the chart's slice geometry was already using `data[i]` as the value, so dividing by the dataset's own sum gives the right percent regardless of the toggle.

## [3.16.96] - 2026-04-26

### Added — Fee Stats: segment labels drawn directly onto the donut slices

Each non-trivial slice now carries its own label (UPI / Cash / NEFT / Card / Admin / Checkout / etc.) painted directly on the segment in white, with a soft drop-shadow so the text stays readable across all earth-tone fill colors. The tooltip on hover still surfaces the full breakdown (count + percent + amount + percent) — the segment label is just the at-a-glance identity overlay.

Implementation: a small custom Chart.js plugin (`kdcSegmentLabels`, ~40 LOC inline) registered per-chart-instance. Walks each dataset's `meta.data` arc geometry, computes the midpoint of the slice (average radius × angle bisector), and renders the label there. Two automatic skip conditions keep the chart clean:

- **Zero-value slices** are skipped (they have no arc geometry to label).
- **Slices too thin to fit the label** are skipped — heuristic compares `ctx.measureText(label).width` against the chord length at midradius (90% threshold). Long labels like "Net Banking" auto-hide on small UPI-dominated charts where the slice is a sliver, while "UPI" stays visible on the same slice.

No new vendor dependency — reuses the already-vendored Chart.js v4 plugin API.

## [3.16.95] - 2026-04-26

### Fixed — Fee Stats: legend hover-blue-on-blue (4th report)

The legend `<button>` items had the same hover-blue-on-blue issue that v3.16.93 fixed for the table-cell links: block themes apply default `button:hover { background: #2271b1; color: #fff }` styles that paint the entire row solid blue, clobbering the swatch + label + meta line. The v3.16.93 fix only covered the `__tablecell-link` selector — the legend uses a different class. Now hardened the same way: `!important` on background / color / box-shadow / outline across `:hover` / `:focus` / `:focus-visible` / `:focus-within` / `:active`, with hover indicating selection via border-color + subtle box-shadow only.

### Changed — Fee Stats: 60/40 chart-to-legend split on desktop

The desktop side-by-side layout (introduced in v3.16.94) used a 50/50 split which left the donut feeling cramped — the inner ring needed more room to render the smaller channel slivers legibly. v3.16.95 changes the grid to `3fr 2fr` (60% chart, 40% legend) so the donut gets the larger half. Mobile (<900px) still stacks chart-then-legend with no width split.

### Added — Fee Stats: donut center title

The donut hole was empty space. v3.16.95 lands a permanent center title showing the active measure's grand total at a glance:

- **Top line** — measure label ("Total orders" / "Total amount", caps small grey).
- **Middle line** — the headline number itself (e.g. `161` for count, or `₹2,07,73,575` for amount), 18px bold.
- **Bottom line** — the *other* measure's value as context (so the count view shows the amount underneath, and vice-versa), 11px muted.

Toggling the Count ↔ Amount measure flips the center title in lockstep with the chart. `pointer-events: none` so the title never blocks hover/click on the segments — segment tooltips still surface the per-bucket breakdowns on hover.

### Added — Fee Stats: sectioned legend

The legend was a flat list mixing payment-method rows and created-via-channel rows, which on the desktop stacked-column layout was hard to scan. Now split into two clearly labelled sections:

- **By payment method** — outer-ring buckets (UPI / Card / Cash / NEFT / IMPS / RTGS / Net Banking / Online / Cheque / Wallet / Others).
- **By created-via channel** — inner-ring buckets (Admin / Checkout / WCPOS / WhatsApp / REST API / Other).

Section headings reuse the same i18n keys (`methodTable` / `originTable`) the v3.16.92 tables introduced, so localizers don't need to translate twice.

## [3.16.94] - 2026-04-26

### Changed — Fee Stats: chart on the left, legend stacked on the right (desktop)

On viewports ≥ 900px, the donut chart now sits on the **left** half of the chart area and the legend stacks as a single vertical column on the **right** — using the horizontal whitespace that was previously wasted next to the donut. Result: the entire breakdown reads at a glance without scrolling, and the legend's per-bucket meta lines (count + percentage + amount) get full-width room to breathe instead of squashing into a 4-up grid.

Below 900px (tablet portrait + mobile), the layout collapses back to the previous stack: chart centered, legend below in a responsive auto-fill grid. The `<div class="kdc-qtap-finance-fee-stats__chart-area">` wrapper around the canvas + legend is the new structural element — reuse it if you build other fee-stats variants.

### Fixed — Fee Stats: mobile vertical-space waste

Long INR currency strings (e.g. `₹2,07,73,575`) were making each "Total orders" / "Total amount" pill wide enough to wrap onto its own line, so on a 375px viewport the totals row alone ate ~80px of vertical space before the donut even started rendering. Two-step fix:

- **Totals pills now 2-up via grid** at `max-width: 600px` — `grid-template-columns: 1fr 1fr` with `min-width: 0` and ellipsis-truncation on the value spans, so the pills always sit side-by-side even when the currency string is long.
- **Filter card padding tightened** on mobile (`12px → 10px` block padding, `8px → 6px` row gap) so the entire pre-chart strip (filter card + totals) is visibly shorter, getting the donut into the viewport sooner.

## [3.16.93] - 2026-04-26

### Fixed — Fee Stats: hover-blue-on-blue on the new tables (3rd report)

Block themes apply default `button { background: #2271b1; color: #fff }` styles on `:hover` / `:focus` / `:active`. v3.16.85 hardened this for the Fee Stats Created Via chips, but v3.16.92 reintroduced the same problem on the new table-cell `<button>` row links: hovering a row painted the entire cell solid blue with white text, clobbering the swatch + label. Same root cause, same fix — `!important` on background / background-color / background-image / border / box-shadow / padding / margin across the link's `:hover` / `:focus` / `:focus-visible` / `:focus-within` / `:active` states. Plus a defensive reset on `tr:hover` / `td` background to neutralize any block-theme data-table tinting.

### Added — Receipts: subtle count badge inside each filter pill

Each Status / Source / Payment Method pill now shows the number of matching orders in the current date range as a small muted-grey number after the label. Counts are computed independently of the user's pill selections — i.e. the Status pill for "Completed" shows how many completed orders exist in the date range *regardless* of which Status / Source / Payment pills are currently checked. This makes the pills useful for exploration ("how many UPI orders are there this month?") not just filtering.

Pills with **0** matches in the current scope render at 55% opacity (a soft `is-zero` muted state) so staff can spot empty buckets at a glance and avoid clicking pills that won't change the result.

Counts are computed via direct SQL (one `GROUP BY status` query against `wp_wc_orders` / `wp_posts`, plus one bulk `meta_key IN (…)` scan against the order-meta table) — no per-order WC_Order hydration. Capped at 500 IDs (`search_max_results()` filter) to bound cost on huge ranges. HPOS-aware.

### Fixed — Receipts: `[0]` / `[1]` indices in URLs (`%5B0%5D` encoding)

Pagination links, chip-removal links, and the Reset All link were going through `add_query_arg()` / `remove_query_arg()`, which re-serializes `$_GET['status'] = ['completed', 'processing']` as `status%5B0%5D=completed&status%5B1%5D=processing` (numeric-indexed). Browsers display this as `status[0]=completed` in the address bar — ugly and not what the form sent.

New helper `KDC_qTap_Finance_Block_Editor_Receipts::build_receipts_url( array $set, array $remove )` hand-rolls the query string with literal empty brackets (`status%5B%5D=completed&status%5B%5D=processing` → renders as `status[]=completed&status[]=processing`), matching the form's natural serialization. Both formats parse to the same `array<int,string>` server-side, so it's purely cosmetic — but the cosmetic matters when sharing URLs with colleagues.

### Fixed — Receipts: empty-value query pollution

Empty filter inputs (`q=`, `date_from=`, `date_to=`) were being dragged into the URL by `add_query_arg`, then carried forward through pagination clicks and chip-removals — every URL grew a tail of `&q=&date_from=&date_to=` even when no filter was active.

Two layers of fix:
- The new `build_receipts_url()` helper strips empty values at build time (covers PHP-built URLs — pagination, chips, reset).
- A small inline `<script>` in the filter form's render disables empty text/date inputs right before native form submit so they don't appear in the GET URL at all (covers user-driven submits).

The script re-enables disabled controls on the next tick, so Cancel / Back navigation doesn't leave the form in a half-disabled state.

## [3.16.92] - 2026-04-26

### Added — Fee Stats: tabular view below the chart

Below the donut chart and the legend, the **Fee Stats** tab now renders the same data as two side-by-side tables — one per ring. **No new query** runs to populate them: it's a pure client-side re-render of the response that the chart already received (`state.latest`), so flipping date ranges or `created_via` chips updates the chart and the tables in lockstep on the same fetch.

**By payment method** — one row per non-zero bucket from `get_payment_method_buckets()` (UPI / Card / Cash / Cheque / IMPS / NEFT / Net Banking / Online / RTGS / Wallet / Others).

**By created-via channel** — one row per non-zero bucket from `get_via_channels()` (WCPOS, WhatsApp, Admin shop, Checkout flow, REST API, Other).

Each table has 5 columns: **Label**, **Count**, **Count %**, **Amount**, **Amount %** — every percentage computed against the totals of the active filter set (same denominator as the percentages in the donut tooltip + legend chips, so the numbers cross-check exactly with the totals pills above the chart). A footer row sums the column.

Each row's label is a click-target — clicking drills into the Receipts tab pre-filtered by that bucket + the active date range, the same URL contract the legend chips already used. Color swatches mirror the chart's earth-tone palette so a row's identity is visually obvious without reading the label.

Mobile: each table sits in its own `overflow-x: auto` wrapper, so narrow viewports get horizontal scroll on the table instead of squashed columns. On viewports wider than 900px the two tables sit side-by-side; below 900px they stack.

## [3.16.91] - 2026-04-26

### Changed — Block Editor class split into 8 traits (refactor only, zero behaviour changes)

`includes/class-kdc-qtap-finance-block-editor.php` was a **4,538-LOC monolith** through v3.16.90 — every Staff Console tab render, the user-facing Fees + Report blocks, the WC search infrastructure, the order classifiers, and 7 AJAX handlers all in one class. The rest of the plugin already follows a one-trait-per-domain convention (the Admin class composes from 30 traits) — Block Editor was the lone outlier, and a contributor opening it had to scroll through 1,000+ lines just to find a single tab's render method.

This release moves the methods **verbatim** into 8 cohesive traits under `includes/traits/`. The host class (`KDC_qTap_Finance_Block_Editor`) shrinks from 4,538 LOC to **301 LOC** — class declaration, the four static properties (`$instance`, `$search_cache`, `$user_search_cache`, `$last_search_truncated`), `get_instance()`, `__construct()`, `init_hooks()`, `register_block()`, and 8 `use` statements.

New trait files (under `includes/traits/`):

| Trait file | Trait | Methods | Approx LOC |
|------------|-------|---------|------------|
| `trait-kdc-qtap-finance-block-editor-search.php` | `KDC_qTap_Finance_Block_Editor_Search` | `search_max_results`, `is_hpos_active`, `find_user_ids_for_token`, `find_order_ids_by_meta_keys`, `find_order_ids_by_billing`, `find_order_ids_by_item_name`, `finalize_search_ids`, `search_orders_by_token`, `search_orders_by_field`, `search_orders_by_items` | ~505 |
| `trait-kdc-qtap-finance-block-editor-classifiers.php` | `KDC_qTap_Finance_Block_Editor_Classifiers` | `get_payment_method_buckets`, `classify_paywith_method`, `get_via_channels`, `detect_order_via_channel`, `order_has_fee_item`, `detect_pos_order` | ~278 |
| `trait-kdc-qtap-finance-block-editor-staff-console.php` | `KDC_qTap_Finance_Block_Editor_Staff_Console` | `ajax_backfill_order_enrollment`, `ajax_staff_order_items`, `render_staff_console_block`, `render_staff_console_menu` | ~340 |
| `trait-kdc-qtap-finance-block-editor-fee-stats.php` | `KDC_qTap_Finance_Block_Editor_Fee_Stats` | `register_fee_stats_lucide_icons`, `ajax_fee_stats_data`, `render_staff_console_fee_stats`, `render_fee_stats_filter_row` | ~404 |
| `trait-kdc-qtap-finance-block-editor-receipts.php` | `KDC_qTap_Finance_Block_Editor_Receipts` | `render_staff_console_receipts` | ~1,015 |
| `trait-kdc-qtap-finance-block-editor-overview.php` | `KDC_qTap_Finance_Block_Editor_Overview` | `render_staff_console_landing`, `render_dashboard_search_bar`, `render_dashboard_stats`, `render_dashboard_pending_verifications`, `render_staff_console_orders_table`, `render_staff_console_find_panel` | ~522 |
| `trait-kdc-qtap-finance-block-editor-fees-block.php` | `KDC_qTap_Finance_Block_Editor_Fees_Block` | `render_block`, `render_report_block`, `render_fees_block_associated_users_bar`, `shortcode_fees` | ~648 |
| `trait-kdc-qtap-finance-block-editor-frontend-ajax.php` | `KDC_qTap_Finance_Block_Editor_Frontend_AJAX` | `ajax_get_fees`, `ajax_search_users`, `ajax_get_users_by_grade`, `format_user_for_response` | ~716 |

### Why this is safe

- **Static state stays on the host class.** `$search_cache`, `$user_search_cache`, `$last_search_truncated` keep their declarations on `KDC_qTap_Finance_Block_Editor` — declaring them on a trait would give each using class its own copy. Trait static methods reference `self::$cache` and resolve correctly to the using class's storage.
- **Cross-trait `self::` and `$this->` calls just work.** PHP resolves both at the using-class scope. `Search` trait's `search_orders_by_token()` calling `self::find_user_ids_for_token()` (also in Search), or `Fee_Stats` trait's `ajax_fee_stats_data()` calling `self::detect_order_via_channel()` (in Classifiers), both resolve on `KDC_qTap_Finance_Block_Editor` — same as before.
- **Visibility preserved.** `private` methods in a trait are private *to the using class*, so they remain callable from any other trait the same class composes. No method had to be widened to `protected` or `public`.
- **`init_hooks()` and `register_block()` stay on the host class.** They're the wiring tables (AJAX hooks + Lucide filter; asset enqueue + `register_block_type()` calls) — keeping them on the host means the wiring is visible in one place when you open the class file. The handler methods themselves now live in their respective traits, but `array( $this, 'ajax_fee_stats_data' )` resolves through `$this` to the trait method.
- **External callers unchanged.** `KDC_qTap_Finance_Block_Editor::classify_paywith_method()`, `::get_payment_method_buckets()`, `::detect_order_via_channel()`, `::get_via_channels()`, `::order_has_fee_item()`, `::detect_pos_order()`, and the public static `::$last_search_truncated` are all called from `trait-kdc-qtap-finance-admin-tab-receipts.php` and `class-kdc-qtap-finance-wc-orders-admin.php`. Every one continues to resolve through the host class — trait methods are accessible by the host class name exactly like methods declared inline.

### File-size sanity

- Host class: **301 LOC** (down from 4,538).
- Largest trait: Receipts at ~1,015 LOC (single 1,000-LOC `render_staff_console_receipts()` method — kept whole; subdividing it would be a code refactor, not a structural split).
- Total LOC across the 9 final files (1 host + 8 traits): ~4,729, accounting for trait headers + closing braces.

## [3.16.90] - 2026-04-26

### Changed — Reminder Queue + Schedule merged into one tab

The standalone `Reminder Queue` and `Reminder Schedule` tabs are now stacked under a single **Reminder** tab in the Finance settings nav. Queue (manual-trigger button + pending list) renders on top, schedule (before/after-due day offsets, sending time window) below — separated by a thin divider rule. Each section keeps its own `<h2>` heading so the visual hierarchy stays clear without nested sub-tabs.

Legacy URLs `?tab=reminder-queue` and `?tab=reminder-schedule` 302-redirect to `?tab=reminder`. The redirect runs at the top of `render_settings_page()` *before* `get_current_tab()` filters unknown slugs, so old bookmarks land on the new tab instead of silently bouncing to Enrollments.

## [3.16.89] - 2026-04-26

### Restored — Reminder Schedule tab

v3.16.87 deleted the entire Notifications tab when the per-type/per-channel toggles centralized to the parent. That removal also took down the **Sending Time Window**, **Reminders Before Due Date**, and **Overdue Reminders After Due Date** controls — those configure WHEN the Finance reminder cron fires, which is Finance-specific cron timing the parent's Templates UI doesn't (and shouldn't) handle. The underlying settings (`reminder_before_days`, `reminder_after_days`, `reminder_window_start`, `reminder_window_end`) were never deprecated — `KDC_qTap_Finance_Notifications::get_reminder_schedule_settings()` still reads them — so admins were left with a hidden, uneditable cron config.

This release restores those controls as a dedicated **Reminder Schedule** tab (placed right after Reminder Queue in the Finance settings nav). The form posts to `options.php` against the existing `kdc_qtap_finance_settings_group` setting group, and uses the same explicit-list hidden-field preservation pattern as the Labels tab so saving this tab doesn't wipe other Finance settings. No data migration needed — existing values populate the tab on first load.

What's NOT restored: the per-type enable checkboxes, the "Enable Notifications" master toggle, and the Edit Template buttons. Those genuinely belong at `qTap > Notifications > Templates` (parent v2.7.12+) and the global pause (parent v2.7.13+).

## [3.16.88] - 2026-04-26

### Added
- **Fee Stats — percentages alongside count and amount.** The donut tooltip and the legend chips now show each segment's share of the ring total next to the raw numbers, so you can read the mix at a glance without doing the division in your head. Tooltip body becomes `Method: 12 orders (15.4%) · ₹45,000 (12.3%)`; legend rows become `12 (15.4%) · ₹45,000 (12.3%)`. Each percentage is computed against the totals from the active filter set (date range + Created Via chips), so it always matches the "Total orders / Total amount" pills above the chart. Percentages are suppressed when the ring total is zero (no `NaN%` artefacts on empty ranges).

## [3.16.87] - 2026-04-26

### Removed — Duplicate Notifications tab

The `qTap Finance > Notifications` admin tab is gone. After parent v2.7.12 centralized notification templates and per-type/per-channel toggles at `qTap > Notifications > Templates`, the Finance tab's "Enable Notifications" master toggle and per-type checkboxes became a parallel source of truth — admins had to keep two places in sync, and Finance's "Edit Template" deep-links pointed at the wrong URL anyway.

What you'll notice:
- The **Notifications** tab no longer appears in the Finance settings nav.
- Old links to `?tab=notifications` (and the legacy `?tab=templates`) **redirect** to `qTap > Notifications > Templates`, preserving any `edit=` / `channel=` query args as `type=` / `channel=` on the parent editor.
- Per-type enable lives at the parent's card-row checkbox (`type_enabled`); per-channel enable lives at the parent's editor toggle (`email.enabled`, `sms.enabled`, `whatsapp.enabled`, `webhook.enabled`).
- Global pause/"nuke switch" at parent v2.7.13+ replaces the Finance-wide "Enable Notifications" master toggle.

### Migrated — legacy gating preserves admin intent

A one-shot `admin_init` migration (`maybe_migrate_legacy_notification_gating`) lifts the existing Finance settings into parent storage so opt-outs survive the cutover:

- If `kdc_qtap_finance_settings['notifications_enabled']` was **0** (Finance-wide off): every Finance type id (`finance_payment_due_reminder`, `finance_payment_overdue`, `finance_payment_received`, `finance_payment_refunded`, `finance_payment_verified`, `finance_payment_rejected`, `finance_offline_submitted`, `finance_enrollment_created`, `finance_fee_assigned`) gets `type_enabled = false` written to `kdc_qtap_notification_templates`.
- If `kdc_qtap_finance_settings['enabled_notifications']` was an explicit allow-list: any Finance type id NOT in the list gets `type_enabled = false`. Types in the list keep the default-enabled behaviour.
- Idempotent — flagged via `kdc_qtap_finance_notifications_migrated` so subsequent runs short-circuit. Logged via `kdc_qtap_debug_log()` with the count of types disabled.

The legacy keys themselves stay in `kdc_qtap_finance_settings` (no destructive cleanup) — they're simply no longer consulted by `Finance::send()`. The settings sanitizer + labels-tab hidden-field preservation remain so importing old config still parses cleanly.

### Changed — `Finance::send()` gating

`KDC_qTap_Finance_Notifications::send()` no longer reads `kdc_qtap_finance_settings['notifications_enabled']` or `enabled_notifications`. It now consults `kdc_qtap_notification_templates[$full_type]['type_enabled']` directly via the new `is_type_enabled_in_parent()` helper. Returns `{ success: true, skipped: true, reason: 'type_disabled' }` when the parent has the type switched off — same response shape as before.

## [3.16.86] - 2026-04-27

### Fixed
- **Fee Stats currency formatting now uses the canonical `kdcQtapFinanceFmtAmount()` helper** from `assets/js/kdc-qtap-finance-currency.js` (the same formatter every other Finance frontend uses). v3.16.71 shipped Fee Stats with a local `formatAmount()` that did `n.toLocaleString()` — that always emits international grouping (`12,345,678`) regardless of the configured currency, so an INR site rendered `₹12,345,678` instead of the expected Indian `₹1,23,45,678`. The shared helper consults the localized `kdcQtapFinanceCurrency.code` and switches to the Indian grouping pattern when `code === 'INR'`. Affects the Total amount pill, the donut tooltip body, and the legend chip amounts. Defensive fallback to the previous inline formatter only if the helper script failed to load.
- **Script registration:** `kdc-qtap-finance-currency` is now a hard dep on the `kdc-qtap-finance-fee-stats` script handle, so the formatter is always loaded before the chart handler executes.

## [3.16.85] - 2026-04-27

### Changed
- **Receipts `q` text-search refactored from 7–14 unbounded queries to 4–5 bounded ones per token.** v3.16.41's `search_orders_by_token()` issued one `wc_get_orders` per meta key (7 keys + WC native search + customer-by-name with two `get_users` calls + customer orders) — every call had `'limit' => -1`, so on a 50K-order site a token like "John" hydrated tens of thousands of orders into memory before deduping. Multi-token searches ("John Doe") doubled or tripled the pain because the loop intersected per-token IDs, calling each helper afresh per token. New helpers replace the fan-out:
  - `find_user_ids_for_token()` — one indexed JOIN across `wp_users` + `wp_usermeta` (login / email / nicename / display + first/last/billing names), `LIMIT 200`.
  - `find_order_ids_by_meta_keys()` — one SQL with `meta_key IN (…) AND meta_value LIKE %s` against `wp_wc_orders_meta` (HPOS) or `wp_postmeta` (legacy); `LIMIT 500` (configurable via filter `kdc_qtap_finance_search_max_results`).
  - `find_order_ids_by_billing()` — single SQL against `wp_wc_order_addresses` (HPOS) or postmeta `_billing_*` keys (legacy).
  - `find_order_ids_by_item_name()` — bounded version of the existing items-table lookup.
  - `finalize_search_ids()` — central exit point that applies the caller's status filter via `post__in` and trips `$last_search_truncated` when the cap is hit.
  Per-token results are cached in a request-scoped `self::$search_cache[ field|token ]`, so multi-token searches re-use lookups instead of re-querying. Public method signatures are unchanged — `search_orders_by_token()`, `search_orders_by_field()`, `search_orders_by_items()` all keep their existing protected-static contracts.
- **"Showing first 500 matches" hint** above the Receipts table when the search ceiling is hit. Tells staff to narrow the search (use a targeted field, add a more specific token) instead of silently truncating.
- **Fee Stats default Created Via selection now includes both `admin` and `checkout`** (was `admin` only). Tridha staff routinely reconcile both bulk channels at once on first load. PHP default-source list, AJAX `sources[]` fallback, and JS `state.sources` initial value all updated in lockstep.

### Fixed
- **Filter chip hover contrast on the Fee Stats tab — second pass.** v3.16.77 added `:not(.is-active)` so active chips kept their white-on-blue text on hover, but the user reported the blue-on-blue still showed up under some frontend themes. The earlier rules used theme-color CSS variables without `!important`, so any theme rule on `button:hover` (FSE block themes, Storefront, Astra, etc.) with higher specificity overrode them. v3.16.85 pins active-chip `color: #fff !important` (and the source-chip earthy-palette colors) and explicitly anchors inactive-chip hover to `background: #fff !important` so theme styles can't bleed through.

### Audited
- Same Staff Console block audit as v3.16.84 — verified that the search refactor doesn't reintroduce any of the patterns the v3.16.81 → v3.16.82 audit flagged (no new self-joins, no HPOS+postmeta double-querying, no unbounded fan-out).

## [3.16.84] - 2026-04-27

### Changed
- **Fee Stats AJAX: removed N+1 hydration in `ajax_fee_stats_data()`.** Since v3.16.71 the aggregator ran two `wc_get_orders( return='ids' )` calls (primary by `payment_date`, fallback by `date_created`), merged the IDs, then `wc_get_order( $id )` per ID inside the foreach — that's 2 + N round-trips, ~5,002 hits on a 5,000-order date range. Switched both queries to `return='objects'` so WC hydrates everything in one batched DB load per query (2 round-trips total, regardless of result-set size). The dedupe loop now compares `$o->get_id()` instead of merging two ID arrays through `array_unique`. No semantic change — same date filter, same status set, same fee-line-item gate, same created_via classification — just dropped ~N redundant order lookups per chart render.

### Audited
- **Reviewed every Staff Console frontend block tab** (Overview / Receipts / Report / Fee Stats) for repeat or unbounded query patterns:
  - **Receipts paginated query** — OK. One `wc_get_orders` with `paged + paginate=true`. Source / payment_method post-fetch filtering acts on a 20-row page, bounded.
  - **Receipts `q` text search (`search_orders_by_token` / `search_orders_by_field`)** — heavy. Each search token fans out to up to 7–14 unbounded `wc_get_orders` / `get_users` calls (WC native search + first/last name meta + customer-id orders + 3 meta-key lookups). On a 50K-order site a single multi-token query (e.g. "John Doe") could touch 70K+ order objects in memory before deduping. Pre-existing pattern (predates this audit). Documented but not changed in this release — refactor needs careful scope (caching layer + LIMIT-bounded SQL).
  - **Fee Stats AJAX** — fixed in this release (above).
  - **Overview tab** — OK. Defers to `KDC_qTap_Finance_Enrollment::get_enrolled_users()` and `KDC_qTap_Finance_Payment::get_year_summary()`; no per-render fan-out on the tab handler itself.
  - **Report tab** — OK. PHP just emits the scaffold; data is fetched client-side via REST API + DataTables AJAX.

## [3.16.83] - 2026-04-27

### Added
- **`declare_supported_channels()` filter handler** — declares which channels each finance notification type's editor renders tabs for. Drives the new card-row UI in parent v2.7.12+ at `qTap > Notifications > Templates`. The 7 user-facing reminder/payment types support `email + whatsapp`; `enrollment_created`, `fee_assigned`, `offline_submitted` are email-only.
- **`declare_type_meta()` filter handler** — provides display metadata (name, description, icon, audience) for each finance type. Mirrors the old finance template_info array verbatim so the parent's card-row list shows exactly the same labels (Payment Due Reminder, Payment Overdue, Payment Received, etc.) admins were used to.

**Requires:** kdc-qtap v2.7.12+ for the new card-row Templates UI to render correctly.

## [3.16.82] - 2026-04-27

### Changed
- **Paid With filter: dropped the wasteful HPOS+postmeta double-query.** v3.16.81's `get_order_ids_for_paywith_bucket()` ran the same `SELECT order_id` against both `wp_wc_orders_meta` AND `wp_postmeta` and merged with `array_unique` — but on HPOS-active sites the HPOS table already has every record (whether HPOS-only or sync mode dual-writes), so the postmeta query was wasted work. Now picks one table based on `OrderUtil::custom_orders_table_usage_is_enabled()`: HPOS active → `wp_wc_orders_meta`, otherwise → `wp_postmeta`. One indexed query per filter application.

### Audited
- **Reviewed every `restrict_manage_*` filter and its query handler across the qTap plugin family** (kdc-qtap, kdc-qtap-finance, kdc-qtap-mobile, kdc-qtap-events, kdc-qtap-education) for the same self-join risk that produced the v3.16.79 → v3.16.81 502. Findings:
  - **kdc-qtap "Created Via" filter** (parent's 6-channel order source): clean. Each channel emits ≤5 EXISTS / equality clauses on `_created_via` or single-key meta. The "other" channel is intentionally a no-op (returns empty meta_query). No NOT-LIKE chains.
  - **kdc-qtap-finance user-list filters** (Year/Grade/Division/Exempt on `wp-admin/users.php`): clean. Single LIKE against the `kdc_qtap_finance_enrollments_index` meta key per filter; no chains.
  - **kdc-qtap-events attendee ticket-type filter**: mild. Two-step (fetches all ticket post IDs, then filters attendees via `post__in`); fine at current scale, candidate for direct-SQL optimization if event/ticket volumes grow 10×. Not changed in this release.

## [3.16.81] - 2026-04-27

### Fixed
- **502 Bad Gateway when applying the v3.16.79 "Paid With" filter.** The filter built a `meta_query` of OR/NOT-LIKE clauses against `paywith_method` — the "Others" bucket alone produced ~17 NOT-LIKE clauses against the same meta key, which translated into 17 self-joins on `wp_wc_orders_meta` (HPOS) or `wp_postmeta` (legacy). The query timed out at the nginx upstream and surfaced as a 502. Replaced with a direct-SQL pre-fetch in `KDC_qTap_Finance_WC_Orders_Admin::get_order_ids_for_paywith_bucket()` — one indexed `SELECT order_id` against the appropriate meta table, fed back via `post__in`. Same per-bucket semantics, same source of truth (the canonical `KDC_qTap_Finance_Block_Editor::get_payment_method_buckets()` map), drastically cheaper. HPOS + legacy storage both queried so dual-write installations are covered.
- **Filter URL pollution on the WC Orders admin table.** Every WC Orders filter dropdown ships with an empty default option ("All Status", "All Sources", "All Paid With", etc.); submitting the form sent every named field, so the URL accumulated empty params like `&qtap_paywith_filter=&qtap_order_source=&_customer_user=` every time any filter changed. New `KDC_qTap_Finance_WC_Orders_Admin::orders_admin_filter_cleanup_js()` runs once on the orders screen and disables empty fields just before submit — disabled fields don't appear in the resulting URL. Scoped to `#wc-orders-filter` (HPOS) and `#posts-filter` (legacy); leaves WP-wired hidden fields (page, paged, action, nonce) alone.

## [3.16.80] - 2026-04-27

### Fixed
- **Critical error on Finance > Notifications tab.** v3.16.78 deleted `trait-kdc-qtap-finance-admin-tab-templates.php`, which carried `get_template_key_for_notification()`. That helper is also called from `render_notifications_tab()` to compute the per-row Email/WhatsApp status chips, so removing the trait turned every visit to Finance > Notifications into a fatal `Call to undefined method` — exactly the page that already shouldn't have templates anymore. Restored the helper inline in `trait-kdc-qtap-finance-admin-tab-notifications.php` (where its only remaining caller lives).

**Requires:** kdc-qtap v2.7.11+ for the Templates tab to actually render at qTap > Notifications > Templates.

## [3.16.79] - 2026-04-27

### Added
- **New "Paid With" column on the WC Orders admin table** (HPOS + legacy CPT). Shows the order's `paywith_method` meta value with the matching bucket label as a small subtitle (e.g. "GooglePay" / "UPI", "ICICI Net Banking 6783" / "Net Banking"). Inserts immediately after the parent's Transaction ID column at priority 50; falls back to appending if the parent's "By"/"Transaction ID" columns aren't loaded. The column constant is `KDC_qTap_Finance_WC_Orders_Admin::COLUMN_PAYWITH_KEY`.
- **"Paid With" filter dropdown** above the orders table, mirroring the parent's "Created Via" filter. Options come from the canonical `KDC_qTap_Finance_Block_Editor::get_payment_method_buckets()` map — the same source of truth as the Receipts pill row and Fee Stats donut: All / Card / Cash / Cheque / IMPS / NEFT / Net Banking / Online / RTGS / UPI / Wallet / Others. URL param: `qtap_paywith_filter`.
- **Filter query handlers** for HPOS (`woocommerce_order_list_table_prepare_items_query_args`) and legacy CPT (`pre_get_posts`). Each bucket OR-LIKEs every substring pattern in the bucket map. The `others` bucket is the negative space — `paywith_method` exists, is non-empty, and AND-NOT-LIKEs every pattern across all other buckets.

## [3.16.78] - 2026-04-26

### Removed
- **Finance > Notifications > Templates tab.** Templates editor moved to the parent at `qTap > Notifications > Templates` (kdc-qtap v2.7.10+). Same UI fields, same labels — only the URL changed. The local `trait-kdc-qtap-finance-admin-tab-templates.php` was deleted.
- **Inline template editor inside Finance > Notifications.** The `?edit=` deep-link previously rendered an inline editor on the Notifications tab; it now redirects to the parent's editor with the type key remapped (short → full prefixed, e.g. `payment_received` → `finance_payment_received`).

### Changed
- **`Edit Template` buttons in the per-type list** now point to the parent editor (`admin.php?page=kdc-qtap&tab=notifications&section=templates&type=finance_<short>`) instead of finance's inline view. The Reminder Schedule UI and per-type enable/disable toggles remain on Finance > Notifications unchanged — they're fee-domain config, not template content.
- **`KDC_qTap_Finance_Notifications::get_template()`** now reads the centralized parent option (`kdc_qtap_notification_templates[$full_type]`) first, then falls back to the legacy `kdc_qtap_finance_settings.notification_templates[$short_key]` storage so customizations made before the upgrade keep working until the parent's v2 migration ports them across.
- **`is_email_enabled()`** and **`get_whatsapp_templates()`** updated the same way: parent option (full prefixed key) wins, legacy finance settings (short key) is the fallback.
- **`?tab=templates` legacy URL** now redirects to the parent's Templates section, mapping `?edit=<short>` to `?type=finance_<short>` so existing bookmarks land on the right editor.
- **`filter_template_edit_url`** stays registered as a hook but is now a pass-through — the parent's default URL is correct because the editor lives there.

**Requires:** kdc-qtap v2.7.10+ (the parent's centralized Templates editor and prefixed-key migration). Older parents will keep using the legacy finance-side fallback.

## [3.16.77] - 2026-04-26

### Added
- **Two new Fee Stats date presets: "This Month" and "Last Month".** This Month = 1st of the current calendar month → today. Last Month = full previous calendar month (1st through last day; computed as `new Date(year, currentMonth, 0)` so February's 28/29-day boundary handles itself).

### Changed
- **Default Fee Stats range is now "This Month"** (was "Today"). On first load, both server-side initial state and the client preset highlight default to the 1st of the current month → today. The Today chip stays available for single-day reads.
- **Initial preset highlight extended.** Previously only "Today" was auto-detected on load if the range matched; now the loader walks all 6 presets in priority order (Today, This Month, Last Month, Last 7d, Last 30d, This Year) and highlights the first match.

### Fixed
- **Filter chip hover contrast.** Hovering an active chip (e.g. the blue "This Year" pill in the screenshot from the user) was overriding its white text with the theme blue, producing blue-on-blue and rendering the label illegible. The hover rule now uses `:not(.is-active)` so active chips keep their white-on-blue contrast; an active-chip hover instead applies a small `filter: brightness(0.92)` for a tactile press feel.

## [3.16.76] - 2026-04-26

### Fixed
- **Fee Stats donut tooltip header.** Hovering an inner-ring (Created Via) segment was showing the tooltip header from the *outer* ring's labels — e.g. hovering "Checkout" rendered as "Cheque" because the index aligned. Chart.js defaults the tooltip title to `data.labels[i]`, which is fixed to the outer ring; we now derive the title from the hovered dataset's own `_labels` array. Body row also reformatted to lead with the ring kind ("Method" / "Origin") so a truncated header still leaves enough context.

## [3.16.75] - 2026-04-26

### Added
- **Notification cross-referencing card on the Finance > General tab.** Calls the parent's new `kdc_qtap_render_notifications_summary( 'kdc-qtap-finance' )` (parent v2.7.9+) at the end of the General tab. Admins now see, without leaving Finance: every Finance notification type with 7-day Sent / Failed counts, the latest-sent timestamp, and inline **Edit Template** + **View Logs** deep-links into the parent's Notifications log/templates UI.
- **`KDC_qTap_Finance_Notifications::register_type_owners()`** — declares Finance's 9 notification types (`finance_payment_due_reminder`, `finance_payment_overdue`, `finance_payment_received`, `finance_payment_refunded`, `finance_payment_verified`, `finance_payment_rejected`, `finance_offline_submitted`, `finance_enrollment_created`, `finance_fee_assigned`) to the parent via the new `kdc_qtap_notification_type_owners` filter. Drives the summary card scope, the parent's Source filter on the Notifications log, and Edit Template deep-link routing.
- **`KDC_qTap_Finance_Notifications::filter_template_edit_url()`** — hooks `kdc_qtap_notification_template_edit_url` so Edit Template buttons rendered from the parent's summary card route to Finance's existing per-type editor at `?page=kdc-qtap-finance&tab=notifications&edit={type}`. Preserves the current template-edit UX intentionally — no regression to admins who already know it.

**Requires:** kdc-qtap v2.7.9+ (the parent's cross-referencing infrastructure). Older parents will silently no-op the registration filter; the General tab card will not render.

## [3.16.74] - 2026-04-26

### Changed
- **Renamed the staff console tab from "Pay Stats" to "Fee Stats"**, with the URL slug shortened to `?tab=stats`. The chart only ever counted orders that had a fee component — the new name reflects what the tab actually shows. Visible label, page heading, and menu item all updated. The JS file moved from `kdc-qtap-finance-pay-stats.js` → `kdc-qtap-finance-fee-stats.js`; the AJAX action renamed from `kdc_qtap_finance_pay_stats_data` → `kdc_qtap_finance_fee_stats_data`; localized JS global `kdcQtapFinancePayStats` → `kdcQtapFinanceFeeStats`. CSS BEM blocks renamed from `kdc-qtap-finance-pay-stats__*` → `kdc-qtap-finance-fee-stats__*`.
- **Fee-line-item filter applied to every counted order.** Regardless of source (Admin / POS / Others), an order is only counted on the chart if it contains at least one line item with `_kdc_qtap_finance_product_type === 'kdc_qtap_finance_fee'` (the meta stamped by `add_fee_line_item()`). Non-fee Woo orders — uniforms, lunch sales, anything that isn't a finance fee — are skipped. New `KDC_qTap_Finance_Block_Editor::order_has_fee_item( $order )` static helper.
- **Source classification now uses the 6-channel `created_via` taxonomy** that already drives the parent plugin's WC Order admin "By" column and the Receipts admin tab — one source of truth. Channels: `wcpos`, `whatsapp`, `admin`, `checkout`, `rest-api`, `other`. The Fee Stats source chip row was rebuilt with one chip per channel (was 3 chips in v3.16.73: Admin/POS/Others). The classifier was lifted from the Receipts admin trait (`detect_order_via()`) into a public static helper `KDC_qTap_Finance_Block_Editor::detect_order_via_channel()`; the trait now delegates. Default selection on first load = `admin` only — the other 5 channels are opt-ins. Click-through to the staff Receipts tab maps the 6 channels onto its 3-source URL contract: `admin` → `source[]=fee`, `wcpos` → `source[]=pos`, everything else (`checkout`/`whatsapp`/`rest-api`/`other`) → `source[]=other`.

## [3.16.73] - 2026-04-26

### Added
- **New "This Year" preset chip** in the Pay Stats date-range row. Sets the range to Jan 1 of the current calendar year through today — useful for the year-to-date view that staff want when reconciling against the academic finance log.

### Changed
- **Pay Stats inner ring now splits Admin and POS into separate segments.** Previous behaviour folded WCPOS-detected orders into the Admin segment alongside Record-Payment / fee orders; v3.16.73 keeps them distinct so staff can read POS volume independently. Three mutually-exclusive origin buckets:
  - `fee` → orders with `_kdc_qtap_finance_is_fee_payment=yes` (Admin Record-Payment)
  - `pos` → POS-detected orders (`detect_pos_order()`) that are NOT also fee payments
  - `other` → everything else (customer-facing checkout)
- **New "Source" filter row on the Pay Stats tab** with three toggle chips: **Admin** (active by default), **POS**, **Others**. The chart aggregates only orders matching the active sources — both rings honour the filter so the outer ring (method bucket) shows methods used within the selected sources only. At least one source must stay active (the chip click is ignored if it would clear the last one).
- **Default chart now shows fee items only.** Per the staff workflow at the school, the bulk of activity is admin-recorded fee payments — that's the headline number. POS and customer-facing checkout are opt-ins via the new chips.
- **Click-through routing updated.** Outer-ring (method) clicks now also append `source[]=` for each active source so the Receipts list matches what was on the chart. Inner-ring clicks drill into just that one source (`source[]=fee|pos|other`), independent of the rest of the active filter — a single-segment click is interpreted as "show me only this slice".
- **Earth-tone palette** for the donut. Replaced the saturated WP-blue / red / indigo / teal mix with a warm low-saturation set: taupe, sage moss, mauve, ochre, rust, pine, olive, sand, burnt sienna, terracotta, walnut for the 11 method buckets; deep walnut / copper / sage for the three origin buckets. Source chips inherit the matching segment colour in their active state.

## [3.16.72] - 2026-04-25

### Added
- **New "Exempt" checkbox in the Additional Information block on the user-edit screen.** Stored as `kdc_qtap_finance_exempt` user meta (boolean `'yes'`/`''`). Acts as the default state for **NEW** enrollments — when staff create an enrollment for the user, the per-year `exempt` flag is pre-populated from this meta. The per-enrollment exempt toggle is still independently editable on the enrollment edit form, and toggling the user-meta after enrollments already exist does NOT retroactively mutate historical years (each year's enrollment owns its own flag once created).

### Removed
- **Legacy `is_rte` field eliminated entirely.** The plugin renamed it to `exempt` in v3.0.3 and read both names with `exempt` preferred. v3.16.72 drops the back-compat read everywhere and the per-save strip loop introduced in v3.16.28 — every read site now consults `exempt` only, every write site stores `exempt` only. AJAX handlers in [trait-kdc-qtap-finance-user-meta-enrollments.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-enrollments.php) now read `$_POST['exempt']` (was `$_POST['is_rte']`) and the form input id renamed to `kdc_qtap_finance_enrollment_exempt`. JS payloads in [kdc-qtap-finance-user-profile-enrollments.js](kdc-qtap-finance/assets/js/kdc-qtap-finance-user-profile-enrollments.js) and [kdc-qtap-finance-user-profile.js](kdc-qtap-finance/assets/js/kdc-qtap-finance-user-profile.js) updated to send `exempt:` matching the renamed POST field. AJAX response keys, HTML `data-rte` → `data-exempt`, JS `isRte` → `isExempt`. The `data.user.is_rte` read in the fees-block frontend renamed to `data.user.exempt`. The `KDC_qTap_Finance_Fee_Matrix::is_rte_exempt()` static (a fee-matrix slab classifier — different concern) and the CSV import alias array entry `'is_rte'` (back-compat for user CSV files) are intentionally preserved.

### Migrated
- **One-time `migrate_strip_is_rte_3_16_72()`** sweeps every user's `kdc_qtap_finance_enrollments` meta blob (version-gated; runs once via `kdc_qtap_finance_strip_is_rte_3_16_72_done` option). For each year-record carrying `is_rte`: copies a truthy value into `exempt` if `exempt` is absent/false (preserves the user's exempt status if they happen to be on an old record), then unsets `is_rte`. Re-saves the meta only when something actually changed. Idempotent — once `is_rte` is gone everywhere, re-runs are no-ops. Audit counts (users_scanned / users_updated / records_scanned / records_dropped_rte / records_promoted_rte) emitted via `kdc_qtap_debug_log()`.

## [3.16.71] - 2026-04-25

### Added
- **New Staff Console tab: Pay Stats** (`?tab=pay-stats`). Interactive 2-ring doughnut chart of WC orders for a date range — outer ring shows the 11 payment-method buckets (Card / Cash / Cheque / IMPS / NEFT / Net Banking / Online / RTGS / UPI / Wallet / Others), inner ring shows Checkout vs Admin (POS folded into Admin per the staff workflow). Date range presets: **Today** (default) / **Last 7 days** / **Last 30 days** / **Custom** (start–end ISO date inputs). Count / Amount measure toggle re-skins segment proportions in place — both metrics surface in tooltips regardless. Custom legend grid below the chart lists every non-zero bucket with count + amount; clicking a legend chip is the same as clicking the matching segment.
- **Click-through to Receipts**: clicking any segment (or legend chip) navigates to the Receipts tab pre-filtered by that bucket + the active date range. Method segment → `payment_method[]=<bucket>`. Origin segment → `source[]=fee&source[]=pos` (Admin) or `source[]=other` (Checkout). The URL contract was already supported by the v3.16.60 Receipts filter — no changes to the Receipts handler.
- **Vendored Chart.js v4.4.1** at [assets/lib/chart.js/chart.umd.min.js](kdc-qtap-finance/assets/lib/chart.js/chart.umd.min.js). Self-hosted (no third-party CDN), enqueued only on the Pay Stats tab so the 200 KB library doesn't load on Overview / Receipts / Report.
- **`pie-chart` Lucide icon** registered into the parent's `kdc_qtap_lucide_icons` filter map for the new tab. Parent v2.7.6 made the registry filterable specifically so child plugins could extend it without a parent release.

### Changed
- **Refactor**: lifted the inline `$payment_method_patterns` / `$all_payment_methods` declarations from `render_staff_console_receipts()` into two new public static helpers — `KDC_qTap_Finance_Block_Editor::get_payment_method_buckets()` (canonical bucket map: label + substring patterns) and `KDC_qTap_Finance_Block_Editor::classify_paywith_method( $raw )` (returns the bucket key for any free-text `paywith_method` value, or `'others'`). The Receipts filter now reads from the helper. Pure refactor — same buckets, same patterns, same classification — but the new Pay Stats AJAX endpoint reads from the same source of truth instead of re-declaring the map.

## [3.16.70] - 2026-04-25

### Removed
- **`kdc_qtap_finance_lucide()` shim removed.** The wrapper had been thin since v3.15.38 — it just delegated to the parent's `kdc_qtap_lucide()` — but kept the call-site convention `kdc_qtap_finance_*` everywhere. After v2.7.6's "parent is the source of truth" mandate, that prefix was misleading: every call really resolved to the parent's registry. Renamed all 27 call sites across [class-kdc-qtap-finance-block-editor.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-block-editor.php) (×17), [trait-kdc-qtap-finance-user-meta-rendering.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-rendering.php) (×6), and [kdc-qtap-finance-helper-functions.php](kdc-qtap-finance/includes/kdc-qtap-finance-helper-functions.php) (×2 internal); the `function_exists()` guards that referenced the old name were renamed too. The shim definition is gone — calls go straight to `kdc_qtap_lucide()` from the parent. The Finance dependency check at activation already guarantees the parent is loaded, so no fallback is needed.

## [3.16.69] - 2026-04-25

### Fixed
- **Stop minting synthetic `TXN-{n}` / `IMPORT-{n}` values into the WC `transaction_id` meta.** Three write paths (admin Record Payment, frontend Submit Offline, CSV import) used to mirror the synthetic UTR fallback into BOTH `pay_utr` (our meta) AND `transaction_id` (WC's gateway field). That created two real bugs: (1) the synthetic collided with whatever the payment gateway later wrote into `transaction_id`, destroying the gateway's actual transaction reference; (2) once an admin entered the real UTR through any other path (admin metabox, gateway callback, manual edit), only `pay_utr` got updated, so `transaction_id` stayed frozen on the stale synthetic — the symptom that showed up as "pay_utr has the real UTR but transaction_id still says TXN-42." Removed the duplicate `update_meta_data( 'transaction_id', … )` line at all three write sites and dropped the synthetic fallback (`$reference ?: sprintf( 'TXN-%d', $transaction_id )`) — now an empty UTR stays empty, which receipts render as "—", more honest than a fake-looking ID. Removed the `?: $order->get_meta( 'transaction_id' )` fallback at four reader sites ([class-kdc-qtap-finance-block-editor.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-block-editor.php) ×2, [trait-kdc-qtap-finance-user-meta-rendering.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-rendering.php), [trait-kdc-qtap-finance-admin-tab-receipts.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-admin-tab-receipts.php)) so the UI no longer presents synthetic-stamped `transaction_id` values as if they were UTRs. `sync_receipt_metas_from_transaction()` is unchanged — it already gated on "only when `pay_utr` is missing" and is the right place to do the source-of-truth waterfall.

### Migrated
- **One-time `migrate_clean_synthetic_txn_3_16_69()`** sweeps existing fee orders (version-gated; runs once via `kdc_qtap_finance_clean_synthetic_txn_3_16_69_done` option). For each order whose `transaction_id` matches `^(?:TXN|IMPORT)-\d+$`: copies the synthetic into `pay_utr` if `pay_utr` is empty (preserves whatever was being rendered on receipts), then clears `transaction_id` so future gateway writes win cleanly. Orders whose `transaction_id` carries a real (non-synthetic) value — gateway-stamped, admin-typed — are left alone. Idempotent: rerun is a no-op. Audit counts (scanned / copied_to_utr / cleared_txn / skipped_real) emitted via `kdc_qtap_debug_log()`.

## [3.16.68] - 2026-04-25

### Fixed
- **WCPDF receipts no longer render the un-broken-up "full fee" line item, removing the manual "Recompute Breakup" step from every payment flow.** Two new gates establish the user-requested pipeline `recompute (conditional) → process_order → WCPDF` for every order, regardless of how it was created.
  - **Pre-completion gate** — new `maybe_recompute_breakup_for_amount_mismatch()` in [trait-kdc-qtap-finance-wc-orders.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-orders.php) hooked at priority **5** on `woocommerce_order_status_completed` + `woocommerce_order_status_processing` (ahead of the default-priority `process_completed_order()` at 10). Compares the WC order total against the sum of `Payment.amount_due` across linked payments; if they differ by more than ₹0.01 it invokes `recompute_order_allocation_breakup( $order, 'auto-mismatch' )` first. In the normal full-payment path the gate is a no-op and `process_completed_order()` retains ownership of the breakup at priority 10 — no double-work.
  - **Pre-PDF defensive guard** — `maybe_recompute_breakup_before_pdf()` hooked on `wpo_wcpdf_init_document`. Catches the legacy / direct-download case where an order skipped the status flow entirely (manual import, never went through `processing`/`completed`). Idempotent: skips when `_kdc_qtap_finance_breakup_applied_at` is already set, so existing orders aren't re-touched on every PDF download.
  - The "amount mismatch" trigger condition is exactly what the user asked for — the gate doesn't fire when line items already sum to the intended fee total. Partial payments, manual edits, refunds, and coupons that drift the total are the cases it catches.

## [3.16.67] - 2026-04-25

### Changed
- **Page loader now uses the parent's `KdcQtapUI.showPageLoader()` (parent v2.7.8+) instead of the Finance-local overlay shipped in v3.16.66.** The previous release shipped its own CSS markup, JS helpers, and `window.kdcQtapFinancePageLoader` global — the moment another child plugin needed the same UX, that copy would have drifted away from theirs. Source-of-truth rule applied: visual components owned by the parent. Removed the local `.kdc-qtap-finance-page-loader*` CSS block, the local `showPageLoader()`/`hidePageLoader()` definitions, and the per-call sub-line strings (parent's API takes a single message). All four transaction call sites — `initiateSinglePayment`, `initiateMultiPayment`, `initiateTermPayment`, `submitInlinePaymentForm` — now obtain a handle from `KdcQtapUI.showPageLoader()` and use `loader.setMessage()` to flip "Processing…" → "Redirecting…" before the `window.location.href` jump. The thin wrapper `showPageLoader( msg )` is kept call-site-side as a no-op fallback (returns null) so the page still works on older parent versions without throwing. The fees-block frontend script now declares `kdc-qtap-frontend-helpers` as a `wp_register_script` dependency so the parent's helper is guaranteed to be loaded before our handlers run. Three i18n keys added in v3.16.66 (`pageLoaderPaying` / `pageLoaderRedirecting` / `pageLoaderSubmitting`) are removed — the parent's API uses the existing `processing` / `redirecting` / `submitting` strings instead.

## [3.16.66] - 2026-04-25

### Added
- **Page-level loader overlay during transactional actions on the Fees block.** Previously, clicking Pay Now / Pay All Dues / Submit Offline Payment only flipped the button text to "Processing…" and disabled it — easy to miss on mobile, when scrolled past the button, or when the AJAX takes a few seconds to respond. The new overlay covers the full viewport with a soft-blurred backdrop, a centered card with a spinner, a primary message ("Processing…" / "Redirecting…" / "Submitting…"), and a secondary line explaining the action ("Preparing your payment…" / "Sending you to checkout…" / "Recording your payment details…"). Shows on action start; updates the message before the `window.location.href` redirect so the user sees a clear "leaving this page" cue; hides on AJAX error so the user can retry. Singleton — re-calling `showPageLoader()` updates the message instead of stacking overlays. Wired into all four transaction triggers in [kdc-qtap-finance-block-frontend.js](kdc-qtap-finance/blocks/fees-block/kdc-qtap-finance-block-frontend.js): `initiateSinglePayment()`, `initiateMultiPayment()`, `initiateTermPayment()`, and `submitInlinePaymentForm()`. The `window.kdcQtapFinancePageLoader` global is exposed so future blocks (Staff Console transactional actions, etc.) can reuse the same overlay without duplicating markup. Strings are translatable via three new i18n keys: `pageLoaderPaying`, `pageLoaderRedirecting`, `pageLoaderSubmitting`.

## [3.16.65] - 2026-04-25

### Fixed
- **Receipts tab icon swapped from Lucide `receipt` to `clipboard-list`.** The `receipt` glyph contains an inline `$` shape inside the receipt body (visible at the 18px the tabs render at), which violates the absolute icon policy: no currency symbol anywhere, even on icons nominally about receipts/invoices. v2.7.7 of the parent plugin removes `receipt` from the central registry and adds `clipboard-list` + `clipboard` as non-currency document alternatives — `clipboard-list` is the closest visual analogue (a clipboard with horizontal lines, suitable for "list of receipts/invoices") so the Receipts tab now uses it.

## [3.16.64] - 2026-04-25

### Changed
- **Staff Console tabs (Overview / Receipts / Report / POS) now use Lucide icons** instead of Dashicons. Per the icon policy: Lucide on the frontend, Dashicons on the backend — the Staff Console is rendered as a frontend Gutenberg block, so Dashicons were the wrong primitive there (they pull in the wp-admin icon font and look out of place against block-themed pages). New mapping: `dashicons-dashboard` → `layout-dashboard`, `dashicons-media-document` → `receipt`, `dashicons-chart-bar` → `bar-chart-3`, `dashicons-cart` → `shopping-cart`. All four Lucide names resolve via the parent's filterable `kdc_qtap_lucide_icons` map (parent v2.7.6+); the Finance shim delegates there. Icons render at 18px with a small right margin to match the visual weight of the previous Dashicons.

## [3.16.63] - 2026-04-25

### Fixed
- **Apps List card icon now uses `dashicons-portfolio` (briefcase) instead of the 🪙 emoji shipped briefly in v3.16.62.** The icon policy is: Lucide on the frontend, **Dashicons on the backend**, and **never an emoji** anywhere — wp-admin should blend with the native admin chrome and emoji rendering varies too much across OS/browser combos to feel polished there. v3.16.62 violated the backend half of the rule. `dashicons-portfolio` keeps the "finance/business" association without invoking any currency symbol (the `$` glyph and `dashicons-money*` are also forbidden by the same policy). Still avoids the v3.16.61-and-earlier collision with the Education plugin's `dashicons-welcome-learn-more` flag.

## [3.16.62] - 2026-04-25

### Changed
- **Apps List dashboard icon switched from `dashicons-welcome-learn-more` (a flag) to 🪙 (coins emoji)** in the qTap App > Apps List card. The previous flag dashicon collided with the new `kdc-qtap-education` plugin which uses the same icon, making the two cards visually identical and forcing staff to read the title to tell them apart. Coins is the canonical "finance" imagery without invoking the $ glyph or any other currency symbol (per the codebase icon policy: never use a currency-symbol icon — coins or banknote imagery only). The parent plugin's card renderer in `trait-kdc-qtap-admin-apps.php` treats any non-`dashicons-` prefixed string as inline text, so the emoji renders at the same 24px slot alongside its dashicon-using siblings.

## [3.16.61] - 2026-04-25

### Removed
- **Legacy education-plugin migration scaffolding.** Deleted `KDC_qTap_Finance_Migration` (`includes/class-kdc-qtap-finance-migration.php`), the `maybe_migrate_from_education()` method, and the activation-time call that triggered it. The migration handled the v3.0.0 plugin rename (kdc-qtap-education → kdc-qtap-finance) and is no longer needed. **Why now:** the new `kdc-qtap-education` plugin uses the `kdc_qtap_education_*` option/meta namespace; leaving the migration in place would cause Finance to misread those new options as v2.x leftover data on next reactivation and silently rename them into `kdc_qtap_finance_*`, corrupting the new plugin's data.
- Legacy `kdc_qtap_education()` function alias (deprecated since v3.0.0). The function name is now owned by the new `kdc-qtap-education` plugin.

### Fixed
- **Orphan WP-cron event from v3.0.0 rename.** Production was carrying a daily `kdc_qtap_education_check_overdue` cron schedule with no listener — leftover from the v3.0.0 rename which migrated options/tables but never touched the cron schedule. Activation now unschedules it on first run. Harmless until/unless the new `kdc-qtap-education` plugin registers a listener for that hook name; defensive cleanup so the new plugin can use the namespace without surprise.

## [3.16.60] - 2026-04-24

### Added
- **Staff Receipts tab: Payment Method filter row** with 11 alphabetically-sorted pills — **Card, Cash, Cheque, IMPS, NEFT, Net Banking, Online, Others, RTGS, UPI, Wallet**. Each pill maps to a list of case-insensitive substrings matched against the `paywith_method` order meta (e.g., UPI pill matches "UPI", "GPAY", "PhonePe", "BHIM"; Card matches "Card", "Credit", "Debit"; Online matches "Zaakpay", "Razorpay", "Stripe", "PayU", "PayPal", "CCAvenue", "Gateway"; Cheque matches "Cheque", "Check", "Demand Draft", "DD"; individual rails NEFT / RTGS / IMPS / Net Banking each have their own pill). "Others" is the negative space — orders whose `paywith_method` doesn't match any known pattern (or is empty) fall under it. Defaults to all pills selected (= no filtering); deselect pills to narrow the view. Filter state is mirrored into Active Filters chips and the Reset-all link, and the `payment_method[]` param is preserved through pagination. Same post-query pagination tradeoff as the existing Source filter (meta is free-text so partial matching doesn't translate cleanly to `WC_Order_Query` args).

## [3.16.59] - 2026-04-24

### Added
- **Order Items modal now shows Order ID + Order Date, Receipt Number + Receipt Date, Payment Mode, and UTR / Transaction ID** in a dedicated summary panel at the top of the modal body. Previously the modal jumped straight from a thin customer/year/grade line into the line-item list — staff had to cross-reference the Receipts tab or the order edit screen to see how (and via which receipt) a payment landed. The panel renders as a 2-column grid (single column on narrow viewports) with caps labels over large values and muted sub-lines for the accompanying dates. Extended `ajax_staff_order_items()` in [class-kdc-qtap-finance-block-editor.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-block-editor.php) to include `order_date`, `receipt_number`, `receipt_date`, `payment_method`, `utr`; receipt-date fallback chain (`payment_date` → `date_paid` → `date_created`) matches the Receipts tab + user-edit WC Orders table so the displayed date is consistent everywhere. UTR rendered in monospace with `word-break: break-all` so long reference strings don't overflow the cell.

## [3.16.58] - 2026-04-24

### Added
- **Overview tab search now supports a "Search in" field selector** modeled on the Receipts tab. Staff can narrow the search to `{Student} Name` (first_name / last_name / display_name / nickname — explicitly NOT email/login/mobile, so short surname searches don't fan out to unrelated accounts) or `Payee Name` (WC order `_kdc_qtap_finance_payee_name` + `_billing_first_name` + `_billing_last_name` — returns the *student* whose order carries that payee), keeping `All fields` as the catch-all. Partial + reorder-tolerant matching is preserved: "John Smith", "Smith John", "Joh Smi" all match via the existing per-token AND semantics. Switching the field selector with an active query auto-reruns the search without a Find click. REST `staff-search` endpoint extended with a `q_field` param in [class-kdc-qtap-finance-rest-api.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-rest-api.php) (`staff_search_by_user_columns()` + `staff_search_by_payee_name()` helpers).
- **Search results render as clickable "{student} cards"** in a responsive grid — student name, email, and Year · Grade · Division on each card. The entire card is a link; clicking opens `?user={id}` in a new browser tab with every other query parameter stripped (clean canonical URL regardless of the admin's current filter state). Hover lift + WP-admin-theme-tinted border for feedback.

### Changed
- **Overview tab: "Create New" user flow removed** pending a proper UX pass — re-planned for a later release. The dashboard search bar is now a single row: `SEARCH IN [dropdown] [input] [Find]`, matching the Receipts tab cadence exactly. Create-new REST endpoints are untouched and ready to re-wire when the new flow ships.
- **User-edit screen: WooCommerce Orders table re-skinned for the frontend staff console.** wp-admin's `.widefat` / `.form-table` rules don't load on the frontend, which is why v3.16.57 still looked unstyled. Added staff-console-scoped CSS in [kdc-qtap-finance-staff-console.css](kdc-qtap-finance/blocks/staff-console/kdc-qtap-finance-staff-console.css) for the `.kdc-qtap-finance-wc-orders-section` table: rounded card container + soft shadow, generous header/cell padding, hover-row tint, and the "View items" action button styled as a solid primary (matching the Receipts tab's blue list button).
- **User-edit screen: "Additional Information" now renders as a card with a 2-column grid layout** — label column + control column, so the Date of Birth input no longer stretches to full width and the Associated Students search input sits next to its label instead of below it. The date input width is capped at 170px (was `regular-text` = ~240px) and all controls use the same 34px height / 4px radius as other staff-console inputs. `render_associated_users_field()` in [trait-kdc-qtap-finance-user-meta-associations.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-associations.php) gained an optional `$args['layout']` with `'grid'` option to emit the standalone label + value pair (`'table'` fallback keeps wp-admin form-table rendering unchanged).

## [3.16.57] - 2026-04-24

### Changed
- **Staff user-edit screen: moved "Additional Information" (Gender / Date of Birth / Associated Students) below the WooCommerce Orders table.** Previously the profile opened with demographics before Finance Enrollment and order activity. Staff doing day-to-day fee work need the enrollment state and payment history up top; demographics are referenced rarely. Reordered in [trait-kdc-qtap-finance-user-meta-rendering.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-rendering.php) so the flow now reads: Finance Enrollment → WooCommerce Orders → Additional Information.
- **WooCommerce Orders table on the user-edit screen now matches the Staff Console Receipts tab layout.** Replaced the compact v3.15.0 table (Order / Payment / Amount / Receipt # / Status) with the same visual cadence used on `/staff/?tab=receipts` (minus the Student column — the profile already identifies the user). Each row now shows: order number with FEE / POS badges + order-date subline; payment method + UTR/txn-id subline; right-aligned amount; receipt number with external-link glyph + receipt-date subline (same canonical `payment_date` → `date_paid` → `date_created` fallback chain as the Receipts tab); Lucide-iconed status chip via `kdc_qtap_finance_render_order_status()`; and a "View items" button that opens the shared items modal (lazy-loaded via `kdc_qtap_finance_staff_order_items` AJAX). The items-modal CSS + JS are enqueued defensively so the button works in both contexts (wp-admin profile and frontend staff console).

## [3.16.56] - 2026-04-24

### Fixed
- **Staff Console Receipts targeted search now actually restricts results to matching orders.** Filtering by any field (Items purchased, Payee Name, Receipt Number, UTR, Payment Method, etc.) previously returned every order matching the status filter instead of only the handful of matching orders — e.g., searching `Items (purchased) = Plate` on a dataset with ~8–10 plate items returned 1,186 tuition-fee receipts. Root cause: the code passed `'include' => $ids` to `wc_get_orders()`, but WooCommerce's HPOS `OrdersTableQuery` parameter mapping has no entry for `include` (only `post__in` → `id`), so the ID restriction was silently dropped on HPOS installs and only the status filter remained in force. Switched all three call sites in [includes/class-kdc-qtap-finance-block-editor.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-block-editor.php) from `include` → `post__in`, which is mapped on both HPOS (→ `id`) and legacy CPT (native WP_Query param). The same bug silently broke every targeted-field search added in v3.16.52 — all restored with this one-arg change.

## [3.16.55] - 2026-04-24

### Fixed
- **Date range on the Staff Receipts filter bar now renders inline on one row** (From / To side-by-side). v3.16.54 intended the single-row layout but block themes and WooCommerce form styles commonly ship a broad `input[type="date"] { width: 100%; }` rule that overrode our `width: 150px`, stretching the From date input to fill the content column and forcing the "To" label+input onto a new line. Two-part fix:
  - The date input's width now uses `!important` + `flex: 0 0 auto` + `box-sizing: border-box` so theme/WC rules can't stretch it.
  - From/To + their inputs are wrapped in a new `.kdc-qtap-rx-daterange` span with `flex-wrap: nowrap` — the group wraps as a single unit on narrow viewports instead of the inner items wrapping individually.

### Files changed
- [includes/class-kdc-qtap-finance-block-editor.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-block-editor.php) — CSS fix for `.kdc-qtap-rx-date` width + new `.kdc-qtap-rx-daterange` nowrap sub-group; Date-row markup updated to wrap From/To inside the new span.

## [3.16.54] - 2026-04-24

### Fixed
- **Full Tenure retitle now runs synchronously on payment record**, not deferred to `shutdown`. `KDC_qTap_Finance_Admin::ajax_record_payment()` at [trait-kdc-qtap-finance-user-meta-payments.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-payments.php) previously handed `link_order_to_payment()` + `apply_allocation_breakup()` (which internally calls `apply_full_tenure_retitle()`) off to a `shutdown` callback after `fastcgi_finish_request()` so the admin saw the success response instantly. The tradeoff documented inline — *"if the deferred work throws, staff can hit 'Recompute Breakup' on the order to fix it"* — was the recurring friction admins were hitting: a freshly-recorded full-tenure payment would still read "1st Term" on the order header until they manually clicked Recompute. Now inline. Cost: ~50–200ms added to the success response. Wrapped in try/catch so a breakup exception still lets the transaction/order persistence succeed (same failure mode as before — logged and swallowed), but the success path now always produces a correct receipt.
- **Recompute Breakup no longer explodes Tuition into 12 monthly rows on per_tenure / per_cycle enrollments.** `apply_allocation_breakup()` at [trait-kdc-qtap-finance-wc-orders.php:1264](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-orders.php#L1264) iterates every Payment_Item contribution and, before v3.16.54, emitted one breakup row per item — so a per_tenure year's 12 monthly `per_month` tuition items produced 12 "Jun 2025 @ ₹10,000", "Jul 2025 @ ₹10,000", … rows instead of a single consolidated "Tuition Fee: ₹120,000". v3.16.54 detects the enrollment's `payment_cycle` via `KDC_qTap_Finance_Enrollment::get()`; when it's `per_tenure` or `per_cycle`, the contribution rows are aggregated by slab (sum deltas, span min-start / max-end periods, take lowest sort_order, first-seen fee_type) and emitted as ONE row per slab with just the currency value (no "@ period" prefix). Per_term / per_month enrollments keep the old per-item granularity — admins reading those receipts want each term/month visible.

### Changed
- **Staff Console → Receipts filter bar layout polish** at [class-kdc-qtap-finance-block-editor.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-block-editor.php):
  - **Rows are now a 2-column CSS grid** (`grid-template-columns: 72px 1fr`) with a new `.kdc-qtap-rx-row__content` flex container inside. Wrapped pills (Status / Source) stay inside the content column instead of spilling under the uppercase label.
  - **Date range inline** on one row — `<select>` + From (date) + To (date) side-by-side at a compact 150px each. Previously the date inputs wrapped to their own lines because of `flex-wrap: wrap` on the daterange span, ballooning the DATE row height.
  - **Mobile `<details>` wrapper**. The filter bar is wrapped in a native `<details class="kdc-qtap-rx-wrap">` with a summary bar that reads "Filters [N active]" where N is the number of applied chips. On mobile (≤ 720px) the wrap is closed by default and the summary is visible. On desktop (≥ 721px) the summary is hidden via CSS and a small inline script sets the `open` attribute to keep the filters permanently expanded. The script also listens for viewport changes and flips state on resize; a `toggle` listener on the details element records "user has explicitly clicked" so the script doesn't undo an intentional mobile expand.
  - **Mobile single-column layout**. At ≤ 720px the grid collapses to `1fr` so labels stack above their content rows — labels that were fixed-width on desktop now read as section headers on narrow viewports.
  - **Status pills sorted alphabetically** via `asort( $all_statuses, SORT_NATURAL | SORT_FLAG_CASE )` so the order no longer depends on which plugin registered the status first.

### Files changed
- [includes/traits/trait-kdc-qtap-finance-user-meta-payments.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-payments.php) — removed the `add_action( 'shutdown', … )` wrapper around `apply_allocation_breakup()`; now runs inline inside the same try/catch the deferred path used.
- [includes/traits/trait-kdc-qtap-finance-wc-orders.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-orders.php) — `apply_allocation_breakup()` detects `payment_cycle` once per call and, when per_tenure / per_cycle, aggregates contribution rows by slab before emitting the breakup rows. Also suppresses the "@ period" prefix for the collapsed output.
- [includes/class-kdc-qtap-finance-block-editor.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-block-editor.php) — filter bar markup rewritten around the new 2-column grid; CSS `<style>` block rewritten with the grid + `<details>` + mobile-collapsible rules; tiny inline viewport-watching script added; status pills sorted alphabetically.

## [3.16.53] - 2026-04-24

### Changed
- **Staff Console → Receipts tab: table header polish.** The "Customer" column header now renders the configurable `kdc_qtap_finance_label('student')` label (so sites that renamed "Student" to "Member" / "Player" / etc. see the right word). The "Receipt #" header shortens to "Receipt" since the date now shares the same cell. Only the staff-console receipts table is touched — the user-detail orders table at the same file further down (line 1726-ish) still reads "Customer" / "Receipt #" because that surface is a different context.
- **Receipt cell now shows the Receipt Date** as a muted second line below the receipt number. Date is computed via the same `payment_date` meta → `date_paid` → `date_created` fallback chain used by the admin Receipts tab, so both surfaces always agree on the rendered date. Uses the site's configured `date_format` (not a hard-coded Y-m-d).
- **Filter bar visual rework.** Inline styles that had accumulated on the v3.16.52 markup are collected into a dedicated `<style>` block with proper CSS classes (`kdc-qtap-rx-filters`, `kdc-qtap-rx-row`, `kdc-qtap-rx-pill`, `kdc-qtap-rx-chip`, etc.). The form is now a soft-shadowed rounded card; rows are separated by subtle bottom borders; labels use small-caps uppercase for stronger hierarchy without shouting.
- **Status + Source checkboxes become pill buttons.** Native checkboxes are visually hidden (still keyboard-accessible via focus-visible outline); the surrounding `<label>` renders as a rounded pill. Selected state uses the modern CSS `:has(input:checked)` pseudo-class so the pill swaps to a green-filled look the instant the user ticks it — no JS round-trip, no wait for form submit. Supported in Chrome 105+/Safari 15.4+/Firefox 121+, matches the WP evergreen-browser target.
- **Date range now renders inline** (From/To side-by-side in a single `.kdc-qtap-rx-daterange` span) instead of stacking on two separate lines.
- **Apply + Reset anchor right** of row 1 via `margin-left: auto`.
- **Active filter chips** swap from the grey "neutral" look to a warm amber (matches the on-hold status palette) so they visually stand out as temporary overlay state rather than blending back into the filter bar.

### Files changed
- [includes/class-kdc-qtap-finance-block-editor.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-block-editor.php) — `render_staff_console_receipts()` table header renames; receipt-date computation + rendering in the Receipt cell; filter bar markup rewritten with the new CSS class hierarchy; new `<style>` block inlined above the filter card.

## [3.16.52] - 2026-04-24

### Changed
- **Staff Console → Receipts tab: filter bar rewritten from scratch** at [class-kdc-qtap-finance-block-editor.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-block-editor.php). The previous design — a single `search` text input paired with a collapsible `<details>` containing Status + Source checkboxes — was replaced with an always-visible three-row layout inside one form:
  - **Row 1 — Targeted query.** `Search in: [parameter ▾]` dropdown paired with a text input. Parameters: `All fields`, `{Student} Name`, `Payee Name`, `Receipt Number`, `Order ID`, `UTR / Txn ID`, `Payment Method`, `Items (purchased)`. "All fields" keeps the pre-existing multi-field union semantics (WC native `s` + customer user lookup + canonical order metas) and **now also searches line-item names** via the new items-table helper. Every other parameter narrows to that one field only — so "Receipt Number: 0042" no longer spuriously matches payee names containing "0042".
  - **Row 2 — Date range.** `Date: [Receipt Date | Order Date ▾]` + From / To date inputs. Receipt Date filters on the `payment_date` meta via `meta_query BETWEEN` (CHAR compare — ISO-8601 YYYY-MM-DD sorts lexically); Order Date filters on `date_created` via the native `wc_get_orders` arg.
  - **Row 3 — Status + Source pills.** Always visible (not collapsed). Same checkbox semantics as before. All registered WC statuses are listed (including custom statuses contributed by other plugins).
- **"Active filters" chip row below the bar.** Each applied filter (query, date range, non-default status, non-default source) renders as a pill with an × link that clears just that one filter while preserving the rest of the URL. "Reset all" button clears every filter in one click.
- **Legacy `?search=` param aliased.** Bookmarked / cached URLs from the v3.16.41 era still load — the old param is read into the new `q` field when `q` itself isn't present.

### Added
- **`search_orders_by_field( $field, $token, $base_args )`** — new protected static on `KDC_qTap_Finance_Block_Editor`. Per-field dispatcher that knows the canonical data source for each parameter (user-table search, order meta, WCPDF receipt meta, order-items table, etc.). `field = 'all'` delegates to the pre-existing `search_orders_by_token()` union (plus the new items-table search for "All fields" coverage parity).
- **`search_orders_by_items( $token, $base_args )`** — new protected static that runs a direct `SELECT DISTINCT order_id FROM wp_woocommerce_order_items WHERE order_item_name LIKE …` then re-filters by status when provided. Works for HPOS + legacy (both backends share the `order_items` table).

### Files changed
- [includes/class-kdc-qtap-finance-block-editor.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-block-editor.php) — `render_staff_console_receipts()` rewrites parameter parsing, query-building, and the filter-bar markup; two new static helpers (`search_orders_by_field`, `search_orders_by_items`).

## [3.16.51] - 2026-04-24

### Changed
- **Replaced the vendor-specific "Zaakpay only" toggle** (v3.16.50) **with a generic "Created Via" checkbox pill** — qTap Finance is a commercial plugin that ships to sites running any gateway, so the primary filter on top of date / source should classify *how the order was received*, not identify one specific gateway by meta key. The new pill mirrors the parent qTap App plugin's canonical six-channel taxonomy at [class-kdc-qtap-woocommerce-orders-admin.php:140-146](kdc-qtap/includes/class-kdc-qtap-woocommerce-orders-admin.php#L140-L146): **Checkout**, **Admin**, **WCPOS**, **WhatsApp**, **REST API**, **Other**. All six default checked — zero-config state is "show everything".
- **Detection logic duplicated from parent plugin** (intentionally — keeps the finance release self-contained rather than forcing a coordinated parent bump). The new `detect_order_via()` private helper on the Receipts trait copies the parent's priority ladder verbatim: WCPOS (meta marker + `created_via` stripos + meta-key regex) → WhatsApp (`created_via` + WA-specific meta + custom `created_via` meta_data entry) → Admin (explicit `admin` or empty `created_via`) → Checkout (`checkout`/`store-api`) → REST API (`rest` stripos) → Other. Kept in sync via a docblock cross-reference.
- **`resolve_receipt_filters_from_request()` restructured** — `zaakpay_only` bool removed; `via` array added keyed by channel slug. The v3.16.50 `zaakpay` POST param is silently ignored (no browser-cache compat needed — feature lived for one release cycle). The sources + paywith_methods groups are unchanged.
- **`build_receipt_row()` + the ZIP filter loop** — Zaakpay meta check replaced with the 6-way `detect_order_via()` classification. The paywith_method multi-select is now always applied when non-empty (previously skipped while Zaakpay-only was on — that mutual-exclusion no longer applies). Each returned row now carries a `via` key, available for future UI enhancements (e.g., a compact "Via" column in the table).
- **`query_receipt_order_ids()` signature reverted** — the `$zaakpay_only` parameter added in v3.16.50 is removed. Created Via detection happens in PHP post-query because the heuristics span column values + multiple meta keys + meta_data iteration — too much to express cleanly as a meta_query, and post-filtering is fast enough for the bounded date-range result set.

### UI
- New `.kdc-receipts-via-group` dashed-border pill at [assets/css/kdc-qtap-finance-receipts.css](kdc-qtap-finance/assets/css/kdc-qtap-finance-receipts.css), visually matching the existing Source pill. The (now-removed) Zaakpay-specific styling + paywith visibility toggle are cleaned up.

### Files changed
- [includes/traits/trait-kdc-qtap-finance-admin-tab-receipts.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-admin-tab-receipts.php) — Zaakpay checkbox swapped for the six Via checkboxes in `render_receipts_tab()`; new `$via_channels` property + `detect_order_via()` helper; filter resolver + query + build_receipt_row + ZIP post-filter all updated to the Via shape. Each row now emits `via` for client consumption.
- [assets/js/kdc-qtap-finance-receipts.js](kdc-qtap-finance/assets/js/kdc-qtap-finance-receipts.js) — `readViaFlags()` replaces the Zaakpay toggle read; `readPaywithMethods()` split out and used unconditionally; `syncPaywithVisibility()` removed (no longer needed); both list and ZIP requests send `via_<channel>` flags.
- [assets/css/kdc-qtap-finance-receipts.css](kdc-qtap-finance/assets/css/kdc-qtap-finance-receipts.css) — Zaakpay styling dropped; new `.kdc-receipts-via-group` block added.

## [3.16.50] - 2026-04-24

### Added
- **Receipts tab — "Zaakpay only" toggle.** Adds a checkbox in the toolbar that restricts the list (and the ZIP bundler) to orders marked with `_zaakpay_order=yes` meta — the flag the Zaakpay gateway plugin stamps at checkout completion. Identifies gateway-paid orders reliably even when staff have later overwritten `paywith_method` to something like "Online - Other" or a channel-specific label. Applied at the SQL layer via a `meta_query` AND-clause on the primary `payment_date`-range query; the legacy fallback query is skipped when Zaakpay-only is on (Zaakpay orders always carry `payment_date` so the union would match zero rows there anyway).
- **Receipts tab — "Method" multi-select filter** powered by the distinct `paywith_method` values actually present on the site. `get_distinct_paywith_methods()` queries `{prefix}wc_orders_meta` first (HPOS-native) and falls back to `wp_postmeta` when HPOS isn't storing data — so the dropdown auto-reflects whatever methods the site is actually using (Cash, UPI, Cheque, gateway titles, etc.). Values are lowercased for case-insensitive matching — "Cash", "cash", and "CASH" collapse to one logical option while the UI keeps the first-seen casing as the display label. Hidden when "Zaakpay only" is on (the two filters are mutually exclusive by design). An "All" shortcut button re-selects every option with a single click.
- **New unified filter resolver** `resolve_receipt_filters_from_request()` on the Receipts trait — returns a struct with `sources`, `zaakpay_only`, and `paywith_methods` in one call. The old `resolve_receipt_sources_from_request()` is preserved as a thin shim that extracts just `sources`, so pre-v3.16.50 call sites keep working.
- **`query_receipt_order_ids()` gains a `$zaakpay_only` parameter** that injects the `_zaakpay_order=yes` meta AND-clause at SQL time.

### Changed
- **`build_receipt_row()` signature** — third argument is now the full filters struct (`sources` + `zaakpay_only` + `paywith_methods`). A legacy-shape detection at the top auto-promotes any caller still passing a plain sources array into the struct, so no downstream code breaks.
- ZIP handler narrows by Zaakpay flag + method filter BEFORE the WCPDF render loop, mirroring the list endpoint — no wasted PDF regeneration for orders the filter set would exclude.

### UI
- New `.kdc-receipts-paywith-group` toolbar group at [assets/css/kdc-qtap-finance-receipts.css](kdc-qtap-finance/assets/css/kdc-qtap-finance-receipts.css) — dashed-border pill matching the Source group's visual weight, containing the Zaakpay checkbox and the method multi-select.

### Files changed
- [includes/traits/trait-kdc-qtap-finance-admin-tab-receipts.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-admin-tab-receipts.php) — new toolbar controls in `render_receipts_tab()`; new `get_distinct_paywith_methods()` + `resolve_receipt_filters_from_request()` helpers; extended `query_receipt_order_ids()` with `$zaakpay_only`; `build_receipt_row()` accepts the full filters struct and applies the zaakpay + method filters post-row.
- [assets/js/kdc-qtap-finance-receipts.js](kdc-qtap-finance/assets/js/kdc-qtap-finance-receipts.js) — new `readPaywithFilter()` + `syncPaywithVisibility()` helpers; sends `zaakpay` + `paywith_methods[]` on both list and ZIP requests; auto-reload wired to all new controls; "All" button re-selects every option.
- [assets/css/kdc-qtap-finance-receipts.css](kdc-qtap-finance/assets/css/kdc-qtap-finance-receipts.css) — new `.kdc-receipts-paywith-group` + method-select styling.

## [3.16.49] - 2026-04-24

### Fixed
- **WCPDF Receipt/Invoice Payment Date no longer shifts by ±1 day** when the server timezone differs from the WP timezone. Previously, `pdf_receipt_payment_details()` at [trait-kdc-qtap-finance-wc-helpers.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-helpers.php) parsed the bare `payment_date` meta via `strtotime($meta . ' 00:00:00')` (server TZ) and pushed the result into `$order->set_date_paid()` (stored as UTC), which WCPDF then formatted back in the WP TZ — a round-trip that could silently drop or gain a day depending on TZ differences.
- The Payment Date row is now rendered directly from the `payment_date` order meta via the WCPDF filter `wpo_wcpdf_payment_date`. The meta is parsed as midnight in `wp_timezone()` using `DateTimeImmutable::createFromFormat('!Y-m-d', …)` and formatted with `wcpdf_date_format($document, 'order_date_paid')` via `wp_date()` — so the rendered day is exactly the day in the meta, regardless of server TZ, UTC, or WP TZ.
- Side-effect removed: the receipt hook no longer mutates `$order->set_date_paid()` / `save()` on every PDF render. WCPDF output is unchanged for orders with no `payment_date` meta (falls through to default behaviour). Refund documents resolve Payment Date from the parent order's `payment_date` meta, matching WCPDF's own refund handling.

### Changed
- **Receipts tab date filter now runs on Receipt Date, not Order Date.** `KDC_qTap_Finance_Admin_Tab_Receipts::ajax_receipts_data()` and `ajax_receipts_download_zip()` at [trait-kdc-qtap-finance-admin-tab-receipts.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-admin-tab-receipts.php) previously queried orders via WC's `date_created` range — meaning an order created last week but only paid today wouldn't show up in a "today" filter even though its receipt is dated today. The new primary query is a `meta_query` BETWEEN on `payment_date` (the canonical meta the plugin stamps when a transaction is recorded, and the same value the Receipt Date column uses). A secondary query unions in orders WITHOUT `payment_date` meta falling inside the `date_created` range — this handles legacy orders pre-v3.16.17 that never received the stamp. Rows are sorted in PHP by the computed `receipt_date` after the union (WC's `orderby` doesn't survive a union).
- **New shared helpers** `resolve_receipt_sources_from_request()` + `query_receipt_order_ids()` on the Receipts trait — both AJAX endpoints (list + ZIP) consume the same date-range + source-flag logic instead of each re-implementing the WC query and checkbox plumbing.

### Added
- **Three-way Source filter on the Receipts tab — Fee / POS / Other.** Replaces the previous single "Include POS / non-fee orders" checkbox with three distinct checkboxes grouped under a "Source:" label in the toolbar. `build_receipt_row()` now accepts a `$sources` array (`{fee, pos, other}`) instead of the old `$include_non_fee` bool; an order's row is emitted only when at least one selected source matches. Same three-way precedent as the staff-console receipts block at [class-kdc-qtap-finance-block-editor.php:719-733](kdc-qtap-finance/includes/class-kdc-qtap-finance-block-editor.php#L719-L733). ZIP bundler narrows by source before the WCPDF render loop so we don't regen PDFs the filter would exclude. All three checkboxes default checked — so sites that previously had "Include POS / non-fee" ticked see no behavioural change.
- **Backward-compat shim** inside `resolve_receipt_sources_from_request()` — if an older browser cache posts the legacy `include_non_fee` param instead of the three new flags, it maps to `{fee:true, pos:true, other: <legacy bool>}`, preserving the previous semantics exactly until the page is refreshed.

### UI
- New `.kdc-receipts-source-group` styling at [assets/css/kdc-qtap-finance-receipts.css](kdc-qtap-finance/assets/css/kdc-qtap-finance-receipts.css) — dashed-border pill containing the three source checkboxes plus a "Source:" label, visually distinct from the date-range controls without occupying a second toolbar row.

### Files changed
- [includes/class-kdc-qtap-finance-woocommerce.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-woocommerce.php) — registered a new `add_filter( 'wpo_wcpdf_payment_date', … )` next to the existing WCPDF action hooks.
- [includes/traits/trait-kdc-qtap-finance-wc-helpers.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-helpers.php) — removed the `set_date_paid` sync block from `pdf_receipt_payment_details()`; added `pdf_payment_date_from_meta( $formatted_date, $document )` which returns a site-TZ-anchored formatted string, bypassing WCPDF's `date_paid`-based computation.
- [includes/traits/trait-kdc-qtap-finance-admin-tab-receipts.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-admin-tab-receipts.php) — UI replacement (three source checkboxes); rewritten list + ZIP AJAX handlers around new helpers; `build_receipt_row()` signature updated to accept a `$sources` array.
- [assets/js/kdc-qtap-finance-receipts.js](kdc-qtap-finance/assets/js/kdc-qtap-finance-receipts.js) — reads three source checkboxes; sends `include_fee` / `include_pos` / `include_other` params on both list and ZIP requests; auto-reload bound to all three checkbox `change` events.
- [assets/css/kdc-qtap-finance-receipts.css](kdc-qtap-finance/assets/css/kdc-qtap-finance-receipts.css) — new `.kdc-receipts-source-group` block.

## [3.16.48] - 2026-04-24

### Added
- **"Receipt Date" column on the WooCommerce Orders admin list**, positioned immediately after the existing "Receipt #" column. Displays the order's `payment_date` meta (the canonical receipt date the plugin writes when `KDC_qTap_Finance_Payment::record_transaction()` fires) formatted as `YYYY-MM-DD`. Falls back to WooCommerce's `date_paid` and then `date_created` when `payment_date` is absent — so legacy / pre-plugin orders still surface a meaningful date instead of a blank cell. Shows a muted em-dash only when none of the three are available.
- Works on both storage backends: HPOS (`manage_woocommerce_page_wc-orders_custom_column` at [class-kdc-qtap-finance-wc-orders-admin.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-wc-orders-admin.php)) and legacy post-type (`manage_shop_order_posts_custom_column`). Column width is fixed at 105px with `white-space: nowrap` to keep the table compact.

### Files changed
- [includes/class-kdc-qtap-finance-wc-orders-admin.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-wc-orders-admin.php) — new `COLUMN_RECEIPT_DATE_KEY` constant; `add_receipt_column()` now injects both Receipt # and Receipt Date columns in order; `render_column_legacy()` recognises the new key; `render_column()` dispatches to a new `render_receipt_date_cell()` helper; admin CSS gains a width rule for the new column.

## [3.16.47] - 2026-04-22

### Fixed
- **Report tab DataTables "Total" footer row was blank for every numeric column.** The column-builder at [kdc-qtap-finance-report.js:743-752](kdc-qtap-finance/assets/js/kdc-qtap-finance-report.js#L743) created `dtColumns` entries with `title` / `data` / `className` / optional `render` — but never propagated the server-side `col.type` ("collection" / "received" / "balance" / etc.) onto the DT column object. The `footerCallback` at [kdc-qtap-finance-report.js:813-850](kdc-qtap-finance/assets/js/kdc-qtap-finance-report.js#L813) then checked `numericTypes[ col.type ]` — which was always `undefined` because the property didn't exist — so every numeric cell fell through to the empty-string branch. **Fix**: column type now carries through via a custom `_kdcType` key (DT's own `type` option is reserved for sort-type hints — setting it to domain labels like "collection" confuses the sort algorithm). Footer reads `col._kdcType` and sums correctly. Both the original `tfoot` row and the `.dataTables_scrollFootInner` clone get written.

### Added
- **New setting "Auto-apply User Credit"** on the Finance → Institute settings tab. Toggles whether `KDC_qTap_Finance_Payment::record_transaction()` auto-consumes the user's parked `kdc_qtap_finance_credit` balance against the remaining amount on a regular-fee Payment. Default **ON** — preserves v3.15.48+ behaviour so existing sites don't observe a silent change. When OFF, credit stays parked until an admin explicitly applies it via a `Payment Method = Credit` transaction. Custom / user-fee / bracket-prefixed slabs are never auto-consumed regardless of this setting.
- **Setting sanitizer** + default at [trait-kdc-qtap-finance-admin-settings.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-admin-settings.php) — `auto_apply_credit` stored on the main `kdc_qtap_finance_settings` option.
- **Settings UI** at [trait-kdc-qtap-finance-admin-tab-institute.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-admin-tab-institute.php) — checkbox + descriptive help text explaining the three regular-slab fee-type targets (`per_month` / `per_term` / `per_tenure`) and that non-regular slabs are excluded.

### Changed
- **Regular-fee trickle boundary enforced.** `trickle_excess_forward()` at [class-kdc-qtap-finance-payment.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-payment.php) now inspects the source Payment's `slab`. If the source is non-regular (starts with `_custom_` / `_user_fee_` / `[`), any excess is parked as user credit immediately — no attempt to cascade into the next pending regular-fee payment. This closes a boundary leak where an educational-trip overpayment could auto-apply to next term's tuition via the existing `find_next_pending_regular()` lookup. Regular → regular cascading is unchanged.
- **Auto-credit consumption inside `record_transaction()` gated on the new `auto_apply_credit` setting.** The existing `$is_regular && 'credit' !== $payment_method` guards remain; the new setting adds a third guard. Skipped auto-consumption paths don't write a `credit_parked` stamp because no credit was consumed.

### Deferred
- **Term Split checkbox on the Report tab** — pending clarification. The Reports tab already exposes a "Reporting Breakup: Term-wise / Month-wise" dropdown; confirm what the Term Split checkbox should do on top of that selector before I add anything.

### Files changed
- [assets/js/kdc-qtap-finance-report.js](kdc-qtap-finance/assets/js/kdc-qtap-finance-report.js) — column-builder stamps `_kdcType`; footer callback reads it instead of the phantom `col.type`.
- [includes/traits/trait-kdc-qtap-finance-admin-settings.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-admin-settings.php) — sanitize `auto_apply_credit` with default-ON.
- [includes/traits/trait-kdc-qtap-finance-admin-tab-institute.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-admin-tab-institute.php) — new checkbox UI.
- [includes/class-kdc-qtap-finance-payment.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-payment.php) — `record_transaction()` reads setting and gates auto-credit; `trickle_excess_forward()` parks non-regular excess as credit instead of cascading.

## [3.16.46] - 2026-04-22

### Fixed
- **"Error updating transaction." on every Edit Transaction save.** The v3.16.36 Edit Transaction enhancements added a Reference/UTR field and, as part of that, included a `reference` column in the direct `$wpdb->update()` payload against `kdc_qtap_finance_transactions`. But that column doesn't exist on the transactions table — the schema at [class-kdc-qtap-finance-database.php:308-333](kdc-qtap-finance/includes/class-kdc-qtap-finance-database.php#L308-L333) never defined one (columns are `payment_id / amount / credit_parked / payment_method / payment_method_title / payment_date / receipt_file / attachment_id / wc_order_id / parent_transaction_id / verification_status / verified_by / verified_at / transaction_date / notes / created_at / created_by`). MySQL rejected every update with `Unknown column 'reference' in 'field list'`, `$wpdb->update()` returned `false`, and every save surfaced the generic "Error updating transaction." error. **Fix**: the `reference` field is no longer included in the transactions-table update payload. Instead, when the admin enters a value, the handler writes it to the linked WC order's `pay_utr` meta — the same canonical store the PDF renderer reads (shared with Record Payment's flow). An empty form submit is a no-op on `pay_utr` so a real gateway `transaction_id` or prior UTR is never clobbered by saving a transaction without touching the Reference field.
- **Edit Transaction button's `data-reference` attr read `$txn->reference`**, a phantom property on transaction rows (no column → always `null` → serialized as empty string). The modal's Reference/UTR field therefore always opened blank even for transactions that DID have a UTR on record. **Fix**: the attr now reads the linked WC order's `pay_utr` meta directly and skips synthetic `TXN-N` placeholders (internal fallbacks the admin would never want to re-submit as the reference). Real UTRs — whether entered manually during Record Payment, stamped by a payment gateway, or imported via CSV — populate the field correctly.

### Tech notes
- The Payee Name save at [trait-kdc-qtap-finance-user-meta-payments.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-payments.php) was refactored alongside the reference fix — both payee-name and pay_utr writes now share a single `$wc_order->save()` call instead of two redundant saves in the same handler.

### Files changed
- [includes/traits/trait-kdc-qtap-finance-user-meta-payments.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-payments.php) — remove `reference` from the `$wpdb->update()` payload and route it into the WC order's `pay_utr` meta.
- [includes/traits/trait-kdc-qtap-finance-user-meta-rendering.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-rendering.php) — Edit button `data-reference` attr pulls from WC order `pay_utr` meta (skipping `TXN-N` synthetics) instead of phantom `$txn->reference` property.

## [3.16.45] - 2026-04-22

### Fixed
- **Receipt "UTR / Ref" row showed our synthetic `TXN-N` placeholder instead of the real gateway transaction id** (e.g., Zaakpay `ZP64fddc3b0e291`). The PDF renderer's previous logic at [trait-kdc-qtap-finance-wc-helpers.php::pdf_receipt_payment_details()](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-helpers.php) was:
  ```
  utr = pay_utr
  if utr is empty → fall back to transaction_id
  ```
  If `sync_receipt_metas_from_transaction()` had to write the `TXN-N` synthetic fallback to `pay_utr` (because no transaction.reference / receipt_file was known at record-payment time), that synthetic preempted the real `transaction_id` the WC payment gateway later stamped — a real reference is always more useful on a receipt than an internal `TXN-N` string. **Fix**: the renderer now resolves the UTR via a three-step preference:
  1. If `transaction_id` is a valid non-synthetic gateway reference (non-empty, not `na`, not prefixed `TXN-`) AND `pay_utr` is either empty or a `TXN-N` synthetic → use `transaction_id`.
  2. Otherwise use `pay_utr` when set (this preserves manually-entered UTRs / offline references that ARE real).
  3. Fall back to `transaction_id` in any remaining case.
  4. Skip the row entirely when nothing usable is available.

### Confirmed (no code change needed)
- **`sync_receipt_metas_from_transaction()` already guards incoming gateway data** at [trait-kdc-qtap-finance-wc-helpers.php:475,494](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-helpers.php#L475). `pay_utr` is only written when the existing value is empty or literally `'na'`; `transaction_id` is only written when the existing value is empty. Real gateway stamps (WC's native `$order->set_transaction_id()` at payment completion, Zaakpay's `ZP…`, etc.) are never overwritten by the plugin's sync.

### Files changed
- [includes/traits/trait-kdc-qtap-finance-wc-helpers.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-helpers.php) — `pdf_receipt_payment_details()` UTR/Ref resolution rewritten for the three-step preference.

## [3.16.44] - 2026-04-22

### Added
- **"Only Exempt" filter on the WP Users admin list.** Checkbox alongside the existing Year / Grade / Division filter controls. When ticked:
  - paired with a year → restricts to users whose enrollment FOR THAT YEAR is marked exempt
  - with no year selected → restricts to users with an exempt enrollment in ANY year
  Label auto-adapts to the institute's configured `kdc_qtap_finance_label('exempt')` (labeled as "Exception" in the screenshot from the user who reported this — historically "RTE" — shows whatever the plugin setting says).
- **New single-row meta `kdc_qtap_finance_exempt_index`** — mirrors the v3.16.28 `kdc_qtap_finance_enrollment_index` pattern. Format: `|YEAR|YEAR|` with leading + trailing pipes containing only the years where the user's enrollment is exempt (reads both canonical `exempt` key and legacy `is_rte` key for backward compat). Absent when no enrollments are exempt. Independent of the main index key so existing Year / Grade / Division filters are unchanged.
- **New helpers on `KDC_qTap_Finance_Enrollment`**:
  - `INDEX_META_KEY_EXEMPT` class constant (`'kdc_qtap_finance_exempt_index'`)
  - `build_exempt_index( array $enrollments )` static — pure function returning the compact index string from a decoded enrollments map. Reads both `exempt` and legacy `is_rte`.
  - `write_enrollment_index()` extended to write both the main index AND the exempt index in lockstep, deleting the exempt row when the user has no exempt enrollments.

### Changed
- **`handle_users_list_sorting()`** at [trait-kdc-qtap-finance-user-meta-associations.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-associations.php) now reads the `qtap_exempt` GET param. When set, adds a second `meta_query` clause targeting `kdc_qtap_finance_exempt_index` alongside the existing Year/Grade/Division clause. The two clauses combine via `AND` so all active filters must match.
- **Year no longer auto-defaults to current** when ONLY the Exempt filter is active — enables the "any year exempt" matching described above. When Grade/Division filters are also active, the year fallback to current year is preserved.

### Post-upgrade cleanup
- **One-time migration `migrate_exempt_index_3_16_44()`** runs on first load after upgrade. Iterates every user with a `kdc_qtap_finance_enrollments` meta row in batches of 500, calls `write_enrollment_index()` on each (which now writes both indexes), then flushes the user_meta cache group. Version-gated on `kdc_qtap_finance_exempt_index_3_16_44_done` + try/catch-wrapped. Debug log reports users scanned + exempt users found.

### Files changed
- [includes/class-kdc-qtap-finance-enrollment.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-enrollment.php) — new `INDEX_META_KEY_EXEMPT` constant + `build_exempt_index()` helper; `write_enrollment_index()` writes both indexes in lockstep.
- [includes/traits/trait-kdc-qtap-finance-user-meta-associations.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-associations.php) — `handle_users_list_sorting()` reads `qtap_exempt` + adds the second meta_query clause; `render_users_list_filters()` adds the "Only Exempt" checkbox.
- [kdc-qtap-finance.php](kdc-qtap-finance/kdc-qtap-finance.php) — new `migrate_exempt_index_3_16_44()` + version-gated runner.

## [3.16.43] - 2026-04-22

### Fixed
- **`recompute_order_allocation_breakup()` silently skipped multi-fee orders.** The function resolved `$user_id` + `$year` ONLY via the first line item's singular `_kdc_qtap_finance_payment_id` meta (at [trait-kdc-qtap-finance-wc-orders.php:1431](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-orders.php#L1431)). Multi-fee items (created by `create_multi_fee_order()`) only carry the plural `_kdc_qtap_finance_payment_ids` JSON array — the singular lookup returned 0, the `if ( ! $user_id || '' === $year ) return 0` gate fired, and the breakup rows never got rewritten. Observable symptom on v3.16.42's new "Rebuild Payment Items from Enrollment (force)" bulk action: admin notice reported `recomputed 1 WC order(s)` but the line-item `breakup_items` meta stayed stale, because `recompute_order_allocation_breakup` early-exited before calling `apply_allocation_breakup`. Same class of bug I fixed in `apply_allocation_breakup` itself at v3.16.34 — the sibling recompute path needed the same treatment.
- **Fix**: lookup now tries three sources in order:
  1. Item-level `_kdc_qtap_finance_user_id` + `_kdc_qtap_finance_academic_year` metas (stamped by every order creator — most reliable).
  2. Singular `_kdc_qtap_finance_payment_id` → `Payment::get()` → user_id / academic_year.
  3. Plural `_kdc_qtap_finance_payment_ids` JSON (with `maybe_unserialize` fallback for legacy serialised arrays) → first resolvable Payment → user_id / academic_year.
  First successful source wins. Recompute now fires correctly on multi-fee orders, and the rebuild action's breakup rewrite step lands properly.

### Files changed
- [includes/traits/trait-kdc-qtap-finance-wc-orders.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-orders.php) — `recompute_order_allocation_breakup()` now reads item-level metas first, with singular + plural Payment-id lookups as fallback.

## [3.16.42] - 2026-04-22

### Added
- **Bulk action "qTap: Rebuild Payment Items from Enrollment (force)"** on the WC Orders list (HPOS + legacy). Addresses cases where the installment generator stored wrong per-period amounts on Payment_Item rows — e.g. a ±X% tuition discount that should apply to all 6 monthly tuition items only landed on some, so the receipt shows Jun at ₹17,060 but Jul–Nov at ₹3,412. The overall Payment total can still be correct by coincidence while the per-period breakup is misleading.

  This action resolves each selected order's linked Payment(s) — reading both the singular `_kdc_qtap_finance_payment_id` and the plural `_kdc_qtap_finance_payment_ids` JSON meta — and routes each Payment through the new `KDC_qTap_Finance_Enrollment::rebuild_payment_items_for_payment()` helper, which:
  1. Reads the current enrollment state (`grade`, `fee_slabs`, `adjustments`).
  2. Calls `KDC_qTap_Finance_Installment_Generator::generate_term_payments()` and `apply_adjustments_to_term_data()` to produce fresh per-period items with adjustments applied uniformly.
  3. Matches the Payment's `term_key` (or falls back to `period_start`) to the correct billing period in the regenerated schedule.
  4. **Deletes every existing Payment_Item** on the Payment (ignoring the `covered_items` paid-item guard the auto-sync uses).
  5. Inserts fresh Payment_Items from the regenerated schedule.
  6. Recomputes `Payment.amount_due` from the new items, caps `Payment.amount_paid` to the new due, and parks any overpayment excess on the user's credit balance via `kdc_qtap_finance_add_user_credit()`.
  7. Re-runs `Payment::allocate_payment_to_items()` to redistribute `amount_paid` across the fresh items via the waterfall.
  8. Triggers `recompute_order_allocation_breakup()` on every linked WC order so the receipt re-renders with the corrected per-period amounts.

  Skips Payments whose `slab` starts with `_user_fee_` (manually-created user fees aren't managed by the fee matrix and shouldn't be regenerated). De-dupes Payment IDs when multiple selected orders share a Payment. Capability gate: `manage_options` OR `manage_woocommerce`. Admin notice reports `{N} Payment(s) rebuilt with {M} fresh items; recomputed {K} WC order(s). Overpayment credited: ₹X. ({E} failed.)`.

### Added (helper)
- **`KDC_qTap_Finance_Enrollment::rebuild_payment_items_for_payment( $payment_id )`** — static helper encapsulating the force-regen flow above. Returns `{ success, new_items, new_due, excess_credited, orders_recomputed }` or a `WP_Error` on failure (not-found, no enrollment, no matching billing period, user-fee payment). Intended for explicit admin invocation only — NOT wired to any automatic trigger — since unconditionally rewriting paid items is an accounting-sensitive operation.

### Files changed
- [includes/class-kdc-qtap-finance-enrollment.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-enrollment.php) — new `rebuild_payment_items_for_payment()` static helper at the end of the class.
- [includes/class-kdc-qtap-finance-wc-orders-admin.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-wc-orders-admin.php) — new `'kdc_qtap_finance_rebuild_payment_items'` bulk action + handler + admin notice.

## [3.16.41] - 2026-04-22

### Fixed
- **Edit Enrollment modal ticked every available fee slab by default, ignoring the saved enrollment's actual selection.** Root cause: `ajax_get_available_slabs()` at [trait-kdc-qtap-finance-user-meta-enrollments.php:123](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-enrollments.php#L123) renders every checkbox with the `checked` attribute — intended as the default for NEW enrollments where admin typically wants all slabs enabled. The edit-path JS handler only `prop('checked', true)`'d matching slab slugs without ever un-checking the rest, so all stayed ticked. **Fix**: both edit-path JS call sites in [kdc-qtap-finance-user-profile-enrollments.js](kdc-qtap-finance/assets/js/kdc-qtap-finance-user-profile-enrollments.js) now reset every checkbox to unchecked via `prop('checked', false)` before re-ticking the subset matching the saved enrollment. New-enrollment flow unchanged (still relies on the server's all-checked default).

### Changed
- **Staff Console Reports search scope expanded.** The previous implementation only ran the search string through WC's native `s` param (billing name, order number). Staff who searched by a customer's user_login, email, explicit first / last name, the order's payee name meta, UTR, or a WCPDF receipt number often got empty results. New helper `KDC_qTap_Finance_Block_Editor::search_orders_by_token()` now unions matching order IDs from three sources per token:
  1. **WC native `s`** (unchanged scope)
  2. **Customer user lookup** — `user_login` / `user_email` / `user_nicename` / `display_name` via `get_users( 'search_columns' )` + `first_name` / `last_name` user_meta via `meta_query`, then filters orders by matching `customer_id`
  3. **Order-meta LIKE** across `_kdc_qtap_finance_payee_name`, `_kdc_qtap_finance_student_name`, `pay_utr`, `paywith_method`, `transaction_id`, `_wcpdf_receipt_number`, `_wcpdf_invoice_number`
  Multi-token semantics preserved — each whitespace-separated token must match somewhere on the order, tokens INTERSECT across sub-queries (so "John Smith" still requires both words to hit on the same order, possibly from different fields).

### Files changed
- [assets/js/kdc-qtap-finance-user-profile-enrollments.js](kdc-qtap-finance/assets/js/kdc-qtap-finance-user-profile-enrollments.js) — uncheck-all-first pattern added to both edit-enrollment slab-loading paths.
- [includes/class-kdc-qtap-finance-block-editor.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-block-editor.php) — new `search_orders_by_token()` helper + refactored search branch in the orders-list render.

### Deferred
- **Item breakup reflecting enrollment adjustments** — the code path at [class-kdc-qtap-finance-enrollment.php:517-518](kdc-qtap-finance/includes/class-kdc-qtap-finance-enrollment.php#L517-L518) already applies the enrollment's `adjustments` array to `$term['items'][]` amounts BEFORE Payment_Item creation via `apply_adjustments_to_term_data()`, so per-period `amount` values should already reflect discount / surcharge. If a specific receipt is showing unadjusted values, paste the receipt screenshot + the linked Payment row's `amount`, `amount_due`, `amount_paid` + the enrollment's `adjustments` array so I can trace the discrepancy — not enough signal yet to know whether this is a scheduling-time bug, a breakup-display bug, or an order that predates the adjustment being added.

## [3.16.40] - 2026-04-22

### Changed
- **Receipts tab: "Include POS / non-fee orders" checkbox ticked by default.** The initial load now includes every completed order in the date range (fee + POS + other non-fee) instead of restricting to fee-payment orders only. Admins can uncheck to restrict to fee-only when they need the narrower scope. Server query narrows the `wc_get_orders` call with `meta_key=_kdc_qtap_finance_is_fee_payment` only when the toggle is off — unchanged from v3.16.29.

### Files changed
- [includes/traits/trait-kdc-qtap-finance-admin-tab-receipts.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-admin-tab-receipts.php) — `<input type="checkbox" id="kdc-receipts-include-non-fee">` now ships with the `checked` attribute.

## [3.16.39] - 2026-04-22

### Fixed
- **Per-tenure enrollments weren't getting the Full Tenure title.** `apply_full_tenure_retitle()`'s coverage gate required the order's line items to reference EVERY regular Payment row for user+year. For an enrollment configured as `payment_cycle = per_tenure` with multiple fee slabs — e.g., PTA + Term Fee + Tuition Fee — the fee matrix could produce separate Payment rows per slab, and a single order legitimately linked to one Payment couldn't pass the gate (even though that one Payment aggregated items covering the whole year). **Fix**: gate is now skipped entirely when the enrollment's `payment_cycle` is `per_tenure`. A per-tenure enrollment is semantically "Full Tenure" regardless of fee-matrix structure — the coverage check was an artefact of the per-term / per-month design path. For per_term / per_month / per_cycle enrollments the gate still applies as before (the order must cover every regular Payment to promote to Full Tenure).

### Added
- **Bulk action "qTap: Apply Full Tenure title"** on the WC Orders list (HPOS + legacy). Forces `apply_full_tenure_retitle()` on every selected order. Intended as a manual override for orders the automatic gate skipped — including cases the gate *correctly* skipped by its per-term logic but staff want to rename anyway. Uses the same idempotent retitle path; orders whose name is already "Fees - Full Tenure - …" no-op the save. Admin notice reports `{N} line items retitled across {M} orders scanned`.

### Post-upgrade cleanup
- **Re-run of the sweep migration** under `kdc_qtap_finance_rebuild_item_names_3_16_39_done` so the per-tenure fix back-applies to existing orders on first admin page load after upgrade.

### Files changed
- [includes/traits/trait-kdc-qtap-finance-wc-orders.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-orders.php) — `apply_full_tenure_retitle()` now probes the enrollment's `payment_cycle` before the coverage gate and skips the gate for `per_tenure`.
- [includes/class-kdc-qtap-finance-wc-orders-admin.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-wc-orders-admin.php) — new `'kdc_qtap_finance_full_tenure_retitle'` bulk action + handler + admin notice.
- [kdc-qtap-finance.php](kdc-qtap-finance/kdc-qtap-finance.php) — new v3.16.39-gated re-run of `migrate_rebuild_order_item_names_3_16_33()`.

## [3.16.38] - 2026-04-22

### Added
- **Bulk action "qTap: Strip 'Fees -' prefix from line-item titles"** on the WC Orders list (HPOS + legacy). The v3.16.33 / v3.16.37 automatic non-regular-slab detection relies on the Payment row's `slab` column starting with one of the known prefixes (`_custom_`, `_user_fee_`, `[`). On this codebase, at least one more slab family — `_grade_fees` (used for grade-wise fees like Europe Trip installments) — exists and was slipping through. Rather than keep expanding the prefix allowlist, this release ships a direct manual tool:
  - Select any set of orders on the WC Orders list.
  - Pick **"qTap: Strip 'Fees -' prefix from line-item titles"** from the bulk actions dropdown.
  - The handler runs a `preg_replace` against every line item's name, stripping the configured `kdc_qtap_finance_label('fee', true)` prefix (e.g., "Fees - ") if present. No slab detection, no regular-vs-non-regular branching — if the title starts with the label, it's stripped.
  - Idempotent: the regex matches only the literal prefix, so re-running on already-stripped titles is a no-op save (WC item `set_name` on same value doesn't write).
  - Capability: `edit_shop_orders` or `manage_woocommerce`. Admin notice after the action reports `{N} line items stripped across {M} orders. {K} skipped.`

### Files changed
- [includes/class-kdc-qtap-finance-wc-orders-admin.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-wc-orders-admin.php) — new `'kdc_qtap_finance_strip_fees_prefix'` bulk action registration + handler branch + admin notice (reuses the existing bulk-action infrastructure from v3.16.20 / v3.16.21).

## [3.16.37] - 2026-04-22

### Fixed
- **Receipts tab date filter was silently no-op.** The `wc_get_orders` `date_created` argument was being constructed as `'>=2026-04-01T00:00:00...<=2026-04-15T23:59:59'` in both `ajax_receipts_data` and `ajax_receipts_download_zip`. WooCommerce's `OrdersTableQuery::process_date_arg` accepts:
  - a comparison string like `'>2018-01-01'`, OR
  - a range string like `'A...B'` (no operators inside),

  but **not** the combined `'>=A...<=B'` form I shipped — the malformed predicate failed to parse, the date constraint got dropped, and every matching order was returned regardless of the user's date range. The Receipts table showed orders outside the selected window, and the "Download PDFs (ZIP)" export bundled every fee order on the site instead of just the filtered slice. **Fix**: switched to UNIX-timestamp range form (`$from_ts . '...' . $to_ts`) which WC parses correctly. Both endpoints now honour the date inputs.
- **"Fees -" prefix sweep skipped multi-fee-order line items.** v3.16.33's `migrate_rebuild_order_item_names_3_16_33` resolved the line item's slab via:
  1. `_kdc_qtap_finance_slab` meta (not stamped by `create_multi_fee_order`), OR
  2. the singular `_kdc_qtap_finance_payment_id` → Payment.slab.

  Multi-fee orders only carry the plural `_kdc_qtap_finance_payment_ids` JSON array — both lookups failed, the slab was empty, the migration's `if ( '' === $slab ) continue;` short-circuit kicked in, and the "Fees -" prefix never got stripped. Same root-cause class as the v3.16.34 retitle fix (multi-fee items use a different meta shape than create_fee_order items). **Fix**: the slab lookup now reads both `_payment_id` (singular) and `_payment_ids` (plural JSON, with `maybe_unserialize` fallback for legacy serialised arrays), unions the candidates, and resolves the slab from the first matching Payment row. Re-runs the migration under a new `kdc_qtap_finance_rebuild_item_names_3_16_37_done` flag so historical multi-fee orders for non-regular slabs (Trip installments, custom-slab orders, user-fee charges) get corrected on first load after upgrade.

### Files changed
- [includes/traits/trait-kdc-qtap-finance-admin-tab-receipts.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-admin-tab-receipts.php) — `date_created` query arg now uses the `'A...B'` timestamp range form (both `ajax_receipts_data` + `ajax_receipts_download_zip`).
- [kdc-qtap-finance.php](kdc-qtap-finance/kdc-qtap-finance.php) — `migrate_rebuild_order_item_names_3_16_33` slab lookup now reads both singular + plural payment-id metas, with `json_decode` → `maybe_unserialize` defensive decoding. New v3.16.37-gated re-run of the sweep.

## [3.16.36] - 2026-04-22

### Added — Edit Transaction form parity with Record Payment
The Edit Transaction modal only captured amount / date / method / notes. Staff who needed to correct a payee name, a missing UTR, or attach a receipt they'd forgotten at submission time had to either delete + re-record (losing history) or edit the WC order by hand. Three new fields close the gap:

- **Reference / UTR** text input — pre-filled from the transaction's `reference` column via the new `data-reference` attr on the Edit button. Saved back to `transactions.reference`.
- **Payee Name** text input — pre-filled from the linked WC order's `_kdc_qtap_finance_payee_name` meta (falling back to `billing_first_name`). On save, routed through the existing `KDC_qTap_Finance_WooCommerce::apply_payee_name_to_order()` helper so the payee meta + `billing_first_name` mirror-write match Record Payment's semantics exactly.
- **UTR / Reference Upload** file input — accepts JPG/PNG/GIF/WebP/PDF up to 2MB. Reuses the same `handle_receipt_upload()` helper Record Payment uses. The form detects whether the transaction already has a receipt attached:
  - **Has receipt** → shows a "View current upload" link (opens in new tab via `Payment_Transaction::get_receipt_url()`) above the file input; choosing a new file replaces it, leaving the field empty keeps the existing file.
  - **No receipt** → the file input stands alone with upload-only UX.
  The Edit button carries the receipt URL via a new `data-receipt-url` attr so the JS can toggle the "View current" block without an extra AJAX fetch.
- **Post-save WC order sync**: when the transaction is linked to a WC order, the handler calls `sync_receipt_metas_from_transaction()` after the update so the order's `payment_date` / `paywith_method` / `pay_utr` metas pick up the edit — the WCPDF receipt re-renders with current values on next view.

### Changed — "Receipt" label renamed to "UTR / Reference Upload" globally
Single consistent label across every surface staff upload proof on:
- Record Payment modal ([trait-kdc-qtap-finance-user-meta-rendering.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-rendering.php))
- Edit Transaction modal (newly added, same label)
- Fees block frontend (the `receiptUpload` i18n key in [class-kdc-qtap-finance-block-editor.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-block-editor.php))
The helper text under each upload input also says "Upload proof image or PDF (max 2MB)" instead of "Upload receipt image or PDF".

### Tech details
- **Edit Transaction now submits via FormData** (JS) so the file upload lands on the server. Previously used `$.post()` which can't carry files.
- **AJAX handler** (`ajax_update_transaction`) now:
  - Accepts `reference`, `payee_name`, and `$_FILES['receipt_file']`.
  - Builds the `$wpdb->update()` payload dynamically so an empty `receipt_file` doesn't clobber the existing column value (same pattern as Record Payment's transaction patch).
  - Validates method + file constraints identically to Record Payment.
  - Wraps `apply_payee_name_to_order()` + `sync_receipt_metas_from_transaction()` behind `class_exists` / `method_exists` guards so older plugin versions of the linked order helper still fail gracefully.

### Files changed
- [includes/traits/trait-kdc-qtap-finance-user-meta-rendering.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-rendering.php) — Edit Transaction form markup (new fields + View-current link), Record Payment label rename, Edit button data attrs (`data-reference`, `data-payee`, `data-receipt-url`).
- [assets/js/kdc-qtap-finance-user-profile-payments.js](kdc-qtap-finance/assets/js/kdc-qtap-finance-user-profile-payments.js) — Edit Transaction open handler populates new fields and toggles the View-current link; Save handler uses FormData + includes file input.
- [includes/traits/trait-kdc-qtap-finance-user-meta-payments.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-payments.php) — `ajax_update_transaction` accepts + persists reference / payee_name / receipt_file and post-syncs the linked WC order.
- [includes/class-kdc-qtap-finance-block-editor.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-block-editor.php) — Fees block `receiptUpload` label renamed.

## [3.16.35] - 2026-04-22

### Added
- **"Download PDFs (ZIP)" button on Finance → Receipts.** Bulk-exports freshly-regenerated WCPDF receipts for the current date range as a single ZIP archive. Each receipt's cached WCPDF-Pro file is purged first (`_wcpdf_receipt_file` / `_wcpdf_invoice_file` meta deletes + on-disk unlink) and the document is re-rendered from current order data via `wcpdf_get_document( 'receipt', $order, true )` (falls back to `invoice` document type if `receipt` isn't configured). The PDF binary is read via `$doc->get_pdf()` and added to a server-side `ZipArchive` (temp-file backed to bound memory). Response is streamed back with `Content-Type: application/zip` + `Content-Disposition: attachment; filename="receipts_{from}_to_{to}.zip"` — the browser triggers the download via a JS blob-to-anchor pattern.
  - **Capped at 100 orders per request** to stay inside the typical 30-60s PHP execution window. If the date range matches more than the cap, the request returns a JSON error naming the match count and asks the admin to narrow the range.
  - **Same filter state as the rest of the tab** — respects the current `from` / `to` date inputs and the "Include POS / non-fee" checkbox.
  - **Filename de-dup guard** — if two orders produce the same WCPDF filename (rare but possible after renumbering), subsequent collisions get a `_2` / `_3` suffix so no entry gets overwritten inside the ZIP.
  - **Response headers carry the audit counts** — `X-KDC-Receipts-Added` and `X-KDC-Receipts-Skipped` — so the UI can show "{N} receipts in ZIP ({M} skipped)" after the download triggers.
  - **Graceful degradation**: clear error messages when WooCommerce, WCPDF, or PHP's `ZipArchive` extension is missing, when the date range is invalid, or when no orders match.
- **New AJAX action `kdc_qtap_finance_receipts_download_zip`** — capability `manage_options` or `manage_woocommerce`, nonce `kdc_qtap_finance_nonce`.

### Files changed
- [includes/traits/trait-kdc-qtap-finance-admin-tab-receipts.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-admin-tab-receipts.php) — new `ajax_receipts_download_zip()` handler + "Download PDFs (ZIP)" button markup.
- [assets/js/kdc-qtap-finance-receipts.js](kdc-qtap-finance/assets/js/kdc-qtap-finance-receipts.js) — `downloadZip()` client uses `fetch` + blob to trigger the browser download, with loading state + post-download status message.
- [includes/class-kdc-qtap-finance-admin.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-admin.php) — register new AJAX hook + add i18n keys (`zip_building`, `zip_failed`, `zip_added_suffix`, `zip_skipped_suffix`).

## [3.16.34] - 2026-04-22

### Fixed
- **Full Tenure retitle was silently skipping multi-fee-order line items even after v3.16.33's `payment_ids` plural read.** Investigation on order 6999 (Veer Bhupali, Nursery, 2026-2027): the Payment row data, line-item meta, and enrollment cycle all satisfied the coverage gate on paper, but the retitle never renamed the item. Root cause: v3.16.33 still anchored the group construction on `KDC_qTap_Finance_Payment::get( $pid )` — a single chain of lookups that could fail silently for reasons not fully pinned down (suspected combination of stale cache / unexpected meta serialisation on the specific customer's data). **Fix**: `apply_full_tenure_retitle()` now anchors directly from the item-level metas `_kdc_qtap_finance_user_id`, `_kdc_qtap_finance_academic_year`, and `_kdc_qtap_finance_grade` — values stamped on every line item by both `create_fee_order()` and `create_multi_fee_order()`. No Payment lookup is needed to learn the line item's user+year+grade. The Payment::get fallback is still in place for ancient orders that don't carry those item metas.
- **Recompute Breakup button did nothing for multi-fee orders.** `apply_allocation_breakup()` at [trait-kdc-qtap-finance-wc-orders.php:1068](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-orders.php#L1068) indexed existing line items by the singular `_kdc_qtap_finance_payment_id` only. For multi-fee orders (items carry `_kdc_qtap_finance_payment_ids` plural), that index came back empty and the function hit its `return 0` early-exit at line 1085 — never reaching the retitle call I moved outside the `$touched > 0` gate in v3.16.31. **Fix**: the index now reads both the singular and the plural (JSON-decoded, with `maybe_unserialize` fallback) payment IDs. Multi-fee orders build a non-empty index, `apply_allocation_breakup()` proceeds to the retitle, and Recompute Breakup now promotes the header on the first click.
- **Defensive decoding on the plural `payment_ids` meta.** Falls through `json_decode` → `maybe_unserialize` → skip, so whatever shape the meta value was stored in (JSON string, already-deserialised PHP array, or legacy serialised array), the retitle still picks up the IDs.

### Added
- **Debug-log diagnostics on retitle skips.** `apply_full_tenure_retitle()` now writes `kdc_qtap_debug_log` entries when it skips a group:
  - `"Full Tenure retitle skipped — no regular payments for user+year"` with `order_id` / `user_id` / `year`.
  - `"Full Tenure retitle skipped — coverage gate not met"` with the exact `regular_ids` + `covered` sets.
  Future mismatches can be traced from the debug log without shipping a fresh diagnostic build.
- **Safety net in `apply_allocation_breakup()`**: when the line-item → payment-id index is empty (all metas missing or unreadable), the function still calls `apply_full_tenure_retitle()` before returning 0. The retitle's own item-meta anchor can rename the item without needing the breakup rewrite, so the header gets corrected even in degenerate cases.

### Post-upgrade cleanup
- **Re-run of v3.16.33's sweep migration** (`migrate_rebuild_order_item_names_3_16_33`) under a new done-flag `kdc_qtap_finance_rebuild_item_names_3_16_34_done`. Orders that the v3.16.33 sweep silently skipped (order 6999 and any others with the same shape) are corrected on first load after upgrade — the fixed retitle now fires and rewrites the WC line-item header to "Fees - Full Tenure - {grade} [{year}]".

### Files changed
- [includes/traits/trait-kdc-qtap-finance-wc-orders.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-orders.php) — `apply_full_tenure_retitle()` anchors from item metas directly + emits diagnostic log on skip. `apply_allocation_breakup()` reads both singular + plural `payment_id(s)` for line-item indexing and calls retitle even when the index is empty.
- [kdc-qtap-finance.php](kdc-qtap-finance/kdc-qtap-finance.php) — v3.16.34-gated re-run of the sweep migration.

## [3.16.33] - 2026-04-22

### Fixed
- **Multi-fee-order line items weren't participating in the Full Tenure retitle.** `apply_full_tenure_retitle()` in [trait-kdc-qtap-finance-wc-orders.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-orders.php) only read the singular `_kdc_qtap_finance_payment_id` order-item meta. But orders built by `create_multi_fee_order()` stamp the plural `_kdc_qtap_finance_payment_ids` (JSON array of payment IDs — multiple Payment rows collapsed into one line item). Those items were skipped by the retitle grouping loop, so an order that genuinely covered the full year but was constructed via the multi-fee path kept its "1st Term [2026-2027]: Jun 2026 to May 2027" header forever. **Fix**: the grouping loop now reads both meta shapes — singular int + plural JSON array — decodes the array via `json_decode`, unions the IDs, and resolves user+year+grade from the first successfully-fetched Payment row. All IDs feed into the coverage check against `get_by_user_year()` so multi-payment items count toward the Full Tenure gate correctly.

### Changed
- **Retitle label honours the enrollment's `payment_cycle`.** `apply_full_tenure_retitle()` now loads `KDC_qTap_Finance_Enrollment::get( $user_id, $year )` and picks the label from a small cycle map:
  - `per_tenure` → "Full Tenure"
  - `per_cycle` → "Full Cycle" (matches `build_item_name()`'s create-time homogeneous-cycle branch)
  - `per_term` / `per_month` → "Full Tenure" (a per-term/month schedule that covers everything is still the "full tenure")
  - missing / unknown → falls back to "Full Tenure"
  Previously always stamped "Full Tenure" regardless of the user's configured cycle.
- **Non-regular fees no longer carry the "Fees -" prefix.** `build_item_name()` in [trait-kdc-qtap-finance-wc-orders.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-orders.php) now branches on whether the payment's `slab` is regular. Regular slabs (tuition, term fees, anything not prefixed `_custom_` / `_user_fee_` / `[`) keep the canonical `{Fees} - {category} - {grade} [{year}]` format. Non-regular slabs — educational trips, grade-wise custom slabs, user-fee ad-hoc charges — drop the "Fees" prefix entirely and render as `{category} - {grade} [{year}]`. Stands on its own: "Fees - Educational Trip - Nursery [2026-2027]" was misleading because the charge isn't a scheduled fee; it's a one-off item.

### Post-upgrade cleanup
- **One-time migration `migrate_rebuild_order_item_names_3_16_33()`** runs on first load after upgrade. For every order flagged `_kdc_qtap_finance_is_fee_payment=yes`:
  1. Re-runs `apply_full_tenure_retitle()` with the new multi-fee-order payment_ids read + payment_cycle-aware label. Orders the v3.16.31 sweep silently skipped or mislabelled are corrected.
  2. For each line item whose slab is non-regular (custom / user_fee / bracket-prefixed), strips the current "Fees -" prefix from the WC item name via a `preg_replace` targeting the configured `kdc_qtap_finance_label('fee', true)` value.
  Version-gated on `kdc_qtap_finance_rebuild_item_names_3_16_33_done` + try/catch with debug-log audit counts (`orders_scanned`, `full_tenure_named`, `prefix_dropped`). Idempotent: retitle's coverage gate + "same name → no-op" saves, and the regex strip matches only a literal "Fees - " prefix so re-running doesn't chomp further.

### Files changed
- [includes/traits/trait-kdc-qtap-finance-wc-orders.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-orders.php) — `apply_full_tenure_retitle()` reads both `payment_id` + `payment_ids`, picks label from enrollment's `payment_cycle`. `build_item_name()` branches on regular-vs-non-regular slabs to drop the "Fees -" prefix on the non-regular side.
- [kdc-qtap-finance.php](kdc-qtap-finance/kdc-qtap-finance.php) — new `migrate_rebuild_order_item_names_3_16_33()` + version-gated runner.

## [3.16.32] - 2026-04-22

### Changed — Receipts tab
- **Receipt # is now a clickable link** that opens the WCPDF receipt PDF in a new tab. Reuses the existing `KDC_qTap_Finance_WooCommerce::get_wcpdf_receipt_url()` helper — same endpoint the WC admin Orders table uses.
- **Student name is now a clickable link** to the user's profile (`user-edit.php?user_id=…`) in a new tab. Stamped from `$order->get_customer_id()` at row build time.
- **Grade and Division split into two separate columns.** Column titles pull from `kdc_qtap_finance_label('grade')` / `kdc_qtap_finance_label('division')` so the configured terminology surfaces. Single `grade_division` combined field retired from the row payload.
- **Column order swapped** so Flags sit before Items. The Items summary is the longest column on the row — keeping it as the rightmost ensures neighbouring columns stay readable.

### Fixed — Receipts tab
- **CSV export buttons were silently no-op.** DataTables Buttons' `csvHtml5` extension wasn't triggering downloads on the DT 2.1.8 + Buttons 3.1.2 build bundled with this plugin. Replaced the two `csvHtml5` buttons and the XLSX button with a single unified `exportBook( dt, ext, scope )` path that uses SheetJS (already enqueued; same library the Report tab relies on). Writes CSV and XLSX via `XLSX.writeFile` which handles format auto-detection from the file extension. Filenames preserved: `receipts_YYYY-MM-DD_to_YYYY-MM-DD.csv` / `..._all.csv` / `.xlsx`.

### Tech details
- **Column renderers** now follow the canonical `(data, type, row)` DataTables signature so export-time (`type === 'export'` / `'filter'`) returns clean raw strings while `type === 'display'` returns the linked, escaped HTML.
- **New row fields** on the AJAX payload: `receipt_url`, `student_url`, `grade`, `division`. Dropped: `grade_division` (concatenated form no longer needed).

### Files changed
- [includes/traits/trait-kdc-qtap-finance-admin-tab-receipts.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-admin-tab-receipts.php) — split grade/division, add `receipt_url` + `student_url`, swap Flags/Items table header order.
- [assets/js/kdc-qtap-finance-receipts.js](kdc-qtap-finance/assets/js/kdc-qtap-finance-receipts.js) — column defs rewritten with `(v, type, row)` signature + Receipt/Student link renderers + split grade/division + Flags→Items reorder; `exportXlsx()` replaced with `exportBook()` handling both CSV + XLSX via SheetJS.
- [includes/class-kdc-qtap-finance-admin.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-admin.php) — i18n keys updated: `grade_division` replaced by `grade` + `division`.

## [3.16.31] - 2026-04-22

### Fixed
- **Recompute Breakup wasn't refreshing the line-item header title.** v3.16.23 wired `apply_full_tenure_retitle()` into `apply_allocation_breakup()` so that trickle-adding a downstream-term line item would promote the header from "1st Term" to "Full Tenure". But the retitle call sat inside the `if ( $touched > 0 )` guard — so when the recompute ran and the breakup rows were already correct (nothing to rewrite, `$touched` stayed 0), the retitle was skipped too. Orders created before v3.16.12 shipped the retitle, or any order whose title drifted from its coverage, couldn't be corrected via the Recompute Breakup button. **Fix**: the retitle is now called unconditionally when `apply_allocation_breakup()` runs. `apply_full_tenure_retitle()` is already idempotent — it no-ops when the order doesn't cover every regular payment for the user+year, and re-saving the same name is a no-op write — so moving it out of the gate is safe for every caller (event-time create, migration backfill, manual recompute, bulk recompute, Rebuild All).

### Post-upgrade cleanup
- **One-time sweep: `migrate_sweep_full_tenure_retitle_3_16_31()`** runs on first load after upgrade. Loops every order flagged `_kdc_qtap_finance_is_fee_payment=yes` and calls `apply_full_tenure_retitle()` — so historical orders whose header drifted ("1st Term [2026-2027]: Jun 2026 to May 2027" on an order that actually covers the full year) are promoted to "Full Tenure" in bulk. Version-gated on `kdc_qtap_finance_sweep_full_tenure_retitle_3_16_31_done` + try/catch with debug-log audit counts (`orders_scanned`, `line_items_renamed`). Idempotent: retitle's coverage gate leaves non-full-tenure orders alone, re-saving the same name is a no-op write.

### Files changed
- [includes/traits/trait-kdc-qtap-finance-wc-orders.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-orders.php) — retitle moved outside the `$touched > 0` guard in `apply_allocation_breakup()`.
- [kdc-qtap-finance.php](kdc-qtap-finance/kdc-qtap-finance.php) — new `migrate_sweep_full_tenure_retitle_3_16_31()` + version-gated runner.

## [3.16.30] - 2026-04-22

### Added
- **Shortfall info banner on the Record Payment modal.** Parallel to the existing yellow overflow warning (shown when staff enter more than the term balance — `₹X will settle this term, ₹Y carried forward`), a blue info banner now fires when the entered amount is LESS than the balance. Message names the exact shortfall, states it remains outstanding until collected, and explicitly reassures staff that the partial payment is valid to record (the term stays unpaid in the UI until the remainder is settled). Same DOM location as the overflow notice; three mutually-exclusive states managed by the live `input` handler:
  - `entered > balance` → yellow overflow banner.
  - `entered < balance && entered > 0` → new blue shortfall banner.
  - `entered == balance` or empty → both hidden.

### Files changed
- [includes/traits/trait-kdc-qtap-finance-user-meta-rendering.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-rendering.php) — new `#kdc-qtap-finance-payment-shortfall-notice` div alongside the existing overflow notice, styled as a WP-admin info card (blue `#2271b1` left-border, `#e7f3fb` fill).
- [assets/js/kdc-qtap-finance-user-profile-payments.js](kdc-qtap-finance/assets/js/kdc-qtap-finance-user-profile-payments.js) — modal-open reset clears both notices; `input` handler extended with the shortfall branch and keeps overflow/shortfall mutually exclusive.

## [3.16.29] - 2026-04-22

### Added
- **New admin tab: Receipts** at Finance → Receipts. Admin-side equivalent of the staff-frontend receipts view with the extra columns finance/audit teams need.
  - **Columns**: Receipt Number (WCPDF), Receipt Date (canonical `payment_date` meta → `date_paid` → `date_created`), Order Number (clickable → WC order edit), Order Date, Paid With (`paywith_method` meta → WC payment method title), Amount (order total), UTR / Transaction ID (`pay_utr` → `transaction_id` meta), Payee Name (`_kdc_qtap_finance_payee_name` → billing name), Student (`_kdc_qtap_finance_student_name` → customer full name), Year (`_kdc_qtap_finance_academic_year`), Grade / Division (canonical metas), short Items summary from line-item names, and Flags column with `Fee` / `POS` badges.
  - **Filters**: server-side date range (default last 90 days to bound initial payload on large sites) + a checkbox to include POS / non-fee orders alongside fee-payment receipts. Date range triggers a fresh AJAX fetch; the checkbox re-fetches immediately.
  - **Search**: client-side DataTables multi-column search — instant filtering across payee, student, UTR, grade/division, etc. without hitting the server.
  - **Sort + pagination**: DataTables handles both client-side. Default sort is Receipt Date DESC; page sizes 25 / 50 / 100 / 250 / All.
  - **Export**: three DataTables Buttons entries — CSV of the currently filtered view, CSV of all loaded rows (ignoring search filter), and XLSX via the bundled SheetJS library. File names auto-encode the date range (e.g. `receipts_2026-01-01_to_2026-04-22.csv`).
- **POS order detection** reuses the existing `KDC_qTap_Finance_Block_Editor::detect_pos_order()` helper — checks `_woocommerce_pos_uuid`, `created_via` containing 'pos', and a regex sweep of meta keys for `pos` / `cashier`.
- **New AJAX action**: `kdc_qtap_finance_receipts_data` (capability: `manage_options` or `manage_woocommerce`, nonce: `kdc_qtap_finance_nonce`). Returns `{ rows: [...], count: N }`. When the "include non-fee" toggle is off, the server narrows the `wc_get_orders()` query with `meta_key=_kdc_qtap_finance_is_fee_payment` for cheaper fetches on fee-heavy sites.

### Files changed
- New [includes/traits/trait-kdc-qtap-finance-admin-tab-receipts.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-admin-tab-receipts.php) — tab render + `ajax_receipts_data()` handler + `build_receipt_row()` helper.
- New [assets/js/kdc-qtap-finance-receipts.js](kdc-qtap-finance/assets/js/kdc-qtap-finance-receipts.js) — DataTables init, date-range reload, CSV/XLSX export buttons, XSS-safe column renderers.
- New [assets/css/kdc-qtap-finance-receipts.css](kdc-qtap-finance/assets/css/kdc-qtap-finance-receipts.css) — tab toolbar + flag badge styles.
- [includes/class-kdc-qtap-finance-admin.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-admin.php) — require new trait + add `use`, register `'receipts'` tab (placed between Report and Enrollments), switch case, AJAX hook, and extend the existing DataTables-asset conditional to include the new tab with receipts-specific CSS / JS / `kdcQtapFinanceReceipts` localize.
- [readme.txt](kdc-qtap-finance/readme.txt) + [CHANGELOG.md](kdc-qtap-finance/CHANGELOG.md) — document.

## [3.16.28] - 2026-04-21

### Fixed
- **User_meta bloat — multi-year students tripped WooCommerce's 50-entry threshold warning ("Customer #253991744 has 52 meta_data entries (threshold: 50). This may indicate plugin meta bloat.").** Root cause: every enrollment save wrote two denormalised shadow rows (`kdc_qtap_finance_grade_{year}` + `kdc_qtap_finance_division_{year}`) on top of the canonical `kdc_qtap_finance_enrollments` JSON blob, purely so the admin users-list `meta_query` filter could do exact-match lookups. A 5-year student carried 10 extra rows for zero functional reason. Consolidated into a single `kdc_qtap_finance_enrollment_index` row per user with format `|YEAR:GRADE:DIVISION|YEAR:GRADE:DIVISION|` (leading + trailing pipes so `LIKE '%|YEAR:GRADE:DIVISION|%'` matches exactly, and year-only / year+grade filters collapse to shorter LIKE patterns against the same key). One row per user regardless of enrolled years. Row count now drops by `2 × (enrolled years - 1) - 1` per multi-year student on first load after upgrade.

### Changed
- **Admin users-list filter rewritten** at [trait-kdc-qtap-finance-user-meta-associations.php::handle_users_list_sorting()](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-associations.php) — three filter shapes (year-only / year+grade / year+grade+division) now collapse into a single `meta_query` LIKE clause against the `kdc_qtap_finance_enrollment_index` key. Filter precision, pagination, and UX identical. `WP_User_Query` still does all filtering at SQL level — no in-memory post-filter.
- **Enrollment save / delete** at [class-kdc-qtap-finance-enrollment.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-enrollment.php) — replaced per-year `update_user_meta(grade_{year})` + `update_user_meta(division_{year})` with a single `KDC_qTap_Finance_Enrollment::write_enrollment_index()` call that rebuilds the full index. Delete path no longer needs per-year cleanup — the index rebuild naturally drops the deleted year's segment. Both paths also purge any lingering legacy shadow rows for the user (belt-and-suspenders — the migration already swept them, but manual DB edits could theoretically re-introduce them).
- **Credit adjustments audit log** at [trait-kdc-qtap-finance-admin-tab-credits.php::ajax_zero_user_credit()](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-admin-tab-credits.php) — capped at the last 50 entries. A user zeroed 100 times previously accumulated 100 adjustment records in a single (unbounded) row; now oldest entries are pruned on append. Preserves audit breadth without unbounded byte growth.
- **Legacy `is_rte` field stripped from enrollments JSON on save.** Code reads both `exempt` and `is_rte` (with `exempt` preferred) — so dropping `is_rte` on every save trims dead bytes without altering read behaviour. Existing untouched records continue to work via the reader's fallback until next save.

### Added
- `KDC_qTap_Finance_Enrollment::INDEX_META_KEY` class constant (`'kdc_qtap_finance_enrollment_index'`) — the single consolidated meta key name.
- `KDC_qTap_Finance_Enrollment::build_enrollment_index( array $enrollments )` — pure function that builds the index string from a decoded enrollments map. Used by both the live write path and the migration.
- `KDC_qTap_Finance_Enrollment::write_enrollment_index( int $user_id, array $enrollments )` — persists the index row (or deletes it if empty) and purges lingering legacy shadow rows for that user.

### Post-upgrade cleanup
- **One-time migration `migrate_consolidate_enrollment_index_3_16_28()`** runs on first load after upgrade. Iterates every user with a `kdc_qtap_finance_enrollments` meta row in batches of 500, writes the consolidated index row for each, then a single bulk `DELETE FROM {$wpdb->usermeta} WHERE meta_key LIKE 'kdc_qtap_finance_grade_%' OR meta_key LIKE 'kdc_qtap_finance_division_%'` wipes every legacy shadow row in one query. Followed by a `wp_cache_flush_group('user_meta')` (fallback to `wp_cache_flush()` on WP < 6.1) so cached reads pick up the new state. Version-gated on `kdc_qtap_finance_consolidate_enrollment_index_3_16_28_done` + try/catch-wrapped with debug-log audit counts (`users_scanned`, `index_rows_written`, `legacy_rows_deleted`). Idempotent — re-running rebuilds the same index from the same canonical JSON.

### Risk / compatibility
- **Single-reader audit**: the only code path that reads the flat `kdc_qtap_finance_grade_{year}` / `kdc_qtap_finance_division_{year}` keys is the admin users-list `meta_query` filter — rewritten in this release to use the new index. No REST, no AJAX, no sibling-plugin readers (parallel Explore agent confirmed across `kdc-qtap`, `kdc-qtap-mobile`, and all `kdc-qtap-*` plugins).
- **No sibling-plugin bumps needed** — mobile / parent plugins neither read nor write the flat keys; their own user_meta writes are already single-row atomic.
- **No data loss**: canonical `kdc_qtap_finance_enrollments` JSON is untouched. The deleted flat keys were shadow copies only; every value in them was derivable from the canonical JSON.
- **Uninstall**: `uninstall.php`'s existing wildcard `meta_key LIKE 'kdc_qtap_finance_%'` already covers the new index key — no uninstall changes needed.

### Files changed
- [includes/class-kdc-qtap-finance-enrollment.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-enrollment.php) — new `INDEX_META_KEY` constant + `build_enrollment_index()` / `write_enrollment_index()` helpers; `save()` and `delete()` paths rewritten; `is_rte` stripped on save.
- [includes/traits/trait-kdc-qtap-finance-user-meta-associations.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-associations.php) — `handle_users_list_sorting()` meta_query collapses to a single LIKE clause against the index key.
- [includes/traits/trait-kdc-qtap-finance-admin-tab-credits.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-admin-tab-credits.php) — credit adjustments log truncated to last 50 entries.
- [kdc-qtap-finance.php](kdc-qtap-finance/kdc-qtap-finance.php) — new `migrate_consolidate_enrollment_index_3_16_28()` + version-gated runner.

## [3.16.27] - 2026-04-21

### Performance
- **Record Payment form submission is noticeably faster.** The staff console and the admin Record Payment modal both submit through `ajax_record_payment()` in [trait-kdc-qtap-finance-user-meta-payments.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-payments.php). Two changes cut user-perceived latency:
  1. **`Payment::record_transaction()` now returns the new transaction id** (int) on success, `false` on failure. Previously returned a bool. The handler's next step — patching `receipt_file` / `reference` / `recorded_by` onto the just-created transaction row — required a `Payment_Transaction::get_by_payment()` scan just to find the id of the row we'd literally just inserted. That scan is gone; the handler uses the returned id directly. All existing callers of `record_transaction()` check the result with boolean truthiness (`if ( $result )`), and a positive int is truthy, so the contract is preserved.
  2. **Presentation-layer work moved to `shutdown`-after-flush.** Two steps weren't needed before the JSON response could be honest with the admin: `apply_allocation_breakup()` (rewrites the line-item breakup meta from the pre-event snapshot diff — only used when the receipt renders) and `link_order_to_payment()` (stamps a post_meta linking the WC order to the payment row). Both are now registered on the `shutdown` action at priority 5. The callback first calls `fastcgi_finish_request()` (when available — PHP-FPM) to flush the response bytes to the client, then runs the deferred work. Admin gets the success response as soon as the transaction + WC order + status are fully persisted; the rest finishes server-side moments later.

### Safety
- Deferred work is wrapped in try/catch; any failure is captured via `kdc_qtap_debug_log` with order_id + payment_id + error trace. The admin already saw the payment recorded successfully (because it WAS recorded — the deferred work is presentation-only), so the failure mode is "receipt breakup needs a manual Recompute" rather than "payment lost".
- No callers of `record_transaction()` rely on the returned value being strictly `true` — all use `if ( $result )` / `if ( $ok )`. The five callsites (admin Record Payment, REST API, CSV import, WC status change, recursive trickle) all continue to work unchanged.

### Files changed
- [includes/class-kdc-qtap-finance-payment.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-payment.php) — `record_transaction()` now returns `(int) $transaction_id` on success.
- [includes/traits/trait-kdc-qtap-finance-user-meta-payments.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-payments.php) — use the returned tx_id directly (skip `get_by_payment()` scan); move `apply_allocation_breakup()` + `link_order_to_payment()` to a shutdown callback that flushes the response first.

## [3.16.26] - 2026-04-21

### Changed
- **Rebuild Receipt Metas action now also backfills Payee Name.** The Finance → Maintenance → "Rebuild Metas" AJAX handler (`kdc_qtap_finance_rebuild_receipt_metas`) runs both migrations back-to-back: `migrate_backfill_payment_date_3_16_23()` (payment_date / paywith_method / pay_utr from the earliest linked transaction) + `migrate_backfill_payee_name_3_16_19()` (seeds `_kdc_qtap_finance_payee_name` from the order's `billing_first_name` when the canonical meta is empty). Both migrations are idempotent — safe to rerun — and the handler resets both done-flags before invoking. Payee Name has no transaction-side source, so `billing_first_name` is the canonical seed (set at order creation from the Record Payment form and overridable via the Student Assignment metabox). Card copy + button label updated to reflect the four-meta scope.

### Files changed
- [includes/class-kdc-qtap-finance-wc-orders-admin.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-wc-orders-admin.php) — `ajax_rebuild_receipt_metas()` chains the payee-name migration after payment-date.
- [includes/traits/trait-kdc-qtap-finance-admin-tab-maintenance.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-admin-tab-maintenance.php) — card copy now lists all four metas (Payment Date / Paid With / UTR / Payee Name).

## [3.16.25] - 2026-04-21

### Added
- **New "Maintenance" admin tab** at Finance → Maintenance. Dedicated home for sitewide data-rebuild actions. Hosts two cards:
  - **Rebuild Breakups** — moved from the Credits tab. Re-runs the waterfall simulation across every fee order and rewrites per-period receipt breakup rows. Wraps the existing `migrate_backfill_allocation_breakup_3_16_13()` migration via the existing `kdc_qtap_finance_rebuild_all_breakups` AJAX action.
  - **Rebuild Receipt Metas** — NEW. Re-syncs `payment_date` / `paywith_method` / `pay_utr` on every fee order from the earliest linked transaction. Wraps the v3.16.24 `migrate_backfill_payment_date_3_16_23()` via a new `kdc_qtap_finance_rebuild_receipt_metas` AJAX action (capability: `manage_options`, nonce: `kdc_qtap_finance_rebuild_receipt_metas`). Resets the `_3_16_24_done` flag and re-invokes. Inner `sync_receipt_metas_from_transaction()` is idempotent (per-field "only update if different") so reruns are safe and still correct stale values — same semantics as the v3.16.24 one-time migration, now on-demand.

### Changed
- **Credits tab renamed to "Adjustment"** in the admin nav. URL slug preserved (`?tab=credits`) so existing bookmarks keep working — only the display label changed.
- **Rebuild All card removed from the Adjustment tab.** The tab is now purely a per-user credit reconciliation audit + Zero-Out workflow; sitewide rebuild tools live on the Maintenance tab. Keeps the Adjustment workflow focused and gives Maintenance room to grow without crowding the audit table.

### Files changed
- New [includes/traits/trait-kdc-qtap-finance-admin-tab-maintenance.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-admin-tab-maintenance.php) — two-card render + shared JS runner for both rebuild actions.
- [includes/class-kdc-qtap-finance-admin.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-admin.php) — require new trait, add `use`, register `'maintenance'` tab + switch case, rename Credits label.
- [includes/class-kdc-qtap-finance-wc-orders-admin.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-wc-orders-admin.php) — new `ajax_rebuild_receipt_metas` handler + hook registration.
- [includes/traits/trait-kdc-qtap-finance-admin-tab-credits.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-admin-tab-credits.php) — remove Rebuild All card markup + JS.

## [3.16.24] - 2026-04-21

### Fixed
- **v3.16.23's payment-date backfill skipped orders with a stale date meta — receipt still showed the wrong date after regeneration.** The migration's gate was `if ( $has_date && $has_method && $has_utr ) skip` — but "has" only meant "non-empty", not "correct". Orders whose `payment_date` had been stamped at order-creation or at status transition (i.e. to the admin's current date, not the transaction's real payment date) passed the check and were skipped entirely. For offline-verified orders in particular, the v3.16.17 PDF hook had synced `date_paid` from whatever `payment_date` meta happened to be at render time, so a stale meta rendered on every re-generation. The receipt showed, say, 20-Apr-2026 (order creation) when the transaction's real payment date was 06-Apr-2026. **Fix**: when an order has a linked transaction, the migration now ALWAYS routes it through `KDC_qTap_Finance_WooCommerce::sync_receipt_metas_from_transaction()` — the helper's per-field "only update if different" logic means re-runs are safe, and it corrects a stale `payment_date` by overwriting it with `$transaction->payment_date`. The fallback path (no linked transaction — stamp `payment_date` from `date_paid` / `date_created`) is unchanged. A new done-flag `kdc_qtap_finance_backfill_payment_date_3_16_24_done` lets the corrected migration re-touch orders the v3.16.23 pass had already flagged as done.

### Files changed
- [kdc-qtap-finance.php](kdc-qtap-finance/kdc-qtap-finance.php) — migration helper: drop the short-circuit, route every order-with-transaction through `sync_receipt_metas_from_transaction()`. New v3.16.24 done-flag + version gate.
- [readme.txt](kdc-qtap-finance/readme.txt) / [CHANGELOG.md](kdc-qtap-finance/CHANGELOG.md) — bump + document.

## [3.16.23] - 2026-04-21

### Fixed
- **Receipt breakup rows were rendered in DB fetch order, not the admin-configured "Fee Item Sort Order".** The plugin already exposes two settings under Finance → Settings that drive the order of rows in Fees-block and payment views: `fee_sort_primary` (radio — fee type first vs slab position first) and `fee_type_order` (drag list — Per Tenure / Per Cycle / Per Term / Per Month). The receipt line-item `breakup_items` meta ignored both and just appended rows in the order `KDC_qTap_Finance_Payment_Item::get_by_payment()` returned them. **Fix**: all three breakup-producing paths — `create_fee_order()` (single-payment order), `create_multi_fee_order()` (multi-term order), and `apply_allocation_breakup()` (event-time + recompute rewrites) — now route their rows through a shared `sort_breakup_rows()` helper that reads the admin setting and sorts by fee-type priority → slab position → period (or slab position → fee-type priority → period when the primary is set to `slab`). Matches the live allocator's waterfall and the Fees-block row order, so receipts, schedule UI, and waterfall all stay in lock-step.
- **"1st Term" header stuck on orders that recompute turned into Full Tenure coverage.** `apply_full_tenure_retitle()` runs at the end of `create_fee_order()` / `create_multi_fee_order()` and flips the line-item title to "Full Tenure" when the order's line items cover every regular payment for the user+year. But when `apply_allocation_breakup()` adds trickle informational line items during a recompute, the order's covered-payment-id set GROWS — so an order that was "1st Term …" on creation can end up covering the full tenure post-recompute. The retitle wasn't re-running in that flow, so the header stayed "1st Term …" while the breakup showed every period. **Fix**: `apply_allocation_breakup()` now calls `apply_full_tenure_retitle()` after its save. The call is idempotent — orders that don't cover the full tenure are left alone.

### Post-upgrade cleanup
- **Backfill of all three receipt metas on historical fee orders — `payment_date` + `paywith_method` + `pay_utr`.** v3.16.22 fixed the meta writes in admin Record Payment / CSV import / gateway paths, but orders CREATED before that release had missing or stale metas — their WCPDF receipts render with today's date, empty Paid With, and no UTR/Ref row. A one-time migration (`migrate_backfill_payment_date_3_16_23()`) now runs on first load after upgrade. For each fee order missing any of the three metas:
  - **Primary path**: finds the earliest transaction linked to the order (via `Payment_Transaction::get_all_by_wc_order_id()`) and routes it through the existing `KDC_qTap_Finance_WooCommerce::sync_receipt_metas_from_transaction()` helper — the same function the live event-time flows use. That one call stamps all three metas from the canonical transaction fields (`payment_date`, `payment_method_title` → `paywith_method`, `reference` / `receipt_file` → `pay_utr`), and also keeps the legacy `transaction_id` order meta in sync with `pay_utr`.
  - **Fallback path**: when no transaction is linked to the order, only `payment_date` can be reconstructed — stamps it from `date_paid` then `date_created`. `paywith_method` and `pay_utr` are left empty (there's no truth source without a transaction).
  Run is version-gated on `kdc_qtap_finance_backfill_payment_date_3_16_23_done` + wrapped in try/catch so a single-order failure doesn't block the rest. Debug-log emits scan / sync / fallback / skip counts.

### Files changed
- [includes/traits/trait-kdc-qtap-finance-wc-orders.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-orders.php) — new `get_fee_sort_config()` + `sort_breakup_rows()` helpers; wired into `create_fee_order()` / `create_multi_fee_order()` / `apply_allocation_breakup()`; `apply_allocation_breakup()` calls `apply_full_tenure_retitle()` on success.
- [kdc-qtap-finance.php](kdc-qtap-finance/kdc-qtap-finance.php) — new `migrate_backfill_payment_date_3_16_23()` + version-gated runner.
- [readme.txt](kdc-qtap-finance/readme.txt) / [CHANGELOG.md](kdc-qtap-finance/CHANGELOG.md) — bump + document.

## [3.16.22] - 2026-04-21

### Fixed
- **Stray breakup rows on the receipt (sub-total didn't match line total).** `apply_allocation_breakup()`'s strip-before-write pass only removed public line-item meta whose KEY matched a new row's slab label. Any slab label the current rewrite no longer produces — e.g. a prior rewrite stamped a `Term Fee` row but this payment only contributed to `Tuition Fee` items — survived as a stale row with an outdated amount. The sub-total of the breakup rows then diverged from the order total (ticket: Fees 2nd Term showed `Term Fee ₹14,990 + Tuition Fee ₹5,721 + Tuition Fee ₹14,279 = ₹34,990` on a ₹20,000 line). **Fix**: the strip is now format-aware. Any public meta whose VALUE contains the breakup-row pattern (` @ `, matching both `Month-Year @ ₹x` and `Period-to-Period @ ₹x`) is deleted before the new rows are written. Slab-label-matching is kept as a secondary guard for the currency-only shape. Neither pattern collides with the other public meta the plugin stamps on fee line items (student / year / grade / fee category — all plain strings).
- **`payment_date` order meta wasn't being written by admin Record Payment or CSV Transactions import.** The WCPDF receipt hook was redesigned in v3.16.17 to be meta-first — reading `payment_date` directly as the canonical source for the Payment Date row and syncing `date_paid` from it. But the two internal flows that already had the payment_date in hand only called `set_date_paid()`, never wrote the meta. Result: the hook fell back to date_paid-only, and downstream consumers of the meta (the `wcpdf_*_file` cache check, the Student Assignment metabox, any custom reports) saw an empty value. **Fix**: both flows now stamp `update_meta_data('payment_date', $payment_date)` alongside `set_date_paid()`. The shared helper `sync_receipt_metas_from_transaction()` (used by gateway checkout completion + offline-verify) also writes the meta from the transaction's `payment_date` so every path populates it consistently.

### Files changed
- [includes/traits/trait-kdc-qtap-finance-wc-orders.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-orders.php) — `apply_allocation_breakup()` format-aware strip.
- [includes/traits/trait-kdc-qtap-finance-wc-helpers.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-helpers.php) — `sync_receipt_metas_from_transaction()` also writes `payment_date` meta.
- [includes/traits/trait-kdc-qtap-finance-user-meta-payments.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-payments.php) — admin Record Payment stamps `payment_date` meta.
- [includes/traits/trait-kdc-qtap-finance-import-csv-processors.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-import-csv-processors.php) — CSV import stamps `payment_date` meta.

### Post-upgrade cleanup
- For orders that still carry stray breakup rows from before this release, hit **Recompute Breakup** on the Student Assignment metabox, or run the bulk action **qTap: Recompute receipt breakup** on the WC Orders list. The first rewrite with v3.16.22 code does a clean sweep.
- For orders that should have `payment_date` meta set but don't, run the **Rebuild All** button on the Credits tab. That re-runs the v3.16.13 backfill which flows through `sync_receipt_metas_from_transaction()` and populates the meta from the linked transaction.

## [3.16.21] - 2026-04-21

### Added
- **New bulk action: "qTap: Regenerate WCPDF receipt (purge cache)"** on the WC Orders list (HPOS + legacy CPT). For each selected order:
  - Calls `wcpdf_get_document( 'receipt', $order, true )` (falls back to `invoice` if the receipt doc type isn't configured) so a WCPDF document number is guaranteed to be assigned.
  - Iterates the WCPDF-Pro file-cache meta keys (`_wcpdf_receipt_file`, `_wcpdf_invoice_file`), removes any on-disk PDF the path points at, and deletes the meta. Next email / next receipt view re-renders from current order data — picks up any breakup recompute, date_paid fix, payee change, etc. that happened since the cached file was generated.
  - Emits an admin notice with regenerated / purged / skipped counts.
  - No-op with a warning notice if WCPDF isn't active.

### Files changed
- [includes/class-kdc-qtap-finance-wc-orders-admin.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-wc-orders-admin.php) — `register_bulk_recompute_action()` exposes the new action; `handle_bulk_recompute_action()` gains the regenerate branch; `bulk_recompute_notice()` renders the new notice variant.

## [3.16.20] - 2026-04-21

### Added
- **On-demand "recompute receipt breakup" action across three surfaces**, all wired through a shared per-order helper so the behaviour is identical everywhere:
  - **Per-order**: new **"Recompute Breakup"** button on the WC Order edit screen's *qTap: Student Assignment* metabox. Confirms, AJAX-calls the recompute, shows success (`N line item(s) rewritten`) or error inline, and adds an audit order note documenting who hit the button and how many items were touched.
  - **Group**: new **"qTap: Recompute receipt breakup"** bulk action on the WC Orders list (HPOS + legacy CPT). Select orders → apply → admin notice reports touched / skipped counts.
  - **Global**: new **"Rebuild All"** button on *Finance > Credits* tab. Re-runs the v3.16.13 waterfall migration across every fee order (after clearing its done-flag so it executes again), writes results back to the audit meta with source `backfill`.
- **New shared helper `KDC_qTap_Finance_WooCommerce::recompute_order_allocation_breakup( $order, $source = 'manual' )`** in [trait-kdc-qtap-finance-wc-orders.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-orders.php). Factored out of the v3.16.13 migration's inline simulation so all three entry points reuse it verbatim. Unlike the migration it does NOT skip orders whose `breakup_source` is already `event` — by the time the admin hits a recompute button they've decided the current breakup is wrong, so we override. The source tag passed in (`manual` / `bulk` / `rebuild-all` / `backfill`) is written to the audit meta.
- **New AJAX handlers** in [class-kdc-qtap-finance-wc-orders-admin.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-wc-orders-admin.php):
  - `kdc_qtap_finance_order_recompute_breakup` — single-order (metabox button). Capability: `edit_shop_orders` / `manage_woocommerce`. Nonce: order-scoped. Adds an order note.
  - `kdc_qtap_finance_rebuild_all_breakups` — global (Credits tab button). Capability: `manage_options`. Nonce: `kdc_qtap_finance_rebuild_all_breakups`. Resets the v3.16.13 done-flag and re-invokes the migration.
- **New bulk-action plumbing** — `register_bulk_recompute_action()` + `handle_bulk_recompute_action()` + `bulk_recompute_notice()` cover both HPOS (`bulk_actions-woocommerce_page_wc-orders`, `handle_bulk_actions-woocommerce_page_wc-orders`) and legacy CPT (`bulk_actions-edit-shop_order`, `handle_bulk_actions-edit-shop_order`).

### Use cases
Admins now have a UI for the three scenarios that used to need DB edits: (a) manual transaction edits / deletions on a specific order, (b) reconciling a group of orders after an import cleanup, (c) plugin-wide sanity rebuild after any upstream data change.

### Files changed
- [includes/traits/trait-kdc-qtap-finance-wc-orders.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-orders.php) — new `recompute_order_allocation_breakup()` helper.
- [includes/class-kdc-qtap-finance-wc-orders-admin.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-wc-orders-admin.php) — metabox button + JS, two new AJAX handlers, bulk action registration + handler + notice, new i18n strings.
- [includes/traits/trait-kdc-qtap-finance-admin-tab-credits.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-admin-tab-credits.php) — new "Rebuild All" summary card + JS handler.

## [3.16.19] - 2026-04-21

### Added
- **Canonical `_kdc_qtap_finance_payee_name` order meta** populated by every order-creation flow, with the value mirrored into `billing_first_name` so the WCPDF receipt header and admin Orders UI render the real payer. Falls back to the student name when no payee was captured.
- **New shared helper `KDC_qTap_Finance_WooCommerce::apply_payee_name_to_order( $order, $payee_name, $student_name_fallback )`** in [trait-kdc-qtap-finance-wc-helpers.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-helpers.php). Stamps the payee meta when a non-empty value is provided, resolves `billing_first_name` as `payee_name → student_name_fallback`, and never saves the order itself so the write batches with the caller's other changes.
- **Wire-up in 5 flows**:
  - Admin Record Payment (`ajax_record_payment`) — routes the modal's `payee_name` field through the helper.
  - CSV Transactions import (`import_transactions_csv`) — the CSV `payee_name` column now stamps the meta too (previously only set `billing_first_name`).
  - Gateway checkout completion (`process_completed_order`) — seeds the payee meta from the checkout's billing first name when the meta isn't already set, so online-paid orders carry a payee for later edit + receipt regeneration.
  - Student Assignment metabox — new "Payee Name" input in the edit panel, shown in the read-only view too when it differs from the student name.
  - Metabox AJAX handler (`ajax_order_assign_student`) — accepts a `payee_name` param and pipes it through the helper.
- **Backfill migration `migrate_backfill_payee_name_3_16_19()`** — stamps `_kdc_qtap_finance_payee_name` on every `_kdc_qtap_finance_is_fee_payment = yes` order that doesn't already have it, seeding from the order's existing `billing_first_name`. Idempotent, gated by `kdc_qtap_finance_backfill_payee_name_3_16_19_done`, wrapped in try/catch per the v3.16.7 lesson.

### Files changed
- [includes/traits/trait-kdc-qtap-finance-wc-helpers.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-helpers.php) — new `apply_payee_name_to_order()` helper.
- [includes/traits/trait-kdc-qtap-finance-user-meta-payments.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-payments.php) — `ajax_record_payment()` replaces its inline billing-first-name set with the helper.
- [includes/traits/trait-kdc-qtap-finance-import-csv-processors.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-import-csv-processors.php) — CSV import routes through the helper.
- [includes/traits/trait-kdc-qtap-finance-wc-status.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-status.php) — `process_completed_order()` seeds the meta from billing_first_name when missing.
- [includes/traits/trait-kdc-qtap-finance-wc-orders.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-orders.php) — doc comments on `create_fee_order()` / `create_multi_fee_order()` clarify the billing-first-name / payee relationship.
- [includes/class-kdc-qtap-finance-wc-orders-admin.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-wc-orders-admin.php) — metabox renders + persists `payee_name`.
- [kdc-qtap-finance.php](kdc-qtap-finance/kdc-qtap-finance.php) — new `migrate_backfill_payee_name_3_16_19()` method + version-gated migration block.

## [3.16.18] - 2026-04-21

### Added
- **"Custom" enrollment option** on the WC Order admin Student Assignment metabox. After picking a student, the Enrollment Year dropdown now always has a final entry "Custom (this order only)". Selecting it reveals three inputs — Custom Year (required), Custom Grade, Custom Division — that persist as the order's canonical `_kdc_qtap_finance_{academic_year, grade, division}` meta for that order only. The values are **not** written back to the user's enrollment records, so a one-off POS sale or a manually-mapped gateway order can carry year/grade/division context on its WCPDF receipt without affecting finance data.
- The audit order note tags custom-path assignments with `[custom / order-only]` so the order history makes the distinction visible.

### Files changed
- [includes/class-kdc-qtap-finance-wc-orders-admin.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-wc-orders-admin.php) — metabox HTML adds a conditional custom-fields panel; inline JS appends a `__custom__` sentinel option, toggles the panel on change, and sends `custom` + `custom_grade` + `custom_division` in the save payload when active; `ajax_order_assign_student()` handles the `custom` branch (requires year, skips enrollment lookup, uses the typed grade/division directly) and tags the audit note.

## [3.16.17] - 2026-04-21

### Fixed
- **WCPDF receipt showed wrong Payment Date / Paid With / UTR on POS and meta-only orders.** The `pdf_receipt_payment_details` hook used to bail early when `_kdc_qtap_finance_is_fee_payment` wasn't set or no linked transaction was found — so any order where staff stamped `payment_date` / `paywith_method` / `pay_utr` metas directly (POS gateways, manual fixes, third-party integrations) rendered with the WC status-transition timestamp for Payment Date, the raw gateway label ("Pay With") as Payment Method, and no UTR row at all. Reworked the hook as **meta-first**:
  - Reads `payment_date`, `paywith_method`, `pay_utr` order metas directly — no fee-order gate, no transaction lookup.
  - Syncs `date_paid` from `payment_date` meta at render time (day-precision compare) so WCPDF's native Payment Date row reads the right date.
  - Skips the UTR row entirely when no `pay_utr` / `transaction_id` meta is set (previously synthesised a `TXN-N` placeholder from a required transaction lookup).

### Changed
- **Collapsed the duplicate `_kdc_qtap_finance_enrollment_*` order meta set into the canonical `_kdc_qtap_finance_*` set.** The plugin used to dual-write both (`_kdc_qtap_finance_student_name` AND `_kdc_qtap_finance_enrollment_student`, etc.) as a backwards-compat hedge. Any time one got updated via a newer flow and the other didn't, the two keys drifted — a user screenshot showed `academic_year=2026-2027` alongside `enrollment_year=2026-2028` on the same order.
- **Stopped the dual-write** in `ensure_enrollment_metas_on_order()` — only the canonical keys get written going forward.
- **Dropped the legacy fallback** in `pdf_invoice_enrollment_details()` — reads canonical keys only.
- **New one-shot migration** `migrate_collapse_enrollment_metas_3_16_17()` — walks every order with any `_kdc_qtap_finance_enrollment_*` meta, per pair: if the canonical key is empty and the legacy key has a value → promote legacy → canonical; if both are set and differ → keep canonical (newer flow wrote it) and log the drift for audit; then delete the legacy meta. Idempotent. Gated by `kdc_qtap_finance_collapse_enrollment_metas_3_16_17_done`. Emits a debug-log summary including a sample of drift events.

### Added
- **"qTap: Student Assignment" metabox** on the WC order edit screen (HPOS + legacy CPT via dual hook registration). Sidebar metabox shows the order's current student / year / grade / division, with an inline edit form:
  - Student search field (reuses the `/kdc/v1/qtap/staff-search` REST endpoint — multi-token AND match, shows name + email + year · grade · division).
  - Enrollment-year dropdown populated after user pick via a new `kdc_qtap_finance_order_user_enrollments` AJAX handler.
  - Save action writes to `_kdc_qtap_finance_{student_name, academic_year, grade, division}` (canonical set only — legacy keys collapsed) via a new `kdc_qtap_finance_order_assign_student` AJAX handler, and adds an audit order note: *"qTap Finance: student assignment changed by {admin}. Was: {old student} ({old year+grade}). Now: {new student} ({new year+grade})."*
  - Capability-gated on `edit_shop_orders` / `manage_woocommerce`.
  - Nonce-verified per order id (`kdc_qtap_finance_order_assign_student_{order_id}`).
  - Metabox help-text clarifies the scope: order-level display meta only — does NOT re-point the linked Payment / Transaction rows (staff delete + re-record for a full re-link).

### Files changed
- [includes/traits/trait-kdc-qtap-finance-wc-helpers.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-helpers.php) — `pdf_receipt_payment_details()` reworked as meta-first; `pdf_invoice_enrollment_details()` drops legacy fallback; `ensure_enrollment_metas_on_order()` drops the dual-write.
- [includes/class-kdc-qtap-finance-wc-orders-admin.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-wc-orders-admin.php) — new metabox (`register_student_metabox`, `render_student_metabox`, `render_student_metabox_js`, `ajax_order_user_enrollments`, `ajax_order_assign_student`).
- [kdc-qtap-finance.php](kdc-qtap-finance/kdc-qtap-finance.php) — new `migrate_collapse_enrollment_metas_3_16_17()` method + version-gated migration block.

## [3.16.16] - 2026-04-21

### Fixed
- **Three WCPDF-critical receipt metas now get populated regardless of how the order originated.** Before this release the gateway-checkout path (`process_completed_order()`) recorded transactions but never stamped `paywith_method`, `pay_utr`, or `date_paid` on the order. So a real online gateway / POS-created order rendered on the WCPDF receipt with an empty "Paid With" row, no "UTR / Ref" row, and "Payment Date" set to the status-transition timestamp instead of the transaction's payment_date.
- Admin Record Payment + CSV import were already explicit about these metas. Now every creation path flows through a single helper, so future surfaces (POS gateways, new payment methods) get correct receipts by default.

### Added
- **New helper: `KDC_qTap_Finance_WooCommerce::sync_receipt_metas_from_transaction( $order, $transaction )`** in [trait-kdc-qtap-finance-wc-helpers.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-helpers.php). Writes `date_paid` from `transaction.payment_date` (only when day-precision differs), `paywith_method` from `transaction.payment_method_title` (only when missing), and `pay_utr` from a first-non-empty waterfall of `transaction.reference` → `transaction.receipt_file` → existing order `transaction_id` meta → `TXN-{id}` fallback (only when missing). Keeps `transaction_id` order meta in sync with `pay_utr`. Returns a per-field map of what changed so callers can audit. Saves the order exactly once.
- **Forward-path wiring**:
  - [trait-kdc-qtap-finance-wc-status.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-status.php) — `process_completed_order()` calls the helper after recording transactions, using the order's latest linked transaction. Covers WC gateway checkouts and POS completions that flow through `woocommerce_order_status_completed`.
  - [class-kdc-qtap-finance-payment-transaction.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-payment-transaction.php) — `verify()` calls the helper on the linked on-hold order after verification. Offline payment verification now produces the same receipt shape as other paths.
- **Broader backfill migration: `migrate_backfill_receipt_metas_3_16_16()`**. Supersedes the narrower v3.16.12 date_paid backfill (which only covered `_kdc_qtap_finance_admin_recorded = yes` orders). Walks every `_kdc_qtap_finance_is_fee_payment = yes` order, resolves its linked transaction (prefers `_kdc_qtap_finance_transaction_id` meta, falls back to reverse lookup by `wc_order_id`), and calls the helper. Gated by `kdc_qtap_finance_backfill_receipt_metas_3_16_16_done`. Wrapped in try/catch. Emits a debug-log summary (`scanned`, `date_set`, `method_set`, `utr_set`, `no_tx_found`).

### Files changed
- [includes/traits/trait-kdc-qtap-finance-wc-helpers.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-helpers.php) — new `sync_receipt_metas_from_transaction()` static helper.
- [includes/traits/trait-kdc-qtap-finance-wc-status.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-status.php) — `process_completed_order()` wires in the helper.
- [includes/class-kdc-qtap-finance-payment-transaction.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-payment-transaction.php) — `verify()` wires in the helper.
- [kdc-qtap-finance.php](kdc-qtap-finance/kdc-qtap-finance.php) — new `migrate_backfill_receipt_metas_3_16_16()` method + version-gated migration block.

## [3.16.15] - 2026-04-21

### Removed
- **Notification Preview section** on the user profile (the "🔔 Notification Preview" header + "Preview Notifications" button + the associated modal). The underlying notification-preview JS module is left in place so the AJAX endpoint remains callable, but the UI trigger no longer ships.

### Changed
- **WooCommerce Orders table on the user profile now matches the Staff Console Receipts tab styling.** Previously a plain 6-column list (Order / Date / Amount / Status / Receipt # / View). Now:
  - **Order** column: `#order_number` + inline **FEE** and/or **POS** badges (Lucide icons + coloured pills), with the order date as a muted sub-line underneath.
  - **Payment** column: payment method (`paywith_method` meta with fallback to `payment_method_title`) with UTR / transaction reference as a monospace sub-line.
  - **Amount** column: unchanged (`wc_price()`).
  - **Receipt #** column: WCPDF receipt number rendered as an external-tab link via the `get_wcpdf_receipt_url()` helper (v3.16.10) with a Lucide `external-link` glyph.
  - **Status** column: compact status icon via `kdc_qtap_finance_render_order_status()` (matches Receipts tab).
  - **Actions** column: View (customer-facing view-order URL) + Edit (wp-admin order editor — gated by `edit_shop_orders` / `manage_woocommerce` capability). Both use Lucide icons.
- Since the render function (`render_user_wc_orders_section()`) is called by both the wp-admin user profile screen AND the frontend Staff Console block, this change updates both surfaces at once.

### Files changed
- [includes/traits/trait-kdc-qtap-finance-user-meta-rendering.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-rendering.php) — dropped the Notification Preview section header + trigger button + modal; rewrote `render_user_wc_orders_section()` table to match the Receipts tab pattern.

## [3.16.14] - 2026-04-21

### Changed
- **Staff Console find-user search is now live.** Typing 3+ characters triggers the search automatically after a 300 ms debounce — no need to click Find or press Enter. The Find button still works as an explicit submit and retains its single-match auto-redirect shortcut; the live path always renders the match list so the admin isn't yanked to a user's profile mid-typing if a single match happens to surface. A short input (< 3 chars) clears any stale result panel. Out-of-order API responses are discarded via a monotonic sequence counter so a slow earlier search can't overwrite a faster later one.

### Files changed
- [blocks/staff-console/kdc-qtap-finance-staff-console-frontend.js](kdc-qtap-finance/blocks/staff-console/kdc-qtap-finance-staff-console-frontend.js) — refactored the Find button handler into a shared `runSearch(query, { autoRedirect })` and added a debounced `input` handler on the search field.

## [3.16.13] - 2026-04-20

### Fixed
- **Receipt line-item breakup now reflects the real per-period contribution of the payment event, not the scheduled amount.** Before this release `create_fee_order()` built the line-item `breakup_items` meta from `Payment_Item.amount` (the schedule) — so a ₹15,000 payment against a ₹10,000/month schedule rendered on the WCPDF receipt as *May-2026 ₹10,000 · Jun-2026 ₹10,000 · Jul-2026 ₹10,000 · …* (all 12 months at their scheduled amount). And when `record_transaction()` trickled excess into the next term's payment, that spill was invisible on the receipt — the WC order had no line item for the downstream term.

### Added
- **`KDC_qTap_Finance_Payment::snapshot_user_year_item_paid( $user_id, $year )`** — returns a keyed `[payment_item_id => amount_paid]` map of every Payment_Item belonging to the user+year.
- **`KDC_qTap_Finance_WooCommerce::apply_allocation_breakup( $order, $snapshot, $source )`** — diffs the current Payment_Item.amount_paid values against a pre-event snapshot, groups the non-zero deltas by their parent Payment row, rewrites each order line item's public `breakup_items` meta from the deltas, and adds a zero-priced informational line item on the order for any trickle target Payment that isn't already on the order. Writes an audit trail: `_kdc_qtap_finance_breakup_original` (first-run snapshot of the pre-rewrite line-item state), `_kdc_qtap_finance_breakup_source` (`event` or `backfill`), `_kdc_qtap_finance_breakup_applied_at`. Idempotent.
- **Forward-path wiring in all 4 payment-event flows** — admin Record Payment, CSV Transactions importer, gateway checkout completion, and offline payment verification all now snapshot before / apply after the payment event so new orders render the correct breakup.
- **Backfill migration: `migrate_backfill_allocation_breakup_3_16_13()`** — reconstructs historical per-order allocation by replaying every transaction for each (user, academic_year) pair chronologically. For each transaction it simulates the waterfall distribution (matching `Payment::allocate_payment_to_items()` priority: per_tenure → per_cycle → per_term → per_month, then period_start, then sort_order), records per-order contribution per item, then calls `apply_allocation_breakup()` with a synthetic pre-snapshot (`live - contribution`). Never writes to Payment_Item.amount_paid — the live values stay intact; only the WC orders' breakup metas get rewritten. Skips orders whose breakup_source is already `event` so a fresh forward-path rewrite isn't clobbered by a slightly-less-precise reconstruction. Gated by `kdc_qtap_finance_backfill_allocation_breakup_3_16_13_done`. Wrapped in try/catch per the v3.16.7 lesson.

### Out of scope
- Refund handling — negative diffs are filtered out in this pass; refund breakup will ship as a follow-up.
- A dedicated admin audit UI — the audit meta is present on every touched order, inspectable via wp_postmeta / wc_orders_meta or the order edit screen. A visual before/after comparison screen is a follow-up.

### Files changed
- [class-kdc-qtap-finance-payment.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-payment.php) — new `snapshot_user_year_item_paid()` static helper.
- [includes/traits/trait-kdc-qtap-finance-wc-orders.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-orders.php) — new `apply_allocation_breakup()` method on `KDC_qTap_Finance_WooCommerce`.
- [includes/traits/trait-kdc-qtap-finance-user-meta-payments.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-payments.php) — `ajax_record_payment()` snapshot + apply.
- [includes/traits/trait-kdc-qtap-finance-import-csv-processors.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-import-csv-processors.php) — `import_transactions_csv()` snapshot + apply.
- [includes/traits/trait-kdc-qtap-finance-wc-status.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-status.php) — `process_completed_order()` snapshot + apply.
- [includes/class-kdc-qtap-finance-payment-transaction.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-payment-transaction.php) — `verify()` snapshot + apply for the offline-verify flow.
- [kdc-qtap-finance.php](kdc-qtap-finance/kdc-qtap-finance.php) — new `migrate_backfill_allocation_breakup_3_16_13()` method + version-gated migration block.

## [3.16.12] - 2026-04-20

### Added
- **One-shot migration: backfill WC `date_paid` on historical fee orders** (`migrate_backfill_receipt_date_paid_3_16_12()`). Walks every WC order flagged `_kdc_qtap_finance_admin_recorded = yes`, finds its linked finance transaction (via the explicit `_kdc_qtap_finance_transaction_id` order meta, falling back to reverse lookup on `wc_order_id`), compares the order's `date_paid` day-precision to the transaction's `payment_date`, and updates + saves when they differ. Pairs with the v3.16.11 forward-path fix so existing receipts generated before the fix (which showed today as the Payment Date) regenerate with the correct date on next render. Idempotent: already-matching rows skip silently. Gated by `kdc_qtap_finance_backfill_receipt_date_paid_3_16_12_done`. Wrapped in try/catch with debug log on failure.
- **"Full Tenure" line item title** for fee orders that cover a user's entire regular academic-year payment set.
  - New public helper **`KDC_qTap_Finance_WooCommerce::apply_full_tenure_retitle( $order )`** — groups the order's line items by `(user_id, academic_year)` via their `_kdc_qtap_finance_payment_id` meta, compares the covered Payment ids against all regular (non-custom, non-user-fee, non-bracket, non-inactive) Payment rows for that user+year, and renames the line items to `"{Fees} - Full Tenure - {grade} [{year}]"` when every regular payment is represented. No-op when partial / mixed. Returns the number of items renamed.
  - Called from both `create_fee_order()` and `create_multi_fee_order()` after `$order->save()` so new full-tenure payments render cleanly.
- **One-shot migration: backfill Full Tenure titles on historical fee orders** (`migrate_backfill_full_tenure_titles_3_16_12()`). Iterates all `_kdc_qtap_finance_is_fee_payment = yes` orders and calls the helper. Idempotent. Gated by `kdc_qtap_finance_backfill_full_tenure_titles_3_16_12_done`. Wrapped in try/catch.

### Files changed
- [kdc-qtap-finance.php](kdc-qtap-finance/kdc-qtap-finance.php) — two new migration blocks + two new `migrate_backfill_*_3_16_12()` methods.
- [includes/traits/trait-kdc-qtap-finance-wc-orders.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-orders.php) — new `apply_full_tenure_retitle()` helper + wire-up in `create_fee_order()` and `create_multi_fee_order()`.

## [3.16.11] - 2026-04-20

### Fixed
- **WCPDF receipt "Payment Date" row showed today instead of the actual payment date.** Both the admin Record Payment modal and the CSV Transactions importer create the WC order and then immediately `set_status( 'completed' )`. WC's status transition auto-stamps `date_paid` to the current timestamp when it's empty — which is the value WCPDF uses for the Payment Date row on the receipt. So a payment the admin recorded as "made on 2026-04-11" rendered on the PDF as "Payment Date: 20-Apr-2026" (today). Fix: both flows now call `$wc_order->set_date_paid( strtotime( $payment_date . ' 00:00:00' ) )` **before** `set_status( 'completed' )`, pre-empting WC's auto-stamp. Historical orders keep their existing `date_paid` (no backfill) — the next PDF regeneration for a future-recorded payment will reflect the correct date.
- **"Paid With" row on the WCPDF receipt was redundant when equal to Payment Method.** Our `pdf_receipt_payment_details()` hook always rendered a "Paid With" row, even when the value matched WC's native Payment Method row (e.g. both "Bank Transfer"). The row now skips rendering when the two values compare case-insensitively equal, and still prints when they genuinely differ (e.g. Payment Method "Online Payments" / Paid With "UPIAPP").

### Files changed
- [includes/traits/trait-kdc-qtap-finance-user-meta-payments.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-payments.php) — `ajax_record_payment()` stamps `date_paid` from the submitted payment_date before completing the order.
- [includes/traits/trait-kdc-qtap-finance-import-csv-processors.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-import-csv-processors.php) — `import_transactions_csv()` stamps `date_paid` from each row's parsed payment_date before completing the order.
- [includes/traits/trait-kdc-qtap-finance-wc-helpers.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-helpers.php) — `pdf_receipt_payment_details()` adds a case-insensitive dedupe check against `$order->get_payment_method_title()` before printing the Paid With row.

## [3.16.10] - 2026-04-20

### Fixed
- **Staff Receipts tab: full-name search returned zero results.** `wc_get_orders()` passes the `s` parameter to WC's order search, which treats `_billing_first_name` and `_billing_last_name` as independent columns — so a query like "John Smith" never matches (neither column alone contains the whole string). The Receipts tab search now splits the query on whitespace, runs `wc_get_orders` per token with `return => 'ids'`, intersects the resulting order-ID sets, and re-queries with `include`. "John Smith", "Smith John", partials like "Joh Smi" all work. Single-token queries fall through to the original fast path.
- **Staff Console user-search had the same root issue.** "John Smith" only matched if the user's `display_name` happened to be "John Smith" verbatim; `first_name` / `last_name` meta LIKE misses were silent. Both the REST endpoint (`/qtap/staff-search`) and the legacy AJAX handler (`kdc_qtap_finance_search_users`) now do multi-token AND intersection across `display_name`, `user_login`, `user_email`, `first_name`, `last_name`, `CONCAT_WS(' ', first, last)`, `CONCAT_WS(' ', last, first)`, `phone`, `mobile`. Every token must match at least one field.

### Changed
- **Search result cards show Year · Grade · Division** under the user name / email. Pulled from the user's current academic year enrollment via `KDC_qTap_Finance_Enrollment::get_all()` and `kdc_qtap_finance_get_current_academic_year()`. Falls back to the most-recent year with data when the user isn't enrolled in the current year. Empty string when the user has no enrollment at all. JS renderer displays a muted meta line when any of the three fields are present.
- **Staff Console menu stays visible when drilled into a user.** Previously hidden while viewing a student's profile, forcing staff to click "← Change user" just to switch to Receipts / Reports. The menu now renders regardless of `$user` state.
- **Transaction records + user-profile WC Orders table: receipt number renders as an external-tab PDF link** when the WC order has a generated WCPDF document. Staff can click the receipt number from the Payment History transaction row (or the "Receipt #" column in the WooCommerce Orders card) to open the PDF in a new tab without leaving the profile.

### Added
- **`KDC_qTap_Finance_WooCommerce::get_wcpdf_receipt_url( $order_or_id )`** — centralised helper for the `WPO_WCPDF()->endpoint->get_document_link()` lookup. Tries `receipt` type first, falls back to `invoice`. Returns empty string when WCPDF isn't installed, the endpoint method is missing, or no document has been generated yet. Never triggers PDF generation. Used by `class-kdc-qtap-finance-wc-orders-admin.php` (admin orders column), `trait-kdc-qtap-finance-user-meta-rendering.php` (transaction rows + WC Orders card), and available for future call sites (e.g. the Staff Console receipts tab).

### Files changed
- [includes/class-kdc-qtap-finance-block-editor.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-block-editor.php) — Receipts-tab token-intersection search (`render_staff_console_receipts`); multi-token AJAX user-search (`ajax_search_users`); enrolled year/grade/division in the result card payload (`format_user_for_response`); staff menu rendered unconditionally in `render_staff_console_block`.
- [includes/class-kdc-qtap-finance-rest-api.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-rest-api.php) — `staff_search_users()` rewritten as a single indexed usermeta query with per-token AND intersection + new private `enrich_user_for_staff_search()` that adds year/grade/division.
- [blocks/staff-console/kdc-qtap-finance-staff-console-frontend.js](kdc-qtap-finance/blocks/staff-console/kdc-qtap-finance-staff-console-frontend.js) — `renderMatchList()` now displays a meta line with Year · Grade · Division beneath the name/email when the server returns those fields.
- [includes/traits/trait-kdc-qtap-finance-wc-helpers.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-wc-helpers.php) — new `get_wcpdf_receipt_url()` helper.
- [includes/traits/trait-kdc-qtap-finance-user-meta-rendering.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-rendering.php) — transaction row + WC Orders card receipt number wrapped in a new-tab PDF link when available.

## [3.16.9] - 2026-04-20

### Fixed
- **Staff Console block couldn't save any form.** The shared user-profile JS (payments, refunds, enrollments, user-fees, notifications, associations) all call `$.post(ajaxurl, …)`. `ajaxurl` is a global WordPress auto-defines **only on admin pages** via `wp-admin/js/common.js` — it's `undefined` on the frontend. The Staff Console block renders the profile UI on the frontend, so every AJAX write was firing against `undefined` and silently failing. Staff saw form submits that looked successful client-side but never persisted.

### Changed
- **`enqueue_profile_assets_for_user()`** (shared between admin user-edit and the frontend Staff Console) now adds `ajaxUrl => admin_url('admin-ajax.php')` to the localized `kdcQtapFinanceUserProfile` object, and calls `wp_add_inline_script( …, 'before' )` to prime `window.ajaxurl` before the common/feature JS runs. The existing `$.post(ajaxurl, …)` call sites need no changes; both admin and frontend contexts resolve to the same endpoint.

### Files changed
- [class-kdc-qtap-finance-user-meta.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-user-meta.php) — `enqueue_profile_assets_for_user()` adds `ajaxUrl` to localized data and `window.ajaxurl` polyfill via `wp_add_inline_script()`.

## [3.16.8] - 2026-04-20

### Fixed
- **RTE enrollments didn't show the RTE badge, RTE metadata, or RTE state in the edit dialog.** `KDC_qTap_Finance_Enrollment::save()` writes the RTE flag under the `exempt` key, but three spots in the enrollment-card render (`trait-kdc-qtap-finance-user-meta-rendering.php`) and two in the enrollment-edit dialog (`trait-kdc-qtap-finance-user-meta-enrollments.php`) were still reading only the legacy `is_rte` key. Result: a student saved as RTE rendered with no RTE badge in the header, `RTE: —` in the metadata row, and the edit button's `data-rte="0"` — so reopening the dialog cleared the checkbox. Visible symptom on the screenshot: RTE-exempt payments showed `Exempt (RTE)` status correctly (that path reads the payment row directly), while the enrollment card reported RTE as empty.
- Fix: every reader now accepts either key via `! empty( $enrollment['exempt'] ) || ! empty( $enrollment['is_rte'] )`, matching the same fallback pattern already used by the rest of the codebase (save path, export, block editor, REST, CSV import, admin enrollments tab). Legacy data on `is_rte` keeps working; new data on `exempt` now renders correctly.

### Files changed
- [trait-kdc-qtap-finance-user-meta-rendering.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-rendering.php) — RTE badge (line 693), edit button `data-rte` (line 713), RTE metadata cell (line 747) all now read both keys via a single `$is_rte_enrollment` local at card render time.
- [trait-kdc-qtap-finance-user-meta-enrollments.php](kdc-qtap-finance/includes/traits/trait-kdc-qtap-finance-user-meta-enrollments.php) — two read sites (preserve-on-save fallback at line 59, edit-dialog initial state at line 183) now fall back to either key.

## [3.16.7] - 2026-04-20

### Fixed (hotfix for 3.16.6)
- **Critical error on upgrade to 3.16.6.** The new `migrate_backfill_grade_custom_fees_3_16_6()` migration instantiated the enrollment helper via `new KDC_qTap_Finance_Enrollment()`, but that class is a singleton with a `private __construct()` — PHP threw `Cannot access private KDC_qTap_Finance_Enrollment::__construct()` during `maybe_upgrade()`. Because `maybe_upgrade()` runs inside `init_components()` early in the admin bootstrap, the fatal blocked every admin page load. And because the migration had no try/catch and `update_option('…_done', …)` is only called after success, the fatal re-fired on every retry — the site was stuck until downgraded.
- Fix: switched to `KDC_qTap_Finance_Enrollment::get_instance()`, wrapped the migration call in a `try { … } catch ( Throwable $e ) { debug_log; }` so a future partial failure can't brick the admin, and bumped the "done" option key to `kdc_qtap_finance_backfill_custom_fees_3_16_7_done` so sites that already upgraded to 3.16.6 pick up the fix and re-run the backfill automatically.

### Files changed
- [kdc-qtap-finance.php](kdc-qtap-finance/kdc-qtap-finance.php) — `migrate_backfill_grade_custom_fees_3_16_6()` uses `get_instance()` instead of `new`; migration invocation wrapped in try/catch; version-gate and done-option renamed to `3_16_7`.

## [3.16.6] - 2026-04-20

### Fixed
- **Grade-level custom fees now materialise as Payment rows and populate the Report tab's Custom Fees columns.** Before this release the Report's Custom Fees Collection / Received / Balance columns showed dashes for every student even when a custom fee was configured at the grade level in the Fee Matrix. Root cause: `KDC_qTap_Finance_Fee_Matrix::get_applicable_slabs()` iterated only top-level matrix entries, but grade-level custom slabs live nested under the special `_custom_slabs` key — so the enrollment auto-assignment path never emitted the `_custom_<index>` slugs, which meant the enrollment's `fee_slabs` never included them, which meant the Payment rows were never created, which meant the Report had nothing to aggregate.
- **Fix**: `get_applicable_slabs()` now descends into `_custom_slabs` and returns the per-index synthetic slug (`_custom_<N>` or the slab's `slug` field if set) whenever the grade row is enabled with a positive amount.

### Added
- **One-shot backfill migration** for existing enrollments saved before the fix. Walks every user with `kdc_qtap_finance_enrollments` user meta, for each academic year compares the enrollment's `fee_slabs` to the current fee matrix's grade-level custom slabs, adds any missing `_custom_*` slugs to `fee_slabs`, and creates Payment rows via the existing `create_custom_slab_payment()` helper (idempotent — short-circuits if a row already exists). Touches only enrollment meta and creates new payments; does **not** delete anything, does **not** modify paid/partial rows. Emits a `kdc_qtap_debug_log` summary at the end. Gated by `kdc_qtap_finance_backfill_custom_fees_3_16_6_done` option so it runs exactly once.

### Files changed
- [class-kdc-qtap-finance-fee-matrix.php](kdc-qtap-finance/includes/class-kdc-qtap-finance-fee-matrix.php) — `get_applicable_slabs()` descends into `_custom_slabs`.
- [kdc-qtap-finance.php](kdc-qtap-finance/kdc-qtap-finance.php) — new `migrate_backfill_grade_custom_fees_3_16_6()` method + version-gated migration block.

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
