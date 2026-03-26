# Changelog

All notable changes to qTap App are documented in this file.

## [2.6.7] - 2026-03-26
### Fixed
- **Export toggle-all not working** — the "— toggle all" link on export groups did nothing on the Export tab; JS handler was missing from `render_export_content()`, also now triggers `change` event so filter panels show/hide correctly

### Changed
- **`.gitignore` updates** — added macOS `Icon?` and `_*/` to all plugin `.gitignore` files; created `.gitignore` for kdc-qtap-finance and kdc-qtap-admin

## [2.6.6] - 2026-03-25
### Changed
- **Update highlight border** — plugin cards with available updates now show a border in the WP admin theme color for visual emphasis

## [2.6.5] - 2026-03-25
### Added
- **Changelog page links** — version badges on dashboard plugin cards now link to changelog pages at changelog.qtap.app
- **Docs support on GitHub Pages** — plugin cards show doc buttons (Notifications, Template Variables) linking to rendered documentation
- **Dark/light mode on changelog site** — system-aware theme with manual toggle, qTap SVG logo
- **Cross-sell for uninstalled plugins** — dashboard shows available qTap apps from apps-registry.json

### Changed
- **Version registry URL** — moved from kdctek.github.io to changelog.qtap.app custom domain
- **Settings buttons** — use WordPress admin theme classes instead of hardcoded styles

### Fixed
- **Double-prefix slug** — `register_plugin()` no longer prepends `kdc-qtap-` to IDs that already include it
- **Cross-sell comparison** — uninstalled plugin detection now matches on slug field instead of short ID

## [2.5.8] - 2026-03-24
### Fixed
- **Export data missing — sections fallback** — when stored POST data is empty/corrupted (JSON encoding failure), export now reconstructs checkbox selections from the separately stored `sections` array which always survives encoding; this is the definitive fix for CSV exports producing only summary data

## [2.5.7] - 2026-03-24
### Fixed
- **Export job options lost during insert** — `KDC_qTap_Job::insert()` used bare `wp_json_encode()` which silently returns `false` on non-UTF-8 data; added the same `sanitize_for_json()` fallback that `update()` already had; this caused all export checkboxes (Fee Matrix, Enrollments, Payments, User Data) to be lost, resulting in only the summary CSV being produced
- **Export debug logging** — added trace logging to `process_export_job()` showing stored POST keys and collected data sections; enable debug mode to diagnose export issues

## [2.5.6] - 2026-03-24
### Fixed
- **CSV export single data type produces single CSV** — when exporting one data type (e.g., just enrollments), the export now outputs a single CSV with the data instead of a ZIP with summary+data sheets; only multi-type exports create ZIPs
- **POST data preservation for export filters** — `array_map('sanitize_text_field')` corrupted nested array values (e.g., grade filter checkboxes) when storing job options; replaced with recursive sanitizer that preserves array structure

## [2.5.4] - 2026-03-18
### Added
- **Admin bar menu** — qTap App menu in the WordPress top admin bar with Dashboard, registered child plugins (Mobile, Finance, Events), Notifications, and Import/Export submenus for quick access from any page
- **Force admin bar on switch** — admin bar stays visible on frontend when impersonating a user, ensuring "Switch back" is always accessible

## [2.5.3] - 2026-03-18
### Added
- **Login as User** — button on user profile pages for admins with REST API access; switches session to the target user with a red admin bar "Switch back" node to restore the original session; 1-hour transient TTL with audit logging

## [2.5.2] - 2026-03-17
### Fixed
- **Import job results lost (counters show 0)** — `wp_json_encode()` silently returns `false` when row tracking messages contain non-UTF-8 characters from CSV data; added UTF-8 sanitization with fallback to strip row tracking arrays while preserving summary counts
- **Import report "UNKNOWN" status** — rows showed UNKNOWN because lost results meant no row tracking data; resolved by the JSON encoding fix above
- **Import notice relocation** — `display_import_notice()` was missing the `inline` class, allowing WordPress core JS to move it to the admin notice area

## [2.5.1] - 2026-03-17
### Fixed
- **Stray admin notice bar on Import/Export page** — empty `<div class="notice">` containers (used for JS-populated import results) were being relocated by WordPress core JS (`common.js`) to after the page heading; added `inline` class to prevent relocation
- **Admin notice positioning** — added `wp-header-end` marker to Import/Export page so legitimate admin notices render between heading and tabs instead of at the top of the page body

## [2.5.0] - 2025-01-19
### Added
- Import info section for child plugins to display supported data types
- `kdc_qtap_import_info` action hook for import information
- `kdc_qtap_render_import_info()` helper function for consistent UI
- Two-phase import progress (upload + processing)
- File upload progress bar with bytes, percentage, and time estimation
- JSON batch processing with live counters (Imported/Updated/Skipped/Errors)
- AJAX handlers for upload and batch processing

## [2.4.17] - 2025-01-15
### Changed
- OTP email template with cleaner layout
- Styles moved to H2 element for better rendering

## [2.4.16] - 2025-01-14
### Fixed
- Email OTP HTML rendering

### Added
- Auto-detection of HTML content in email channel
- `is_html` notification flag for explicit control

## [2.4.15] - 2025-01-13
### Improved
- Email OTP displays code in styled H2 tag
- OTP code is clickable when `wa-from` parameter provided

### Added
- Green "Verify via WhatsApp" button in email
- Filter `kdc_qtap_otp_email_html` for customization

## [2.4.14] - 2025-01-12
### Added
- `wa-from` query parameter for OTP endpoint
- WhatsApp link in email when `wa-from` is provided

## [2.4.13] - 2025-01-11
### Fixed
- OTP endpoint uses `recipient` field instead of `to`
- OTP endpoint accepts both `channel` and `channels` parameters
- Improved result checking for nested success/failure

## [2.4.12] - 2025-01-10
### Added
- REST API endpoint `POST /wp-json/kdc/v1/qtap/otp/{identity}` for OTP generation
- REST API endpoint `POST /wp-json/kdc/v1/qtap/otp/{identity}/verify` for verification
- Auto-detection of identity type (email vs phone)
- Custom OTP code support via `code` parameter
- OTP stored as transient with 5-minute expiry
- Filters: `kdc_qtap_otp_permission`, `kdc_qtap_otp_send_response`, `kdc_qtap_otp_verify_response`
- Action hook `kdc_qtap_otp_verified`

## [2.4.11] - 2025-01-09
### Fixed
- PHP 8.1+ deprecation warning for fputcsv() escape parameter

## [2.4.10] - 2025-01-08
### Fixed
- Sample CSV template ZIP download produces valid files
- Output buffer cleaning prevents corrupted downloads
- Associative array rows normalized to match headers

### Added
- Proper cache control headers for file downloads
- ZIP file validation before sending

## [2.4.9] - 2025-01-07
### Improved
- JSON-only export options completely hidden when CSV selected
- Checkboxes unchecked and disabled for CSV format

## [2.4.8] - 2025-01-06
### Removed
- margin-left and margin-bottom inline styles from export items

## [2.4.7] - 2025-01-05
### Removed
- margin-left inline style from export group label

## [2.4.6] - 2025-01-04
### Fixed
- Export sub-options container overflow on mobile
- Added overflow-x handling for filter panels
- Removed duplicate closing brace in CSS

## [2.4.5] - 2025-01-03
### Removed
- All !important declarations from export filter CSS

### Fixed
- Radio buttons and checkboxes display correctly on mobile

## [2.4.4] - 2025-01-02
### Fixed
- Radio buttons and checkboxes no longer stretch on mobile
- Labels display correctly with proper alignment

## [2.4.3] - 2025-01-01
### Fixed
- Export filter UI no longer overflows on mobile
- Date/amount fields stack vertically on narrow screens

## [2.4.2] - 2024-12-31
### Improved
- Date and Amount range rows display side-by-side when space allows
- Compact rows use inline-flex for natural flow

## [2.4.1] - 2024-12-30
### Fixed
- Date and amount range fields display inline in export filters

## [2.4.0] - 2024-12-29
### Improved
- Export/Import UI overhaul with card-style format selectors
- Export filter fields stack vertically for better visibility
- All admin CSS classes use `kdc-qtap-*` prefix

### Fixed
- Toggle all links in export filter panels
- Sample CSV download link visibility
- Replaced WordPress `.card` class with prefixed version

## [2.3.37] - 2024-12-28
### Improved
- Export filter fields stack vertically
- Date/amount range fields still display inline

## [2.3.36] - 2024-12-27
### Fixed
- Toggle-all JavaScript added to Export tab
- Export filter panel CSS added to Export tab

## [2.3.35] - 2024-12-26
### Improved
- Export Format selector uses card-style UI matching Import tab
- Toggle all links use DOM traversal for reliable checkbox finding

## [2.3.34] - 2024-12-25
### Improved
- Export format selector uses card-style UI

### Fixed
- Toggle all link escapes bracket characters in checkbox names

## [2.3.33] - 2024-12-24
### Fixed
- Export filter date/amount fields use inline styles for side-by-side display

## [2.3.32] - 2024-12-23
### Improved
- Export filter CSS uses !important for compact row styles
- Better spacing and alignment for date/amount range fields

## [2.3.31] - 2024-12-22
### Improved
- Export filter date and amount range fields display inline

### Added
- Compact row CSS classes for date/number filter fields

## [2.3.30] - 2024-12-21
### Added
- Import Format selection (JSON/CSV) with visual radio buttons
- CSV import target selection dropdown

### Fixed
- Sample CSV download link uses admin-post.php handler
- Replaced WordPress `.card` class with prefixed `kdc-qtap-card`

## [2.3.28] - 2024-12-19
### Added
- Timestamp suffix to WhatsApp order `reference_id`
- `kdc_qtap_whatsapp_order_sent` action for tracking
- Message ID (wamid) tracking for reply threading

## [2.3.27] - 2024-12-18
### Fixed
- WhatsApp order button `fee_name` truncated to 60 characters

## [2.3.26] - 2024-12-17
### Improved
- Replaced string interpolation with concatenation in gateway fields

## [2.3.25] - 2024-12-16
### Fixed
- WhatsApp order button payload structure uses `order_details`

## [2.3.24] - 2024-12-15
### Fixed
- Fatal error - Undefined constant "KDC_QTAP_URL"

## [2.3.23] - 2024-12-14
### Added
- Plugin default fallback image for {{site_icon}} and {{site_logo}}

### Removed
- Duplicate icon-qtap.svg

## [2.3.22] - 2024-12-13
### Improved
- Site icon detection with multiple fallback methods
- Checks WordPress site_icon option directly
- Fallback to favicon.png in root
- Fallback to theme directory favicon locations

## [2.3.21] - 2024-12-12
### Fixed
- Order button uses message recipient phone as `recipient_number` automatically

## [2.3.20] - 2024-12-11
### Added
- Dynamic header type detection for WhatsApp templates
- Location header support (`lat,long|name|address` format)
- Image header support (auto-detected from URL extension)
- Video header support (auto-detected from URL extension)
- Document header support (fallback for other URLs)

## [2.3.19] - 2024-12-10
### Fixed
- Order button type validation in admin UI

## [2.3.18] - 2024-12-09
### Added
- `{{site_icon}}` built-in variable - Site favicon/icon URL
- `{{site_logo}}` built-in variable - Site logo URL
- `kdc_qtap_get_site_icon_url()` helper function
- `kdc_qtap_get_site_logo_url()` helper function
- `kdc_qtap_site_icon_url` filter
- `kdc_qtap_site_logo_url` filter

## [2.3.17] - 2024-12-08
### Added
- Order button type for WhatsApp Pay (`order|{{payment_id}}`)
- `kdc_qtap_whatsapp_order_data` filter for order details
- Gateway-specific payment fields (Razorpay, PayU, Billdesk, Zaakpay)
- `kdc_qtap_whatsapp_order_action` filter
- `kdc_qtap_whatsapp_payment_gateway_config` filter

## [2.3.16] - 2024-12-07
### Added
- URL format modifiers - `:suffix`, `:path`, `:domain`, `:query`
- `{{payment_url:suffix}}` strips site URL

## [2.3.15] - 2024-12-06
### Fixed
- Case-insensitive variable matching

### Added
- Debug logging for variable replacement

## [2.3.14] - 2024-12-05
### Added
- `kdc_qtap_replace_variables()` - Global utility for variable replacement
- `kdc_qtap_replace_variables_in_array()` - Recursive array processing
- Format modifiers - `{{variable:format}}` syntax
- Amount formats - `:value`, `:lakh`, `:international`, `:words`, `:compact`
- Date formats - `:human`, `:relative`, `:Y-m-d`, `:short`, `:long`
- String formats - `:upper`, `:lower`, `:title`
- Indian numbering system (lakhs, crores)
- `kdc_qtap_currency_symbol` filter

## [2.3.13] - 2024-12-04
### Added
- `kdc_qtap_replace_variables()` - Channel-agnostic utility
- `kdc_qtap_replace_variables_in_array()` - Recursively replace in arrays
- `kdc_qtap_notification_variable_data` filter

## [2.3.12] - 2024-12-03
### Added
- `kdc_qtap_send_whatsapp_notification()` helper function
- Variable processing for `whatsapp_data` array
- Multi-recipient response logging
- Debug logging for variable replacement

## [2.3.11] - 2024-12-02
### Added
- Centralized variable replacement via `process_notification_variables()`
- `kdc_qtap_notification_variable_fields` filter
- `kdc_qtap_notification_variables_processed` filter

## [2.3.10] - 2024-12-01
### Fixed
- WhatsApp template value parses "template_name|language_code" format

## [2.3.9] - 2024-11-30
### Fixed
- WhatsApp body/button variables split by newlines when passed as strings

## [2.3.8] - 2024-11-29
### Fixed
- Removed duplicate disclosure icons in notification log details

## [2.3.7] - 2024-11-28
### Improved
- Button Variables field description shows per-line examples

## [2.3.6] - 2024-11-27
### Refactored
- Split `class-kdc-qtap-admin.php` into 9 trait files
- Main admin class reduced to ~1200 lines

### Added
- trait-kdc-qtap-admin-logs.php
- trait-kdc-qtap-admin-scheduled.php
- trait-kdc-qtap-admin-channels.php
- trait-kdc-qtap-admin-data.php
- trait-kdc-qtap-admin-apps.php
- trait-kdc-qtap-admin-ui.php
- trait-kdc-qtap-admin-notifications.php
- trait-kdc-qtap-admin-export.php
- trait-kdc-qtap-admin-import.php

## [2.3.5] - 2024-11-26
### Changed
- Renamed `assets/admin.css` to `assets/css/kdc-qtap-admin.css`

## [2.3.4] - 2024-11-25
### Added
- `kdc_qtap_render_whatsapp_header_field()` form helper
- `kdc_qtap_render_whatsapp_footer_field()` form helper

## [2.3.3] - 2024-11-24
### Added
- Test Template field for WhatsApp channel
- `kdc_qtap_render_whatsapp_template_field()` form helper
- `kdc_qtap_render_whatsapp_body_field()` form helper
- `kdc_qtap_render_whatsapp_buttons_field()` form helper

### Changed
- WhatsApp settings reordered
- Test notification uses configured Test Template

## [2.3.2] - 2024-11-23
### Removed
- Phone Number ID field from WhatsApp settings

### Changed
- API URL accepts full endpoint URL
- API Token used directly as Bearer token

## [2.3.1] - 2024-11-22
### Added
- WhatsApp Payment Provider dropdown
- WhatsApp Payment Configuration field
- `KDC_qTap_Channel_WhatsApp::get_payment_providers()`
- `KDC_qTap_Channel_WhatsApp::validate_payment_configuration()`

## [2.3.0] - 2024-11-21
### Refactored
- Modular channel architecture - each channel in separate class file
- Main notifications class reduced from ~3900 to ~1400 lines

### Added
- `KDC_qTap_Channel_Base` abstract class
- `KDC_qTap_Channel_Email` class
- `KDC_qTap_Channel_SMS` class
- `KDC_qTap_Channel_WhatsApp` class
- `KDC_qTap_Channel_Webhook` class
- `KDC_qTap_Channel_Log` class
- `kdc-qtap-notification-functions.php`
- `kdc-qtap-whatsapp-helpers.php`

## [2.2.32] - 2024-11-20
### Added
- `kdc_qtap_validate_whatsapp_template()` public helper
- `kdc_qtap_validate_whatsapp_buttons()` public helper
- `kdc_qtap_get_whatsapp_button_types()` for form builders

## [2.2.31] - 2024-11-19
### Added
- WhatsApp template validation (template_name|language_code format)
- Template name validation (alphanumeric and underscore)
- Button format validation (type|value format)
- Support for reply, url, phone, call, code, flow button types

## [2.2.30] - 2024-11-18
### Added
- Unified response structure for ALL notification channels
- Per-recipient results accessible via recipients array
- CLAUDE.md prompt file for AI assistants

## [2.2.29] - 2024-11-17
### Fixed
- WhatsApp notification request/response captured in logs
- Child plugin whatsapp_* fields preserved before processing

## [2.2.28] - 2024-11-16
### Added
- WhatsApp button component support in template messages
- Auto-detection of button type (URL vs Quick Reply)

## [2.2.27] - 2024-11-15
### Added
- WhatsApp template language code support
- Language code defaults to `en` when not specified

## [2.2.26] - 2024-11-14
### Fixed
- "Array to string conversion" warning in Notification Logs

## [2.2.25] - 2024-11-13
### Added
- WhatsApp multi-recipient support
- Auto-webhook feature for automatic webhook triggering
- Aggregated results for multi-recipient WhatsApp

## [2.2.24] - 2024-11-12
### Added
- Bulk delete confirmation dialog
- Validation to prevent bulk actions without selection

## [2.2.23] - 2024-11-11
### Fixed
- WhatsApp test notification validates phone number after cleaning

## [2.2.22] - 2024-11-10
### Fixed
- Bulk delete action in Notification Logs

## [2.2.21] - 2024-11-09
### Fixed
- WhatsApp Meta Cloud API URL construction

## [2.2.20] - 2024-11-08
### Added
- Request payload capture for all channels
- Collapsible Request and Response Body sections in view modal

## [2.2.19] - 2024-11-07
### Changed
- Webhook channel triggers last, after all other channels
- Webhook payload includes `channel_results`

## [2.2.18] - 2024-11-06
### Added
- Channel-wise API response capture in notification logs
- HTTP status code, response message, and body stored per channel

## [2.2.17] - 2024-11-05
### Improved
- Notification Logs channel column displays icons
- Channel icons show status (green/red)

## [2.2.16] - 2024-11-04
### Fixed
- JavaScript alert showing [object Object]

### Added
- Admin Mobile Number field to WhatsApp settings
- WhatsApp test notification sends hello_world template

## [2.2.15] - 2024-11-03
### Fixed
- Notification logging incorrectly parsing results structure

### Added
- Debug logging for webhook and test notifications

## [2.2.14] - 2024-11-02
### Added
- WhatsApp Gateway dropdown (Meta Cloud API, qTap.buzz, Custom)
- WhatsApp Phone Number ID field
- SMS Gateway dropdown (qTap.buzz, Custom)
- SMS API Secret and Sender ID fields

## [2.2.13] - 2024-11-01
### Simplified
- WhatsApp channel requires only API URL and API Token
- SMS channel requires only API URL and API Token

### Added
- `kdc_qtap_whatsapp_payload` filter
- `kdc_qtap_sms_payload` filter

## [2.2.12] - 2024-10-31
### Changed
- WhatsApp gateway options: META Cloud API, qTap.buzz, Custom
- SMS gateway options: qTap.buzz, Custom

### Fixed
- Test notification sends to ALL enabled channels

## [2.2.11] - 2024-10-30
### Improved
- Channel cards use WordPress admin theme color
- Disabled channels show greyed-out styling

## [2.2.10] - 2024-10-29
### Added
- Header HTML and Footer HTML fields for custom email templates
- Dynamic show/hide when WooCommerce template toggled

## [2.2.9] - 2024-10-28
### Improved
- Channel settings UI follows WordPress admin standards
- Channel toggle uses WordPress-style switch

### Changed
- Channel order: Email, WhatsApp, SMS, Webhook, Database Log

## [2.2.8] - 2024-10-27
### Removed
- Admin Notice channel

### Improved
- Channel settings UI with modern card-based layout
- Toggle switches with On/Off design

## [2.2.7] - 2024-10-26
### Fixed
- Fatal error with WC_Emails::style_inline()

## [2.2.6] - 2024-10-25
### Improved
- WhatsApp template data format with field-based approach

### Added
- Individual notification fields: whatsapp_template, whatsapp_header, whatsapp_body, whatsapp_footer, whatsapp_buttons
- `normalize_whatsapp_data()` method
- `parse_whatsapp_from_message()` for legacy support

## [2.2.5] - 2024-10-24
### Fixed
- Webhook headers not saved due to field name mismatch

## [2.2.4] - 2024-10-23
### Added
- SMS notification channel with gateway support
- WhatsApp notification channel with Business API support
- `kdc_qtap_send_sms` filter
- `kdc_qtap_send_whatsapp` filter
- SMS channel settings UI
- WhatsApp channel settings UI

## [2.2.3] - 2024-10-22
### Added
- Email template system with WooCommerce integration
- Custom Email Header HTML and Footer HTML fields
- Template variables {{site_name}}, {{site_url}}, {{current_year}}
- Distributed notification architecture for child plugins
- `kdc_qtap_process_scheduled_notifications` action
- `kdc_qtap_pending_notifications_summary` filter
- `kdc_qtap_daily_maintenance` cron event

### Fixed
- Missing `get_admin_theme_color()` method
- Cron schedule registration
- Infinite recursion in singleton pattern

## [2.2.0] - 2024-10-20
### Added
- Scheduled Notifications system
- "Scheduled" tab in Notifications page
- Stats cards for scheduled notifications
- Manual trigger functionality
- Bulk actions: Send Now, Cancel, Delete
- "Process All Due Now" button
- Automatic cron processing every 5 minutes
- `scheduled_at` column in notification log (DB v1.1.0)
- Helper functions: `kdc_qtap_schedule_notification()`, `kdc_qtap_trigger_notification()`
- New statuses: `scheduled`, `cancelled`
- Priority-based badge styling
- Human-readable time display

## [2.1.5] - 2024-10-18
### Fixed
- UI helper functions follow functional hierarchy
- All helpers validate variants and return valid classes

## [2.1.4] - 2024-10-17
### Fixed
- Removed max-width constraints from settings pages

### Improved
- Export Format options with card-style layout

## [2.1.3] - 2024-10-16
### Changed
- Renamed "Log Only" to "Database Log"

### Added
- Admin theme color support
- WCAG accessibility compliance with ARIA labels

## [2.1.2] - 2024-10-15
### Fixed
- Removed include reference to non-existent view file

## [2.1.1] - 2024-10-14
### Changed
- Renamed "Data Management" to "Import/Export"

### Added
- Notifications page tabs: Notification Logs, Channel Settings, Log Settings
- Import/Export page tabs: Data Retention, Import, Export

## [2.1.0] - 2024-10-13
### Changed
- Notifications is separate submenu page
- Data Management is separate submenu page

### Added
- Standalone page wrappers with "Back to Dashboard" navigation

## [2.0.5] - 2024-10-12
### Added
- Comprehensive notification template variables system
- Built-in variables for Site, User, Post, Date/Time
- `wp_meta_*` prefix for dynamic WordPress data
- Variable groups for organized documentation
- `kdc_qtap_register_notification_variables` action
- `kdc_qtap_register_notification_variable()` helper
- `kdc_qtap_register_notification_variable_group()` helper
- `kdc_qtap_process_notification_template()` helper
- Click-to-copy functionality for variable codes

## [2.0.4] - 2024-10-11
### Added
- Channel Settings section in Notification Log tab
- Admin UI to enable/disable notification channels
- Email channel settings (From Name, From Email)
- Webhook channel settings (URL, HTTP Method, Custom Headers)
- "Send Test Notification" button
- Hook `kdc_qtap_channel_settings_fields`
- Hook `kdc_qtap_save_channel_settings`

## [2.0.3] - 2024-10-10
### Added
- "Auto-Cleanup" toggle in Log Settings
- Retention Period field hidden when auto-cleanup disabled

## [2.0.2] - 2024-10-09
### Fixed
- Notification Log tab UI layout issues

### Improved
- Complete redesign of Notification Log interface
- Filters in dedicated card
- Empty state with icon and message
- Log Settings in collapsible section

## [2.0.1] - 2024-10-08
### Fixed
- Translation loading timing for WordPress 6.7+

## [2.0.0] - 2024-10-07
### Major
- Notification Log System with database storage
- Custom database table `{prefix}_kdc_qtap_notification_log`
- "Notification Log" admin tab
- Statistics cards
- Filters: Status, Type, Priority, Date Range, Search
- Sortable columns
- Pagination
- Bulk actions: Delete, Resend Failed
- Export to CSV
- Clear All Logs
- Log Settings: Enable/disable, retention period
- Daily cron for automatic cleanup
- AJAX detail modal

## [1.9.21] - 2024-10-06
### Added
- Extensible notification system with multi-channel support
- Built-in email channel
- Built-in admin notice channel
- Built-in webhook channel
- Built-in log channel
- `kdc_qtap_send_notification()` helper
- `kdc_qtap_register_notification_channel()`
- `kdc_qtap_register_notification_type()`
- Priority levels: low, normal, high, urgent
- Template system with {{placeholder}} variables

## [1.9.20] - 2024-10-05
### Fixed
- `tel:` and `mailto:` links inherit theme color

## [1.9.19] - 2024-10-04
### Added
- Link/anchor styling with primary color
- `.kdc-qtap-link` class

## [1.9.18] - 2024-10-03
### Fixed
- Input fields use explicit text color

## [1.9.17] - 2024-10-02
### Changed
- UI Framework uses color pickers and typography controls

### Added
- Color settings panel
- Typography settings
- Live preview panel
- Reset to Defaults button
- Full HTML5 input type support

## [1.9.16] - 2024-10-01
### Added
- "UI Framework" tab in settings
- UI Mode toggle
- Automatic theme environment detection
- `kdc_qtap_detect_theme_environment()`
- `kdc_qtap_resolve_ui_class()`
- `kdc_qtap_get_message_class()`
- `kdc_qtap_is_block_theme()`

## [1.9.15] - 2024-09-30
### Improved
- Button and input CSS self-contained

### Added
- CSS custom properties
- Button and input states
- Input validation states
- `kdc_qtap_get_select_class()`
- `kdc_qtap_get_textarea_class()`
- High contrast mode support
- Reduced motion support

## [1.9.14] - 2024-09-29
### Added
- Shared frontend UI component library
- Frontend JavaScript helpers
- PHP helper functions for frontend rendering
- `kdc_qtap_enqueue_frontend_components()`
- `kdc_qtap_get_button_class()`
- `kdc_qtap_get_input_class()`
- `kdc_qtap_render_message()`
- `kdc_qtap_render_login_required()`
- `kdc_qtap_render_loading()`
- `kdc_qtap_render_empty_state()`
- `kdc_qtap_render_badge()`

## [1.9.13] - 2024-09-28
### Added
- `kdc_qtap_render_export_filter_panel()` helper
- `kdc_qtap_render_export_filter_field()` helper
- Support for text, number, date, select, multiselect, radio_group, checkbox_group
- Toggle all functionality
- Date/number range validation

## [1.9.12] - 2024-09-27
### Added
- Comprehensive CSS classes for export filter panels
- Responsive styles for filter fields

## [1.9.11] - 2024-09-26
### Added
- Parent plugin version in dashboard header

## [1.9.10] - 2024-09-25
### Fixed
- Export filter panels contained within checkbox items

## [1.9.9] - 2024-09-24
### Added
- `kdc_qtap_export_option_after` action hook
- CSS for `.kdc-export-sub-options`
- JavaScript for `is-checked` class toggle

## [1.9.8] - 2024-09-23
### Changed
- JSON-only export options disabled instead of hidden

## [1.9.7] - 2024-09-22
### Added
- `kdc_qtap_export_groups_rendered` JavaScript event

## [1.9.6] - 2024-09-21
### Changed
- JSON export uses minified format

## [1.9.5] - 2024-09-20
### Added
- `json_only` option in export groups
- "(JSON only)" badge for JSON-only options

## [1.9.4] - 2024-09-19
### Changed
- Import report UI uses dashicons
- Import report uses admin theme color

## [1.9.3] - 2024-09-18
### Changed
- Import report download always available

### Added
- Track all row statuses: IMPORTED, UPDATED, SKIPPED, ERROR, UNKNOWN

## [1.9.2] - 2024-09-17
### Fixed
- Import report download shows for skipped rows

### Added
- `row_skipped` tracking array

## [1.9.1] - 2024-09-16
### Added
- CSV import option "Create new WordPress user if not found"
- CSV import option "Send WordPress welcome email to new users"

## [1.9.0] - 2024-09-15
### Fixed
- CSV import report showing wrong user emails

## [1.8.9] - 2024-09-14
### Restored
- qTap App settings export checkbox (JSON only)

## [1.8.8] - 2024-09-13
### Changed
- qTap settings only exported with JSON format

## [1.8.7] - 2024-09-12
### Removed
- qtap_settings.csv from CSV import templates

## [1.8.6] - 2024-09-11
### Improved
- CSV sample template download link styling

## [1.8.5] - 2024-09-10
### Added
- Comprehensive developer documentation

## [1.8.4] - 2024-09-09
### Added
- Sample CSV template download link

## [1.8.3] - 2024-09-08
### Fixed
- Missing closing bracket on wrapper div

## [1.8.2] - 2024-09-07
### Added
- qTap SVG logo to settings page header

## [1.8.1] - 2024-09-06
### Improved
- Import success message displays within Import Data section

## [1.8.0] - 2024-09-05
### Improved
- App cards have consistent height with flexbox

## [1.7.9] - 2024-09-04
### Improved
- qTap App export group uses inline SVG logo

## [1.7.8] - 2024-09-03
### Fixed
- phpcs nonce verification warning

## [1.7.6] - 2024-09-02
### Fixed
- WordPress Plugin Checker compliance

## [1.7.5] - 2024-09-01
### Added
- Export groups with per-plugin styling
- `kdc_qtap_render_export_group()` helper
- `kdc_qtap_get_plugin_data()` helper
- Toggle all functionality

## [1.7.4] - 2024-08-31
### Improved
- Data Selection export group uses admin theme color

## [1.7.3] - 2024-08-30
### Improved
- Progress bar uses WordPress admin theme color

## [1.7.2] - 2024-08-29
### Added
- Live progress page for CSV import
- Import counters (imported/updated/skipped/errors)
- Estimated time remaining
- Batch processing via AJAX
- Download button for import report

## [1.7.1] - 2024-08-28
### Improved
- CSV mapping on dedicated page

### Added
- Import report CSV with errors

## [1.7.0] - 2024-08-27
### Added
- CSV import with interactive header mapping
- UTF-8 BOM detection for Google Sheets/Excel
- Auto-mapping of CSV headers via aliases
- Import options: skip duplicates, update existing
- `kdc_qtap_csv_import_targets` filter
- `kdc_qtap_process_csv_import` filter

## [1.6.0] - 2024-08-26
### Added
- Export format selection (JSON/CSV)
- CSV export for Google Sheets
- Sample CSV template download
- `kdc_qtap_export_csv_data` filter
- `kdc_qtap_sample_csv_data` filter
- Multi-sheet CSV export as ZIP
- UTF-8 BOM for Excel compatibility

## [1.5.0] - 2024-08-25
### Added
- REST API Access settings
- `kdc_qtap_can_access_rest_api()` helper
- `kdc_qtap_rest_permission_check()` callback
- `kdc_qtap_get_rest_api_roles()`
- `kdc_qtap_rest_permission_check` filter
- `kdc_qtap_rest_api_roles` filter

## [1.4.2] - 2024-08-24
### Added
- Warning alert for "Remove all qTap data" checkbox

## [1.4.1] - 2024-08-23
### Fixed
- Prefixed global variables in uninstall.php
- Added kdc-qtap-accessible wrapper class

## [1.4.0] - 2024-08-22
### Added
- Data Management tab
- "Remove data on uninstall" setting
- Export functionality with `kdc_qtap_export_data` filter
- Import functionality with `kdc_qtap_process_import` action
- `kdc_qtap_should_remove_data()` helper
- `kdc_qtap_get_settings()` helper
- Multiple action and filter hooks

## [1.3.0] - 2024-08-21
### Added
- Tabbed interface with "Apps List" and "Common Settings"
- Accessibility Mode setting
- `kdc_qtap_is_accessibility_enabled()` helper

## [1.2.3] - 2024-08-20
### Fixed
- WordPress.org Plugin Checker compliance

## [1.2.2] - 2024-08-19
### Added
- WCAG Level AAA accessibility compliance
- Skip link for keyboard navigation
- ARIA landmarks and labels
- Screen reader announcements
- Enhanced focus indicators
- Reduced motion support
- High contrast mode support

## [1.2.1] - 2024-08-18
### Fixed
- Removed load_plugin_textdomain()
- Added proper readme.txt

## [1.2.0] - 2024-08-17
### Improved
- Simplified menu icon using base64 SVG

## [1.1.2] - 2024-08-16
### Fixed
- WordPress 6.7+ translation loading

## [1.1.0] - 2024-08-15
### Added
- Extensibility hooks for child plugins
- `kdc_qtap_loaded` action hook
- `kdc_qtap_admin_menu` action hook

## [1.0.0] - 2024-08-14
### Initial Release
- Central dashboard for qTap apps
- App registration API
- Visual app cards
