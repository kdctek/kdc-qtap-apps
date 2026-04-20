# Changelog

All notable changes to qTap Mobile are documented in this file.

## [2.14.0] - 2026-04-16

### Removed
- **WC Admin Orders enhancements moved to parent plugin kdc-qtap v2.7.0**: "By" column, Transaction ID column, Created Via filter, bulk Copy Transaction IDs. These features are not mobile-specific and now ship with the parent plugin so every WC-enabled site benefits from them
- Stripped ~540 lines (17 methods + hook registrations) from `class-kdc-qtap-mobile-woocommerce-integration.php`

### Migration Notes
- **Requires kdc-qtap v2.7.0+** for the moved features to be available. Upgrade the parent plugin BEFORE upgrading mobile to avoid a transient gap in the Orders UI
- Parent uses new column keys (`kdc_qtap_order_by`, `kdc_qtap_order_txn_id`) and filter param (`qtap_order_source`) — no collision with older mobile versions

## [2.13.11] - 2026-04-16

### Added
- **Transaction ID column** in WC admin Orders table (HPOS + legacy) with click-to-copy. Single click copies txn ID; double-click copies `Order #N | Transaction: TXN`; Enter/Space on focus also copies; toast confirmation
- **Created Via filter dropdown** above the orders list: All / Checkout / Admin / WCPOS / WhatsApp / REST API. Filters orders via `meta_query` matching `_created_via` and related WhatsApp markers (both meta key variants plus WA gateway meta)
- **Bulk action: Copy Transaction IDs** — copies selected orders' txn IDs (one per line, `Order #N | TXN`) to clipboard

## [2.13.10] - 2026-04-16

### Changed
- "By" column WhatsApp detection: REST-API orders with custom `created_via` meta_data entry valued "whatsapp" / "wa-bot" now show the WhatsApp SVG (green), falling back to REST API icon (grey) otherwise
- Iterates `$order->get_meta_data()` directly to reliably catch custom meta entries that collide with built-in field names

## [2.13.9] - 2026-04-16

### Changed
- WC Orders column renamed from "Via" to "By"
- Rewrote detection logic with first-match-wins priority: WCPOS → WhatsApp → Admin → Checkout → REST API
- Distinct icons per source: Checkout (cart, blue), Admin (admin-users, gray), WCPOS (store, purple), WhatsApp (inline SVG logo, green)
- Empty `created_via` now correctly detected as Admin (matches WC's default behavior for wp-admin orders)
- WhatsApp uses an inline WhatsApp SVG logo (dashicons has no WhatsApp icon)

## [2.13.8] - 2026-04-16

### Added
- **"Via" column in WC admin Orders table** (HPOS + legacy CPT) — narrow column (2.2em, ~checkbox width) inserted right after the checkbox; renders a dashicon keyed off `$order->get_created_via()` with the raw value as a tooltip:
  - **POS** (purple cart): canonical detection via `_woocommerce_pos_uuid` meta + `created_via` stripos for `pos` + meta-key regex scan for `(^|_)(wc)?pos(_|$)` / `cashier` (covers wcpos and other POS plugin variants)
  - **WhatsApp** (green chat icon): `created_via` stripos for `whatsapp` / `wa-bot` + `_whatsapp_order` / `_kdc_qtap_whatsapp_order` / `_wa_order_id` meta
  - **Checkout / store-api** (blue cart), **Admin** (users icon), **REST** (rest icon), **Unknown** (dash)
- Width clamped via `admin_head` inline CSS scoped to the orders screen only via `get_current_screen()` — uses `!important` to win over any other plugin's column width rules.

## [2.13.7] - 2026-04-14

### Changed
- qTap FAB "Back to Admin" button now only appears during an active "Login as {user}" session and properly triggers the parent switch-back flow, showing the original admin's name.

## [2.13.6] - 2026-04-03

### Added
- Email row with mail icon on new line in mobile list
- Email verification OTP form redesign — hint text, compact 100px input, card-style container
- OTP required for removing mobile numbers — inline OTP form with confirm/cancel
- OTP required for toggling WhatsApp notification preference
- WooCommerce email template for OTP emails when WC is active (`WC()->mailer()->wrap_message()`)
- Phone icon (Lucide) for numbers with WhatsApp disabled

### Changed
- Email OTP priority: WooCommerce mailer → parent notification → wp_mail fallback
- WhatsApp toggle change on edit triggers "Send OTP" (same as number change)

## [2.13.5] - 2026-04-03

### Added
- WhatsApp notification toggle per mobile number — checkbox in add/edit form, default enabled
- List UI shows WhatsApp icon for enabled numbers, phone icon for disabled
- `whatsapp` field saved in user meta mobile data structure
- Helper functions: `kdc_qtap_get_whatsapp_enabled_numbers()` and `kdc_qtap_is_whatsapp_enabled()` for notification system filtering
- Indian landline auto-detection: numbers starting with +91[2-5] default to WhatsApp unticked

## [2.13.4] - 2026-04-02

### Added
- "Back to Admin" link in FAB menu for admin users (`manage_options` capability) — arrow-left Lucide SVG icon, links to wp-admin

## [2.13.3] - 2026-04-02

### Added
- **Menu icons** — Lucide SVG icons for `downloads` (download), `switch` (users-round), `payment-methods` (credit-card), `lost-password` (key-round) in My Account sidebar
- **Cart FAB on desktop** — cart FAB now shows on desktop when qTap FAB menu is enabled (positioned above FAB at 90px)

### Changed
- **Orders label** — renamed from "Receipt" to "Receipts" (plural) in both menu and page title
- **Orders icon** — restored to Lucide `receipt` SVG
- **Menu reorder** — added `downloads`, `payment-methods`, `switch`, `lost-password` to the ordered menu array

## [2.13.3] - 2026-04-02

### Changed
- FAB menu icons replaced from dashicons to Lucide SVGs matching My Account sidebar icons (Fees, Mobile/WhatsApp, Dashboard, Switch Student, Logout)

## [2.13.2] - 2026-04-02

### Changed
- Moved qTap Menu (FAB) setting from WhatsApp API tab to General tab under new "UI Enhancements" section
- FAB setting now saves via General tab form handler (previously used `register_setting` which didn't work with manual form submission)

## [2.13.1] - 2026-04-02

### Changed
- Login with OTP settings moved to a dedicated **"Login with OTP"** tab (extracted from General tab)
- JS auto-enters OTP mode on all login forms when enabled — hides username/password, shows phone field immediately
- Form Label injected via JS above phone field on `wp_login_form()` forms
- Phone field font size fixed to 16px in login OTP and standalone form CSS
- `render_otp_login_form()` fires `login_form` action for third-party plugins (Google Site Kit)

### Added
- `otpEnabled`, `formLabel`, `fieldLabel` passed to frontend JS via `wp_localize_script`
- qTap FAB menu: draggable floating action button with contextual links (from separate agent)

## [2.13.0] - 2026-04-02

### Added
- Standalone OTP login form: when "Login with OTP" is enabled, the `qtap/mobile-login` block and shortcode render phone + OTP fields directly without username/password
- New admin settings: **Form Label** (text above phone field) and **Field Label** (phone input label)
- `render_otp_login_form()` method for standalone OTP HTML generation
- `setupStandaloneOtpFlow()` JS function for the OTP-only form with full verify/login/multi-account flow

### Changed
- Social login hooks (`kdc_qtap_login_social_buttons`) always fire regardless of OTP mode
- When OTP disabled, `wp_login_form()` renders as before with JS-based OTP injection on mobile detection

## [2.12.4] - 2026-04-01

### Changed
- Login block/shortcode nav links reordered: My Account → Fees → Mobile
- Fees and Mobile links now use WooCommerce My Account endpoint URLs (`/my-account/fees/`, `/my-account/mobile/`) when WooCommerce is active, falling back to standalone page URLs (`/fees/`, `/mobile/`) otherwise

## [2.12.3] - 2026-04-01

### Fixed
- Desktop My Account nav: proper padding (`8px 20px 8px 10px`), min-width 140px, no nav container padding
- Matched theme selector specificity (`.woocommerce .woocommerce-MyAccount-navigation ul li a`) to override theme's `padding: 8px`
- Logout separator moved to `<li>` for full-width border line
- Content margin increased to 20px for better spacing from nav border

## [2.12.2] - 2026-04-01

### Changed
- Desktop My Account nav: compact width with `fit-content`, right border separator, no background box
- Dashboard visible on desktop, hidden on mobile bottom nav
- Removed all stray borders from theme WooCommerce nav styles with aggressive resets

### Fixed
- Nav width override with `!important` to defeat theme's explicit width rules
- Stray borders on nav items from theme's default WooCommerce list styling

## [2.12.1] - 2026-04-01

### Changed
- My Account menu reordered: Fees, Mobile, Receipt, Addresses, Edit, Log out
- Desktop nav column narrowed to 160px with flex layout for more content space

## [2.12.0] - 2026-04-01

### Added
- WooCommerce My Account UI enhancements: mobile bottom navigation bar with Lucide icons, floating cart FAB, auto-hide on scroll
- Desktop sidebar icons for My Account navigation items
- Dashboard menu item removed, "Orders" renamed to "Receipt"
- Password fields hidden on Edit Account page (OTP-based login replaces password)

### Fixed
- `rename_orders_title` filter now checks `is_main_query()` to avoid renaming titles in nav menus and widgets
- Password fieldset hiding scoped to `.woocommerce-EditAccountForm` only, checking legend text instead of all fieldset content

## [2.11.9] - 2026-04-01

### Added
- Session expiry timer on multi-account selection screen — auto-resets login form with "Login took longer than expected" message when verify token expires
- Countdown shown in last 30 seconds before session expiry

### Changed
- Verify token transient now uses OTP validity setting (default 5 minutes, minimum 5 minutes) instead of hardcoded 120 seconds

## [2.11.8] - 2026-03-31

### Fixed
- Fixed actual root cause of horizontal scroll on mobile — intl-tel-input library sets `.iti { display: inline-block }` which ignores parent width constraints; overridden to `display: block` within mobile block context

## [2.11.7] - 2026-03-31

### Fixed
- Fixed root cause of horizontal scroll on mobile — added `box-sizing: border-box` to `.kdc-qtap-card` within mobile block so padding is included in width calculation
- Reduced card padding from 16px to 12px on small screens (≤480px) for more breathing room
- Removed `overflow: hidden` band-aids from block wrapper, form fields, and endpoint CSS that were masking the issue

## [2.11.6] - 2026-03-31

### Fixed
- Fixed horizontal scroll on `/mobile/` endpoint page — added `overflow-x: hidden` to endpoint wrapper and content area, reduced mobile padding

## [2.11.5] - 2026-03-31

### Fixed
- Fixed horizontal scroll on mobile screens when add mobile form opens — constrained `.iti` phone field container and block wrapper to `max-width: 100%` with `overflow: hidden`

## [2.11.4] - 2026-03-31

### Fixed
- Mobile editor block form fields now match finance block sizing — overrides `--kdc-qtap-font-size-*` CSS vars with absolute `px` values at block root (same pattern as kdc-qtap-finance)

## [2.11.3] - 2026-03-31

### Added
- Last login datetime shown on multi-user account selection cards
- Remember last selected account via localStorage — highlighted and sorted first on next login
- `kdc_qtap_last_login` user meta saved on each OTP login for last login tracking

### Changed
- Stronger hover effect on account selection cards — visible lift, shadow, and border color change
- Last-used account card highlighted with primary color border

## [2.11.2] - 2026-03-31

### Fixed
- User selection card text now uses `inherit` for theme-first color cascade (Theme → qTap → WooCommerce → WordPress defaults)
- Muted text uses `opacity` instead of hardcoded colors — works with any theme
- Back link inherits theme link color

## [2.11.1] - 2026-03-31

### Changed
- Multi-user account selection now renders inline within the login form instead of a fixed-position modal overlay
- User selection styles moved from inline JS `<style>` tag to CSS file
- "Logout" button in modal replaced with "Back" link for inline flow

### Added
- WooCommerce My Account link in login block nav row
- Full-page loading overlay with spinner on OTP login and social login button clicks
- `showForm` block/shortcode attribute — hide login form for logged-out users
- `showLinks` block/shortcode attribute — show nav links when form is hidden
- Shortcode boolean attributes accept `true`/`false`/`yes`/`no` (not just `1`/`0`)
- `get_login_block_nav_links()` private helper for reusable nav link generation

### Fixed
- Hidden "Remember Me" checkbox across all login forms via CSS and `login_form_defaults` filter

## [2.11.0] - 2026-03-31

### Changed
- `/mobile/` endpoint page now renders `wp_login_form()` with full OTP support instead of a plain "Log In" link
- Endpoint login notice reuses `render_login_block()` for consistent login experience across all pages

## [2.10.22] - 2026-03-31

### Fixed
- Shortcode boolean attributes now accept `true`/`false`/`yes`/`no` in addition to `1`/`0` — fixes `[kdc_qtap_mobile_login form=false links=true]` not working

## [2.10.21] - 2026-03-31

### Added
- WooCommerce My Account link in login block nav row (when WooCommerce active)
- Full-page loading overlay with spinner on OTP login and social login button clicks
- `showForm` block attribute and `form` shortcode attribute — hide login form for logged-out users while keeping logged-in state
- `showLinks` block attribute and `links` shortcode attribute — show nav links (Mobile, Fees, My Account) for logged-out users when form is hidden
- "Show Nav Links" toggle in block editor (only visible when "Show Login Form" is off)
- Extracted `get_login_block_nav_links()` private helper for reuse across logged-in/logged-out states
- Login overlay removed on AJAX error so users can retry
- Social login button click interceptor for Google Sign-In and common social login plugins

### Changed
- Hidden "Remember Me" checkbox across all login forms via CSS and `login_form_defaults` filter

## [2.10.20] - 2026-03-31

### Added
- Alignment option for `qtap/mobile-login` block (left, center, right, fullwidth) and `align` shortcode attribute
- Logged-in state shows "Logout {name}" primary button with navigation links (Mobile page, Fees page if finance plugin active)
- `login_form_defaults` filter hides "Remember Me" checkbox from `wp_login_form()` output

### Changed
- CSS hides "Remember Me" across all login forms (wp-login.php, WooCommerce, frontend) — auth cookie always set with `remember=true`

## [2.10.19] - 2026-03-31

### Added
- New `qtap/mobile-login` Gutenberg block — renders `wp_login_form()` with full OTP support
- New `[kdc_qtap_mobile_login]` shortcode — same login form for any page builder (BeaverBuilder, Elementor, etc.)
- Block attributes: heading, description, social login toggle, redirect URL
- BeaverBuilder login module field selector (`input[name="fl-login-form-name"]`) added to generic detection
- Generic username autocomplete selector (`input[type="text"][autocomplete="username"]`)

### Changed
- Login form block shows "You are logged in as X" with logout link when user is already authenticated

## [2.10.18] - 2026-03-31

### Added
- Generic login form detection — OTP login now works with any login form (BeaverBuilder, Elementor, Ultimate Member, custom shortcodes, modals)
- MutationObserver watches for dynamically inserted login forms (AJAX, modals, page builder renders)
- Hidden `<template>` in wp_footer — JS clones OTP fields into detected forms that weren't covered by PHP hooks
- BeaverBuilder login module hook (`fl_builder_after_render_module`) for direct PHP injection
- Page builder CSS overrides (`.fl-login-form`, `.elementor-form`, `.um-form`) for phone field width

### Changed
- Refactored login OTP JS from single-form global caching to per-form scoped `setupOtpFlow()` — supports multiple login forms on same page
- Removed hardcoded font-size/padding from WooCommerce CSS — parent framework's `.kdc-qtap-input` handles styling
- Replaced inline styles in mobile editor block JS with CSS classes (`.kdc-qtap-btn--compact`, `.kdc-qtap-email-otp-input`)
- wp-login.php phone field now matches native input size (24px font, 3px padding)

## [2.10.17] - 2026-03-25

### Added
- Single-row numbered-column CSV import format — use `mobile [1]`, `name [1]`, `email [1]`, `mobile [2]`, etc. as an alternative to the multi-row format
- Up to 5 contacts per row with automatic expansion into the existing multi-row processing pipeline
- Numbered fields appear in the parent's column mapping UI for manual mapping
- New sample CSV template demonstrating the single-row format
- Documentation for the alternative format in sample CSV README

## [2.10.16] - 2026-03-24

### Changed
- Renamed `get_user_mobile()` to `get_user_mobile_strings()` for clarity (old method kept as deprecated wrapper)
- Aligned `get_user_contacts()` to use `'number'` key instead of `'mobile'`, consistent with all other methods

## [2.10.15] - 2026-03-24

### Added
- Contact email in CSV export — new `contact_email` column in user mobile data exports
- Contact email in CSV import — new `contact_email` field with aliases `mobile_email`, `number_email`
- Email and email_verified fields in JSON export/import for full round-trip support
- JSON import now maps `label→name` and carries email/email_verified when merging contacts

### Changed
- CSV import: new entries include email with `email_verified=false`; updates preserve verified status when email unchanged
- Sample CSV template and readme updated with `contact_email` column and examples

## [2.10.14] - 2026-03-18

### Added
- Optional email field per contact entry — data model expanded to `{number, name, email, email_verified}`
- Email input on admin user profile card and frontend mobile editor block
- Email verification via OTP — sends verification code through parent notification system, logged in notification log
- New AJAX actions: `kdc_qtap_send_email_otp`, `kdc_qtap_verify_email_otp`
- Verified/Unverified badge on contact email in frontend block with inline verify flow
- REST API identity search now also finds users by contact emails in `kdc_qtap_mobile_numbers` meta
- New helper function `kdc_qtap_get_user_contact_emails()` for theme/plugin developers
- `email_verified` status preserved on admin save when email unchanged

### Changed
- `normalize_mobile_data()` includes `email` and `email_verified` fields (backward compatible)
- `get_user_contacts()` returns email and email_verified in contact objects
- `save_user_mobile_numbers()` handles email and email_verified fields
- Frontend block form validates email changes alongside name for "Save" button visibility
- REST API `POST /kdc/v1/qtap/mobile/{identity}` accepts optional `email` parameter

## [2.10.13] - 2026-03-18

### Fixed
- User selection modal no longer auto-closes on overlay click — only closes via account selection or Logout button
- Frontend OTP login via `wp_login_form()` now properly persists login session
- Replaced unreliable PHP `$_SESSION` with WordPress transients for OTP verification state
- Added `do_action('wp_login')` after `wp_set_auth_cookie()` so WooCommerce and other plugins process the login event

### Changed
- Modal "Cancel" button renamed to "Logout" — fully resets OTP state and returns to initial login form
- OTP verification token passed from server to client as one-time-use transient (2-minute expiry)

## [2.10.12] - 2026-03-18

### Added
- Login with OTP now works on any page using `wp_login_form()` (e.g., finance fees block, custom login pages)
- Frontend script auto-enqueue when `login_form` action fires outside wp-login.php
- Username label changed to "Mobile, Email or Username" on frontend login forms via `wp_footer`
- Context-aware CSS classes for OTP buttons/inputs: uses parent's theme helpers on frontend pages
- CSS overrides for `intl-tel-input` phone field to match surrounding login form inputs in all contexts (wp-login.php, WooCommerce, frontend)

### Fixed
- OTP fields not rendering in frontend `wp_login_form()` — uses `login_form_middle` filter (not `login_form` action)

### Changed
- Login OTP JS rewritten to use parent's `intl-tel-input` phone field with country selector
- Mobile number detection swaps username field to intl-tel-input phone field with E.164 sync
- OTP fields now use 3-way context detection (WooCommerce, wp-login.php, frontend) for styling
- Phone field dynamically copies username input CSS classes for theme-consistent styling (no hardcoded sizes)

## [2.10.10] - 2026-02-16

### Removed
- Fallback `lib/libphonenumber.min.js` and `lib/kdc-qtap-mobile-phone-validator.js` (now in parent plugin)
- `function_exists()` guards around parent phone helpers in user mobile class
- `wp_script_is()` fallback script registration in block editor class

## [2.10.9] - 2026-02-13

### Added
- REST API endpoint `POST /kdc/v1/qtap/user` to create a new WordPress user and optionally add a mobile number in a single call
- Supports idempotent behavior: if email already exists, adds mobile to existing user instead of failing

## [2.10.5] - 2025-01-19

### Added
- "Update existing user profiles" import option - updates first_name, last_name, display_name from CSV
- "Send WhatsApp OTP verification" import option - queues OTP messages for imported numbers
- WhatsApp OTP rate limiting (5 second delay between sends) to avoid API throttling
- Verification queue with WordPress scheduled events for async OTP processing
- `update_existing_user_profile()` helper method for profile updates during import
- `queue_whatsapp_verification()` and `process_verification_queue()` methods
- `kdc_qtap_mobile_process_verification_queue` scheduled event hook

### Changed
- Profile updates only apply to top_row entries (prevents partial updates from multi-row imports)

### Updated
- Sample CSV README with documentation for new import options

## [2.10.4]

### Changed
- REST API now uses same pattern as AJAX handler (get → modify → save)

### Removed
- Duplicate `add_mobile_number()` and `remove_mobile_number()` methods from User Mobile class
- Uses `get_user_mobile_numbers()` + `save_user_mobile_numbers()` for consistency

## [2.10.3]

### Added
- `add_mobile_number()` method to `KDC_qTap_Mobile_User_Mobile` class
- `remove_mobile_number()` method to `KDC_qTap_Mobile_User_Mobile` class

### Fixed
- REST API 500 error when adding/removing mobile numbers

## [2.10.2]

### Changed
- REST API now uses WordPress Basic Auth (application passwords) instead of custom X-QTAP-API-Key header
- Requires `edit_users` capability for REST API access

### Removed
- Dependency on parent plugin's `kdc_qtap_rest_permission_check()` function

## [2.10.1]

### Added
- REST API POST endpoint to add mobile number to user
- REST API DELETE endpoint to remove mobile number from user
- Identity can be email, user ID, or username
- Bypasses OTP verification for REST API (server-to-server)
- Validation for duplicate numbers and max limit

## [2.10.0]

### Breaking
- Renamed all classes to include `Mobile` prefix to avoid conflicts with parent plugin:
  - `KDC_qTap_REST_API` → `KDC_qTap_Mobile_REST_API`
  - `KDC_qTap_AJAX_Handler` → `KDC_qTap_Mobile_AJAX_Handler`
  - `KDC_qTap_User_Mobile` → `KDC_qTap_Mobile_User_Mobile`
  - `KDC_qTap_OTP_Handler` → `KDC_qTap_Mobile_OTP_Handler`
  - `KDC_qTap_Block_Editor` → `KDC_qTap_Mobile_Block_Editor`
  - `KDC_qTap_WhatsApp_API` → `KDC_qTap_Mobile_WhatsApp_API`
  - `KDC_qTap_WooCommerce_Integration` → `KDC_qTap_Mobile_WooCommerce_Integration`
  - `KDC_qTap_Admin_Settings` → `KDC_qTap_Mobile_Admin_Settings`
  - `KDC_qTap_Login_OTP` → `KDC_qTap_Mobile_Login_OTP`
  - Note: `KDC_qTap_Mobile_Endpoint` was already correctly named

## [2.9.29]

### Refactored
- WooCommerce endpoint now uses `do_blocks()` to render the standard block

### Removed
- ~150 lines of duplicate rendering code from WooCommerce integration
- Duplicate script localization - block handles its own enqueueing

### Simplified
- WooCommerce integration now only handles endpoint registration and menu item

## [2.9.28]

### Fixed
- WooCommerce My Account `/mobile/` endpoint now renders correctly
- WooCommerce HTML structure now matches block editor (same class names)
- Added missing `classes` config (button, buttonSecondary, input) to WooCommerce localized script
- Added missing `resendOtpSeconds` to WooCommerce localized script
- Added missing `i18n` strings to WooCommerce localized script

## [2.9.27]

### Changed
- Mobile number row now uses regular text (no `__meta` class)
- Phone/WhatsApp row inherits default text styling from `__display`

## [2.9.26]

### Fixed
- OTP input now wrapped in `kdc-qtap-inline-form__field` like other inputs
- OTP input now inherits same styling from parent framework as name/mobile inputs

### Added
- `.kdc-qtap-otp-field` wrapper class to control OTP field width in flex row

## [2.9.25]

### Fixed
- OTP input now fully inherits parent input styling (removed width constraint)
- OTP row uses `align-items: baseline` instead of `center` to preserve input height
- Removed all size overrides from OTP input - only typography (text-align, letter-spacing) remains

## [2.9.24]

### Changed
- Mobile number and WhatsApp link row now uses `__meta` class instead of `__name`

### Fixed
- Name uses `__name` (primary), phone/WhatsApp uses `__meta` (secondary)

## [2.9.23]

### Added
- WCAG compliant ARIA attributes on all frontend elements
- `role="list"`, `role="listitem"`, `role="alert"`, `role="status"`, `role="timer"`, `role="group"`, `role="form"`
- `aria-label`, `aria-labelledby`, `aria-describedby`, `aria-live`, `aria-busy`, `aria-invalid`, `aria-required`
- WhatsApp icon before phone number with link to `https://wa.me/{number}` (opens in new tab)
- Screen reader text for form labels and descriptions

### Fixed
- OTP input now uses parent input class without `font-family: monospace` override
- Only `text-align: center` and `letter-spacing` applied to OTP input

### Improved
- All buttons have descriptive aria-labels including the number they affect

## [2.9.22]

### Changed
- Moved "Contact Label" to top of Mobile Number Settings
- Moved "Resend OTP Timer" to second position after Contact Label

### Fixed
- Removed `<strong>` tag from phone number link (name stays bold, number is normal weight)

## [2.9.21]

### Fixed
- Contact name now uses same font size as phone number (both use __name class)
- Removed meta class from contact name - both fields now equally important

## [2.9.20]

### Changed
- Resend OTP Timer setting now uses minutes instead of seconds for better UX
- Range is now 1-10 minutes (default: 5 minutes)
- Option renamed from `kdc_qtap_block_resend_otp_seconds` to `kdc_qtap_block_resend_otp_minutes`

## [2.9.19]

### Changed
- Phone number now uses primary font size (same as name) in list items
- Contact name shown as secondary/meta text above phone number

### Fixed
- Phone number no longer uses smaller meta font size

## [2.9.18]

### Added
- "Resend OTP Timer" admin setting under General > Mobile Number Settings
- Setting allows 30-600 seconds, default 300 (5 minutes)
- Helper text recommending 5 minutes (300 seconds)

### Changed
- Timer now uses dedicated `resendOtpSeconds` setting instead of `otpValidityMinutes`
- Note: `otpValidityMinutes` still controls actual OTP expiration on server, `resendOtpSeconds` controls UI countdown

## [2.9.17]

### Fixed
- OTP input now inherits color, font-size, height, padding, border from parent input class
- CSS only overrides typography (letter-spacing, text-align, font-family) not input styling
- Width properties use min-width/max-width hints, not forced width

### Clarified
- Parent framework handles ALL input styling - child only adds typography overrides

## [2.9.16]

### Critical Fix
- Never hardcode theme classes - use helper functions only
- PHP: Fallback uses `kdc-qtap-btn kdc-qtap-btn--primary` not `button`
- PHP: Passes button/input classes to JS via `kdcQtapMobileBlock.classes`
- PHP: Uses `kdc_qtap_get_button_class('secondary')` for secondary buttons
- PHP: Uses `kdc_qtap_get_input_class()` for inputs
- JS: Gets classes from PHP config, no hardcoding
- JS: Uses `UI.setButtonLoading()` for loading states
- JS: Proper fallback UI object with all methods
- CSS: Only 50 lines - truly plugin-specific only
- CSS: No backgrounds, no borders, no colors - all from parent

### Removed
- All hardcoded `button`, `wp-element-button`, `input-text` classes

## [2.9.15]

### Major Refactor
- Complete rewrite following parent UI framework guide
- PHP: Follows kdc-qtap UI framework pattern (load components → auth → classes → output)
- PHP: Uses `kdc_qtap_enqueue_frontend_components()` first
- PHP: Uses `kdc_qtap_render_login_required()` for auth
- PHP: Uses `kdc_qtap_get_button_class()` for buttons
- JS: Uses `KdcQtapUI` with complete fallback
- JS: Uses `UI.showMessage()`, `UI.renderListItem()`, `UI.renderEmptyState()`
- JS: Cleaner event delegation pattern
- CSS: Reduced from 250+ lines to ~70 lines
- CSS: Only contains plugin-specific styles (OTP input, phone input, timer)
- CSS: All list/card/message styling from parent framework

### Removed
- Duplicate button/input/message styling
- WooCommerce detection (parent handles it)

### Fixed
- Proper i18n strings for all UI text

## [2.9.14]

### Fixed
- Removed broken opacity styling that made cards invisible
- Removed all custom background colors - fully inherits from theme

### Refactored
- CSS now only contains plugin-specific styles (OTP inputs, phone fields)
- Uses parent UI framework classes: `.kdc-qtap-list`, `.kdc-qtap-list-item`, `.kdc-qtap-card`
- Inline form uses `.kdc-qtap-card` and `.kdc-qtap-inline-form__field` classes

### Removed
- Custom `.kdc-qtap-number-item` styling - uses parent `.kdc-qtap-list-item`
- Custom message styling - uses parent `.kdc-qtap-message--*` classes

## [2.9.13]

### Fixed
- Contact name not being saved when adding a new number
- Contact name now updates correctly for existing numbers
- Removed forced background colors from edit form - uses theme colors
- Contact name (title) now uses `<strong>` with inherited theme color
- Validation messages no longer have forced background colors

### Updated
- All borders and colors now use CSS custom properties with theme fallbacks
- Uses `var(--wp--preset--color--*)` for theme compatibility

## [2.9.12]

### Added
- Integrated parent plugin's KdcQtapUI framework for consistent styling
- Uses `kdc_qtap_enqueue_frontend_components()` to load shared CSS/JS
- Uses `kdc_qtap_get_button_class()` for consistent button styling
- Uses `kdc_qtap_render_login_required()` for login prompts
- Uses `KdcQtapUI.showMessage()` for notifications
- Updated list items to use `.kdc-qtap-list` and `.kdc-qtap-list-item` classes
- Updated empty states to use `KdcQtapUI.renderEmptyState()`
- Graceful fallback when parent plugin functions not available
- Added i18n strings for empty state messages

## [2.9.11]

### Performance Fix
- Removed expensive `gettext` filter that ran on every translation
- Removed `send_headers` hook that ran on every page request

### Changed
- Username label modification now uses JavaScript instead of PHP gettext filter

### Added
- `add_username_label_script()` - lightweight JS for wp-login.php label
- `add_woocommerce_label_script()` - lightweight JS for WooCommerce label

### Deprecated
- `modify_username_label()` - no longer hooked to gettext

### Simplified
- `fix_coop_headers_for_oauth()` - only runs on login_init
- This should significantly improve page load times across all pages

## [2.9.10]

### Fixed
- Cross-Origin-Opener-Policy header blocking Google OAuth postMessage

### Added
- `fix_coop_headers_for_oauth()` method to set proper COOP headers
- Sets `Cross-Origin-Opener-Policy: same-origin-allow-popups` on login pages
- This allows Google Sign-In popup to communicate with parent window
- Hooks into `send_headers` and `login_init` at priority 1 (early)
- Detects login pages: wp-login.php, WooCommerce my-account, pages with login forms

## [2.9.9]

### Fixed
- Google OAuth "Cross-Origin-Opener-Policy" postMessage error

### Changed
- No longer redirects wp-login.php to custom login page
- Custom OTP login page only available at the slug URL (e.g., /signin/)
- wp-login.php now works normally with OTP fields added via hooks

### Deprecated
- `redirect_wp_login()` method - no longer used
- `filter_login_url()` method - no longer modifies login URLs

### Removed
- `skip_redirect` parameter no longer needed

### Note
- `/signin/` → Shows custom OTP-only login page
- `/wp-login.php` → Shows standard WordPress login with OTP option + Google login
- WooCommerce my-account → Shows standard login with OTP option + Google login

## [2.9.8]

### Fixed
- Google/Social login buttons now appear on custom login page
- Moved `login_enqueue_scripts` action to fire BEFORE `login_head` (matching wp-login.php behavior)

### Added
- Set `$pagenow = 'wp-login.php'` global so social login plugins detect the login page
- `login_footer_social` action hook for social login plugins
- `woocommerce_login_form_end` action hook for WooCommerce social login compatibility
- `#kdc-qtap-social-login-wrap` container with proper styling
- CSS for social login buttons and "or" separator

### Improved
- Social login plugins (Site Kit by Google, WooCommerce Social Login, etc.) should now work

## [2.9.7]

### Fixed
- When user cancels multi-user selection modal, Resend OTP button is now immediately enabled (skips timer)

### Added
- Shows "OTP expired or not found. Please request a new OTP." message when modal is cancelled
- `otp_expired` i18n string for localization

### Improved
- Clicking outside modal (overlay) also triggers resend state
- OTP input is cleared and focused after cancel for easy retry

## [2.9.6]

### Fixed
- Social login (Site Kit by Google) now properly completes login
- Changed `login_init` hook priority from 10 to 99 to let social login plugins process OAuth first
- Logged-in users visiting custom login page now use proper redirect logic

### Added
- `filter_login_redirect` method hooked to WordPress core `login_redirect` filter (priority 99)
- This ensures proper redirect handling for ALL login methods including:
  - OTP login via custom page
  - OTP login via wp-login.php
  - Social login (Google, Facebook, etc.)
  - Standard username/password login

### Redirect Flow
1. `redirect_to` parameter (verified for admin access)
2. Plugin settings (Redirect After Login)
3. Role-based: REST API access → wp-admin
4. WooCommerce active → my-account
5. Homepage as fallback

## [2.9.5]

### Fixed
- Social login (Site Kit by Google) now works - OAuth callbacks are no longer intercepted
- WooCommerce my-account redirect for non-admin users
- Redirect flow now properly follows priority:
  1. `redirect_to` URL parameter (verified for admin access)
  2. Plugin settings (Redirect After Login)
  3. Role-based redirect:
     - REST API access users → wp-admin
     - WooCommerce active → my-account page
     - Others → homepage

### Added
- OAuth callback parameter detection (code, state, oauth_token, etc.)
- Social login action whitelist (google_login, sitekit_proxy, etc.)
- `redirect_to` now properly passed from frontend to AJAX handler
- Helper methods `is_admin_url()` and `user_has_admin_access()`

## [2.9.4]

### Fixed
- Custom login page now matches wp-login.php OTP customizations

### Added
- Auto-verify OTP when 6 digits are entered (no need to click button)
- Multi-user selection modal when multiple users share same mobile number
- New i18n strings for select_account, multiple_accounts, cancel

### Changed
- Button text "Login" → "Login with OTP" for consistency
- "Login with Password" → "Login with Email" link text
- `skip_redirect=1` parameter to "Login with Email" link to prevent redirect loop

### Improved
- OTP input now strips non-digits automatically

## [2.9.3]

### Fixed
- Option key mismatch between admin settings and login OTP class
- Admin settings was saving to `kdc_qtap_login_otp_*` but login class was reading `kdc_qtap_mobile_login_otp_*`
- All option keys now consistently use `kdc_qtap_mobile_` prefix:
  - `kdc_qtap_mobile_login_otp_enabled`
  - `kdc_qtap_mobile_login_otp_default`
  - `kdc_qtap_mobile_login_otp_slug`
  - `kdc_qtap_mobile_login_redirect_type`
  - `kdc_qtap_mobile_login_redirect_page`
  - `kdc_qtap_mobile_login_redirect_custom`
  - `kdc_qtap_mobile_login_conflict_dismissed`

### Note
- After updating, you may need to re-save settings once

## [2.9.2]

### Reverted
- Back to clean URL slug approach (e.g., `/signin/`) instead of query parameters

### Fixed
- Simplified detection using `template_redirect` hook with priority -999 (runs before everything)
- Added `is_login_slug_request()` helper that directly parses REQUEST_URI
- No longer relies on WordPress rewrite rules - direct URL path matching
- Works on both root and subdirectory WordPress installations

### Changed
- Detection flow: parse_request (early) → template_redirect (fallback)

### Note
- This approach bypasses WordPress routing entirely for reliability

## [2.9.1]

### Fixed
- Admin settings now show query parameter format instead of slug format
- Option name mismatch between admin settings and login OTP class

### Changed
- Label "Login Page Slug" → "Login Parameter"
- Label "Create a dedicated OTP login page" → "Enable OTP login overlay"
- Preview URL shows `?param=1` format instead of `/slug/` format

## [2.9.0]

### Major Rewrite
- Completely redesigned login flow using overlay approach

### Removed
- Custom rewrite rules that caused 404 errors
- Complex endpoint detection logic

### Added
- Full-screen login overlay with modern design
- Query parameter-based activation (e.g., `?login=1`)
- Close button (X) at top-right corner
- Cancel button at bottom of form
- Click-outside-to-close functionality
- Escape key to close overlay
- Dark mode support
- Responsive design for mobile devices
- Smooth slide-in animation

### Changed
- wp-login.php now redirects to home page with overlay parameter
- `login_url` filter returns overlay URL instead of custom endpoint

### Note
- Settings label changed to "Login Page Parameter" for clarity

## [2.8.x and earlier]

For older changelog entries, see the [GitHub repository](https://github.com/kdctek/kdc-qtap-mobile).
